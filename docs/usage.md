# `moshi-hook` Usage

```bash
moshi-hook pair --token <pairing-token>   # 1. pair the agent-hooks daemon
moshi-hook install                        # 2. write hook configs for installed agents
moshi-hook service install                # 3. run persistently on Linux or Windows
moshi-hook serve                          #    foreground fallback (or `brew services start moshi-hook` on macOS)
```

Native Windows (**experimental**) bootstrap:

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://getmoshi.app/install.ps1 | iex"
```

The PowerShell installer verifies the published ZIP checksum, installs without
elevation into versioned per-user directories, and places a stable junction on
the user `PATH`. See [windows.md](windows.md) for the beta support boundary.

After step 2, supported agents route their hooks through `moshi-hook`: Claude Code, Codex, OpenCode, Gemini CLI, Antigravity, Cursor, Kimi, Qwen Code, Grok Build, OMP (Oh My Pi), Pi, and Hermes Agent. The daemon (`serve`) holds the WebSocket to Moshi and the local Unix socket that hooks talk to.

## Local web client

Run the embedded web client in the foreground with the `moshi` alias and no
arguments. It opens the default browser at `http://127.0.0.1:24544` after the
listener is ready:

```bash
moshi
```

Pass `moshi --no-open` to print the URL without opening a browser.

Pass `moshi --listen 0.0.0.0:24544` to bind the web client on all interfaces
so other devices on your network can reach it. The daemon gateway stays on
loopback; only the web listener is exposed, and anyone who can reach it has
full control of the daemon, so only do this on trusted networks.

The foreground web process serves static UI assets and proxies API and
WebSocket requests to the daemon on `127.0.0.1:24543`. If a persistent
`moshi serve` daemon is already running, the web client reuses it. Otherwise,
the web client starts a temporary daemon automatically and stops it when the
web client exits. API requests log at INFO, static assets at DEBUG (`moshi -v`),
and proxy/server failures at ERROR.

## Project tmux launcher

The installed package also exposes `moshi` as a convenience alias for `moshi-hook`. Passing one directory argument starts or attaches a tmux session for that project:

```bash
moshi .
moshi ~/a/b/name
moshi diff .
```

The session name is the directory basename (`name` above). Moshi resolves the directory, then replaces itself with:

```bash
tmux new-session -A -s name -c /absolute/path/to/name
```

Because this uses `exec`, no Moshi wrapper process stays alive after tmux starts. tmux has no native
Windows build, so this launcher refuses to run there — see [windows.md](windows.md).

### Enterprise Linux 10 and tmux 3.3a

RHEL, AlmaLinux, Rocky Linux, and related Enterprise Linux 10 distributions ship a downstream tmux patch that can corrupt or abort the entire tmux server on the first `capture-pane` call. The affected patch was introduced in the `tmux-3.3a-12` RPM; `3.3a-13` is also affected. The CentOS Stream `3.3a-14` change protects the print path but leaves the non-print buffer path unsafe, so Moshi treats downstream EL10 `3.3a-12` through `3.3a-14` as unsafe.

Do not use `tmux -V` to identify this bug: the affected snapshot can report `next-3.4`. Inspect the installed RPM instead:

```bash
rpm -q tmux
```

Before its first pane capture, Moshi checks the installed tmux RPM. It disables every `capture-pane` feature on affected EL10 builds. Hook-, title-, and transcript-based status continues to work; screen-only prompt detection and pane-based verification are unavailable. Serialization, caching, timeouts, and retry backoff remain enabled for unrelated failures on safe builds.

Do not assume a normal package upgrade has removed the downstream patch. Until your distribution publishes a complete fix, the practical workaround is to remove the affected RPM and install an upstream tmux build (3.5a is known to work). Replacing the executable does not update an already-running tmux server: preserve or finish important work, then restart that server at a planned time.

## Git diff viewer

The `moshi` alias can also open a local browser-based diff viewer for a Git project:

```bash
moshi diff .
moshi diff ~/a/b/name --no-open
```

The viewer is embedded in `moshi-hook`, starts a localhost-only HTTP server, and reads Git state directly from the selected directory. Diff contents stay on the host. The default port is stable (`24543`); running `moshi diff` again for another workspace updates the existing diff server and reopens the same local URL. If the daemon gateway already owns that port, `moshi diff` falls back to a free ephemeral port. Pass `--port 0` to force an ephemeral free port.

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
| `moshi [--no-open] [--listen <addr>]` (no path) | Run the embedded web client on `127.0.0.1:24544` (or `--listen`, e.g. `0.0.0.0:24544` for all interfaces) until Ctrl-C and open it in the default browser unless `--no-open` is set. Reuses a persistent daemon or starts a temporary one automatically, proxies API/WebSocket traffic to it, and prints request logs. |
| `host setup [--name <n>] [--host <h>] [--port <p>] [--user <u>] [--force]` | Start an Easy Pair setup session, print the QR, and pair this daemon after claim. |
| `host list` | List local Moshi SSH/Mosh pairings installed on this host. |
| `host revoke <id>` | Remove a Moshi host public key from `authorized_keys`. |
| `host enable-ssh` | On macOS, open/enable Remote Login prerequisites where supported. |

`moshi-hook pair --token` is still available for manual daemon re-pairing. Easy Pair does not store the phone's user token on the host; it stores the host-scoped `hostSecret` used by inbox, Live Activity, Apple Watch events, usage sync, and approvals.

By default, `host setup` shows an address selector before generating the QR. Use up/down, `1..n`, or Enter to choose a detected address, or choose the final option to type a public IP/hostname for VPS and cloud hosts. Pass `--host <hostname-or-ip>` to skip the selector in scripts. Non-interactive and `--json` runs keep the old automatic first match: Tailscale MagicDNS/IPv4, then LAN IPv4 on Linux, then Bonjour/hostname fallback.

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
| `MOSHI_HERDR_PATH` | Absolute path to the `herdr` CLI when a service manager cannot discover it from `PATH`. |
| `MOSHI_HOOK_CDN` | CDN base URL for `update` downloads. |
| `MOSHI_HOOK_GATEWAY_LISTEN` | Override the host gateway listen address. |
| `MOSHI_STATE_DIR` / `MOSHI_CONFIG_DIR` | Override state/config dirs. On macOS either override also isolates the secret store: pairing secrets come from `secrets.json` inside that dir instead of the login Keychain, so a scratch or e2e daemon starts unpaired rather than inheriting the real host's identity. Run `pair --store keychain` in that dir to opt back in. |
| `MOSHI_HOOK_CONFIG_DIR` | Override the config dir for `config.toml` gateway settings. |
| `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` | Proxy usage-provider HTTP requests. On macOS, when these are unset, usage polling falls back to the active system HTTP/HTTPS proxy reported by `scutil --proxy`, including for launchd/Homebrew services. |
| `CLAUDE_CONFIG_DIR` | Override the Claude Code profile used by hook install/status/uninstall; active hooked sessions also register this profile for Chat View, usage, and account labels. |
| `CODEX_HOME` | Override the Codex home used for hook install/status/uninstall, session-log lifecycle monitoring, Chat View transcripts, and usage collection. Keyring-backed Codex usage is read through Codex's local app-server; file-backed `auth.json` and OpenCode OAuth remain supported. Lifecycle monitoring reads the daemon's environment at startup; restart the daemon after changing it. |
| `OPENCODE_CONFIG_DIR` | Override OpenCode global config/plugin dir for hook install/status/uninstall. |
| `ANTIGRAVITY_CONFIG_DIR` | Override Antigravity's global `config` directory for hook install/status/uninstall. |
| `OMP_PROFILE` / `PI_PROFILE` | Select the active named OMP profile (`OMP_PROFILE` takes precedence). |
| `PI_CODING_AGENT_DIR` / `PI_CONFIG_DIR` | Override Pi/OMP agent and config directories; Chat View follows the exact transcript path reported by their installed extensions. |
| `PI_CODING_AGENT_SESSION_DIR` | Override Pi's session storage directory. |
| `CURSOR_CONFIG_DIR` | Override Cursor config dir for hook install/status/uninstall. |
| `KIMI_CODE_HOME` | Override the current Kimi Code data/config directory for hooks, Chat View transcripts, account identity, and usage credentials. |
| `KIMI_SHARE_DIR` | Override the legacy kimi-cli config/share directory when `KIMI_CODE_HOME` is unset; Chat View and usage credentials read from this root too. |
| `GROK_HOME` | Override Grok Build config dir for hook install/status/uninstall, Chat View sessions, and SuperGrok usage credentials (`auth.json`). |
| `HERMES_HOME` | Override Hermes Agent's config, plugin, and state directory. |
| `XDG_STATE_HOME` / `XDG_CONFIG_HOME` / `XDG_RUNTIME_DIR` | Standard XDG dirs (Linux). |

Claude profiles seen in hook events are discovered automatically, including
their exact transcript paths, usage, and account identities. Discovery persists
across daemon restarts.

Kimi Code usage reads the official CLI credential at
`$KIMI_CODE_HOME/credentials/kimi-code.json` (default
`~/.kimi-code/credentials/kimi-code.json`) and fetches the managed account's
5-hour and weekly quotas from `https://api.kimi.com/coding/v1/usages`. The
credential remains owned by Kimi Code: `moshi-hook` uses only a fresh access
token and never uses, refreshes, or rewrites the rotating refresh token.
Legacy `~/.kimi/credentials/kimi-code.json` is supported as a read-only
fallback.

Grok Build SuperGrok / X Premium usage reads `$GROK_HOME/auth.json` (default
`~/.grok/auth.json`) and fetches the weekly credit pool from
`https://cli-chat-proxy.grok.com/v1/billing?format=credits`. The credential
remains owned by Grok: `moshi-hook` uses only a fresh OIDC access token and
never uses, refreshes, or rewrites the refresh token.

## Subcommands

| Command | What it does |
|---|---|
| `pair --token <t> [--name <n>] [--store keychain\|file]` | Pair the agent-hooks daemon. Token comes from the Moshi mobile app. Re-running rotates the secret and can repair this host after a license/subscription change. macOS defaults to `keychain` until a store preference is saved; use `file` for headless sessions. |
| `unpair` | Remove the agent-hooks secret and server registration. |
| `host setup` | Start an Easy Pair SSH/Mosh setup session and print a QR for Moshi to scan. |
| `host list` | List local Moshi host SSH pairings. |
| `host revoke <id>` | Remove a Moshi host SSH key from `authorized_keys`. |
| `host enable-ssh` | Help enable SSH prerequisites on macOS. |
| `diff [path] [--no-open] [--port N]` | Serve the embedded Git diff viewer for a local project directory. |
| `install` | Write Moshi entries into supported agent config files. By default, only installs targets whose config root already exists and reports missing agents as skipped. Use `--target claude,codex,opencode,gemini,antigravity,cursor,kimi,qwen,grok,omp,pi,hermes` to force or limit the set. Non-destructive: leaves user-owned hooks alone. OpenCode installs globally by default; use `--local` for `.opencode/plugins` in the current project. When Codex 0.157+ is installed with its shared background server on, `install` asks whether to turn it off (non-interactive runs print a pointer to `doctor`). |
| `uninstall` | Remove Moshi-owned entries from those files. For OpenCode, pass `--local` to remove a project-local install. |
| `service install` | Linux: install and start a systemd user service. Windows groundwork: register current-user logon startup under `HKCU\...\Run` and start a detached daemon without elevation. |
| `service uninstall` | Disable/remove the Linux systemd service or Windows logon value and stop the daemon. |
| `service status` | Show systemd status on Linux or the Windows logon registration. |
| `serve [--gateway-listen 127.0.0.1:24543]` | Run the daemon and localhost API/diff gateway in the foreground. Does not serve the general web UI. Single-instance via an OS file lock under the state directory. |
| `status [--json]` | Pairing state, paths, and best-effort server attachment status for the paired host. `--json` also carries hook install state and stays local (no server round-trip). For hooks, multiplexers, and Chat View readiness, run `doctor`. |
| `doctor [--yes]` | Check which Moshi features work on this host and what each one is missing: agent inbox and alerts, Chat View, Workspaces and Jump To, the session picker, the diff viewer, Browser/Simulator preview, and usage. It checks the daemon and its gateway (including a version mismatch with the installed CLI), pairing (confirmed with Moshi), agent hooks, settings that turn a feature off, and tmux/herdr — including duplicate installs where the copy the app's session picker or the daemon would run differs from the one running your server, tmux older than 2.6 (Moshi needs `select-pane -T` and `send-keys -X`), the EL10 tmux 3.3a capture bug, and herdr older than 0.9 (slower fallback). Prints numbered fixes and a checklist of things only you can confirm (phone notifications, Live Activity, connecting from the app, starting agents in tmux/herdr, restarting agents after hook changes). When Codex 0.157+ runs its shared background server, `doctor` offers to turn it off (`--yes` applies without asking); see [Codex background server](#codex-background-server). |
| `update [--version vX.Y.Z]` | Update a Linux or Windows manual install from `cdn.getmoshi.app`. Verifies the release checksum before replacing the current binary. Windows uses ZIP assets and can rotate the currently running `.exe`. Homebrew installs are left untouched; use `brew upgrade moshi-hook`. |
| `usage [--sync]` | Cached Codex, Claude, OpenCode, Kimi, Grok, and Antigravity snapshots; refreshes missing Claude profiles and missing/stale Codex, Kimi, Grok, and Antigravity API caches first (including OpenCode-only Codex with no rollouts). `--sync` pushes them to the server and reports whether this host is attached to Moshi Pro. On-demand: works even when background collection is off (`set usage-collection off`). JSON output also carries a local `cost` rollup per account — tokens burned today and over the last 7 days, split by model, with an estimated cost at public API list rates (not what a plan charged). Scanning is incremental and capped per refresh, so a large history fills in over several background passes; unpriced models report tokens with `pricingComplete: false` and no dollar figure. `--sync` does not upload it. |
| `set [setting] [value]` | Show or change settings in `~/.config/moshi/config.toml`. `usage-collection` accepts `on`, `off`, or a polling interval such as `5m`; `on` restores the previously chosen interval (default `1m`). `scan-ports 3000,5173` restricts Browser Preview HTTP probes to those ports and accepts inclusive ranges (`scan-ports 3000,8000-8010`), which is also how you keep other local services out; `scan-ports all` restores the default scan-all behavior and `scan-ports none` disables HTTP probing. Scan-port changes apply on the next discovery refresh; restart the daemon after changing background settings: `always-on-discovery`, `usage-collection`, `suppress-nested-agent-push`, or `suppress-push-while-unlocked`. `install.sh` runs `moshi-hook set --first-run` at the end (both onboarding options checked/on; opens `/dev/tty` under `curl\|sh`). Skip with `MOSHI_HOOK_SKIP_FIRST_RUN=1`. Homebrew does not run that step — defaults stay on until `host setup` / `serve` / `pair` / `install` (interactive) or an explicit `set --first-run`. Machine-readable commands (`status --json`, etc.) never auto-prompt. |
| `cwd-list [--json] [--limit N]` | Recent project working directories from local agent state (Claude, Codex, Cursor). Plain-text table by default; `--json` emits the shape the iOS preflight consumes. |
| `servers [--ssh-connection \"<value>\"] [--mosh-port <p> [--mosh-host <ip>]] [--et-client-id <id>\|--et]` | Probe local TCP listeners and print HTTP web servers for SSH preflight (filtered to `text/html` responses, tagged with owning process + PID, one-entry-per-PID). With a session lookup, decorates each row with `isCurrentContext`. |
| `servers kill --pid <pid> --port <port> [--host <host>] [--force=false]` | Terminate a discovered local HTTP server after re-validating that the PID and port still match the server list. |
| `context [--ssh-connection \"<value>\"] [--mosh-port <p> [--mosh-host <ip>]] [--et-client-id <id>\|--et]` | Print terminal context (kind=tmux or shell, cwd, git) for the caller or a remote SSH/Mosh/ET session. With no flags, auto-detects from `$TMUX_PANE` or falls back to the caller's cwd. With a remote-session identifier, looks up the iOS-owned session's login shell and reports whether the user is currently in tmux. Used by Moshi clients over SSH preflight. |
| `logs [-f]` | Tail the daemon log. |
| `version` | Version, commit SHA, build date. |

Hidden subcommands (`claude-hook`, `codex-hook`, `opencode-event`, `opencode-permission`, `gemini-hook`, `antigravity-hook`, `cursor-hook`, `kimi-hook`, `qwen-hook`, `grok-hook`, `omp-hook`, `pi-hook`, `hermes-hook`) are invoked by the agents themselves through the configs `install` writes — you won't run them by hand.

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

### `context` — live terminal state for an SSH/Mosh/ET session

The iOS app uses this to ask "is the user currently in tmux/zellij on this session, and where is their cwd?" Tmux detection is live: when the user attaches/detaches tmux, the next call reflects it. Zellij detection uses the shell environment (`ZELLIJ`, `ZELLIJ_SESSION_NAME`, `ZELLIJ_PANE_ID`) when those variables are visible to the caller.

Identifiers (exactly one):

| Flag | Where iOS gets it |
|---|---|
| `--ssh-connection "<client_ip> <client_port> <server_ip> <server_port>"` | Captured once via ssh-exec right after the SSH session opens (`echo $SSH_CONNECTION`). |
| `--mosh-port <port>` | Already known from the `MOSH CONNECT <port> <key>` handshake. |
| `--mosh-host <ip>` | Optional disambiguation hint for the rare case where two mosh-servers share a port number on different interfaces (e.g. Tailscale + LAN at once). If the hint does not match but the port has only one local binding, the daemon uses that binding. Without a matching hint, ambiguous lookups error rather than guessing. |
| `--et-client-id <id>` | Eternal Terminal's 16-character client id from the ET handshake. |
| `--et` | Manual ET fallback. Only works when exactly one `etterminal` process is visible; otherwise pass `--et-client-id`. |

Resolution chain: identifier → login shell PID → controlling TTY → `tmux list-clients` match → zellij env probe. SSH uses `$SSH_CONNECTION`, Mosh uses the `mosh-server` UDP listener, and ET uses the per-session `etterminal` child shell. If tmux matches, returns the session's active pane via `tmux display-message`; if zellij env is present, returns `kind: "zellij"` with the zellij session/pane values; otherwise returns `kind: "shell"` with cwd from `/proc/<pid>/cwd` (Linux) or `lsof -d cwd` (macOS). The `tmux` or `zellij` block is omitted when not applicable.

```bash
# CLI auto-detect inside the caller's own shell
moshi-hook context

# iOS query for a remote SSH session
moshi-hook context --ssh-connection "192.168.68.55 60688 192.168.68.54 22"

# iOS query for a remote Mosh session with host disambiguator
moshi-hook context --mosh-port 60001 --mosh-host 192.168.68.54

# iOS query for a remote Eternal Terminal session
moshi-hook context --et-client-id abcdefghijklmnop
```

## Installed agent files

| Agent | Managed file |
|---|---|
| Claude Code | `$CLAUDE_CONFIG_DIR/settings.json` or `~/.claude/settings.json` |
| Codex | `$CODEX_HOME/hooks.json` plus `$CODEX_HOME/config.toml` feature flag, or `~/.codex/...` |
| OpenCode | `$OPENCODE_CONFIG_DIR/plugins/moshi-hooks.ts`, `$XDG_CONFIG_HOME/opencode/plugins/moshi-hooks.ts`, or `~/.config/opencode/plugins/moshi-hooks.ts`; `.opencode/plugins/moshi-hooks.ts` with `--local` |
| Gemini CLI | `~/.gemini/settings.json` |
| Antigravity | `$ANTIGRAVITY_CONFIG_DIR/hooks.json` or `~/.gemini/config/hooks.json` |
| Cursor | `$CURSOR_CONFIG_DIR/hooks.json` or `~/.cursor/hooks.json` |
| Kimi | `$KIMI_CODE_HOME/config.toml` or `~/.kimi-code/config.toml` (`KIMI_SHARE_DIR` remains supported for legacy kimi-cli) |
| Qwen Code | `~/.qwen/settings.json` |
| Grok Build | `$GROK_HOME/hooks/moshi-hooks.json` or `~/.grok/hooks/moshi-hooks.json` |
| OMP (Oh My Pi) | `$OMP_CODING_AGENT_DIR/extensions/moshi-hooks.ts`, `$OMP_PROCESSING_AGENT_DIR/extensions/moshi-hooks.ts`, `$PI_CODING_AGENT_DIR/extensions/moshi-hooks.ts`, `$PI_CONFIG_DIR/agent/extensions/moshi-hooks.ts`, `~/.omp/profiles/$OMP_PROFILE/agent/extensions/moshi-hooks.ts`, or `~/.omp/agent/extensions/moshi-hooks.ts` |
| Pi | `$PI_CODING_AGENT_DIR/extensions/moshi-hooks.ts`, `$PI_CONFIG_DIR/agent/extensions/moshi-hooks.ts`, or `~/.pi/agent/extensions/moshi-hooks.ts` |
| Hermes Agent | `$HERMES_HOME/plugins/moshi-hooks/{plugin.yaml,__init__.py}` or `~/.hermes/plugins/moshi-hooks/...`; installer also enables `moshi-hooks` in the matching `config.yaml` |

Default `install` skips a managed file when the agent's config root is missing, for example `~/.cursor` or `~/.gemini`. Passing `--target` preserves the old create-if-missing behavior for that target.

### Codex background server

Codex 0.157 and later start interactive sessions inside one shared background server (`codex app-server --managed-daemon`, controlled by `features.daemon_auto_start`, on by default). That server runs the hooks and holds every session's transcript, and it keeps the environment of whichever terminal started it. Moshi then attributes every Codex session to that first terminal, so Chat View, replies, and approvals follow the wrong pane.

`moshi-hook install` and `moshi-hook doctor` detect this and, after asking, set `daemon_auto_start = false` under `[features]` in `$CODEX_HOME/config.toml` and stop the running server along with its updater. Codex sessions that were attached to it disconnect; restart them. Turning the feature off alone is not enough while a server is still running, because new Codex sessions keep attaching to it. `moshi-hook uninstall` leaves the setting in place.

To do the same by hand:

```sh
codex features disable daemon_auto_start
codex app-server daemon stop
pkill -f 'app-server daemon pid-update-loop'   # `daemon stop` leaves the updater running
```

### Repairing the OpenCode integration

If Integrations says the OpenCode plugin is missing or outdated (older daemons
say "plugin differs from current installer output"), run
`moshi-hook install --target opencode` **on that host**, then restart OpenCode.
For a project-local installation, run the command in that project with `--local`.
The check compares the entire generated file, including the helper binary path.
A Moshi upgrade, manual edits, or installing/checking with different binary paths
can therefore trigger it. Reinstallation replaces the generated plugin; keep
custom plugins in separate files. The existing Integrations install action also
rewrites the host's global plugin.

The generated plugin has separate V1 `server` and V2 `setup` entrypoints, so
OpenCode 2.x does not need a manually edited plugin. With OpenCode 2.x, launch
`opencode --standalone` for pane-bound Chat View and Stop: the default shared
background service does not carry the current terminal's identity. The V2
transcript relay reads OpenCode's public `session.context` API; history removed
from that context by compaction is not available through this API.

Hermes Agent keeps its conversation history in `$HERMES_HOME/state.db` rather than in per-session transcript files. Chat View reads that database through Moshi's bundled read-only SQLite driver; no separate `sqlite3` command is required.

Kimi's managed install targets the current Kimi Code lifecycle, including native `PermissionRequest` / `PermissionResult`, interruption, failure, and session-end callbacks. Approval hooks are observation-only: Kimi's terminal prompt remains authoritative while Moshi mirrors and can drive that verified prompt.

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

| What | macOS | Linux | Windows groundwork |
|---|---|---|---|
| State + log | `~/Library/Application Support/Moshi/` | `$XDG_STATE_HOME/moshi/` | `%LOCALAPPDATA%\Moshi\` |
| Local IPC | `<state>/moshi-hook.sock` | `$XDG_RUNTIME_DIR/moshi-hook.sock` | `\\.\pipe\moshi-hook` |
| Secrets | Keychain (`app.getmoshi.hook`) by default; `~/.config/moshi/secrets.json` with `--store file` | `<state>/secrets.json` (0600) | `<state>\secrets.json` |

When run via `brew services`, logs land at `$(brew --prefix)/var/log/moshi-hook.log`.

Native Windows is experimental: unsigned preview binaries, a narrower agent matrix, and no graceful
process stop. WSL2 remains the stable layout — run `moshi-hook` and your agents inside the same
distribution. See [windows.md](windows.md) for both layouts and their constraints.
