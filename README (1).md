# TP3 : Ansible, Docker et déploiement d'application

Déploiement automatisé d'une application (base de données, API, proxy HTTP) sur un serveur distant avec Ansible et Docker.

## Environnement

| Élément | Valeur |
|---|---|
| Machine de contrôle | VM Ubuntu (`Ubuntu-Ansible`) sous VMware Workstation, hébergée sur Windows |
| Serveur cible | Serveur Debian fourni par Takima, accessible en SSH par clé privée |
| Ansible | 2.10.8 |
| Collection Docker | `community.docker` 2.7.0 |
| Python du SDK Docker (serveur) | virtualenv `/opt/docker_venv` |

Installation de la collection (version compatible avec Ansible 2.10) :

```bash
ansible-galaxy collection install community.docker:2.7.0 --force
```

## Structure du projet

```text
ansible/
├── inventories/
│   └── setup.yml          # hôte, utilisateur SSH, clé privée
├── group_vars/
│   └── all.yml            # variables communes (noms, base de données)
├── roles/
│   ├── docker/            # installation de Docker + SDK Python
│   ├── docker_network/    # réseau Docker app_network
│   ├── database/          # conteneur de la base de données
│   ├── app/               # conteneur de l'API
│   └── proxy/             # conteneur httpd (proxy)
└── playbook.yml           # playbook principal
```

## Question 3-1 : inventaire et commandes de base

L'inventaire `inventories/setup.yml` décrit le serveur et la façon de s'y connecter (utilisateur SSH, clé privée, hôte), regroupé dans un groupe `prod`.

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Vérifie la connexion SSH et la présence de Python |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Affiche des facts (informations découvertes sur le serveur) |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | Supprime Apache2 du serveur |

La suppression d'Apache est idempotente : une seconde exécution ne change plus rien, car l'état demandé (`absent`) est déjà atteint.

## Question 3-2 : playbook et rôles

Le playbook `playbook.yml` s'applique à tous les hôtes, collecte les facts et élève ses privilèges (`become`). Il appelle les rôles dans cet ordre : `docker`, `docker_network`, `database`, `app`, `proxy`.

Le rôle `docker` suit la documentation officielle Docker pour Debian :

1. Installation des prérequis (`apt-transport-https`, `ca-certificates`, `curl`, `gnupg`, `lsb-release`, `python3-venv`).
2. Ajout de la clé GPG Docker.
3. Ajout du dépôt APT Docker.
4. Installation de `docker-ce`.
5. Installation de Python 3 et pip.
6. Création du virtualenv `/opt/docker_venv`.
7. Installation du SDK Docker pour Python dans ce virtualenv.
8. Démarrage du service Docker.

Les rôles ont été créés avec `ansible-galaxy init roles/<nom>`.

## Problème rencontré : interpréteurs Python

Deux modules ont besoin de deux Python différents sur le serveur :

| Module | Python nécessaire | Raison |
|---|---|---|
| `apt` (rôle `docker`) | `/usr/bin/python3` | Les modules `apt` et `apt_pkg` sont installés pour le Python système |
| `docker_network`, `docker_container` | `/opt/docker_venv/bin/python` | Le SDK Docker est installé dans le virtualenv |

Symptômes observés :

- `Could not import python modules: apt, apt_pkg` quand `apt` tournait avec le Python du virtualenv.
- `No module named 'docker'` quand le module Docker tournait avec le Python système.

Cause : la variable `ansible_python_interpreter: /opt/docker_venv/bin/python` était restée dans l'inventaire, et s'appliquait donc à tout l'hôte.

Solution retenue :

- retirer cette variable de l'inventaire ;
- fixer `ansible_python_interpreter: /usr/bin/python3` dans le playbook (global) ;
- poser `ansible_python_interpreter: /opt/docker_venv/bin/python` directement sur chaque tâche qui utilise le SDK Docker (`vars:` au niveau de la tâche).

Exemple (`roles/docker_network/tasks/main.yml`) :

```yaml
- name: Create Docker network
  community.docker.docker_network:
    name: app_network
    state: present
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
```

Résultat : exécution complète du playbook avec `failed=0`, réseau `app_network` conservé.

## Déploiement de l'application

Trois images Docker Hub sont déployées :

| Rôle | Image |
|---|---|
| `database` | `frangipanne/tp-devops-database` |
| `app` | `frangipanne/tp-devops-simple-api` |
| `proxy` | `frangipanne/tp-devops-httpd` |

Les conteneurs sont lancés avec `community.docker.docker_container` et rejoignent le réseau `app_network`, ce qui permet à l'API de joindre la base par son nom de conteneur. Seul le proxy publie un port (80). Les variables d'environnement de la base (`POSTGRES_*`) et de l'API (`SPRING_DATASOURCE_*`) sont définies dans les tâches à partir de `group_vars/all.yml`.

## Lancer le déploiement

```bash
cd ~/my-project/ansible
ansible all -i inventories/setup.yml -m ping
ansible-playbook -i inventories/setup.yml playbook.yml
```

Vérification sur le serveur :

```bash
ansible all -i inventories/setup.yml -b -m raw -a "docker ps"
```

## Avancement

| Étape | État |
|---|---|
| Inventaire, ping, facts, suppression d'Apache | Fait |
| Playbook Docker et rôle `docker` | Fait |
| Rôle `docker_network` | Fait |
| Résolution du problème d'interpréteurs Python | Fait |
| Rôles `database`, `app`, `proxy` | Écrits, exécution et accès à l'API à valider |
| Question 3-3 (documentation des tâches `docker_container`) | À rédiger |
| Déploiement continu avec GitHub Actions | À faire |
| Frontend et adaptation du proxy | À faire |
| Ansible Vault et Ansible dans un conteneur (bonus) | À faire |

## Notes de sécurité

- La clé privée SSH ne doit jamais être versionnée ; seul son chemin figure dans l'inventaire.
- Les mots de passe de la base de données sont pour l'instant dans `group_vars/all.yml` ; ils seront chiffrés avec Ansible Vault.
- Un token d'accès personnel GitHub (et non le mot de passe du compte) est utilisé pour `git push`.
