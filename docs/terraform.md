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

### Provider and state boundaries

The cluster roots currently constrain the AWS provider to major version 5, Google to 5, AzureRM to 3, Civo to 1.1, and Linode to 2. The Cloudflare root requires Terraform 1.5 or newer and constrains Cloudflare to major version 4. Treat these constraints as compatibility ranges, not an instruction to upgrade automatically: refresh lock files and inspect provider release notes in the repository that owns the root.

Every root has an independent state address space. Resource addresses such as `aws_instance.master[0]` are meaningful only with the matching root configuration, variables, provider account, and state. Copying resources between directories without an explicit state migration can cause Terraform to propose destruction and recreation.

> [!NOTE]
> **Treat each provider root independently**
>
> Each directory has its own providers, variables, outputs, and backend expectations. Initialize, plan, and apply from the selected directory; do not assume provider roots have identical feature depth.

### Handoff to Ansible

Terraform outputs supply the public/private addresses and node topology for the environment's Ansible inventory. K3s join traffic must use an address reachable by all nodes—private VPC addresses in non-production—while public addresses are reserved for operator access.

~~~bash
cd terraform-multicloud-infra/gcp
terraform init
terraform plan
terraform apply
terraform output
~~~

Read a plan by operation, not only by its summary:

- `+` creates a new remote object;
- `~` changes one in place;
- `-/+` replaces it and can interrupt a node or network path;
- `-` destroys it;
- `(known after apply)` means a downstream value cannot be finalized during planning.

Save an authorized production plan and apply that exact artifact so the reviewed operations and executed operations are the same:

~~~bash
terraform plan -out=tfplan
terraform show tfplan
terraform apply tfplan
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

> [!WARNING]
> **Production state**
>
> The committed backend is local, while production automation owns the live state. Do not apply production from a fresh laptop state: Terraform would not know which edge resources already exist.

## CI runner infrastructure

[terraform-multicloud-runner](https://github.com/freecloudinitiative/terraform-multicloud-runner) provisions organization runners separately from application infrastructure. GCP, Azure, and Civo roots include VM bootstrap scripts that register multiple GitHub Actions runners per VM and support simple/HA counts. AWS and Linode currently contain provider scaffolding, not complete runner deployments.

This repository exists because many FCI builds target ARM64 and some workflows need access to project-local infrastructure. Runner capacity can change without touching the K3s cluster.

> [!CAUTION]
> **Runner registration credentials**
>
> GitHub registration tokens and PATs are sensitive Terraform inputs. Supply them through the workflow secret store or environment variables and keep state access tightly controlled.

## Validation and workflow ownership

Repository workflows call reusable jobs maintained in [.github](https://github.com/freecloudinitiative/.github). The shared Terraform checks format and validate every selected root; apply and destroy remain explicit workflows with provider credentials supplied as secrets.

Before a pull request:

~~~bash
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
~~~

Run <code>init</code> and <code>validate</code> in each root you changed.

## Change workflow and recovery

1. Select exactly one provider root and confirm the workspace/state backend.
2. Format and validate without credentials where possible.
3. Plan with the intended account and variable set; review replacements and firewall exposure.
4. Apply the saved plan through the authorized workflow.
5. Export the resulting node addresses into the correct Ansible inventory.
6. Verify reachability before bootstrap; Terraform success does not prove SSH or K3s join paths are usable.

If apply fails partway, do not delete state or immediately rerun with changed configuration. Refresh, inspect state and the cloud console, then produce a new plan. Import existing objects or move state addresses only after documenting the ownership correction.

## Practice

1. Run `terraform providers` in two roots and compare their dependency graphs.
2. Identify every value that crosses from Terraform output into Ansible inventory.
3. Change a harmless tag in a test root and explain the plan symbols.
4. Describe the recovery path for a resource created remotely but missing from state.

[Continue to cluster bootstrap →](ansible-k3s.md){ .md-button .md-button--primary }
