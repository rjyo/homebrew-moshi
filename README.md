# homebrew-moshi

Homebrew tap for [Moshi](https://getmoshi.app) — currently shipping `moshi-hook`, the portable agent-hook daemon for Claude Code, Codex CLI, and OpenCode.

## Install

```bash
brew tap rjyo/moshi
brew install moshi-hook
```

Release tarballs are served from `cdn.getmoshi.app` — no GitHub token needed for the binary download.

## Run as a service

`moshi-hook` is a daemon — pair it once, then keep it running in the background:

```bash
moshi-hook pair --token $MOSHI_PAIRING_TOKEN   # one-time, get token from the Moshi app
moshi-hook install                             # writes hook configs for installed agents
brew services start moshi-hook                 # launchd (macOS) / systemd-user (Linux)
```

`brew services` keeps the daemon alive across reboots and restarts it on crash. Logs go to `$(brew --prefix)/var/log/moshi-hook.log`.

To run it in the foreground instead (useful while debugging):

```bash
brew services stop moshi-hook
moshi-hook serve
```

## Verify

```bash
moshi-hook status         # pairing state, socket path, connected hosts
moshi-hook logs -f        # tail the log file
moshi-hook version        # version + commit
```

## Upgrade

```bash
brew update
brew upgrade moshi-hook
brew services restart moshi-hook
```

The hook configs written into `~/.claude/settings.json` and `~/.codex/config.toml` reference `/opt/homebrew/bin/moshi-hook` (or `/usr/local/bin/moshi-hook` on Intel Macs / Linuxbrew), so upgrades don't require re-running `moshi-hook install`.

## Uninstall

```bash
brew services stop moshi-hook
moshi-hook uninstall      # removes the hook entries from agent configs
moshi-hook unpair         # forgets the host secret
brew uninstall moshi-hook
brew untap rjyo/moshi
```

## Documentation

- [Usage](docs/usage.md) — every subcommand, every flag
- [API](docs/api.md) — local socket protocol, Moshi HTTP, WebSocket bridge

These docs are mirrored from the source repo on every release; the source is the single source of truth.

## Reporting issues

Email <support@getmoshi.app> or DM the team — this tap repo doesn't track issues.

`Formula/moshi-hook.rb` and `docs/` are auto-published by [GoReleaser](https://goreleaser.com) on every tagged release; manual edits will be overwritten.
