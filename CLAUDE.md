# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal bilingual (en + zh) blog at `https://5o1.github.io/`, built with **Hugo 0.162 extended** + the **PaperMod** theme (vendored as a git submodule at `themes/PaperMod` — do not edit). CI builds and deploys via `.github/workflows/hugo.yaml` on every push to the `hugo` branch. The homepage (`content/_index*.md`) is a CV page, not a blog roll.

## Build & develop

| Task | Command |
| --- | --- |
| Local dev server (live reload, drafts) | `hugo server -D` |
| Production build (matches CI) | `hugo build --gc --minify --baseURL "https://5o1.github.io/" --cacheDir /tmp/hugo_cache` |
| Install Dart Sass locally | Required to compile `assets/css/extended/*.css`; CI installs v1.99.0 to `~/.local/dart-sass` |

Hugo extended + Dart Sass must both be on `$PATH`; `assets/css/extended/` uses SCSS-style features that depend on Sass.

## Playwright MCP

`.mcp.json` registers `@playwright/mcp` (npm `devDependency`, v0.0.75) so Claude can drive a Chromium browser via MCP tools: navigate to pages, click, screenshot, run assertions, watch network/console. Configured flags:

- `--headless --no-sandbox` — headless Chromium that works in containerized sandboxes.
- `--user-data-dir=/tmp/playwright-mcp-userdata` — **persistent** profile on disk (so `pubCoverCache` IndexedDB and cookies survive across sessions). Note: `--user-data-dir` is mutually exclusive with `--isolated`; pick one.
- `--output-dir=/tmp/playwright-mcp-output` — screenshots and other artifacts land here.

Browsers are preinstalled under `~/.cache/ms-playwright/` (chromium-1223, firefox-1522, webkit-2287). To point the MCP server at a live dev server while debugging, run `hugo server -D` separately, then have Claude navigate to `http://localhost:1313/...`.

## Layout architecture

`layouts/` shadows the PaperMod theme for the homepage, lists, singles, and most partials. The taxonomy in this repo:

- **Homepage (`index.html` / `index.zh.html`)** — both call `partials/cv_home.html`. The CV layout is geometry-based: a fixed-position sidebar (name + photo + contact links) that scroll-syncs with the main column, defined in `assets/css/extended/cv.css`. The sidebar pieces are collected by the `cvside` shortcode into `$.Page.Scratch`, then rendered by `partials/cv_profile.html`.
- **List (`list.html`)** — overrides PaperMod's list to add a **cards ↔ text view toggle** (the toolbar JS lives in PaperMod; this layout's CSS rules in `assets/css/extended/cover-side.css` are gated on `body[data-list-view="cards|details"]`). Supports a per-page **side-cover layout** configured via `params.cover.pages.<slug>`.
- **Single (`single.html`)** — adds cover image, breadcrumbs, TOC, edit-post link, and translation list. Per-page cover config flows through `partials/cover.html`.
- **Code fences (`_markup/render-codeblock.html`)** — global render hook. Wraps every fenced code block in a header bar (language label + copy-to-clipboard button). Language labels and copy button text come from per-language `params.code.aliases` and `params.code.copyButtonText` in `hugo.yaml` (defined once with a YAML anchor `&code_aliases` and reused in both `en` and `zh`).

## Custom shortcodes (`layouts/shortcodes/`)

- `publications` — `{{< publications bib="conference.bib" defaultCover="/images/paper-placeholder.svg" >}}`. Reads a `.bib` file from the **current page bundle** via `.Page.Resources.GetMatch`, parses `@…{…}` entries with regex (no Python/BibTeX toolchain), sorts by year descending, and renders one card per entry via `partials/publication_entry.html`. Supported fields: `title, author, year, doi, url, booktitle, journal, cover`. DOI takes precedence over `url` for the card link. The `cover` field, if it starts with `/`, resolves under `static/`; otherwise it's looked up in the page bundle.
- `cvside` — pushes CV sidebar data (`name`, `photo`, contact links with `name/icon/url`) into `$.Page.Scratch`. Used only on the home/CV page; consumed later by `partials/cv_profile.html`.
- `row` — wraps `.Inner` in `<div class="row-text-wrap">` whose first-paragraph image floats right. Used in `projects/torchextractx/index.*.md`.

## Bilingual content convention

- `defaultContentLanguageInSubdir: true` → URLs are `/en/...` and `/zh/...`. Home is `/en/` and `/zh/`.
- Per-page translations live as siblings: `index.en.md` + `index.zh.md` (or `_index.md` + `_index.zh.md` for sections). `layouts/partials/translation_list.html` links between them.
- New post: `hugo new content/projects/<slug>/index.en.md` then create `index.zh.md` next to it. Add a `cover.png` (≈16:9, ≤ a few MB) to the page bundle if you want the side cover.
- Page bundles must be **leaf bundles** (have an `index.*.md`) so `.Page.Resources.GetMatch` can find sibling `*.bib` and cover images.

## Cover configuration

Per-page cover settings live in `params.cover.pages` (map keyed by section-relative slug, e.g. `projects/torchextractx`):

```yaml
cover:
  hidden: true
  hiddenInList: true
  hiddenInSingle: true
  pages:
    projects/torchextractx:
      image: "cover.png"
      hiddenInSingle: false
      hiddenInList: false
      listLayout: "side"      # "side" enables cover-side list layout
      singleLayout: "strip"   # reserved; consumed by partials/cover.html
      position: "right"       # "left" | "right"
      width: "240px"
      aspectRatio: "16 / 9"
      singleAspectRatio: "28 / 9"
```

The CSS variables `--entry-cover-width` and `--entry-cover-ratio` (set inline on the `<article>` in `list.html`) drive the side-cover geometry.

## Things not to touch

- `themes/PaperMod/` — git submodule pointing at `adityatelange/hugo-PaperMod`. If you need theme changes, fork the submodule or override via the existing files in `layouts/`. Do not commit into the submodule from this repo.
- `public/`, `resources/`, `.hugo_build.lock` — build artifacts, gitignored.
- `.claude/settings.json` — tracked project-level Claude Code settings; `.claude/settings.local.json` is per-user (excluded via the user's global gitignore).

## Commit message convention

Every commit message ends with the dual `Co-authored-by` trailer (matches the format used in this repo's prior commits — `git log --format=%B` to verify):

```
Co-authored-by: MiniMax M3 <minimax-m3@users.noreply.github.com>
Co-authored-by: Claude Code for VS Code <claude-code-vscode@users.noreply.github.com>
```

The two lines are required together (one for the model, one for the IDE). Do not substitute a single Claude line or any other address.

## Deployment

Push to the `hugo` branch → `.github/workflows/hugo.yaml` runs:

1. Checkout with `submodules: recursive` (needed for PaperMod)
2. Install Go 1.26.1, Node 24.14.1, Hugo extended 0.160.0, Dart Sass 1.99.0
3. `hugo build --gc --minify --baseURL ${{ steps.pages.outputs.base_url }}/ --cacheDir ...`
4. Upload `./public` as a Pages artifact, then `actions/deploy-pages` publishes it.

`hugo.yaml` `editPost.URL` points at `https://github.com/5o1/5o1.github.io/edit/hugo/content`; the rendered "Edit" link goes to that path.
