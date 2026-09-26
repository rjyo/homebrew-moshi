# `moshi-hook` API Reference

Wire protocols `moshi-hook` participates in. Five surfaces:

1. **Local socket** — hooks ↔ daemon. Newline-delimited JSON over Unix socket.
2. **Moshi HTTP** — daemon ↔ Moshi server. JSON over HTTPS, bearer auth.
3. **Moshi WebSocket** — daemon ↔ Moshi server. JSON frames, bearer on upgrade.
4. **Host gateway HTTP** — Moshi app ↔ `moshi-hook serve` over an SSH local forward. Diff viewer JSON/static HTTP on localhost, no bearer auth.
5. **CLI JSON** — clients ↔ CLI subcommands over SSH preflight. Stdout JSON for server discovery, terminal context, and cwd-list.

## Transport doctrine

Use HTTP for bounded request/response operations. Use WebSockets for state or
ordered data that changes over time and must reach the client without polling.

| Operation | Transport | Contract |
|---|---|---|
| Read one snapshot, query, page, or blob | HTTP `GET` | One request produces one bounded response. Reads may be cached or retried when their resource semantics allow it. |
| Perform a user-initiated mutation | HTTP `POST` | One request produces an acknowledgement or a typed HTTP error. Non-idempotent commands are not retried automatically. |
| Observe live state | WebSocket | The server publishes initial state and subsequent changes. This stream is the source of truth; do not poll an equivalent HTTP endpoint. |
| Follow ordered or append-only data | WebSocket | One subscription produces multiple ordered results over its lifetime. |
| Subscribe, unsubscribe, or page an existing stream | WebSocket control frame | The message controls that socket's existing stream; it is not a general RPC mechanism. |

Commands and their resulting state deliberately use different transports. For
example, `POST /v1/keys` returning `ok` proves that the bounded Escape command
was delivered, not that the agent stopped. The `/events` WebSocket publishes
the authoritative later `agentStatus` transition. Clients must not infer the
state transition from the POST acknowledgement.

Do not use a WebSocket as a general request/response bus. If an operation would
need newly invented request IDs, status codes, timeouts, and retry rules, it
belongs on HTTP. Conversely, do not add HTTP polling for state already
published by `/events` or another live stream; parallel transports create two
sources of truth and race during session changes.

The following narrow cases are intentional:

- WebSocket endpoints appear as `GET` in this reference because WebSocket starts
  as an HTTP Upgrade request. After upgrade they follow the streaming contract,
  not the HTTP read contract.
- `GET /setup/host/:setupId/wait` is bootstrap polling before the host has the
  identity and secret needed to open its long-lived WebSocket. Its setup session
  is short-lived and it is not an alternative source of live host state.
- `/events` watch messages and `/v1/transcripts` `older` messages configure or
  page an already-open stream. New user mutations still use bounded HTTP POSTs.
- The cloud approval bridge carries a blocking hook request and its asynchronous
  decision over the already-required host WebSocket. The phone's user command
  remains an HTTP POST; the socket routes the resulting server-initiated
  decision to the waiting daemon.

## How the pieces fit

Approval round-trip across all four components. Numbers reference the steps below.

```
  user's host                                 cloud                user's phone

  +------------+
  | coding     |  Claude / Codex / Grok / OpenCode / Pi / OMP / Kimi
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

Host-secret authenticated status check used by `moshi-hook status` (human/TTY output only — `status --json` answers entirely from local state so high-frequency programmatic probes never hit the API).

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

`moshi-hook serve` starts a localhost-only API/diff gateway in the same daemon process as the Unix socket and WebSocket bridge. It does not expose the general web UI; bare `moshi` runs that UI as a separate foreground process on `127.0.0.1:24544` and proxies its API traffic here.

Listen-address precedence:

1. `moshi-hook serve --gateway-listen 127.0.0.1:24543`
2. `MOSHI_HOOK_GATEWAY_LISTEN=127.0.0.1:24543`
3. `~/.config/moshi/config.toml`

```toml
[gateway]
listen = "127.0.0.1:24543"
# Discovery keeps running on a 45-second cadence when no gateway client is
# attached, so the app can list servers without a terminal session. On by
# default; set false to opt out. Prefer the CLI over hand-editing:
#   moshi-hook set always-on-discovery off
always_on_discovery = true
# Background usage collection polls Claude/Codex/Kimi/Grok/Antigravity
# rate-limit APIs and uploads snapshots to Moshi. On by default; set false to
# opt out, or set a duration (minimum 1m) through the same CLI setting:
#   moshi-hook set usage-collection off
#   moshi-hook set usage-collection 5m
# Manual `moshi-hook usage` still works when this is off.
usage_collection = true
usage_poll_interval = "5m"
# Fresh installs leave the two keys above unset until `moshi-hook set
# --first-run` (install.sh) or an interactive setup/serve/pair/install
# command. Homebrew users keep defaults until one of those runs.
#
# An agent launched by another agent (a Codex started from inside a Claude
# session, say) notifies as if you had started it yourself. Set true to
# silence those. Off by default, and deliberately not part of first-run:
# it hides their approval requests too, so a blocked nested agent is
# invisible from the phone.
#   moshi-hook set suppress-nested-agent-push on
suppress_nested_agent_push = false
# On macOS, keep publishing events to Moshi but make them silent while the
# local console is unlocked. "Unlocked" is only an OS lock-state signal; it
# does not prove somebody is looking at the display. If lock state cannot be
# read, the push is sent.
#   moshi-hook set suppress-push-while-unlocked on
suppress_push_while_unlocked = false
# Optional HTTP probe allowlist for Browser Preview discovery. Omit it (or use
# "all") to scan every eligible loopback listener. An empty array disables
# HTTP probing entirely. Entries are single ports or inclusive "lo-hi" ranges,
# so listing the range you want is also how you leave other ports alone.
#   moshi-hook set scan-ports 3000,5173,8000
#   moshi-hook set scan-ports 3000,8000-8010
#   moshi-hook set scan-ports none
scan_ports = "all"
```

4. Default `127.0.0.1:24543`

Gateway HTTP is loopback-only and does not require `Authorization`. Clients reach it through SSH local forwarding. Host/account auth remains limited to daemon-to-Moshi server calls that deliver pushes, usage, approvals, and WebSocket frames.

The HTTP gateway serves diff and bounded control actions for servers surfaced by discovery. Pull-only host inspection is intentionally not exposed here: `GET /v1/capabilities`, `GET /v1/servers`, and `GET /v1/sessions/context` return `404`. Use the SSH preflight CLI commands below instead.

### Client mode: embedded web UI + SSH host bridge

See `docs/design/client-mode.md` for the full design. Bare `moshi` serves the embedded Moshi Web client and proxies its API traffic to the persistent daemon gateway. SSH is both auth and transport for remote hosts, and no machine opens a non-loopback port:

- The built app-moshi assets (populate with `scripts/build-webapp.sh`) are served by the foreground web listener on `127.0.0.1:24544`; extensionless paths fall back to the SPA shell. `/gateway/*`, `/events`, `/hosts/*`, `/v1/*`, and `/apps/*` proxy to the daemon at `127.0.0.1:24543`, preserving same-origin HTTP and WebSocket behavior. Ctrl-C stops the web listener without stopping agent hooks.
- `/hosts/<name>/<rest>` reverse-proxies `<rest>` (HTTP and WebSocket) to `127.0.0.1:24543` on `<name>` via `ssh -W` with `ControlMaster` reuse. `<name>` is any syntactically safe ssh destination (optional `user@` + hostname — config aliases and MagicDNS names alike; the daemon reads no ssh config of its own, clients remember their own host lists); unsafe names are `404`, ssh failures are `502` with the last ssh stderr line included (e.g. `Permission denied (publickey)`). BatchMode is forced: hosts needing interactive auth (passwords, locked agents like 1Password) fail fast — verify with plain `ssh <name>` first.
- `GET /v1/pty?mux=herdr&hideSidebar=true` sets `[ui] sidebar_collapsed_mode = "hidden"` in the host’s shared Herdr config and reloads the selected session before attaching; `false` sets `"compact"`. This affects other clients using that config and only changes the collapsed rail: collapse the sidebar in Herdr to hide it. Omit the parameter to leave configuration untouched. Requires Bash, Perl, and a Herdr version supporting `sidebar_collapsed_mode` (no `--hide-sidebar` flag). Respects `HERDR_CONFIG_PATH` and `XDG_CONFIG_HOME`; SSH edits run on the remote host. Windows hosts do not support this option. POST to the same URL applies the setting and reloads config without opening or closing a PTY. The app uses POST when the preference changes, including for parked terminals. Use your Herdr prefix followed by B (or your custom sidebar binding) to collapse or expand.
- `GET /v1/pty?mux=…&host=<name>` runs the multiplexer attach through `ssh -t <name>` on a locally-owned PTY; terminal bytes never transit the remote gateway.
- `POST /v1/hosts/forward` `{"host": "<name>", "ports": [3000, …]}` opens same-port ssh local forwards (`127.0.0.1:<p>` → remote `127.0.0.1:<p>`, max 16 per request) on the host's ControlMaster, so the client can load a remote dev server or simulator preview at `http://localhost:<p>` per the same-port doctrine (no path-prefix reverse proxy — see Transport under `/events`). Idempotent per live master; forwards die with it (ControlPersist reaps an idle master after 10 minutes) and the next request re-establishes them. Unsafe host names are `400`, ssh failures `502` with the last stderr line. A local port collision is detected before the mux request and surfaces as a plain-language `502` (a leftover forward held by the live master is cancelled and re-added instead); the URL is never rewritten to a different port.
- `GET /v1/hosts/forwards` lists the live tunnels on this machine as `{"forwards": [{"host", "port", "pid"?}]}` — daemon bookkeeping (pruned when the local port has come free) merged with every discovered ssh-owned listener (hand-rolled `ssh -L`, or daemon forwards a restart forgot; `host` is parsed best-effort from the ssh command line, `pid` is the listener's). `POST /v1/hosts/unforward` `{"host", "port", "pid"?}` tears one down: a bridgeable host gets a mux cancel; otherwise the pid — verified to still be an ssh listener on that port — is terminated. Idempotent, judged by the local port coming free rather than ssh's unreliable `-O cancel` exit code. Both act on the LOCAL daemon only; tunnels are invisible to the remote gateway. Discovery never lists ssh-owned listeners as dev servers — a tunnel answers probes with the remote end's content and belongs in the tunnels list.

### `GET /events`

Opens the local WebSocket carrying the current terminal context, discovered
web servers in `servers`, and serve-sim previews in the optional `simulators`
array. A serve-sim preview is never duplicated in `servers`. For a working
agent in Herdr or tmux, the context may include a
replaceable, non-persisted terminal preview:

The first snapshot identifies the hook and its additive API capabilities:

```json
{
  "gateway": {
    "version": "0.3.1",
    "protocolVersion": 1,
    "capabilities": [
      "events.watch.workspaces",
      "events.watch.agent-status",
      "events.license",
      "transcripts.limit",
      "terminal.prompt",
      "terminal.keys",
      "workspaces.live-session",
      "events.watch.usage"
    ]
  }
}
```

Clients should select API paths by capability rather than comparing release
versions. `version` is still useful for diagnostics, UI, and coarse handling of
older daemons that omit `gateway`; `protocolVersion` versions the envelope and
does not change for additive fields or capabilities.

### `GET /v1/version`

The same `gateway` object, served over plain HTTP (added in 0.3.1). Exists for
one job: a pre-connect compatibility probe over the ssh host bridge, before
the client commits to a host switch. A remote daemon answering `404` here is
by definition older than 0.3.1 (pre watch-protocol) — clients block the switch
and tell the user to upgrade that host's moshi-hook instead of connecting with
degraded behavior.

```jsonc
{
  "context": {
    "kind": "herdr",
    "agent": {
      "name": "kimi",
      "status": "working",
      "session": "agent-session-id",
      "pane": "w1:p1",
      "ephemeral": {
        "type": "working",
        "lines": [
          "Reading internal/gateway/events.go…"
        ]
      }
    }
  }
}
```

`ephemeral` is current display state, not a transcript message. Clients replace
it on each context update and discard it when the agent stops working. The
gateway polls terminal text every 250 ms. Herdr previews compare the last ten
visible rows and publish the latest changed row. Tmux previews inspect the last
ten non-empty rows from the live pane buffer, scan upward, and publish the
newest trusted agent status line. If no trusted tmux line is present, clients
keep showing their generic Working label. Herdr reads intentionally omit the
`--lines` option so previews also work with versions before 0.7.5.

When a blocked agent is showing a numbered plan-decision menu, `agent` may
also contain `planMenu`. Its title and option labels are extracted from that
session's live terminal. They are not inferred from the agent name, account
tier, or a version profile:

```json
{
  "planMenu": {
    "title": "Implement this plan?",
    "options": ["Approve", "Keep planning"]
  }
}
```

#### Watch protocol

A connected client can send a watch request at any time to subscribe to
server-side pushes. On a session-scoped `/events` connection (SSH, Mosh, or
ET lookup), workspace trees follow the resolved terminal’s mux/session,
including its tmux socket. The watch `mux` selection is ignored for those
connections. Without a session lookup, workspace trees use the loopback mux
selection (Herdr by default); `context: true` enables loopback context pushes.

```jsonc
{ "watch": { "workspaces": true, "agent": { "source": "claude", "session": "agent-session-id" }, "context": true, "usage": true } }
```

The gateway acks with `{"watching": {"workspaces": true, "agent": true, "context": true, "usage": true}}`
and then pushes frames whenever their content changes (workspaces and context
on a tick, agent status on a 250 ms tick, all deduped by JSON). The workspace
tree rebuilds every 5 s at rest and every 1 s for 3 s after any hook event,
which also wakes the agent watch:

```jsonc
// the loopback mux tree, same shape as GET /v1/workspaces
{ "workspaces": { "kind": "herdr", "groups": [ /* … */ ] } }

// the loopback mux's terminal context — the same TerminalContext shape the
// session-scoped context mode serves iOS, detected for the daemon's own
// machine (herdr's focused pane; copyMode/scrollPosition drive the web
// terminal's scroll-to-bottom control). Currently herdr only.
{ "context": { "kind": "herdr", "herdr": {
    "session": "work", "paneId": "wB:p6", "copyMode": true,
    "scrollPosition": 42, "historySize": 900 }, "cwd": "/Users/me/proj" } }

// one agent session's live state: status + model from the hook stamps,
// the scraped status line while working, the plan menu while blocked,
// and any pending interactive prompt (the hook-captured agent.prompt blob)
{ "agentStatus": {
    "source": "claude", "session": "agent-session-id",
    "status": "working", "modelName": "fable-5",
    "statusChangedAt": 1787219601.2,     // when "status" began; see /v1/workspaces
    "title": "Fix login redirect loop",  // conversation title, see /v1/workspaces
    "contextRemaining": 42,  // % of context window left, 1..100; omitted = unknown
    "commands": [
      { "name": "clear", "description": "Clear conversation history" },
      { "name": "compact", "description": "Summarize and compact the conversation", "inputHint": "[instructions]", "openTerminalOnRun": true }
    ],
    "ephemeral": { "type": "working", "lines": ["✳ Thinking… (12s)"] },
    "planMenu": { "title": "Implement this plan?", "options": ["Approve", "Keep planning"] },
    "pendingPrompt": { "kind": "question", "toolUseId": "…", "questions": [ /* … */ ] },
    "pendingPromptAt": 1787219608.6,
    "pendingApproval": {
      "actionId": "122fa92b…",
      "title": "Bash command",
      "message": "touch /tmp/probe",
      "toolName": "Bash",
      "openedAt": 1787219608.6
    }
} }
```

`usage: true` (capability `events.watch.usage`) pushes this machine's agent
account rate-limit windows — the same local reads as `moshi-hook usage`, taken
from each agent's own credential/cache files, never from the Moshi server, so
it works unpaired. The current list arrives right after the ack (once the
daemon has collected; nothing while usage collection is off), then again
whenever it changes. The list replaces the previous one:

```jsonc
{ "usage": [ {
    "accountId": "…", "accountLabel": "Max", "agent": "claude-code",
    "hostName": "laptop", "capturedAt": "2026-09-25T03:00:00Z",
    "windows": [ { "label": "5h", "usedPercentage": 42, "resetsAt": "2026-09-25T06:00:00Z" } ]
} ] }
```

`pendingApproval` is the live tool-approval interaction from the daemon's TUI
bridge — the same verified prompt the phone inbox sees — so Chat View can
render an approve/deny card for terminal permission dialogs. It is never set
alongside `pendingPrompt` — the richer native question/plan card wins — but it
may coexist with `planMenu`, whose numbered-menu scraper matches permission
dialogs too. Answer it via `POST /v1/approvals/answer`
(capability `approvals.answer`); the frame drops the field within a tick of
the dialog resolving, however it was answered.

`commands` is a complete replacement snapshot for the active terminal agent's
Chat View slash-command menu. Names omit the leading `/`. The daemon owns this
agent-specific catalog so desktop and mobile clients do not maintain divergent
lists. Catalogs are deliberately version-tolerant and describe commands to send
to the already-running TUI; they are not ACP capability claims. Clients should
replace their cached list whenever a new `agentStatus` frame arrives.
`openTerminalOnRun: true` means the command's result belongs to the agent TUI:
after dispatching it, clients should immediately reveal the owning terminal
instead of leaving Chat View waiting for a JSONL row. Session-boundary
commands (`/new`, `/clear`) omit the flag — Chat View renders their outcome
itself (session follow → fresh empty chat). Custom skills may be
presented alongside this list, but must retain their agent-native
prefix (`/` or `$`) and should not overwrite a built-in command with the same
name.

### `GET /v1/composer/suggestions?source=<agent>&session=<id>&kind=commands|files|skills[&query=<text>&limit=<n>]`

Returns the host-owned autocomplete data for one live Chat View session. This
is a bounded UI read rather than general filesystem access: `source` and
`session` must resolve to hook-written session state, and the workspace root is
always taken from that state. Callers cannot supply a path to scan. `limit`
defaults to 20 and is capped at 50.

```jsonc
// kind=commands
{
  "source": "codex", "sessionId": "agent-session-id", "kind": "commands",
  "items": [{
    "kind": "command", "name": "compact", "prefix": "/",
    "description": "Compact the conversation", "openTerminalOnRun": true
  }]
}

// kind=files&query=app
{
  "source": "codex", "sessionId": "agent-session-id", "kind": "files",
  "items": [{ "kind": "file", "name": "src/App.tsx", "path": "src/App.tsx", "prefix": "@" }]
}

// kind=skills&query=review
{
  "source": "codex", "sessionId": "agent-session-id", "kind": "skills",
  "items": [{
    "kind": "skill", "name": "review", "prefix": "$", "scope": "project",
    "description": "Review the current changes carefully",
    "path": "/Users/me/project/.codex/skills/review/SKILL.md"
  }]
}
```

`kind=commands` merges the built-in catalog with user-authored custom commands
discovered on disk: Claude Code `.claude/commands/*.md` (project) and
`<config>/commands/*.md` (user, honoring a session-specific
`CLAUDE_CONFIG_DIR`), Codex `$CODEX_HOME/prompts/*.md`, OpenCode
`.opencode/command/*.md` plus its config-dir `command/*.md`, and Gemini/Qwen
`commands/*.toml` (nested TOML directories namespace with `:`, e.g.
`/git:commit`). Custom entries carry `scope: "project" | "user"` and a `path`,
never shadow a same-named built-in, and omit `openTerminalOnRun` because they
expand to prompts that produce transcript rows. Custom-command discovery is
best effort: a missing root or unresolved cwd drops the custom entries, never
the catalog.

File results are relative to the session workspace, never follow symlinks, and
skip common generated/vendor directories. Skill discovery reads `SKILL.md`
frontmatter from the active agent's project and profile roots, including a
session-specific `CLAUDE_CONFIG_DIR`, `$CODEX_HOME`, and `$GROK_HOME`. Project
skills win name collisions over user and bundled skills. Codex skills use `$`;
other currently supported skill surfaces use `/`. `truncated: true` means the
result or bounded filesystem walk reached a limit and the client should refine
`query`.

### `GET /v1/session/options?source=<agent>&session=<id>`

Returns the model and model-dependent option catalog for one live Chat View
session. The first catalog version follows Orca IDE's session-option structure
and covers Claude, Codex, Gemini, Cursor, and Grok. Other recognized agents
return `supported: false` with an empty `models` array, so clients can hide the
picker without maintaining their own support list.

```jsonc
{
  "source": "codex",
  "sessionId": "agent-session-id",
  "catalogSource": "curated",
  "catalogVersion": "2026-08-23",
  "authoritative": false,
  "supported": true,
  "current": { "model": "gpt-5.6-sol", "effort": "high" },
  "models": [{
    "id": "gpt-5.6-sol",
    "label": "GPT-5.6 Sol",
    "options": [{
      "id": "effort",
      "label": "Reasoning effort",
      "category": "thought_level",
      "type": "select",
      "defaultValue": "medium",
      "choices": [
        { "value": "minimal", "label": "Minimal" },
        { "value": "low", "label": "Low" },
        { "value": "medium", "label": "Medium" },
        { "value": "high", "label": "High" },
        { "value": "xhigh", "label": "Extra high" },
        { "value": "max", "label": "Max" },
        { "value": "ultra", "label": "Ultra" }
      ],
      "apply": {
        "enabled": false,
        "mode": "agent-picker",
        "command": "/model",
        "openTerminalOnRun": true
      }
    }]
  }],
  "modelApply": {
    "enabled": false,
    "mode": "agent-picker",
    "command": "/model",
    "openTerminalOnRun": true
  }
}
```

`current` is read from durable agent state and, where available, refined from
the latest native transcript events so a model changed inside the TUI is not
reported from stale hook state. Grok is refined from its session `summary.json`
(`current_model_id`, `reasoning_effort`) instead: Grok rewrites that file the
moment `/model` or `/effort` runs, while the transcript's `_meta.modelId` is
stamped per user message, never revised, and carries no effort at all. A
current model need not appear in `models`:
catalogs are versioned suggestions, while account-specific and custom model IDs
remain valid session state.

The `apply` fields reserve the eventual live-update transport. They are
deliberately returned with `enabled: false` in this API slice: clients may render
the picker but must not yet submit mutations. `mode: "command"` means a future
implementation can send the formatted command to the TUI; `agent-picker` means
the harness must own the final choice and the client should reveal Terminal View.

Re-sending `watch` reconfigures the subscription (one agent session at a
time; `"agent": null` clears it) and always answers with a fresh snapshot.

### `GET /v1/integrations`

Reports the hook install state of every supported agent, for the web Settings
→ Integrations list. Same probe as `moshi-hook pair --json`'s `hooks` rows:
`not_found` means the agent's config root doesn't exist on this machine,
`stale` means the agent exists but its hook config is missing entries (a
reinstall fixes it), `current` means everything the installer would write is
present.

```jsonc
{
  "integrations": [
    { "target": "claude", "status": "current", "path": "/Users/me/.claude/settings.json" },
    { "target": "codex", "status": "stale", "path": "/Users/me/.codex/hooks.json",
      "missing": ["SessionStart"] },
    { "target": "kimi", "status": "not_found" }
    // "error" rows carry an "error" string; "advisories" list prerequisites
    // a reinstall cannot supply (e.g. an external binary to install).
  ]
}
```

### `POST /v1/integrations/install`

Runs the implicit `moshi-hook install` over HTTP: each requested target whose
agent config root exists gets its hooks (re)written; agents never configured on
this machine are skipped, not created. Body is `{}` for all targets or
`{ "targets": ["claude", "codex"] }` for a subset. Responds with per-target
results plus the refreshed status list so clients update in one round trip.

```jsonc
{
  "results": [
    { "target": "claude", "action": "installed" },
    { "target": "codex", "action": "skipped", "reason": "agent not found" }
    // "error" actions carry the failure in "reason"
  ],
  "integrations": [ /* same shape as GET /v1/integrations */ ]
}
```

### `POST /v1/diff/start`

Starts or reuses an embedded diff viewer session for a Git repository.

```jsonc
// request
{ "cwd": "/Users/me/projects/foo" }

// response
{ "diffSessionId": "diff_abc123", "url": "/apps/diff/diff_abc123/" }
```

Diff sessions expire after 15 minutes idle and are served under `/apps/diff/:sessionId/`. Diff payloads are read from the host filesystem and never sent to the Moshi backend.

### `POST /v1/questions/answer`

Submits a complete Chat View answer form to the live Claude Code, Codex, Cursor,
Grok Build, Hermes, Kimi, Pi, OMP, or OpenCode terminal prompt in tmux, Zellij, or Herdr. The request
uses the same SSH/Mosh/ET session query parameters as `/events`. The daemon
re-resolves the multiplexer pane and checks the agent name, agent session id,
first question, option labels, and native TUI prompt markers before injecting
any keys. A changed/stale prompt returns `409` without sending input.

The session-lookup params are optional for loopback callers (same rule as
`/v1/prompt`): without them, the pane and the hook-captured prompt blob are
resolved from the recorded state for the given `source`/`sessionId` (tmux and
herdr; `404` when no state exists), and the on-screen verification still runs
before any key is sent.

Cursor is answered one question per request: its form draws a single question at
a time and Enter advances to the next, so `questions` must hold exactly the
question the app is showing. The form itself submits once every question has an
answer.

```jsonc
{
  "source": "claude",
  "sessionId": "agent-session-id",
  "toolUseId": "toolu_123",
  "questions": [
    { "question": "Choose one?", "options": ["Alpha", "Beta"] },
    { "question": "Choose colors?", "multiSelect": true, "options": ["Red", "Blue", "Green"] }
  ],
  "answers": [
    { "optionIndexes": [1] },
    { "optionIndexes": [0, 2] }
  ]
}
```

Option indexes are zero-based. Every question must have at least one selection;
single-select questions require exactly one. Codex does not support
multi-select questions. Herdr uses the same harness-specific answer sequence;
the bridge translates it through `herdr pane send-keys` after resolving and
verifying the focused pane.

### `POST /v1/plans/answer`

Selects one option from a live ExitPlanMode / plan-review menu through the
same verified TUI bridge. It supports Claude Code, Codex, Kimi, Pi, OMP, and
OpenCode. The app sends the complete
visible menu and the plan markdown, not a raw key. The daemon verifies the
agent session, menu title, every numbered option label, and a visible
fingerprint from the plan before it selects anything. As with question
answers, the session-lookup params are optional for loopback callers, which
resolve the pane and captured prompt from the recorded agent state instead.

```jsonc
{
  "source": "codex",
  "sessionId": "agent-session-id",
  "toolUseId": "plan-call-id",
  "prompt": {
    "title": "Implement this plan?",
    "plan": "# Plan\n\n1. Add the endpoint.\n2. Cover it end to end.",
    "options": [
      "Yes, implement this plan",
      "Yes, clear context and implement",
      "No, stay in Plan mode"
    ],
    "optionIndex": 0
  }
}
```

`optionIndex` is zero-based. Claude, Codex, Kimi, and OpenCode menus use their
numbered choice bindings. Pi and OMP use the native arrow-and-confirm
sequence, paced as separate terminal events so redraws cannot swallow input.
A stale plan, changed menu, or different agent session returns `409`; an
unavailable pane also fails without sending a partial decision.

### `POST /v1/approvals/answer`

Routes a Chat View approve/deny into the agent's native terminal permission
dialog through the daemon's TUI bridge (capability `approvals.answer`). The
`actionId` comes from the `pendingApproval` field of the `/events` agentStatus
frame. The bridge re-verifies the visible screen before typing — the same
verification the phone's remote decision path runs — so a stale card cannot
answer a different question.

```jsonc
{
  "source": "claude",
  "sessionId": "agent-session-id",
  "actionId": "122fa92b…",
  "decision": "approve"                  // "approve" | "deny"
}
```

Returns `200` with `{"ok": true, …}` once the keys are delivered. A changed,
superseded, or already-answered approval returns `409`; a missing pane `422`;
a daemon running without its TUI bridge `503`. The resolved state reaches
clients as the next agentStatus frame (the `pendingApproval` field disappears
and the status leaves `blocked`).

### `POST /v1/prompt[?<session lookup>]`

Types a free-form prompt into the pane running the given agent session and
submits it with Enter, so a chat client can message the agent without owning a
terminal connection. Unlike question and plan answers there is no on-screen
form to replay, so no screen-content verification happens — the text lands in
whatever state the agent's composer is in.

The session-lookup query params are optional here. With them (`ssh-connection`,
`mosh-port`[+`mosh-host`], or `et-client-id`), the caller's terminal is
resolved live exactly like `/v1/questions/answer`, and a terminal that no
longer runs the expected agent or session returns `409` without injecting
anything. Without them — a local loopback client has no SSH session — the pane
is resolved from the hook-recorded state for that agent session (tmux and
herdr only), and the pane's existence is re-checked before any input is sent.

```jsonc
// request
{ "source": "claude", "sessionId": "agent-session-id", "text": "fix the failing test" }

// response
{ "ok": true, "source": "claude", "sessionId": "agent-session-id" }
```

Desktop/web composers can preserve clipboard blocks separately from the typed
instruction in the same request:

```json
{ "source": "claude", "sessionId": "agent-session-id", "text": "Summarize this", "textMode": "typed", "pastedText": ["reference text"] }
```

`pastedText` accepts up to 16 nonempty blocks. Each is delivered as bracketed
paste before `text`; one final Enter submits the whole message. With
`textMode: "typed"`, the instruction is written once without paste markers;
line breaks use Meta-Enter to avoid submitting early. Native agents may still
classify large writes as pasted input. Omitting `textMode` preserves the
existing multiline bracketed-paste behavior. Paste-only requests may use an
empty `text`. The JSON body limit remains 1 MiB. Invalid blocks are rejected
before injection. Do not automatically retry failed submissions: earlier
blocks may already be in the native composer. Older daemons reject these new
fields before writing any input.

For an agent with no observable session ID yet (for example Codex after
`/new`), send `{ "source": "codex", "pane": "<pane-id>", "text": "hello" }`.
With a session lookup, the pane must exactly match the caller's resolved live
terminal, which must still run the expected agent without a session ID.
A mismatch returns `409` before input; remote tab-only addresses are rejected
with `400`. The resolved context supplies the mux session and socket, so a
pane address cannot select a different terminal. Loopback clients may address
an agent by pane or tab, checked against the live mux agent list.

Multi-line text is wrapped in bracketed-paste markers so harnesses that enable
bracketed paste keep the newlines in the composer instead of submitting on
each one; the submitting Enter is sent as its own terminal event after a short
render window. Failure codes: `400` missing `source`/`sessionId`/`text` or bad
lookup params; `404` no live terminal (with lookup) or no recorded state for
the session (without); `409` the live terminal changed agent or session; `422`
the terminal kind cannot take injected text; `500` the pane vanished or the
multiplexer command failed.

### `POST /v1/keys[?<session lookup>]`

Injects a short sequence of named control/navigation keys (interrupt, mode
toggles, menu movement) into the pane running the given agent session — the
raw-control counterpart to `/v1/prompt`, with the same dual pane resolution
(verified live terminal with lookup params, hook-recorded state for loopback
callers). Keys come from a fixed allowlist: `Enter`, `Escape`, `Tab`, `BTab`,
arrows, `Space`, `Home`/`End`, `PageUp`/`PageDn`, `BSpace`, `C-a`…`C-z`
chords, and single printable characters; at most 8 per request, paced as
separate terminal events. Free-form text is rejected — it belongs to
`/v1/prompt`.

```jsonc
// request
{ "source": "claude", "sessionId": "agent-session-id", "keys": ["Escape"] }

// response
{ "ok": true, "source": "claude", "sessionId": "agent-session-id" }
```

### `GET /v1/muxes`

Enumerates the loopback muxes a session-less client can pin: every herdr
session the CLI knows (running or not — a stopped session is still
selectable, the PTY attach `herdr --session <name>` starts it) plus tmux
when installed. `active` marks the option the default loopback resolution
currently picks; `id` is exactly what clients pass back as the `mux`
selection. Through the `/hosts/<name>/` bridge the list describes that
remote machine.

```jsonc
{
  "muxes": [
    { "id": "herdr:default", "kind": "herdr", "session": "default", "running": true, "active": true },
    { "id": "herdr:ztest", "kind": "herdr", "session": "ztest", "running": false },
    { "id": "tmux", "kind": "tmux", "running": true }   // running: the server has sessions
  ]
}
```

### `GET /v1/workspaces[?<session lookup>]`

Enumerates the normalized two-level workspace tree of a multiplexer — the
data structure behind the apps' Jump To menu. Both herdr (workspace → tab)
and tmux (session → window) collapse onto one shape; herdr additionally
reports a live per-node agent status and per-tab agent kind.

With session-lookup params (`ssh-connection`, `mosh-port`[+`mosh-host`], or
`et-client-id`) the tree describes the mux the caller's terminal runs inside,
and `focused` marks the caller's current branch. Without them — a loopback
desktop client has no terminal session — the mux resolves to herdr when its
server responds, or the default tmux server otherwise, and no group is marked
focused; `mux=herdr:<session>` / `mux=tmux` pins another local mux instead
(see `GET /v1/muxes`). The same optional `mux` param rides every loopback
`/v1/workspaces/*` call and, as a `mux` field, the `/events` watch frame.
`422` when the
terminal's mux is unsupported (zellij) or, for loopback, when no local mux
exists (or the pinned herdr session is not running); `400` on a malformed
`mux` value.

```jsonc
{
  "kind": "herdr",                              // "herdr" | "tmux"
  "capabilities": { "paneList": true, "paneFocus": "exact" },  // "agent-only" on Herdr < 0.9.0
  "groups": [{
    "id": "wB", "label": "app-moshi", "focused": true, "agentStatus": "working",
    "worktree": { /* herdr repo membership, when known */ },
    "children": [{
      "id": "wB:t1", "label": "1", "focused": true,
      "agentStatus": "working",                  // working|blocked|done|idle|unknown
      "agent": "claude",
      "sessionId": "85d203b1-…",                 // live agent session in this tab
      "title": "Fix login redirect loop",        // conversation title (see below)
      "model": "fable-5",                        // display label; omitted when unknown
      "contextRemaining": 42,                     // 1..100; omitted when unknown
      "cwd": "/Users/me/projects/app-moshi",
      "statusChangedAt": 1787219601.2,           // when agentStatus began (Unix s)
      "paneCount": 1, "stateChangeOrder": 18
    }, {
      "id": "wB:t2", "label": "2", "focused": false,
      "cwd": "/Users/me/projects/app-moshi",
      "command": "node",                         // shell tab with a job in the foreground
      "paneCount": 1
    }]
  }]
}
```

Herdr children include `panes` in this response and in `/events` workspace
frames, using the same pane fields as `/v1/workspaces/panes`. An empty list
(`"panes": []`) means loaded and empty; an absent field means panes must be
fetched separately (tmux). Clients should use inline panes immediately and
must not replace them with an older cached pane list.

The Herdr tree uses `session.snapshot` for workspaces, tabs, panes, and agents
in one topology read (verified on Herdr 0.9.0, protocol 22); a failed or
incomplete snapshot fails the tree request. A server that predates the method
answers it with `invalid_request`, and the daemon then assembles the same
topology from `workspace.list`, per-workspace `tab.list`, `pane.list`, and
`agent.list`, so Herdr releases before 0.9.0 keep working; such trees advertise
`"paneFocus": "agent-only"`. Separate foreground-process verification remains
necessary because neither path contains process details.

`statusChangedAt` is when the node's current `agentStatus` began, in Unix
seconds with millisecond precision. Use it to show "working for 3m" or
"finished 5m ago", and to sort agents by recency; unlike `stateChangeOrder`
(herdr only, resets with the herdr server) it exists on both muxes, is
comparable across hosts, and survives restarts wherever hooks are installed.
It is present on agent panes, tabs/windows, groups, the `agentStatus` watch
frame, and `context.agent`. Neither mux exposes transition times, so the
daemon derives it:

- **Hook time** when the session's hook state explains the shown status: the
  prompt-submit stamp for `working`, the turn-stop stamp for `idle`/`done`,
  the permission/question stamp for `blocked`. Exact, and stable across
  daemon restarts.
- **Observed time** otherwise (agents without hooks, a status only the screen
  shows, a turn that resumed after a permission wait): when the daemon first
  saw the status, or — herdr — a new `stateChangeOrder`. Accurate to the
  sampling cadence (250 ms for a watched agent, 1–5 s for the tree).
- `statusChangedAtApprox: true` marks an observed time from the first sighting
  (typically after a daemon restart): the status began *at or before* it, so
  render it as "≥ 5m" rather than an exact age.

`done` and `idle` share one clock (visiting a done tab does not reset it). A
tab/window takes the newest time among its agent panes that show the tab's
status; a group the newest among children showing its status, or among all
timed children when the mux reports no group status (tmux). Shell panes and
tabs without an agent carry no time. A suggested sort is attention first
(`blocked`, then `done`), then `statusChangedAt` descending.

`command` marks a terminal that has something running: the base name of the
foreground process of a shell (non-agent) tab or pane — `node`, `vim`, `go` —
omitted when the pane is sitting at its prompt, when the tab hosts an agent
(its `agentStatus` says what it is doing), or when nothing can be resolved.
Shells, login wrappers, and nested multiplexers (`tmux`, `zellij`, `screen`,
`herdr`) count as idle. tmux reads it from `#{pane_current_command}`; herdr
from `pane.process_info` (the first non-shell process of the foreground job).
On a multi-pane shell tab the focused pane's job wins, else the first busy
pane's. It is sampled on the tree tick (~1s), so sub-second commands never
show; long-lived but idle programs (an editor, a dev server) do — the field
answers "is something running here", not "is it busy".

`title` is the session's conversation title, filled for session-bearing nodes
(tabs and inline panes here, panes on `/v1/workspaces/panes`, and the `agentStatus` watch
frame). It is read from the agent's own records where one exists — Claude's
`custom-title` (user rename, wins) and `ai-title` rows in the session JSONL;
Codex's `thread_name` in `<CODEX_HOME>/session_index.jsonl`; Grok's
`generated_title`/`session_summary` in the session dir's summary.json; Kimi's
`title` in the session dir's state.json; OpenCode's live-server session title
(`GET /session/:id`); OMP's leading `{"type":"title"}` slot row — with a short
derivation from the first real user prompt as the fallback (Pi, Cursor,
Hermes have only the derived form; harness-injected turns are skipped).
Scans are cached and incremental, so tree ticks stay cheap.

### `GET /v1/workspaces/panes?groupId=<id>&childId=<id>[&<session lookup>]`

An explicit refresh of panes for one tree child, also used for the lazy tmux
third level. It echoes the requested target so
a late response can be matched to its row. Same session-lookup rules and
loopback fallback as `/v1/workspaces`.

### `POST /v1/workspaces/focus[?<session lookup>]`

Focuses a workspace/tab/pane (herdr) or session/window (tmux) in the caller's
mux. Requires the session lookup: tmux focus switches the caller's own
attached client, which a loopback caller does not have. Herdr pane targets use
`pane.focus` for both agents and shells, so attached terminals follow the
selection even after client-local navigation (Herdr 0.9.0 and newer). When the
server rejects `pane.focus` as unsupported, the daemon falls back to the older
contract: `agent.focus` for agent panes, and a bounded neighbor-by-neighbor
focus walk toward shell panes; a walk that cannot reach the pane reports the
`not-an-agent-pane` fallback reason.

### `GET /v1/transcripts?session=<id>[&source=claude|codex|cursor|grok|opencode|hermes|pi|omp|kimi|antigravity][&limit=<n>][&cursor=<opaque>]`

Opens a local WebSocket stream for a live agent transcript. New clients should pass `source`; when omitted for backward compatibility, the gateway tries Claude first, then Codex. Claude transcripts prefer the exact per-session path captured from hook events (including `CLAUDE_CONFIG_DIR` profiles), then fall back to `~/.claude/projects` for older session state. Codex transcripts are resolved from `$CODEX_HOME/sessions` or `~/.codex/sessions` rollout files. Cursor resolves its native `~/.cursor/chats/<workspace>/<conversation-id>/store.db` and streams role-bearing message blobs in insertion order, polling the live SQLite store for appended messages. Grok streams the authoritative ACP `updates.jsonl` reported by its hooks, with a `$GROK_HOME/sessions/<encoded-cwd>/<session-id>/` scan as a fallback for older sessions. Completed Grok `image_gen` results are exposed as lazy ACP image blocks backed by the generated file, so Chat View can render them without terminal graphics support. Pi and OMP transcripts use the exact JSONL path reported by the installed extension, so profiles and custom session locations work without a directory scan. OMP validation understands its v3 fixed-width title slot before the session header. Kimi transcripts resolve through its profile-aware `session_index.jsonl` and stream the main agent's live `wire.jsonl`. OpenCode is proxied through the live local server recorded by its plugin. Transcript bytes stay on the host and are streamed only over the local forwarded gateway. If Codex resume creates a newer rollout for the same session id, reconnect to resolve the newest file.

Server messages are JSON objects with `type` (`backlog`, `older`, `append`, `resumed`, `reset`, or `error`), `source`, physical `line` numbers, and raw JSONL rows for client-side rendering. Clients can request older rows with `{"type":"older","beforeLine":123,"limit":50}`.

The optional `limit` query parameter shrinks the opening `backlog` page, which matters on long-haul links where the default 200-row page costs several round trips of TCP slow start. It counts **kept source rows** — the physical rows the daemon streams — not rendered chat messages; several source rows routinely collapse into one message, so a client asking for 30 rows should expect noticeably fewer. Values that are missing, unparseable, non-positive, or above the 200-row default are ignored and produce exactly the default page, so older clients are unaffected. `startLine`, `totalLines`, and `hasMore` stay physical and correct, so paging back with `older` converges on the same history as an unlimited backlog. `limit` applies to every `backlog` message on the connection, including the one re-sent after a `reset`.

Oversized rows are redacted before streaming: long strings are truncated, and inline image payloads (Claude `source.data`, Grok/Pi/OMP `data`, Codex `image_url` data URLs, and OpenCode/Kimi `url` data URLs) are replaced with a stub carrying `truncated: true`, `media_type`, decoded `bytes`, and `width`/`height` when the format is recognized. Grok `image_gen`/`image_edit` file results and local source images passed to `image_edit` receive the same stub without embedding their bytes. Clients fetch the actual bytes via the blob endpoint below.

The optional `cursor` resumes a previously committed transcript checkpoint. Cursors are opaque, versioned, and bound to the source, session and storage backend. The gateway validates the complete prefix (including its physical row count and byte offset) using SHA-256. An unchanged prefix produces only suffix `append` transactions followed by `resumed`, including when the suffix is empty. Large initial suffixes are committed in transactions of at most 200 physical rows. A changed or truncated prefix produces an ordinary authoritative `backlog`; the client must rebuild its reducer. A file rewrite that does not change its size is detected on reconnect; the live file tailer's equal-size fast path does not detect it immediately.

A `cursor` on an `append` is the **commit marker for the entire burst**, including preceding cursorless fragments. Buffer those fragments and reduce them only after the commit marker arrives. On disconnect, discard uncommitted fragments and resume from the last committed cursor. Never use `totalLines` as a checkpoint: every fragment may report the final total before all rows have arrived. Persist the cursor and corresponding reducer inputs together. `resumed` completes catch-up without changing the oldest loaded boundary or older-page availability. `older` responses never advance the forward cursor. Materialized `backlog` responses carry a cursor; virtual pending rows do not. A cursorless backlog is still authoritative and requires a full refresh. Mutable OpenCode rows can invalidate an earlier prefix, so an active-turn reconnect may legitimately require a full refresh.

### `GET /v1/transcripts/blob?session=<id>&line=<n>[&block=<i>][&source=claude|codex|cursor|grok|opencode|hermes|pi|omp|kimi|antigravity]`

Serves the raw image bytes of one content block of one transcript line, re-read from disk or re-fetched from OpenCode on demand (so redaction never loses data). `line` is the physical transcript line index reported by the stream; `block` (default 0) indexes `message.content[i]` for Claude/Pi/OMP, including a Claude `tool_result` whose content contains an image; ACP `update.content[i]` for Grok; `event.result.output[i]` for Kimi; `payload.content[i]` / `payload.output[i]` for Codex; or the flattened OpenCode image attachments. Grok's synthetic Imagine blocks map back to `rawInput.image[i]` or `rawOutput.path`; OMP `blob:sha256:` references resolve through the profile/XDG-aware `blobs/` directory beside its managed `sessions/` tree; Kimi `blobref:<mime>;<sha256>` references resolve through the `blobs/` directory beside its main-agent `wire.jsonl`. For Codex `view_image` function calls the endpoint resolves the call's absolute file `path` on the host and serves the file when it sniffs as an image (capped at 32 MB). Responds with the image `Content-Type` and cache headers; returns 404 when the addressed block is not an image.

### `POST /v1/servers/kill`

Terminates a discovered local HTTP server. The daemon re-runs server discovery and only signals a process whose current PID and port match the request; arbitrary PIDs are rejected. By default callers should send `force: true`, which sends `SIGTERM` first and falls back to `SIGKILL` if the process does not exit within the grace period.

```jsonc
// request
{ "host": "127.0.0.1", "port": 5173, "pid": 27753, "force": true }

// response
{ "killed": true, "forced": false, "pid": 27753, "port": 5173, "server": { /* discovered server */ } }
```

### `POST /v1/paste[?<session lookup>]`

Injects an image into the caller's multiplexer pane: the tmux pane, the focused herdr pane, or the zellij pane (falling back to the session's focused pane when the context carries no pane id, since zellij focused-pane actions silently no-op without an attached client). Takes the same session-lookup query params as `/events` (`ssh-connection`, `mosh-port`[+`mosh-host`], or `et-client-id`); the pane is resolved live on the host, so the app never passes (possibly stale) pane ids. Loopback callers may omit the lookup and pass `source` + `sessionId` in the body instead; the pane then comes from the recorded agent state (tmux and herdr). Bodies are capped at 64 MB.

```jsonc
// request (with session lookup)
{ "data": "<base64 image bytes>", "mimeType": "image/png" }

// request (loopback, no session lookup)
{ "data": "<base64 image bytes>", "mimeType": "image/png", "source": "claude", "sessionId": "agent-session-id" }

// response
{ "ok": true, "mode": "clipboard", "verified": true, "path": "/tmp/moshi-paste-123.png" }
```

The daemon writes the image to `$TMPDIR/moshi-paste-*` (stale files are swept after 24h) and picks a mode:

- `clipboard` — when the pane runs an agent and a clipboard is reachable (macOS GUI session via `osascript`, Wayland via `wl-copy`, X11 via `xclip`; display env is read from the tmux session/global environment since the daemon itself has none — zellij and herdr have no queryable session env, so those targets use the daemon's env), seed the OS clipboard and send Ctrl+V so the agent picks up the image inline. The key grammar differs per multiplexer: `C-v` (tmux send-keys), `ctrl+v` (`herdr pane send-keys`), `"Ctrl v"` (`zellij action send-keys`) — each rejects the others' tokens.
- `path` — otherwise (headless hosts, plain shell panes), type the temp-file path literally into the pane (`tmux send-keys -l` / `herdr pane send-text` / `zellij action write-chars`).

The hook checks supported image signatures before writing the temporary file, correcting stale client MIME declarations such as JPEG bytes labeled `image/png`. Pane content is read with tmux `capture-pane` + cursor, `herdr pane read`, or `zellij action dump-screen`, polled up to 1.5s.

Clipboard mode is confirmed by a **new image placeholder** in the pane (`[Image #N]`, as Grok and Claude Code render it), not by a pane diff. A diff is not evidence: agent panes redraw on their own (spinners, token counters, clocks), so a diff-based check reports success within ~150ms of the keystroke whatever happened — which silently masked agent builds that ignore an injected Ctrl+V, reporting `verified` while delivering nothing. When no placeholder appears, the request falls through to path injection, so those panes still get the image. A false negative (an agent whose placeholder we don't recognize) costs a duplicate rather than a loss.

Once a mode is settled the seed is cleared from the OS clipboard (`osascript` / `wl-copy --clear` / empty `xclip` selection) — after confirmed ingestion, or after ruling it out. The seed is a one-image handoff: TUI agents re-read the clipboard on *any* paste event, so a client that pastes prompt text after the image makes the agent attach it twice, and on macOS a seeded pasteboard is broadcast to the user's other devices over Universal Clipboard. Callers should not assume the image is still on the clipboard after the response.

Moshi disables tmux pane capture before execution on affected Enterprise Linux 10 RPMs. There the baseline read fails, so clipboard mode returns `verified: false` and keeps the seed — nothing can be confirmed, and both path injection and a clear would risk making it worse. Errors: `404` when no live session matches the lookup, `422` when the session is a bare shell or the pane/session can't be identified (callers should fall back to their SSH paste path).

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
