# molecule

## 1) Prerequisitos: Instalar en un entorno virtual (recomendado)
````bash
python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip
pip install ansible-core
pip install "molecule>=6.0.0"
pip install "molecule-plugins[docker]" ansible-lint yamllint pytest testinfra
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
---
dependency:
  name: galaxy

driver:
  name: docker

platforms:
  - name: rockylinux9
    image: "geerlingguy/docker-rockylinux9-ansible:latest"
    command: /usr/sbin/init
    privileged: true
    pre_build_image: true
  - name: almalinux8
    image: "geerlingguy/docker-almalinux8-ansible:latest"
    command: /usr/sbin/init
    privileged: true
    pre_build_image: true
  - name: ubuntu2204
    image: "geerlingguy/docker-ubuntu2204-ansible:latest"
    command: /sbin/init
    privileged: true
    pre_build_image: true

provisioner:
  name: ansible
  inventory:
    hosts:
      all:
        vars:
          # Ajustes de test: que el rol apunte al propio contenedor
          node_exporter_port: 9100
          prometheus_server: "localhost"
          prometheus_file_sd_dir: "/tmp/prometheus/file_sd/node"
          node_labels:
            job: node_exporter
            app: "SYSTEM"      # en tus tests puedes sobreescribirlo con group_vars
            env: "TEST"
  playbooks:
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
    # Puedes sobreescribir variables aquí si necesitas
    node_exporter_state: present
  pre_tasks:
    - name: Ensure basic tools are present (curl, ps, netstat/iproute)
      package:
        name: "{{ item }}"
        state: present
      loop:
        - curl
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

  roles:
    - role: node_exporter
    - 

````

## 5) molecule/default/verify.yml

Comprueba: usuario/grupo, binario/servicio, puerto 9100, y que se generó el fichero file_sd con labels esperadas.

````yaml
---
- name: Verify
  hosts: all
  become: true
  tasks:
    - name: Check user exists
      ansible.builtin.command: id node_exporter
      register: user_id
      changed_when: false

    - name: Assert user exists
      ansible.builtin.assert:
        that:
          - user_id.rc == 0
        fail_msg: "User node_exporter does not exist"

    - name: Check node_exporter process
      ansible.builtin.shell: "ps aux | grep -w node_exporter | grep -v grep"
      register: ps_node
      changed_when: false

    - name: Assert process running
      ansible.builtin.assert:
        that:
          - ps_node.rc == 0
        fail_msg: "node_exporter process not running"

    - name: Check port 9100 listening
      ansible.builtin.shell: |
        (ss -lntp || netstat -lntp) 2>/dev/null | grep -q ':9100 '
      register: port_check
      changed_when: false

    - name: Assert port is listening
      ansible.builtin.assert:
        that:
          - port_check.rc == 0
        fail_msg: "Port 9100 is not listening"

    - name: Curl metrics endpoint
      ansible.builtin.command: "curl -sSf http://127.0.0.1:9100/metrics"
      register: curl_metrics
      changed_when: false

    - name: Assert metrics reachable
      ansible.builtin.assert:
        that:
          - curl_metrics.rc == 0
          - "'node_exporter_build_info' in curl_metrics.stdout"
        fail_msg: "Metrics endpoint not reachable or unexpected output"

    # File SD checks (el rol delega a prometheus_server=localhost en tests)
    - name: Check file_sd directory exists
      ansible.builtin.file:
        path: "/tmp/prometheus/file_sd/node"
        state: directory
      check_mode: yes
      register: file_sd_dir

    - name: Assert file_sd dir exists
      ansible.builtin.assert:
        that:
          - file_sd_dir.state == "directory"
        fail_msg: "file_sd dir not created"

    - name: Find target files
      ansible.builtin.find:
        paths: "/tmp/prometheus/file_sd/node"
        patterns: "node-*.yml"
      register: targets_files

    - name: Assert at least one target file was created
      ansible.builtin.assert:
        that:
          - targets_files.matched | int >= 1
        fail_msg: "No file_sd target file found"

    - name: Read one target file
      ansible.builtin.slurp:
        src: "{{ targets_files.files[0].path }}"
      register: target_content
      when: targets_files.matched | int >= 1

    - name: Parse YAML content
      ansible.builtin.set_fact:
        parsed: "{{ target_content['content'] | b64decode | from_yaml }}"
      when: targets_files.matched | int >= 1

    - name: Assert labels inside file_sd
      ansible.builtin.assert:
        that:
          - parsed[0].labels.job == "node_exporter"
          - parsed[0].labels.env == "TEST"
          - parsed[0].labels.app is defined
          - parsed[0].targets | length >= 1
        fail_msg: "file_sd labels/targets not as expected"
      when: targets_files.matched | int >= 1

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
molecule test
# o en ciclos rápidos
molecule create
molecule converge
molecule verify
molecule destroy
````
