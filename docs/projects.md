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
| **Image** | none | a prebuilt image you name directly (e.g. `mariadb:11`), with optional named volumes; no fetch, no build |
| **Inline compose** | none | a `docker-compose.yml` you paste into the UI; no fetch, no build |

**Compose** and **Dockerfile** are *source* modes: they fetch from git or a
ZIP and rebuild every run, so webhooks and manual git/ZIP runs apply. **Image**
and **Inline compose** are *sourceless*: there is nothing to clone or build, so
they have no webhook endpoints — deploy them with the **Deploy** button on the
project page (trigger `manual_deploy`). Use them for off-the-shelf images
(databases, caches) or a hand-written stack you don't keep in git.

The conventions below describe **Compose** mode in full; Dockerfile and Image
modes follow the same env/port/volume rules through the compose file the agent
synthesizes for them.

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

## Git access

- Public/internal repos over `https://` or `file://` work as-is.
- **Private GitHub repos over HTTPS**: register a personal access token for the
  repository owner on the **GitHub tokens** page (one token per GitHub
  username, with `repo` read access). When a project clones
  `https://github.com/<owner>/...`, the token registered for `<owner>`
  authenticates it automatically. Tokens are encrypted at rest, only ever sent
  to `github.com` exactly (lookalike hosts and userinfo tricks in
  webhook-supplied URLs are rejected), never appear in command lines or run
  logs, and are masked like any project secret.
- Clones never prompt (`GIT_TERMINAL_PROMPT=0`): a private repo without
  working credentials fails fast with a clear error instead of hanging.
