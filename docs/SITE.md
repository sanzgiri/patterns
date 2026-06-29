# Patterns site (MkDocs Material)

The repo now ships a MkDocs Material site that renders both reference corpora as a single navigable, searchable web app.

## Quick start

```bash
# one-time setup
python3 -m venv .venv
.venv/bin/pip install -r requirements-docs.txt

# run the dev server (live-reloads on file changes)
.venv/bin/mkdocs serve
# → open http://127.0.0.1:8000/patterns/

# or build the static site to ./site/
.venv/bin/mkdocs build
```

## How it's wired

```
patterns/
├── reference/              # 86 system-design markdown pages + diagrams
├── reference-llm/          #  8+ LLM/agentic markdown pages + diagrams
├── site_src/               # MkDocs docs_dir (small — mostly symlinks)
│   ├── index.md            # site landing page
│   ├── system-design/      # symlink → ../reference
│   ├── llm/                # symlink → ../reference-llm
│   ├── assets/extra.css    # site-wide tweaks
│   └── .pages              # nav ordering (awesome-pages plugin)
├── mkdocs.yml              # MkDocs config
├── requirements-docs.txt   # build deps
└── .github/workflows/
    └── deploy-docs.yml     # auto-deploy to GitHub Pages on push to main
```

Two symlinks let MkDocs see the existing `reference/` and `reference-llm/` trees as `system-design/` and `llm/` in the URL space, so all the internal relative links (`[compaction.md](compaction.md)`, `../diagrams/svg/X.svg`) keep working unchanged.

## Plugins / extensions

- **`mkdocs-material`** — theme
- **`mkdocs-callouts`** — converts GitHub-flavored `> [!TIP]` admonitions to Material's admonition boxes (so we don't have to rewrite ~94 markdown files)
- **`mkdocs-awesome-pages-plugin`** — folder-structure-driven nav, honoring `.pages` files
- **`pymdownx.*`** — code highlighting, tabs, task lists, details, etc.

## Deploying to GitHub Pages

The workflow at `.github/workflows/deploy-docs.yml` builds and deploys automatically on every push to `main` that touches the docs. One-time setup:

1. In the GitHub repo: **Settings → Pages → Source → GitHub Actions**
2. Push to `main`. The workflow runs and the site is published at `https://sanzgiri.github.io/patterns/`

(Until that's enabled, the site only exists locally via `mkdocs serve`.)
