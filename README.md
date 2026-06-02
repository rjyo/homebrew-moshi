# homebrew-moshi

Homebrew tap for [Moshi](https://getmoshi.app).

This installs the `moshi` CLI and the `moshi-hook` service package for macOS and Linuxbrew.

## Install

```bash
brew tap rjyo/moshi
brew install moshi-hook
brew services start moshi-hook
```

Release tarballs are served from `cdn.getmoshi.app` — no GitHub token needed for the binary download.

## Setup

Pair this host from the Moshi app, then install supported agent hooks:

```bash
moshi host setup
moshi install
```

`brew services` keeps the daemon alive across reboots and restarts it on crash. Logs go to `$(brew --prefix)/var/log/moshi-hook.log`.

To run it in the foreground instead (useful while debugging):

```bash
brew services stop moshi-hook
moshi serve -v
```

## Common commands

```bash
moshi .              # open or attach to a tmux session for this directory
moshi diff .         # open the local diff viewer
moshi status
moshi logs -f
moshi version
```

## Upgrade

```bash
brew update
brew upgrade moshi-hook
brew services restart moshi-hook
```

Homebrew owns the installed binary, so use `brew upgrade moshi-hook` instead of `moshi update`.

## Uninstall

```bash
brew services stop moshi-hook
moshi uninstall
moshi unpair
brew uninstall moshi-hook
brew untap rjyo/moshi
```

## Reporting issues

Email <support@getmoshi.app> or DM the team — this tap repo doesn't track issues.

`Formula/moshi-hook.rb` is auto-published on every release; manual edits will be overwritten.
