# daily-update

Updates, rebuilds, and restarts a local Fleet dev environment — git,
docker, db, ngrok, scep, and a shared-folder server, all in the right
order — plus a separate `--tuf` build mode.

## What it does

1. `git status` / `git pull` (asks to continue on uncommitted changes,
   default yes).
2. Prompts for a branch to switch to. If you switch, offers a db backup
   and/or restore *before* anything is torn down (needs the docker network
   from the still-running containers).
3. Spin down: stop SCEP (if configured), Ctrl-C pyserver/ngrok/db, then
   `docker compose down`. Launches Docker Desktop automatically if it's not
   running.
4. `make deps` (auto-retries via NVM on a node-version failure) → `make
   generate` → `make build`.
5. Spin up: `docker compose up -d` → `fleet prepare db --dev` (on a
   migration-looking failure: backup, then `reset` or restore, then
   retries) → starts db/ngrok/pyserver/scep/watchdog in tmux.

## Prerequisites

| Tool | Used for | Install |
|---|---|---|
| `git`, `make` | everything | usually preinstalled (Xcode CLT) |
| Docker Desktop | `docker compose` | https://docker.com |
| `tmux` | keeps db/ngrok/scep/python/watchdog running | `brew install tmux` |
| `ngrok` | tunneling | `brew install ngrok` |
| `python3` | optional shared-folder server | preinstalled on macOS |
| `nvm` | only if `make deps` hits a node-version mismatch | `brew install nvm` |
| SCEP server (`scepserver-darwin-arm64`) | optional local SCEP/CA server | build from [micromdm/scep](https://github.com/micromdm/scep), see `--setup` |

`osascript` and `ipconfig` are built into macOS. If watchdog notifications
don't show, check **System Settings → Notifications** for your terminal app.

## Installation

```bash
mkdir -p ~/bin
mv daily-update ~/bin/daily-update
chmod +x ~/bin/daily-update
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc   # if not already on PATH
source ~/.zshrc
which daily-update
```

## First run / configuration

First run walks you through setup, saved to `~/.daily-update.conf`:

```
Fleet repo directory [/Users/you/fleet]:
Directory to serve via 'python3 -m http.server' (leave blank to skip this entirely):
Db backup directory [/Users/you/fleet/tools/backup_db]:
SCEP server directory (leave blank to skip this entirely):
```

Fleet directory is required and validated (needs a `Makefile`). Shared
folder and SCEP are optional — blank disables that step completely (no
window, no shutdown, no watchdog entry). Setup reruns automatically if the
config is missing or Fleet/SCEP paths stop being valid. Run
`daily-update --setup` anytime to reconfigure by choice.

## Usage

```
daily-update [branch] [options]
daily-update --tuf
```

| Flag | Description |
|---|---|
| `branch` (positional) | Switch to this branch instead of being prompted. |
| `-d`, `--defaults` | Fully automatic — no prompts, stays on current branch (unless one is given), skips the backup/restore menu. A migration failure still gets a backup first (auto-named), it's just not interactive — then auto-runs `make db-reset`. |
| `-s`, `--silent` | Hides raw command output; step announcements, warnings, and errors still print. |
| `-h`, `--help` | Show usage and exit. |
| `--setup` | Run the config wizard and exit. |
| `--tuf` | Run the separate TUF flow instead of the normal steps. |

Short flags combine: `-ds`/`-sd` = `-d -s`. Order doesn't matter for flags
or the branch name.

```bash
daily-update                  # fully interactive
daily-update my-branch -d     # auto mode, switch branch first
daily-update -ds              # automatic AND quiet
daily-update --tuf -s         # TUF flow, quiet
```

## `--tuf` mode

Separate from the normal flow — no docker/tmux/db/scep involved:

1. Checks for uncommitted changes (same default-yes prompt).
2. Remembers current branch, checks out `main`, `git pull`.
3. `make deps` / `make generate` / `make build`.
4. Runs `./my_tuf.sh` in the Fleet directory.
5. Switches back to your original branch, builds again.

`-s` works here; `-d` and a branch argument don't apply.

## Db backup & restore

On a branch switch, and on a migration-looking `fleet prepare db --dev`
failure, you can back up (type a name, `.sql.gz` added automatically) and/or
restore an existing backup (listed by name) or `reset` (`make db-reset`).
Typing `reset` requires a second `YES` confirmation.

Backups are verified, not just assumed — the script confirms the file
actually exists in `BACKUP_DIR` (relocating it there if `backup.sh` wrote to
the repo root instead, which it does) before treating it as successful, and
stops rather than continuing if it can't confirm one. This applies even in
`-d` mode: the pre-reset backup is never skipped, just non-interactive.

## SCEP server

If configured, starts `scepserver-darwin-arm64 -depot depot -port 2016
-challenge=secret -allowrenew 0` in its own tmux window, and verifies it's
still running a couple seconds later (shown either way, even with `-s`).
Shutdown uses `pkill` instead of Ctrl-C — the binary has a real bug in its
SIGINT handling but no SIGTERM handler, so a plain kill is the reliable
option, scoped tightly to that binary + port to avoid touching anything
else.

## Shared folder

If configured, starts `python3 -m http.server` and prints the reachable
IP (`ipconfig getifaddr en0`, looked up fresh each run since it can change).

## Tmux session (`fleet-dev`)

| Window | Runs |
|---|---|
| `db` | `fleet serve --dev --config fleet.yml --dev_license --debug --logging_debug` |
| `ngrok` | `ngrok start local` |
| `pyserver` | `python3 -m http.server` (if configured) |
| `scep` | scepserver (if configured) |
| `watchdog` | polls the above, macOS notification on crash |

Startup: docker → db → ngrok → pyserver → scep. Shutdown reverses that.

```bash
tmux attach -t fleet-dev
# Ctrl-b w   list windows
# Ctrl-b d   detach (leaves everything running)
```

## Known limitations / assumptions

- `BACKUP_DIR` defaults to `<fleet dir>/tools/backup_db` — adjust via
  `--setup` if `backup.sh`/`restore.sh` write elsewhere.
- `fleet.yml` is assumed to live in the Fleet directory root.
- Migration-failure detection looks for `FAIL`/`WARNING`/`missing...migration`
  in `prepare db --dev`'s output; different wording falls through to a plain
  failure instead of offering backup/reset.
- NVM-fallback detection looks for `nvm`, or `node` + `version`/`engine`/
  `incompatible`, anywhere in `make deps`'s output (order-independent).
- Shared-folder IP lookup assumes `en0` is your active interface.
- SCEP shutdown assumes the binary name is unique on your machine.
- Docker auto-start assumes it's installed at the default location, named
  "Docker".
- If the retried `prepare db --dev` fails again, docker is left running
  without the app layer on top — not data loss, just a state to be aware of.
- No rollback/resume if the script itself is interrupted mid-run.