# Free Cloud Initiative documentation

Architecture and contributor documentation for the complete
[Free Cloud Initiative](https://github.com/freecloudinitiative) platform.

The site covers all 16 organization repositories, including infrastructure,
cluster bootstrap, production and non-production GitOps, the React console,
Go control-plane services, shared libraries, and CI ownership.

The Learning section uses those live implementations as a structured DevOps
curriculum covering Terraform, Ansible, Docker/OCI, Kubernetes/K3s, Argo CD,
networking, identity and secrets, data systems, observability, CI/CD, Go
reconciliation patterns, and the React operator console.

## Preview locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Run the same strict build used for validation:

```bash
mkdocs build --strict
```

Documentation source lives in [`docs/`](docs/), navigation and theme
configuration live in [`mkdocs.yml`](mkdocs.yml), and the site deploys to
GitHub Pages after changes merge to `main`.
