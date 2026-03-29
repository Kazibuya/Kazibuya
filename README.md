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
| Conteneurisation | Docker (PostgreSQL hors K3s) | ✅ |
| Gestion des secrets | HashiCorp Vault (KV v2, auto-unseal via AWS KMS) | ✅ |
| Vault distribution | Vault Agent Injector (sidecar K8s) | 🔄 en cours |
| Vault PKI + mTLS | Vault PKI secrets engine | ⏳ à venir |
| Base de données | PostgreSQL (Docker, secrets via Vault Agent) | ✅ |
| Logs | Elasticsearch · Kibana · Logstash — ELK 8.x | ⏳ à venir |
| Monitoring | Grafana · Prometheus · Node Exporter | ⏳ à venir |
| WAF | Nginx Ingress + ModSecurity (OWASP CRS) | 🔄 en cours |
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
              │  │   Vault (K3s)   │    │  port 30820 (NodePort)
              │  │   PostgreSQL    │    │  Docker (hors K3s)
              │  │   App Go (K3s)  │    │  port 30080/30443
              │  │   Nginx WAF     │    │  Ingress Controller
              │  └────────┬────────┘    │
              │           │ KMS unseal  │
              │  ┌────────▼────────┐    │
              │  │    AWS KMS      │    │
              │  └─────────────────┘    │
              │                         │
              │  ┌─────────────────┐    │
              │  │   EC2 WORKER1   │    │  t4g.medium / 20GB
              │  │   K3s agent     │    │
              │  │   ELK pods      │    │  ⏳ à venir
              │  └─────────────────┘    │
              │                         │
              │  ┌─────────────────┐    │
              │  │   EC2 WORKER2   │    │  t4g.small / 8GB
              │  │   K3s agent     │    │
              │  │   Grafana pods  │    │  ⏳ à venir
              │  └─────────────────┘    │
              └─────────────────────────┘

  Réseau inter-nodes : chiffré via Flannel + WireGuard natif K3s
  Secrets : distribués via Vault Agent Injector (sidecar tmpfs)
  WAF : Nginx Ingress Controller + ModSecurity OWASP CRS
```

L'inventaire Ansible est généré automatiquement par Terraform à partir des IPs publiques des instances.

---

## Décisions d'architecture notables

### Module Terraform `vault-config` — principe du moindre privilège

![Sceenshot HCL][./HCL.png]

Chaque service dispose de sa propre policy Vault générée automatiquement par ce module. La policy donne accès uniquement à `secret/data/{service_name}/*` — jamais aux secrets des autres services. Le paramètre `extra_paths` permet d'ajouter des accès supplémentaires de façon explicite (ex: l'app Go accède aussi à `secret/data/db/*` pour les credentials de connexion PostgreSQL). C'est le **principe du moindre privilège** appliqué à Vault — chaque service ne voit que ce dont il a besoin.

### Firewalld par groupe de nodes — ports ouverts selon le rôle

![Screenshot YAML][YAML.png]

Les ports firewalld sont définis par groupe Ansible (`group_vars/MASTER.yml`, `group_vars/WORKERS.yml`) et non dans l'inventory ou dans le rôle. La variable `extra_ports` est surchargée par groupe — chaque node ouvre uniquement les ports correspondant à son rôle dans le cluster. Le rôle `security_os` reste générique et réutilisable : il ne connaît pas les ports K3s ou Vault. C'est la **séparation des responsabilités** — le rôle gère le mécanisme, les `group_vars` gèrent la configuration.

### PostgreSQL hors K3s

PostgreSQL tourne en Docker directement sur le master, hors du cluster K3s. Ce choix est intentionnel — les bases de données sont des **stateful workloads** difficiles à gérer dans Kubernetes. Garder la DB hors K8s garantit l'indépendance : si K3s a un problème, la DB reste accessible. Les credentials sont gérés via Vault Agent sidecar Docker, montés en tmpfs via `*_FILE`.

### Vault dans K3s

Vault tourne dans K3s via le chart officiel HashiCorp. La configuration (KMS unseal, storage) est injectée via `standalone.config` dans `values.yaml.j2` — Ansible template le fichier avec `kms_key_id` avant le `helm install`. Cette approche évite les ConfigMaps manuels et laisse Helm gérer ses propres ressources nativement.

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
│   │   ├── MASTER.yml              # Ports firewalld master (K3s + Vault + WireGuard)
│   │   └── WORKERS.yml             # Ports firewalld workers (K3s + WireGuard)
│   └── roles/
│       ├── security_os/            # Hardening SSH + Firewall
│       ├── docker_install/         # Installation Docker CE
│       ├── security_vault/         # Déploiement Vault (Helm) + init + feed secrets
│       ├── postgres/               # PostgreSQL Docker + Vault Agent sidecar
│       ├── k3s_master/             # K3s master + Helm repos + Ingress nginx
│       ├── k3s_worker/             # K3s worker nodes
│       ├── go-server/              # [archivé → Helm chart]
│       ├── elk-stack/              # [archivé → Helm chart officiel]
│       └── grafana-stack/          # [archivé → Helm chart officiel]
├── helm/
│   ├── app/                        # Chart custom app Go 🔄
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml     # Pod + Vault Agent Injector annotations
│   │       ├── service.yaml        # ClusterIP service
│   │       └── ingress.yaml        # Nginx + ModSecurity WAF
│   └── values/
│       ├── vault.yaml.j2           # Values Vault (kms_key_id injecté par Ansible)
│       └── ingress-nginx.yaml      # Values Nginx Ingress + ModSecurity OWASP CRS
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
│   │   ├── main.tf                 # Vault AWS auth backend + policies + roles
│   │   ├── providers.tf            # Provider Vault + remote_state infra
│   │   ├── backend.tf              # State S3 (terraform-vault.tfstate)
│   │   └── variables.tf
│   └── modules/
│       ├── compute/                # Module EC2 réutilisable
│       └── vault-config/           # Module policy + role Vault (1 par service)
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
                       ├── K3s master (+ Helm repos + Ingress nginx)
                       ├── Vault (Helm install + init + feed secrets)
                       ├── PostgreSQL (Docker + Vault Agent sidecar)
                       └── K3s workers (join cluster)
4. Terraform vault →  Configuration Vault :
                       ├── AWS auth backend
                       └── Policies + roles (app, db, elk, grafana)
5. Helm            →  Déploiement des services dans K3s :
                       ├── Vault Agent Injector (déjà installé avec Vault)
                       ├── Nginx Ingress + ModSecurity WAF ✅
                       ├── App Go (chart custom) 🔄
                       ├── ELK (chart officiel Elastic) ⏳
                       └── Grafana + Prometheus ⏳
```

---

## Sécurité

### Gestion des secrets

Les secrets ne transitent jamais en clair dans le code :

- **Vault** tourne dans K3s, auto-unsealed via **AWS KMS**. Root token chiffré AES256 (Ansible Vault) et stocké dans S3 versionné.
- **Vault Agent Injector** — pour les pods K8s : injecte les secrets via sidecar, montés en tmpfs. Zéro variable d'environnement exposée.
- **Vault Agent Docker** — pour PostgreSQL hors K3s : même pattern, sidecar Docker, tmpfs, `*_FILE` Postgres.
- Les secrets sont toujours lus depuis `/vault/secrets/` ou `/secrets/` — jamais depuis des variables d'environnement.

### Hardening OS

- Accès SSH par clés uniquement, `PermitRootLogin no`
- Firewall `firewalld` avec ouverture minimale des ports par groupe de nodes
- Création de comptes utilisateurs nominatifs par membre de l'équipe
- `sudo secure_path` étendu pour inclure `/usr/local/bin`

### Réseau

- Chiffrement inter-nodes via **Flannel + WireGuard natif** (K3s built-in)
- WAF **ModSecurity OWASP CRS** sur l'Ingress Controller nginx
- Un seul rôle IAM `k3s-role` pour toutes les instances

### IAM

- Principe du moindre privilège : un seul rôle IAM `k3s-role`
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
make plan                                    # Affiche le plan Terraform
make ansible                                 # Relance Ansible seul
make ansible role=k3s_master                 # Relance un rôle spécifique
make kubectl cmd="get nodes"                 # Liste les nodes K3s
make kubectl cmd="get pods -A"               # Liste tous les pods
make kubectl cmd="logs vault-0 -n vault"     # Logs d'un pod
make output                                  # Affiche les IPs et outputs
make destroy                                 # Détruit les instances EC2
make destroy-full                            # Détruit toute l'infrastructure
```

---

## CI/CD (GitHub Actions)

Le pipeline se déclenche automatiquement sur push vers `main` ou `feature/infra`.

| Action | Description |
|---|---|
| `deploy` | Build image → push ECR → terraform → ansible → helm |
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
| MASTER | Vault (NodePort) | 30820 | TCP |
| MASTER | Vault UI (NodePort) | 30821 | TCP |
| MASTER | HTTP (Ingress nginx) | 30080 | TCP |
| MASTER | HTTPS (Ingress nginx) | 30443 | TCP |
| MASTER | PostgreSQL | 5432 | TCP |
| MASTER | etcd | 2379-2380 | TCP |
| ALL | Kubelet | 10250 | TCP |
| ALL | Flannel VXLAN | 8472 | UDP |
| ALL | WireGuard | 51820 | UDP |
| ALL | NodePort range | 30000-32767 | TCP |
| ALL | Node Exporter | 9100 | TCP |

---

## Roadmap

### App Go dans K3s *(en cours)*

Helm chart custom avec Vault Agent Injector pour l'injection des secrets (zéro variable d'environnement), Ingress nginx avec ModSecurity OWASP CRS pour le WAF.

### ELK dans K3s *(à venir)*

Charts officiels Elastic déployés sur worker1, logs centralisés depuis tous les services.

### Grafana + Prometheus *(à venir)*

Charts officiels sur worker2, dashboards préconfigurés, alerting Prometheus.

### Vault PKI + mTLS *(à venir)*

PKI secrets engine pour générer et renouveler automatiquement les certificats TLS. mTLS entre tous les services du cluster via Vault Agent Injector.

### HA K3s *(à venir)*

Migration vers 3 masters + etcd distribué + NLB AWS. Le code Terraform est structuré pour faciliter cette migration depuis l'architecture 1 master actuelle.

### eBPF + Cilium *(vision cible)*

Remplacement de Flannel par Cilium CNI :
- Network policies L3/L4/L7 via eBPF
- Transparent encryption WireGuard inter-nodes
- Observabilité kernel-level via **Tetragon**
- mTLS natif sans modification applicative

### Tests *(à venir)*

- **Molecule** — tests des rôles Ansible
- **Terratest** — tests des modules Terraform

### devops-kit *(à venir)*

Extraction en repo réutilisable avec documentation et variables standardisées.

### Rôles Ansible supplémentaires *(à venir)*

Le projet a vocation à couvrir un maximum de cas d'usage infrastructure. Des rôles sont prévus pour enrichir le template :

- **waf** — durcissement ModSecurity standalone (règles custom OWASP, whitelisting, audit log)
- **wireguard** — VPN mesh inter-instances hors K3s (backup si Flannel indisponible)
- **hardening** — CIS Benchmark AlmaLinux 9 (auditd, sysctl, PAM, SELinux)
- **backup** — pg_dump PostgreSQL → S3 chiffré, rotation automatique
- **vault-pki** — génération et renouvellement automatique des certificats TLS via Vault PKI engine
- **trivy** — scan des images Docker/Wolfi avant push ECR (intégré dans le pipeline CI)
- **falco** — détection d'intrusion runtime (comportements suspects dans les pods K8s)

---

## Vision long terme

### Images Wolfi — sécurité by design

L'objectif à terme est de n'utiliser **que des images Wolfi** pour tous les services du projet. Wolfi est une distribution Linux minimaliste basée sur musl libc, conçue spécifiquement pour les containers :

```
Images standard :
└── Ubuntu/Alpine base → des centaines de packages inutiles
    → surface d'attaque élevée
    → CVE réguliers sur des composants jamais utilisés

Images Wolfi :
└── Seulement les binaires nécessaires
    → zéro shell par défaut
    → zéro package manager
    → surface d'attaque minimale
    → CVE quasi inexistants
```

Elastic propose déjà des images Wolfi (`elasticsearch-wolfi`, `kibana-wolfi`) — elles sont actuellement commentées dans le projet le temps de résoudre les problèmes de compatibilité avec les mécanismes d'injection de secrets (`*_FILE` absent des entrypoints Wolfi). La solution envisagée passe par eBPF.

### eBPF — la vision kernel-level *(projet d'une vie 💎)*

> 💎 C'est le cœur du projet à long terme. Tout ce qui précède (K3s, Vault, Helm) n'est que le socle sur lequel cette vision repose. L'objectif final est de construire une infrastructure où la sécurité, l'observabilité et la gestion des secrets sont gérées **au niveau kernel**, de façon transparente pour toutes les applications, sans modifier une seule ligne de code applicatif.

Cette vision nécessite une maîtrise profonde du kernel Linux, du verifier eBPF, des BPF maps, et des outils comme Cilium, Tetragon, libbpf et bpftool. C'est un travail de longue haleine, documenté ici comme feuille de route personnelle.

---

**1. Gestion du cycle de vie des secrets via sondes eBPF**

Plutôt que de gérer la rotation des secrets au niveau applicatif (Vault Agent qui écrit des fichiers, l'app qui relit), l'idée est d'intercepter les appels système directement au niveau kernel :

```
Sonde eBPF sur sys_read / sys_open :
→ intercepte quand un process lit /vault/secrets/
→ vérifie si le secret est expiré (TTL Vault via BPF map)
→ si expiré : déclenche le renouvellement automatique
→ injecte la nouvelle valeur directement dans l'espace mémoire du process
→ zéro restart, zéro downtime, transparent pour l'application
→ l'app ne sait même pas que Vault existe
```

C'est le pattern **"secret injection sans modification applicative"** — et c'est la solution propre au problème des images Wolfi qui ne supportent pas les entrypoints `*_FILE`.

---

**2. Bloom filter eBPF pour ELK — déduplication et vitesse**

Les logs volumiques posent deux problèmes : la déduplication (événements identiques répétés) et la vitesse de recherche. La solution envisagée attaque les deux simultanément via eBPF + Redis.

```
Flux de logs → sonde eBPF XDP (eXpress Data Path) :
→ hash de chaque événement log (xxHash 64bit)
→ vérification dans le Bloom filter (BPF map probabiliste)
→ si déjà vu récemment : drop au niveau kernel (< 1μs)
→ si nouveau :
   ├── tee eBPF TC (Traffic Control) → clone le paquet
   ├── original → Logstash → Elasticsearch (stockage long terme)
   └── clone    → Redis Streams (stockage court terme, haute vélocité)
```

**Tee eBPF** — duplication des logs sans overhead :
```
Un seul flux entrant → deux destinations simultanées
→ Elasticsearch : recherche historique (secondes)
→ Redis Streams : recherche temps réel (< 1ms, last 1h)

Pattern de recherche :
→ query → Redis d'abord (fenêtre glissante 1h)
→ cache miss → Elasticsearch (historique complet)
→ TTL Redis = 1h → rotation automatique, zéro maintenance
```

C'est un ordre de magnitude plus rapide qu'un filtre applicatif Logstash (μs vs ms), sans modifier une seule ligne de code d'ELK.

---

**3. Persistance des BPF maps via Redis — survie aux crashes**

Les BPF maps vivent dans le kernel space — elles sont perdues si le node crashe ou si le programme eBPF est rechargé. Pour le Bloom filter, cela signifie un warmup period pendant lequel des doublons passent, dégradant la qualité des données.

```
Solution : state externalization via Redis

Daemon Go (userspace) :
→ snapshot des BPF maps toutes les 30s
→ sérialise en binaire (.bin) via encoding/binary
→ stocke dans Redis avec TTL = 2x la fenêtre de déduplication
→ clé : "bpf:bloom:{node_name}:{timestamp}"

Au reload du programme eBPF :
→ lit le dernier snapshot depuis Redis
→ restaure le state dans les BPF maps
→ warmup = 0, continuité parfaite
```

```
Architecture complète :
┌─────────────────────────────────────────────┐
│  Kernel space                               │
│  ┌──────────────────┐  ┌─────────────────┐  │
│  │  eBPF XDP prog   │  │   BPF maps      │  │
│  │  (Bloom filter)  │←→│  (ring buffer)  │  │
│  └──────────────────┘  └────────┬────────┘  │
└───────────────────────────────── │ ──────────┘
                                   │ snapshot
┌──────────────────────────────────▼──────────┐
│  Userspace                                  │
│  ┌──────────────────┐                       │
│  │  Go daemon       │                       │
│  │  (bpf2redis)     │──→ Redis (.bin state) │
│  └──────────────────┘                       │
└─────────────────────────────────────────────┘
```

---

**4. Observabilité kernel-level avec Tetragon**

Tetragon (projet Cilium) permet d'observer et de bloquer des comportements suspects directement au niveau kernel, sans agent applicatif :

```
Exemples de politiques eBPF Tetragon :
→ "aucun process dans vault-0 ne peut lire /etc/passwd"
→ "aucune connexion réseau sortante depuis postgres vers l'extérieur"
→ "tout appel execve() dans un pod de production → alerte immédiate"
→ "si un process écrit dans /vault/data/ sans être vault → kill -9"
→ "tout appel open() sur une clé privée TLS → log + alerte SIEM"
```

C'est du **Zero Trust au niveau kernel** — les policies sont appliquées avant même que le syscall n'atteigne le process, impossible à contourner depuis l'espace utilisateur.

---

> La beauté de cette approche : une infrastructure où **zéro application ne connaît Vault**, **zéro log n'est dupliqué inutilement**, **zéro secret n'est jamais en clair en mémoire userspace**, et **zéro comportement suspect ne peut passer inaperçu** — le tout géré par des programmes de quelques centaines de lignes qui s'exécutent directement dans le kernel Linux.

---

### Multi-roles Ansible *(à venir)*

À terme, chaque responsabilité sera isolée dans un rôle Ansible dédié et indépendant — pas seulement `security_os` ou `k3s_master`, mais des rôles fins pour chaque composant : hardening kernel, configuration réseau, rotation de secrets, observabilité... L'objectif est un catalogue de rôles composables, testés avec Molecule, utilisables indépendamment sur n'importe quelle infra.

### Images Wolfi — baseline sécurité *(à venir)*

Migration complète vers des images **Wolfi** pour tous les containers du projet. Wolfi est une distro minimaliste (musl, pas de shell, pas de package manager) qui réduit drastiquement la surface d'attaque :

```
Image standard ubuntu:22.04   → ~700 CVE potentielles
Image Wolfi équivalente       → ~0-5 CVE
```

Pas de shell exploitable, pas de packages inutiles, images 3 à 10x plus légères. C'est la baseline sécurité pour toute image de production sérieuse — Elasticsearch, Kibana, Logstash, Grafana, app Go, tout passera en Wolfi.

---

## Auteur

Nathan Michelle
