# OutcomeOps AI Assist Docs

This repository is the source for **[docs.outcomeops.ai](https://docs.outcomeops.ai)**.

**To read the documentation, go to [docs.outcomeops.ai](https://docs.outcomeops.ai) --- this repo is for editing them.**

## About OutcomeOps

[OutcomeOps.ai](https://outcomeops.ai) is a self-hosted, regulated-grade AI code assistant that runs entirely in your own AWS account. The platform is open source under Apache 2.0: [bcarpio/outcome-ops-ai-assist](https://github.com/bcarpio/outcome-ops-ai-assist).

Product bugs and feature requests go to the [product repo's issues](https://github.com/bcarpio/outcome-ops-ai-assist/issues); this repo is only for the documentation.

## Reporting a doc issue

Typos, broken links, unclear explanations, missing content --- please [open an issue](https://github.com/outcomeops/docs/issues/new). The more specific, the better (page URL + what's confusing).

## Contributing

Pull requests are welcome. Fix a typo, correct a wrong detail, or fill in a missing section: edit the Markdown under `docs/`, check it renders with `mkdocs serve` (see below), and open a PR. Every page on the site also has an edit link that takes you straight to its source file.

For larger changes (a new page, a restructured section), open an issue first so we can agree on the shape before you write it.

## Building locally

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
