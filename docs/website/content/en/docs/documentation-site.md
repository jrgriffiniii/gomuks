---
title: Documentation site (Hugo + Docsy)
linkTitle: Docs site
weight: 90
description: >-
  How this Hugo/Docsy site is structured, built, and related to docs.mau.fi.
---

## Purpose

gomuks already has **user** documentation on
[docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/) (install, FAQ, multi-account,
storage). That book is maintained in the [mautrix/docs](https://github.com/mautrix/docs)
project.

This **Hugo + Docsy** site covers **in-repo developer and operator** material:

- Architecture and design notes
- Server / `config.yaml` operations
- Future: embedding `rpcdocgen` output, contributing guides, ADRs

Keeping both is intentional: mau.fi stays the polished end-user book; this site
tracks the monorepo closely and can be published to GitHub Pages or any static
host from CI.

## Layout

```text
docs/
  ARCHITECTURE.md          # canonical source
  SERVER.md                # canonical source
  website/                 # Hugo project (Docsy as Hugo Module)
    hugo.yaml
    go.mod / go.sum
    content/en/            # Docsy content tree + front matter
    layouts/shortcodes/    # include.html pulls parent markdown
    package.json           # PostCSS deps for Docsy SCSS
    README.md
```

Pages such as `content/en/docs/architecture.md` only hold front matter (title,
weight, description) and an `include` shortcode. **Edit `docs/*.md`**, not a
duplicated body inside `content/`.

## Prerequisites

| Tool | Why |
|------|-----|
| [Hugo extended](https://gohugo.io/installation/) ≥ 0.157 | Docsy SCSS / modules |
| Go ≥ 1.22 | `hugo mod` downloads Docsy |
| Node.js + npm | PostCSS / autoprefixer for production CSS |
| Git | Module fetch |

macOS (Homebrew):

```sh
brew install hugo
hugo version   # must say +extended
```

Or pin Hugo via npm (see `package.json` `hugo-extended` if added).

## Local preview

```sh
cd docs/website
npm install          # bootstrap, fontawesome, postcss
hugo mod tidy        # Docsy module
hugo server -D
```

Open http://localhost:1313/

Default `baseURL` is `http://localhost:1313/` so section pages are at the
**site root**, not under a `/gomuks/` prefix:

| Page | Local URL |
|------|-----------|
| Server | http://localhost:1313/docs/server/ |
| Architecture | http://localhost:1313/docs/architecture/ |

If you previously built with `baseURL: https://…/gomuks/`, restart Hugo after
pulling config changes (`Ctrl+C`, then `hugo server -D` again). Old tabs may
still request `/gomuks/docs/server/`, which 404s with the root baseURL.

Stub pages load parent markdown via Hugo mounts:

| Canonical file | Mounted asset |
|----------------|---------------|
| `docs/ARCHITECTURE.md` | `assets/includes/ARCHITECTURE.md` |
| `docs/SERVER.md` | `assets/includes/SERVER.md` |

## Production build

```sh
cd docs/website
npm ci
hugo --minify
```

Output: `docs/website/public/` (gitignored).

## Publishing options

| Target | Notes |
|--------|--------|
| GitHub Pages | Workflow: build on push to `main`, upload `public/`; set `baseURL` |
| GitLab Pages | Similar; project already uses GitLab CI for binaries |
| Netlify / Cloudflare Pages | Point build command at `docs/website` |
| Embedded in binary | Not recommended; keep docs static and external |

Suggested GitHub Actions sketch (not enabled by default):

```yaml
# .github/workflows/docs.yml
name: docs
on:
  push:
    paths: ['docs/**']
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.164.0"
          extended: true
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25"
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
      - working-directory: docs/website
        run: |
          npm ci
          hugo mod tidy
          hugo --minify
      # then peaceiris/actions-gh-pages or actions/upload-pages-artifact
```

## Relationship to `rpcdocgen`

`go run ./cmd/rpcdocgen -o rpc.html` emits a **self-contained HTML** reference
from `pkg/hicli/jsoncmd`. Options:

1. Link out to the published spec (e.g. spec.mau.fi) — current approach on the home page.
2. CI: run rpcdocgen into `docs/website/static/rpc/` and link `/rpc/`.
3. Longer term: teach rpcdocgen to emit Markdown into `content/en/docs/rpc/`.

## Why Docsy

| Need | Docsy provides |
|------|----------------|
| Docs section nav + left sidebar | Built-in `docs` section layout |
| Search without a SaaS key | Offline search (`params.offlineSearch`) |
| Dark mode, anchors, code highlight | Theme defaults |
| Edit-on-GitHub links | `github_repo` / branch params |
| Familiar CNCF-style docs UX | Wide contributor recognition |

Alternatives considered: plain Hugo (more DIY nav), MkDocs Material (Python),
mdBook (matches mau.fi stack but separate from Docsy request). Docsy fits a Go
monorepo that already uses Go modules for tooling.

## Limitations of this integration

1. **Shortcode include** re-renders markdown inside Hugo; rare Goldmark edge
   cases may differ from GitHub’s markdown preview.
2. **Leading H1** in included files is stripped so Docsy titles are not doubled.
3. **Edit this page** GitHub links point at `content/en/...` stubs unless you
   set page-level `github_url` to the canonical `docs/*.md` file.
4. **docs.mau.fi** remains the user install book — avoid duplicating FAQ content
   here; link instead.
