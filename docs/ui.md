# Admin UI guide

## Logging in & two-factor authentication

Login has two steps:

1. Enter the admin **password**.
2. Enter the 6-digit **code** from your authenticator app (or a one-time
   backup code).

The first time you log in, the panel makes you set up two-factor
authentication (2FA) before anything else. It shows a **QR code** — scan it
with an authenticator app such as Google Authenticator — and **10 backup
codes**. Save the backup codes somewhere safe; each one works once if you
ever lose your phone.

> **Lost your phone?** Use a backup code on the code-entry screen, or run
> `ci-agent reset-2fa` on the server to clear 2FA and enroll again. See
> [Security](./security.md) for the full 2FA model.

## Light & dark theme

A theme toggle at the bottom of the sidebar switches the whole panel between
**dark** (default) and **light**. The choice is remembered in the browser
(`localStorage`) and applied before first paint, so there's no flash on reload.
It's purely cosmetic and per-browser — nothing server-side changes.

## Dashboard `/`

Live server gauges (CPU, memory, per-disk usage, load, 10-minute
sparklines), active and queued runs, and the most recent run history.
Refreshes every few seconds.

## Projects `/projects`

Create and edit projects. A project is: name, slug (immutable), a
[deploy mode](./projects.md) and its source (git URL + branch, a Dockerfile
path, a prebuilt image, or pasted compose), HTTP port, env vars (encrypted at
rest; shown in **cleartext** in the form so you can inspect and edit them — the
admin is the sole operator of their own box), optional per-project working
directory (monorepos) and run timeout. Private GitHub clones authenticate with a
token registered on the **GitHub tokens** page (see below).

The same edit form also exposes **per-project nginx tuning** — max upload size
(`client_max_body_size`), proxy timeout, a WebSocket/SSE forwarding toggle, and
a free-text box for extra directives — validated by `nginx -t` on save and
rolled back if invalid. See [Domains & TLS](./domains.md).

Per project you also get:

- **Webhook URLs** with a rotate-token action (source modes only).
- **Manual triggers** — run from the configured repo (with optional
  URL/branch override), from a ZIP URL, or by uploading a ZIP file; sourceless
  (Image / Inline-compose) projects get a **Deploy** button instead.
- **Run history** with per-run status, commit, duration and error summary.
- **Redeploy** buttons on successful runs — enabled only while the run's
  images still exist on the host.
- **Domains** — add multiple hostnames, mark one canonical (others 301), and
  opt each into HTTPS. When a global **base domain** is set (Settings → Default
  subdomain), the project also gets an auto-generated `<slug>.<base>` URL with
  its own enable toggle. See [Domains & TLS](./domains.md).
- **nginx** page — live status, the generated server block with a drift badge,
  and **one-click apply** (write → `nginx -t` → reload) or the equivalent
  copy-paste commands.
- **Container lifecycle** — a table of the deployed stack's containers
  (service, container, state) with per-container **Start / Stop / Restart**
  buttons. These act on the running containers, not on a redeploy.
- **Custom actions** `/projects/<slug>/actions` — operator-defined operations
  that run inside a chosen container: a plain **run** command, a **backup**, or
  an **upload-and-restore** (upload a file, then run a load command with
  `{{FILE}}` substituted). Actions are defined as a small JSON array on the
  Actions page; once any exist, an **Actions** button appears on the project
  page. Handy for database dumps/restores. See [Projects](./projects.md).
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

## Terminals

A **terminal** is an interactive command-line session you drive from the
browser — the same as typing in a real shell on the server.

- **Server shell** `/terminal` — a shell on the **host** itself (the machine
  running the agent). Use it for quick checks and fixes without a separate
  SSH login.
- **Container shell** `/projects/<slug>/terminal` — a shell **inside** one of
  a project's running containers (like `docker exec`). Pick the container
  from the list, then run commands inside it.

Both are full terminals: colours, interactive programs and window-resize all
work.

> **Important:** the server shell runs with the agent's own permissions,
> which include Docker access (root-equivalent on the host). Treat it like
> root SSH access.

**Copy a file into a container** — below the container shell, a small form
pushes a file into the selected container at a destination path (blank =
`/tmp/`). The source is either a **file upload** from your browser or a **path
on the host** (a file or directory the agent can read). Useful for dropping a
config or a restore artifact next to a running app.

## Files

A point-and-click file manager — the same operations as the shell, without the
typing. Two surfaces share one UI:

- **Host files** `/files` — browse the **host filesystem** as the agent's user.
  It only sees and changes what that user owns (the same trust level as the
  server shell — treat it as root-equivalent).
- **Container files** `/projects/<slug>/files` (the **Files** button on a
  project, or the project shell page) — browse **inside** a container of the
  project's stack, picked from a dropdown. Browsing needs a shell in the image,
  so distroless / `scratch` containers show no listing.

In either mode you can navigate directories, **download** a file to your
browser, **upload** a file into the current directory, **edit** small text
files inline, create folders, and rename or **delete** entries. Delete and
overwrite are confirmed in the browser and recorded in the [audit log](./audit.md);
deletes are recursive for folders, so double-check the path. Very large files
should be moved over the shell rather than the inline editor (capped at 1 MB).

## Host `/host`

Host-level controls in one place:

- **Services** — status plus restart / reload / log-tail buttons for `nginx`,
  the `docker` engine, and `ci-agent` itself. Restarting Docker bounces running
  containers; restarting ci-agent reloads this panel (it reconnects after a few
  seconds). "Logs" tails the last 200 `journalctl` lines for the unit.
- **Admin panel nginx** — tunes the vhost that serves this panel
  (`/etc/nginx/conf.d/ci-agent.conf`). The knobs (max upload, proxy timeout,
  extra `location` directives) keep the core proxy generated so you can't lock
  yourself out. An **Advanced** section can replace the whole config verbatim —
  `nginx -t` catches syntax errors and rolls back, but a wrong `proxy_pass`
  still passes the test and would cut off the panel, so a **Reset to default**
  button and the SSH / loopback fallback are the recovery path.
- **Global nginx config** — an http-level snippet included before every site
  (`gzip`, `server_tokens off;`, a shared `limit_req_zone`, …). Server blocks
  belong to projects, not here.

All three apply through the same safe path as the project nginx page
(write → `nginx -t` → reload, rolled back on a failed test).

## Storage `/storage`

A read-only storage report on top, then disk cleanup actions.

**Report (read-only):**

- per-filesystem usage meters
- a composition estimate of the disk holding the data dir
  (CI Agent data / Docker / other / free) — “other / system” is whatever the
  unprivileged agent can't attribute (OS packages, apt cache, system logs)
- a breakdown of the agent's own data dir (database, run snapshots, backups,
  workspaces, uploads)

**Cleanup** — each section shows what it removes, an estimate of how much it
will free, the items affected, and a single button:

- dangling images
- stopped containers
- build cache
- unused volumes (double-confirm — volumes hold data!)
- delete a specific agent image
- build workspaces (recreated on the next deploy)
- old run logs & snapshots (trimmed to retention)
- stale temp files left by interrupted update/restore uploads

Images belonging to the last *N* successful runs of every project
(`redeployable_runs_per_project`) and project-data volumes are **protected** and
**never** removed, so recent versions stay redeployable and app data is safe.
**The Docker actions operate on the whole Docker daemon** — if other software
shares this host's Docker, its dangling images and stopped containers are
affected too. OS-level cleanup (apt cache, system logs) is intentionally not
offered: the agent runs unprivileged — use the **Server shell** for that.

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
- **Data dir** — a quick size breakdown of the data dir (database, run
  snapshots, backups, workspaces, uploads) and free disk. The full storage
  report and cleanup actions live on the [Storage](./ui.md) page.
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

Change the admin password and toggle nightly auto-prune. Set a global
**Default subdomain** base domain here — every project then becomes reachable at
`<slug>.<base>` (a wildcard cert for the zone, if present, auto-enables HTTPS);
blank disables it. This page also
shows your **two-factor status** — whether 2FA is on and how many backup
codes you have left — and lets you **regenerate backup codes** or **change
the 2FA secret** (which makes you scan a new QR code). Operational knobs
(concurrency, timeouts, retention, pull policy…) are shown read-only — they
live in `/etc/ci-agent/config.toml`, the single source of truth; edit the
file and restart the service to change them.
