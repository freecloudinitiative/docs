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

### Sync-wave implementation

Infrastructure Applications use `argocd.argoproj.io/sync-wave` to express dependency order. For example, cert-manager, External Secrets, and Kyverno are wave 1; CloudNativePG is wave 2; Garage is wave 4 after secret configuration; Authentik is wave 5; Alloy is wave 6; kube-prometheus-stack is wave 9; and product applications follow at wave 11. Health-aware annotations and retry policies handle the difference between an object being accepted and its dependency becoming usable.

Waves order one sync operation; they are not a substitute for readiness. Each later component must still retry safe startup dependencies, expose useful health, and tolerate a controller restart.

### Multi-source Applications

Infrastructure Applications commonly combine an upstream Helm chart with values and supplemental resources from the FCI environment repository. The first source supplies versioned upstream templates; the `$values` source keeps local policy reviewable beside the rest of the environment.

The example uses the reserved `.invalid` domain as a placeholder. Replace `https://charts.example.invalid` with the upstream chart repository URL before using the manifest.

~~~yaml
spec:
  sources:
    - repoURL: https://charts.example.invalid
      chart: operator
      targetRevision: 1.2.3
      helm:
        valueFiles:
          - $values/infrastructure/operator/values.yaml
    - repoURL: https://github.com/freecloudinitiative/k3s-manifests.git
      targetRevision: HEAD
      ref: values
~~~

Pin upstream chart versions or source commits. `HEAD` is appropriate for the environment repository because the Application is intentionally following its own reviewed branch; it is not appropriate for an untrusted upstream dependency.

## Application promotion

FCI service charts require an explicit image tag or digest and do not silently fall back to an app version that may not exist. `image.digest` pins exact content and is required for immutable promotion; `image.tag` remains supported but is a mutable registry pointer. Environment Applications that still reference release tags are traceable, not content-pinned, and should move to the published digest during promotion.

~~~yaml
spec:
  source:
    helm:
      parameters:
        - name: image.digest
          value: sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
~~~

Application images live in <code>ghcr.io/freecloudinitiative</code>, and External Secrets materializes pull credentials into the backend and frontend namespaces.

## Adding or changing an application

1. Change the service source repository, publish the ARM64 image, and capture its `sha256:` digest.
2. Update `image.digest` in the intended environment repository.
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

Inspect the rendered object when validation fails instead of editing the cluster:

~~~bash
helm lint applications/api-gateway/chart
helm template api-gateway applications/api-gateway/chart \
  --namespace backend \
  --set image.digest=sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
kubectl -n argocd get application api-gateway -o yaml
kubectl -n argocd get application api-gateway \
  -o jsonpath='{.status.operationState.message}{"\n"}'
~~~

## Drift and rollback

Argo CD reports desired-versus-live differences. An Application prunes resources removed from Git only when `spec.syncPolicy.automated.prune` is enabled, and self-heals managed fields only when `spec.syncPolicy.automated.selfHeal` is enabled; the FCI environment Applications set both options. A rollback is therefore a Git operation: revert or promote the last known-good image/configuration, validate it, and let Argo CD converge. Before pruning a stateful resource, confirm whether its finalizer, PVC retention, or operator-specific deletion policy preserves data.

## Practice

1. Draw the sync dependency path for one product service.
2. Render its chart with the promoted image and inspect RBAC, probes, and resource limits.
3. Find one `ignoreDifferences` rule and explain which controller mutates the ignored field.
4. Describe a Git rollback that does not accidentally delete persistent data.

> [!WARNING]
> **No manual drift as a deployment strategy**
>
> A manual <code>kubectl</code> edit may be reverted by self-heal and is absent from the audit trail. Commit the desired state to the owning environment repository.

[Trace the deployed control plane →](platform-services.md){ .md-button .md-button--primary }
