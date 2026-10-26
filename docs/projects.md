# Project conventions

There is no pipeline DSL. The pipeline is always *fetch → build → (test) →
deploy*, and a project's **deploy mode** decides where the compose file comes
from.

## Deploy modes

Pick one when you create the project:

| Mode | Source | What it deploys |
|---|---|---|
| **Compose** (default) | git repo or ZIP | the repo's `docker-compose.yml` (path configurable) |
| **Dockerfile** | git repo or ZIP | a single service built from a `Dockerfile`; the agent synthesizes a one-service compose file around it, publishing `container_port` on the registered HTTP port |
| **Image** | none | a prebuilt image you name directly (e.g. `mariadb:11`), with optional volumes; no fetch, no build |
| **Inline compose** | none | a `docker-compose.yml` you paste into the UI; no fetch, no build |

**Compose** and **Dockerfile** are *source* modes: they fetch from git or a
ZIP and rebuild every run, so webhooks and manual git/ZIP runs apply. **Image**
and **Inline compose** are *sourceless*: there is nothing to clone or build.
Deploy them with the **Deploy now** button in the **Trigger a run** card on the
project's **Deployments** tab (trigger `manual_deploy`). Use them for
off-the-shelf images (databases, caches) or a hand-written stack you don't keep
in git.

The **Webhooks** card on the Deployments tab shows the webhook URLs for every
mode. A sourceless project with no **Git URL** does not need them: a call to its
`/git` webhook is answered with `422` and the message
`no git_url in payload and project has none`.

The conventions below describe **Compose** mode in full; Dockerfile and Image
modes follow the same env/port/volume rules through the compose file the agent
synthesizes for them.

## One-click app catalog

Instead of writing a compose file by hand, **Projects → Browse app catalog**
offers a searchable grid of ready-made app templates (Redis, Postgres, …).
Pick one, fill a short form (project name, port, and any app variables such as
a password — generated ones are pre-filled), and the agent creates an **Inline
compose** project from the template with your values substituted.

Leave the **port blank** for an *internal-only* app: the agent strips the
`{{HTTP_PORT}}` mappings so nothing is published to the host and no nginx vhost
is created. This is the right choice for databases and caches that only other
containers reach over the shared Docker network.

The catalog is populated from **catalog sources** holding `apps/*.toml`
templates, managed under **Browse app catalog → Manage sources**. A source is
one of two kinds:

- **Git source** — a git repository the agent clones. **Sync** clones it to
  this server and parses its templates; sources are cached locally, so the
  catalog keeps working offline after a sync. An official git source is
  pre-configured; you can disable or delete it, and add your own.
- **Uploaded bundle** — for offline / air-gapped servers with no git access.
  Add a bundle source (name only), then use **Upload bundle** on its row to
  upload a `.zip` of `apps/*.toml` (the same files a git source would hold —
  the archive may contain `apps/` at its root or inside a single wrapping
  directory). The agent validates the templates and swaps them into the same
  on-disk cache a clone produces, so the rest of the catalog is identical.
  Re-upload to refresh; this reuses the file-upload path used for self-update,
  so no network is involved.

Apps from every enabled source — of either kind — are merged into one list.
Each source can hold up to 500 apps, and each template file can be up to
256 KB; anything beyond that is skipped.

- ⚠ Only add sources you trust: deploying one of their apps runs its containers
  on this host.

Catalog apps reference public images (e.g. `redis:7-alpine`). Every deploy
tries to pull images first (see *Images and pulling* below), so an
internet-connected host fetches them by itself. On an air-gapped host the pull
fails quietly, so pre-load the images first (see
[Troubleshooting](./troubleshooting.md)). Generated passwords live inside the
project's compose file — copy them from the configure form before creating.

A source template (`apps/redis.toml`) looks like:

```toml
slug = "redis"
name = "Redis"
category = "Database"
description = "In-memory key-value store, cache and message broker."
keywords = ["cache", "kv", "queue"]
http_port = 6379
compose = '''
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--requirepass", "{{REDIS_PASSWORD}}"]
    ports: ["{{HTTP_PORT}}:6379"]
    volumes: ["redis_data:/data"]
volumes:
  redis_data:
'''

# All top-level keys (including `compose`) must come BEFORE the variables
# blocks — TOML absorbs anything after `[[variables]]` into that table.
[[variables]]
key = "REDIS_PASSWORD"
label = "Redis password"
generate = true            # form field pre-filled with a generated secret
```

`{{HTTP_PORT}}` is replaced by the chosen port; every other `{{TOKEN}}` must be
declared in a `[[variables]]` block (validated on sync).

A `[[variables]]` block has these fields:

| Field | Required | Meaning |
|---|---|---|
| `key` | yes | the `{{KEY}}` placeholder it fills |
| `label` | yes | the form field's label |
| `default` | no | a value pre-filled in the form |
| `generate` | no | `true` pre-fills a freshly generated secret (passwords, tokens) |
| `description` | no | help text for the field |

### App actions in a template

A template can also ship ready-made **actions** — for example a database backup
and restore. They are copied onto the project when you create it, and you can
run or edit them later on the project's **Actions** page — open it with the
**Actions** button at the top of the project page (see the
[Admin UI guide](./ui.md)). Add them as `[[actions]]` blocks. Like
`[[variables]]`, they must come after all the top-level keys:

```toml
[[actions]]
key = "backup"
label = "Backup (SQL dump)"
kind = "backup"            # run | backup | restore
service = "db"             # compose service the command runs in
command = "mariadb-dump -uroot -p\"$MARIADB_ROOT_PASSWORD\" --all-databases"
filename = "mariadb-{{DATE}}.sql"

[[actions]]
key = "restore"
label = "Restore from SQL dump"
kind = "restore"
service = "db"
upload_path = "/tmp/restore.sql"
load_command = "mariadb -uroot -p\"$MARIADB_ROOT_PASSWORD\" < {{FILE}}"
```

- Every action needs `key`, `label`, `kind` and `service`. Keys must be unique.
- `run` and `backup` need a `command`. `run` shows the output; `backup` sends the
  output to your browser as a file named by `filename`.
- `restore` needs `upload_path` and `load_command`. The uploaded file is copied to
  `upload_path` inside the container, then `load_command` runs.
- Commands run **inside** the container (`sh -c`), so they can read the
  container's own environment variables. Only `{{DATE}}` (a `YYYYMMDD-HHMMSS`
  stamp) and, for restore, `{{FILE}}` are filled in by the agent.

## The compose file

- Default path `docker-compose.yml`, configurable per project.
- Services **with a `build:` section** are built every run. The agent pins
  their image to `ci-agent/<project>-<service>:run-<N>` — do **not** set
  `image:` on build services yourself.
- Services without `build:` use whatever image they name. The agent tries to
  pull it before each deploy (see *Images and pulling* below).
- Service names may only use letters, digits, `.`, `_` and `-`. Any other
  character fails the deploy with
  `service name "…" contains unsupported characters`.

### What the agent adds to your compose file

The agent never edits your file. It adds a small extra file (an *override*) on
top of it at deploy time. For every service the override:

- pins build services to `ci-agent/<project>-<service>:run-<N>`;
- sets `container_name: ci-agent--<project>-<service>`, so containers have
  predictable names in `docker ps` — any `container_name` you set is replaced;
- adds `restart: unless-stopped` when the service has no `restart:` of its own
  (a service with `restart: "no"` keeps it);
- attaches the service to the project's category networks (see
  *Categories & networking* below);
- adds the ports, volumes, resource limits and GPU access you set on the
  **Settings** tab.

### Images and pulling

Before every deploy and every redeploy the agent runs
`docker compose pull --ignore-buildable`. This fetches a newer copy of each
`image:` service (for example a moving `:latest` tag). Services with `build:`
are skipped here.

If the pull fails — always the case on an air-gapped box — the run log shows a
warning and the deploy carries on with the images already on the server.

The `pull_policy` config key only changes the **build** step: `always` adds
`--pull` to `docker compose build`, so base images in your Dockerfiles are
refreshed. `never` and `missing` do the same thing (no `--pull`). See
[Configuration](./configuration.md).

> **What this means.** On a server with internet access, images update
> themselves on each deploy. On an offline server nothing breaks — the agent
> just uses what you loaded with `docker load`.

### Editing the compose file in the UI

In **Compose** mode the compose file normally comes from the repo, but the
project's **Settings** tab has an optional **Edit compose file** box where you
can paste an **inline override**. When set, that text is deployed instead of the
file at the compose path — the repo is still cloned and the deploy runs in the
checkout, so Dockerfiles and relative build contexts still resolve. Leave it
blank to deploy the committed file as-is.

Two buttons pre-fill the editor so you don't start from scratch:

- **Load from last commit** — the compose file exactly as it is in the repo.
- **Load from last deploy** — the compose file from this project's last
  successful deploy.

(In **Inline compose** mode the same box holds the full `docker-compose.yml`
that is deployed verbatim — there is no repo to load from.)

### Monorepos

Set a **working directory** on the project to build from a subdirectory of the
repo. When set, the compose-file path, the `Dockerfile` path, the synthesized
override, and the Docker build context all resolve under it; leave it blank to
use the repo root. The path is validated (no `..` escapes outside the
checkout).

## Web exposure

Host nginx proxies `your-domain → 127.0.0.1:<port>`, where `<port>` is the
project's registered HTTP port. Your app only has to publish that port on the
host.

**In Compose mode** you can write the mapping yourself. Bind it to loopback so
only nginx on this server can reach it:

```yaml
services:
  web:
    build: .
    ports:
      - "127.0.0.1:8101:80"
```

**Ports the agent writes for you are open on every network interface.** This
covers:

- Dockerfile and Image projects (the agent writes the whole compose file), and
- ports you add on the **Ports** card of the **Settings** tab and attach to a
  **Service** (Compose and Inline compose modes).

These are written as `host:container` (for example `8101:80`), not
`127.0.0.1:8101:80`. So other machines can reach the app on that port directly,
not only through nginx. If your compose file already publishes a port, the agent
does not add it a second time — so a loopback mapping you write yourself wins.

To keep such a port private, block it **in front of** the server (a network or
cloud firewall), or leave the port blank for an internal-only app (see
*Categories & networking* below). Note that Docker writes its own firewall rules
for published ports, so a UFW rule on the
[Firewall page](./ui.md) does **not** block them by default — see
[Security](./security.md).

> **What this means.** A *loopback* address (`127.0.0.1`) is reachable only
> from the server itself. A port published on *every interface* is reachable
> from any machine that can reach the server's IP.

The project's **nginx**
page generates the server block from the domains + port you registered and can
**apply it in one click** (write → `nginx -t` → reload) when the sudo allowlist
is installed; otherwise it shows the equivalent copy-paste commands. A project
can serve **several domains** (one **Primary** domain; the others 301-redirect to it) and opt
each one into HTTPS — see **[Domains & TLS](./domains.md)**.

The agent warns in the run log (without blocking) when no service publishes the
registered port (`project http_port … not published by any service`), or when
another project claims the same port (`http_port … also claimed by: …`).

A project can also expose **extra ports** (on the **Ports** card of the
**Settings** tab) when it runs more than one service — each extra port gets its
own domains (attached on the **Domains** tab) and default URL. See
[Domains & TLS](./domains.md).

## Categories & networking

Each project can have **one or more categories**. A category is a private Docker
network: the project joins the network of **every** category it lists, and any
two projects that share **at least one** category can reach each other by
**hostname** (handy when an app needs to talk to a database you run as a
separate project). Projects with **no category in common** are network-isolated
and cannot see each other.

- Enter categories on the create or Settings form as a comma-separated list
  (each becomes a removable chip). Lowercase letters, digits and dashes.
- Leave it **blank** to use the shared `default` category. Add `default`
  alongside named categories to stay on the shared network *and* a private one.
- Each category maps to a network named `ci-agent-shared-<category>`; the
  built-in `default` category maps to `ci-agent-shared`.
- Within a network a project is reachable at `ci-agent--<slug>` (single-service
  apps) or `ci-agent--<slug>-<service>` (one per service in a multi-service
  stack). The project's own hostnames are shown on its **Overview** tab under
  **Internal access**.
- Changing a project's category list **re-homes its running stack immediately**
  on save — no redeploy needed.
- There is no separate per-project network: a project's services meet **only**
  on its category networks.

### Service names must be unique within a category

Projects in the same category share one network, and Docker registers every
**service name** on it. If a **multi-service** project (say, `web` + `db`) shares
a category with another project that also has a service called `db`, the name
`db` can point to the **other** project's container — so your app could talk to
the wrong database.

The agent guards against this:

- The **create** and **Settings** forms refuse to save a project whose service
  names clash with another project in a shared category. The message reads
  *Service name collision on the shared category network: … rename the service
  or pick a different category.*
- A deploy that **newly** brings in a clash fails with
  *shared-network service-name collision: …*. A clash that already existed
  before only logs a warning in the run log, so an existing project keeps
  deploying.
- The **Projects** page shows a **Shared-network service-name
  collisions** warning banner listing any clashes.

Two **single-service** projects with the same service name (two `app`s, for
example) are fine — neither looks up the bare name.

**Fix:** rename the service in one of the compose files, or move one project to
a different category.

> **What this means.** A *Docker network* is a private network that only the
> containers on it can use. Giving two projects a category in common is how you
> let them connect — for example, a web app in category `shop` can reach a
> database also in category `shop`, with nothing exposed to the outside.

> **Example.** Run Postgres as one project and your web app as another, both in
> category `shop`. The web app connects to the database at its hostname over the
> shared network — you never publish the database port to the host. Put a shared
> cache in categories `shop, blog` and both apps can reach it while staying
> isolated from each other.

## Tests (optional)

Add a service named exactly `ci-test` with the `ci` profile:

```yaml
  ci-test:
    build: .            # or image: ...
    profiles: ["ci"]
    command: ["./run-tests.sh"]
```

It runs after build, before deploy, as
`docker compose --profile ci run --rm --no-deps ci-test`.
A non-zero exit aborts the run — nothing gets deployed. The profile keeps it
out of the deployed stack.

`--no-deps` means **no other service is started for the test**. If your tests
need a database, start one inside the test command, or use a database that is
already running (for example another project in the same category).

## Environment

- Project env vars are set in the UI (KEY=VALUE lines), stored encrypted,
  injected into every compose command, and masked in run logs (`*****`):
  - a value whose name sounds secret (it contains `PASS`, `SECRET`, `TOKEN`,
    `KEY`, `AUTH`, `PRIVATE`, `CREDENTIAL`, `SESSION`, `COOKIE`, `SIGN`,
    `CERT`, `SALT`, `PIN` or `DSN`) is always masked;
  - any other value is masked when it is 6 characters or longer, unless it is
    a number or a common setting (`true`, `false`, `production`, …). So
    `REPLICAS=3` doesn't turn every `3` in the log into `*****`.
- **Shared environment** — set global `KEY=VALUE` lines under
  **Settings → Shared environment** and they are merged into **every** project's
  build and deploy. Use it for values every stack needs (an internal registry
  host, `TZ`, an SMTP relay, a CA-bundle path). A per-project variable with the
  same key **overrides** the shared one. Shared vars are encrypted at rest and
  apply on each project's next deploy. See the [Admin UI guide](./ui.md).
- `CI_RUN_ID` and `CI_COMMIT_SHA` are provided during build and deploy.

## Data

Use **named volumes** for anything that must survive a redeploy (databases,
uploads, and so on). The checkout/workspace is wiped after every run, so
bind mounts into the source tree will not survive.

> **Note:** a *named volume* is Docker storage with a name (e.g.
> `db-data:/var/lib/mysql`) that lives outside the container, so it stays put
> when the container is rebuilt. A *bind mount* into the cloned source
> (e.g. `./data:/data`) does **not** survive, because the clone is deleted
> after each run.

You can also attach storage from the UI: the **Volumes** card on the project's
**Settings** tab adds a named volume or a host-path bind mount. It works in every
mode. For **Image** and **Dockerfile** projects it is the only way, because you
don't write their compose file. For **Compose** and **Inline compose** projects,
pick the **Service** the volume belongs to, and the agent adds it to that
service for you. See the [Admin UI guide](./ui.md).

## Resource limits & GPU

Each project's **Settings** tab has a **Resource limits** card with one row per
discovered service:

- **CPU / memory caps** — cap a service's CPU (fractional cores, `0.5` = half a
  core) and memory (in MB) so one container can't starve the box. Blank means no
  cap. These become a `deploy.resources.limits` block on the service.
- **GPU access** — when the host has an NVIDIA GPU (detected via `nvidia-smi`),
  each row also shows a **GPU** checkbox. Ticking it grants that service the
  GPU — the Compose equivalent of `docker run --gpus all`, written as a
  `deploy.resources.reservations.devices` block (driver `nvidia`, `count: all`).
  Select as many services as you need; the choice is per service.

Both apply on the **next deploy**. For Image / Dockerfile projects a **Redeploy**
or **Recreate** also applies them (the agent regenerates the compose override
from the current settings).

> **GPU prerequisite:** a detected GPU only proves the *driver* is installed.
> Docker also needs the **NVIDIA Container Toolkit** registered as a runtime.
> ci-agent pre-checks for it: if it's missing, the Resource limits card warns
> **"GPU toolkit not configured"** and a GPU deploy fails early with the setup
> guidance — rather than the opaque `could not select device driver "nvidia"`
> error mid-deploy. One time on the host:
>
> ```
> sudo apt-get install -y nvidia-container-toolkit
> sudo nvidia-ctk runtime configure --runtime=docker
> sudo systemctl restart docker
> ```
>
> Verify with `docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi`.
> The Resource limits card surfaces these same commands when you tick a GPU box.
> On an air-gapped box, side-load the `nvidia-container-toolkit` package first.

## Git access

- Public/internal repos over `https://`, `http://` (handy on a trusted LAN) or
  `file://` work as-is.
- **SSH keys are not supported.** An SSH-style URL — `git@host:owner/repo.git`
  or `ssh://git@host:2222/owner/repo.git` — is rewritten to
  `https://host/owner/repo.git` before cloning (any SSH port is dropped), and
  then uses the HTTPS credentials below. If your git server only serves SSH,
  enable HTTPS on it or use the ZIP webhook instead.
- **Private repos over HTTPS**: register a credential once on the
  **Git credentials** page (in the sidebar) — see the
  [Admin UI guide](./ui.md). Two shapes:
  - **GitHub** — host `github.com`, the repo **Owner**, and a personal access
    token (with `repo` read access) as the **Secret**. Leave **Username** blank
    (it defaults to `x-access-token`). When a project clones
    `https://github.com/<owner>/...`, the matching credential authenticates it
    automatically. Leave **Owner** blank for a *host-wide* credential that is
    used for every repo on `github.com`; a credential for the exact owner always
    wins over the host-wide one. GitHub tokens are only ever sent to `github.com`
    exactly (lookalike hosts and userinfo tricks in webhook-supplied URLs are
    rejected).
  - **Other git hosts** — the **Host** (with `:port` if it isn't the default,
    e.g. `gitea.example.com:3000`), an account **Username**, and its password
    or token as the **Secret**. The credential is used for any repo on that
    exact host.
  - Saving the same host + owner again replaces the old credential.
- Credentials are encrypted at rest, never appear in command lines or run logs,
  and are masked like any project secret.
- Clones never prompt (`GIT_TERMINAL_PROMPT=0`): a private repo without
  working credentials fails fast with a clear error instead of hanging.
