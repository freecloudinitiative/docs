# Repository catalog

<span class="page-lede">The organization contains 16 active repositories. Each repository owns one lifecycle, product domain, or shared concern.</span>

## At a glance

| Repository | Visibility | Owns |
| --- | --- | --- |
| [`.github`](https://github.com/freecloudinitiative/.github) | Public | Organization profile and reusable GitHub Actions workflows |
| [`docs`](https://github.com/freecloudinitiative/docs) | Public | This MkDocs architecture and learning site |
| [`frontend`](https://github.com/freecloudinitiative/frontend) | Public | React cloud console |
| [`api-gateway`](https://github.com/freecloudinitiative/api-gateway) | Private | External API boundary, authentication, routing, resilience |
| [`iam-service`](https://github.com/freecloudinitiative/iam-service) | Private | Accounts, users, roles, API keys, quotas, audit |
| [`compute-service`](https://github.com/freecloudinitiative/compute-service) | Private | Compute engines, namespaces, lifecycle reconciliation, disk backups |
| [`database-service`](https://github.com/freecloudinitiative/database-service) | Private | Managed PostgreSQL, SQL execution, imports, CNPG reconciliation |
| [`storage-service`](https://github.com/freecloudinitiative/storage-service) | Private | Object storage, logical networks, firewall projection |
| [`terminal-gateway`](https://github.com/freecloudinitiative/terminal-gateway) | Private | Browser WebSocket to Kubernetes exec bridge |
| [`platform-common`](https://github.com/freecloudinitiative/platform-common) | Private | Shared Go libraries and wire-level conventions |
| [`ansible-automation`](https://github.com/freecloudinitiative/ansible-automation) | Public | Node configuration and K3s/OpenBao/Argo CD bootstrap |
| [`k3s-manifests`](https://github.com/freecloudinitiative/k3s-manifests) | Public | Production GitOps desired state |
| [`nonprod-k3s-manifests`](https://github.com/freecloudinitiative/nonprod-k3s-manifests) | Public | AWS non-production GitOps desired state |
| [`terraform-multicloud-infra`](https://github.com/freecloudinitiative/terraform-multicloud-infra) | Public | Cluster nodes and networks across five providers |
| [`terraform-cloudflare-infra`](https://github.com/freecloudinitiative/terraform-cloudflare-infra) | Private | Production DNS and Cloudflare Tunnel |
| [`terraform-multicloud-runner`](https://github.com/freecloudinitiative/terraform-multicloud-runner) | Public | Self-hosted GitHub Actions runner infrastructure |

!!! note "Private implementation repositories"
    Private repositories appear here because they are part of the deployed architecture. Their links require organization access. Public users can still understand their contracts and runtime roles from this site.

## Product experience

### `frontend`

The React 19 and TypeScript 6 single-page application is the platform console. It uses Vite, TanStack Query, Zustand, Monaco, and xterm.js. Product modules cover compute engines, managed databases, buckets, networks, IAM, account settings, metrics, and interactive terminals.

The frontend has two runtime modes:

- `nonprod` uses Mock Service Worker for local UI development without a backend.
- `prod` performs Authentik OIDC and sends real API/WebSocket traffic through api-gateway.

**Depends on:** Authentik and api-gateway.  
**Deployed by:** the `frontend` chart in each GitOps repository.

### `docs`

MkDocs Material site deployed to GitHub Pages. It owns cross-repository architecture, environment boundaries, contributor entry points, and the build path. Detailed API and file-level documentation remains in the repository that owns the code.

## Control-plane services

### `api-gateway`

The only external API boundary. It validates Authentik OIDC tokens or FCI API keys, resolves accounts, rate-limits, handles idempotency, applies timeouts/circuit breakers, mints audience-bound internal Ed25519 JWTs, and reverse-proxies requests to five upstream services. It also mints terminal tickets and proxies terminal WebSockets without forwarding browser credentials.

**State:** Valkey only; no Postgres schema.  
**Calls:** IAM, compute, database, storage, terminal.

### `iam-service`

The authority for platform accounts and access. First login transactionally creates an account, owner, default quotas, and admin policy. It also manages API keys, users, managed roles, settings, activity, terminal audit records, and Authentik drift reconciliation.

**State:** `iam` Postgres schema; Authentik integration.  
**Called by:** gateway and the domain services.

### `compute-service`

Stores compute-engine intent and reconciles it into per-account Kubernetes namespaces, Deployments, Services, PVCs, ResourceQuotas, and LimitRanges. It runs a namespace reaper and an optional nightly backup scheduler. Backups are uploaded through a platform-reserved storage bucket; restore exists as an operator-only workflow and is not exposed through the customer API.

**State:** `compute` Postgres schema and Kubernetes.  
**Calls:** IAM for quotas and storage-service for backup-bucket resolution.

### `database-service`

Turns database API state into CloudNativePG `Cluster` resources. It exposes CRUD, SQL execution, CSV/JSON import, metrics, and connection information. CNPG generates customer credentials, which are read live from Kubernetes Secrets rather than stored in platform Postgres.

**State:** `database` Postgres schema, Kubernetes, and customer CNPG clusters.  
**Calls:** IAM for quotas and compute-service for namespace provisioning when configured.

### `storage-service`

Combines two domains: S3-compatible buckets/objects and virtual networks/firewall rules. Objects live in one physical Garage bucket under server-derived account/bucket prefixes. A usage collector snapshots object counts and bytes. A reconciler projects representable allow rules into Kubernetes NetworkPolicies.

Stored access-policy records are not enforced by the S3 backend in v1 because the platform currently uses one shared storage credential.

**State:** `storage` Postgres schema, Garage, and Kubernetes.  
**Calls:** IAM for limits and principal validation; accepts compute-service backup-bucket requests.

### `terminal-gateway`

A deliberately narrow WebSocket service. It atomically redeems a gateway-minted ticket from Valkey, asks compute-service for the trusted pod target, acquires a per-account session slot, upgrades the connection, and relays browser frames to Kubernetes `pods/exec`. Session open/close events go to IAM.

**State:** Valkey only; no Postgres schema.  
**Calls:** compute-service, Kubernetes API, and iam-service.

### `platform-common`

A versioned Go module, not a process. Backend services import its packages for:

- Ed25519 internal JWTs and Authentik OIDC validation
- Valkey clients, cache keys, rate limiting, idempotency, and session slots
- environment configuration and validation
- uniform JSON errors, health checks, middleware, and graceful servers
- structured logging and OpenTelemetry
- TLS Postgres pools, schema pinning, and migrations
- shared Postgres/Valkey/auth test harnesses

Consumers pin released module versions. The workspace `go.work` supports cross-repository development, while isolated builds verify the real pinned dependency.

## Infrastructure and delivery

### `terraform-multicloud-infra`

Independent Terraform roots for AWS, Azure, GCP, Civo, and Linode. They provision cluster machines, networks, firewall/security rules, and provider-specific connectivity. AWS includes a secondary VPC and peering for split node placement.

**Hands off:** node addresses and topology to Ansible inventories.

### `terraform-cloudflare-infra`

Production edge infrastructure only. It creates DNS records and one outbound Cloudflare Zero Trust tunnel configuration. The in-cluster `cloudflared` workload is owned by `k3s-manifests`; Traefik performs origin routing. Internal-only hostnames receive LAN records and are excluded from tunnel ingress.

**Hands off:** the tunnel credential to OpenBao/External Secrets and DNS traffic to Traefik.

### `terraform-multicloud-runner`

Terraform for organization self-hosted GitHub Actions runners. Implemented roots provision runner VMs on GCP, Azure, and Civo with simple/HA sizing and multiple runners per VM. AWS and Linode currently contain provider scaffolding only.

**Supports:** reusable workflows from `.github` and repository CI workloads.

### `ansible-automation`

Configures production and non-production nodes, installs K3s, joins workers, labels/taints tiers, optionally installs Kata Containers and Tailscale, and fetches kubeconfig. On the first master it installs and seeds OpenBao, installs Argo CD, and applies the selected GitOps root application.

Ansible stops at the bootstrap boundary. Steady-state ingress, certificates, operators, observability, data services, and FCI applications belong to Argo CD.

### `k3s-manifests`

Production source of truth for Argo CD. Infrastructure includes namespaces, CoreDNS customization, MetalLB, cert-manager, Longhorn, Traefik, External Secrets, CloudNativePG, Kyverno, platform PostgreSQL, Valkey, Garage, Authentik, cloudflared, Argo CD self-management, and the observability stack. Applications include the frontend and all six control-plane services.

OpenBao values live here for the Ansible install, but OpenBao itself is not an Argo CD application.

### `nonprod-k3s-manifests`

Isolated desired state for a five-node AWS test cluster. It mirrors the real platform while replacing production-specific edges: no Cloudflare, no MetalLB, reserved `.test` hostnames, a private CA, and host-port Traefik on the control-plane node.

### `.github`

Organization profile plus centrally maintained reusable workflows for Go checks, frontend checks, Terraform plan/apply/destroy, Cloudflare, Ansible validation, image publication/cleanup, security scanning, and release-related automation. Individual repositories call these workflows instead of duplicating CI logic.

## Change routing

| If you need to change… | Start in… |
| --- | --- |
| Browser behavior or visual design | `frontend` |
| Public HTTP auth, cross-cutting request policy, or route dispatch | `api-gateway` |
| Accounts, users, roles, keys, quotas, or audit | `iam-service` |
| Compute lifecycle, namespaces, engine disks, or compute backups | `compute-service` |
| Managed PostgreSQL or SQL/import behavior | `database-service` |
| Buckets, objects, logical networks, or firewall projection | `storage-service` |
| Browser terminal transport or exec isolation | `terminal-gateway` |
| Shared backend convention | `platform-common`, then each consumer's pinned version |
| Production workload configuration or image promotion | `k3s-manifests` |
| Non-production-only desired state | `nonprod-k3s-manifests` |
| Node bootstrap or pre-GitOps secret initialization | `ansible-automation` |
| Cloud nodes/networking | `terraform-multicloud-infra` |
| Production DNS/tunnel ingress | `terraform-cloudflare-infra` |
| Self-hosted CI runner capacity | `terraform-multicloud-runner` |
| Shared CI implementation | `.github` |
| Cross-repository explanation | `docs` |

[See how requests cross the services →](platform-services.md){ .md-button .md-button--primary }
