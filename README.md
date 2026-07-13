# daily-update

A single command that updates, rebuilds, and restarts a local Fleet dev
environment — git pull, rebuild, and bring docker/db/ngrok/python back up in
the right order — plus an optional separate mode for TUF builds.

## What it does (normal flow)

Running `daily-update` with no flags does all of the following, in order:

1. **`git status`** / **`git pull`**
   - If there are uncommitted changes, asks `Continue anyway? [Y/n]` (default
     yes — press Enter to continue, type `n` to stop).
2. **Branch selection.** Prompts for a branch to switch to, or press Enter to
   stay on the current one.
   - If you *do* switch branches, and only then: offers to back up the
     current db to a named `.sql.gz` file, then offers to restore a
     different existing backup instead. Both happen here, before anything
     is torn down, because the backup/restore scripts need the docker
     network from the *currently running* containers.
3. **Spin down.** Sends Ctrl-C into the `pyserver`, `ngrok`, and `db` tmux
   windows (in that order), then runs `docker compose down`.
4. **Build.** `make deps` (auto-retries once via NVM if the failure looks
   like a node version mismatch), then `make generate`, then `make build`.
5. **Spin back up.**
   - `docker compose up -d`
   - `fleet prepare db --dev` — if this fails with a migration-looking
     error (common after switching to an older branch), it shows the error,
     offers a db backup, then asks you to type `reset` (runs
     `make db-reset`) or the name of a backup file to restore, then retries
     automatically.
   - Starts `fleet serve`, `ngrok start local`, and (if configured)
     `python3 -m http.server` in their own tmux windows, which keep running
     after the script exits.
   - Starts a `watchdog` window that polls those processes and fires a
     macOS notification if one of them exits unexpectedly.

## Prerequisites

| Tool | Used for | Install |
|---|---|---|
| `git` | pulling/switching branches | usually preinstalled (Xcode CLT) |
| `make` | deps/generate/build/db-reset | usually preinstalled (Xcode CLT) |
| Docker Desktop | `docker compose` | https://docker.com |
| `tmux` | keeping db/ngrok/python/watchdog running in the background | `brew install tmux` |
| `ngrok` | tunneling | `brew install ngrok` |
| `python3` | optional shared-folder server | preinstalled on macOS |
| `nvm` | only used if `make deps` fails on a node version mismatch | `brew install nvm` |

`osascript` (for watchdog notifications) is built into macOS — nothing to
install. If you don't see notification banners, check
**System Settings → Notifications** for your terminal app.

## Installation

```bash
mkdir -p ~/bin
mv daily-update ~/bin/daily-update
chmod +x ~/bin/daily-update
```

Make sure `~/bin` is on your `PATH` (add to `~/.zshrc` if it isn't):

```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Confirm it's found:

```bash
which daily-update
```

## First run / configuration

The first time you run `daily-update`, it walks you through a short setup:

```
Fleet repo directory [/Users/you/fleet]:
Directory to serve via 'python3 -m http.server' (leave blank to skip this entirely):
Db backup directory [/Users/you/fleet/tools/backup_db]:
```

This is saved to `~/.daily-update.conf` and reused on every future run — you
won't see these prompts again unless:

- the config file goes missing, or
- the Fleet directory it points to no longer has a `Makefile` in it (e.g. you
  moved or renamed the repo)

in which case it re-runs the wizard automatically before continuing.

Leaving the shared-folder prompt blank disables `python3 -m http.server`
entirely — no window, no shutdown attempt, no watchdog entry for it.

You can also trigger the wizard manually anytime you want to change something
(e.g. add/remove the shared folder, or point at a different Fleet checkout):

```bash
daily-update --setup
```

## Usage

```
daily-update [branch] [options]
daily-update --tuf
```

### Arguments

| Argument | Description |
|---|---|
| `branch` | Optional. Switch to this branch instead of being prompted. |

### Options

| Flag | Description |
|---|---|
| `-d`, `--defaults` | Fully automatic — no prompts. Stays on the current branch (unless one is also given), skips db backup/restore prompts, and auto-runs `make db-reset` if a migration failure is hit. Every step still runs; this only skips the interactive parts. |
| `-s`, `--silent` | Suppresses the raw (dimmed) command output from each step. Step announcements, warnings, and full error output on failure still print. |
| `-h`, `--help` | Show usage and exit. Doesn't touch tmux, git, or anything else. |
| `--setup` | Run the configuration wizard and exit. |
| `--tuf` | Run the separate TUF build flow instead of the normal steps (see below). |

### Examples

```bash
daily-update                  # fully interactive, prompts for everything
daily-update my-branch        # switch to my-branch, still prompts for backups
daily-update -d               # fully automatic, stay on current branch
daily-update my-branch -d     # auto mode, but switch to my-branch first
daily-update -s               # normal interactive flow, quiet output
daily-update -d -s            # fully automatic AND quiet
daily-update --setup          # reconfigure FLEET_DIR/SHARED_DIR/BACKUP_DIR
daily-update --tuf             # run the separate TUF build flow
daily-update --tuf -s          # TUF build flow, quiet output
```

## `--tuf` mode

A completely separate flow — does **not** run any of the normal steps above,
and doesn't touch docker, tmux, or the db at all:

1. Remembers your current branch, checks out `main`
2. `git pull`
3. `make deps` / `make generate` / `make build`
4. Runs `./my_tuf.sh` in the Fleet directory
5. Switches back to your original branch
6. `make deps` / `make generate` / `make build` again, to leave your working
   tree matching where you started

`-s` works with `--tuf`; `-d` and a branch argument are not used in this mode.

## Db backup & restore

Whenever you switch branches, and whenever `fleet prepare db --dev` fails
with a migration-looking error, you'll be offered:

- **Back up first?** Type a name like `v4.89.0-backup` (the `.sql.gz`
  extension is added automatically if you leave it off) to run
  `tools/backup_db/backup.sh`, or press Enter to skip.
- **Restore or reset?** Existing `.sql.gz` files in your configured backup
  directory are listed by name. Type one of those names to run
  `tools/backup_db/restore.sh`, type `reset` to run `make db-reset` (only
  offered after an actual failure, not on a plain branch switch), or press
  Enter to keep the current db (branch-switch prompt only).

In `-d` (auto) mode, backups are skipped and a migration failure
auto-resolves via `make db-reset`, without asking.

## Tmux session

Everything long-running lives in a tmux session named `fleet-dev`:

| Window | Runs |
|---|---|
| `db` | `fleet serve --dev --config fleet.yml --dev_license --debug --logging_debug` |
| `ngrok` | `ngrok start local` |
| `pyserver` | `python3 -m http.server` (only if configured) |
| `watchdog` | polls the above and sends a macOS notification if one crashes |

```bash
tmux attach -t fleet-dev     # view any window
# Ctrl-b w                   # list windows and pick one
# Ctrl-b d                   # detach again (leaves everything running)
```

## Known assumptions to double-check

- `BACKUP_DIR` defaults to `<fleet dir>/tools/backup_db` — adjust via
  `daily-update --setup` if `backup.sh`/`restore.sh` actually write
  elsewhere.
- `fleet.yml` is assumed to be in the root of the Fleet directory (used
  as-is by the `serve` command).
- The migration-failure detector looks for `FAIL`, `WARNING`, or
  `missing...migration` (case-insensitive) in `fleet prepare db --dev`'s
  output — if a real migration failure ever uses different wording, it'll
  fall through to a plain failure instead of offering backup/reset.