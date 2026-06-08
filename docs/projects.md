# Project conventions

There is no pipeline DSL. A deployable project is a git repo (or ZIP) that
contains a `docker-compose.yml`; the pipeline is always *fetch → build →
(test) → deploy*.

## The compose file

- Default path `docker-compose.yml`, configurable per project.
- Services **with a `build:` section** are built every run. The agent pins
  their image to `ci-agent/<project>-<service>:run-<N>` — do **not** set
  `image:` on build services yourself.
- Services without `build:` use whatever image they name; it must already
  exist locally (see `pull_policy`).
- Service names should stick to letters, digits, `.`, `_`, `-`.

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

Host nginx proxies `your-domain → 127.0.0.1:8101`. The project's **nginx
guide** page in the UI generates the exact one-time commands (server block,
`nginx -t`, reload) from the domain + port you registered.

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

Use **named volumes** for anything persistent. The checkout/workspace is
wiped after every run, so bind mounts into the source tree will not survive.

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
