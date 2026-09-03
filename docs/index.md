<div class="hero" markdown>

<span class="terminal-kicker">FCI://DOCS</span>

# [ FREE CLOUD INITIATIVE ]

**A self-hosted cloud control plane, built in public.**<br>
Explore the infrastructure, GitOps environments, Go services, and terminal-style web console that turn a K3s cluster into a small cloud platform.

<div class="hero-buttons" markdown>
<a href="architecture/" class="btn-primary">[ EXPLORE ARCHITECTURE ]</a>
<a href="learning-path/" class="btn-outline">[ START LEARNING ]</a>
</div>

<div class="terminal-status" markdown>
<span><b>PLATFORM</b> self-hosted</span>
<span><b>CONTROL_PLANE</b> Go + React</span>
<span><b>ORCHESTRATOR</b> K3s + Argo CD</span>
</div>

</div>

## What the platform provides

<div class="feature-grid" markdown>

<div class="feature-card" markdown>
<span class="card-index">01 / COMPUTE</span>

### Compute engines

Container-backed Linux environments with persistent disks, lifecycle controls, metrics, terminal access, quotas, and scheduled crash-consistent backups.
</div>

<div class="feature-card" markdown>
<span class="card-index">02 / DATABASE</span>

### Managed PostgreSQL

CloudNativePG clusters reconciled from API state, with live credentials, SQL execution, data import, connection limits, metrics, and backup configuration.
</div>

<div class="feature-card" markdown>
<span class="card-index">03 / STORAGE</span>

### Buckets and networks

S3-compatible object storage on Garage plus account-scoped virtual networks and firewall rules projected into Kubernetes NetworkPolicies.
</div>

<div class="feature-card" markdown>
<span class="card-index">04 / ACCESS</span>

### Identity and terminals

Authentik OIDC, API keys, roles, quotas, audit records, and short-lived single-use tickets for browser-to-pod terminal sessions.
</div>

</div>

## One system, six concerns

```mermaid
flowchart LR
    I["01  INFRA\nTerraform"] --> B["02  BOOTSTRAP\nAnsible"]
    B --> G["03  DELIVERY\nArgo CD"]
    G --> P["04  PLATFORM\nK3s services"]
    P --> C["05  CONTROL PLANE\nGo APIs + React"]
    C --> O["06  OPERATIONS\nMetrics · logs · traces"]
```

The repositories are deliberately separated by lifecycle and trust boundary. Terraform creates machines and edge resources. Ansible turns nodes into a cluster and establishes the secret/GitOps bootstrap. Argo CD owns steady-state Kubernetes resources. The control plane exposes cloud-like products to users without giving them direct Kubernetes access.

<div class="callout-grid" markdown>

<div class="terminal-panel" markdown>
<span class="panel-label">[ START_HERE ]</span>

New to the project? Follow the [build path](learning-path.md) from infrastructure to a working platform.
</div>

<div class="terminal-panel" markdown>
<span class="panel-label">[ SYSTEM_MAP ]</span>

Need the runtime picture? Open the [architecture overview](architecture.md) and [control-plane guide](platform-services.md).
</div>

<div class="terminal-panel" markdown>
<span class="panel-label">[ SOURCE_INDEX ]</span>

Looking for ownership? The [repository catalog](repositories.md) covers every organization repository and its handoffs.
</div>

</div>

## Learn by operating the real stack

Education is a core project output, not a side effect of the source code. The [Learning section](learning-path.md) turns FCI's working repositories into a structured DevOps curriculum: provision with Terraform, bootstrap with Ansible, build OCI images, operate Kubernetes and Argo CD, then study networking, security, data, observability, CI/CD, Go reconcilers, and the React console.

Each guide explains the underlying technology, points to its concrete FCI implementation, calls out production tradeoffs, and ends with practical repository or cluster exercises.

> [!NOTE]
> **An evolving reference platform**
>
> Free Cloud Initiative applies production-oriented patterns on small, self-hosted infrastructure. Repository documentation is the source of truth for implementation details and operational constraints; this site explains how those pieces fit together.
