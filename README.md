# OutcomeOps AI Assist Docs

Source for [docs.outcomeops.ai](https://docs.outcomeops.ai). Public. Contributions welcome.

## Local preview

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

Opens at http://127.0.0.1:8000. Edits to any `docs/*.md` file live-reload.

## Publishing

Every push to `main` triggers `.github/workflows/deploy.yml`, which runs `mkdocs gh-deploy` and pushes the built site to the `gh-pages` branch. GitHub Pages serves that branch at `docs.outcomeops.ai` (CNAME lives at `docs/CNAME`).

First-time setup (already done): in GitHub → repo Settings → Pages → set source to "Deploy from a branch" → branch `gh-pages` / `/` (root).

## Structure

- `docs/` — markdown content, one page per `*.md` file
- `mkdocs.yml` — theme + nav config
- `docs/CNAME` — custom domain, copied to the built site so GitHub Pages picks it up

## Adding a page

1. Create a new `docs/some-section/new-page.md`
2. Reference it in the `nav:` block of `mkdocs.yml`
3. Commit + push. CI builds + deploys.
