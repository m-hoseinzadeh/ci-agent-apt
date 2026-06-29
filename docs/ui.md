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

## Branding (server name & logo)

You can give this install its own **name** and **logo**. Both are optional.

- Set them during first-run setup (the **"Name your server"** step) or later on
  the **[Settings](./ui.md)** page under **Branding**.
- The name shows at the top of the sidebar and in the browser tab title; the
  logo replaces the default mark in the sidebar and is used as the browser
  **favicon** (and on the login page).
- A logo can be a **PNG, SVG, JPEG, WebP, GIF or ICO** file, up to **512 KB**.
- Leave the name blank to use the default **"CI Agent"**; tick **Remove current
  logo** in Settings to drop a logo and go back to the default mark.

> **What this means.** Branding is cosmetic and per-install (stored on the
> server, so everyone who logs in sees it). It does not change how anything
> works — it just makes one box easy to tell apart from another.

## Themes

A **theme picker** sits at the bottom of the sidebar. Click it to open a list of
**16 themes**, grouped into **Dark** (8) and **Light** (8), each with a small
colour swatch. Examples: *Dark · slate* (the default), *Ocean*, *Crimson*,
*Light*, *Rose*.

Pick one and the whole panel changes instantly. Your choice is remembered in the
browser (`localStorage`) and applied before the page draws, so there's no flash
on reload. It's purely cosmetic and per-browser — nothing on the server changes,
and other people who log in keep their own choice.

## Notes

A **Notes** tab on the right edge of every admin page (the docs site
excepted) slides out a scratchpad. Each note is a free-text box you edit in
place — it **autosaves** as you type, with no separate form or save button.
"**+ New note**" adds a row; the trash icon deletes one; a note left empty is
discarded automatically. Notes are stored server-side (in the database), so
they're the same in every browser you log in from. Only whether the drawer is
open or closed is remembered per-browser. Press `Esc` to close it.

## Dashboard `/`

The home page, refreshed every few seconds. It shows:

- a **System** card — gauges for CPU, memory and each disk, plus a meta-line
  with **Load (1m)** and how many runs are **queued**. If the server has an
  **NVIDIA GPU**, extra gauges show its utilization (with temperature) and
  video memory;
- two **last-10-minute sparklines** (CPU and memory) that link to the full
  **[Resources](./ui.md)** page;
- an **Active runs** table (what's running right now); and
- a **Recent runs** table (the latest deploy history).

## Resources `/metrics`

A detailed, live view of how busy the server is — open it from the sidebar or
by clicking a dashboard sparkline. It covers the **last 10 minutes** and shows:

- **CPU** — overall usage with a line chart, plus a small **per-core** grid.
- **GPU** — one card per **NVIDIA GPU** (only shown when present), with a
  utilization line chart, current temperature and video-memory usage. Detected
  through `nvidia-smi`; other vendors aren't shown yet.
- **Memory** — used / total, available, and swap usage.
- **Load average (1m)**.
- **Disks** — a usage meter for every filesystem.
- **Containers** — a table of every running container with **CPU**, **Memory**,
  **Mem %**, **Net I/O**, **Block I/O** and **PIDs**. Click a column heading to
  **sort** by it.
- **Host processes** — the top processes by memory (container processes
  excluded), with **PID**, **Process**, **CPU** and **Memory**. Also sortable.

## Projects `/projects`

The **Projects** list shows every project with a live **health badge** (see
[Operations](./operations.md) for what the badge means) and its categories. A
**category filter** at the top narrows the list to one category (your last
choice is remembered in the browser). Two buttons sit at the top:

- **New project** — opens the create form.
- **Browse app catalog** — install a ready-made app instead of writing compose
  by hand (see [Project conventions](./projects.md)).

### Creating a project

A project is: **name**, **slug** (lowercase, immutable after creation), one or
more optional **categories** (see below), a [deploy mode](./projects.md) and its
source (git URL + branch, a Dockerfile path, a prebuilt image, or pasted
compose), **HTTP port**, **env vars** (encrypted at rest; shown in **cleartext**
in the form so you can inspect and edit them — the admin is the sole operator of
their own box), an optional **working directory** (for monorepos) and a **run
timeout**. The button is **Create project**.

> **What is a category?** A category is a private Docker network. Type **one or
> more** categories (separated by commas — each becomes a removable chip); the
> project joins **every** network you list and can reach, and be reached by, any
> project that shares **at least one** of those networks by hostname. Projects
> with no category in common are isolated from each other. Leave it blank for
> the shared `default` category. Typing a brand-new category shows a small
> warning so you can catch typos. Changing the list **re-homes the running stack
> immediately** — no redeploy needed. See [Project conventions](./projects.md).

Private clones authenticate with a credential you register on the
**Git credentials** page (see below).

### The project page — four tabs

Open a project and you get four tabs:

- **Overview** — the **Health** card (a roll-up of the running stack's state),
  the project's **webhook URLs** with a *Rotate token* button (source modes
  only), a **Trigger a run** card, and a **Network** card showing the project's
  categories, the Docker networks it shares, and any sibling projects on those
  networks.
- **Domains & Ports** — the ports table (add extra ports, each with its own
  slug and per-port *Subdomain* / *Path* default-URL toggles), the domains for
  each port, and **per-port nginx proxy tuning** (see
  [Domains & TLS](./domains.md)).
- **Deployments** — the **Run history** table and the **Containers** card.
- **Settings** — the edit form (name, categories, git, branch, mode, paths,
  ports, run timeout, env vars), a **Resource limits** card (per-service CPU /
  memory caps, plus **GPU** access when the host has one) and a **Volumes** card
  on Image / Dockerfile projects (see below), **Archive / Unarchive**, and the
  **Danger zone** (delete).

A fifth **Actions** tab appears only when the project has custom actions defined
(see below). The panel remembers which tab you last used per project.

### What you can do on the project page

- **Trigger a run** (Overview) — run from the configured repo (with an optional
  URL/branch override), from a ZIP URL, or by uploading a ZIP file. Sourceless
  projects (Image / Inline compose) show a **Deploy now** button instead.
- **Run history** (Deployments) — each row shows the **run number** (counted per
  project from `#1`), status, commit message + committer, start time and
  duration. A failed run expands to show its error.
- **Redeploy** (Deployments) — re-run a past **successful** deploy with no fetch
  or build. It first **pulls the latest image** for each registry-backed service
  (on an air-gapped box a failed pull is ignored), then brings the stack up from
  that run's saved snapshot. The button is greyed out once that run's images are
  pruned.
- **Recreate** (Deployments → Containers card) — rebuild the containers from the
  last successful run's snapshot using the project's **current env vars and
  config**. Use it to apply env, volume, resource-limit or GPU changes; a plain
  **Restart** keeps the old values. (The agent also recreates automatically when
  you save changed env vars or volumes.)
- **Resource limits** (Settings) — one row per discovered service. Cap a
  service's **CPU** (fractional cores, `0.5` = half a core) and **memory** (in
  MB) so one container can't starve the box; leave a field blank for no cap.
  When the host has an **NVIDIA GPU**, each row also gets a **GPU** checkbox that
  grants that service the GPU (the `--gpus all` equivalent, written as
  `deploy.resources.reservations.devices`). GPU access additionally requires the
  **NVIDIA Container Toolkit** on the host (`nvidia-ctk runtime configure
  --runtime=docker`); ticking the box reveals the exact setup commands. All of
  these take effect on the **next deploy** (or a **Redeploy** / **Recreate** for
  Image / Dockerfile projects).
- **Volumes** (Settings, Image / Dockerfile modes) — attach storage to the
  container. Pick a type — a **Named volume** (Docker-managed, survives
  redeploys) or a **Host path** (a bind mount to an absolute path on the server)
  — then give the **source** (volume name or `/host/path`) and the **container
  path**, and optionally mark it **read-only**. Host paths under system
  directories (`/etc`, `/usr`, `/var/lib/docker`, …) are refused for safety; use
  a location like `/opt`, `/srv` or `/mnt`. Saving a volume recreates the stack.
- **Containers** (Deployments) — a table of the deployed stack's containers
  with per-container **Start / Stop / Restart / Pause** buttons. These act on
  the live containers, not on a redeploy.
- **Custom actions** `/projects/<slug>/actions` — operator-defined operations
  that run inside a chosen container: a plain **run** command, a **backup**, or
  an **upload-and-restore** (upload a file, then run a load command with
  `{{FILE}}` substituted). Define them as a small JSON array on the Actions
  page; once any exist, the **Actions** tab appears. Handy for database
  dumps/restores.
- **nginx** page `/projects/<slug>/nginx` — live status, the generated server
  block with a **drift badge**, and **one-click apply** (write → `nginx -t` →
  reload) or the equivalent copy-paste commands.
- **Container logs** `/projects/<slug>/logs` — live `docker compose logs` for
  the deployed stack, with a service picker, word-wrap toggle, and a clear
  button (see below).
- **Archive / Unarchive** (Settings) — archived projects reject webhooks.
- **Delete** (Settings → Danger zone) — a hard delete that removes the project
  and **all** its Docker resources (containers, images, networks), run history,
  workspace and nginx config. Type the slug to confirm; blocked while a run is
  in flight.

## Run detail `/projects/<slug>/runs/<number>`

Status timeline, source/commit/image metadata, and the **live log**: the full
run log replays instantly, then streams in real time while the run is active. A
`Cancel` button TERM→KILLs the run's process tree. The log viewer has the same
controls described under [Container logs](./ui.md) below.

## Container logs `/projects/<slug>/logs`

Live output of the project's **currently deployed** containers (distinct from
run logs, which capture a build/deploy). Pick a single service or stream all of
them. Both this page and the run log use the same viewer:

- **Colour by level** — lines that look like errors, warnings, info or debug are
  tinted, and any **ANSI colours** the program prints are shown too.
- **Level filter** — chips for **Error / Warn / Info / Debug / Other**, each with
  a count. Click one to hide or show that level.
- **Search** — type to highlight matches; the **↑ / ↓** buttons (or
  Enter / Shift+Enter) jump between them, with a `match / total` counter.
- **Wrap** — toggle word-wrap for long lines.
- **Clear** — empty the view to watch only new output.
- **↓ Jump to latest** — appears when you scroll up; the view otherwise sticks
  to the newest line on its own.

Very large logs stay fast — the viewer only draws the lines on screen and, once
a log gets huge, drops the oldest lines (shown as a "*… earlier lines dropped*"
marker at the top).

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

The server shell is **sandboxed**: it runs as the agent's own (passwordless)
user with the host filesystem mounted **read-only** apart from the agent's data
dirs, so `sudo` can't authenticate and host files can't be edited from here. Use
it for quick read-only checks and Docker commands (which need no `sudo`) — not
general host administration.

**Open host SSH session** — for privileged or filesystem-mutating work, the
server shell's **Snippets** drawer has an **Open host SSH session** button at
the top. It drops an `ssh <admin>@localhost` command at the prompt (the admin
account is auto-detected) that logs you into a **real host session** via `sshd`,
outside the sandbox — writable filesystem and full `sudo`. You authenticate with
your own login password, so the privilege stays tied to a real account rather
than the service. (First use trusts localhost's host key automatically, so you
land straight on the password prompt. A *Permission denied* means a wrong
password, or that this box accepts **SSH keys only** — enable
`PasswordAuthentication` in `sshd`, or add a key for the agent account.) When
`tmux` is installed the session is wrapped in a
persistent `tmux` session, so a browser refresh, network drop, or interrupt only
**detaches** it — your work keeps running and clicking the button again
re-attaches.

> **Important:** although the shell itself is read-only, the agent's Docker
> access is root-equivalent on the host (e.g. `docker run --privileged`). Treat
> the server shell's reach as root SSH access regardless.

**Copy a file into a container** — below the container shell, a small form
pushes a file into the selected container at a destination path (blank =
`/tmp/`). The source is either a **file upload** from your browser or a **path
on the host** (a file or directory the agent can read). Useful for dropping a
config or a restore artifact next to a running app.

**Snippets** — a **Snippets** tab on the right edge of the shell pages keeps a
list of reusable commands (an optional label plus the command), edited in place
with autosave like [Notes](./ui.md). Hover or focus a snippet and hit
**Paste** to drop its command into the live shell — it lands at the prompt
**ready to run** rather than executing on its own, so you can review or tweak it
first. Snippets are stored server-side and shared across both the server shell
and every container shell.

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

- **nginx configs** — an inventory of every `.conf` file in
  `/etc/nginx/conf.d/`, with a status badge on each:
  - **ours** — a project's vhost or a ci-agent-managed file (the admin panel,
    the global snippet). Project rows link to that project's nginx page.
  - **default** — nginx's stock site (`default.conf`), left untouched.
  - **unmanaged** — a file not tied to any current project (a leftover from a
    deleted project, or one ci-agent never wrote).

  Each row shows the file name, owner, the hostnames it serves, size and last
  modified. **Unmanaged** files have a **View** button to read the file and a
  **Delete config** button to remove it; managed and default files can't be
  deleted here, so you can't lock yourself out by accident.

## Firewall `/firewall`

A simple front end for the host's **UFW** firewall — no command line needed.

- **Status** — shows whether the firewall is **active** or **inactive**, with an
  **Enable** or **Disable** button. Turning it on first allows port **22 (SSH)**
  and the panel's own port, so you don't lock yourself out. A **Reload** button
  re-reads the rules.
- **Rules** — a numbered list of the current rules. Each row has a **Delete**
  button.
- **Add rule** — pick an **action** (Allow / Deny / Reject), a **protocol**
  (TCP / UDP / Any), a **port** (e.g. `80`, or a range like `1000:2000`), and an
  optional **source** (e.g. `10.0.0.0/8`; blank means anywhere). Click **Add
  rule**.

> **What this means.** UFW (Uncomplicated Firewall) decides which network
> connections the server accepts. "Allow port 80" lets web traffic in; a "Deny"
> rule blocks it. Rule numbers shift after each delete, so the list is re-read
> from the firewall every time the page loads.

> **Common problems and fixes.** *Locked out after enabling?* The panel always
> allows SSH (22) and its own port first, so you should stay connected — reach
> the box over SSH and run `sudo ufw status` to check. *Rule didn't apply?* The
> agent needs its firewall sudo rule; on a package install this is set up for
> you (see [Security](./security.md)).

## Storage `/storage`

A read-only storage report on top, then disk cleanup actions.

**Report (read-only):**

- per-filesystem usage meters
- a composition estimate of the disk holding the data dir
  (CI Agent data / Docker / other / free) — “other / system” is whatever the
  unprivileged agent can't attribute (OS packages, apt cache, system logs)
- a breakdown of the agent's own data dir (database, run snapshots, backups,
  workspaces, uploads)
- an interactive **Disk explorer** donut — click a slice to drill into a folder
  and see what's using space; use the breadcrumb to climb back out. At the top
  of a filesystem the chart fills the whole disk (used folders + free space +
  any space the agent can't read)

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
- **CI Agent update** — three ways to update from this page:
  - *One-click online (connected server)* — **Download & stage latest** fetches
    the newest signed release straight from the public apt repository, verifies
    it (see below), and stages it — no file handling. When **Check for updates**
    finds a newer version, the same action also appears as an **Update now**
    button on the CI Agent row of the results table. On an air-gapped server this
    simply reports the repo unreachable.
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

## Git credentials `/git-credentials`

Store login details for cloning **private repositories**, once per git host —
not per project. There are two shapes:

- **GitHub** — set **Host** to `github.com`, the repo **Owner** (the account or
  org that owns the repo), and a **personal access token** with `repo` read
  access as the secret. When a project clones
  `https://github.com/<owner>/…`, the matching credential authenticates it
  automatically.
- **Other git hosts** — set **Host** to your server (e.g. `git.example.lan`),
  the account **Username**, and its **password or token** as the secret.

Fill the form (**Host**, optional **Owner**, optional **Username**, **Secret**)
and click **Save credential**. Secrets are **encrypted at rest** with
`secret_key_file` and **never shown again** — the table shows only "••••• set".
Delete a credential to revoke access.

> **What this means.** A *credential* is just a saved username + secret for one
> git host. You add it here once, and every project that clones from that host
> uses it — you never paste tokens into individual projects.

## Settings `/settings`

The Settings page collects everything you change from the UI:

- **Branding** — set this server's **name** (shown in the sidebar and the
  browser tab) and an optional **logo** (used as the sidebar mark and the
  favicon). Upload a PNG / SVG / JPEG / WebP / GIF / ICO up to 512 KB, or tick
  **Remove current logo** to go back to the default mark; click **Save**. See
  [Branding](./ui.md) above.
- **Change password** — current password, new password (8+ characters),
  confirm; click **Update password**. Changing it **signs out every other
  session** (you stay logged in on this browser).
- **Two-factor authentication** — shows whether 2FA is on and how many backup
  codes you have left. You can **Regenerate backup codes** or **Set up a new
  device** (this shows a fresh QR code to scan). Both ask for your password
  first.
- **Maintenance** — a checkbox to turn on **nightly auto-prune** of dangling
  images and build cache.
- **Default subdomain** — set a global **base domain** here; each project port
  can then be reached at `<slug>.<base>` (a subdomain, on by default) and/or
  `<base>/<slug>/` (an apex path, off by default), toggled per port on the
  project's **Domains & Ports** tab. An optional **base path** here nests every
  path route under a shared prefix (e.g. `/p` → `<base>/p/<slug>/`). A wildcard
  cert for the zone, if present, auto-enables HTTPS for both. Blank disables them.
  Setting the base domain to the **same hostname as the admin panel** turns on
  single-domain mode — the panel stays at `<domain>/` and projects sit under the
  base path (which is then required). See [Domains & TLS](./domains.md).
- **Failed-deployment email alerts** — send an email whenever a deploy fails.
  Fill in the **admin email**, **SMTP host** and **port**, the **security** mode
  (*STARTTLS (587)*, *Implicit TLS (465)*, or *None (25)*), optional **username**
  and **password**, and a **From** address. Click **Save**, or **Save & send
  test** to fire a test email right away. When a real deploy fails, the agent
  emails both the admin and the commit's committer. See
  [Operations](./operations.md).
- **Effective configuration** — the operational knobs (concurrency, timeouts,
  retention, pull policy…) shown **read-only**. They live in
  `/etc/ci-agent/config.toml`, the single source of truth; edit the file and
  restart the service to change them.

> **What this means.** *SMTP* is the standard way programs send email. You point
> the agent at a mail server (your own internal relay, or any SMTP server it can
> reach) and it sends through that. On an air-gapped network, use an internal
> relay; choose **None (25)** only on a trusted LAN.
