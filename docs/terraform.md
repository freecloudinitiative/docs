# Infrastructure Management with Terraform

Terraform forms **Layer 1** of the Free Cloud Initiative platform. It handles cloud resource allocation, compute instance creation, network firewall provisioning, and edge DNS routing.

The platform splits infrastructure into two dedicated repositories:
1. **`terraform-multicloud-infra`**: Manages cloud VMs, networking, and security rules across cloud providers.
2. **`terraform-cloudflare-infra`**: Manages Cloudflare DNS records, TLS certificates, and Zero Trust Tunnels.

---

## 🌩 1. Multi-Cloud Provisioning (`terraform-multicloud-infra`)

This repository provisions virtual machines (VMs) for Kubernetes master and worker nodes across GCP, Azure, or AWS.

### Directory Structure

```text
terraform-multicloud-infra/
├── gcp/     # GCP Compute Engine instances, VPC, Firewall rules
├── azure/   # Azure Virtual Machines, Virtual Network, NSGs
└── aws/     # AWS EC2 instances, VPC, Security Groups
```

### What You Do With This Repo:

1. **Spin up compute nodes**: Define how many master nodes (e.g. `master-1`, `master-2`, `master-3` for HA) and worker nodes you require.
2. **Configure network access**: Restrict SSH access to allowed admin IP ranges via firewall rules.
3. **Export Node IPs**: Terraform outputs public and private IP addresses of the created nodes. These IPs are fed directly into Ansible's inventory (`inventory.ini`).

### Quick Start Example (GCP)

```bash
cd gcp

# Set project ID and limit SSH access to current public IP
export TF_VAR_gcp_project_id="your-gcp-project-id"
export TF_VAR_gcp_admin_ip_ranges="[\"$(curl -s https://ipinfo.io/ip)/32\"]"

# Initialize state and apply
terraform init -backend-config="bucket=tf-state-$TF_VAR_gcp_project_id"
terraform plan
terraform apply
```

---

## 🌐 2. DNS & Edge Networking (`terraform-cloudflare-infra`)

This repository manages domain mapping for `freecloudinitiative.com` and Zero Trust Tunnels to securely expose services without opening public ports to the internet.

### What You Do With This Repo:

1. **DNS Record Management**: Create `A` and `CNAME` records pointing subdomains (e.g., `argocd.freecloudinitiative.com`, `gitea.freecloudinitiative.com`) to ingress controllers or Cloudflare Tunnels.
2. **Cloudflare Zero Trust Tunnels**: Provision `cloudflared` tunnels to route encrypted traffic directly to internal cluster services.
3. **Automated CI/CD**: Integrated with Terraform Cloud and GitHub Actions for state locking and automated PR checking.

### Quick Start Example

```bash
# Set Cloudflare credentials
export TF_VAR_cloudflare_api_token="your-token"
export TF_VAR_account_id="your-account-id"
export TF_VAR_zone_id="your-zone-id"

terraform init
terraform plan
terraform apply
```
