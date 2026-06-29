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
and **Inline compose** are *sourceless*: there is nothing to clone or build, so
they have no webhook endpoints — deploy them with the **Deploy now** button on
the project's Overview tab (trigger `manual_deploy`). Use them for off-the-shelf
images (databases, caches) or a hand-written stack you don't keep in git.

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

- ⚠ Only add sources you trust: deploying one of their apps runs its containers
  on this host.

Catalog apps reference public images (e.g. `redis:7-alpine`). On an
internet-connected host `docker compose up` pulls any missing image
automatically; on an air-gapped host, pre-load the images first (see
`pull_policy` and Troubleshooting). Generated passwords live inside the
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

## The compose file

- Default path `docker-compose.yml`, configurable per project.
- Services **with a `build:` section** are built every run. The agent pins
  their image to `ci-agent/<project>-<service>:run-<N>` — do **not** set
  `image:` on build services yourself.
- Services without `build:` use whatever image they name; it must already
  exist locally (see `pull_policy`).
- Service names should stick to letters, digits, `.`, `_`, `-`.

### Monorepos

Set a **working directory** on the project to build from a subdirectory of the
repo. When set, the compose-file path, the `Dockerfile` path, the synthesized
override, and the Docker build context all resolve under it; leave it blank to
use the repo root. The path is validated (no `..` escapes outside the
checkout).

## Web exposure

Publish a **fixed loopback port** that matches the project's registered HTTP
port:

```yaml
services:
  web:
    build: .
    ports:
      - "127.0.0.1:8101:80"
```

Host nginx proxies `your-domain → 127.0.0.1:8101`. The project's **nginx**
page generates the server block from the domains + port you registered and can
**apply it in one click** (write → `nginx -t` → reload) when the sudo allowlist
is installed; otherwise it shows the equivalent copy-paste commands. A project
can serve **several domains** (with one canonical and 301 redirects) and opt
each one into HTTPS — see **[Domains & TLS](./domains.md)**.

The agent warns (without blocking) when the compose file doesn't publish the
registered port, or when another project claims the same port.

A project can also expose **extra ports** (on the **Domains & Ports** tab) when
it runs more than one service — each extra port gets its own domains and default
URL. See [Domains & TLS](./domains.md).

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

It runs after build, before deploy (`docker compose run --rm ci-test`).
A non-zero exit aborts the run — nothing gets deployed. The profile keeps it
out of the deployed stack.

## Environment

- Project env vars are set in the UI (KEY=VALUE lines), stored encrypted,
  injected into every compose command, and masked in run logs.
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

For **Image** and **Dockerfile** projects (which have no compose file you
write), attach storage from the UI instead: the **Volumes** card on the
project's Settings tab adds a named volume or a host-path bind mount. See the
[Admin UI guide](./ui.md).

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
> Docker also needs the **NVIDIA Container Toolkit** registered as a runtime, or
> the deploy fails with `could not select device driver "nvidia"` and rolls back.
> One time on the host:
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

- Public/internal repos over `https://` or `file://` work as-is.
- **Private repos over HTTPS**: register a credential once on the
  **Git credentials** page (in the sidebar) — see the
  [Admin UI guide](./ui.md). Two shapes:
  - **GitHub** — host `github.com`, the repo **owner**, and a personal access
    token (with `repo` read access) as the secret. When a project clones
    `https://github.com/<owner>/...`, the matching credential authenticates it
    automatically. GitHub tokens are only ever sent to `github.com` exactly
    (lookalike hosts and userinfo tricks in webhook-supplied URLs are rejected).
  - **Other git hosts** — the host name, an account **username**, and its
    password or token as the secret.
- Credentials are encrypted at rest, never appear in command lines or run logs,
  and are masked like any project secret.
- Clones never prompt (`GIT_TERMINAL_PROMPT=0`): a private repo without
  working credentials fails fast with a clear error instead of hanging.
