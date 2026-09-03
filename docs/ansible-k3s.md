# Cluster bootstrap with Ansible

<span class="step-badge">02 / BOOTSTRAP</span>

The [ansible-automation](https://github.com/freecloudinitiative/ansible-automation) repository turns reachable Linux nodes into an FCI K3s cluster and then hands steady-state ownership to Argo CD.

## The bootstrap boundary

~~~mermaid
flowchart TD
    INV["validated inventory"] --> M["install K3s servers"]
    M --> W["join K3s workers"]
    W --> NODE["labels · taints · optional Kata/Tailscale"]
    NODE --> B["install, initialize, unseal, seed OpenBao"]
    B --> A["install Argo CD"]
    A --> ROOT["apply selected GitOps root app"]
    ROOT --> G["Argo CD owns steady state"]
~~~

Ansible owns the steps that must exist before GitOps can function. It does not own normal application rollout, ingress, observability, or operator lifecycle after the handoff.

## Production and non-production entry points

~~~text
ansible-automation/
├── playbook.yml                 # production cluster
├── nonprod-playbook.yml         # non-production cluster
├── inventory.ini               # production nodes
├── nonprod-inventory.ini       # non-production nodes
├── prod-ssh-config.yml
├── nonprod-ssh-config.yml
├── reset-k3s.yml
├── thermal-check.yml
├── group_vars/all/
└── roles/
~~~

The production playbook supports the bare-metal Raspberry Pi topology. The non-production playbook targets the separate cloud inventory and must override the GitOps repository and OpenBao values source to their non-production equivalents.

## Play order

1. Validate inventory and the first master's reachable join address.
2. Prepare server nodes and initialize the first K3s server with <code>--cluster-init</code>.
3. Join remaining servers and workers with the discovered node token.
4. Apply node-tier labels and control-plane taints.
5. Install optional host features:
   - Kata Containers on <code>high_memory</code> workers with <code>/dev/kvm</code>;
   - Tailscale when an auth key is supplied;
   - Raspberry Pi boot configuration and operational tooling where applicable.
6. Install OpenBao on the first master, initialize/unseal it, configure Kubernetes auth, and seed platform secrets.
7. Install Argo CD and apply the root application for the selected GitOps repository.
8. Seed the few CA-dependent OpenBao values after cert-manager has reconciled.
9. Fetch kubeconfig and configure local k9s.

> [!NOTE]
> **Why OpenBao is bootstrap-owned**
>
> External Secrets cannot materialize application secrets until its OpenBao <code>ClusterSecretStore</code> can authenticate. Preparing OpenBao before the first Argo CD sync removes that startup race. GitOps stores the reviewed Helm values, while Ansible performs the install and initialization.

## K3s ownership choices

K3s starts with its packaged Traefik and ServiceLB disabled. The environment GitOps repository installs the selected ingress/load-balancer topology instead:

- production uses Traefik plus MetalLB and Cloudflare Tunnel;
- non-production uses Traefik host ports, with no MetalLB or Cloudflare.

This keeps environment differences in declarative desired state rather than hidden in node installation flags.

## Inventory contract

The first server address is required before any node is modified:

~~~yaml
k3s_master1_public_ip: "203.0.113.10"
~~~

Despite the historical variable name, non-production workers must join over the server's private VPC address. Public addresses are for operator SSH only.

At least one non-production worker must be in the <code>high_memory</code> group because Authentik and Argo CD select that node tier. Other workers should be grouped according to the memory tiers expected by the GitOps scheduling rules.

## Secrets

<code>group_vars/all/secret.yml</code> is an Ansible Vault-encrypted file. Bootstrap values can also come from environment variables. The playbook validates lengths and PEM formats before writing to OpenBao, including distinct Ed25519 keypairs for:

- api-gateway;
- terminal-gateway;
- compute-service;
- database-service;
- storage-service.

These keys establish separate internal service identities. Reusing a key creates a key-ID collision and causes IAM startup validation to fail.

Activate the repository's staged-content guard after cloning:

~~~bash
git config core.hooksPath hooks
~~~

> [!CAUTION]
> **Never commit plaintext bootstrap material**
>
> Keep <code>secret.yml</code> encrypted, revoke the OpenBao bootstrap token after seeding, and never reuse production OpenBao data or credentials in non-production.

## Run and verify

~~~bash
ansible-playbook playbook.yml --ask-vault-pass
kubectl get nodes -o wide
kubectl -n argocd get applications.argoproj.io
kubectl get pods -A
~~~

For non-production, use <code>nonprod-playbook.yml</code> with the non-production inventory and repository overrides. After Argo CD installs Garage, run the environment GitOps repository's idempotent <code>scripts/garage-bootstrap.sh</code> before expecting storage-service readiness.

## Reset semantics

<code>reset-k3s.yml</code> removes K3s agents first and then server/cluster data. It does not reinstall the cluster and requires explicit confirmation:

~~~bash
ansible-playbook reset-k3s.yml \
  -e confirm_k3s_reset=true \
  --ask-vault-pass
~~~

Run the appropriate bootstrap playbook separately afterward.

[Continue to GitOps environments →](gitops-manifests.md){ .md-button .md-button--primary }
