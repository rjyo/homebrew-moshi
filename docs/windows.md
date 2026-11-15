# Windows and WSL2

> **Native Windows support is experimental.** The builds are unsigned preview binaries. They work on the
> agents listed below, but the support boundary is narrower than macOS and Linux and may change between
> releases.

There are two ways to run `moshi-hook` on Windows:

- **Native (experimental)** — the PowerShell installer below, with [Herdr](https://herdr.dev) as the
  terminal multiplexer. See [Native Windows](#native-windows).
- **WSL2 (recommended for stability)** — install it inside the distribution and run your coding agent in
  the same distribution. The WSL2 layout has not yet been verified end to end — see
  [Verification status](#verification-status).

That last clause is the whole story, so it comes first:

> **Moshi hooks the agents that live where `moshi-hook install` runs.** A `moshi-hook` installed in
> WSL2 hooks agents running in WSL2. It does **not** hook Claude Code, Codex, or Cursor installed
> natively on Windows.

## Why the boundary exists

`moshi-hook install` writes each agent's hook config next to that agent's own settings, resolved from
the current user's home directory:

| Agent | Config written by `moshi-hook install` |
|---|---|
| Claude Code | `$CLAUDE_CONFIG_DIR/settings.json`, else `~/.claude/settings.json` |
| Codex | `$CODEX_HOME/config.toml`, else `~/.codex/config.toml` |
| Gemini CLI | `~/.gemini/settings.json` |
| others | same pattern under `~` |

Inside WSL2, `~` is `/home/<user>`. A natively-installed Windows agent reads
`C:\Users\<user>\.claude\settings.json`. Those are different files on different filesystems, so a WSL2
install leaves a native Windows agent completely unhooked — it will not error, it simply will not be
connected.

The helper binary is also an ELF executable. Even if a native Windows agent were pointed at the WSL2
config, it could not execute `moshi-hook` directly as its hook command.

## Recommended layout

Put everything on one side of the boundary:

```
Windows
└── WSL2 (Ubuntu, Debian, …)
    ├── moshi-hook        ← installed here
    ├── Claude Code / Codex / …   ← running here
    └── tmux / Herdr      ← multiplexer here
```

Install exactly as on Linux:

```bash
moshi-hook pair --token <pairing-token>
moshi-hook install
moshi-hook service install     # systemd user service; WSL2 needs systemd enabled
moshi-hook serve               # foreground fallback if systemd is unavailable
```

`moshi-hook install` prints which agents it found. If an agent you use is missing from that list,
it is almost certainly installed on the Windows side rather than in WSL2.

### systemd in WSL2

`moshi-hook service install` writes a systemd **user** unit. WSL2 only runs systemd when the
distribution is configured for it:

```ini
# /etc/wsl.conf
[boot]
systemd=true
```

Then `wsl --shutdown` from Windows and reopen the distribution.

Without systemd, run `moshi-hook serve` in a dedicated long-lived pane (a tmux window or a Herdr pane)
and leave it there. Do not start it from your shell profile: `serve` runs in the foreground, so it
either blocks every new shell or — if backgrounded — leaves a fresh daemon behind on each shell start,
all contending for the same socket.

## Known constraints

**Working directories.** Keep repositories on the Linux filesystem (`/home/...`), not under `/mnt/c`.
Moshi resolves and reports absolute working directories, and a `/mnt/c/...` path is what will be shown
and sent — correct, but not the `C:\...` path Windows tooling expects. `/mnt/c` also has substantially
slower file I/O and different file-watching behavior, which affects any feature that watches a
transcript or a repository.

**Clock skew.** WSL2 clocks can drift after the Windows host sleeps. A skewed clock breaks TLS to the
Moshi API and makes transcript timestamps inconsistent. `sudo hwclock -s` resyncs it.

**Networking.** The daemon's local socket lives in `$XDG_RUNTIME_DIR` or `/tmp` inside WSL2 and is not
reachable from Windows processes. The WebSocket to Moshi is outbound and works normally.

**One distribution.** Each WSL2 distribution has its own filesystem, home directory, and daemon. Hooks
installed in one distribution do not apply to another.

## Multiplexers

Run your multiplexer inside the WSL2 distribution, alongside the agents.

Inside WSL2, tmux is a Linux process like any other, so `moshi <dir>` should behave as documented in
[usage.md](usage.md).

Herdr is the more interesting case, because Herdr ships native Windows builds. Two arrangements are
**not** equivalent:

- **Herdr inside WSL2** — the intended layout. `moshi-hook` and Herdr are processes in the same
  namespace, so pane discovery, capture, and send-keys have everything they need.
- **Herdr on Windows, `moshi-hook` in WSL2** — *not* supported. Moshi shells out to a `herdr` binary
  that must be visible from the daemon's own filesystem, and detects the current pane from
  `HERDR_ENV`/`HERDR_PANE_ID` in the hook process's environment. Some paths (SSH-attach detection,
  server-origin attribution) additionally read another process's environment via `/proc` or `ps`.
  None of that crosses the WSL2 boundary.

The Windows-Herdr/WSL-daemon boundary is a structural consequence of how panes are discovered. The
in-WSL2 arrangement is expected to work but has not been exercised on a Windows host — see below.

## Native Windows

**Experimental.** Native Windows is a preview: unsigned binaries, a narrower agent matrix than macOS and
Linux (see below), and no MSI/winget/Scoop packaging. Prefer WSL2 if you need the fully supported layout.

Preview builds use the same checksummed, per-user installer model as Herdr's
Windows beta. From PowerShell:

```powershell
powershell -ExecutionPolicy Bypass -c "irm https://getmoshi.app/install.ps1 | iex"
```

The installer verifies the published GoReleaser ZIP against `checksums.txt`,
installs it under `%USERPROFILE%\.moshi\packages\standalone\releases`, hardlinks
(or copies) the active version's executables into the real directory
`%LOCALAPPDATA%\Programs\Moshi Hook\bin`, adds that stable directory to the user
`PATH`, and retains the three newest releases. bin is not a junction: sshd logons
refuse to traverse a junction a non-admin user created, which hid the daemon from
the Moshi app over SSH. Older installs have that junction; rerunning the installer
replaces it. It does not require elevation. Open a new PowerShell window,
then run `moshi-hook install` and `moshi-hook service install`.

Native Windows remains experimental, preview-only, and unsigned, so SmartScreen may require
**More info → Run anyway**.

- The tree **compiles** for `windows/amd64` and `windows/arm64`.
- File locking and process signalling have Windows implementations (`LockFileEx`,
  `OpenProcess`/`TerminateProcess`). They are covered by native Windows tests, including lock
  contention and the lifecycle of a real child process.
- State, logs, and the daemon singleton lock resolve under `%LOCALAPPDATA%\Moshi`.
- Local IPC uses the current-user-only `\\.\pipe\moshi-hook` named pipe. The endpoint is independent
  from the regular lock file, so no filesystem operation is attempted on the pipe address.
- Native Go clients plus Node's and Bun's `node:net` reach the same live daemon over that pipe.
- SQLite-backed agent transcripts use the bundled read-only driver and do not require a separately
  installed `sqlite3.exe`.
- Secrets written by the Windows file store are encrypted for the current user with DPAPI. Existing
  plaintext values remain readable and are encrypted the next time they are written.
- The Herdr CLI resolves from the installer's own layout before `PATH`: `%HERDR_INSTALL_DIR%`, then
  `%LOCALAPPDATA%\Programs\Herdr\bin`, then
  `%HERDR_HOME%` (default `%USERPROFILE%\.herdr`) `\packages\standalone\current`.
  Windows has one supported install path (`herdr.dev/install.ps1`, or its zip
  extracted by hand), which junctions both of those directories onto one versioned release, so the
  layout is knowable rather than guessed. `PATH` comes second because the Windows build ships an
  app-local ConPTY runtime beside `herdr.exe` and breaks when only the executable is copied
  elsewhere. `MOSHI_HERDR_PATH` still overrides everything.

What remains unresolved before native Windows can drop the experimental label:

- **No graceful process stop.** Windows has no equivalent of `SIGTERM` for an arbitrary discovered
  process, so stopping a preview server without `--force` reports that it is unsupported rather than
  hard-killing it.
- Self-update downloads the checksummed Windows ZIP, moves the running executable aside, and installs
  the replacement in place. `service install` uses a non-elevated
  current-user logon registration and starts a detached daemon; `service uninstall` removes it and
  stops the process.

### Native agent support matrix

| Agent/integration | Native Windows status | Evidence or remaining boundary |
|---|---|---|
| Codex CLI | Lifecycle validated | Real Herdr session, `/new` live-empty handoff, durable `turn_aborted`, local retirement while unpaired, and delayed non-resurrection passed. |
| Claude Code | Lifecycle validated | Real `/clear` replacement, transcript handoff, durable interruption row, watcher retirement, and delayed non-resurrection passed. |
| OpenCode | Lifecycle validated | Named-pipe transport, real `/new` same-pane ownership transfer, transcript replacement, and native two-Escape cancellation passed. |
| Grok Build | Lifecycle validated | Real `/new` live-empty replacement and durable `turn_completed/cancelled` passed; the transcript watcher retires cancellation while unpaired. |
| OMP | Lifecycle validated | Real `/new` live-empty replacement and durable `stopReason: aborted` passed; the transcript watcher covers OMP's missing `session_stop` callback. |
| Pi | Lifecycle validated | Real `/new` live-empty replacement and durable aborted tool/assistant rows passed. Scoop Git requires `shellPath` to point at its Git Bash executable. |
| Hermes | Unsupported natively | Turns and Ctrl+C cancellation work after installing the plugin under `%LOCALAPPDATA%\hermes`, but Hermes creates the replacement session id only on the first prompt after `/new`; it cannot satisfy live-empty replacement. |
| Cursor Agent | Unsupported on tested build/account | Cursor v2026.08.11 did not invoke either global or project hooks, including a one-line debug hook, although Moshi's command succeeds directly. |
| Antigravity CLI (`agy`) | Unsupported | The CLI is distinct from the supported Antigravity IDE integration and does not consume the IDE hook configuration. |
| Gemini CLI | Unsupported on tested account | Individual Gemini Code Assist login is disabled upstream in favor of Antigravity. |
| Kimi, Qwen | Unverified | Installed but intentionally skipped during this validation run; no native support claim. |
| Herdr terminal control | Validated groundwork | Native ConPTY delivered Escape, Ctrl+V, and Ctrl+C; image clipboard seed/detect/restore passed. |
| tmux and zellij | Unsupported natively | Use Herdr, or run the entire stack inside WSL2. |

### Distribution decision

The initial native delivery is the checksumming PowerShell bootstrap above,
backed by a GoReleaser ZIP and the built-in `moshi-hook update` path. There is
deliberately no MSI, winget, or Scoop claim yet. The installer keeps everything
in user-owned directories and `service install` provides logon startup, so the
entire flow remains non-elevated.

Artifacts remain unsigned until code-signing procurement is complete. Windows may therefore show a
SmartScreen warning; users should verify the published checksum before running the executable. A
signed production release is an external release prerequisite, not something the source tree or this
test machine can manufacture.

The platform wire contract remains `other` for this branch. The intended long-term value is the
explicit string `windows`, but emitting it from the hook alone would break the current server and app
unions (`macos | linux | other`). It must land atomically across moshi-hook, the API, and clients; until
then the backward-compatible value is intentional rather than an undetected platform.

Tracking issue: [rjyo/moshi#494](https://github.com/rjyo/moshi/issues/494).

## Verification status

Native groundwork was exercised on Windows x64 (build 26200) with Go 1.26.7:

- the native binary builds, starts its daemon, and exposes state beneath `%LOCALAPPDATA%\Moshi`,
- a separate native CLI reaches the daemon through its named pipe (`probe`, `status`, and
  `context`),
- `LockFileEx` rejects a second holder and releases the lock for a later holder; a second daemon
  process also observes the singleton lock,
- `OpenProcess` sees a live child and `TerminateProcess` ends it, while the SIGTERM path leaves it
  running and reports graceful stop as unsupported,
- the native tmux launcher refuses up front and points users to Herdr or WSL2,
- Herdr's ConPTY input path delivers `Escape`, `ctrl+v`, and `ctrl+c` with the expected key and control
  characters; Windows image clipboard seeding round-trips through `System.Windows.Forms.Clipboard`,
- process-tree discovery uses the native Toolhelp snapshot API instead of invoking a POSIX `ps`;
  Windows does not expose other processes' environment through that API, so SSH-origin and automatic
  server-to-pane attribution that depend on inherited environment variables remain unavailable,
- a sandboxed Codex install writes a `cmd.exe`-compatible quoted hook command and recognizes its own
  fresh hook configuration as complete,
- real native Codex, Claude, OpenCode, Grok, OMP, and Pi sessions completed the session-transition
  matrix; `/new` transfers ownership from the old session
  to the new id in the same process, and the new transcript renders without falling back to the old
  session,
- interrupting live long-running turns records each agent's durable cancellation outcome, retires
  Moshi's exact pending prompt even while unpaired, and remains stopped after the watcher/idle delay,
- Node 24 connects to the named pipe and receives a valid daemon protocol response.

The complete test suite is not Windows-clean yet. Native execution exposes existing POSIX assumptions
in tests and features outside this groundwork, including path fixtures, executable discovery,
service-unit generation, and terminal fixtures.

Stated with confidence, because each follows directly from how `moshi-hook` resolves paths and
discovers processes:

- the home-directory boundary between WSL2 and native Windows agents,
- the ELF-helper constraint,
- the Herdr process-discovery constraint,
- where the daemon's socket and state live.

Still **unverified** — treat as a starting point, not a guarantee:

- that an end-to-end WSL2 install works at all,
- systemd availability and `service install` behavior,
- `/mnt/c` behavior and clock skew, which vary by Windows build and distribution,
- tmux and Herdr operation inside WSL2.

If you hit something this page does not describe, please report it on
[rjyo/moshi#494](https://github.com/rjyo/moshi/issues/494).
