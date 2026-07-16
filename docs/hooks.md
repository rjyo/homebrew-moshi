# Agent Hooks

High-level reference for which agent events Moshi surfaces in the inbox.

This document is intentionally not an implementation guide. Keep it focused on
product behavior and public integration concepts. Do not add local filesystem
paths, socket names, endpoint shapes, transcript formats, token keys, terminal
control details, or approval automation mechanics.

---

## What We Capture

Moshi normalizes agent-specific events into a small category model. Events
without one of these categories are acknowledged locally and are not published.

| Category | Source behavior | Inbox effect |
| --- | --- | --- |
| `approval_required` | Agent asks the user for a decision or answer | Upsert a pending approval row |
| `task_complete` | Agent turn finishes or is interrupted | Upsert a completed row |
| `session_started` | A user prompt starts or resumes work | Upsert a running row |
| `session_ended` | Session is cleared or reset | Clear active running state |
| `tool_running` | Agent starts tool activity | Upsert a running row |
| `tool_finished` | Agent finishes tool activity | Upsert a running row |

Inbox behavior:

- One active row per `sessionId`; newer events replace older row content.
- Pending approvals render as approval cards.
- Non-pending events render as task rows with different title/subtitle text.
- Completed rows age out of the active list after a short freshness window.

---

## Per-Agent Coverage

### Claude Code

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Tool activity | Publishes throttled tool progress |
| Permission request | Publishes `approval_required` |
| Agent stops | Publishes `task_complete` |
| Session ends or clears | Ignored; the turn result or next prompt carries the visible state |
| User interrupt | Publishes `task_complete`; if the user immediately gives new instructions, publishes `session_started` again |
| Agent asks the user a question | Publishes `approval_required`; when answered outside Moshi, follows with a resumed running state |

Claude approvals remain compatible with Claude's own terminal flow. Moshi may
offer a remote approval experience when the local environment can be verified.
Multiple Claude profiles remain isolated: Chat View follows the session's own
transcript, while usage and inbox account labels use that profile's identity.
Claude-specific debug, notification, subagent, batch, and compaction events are
not surfaced unless they map cleanly to the category model above.

### Codex CLI

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Tool activity | Publishes throttled tool progress |
| Permission request | Publishes `approval_required` |
| Agent stops | Publishes `task_complete` |
| Session clears | Publishes `session_ended` before the next visible turn |
| User interrupt | Publishes `task_complete` when detected |

Codex approvals use the same native-terminal model: Moshi may offer remote
approval when the local environment can be verified, but the agent's terminal
prompt remains compatible.

### OpenCode

| Agent behavior | Moshi behavior |
| --- | --- |
| Session created or marked busy | Publishes `session_started` |
| Session becomes idle | Publishes `task_complete` |
| Permission request | Publishes `approval_required` |
| Tool activity | Publishes tool progress |

Streaming messages, file events, LSP events, TUI events, and miscellaneous
server or command notifications are not surfaced.

### Grok Build

Grok Build follows the same broad behavior as Claude-compatible hooks:

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Permission request | Publishes `approval_required` |
| Agent stops | Publishes `task_complete` |
| Session ends | Publishes `session_ended` to clear active running state |
| Tool activity | Supported only when explicitly enabled |

### OMP (Oh My Pi)

OMP uses its TypeScript extension API. Moshi installs a global extension and
keeps the default coverage to low-volume lifecycle and native approval events.
The extension also records OMP's stable session id and exact v3 JSONL path so
Chat View can follow the active profile and XDG/custom agent directory:

| Agent behavior | Moshi behavior |
| --- | --- |
| Session loads | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Main session settles | Publishes `task_complete` |
| Permission request | Publishes `approval_required`; the native terminal remains the source of truth |
| Session shuts down | Publishes `session_ended` |
| Tool activity | Not installed by default |

### Pi

Pi uses its TypeScript extension API. Moshi installs a global extension module
and keeps the default coverage to the same low-volume lifecycle events as OMP.
The extension also records Pi's stable session id and exact JSONL path so Chat
View can follow sessions stored under either the default or a custom session
directory:

| Agent behavior | Moshi behavior |
| --- | --- |
| Session loads | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Agent fully settles | Publishes `task_complete` after retries, compaction, and queued follow-ups finish |
| Session shuts down | Publishes `session_ended` |
| Tool activity | Not installed by default; Pi tool hooks are synchronous and should stay opt-in |

### Hermes Agent

Hermes uses a user plugin enabled through its normal plugin configuration. The
plugin observes lifecycle and approval events but never approves, denies, or
blocks Hermes itself.

| Agent behavior | Moshi behavior |
| --- | --- |
| New session | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Agent completes or is interrupted | Publishes `task_complete` |
| Approval request in the interactive CLI/TUI | Publishes `approval_required` |
| Approval resolves | Clears the pending action without publishing another completion |
| Session finalizes | Publishes `session_ended` |

Hermes keeps ownership of its approval prompt. Moshi may answer that prompt
remotely only when its terminal bridge can verify the visible command and
approval menu. Smart-mode and gateway decisions are observed by Hermes alone
and are not exposed as terminal actions.

### Antigravity

Antigravity uses its global command-hook configuration. Moshi registers only
model-invocation and stop lifecycle hooks; it does not register a tool-policy
hook or change Antigravity's native approval behavior.

| Agent behavior | Moshi behavior |
| --- | --- |
| Model invocation begins | Publishes or updates `session_started` once per visible turn |
| Agent becomes fully idle | Publishes `task_complete` |
| Prompt and result text | Extracted best-effort when the available transcript contains it |
| Tool approval | Not intercepted |

### Cursor CLI

| Agent behavior | Moshi behavior |
| --- | --- |
| User submits a prompt | Publishes or updates `session_started` |
| Permission request (shell command, MCP call, or file read) | Publishes `approval_required` |
| File edit | Publishes file-edit progress |
| Agent responds | Stored silently; the turn result carries the visible state |
| Agent stops | Publishes `task_complete` |

Cursor approvals use the same native-terminal model: Moshi may offer remote
approval when the local environment can be verified, while the terminal prompt
remains compatible. Cursor `--force` / `--yolo` actions stay local and do not
create approval rows because Run Everything has already removed the human
decision. Reasoning traces, editor-internal events, file watchers, and streaming
partials are not surfaced.

---

## Events We Do Not Surface

Drop events that do not create a meaningful user-facing inbox state.

| Event family | Reason |
| --- | --- |
| Empty session start | Avoid blank inbox rows |
| Streaming message partials | Too chatty for push and inbox |
| File, LSP, editor, or TUI events | Local implementation detail |
| Subagent lifecycle events | Parent turn carries the user-visible signal |
| Batch/debug/diagnostic events | No direct user action |
| Config, environment, or worktree events | Local state, not agent activity |
| Reasoning trace events | Not appropriate for inbox |
| Compaction events | Not surfaced today |

---

## Filter Rules

Use this rule set when changing hooks or adding a new agent:

1. Capture lifecycle boundaries: prompt starts, turn completes, session ends,
   and approval requests.
2. Capture tool activity only through throttled progress updates.
3. Treat user questions as approvals or pending-answer rows only when they
   require user action.
4. Preserve one-row-per-session behavior.
5. Prefer dropping new event types over adding new categories.
6. Keep implementation details out of this document.

The inbox shape is deliberate. New agents should map into the existing
categories instead of expanding the event model by default.
