# daily-update

Updates, rebuilds, and restarts a local Fleet dev environment — git,
docker, db, ngrok, scep, and a shared-folder server, all in the right
order — plus a separate `--tuf` build mode.

The script is built from four phases that always run in this order. Run
them all with plain `daily-update`, or run just one via `--only=<phase>`.

## Phases

| Phase | What it does |
|---|---|
| `branch` | `git status` / `git pull` (asks to continue on uncommitted changes, default yes). Prompts for a branch to switch to. If you switch, offers a db backup and/or restore *before* anything is torn down (needs the docker network from the still-running containers). |
| `down` | Stops SCEP (if configured), Ctrl-C pyserver/ngrok/db, then `docker compose down`. Launches Docker Desktop automatically first if it isn't running. |
| `update` | `make deps` (auto-retries via NVM on a node-version failure) → `make generate` → `make build`. |
| `up` | `docker compose up -d` → `fleet prepare db --dev` (on a migration-looking failure: auto-checks `migration-cleanup`, then backup, then `cleanup`/`reset`/restore, then retries) → starts db/ngrok/pyserver/scep/watchdog in tmux. |

```bash
daily-update                  # all four phases, in order
daily-update --only=down      # just this phase, then exit
daily-update --only=branch my-branch   # branch phase, with a branch name
```

`--only=down` is the one to reach for when you just need everything
stopped without a rebuild — e.g. Docker Desktop wants to quit for an
update and you don't want to shut down each tmux window by hand.

## Prerequisites

| Tool | Used for | Install |
|---|---|---|
| `git`, `make` | everything | usually preinstalled (Xcode CLT) |
| Docker Desktop | `docker compose` | https://docker.com |
| `tmux` | keeps db/ngrok/scep/python/watchdog running (needed for `up`, `down`, and the full run — not `branch`/`update` alone) | `brew install tmux` |
| `ngrok` | tunneling | `brew install ngrok` |
| `python3` | optional shared-folder server | preinstalled on macOS |
| `nvm` | only if `make deps` hits a node-version mismatch | `brew install nvm` |
| SCEP server (`scepserver-darwin-arm64`) | optional local SCEP/CA server | build from [micromdm/scep](https://github.com/micromdm/scep), see `--setup` |
| `go` | powers `migration-cleanup` (`go run`) during a migration failure | usually already needed to build Fleet itself |

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
  Db user [fleet]:
  Db password [insecure]:
  Db name [fleet]:
```

Fleet directory is required and validated (needs a `Makefile`). Shared
folder and SCEP are optional — blank disables that step completely (no
window, no shutdown, no watchdog entry). The db user/password/name are for
`migration-cleanup` (see below) — defaults match Fleet's typical local dev
values, but confirm against your own `docker-compose.yml` if unsure. Setup
reruns automatically if the config is missing or Fleet/SCEP paths stop
being valid. Run `daily-update --setup` anytime to reconfigure by choice.

## Usage

```
daily-update [branch] [options]
daily-update --tuf
daily-update --only=<down|branch|update|up>
```

| Flag | Description |
|---|---|
| `branch` (positional) | Switch to this branch instead of being prompted. Only meaningful for the full run or `--only=branch`. |
| `-d`, `--defaults` | Fully automatic — no prompts, stays on current branch (unless one is given), skips the backup/restore menu. A migration failure still gets a backup first (auto-named), it's just not interactive — then prefers `migration-cleanup` if it reports a clean fix, otherwise `make db-reset`. |
| `-s`, `--silent` | Hides raw command output; step announcements, warnings, and errors still print. |
| `-h`, `--help` | Show usage and exit. |
| `--setup` | Run the config wizard and exit. |
| `--tuf` | Run the separate TUF flow instead of the normal steps. |
| `--only=<phase>` | Run just one phase (`down`/`branch`/`update`/`up`) instead of all four. |

Short flags combine: `-ds`/`-sd` = `-d -s`. Order doesn't matter for flags
or the branch name.

```bash
daily-update                  # fully interactive, all phases
daily-update my-branch -d     # auto mode, switch branch first
daily-update -ds              # automatic AND quiet
daily-update --tuf -s         # TUF flow, quiet
daily-update --only=up        # just bring it back up
```

## `--tuf` mode

Separate from the normal flow — no docker/tmux/db/scep involved:

1. Checks for uncommitted changes (same default-yes prompt).
2. Remembers current branch, checks out `main`, `git pull`.
3. `make deps` / `make generate` / `make build`.
4. Runs `./my_tuf.sh` in the Fleet directory.
5. Switches back to your original branch, builds again.

`-s` works here; `-d`, a branch argument, and `--only=` don't apply.

## Db backup, restore & migration-cleanup

On a branch switch, you're offered a db backup and/or restore before
continuing (same backup/restore mechanics as below).

On a migration-looking `fleet prepare db --dev` failure, a `migration-cleanup
--dry-run` (read-only) runs automatically against your current branch first
— connecting via `127.0.0.1` with the db user/password/name from `--setup`,
not `--config fleet.yml` (that file's `mysql` address is docker's *internal*
hostname, which only resolves inside the compose network and hangs when run
from the host). `cleanup` is only offered/auto-applied when the tool reports
`Dry-run: SQL will apply cleanly.` — if it finds renumbers but reports
issues instead, it's shown as a warning and you fall back to backup +
`reset` (also requires `YES`) or restore, though `cleanup` is still there to
try manually if you want to inspect it yourself. In `-d` mode: mandatory
backup first, then `cleanup` auto-applies only on that same clean signal,
otherwise `make db-reset` — no other prompts either way.

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

Startup (the `up` phase): docker → db → ngrok → pyserver → scep. The `down`
phase reverses that.

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
  failure instead of offering backup/reset/cleanup.
- NVM-fallback detection looks for `nvm`, or `node` + `version`/`engine`/
  `incompatible`, anywhere in `make deps`'s output (order-independent).
- `migration-cleanup`'s db user/password/name default to Fleet's typical
  local dev values (`fleet`/`insecure`/`fleet`) if not set during `--setup`
  — confirm these match your actual containers.
- Shared-folder IP lookup assumes `en0` is your active interface.
- SCEP shutdown assumes the binary name is unique on your machine.
- Docker auto-start assumes it's installed at the default location, named
  "Docker".
- If the retried `prepare db --dev` fails again, docker is left running
  without the app layer on top — not data loss, just a state to be aware of.
- No rollback/resume if the script itself is interrupted mid-run (though
  Ctrl-C during any single step now exits cleanly rather than continuing
  with stale state).