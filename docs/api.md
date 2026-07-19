# `moshi-hook` API Reference

Wire protocols `moshi-hook` participates in. Five surfaces:

1. **Local socket** — hooks ↔ daemon. Newline-delimited JSON over Unix socket.
2. **Moshi HTTP** — daemon ↔ Moshi server. JSON over HTTPS, bearer auth.
3. **Moshi WebSocket** — daemon ↔ Moshi server. JSON frames, bearer on upgrade.
4. **Host gateway HTTP** — Moshi app ↔ `moshi-hook serve` over an SSH local forward. Diff viewer JSON/static HTTP on localhost, no bearer auth.
5. **CLI JSON** — clients ↔ CLI subcommands over SSH preflight. Stdout JSON for server discovery, terminal context, and cwd-list.

## How the pieces fit

Approval round-trip across all four components. Numbers reference the steps below.

```
  user's host                                 cloud                user's phone

  +------------+
  | coding     |  Claude / Codex / OpenCode / Pi / OMP / Kimi
  | agent      |
  +-----+------+
        | (1) hook fires; agent spawns short-lived subprocess
        v
  +------------+
  | moshi-hook |  one process per hook invocation
  | <subcmd>   |  (claude-hook, codex-hook, opencode-permission, ...)
  +-----+------+
        | (2) approval.request, newline JSON over Unix socket
        v
  +------------+   (3) approval.request           +-------------+
  | moshi-hook | ----------- WSS ---------------> |  Moshi      |
  | serve      |   bearer hostSecret              |  server     | ---+
  | (daemon)   |                                  |             |    | (4) push
  |            |   (6) approval.decision          |             |    |
  |            | <----------- WSS --------------- |             | <--+
  +-----+------+                                  +------+------+    |
        | (7) approval.response                          ^           v
        |     back over Unix socket                      |    +-----------+
        v                                                |    | iPhone /  |
  +------------+                                         |    | Mac app   |
  | moshi-hook |  (8) writes decision JSON to stdout,    |    +-----+-----+
  | <subcmd>   |      exits; agent unblocks (or blocks)  |          |
  +------------+                                         +----------+
                                                          (5) user taps
                                                              Approve / Deny;
                                                              decision POSTed
                                                              to server (HTTPS)
```

- **Steps 1–2**: process boundary on the local host. Section [1. Local socket](#1-local-socket).
- **Steps 3 & 6**: long-lived bidirectional WebSocket. Section [3. Moshi WebSocket bridge](#3-moshi-websocket-bridge).
- **Step 5**: the iPhone / Mac client posts the decision back to the server. Out of scope for this document — see the Moshi server reference.
- **Steps 4, 5–6**: notification fan-out and decision routing happen on the server side; `moshi-hook` only sees them as inbound WS frames.

A daemon that's never told to forward (no pending approval) still maintains the WebSocket so it can receive other server-initiated frames (session subscriptions, future remote-trigger events).

---

## 1. Local socket

- **Address**: see [paths](usage.md#paths). Override with `MOSHI_SOCKET_PATH`.
- **Framing**: newline-delimited JSON. One object per line.
- **Auth**: filesystem (`chmod 0600`). Owner-only.
- **Lifetime**: one logical exchange per connection.

### Envelope

Every frame uses the same shape. Fields are optional, interpreted per `type`. Unknown fields are ignored.

```jsonc
{
  "type": "approval.request",
  "source": "opencode",                  // blocking approval adapters
  "sessionId": "abc-123",
  "actionId": "act_01HXY…",              // correlates request/response

  // approval.request context
  "eventName": "PreToolUse",
  "phase": "pre_tool",
  "category": "shell",
  "cwd": "/Users/jyo/projects/foo",
  "projectName": "foo",
  "terminalKind": "zellij",              // "tmux" | "zellij" | "herdr"
  "zellijSession": "main",
  "zellijPane": "terminal_1",
  "herdrSession": "main",
  "herdrPane": "w6533c030139461-1",
  "tmuxSession": "main",                 // present when terminalKind is "tmux"
  "tmuxWindow": "0",
  "tmuxPane": "%7",
  "toolName": "Bash",
  "modelName": "claude-opus-4-7",
  "contextPercent": 42,                  // 0..100, 0 = unknown
  "title": "Run shell command",
  "subtitle": "rm -rf node_modules",
  "message": "claude wants to run …",
  "expiresAt": "2026-04-26T12:01:00Z",
  "requestedAt": "2026-04-26T12:00:00Z",

  // approval.response
  "accepted": true,
  "decision": "approve",                 // "approve" | "deny"
  "reason": "user accepted on iPhone",

  // error
  "error": "daemon shutting down"
}
```

### Message types

| Direction | `type` | Purpose |
|---|---|---|
| Hook → daemon | `approval.request` | Block until daemon returns a decision. Used by blocking approval adapters. |
| Hook → daemon | `session.update` | Notify daemon of session state change. Claude/Codex/Hermes terminal approvals use this with `category:"approval_required"` and, when a terminal target is available, `actionId` + `phase:"waitingForApproval"`. |
| Hook → daemon | `session.closed` | Session ended. |
| Daemon → hook | `approval.response` | Decision for prior `approval.request` (matched by `actionId`). |
| Daemon → hook | `ack` | Ack of a fire-and-forget message. |
| Daemon → hook | `error` | Protocol/transport error. |

### Example

```jsonc
// → daemon
{"type":"session.update","source":"claude","sessionId":"s1","actionId":"act_1",
 "phase":"waitingForApproval","category":"approval_required","toolName":"Bash",
 "title":"Run shell","message":"rm -rf …","terminalKind":"tmux","tmuxPane":"%7",
 "requestedAt":"2026-04-26T12:00:00Z"}

// ← daemon
{"type":"ack","sessionId":"s1"}
```

---

## 2. Moshi HTTP API

Base URL: `https://api.getmoshi.app/api/v1` (override with `--base-url` / `MOSHI_API_BASE`). Auth: `Authorization: Bearer <hostSecret>` on daemon runtime endpoints.

### `POST /setup/host` -> `GET /setup/host/:setupId/wait`

Easy Pair creates a short-lived setup session. The phone claims it with the user's app token and public SSH key. The host polls `wait` with the setup secret; after claim, the response includes the public key plus the host-scoped `hostSecret`.

```jsonc
{"hostId":"host_aBcD…","hostSecret":"secret_…","displayName":"jyo-mbp",
 "publicKey":"ssh-ed25519 …","publicKeyFingerprint":"SHA256:…"}
```

The hook stores `hostId`, `hostSecret`, and `displayName`; it does not store the phone's user token.

### `POST /hosts/register`

Manual daemon pairing/re-pairing. Authenticated with the **pairing token**.

```jsonc
// request
{"hostId":"host_existing","hostSecret":"secret_existing","displayName":"jyo-mbp","platform":"macos"}
// hostId optional — pass to rotate secret; omit on first pair.
// hostSecret optional — pass with hostId to prove ownership when re-attaching
// the host to a new pairing token/license after subscription changes.
// platform: "macos" | "linux" | "other"

// response
{"hostId":"host_aBcD…","hostSecret":"hs_…","displayName":"jyo-mbp",
 "platform":"macos","registeredAt":"2026-04-26T12:00:00Z"}
```

`hostSecret` is shown once; the daemon stores it in the platform secret store.

### `POST /hosts/:hostId/events`

Publish an agent event.

```jsonc
{
  "eventId": "evt_01HXY…",
  "source": "claude",
  "eventType": "pre_tool",
  "sessionId": "s1",
  "category": "shell",
  "title": "Run shell command",
  "message": "rm -rf node_modules",
  "projectName": "foo",
  "tmuxSession": "main",
  "tmuxWindow": "0",
  "modelName": "claude-opus-4-7",
  "toolName": "Bash",
  "pendingActionId": "act_…",            // present when a decision is required
  "expiresAt": "2026-04-26T12:01:00Z",
  "contextPercent": 42,
  "accountId": "claude:plan_max"         // <source>:<id>; omit if signed out
}
```

Returns `200 OK` with `{}`. `hostId` in the URL is authoritative.

### `POST /hosts/:hostId/usage`

Push usage snapshots.

```json
{
  "snapshots": [{
    "accountId": "claude:plan_max",
    "accountLabel": "Claude Max",
    "agent": "claude",
    "hostName": "jyo-mbp",
    "capturedAt": "2026-04-26T12:00:00Z",
    "windows": [{"label":"5h","usedPercentage":42.0,"resetsAt":"2026-04-26T17:00:00Z"}]
  }]
}
```

Returns the sync count plus the server's current view of this host's license
attachment. The daemon prints this so users can distinguish "usage synced" from
"the app is reading a different license bucket".

```jsonc
{
  "success": true,
  "count": 1,
  "host": {
    "hostId": "host_aBcD…",
    "displayName": "jyo-mbp",
    "premiumAttached": true,
    "licenseAttached": true,
    "licenseUsable": true,
    "licenseStatus": "active",
    "usageScope": "license",              // "license" | "direct"
    "reason": "license_active",
    "message": "This host is attached to an active Moshi Pro license."
  }
}
```

### `GET /hosts/:hostId/status`

Host-secret authenticated status check used by `moshi-hook status`.

```jsonc
{
  "hostId": "host_aBcD…",
  "displayName": "jyo-mbp",
  "premiumAttached": false,
  "licenseAttached": false,
  "licenseUsable": false,
  "usageScope": "direct",
  "reason": "no_license",
  "message": "This host is not attached to a Moshi Pro license. Usage sync is private to the paired device."
}
```

### Errors

```json
{ "error": "human-readable message" }
```

---

## 3. Moshi WebSocket bridge

A long-lived bidirectional link from the daemon to the Moshi server, scoped to one host.

```
GET wss://api.getmoshi.app/api/v1/hosts/<hostId>/connect
Authorization: Bearer <hostSecret>
```

The bearer is validated on the upgrade itself — no separate mint round-trip. Reconnects with bounded exponential backoff (1s → 30s); the secret is re-read from the keystore on each attempt, so re-pairing is picked up without restart.

### Frames

Both directions share one shape:

```jsonc
{
  "type": "approval.request",
  "hostId": "host_aBcD…",
  "actionId": "act_01HXY…",
  "decision": "approve",                 // "approve" | "deny"
  "title": "Run shell command",
  "message": "rm -rf node_modules",
  "expiresAt": "2026-04-26T12:01:00Z",
  "requestedAt": "2026-04-26T12:00:00Z"
}
```

### Frame types

| Direction | `type` | Purpose |
|---|---|---|
| Daemon → server | `hello` | Sent immediately after connect. |
| Daemon → server | `approval.request` / `pending-action.open` | Forwarded from a blocking hook or opened by the TUI bridge. |
| Server → daemon | `approval.decision` | `actionId` + `decision` + optional `reason`. |
| Server → daemon | `ping` | Keepalive. |
| Daemon → server | `pong` | Reply to `ping`. |

Unknown types are logged and ignored — receivers must be lenient so the server can ship new frames ahead of daemon upgrades.

---

## 4. Host Gateway HTTP

`moshi-hook serve` starts a localhost-only diff gateway in the same daemon process as the Unix socket and WebSocket bridge.

Listen-address precedence:

1. `moshi-hook serve --gateway-listen 127.0.0.1:24543`
2. `MOSHI_HOOK_GATEWAY_LISTEN=127.0.0.1:24543`
3. `~/.config/moshi-hook/config.toml`

```toml
[gateway]
listen = "127.0.0.1:24543"
```

4. Default `127.0.0.1:24543`

Gateway HTTP is loopback-only and does not require `Authorization`. Clients reach it through SSH local forwarding. Host/account auth remains limited to daemon-to-Moshi server calls that deliver pushes, usage, approvals, and WebSocket frames.

The HTTP gateway serves diff and bounded control actions for servers surfaced by discovery. Pull-only host inspection is intentionally not exposed here: `GET /v1/capabilities`, `GET /v1/servers`, and `GET /v1/sessions/context` return `404`. Use the SSH preflight CLI commands below instead.

### `POST /v1/diff/start`

Starts or reuses an embedded diff viewer session for a Git repository.

```jsonc
// request
{ "cwd": "/Users/me/projects/foo" }

// response
{ "diffSessionId": "diff_abc123", "url": "/apps/diff/diff_abc123/" }
```

Diff sessions expire after 15 minutes idle and are served under `/apps/diff/:sessionId/`. Diff payloads are read from the host filesystem and never sent to the Moshi backend.

### `GET /v1/transcripts?session=<id>[&source=claude|codex|opencode|pi|omp|kimi]`

Opens a local WebSocket stream for a live agent transcript. New clients should pass `source`; when omitted for backward compatibility, the gateway tries Claude first, then Codex. Claude transcripts prefer the exact per-session path captured from hook events (including `CLAUDE_CONFIG_DIR` profiles), then fall back to `~/.claude/projects` for older session state. Codex transcripts are resolved from `$CODEX_HOME/sessions` or `~/.codex/sessions` rollout files. Pi and OMP transcripts use the exact JSONL path reported by the installed extension, so profiles and custom session locations work without a directory scan. OMP validation understands its v3 fixed-width title slot before the session header. Kimi transcripts resolve through its profile-aware `session_index.jsonl` and stream the main agent's live `wire.jsonl`. OpenCode is proxied through the live local server recorded by its plugin. Transcript bytes stay on the host and are streamed only over the local forwarded gateway. If Codex resume creates a newer rollout for the same session id, reconnect to resolve the newest file.

Server messages are JSON objects with `type` (`backlog`, `older`, `append`, `reset`, or `error`), `source`, physical `line` numbers, and raw JSONL rows for client-side rendering. Clients can request older rows with `{"type":"older","beforeLine":123,"limit":50}`.

Oversized rows are redacted before streaming: long strings are truncated, and inline image payloads (Claude `source.data`, Pi/OMP `data`, Codex `image_url` data URLs) are replaced with a stub carrying `truncated: true`, `media_type`, decoded `bytes`, and `width`/`height` when the format is recognized. Clients fetch the actual bytes via the blob endpoint below.

### `GET /v1/transcripts/blob?session=<id>&line=<n>[&block=<i>][&source=claude|codex|opencode|pi|omp]`

Serves the raw image bytes of one content block of one transcript line, re-read from disk or re-fetched from OpenCode on demand (so redaction never loses data). `line` is the physical transcript line index reported by the stream; `block` (default 0) indexes `message.content[i]` for Claude/Pi/OMP, `payload.content[i]` / `payload.output[i]` for Codex, or the flattened OpenCode image attachments. OMP `blob:sha256:` image references resolve through the profile/XDG-aware `blobs/` directory beside its managed `sessions/` tree. For Codex `view_image` function calls the endpoint resolves the call's absolute file `path` on the host and serves the file when it sniffs as an image (capped at 32 MB). Responds with the image `Content-Type` and cache headers; returns 404 when the addressed block is not an image.

### `POST /v1/servers/kill`

Terminates a discovered local HTTP server. The daemon re-runs server discovery and only signals a process whose current PID and port match the request; arbitrary PIDs are rejected. By default callers should send `force: true`, which sends `SIGTERM` first and falls back to `SIGKILL` if the process does not exit within the grace period.

```jsonc
// request
{ "host": "127.0.0.1", "port": 5173, "pid": 27753, "force": true }

// response
{ "killed": true, "forced": false, "pid": 27753, "port": 5173, "server": { /* discovered server */ } }
```

### `POST /v1/paste?<session lookup>`

Injects an image into the caller's multiplexer pane: the tmux pane, the focused herdr pane, or the zellij pane (falling back to the session's focused pane when the context carries no pane id, since zellij focused-pane actions silently no-op without an attached client). Takes the same session-lookup query params as `/events` (`ssh-connection`, `mosh-port`[+`mosh-host`], or `et-client-id`); the pane is resolved live on the host, so the app never passes (possibly stale) pane ids. Bodies are capped at 64 MB.

```jsonc
// request
{ "data": "<base64 image bytes>", "mimeType": "image/png" }

// response
{ "ok": true, "mode": "clipboard", "verified": true, "path": "/tmp/moshi-paste-123.png" }
```

The daemon writes the image to `$TMPDIR/moshi-paste-*` (stale files are swept after 24h) and picks a mode:

- `clipboard` — when the pane runs an agent and a clipboard is reachable (macOS GUI session via `osascript`, Wayland via `wl-copy`, X11 via `xclip`; display env is read from the tmux session/global environment since the daemon itself has none — zellij and herdr have no queryable session env, so those targets use the daemon's env), seed the OS clipboard and send Ctrl+V so the agent picks up the image inline. The key grammar differs per multiplexer: `C-v` (tmux send-keys), `ctrl+v` (`herdr pane send-keys`), `"Ctrl v"` (`zellij action send-keys`) — each rejects the others' tokens.
- `path` — otherwise (headless hosts, plain shell panes), type the temp-file path literally into the pane (`tmux send-keys -l` / `herdr pane send-text` / `zellij action write-chars`).

`verified` reports whether the pane's content fingerprint changed after injection (tmux `capture-pane` + cursor, `herdr pane read`, `zellij action dump-screen`; polled up to 1.5s). An unverified Ctrl+V that provably changed nothing falls back to path injection within the same request. Errors: `404` when no live session matches the lookup, `422` when the session is a bare shell or the pane/session can't be identified (callers should fall back to their SSH paste path).

---

## 5. CLI JSON (SSH preflight)

Moshi clients use SSH exec/preflight for host inspection commands. These commands print JSON to stdout and do not require host pairing or a bearer token. There is no separate capabilities manifest; clients should run the specific command they need and handle command failure as unsupported/unavailable.

### `moshi-hook servers [--ssh-connection "..."] [--mosh-port <p> [--mosh-host <ip>]] [--et-client-id <id>|--et]`

Discovers listening loopback HTTP services and returns each origin as the host sees it.

```json
{
  "servers": [{
    "id": "server_1",
    "name": "Vite",
    "host": "127.0.0.1",
    "port": 5173,
    "origin": "http://127.0.0.1:5173",
    "process": "node",
    "pid": 27753,
    "isCurrentContext": false
  }]
}
```

Filtering: only responses whose `Content-Type` is `text/html` (or `application/xhtml+xml`) are surfaced. AirTunes, proxy admin UIs, and JSON-only APIs are dropped — they're not openable in a WebView.

Container servers: ports published from Docker/OrbStack containers are discovered via `docker inspect` and surfaced with `source: "docker"`, `process: "docker:<container>"`, and no `pid` (the host-side listener is a forwarding proxy, not the dev server; `cwd`/`git` are resolved from the container's bind-mounted working dir). Servers without a `pid` are not killable — `POST /v1/servers/kill` requires a PID — so clients should hide the kill affordance when `pid` is absent and can use `source` to badge the entry as a container.

`isCurrentContext` is always `false` for the context-less global server list. When a session lookup is supplied, the CLI mirrors the `/events` WebSocket decoration: `true` means Moshi could attribute the listener to the current shell context or the current tmux session.

Transport: **same-port forwarding only.** Clients are expected to open an SSH local forward `phone:<port> → host:<port>` for each origin and load `http://localhost:<port>` in the WebView, matching the host URL exactly. The gateway does not implement a path-prefix reverse proxy (`/proxy/http/...`) and will not — path-prefix proxying breaks HMR (absolute WebSocket paths), OAuth (`Origin` / redirect URI), and `SameSite` cookies. If the phone hits a local port collision, surface an error to the user; do not rewrite URLs.

### `moshi-hook servers kill --pid <pid> --port <port> [--host <host>] [--force=false]`

Terminates a discovered server after re-validating that the PID and port still belong to a surfaced HTTP server. The JSON response matches `POST /v1/servers/kill`.

### `moshi-hook context [--ssh-connection "..."] [--mosh-port <p> [--mosh-host <ip>]] [--et-client-id <id>|--et]`

Returns the current terminal state for an iOS-owned SSH, Mosh, or Eternal Terminal session: tmux pane (if the user has tmux attached on the session's TTY), zellij pane when detected from the shell environment, or bare shell. Tmux detection is live — attaching or detaching tmux changes the next response immediately.

Remote-session flags (set exactly one identifier; `--mosh-host` only applies with `--mosh-port`):

| param | value |
|---|---|
| `ssh-connection` | Verbatim `$SSH_CONNECTION` from inside the session (`"<client_ip> <client_port> <server_ip> <server_port>"`). iOS captures this once via ssh-exec right after the session opens. |
| `mosh-port` | Server-side UDP port that `mosh-server` is listening on for the session. iOS already knows it from the `MOSH CONNECT <port> <key>` handshake. |
| `mosh-host` | Optional disambiguation hint for the server-side bind address. It is only needed when two mosh-servers share the same port on different interfaces (e.g. one over Tailscale, one over LAN). If the hint does not match but the port has only one local binding, the daemon uses that binding. Without a matching hint, ambiguous lookups fail explicitly rather than returning a guessed session. |
| `et-client-id` | Eternal Terminal's 16-character client id from the ET handshake. ET uses a shared `etserver`, so the daemon resolves the per-session `etterminal` process by client id. |
| `et` | Eternal Terminal fallback for manual smoke tests. Only succeeds when exactly one `etterminal` process is visible; otherwise use `et-client-id`. |

Tmux response:

```json
{
  "kind": "tmux",
  "tmux": {
    "session": "work",
    "window": "2",
    "pane": "%7",
    "copyMode": true,
    "scrollPosition": 42,
    "historySize": 900
  },
  "cwd": "/Users/me/projects/foo",
  "git": { "repo": "/Users/me/projects/foo", "branch": "main", "dirty": true }
}
```

Zellij response:

```json
{
  "kind": "zellij",
  "zellij": { "session": "work", "pane": "terminal_7" },
  "cwd": "/Users/me/projects/foo",
  "git": { "repo": "/Users/me/projects/foo", "branch": "main", "dirty": true }
}
```

Herdr response (scroll fields mirror tmux; `copyMode` is true while the pane is scrolled back):

```json
{
  "kind": "herdr",
  "herdr": {
    "session": "work/api",
    "rawSession": "work",
    "paneId": "w-api-2",
    "workspaceId": "w-api",
    "tabId": "w-api:2",
    "tab": "codex",
    "copyMode": true,
    "scrollPosition": 42,
    "historySize": 900
  },
  "cwd": "/Users/me/projects/foo"
}
```

Shell response (no multiplexer detected):

```json
{
  "kind": "shell",
  "cwd": "/Users/me/projects/foo",
  "git": { "repo": "/Users/me/projects/foo", "branch": "main", "dirty": true }
}
```

Resolution: the daemon finds the session's login shell (env-walk for SSH, UDP-port-owner for Mosh, `etterminal` child shell for ET), reads its controlling TTY, and asks `tmux list-clients` whether anything is attached. If yes, returns that session's active pane via `tmux display-message`. If no, reads the shell's cwd directly (`/proc/<pid>/cwd` on Linux, `lsof -d cwd` on macOS) and resolves Git state.

### `moshi-hook cwd-list --json`

`moshi-hook cwd-list --json` prints a deduped, recency-ranked list of recent project working directories scraped from local agent state. The Moshi iOS app calls this during connection preflight so the picker can offer one-tap "open recent project" entries when no tmux/zellij session exists. See [usage.md](usage.md#cwd-list--recent-project-directories) for the human-readable default output and the list of agents scanned.

```jsonc
[
  {
    "cwd": "/Users/jyo/projects/ai/moshi/app-ios",
    "sources": ["claude", "codex"],     // ordered by per-source mtime desc
    "lastUsed": 1778645115              // max across sources, unix seconds
  },
  // …
]
```

Always a JSON array — empty (`[]`) when nothing is found. `sources` lists every agent that touched this cwd, with the most recently active source first; clients can render a primary icon + "+N more" treatment from that order. `lastUsed` is the freshest mtime across all sightings, used for the top-level recency sort. Non-existent paths are filtered out before output, so consumers don't need to re-`stat` each entry.

Flags:

| Flag | Default | Notes |
|---|---|---|
| `--json` | off | Emit JSON instead of the default tabular text. |
| `--limit N` | 10 | Maximum entries returned after dedup. |

Read-only and side-effect-free. Errors from individual sources are swallowed so a single unreadable agent dir can't blank the list.
