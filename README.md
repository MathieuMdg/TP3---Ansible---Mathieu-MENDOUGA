# TP3: Ansible, Docker and application deployment

## Environment

| Item | Value |
|---|---|
| Control machine | Ubuntu VM (`Ubuntu-Ansible`) on VMware Workstation, hosted on Windows |
| File editing | Visual Studio Code on Windows, synchronized with the VM through Git |
| Target server | Debian server provided by Takima, reachable over SSH with a private key |
| Ansible | 2.10.8 |
| Docker collection | `community.docker` 2.7.0 |
| Docker SDK Python (server) | virtualenv `/opt/docker_venv` |

The Docker collection is installed in a version compatible with Ansible 2.10:

```bash
ansible-galaxy collection install community.docker:2.7.0 --force
```

## Project structure

```text
.
├── inventories/
│   └── setup.yml          # host, SSH user, private key
├── group_vars/
│   └── all.yml            # shared variables (images, names, database)
├── roles/
│   ├── docker/            # Docker and Python SDK installation
│   ├── docker_network/    # Docker network app_network
│   ├── database/          # PostgreSQL container
│   ├── app/               # API container
│   └── proxy/             # httpd container
├── playbook.yml           # main playbook
└── README.md
```

The five roles were generated with `ansible-galaxy init roles/<name>`. Only the `tasks/main.yml` files are used; the other generated files (defaults, handlers, meta, tests, vars) are kept as they are.

## Question 3-1: inventory and basic commands

The `inventories/setup.yml` inventory describes the server and how to connect to it (SSH user, path to the private key, host), grouped under a `prod` group.

```yaml
all:
  vars:
    ansible_user: <user>
    ansible_ssh_private_key_file: <path to the private key>
  children:
    prod:
      hosts: <hostname or IP>
```

Commands used:

| Command | Purpose |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Checks the SSH connection and that Python is present on the server |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Displays facts |
| `ansible all -i inventories/setup.yml -m setup -a 'filter=ansible_python*'` | Displays the Python used by Ansible |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | Removes Apache2 from the server |


## Question 3-2: playbook and roles

The `playbook.yml` playbook applies to all hosts, gathers facts and escalates privileges (`become`). It calls the roles in this order:

```yaml
---
- hosts: all
  gather_facts: true
  become: true
  vars:
    ansible_python_interpreter: /usr/bin/python3
  roles:
    - docker
    - docker_network
    - database
    - app
    - proxy
```

The `docker` role follows the official Docker documentation for Debian. Its tasks, in order:

1. `Install required packages`: `apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`, `python3-venv`.
2. `Add Docker GPG key`: adds the official GPG key.
3. `Add Docker APT repository`: adds the Docker stable repository.
4. `Install Docker`: installs `docker-ce`.
5. `Install Python3 and pip3`: installs `python3` and `python3-pip`.
6. `Create a virtual environment for Docker SDK`: creates `/opt/docker_venv`.
7. `Install Docker SDK for Python in virtual environment`: installs the `docker` package in this virtualenv.
8. `Make sure Docker is running`: starts the Docker service.

The `docker_network` role creates the `app_network` network with the `community.docker.docker_network` module.


Example (`roles/docker_network/tasks/main.yml`):

```yaml
---
- name: Create Docker network
  community.docker.docker_network:
    name: app_network
    state: present
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

Result of a full playbook run with the `docker` and `docker_network` roles: `ok=10 changed=1 failed=0`. The `app_network` network was kept, since the module's idempotence prevents it from being recreated.

## Application deployment

Three images, published on Docker Hub by the CI of the previous lab, are deployed:

| Role | Container | Image |
|---|---|---|
| `database` | `database` | `mathieumdg/github-actions-my-database:latest` |
| `app` | `backend` | `mathieumdg/github-actions-my-backend:latest` |
| `proxy` | `httpd` | `mathieumdg/github-actions-my-httpd:latest` |

The containers are started with `community.docker.docker_container` and join the `app_network` network, which lets them reach each other by name.

Shared values are centralized in `group_vars/all.yml`:

```yaml
---
docker_python: /opt/docker_venv/bin/python
network_name: app_network

db_image: mathieumdg/github-actions-my-database:latest
api_image: mathieumdg/github-actions-my-backend:latest
proxy_image: mathieumdg/github-actions-my-httpd:latest

db_container: database
api_container: backend
proxy_container: httpd

db_name: db
db_user: usr
db_password: pwd
```

These values come from the previous labs:

- backend `application.yml`: `jdbc:postgresql://database:5432/db`, user `usr`, password `pwd`;
- httpd configuration: `ProxyPass / http://backend:8080/`;
- `docker-compose.yml`: container names `database`, `backend`, `httpd`, port `80:80` for the proxy, `db-data` volume for the database.

### `database` role

```yaml
---
- name: Launch database
  community.docker.docker_container:
    name: "{{ db_container }}"
    image: "{{ db_image }}"
    pull: true
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ network_name }}"
    volumes:
      - db-data:/var/lib/postgresql/data
    env:
      POSTGRES_DB: "{{ db_name }}"
      POSTGRES_USER: "{{ db_user }}"
      POSTGRES_PASSWORD: "{{ db_password }}"
  vars:
    ansible_python_interpreter: "{{ docker_python }}"
```

### `app` role

```yaml
---
- name: Launch API
  community.docker.docker_container:
    name: "{{ api_container }}"
    image: "{{ api_image }}"
    pull: true
    state: started
    restart_policy: unless-stopped
    networks:
      - name: "{{ network_name }}"
    env:
      SPRING_DATASOURCE_URL: "jdbc:postgresql://{{ db_container }}:5432/{{ db_name }}"
      SPRING_DATASOURCE_USERNAME: "{{ db_user }}"
      SPRING_DATASOURCE_PASSWORD: "{{ db_password }}"
  vars:
    ansible_python_interpreter: "{{ docker_python }}"
```

### `proxy` role

```yaml
---
- name: Launch proxy
  community.docker.docker_container:
    name: "{{ proxy_container }}"
    image: "{{ proxy_image }}"
    pull: true
    state: started
    restart_policy: unless-stopped
    ports:
      - "80:80"
    networks:
      - name: "{{ network_name }}"
  vars:
    ansible_python_interpreter: "{{ docker_python }}"
```

## Question 3-3: docker_container tasks configuration

| Parameter | database | app (API) | proxy (httpd) | Purpose |
|---|---|---|---|---|
| `name` | `database` | `backend` | `httpd` | Container name, which also serves as its DNS name on the Docker network |
| `image` | `github-actions-my-database` | `github-actions-my-backend` | `github-actions-my-httpd` | Images built and published by the CI of the previous lab |
| `pull` | `true` | `true` | `true` | Downloads the latest version of the image on every run |
| `state` | `started` | `started` | `started` | The container must be running |
| `restart_policy` | `unless-stopped` | `unless-stopped` | `unless-stopped` | Restarts after a crash or a server reboot, unless manually stopped |
| `networks` | `app_network` | `app_network` | `app_network` | The three containers communicate by name |
| `ports` | none | none | `80:80` | Only the proxy is reachable from the outside |
| `volumes` | `db-data` | none | none | Keeps the database data when the container is recreated |
| `env` | `POSTGRES_*` | `SPRING_DATASOURCE_*` | none | Database configuration and the API's connection to it |


## Running the deployment

```bash
ansible all -i inventories/setup.yml -m ping
ansible-playbook -i inventories/setup.yml playbook.yml
```

Check on the server:

```bash
ansible all -i inventories/setup.yml -b -m raw -a "docker ps"
```

Checks without a server:

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml --list-tasks
```
