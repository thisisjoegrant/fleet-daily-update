# daily-update

A single command that updates, rebuilds, and restarts a local Fleet dev
environment — git pull, rebuild, and bring docker/db/ngrok/scep/python back
up in the right order — plus an optional separate mode for TUF builds.

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
3. **Spin down.** Stops the SCEP server if configured (see below for why
   this isn't a simple Ctrl-C), sends Ctrl-C into the `pyserver`, `ngrok`,
   and `db` tmux windows (in that order), then runs `docker compose down`.
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
     `python3 -m http.server` and the SCEP server, each in their own tmux
     windows, which keep running after the script exits.
   - If the shared folder is enabled, prints the local IP so you can reach
     it from another device (see below — your IP can change between runs).
   - If the SCEP server is enabled, checks that it actually stayed running
     (see below) — shown in the output either way, even in `-s` mode.
   - Starts a `watchdog` window that polls whichever of db/ngrok/scep/
     pyserver are enabled and fires a macOS notification if one exits
     unexpectedly.

## Prerequisites

| Tool | Used for | Install |
|---|---|---|
| `git` | pulling/switching branches | usually preinstalled (Xcode CLT) |
| `make` | deps/generate/build/db-reset | usually preinstalled (Xcode CLT) |
| Docker Desktop | `docker compose` | https://docker.com |
| `tmux` | keeping db/ngrok/scep/python/watchdog running in the background | `brew install tmux` |
| `ngrok` | tunneling | `brew install ngrok` |
| `python3` | optional shared-folder server | preinstalled on macOS |
| `nvm` | only used if `make deps` fails on a node version mismatch | `brew install nvm` |
| SCEP server (`scepserver-darwin-arm64`) | optional local SCEP/CA server for MDM enrollment | build/download from [micromdm/scep](https://github.com/micromdm/scep) into its own directory (see `--setup`) |

`osascript` (for watchdog notifications) and `ipconfig` (for the shared-folder
IP) are both built into macOS — nothing to install. If you don't see
notification banners, check **System Settings → Notifications** for your
terminal app.

### Setting up the SCEP server (optional)

Like the shared folder, the SCEP server is entirely optional — leave it
unconfigured and `daily-update` skips it completely (no window, no shutdown
attempt, no watchdog entry). If you do want it:

1. Get `scepserver-darwin-arm64` (build from [micromdm/scep](https://github.com/micromdm/scep)
   or download a release) and place it in its own directory — there's no
   fixed default, since it's off unless you set one.
2. Initialize a CA/depot in that directory if you haven't already (see the
   micromdm/scep docs — typically `./scepserver-darwin-arm64 ca -init`,
   which creates the `depot` folder `daily-update` expects).
3. Point `daily-update --setup` at that directory (see below). If you give
   it a path, it checks for the executable and keeps re-prompting until it
   finds it or you leave it blank.

When enabled, `daily-update` always runs it as:

```bash
./scepserver-darwin-arm64 -depot depot -port 2016 -challenge=secret -allowrenew 0
```

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
SCEP server directory (leave blank to skip this entirely):
```

This is saved to `~/.daily-update.conf` and reused on every future run — you
won't see these prompts again unless:

- the config file goes missing,
- the Fleet directory it points to no longer has a `Makefile` in it (e.g. you
  moved or renamed the repo), or
- a SCEP directory is set but no longer has an executable
  `scepserver-darwin-arm64` in it

in which case it re-runs the wizard automatically before continuing. The
Fleet directory prompt is required and will keep re-asking until it's valid.
The shared-folder and SCEP-server prompts are both optional — leaving either
blank disables that step entirely (no window, no shutdown attempt, no
watchdog entry, no IP lookup for the shared folder). If you do give a path
for either, it's validated and will keep re-prompting until it's right or
you clear it back to blank.

You can also trigger the wizard manually anytime you want to change something
(e.g. add/remove the shared folder or SCEP server, or point at a different
Fleet directory):

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
| `-s`, `--silent` | Suppresses the raw (dimmed) command output from each step. Step announcements, warnings, and full error output on failure still print — including SCEP start/stop status, if enabled. |
| `-h`, `--help` | Show usage and exit. Doesn't touch tmux, git, or anything else. |
| `--setup` | Run the configuration wizard and exit. |
| `--tuf` | Run the separate TUF build flow instead of the normal steps (see below). |

Short flags can be combined: `-d` and `-s` can be written together as `-ds`
or `-sd`, same as `-d -s`. This works for any current or future single-letter
flags, so `-dsh` etc. would also expand correctly.

### Examples

```bash
daily-update                  # fully interactive, prompts for everything
daily-update my-branch        # switch to my-branch, still prompts for backups
daily-update -d               # fully automatic, stay on current branch
daily-update my-branch -d     # auto mode, but switch to my-branch first
daily-update -s               # normal interactive flow, quiet output
daily-update -ds              # fully automatic AND quiet (same as -d -s)
daily-update --setup          # reconfigure FLEET_DIR/SHARED_DIR/BACKUP_DIR/SCEP_DIR
daily-update --tuf            # run the separate TUF build flow
daily-update --tuf -s         # TUF build flow, quiet output
```

## `--tuf` mode

A completely separate flow — does **not** run any of the normal steps above,
and doesn't touch docker, tmux, the db, or the SCEP server at all:

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

`backup.sh`/`restore.sh` write/read relative to the Fleet repo root rather
than into `BACKUP_DIR` directly — `daily-update` relocates the file into
`BACKUP_DIR` right after a backup, and stages it back into the repo root
temporarily for a restore, so the canonical copy always lives in
`BACKUP_DIR` regardless of where the scripts themselves actually write.

## Shared folder (python http.server)

If `SHARED_DIR` is configured, `daily-update` starts
`python3 -m http.server` in that directory as part of spinning back up, and
prints where to reach it:

```
==> Shared folder available at 192.168.1.42:8000
```

The IP is looked up fresh on every run via `ipconfig getifaddr en0`, since
local IPs can change. If it can't be determined (e.g. `en0` isn't your active
network interface), you'll get a warning telling you to check the `pyserver`
tmux window directly instead.

## SCEP server (optional)

If `SCEP_DIR` is configured, `daily-update` starts the SCEP server as part
of spinning back up:

```bash
cd "$SCEP_DIR" && ./scepserver-darwin-arm64 -depot depot -port 2016 -challenge=secret -allowrenew 0
```

After starting it, the script waits 2 seconds and checks that it's actually
still running (rather than having immediately crashed) — you'll see a
success or failure message either way, even with `-s`. On failure, the last
20 lines of that window's output are shown directly in the main output so
you don't have to go attach to tmux to see what went wrong.

If `SCEP_DIR` isn't set, this entire step — window, shutdown, watchdog
entry — is skipped, same as the shared folder.

**Why shutdown doesn't use Ctrl-C:** `scepserver-darwin-arm64` has a
confirmed bug in its own SIGINT handling — it uses an unbuffered Go channel
with `signal.Notify`, which Go's own documentation warns can silently drop
signals if the receiving goroutine isn't parked on the exact receive at the
moment the signal arrives. That's why Ctrl-C sometimes works and sometimes
just leaves it running in the background. It has no handler for `SIGTERM`
at all, so a plain `kill` hits the default OS disposition (immediate
termination) — reliable every time. `daily-update` automates exactly that:
`pkill -f scepserver-darwin-arm64`, scoped tightly to that binary name so it
can't affect anything else running on your machine.

## Tmux session

Everything long-running lives in a tmux session named `fleet-dev`:

| Window | Runs |
|---|---|
| `db` | `fleet serve --dev --config fleet.yml --dev_license --debug --logging_debug` |
| `ngrok` | `ngrok start local` |
| `pyserver` | `python3 -m http.server` (only if configured) |
| `scep` | `scepserver-darwin-arm64 -depot depot -port 2016 -challenge=secret -allowrenew 0` (only if configured) |
| `watchdog` | polls whichever of the above are enabled and sends a macOS notification if one crashes |

Startup order is docker → db → ngrok → pyserver (if enabled) → scep (if
enabled). Shutdown reverses that: scep (if enabled) → pyserver (if enabled)
→ ngrok → db → docker.

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
- The NVM-fallback detector for `make deps` looks for the word `nvm`
  anywhere, or `node` together with `version`/`engine`/`incompatible`
  anywhere in the output (order-independent) — this covers the yarn
  "engine incompatible" error format as well as more typical phrasing.
- The shared-folder IP lookup assumes `en0` is your active network
  interface (typical for Wi-Fi on most Macs). If you're primarily on a
  different interface, the printed IP may be wrong or missing.
- SCEP shutdown assumes the binary is exactly named `scepserver-darwin-arm64`
  and that no other unrelated process on your machine matches that name —
  `pkill -f` is scoped to that string specifically to keep this safe.
