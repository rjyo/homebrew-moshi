# `moshi-hook` Usage

```bash
moshi-hook pair --token <pairing-token>   # 1. pair the agent-hooks daemon
moshi-hook install                        # 2. write hook configs for installed agents
moshi-hook serve                          # 3. run the daemon (or `brew services start moshi-hook`)
```

After step 2, supported agents route their hooks through `moshi-hook`: Claude Code, Codex, OpenCode, Gemini CLI, Cursor, Kimi, and Qwen Code. The daemon (`serve`) holds the WebSocket to Moshi and the local Unix socket that hooks talk to.

## Host Easy Pair

Easy Pair is the QR-based setup flow for SSH/Mosh host access and agent-hooks daemon pairing. Run it on the machine you want Moshi to connect to:

```bash
moshi-hook host setup
```

The command prints an Easy Pair QR. Scan it from Moshi onboarding or the iOS Camera. Moshi creates the saved host connection, generates the phone-side private key, and sends only the public key plus a host-scoped daemon secret to the setup session. The host adds the public key to `authorized_keys` and stores the daemon secret in the configured secret store.

Treat the QR like a temporary access token. Anyone who scans it before it expires can claim SSH access and pair Moshi services for this host. Do not share your screen, screenshot it, or paste the setup link.

Useful host commands:

| Command | What it does |
|---|---|
| `host setup [--name <n>] [--port <p>] [--user <u>] [--force]` | Start an Easy Pair setup session, print the QR, and pair this daemon after claim. |
| `host list` | List local Moshi SSH/Mosh pairings installed on this host. |
| `host revoke <id>` | Remove a Moshi host public key from `authorized_keys`. |
| `host enable-ssh` | On macOS, open/enable Remote Login prerequisites where supported. |

`moshi-hook pair --token` is still available for manual daemon re-pairing. Easy Pair does not store the phone's user token on the host; it stores the host-scoped `hostSecret` used by inbox, Live Activity, Apple Watch events, usage sync, and approvals.

By default, `host setup` picks the best reachable connection target it can detect. If the `tailscale` CLI is on PATH and reports `Running`, it uses your Tailscale MagicDNS name (or IPv4 fallback). Otherwise on Linux it uses the first usable private LAN IPv4; on macOS it uses Bonjour (`.local`) or the OS hostname. If the chosen address is wrong for your network, edit the host on the iOS app after pairing.

On macOS, `pair` stores secrets in Keychain by default. If you are pairing from SSH or another session where Keychain is locked or unavailable, either unlock the login keychain first:

```bash
security unlock-keychain ~/Library/Keychains/login.keychain-db
moshi-hook pair --token <pairing-token>
```

or deliberately use file-backed storage:

```bash
moshi-hook pair --token <pairing-token> --store file
```

File-backed storage writes secrets to `~/.config/moshi/secrets.json` with `0600` permissions. The selected store is remembered for future `serve`, `status`, `usage --sync`, and `pair` commands.

## Global flags

| Flag | Description | Default |
|---|---|---|
| `-v`, `--verbose` | Debug-level logging | off |
| `--base-url URL` | Override Moshi API base | `https://api.getmoshi.app/api/v1` |

## Environment variables

| Variable | Description |
|---|---|
| `MOSHI_PAIRING_TOKEN` | Pairing token (alternative to `--token`). |
| `MOSHI_API_BASE` | API base URL (alternative to `--base-url`). |
| `MOSHI_SOCKET_PATH` | Override the Unix socket path. |
| `MOSHI_STATE_DIR` / `MOSHI_CONFIG_DIR` | Override state/config dirs. |
| `CLAUDE_CONFIG_DIR` | Override Claude Code config dir for Claude hook install/status/uninstall. |
| `XDG_STATE_HOME` / `XDG_CONFIG_HOME` / `XDG_RUNTIME_DIR` | Standard XDG dirs (Linux). |

## Subcommands

| Command | What it does |
|---|---|
| `pair --token <t> [--name <n>] [--store keychain\|file]` | Pair the agent-hooks daemon. Token comes from the Moshi mobile app. Re-running rotates the secret. macOS defaults to `keychain` until a store preference is saved; use `file` for headless sessions. |
| `unpair` | Remove the agent-hooks secret and server registration. |
| `host setup` | Start an Easy Pair SSH/Mosh setup session and print a QR for Moshi to scan. |
| `host list` | List local Moshi host SSH pairings. |
| `host revoke <id>` | Remove a Moshi host SSH key from `authorized_keys`. |
| `host enable-ssh` | Help enable SSH prerequisites on macOS. |
| `install` | Write Moshi entries into supported agent config files. Use `--target claude,codex,opencode,gemini,cursor,kimi,qwen` to limit the set. Non-destructive: leaves user-owned hooks alone. OpenCode installs globally by default; use `--local` for `.opencode/plugins` in the current project. |
| `uninstall` | Remove Moshi-owned entries from those files. For OpenCode, pass `--local` to remove a project-local install. |
| `serve` | Run the daemon in the foreground. Single-instance via `flock` on a lockfile next to the socket. |
| `status [--json]` | Pairing state, paths, WS connection. |
| `usage [--sync]` | Cached usage snapshots. `--sync` pushes them to the server. |
| `cwd-list [--json] [--limit N]` | Recent project working directories from local agent state (Claude, Codex, Cursor). Plain-text table by default; `--json` emits the shape the iOS preflight consumes. |
| `logs [-f]` | Tail the daemon log. |
| `version` | Version, commit SHA, build date. |

Hidden subcommands (`claude-hook`, `codex-hook`, `opencode-event`, `opencode-permission`, `gemini-hook`, `cursor-hook`, `kimi-hook`, `qwen-hook`) are invoked by the agents themselves through the configs `install` writes — you won't run them by hand.

### `cwd-list` — recent project directories

Scans local agent state for the working directories you've used recently and prints them deduped + ranked by recency. Used by the Moshi iOS app at connection time to offer one-tap "jump into a recent project" entries when no tmux/zellij session exists on the host.

Sources covered:

| Source | Where it reads from | How cwd is recovered |
|---|---|---|
| `claude` | `~/.claude/projects/<encoded>/*.jsonl` | Authoritative `cwd` field inside the transcript (folder names are ambiguous on hyphenated paths). |
| `codex` | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | `session_meta.payload.cwd` on line 1. |
| `cursor` | `~/.cursor/projects/<encoded>/` | Prefix-DFS over the encoded folder name; at each `-` it tries both `/` and a literal hyphen, recursing only into directories that exist. Resolves real paths like `…/ghostty-android` correctly. |

Non-existent paths are filtered out, sources for the same cwd are merged (so a project you've touched in both Claude and Codex appears once with both sources listed), and the list is capped at `--limit` (default 10). The iOS preflight calls this with `--json`; pass it manually for a quick "what does Moshi think I've been working on" check.

```bash
$ moshi-hook cwd-list --limit 5
claude,codex   /Users/jyo/projects/ai/moshi/app-ios
claude         /Users/jyo/projects/ai/poc-me
codex,claude   /Users/jyo/projects/ai/moshi/app-hook
cursor         /Users/jyo/projects/ai/mbb-app
codex,claude   /Users/jyo/projects/ai/moshi/app-android
```

Read-only: no network, no writes. Best-effort by design — a missing agent dir is silently skipped so a single broken source can't blank the list.

## Installed agent files

| Agent | Managed file |
|---|---|
| Claude Code | `~/.claude/settings.json` |
| Codex | `~/.codex/hooks.json` plus `~/.codex/config.toml` feature flag |
| OpenCode | `~/.config/opencode/plugins/moshi-hooks.ts` by default, or `.opencode/plugins/moshi-hooks.ts` with `--local` |
| Gemini CLI | `~/.gemini/settings.json` |
| Cursor | `~/.cursor/hooks.json` |
| Kimi | `~/.kimi/config.toml` |
| Qwen Code | `~/.qwen/settings.json` |

Kimi's managed install uses the verified Kimi hook set and does not install `PermissionRequest` by default. The `kimi-hook` dispatcher can still handle `PermissionRequest` if a user adds that event manually.

## Tool-event hooks (opt-in)

`install` does **not** wire `PreToolUse` / `PostToolUse` for Claude or Codex. They fire on every tool call (10–20 per turn) but the inbox row only renders one event at a time, so most users prefer the quieter default of just session start, prompts, approvals, and turn end.

If you want a "Running Bash …" row to appear mid-turn, add the entry by hand. Re-running `moshi-hook install` won't touch user-added entries.

**Claude** — append to `~/.claude/settings.json` under `hooks`:

```json
"PreToolUse": [
  { "hooks": [{ "type": "command", "command": "/opt/homebrew/bin/moshi-hook claude-hook", "async": true }] }
],
"PostToolUse": [
  { "hooks": [{ "type": "command", "command": "/opt/homebrew/bin/moshi-hook claude-hook", "async": true }] }
]
```

**Codex** — append to `~/.codex/hooks.json` under `hooks`:

```json
"PreToolUse": [
  { "hooks": [{ "type": "command", "command": "/opt/homebrew/bin/moshi-hook codex-hook" }] }
],
"PostToolUse": [
  { "hooks": [{ "type": "command", "command": "/opt/homebrew/bin/moshi-hook codex-hook" }] }
]
```

Replace `/opt/homebrew/bin/moshi-hook` with `which moshi-hook` if installed elsewhere. The dispatcher already throttles tool events to one push per 5 s per session.

## Paths

| What | macOS | Linux |
|---|---|---|
| State + log | `~/Library/Application Support/Moshi/` | `$XDG_STATE_HOME/moshi/` |
| Socket | `<state>/moshi-hook.sock` | `$XDG_RUNTIME_DIR/moshi-hook.sock` |
| Secrets | Keychain (`app.getmoshi.hook`) by default; `~/.config/moshi/secrets.json` with `--store file` | `<state>/secrets.json` (0600) |

When run via `brew services`, logs land at `$(brew --prefix)/var/log/moshi-hook.log`.
