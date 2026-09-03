# Contribute to the docs

<span class="page-lede">The docs site is a small MkDocs project. Cross-repository explanations live here; API, runbook, and file-level details should stay beside the code that owns them.</span>

## Local preview

### 1 / Create an environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2 / Serve with live reload

```bash
mkdocs serve
```

Open [http://127.0.0.1:8000](http://127.0.0.1:8000).

### 3 / Build exactly as CI does

```bash
mkdocs build --strict
```

The generated `site/` directory is deployed to GitHub Pages by `.github/workflows/deploy-docs.yml` after changes merge to `main`.

## Source layout

```text
docs/
├── index.md                # landing page
├── architecture.md         # complete system and runtime boundaries
├── platform-services.md    # control-plane request/resource flows
├── repositories.md         # all organization repositories and ownership
├── learning-path.md        # recommended exploration order
├── terraform.md            # infrastructure layer
├── ansible-k3s.md          # bootstrap layer
├── gitops-manifests.md     # production/non-production desired state
├── learning/               # technology and platform operations guides
├── getting-started.md      # this contributor guide
└── assets/
    ├── logo.png            # frontend logo used in header and browser tab
    └── custom.css          # frontend-aligned design tokens and components
```

Navigation, Markdown extensions, theme features, and site metadata live in `mkdocs.yml`.

## Documentation ownership

| Content | Correct home |
| --- | --- |
| Multi-repository architecture and ownership | This site |
| Public learning path and environment comparison | This site |
| Exact API request/response contract | Owning service's `API.md` |
| Package/file map and implementation internals | Owning repository's `FILES.md` and `ARCHITECTURE.md` |
| Operational command specific to one component | Owning repository README/runbook |
| Desired production/non-production Kubernetes state | Corresponding GitOps repository |

When changing a cross-repository flow, update the detailed source documentation first, then update this site so its summary remains traceable to the implementation.

## Style

- Use direct English and name the repository that owns a behavior.
- Distinguish current implementation from planned work.
- State environment differences explicitly.
- Prefer Mermaid for flows and small tables for ownership mappings.
- Use fenced code blocks with a language identifier.
- Use GitHub-style alerts (`> [!NOTE]`, `> [!WARNING]`, and related forms) for constraints and hazards.
- Keep the terminal visual vocabulary purposeful: bracketed labels, numbered stages, and exact frontend color tokens.

## Pull request checklist

- [ ] Every added page is present in `mkdocs.yml`.
- [ ] Repository names and links are exact.
- [ ] Architecture claims match current repository source.
- [ ] Production and non-production behavior are not conflated.
- [ ] `mkdocs build --strict` passes.
- [ ] Both palettes are checked at desktop and mobile widths; dark remains the default.

[Browse repository ownership →](repositories.md){ .md-button .md-button--primary }
