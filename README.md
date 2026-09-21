# Déploiement Automatisé et Sécurisé d'une Infrastructure de Supervision

## 📌 Description du projet

Ce projet met en place, de bout en bout, une infrastructure de supervision (monitoring) pour un parc de serveurs Linux, entièrement automatisée avec **Ansible** et conteneurisée avec **Docker**. Il a été réalisé dans le cadre de ma formation en Administration Systèmes et Réseaux, en environnement de laboratoire virtualisé (VirtualBox).

L'objectif : démontrer une maîtrise pratique de l'**Infrastructure as Code**, de la **conteneurisation**, et des **bonnes pratiques de sécurité** (moindre privilège, chiffrement des secrets, segmentation réseau) — des compétences directement transposables à la gestion d'une infrastructure de production.

## 🏗️ Architecture

Le laboratoire est composé de 4 machines virtuelles Ubuntu Server 22.04 LTS :

| Machine | Rôle | Services |
|---|---|---|
| `control-node` | Poste d'administration | Ansible, Git |
| `monitoring-server` | Serveur central de supervision | Prometheus, Alertmanager, Grafana, Nginx (reverse proxy) |
| `node-1` | Serveur surveillé | Node Exporter |
| `node-2` | Serveur surveillé | Node Exporter |

Toutes les machines communiquent sur un réseau privé dédié, isolé du reste du réseau. `control-node` pilote l'ensemble via Ansible, en SSH par authentification par clé (aucun mot de passe utilisé au quotidien).

## 🛠️ Technologies utilisées

- **Automatisation / IaC :** Ansible (playbooks, inventaire, templates Jinja2, Ansible Vault)
- **Conteneurisation :** Docker & Docker Compose
- **Supervision :** Prometheus (collecte de métriques) + Node Exporter (agent système)
- **Visualisation :** Grafana (dashboard "Node Exporter Full")
- **Alerting :** Alertmanager, notifications par email (SMTP/Gmail) et Telegram
- **Sécurité :** UFW (pare-feu), Nginx en reverse proxy HTTPS (certificat auto-signé), Ansible Vault (chiffrement des secrets), utilisateurs système dédiés sans privilèges

## 📂 Structure du dépôt

├── ansible/
│ ├── inventory.ini # Inventaire des machines (groupes monitoring_server / managed_nodes)
│ ├── playbooks/
│ │ ├── node_exporter.yml # Installation de Node Exporter sur les noeuds
│ │ ├── docker_install.yml # Installation de Docker sur le serveur de monitoring
│ │ ├── deploy_monitoring.yml # Déploiement de la stack Prometheus/Alertmanager/Grafana
│ │ ├── firewall.yml # Configuration du pare-feu UFW (moindre privilège)
│ │ └── https_grafana.yml # Reverse proxy Nginx + HTTPS pour Grafana
│ ├── templates/
│ │ └── alertmanager.yml.j2 # Template de config Alertmanager (secrets injectés via Vault)
│ └── vars/
│ └── secrets.yml # Secrets chiffrés (Ansible Vault) — non versionné
└── docker/
├── docker-compose.yml # Définition des services (Prometheus, Alertmanager, Grafana)
└── prometheus/
├── prometheus.yml # Config Prometheus (cibles de scraping, alerting)
└── rules/
└── alerts.yml # Règles d'alerte (CPU, disque, disponibilité)
## 🚀 Déploiement

### Prérequis

- Ansible et Git installés sur `control-node`
- Accès SSH par clé aux machines cibles, `sudo` sans mot de passe configuré
- Un mot de passe de coffre-fort Ansible Vault pour les secrets (email, Telegram)

### Étapes

```bash
# 1. Cloner le dépôt
git clone git@github.com:MGFLF/monitoring-sysadmin-project.git
cd monitoring-sysadmin-project

# 2. Vérifier la connectivité Ansible
ansible all -i ansible/inventory.ini -m ping

# 3. Installer Docker sur le serveur de monitoring
ansible-playbook -i ansible/inventory.ini ansible/playbooks/docker_install.yml

# 4. Installer Node Exporter sur les noeuds surveillés
ansible-playbook -i ansible/inventory.ini ansible/playbooks/node_exporter.yml

# 5. Déployer la stack de supervision (nécessite le mot de passe du coffre-fort)
ansible-playbook -i ansible/inventory.ini ansible/playbooks/deploy_monitoring.yml --ask-vault-pass

# 6. Sécuriser avec le pare-feu
ansible-playbook -i ansible/inventory.ini ansible/playbooks/firewall.yml

# 7. Activer HTTPS pour Grafana
ansible-playbook -i ansible/inventory.ini ansible/playbooks/https_grafana.yml
```

## 🔒 Sécurité — mesures mises en œuvre

- **Principe du moindre privilège** : Node Exporter et Alertmanager tournent avec des utilisateurs système dédiés, sans droits root.
- **Pare-feu par défaut restrictif** (UFW) : tout le trafic entrant est refusé sauf exceptions explicites. Le port de Node Exporter (9100) n'est ouvert que depuis l'adresse IP du serveur de monitoring, jamais depuis l'extérieur.
- **Chiffrement en transit** : l'accès à Grafana se fait exclusivement en HTTPS via un reverse proxy Nginx (Grafana lui-même n'écoute qu'en local, `127.0.0.1`).
- **Secrets chiffrés** : les identifiants email et le token du bot Telegram sont stockés dans un fichier chiffré avec Ansible Vault, jamais en clair, jamais versionnés sur Git.
- **Infrastructure reproductible** : l'intégralité du déploiement est scriptée (Ansible), aucune configuration manuelle non documentée.

## 📊 Supervision et alerting

Le dashboard Grafana "Node Exporter Full" (ID communautaire 1860) offre une vue en temps réel de l'état de chaque machine : CPU, mémoire, disque, réseau, uptime.

Trois règles d'alerte Prometheus surveillent en continu l'infrastructure :
- **CPUChargeElevee** : charge CPU supérieure à 90%
- **DisqueSature** : moins de 15% d'espace disque libre
- **NoeudInjoignable** : un nœud ne répond plus au scraping Prometheus

Chaque alerte déclenche une notification simultanée par **email** et **Telegram**, avec un message de résolution automatique une fois le problème réglé.


## 📈 Aperçu

### Accès sécurisé à Grafana (HTTPS via reverse proxy Nginx)
![Page de connexion Grafana](docs/images/Page_Login_Grafana.png)

### Dashboards en fonctionnement normal
![Dashboard Grafana - Node 1](docs/images/Dashboard_Grafana_Node1.png)
![Dashboard Grafana - Node 2](docs/images/Dashboard_Grafana_Node2.png)

### Scénario 1 — Surcharge CPU détectée et résolue
| Déclenchement du test | Impact visible sur le dashboard |
|---|---|
| ![Test de charge CPU](docs/images/Test_Alert_ChargeCPU_UP.png) | ![Dashboard pendant la surcharge](docs/images/Dashboard_Grafana_CPU_UP.png) |

L'alerte apparaît immédiatement dans Alertmanager, puis les notifications partent sur les deux canaux configurés :

| Alertmanager | Email | Telegram |
|---|---|---|
| ![Alerte CPU dans Alertmanager](docs/images/Alerte_Alertemanager_CPU_UP.png) | ![Email d'alerte reçu](docs/images/Alertes_Mail_Msg.png) | ![Alerte reçue sur Telegram](docs/images/Alertes_Telegram.png) |

### Scénario 2 — Nœud injoignable détecté et résolu
| Dashboard pendant la panne | Alerte dans Alertmanager |
|---|---|
| ![Dashboard nœud injoignable](docs/images/Dashboard_Grafana_node_Down.png) | ![Alerte nœud injoignable](docs/images/Alerte_Alertemanager_Node_Down.png) |


Statut de la cible dans Prometheus, avant et après résolution :

| Nœud down | Nœud de nouveau up |
|---|---|
| ![Cible down](docs/images/Neud_DOWN.png) | ![Cible up](docs/images/Neud_UP.png) |

### Boîte de réception : cycle complet alerte → résolution
![Historique des alertes reçues par email](docs/images/Alertes_Mail.png)
![Email de résolution automatique](docs/images/Mail_Resolution_Alertes.png)

## 🎓 Ce que ce projet démontre

- Écriture de playbooks Ansible idempotents et modulaires
- Gestion sécurisée de secrets dans un pipeline d'automatisation
- Déploiement et orchestration de services avec Docker Compose
- Configuration d'un reverse proxy et de certificats TLS
- Mise en place d'un pare-feu suivant le principe du moindre privilège
- Diagnostic et résolution de problèmes réels (permissions, réseau, configuration YAML)

## 👤 Auteur

**Faha_laurence_Fermandez** — Administration Systèmes et Réseaux
