# Admin UI guide

## Dashboard `/`

Live server gauges (CPU, memory, per-disk usage, load, 10-minute
sparklines), active and queued runs, and the most recent run history.
Refreshes every few seconds.

## Projects `/projects`

Create and edit projects. A project is: name, slug (immutable), a
[deploy mode](./projects.md) and its source (git URL + branch, a Dockerfile
path, a prebuilt image, or pasted compose), HTTP port, env vars (encrypted at
rest; shown masked), optional per-project working directory (monorepos) and run
timeout. Private GitHub clones authenticate with a token registered on the
**GitHub tokens** page (see below).

Per project you also get:

- **Webhook URLs** with a rotate-token action (source modes only).
- **Manual triggers** — run from the configured repo (with optional
  URL/branch override), from a ZIP URL, or by uploading a ZIP file; sourceless
  (Image / Inline-compose) projects get a **Deploy** button instead.
- **Run history** with per-run status, commit, duration and error summary.
- **Redeploy** buttons on successful runs — enabled only while the run's
  images still exist on the host.
- **Domains** — add multiple hostnames, mark one canonical (others 301), and
  opt each into HTTPS. See [Domains & TLS](./domains.md).
- **nginx** page — live status, the generated server block with a drift badge,
  and **one-click apply** (write → `nginx -t` → reload) or the equivalent
  copy-paste commands.
- **Container logs** `/projects/<slug>/logs` — live `docker compose logs`
  for the deployed stack, with a service picker, word-wrap toggle, and a clear
  button (see below).
- Archive/unarchive (archived projects reject webhooks).
- **Delete** — a hard delete that removes the project and **all** its Docker
  resources (containers, images, networks), run history, workspace and nginx
  config. Type the slug to confirm; blocked while a run is in flight.

## Run detail `/runs/<id>`

Status timeline, source/commit/image metadata, and the **live log**: the
full run log replays instantly, then streams in real time while the run is
active. A `Cancel` button TERM→KILLs the run's process tree.

## Container logs `/projects/<slug>/logs`

Live output of the project's **currently deployed** containers (distinct from
run logs, which capture a build/deploy). Pick a single service or stream all of
them, toggle word-wrap for long lines, and clear the view to watch only new
output. The stream auto-sticks to the tail unless you scroll up to read
scrollback.

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

- **System & versions** — CI Agent version + uptime, OS/kernel, and the
  installed versions of Docker Engine, Docker Compose, nginx, git and curl.
- **Check for updates** — an *explicit* button (the agent never phones home
  on its own) that compares installed versions against the latest upstream.
  CI Agent's own latest version comes from the **public apt repository** (no
  token needed, even while the source repo is private); Docker/Compose/nginx
  come from api.github.com. On an air-gapped server it simply reports
  "unreachable" — expected and harmless. Docker/Compose/nginx results are
  informational; install those through your distribution's packages.
- **CI Agent update** — two ways to update from this page:
  - *apt* — on a connected server, run the shown `apt-get` upgrade commands.
  - *Signed upload (offline)* — upload either the whole
    `ci-agent-offline-*.tar.gz` release bundle (the agent takes the `.deb` and
    its `.deb.asc` out of it; the bundle's own public key is ignored), **or**
    the release `.deb` **and** its detached `.asc` signature directly. The agent
    verifies the signature **offline** against the
    embedded release key (pinned to fingerprint
    `A80F7309CF26133DBE88E2EC9C5489AAE86BF159`; any other key is rejected) and
    refuses downgrades, then **stages** the new binary. Click **Apply &
    restart** to install it: the service restarts and a privileged pre-start
    step swaps the binary in. If the new binary crash-loops it is automatically
    rolled back to the previous one after three failed starts. A staged update
    can be discarded until you apply it.
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

## GitHub tokens `/github`

Register one GitHub personal access token per username (the repository owner).
When a project clones `https://github.com/<username>/…`, the token registered
for that username authenticates the clone. Tokens are encrypted at rest with
`secret_key_file` and never shown again — re-submitting a username replaces its
token. Delete a token to revoke access. Add the token once here instead of
per project; it covers every private repo owned by that account.

## Settings `/settings`

Change the admin password, toggle nightly auto-prune. Operational knobs
(concurrency, timeouts, retention, pull policy…) are shown read-only — they
live in `/etc/ci-agent/config.toml`, the single source of truth; edit the
file and restart the service to change them.
