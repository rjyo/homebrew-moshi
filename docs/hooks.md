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
| Agent asks the user a question | Publishes `approval_required`; Chat View can submit supported choices through the verified TUI bridge, then follows with a resumed running state |

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

Codex lifecycle delivery is hybrid. Hooks report normal prompts, permissions,
and stops, while the daemon also follows Codex's native session log for
interruptions, input requests, and completion fallback when no hook arrives.

### OpenCode

| Agent behavior | Moshi behavior |
| --- | --- |
| Session created or marked busy | Publishes `session_started` |
| Session becomes idle | Publishes `task_complete` |
| Permission request | Publishes `approval_required` |
| Question (V1 `question.asked`, V2 form) | Publishes `approval_required` with "Answer in terminal" |
| Tool activity | Publishes tool progress |

Streaming messages, file events, LSP events, TUI events, and miscellaneous
server or command notifications are not surfaced.

### Grok Build

Requires the Moshi app 3.10.2 or newer.

Grok Build follows the same broad behavior as Claude-compatible hooks:

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Permission request | Publishes only after the daemon verifies Grok parked on a human; Moshi can then approve once or reject through the native UI |
| Agent stops | Publishes `task_complete` |
| Session ends | Publishes `session_ended` to clear active running state |
| Agent asks the user a question | Publishes `approval_required`; Chat View can submit supported choices through the verified TUI bridge |
| Tool activity | Broad `PreToolUse` detects permission panels; auto-approved tools are discarded before publication, while `PostToolUse` stays targeted to `ask_user_question` |

Grok's hook protocol has no permission event — `PreToolUse` fires for every
tool, before the permission system decides — so a candidate is verified against
`events.jsonl`, the lifecycle log Grok writes beside its ACP transcript. A
`permission_requested` with nothing after it means a human is being waited on;
an auto-approved tool records `permission_resolved` in the same millisecond.
Moshi briefly settles an unresolved request before publishing it so it cannot
race between those two adjacent writes and push a phantom approval.
Panes are the fallback when no log is readable. Verification runs off the
socket handler because Grok holds the tool until `PreToolUse` hooks return: the
hook waits on its ack, so waiting for the panel before acking would deadlock
against the very prompt being waited for. The same log makes a session report
`blocked` while the panel is up, without scraping the pane.

Grok's first permission option enables always-approve mode, and its edit and
scoped-command panels add "allow all edits", "Always allow" and "Never allow"
rules around the one-time choices. Moshi never sends those: remote approve
selects the one-time option (`Yes, proceed`, or `Yes` on the edit panel) and
deny selects `No, reject`, reading the digits off the panel on screen.

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
| Permission request | Publishes `approval_required`; a terminal answer follows with `OMP resumed` |
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
| Permission request | Publishes `approval_required`; a terminal answer follows with `Pi resumed` |
| Session shuts down | Publishes `session_ended` |
| Tool activity | Not installed by default; Pi tool hooks are synchronous and should stay opt-in |

### OmO

OmO (oh-my-openagent's standalone `omo` CLI) runs on senpi, a Pi fork, and
loads the same kind of TypeScript extension from its own agent directory.
Moshi installs a global extension there with Pi's lifecycle coverage. Senpi
prompts for permissions only when a permission preset asks it to; those
prompts arrive on its event bus and are mirrored as answer-in-terminal rows.

| Agent behavior | Moshi behavior |
| --- | --- |
| Session loads | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Agent fully settles | Publishes `task_complete` |
| Permission prompt | Publishes `approval_required` with "Answer in terminal"; the answer follows with `OmO resumed` |
| Session shuts down (`/new`, `/resume`, exit) | Publishes `session_ended` |
| Conversation | Chat View from its session file |

### jcode

jcode runs lifecycle hooks from its `[hooks]` config table. Moshi adds its
command to `session_start`, `turn_start`, `turn_end`, and `session_end`,
keeping any commands you already configured for those events. jcode has no
interactive approval or question prompt, so none is surfaced. Its hooks run
from jcode's shared background server, so Moshi only follows sessions a
terminal client is driving; swarm workers and ambient runs stay out of the
inbox.

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts, attaches or resumes | Stored silently until the first prompt |
| A turn starts | Publishes or updates `session_started` with the prompt |
| A turn ends (including Esc) | Publishes `task_complete` with the final reply, or the error |
| Esc during a tool call (jcode aborts the turn without its hook) | Publishes `task_complete` titled `jcode interrupted` from jcode's own log |
| `/clear` or client exit | Publishes `session_ended` |
| Conversation | Chat View from its saved session |

### Goose

Goose runs Open Plugins hooks. Moshi installs a `moshi-hooks` plugin in
Goose's user plugin directory for the session, prompt and stop events, and
sets Goose's status hook so it can tell when the CLI is back at its prompt.
If you already use your own status hook, Moshi leaves it in place; turns you
interrupt then stay working until the next prompt, and a `/new` session is
picked up on its first prompt. Goose has no hook for its tool approval prompt,
so approvals are not surfaced.

| Agent behavior | Moshi behavior |
| --- | --- |
| Session starts | Stored silently until the first prompt |
| User submits a prompt | Publishes or updates `session_started` |
| Turn finishes | Publishes `task_complete` with the final reply |
| Ctrl-C during a turn | Publishes `task_complete` titled `goose interrupted` |
| `/new` | Ends the old session and follows the new one right away |
| Session exits | Publishes `session_ended` |
| Conversation | Chat View from Goose's session store |

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

Antigravity CLI 1.2.5 supports full Chat View through its native SQLite
conversation store. In Herdr, Moshi follows `/new` before the first prompt.
Escape interrupts the active response; Antigravity records this in its CLI log
without calling `Stop`, so Moshi watches that native record to retire the turn.
Background tasks retain Antigravity's native Escape behavior.

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
| Permission request (shell command or MCP call) | Publishes `approval_required`, but only once the terminal shows Cursor's prompt |
| Auto-approved tool call (allowlisted command, any file read) | Nothing published |
| File edit | Publishes file-edit progress |
| Agent responds | Stored silently; the turn result carries the visible state |
| Agent stops | Publishes `task_complete` |

Cursor approvals use the same native-terminal model: Moshi may offer remote
approval when the local environment can be verified, while the terminal prompt
remains compatible. Cursor's permission hooks run ahead of its own decision and
fire for every tool call, so Moshi waits for Cursor's matching
`afterShellExecution` / `afterMCPExecution` before deciding: work that finishes
promptly was resolved by policy and stays silent, while a call Cursor goes quiet
on is one it is asking you about. A turn that reads a thousand files or runs a
hundred allowlisted commands therefore produces no notifications. Cursor `--force` / `--yolo`
actions stay local for the same reason: Run Everything has already removed the
human decision. Reasoning traces, editor-internal events, file watchers, and
streaming partials are not surfaced.

### Qoder CLI

Qoder CLI copies Claude Code's hook protocol, so Moshi installs the same event
set into `~/.qoder/settings.json` (or `$QODER_CONFIG_DIR/settings.json`).
Chat View reads Qoder's Claude-format JSONL from the exact path its hooks
report.

| Agent behavior | Moshi behavior |
| --- | --- |
| User submits a prompt | Publishes or updates `session_started` |
| Permission request | Publishes `approval_required`; a remote allow or deny answers Qoder |
| Permission answered in the terminal | The remote request closes as "Answered in terminal" when the turn ends |
| `AskUserQuestion` | Captures the form; Chat View answers it through the pane |
| Agent stops | Publishes `task_complete` with the final reply |
| Subagent lifecycle (payload carries `agent_id`) | Not surfaced |

Qoder runs no hooks at all in a folder it has not been told to trust, and a
prompt passed on the command line skips its trust dialog. Trust the folder once
(start `qodercli` without a prompt) or Moshi sees nothing from that project.

Qoder shows its own prompt while the permission hook waits and does not cancel
the hook when the terminal answer wins, so Moshi retires that request on the
session's next turn event.

### Devin CLI

Devin reads Claude-shaped hook groups from the `hooks` key of
`~/.config/devin/config.json`. Its matchers are regexes, so lifecycle entries
omit them (Devin treats an omitted matcher as "all") and the question hooks
name `ask_user_question` literally. Devin has no `cwd` in its payloads; Moshi
uses `DEVIN_PROJECT_DIR` to bind the terminal pane.

| Agent behavior | Moshi behavior |
| --- | --- |
| User submits a prompt | Publishes or updates `session_started` |
| Permission request | Publishes `approval_required`; Devin waits for the hook, so the remote allow or deny answers it (`{"decision":"approve"\|"block"}`) |
| `ask_user_question` | Captures the form and publishes `approval_required`; Chat View answers a single question through the pane |
| Agent stops | Publishes `task_complete` |

Chat View reads Devin's per-session ATIF document
(`~/.local/share/devin/cli/transcripts/<id>.json`), converted into
Claude-shaped rows by the gateway.

Devin also loads `~/.claude` hooks by default (`read_config_from.claude`).
Moshi's Claude hook stays inert whenever `DEVIN_PROJECT_DIR` is set, so those
imported copies never surface Devin turns as Claude sessions.

### Amp

Amp has no shell hooks. Moshi installs a Bun plugin at
`~/.config/amp/plugins/moshi-hooks.ts` that forwards `agent.start` and
`agent.end` (with the final assistant text) to `moshi-hook amp-hook`. Plugins
cannot see Amp's own approval prompt, so approvals are not surfaced.

Amp keeps threads on its servers, so Chat View runs `amp threads export` while
a Chat View stream is open. The plugin touches a per-thread marker on every
turn and tool result; the gateway re-exports only when that marker moves (or
every 30s), and viewers of one thread share a single cached export.

| Agent behavior | Moshi behavior |
| --- | --- |
| User submits a prompt | Publishes or updates `session_started` |
| Agent finishes the turn | Publishes `task_complete` with the final reply |
| Tool approval | Not surfaced |
| Conversation | Chat View via `amp threads export`, refreshed on Amp activity |

### Factory Droid and GitHub Copilot CLI

Neither exposes a hook that can answer an approval before its own policy
runs, so approvals stay in the terminal. Copilot also gets Chat View: the
gateway rewrites its `events.jsonl` conversation into Claude-shaped rows.
Droid's session JSONL already uses Claude content blocks, so Chat View unwraps
it and hides TUI-only rows (hook status lines, plan notices).

| Agent behavior | Moshi behavior |
| --- | --- |
| User submits a prompt | Publishes or updates `session_started` |
| Waiting on a permission prompt or question (`Notification`) | Publishes `approval_required` with "Answer in terminal" |
| A tool runs after the prompt | Clears the waiting state without publishing |
| Agent stops | Publishes `task_complete` |

Droid hooks live in `~/.factory/hooks.json`. Because Droid lets that file
replace `settings.json` hooks event by event, install first copies any
`settings.json` hooks for the events Moshi adds. Droid subagents (run with
`DROID_PARENT_SESSION_ID`) are not surfaced. Copilot hooks live in a
Moshi-owned `~/.copilot/hooks/moshi-hooks.json`; Moshi does not install
Copilot's `permissionRequest`, which fires before Copilot's own allow/deny
policy and so cannot tell a real prompt from an auto-approved tool.

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
