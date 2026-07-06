# OutcomeOps AI Assist Docs

This repository is the source for **[docs.outcomeops.ai](https://docs.outcomeops.ai)**.

**To read the documentation, go to [docs.outcomeops.ai](https://docs.outcomeops.ai) --- this repo is for editing them.**

## About OutcomeOps

[OutcomeOps.ai](https://outcomeops.ai) is a self-hosted, regulated-grade AI code assistant that runs entirely in your own AWS account.

## Reporting a doc issue

Typos, broken links, unclear explanations, missing content --- please [open an issue](https://github.com/outcomeops/docs/issues/new). The more specific, the better (page URL + what's confusing).

## Contributing

**Pull requests are not accepted at this time.** OutcomeOps maintains editorial control over the documentation to keep voice, framing, and technical accuracy consistent.

If something is wrong or missing, [open an issue](https://github.com/outcomeops/docs/issues/new) and someone from the team will edit the source and ship the fix. Issues are triaged weekly.

If you feel strongly about a change, include suggested wording in the issue --- we'll happily lift good language when it fits.

## Building locally (maintainers)

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

Opens at `http://0.0.0.0:8123` (bound to all interfaces so LAN + SSH-tunneled clients can reach it). Edits to any `docs/*.md` file live-reload.

## Publishing

Every push to `main` triggers `.github/workflows/deploy.yml`:

1. Installs pinned `mkdocs` + `mkdocs-material` from `requirements-docs.txt`
2. Runs `mkdocs gh-deploy --force --clean --verbose`
3. Publishes the built site to the `gh-pages` branch

GitHub Pages serves the `gh-pages` branch at `docs.outcomeops.ai` (custom domain lives at `docs/CNAME`, copied into every build).

## Structure

- `docs/` --- markdown content, one page per `*.md` file
- `mkdocs.yml` --- theme, nav, plugins
- `docs/CNAME` --- custom domain, copied into build output so GitHub Pages picks it up
- `docs/stylesheets/brand.css` --- OutcomeOps color palette overrides on top of the Material default
- `docs/assets/` --- logo + shared image assets

## License

Documentation content is licensed under **[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)** ([full text](./LICENSE)). You are free to share and adapt the content for any purpose, provided you give appropriate credit ("OutcomeOps.ai documentation, licensed under CC BY 4.0") and link back to the source.

Repository configuration files (`mkdocs.yml`, `.github/workflows/`, `requirements-docs.txt`, `.gitignore`) are provided under the same terms.
