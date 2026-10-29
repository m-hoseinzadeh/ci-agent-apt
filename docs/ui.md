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

You can enrol **more than one device**, and a code from any of them signs you
in. Add and remove them on the **Settings** page.

> **Lost your phone?** Use a backup code on the code-entry screen, then remove
> the lost device from **Settings** and add a new one. No codes left either?
> Run `ci-agent reset-2fa` on the server to clear 2FA and enroll again. See
> [Security](./security.md) for the full 2FA model.

## Branding (server name & logo)

You can give this install its own **name** and **logo**. Both are optional.

- Set them during first-run setup (the **"Name your server"** step) or later on
  the **Settings** page under **Branding** (see *Settings* below).
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
colour swatch. Examples: *Dark · slate* (the default), *Ocean · azure*,
*Crimson · red*, *Light · gray*, *Rose · blush*.

Pick one and the whole panel changes instantly. The choice is **one setting for
the whole server**, saved in the agent's database: it follows you to every
browser and device you log in from, it is used on the login and 2FA pages too,
and it is included in backups. It is applied before the page draws, so there's
no flash on reload. It only changes colours — nothing else.

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
  **Resources** page (see *Resources* below);
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

The **Projects** page lists every project. Each row shows the project's
**Name**, **Slug**, **Category**, **Endpoint**, a live **Health** badge (see
[Operations](./operations.md) for what it means) and its **Last run**. Each row
has two buttons: **Open** and **Copy** (see *Copying a project* below).

- The **Category** filter above the table shows only one category. Your browser
  remembers your last choice.
- **Browse app catalog** (top of the page) installs a ready-made app, so you
  don't have to write compose by hand (see [Project conventions](./projects.md)).
- If two projects in the same category use the same service name, a yellow
  **Shared-network service-name collisions.** banner lists them. Rename one of
  the services, or move one project to another category.

### Creating a project

The **New project** form is the card **below** the project list.

1. Type a **Name**. The **Slug** fills in by itself. You can change the slug
   now, but never after the project is created. Use lowercase letters, numbers
   and dashes.
2. Optional: add one or more **Category** names (see the note below).
3. Pick **Build from**: *Docker Compose*, *Dockerfile*, *Prebuilt image* or
   *Inline compose* (see [deploy modes](./projects.md)).
4. Fill in the fields for that mode: **Git URL** and **Branch**, **Working
   directory** (for monorepos), **Compose path**, **Dockerfile path**, **Image**
   or **Compose file**.
5. Optional: set the **HTTP port**. Leave it blank and a free port is picked for
   you. In Dockerfile and Prebuilt image mode, **Container port** is the port the
   app listens on inside the container (blank = same as the HTTP port).
6. Click **Create project**.

Env vars, the run timeout, volumes and domains are not on this form. You set
them on the project page after it is created. Env vars are encrypted on disk but
shown in **cleartext** in the form, so you can read and edit them.

> **What is a category?** A category is a private Docker network. Type **one or
> more** categories (separated by commas — each becomes a removable chip); the
> project joins **every** network you list and can reach, and be reached by, any
> project that shares **at least one** of those networks by hostname. Projects
> with no category in common are isolated from each other. Leave it blank for
> the shared `default` category. Typing a brand-new category shows a small
> warning so you can catch typos. Changing the list **re-homes the running stack
> immediately** — no redeploy needed. See [Project conventions](./projects.md).

Private clones authenticate with a credential you register on the
**Git credentials** page (see *Git credentials* below).

### Copying a project

You can duplicate a project with its settings.

1. Click **Copy** on the project's row in the list, or in the project page
   header.
2. In the **Copy project** dialog, type **Copy to name** and **New slug**. The
   slug must be unique.
3. Click **Copy project**.

The copy gets the same env vars, volumes, service limits and proxy settings.
Its host ports are changed to free ports, so it doesn't clash with the original.
Domains and run history are **not** copied, and the copy gets its own new
webhook token.

![screenshot: the Copy project dialog](TODO)

### The project page

The header shows the project name with an **active** or **archived** chip and
when it was created. Buttons on the right:

- **Logs** — live container logs (see *Container logs* below).
- **Shell** — a terminal inside a container (see *Terminals* below).
- **Files** — browse files inside a container (see *Files* below).
- **Actions** — only shown when the project has custom actions (see below).
- **nginx guide** — the project's nginx page.
- **Copy** — duplicate the project (see *Copying a project* above).

If another project uses the same port, a banner says **Port … is also claimed
by: …**.

Below the header are four tabs. Each tab has its own address, so you can
bookmark it:

| Tab | Address | What's on it |
|---|---|---|
| **Overview** | `/projects/<slug>` | **Internal access**, **Health**, **Network** |
| **Domains** | `/projects/<slug>/routing` | the **Domains** card: domains per port, default URLs, **Proxy tuning** |
| **Deployments** | `/projects/<slug>/deploy` | **Webhooks**, **Trigger a run**, **Run history**, **Containers** |
| **Settings** | `/projects/<slug>/settings` | the **Settings** card, **Ports**, **Resource limits**, **Volumes**, **Danger zone** |

**Overview tab:**

- **Internal access** — the hostname(s) other projects on a shared Docker
  network use to reach this app (for example as `DB_HOST`), plus the list of the
  app's domains. If the running containers have old names, a warning asks you to
  **Recreate** so the hostnames work.
- **Health** — a roll-up of the running stack's state. The agent also checks this
  in the background and can email you when a project goes down (see
  [Operations](./operations.md)).
- **Network** — the project's categories, the Docker networks it shares, and
  every other project you can reach from here, with its hostname.

**Settings tab — the Settings card** holds several small forms, each with its
own button:

- the edit form (**Name**, **Category**, **Git URL**, **Branch**, **Working
  directory**, **Build from**, the mode's path or image, **Compose file**, **Run
  timeout (s)**, **Env vars**) — click **Save**;
- **Auto-redeploy every (hours)** — click **Save schedule**;
- **Watch paths** — click **Save watch paths**;
- **Status callback URL** and **Callback token** — click **Save callback**;
- **Pause deploys** / **Resume deploys**;
- **Archive** / **Unarchive**.

### Domains tab

![screenshot: the Domains tab with the primary radio, HTTPS chips and Maintenance checkbox](TODO)

The **Domains** card lists the hostnames that serve this app. When the project
has more than one port, there is one table per port. Every change is written to
nginx and reloaded right away.

- **Add domain** — type a hostname (and pick a port, if there are several), then
  click **Add**.
- **Primary** — pick one domain with the radio button. The other domains on that
  port then redirect (301) to it. Pick **None — every domain serves the app
  directly** to turn the redirect off.
- **Rename** — edit the hostname in its box and click **Rename**.
- **Port** (only with several ports) — move the domain to another port.
- **HTTPS** — a chip shows the certificate state:
  - **HTTPS** — HTTPS is on.
  - **covered** — a wildcard certificate already covers this host.
  - **ready** — a certificate exists; turn HTTPS on from the SSL page.
  - **renew** — the certificate is close to expiry.

  **Add SSL** (no certificate yet) or **Manage** opens the domain's SSL page
  (see *SSL for one domain* below).
- **Maintenance** — tick the checkbox to show a "Down for maintenance" page on
  this domain instead of the app. An **offline** chip appears. The container
  keeps running. Untick it to bring the app back.
- **×** — delete the domain (you are asked to confirm).

Under each port you may also see:

- **Default URL** — only when a base domain is set in **Settings → Default
  subdomain**. Turn it on or off in the **Ports** card on the **Settings** tab.
- **Proxy tuning** — click to open. Set **Max upload (MB)** (default 256),
  **Proxy timeout (s)** (default 300), **WebSockets** (*Forward WebSocket / SSE
  upgrades*) and **Extra directives**, then click **Save proxy settings**. It
  applies to every domain on that port. See [Domains & TLS](./domains.md).

> **What this means.** The *primary* domain is the "real" address. Visitors who
> type another of the app's domains are sent there, so search engines and
> cookies see one name.

### What you can do on the project page

- **Trigger a run** (Deployments) — **Run from git** (with optional **Git URL**
  / **Branch** overrides), **Run from zip URL**, or **Upload & run** a ZIP file.
  Projects with no source (Prebuilt image / Inline compose) show a **Deploy now**
  button instead.
- **Webhooks** (Deployments) — the project's **Git** and **Zip** webhook URLs,
  and a **Rotate token** button. See [Webhooks](./webhooks.md).
- **Run history** (Deployments) — each row shows the **run number** (counted per
  project from `#1`) and its trigger, the status, the commit message + short hash
  + committer, the start time and the duration. A failed run shows its error
  under the row. Pick how many rows to **Show**: **5 / 10 / 50 / All** (default
  **5**), and page with **Prev** / **Next**.
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
- **Containers** (Deployments) — a table of the deployed stack's containers.
  The buttons depend on the state: a running container has **Restart**,
  **Pause** and **Stop**; a paused one has **Unpause** and **Stop**; a stopped
  one has **Start** and **Restart**. These act on the live containers, not on a
  redeploy.
- **Ports** (Settings) — every port the app publishes, in one table: **Host**
  (the port published to the outside), **Container** (the port inside; blank =
  same as host), **Service** (which compose service publishes it; blank = the
  single `app` service) and **Slug & label**. The **primary** port is the first
  row. Edit a row and click **Save**; **×** removes an extra port (its domains
  move back to the primary port). Use the **Add port** row and **Add** for more
  ports. When a base domain is set, the **Default URL** column has a
  **Subdomain** and a **Path** checkbox per port. See
  [Domains & TLS](./domains.md).
- **Resource limits** (Settings) — one row per discovered service. Cap a
  service's **CPU** (fractional cores, `0.5` = half a core) and **memory** (in
  MB) so one container can't starve the box; leave a field blank for no cap.
  When the host has an **NVIDIA GPU**, each row also gets a **GPU** checkbox that
  grants that service the GPU (the `--gpus all` equivalent, written as
  `deploy.resources.reservations.devices`). GPU access additionally requires the
  **NVIDIA Container Toolkit** on the host (`nvidia-ctk runtime configure
  --runtime=docker`). ci-agent pre-checks for Docker's `nvidia` runtime: when
  it's missing the card shows a **"GPU toolkit not configured"** warning with the
  exact setup commands, and a deploy that requests the GPU **fails early with the
  same guidance** instead of the opaque `could not select device driver "nvidia"`
  error. All of these take effect on the **next deploy** (or a **Redeploy** /
  **Recreate**).
- **Volumes** (Settings, every mode) — attach storage to a container. Pick a
  type — a **Named volume** (Docker-managed, survives redeploys) or a **Host
  path** (a bind mount to an absolute path on the server) — then give the
  **source** (volume name or `/host/path`), the **container path**, and for
  Compose / Inline compose the **service** (blank = the single `app` service).
  Optionally tick **read-only**, then click **Add**. Host paths under system
  directories (`/etc`, `/usr`, `/home`, `/tmp`, `/var/lib/docker`, …) are
  refused for safety; use a location like `/opt`, `/srv` or `/mnt`. Adding or
  removing a volume recreates the stack.
- **Custom actions** `/projects/<slug>/actions` — operator-defined operations
  that run inside a chosen container: a plain **run** command, a **backup**, or
  an **upload-and-restore** (upload a file, then run a load command with
  `{{FILE}}` substituted). Handy for database dumps and restores. Apps from the
  catalog can bring their own actions. To add your own, open
  `/projects/<slug>/actions`, write them as a small JSON array in **Edit
  actions** and click **Save actions**. The **Actions** header button only
  appears once at least one action exists.
- **nginx guide** `/projects/<slug>/nginx` — live status, the generated server
  block with a **drift badge**, and **Apply config & reload** (write → `nginx
  -t` → reload) or the equivalent copy-paste commands.
- **Container logs** `/projects/<slug>/logs` — live `docker compose logs` for
  the deployed stack (see *Container logs* below).
- **Edit compose file** (Settings → **Compose file**) — in **Compose** mode,
  an optional box to paste an **inline override** of the repo's compose file;
  when set it is deployed instead of the committed file (the repo is still
  cloned, so build contexts resolve). **Load from last commit** fills it with
  the file as it is in the repo; **Load from last deploy** fills it with the last
  successful deploy's file. Leave blank to deploy the committed file as-is. In
  **Inline compose** mode this box *is* the whole compose file. See
  [Projects](./projects.md).
- **Deploy controls** (Settings) — **Pause deploys** stops webhook-triggered
  deploys (a **deploys paused** chip shows) while leaving the running stack up;
  manual deploys and redeploys still work, and **Resume deploys** re-enables
  webhooks. **Auto-redeploy every (hours)** (0–168, `0` = off) re-runs the last
  successful deploy on that interval; click **Save schedule**. See
  [Operations](./operations.md).
- **Watch paths** (Settings) — one repository path per line. A git push then
  deploys this project only when it changed a file under one of them, so
  several projects can build from one repository. Blank = every push deploys.
  Deploys you start from the panel or the API always run. See
  [Webhooks](./webhooks.md).
- **Status callback** (Settings) — **Status callback URL** and **Callback
  token**, saved with **Save callback**. The agent POSTs each status change of
  runs started by a webhook or the API to this URL, with the token in the
  `X-CiAgent-Token` header. Runs you start from the panel are not reported.
  Blank URL = off. See [Webhooks → Status callback](./webhooks.md).
- **Archive / Unarchive** (Settings) — archived projects reject webhooks.
- **Remove project** (Settings → **Danger zone**) — a hard delete. It removes the
  project and **everything** it created: containers, built images, **named
  volumes (their data is destroyed)**, the project network, run history, build
  files and nginx config. Type the slug in **Type `<slug>` to confirm**; the
  button only works once the slug matches. It is blocked while a deploy is
  running.

## SSL for one domain `/projects/<slug>/domains/<id>/ssl`

Click **Add SSL** or **Manage** next to a domain on the **Domains** tab. The page
title is **SSL · *your-domain***.

- **Certificate** — the current status, source, what it covers and when it
  expires. **Serve HTTPS** turns HTTPS on (HTTP then redirects to HTTPS).
  **Renew now** renews it.
- **Issue a certificate** — three tabs:
  - **Let's Encrypt** — **Issue via HTTP-01** (one host, renews automatically)
    or **Start wildcard order** (DNS-01). Pick a DNS provider in **TXT
    records** and it renews automatically; pick **Manual** and you renew it by
    hand.
  - **Internal / Self-signed** — made on this server, fully offline. Pick a
    **Source** and **Wildcard** or **Dedicated**, then **Issue & enable HTTPS**.
  - **Upload** — paste a certificate and key you got elsewhere, then **Upload &
    enable HTTPS**.

Full details: [Domains & TLS](./domains.md).

![screenshot: the SSL page for one domain](TODO)

## Certificates `/certs`

All certificates on this server in one place. Open it from the **Certificates**
link in **Settings → Default subdomain** or on a domain's SSL page.

- **Internal CA** — click **Download root CA (root.crt)** and install it once
  on your computers; then its certificates are trusted without warnings. The CA
  is created the first time you issue an Internal CA certificate.
- **Managed certificates** — every certificate, with what it **Covers**, its
  **Source** and when it **Expires**. **Renew** or **×** (delete).
- **Let's Encrypt** — your **Account email**, a staging switch, **Save Let's
  Encrypt settings**, and **Start wildcard order** for a whole zone.
- **DNS providers** — Cloudflare or ArvanCloud API tokens. With one, a wildcard
  order adds its DNS records itself and renews automatically. **Save & test**,
  **Test**, or **×** (delete).
- **Add an offline certificate** — make or upload one without internet access,
  then **Create certificate**.

Full details: [Domains & TLS](./domains.md).

![screenshot: the Certificates page](TODO)

## Offline certs `/offline-certs`

Get a trusted Let's Encrypt certificate for a server that has **no internet**.

1. On a server **with** internet, open **Offline certs**. Under **Issue &
   export (online server)**, enter the **Domain**, pick **Wildcard** or
   **Dedicated**, and click **Start order**.
2. With a DNS provider picked in **TXT records**, ci-agent does the DNS part
   itself. With **Manual**, publish the DNS TXT record it shows, then click
   **Validate & issue**.
3. **Download** the bundle from the **Offline certificates** table.
4. On the offline server, open **Offline certs**. Under **Import a bundle
   (offline server)**, choose the **Bundle file** and click **Import bundle**.

A certificate issued through a DNS provider renews by itself on the online
server; for a manual one, **Renew** in the table starts a fresh DNS order.
Either way, download and import the new bundle again.
Full details: [Domains & TLS](./domains.md).

![screenshot: the Offline certificates page](TODO)

## Run detail `/projects/<slug>/runs/<number>`

Open a run from **Run history**. The page shows a **Timeline** (queued,
started, finished, duration), a **Details** card, and the **live log**: the full
run log replays instantly, then streams in real time while the run is active.
While the run is active, a **Cancel** button stops it (and force-kills it if it
doesn't stop). The log viewer has the same controls described under *Container
logs* below.

The **Details** card shows the trigger, source, commit, message, committer,
image tags and any error. It also shows:

- **Run ID** — the run's global id. This is the `run_id` that webhooks and the
  API return, and that status callbacks send. It is different from the
  per-project `#number` in the page title.
- **Status callback** — only on runs started by a webhook or the API. It shows
  how the final report to the project's callback URL went: `pending`,
  `delivered (HTTP 200)`, `failed (…)` with the HTTP code or connection error,
  `skipped (no status callback URL set)`, or `not sent`. While the run is
  active it reads `sending progress updates`. See
  [Webhooks → Status callback](./webhooks.md).

![screenshot: run detail page with the Run ID and Status callback rows](TODO)

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
- **Copy** — copy your selection, or the whole log if nothing is selected.
- **Download** — save the full log as a file.
- **Fullscreen** — the expand icon fills the screen with the log.
- **Wrap** — toggle word-wrap for long lines.
- **Clear** — empty the view to watch only new output.
- **↓ Jump to latest** — appears when you scroll up; the view otherwise sticks
  to the newest line on its own.

Very large logs stay fast — the viewer only draws the lines on screen and, once
a log gets huge, drops the oldest lines (shown as a "— *N* earlier lines
dropped —" marker at the top).

## Terminals

A **terminal** is an interactive command-line session you drive from the
browser — the same as typing in a real shell on the server.

- **Server shell** `/terminal` — a shell on the **host** itself (the machine
  running the agent). Use it for quick checks and fixes without a separate
  SSH login.
- **Container shell** `/projects/<slug>/terminal` (the project's **Shell**
  button) — a shell **inside** one of a project's running containers (like
  `docker exec`). Pick it from the **Container** list, then run commands inside
  it.

Both are full terminals: colours, interactive programs and window-resize all
work. If the connection drops, click **Reconnect**.

> **Important: this is root-level access.** The server shell runs as the
> agent's own user (`ci-agent`). Plain `sudo` asks for a password, and that
> account has none, so most `sudo` commands fail. But the account can still do
> root-level things:
>
> - it is in the `docker` group, and Docker access is root-equivalent (for
>   example `docker run --privileged`);
> - it may run a few commands as root **without a password**: nginx test and
>   reload, service controls, firewall commands, and the `ci-agent` program
>   itself, which includes a helper that can read, write and delete any file.
>
> Treat the server shell like a root SSH login. Only give the admin password
> and 2FA to people you would give root to.

**Open host SSH session** — for normal admin work as **your own** account, the
server shell's **Snippets** drawer has an **Open host SSH session** button at
the top. It drops an `ssh <admin>@localhost` command at the prompt (the admin
account is auto-detected). This logs you into a **real host session** via
`sshd`, with full `sudo`. You type your own login password, so the work is tied
to a real person's account rather than the service. (First use trusts
localhost's host key automatically, so you land straight on the password
prompt.) When `tmux` is installed, the session is wrapped in a persistent `tmux`
session, so a browser refresh, network drop, or interrupt only **detaches** it —
your work keeps running and clicking the button again re-attaches.

**Copy a file into a container** — the **Copy file to container** card below
the container shell pushes a file into a container.

1. Pick the **Container**.
2. Pick the **Source**: **File upload** (choose a **File** from your computer)
   or **Path on host** (type a **Host path** — a file or folder on the server).
3. Type the **Destination** path inside the container (blank = `/tmp/`).
4. Click **Copy into container**.

Useful for dropping a config file or a restore file next to a running app.

**Snippets** — a **Snippets** tab on the right edge of the shell pages keeps a
list of reusable commands (an optional label plus the command). Click **+ New
snippet** to add one; it saves as you type, like *Notes* above. Hover or focus a
snippet, then:

- **Paste** — puts the command at the prompt **without running it**, so you can
  check or change it first;
- **Run** — puts the command at the prompt and presses Enter.

Snippets are stored on the server and shared between the server shell and every
container shell. The drawer also has a **Handy commands** list of common
commands; click one to put it at the prompt (it won't run by itself).

| Problem | Fix |
|---|---|
| `sudo` asks for a password and fails | The agent account has no password. Use **Open host SSH session** and log in as your own admin account. |
| *Permission denied* after **Open host SSH session** | Wrong password, or the server only allows SSH keys. Enable `PasswordAuthentication` in `sshd`, or add a key for the agent account. |
| The container shell says no containers were found | Deploy the project first. A container must be running to open a shell in it. |

## Files

A point-and-click file manager — the same operations as the shell, without the
typing. Two surfaces share one UI:

- **Host files** `/files` (sidebar **Files**) — browse the **host
  filesystem**. It works as the agent's user first. When it hits a root-owned
  file or folder, it automatically retries through the agent's **root** helper.
  So it can read, edit and delete **any file on the server**. This is a
  root-level tool — use it with the same care as a root shell.
- **Container files** `/projects/<slug>/files` (the project's **Files**
  button) — browse **inside** a container of the project's stack, picked from
  the **Container** dropdown. Browsing needs a shell in the image, so distroless
  / `scratch` containers show no listing.

In either mode you can:

- move around with **↑ Up**, the path box and **Go**, or the breadcrumb;
- **download** a file to your browser;
- **Upload** a file into the current folder (a progress bar shows, with a ✕ to
  cancel);
- **edit** a small text file inline, then **Save** or **Cancel** (files up to
  1 MB; move bigger files with the shell);
- make a **New folder**, rename, or **delete** entries.

Delete asks you to confirm first, and deletes folders **with everything inside
them** — double-check the path. Changes are recorded in the audit log (see
*Audit* below).

## Host `/host`

The **Host controls** page puts host-level settings in one place:

- **Services** — status plus restart / reload / log-tail buttons for `nginx`,
  the `docker` engine, and `ci-agent` itself. Restarting Docker bounces running
  containers; restarting ci-agent reloads this panel (it reconnects after a few
  seconds). "Logs" tails the last 200 `journalctl` lines for the unit.
- **Admin panel nginx** — tunes the vhost that serves this panel
  (`/etc/nginx/conf.d/ci-agent.conf`). The knobs (max upload, proxy timeout,
  extra `location` directives) keep the core proxy generated so you can't lock
  yourself out. An **Advanced** section can replace the whole config verbatim —
  `nginx -t` catches syntax errors and rolls back, but a wrong `proxy_pass`
  still passes the test and would cut off the panel, so a **Reset to generated
  default** button and the SSH / loopback fallback are the recovery path.
- **Global nginx config** — an http-level snippet included before every site
  (`gzip`, `server_tokens off;`, a shared `limit_req_zone`, …). Server blocks
  belong to projects, not here.

  Both nginx editors apply through the same safe path as the project nginx page
  (write → `nginx -t` → reload, rolled back on a failed test).
- **Direct port access** — an emergency switch to reach this panel straight by
  IP address and port, for when your domain or DNS is down. Normally the panel
  listens on loopback only and you reach it through nginx at your domain. Click
  **Enable — open port 8044** and the panel binds all interfaces
  (`0.0.0.0:8044`), opens that port in the firewall, restarts, and then also
  answers at `http://<server-ip>:8044`. The card shows an **Enabled** chip and
  the current `listen` address while it is on. Click **Disable — lock to
  loopback** to return to loopback only and close the firewall port. Either
  action restarts the panel, so **you are logged out** and log in again after a
  few seconds. Access stays protected by your password and two-factor code the
  whole time. The `ci-agent panel-port` command does the same from the server
  shell — see [Configuration → CLI](./configuration.md). While the port is
  open, each login shows a **lock to loopback** reminder you can skip.
- **Admin domain** — shows the domain nginx serves this panel on (with a
  **verified** chip once it has been proven through nginx). **Remove domain —
  re-open port 8044** is the permanent way back to direct IP:port access:
  unlike Direct port access it clears the stored domain too, so nothing asks
  you to lock down to it again — instead each login shows a skippable reminder
  to bind a domain. The panel restarts (you are logged out), the firewall port
  opens, and the nginx site file is left in place for you to delete. The
  `ci-agent remove-domain` command does the same from the server shell.
- **nginx configs** — an inventory of every `.conf` file in
  `/etc/nginx/conf.d/`, with a status badge on each:
  - **ours** — a project's vhost or a ci-agent-managed file (the admin panel,
    the global snippet). Project rows link to that project's nginx page.
  - **default** — nginx's stock site (`default.conf`), left untouched.
  - **unmanaged** — a file not tied to any current project (a leftover from a
    deleted project, or one ci-agent never wrote).

  Each row shows the file name, owner, the hostnames it serves, size and last
  modified. **Every** row has a **View** button that opens the file's on-disk
  contents in a read-only window. Only **unmanaged** files also get a **Delete**
  button: type the file name and click **Delete config** to confirm. Managed and
  default files can't be deleted here, so you can't lock yourself out by
  accident. **Test config (nginx -t)** checks the whole nginx setup for errors
  without changing anything.

> **Common problem: "Privileged host actions are unavailable."** This yellow
> banner on the Host and Firewall pages means the agent's sudo rules are not
> loaded, so applying nginx config, controlling services and changing firewall
> rules will fail. Fix it from an SSH session:
>
> ```
> sudo /usr/bin/ci-agent reconcile-sudoers
> ```
>
> If that logs *failed validation*, run
> `sudo visudo -cf /usr/share/ci-agent/ci-agent.sudoers` to see the error. See
> [Security](./security.md).

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

## Tunnel `/tunnel`

Set this box's tunnel **Role**. While tunneling is off the page shows two role
cards — **Online** (a public relay that also serves its own apps) and **Local**
(a private/NAT'd box that dials out). Picking one opens that role's screen with
an **On/Off switch** and only its settings: on the Online screen you manage
tunnels and hostname routes; on the Local screen, the relay connection
settings. Nothing changes until you turn the switch on. It lets a box with no
public IP serve its apps through a public one, with TLS terminating on the
private box. Full guide: **[Reverse tunnel](./tunnel.md)**.

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
- **unused networks** (**Remove unused networks**) — Docker networks no
  container is attached to, usually left over by deleted or stopped stacks.
  Each one holds an address range from Docker's pool, so clearing them leaves
  room for new deploys. Shared category networks are recreated on the next
  deploy, so this is safe.

Images belonging to the last *N* successful runs of every project (the
**Redeployable runs / project** setting, default **1**) and project-data volumes
are **protected** and **never** removed, so recent versions stay redeployable and
app data is safe. With auto-prune on (the default), this same cleanup also runs
automatically after each deploy and nightly — so you rarely need these buttons.
**The Docker actions operate on the whole Docker daemon** — if other software
shares this host's Docker, its dangling images and stopped containers are
affected too. OS-level cleanup (apt cache, system logs) is intentionally not
offered: the agent runs unprivileged — use the **Server shell** for that.

## Audit `/audit`

The **Audit log** lists every admin action and accepted webhook, with its
**Time**, **Actor**, **Action** and **Detail**: logins (and failures), project
changes, token rotations, triggers, cancellations, prunes, file changes and
settings changes. Changes made with an API key show the key as the actor
(`api:<key name>`).

## API keys `/api-keys`

API keys let scripts and AI agents use this server through the
[API](./api.md). Each key has an access level:

- **Read** — look at projects, runs, build logs, container logs and health.
- **Operator** — read, plus create image-mode projects, edit projects, change
  environment variables, deploy, cancel, and start/stop/restart containers. It
  can't choose a git repository, zip URL, inline compose file or category.
- **Admin** — operator, plus choose those sources, archive and delete
  projects, run commands inside containers, manage domains, certificates,
  default URLs and proxy tuning, and read the audit log. An admin key means
  full access to the server.

To create a key, fill in **Create key** (a name, the level and when it
expires) and click **Create key**. The page then shows the key **once**, with a
**Prompt for your AI agent** you can paste into the agent. The prompt contains
the key, the API address and links to the docs. Copy both before you leave the
page.

The **Keys** table lists every key with its level, the first characters of the
key, its status (**active**, **expired** or **revoked**), and when and from
which IP address it was last used. **Revoke** stops a key immediately. A
revoked or expired key can then be **deleted**.

**Connect an AI agent** shows the same prompt with placeholders. **What each
level can do** lists every API endpoint and the level it needs.

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
    the release `.deb` **and** its detached `.asc` signature directly, then click
    **Verify & stage update**. The agent
    verifies the signature **offline** against the
    embedded release key (pinned to fingerprint
    `A80F7309CF26133DBE88E2EC9C5489AAE86BF159`; any other key is rejected) and
    refuses downgrades, then **stages** the new binary. Click **Apply &
    restart** to install it: the service restarts and a privileged pre-start
    step swaps the binary in. If the new binary crash-loops it is automatically
    rolled back to the previous one after three failed starts. Until you apply
    it, **Discard staged update** throws the staged update away.
- **Storage** — a quick size breakdown of the data dir (database, run
  snapshots, backups, workspaces, uploads) and **Free disk (data dir)**. The full
  storage report and cleanup actions live on the **Storage** page (see
  *Storage* above).
- **Backups** — **Create backup now** makes a backup on demand (`VACUUM INTO`,
  safe while runs are active). You can download or **delete** existing backups.
  The nightly housekeeping backup (keep 7) continues regardless.
- **Restore** — choose a backup `.db` and click **Stage restore**. It is
  validated and **staged**: the swap happens at the next
  `systemctl restart ci-agent`, and the replaced database is kept beside it as
  `ci-agent.db.pre-restore-<timestamp>`. Until the restart, **Cancel staged
  restore** undoes it. Note: encrypted env vars also need the matching
  `secret.key`.
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
  *Branding* above.
- **Change password** — current password, new password (8+ characters),
  confirm; click **Update password**. Changing it **signs out every other
  session** (you stay logged in on this browser).
- **Two-factor authentication** — lists every enrolled authenticator with when
  it was added and last used, and how many backup codes you have left. A code
  from **any** listed device signs you in.
  - **Add a device** — optionally name it ("iPhone", "Work laptop"; blank gets
    numbered automatically), then scan the QR code it shows. Devices you
    already have keep working.
  - **Remove** — un-enrols that one device; codes from it stop working. Your
    last remaining device can't be removed, so add its replacement first.
  - **Regenerate backup codes** — issues a fresh set of 10 and retires the old
    ones.

  All three ask for your password first.
- **Maintenance** — controls automatic Docker cleanup:
  - **Auto-prune old images & build cache (nightly and after each deploy)** — a
    checkbox, **on by default**. When on, the agent reclaims disk space at two
    times: right after every successful deploy, and once a night. Each cleanup
    removes dangling images, build cache, and each project's **old images** that
    are no longer needed (see the keep count below). Protected images are never
    touched, and an image a running container still uses is never removed.
  - **Redeployable runs / project** — a number from **1 to 50** (default **1**).
    It sets how many of each project's most recent successful runs stay
    redeployable. Those runs keep their snapshot and their built Docker image
    (protected from cleanup); older ones are removed. Lower it to save the most
    disk (1 keeps only the version that's live now); raise it if you want to
    redeploy further back. Changing it takes effect on the next cleanup.
  - **Parallel deploys** — a number from **1 to 16** (default **1**). How many
    projects can build and deploy at the same time; runs of one project always
    go one after another. 1 is safest on small servers; higher values use more
    CPU, RAM and disk during builds. Takes effect **immediately**, no restart.
    It overrides `max_concurrent_runs` from the config file.

  > **What this means.** Every deploy builds a new image, and these can be large.
  > Without cleanup they pile up and fill the disk. The keep count decides how
  > many old versions are worth keeping so you can roll back; everything older is
  > safe to delete.
- **Automation API** — a pointer to the **API keys** page, where keys for
  scripts and AI agents are managed.
- **Shared environment** — `KEY=VALUE` lines (one per line) merged into **every**
  project's build and deploy; a per-project variable of the same name wins.
  Stored encrypted; applied on each project's next deploy. Click **Save**. See
  [Projects](./projects.md).
- **Default subdomain** — set a global **base domain** here; each project port
  can then be reached at `<slug>.<base>` (a subdomain, on by default) and/or
  `<base>/<slug>/` (an apex path, off by default), toggled per port in the
  **Ports** card on the project's **Settings** tab. An optional **base path** here nests every
  path route under a shared prefix (e.g. `/p` → `<base>/p/<slug>/`). A wildcard
  cert for the zone, if present, auto-enables HTTPS for both (get one on the
  **Certificates** page — see *Certificates* above). Blank disables them.
  Setting the base domain to the **same hostname as the admin panel** turns on
  single-domain mode — the panel stays at `<domain>/` and projects sit under the
  base path (which is then required). See [Domains & TLS](./domains.md).
- **Failed-deployment email alerts** — send an email whenever a deploy fails,
  and when a project's health check goes down or recovers (health emails go to
  the admin only).
  Fill in the **admin email**, **SMTP host** and **port**, the **security** mode
  (*STARTTLS (587)*, *Implicit TLS (465)*, or *None (25)*), optional **username**
  and **password**, and a **From** address. Click **Save**, or **Save & send
  test** to fire a test email right away. When a real deploy fails, the agent
  emails both the admin and the commit's committer. See
  [Operations](./operations.md).
- **Effective configuration** — the operational knobs (listen address, data
  dir, concurrency, timeouts, disk and ZIP limits, pull policy, hooks) shown
  **read-only**. Most live in `/etc/ci-agent/config.toml`; edit the file and
  restart the service to change them. **Max concurrent runs** is the exception:
  it shows the live value, which you change with **Parallel deploys** above.

> **What this means.** *SMTP* is the standard way programs send email. You point
> the agent at a mail server (your own internal relay, or any SMTP server it can
> reach) and it sends through that. On an air-gapped network, use an internal
> relay; choose **None (25)** only on a trusted LAN.
