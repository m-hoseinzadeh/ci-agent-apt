# CI Agent

A single-binary CI/CD agent for **offline / air-gapped Ubuntu servers** that
build and deploy **Docker Compose** stacks behind a manually-managed host
nginx.

## What it does

1. Receives a webhook with a **git URL** (or a **ZIP file URL**), fetches the
   sources, and runs a deployment.
2. Builds every `build:` service in the project's `docker-compose.yml`,
   tagging images per run so old versions stay redeployable.
3. Optionally runs your tests (`ci-test` service) before deploying.
4. Deploys with `docker compose up -d`, health-gates the result, and
   automatically rolls back to the last good version on failure.
5. Serves this admin UI: running jobs, deployment history, live logs,
   redeploy, Docker cleanup tools, server resource gauges, and manual runs
   from a git URL or an uploaded ZIP.

## Design at a glance

| Property | Choice |
|---|---|
| Footprint | one ~6 MB fully-static binary, embedded SQLite, no external services |
| Pipeline | convention over configuration: the compose file *is* the pipeline |
| Concurrency | serial per project, capped across projects |
| Offline | nothing is pulled by default (`pull_policy = "never"`); no CDN, no telemetry |
| Proxy | plain host nginx, configured once per project by you (the UI generates the commands) |
| Distribution | signed apt repository (`apt-get upgrade` keeps it current) or a `.deb`/binary you transfer by hand; built automatically from every commit |
| Updates | apt, the Maintenance page's built-in self-update, or scp — your choice |

## Where to go next

- **[Installation](./install.md)** — get it running in five minutes.
- **[Project conventions](./projects.md)** — what a deployable repo must look like.
- **[Webhooks](./webhooks.md)** — trigger deployments from anywhere.
- **[Configuration](./configuration.md)** — every config.toml key.
- **[Operations](./operations.md)** — queueing, rollback, retention, backups.
- **[Security](./security.md)** — auth model and exposure guidance.
