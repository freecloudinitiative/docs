# Infrastructure with Terraform

<span class="step-badge">01 / PROVISION</span>

FCI keeps cluster infrastructure, production edge configuration, and CI runner capacity in separate Terraform repositories so they can be created, changed, and destroyed independently.

## Repository split

| Repository | Lifecycle | Scope |
| --- | --- | --- |
| **terraform-multicloud-infra** | On demand | Cluster nodes, networks, and firewall rules |
| **terraform-cloudflare-infra** | Always on in production | DNS records and one Zero Trust tunnel configuration |
| **terraform-multicloud-runner** | Independent CI capacity | Self-hosted GitHub Actions runner VMs |

~~~mermaid
flowchart LR
    TF["terraform-multicloud-infra"] --> N["cluster nodes + networks"]
    N --> A["Ansible inventory"]
    CF["terraform-cloudflare-infra"] --> E["Cloudflare edge"]
    E --> T["in-cluster cloudflared → Traefik"]
    R["terraform-multicloud-runner"] --> CI["self-hosted Actions runners"]
    CI --> W["shared .github workflows"]
~~~

## Cluster infrastructure

[terraform-multicloud-infra](https://github.com/freecloudinitiative/terraform-multicloud-infra) contains an independent root per provider:

~~~text
terraform-multicloud-infra/
├── aws/
├── azure/
├── civo/
├── gcp/
└── linode/
~~~

Implemented resources vary by provider. GCP and AWS include the fullest node/network/firewall models; AWS also supports a secondary VPC with peering. Civo and Linode define provider-native instances and firewalls. The Azure root is earlier-stage and currently contains only part of the intended cluster resource set.

!!! info "Treat each provider root independently"
    Each directory has its own providers, variables, outputs, and backend expectations. Initialize, plan, and apply from the selected directory; do not assume provider roots have identical feature depth.

### Handoff to Ansible

Terraform outputs supply the public/private addresses and node topology for the environment's Ansible inventory. K3s join traffic must use an address reachable by all nodes—private VPC addresses in non-production—while public addresses are reserved for operator access.

~~~bash
cd terraform-multicloud-infra/gcp
terraform init
terraform plan
terraform apply
terraform output
~~~

Restrict SSH CIDRs before applying. Defaults that allow <code>0.0.0.0/0</code> are development conveniences, not a safe deployment policy.

## Production edge

[terraform-cloudflare-infra](https://github.com/freecloudinitiative/terraform-cloudflare-infra) creates:

1. the root and configured service DNS records;
2. the <code>freecloud-k3s-tunnel</code> Cloudflare tunnel;
3. hostname-to-origin ingress rules with a catch-all 404;
4. the tunnel token consumed by the in-cluster <code>cloudflared</code> workload.

It does **not** run <code>cloudflared</code> or terminate application HTTP. <code>k3s-manifests/infrastructure/cloudflared</code> runs the connector, and Traefik owns routing inside the cluster.

Services marked <code>internal_only</code> are removed from tunnel ingress and receive an unproxied LAN address instead. Administrative endpoints should remain internal.

~~~bash
cd terraform-cloudflare-infra
export CLOUDFLARE_API_TOKEN="..."
export TF_VAR_account_id="..."
export TF_VAR_zone_id="..."
terraform init
terraform plan
~~~

!!! warning "Production state"
    The committed backend is local, while production automation owns the live state. Do not apply production from a fresh laptop state: Terraform would not know which edge resources already exist.

## CI runner infrastructure

[terraform-multicloud-runner](https://github.com/freecloudinitiative/terraform-multicloud-runner) provisions organization runners separately from application infrastructure. GCP, Azure, and Civo roots include VM bootstrap scripts that register multiple GitHub Actions runners per VM and support simple/HA counts. AWS and Linode currently contain provider scaffolding, not complete runner deployments.

This repository exists because many FCI builds target ARM64 and some workflows need access to project-local infrastructure. Runner capacity can change without touching the K3s cluster.

!!! danger "Runner registration credentials"
    GitHub registration tokens and PATs are sensitive Terraform inputs. Supply them through the workflow secret store or environment variables and keep state access tightly controlled.

## Validation and workflow ownership

Repository workflows call reusable jobs maintained in [.github](https://github.com/freecloudinitiative/.github). The shared Terraform checks format and validate every selected root; apply and destroy remain explicit workflows with provider credentials supplied as secrets.

Before a pull request:

~~~bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

Run <code>init</code> and <code>validate</code> in each root you changed.

[Continue to cluster bootstrap →](ansible-k3s.md){ .md-button .md-button--primary }
