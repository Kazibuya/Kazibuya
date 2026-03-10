# 🛡️ Nathan MICHELLE (Kazibuya)
> **Ex-Sauveteur en Mer 🌊 | Ex-Mesures Physiques (Orsay) 🔬 | Étudiant à 42 💻**

Ancien sauveteur en mer, j'applique aujourd'hui la même rigueur à la survie des infrastructures critiques. Spécialisé en **Cloud Architecture** et **Security Automation**, avec une obsession pour le **Zero Trust**.

---

### 🏗️ Current Project: Secure CI/CD Architecture Template
*Architecture multi-instances sur **AWS** pilotée par **Terraform**, **Packer** & **Ansible**.*

* **Instance 1 (App):** Backend Go + WAF + **HashiCorp Vault** (Agent-side sidecar) + PostgreSQL.
* **Instance 2 (Monitoring):** Stack Prometheus & Grafana avec injection de secrets via Vault.
* **Instance 3 (Logging):** Stack **ELK** basée sur des images **Wolfi** (Distroless/Hardened).
* **Hardening:** Utilisation intensive de **Packer** pour baker des AMI immuables.
* **Security Core:** Auto-unseal via **AWS KMS**, isolation réseau stricte et rotation dynamique des secrets via Vault.

---

### 🛠️ Tooling & Stack
* **IaC / Config:** Terraform, **Packer**, Ansible, Terragrunt.
* **Languages:** **Go** (Mastering for Molecule & Terratest), C.
* **Cloud & Security:** AWS (KMS, IAM, VPC), HashiCorp Vault, Zero Trust Architecture.
* **Observability:** Prometheus, Grafana, ELK... **next stop: eBPF 🐝** (pour l'observabilité et la sécurité kernel-level).
