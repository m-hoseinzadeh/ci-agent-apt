# API & AI agents

ci-agent has an **HTTP API** for scripts and **AI agents** (Claude Code, Cursor,
your own bots). With an API key, an agent can look at your projects, read build
and container logs to find bugs, create and change projects, set environment
variables, and deploy. You decide how much each key can do.

## Quick start

1. In the admin panel, open **API keys** (in the sidebar).
2. Under **Create key**, type a name (for example `claude-ops`), pick an
   **access level** and an expiry, then click **Create key**.
3. Copy the key. **It is shown only once.** The page also shows a
   **Prompt for your AI agent**. Paste that prompt into your agent. It contains
   the key, the API address and links to these docs.
4. Test it:

```bash
curl -H 'Authorization: Bearer cia_…' https://ci.example.com/api/v1
```

The answer lists your key's level, every endpoint, whether your key may use
it, and where the docs are.

## For AI agents: how to start

If you are an AI agent and a human gave you a ci-agent key, do this:

1. Call `GET /api/v1`. Read `key.level` and the `endpoints` list. Only call
   endpoints where `allowed` is `true`.
2. Read this guide as Markdown at `GET /api/v1/docs` (needs the key) or on the
   public site: https://m-hoseinzadeh.github.io/ci-agent-apt/docs/api.html
3. Deploys are **asynchronous**. A deploy call returns a queued run right away.
   Poll the run until its `status` is `success`, `failed` or `cancelled`.
4. To find a bug, read logs first: the run log (build and deploy output), then
   the container logs (the running app). Change things only after that.
5. Ask the human before you delete or archive a project, or run commands inside
   containers.

## Keys and access levels

A key looks like `cia_` followed by 64 hex characters. Send it on every request:

```
Authorization: Bearer cia_…
```

Each key has one level. A higher level can do everything a lower level can.

| Level | What it can do |
|---|---|
| `read` | See projects, runs, build logs, container logs, containers and health. |
| `operator` | Everything in `read`, plus: create image-mode projects, edit projects, read and change environment variables, deploy, redeploy, cancel, recreate, start/stop/restart containers, pause deploys. Works with the source an admin set up; it **cannot** choose a git repository, zip URL, inline compose file or category. |
| `admin` | Everything in `operator`, plus: choose git repositories, zip URLs, inline compose files and categories; archive and delete projects; run commands inside containers; read the audit log; rotate webhook tokens. |

**Why operator keys can't choose the source:** a compose file or repository
decides what runs, and a compose file can ask for a privileged container or
mount host folders, which gives full control of the server. Operator keys can
only run what an admin already set up, plus image-mode projects, where
ci-agent writes the compose file itself.

Good to know:

- ci-agent stores only a **hash** of each key. If you lose a key, revoke it and
  create a new one.
- Keys can **expire** (7, 30, 90 or 365 days, or never). Expired and revoked
  keys stop working at once.
- The **API keys** page shows when and from which IP address each key was last
  used.
- Every change made with a key is written to the **Audit** page as
  `api:<key name>`.
- Keys **skip the password and 2FA login**. Give each agent its own key with the
  lowest level it needs.

## Conventions

- **Base URL:** `https://<your admin domain>/api/v1`. An agent running on the
  server itself can also use `http://127.0.0.1:8044/api/v1` (8044 is the
  default port; see `listen` in [Configuration](./configuration.md)). Other
  machines reach it through the admin domain, or on the server's IP and port
  only if you opened the panel port.
- **Projects** are named by their **slug** (for example `shop`).
- **Runs** (deployments) have a number per project:
  `/api/v1/projects/shop/runs/12`. Each run also has a global `id`, used by
  `/api/v1/runs/{id}`.
- Request and response bodies are **JSON**. Unknown fields in a request are
  rejected, so typos don't pass silently.
- Passwords inside URLs (`https://user:token@…`) are shown as `***@`, and the
  query part of a URL is replaced with `?…`, so keys don't leak them.
- **Errors** look like this:

```json
{ "error": { "code": "insufficient_level", "message": "this endpoint needs an admin key; this key is operator", "docs": "/api/v1/docs" } }
```

| Status | Meaning |
|---|---|
| 401 | No key, wrong key, revoked key or expired key (`missing_key`, `invalid_key`, `revoked_key`, `expired_key`). |
| 403 | The key's level is too low (`insufficient_level`; the error shows `required_level` and `your_level`). |
| 404 | No such project, run, container or endpoint. |
| 409 | Conflict, e.g. `slug_taken`, `deploys_paused`, `run_active`, `nothing_deployed`. |
| 422 | The request is invalid (`invalid_request`); the message says what to fix. |
| 429 | Too many wrong keys from your address (wait a minute), or another `exec` is still running. |

## Read endpoints (`read` level)

### Discovery and docs

- `GET /api/v1` — your key, its level, every endpoint (with `allowed`), docs links.
- `GET /api/v1/docs` — this guide as Markdown.

### Health and projects

- `GET /api/v1/health` — every project with `status` (`healthy`, `degraded`,
  `problem`) and a list of `problems`.
- `GET /api/v1/projects` — all projects with their mode, categories and last
  run. Add `?archived=true` to include archived projects.
- `GET /api/v1/projects/{slug}` — one project: git URL and branch, mode, ports,
  services, domains, health and last run. Operator and admin keys also get
  `compose_inline`.

### Runs and build logs

- `GET /api/v1/projects/{slug}/runs` — recent runs (`?limit=20`, at most 200).
- `GET /api/v1/projects/{slug}/runs/{number}` — one run: status, trigger,
  commit, times and `error`.
- `GET /api/v1/projects/{slug}/runs/{number}/log` — the end of the run's build
  and deploy log (`?tail=500` lines, at most 5000).
- `GET /api/v1/runs` — recent runs of all projects (`?limit=50`), or only the
  queued and running ones with `?active=true`.
- `GET /api/v1/runs/{id}` — one run by its global id.

### Containers and app logs

- `GET /api/v1/projects/{slug}/containers` — the project's containers with
  service name, state, CPU and memory.
- `GET /api/v1/projects/{slug}/logs` — the end of the containers' logs.
  Options: `?service=web` (one service), `?tail=200` (at most 2000),
  `?since=10m` (`30s`, `10m`, `2h`, `1d`).

Example: why did the last deploy of `shop` fail?

```bash
KEY='cia_…'; API=https://ci.example.com/api/v1
curl -H "Authorization: Bearer $KEY" "$API/projects/shop/runs?limit=1"
curl -H "Authorization: Bearer $KEY" "$API/projects/shop/runs/12/log?tail=200"
curl -H "Authorization: Bearer $KEY" "$API/projects/shop/logs?tail=100&since=30m"
```

## Operator endpoints (`operator` level)

### Create and edit projects

- `POST /api/v1/projects` — create a project. Fields:

| Field | Required | Notes |
|---|---|---|
| `name`, `slug` | yes | slug: lowercase letters, digits, dashes |
| `mode` | no | `compose` (default), `dockerfile`, `image`, `inline` |
| `git_url` | for git deploys | **admin only** |
| `git_branch` | no | defaults to `main` |
| `compose_path` | no | compose mode, default `docker-compose.yml` |
| `dockerfile_path` | no | dockerfile mode, default `Dockerfile` |
| `working_dir` | no | sub-folder of the repo to build in (monorepos) |
| `image_ref` | image mode | e.g. `nginx:1.27` |
| `compose_inline` | inline mode | the full docker-compose.yml text; **admin only** |
| `http_port`, `container_port` | no | host port and app port |
| `category` | no | `"a,b"` or `["a","b"]` — shared Docker networks; **admin only** |
| `env` | no | `{"KEY": "value"}` |
| `deploy` | no | `true` = deploy right after creating |

With an **operator** key, create an image-mode project:

```bash
curl -X POST -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"name":"Cache","slug":"cache","mode":"image","image_ref":"redis:8-alpine","container_port":6379,"deploy":true}' \
  "$API/projects"
```

With an **admin** key, create a project from a git repository:

```bash
curl -X POST -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"name":"Shop","slug":"shop","git_url":"https://git.example.com/me/shop.git","http_port":8081,"env":{"NODE_ENV":"production"},"deploy":true}' \
  "$API/projects"
```

An operator key may also create a compose or dockerfile project without a
`git_url`; an admin then sets the repository before it can deploy.

The answer (status 201) holds the new `project` and, with `deploy: true`, the
queued `run`. A compose or dockerfile project needs a `git_url` to use
`deploy: true`.

- `PATCH /api/v1/projects/{slug}` — change settings. Send only the fields you
  want to change: `name`, `mode`, `category`, `git_url`, `git_branch`,
  `compose_path`, `dockerfile_path`, `working_dir`, `image_ref`,
  `compose_inline`, `run_timeout_s` (seconds, at most 86400; `0` = server
  default). `git_url`, `compose_inline` and `category` need an admin key.
  Ports and domains are changed in the admin panel.

### Environment variables

Values are shown and stored in **cleartext** in the API (they are encrypted on
disk).

- `GET /api/v1/projects/{slug}/env` — all variables.
- `PUT /api/v1/projects/{slug}/env` — replace all variables:
  `{"env": {"KEY": "value"}}`.
- `PATCH /api/v1/projects/{slug}/env` — change some variables:
  `{"set": {"KEY": "value"}, "unset": ["OLD_KEY"]}`.

Names use letters, digits and `_` and don't start with a digit. Values can't
contain line breaks and can't be exactly `*****` (the admin panel uses that as
a placeholder).

If the values changed, ci-agent applies them to the running app (it recreates
the containers, or rebuilds a Dockerfile project). The answer's `applied` field
says what happened. Add `?apply=false` to only save them; the next deploy uses
them.

### Deploy

- `POST /api/v1/projects/{slug}/deploy` — deploy now. With no body, a git
  project deploys its saved git URL and branch, and an image/inline project
  deploys from its own config. Optional fields: `branch`, `force`, and (admin
  keys only) `git_url` or `zip_url` to deploy from a different source.
- `POST /api/v1/projects/{slug}/runs/{number}/redeploy` — deploy the saved files
  of an earlier run again (a rollback).
- `POST /api/v1/projects/{slug}/recreate` — recreate the containers from the
  last successful run, with the current env.
- `POST /api/v1/projects/{slug}/runs/{number}/cancel` — cancel a queued or
  running run.

Deploy calls answer **202** with the queued `run`. Poll
`GET /api/v1/projects/{slug}/runs/{number}` until `status` is `success`,
`failed` or `cancelled`. If deploys are paused for the project, they return 409
`deploys_paused`. Send `{"force": true}` to deploy anyway.

### Containers

- `POST /api/v1/projects/{slug}/containers/{name}/{op}` — `op` is `start`,
  `stop`, `restart`, `pause` or `unpause`. `name` is a container name from
  `/containers`.
- `POST /api/v1/projects/{slug}/pause-deploys` — `{"paused": true}` stops
  webhook deploys; `{"paused": false}` turns them back on.

## Admin endpoints (`admin` level)

- `POST /api/v1/projects/{slug}/archive` — `{"archived": true}` or `false`.
- `DELETE /api/v1/projects/{slug}` — deletes the project **and all its data**
  (containers, volumes, images). You must repeat the slug:
  `DELETE /api/v1/projects/shop?confirm=shop`. Not possible while a deploy is
  running.
- `POST /api/v1/projects/{slug}/exec` — run one command inside a container:

```json
{ "command": "php artisan migrate:status", "service": "web", "timeout_s": 30 }
```

  Pick the container with `service` or `container`. A project with only one
  container needs neither. `timeout_s` is 1–120 (default 30). The answer has
  `exit_code`, `stdout`, `stderr` (each cut at 64 KB) and `timed_out`. This is
  not an interactive shell, and only two commands can run at a time.
- `POST /api/v1/projects/{slug}/rotate-webhook` — make new webhook URLs. The old
  ones stop working.
- `GET /api/v1/audit` — the latest audit log entries (`?limit=100`).

## Old endpoints

Before API keys, ci-agent had one read-only token on the Settings page. On
upgrade, that token becomes a **read** key named `legacy`, so scripts that use
it keep working. (If you later go back to an older ci-agent version, create a
new token on its Settings page.) These paths still work with any key and return the same JSON
as before:

- `GET /api/health`
- `GET /api/projects`
- `GET /api/runs/{id}`

## Security notes

- Treat a key like a password. Don't commit it to git or paste it into logs.
- Operator and admin keys can read every environment variable of every
  project, including passwords.
- **Admin keys can take over the server.** They can deploy any compose file
  or repository (a compose file can ask for a privileged container or mount
  host folders) and run commands inside containers. Give admin keys only to
  agents you trust, with a short expiry.
- **Operator keys** can't choose what code runs, but they can still change
  environment variables, redeploy and stop containers, so a careless agent can
  break a running app. Use a **read** key when the agent only needs to look
  and debug.
- Container logs are not filtered, so a read key can see anything your app
  prints there.
- After 10 wrong keys in a minute, an address is blocked for a minute. All
  programs on the server itself share the address `127.0.0.1`, so one script
  with a wrong key can briefly block the others.
- More in [Security](./security.md).
