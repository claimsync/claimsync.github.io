# claimsync.github.io

Central documentation for ClaimSync, published with
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) to
**https://claimsync.github.io/**.

## Local development

```bash
pip install -r requirements.txt
mkdocs serve   # http://127.0.0.1:8000
```

## How it deploys

Every push to `main` triggers `.github/workflows/deploy.yml`, which builds the
site and deploys it to GitHub Pages. No manual steps after the initial setup.

## Adding content

1. Add a Markdown file under `docs/`.
2. Reference it in the `nav:` section of `mkdocs.yml`.
3. Commit and push to `main`.
