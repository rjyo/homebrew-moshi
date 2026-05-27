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
  | coding     |  Claude / Codex / OpenCode
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
  "source": "claude",                    // "claude" | "codex" | "opencode"
  "sessionId": "abc-123",
  "actionId": "act_01HXY…",              // correlates request/response

  // approval.request context
  "eventName": "PreToolUse",
  "phase": "pre_tool",
  "category": "shell",
  "cwd": "/Users/jyo/projects/foo",
  "projectName": "foo",
  "terminalKind": "zellij",              // "tmux" | "zellij"
  "zellijSession": "main",
  "zellijPane": "terminal_1",
  "tmuxSession": "main",                 // present when terminalKind is "tmux"
  "tmuxWindow": "0",
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
| Hook → daemon | `approval.request` | Block until daemon returns a decision. |
| Hook → daemon | `session.update` | Notify daemon of session state change. |
| Hook → daemon | `session.closed` | Session ended. |
| Daemon → hook | `approval.response` | Decision for prior `approval.request` (matched by `actionId`). |
| Daemon → hook | `ack` | Ack of a fire-and-forget message. |
| Daemon → hook | `error` | Protocol/transport error. |

### Example

```jsonc
// → daemon
{"type":"approval.request","source":"claude","sessionId":"s1","actionId":"act_1",
 "category":"shell","toolName":"Bash","title":"Run shell","message":"rm -rf …",
 "expiresAt":"2026-04-26T12:01:00Z","requestedAt":"2026-04-26T12:00:00Z"}

// ← daemon
{"type":"approval.response","actionId":"act_1","accepted":true,"decision":"approve"}
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
{"hostId":"host_existing","displayName":"jyo-mbp","platform":"macos"}
// hostId optional — pass to rotate secret; omit on first pair.
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
| Daemon → server | `approval.request` | Forwarded from a hook. |
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

### `POST /v1/servers/kill`

Terminates a discovered local HTTP server. The daemon re-runs server discovery and only signals a process whose current PID and port match the request; arbitrary PIDs are rejected. By default callers should send `force: true`, which sends `SIGTERM` first and falls back to `SIGKILL` if the process does not exit within the grace period.

```jsonc
// request
{ "host": "127.0.0.1", "port": 5173, "pid": 27753, "force": true }

// response
{ "killed": true, "forced": false, "pid": 27753, "port": 5173, "server": { /* discovered server */ } }
```

---

## 5. CLI JSON (SSH preflight)

Moshi clients use SSH exec/preflight for host inspection commands. These commands print JSON to stdout and do not require host pairing or a bearer token. There is no separate capabilities manifest; clients should run the specific command they need and handle command failure as unsupported/unavailable.

### `moshi-hook servers [--ssh-connection "..."] [--mosh-port <p> [--mosh-host <ip>]]`

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

`isCurrentContext` is always `false` for the context-less global server list. When a session lookup is supplied, the CLI mirrors the `/events` WebSocket decoration: `true` means Moshi could attribute the listener to the current shell context or the current tmux session.

Transport: **same-port forwarding only.** Clients are expected to open an SSH local forward `phone:<port> → host:<port>` for each origin and load `http://localhost:<port>` in the WebView, matching the host URL exactly. The gateway does not implement a path-prefix reverse proxy (`/proxy/http/...`) and will not — path-prefix proxying breaks HMR (absolute WebSocket paths), OAuth (`Origin` / redirect URI), and `SameSite` cookies. If the phone hits a local port collision, surface an error to the user; do not rewrite URLs.

### `moshi-hook servers kill --pid <pid> --port <port> [--host <host>] [--force=false]`

Terminates a discovered server after re-validating that the PID and port still belong to a surfaced HTTP server. The JSON response matches `POST /v1/servers/kill`.

### `moshi-hook context [--ssh-connection "..."] [--mosh-port <p> [--mosh-host <ip>]]`

Returns the current terminal state for an iOS-owned SSH or Mosh session: tmux pane (if the user has tmux attached on the session's TTY), zellij pane when detected from the shell environment, or bare shell. Tmux detection is live — attaching or detaching tmux changes the next response immediately.

Remote-session flags (set `--ssh-connection` or `--mosh-port`; `--mosh-host` only applies with `--mosh-port`):

| param | value |
|---|---|
| `ssh-connection` | Verbatim `$SSH_CONNECTION` from inside the session (`"<client_ip> <client_port> <server_ip> <server_port>"`). iOS captures this once via ssh-exec right after the session opens. |
| `mosh-port` | Server-side UDP port that `mosh-server` is listening on for the session. iOS already knows it from the `MOSH CONNECT <port> <key>` handshake. |
| `mosh-host` | Optional disambiguation hint for the server-side bind address. It is only needed when two mosh-servers share the same port on different interfaces (e.g. one over Tailscale, one over LAN). If the hint does not match but the port has only one local binding, the daemon uses that binding. Without a matching hint, ambiguous lookups fail explicitly rather than returning a guessed session. |

Tmux response:

```json
{
  "kind": "tmux",
  "tmux": { "session": "work", "window": "2", "pane": "%7" },
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

Shell response (no multiplexer detected):

```json
{
  "kind": "shell",
  "cwd": "/Users/me/projects/foo",
  "git": { "repo": "/Users/me/projects/foo", "branch": "main", "dirty": true }
}
```

Resolution: the daemon finds the session's login shell (env-walk for SSH, UDP-port-owner for Mosh), reads its controlling TTY, and asks `tmux list-clients` whether anything is attached. If yes, returns that session's active pane via `tmux display-message`. If no, reads the shell's cwd directly (`/proc/<pid>/cwd` on Linux, `lsof -d cwd` on macOS) and resolves Git state.

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
