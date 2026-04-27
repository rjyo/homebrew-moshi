# `moshi-hook` API Reference

Wire protocols `moshi-hook` participates in. Three surfaces:

1. **Local socket** — hooks ↔ daemon. Newline-delimited JSON over Unix socket.
2. **Moshi HTTP** — daemon ↔ Moshi server. JSON over HTTPS, bearer auth.
3. **Moshi WebSocket** — daemon ↔ Moshi server. JSON frames, bearer on upgrade.

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
  "tmuxSession": "main",
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

Base URL: `https://api.getmoshi.app/api/v1` (override with `--base-url` / `MOSHI_API_BASE`). Auth: `Authorization: Bearer <hostSecret>` on every endpoint except `register`, which uses the pairing token.

### `POST /hosts/register`

Pair this host. Authenticated with the **pairing token**.

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
