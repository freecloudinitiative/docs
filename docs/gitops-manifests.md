# GitOps environments

<span class="step-badge">03 / RECONCILE</span>

FCI has one Argo CD source-of-truth repository per environment:

- [k3s-manifests](https://github.com/freecloudinitiative/k3s-manifests) for production on bare metal;
- [nonprod-k3s-manifests](https://github.com/freecloudinitiative/nonprod-k3s-manifests) for the isolated AWS test cluster.

Both repositories deploy the real control plane. Their differences encode environment contracts, not separate product implementations.

## Argo CD handoff

~~~mermaid
sequenceDiagram
    participant A as ansible-automation
    participant O as OpenBao
    participant C as Argo CD
    participant G as environment GitOps repo
    participant K as K3s

    A->>O: install, initialize, unseal, seed
    A->>C: install Argo CD
    A->>K: apply root Application
    C->>G: discover infrastructure/* and applications/*
    C->>K: sync in wave order
    loop steady state
        C->>G: detect commits
        C->>K: apply, prune, self-heal
    end
~~~

The root application is the one imperative apply. After that handoff, changes belong in Git. Argo CD uses automated sync, pruning, and self-healing to keep the cluster aligned.

## Shared layout

~~~text
<environment>-k3s-manifests/
├── infrastructure/
│   ├── namespaces/
│   ├── coredns/
│   ├── cert-manager/
│   ├── longhorn/
│   ├── traefik/
│   ├── external-secrets/
│   ├── cloudnative-pg/
│   ├── kyverno/
│   ├── kyverno-policies/
│   ├── platform-postgresql/
│   ├── valkey/
│   ├── garage/
│   ├── authentik/
│   ├── argocd/
│   ├── kube-prometheus-stack/
│   ├── loki/
│   ├── tempo/
│   ├── opentelemetry/
│   └── alloy/
└── applications/
    ├── frontend/
    ├── api-gateway/
    ├── iam-service/
    ├── compute-service/
    ├── database-service/
    ├── storage-service/
    └── terminal-gateway/
~~~

Each application directory contains its Argo CD <code>app.yaml</code> and its colocated Helm chart. There is no separate charts tree.

Production additionally owns <code>metallb</code> and <code>cloudflared</code>. OpenBao has reviewed values under <code>infrastructure/openbao</code>, but no Argo CD Application because its lifecycle is bootstrap-owned.

## Environment differences

| Concern | Production | Non-production |
| --- | --- | --- |
| Nodes | Bare-metal Raspberry Pi fleet | AWS: one server and four workers |
| Git source | <code>k3s-manifests</code> | <code>nonprod-k3s-manifests</code> |
| Public edge | Cloudflare Tunnel | None |
| Load balancer | MetalLB L2 | None |
| Traefik | Cluster ingress behind MetalLB/tunnel | Host ports 80/443 on control-plane |
| Names | Public FCI domain | Reserved <code>.test</code> names via hosts file/CoreDNS |
| Browser TLS | Public/private issuers by endpoint | Environment-local private CA |
| Frontend mode | Production backend | Production backend (not MSW mocks) |
| Secrets | Production OpenBao | Separate non-production OpenBao |

The non-production repository's <code>make environment-check</code> rejects production Git URLs, Cloudflare/MetalLB resources, ACME issuers, and Kubernetes LoadBalancer Services.

## Infrastructure dependency chain

~~~mermaid
flowchart LR
    NS["namespaces"] --> CERT["cert-manager"]
    CERT --> ESO["External Secrets"]
    O["OpenBao\nAnsible-owned"] --> ESO
    ESO --> DATA["Postgres · Valkey · Garage · Authentik"]
    DATA --> API["FCI backend services"]
    API --> UI["frontend"]
    OBS["Prometheus · Loki · Tempo\nOTel · Alloy"] --> API
~~~

Argo CD sync waves and readiness protect this ordering, but two external bootstrap contracts remain:

1. OpenBao must be initialized and seeded before External Secrets can succeed.
2. Garage's layout, physical <code>platform</code> bucket, and service key must be created with <code>scripts/garage-bootstrap.sh</code> after its pods are ready.

Until Garage bootstrap completes, storage-service intentionally remains NotReady because its object-store readiness check performs <code>HeadBucket</code>.

## Application promotion

FCI service charts require an explicit image tag or digest. Argo CD Application parameters pin the promoted version; charts do not silently fall back to an app version that may not exist.

~~~yaml
spec:
  source:
    helm:
      parameters:
        - name: image.tag
          value: sha-abc123def456
~~~

Application images live in <code>ghcr.io/freecloudinitiative</code>, and External Secrets materializes pull credentials into the backend and frontend namespaces.

## Adding or changing an application

1. Change the service source repository and publish an immutable ARM64 image.
2. Update the image tag/digest in the intended environment repository.
3. Modify its colocated Helm chart if runtime configuration or Kubernetes resources changed.
4. Run the repository validation suite.
5. Merge; Argo CD detects the commit and converges the cluster.

Create a new service under <code>applications/&lt;service&gt;</code> with both a chart and an Argo CD Application manifest. If it needs a namespace, declare it under <code>infrastructure/namespaces</code> or use the operator's documented namespace creation contract.

## Validate locally

~~~bash
make validate
~~~

The validation target lints YAML, lints and renders every Helm chart, schema-checks rendered resources with kubeconform, and runs helm-unittest suites. In non-production, also run:

~~~bash
make environment-check
~~~

> [!WARNING]
> **No manual drift as a deployment strategy**
>
> A manual <code>kubectl</code> edit may be reverted by self-heal and is absent from the audit trail. Commit the desired state to the owning environment repository.

[Trace the deployed control plane →](platform-services.md){ .md-button .md-button--primary }
