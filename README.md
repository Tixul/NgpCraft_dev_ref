# NGPC Dev Wiki

A developer reference for the **Neo Geo Pocket Color** (NGPC) and its Toshiba
TLCS-900/H CPU — hardware registers, graphics, audio, CPU/toolchain, and
reusable game-programming patterns.

The wiki is plain **Markdown** and is rendered to a static
HTML site with [MkDocs](https://www.mkdocs.org/) + the
[Material](https://squidfunk.github.io/mkdocs-material/) theme.

## Layout

```
mkdocs.yml                 site config + navigation
requirements.txt           build dependency (mkdocs-material)
docs/                      all wiki content (edit here)
  index.md                 home page
  01_Hardware/ … 06_…/     one folder per section
site/                      generated HTML (do NOT edit; git-ignored)
.github/workflows/         CI that builds + deploys to GitHub Pages
```

## Edit

Just edit the `.md` files under `docs/`. To add a page, create the `.md` file and
add a line to the `nav:` block in `mkdocs.yml`. Links between pages are normal
relative Markdown links (`[Build Toolchain](../02_CPU-and-Toolchain/Build-Toolchain.md)`);
MkDocs rewrites them to HTML automatically.

## Preview locally

```bash
pip install -r requirements.txt
mkdocs serve            # live-reloading preview at http://127.0.0.1:8000
```

Build the static site without serving:

```bash
mkdocs build --strict   # output in site/ ; --strict fails on any broken link
```

## Publish (GitHub Pages)

1. `git init` this folder and push it to a GitHub repository.
2. In the repo: **Settings -> Pages -> Build and deployment -> Source: GitHub Actions**.
3. Push to `main`. The workflow in `.github/workflows/deploy.yml` builds with
   `--strict` and deploys automatically. The published URL appears in the Actions run.
4. (Optional) set `site_url:` in `mkdocs.yml` to the published address.

Manual one-shot alternative (pushes the built site to a `gh-pages` branch):

```bash
mkdocs gh-deploy --force
```
