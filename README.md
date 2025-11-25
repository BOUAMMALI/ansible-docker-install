# 📘 README — Projet : Installer Docker + Docker Compose avec Ansible

## 🎯 Objectif
Automatiser l’installation de **Docker Engine**, **Docker CLI**, **Docker Buildx** et **Docker Compose (plugin officiel)** sur des hôtes Ubuntu/Debian avec **Ansible**. Le rôle est **idempotent**, **lisible** et **prêt pour CI/CD**.

- OS supportés : Ubuntu 20.04/22.04/24.04, Debian 10/11
- Architecture auto-déduite (`amd64`, `arm64`, `armhf`…)
- Ajout automatique des utilisateurs au groupe `docker`

---

## 🏗️ Arborescence du repo
```
ansible-docker-install/
├─ ansible.cfg
├─ inventory.ini
├─ group_vars/
│  └─ all.yml
├─ playbooks/
│  └─ install-docker.yml
└─ roles/
   └─ docker/
      ├─ tasks/
      │  └─ main.yml
      ├─ templates/
      └─ files/
```

---

## ⚙️ Fichiers principaux

### `ansible.cfg`
```ini
[defaults]
inventory = inventory.ini
remote_user = ubuntu
host_key_checking = False
retry_files_enabled = False
roles_path = roles

[ssh_connection]
ssh_args = -o ForwardAgent=yes -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```

### `inventory.ini`
```ini
[nodes]
node01 ansible_host=192.168.33.10 ansible_user=vagrant
node02 ansible_host=192.168.33.11 ansible_user=vagrant
```

### `group_vars/all.yml`
```yaml
docker_users:
  - vagrant  # adapte avec l'utilisateur de tes hôtes (ex: ubuntu)
```

### `playbooks/install-docker.yml`
```yaml
---
- name: Installer Docker Engine sur les nodes
  hosts: all
  become: yes
  roles:
    - docker
```

---

## 🧩 Rôle `roles/docker/tasks/main.yml` (version propre & idempotente)
```yaml
---
# 0) Déduire l'architecture Debian/Ubuntu
- name: Déduire l’architecture Debian/Ubuntu
  ansible.builtin.set_fact:
    docker_arch: >-
      {{ 'amd64' if ansible_architecture in ['x86_64','x86-64'] else
         'arm64' if ansible_architecture in ['aarch64','arm64'] else
         'armhf' if ansible_architecture in ['armv7l','armv7'] else
         ansible_architecture }}

# 1) Dépendances APT
- name: Installer dépendances APT
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gnupg
    state: present
    update_cache: yes

# 2) Répertoire des clés
- name: Créer /etc/apt/keyrings
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

# 3) Télécharger la clé GPG (ASCII)
- name: Télécharger la clé GPG Docker (ASCII)
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'

# 4) Convertir en .gpg (dearmor) une seule fois
- name: Convertir la clé en format .gpg (dearmor)
  ansible.builtin.command:
    cmd: gpg --dearmor -o /etc/apt/keyrings/docker.gpg /etc/apt/keyrings/docker.asc
  args:
    creates: /etc/apt/keyrings/docker.gpg

# 5) Droits de lecture sur la clé
- name: Autoriser la lecture de la clé
  ansible.builtin.file:
    path: /etc/apt/keyrings/docker.gpg
    mode: '0644'

# 6) Dépôt Docker officiel avec signed-by
- name: Ajouter le dépôt Docker (apt_repository)
  ansible.builtin.apt_repository:
    repo: >-
      deb [arch={{ docker_arch }} signed-by=/etc/apt/keyrings/docker.gpg]
      https://download.docker.com/linux/{{ ansible_distribution | lower }}
      {{ ansible_distribution_release }} stable
    filename: docker
    state: present
    update_cache: yes

# 7) Installer Docker Engine + plugins
- name: Installer Docker Engine + plugins
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    state: present

# 8) Démarrer & activer Docker
- name: S’assurer que Docker est actif
  ansible.builtin.service:
    name: docker
    state: started
    enabled: true

# 9) Ajouter les utilisateurs au groupe docker
- name: Ajouter les utilisateurs au groupe docker
  ansible.builtin.user:
    name: "{{ item }}"
    groups: docker
    append: yes
  loop: "{{ docker_users }}"
```

---

## 🚀 Exécution
1) Vérifier la connectivité :
```bash
ansible all -m ping
```
2) Installer Docker :
```bash
ansible-playbook playbooks/install-docker.yml
```

---

## 🧪 Vérifications post-install
```bash
docker --version
docker compose version
docker buildx version
systemctl status docker
groups $(whoami)  # l’utilisateur doit contenir `docker`
```
> Si tu viens d’être ajouté au groupe `docker`, reconnecte-toi (ou `newgrp docker`).

---

## 🔐 Sécurité & bonnes pratiques
- Activer UFW sur les hôtes :
```bash
sudo ufw allow OpenSSH
sudo ufw enable
```
- **Évite** d’exposer la remote API Docker (2375) en clair. Si nécessaire, fais-le derrière TLS et firewall strict.

---

## 🧩 Extensions possibles
- Rôle pour **docker-compose (binaire standalone)** si besoin legacy
- Rôle pour **déployer une stack Docker Compose** (WordPress, Nextcloud, Prometheus…)
- **Registry privé** (Harbor/Registry) managé par Ansible
- Tests **Molecule** sur le rôle `docker`
- Makefile + CI (GitLab/GitHub) : `ansible-lint`, `yamllint`, déploiement

---

## 📜 Licence
Libre pour usage personnel, éducatif et professionnel.

---

