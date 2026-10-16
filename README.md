# TP3 : Ansible, Docker et déploiement d'application

Déploiement automatisé d'une application trois conteneurs (base de données PostgreSQL, API Spring Boot, proxy httpd) sur un serveur distant, avec Ansible et Docker.

## Sommaire

1. [Environnement](#environnement)
2. [Structure du projet](#structure-du-projet)
3. [Question 3-1 : inventaire et commandes de base](#question-3-1--inventaire-et-commandes-de-base)
4. [Question 3-2 : playbook et rôles](#question-3-2--playbook-et-rôles)
5. [Problème rencontré : interpréteurs Python](#problème-rencontré--interpréteurs-python)
6. [Déploiement de l'application](#déploiement-de-lapplication)
7. [Question 3-3 : configuration des tâches docker_container](#question-3-3--configuration-des-tâches-docker_container)
8. [Question de sécurité : déploiement automatique](#question-de-sécurité--déploiement-automatique)
9. [Lancer le déploiement](#lancer-le-déploiement)
10. [État d'avancement](#état-davancement)

## Environnement

| Élément | Valeur |
|---|---|
| Machine de contrôle | VM Ubuntu (`Ubuntu-Ansible`) sous VMware Workstation, hébergée sur Windows |
| Édition des fichiers | Visual Studio Code sur Windows, synchronisation par Git avec la VM |
| Serveur cible | Serveur Debian fourni par Takima, accessible en SSH par clé privée |
| Ansible | 2.10.8 |
| Collection Docker | `community.docker` 2.7.0 |
| Python du SDK Docker (serveur) | virtualenv `/opt/docker_venv` |

La collection Docker est installée dans une version compatible avec Ansible 2.10 :

```bash
ansible-galaxy collection install community.docker:2.7.0 --force
```

## Structure du projet

```text
.
├── inventories/
│   └── setup.yml          # hôte, utilisateur SSH, clé privée
├── group_vars/
│   └── all.yml            # variables communes (images, noms, base de données)
├── roles/
│   ├── docker/            # installation de Docker et du SDK Python
│   ├── docker_network/    # réseau Docker app_network
│   ├── database/          # conteneur PostgreSQL
│   ├── app/               # conteneur de l'API
│   └── proxy/             # conteneur httpd
├── playbook.yml           # playbook principal
└── README.md
```

Les cinq rôles ont été générés avec `ansible-galaxy init roles/<nom>`. Seuls les fichiers `tasks/main.yml` sont utilisés ; les autres fichiers générés (defaults, handlers, meta, tests, vars) sont conservés tels quels.

## Question 3-1 : inventaire et commandes de base

L'inventaire `inventories/setup.yml` décrit le serveur et la façon de s'y connecter (utilisateur SSH, chemin de la clé privée, hôte), regroupé dans un groupe `prod`.

```yaml
all:
  vars:
    ansible_user: <utilisateur>
    ansible_ssh_private_key_file: <chemin de la clé privée>
  children:
    prod:
      hosts: <nom d'hôte ou IP>
```

Commandes utilisées :

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Vérifie la connexion SSH et la présence de Python sur le serveur |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Affiche des facts |
| `ansible all -i inventories/setup.yml -m setup -a 'filter=ansible_python*'` | Affiche le Python utilisé par Ansible |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | Supprime Apache2 du serveur |


## Question 3-2 : playbook et rôles

Le playbook `playbook.yml` s'applique à tous les hôtes, collecte les facts et élève ses privilèges (`become`). Il appelle les rôles dans cet ordre :

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

Le rôle `docker` suit la documentation officielle Docker pour Debian. Ses tâches, dans l'ordre :

1. `Install required packages` : `apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`, `python3-venv`.
2. `Add Docker GPG key` : ajoute la clé GPG officielle.
3. `Add Docker APT repository` : ajoute le dépôt stable de Docker.
4. `Install Docker` : installe `docker-ce`.
5. `Install Python3 and pip3` : installe `python3` et `python3-pip`.
6. `Create a virtual environment for Docker SDK` : crée `/opt/docker_venv`.
7. `Install Docker SDK for Python in virtual environment` : installe le paquet `docker` dans ce virtualenv.
8. `Make sure Docker is running` : démarre le service Docker.

Le rôle `docker_network` crée le réseau `app_network` avec le module `community.docker.docker_network`.


Exemple (`roles/docker_network/tasks/main.yml`) :

```yaml
---
- name: Create Docker network
  community.docker.docker_network:
    name: app_network
    state: present
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

Résultat de l'exécution complète du playbook avec les rôles `docker` et `docker_network` : `ok=10 changed=1 failed=0`. Le réseau `app_network` a été conservé, l'idempotence du module évitant de le recréer.

## Déploiement de l'application

Trois images, publiées sur Docker Hub par la CI du TP précédent, sont déployées :

| Rôle | Conteneur | Image |
|---|---|---|
| `database` | `database` | `mathieumdg/github-actions-my-database:latest` |
| `app` | `backend` | `mathieumdg/github-actions-my-backend:latest` |
| `proxy` | `httpd` | `mathieumdg/github-actions-my-httpd:latest` |

Les conteneurs sont lancés avec `community.docker.docker_container` et rejoignent le réseau `app_network`, ce qui permet de les joindre par leur nom.

Les valeurs communes sont centralisées dans `group_vars/all.yml` :

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

Ces valeurs viennent des TP précédents :

- `application.yml` du backend : `jdbc:postgresql://database:5432/db`, utilisateur `usr`, mot de passe `pwd` ;
- configuration httpd : `ProxyPass / http://backend:8080/` ;
- `docker-compose.yml` : noms de conteneurs `database`, `backend`, `httpd`, port `80:80` pour le proxy, volume `db-data` pour la base.

### Rôle `database`

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

### Rôle `app`

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

### Rôle `proxy`

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

## Question 3-3 : configuration des tâches docker_container

| Paramètre | database | app (API) | proxy (httpd) | Rôle |
|---|---|---|---|---|
| `name` | `database` | `backend` | `httpd` | Nom du conteneur, qui sert aussi de nom DNS sur le réseau Docker |
| `image` | `github-actions-my-database` | `github-actions-my-backend` | `github-actions-my-httpd` | Images construites et publiées par la CI du TP précédent |
| `pull` | `true` | `true` | `true` | Télécharge la dernière version de l'image à chaque exécution |
| `state` | `started` | `started` | `started` | Le conteneur doit tourner |
| `restart_policy` | `unless-stopped` | `unless-stopped` | `unless-stopped` | Redémarre après un crash ou un redémarrage du serveur, sauf arrêt manuel |
| `networks` | `app_network` | `app_network` | `app_network` | Les trois conteneurs communiquent par leur nom |
| `ports` | aucun | aucun | `80:80` | Seul le proxy est accessible de l'extérieur |
| `volumes` | `db-data` | aucun | aucun | Conserve les données de la base quand le conteneur est recréé |
| `env` | `POSTGRES_*` | `SPRING_DATASOURCE_*` | aucune | Configuration de la base et de la connexion de l'API |


## Lancer le déploiement

```bash
ansible all -i inventories/setup.yml -m ping
ansible-playbook -i inventories/setup.yml playbook.yml
```

Vérification sur le serveur :

```bash
ansible all -i inventories/setup.yml -b -m raw -a "docker ps"
```

Vérifications sans serveur :

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml --list-tasks
```
