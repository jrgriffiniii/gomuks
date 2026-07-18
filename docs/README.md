# gomuks documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Monorepo structure, RPC, security notes |
| [SERVER.md](./SERVER.md) | Backend configuration and operations |
| [website/](./website/) | **Hugo + Docsy** site that renders the above |

## Preview with Hugo + Docsy

```sh
cd docs/website
npm install
hugo mod tidy
hugo server -D
```

Open http://localhost:1313/

See [website/README.md](./website/README.md) for details.

## User-facing docs

Installation and FAQ live on [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/)
(maintained in the mautrix docs project), not in this folder.
