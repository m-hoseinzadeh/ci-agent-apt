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
- **Private GitHub repos over HTTPS**: set `github_token` in the agent config
  — all `https://github.com/...` clones authenticate with it automatically
  (one PAT with `repo` read access covers all your private projects). The
  token is only ever sent to `github.com` exactly (lookalike hosts and
  userinfo tricks in webhook-supplied URLs are rejected), never appears in
  command lines or run logs, and is masked like any project secret.
- For SSH repos (GitHub or anywhere else), set a deploy key path on the
  project; the agent clones with that identity (`IdentitiesOnly=yes`).
- Clones never prompt (`GIT_TERMINAL_PROMPT=0`): a private repo without
  working credentials fails fast with a clear error instead of hanging.
