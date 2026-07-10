# CI Agent

A single-binary CI/CD agent for **offline / air-gapped Ubuntu servers** that
build and deploy **Docker Compose** stacks behind a manually-managed host
nginx.

## What it does

1. Receives a webhook with a **git URL** (or a **ZIP file URL**), fetches the
   sources, and runs a deployment — or deploys a prebuilt image / pasted
   compose with no source at all.
2. Builds every `build:` service in the project's `docker-compose.yml`,
   tagging images per run so old versions stay redeployable.
3. Optionally runs your tests (`ci-test` service) before deploying.
4. Deploys with `docker compose up -d`, health-gates the result, and
   automatically rolls back to the last good version on failure.
5. Manages each project's nginx server block and TLS — multiple domains with a
   canonical 301, per-domain HTTPS, and certificates from an internal CA,
   self-signed, uploaded, or Let's Encrypt.
6. Serves this admin UI: running jobs, deployment history, live run and
   container logs, redeploy, Docker cleanup tools, server resource gauges, and
   manual runs from a git URL or an uploaded ZIP.

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

**New here? Start with Installation, then take the UI tour.**

- **[Installation](./install.md)** — get it running in five minutes.
- **[Admin UI guide](./ui.md)** — a tour of every page in the panel.
- **[Project conventions](./projects.md)** — what a deployable repo must look like.
- **[Webhooks](./webhooks.md)** — trigger deployments from anywhere.
- **[Domains & TLS](./domains.md)** — multiple domains, nginx, certificates.
- **[Reverse tunnel](./tunnel.md)** — serve a NAT'd box through a public one.
- **[Configuration](./configuration.md)** — every config.toml key.
- **[Operations](./operations.md)** — queueing, rollback, retention, backups.
- **[Security](./security.md)** — auth model, two-factor login, exposure guidance.
- **[Troubleshooting & FAQ](./troubleshooting.md)** — common problems and quick answers.
