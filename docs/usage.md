# `moshi-hook` Usage

```bash
moshi-hook pair --token <pairing-token>   # 1. register this host with Moshi
moshi-hook install                        # 2. write hook configs for installed agents
moshi-hook serve                          # 3. run the daemon (or `brew services start moshi-hook`)
```

After step 2, `claude` / `codex` / `opencode` route their hooks through `moshi-hook`. The daemon (`serve`) holds the WebSocket to Moshi and the local Unix socket that hooks talk to.

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
| `XDG_STATE_HOME` / `XDG_CONFIG_HOME` / `XDG_RUNTIME_DIR` | Standard XDG dirs (Linux). |

## Subcommands

| Command | What it does |
|---|---|
| `pair --token <t> [--name <n>]` | Register this host. Token comes from the Moshi mobile app. Re-running rotates the secret. |
| `unpair` | Remove host secret + delete the host on the server. |
| `install` | Write Moshi entries into `~/.claude/settings.json`, `~/.codex/config.toml`, and `.opencode/plugins/moshi-hooks.ts`. Non-destructive: leaves user-owned hooks alone. |
| `uninstall` | Remove Moshi-owned entries from those files. |
| `serve` | Run the daemon in the foreground. Single-instance via `flock` on a lockfile next to the socket. |
| `status [--json]` | Pairing state, paths, WS connection. |
| `usage [--sync]` | Cached usage snapshots. `--sync` pushes them to the server. |
| `logs [-f]` | Tail the daemon log. |
| `version` | Version, commit SHA, build date. |

Hidden subcommands (`claude-hook`, `codex-hook`, `opencode-event`, `opencode-permission`) are invoked by the agents themselves through the configs `install` writes — you won't run them by hand.

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
| Secrets | Keychain (`app.getmoshi.hook`) | `<state>/secrets.json` (0600) |

When run via `brew services`, logs land at `$(brew --prefix)/var/log/moshi-hook.log`.
