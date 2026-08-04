# Cluster Setup & Automation with Ansible (`ansible-automation`)

The **`ansible-automation`** repository represents **Layer 2 and Layer 3** of the Free Cloud Initiative platform. It handles server configuration, multi-master K3s Kubernetes installation, and base cluster service bootstrapping.

---

## 🎯 What You Do With The `ansible-automation` Repo

Once Terraform provisions your compute instances, you use Ansible to:

1. **Configure Host OS & Networking**: Set hostnames, adjust kernel parameters (cgroups for container runtimes), configure Raspberry Pi options (`pi-boot.yml`), and manage SSH access (`ssh-config.yml`).
2. **Bootstrap K3s Cluster**:
   - Initialize K3s primary master node (`k3s-master-setup`).
   - Join additional control-plane nodes for High Availability (HA multi-master).
   - Join worker nodes (`k3s-worker-setup`).
   - Assign node labels and gather facts (`k3s-node-labeling`, `k3s-fact-gathering`).
3. **Deploy Core Kubernetes Operators**:
   - **Ingress & Networking**: Traefik Ingress Controller, MetalLB Load Balancer.
   - **Security & TLS**: Cert-Manager for SSL certificates, Sealed Secrets for encrypted secrets management.
   - **GitOps Engine**: ArgoCD setup & initial root application bootstrap (`argocd-setup`, `argocd-bootstrap`).
   - **Internal Tooling**: Gitea, Gitea Act Runners, Harbor Registry.
   - **Observability Stack**: Prometheus, Grafana, Loki, Tempo, OpenTelemetry Collector, Grafana Alloy.

---

## 📂 Key Directory & Role Layout

```text
ansible-automation/
├── inventory.ini           # Master & Worker node IP addresses and groups
├── playbook.yml            # Main playbook running end-to-end setup
├── ssh-config.yml          # Playbook to generate ~/.ssh/config & purge old host keys
├── group_vars/
│   └── all/
│       ├── vars.yml        # Public configuration variables
│       └── secret.yml      # Ansible Vault encrypted secrets (passwords, tokens)
└── roles/                  # Modular Ansible roles (20+ roles)
    ├── k3s-pre-setup       # OS prerequisites, cgroups, swap disabled
    ├── k3s-master-setup    # Installs K3s control plane & extracts cluster token
    ├── k3s-worker-setup    # Joins worker nodes to control plane
    ├── traefik-setup       # Deploys Traefik Helm chart
    ├── argocd-setup        # Deploys ArgoCD operator
    └── argocd-bootstrap    # Registers k3s-manifests GitOps repo with ArgoCD
```

---

## 🚀 Execution Workflow

### 1. Update Inventory (`inventory.ini`)

Populate node IP addresses obtained from Terraform output:

```ini
[masters]
master-1 ansible_host=34.72.134.198 ansible_user=ubuntu
master-2 ansible_host=34.72.135.100 ansible_user=ubuntu
master-3 ansible_host=34.72.136.200 ansible_user=ubuntu

[workers]
worker-1 ansible_host=34.72.137.50 ansible_user=ubuntu
worker-2 ansible_host=34.72.137.51 ansible_user=ubuntu
```

### 2. Configure Local SSH Config

Automatically generate your workstation's `~/.ssh/config` and remove stale host keys:

```bash
ansible-playbook ssh-config.yml
```

### 3. Run Cluster Bootstrap Playbook

Run the master playbook with Ansible Vault credentials:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

---

## 🔒 Secret Management with Ansible Vault

Sensitive credentials (such as admin passwords, tokens, and private keys) are stored in `group_vars/all/secret.yml` using Ansible Vault:

```bash
# Encrypt secrets file
ansible-vault encrypt group_vars/all/secret.yml

# Edit encrypted secrets file
ansible-vault edit group_vars/all/secret.yml
```
