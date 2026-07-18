---
title: gomuks documentation
linkTitle: Home
---

{{% blocks/cover title="gomuks docs" height="min" %}}
A Matrix client written in Go — developer and operator documentation from the
in-repo guides.

<a class="btn btn-lg btn-primary me-3 mb-4" href="{{% ref "/docs" %}}">
  Browse docs
</a>
<a class="btn btn-lg btn-secondary me-3 mb-4" href="https://docs.mau.fi/gomuks/">
  User install & FAQ
</a>
{{% /blocks/cover %}}

{{% blocks/lead %}}
This site is generated with **Hugo** and the **Docsy** theme. Canonical
markdown lives under `docs/` in the monorepo (`ARCHITECTURE.md`, `SERVER.md`,
…). End-user installation remains on
[docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/).
{{% /blocks/lead %}}

{{% blocks/section type="row" color="white" %}}

{{% blocks/feature icon="fa-solid fa-sitemap" title="Architecture" url="/docs/architecture/" %}}
Backend/frontend split, `hicli`, RPC, security notes, and contributor map.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-server" title="Server configuration" url="/docs/server/" %}}
`config.yaml`, directories, auth, reverse proxy, Docker, and ops checklists.
{{% /blocks/feature %}}

{{% blocks/feature icon="fa-solid fa-code" title="RPC API" url="https://spec.mau.fi/gomuks/rpc.html" %}}
JSON command surface (also generable via `cmd/rpcdocgen`).
{{% /blocks/feature %}}

{{% /blocks/section %}}
