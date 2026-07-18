# Configuring the gomuks server

This guide explains how to run and configure the **gomuks web backend**
(`cmd/gomuks` / `pkg/gomuks`): directories, `config.yaml`, authentication,
networking, reverse proxies, Docker, push, and operational pitfalls.

For architecture and layering, see [ARCHITECTURE.md](./ARCHITECTURE.md).
For product install overview and FAQ, see also
[docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/).
To preview this guide in the Hugo + Docsy site: `cd docs/website && hugo server`.

---

## What you are configuring

The gomuks **server** is a single-user Matrix client process that:

1. Speaks Client-Server API to a homeserver (via mautrix-go / `hicli`).
2. Serves the embedded web UI and the `/_gomuks` HTTP + WebSocket API.
3. Stores encryption keys, rooms, and timeline in local SQLite.

It is a **personal bouncer / local backend**, not a multi-tenant Matrix
homeserver. Anyone who can authenticate to this process can use the Matrix
session stored in its data directory.

Default browser URL after first start: **http://localhost:29325**

---

## Quick start

```sh
# Build or install a binary, then:
gomuks
```

On first run (interactive TTY):

1. You are prompted for a **gomuks** username and password (not Matrix credentials).
2. `config.yaml` is written under the config directory (mode `0600`).
3. Secrets (`token_key`, VAPID keys, etc.) are generated and saved.
4. The HTTP server listens on `localhost:29325` by default.
5. Open the URL in a browser and complete **Matrix** login in the UI.

CLI flags on the main binary are minimal:

| Flag | Meaning |
|------|---------|
| `-h` / help | Usage |
| `-v` / `--version` | Print version and exit |
| `--desktop` | Desktop-subprocess mode (reads `GOMUKS_DESKTOP_KEY`) |

There is **no** CLI flag for listen address or config path; use environment
variables and `config.yaml` (below).

---

## Directories and environment variables

On startup, `InitDirectories` creates four logical homes (mode `0700`) plus a
temp dir. Resolution order:

1. Specific `GOMUKS_*_HOME` if set  
2. Else under `GOMUKS_ROOT` (or an empty root argument when calling the API)  
3. Else OS defaults  

`GOMUKS_*_HOME` wins over paths derived from `GOMUKS_ROOT` when both are set
(prefer fine-grained overrides).

| Variable | Contents | Default (*nix, typical) |
|----------|----------|-------------------------|
| `GOMUKS_ROOT` | Parent for config/data/cache/logs if individual vars unset | unset |
| `GOMUKS_CONFIG_HOME` | `config.yaml` | `$XDG_CONFIG_HOME/gomuks` or `~/.config/gomuks` |
| `GOMUKS_DATA_HOME` | SQLite DB (`gomuks.db`), persistent state | `$XDG_DATA_HOME/gomuks` or `~/.local/share/gomuks` |
| `GOMUKS_CACHE_HOME` | Media cache (safe to delete) | `$XDG_CACHE_HOME/gomuks` or `~/.cache/gomuks` |
| `GOMUKS_LOGS_HOME` | Log directory (default log file path is set here on first setup) | `$XDG_STATE_HOME/gomuks` or `~/.local/state/gomuks` |
| `GOMUKS_TMPDIR` | Temp files (downloads, conversion) | `$GOMUKS_CACHE_HOME/tmp` |
| `GOMUKS_DESKTOP_KEY` | Shared secret when started with `--desktop` | unset |
| `https_proxy` / related | Outbound HTTP(S) proxy to homeservers (Go default) | unset |

### Platform defaults (when `GOMUKS_ROOT` is unset)

| OS | Config | Data | Cache | Logs |
|----|--------|------|-------|------|
| *nix | XDG config | XDG data | XDG cache | XDG state |
| macOS | `~/Library/Application Support/gomuks` | same as config | `~/Library/Caches/gomuks` | `~/Library/Logs/gomuks` |
| Windows | `%AppData%\gomuks` | same as config | `%LocalAppData%\gomuks` | `%LocalAppData%\gomuks\logs` |

### What lives where

| Store | Path (conceptually) | Notes |
|-------|---------------------|--------|
| Config | `$CONFIG/config.yaml` | Listen address, auth hash, token key, logging, push keys |
| Database | `$DATA/gomuks.db` (+ WAL/SHM) | Rooms, events, **encryption keys**, tokens |
| Media cache | `$CACHE/media/...` | Content-addressed; deletable; shareable across instances |
| Temp | `$TMPDIR` | Short-lived download/encode work |
| Logs | `$LOGS/gomuks.log` by default | Also stdout pretty logs by default |

**Deleting data logs you out** (keys and session gone). **Deleting cache** only
drops downloaded media/thumbnails.

### Multiple accounts

One process = one Matrix account. Run multiple processes with different
`GOMUKS_ROOT` (or different `*_HOME` sets) and different `listen_address` ports.

Without a reverse proxy, use distinct localhost hostnames to avoid cookie
clashes, e.g. `http://main.localhost:8000` and `http://work.localhost:8001`.

---

## Config file: `config.yaml`

**Path:** `$GOMUKS_CONFIG_HOME/config.yaml` (or `$GOMUKS_ROOT/config/config.yaml`).

**Load behavior:**

- Missing file → defaults + interactive password prompt (unless auth disabled / desktop mode) → save.
- Empty or missing secrets → generate and rewrite config (`0600`).
- Invalid YAML → process exits with code `9`.

**Do not commit this file.** It contains password hashes, HMAC token keys, and
VAPID private keys.

### Annotated example

Values below match code defaults and auto-fill behavior after first run.
Generated secrets will differ on every install.

```yaml
web:
  # TCP bind address for the HTTP server (UI + /_gomuks API).
  # localhost:29325 — loopback only (safest default).
  # 0.0.0.0:29325  — all interfaces (typical for Docker / reverse proxy).
  # localhost:0    — ephemeral port (forced in desktop subprocess mode).
  listen_address: localhost:29325

  # Basic-auth identity for the *gomuks* UI/API (not Matrix).
  username: yourname
  # bcrypt hash (cost 12). Set via first-run prompt; do not hand-edit lightly.
  password_hash: "$2a$12$..."

  # HMAC key for signing session cookies / tokens. Auto-generated (64 chars).
  token_key: "..."

  # Max events kept in the in-memory WebSocket resume buffer. Default 512.
  event_buffer_size: 512

  # Allowed browser Origins for WebSocket upgrade (coder/websocket patterns).
  # Defaults: localhost and *.localhost on any port.
  # For a public HTTPS host, include that host (no scheme), e.g.:
  #   - chat.example.com
  #   - chat.example.com:443   # only if clients include an explicit port
  origin_patterns:
    - "localhost:*"
    - "*.localhost:*"

  # Allow Secure=false cookies (HTTP over non-localhost). Required for
  # plain-http remote access without TLS. Prefer reverse-proxy TLS instead.
  insecure_cookies: false

  # Expose /debug/ (net/http/pprof DefaultServeMux). Keep false on shared nets.
  debug_endpoints: false

  # NEVER enable on a reachable network. Disables cookie checks entirely.
  # disable_auth_because_i_want_my_account_to_be_hacked: true

matrix:
  # If true, do not configure HTTP/2 on the Matrix CS API client.
  disable_http2: false

  # Presence set on sync. Default offline (does not advertise "online" via sync).
  # Matrix presence strings, e.g. offline | online | unavailable
  set_presence: offline

push:
  # Gateway used for FCM-related push plumbing (default community gateway).
  fcm_gateway: https://push.gomuks.app
  # Web Push VAPID keypair (auto-generated). Public key injected into index.html
  # and sent on WebSocket init for browser push subscription.
  vapid_private_key: "..."
  vapid_public_key: "..."

media:
  # Avatar/thumbnail edge length used by the media pipeline. Default 120.
  thumbnail_size: 120

logging:
  # go.mau.fi/zeroconfig — compiled at startup.
  # Default: debug level, pretty colored stdout + rotating JSON file under logs.
  min_level: debug
  writers:
    - type: stdout
      format: pretty-colored
    - type: file
      format: json
      filename: /path/to/logs/gomuks.log   # filled from GOMUKS_LOGS_HOME on first write of defaults
      max_size: 100
      max_backups: 10
```

### Field reference

#### `web`

| Key | Type | Default / auto | Purpose |
|-----|------|----------------|---------|
| `listen_address` | string | `localhost:29325` | `net.Listen("tcp", …)` bind |
| `username` | string | prompted (1–32 chars) | Basic auth user |
| `password_hash` | string | prompted bcrypt | Basic auth password hash |
| `token_key` | string | random 64 | Signs `gomuks_auth` tokens |
| `event_buffer_size` | int | `512` if ≤0 | In-memory event ring for WS resume |
| `origin_patterns` | []string | `localhost:*`, `*.localhost:*` | WS Origin allowlist |
| `insecure_cookies` | bool | `false` | Allow non-Secure cookies for HTTP clients |
| `debug_endpoints` | bool | `false` | Mount `/debug/` (pprof, etc.) |
| `disable_auth_because_i_want_my_account_to_be_hacked` | bool | omit/`false` | Disable API auth (dangerous) |

**Desktop mode** (`--desktop` + `GOMUKS_DESKTOP_KEY`): forces
`listen_address` default to `localhost:0` before YAML apply interaction for
logging/address defaults, logs to stderr by default, skips interactive password
setup, accepts basic auth user `desktop-key` with the env secret, and prints a
JSON line `{"started":true,"address":"..."}` when the listener is up.

#### `matrix`

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `disable_http2` | bool | `false` | Disable HTTP/2 to the homeserver |
| `set_presence` | string | `offline` | Sync presence advertised to the HS |

#### `push`

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `fcm_gateway` | string | `https://push.gomuks.app` | FCM gateway base |
| `vapid_private_key` | string | generated | Web Push signing |
| `vapid_public_key` | string | generated | Exposed to frontend |

Push from the Go server is compiled out / disabled on some targets
(`push_disabled.go` sets `DisablePush = true` for JS builds).

#### `media`

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `thumbnail_size` | int | `120` | Thumbnail generation size |

Media files are cached under `$CACHE/media/` hashed paths. Uploads/downloads go
through `/_gomuks/upload` and `/_gomuks/media/{server}/{media_id}` (cookie or
image-only token).

#### `logging`

Uses [zeroconfig](https://github.com/tulir/zeroconfig) (`go.mau.fi/zeroconfig`).
Defaults are debug + stdout pretty + file JSON with rotation (`max_size` MB,
`max_backups`). Adjust `min_level` for production (`info` / `warn`) if log volume
matters.

Note: the default **file** log path is derived from `GOMUKS_LOGS_HOME` when the
default file writer is initialized. Official FAQ notes that log path handling is
effectively captured when config is first written—if you relocate logs later,
edit `logging.writers` in `config.yaml` explicitly.

---

## Authentication model (operators)

### Layers

```text
Browser / client
    │  Basic auth  OR  existing gomuks_auth cookie
    ▼
POST /_gomuks/auth  →  sets HttpOnly cookie (7-day HMAC token)
    │
    ▼
All other /_gomuks/*  (except auth itself) require valid cookie
    │  media may use Image-only token instead
    ▼
JSON RPC  (WebSocket or POST /exec/{command})
    │
    ▼
HiClient  →  Matrix access token in SQLite
```

### Practical rules

1. **Password** protects the gomuks API only. Matrix login is separate (done in UI after API auth).
2. Cookies are **HttpOnly**, **SameSite=Lax**, **Secure** unless `insecure_cookies` (or a non-browser client opts into insecure cookies with no `Sec-Fetch-Site`).
3. From an insecure browser context without insecure cookies allowed, auth returns **403** with a clear message instead of setting a useless cookie.
4. **Image auth tokens** are short-lived tokens pushed over the WebSocket so `<img src>` can load media without the full session cookie.
5. Rotating `password_hash` or `token_key` invalidates old sessions (users re-auth). Changing `token_key` alone invalidates all cookies immediately.
6. **Disable auth only** if a reverse proxy (or network) already provides equivalent protection **and** untrusted traffic cannot reach the process.

### Non-interactive / headless first boot

Docker and systemd often have no TTY for the password prompt. Options:

1. Run once with `-it` (Docker) or an interactive shell to create credentials.  
2. Pre-create `config.yaml` with `username` + `password_hash` (generate bcrypt cost 12 yourself), plus leave `token_key` empty so it is generated on first start—or supply all secrets.  
3. Temporarily use the disable-auth flag **only** behind a locked-down proxy, then re-enable real auth.

---

## Networking and reverse proxy

### Local only (default)

```yaml
web:
  listen_address: localhost:29325
  origin_patterns:
    - "localhost:*"
    - "*.localhost:*"
  insecure_cookies: false
```

No TLS required. Do not expose the port.

### Reverse proxy + TLS (recommended remote access)

Goals:

- Terminate TLS at the proxy (Caddy, nginx, Traefik, …).
- Proxy HTTP and **WebSockets** to the backend.
- Keep gomuks auth **or** carefully replace it with proxy auth.

Example config shape:

```yaml
web:
  listen_address: 127.0.0.1:29325   # or 0.0.0.0:29325 in Docker
  origin_patterns:
    - "chat.example.com"
  insecure_cookies: false           # browsers use HTTPS → Secure cookies OK
```

Proxy requirements:

| Need | Why |
|------|-----|
| WebSocket upgrade | Primary RPC transport is `/_gomuks/websocket` |
| Long-lived connections | Sync events; avoid aggressive idle timeouts without pings |
| Large bodies | Media upload/download |
| Forward `Host` / correct public URL | Clients connect to your public host; Origin must match `origin_patterns` |

`origin_patterns` must list the host the **browser** uses (no `https://` scheme).
Include an explicit port only if the browser’s Origin includes one.

If the proxy injects its own authentication, you may either:

- Leave gomuks basic auth enabled (users auth twice, or proxy injects
  `Authorization` / cookie carefully), or  
- Disable gomuks auth with the explicit config key and **guarantee** the proxy
  never passes unauthenticated traffic.

### Plain HTTP on a LAN (discouraged)

```yaml
web:
  listen_address: 0.0.0.0:29325
  origin_patterns:
    - "192.168.1.10:29325"   # whatever you type in the address bar
    - "gomuks.local:29325"
  insecure_cookies: true
```

Without TLS, Secure cookies will not stick for remote HTTP; `insecure_cookies`
is required. Prefer WireGuard/Tailscale + TLS over this pattern. Official FAQ:
LANs are hostile; use TLS.

### CORS

The main binary sets `exhttp.AutoAllowCORS = false`. Do not rely on open CORS;
the web UI is same-origin with the backend (or desktop-controlled).

---

## Docker

Image (community/CI): `dock.mau.dev/gomuks/gomuks`  
CI Dockerfile sets:

```dockerfile
ENV GOMUKS_ROOT=/data
VOLUME /data
CMD ["/usr/bin/gomuks"]
```

Persist `/data` (config + DB + cache + logs under one root). Example:

```sh
docker run --name gomuks -it \
  -p 29325:29325 \
  -v /var/lib/gomuks:/data \
  dock.mau.dev/gomuks/gomuks
```

Then set in `/data/config/config.yaml` (paths under `GOMUKS_ROOT`):

```yaml
web:
  listen_address: 0.0.0.0:29325
  origin_patterns:
    - "your.public.host"
  # insecure_cookies only if not using HTTPS in front
```

First run needs credentials (`-it` or pre-written config). The backend holds
**encryption keys**—run it only on hosts you trust.

Image notes from CI Dockerfile: Alpine base with `ca-certificates`, `jq`,
`curl`, `ffmpeg` (video metadata). Host or image should provide `ffmpeg` /
`ffprobe` for video sends as documented upstream.

---

## HTTP surface (for operators)

Base path: `/_gomuks` (static UI is at `/`).

| Route | Auth | Notes |
|-------|------|--------|
| `POST /auth` | basic or existing cookie | Issues `gomuks_auth`; query: `output=json`, `no_prompt`, `insecure_cookie`, `secure` |
| `GET /websocket` | cookie | RPC + events; query: `compress`, `run_id`, `last_received_event` |
| `POST /exec/{command}` | cookie | One-shot RPC |
| `POST /upload` | cookie | Media upload |
| `GET /media/{server}/{media_id}` | cookie **or** image token | Download / thumbnails |
| `POST /keys/export`, import, restore | cookie | Key backup helpers |
| `GET /url_preview` | cookie | Proxied via homeserver API |
| `GET /codeblock/{style}.css` | cookie | Highlight CSS |
| `/debug/*` | if enabled | pprof-style debug; **sensitive** |

OpenAPI sketch: `pkg/gomuks/server.yaml`.  
RPC semantics: `pkg/hicli/jsoncmd/*.md` and `go run ./cmd/rpcdocgen`.

---

## Runtime and process management

### Lifecycle

1. `InitDirectories` → `LoadConfig` → `SetupLog`  
2. `StartServer` (listen + serve in background)  
3. `StartClient` (open DB, start hicli / Matrix)  
4. Block until SIGINT / SIGTERM  
5. Close WS clients with “Server shutting down”, stop client, close HTTP server  

Exit codes (indicative):

| Code | Meaning |
|------|---------|
| 0 | Clean shutdown (or logout after bad token during start) |
| 9 | Config load failure |
| 10 | Database open failure |
| 11 | Account lookup failure |
| 12 | Client start failure |
| 13 | HTTP/2 configure failure |

### systemd sketch

```ini
[Unit]
Description=gomuks Matrix client backend
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=gomuks
Group=gomuks
Environment=GOMUKS_ROOT=/var/lib/gomuks
ExecStart=/usr/local/bin/gomuks
Restart=on-failure
# Hardening examples (adjust as needed):
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Ensure the service user owns `GOMUKS_ROOT` and that first-boot auth is preconfigured.

### Resource notes

- **SQLite** file grows with history; plan disk for data + media cache.  
- **Event buffer** is RAM-only (`event_buffer_size`); resume does not survive restarts.  
- **Media cache** can be large; safe to prune.  
- Optional **ImageMagick** (`magick` in `PATH`) improves some image handling paths.  
- **libolm** (or goolm build tag) required for E2EE at compile time—not a runtime config key.

---

## Configuration scenarios

### 1. Laptop, browser on same machine

Defaults are enough. Run `gomuks`, open `http://localhost:29325`.

### 2. Always-on home server + browser on phone (Tailscale)

- Run gomuks on the home host bound to Tailscale IP or localhost + proxy.  
- Prefer HTTPS reverse proxy on the mesh hostname.  
- Set `origin_patterns` to that hostname.  
- Keep auth enabled; use a strong password.

### 3. Docker on VPS + Caddy

- Mount volume on `/data`.  
- `listen_address: 0.0.0.0:29325` (or publish only to Caddy’s network).  
- Caddy obtains TLS; proxy `/` and WebSockets to the container.  
- `origin_patterns: ["gomuks.example.com"]`.  
- `insecure_cookies: false`.

### 4. Proxy handles SSO; gomuks on private network only

- Bind `127.0.0.1` only.  
- Proxy requires SSO; either inject credentials or use disable-auth **only** if the listen address is not reachable unauthenticated.  
- Document this setup carefully; misconfiguration is account compromise.

### 5. Two Matrix accounts on one machine

```sh
GOMUKS_ROOT=~/gomuks-personal gomuks &   # listen 29325
GOMUKS_ROOT=~/gomuks-work     # edit config listen_address :29326
```

Use `http://personal.localhost:29325` vs `http://work.localhost:29326` if not
using separate reverse proxy paths.

---

## Security checklist

| Check | Recommendation |
|-------|----------------|
| Bind address | Prefer loopback or private network; never open `0.0.0.0` without TLS + auth |
| Auth | Strong unique password; leave disable-auth off |
| TLS | Reverse proxy for any remote browser |
| `origin_patterns` | Exact public hosts only; no overly broad wildcards you do not need |
| `debug_endpoints` | Off in production |
| `config.yaml` mode | `0600`; directory `0700` |
| Backups | Back up **data** dir (keys!); treat like a password manager export |
| Host trust | Backend holds Megolm keys—same threat model as any E2EE client device |
| Updates | Keep binary updated; frontend ETag reload helps when UI is embedded |
| pprof / debug | Do not expose `/debug/` |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Password prompt then exit in Docker | No TTY | `docker run -it` or pre-write `config.yaml` |
| Cookie “doesn’t stick” on `http://LAN-IP` | Secure cookie on insecure context | Use HTTPS, or set `insecure_cookies: true` (with eyes open) |
| WebSocket fails / Origin error | `origin_patterns` mismatch | Add exact browser host (no scheme) |
| 401 on API | Missing/expired cookie | Re-run `/auth` or re-login UI |
| 403 on `/auth` | Insecure context + secure cookies required | TLS or `insecure_cookies` |
| Cannot reach from another host | Still on `localhost` bind | Change `listen_address` |
| Logged out after restore | Restored config without data (or vice versa) | Restore **data** and config together |
| Huge disk use | Media cache | Clear `$CACHE/media` |
| Video upload metadata fails | No ffmpeg | Install `ffmpeg`/`ffprobe` on PATH |
| Push not working | Wrong VAPID / gateway / HTTPS requirements | Check `push` keys; browser needs secure context for Web Push |

---

## Related references

| Resource | Content |
|----------|---------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Layers, RPC, assessment |
| [docs.mau.fi/gomuks](https://docs.mau.fi/gomuks/) | Install, FAQ (remote backend, multi-account, storage) |
| `pkg/gomuks/config.go` | Canonical config struct and defaults |
| `pkg/gomuks/server.go` | HTTP routes and auth implementation |
| `pkg/gomuks/server.yaml` | OpenAPI-ish HTTP description |
| `pkg/hicli/jsoncmd/websocket.md` | WS resume, ping, compression |
| `Dockerfile.ci` | Container defaults (`GOMUKS_ROOT=/data`) |

---

## Document history

- Initial version: server configuration guide derived from `pkg/gomuks` config
  loading, directory layout, auth/network behavior, Docker CI image, and
  alignment with the public gomuks FAQ.
