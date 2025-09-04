# molecule

## 1) Prerequisitos: Instalar en un entorno virtual (recomendado)

Instala las ultimas versiones (puede no ser compatible con hosts con version de python anterior)
````bash
python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install ansible-core
pip install "molecule>=6.0.0"
pip install "molecule-plugins[docker]" ansible-lint yamllint pytest testinfra
````

Comandos para crear un venv limpio con la rama de Ansible 9 (core 2.16, compatible con Python 3.6 en los managed nodes) y Molecule 6.x:

````bash
# 1. Crear el venv (puedes usar Python 3.9, 3.10 o 3.11 en tu portátil)
python3 -m venv .venv
source .venv/bin/activate

# 2. Actualizar pip y setuptools
pip install --upgrade pip setuptools wheel

# 3. Instalar ansible 9.x (trae ansible-core 2.16.x)
pip install  "ansible==9.*" 

# 4. Instalar molecule 6.x
pip install "molecule>=6,<7"

# 5. (Opcional) Instalar controladores de molecule, por ejemplo docker
pip install "molecule-plugins[docker]>=23.5.0,<24.0.0" \
            "ansible-lint>=24.0.0,<25.0.0" \
            "yamllint>=1.30.0,<2.0.0" \
            "pytest>=7.0.0,<8.0.0" \
            "testinfra>=6.0.0,<7.0.0"
````


Así dejas todas las dependencias dentro de .venv/ y no ensucias el Python global.
Después, ejecutas Molecule con:
````bash
.venv/bin/molecule test
````

## 2) Inicializar Molecule en tu rol

Desde la raíz del rol (donde está meta/, tasks/, etc.):

````bash
molecule init scenario default -d docker
````

##  3) molecule/default/molecule.yml

Prueba 3 plataformas y usa Ansible como provisioner y verifier:

````yaml
dependency:
  name: galaxy

driver:
  name: docker

platforms:
  # RHEL 9 (UBI9)
  - name: ubi9
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    command: /sbin/init
    privileged: true
    cgroupns_mode: host
    environment:
      container: docker
    tmpfs:
      - /run
      - /run/lock
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    pre_build_image: true

  # RHEL 8 (UBI8)
  - name: ubi8
    image: registry.access.redhat.com/ubi8/ubi-init:latest
    command: /sbin/init
    privileged: true
    cgroupns_mode: host
    environment:
      container: docker
    tmpfs:
      - /run
      - /run/lock
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    pre_build_image: true

  # # RHEL 7 (usa CentOS 7 como equivalente para pruebas)
  # - name: centos7
  #   image: geerlingguy/docker-centos7-ansible:latest
  #   command: /sbin/init
  #   privileged: true
  #   cgroupns_mode: host
  #   environment:
  #     container: docker
  #   tmpfs:
  #     - /run
  #     - /run/lock
  #   volumes:
  #     - /sys/fs/cgroup:/sys/fs/cgroup:rw
  #   pre_build_image: true

  # openSUSE (familia SUSE)
  - name: opensuse
    image: geerlingguy/docker-opensuseleap15-ansible:latest
    command: /sbin/init
    privileged: true
    cgroupns_mode: host
    environment:
      container: docker
    tmpfs:
      - /run
      - /run/lock
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    pre_build_image: true

  # Ubuntu 22.04
  - name: ubuntu2204
    image: geerlingguy/docker-ubuntu2204-ansible:latest
    command: /sbin/init
    privileged: true
    cgroupns_mode: host
    environment:
      container: docker
    tmpfs:
      - /run
      - /run/lock
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    pre_build_image: true

provisioner:
  name: ansible
  config_options:
    defaults:
      remote_tmp: /tmp
  inventory:
    hosts:
      all:
        vars:
          ansible_remote_tmp: /tmp

          # --- Variables del rol / verify ---
          node_exporter_state: present
          node_exporter_user: prometheus
          node_exporter_group: prometheus

          node_exporter_port: 9100
          node_exporter_version: "1.9.1"

          # En tests no hay Prometheus real (desactiva file_sd checks)
          prometheus_server: "localhost"
          prometheus_file_sd_check: false
          prometheus_file_sd_dir: "/tmp/prometheus/file_sd/node"

          node_labels:
            job: node_exporter
            app: "SYSTEM"
            env: "TEST"
  playbooks:
    prepare: prepare.yml
    converge: converge.yml
    verify: verify.yml
  lint:
    name: ansible-lint

verifier:
  name: ansible

scenario:
  test_sequence:
    - lint
    - destroy
    - create
    - prepare
    - converge
    - idempotence
    - verify
    - destroy

````

Usamos imágenes de Jeff Geerling con systemd para que los servicios funcionen.
Si tu rol usa become: true, estás cubierto por defecto en estas imágenes.


##  4) molecule/default/converge.yml

Ejecuta el rol y deja instaladas dependencias mínimas (curl, etc.) para las comprobaciones:

````yaml
---
- name: Converge
  hosts: all
  become: true
  vars:
    node_exporter_state: present
  pre_tasks:
    - name: Ensure basic tools are present (curl, ps, netstat/iproute)
      package:
        name: "{{ item }}"
        state: present
      loop:
        # - curl
        - procps-ng
        - iproute
        - net-tools
      when: ansible_os_family in ['RedHat']

    - name: Ensure basic tools (Debian/Ubuntu)
      apt:
        name:
          - curl
          - procps
          - iproute2
          - net-tools
        state: present
        update_cache: true
      when: ansible_os_family == 'Debian'

  tasks:
    - name: Run role under test (by absolute path)
      ansible.builtin.include_role:
        name: "{{ lookup('env','MOLECULE_PROJECT_DIRECTORY') }}"


````

## 5) molecule/default/verify.yml

Comprueba: usuario/grupo, binario/servicio, puerto 9100, y que se generó el fichero file_sd con labels esperadas.

````yaml
---
- name: Verify
  hosts: all
  become: true
  gather_facts: true

  vars:
    # Leer overrides desde inventory sin recursión
    ne_state: "{{ hostvars[inventory_hostname].node_exporter_state | default('present') }}"
    ne_user:  "{{ hostvars[inventory_hostname].node_exporter_user  | default('node_exporter') }}"
    ne_port:  "{{ hostvars[inventory_hostname].node_exporter_port  | default(9100) }}"

    # Comprobación opcional de file_sd en el servidor de Prometheus
    prometheus_file_sd_check: "{{ hostvars[inventory_hostname].prometheus_file_sd_check | default(false) | bool }}"
    prometheus_server: "{{ hostvars[inventory_hostname].prometheus_server | default(omit) }}"
    prometheus_file_sd_dir: "{{ hostvars[inventory_hostname].prometheus_file_sd_dir | default('/etc/prometheus/file_sd/node') }}"

    metrics_url: "http://127.0.0.1:{{ ne_port }}/metrics"

  tasks:
    # --- Usuario ---
    - name: Getent passwd (state=present)
      ansible.builtin.getent:
        database: passwd
        key: "{{ ne_user }}"
      register: ge_passwd
      when: ne_state == 'present'
      changed_when: false

    - name: Assert user exists (state=present)
      ansible.builtin.assert:
        that:
          - ge_passwd.ansible_facts.getent_passwd[ne_user] is defined
        fail_msg: "User {{ ne_user }} does not exist"
      when: ne_state == 'present'

    - name: Getent passwd (state=absent)
      ansible.builtin.getent:
        database: passwd
        key: "{{ ne_user }}"
      register: ge_passwd_absent
      failed_when: false
      when: ne_state == 'absent'
      changed_when: false

    - name: Assert user removed (state=absent)
      ansible.builtin.assert:
        that:
          - ge_passwd_absent.ansible_facts.getent_passwd is not defined
            or ge_passwd_absent.ansible_facts.getent_passwd[ne_user] is not defined
        fail_msg: "User {{ ne_user }} still exists but should be absent"
      when: ne_state == 'absent'

    # --- Servicio / Proceso ---
    - name: Gather service facts
      ansible.builtin.service_facts:
      register: svc
      changed_when: false

    - name: Detect node_exporter service name
      ansible.builtin.set_fact:
        ne_service_name: >-
          {{
            'node_exporter' if 'node_exporter.service' in svc.ansible_facts.services
            else (
              'prometheus-node-exporter' if 'prometheus-node-exporter.service' in svc.ansible_facts.services
              else 'node_exporter'
            )
          }}

    - name: Check process running (fallback)
      ansible.builtin.shell: "ps -C node_exporter -o pid= | head -n1"
      register: ps_node
      changed_when: false
      when: ne_state == 'present'

    - name: Assert process/service running (state=present)
      ansible.builtin.assert:
        that:
          - ps_node.rc == 0
            or (svc.ansible_facts.services.get(ne_service_name + '.service', {}).state | default('')) in ['running','started','active']
        fail_msg: "node_exporter not running. ps rc={{ ps_node.rc }}, service={{ svc.ansible_facts.services.get(ne_service_name + '.service', {}).state | default('unknown') }}"
      when: ne_state == 'present'

    - name: Assert service/process stopped (state=absent)
      ansible.builtin.assert:
        that:
          - (svc.ansible_facts.services.get(ne_service_name + '.service', {}).state | default('stopped')) not in ['running','started','active']
        fail_msg: "Service {{ ne_service_name }} is still running but should be absent"
      when: ne_state == 'absent'

    # --- Puerto (usando wait_for) ---
    # Estado PRESENTE: algo debe estar escuchando en 9100
    - name: Ensure node_exporter is listening on {{ ne_port }} (IPv4) (state=present)
      ansible.builtin.wait_for:
        host: 127.0.0.1
        port: "{{ ne_port }}"
        state: started
        timeout: 2
      when: ne_state == 'present'

    - name: Ensure node_exporter is listening on {{ ne_port }} (IPv6) (state=present)
      ansible.builtin.wait_for:
        host: ::1
        port: "{{ ne_port }}"
        state: started
        timeout: 2
      when: ne_state == 'present'
      ignore_errors: true   # Por si el servicio sólo escucha en IPv4 o no hay loopback IPv6

    # Estado AUSENTE: nada debe estar escuchando en 9100
    - name: Ensure port {{ ne_port }} is closed (IPv4) (state=absent)
      ansible.builtin.wait_for:
        host: 127.0.0.1
        port: "{{ ne_port }}"
        state: stopped
        timeout: 2
      when: ne_state == 'absent'

    - name: Ensure port {{ ne_port }} is closed (IPv6) (state=absent)
      ansible.builtin.wait_for:
        host: ::1
        port: "{{ ne_port }}"
        state: stopped
        timeout: 2
      when: ne_state == 'absent'
      ignore_errors: true

    # --- Métricas HTTP ---
    - name: Wait for metrics endpoint (max 30s) (state=present)
      ansible.builtin.uri:
        url: "{{ metrics_url }}"
        return_content: true
        status_code: 200
      register: metrics_resp
      retries: 6
      delay: 5
      until: metrics_resp.status == 200
      changed_when: false
      when: ne_state == 'present'

    - name: Assert metrics reachable and contain build info (state=present)
      ansible.builtin.assert:
        that:
          - metrics_resp.status == 200
          - "'node_exporter_build_info' in metrics_resp.content"
        fail_msg: "Metrics endpoint not reachable or missing build metric. status={{ metrics_resp.status }}"
      when: ne_state == 'present'

    # --- File SD (en servidor Prometheus) ---
    - name: Stat file_sd dir on Prometheus server
      ansible.builtin.stat:
        path: "{{ prometheus_file_sd_dir }}"
      register: file_sd_stat
      when: prometheus_file_sd_check
      delegate_to: "{{ prometheus_server }}"
      run_once: true

    - name: Assert file_sd dir exists on Prometheus server
      ansible.builtin.assert:
        that:
          - file_sd_stat.stat.exists
          - file_sd_stat.stat.isdir
        fail_msg: "file_sd dir {{ prometheus_file_sd_dir }} not found on {{ prometheus_server }}"
      when: prometheus_file_sd_check
      run_once: true

    - name: Find target files on Prometheus server
      ansible.builtin.find:
        paths: "{{ prometheus_file_sd_dir }}"
        patterns: "node-*.yml"
      register: targets_files
      when: prometheus_file_sd_check
      delegate_to: "{{ prometheus_server }}"
      run_once: true

    - name: Assert at least one target file was created (Prometheus server)
      ansible.builtin.assert:
        that:
          - (targets_files.matched | int) >= 1
        fail_msg: "No file_sd target file found in {{ prometheus_file_sd_dir }} on {{ prometheus_server }}"
      when: prometheus_file_sd_check
      run_once: true

    - name: Read one target file (Prometheus server)
      ansible.builtin.slurp:
        src: "{{ targets_files.files[0].path }}"
      register: target_content
      when: prometheus_file_sd_check
      delegate_to: "{{ prometheus_server }}"
      run_once: true

    - name: Parse YAML content (Prometheus server)
      ansible.builtin.set_fact:
        parsed: "{{ target_content.content | b64decode | from_yaml }}"
      when: prometheus_file_sd_check
      run_once: true

    - name: Validate YAML structure and labels (Prometheus server)
      ansible.builtin.assert:
        that:
          - parsed is iterable
          - parsed | length > 0
          - parsed[0] is mapping
          - parsed[0].labels is defined
          - parsed[0].labels.job == "node_exporter"
          - parsed[0].labels.app is defined
          - parsed[0].labels.env is defined
          - parsed[0].targets is defined
          - parsed[0].targets | length >= 1
        fail_msg: "file_sd invalid on {{ prometheus_server }}: {{ parsed | to_nice_yaml }}"
      when: prometheus_file_sd_check
      run_once: true


````
## 6) Prepare

````yaml
---
- name: Prepare containers (bootstrap Python first)
  hosts: all
  become: false
  gather_facts: false

  vars:
    # Si alguna imagen SUSE da problemas de bloqueo en zypper, sube retries/delay
    _retries: 5
    _delay: 3

  tasks:
    - name: Detect OS (no Python yet)
      ansible.builtin.raw: |
        . /etc/os-release 2>/dev/null || true
        echo "ID=${ID}"
        echo "ID_LIKE=${ID_LIKE}"
        echo "PRETTY_NAME=${PRETTY_NAME}"
      register: osrel
      changed_when: false

    - name: Normalize flags per distro family
      ansible.builtin.set_fact:
        is_rh: "{{ (osrel.stdout | lower) is search('rhel') or (osrel.stdout | lower) is search('ubi') or (osrel.stdout | lower) is search('fedora') }}"
        is_deb: "{{ (osrel.stdout | lower) is search('debian') or (osrel.stdout | lower) is search('ubuntu') }}"
        is_suse: "{{ (osrel.stdout | lower) is search('suse') or (osrel.stdout | lower) is search('opensuse') }}"
      changed_when: false

    # -------------------- SUSE --------------------
    - name: zypper refresh (SUSE)
      ansible.builtin.raw: |
        zypper --non-interactive refresh
      register: zref
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: zref.rc == 0
      when: is_suse
      changed_when: false

    - name: Install python3 on SUSE
      ansible.builtin.raw: |
        command -v python3 >/dev/null 2>&1 || zypper --non-interactive install -y --no-recommends python3 python3-base
      register: zpy
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: zpy.rc == 0
      when: is_suse

    - name: Install base tools on SUSE
      ansible.builtin.raw: |
        zypper --non-interactive install -y --no-recommends curl iproute2 procps net-tools tar sudo
      register: ztools
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: ztools.rc == 0
      when: is_suse

    - name: Install compression utils on SUSE (tar/gzip/xz/bzip2/zstd/unzip)
      ansible.builtin.raw: |
        zypper --non-interactive install -y --no-recommends \
          tar gzip xz bzip2 zstd unzip
      register: zcomp
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: zcomp.rc == 0
      when: is_suse

    # -------------------- RedHat / UBI9 --------------------
    - name: Install python3 on UBI9/RHEL9
      ansible.builtin.raw: |
        command -v python3 >/dev/null 2>&1 || dnf -y install python3 python3-libselinux python3-libsemanage
      register: rpy
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: rpy.rc == 0
      when: is_rh

    - name: Install base tools on UBI9/RHEL9 (avoid curl conflict with curl-minimal)
      ansible.builtin.raw: |
        dnf -y install iproute procps-ng net-tools tar sudo
      register: rtools
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: rtools.rc == 0
      when: is_rh

    - name: Ensure dbus is present on UBI9/RHEL9
      ansible.builtin.raw: |
        dnf -y install dbus >/dev/null 2>&1 || true
      when: is_rh

    # -------------------- Debian / Ubuntu --------------------
    - name: apt-get update (Debian/Ubuntu)
      ansible.builtin.raw: |
        apt-get update -y
      register: aupd
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: aupd.rc == 0
      when: is_deb
      changed_when: false

    - name: Install python3 on Debian/Ubuntu
      ansible.builtin.raw: |
        command -v python3 >/dev/null 2>&1 || apt-get install -y python3 python3-apt
      register: apy
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: apy.rc == 0
      when: is_deb

    - name: Install base tools on Debian/Ubuntu
      ansible.builtin.raw: |
        apt-get install -y curl iproute2 procps net-tools tar sudo
      register: atools
      retries: "{{ _retries }}"
      delay: "{{ _delay }}"
      until: atools.rc == 0
      when: is_deb

    # -------------------- Ensure Python link --------------------
    - name: Ensure /usr/bin/python exists (symlink to python3 if missing)
      ansible.builtin.raw: |
        [ -x /usr/bin/python ] || (command -v python3 >/dev/null 2>&1 && ln -s "$(command -v python3)" /usr/bin/python || true)
      changed_when: false

    # -------------------- Ya hay Python: recoger facts --------------------
    - name: Gather facts
      ansible.builtin.setup:

````

## 6) Ajustes en tu rol para que la prueba pase

En tu rol ya tienes algo así (según lo que hablamos):

Tareas que crean el fichero file_sd con:
````yaml
delegate_to: "{{ prometheus_server }}"
dest: "{{ prometheus_file_sd_dir }}/node-{{ inventory_hostname }}.yml"
````

Un handler “Reload Prometheus”. En los tests no tenemos Prometheus, así que:

O noifies ese handler pero que sea “no-op” si no existe Prometheus.

O añade una condición al handler/notify vía variable (prometheus_reload_enabled: false en Molecule) y rodea el handler/tarea con when: prometheus_reload_enabled | default(false).

Ejemplo (handler seguro):

````yaml
# handlers/main.yml

- name: Reload Prometheus
  ansible.builtin.debug:
    msg: "Skipping reload in Molecule (no Prometheus here)"
  when: not (prometheus_reload_enabled | default(false))
````

## 7) Ejecutar
# En el directorio del rol
````bash
molecule test        #Ejecutar todo el ciclo (lint, crear contenedores, preparar, aplicar rol, verificar, destruir)

# o en ciclos rápidos

molecule create      # crea los contenedores (ubi9, ubuntu2204)
molecule prepare     # instala python, curl, etc. dentro de ellos
molecule converge    # aplica tu rol node_exporter
molecule idempotence # comprueba que el rol es idempotente
molecule verify      # ejecuta verify.yml (checks de user, servicio, puerto, file_sd)
molecule destroy     # elimina los contenedores
````
