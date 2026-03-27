# 🛡️ Nathan MICHELLE (Kazibuya)

> **Ex-Sauveteur en Mer 🌊 | Ex-Mesures Physiques (Orsay) 🔬 | Étudiant à 42 💻**

Ancien sauveteur en mer, j'applique aujourd'hui la même rigueur à la survie des infrastructures critiques. Spécialisé en **Cloud Architecture** et **Security Automation**, avec une obsession pour le **Zero Trust**.

---

### 🛠️ Tooling & Stack

- **IaC / Config :** Terraform, Packer, Ansible, Terragrunt
- **Orchestration :** K3s, Helm, kubectl
- **Languages :** Go (Mastering for Molecule & Terratest), C
- **Cloud & Security :** AWS (KMS, IAM, VPC, ECR), HashiCorp Vault, Zero Trust Architecture
- **Observability :** Prometheus, Grafana, ELK — **next stop : eBPF 🐝** (observabilité et sécurité kernel-level)
- **Réseau :** WireGuard, Flannel, Cilium *(à venir)*

### 🏗️ Projet en cours

**Template SANDBOX** — voir ci-dessous pour le détail complet.

### 📂 Portfolio

> ⚠️ Nouvellement arrivé sur GitHub (anciennement sur la **Vogsphere** interne de 42).
> Je migre et documente mes anciens projets (C/Unix/System) progressivement tout en finalisant mes templates d'infrastructure actuels.
>
> 🔒 Étant en phase d'apprentissage, tout dépôt susceptible de contenir des clés ou secrets est maintenu en **privé** par précaution.

---

---

# Template SANDBOX

> Infrastructure as Code — Déploiement automatisé d'un cluster Kubernetes (K3s) multi-nodes sur AWS via Terraform, Packer, Ansible et HashiCorp Vault.

> ⚠️ **Projet en cours de développement** — migration active de Docker Compose vers K3s. Certaines fonctionnalités sont encore en cours d'implémentation ou de stabilisation.

---

## Vue d'ensemble

**Template SANDBOX** est un projet d'infrastructure DevSecOps conçu pour provisionner et configurer automatiquement un environnement complet sur AWS. Il couvre l'ensemble du cycle de vie : création d'AMI durcies, provisionnement cloud, gestion des secrets, orchestration Kubernetes, déploiement applicatif via Helm et observabilité.

L'objectif est de fournir un template réutilisable, sécurisé et reproductible pour tout environnement de type sandbox ou staging, déployable en une seule commande via un pipeline CI/CD.

---

## Stack technique

| Couche | Technologie | Statut |
|---|---|---|
| Cloud | AWS EC2 (ARM64 / Graviton), ECR, KMS, IAM, S3, Budgets | ✅ |
| Infrastructure as Code | Terraform (workspaces infra + vault séparés) | ✅ |
| Build AMI | Packer + AlmaLinux 9 (aarch64) | ✅ |
| Provisionnement | Ansible | ✅ |
| Orchestration | K3s + Flannel WireGuard | ✅ |
| Package manager K8s | Helm | 🔄 en cours |
| Conteneurisation | Docker | ✅ |
| Gestion des secrets | HashiCorp Vault (KV v2, auto-unseal via AWS KMS) | ✅ |
| Vault distribution | Vault Agent Injector (sidecar K8s) | 🔄 en cours |
| Vault PKI + mTLS | Vault PKI secrets engine | ⏳ à venir |
| Logs | Elasticsearch · Kibana · Logstash — ELK 8.x | 🔄 en cours |
| Monitoring | Grafana · Prometheus · Node Exporter | 🔄 en cours |
| WAF | Nginx Ingress + ModSecurity | ⏳ à venir |
| CI/CD | GitHub Actions | ✅ |
| Sécurité OS | Firewalld, sshd hardening, Ansible Vault (AES256) | ✅ |
| Réseau inter-nodes | WireGuard natif (Flannel built-in) | ✅ |
| eBPF / Cilium | Cilium CNI + Tetragon | ⏳ à venir |
| Tests | Molecule + Terratest | ⏳ à venir |

---

## Architecture
```
┌─────────────────────────────────────────────────────────┐
│                      GitHub Actions                     │
│  build image → terraform apply → ansible → helm install │
└──────────────────────────┬──────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │        AWS Cloud        │  eu-north-1
              │                         │
              │  ┌─────────────────┐    │
              │  │   EC2 MASTER    │    │  t4g.medium / 20GB
              │  │   K3s server    │    │  port 6443 (API)
              │  │   Vault         │    │  port 8200
              │  │   etcd          │    │
              │  └────────┬────────┘    │
              │           │ KMS unseal  │
              │  ┌────────▼────────┐    │
              │  │    AWS KMS      │    │
              │  └─────────────────┘    │
              │                         │
              │  ┌─────────────────┐    │
              │  │   EC2 WORKER1   │    │  t4g.medium / 20GB
              │  │   K3s agent     │    │
              │  │   ELK pods      │    │
              │  └─────────────────┘    │
              │                         │
              │  ┌─────────────────┐    │
              │  │   EC2 WORKER2   │    │  t4g.small / 8GB
              │  │   K3s agent     │    │
              │  │   Grafana pods  │    │
              │  └─────────────────┘    │
              └─────────────────────────┘

  Réseau inter-nodes : chiffré via Flannel + WireGuard natif K3s
  Secrets : distribués via Vault Agent Injector (sidecar tmpfs)
```

L'inventaire Ansible est généré automatiquement par Terraform à partir des IPs publiques des instances.

---

## Structure du projet
```
.
├── .github/
│   └── workflows/
│       └── deploy.yml              # Pipeline CI/CD
├── ansible/
│   ├── ansible.cfg
│   ├── deploy.yml                  # Playbook principal
│   ├── packer.yml                  # Playbook pour la construction AMI
│   ├── group_vars/
│   │   ├── MASTER.yml              # Ports firewalld master
│   │   └── WORKERS.yml             # Ports firewalld workers
│   └── roles/
│       ├── security_os/            # Hardening SSH + Firewall
│       ├── docker_install/         # Installation Docker CE
│       ├── security_vault/         # Déploiement et init Vault
│       ├── k3s_master/             # Installation K3s master node
│       ├── k3s_worker/             # Installation K3s worker nodes
│       ├── go-server/              # [archivé → Helm chart]
│       ├── elk-stack/              # [archivé → Helm chart officiel]
│       └── grafana-stack/          # [archivé → Helm chart officiel]
├── helm/
│   ├── app/                        # Chart custom app Go [à venir]
│   └── values/
│       ├── vault.yaml              # Values Vault [à venir]
│       ├── elasticsearch.yaml      # Values ELK [à venir]
│       ├── kibana.yaml             # Values Kibana [à venir]
│       ├── grafana.yaml            # Values Grafana [à venir]
│       └── prometheus.yaml         # Values Prometheus [à venir]
├── terraform/
│   ├── infra/
│   │   ├── main.tf                 # Instances EC2 (master + workers)
│   │   ├── networking.tf           # Security Groups (master_sg + worker_sg)
│   │   ├── iam.tf                  # Rôle IAM unique k3s-role
│   │   ├── kms.tf                  # Clé KMS pour Vault auto-unseal
│   │   ├── backend.tf              # State S3 (terraform-infra.tfstate)
│   │   ├── budget.tf               # Alerte budgétaire AWS
│   │   ├── outputs.tf
│   │   └── variables.tf
│   ├── vault/
│   │   ├── main.tf                 # Vault auth backend + policies
│   │   ├── providers.tf            # Provider Vault + remote_state infra
│   │   ├── backend.tf              # State S3 (terraform-vault.tfstate)
│   │   └── variables.tf
│   └── modules/
│       ├── compute/                # Module EC2 réutilisable
│       └── vault-config/           # Module policy + role Vault
└── packer/
    └── alma.pkr.hcl                # Build AMI AlmaLinux 9 ARM64
```

---

## Flux de déploiement
```
1. Packer          →  Build AMI AlmaLinux 9 (hardening + Docker)
2. Terraform infra →  Provisionnement EC2 (master + workers) / IAM / KMS / SG
                       Génération automatique de l'inventaire Ansible
3. Ansible         →  Configuration des nodes :
                       ├── Hardening OS (firewall, sshd, users)
                       ├── Installation Docker
                       ├── Déploiement Vault + initialisation (master)
                       │   └── Injection des secrets (app, db, elk, grafana)
                       ├── Installation K3s master (+ token génération)
                       └── Installation K3s workers (join cluster)
4. Terraform vault →  Configuration Vault :
                       ├── AWS auth backend
                       └── Policies + roles par service
5. Helm            →  Déploiement des services sur le cluster K3s :
                       ├── Vault Agent Injector
                       ├── App Go (chart custom)
                       ├── ELK (chart officiel Elastic)
                       ├── Grafana + Prometheus (chart officiel)
                       └── Nginx Ingress + ModSecurity [à venir]
```

---

## Sécurité

### Gestion des secrets

Les secrets ne transitent jamais en clair dans le code. Le flux est le suivant :

- **Vault** est déployé sur le master et s'auto-déverrouille via **AWS KMS** (clé dédiée avec rotation activée).
- À l'initialisation, le **root token** est chiffré par **Ansible Vault (AES256)** et stocké dans S3 (versionné + chiffré AES256).
- Les services récupèrent leurs secrets depuis Vault via le **Vault Agent Injector** (sidecar Kubernetes), en s'authentifiant via le **Kubernetes auth method**.
- Les secrets sont montés en **tmpfs** dans les pods — jamais écrits sur disque.

### Hardening OS

- Accès SSH par clés uniquement, `PermitRootLogin no`
- Firewall `firewalld` avec ouverture minimale des ports par groupe de nodes
- Création de comptes utilisateurs nominatifs par membre de l'équipe

### Réseau

- Chiffrement inter-nodes via **Flannel + WireGuard natif** (K3s built-in)
- Un seul rôle IAM `k3s-role` pour toutes les instances
- Communications Vault → services via **Kubernetes auth method**

### IAM

- Principe du moindre privilège : un seul rôle IAM `k3s-role` pour toutes les instances
- Permissions KMS (encrypt/decrypt) pour Vault auto-unseal
- Permissions ECR (pull) pour les images applicatives

---

## Prérequis

- AWS CLI configuré avec les droits suffisants
- Terraform ≥ 1.0
- Packer ≥ 1.9
- Ansible ≥ 2.14
- Helm ≥ 3.0
- kubectl
- Docker (pour le build de l'image applicative)
- Un bucket S3 existant pour le backend Terraform
- Un dépôt ECR existant

---

## Déploiement

### Variables requises

Créer `terraform/infra/terraform.tfvars` :
```hcl
aws_region       = "eu-north-1"
project_name     = "mon-projet"
admin_public_key = "ssh-rsa AAAA..."
email            = "alert@example.com"
```

Créer `ansible/secrets.yml` (chiffré avec Ansible Vault) :
```yaml
vault_root_token:  "..."
elastic_pass:      "..."
kibana_pass:       "..."
elk_encrypt_key:   "..."
grafana_pass:      "..."
db_password:       "..."
db_user:           "..."
db_name:           "..."
app_jwt_secret:    "..."
app_api_key:       "..."
```

### Construction de l'AMI (une seule fois)
```bash
make packer
```

### Déploiement complet
```bash
make deploy
```

### Commandes utiles
```bash
make plan                         # Affiche le plan Terraform
make ansible                      # Relance Ansible seul
make ansible role=k3s_master      # Relance un rôle spécifique
make kubectl cmd="get nodes"      # Exécute kubectl sur le master
make kubectl cmd="get pods -A"    # Liste tous les pods
make output                       # Affiche les IPs et outputs
make destroy                      # Détruit les instances EC2
make destroy-full                 # Détruit toute l'infrastructure
```

---

## CI/CD (GitHub Actions)

Le pipeline se déclenche automatiquement sur push vers `main` ou `feature/infra`.

Il peut aussi être lancé manuellement avec les actions suivantes :

| Action | Description |
|---|---|
| `deploy` | Build image Docker → push ECR → terraform apply → ansible → helm |
| `destroy` | Destruction des instances EC2 uniquement |
| `destroy-full` | Destruction complète de l'infrastructure |

### Secrets GitHub requis

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | Clé d'accès AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret AWS |
| `SSH_PRIVATE_KEY` | Clé SSH privée pour Ansible |
| `ANSIBLE_VAULT_PASSWORD` | Mot de passe de déchiffrement Ansible Vault |

---

## Ports exposés

| Node | Service | Port | Protocole |
|---|---|---|---|
| MASTER | K3s API server | 6443 | TCP |
| MASTER | Vault | 8200 | TCP |
| MASTER | HTTP | 80 | TCP |
| MASTER | HTTPS | 443 | TCP |
| MASTER | etcd | 2379-2380 | TCP |
| ALL | Kubelet | 10250 | TCP |
| ALL | Flannel VXLAN | 8472 | UDP |
| ALL | WireGuard | 51820 | UDP |
| ALL | NodePort range | 30000-32767 | TCP |
| ALL | Node Exporter | 9100 | TCP |

---

## Roadmap

### Helm charts *(en cours)*

Déploiement des services sur le cluster K3s via Helm :
- **Vault Agent Injector** — distribution des secrets dans les pods
- **ELK** — charts officiels Elastic
- **Grafana + Prometheus** — charts officiels
- **App Go** — chart custom

### WAF + Ingress *(à venir)*

Ajout d'un ingress controller **Nginx** avec **ModSecurity** (WAF) pour filtrer le trafic entrant. Terminaison TLS centralisée via **cert-manager** + **Vault PKI**.

### Vault PKI + mTLS *(à venir)*

Utilisation du **PKI secrets engine** de Vault pour générer et renouveler automatiquement les certificats TLS de chaque service. Vault Agent Injector distribue les certificats dans les pods via des volumes tmpfs. Objectif : **mTLS complet** entre tous les services du cluster.

### eBPF + Cilium *(vision cible)*

Remplacement de Flannel par **Cilium** comme CNI Kubernetes pour bénéficier de :
- Network policies L3/L4/L7 via eBPF
- Transparent encryption WireGuard inter-nodes
- Observabilité kernel-level via **Tetragon**
- mTLS natif entre services sans modification applicative

### HA K3s *(à venir)*

Migration vers un cluster **hautement disponible** :
- 3 masters avec etcd distribué
- Network Load Balancer AWS devant les masters
- Migration transparente depuis l'option 1 master actuelle

### Tests *(à venir)*

- **Molecule** — tests des rôles Ansible
- **Terratest** — tests des modules Terraform

### devops-kit *(à venir)*

Extraction du template en repo réutilisable — documentation, exemples, variables d'entrée standardisées.

---

## Auteur

Nathan Michelle
