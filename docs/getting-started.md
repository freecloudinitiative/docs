# Getting Started

This page explains how to run and contribute to this documentation site locally.

!!! tip "Prerequisites"
    - Python **3.8+** installed
    - `pip` package manager
    - (Optional) A Python virtual environment

---

## Local Setup

### 1. Install dependencies

=== "With virtualenv (recommended)"

    ```bash
    python3 -m venv .venv
    source .venv/bin/activate

    pip install -r requirements.txt
    ```

=== "Globally"

    ```bash
    pip install -r requirements.txt
    ```

### 2. Start the live preview server

```bash
mkdocs serve
```

MkDocs will start a local server with **live reload** — any changes you save to `.md` files are instantly reflected in the browser.

Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser.

### 3. Build the static site

To compile all Markdown into a static `site/` folder:

```bash
mkdocs build
```

The output in `site/` is what gets deployed to GitHub Pages.

---

## Editing Documentation

All documentation source files live in the `docs/` subdirectory:

```text
docs/
├── index.md            # Homepage hero
├── learning-path.md    # Step-by-step DevOps learning journey
├── terraform.md        # Step 1: Infrastructure with Terraform
├── ansible-k3s.md      # Step 2: Cluster setup with Ansible
├── gitops-manifests.md # Step 3: GitOps with ArgoCD
├── architecture.md     # System architecture & observability
├── getting-started.md  # This page
└── assets/
    ├── logo.png        # Site logo (shown in navbar)
    ├── favicon.png     # Browser tab icon
    └── custom.css      # Brand color overrides
```

Navigation structure is defined in [`mkdocs.yml`](https://github.com/freecloudinitiative/docs/blob/main/mkdocs.yml).

---

## Contributing

1. Fork the repository
2. Create a branch: `git checkout -b docs/my-topic`
3. Make your edits
4. Preview locally with `mkdocs serve`
5. Open a Pull Request

!!! note "Mermaid diagrams"
    Diagrams are written in [Mermaid syntax](https://mermaid.js.org/) inside fenced code blocks:
    ````
    ```mermaid
    flowchart LR
        A --> B
    ```
    ````
    They render automatically in the built site.
