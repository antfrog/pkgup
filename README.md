# pkgup

A single zsh CLI for macOS that periodically updates **Homebrew (formulae/casks)** and
**npm global packages**, updates **masked packages only after confirmation**, and reports
the results to **Slack**.

## Features

- Updates formulae, casks, and npm globals in one pass
- Masking (hold) system: packages you want pinned are excluded from automatic updates and only applied after confirmation
- Automatic TTY detection switches between interactive and non-interactive (scheduled) modes
- Reports update results via a Slack incoming webhook
- No extra runtime dependencies (only brew / npm / node / curl / coreutils)

## How it works

| Context | Mode | Unmasked packages | Masked packages |
|---|---|---|---|
| `pkgup update` in a terminal | `interactive` (auto-detected) | auto-updated | **updated after y/N confirmation** |
| launchd / cron schedule | `auto` (no TTY) | auto-updated | left untouched, reported as `pending` |

cron/launchd runs are non-interactive, so no confirmation can happen there. Scheduled runs
therefore mark masked package updates as **pending**, notify you via Slack, and you later run
`pkgup update` in a terminal to review and apply them.

## Requirements

- macOS, zsh (the default shell since Catalina)
- Homebrew (for managing formulae/casks)
- Node.js + npm (for managing npm globals — node is used to parse npm outdated)
- The npm global prefix must be **user-writable** to work without sudo
  (Homebrew node / nvm / `npm config set prefix ~/.npm-global` recommended)

## Install

```sh
mkdir -p ~/.local/bin ~/.config/pkgup
install -m 755 pkgup ~/.local/bin/pkgup
cp config.example ~/.config/pkgup/config
$EDITOR ~/.config/pkgup/config        # set SLACK_WEBHOOK_URL etc.

# Add ~/.local/bin to PATH (.zshrc)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && exec zsh
```

## Usage

```sh
pkgup check                       # list outdated/masked only (no changes, dry-run)
pkgup update                      # update (in a terminal, masked packages are confirmed)
pkgup update --yes                # force auto mode (masked packages held without confirmation)
pkgup update --interactive        # force interactive mode

pkgup mask add brew kubectl       # mask a formula
pkgup mask add cask docker        # mask a cask
pkgup mask add npm  pnpm          # mask an npm global
pkgup mask rm  brew kubectl       # unmask
pkgup mask list                   # list masks

pkgup schedule install            # register a weekly launchd agent
pkgup schedule uninstall          # unload the agent + delete the plist
pkgup schedule status             # show registration/load status

pkgup help                        # help
```

## Masking (hold)

Masking means "no automatic updates; confirm before upgrading". The manager kinds are
`brew` · `cask` · `npm`. Masking tools like `kubectl` · `helm` · `terraform`, which must stay
version-compatible with a cluster, keeps them from being bumped arbitrarily by the automatic
loop — they are only updated when you confirm directly in a terminal.

The mask list is stored in `~/.config/pkgup/masks`, one `manager:name` per line.

## Configuration

`~/.config/pkgup/config` (sourced by zsh):

| Key | Default | Description |
|---|---|---|
| `SLACK_WEBHOOK_URL` | (none) | Slack incoming webhook. If unset, sending is skipped (report goes to log/stdout only) |
| `NPM_UPDATE_TARGET` | `latest` | `latest` = newest (including major) / `wanted` = within semver range |
| `BREW_CLEANUP` | `false` | Run `brew cleanup` after upgrading |
| `BREW_CASK_GREEDY` | `false` | Also check casks that auto-update (`brew outdated --greedy`) |
| `EXTRA_PATH` | (none) | Extra PATH entries for scheduled runs (e.g. nvm node path) |
| `SCHEDULE_WEEKDAY` | `0` | Weekday for `schedule install` runs (0/7 = Sunday). Empty (`SCHEDULE_WEEKDAY=`) = daily; a space-separated list (`"1 2 3 4 5"`) = those weekdays only |
| `SCHEDULE_HOUR` | `10` | Hour for `schedule install` runs |
| `SCHEDULE_MINUTE` | `0` | Minute for `schedule install` runs |
| `PKGUP_LABEL` | `com.user.pkgup` | LaunchAgent label |

The `PKGUP_HOME` environment variable changes the config/state directory
(default `~/.config/pkgup`).

## Weekly automatic runs

### pkgup schedule (recommended)

The installed `pkgup` registers/unregisters the launchd agent itself. No need to write a
plist by hand or call `launchctl`.

```sh
pkgup schedule install      # generate ~/Library/LaunchAgents/<label>.plist and register it
pkgup schedule status       # registration/load status
pkgup schedule uninstall    # unload + delete plist

# To trigger one run right after installing (also shown in the install output)
launchctl kickstart -k gui/$(id -u)/com.user.pkgup
```

The run weekday/time comes from `SCHEDULE_WEEKDAY` / `SCHEDULE_HOUR` / `SCHEDULE_MINUTE`
in config (default Sunday 10:00). `SCHEDULE_WEEKDAY` controls which days:

- `SCHEDULE_WEEKDAY=0` — weekly (Sunday); `1`–`6` for Mon–Sat
- `SCHEDULE_WEEKDAY=` (empty) — **daily**, every day at `HOUR:MINUTE`
- `SCHEDULE_WEEKDAY="1 2 3 4 5"` — only those weekdays (here, Mon–Fri)

After changing them, re-run `pkgup schedule install` to apply.

> The agent's PATH includes `~/.npm-global/bin`. If you use another node path (nvm etc.),
> add it via `EXTRA_PATH` in config.

### cron (alternative)

To use cron instead:

```sh
crontab -e
# Every Sunday at 10:00
0 10 * * 0 /Users/<you>/.local/bin/pkgup update --yes >> ~/Library/Logs/pkgup.cron.log 2>&1
```

On macOS, cron may require Full Disk Access, and runs missed while asleep are not made up.
launchd (= `pkgup schedule`) runs once right after waking if the machine was asleep at the
scheduled time, so it is recommended.

## File locations

- Config: `~/.config/pkgup/config` (contains secrets — do not commit to git)
- Mask list: `~/.config/pkgup/masks` (one `manager:name` per line)
- Run logs: `~/.config/pkgup/logs/` (most recent 20 kept automatically)

## Notes / limitations

- The npm update default is `latest` (including major versions). To stay within semver
  ranges, set `NPM_UPDATE_TARGET="wanted"` in config.
- Casks that auto-update themselves are excluded from checks by default. To include them
  all, set `BREW_CASK_GREEDY=true`.
- If the Slack webhook is unset, only the sending is skipped; the report still goes to
  the log/stdout.
- An individual package upgrade failure does not abort the run; failures are collected
  under `Failed` in the report.

## Repository layout

```
.
├── .gitignore
├── CLAUDE.md              # project context for Claude Code
├── README.md
├── pkgup                  # main executable script (zsh)
└── config.example         # config template
```
