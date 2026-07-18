# gomuks documentation website (Hugo + Docsy)

Static site that publishes in-repo guides (`docs/ARCHITECTURE.md`,
`docs/SERVER.md`, …) with the [Docsy](https://www.docsy.dev/) theme.

**User install / FAQ** stay on [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/).
This site is for **developer and operator** documentation that lives next to
the code.

## Quick start

```sh
# Prerequisites: Hugo extended ≥ 0.157, Go, Node.js 20+
brew install hugo   # macOS; must report +extended

cd docs/website
npm install
hugo mod tidy
hugo server -D
```

Open [http://localhost:1313/](http://localhost:1313/).

Useful local paths (with default `baseURL: http://localhost:1313/`):

| Page | URL |
|------|-----|
| Home | http://localhost:1313/ |
| Docs index | http://localhost:1313/docs/ |
| Architecture | http://localhost:1313/docs/architecture/ |
| Server | http://localhost:1313/docs/server/ |

If you build with a subdirectory `baseURL` (for example GitHub Pages
`https://user.github.io/gomuks/`), all links are under that prefix
(e.g. `http://localhost:1313/gomuks/docs/server/`). Prefer the default root
`baseURL` for local work; override only at deploy time:

```sh
hugo --minify --baseURL https://gomuks.github.io/gomuks/
```

Production build:

```sh
npm run build
# → public/
```

## Editing content

| Topic | Edit this file |
|-------|----------------|
| Architecture | [`../ARCHITECTURE.md`](../ARCHITECTURE.md) |
| Server configuration | [`../SERVER.md`](../SERVER.md) |
| How this site works | [`content/en/docs/documentation-site.md`](content/en/docs/documentation-site.md) |
| Home / nav chrome | `content/en/_index.md`, `hugo.yaml` |

Stub pages under `content/en/docs/*.md` only set Docsy front matter and
`{{% include path="..." %}}`. Do not duplicate the full article body there.

## More detail

See the generated page **Documentation site (Hugo + Docsy)** after `hugo server`,
or read `content/en/docs/documentation-site.md` directly.
