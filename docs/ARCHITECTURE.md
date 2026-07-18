# gomuks architecture

This document describes how the gomuks monorepo is structured, how the backend
and frontends communicate, and notes from an internal codebase review (security,
correctness risk, testing, and maintainability).

For end-user install and usage docs, see [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/).
For operator/server configuration, see [SERVER.md](./SERVER.md).
For browsing these guides as a Docsy site, see [website/](./website/) (`hugo server` in `docs/website`).
For the on-wire RPC reference, generate HTML from `pkg/hicli/jsoncmd` with
[`cmd/rpcdocgen`](../cmd/rpcdocgen/ARCHITECTURE.md):

```sh
go run ./cmd/rpcdocgen -o jsoncmd.html
```

---

## High-level model

gomuks is a **Matrix client** built on [mautrix-go](https://github.com/mautrix/go).
It is split into:

1. A **high-level Matrix client core** (`pkg/hicli`) that owns sync, crypto,
   sending, local SQLite state, and the typed JSON RPC command surface.
2. A **server / shell** (`pkg/gomuks`) that exposes that core over HTTP and
   WebSocket, plus media, auth, and push helpers for the web stack.
3. **Multiple frontends** that speak the same JSON RPC (or embed the core
   in-process).

The backend can run as a traditional local process with an embedded UI, or as a
**long-lived bouncer-style server** that frontends reconnect to. That is the
main product shape for gomuks web and the Electron desktop wrapper.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                         Frontends                                        │
│  web (React)  │  desktop (Electron)  │  TUI  │  WASM  │  FFI / Nexus     │
└───────┬───────────────┬─────────────────┬───────┬────────────┬───────────┘
        │               │                 │       │            │
        │  WebSocket /  │  subprocess +   │  pkg/ │  in-process│  C API
        │  HTTP RPC     │  WS to local    │  rpc  │  channel   │
        ▼               ▼                 ▼       ▼            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  pkg/gomuks  — HTTP, cookie auth, media proxy, push, event buffer, WS    │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │ SubmitJSONCommand / EventHandler
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  pkg/hicli  — HiClient (sync, crypto, send, pagination, push rules, …)   │
│       │                                                                  │
│       ├── pkg/hicli/database  — SQLite schema + queries + migrations     │
│       └── pkg/hicli/jsoncmd   — command/event names, params, specs       │
└───────────────────────────────────┬──────────────────────────────────────┘
                                    │
                                    ▼
                         Matrix homeserver (CS API)
                              via mautrix-go
```

---

## Repository layout

| Path | Role |
|------|------|
| `cmd/gomuks` | Main binary: starts `pkg/gomuks` server, embeds web frontend |
| `cmd/gomuks-terminal` | Terminal UI entry (`tui`) |
| `cmd/wasmuks`, `cmd/wasmukserve` | WASM backend build and static serve helpers |
| `cmd/rpcdocgen` | AST-based HTML generator for the JSON RPC API |
| `cmd/archivemuks`, `cmd/chromagen` | Ancillary tools |
| `pkg/hicli` | Core Matrix client library (MPL-2.0) |
| `pkg/hicli/database` | Local SQLite model (rooms, events, media, spaces, …) |
| `pkg/hicli/jsoncmd` | RPC envelope, command/event specs and docs |
| `pkg/gomuks` | Web backend shell: auth, WS, media, push, config (AGPL-3.0) |
| `pkg/rpc` | Go RPC client + client-side store for embedding consumers |
| `pkg/ffi` | C FFI so non-Go frontends (e.g. Flutter Nexus) can embed the backend |
| `pkg/sqlite-wasm-js` | `database/sql` driver bridge for browser SQLite WASM |
| `web/` | Production React + Vite frontend |
| `desktop/` | Electron (Forge) wrapper around the web UI + local backend |
| `tui/` | Terminal UI (experimental; port of legacy gomuks) |
| `version/` | Build-time version metadata |
| `docs/` | In-repo design notes (this file) |

Approximate scale (order of magnitude, source lines only):

| Area | Size |
|------|------|
| `web/src` | ~26k lines TS/TSX |
| `pkg/hicli` (incl. database) | ~12k+ lines Go |
| `tui` | ~7.5k lines Go |
| `pkg/gomuks` | ~3.7k lines Go |
| `pkg/rpc` | ~2k lines Go |

---

## Core: `pkg/hicli`

`HiClient` is the opinionated high-level client. Responsibilities include:

- Login / logout / OAuth-related flows
- Sync loop and state application into SQLite
- E2EE via mautrix crypto (`OlmMachine`, SQL crypto store, key backup hooks)
- Outbound send queue, redactions, reactions, receipts, typing
- Timeline pagination and search (local FTS where enabled)
- Push rules evaluation hooks used by the shell for notifications
- Typed dispatch of JSON RPC commands (`SubmitJSONCommand` / `handleJSONCommand`)
- Outbound events to frontends via `EventHandler`

Concurrency is explicit: sync, login, encrypt, per-room send locks, pagination
interrupters, and a map of cancel functions for in-flight JSON requests.

### Local database

Data lives in SQLite (WAL, foreign keys) under the user data directory
(`gomuks.db` by default when run through `pkg/gomuks`). Schema evolution is
handled by `pkg/hicli/database/upgrades` (numbered `.sql` files plus occasional
Go migrations), embedded and applied through `go.mau.fi/util/dbutil`.

Typical stored domains:

- Account / tokens / next-batch
- Rooms, invited rooms, spaces graph
- Events, timeline, receipts, unread / mentions
- Current state cache
- Media cache metadata (and encryption info for encrypted media)
- Push registrations

Queries are generally parameterized. Mass-insert helpers and `IN (...)`
placeholder expansion are used for batch operations; user input is not
string-concatenated into SQL text as free-form fragments.

Crypto-related tables may share the same DB file or a separate pool depending
on how the host constructs `HiClient`.

### JSON RPC surface (`pkg/hicli/jsoncmd`)

All frontend ↔ backend messages share one envelope (see `envelope.md`):

| Field | Meaning |
|-------|---------|
| `command` | Request, response, or event name |
| `request_id` | Correlates request/response; events use negative, decreasing IDs |
| `data` | Command-specific payload |

Responses for a given `request_id` are exactly one of:

- `command: "response"` with typed success data
- `command: "error"` with an error string

In-flight work can be cancelled with a `cancel` command (best-effort).

Commands and events are declared as typed `CommandSpec` / `EventSpec` values in
Go so `rpcdocgen` can extract names, docs, and JSON shapes from the AST. That
keeps the human-readable API docs close to the source of truth.

The RPC is **transport-agnostic**: WebSocket is the primary remote transport,
but the same JSON can go over HTTP `POST /_gomuks/exec/{command}`, an in-process
channel (WASM / embedded), or the C FFI (envelope folded into function args).

---

## Shell: `pkg/gomuks`

`Gomuks` wraps `HiClient` with process-level concerns:

| Concern | Implementation sketch |
|---------|------------------------|
| Config | `config.yaml` under config dir; bcrypt password hash, HMAC token key, listen address, origin patterns, VAPID keys, media thumbnail size, logging |
| Directories | Config / data / cache / log (XDG on *nix; platform defaults on Windows/macOS); overridable via `GOMUKS_*` / `GOMUKS_ROOT` |
| HTTP API | Under `/_gomuks/` with auth middleware |
| Static UI | Embedded `web` dist; index HTML injected with frontend ETag + VAPID public key |
| Event buffer | In-memory fan-out to WebSocket clients with optional resume |
| Desktop mode | Random listen port, desktop shared secret auth, JSON “started” line on stdout |

### HTTP routes (under `/_gomuks`)

| Method / path | Purpose |
|---------------|---------|
| `GET /websocket` | Primary RPC + event stream |
| `POST /auth` | Login; sets `gomuks_auth` cookie or returns token JSON |
| `POST /exec/{command}` | One-shot JSON RPC (same commands as WS) |
| `POST /upload`, `GET /media/{server}/{media_id}` | Media upload/download proxy + cache |
| `POST /keys/export`, `import`, restore backup | Key backup / export helpers |
| `GET /url_preview` | URL preview via **homeserver** API (not open outbound crawl from gomuks itself) |
| `GET /codeblock/{style}` | Syntax-highlight CSS |

Optional `/debug/` (pprof) when `web.debug_endpoints` is enabled.

Default listen address is `localhost:29325` (or `localhost:0` in desktop
subprocess mode).

### WebSocket behavior

Documented in detail in `pkg/hicli/jsoncmd/websocket.md`. Summary:

- Cookie auth required (same as other `/_gomuks` routes).
- Optional `compress=1`: server→client deflate, with possible newline-batched frames under backpressure.
- Session resume: client stores `run_id` and last negative event id; reconnect with query params. Buffer is **in-memory only** (lost on backend restart).
- Clients must ping (~15s recommended); backend closes after ~60s without client data.
- Pings also ack events (`last_received_id`) so the buffer can drop old events.
- Read limit 1 MiB; full outbound event queue closes the connection with a custom status.
- Periodic **image auth tokens** are pushed so `<img>` / media fetches can use short-lived image-only credentials.

### Authentication model

gomuks web auth protects the **local/bouncer API**, not Matrix’s own login
directly. After Matrix login, the Matrix access token lives in the backend DB;
anyone who can authenticate to gomuks can use that session.

Mechanisms:

1. **Username + bcrypt password** (prompted on first run if unset).
2. **HMAC-SHA256 signed tokens** (JSON payload + signature, username + expiry).
3. **HttpOnly cookie** `gomuks_auth` (Secure by default, SameSite=Lax).
4. **Image-only tokens** for media routes (`Authorization: Image …` or `image_auth` query), so media URLs need not carry the full session cookie.
5. **Desktop key** basic auth (`desktop-key` / shared secret) when run as an Electron subprocess.
6. Explicit escape hatch config key  
   `disable_auth_because_i_want_my_account_to_be_hacked` (name intentional).

WebSocket origin checks use `web.origin_patterns`. Insecure cookies are opt-in
(`insecure_cookies` or non-browser clients with `insecure_cookie=true` and no
`Sec-Fetch-Site`).

**Deployment note:** Treat the process as a **single-user** personal client or
bouncer. Expose it only behind TLS and strong credentials (or keep it on
loopback). It is not a multi-tenant homeserver.

---

## Frontends

### Web (`web/`)

Most mature UI; intended for daily use.

- **Stack:** React 19, Vite, TypeScript, ESLint; Vitest configured.
- **API layer:** `WSClient` / WASM client → `RPCClient` → `statestore`.
- **Features:** timeline, composer, spaces, room settings, widgets, Element Call embed, search, media, push (VAPID service worker assets), etc.
- **Update hook:** Backend injects frontend ETag; WS init can trigger a controlled reload when the embedded UI changes.

Key directories:

- `web/src/api/` — transport, types, client, media, state store
- `web/src/ui/` — screens (timeline, room list, composer, modals, settings, …)
- `web/src/util/` — markdown, emoji data, hooks helpers

### Desktop (`desktop/`)

Electron app that spawns the Go backend with a desktop key and loads the web UI
against the local listen address. Shares the web codebase rather than
reimplementing Matrix logic.

### Terminal (`tui/`, `cmd/gomuks-terminal`)

Port of legacy gomuks TUI. Feature set is intentionally narrower than web;
README marks it experimental. Uses mauview/tcell-oriented widgets and its own
message/HTML rendering path under `tui/messages`.

### WASM (`cmd/wasmuks`, `web` WASM client paths)

Backend compiled to WASM with SQLite via `pkg/sqlite-wasm-js`, talking to the
same frontend through an in-process bridge instead of a network WebSocket.

### FFI (`pkg/ffi`) and external UIs

C ABI for embedding (e.g. Flutter [Nexus](https://git.federated.nexus/Henry-Hiles/nexus)).
Command submit/start lifecycle is strict; misuse panics intentionally.

### Go embed clients (`pkg/rpc`)

Typed Go client and local store for programs that want hicli-like state without
reimplementing WS framing.

---

## Data and control flow (typical web session)

```text
Browser                    gomuks process                     Homeserver
   │                            │                                  │
   │  POST /_gomuks/auth        │                                  │
   │  (basic → cookie)          │                                  │
   │───────────────────────────►│                                  │
   │  GET /_gomuks/websocket    │                                  │
   │───────────────────────────►│                                  │
   │  run_id / sync events      │                                  │
   │◄───────────────────────────│                                  │
   │  send_message (JSON)       │                                  │
   │───────────────────────────►│  Client-Server API               │
   │                            │─────────────────────────────────►│
   │                            │◄─────────────────────────────────│
   │  response + timeline evt   │                                  │
   │◄───────────────────────────│                                  │
   │  GET /_gomuks/media/...    │  (cache miss → HS media)         │
   │  + image_auth token        │─────────────────────────────────►│
```

Sync runs inside `HiClient` independently of any single WebSocket. The shell
subscribes to `EventHandler` and fans events into the in-memory buffer for
connected UIs. A client that successfully resumes only receives missed buffer
entries; a full reconnect after backend restart gets `clear_state` on the first
complete sync payload (see websocket docs).

---

## Licensing

| Component | License |
|-----------|---------|
| Most of the repo (web shell, TUI, `pkg/gomuks`, frontends) | **AGPL-3.0-or-later** |
| `pkg/hicli` (and its `LICENSE`) | **MPL-2.0** |

The split allows the high-level client library to be reused under MPL while the
product shell and UI remain AGPL. Contributors should note which tree they edit.

---

## Build and quality tooling

- Go module: `go.mau.fi/gomuks` (see `go.mod` for toolchain pin).
- Web: `web/package.json` scripts (`dev`, `build`, `lint`, `test`).
- Root helpers: `build.sh`, `build-noweb.sh`, `build-terminal.sh`.
- CI: `.gitlab-ci.yml` (and GitHub metadata as needed).
- Pre-commit: eslint / tsc hooks.
- Optional Nix flake for reproducible env.

---

## Codebase assessment (review notes)

The following summarizes an architecture/quality review of `main` (not a
line-by-line audit of crypto or every UI path). It is meant to guide
contributors on where the design is strong and where risk concentrates.

### Strengths

1. **Clear layering** — Matrix domain logic lives in `hicli`; HTTP/auth/media in
   `gomuks`; UI in frontends. The JSON RPC boundary is documented and codegen’d.
2. **Multi-frontend by design** — Same commands over WS, HTTP, in-process, and
   FFI without forking the Matrix implementation.
3. **Auth hygiene for a personal bouncer** — bcrypt, HMAC cookies, image-only
   media tokens, constant-time compares for secrets, loud insecure-auth naming,
   origin patterns, Secure cookies by default.
4. **Operational WS details** — resume, compression, ping/ack, backpressure
   close, panic recovery in WS goroutines, explicit read limits.
5. **SQLite evolution** — ordered migrations, rich local model (spaces, FTS,
   media cache, push registrations).
6. **Honest product maturity** — web called out as daily-driver; TUI as
   experimental.

### Main gap: automated tests

Compared to surface area, automated tests are **very thin**:

- Go: little beyond `cmd/rpcdocgen` unit tests.
- Web: Vitest is present; almost no component/store tests in-tree.

Highest ROI test targets:

| Priority | Target | Why |
|----------|--------|-----|
| High | Token sign/validate (expiry, `image_only`, bad HMAC, username mismatch) | Security-critical, pure logic |
| High | DB upgrade apply + a few query contracts | Prevents silent schema breakages |
| High | `SubmitJSONCommand` happy path + unknown command / cancel | Locks RPC contract |
| Medium | `statestore` pure merge/update helpers (web) | Dense UI logic, no network needed |
| Medium | Media cache encrypted vs non-encrypted guards | Easy to regress |
| Lower | Sync limited-timeline / reset paths | High value but harder fixtures |

### Correctness risk hotspots

These areas are complex and lightly covered by tests; changes need careful
review and manual Matrix testing:

| Area | Location (indicative) | Why sensitive |
|------|----------------------|---------------|
| Sync / timeline reset | `pkg/hicli/sync.go`, pagination | Ordering, limited timelines, state rebuild |
| Decryption queue | `pkg/hicli/decryptionqueue.go` | Concurrency + crypto + DB |
| Direct chats maps | `pkg/hicli/direct.go` | Malformed account data edge cases |
| Event buffer fan-out | `pkg/gomuks` WS + buffer | Multi-client, full-queue disconnects |
| Media pipeline | `pkg/gomuks/media.go` (+ heic/webp helpers) | Temp files, encryption flags, thumbnails, optional ImageMagick |
| TUI rendering / TODOs | `tui/` | Experimental; higher open TODO density |

`panic` is used mainly for unreachable states, FFI misuse, and hard startup
listen failures—not for routine request handling (WS handlers recover panics).

### Security residual notes

| Severity | Note |
|----------|------|
| Suggestion | Auth gate is **gomuks access**, not multi-user isolation. Document and assume single-user. |
| Suggestion | `debug_endpoints` exposes pprof-style debug routes; keep off on non-loopback binds. |
| Suggestion | URL previews are delegated to the homeserver—good for SSRF posture of gomuks itself; still authenticated. |
| Nit | Startup listen/serve failures `panic` rather than return structured errors (fine for CLI, noisier under supervision). |
| Nit | Token length checks differ slightly between validators (conservative, but could be unified). |

### Maturity by frontend

| Frontend | Maturity | Notes |
|----------|----------|--------|
| Web | Production-oriented | Largest feature set; primary investment |
| Desktop | Wrapper | Depends on web + local backend process |
| WASM | Supported path | Same UI, different transport/DB driver |
| TUI | Experimental | Basic chatting; not feature-parity |
| FFI / Nexus | Early / external | Embedding API exists; UI evolves out of tree |

### What to trust vs scrutinize when changing code

| Prefer to reuse as-is | Scrutinize on every change |
|-----------------------|----------------------------|
| RPC envelope and `jsoncmd` specs | Sync limited timelines / `clear_state` behavior |
| Auth cookie + image token design (for localhost/TLS single-user) | Multi-WS session and buffer ack semantics |
| Parameterized SQL + upgrade runner | Decryption failure recovery and key backup restore |
| MXC validation and encrypted media flags | Image conversion / thumbnail edge cases |
| Web reconnect/resume design | TUI protocol/error handling parity |

---

## Suggested contributor map

| If you are changing… | Start here |
|----------------------|------------|
| A Matrix behavior (send, sync, crypto, DB) | `pkg/hicli`, `pkg/hicli/database` |
| RPC command name or payload shape | `pkg/hicli/jsoncmd` (+ regenerate rpcdocgen HTML) |
| Auth, media HTTP, push, WS framing | `pkg/gomuks` |
| Web UI / client state | `web/src/ui`, `web/src/api/statestore` |
| Desktop packaging | `desktop/` |
| Terminal UI | `tui/` |
| Embedders (Go) | `pkg/rpc` |
| Embedders (C / mobile) | `pkg/ffi` |
| Browser-native backend | `cmd/wasmuks`, `pkg/sqlite-wasm-js`, `web` WASM client |

---

## Related docs

| Doc | Content |
|-----|---------|
| [`pkg/hicli/jsoncmd/envelope.md`](../pkg/hicli/jsoncmd/envelope.md) | RPC envelope, responses, cancel |
| [`pkg/hicli/jsoncmd/websocket.md`](../pkg/hicli/jsoncmd/websocket.md) | WS compression, resume, ping/ack |
| [`cmd/rpcdocgen/ARCHITECTURE.md`](../cmd/rpcdocgen/ARCHITECTURE.md) | How the RPC HTML reference is generated |
| [README.md](../README.md) | Product overview and maturity statements |
| [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/) | Installation and usage |

---

## Document history

- Initial version: architecture overview plus codebase review findings (layering,
  security model, risk hotspots, testing gap, contributor map).
