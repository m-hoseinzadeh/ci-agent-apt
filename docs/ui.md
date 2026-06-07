# Admin UI guide

## Dashboard `/`

Live server gauges (CPU, memory, per-disk usage, load, 10-minute
sparklines), active and queued runs, and the most recent run history.
Refreshes every few seconds.

## Projects `/projects`

Create and edit projects. A project is: name, slug (immutable), git URL +
branch, optional SSH deploy key path, compose file path, domain + HTTP port
(for the nginx guide), env vars (encrypted at rest; shown masked), optional
HMAC secret, optional per-project run timeout.

Per project you also get:

- **Webhook URLs** with a rotate-token action.
- **Manual triggers** — run from the configured repo (with optional
  URL/branch override), from a ZIP URL, or by uploading a ZIP file.
- **Run history** with per-run status, commit, duration and error summary.
- **Redeploy** buttons on successful runs — enabled only while the run's
  images still exist on the host.
- **nginx guide** — copy-paste commands to wire `domain → 127.0.0.1:<port>`
  in the host nginx (one time per project).
- Archive/unarchive (archived projects reject webhooks).

## Run detail `/runs/<id>`

Status timeline, source/commit/image metadata, and the **live log**: the
full run log replays instantly, then streams in real time while the run is
active. A `Cancel` button TERM→KILLs the run's process tree.

## Docker `/docker`

Disk usage breakdown (images / containers / volumes / build cache),
the agent's built images with **protected** badges, and cleanup actions:

- prune dangling images
- prune stopped containers
- prune build cache
- prune unused volumes (double-confirm — volumes hold data!)
- delete a specific run's images

Images belonging to the last *N* successful runs of every project
(`redeployable_runs_per_project`) are **never** pruned, so recent versions
always stay redeployable. **The prune actions operate on the whole Docker
daemon** — if other software shares this host's Docker, its dangling images
and stopped containers are affected too.

## Audit `/audit`

Every admin action and accepted webhook: logins (and failures), project
changes, token rotations, triggers, cancellations, prunes, settings changes.

## Maintenance `/maintenance`

- **System & versions** — ci-agent version + uptime, OS/kernel, and the
  installed versions of Docker Engine, Docker Compose, nginx, git and curl.
- **Check for updates** — an *explicit* button (the agent never phones home
  on its own) that compares installed versions against the latest upstream
  releases via api.github.com. On an air-gapped server it simply reports
  "unreachable" — expected and harmless. Docker/Compose/nginx results are
  informational; install those through your distribution's packages.
- **ci-agent self-update** — the flow adapts to how the agent was installed:
  - **apt package** (detected via dpkg): the page simply shows the two
    `apt-get` upgrade commands — package management stays with apt.
  - otherwise, *Download latest release* fetches the static binary +
    `SHA256SUMS` from this repo's releases (set `github_token` in the config
    while the repo is private) and verifies the checksum; nothing is applied
    yet. Then: if the agent owns its executable path, an **Apply update &
    restart** button swaps the binary (keeping `ci-agent.previous` for
    rollback) and restarts via systemd; with a root-owned binary
    (`/usr/bin`), the page shows the two `sudo` commands to run instead.
    A staged update can be discarded. Requires `Restart=always` in the
    systemd unit (the shipped one has it).
- **Storage** — size breakdown of the data dir (database, run snapshots,
  backups, workspaces, uploads) and free disk.
- **Backups** — create a backup on demand (`VACUUM INTO`, safe while runs are
  active), download or delete existing backups. The nightly housekeeping
  backup (keep 7) continues regardless.
- **Restore** — upload a backup `.db`. It is validated and **staged**: the
  swap happens at the next `systemctl restart ci-agent`, and the replaced
  database is kept beside it as `ci-agent.db.pre-restore-<timestamp>`. A
  staged restore can be cancelled until the restart. Note: encrypted env
  vars also need the matching `secret.key`.
- **Run housekeeping now** — triggers the nightly pass on demand.

## Settings `/settings`

Change the admin password, toggle nightly auto-prune. Operational knobs
(concurrency, timeouts, retention, pull policy…) are shown read-only — they
live in `/etc/ci-agent/config.toml`, the single source of truth; edit the
file and restart the service to change them.
