<div class="hero" markdown>

# Free Cloud Initiative

**Learn DevOps end-to-end through a real, production-grade project.**  
From spinning up cloud VMs with Terraform to shipping apps via GitOps — follow every step, understand every decision.

<div class="hero-buttons" markdown>
<a href="learning-path/" class="btn-primary">Start Learning →</a>
<a href="https://github.com/freecloudinitiative" class="btn-outline" target="_blank">View on GitHub</a>
</div>

</div>

---

## What is the Free Cloud Initiative?

The **Free Cloud Initiative** is an open-source, end-to-end DevOps platform **and** a portfolio teaching project. It is designed for engineers who want to:

- 🎓 **Learn real DevOps** — not toy examples, but production patterns
- 🏗️ **See the full stack** — from raw cloud VMs to deployed microservices
- 💼 **Build a portfolio** — every repo, every commit, is your proof of work

The project walks through four layers of a modern DevOps pipeline, each layer a real GitHub repository you can fork, clone, and run yourself.

---

## The 4-Layer DevOps Journey

<div class="feature-grid" markdown>

<div class="feature-card" markdown>
<div class="card-icon">🌩️</div>
<span class="step-badge">Step 1</span>

### Terraform
Provision cloud VMs, VPCs, and firewall rules across **GCP, Azure, and AWS** using infrastructure-as-code. Learn multi-cloud provisioning and Cloudflare DNS automation.

[→ Terraform Docs](terraform.md)
</div>

<div class="feature-card" markdown>
<div class="card-icon">⚙️</div>
<span class="step-badge">Step 2</span>

### Ansible + K3s
Automate OS configuration and bootstrap a **multi-master HA K3s cluster** using Ansible playbooks and roles. Zero manual SSH commands.

[→ Ansible & K3s Docs](ansible-k3s.md)
</div>

<div class="feature-card" markdown>
<div class="card-icon">🔁</div>
<span class="step-badge">Step 3</span>

### GitOps & ArgoCD
Deploy every application declaratively using **ArgoCD App-of-Apps pattern**. Git is the single source of truth — push a commit, see it live.

[→ GitOps Docs](gitops-manifests.md)
</div>

<div class="feature-card" markdown>
<div class="card-icon">📊</div>
<span class="step-badge">Step 4</span>

### Observability
Full metrics, logs, and traces with **Prometheus, Grafana, Loki, Tempo, and Alloy**. Because you can't run what you can't see.

[→ Architecture Overview](architecture.md)
</div>

</div>

---

## Repository Map

| Repository | Layer | What You'll Learn |
| :--- | :---: | :--- |
| [`terraform-multicloud-infra`](https://github.com/freecloudinitiative/terraform-multicloud-infra) | 1 | Cloud VMs, networking, firewall rules (GCP/Azure/AWS) |
| [`terraform-cloudflare-infra`](https://github.com/freecloudinitiative/terraform-cloudflare-infra) | 1 | DNS, TLS, Cloudflare Zero Trust Tunnels |
| [`ansible-automation`](https://github.com/freecloudinitiative/ansible-automation) | 2–3 | OS config, K3s multi-master install, core operator deployment |
| [`k3s-manifests`](https://github.com/freecloudinitiative/k3s-manifests) | 4 | Kubernetes manifests, Helm releases, GitOps App-of-Apps |
| [`docs`](https://github.com/freecloudinitiative/docs) | — | This documentation site |

---

!!! tip "Portfolio Note"
    Every decision in this project is intentional and explained. Whether you're preparing for a DevOps interview or want to understand how production Kubernetes infrastructure is built — this project has you covered.
