# 🛡️ Nathan MICHELLE (Kazibuya)

> **Ex-Sauveteur en Mer 🌊 | Ex-Mesures Physiques (Orsay) 🔬 | Étudiant à 42 💻**

Ancien sauveteur en mer, j'applique aujourd'hui la même rigueur à la survie des infrastructures critiques. Spécialisé en **Cloud Architecture** et **Security Automation**, avec une obsession pour le **Zero Trust**.

---

### 🛠️ Tooling & Stack

- **IaC / Config :** Terraform, Packer, Ansible, Terragrunt
- **Languages :** Go (Mastering for Molecule & Terratest), C
- **Cloud & Security :** AWS (KMS, IAM, VPC), HashiCorp Vault, Zero Trust Architecture
- **Observability :** Prometheus, Grafana, ELK — **next stop : eBPF 🐝** (observabilité et sécurité kernel-level)

### 📂 Portfolio

> ⚠️ Nouvellement arrivé sur GitHub (anciennement sur la **Vogsphere** interne de 42).
> Je migre et documente mes anciens projets (C/Unix/System) progressivement tout en finalisant mes templates d'infrastructure actuels.

---

---

# Template SANDBOX

> Infrastructure as Code — Déploiement automatisé d'un environnement multi-serveurs sur AWS via Terraform, Packer, Ansible et HashiCorp Vault.

> ⚠️ **Projet en cours de développement** — certaines fonctionnalités sont encore en cours d'implémentation ou de stabilisation.

---

## Vue d'ensemble

**Template SANDBOX** est un projet d'infrastructure DevSecOps conçu pour provisionner et configurer automatiquement un environnement complet sur AWS. Il couvre l'ensemble du cycle de vie : création d'AMI durcies, provisionnement cloud, gestion des secrets, déploiement applicatif et observabilité.

L'objectif est de fournir un template réutilisable, sécurisé et reproductible pour tout environnement de type sandbox ou staging, déployable en une seule commande via un pipeline CI/CD.

---

## Stack technique

| Couche | Technologie |
|---|---|
| Cloud | AWS EC2 (ARM64 / Graviton), ECR, KMS, IAM, S3, Budgets |
| Infrastructure as Code | Terraform |
| Build AMI | Packer + AlmaLinux 9 (aarch64) |
| Provisionnement | Ansible |
| Conteneurisation | Docker, Docker Compose |
| Gestion des secrets | HashiCorp Vault (KV v2, auto-unseal via AWS KMS) |
| Logs | Elasticsearch · Kibana · Logstash — ELK 8.x (images Wolfi) |
| Monitoring | Grafana · Prometheus · Node Exporter |
| CI/CD | GitHub Actions |
| Sécurité OS | Firewalld, sshd hardening, Ansible Vault (AES256) |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    GitHub Actions                   │
│  build image → terraform apply → ansible deploy     │
└────────────────────┬────────────────────────────────┘
                     │
          ┌──────────▼──────────┐
          │      AWS Cloud      │  eu-north-1
          │                     │
          │  ┌───────────────┐  │
          │  │  EC2 APP      │  │  t4g.medium / 8GB
          │  │  Vault        │  │  port 8200
          │  │  App (ECR)    │  │  port 80
          │  └───────┬───────┘  │
          │          │ KMS unseal
          │  ┌───────▼───────┐  │
          │  │  AWS KMS      │  │
          │  └───────────────┘  │
          │                     │
          │  ┌───────────────┐  │
          │  │  EC2 ELK      │  │  t4g.medium / 20GB
          │  │  Elasticsearch│  │  port 9200
          │  │  Kibana       │  │  port 5601
          │  │  Logstash     │  │  port 5044
          │  └───────────────┘  │
          │                     │
          │  ┌───────────────┐  │
          │  │  EC2 GRAFANA  │  │  t4g.small / 8GB
          │  │  Grafana      │  │  port 3000
          │  │  Prometheus   │  │  port 9090
          │  │  Node Exporter│  │  port 9100
          │  └───────────────┘  │
          └─────────────────────┘
```

L'inventaire Ansible est généré automatiquement par Terraform à partir des IPs publiques des instances.

---

## Structure du projet

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml          # Pipeline CI/CD
├── ansible/
│   ├── ansible.cfg
│   ├── deploy.yml              # Playbook principal
│   ├── packer.yml              # Playbook pour la construction AMI
│   ├── group_vars/
│   │   ├── APP.yml
│   │   ├── ELK.yml
│   │   └── GRAFANA.yml
│   └── roles/
│       ├── security_os/        # Hardening SSH + Firewall
│       ├── docker_install/     # Installation Docker CE
│       ├── security_vault/     # Déploiement et init Vault
│       ├── go-server/          # Déploiement applicatif (ECR)
│       ├── elk-stack/          # Stack ELK
│       └── grafana-stack/      # Stack Grafana + Prometheus
├── terraform/
│   └── infra/
│       ├── main.tf             # Instances EC2 + Security Groups
│       ├── networking.tf
│       ├── iam.tf              # Rôles IAM par service
│       ├── kms.tf              # Clé KMS pour Vault auto-unseal
│       ├── backend.tf          # State S3
│       ├── budget.tf           # Alerte budgétaire AWS
│       ├── outputs.tf
│       └── variables.tf
└── packer/
    └── alma_pkr.hcl            # Build AMI AlmaLinux 9 ARM64
```

---

## Flux de déploiement

```
1. Packer          →  Build AMI AlmaLinux 9 (hardening + Docker)
2. Terraform       →  Provisionnement EC2 / IAM / KMS / SG
                       Génération automatique de l'inventaire Ansible
3. Ansible         →  Configuration des instances :
                       ├── Hardening OS (firewall, sshd, users)
                       ├── Installation Docker
                       ├── Déploiement Vault + initialisation
                       │   └── Injection des secrets (ELK, Grafana, DB, App)
                       ├── Déploiement stack ELK
                       ├── Déploiement stack Grafana
                       └── Déploiement application (pull ECR)
```

---

## Sécurité

### Gestion des secrets

Les secrets ne transitent jamais en clair dans le code. Le flux est le suivant :

- **Vault** est déployé sur l'instance APP et s'auto-déverrouille via **AWS KMS** (clé dédiée avec rotation activée).
- À l'initialisation, le **root token** est chiffré par **Ansible Vault (AES256)** et stocké localement dans `secrets.yml`.
- Les autres services (ELK, Grafana) récupèrent leurs secrets depuis Vault via un **Vault Agent**, en s'authentifiant avec leur rôle IAM AWS.

### Hardening OS

- Accès SSH par clés uniquement, `PermitRootLogin no`
- Firewall `firewalld` avec ouverture minimale des ports par groupe d'hôtes
- Création de comptes utilisateurs nominatifs par membre de l'équipe

### IAM

- Principe du moindre privilège : rôle IAM distinct par instance (Vault, ELK, Grafana)
- L'instance APP dispose des permissions KMS (encrypt/decrypt) et ECR (pull)
- Les instances ELK et Grafana ont uniquement `sts:GetCallerIdentity` pour l'authentification Vault

---

## Prérequis

- AWS CLI configuré avec les droits suffisants
- Terraform ≥ 1.0
- Packer ≥ 1.9
- Ansible ≥ 2.14
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
app_jwt_secret:    "..."
app_api_key:       "..."
```

### Construction de l'AMI (une seule fois)

```bash
cd packer
packer init .
packer build alma_pkr.hcl
```

### Déploiement complet

```bash
make deploy
```

### Destruction

```bash
make destroy        # Détruit les instances EC2
make destroy-full   # Détruit l'ensemble de l'infrastructure
```

---

## CI/CD (GitHub Actions)

Le pipeline se déclenche automatiquement sur push vers `main` ou `feature/infra`.

Il peut aussi être lancé manuellement avec les actions suivantes :

| Action | Description |
|---|---|
| `deploy` | Build image Docker → push ECR → terraform apply → ansible |
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

| Instance | Service | Port |
|---|---|---|
| APP | Application | 80 |
| APP | Vault | 8200 |
| ELK | Elasticsearch | 9200 |
| ELK | Kibana | 5601 |
| ELK | Logstash (Beats) | 5044 |
| GRAFANA | Grafana | 3000 |
| GRAFANA | Prometheus | 9090 |
| ALL | Node Exporter | 9100 |

---

## Roadmap

### WAF Nginx *(à venir)*

Ajout d'un reverse proxy Nginx en frontal avec des règles WAF pour filtrer le trafic entrant avant qu'il n'atteigne les services applicatifs.

### Gestion des tokens via eBPF *(vision cible)*

Les images officielles Elastic utilisent la variante **Wolfi** (distro minimaliste basée sur musl), qui ne supporte pas le mécanisme standard `*_FILE` des images Docker. Ce mécanisme — reposant sur le script `docker-entrypoint.sh` — permet normalement d'injecter des secrets depuis des fichiers montés (pattern Vault Agent), mais il est absent des images Wolfi.

La vision cible est de gérer le cycle de vie des tokens Vault via **eBPF** : interception des appels système au niveau kernel pour injecter et renouveler les secrets sans dépendre du comportement de l'image conteneur, et sans patch applicatif. Cette approche s'inspire des projets **Cilium** et **Tetragon**.

---

## Auteur

Nathan Michelle
