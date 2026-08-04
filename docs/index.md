# Welcome to Free Cloud Initiative Documentation

Welcome to the official documentation hub for **Free Cloud Initiative**.

The **Free Cloud Initiative** provides an end-to-end, multi-cloud, open-source platform for provisioning cloud infrastructure, bootstrapping High Availability (HA) Kubernetes (`K3s`) clusters, configuring edge networking, and managing applications via GitOps.

---

## 🗺 Quick Navigation

- 🏗 **[Architecture Overview](architecture.md)**: System design, 4-layer lifecycle, and repository breakdown.
- 🌩 **[Infrastructure with Terraform](terraform.md)**: Multi-cloud VM provisioning (`GCP`, `Azure`, `AWS`) and Cloudflare DNS/Tunnels.
- ⚙️ **[Cluster & Automation with Ansible](ansible-k3s.md)**: K3s multi-master setup, OS configuration, and base operator installation.
- 🔁 **[GitOps with K3s Manifests](gitops-manifests.md)**: Declarative Kubernetes application deployment using ArgoCD.
- 🚀 **[Getting Started](getting-started.md)**: Instructions for running and editing this documentation site locally.

---

## 🌟 Key Highlights

- **Multi-Cloud Freedom**: Seamlessly target GCP, Azure, AWS, or bare-metal hardware like Raspberry Pi clusters.
- **Automated K3s HA Setup**: Complete Ansible playbooks for zero-downtime multi-master control planes.
- **Edge Networking**: Secure Cloudflare Zero Trust Tunnels eliminating open public ports.
- **GitOps Continuous Delivery**: ArgoCD-managed cluster workloads driven entirely by Git repository commits.
