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
   - a Kata Containers role exists for <code>high_memory</code> workers with <code>/dev/kvm</code>, but its production play is currently commented out and must be enabled deliberately;
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

### Variable precedence and host identity

Inventory groups describe topology; `group_vars/all` supplies shared cluster settings; encrypted variables and environment overrides supply sensitive bootstrap values. Keep a node's Ansible host, advertised K3s address, and join address distinct. A public SSH endpoint can reach a node while its private address remains the correct address for server/agent traffic.

Before a full run, verify inventory topology, host connectivity, playbook syntax, targeted hosts, and available tags:

~~~bash
ansible-inventory -i inventory.ini --graph
ansible all -i inventory.ini -m ping
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml --list-hosts --list-tags
~~~

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

### Idempotence and failure recovery

A second playbook run should converge rather than reproduce one-time side effects. Roles should guard initialization steps, use module state instead of unconditional shell commands, and register changes only when the remote state changed. OpenBao initialization is especially sensitive: losing the local bootstrap artifacts and initializing again is not a recovery procedure.

When a run fails, resume only after identifying the boundary:

1. node preparation and K3s installation;
2. server token discovery and node join;
3. node labels/taints and optional host features;
4. OpenBao initialization and secret seeding;
5. Argo CD installation and root application handoff;
6. local kubeconfig/k9s setup.

Use role tags when the repository exposes them, and prefer rerunning the idempotent play over manually completing half a role. After the root Application exists, fix steady-state resources in Git rather than adding Ansible tasks that compete with Argo CD.

## Practice

1. Render the inventory graph and identify the first server and each scheduling tier.
2. Trace the first server's node token into the worker join play.
3. Find the guard that prevents OpenBao reinitialization.
4. Re-run a test bootstrap and explain every task that still reports `changed`.

## Reset semantics

<code>reset-k3s.yml</code> removes K3s agents first and then server/cluster data. It does not reinstall the cluster and requires explicit confirmation:

~~~bash
ansible-playbook reset-k3s.yml \
  -e confirm_k3s_reset=true \
  --ask-vault-pass
~~~

Run the appropriate bootstrap playbook separately afterward.

[Continue to GitOps environments →](gitops-manifests.md){ .md-button .md-button--primary }
