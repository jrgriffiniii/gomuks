---
title: Documentation
linkTitle: Docs
weight: 1
menu:
  main:
    weight: 10
description: >-
  In-repo architecture, server operations, and developer guides for gomuks.
cascade:
  type: docs
---

These pages are built from markdown in the monorepo `docs/` directory (and
related packages). Prefer editing the source files at the repository root of
`docs/` so the site and plain-file readers stay in sync.

| Guide | Source file |
|-------|-------------|
| [Architecture]({{% relref "/docs/architecture" %}}) | `docs/ARCHITECTURE.md` |
| [Server configuration]({{% relref "/docs/server" %}}) | `docs/SERVER.md` |
| [Docs site (Hugo + Docsy)]({{% relref "/docs/documentation-site" %}}) | `docs/website/` |

User-facing install and FAQ: [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/).
