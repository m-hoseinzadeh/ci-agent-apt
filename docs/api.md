# API & AI agents

ci-agent has an **HTTP API** for scripts and **AI agents** (Claude Code, Cursor,
your own bots). With an API key, an agent can look at your projects, read build
and container logs to find bugs, create and change projects, set environment
variables, and deploy. You decide how much each key can do.

## Quick start

1. In the admin panel, open **API keys** (in the sidebar).
2. Under **Create key**, fill in **Name** (for example `claude-ops`). A name
   has 1–40 characters: letters, digits, spaces, `_`, `.` or `-`. Each name
   must be unique.
3. Pick an **Access level** and when the key **Expires**, then click
   **Create key**.
4. Copy the key. **It is shown only once.** The page also shows a
   **Prompt for your AI agent**. Paste that prompt into your agent. It contains
   the key, the API address and links to these docs.
5. Test it:

```bash
curl -H 'Authorization: Bearer cia_…' https://ci.example.com/api/v1
```

The answer lists your key's level, every endpoint, whether your key may use
it, and where the docs are.

The **Connect an AI agent** card at the bottom of the page has the same prompt
with placeholders. Use it if you want to fill in the key yourself.

![screenshot: API keys page with the Keys table, the Create key form and the Connect an AI agent card](TODO)

### Revoke or delete a key

1. In the **Keys** table, click **Revoke** next to the key. Anything that uses
   it stops working at once.
2. Optional: click **Delete** to remove the key from the list. Its audit
   entries stay. **Delete** only shows for revoked or expired keys, so an
   active key must be revoked first.

## For AI agents: how to start

If you are an AI agent and a human gave you a ci-agent key, do this:

1. Call `GET /api/v1`. Read `key.level` and the `endpoints` list. Only call
   endpoints where `allowed` is `true`.
2. Read this guide as Markdown at `GET /api/v1/docs` (needs the key) or on the
   public site: https://m-hoseinzadeh.github.io/ci-agent-apt/docs/api.html
3. Deploys are **asynchronous**. A deploy call returns a queued run right away.
   Poll the run until its `status` is no longer `queued`, `fetching`,
   `building`, `testing` or `deploying`. The final status is `success`,
   `failed`, `timeout`, `interrupted` or `cancelled`.
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
| `admin` | Everything in `operator`, plus: choose git repositories, zip URLs, inline compose files and categories; archive and delete projects; run commands inside containers; manage domains, certificates, default URLs and proxy tuning; read the audit log; rotate webhook tokens. |

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
| 404 | No such project, run, container or endpoint (`not_found`). |
| 409 | Conflict: `slug_taken`, `deploys_paused`, `run_active`, `nothing_deployed`, `no_containers`, or `conflict` (for example a service name that clashes with another project in the same category). |
| 422 | The request is invalid (`invalid_request`); the message says what to fix. |
| 429 | `too_many_failures`: too many wrong keys from your address. `exec_busy`: another `exec` is still running. The answer has a `Retry-After: 60` header. |
| 500 | `internal`: something failed on the server. Details are in the ci-agent log. |
| 504 | `timeout`: reading container logs took too long. Lower `tail` or set `since`. |

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
  services, domains, watch paths, health and last run. Operator and admin keys also get
  `compose_inline`, `status_callback_url` and `status_callback_token_set`
  (`true`/`false` — the token itself is never returned).

### Runs and build logs

- `GET /api/v1/projects/{slug}/runs` — recent runs (`?limit=20`, at most 200).
- `GET /api/v1/projects/{slug}/runs/{number}` — one run. Fields:
  - `id` (global) and `number` (per project), `project` (the slug)
  - `status`, `trigger`, `source_ref` (what was deployed), `commit_sha`,
    `commit_message`, `image_tags`
  - `queued_at`, `started_at`, `finished_at`, `error`
  - `callback`: `requested` (whether this run reports to the status
    callback), `state` (`pending`, `delivered`, `failed`, `skipped`, or `null`
    while the run is still going) and `detail` (for example `HTTP 200`)
  - `links`: `self` and `log`, the API paths of this run and its log
- `GET /api/v1/projects/{slug}/runs/{number}/log` — the end of the run's build
  and deploy log (`?tail=500` lines, at most 5000). The answer has the `run`,
  its `lines` and `truncated`. Only the last 512 KB of the log file is read.
  A run with no log (not started yet, or removed by log retention) gives 404.
- `GET /api/v1/runs` — recent runs of all projects (`?limit=50`, at most 500).
  With `?active=true` you get only the queued and running runs, and `limit` is
  ignored.
- `GET /api/v1/runs/{id}` — one run by its global id.
- `GET /api/v1/projects/{slug}/domains` — the project's domains: `id`,
  `domain`, `canonical`, `tls`, `port_id` (`null` = the primary HTTP port),
  `maintenance` and `certificate`: the stored certificate that covers the
  domain (`name`, `source`, `wildcard`, `challenge`, `not_after`), or `null`.
- `GET /api/v1/projects/{slug}/ports` — the project's ports: the primary HTTP
  port (`"id": "primary"`) and each extra port, with `slug`, `host_port`,
  `container_port`, `label`, `service`, its default URLs (`subdomain`, `path`:
  on or off) and its proxy tuning (`nginx`: `max_body_mb`, `proxy_timeout_s`,
  `websockets`, `extra`). A read key sees `extra_set` (`true`/`false`) instead
  of the `extra` directives.

### Containers and app logs

- `GET /api/v1/projects/{slug}/containers` — the project's containers with
  service name, state, CPU and memory.
- `GET /api/v1/projects/{slug}/logs` — the end of the containers' logs.
  Options: `?service=web` (one service), `?tail=200` (at most 2000),
  `?since=10m` (`30s`, `10m`, `2h`, `1d`). The answer has `lines`,
  `truncated` and `stderr`. Output is cut at 1 MB. If reading takes more than
  15 seconds you get 504 `timeout`; lower `tail` or set `since`.

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
| `working_dir` | no | sub-folder of the repo to build in (monorepos); compose and dockerfile modes only |
| `image_ref` | image mode | e.g. `nginx:1.27` |
| `compose_inline` | inline mode | the full docker-compose.yml text; **admin only** |
| `http_port` | no | host port |
| `container_port` | no | app port inside the container; dockerfile and image modes only (ignored for compose and inline) |
| `category` | no | `"a,b"` or `["a","b"]` — shared Docker networks; **admin only** |
| `env` | no | `{"KEY": "value"}` |
| `status_callback_url`, `status_callback_token` | no | where to send this project's [status callback](./webhooks.md) reports; set before the first deploy, so a `deploy: true` run is reported too; **admin only** |
| `watch_paths` | no | `["services/wallet/**", "Dockerfile"]` (or one pattern per line): a git push deploys only when it changed a matching file. See [Webhooks](./webhooks.md) |
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
  default), `status_callback_url` and `status_callback_token` (an empty string
  clears them), `watch_paths` (`[]` clears them: every push deploys again). `git_url`, `compose_inline`, `category` and the two
  `status_callback_*` fields need an admin key. Ports are added and removed
  in the admin panel; their default URLs and proxy tuning change with the
  [port endpoints](#ports), and domains with the [domain endpoints](#domains). The answer holds the updated `project` and a `note`: a
  short text that starts with `Saved` and says what else happened.

  Example — report this project's API and webhook deploys to your CD system:

  ```bash
  curl -X PATCH -H "Authorization: Bearer $KEY" \
    -H 'Content-Type: application/json' \
    -d '{"status_callback_url":"https://cd.example.internal/ci-agent/status",
         "status_callback_token":"<shared secret>"}' \
    "$API/projects/my-app"
  ```

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
is a short text that says what happened (or `null`). Add `?apply=false` to only
save them; the next deploy uses them.

The run that applies the change is **not** reported to the status callback,
and the answer has no run id. To follow it, list the runs with
`GET /api/v1/projects/{slug}/runs?limit=1`.

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
`GET /api/v1/projects/{slug}/runs/{number}` until `status` is no longer
`queued`, `fetching`, `building`, `testing` or `deploying`.

If deploys are paused for the project, deploy, redeploy and recreate return
409 `deploys_paused`. Send `{"force": true}` to deploy anyway.

API deploys never replace each other in the queue. Only webhook runs do that
(see [Webhooks](./webhooks.md)).

Instead of polling, set a **status callback** on the project. Then these runs
POST each status change to your URL, with `run_id` = the run's `id`:

- runs started by deploy, redeploy and recreate, and by create with
  `deploy: true`;
- runs started by the project's webhook URLs.

Runs started by an env change (`PUT`/`PATCH .../env`), from the admin panel,
or by auto-redeploy are **not** reported. See
[Webhooks → Status callback](/docs/webhooks#status-callback).

### Containers

- `POST /api/v1/projects/{slug}/containers/{name}/{op}` — `op` is `start`,
  `stop`, `restart`, `pause` or `unpause`. `name` is a container name from
  `/containers`.
- `POST /api/v1/projects/{slug}/pause-deploys` — `{"paused": true}` stops
  webhook deploys, and API deploys then need `"force": true`;
  `{"paused": false}` turns them back on.

## Admin endpoints (`admin` level)

- `POST /api/v1/projects/{slug}/archive` — `{"archived": true}` or `false`.
- `DELETE /api/v1/projects/{slug}` — deletes the project **and all its data**
  (containers, volumes, images). You must repeat the slug:
  `DELETE /api/v1/projects/shop?confirm=shop`. Not possible while a deploy is
  running (409 `run_active`). The answer is `{"deleted": true, "had_errors": …}`.
  `had_errors` is `true` when some cleanup step failed; check the ci-agent
  log.
- `POST /api/v1/projects/{slug}/exec` — run one command inside a container:

```json
{ "command": "php artisan migrate:status", "service": "web", "timeout_s": 30 }
```

  Pick the container with `service` or `container`. A project with only one
  container needs neither. `timeout_s` is 1–120 (default 30). The answer has
  `container`, `exit_code`, `ok` (`true` when the exit code is 0), `stdout`
  and `stderr` (each cut at 64 KB), `stdout_truncated`, `stderr_truncated` and
  `timed_out`. This is not an interactive shell. Only two commands can run at
  a time; a third gets 429 `exec_busy`.
- `POST /api/v1/projects/{slug}/domains` — add a domain (see [Domains](#domains)).
- `PATCH /api/v1/projects/{slug}/domains/{id}` — turn HTTPS on or off.
- `DELETE /api/v1/projects/{slug}/domains/{id}` — delete a domain.
- `POST /api/v1/projects/{slug}/domains/{id}/certificate` — get a certificate
  for a domain (see [Domains](#domains)).
- `PATCH /api/v1/projects/{slug}/ports/{port}` — change a port's default URLs
  or proxy tuning (see [Ports](#ports)).
- `POST /api/v1/projects/{slug}/rotate-webhook` — make new webhook URLs. The old
  ones stop working. The answer has the new URLs in `webhook_git` and
  `webhook_zip`. The `webhook_zip` URL also takes a ZIP file as the POST
  body (see [Webhooks](/docs/webhooks#upload-the-zip)).
- `GET /api/v1/audit` — the latest audit log entries (`?limit=100`).

### Domains

A domain decides which hostnames nginx sends to a project, so adding and
deleting need an **admin** key. They do the same checks as the **Domains**
tab and apply nginx right away:

- `POST /api/v1/projects/{slug}/domains` with `{"domain": "app.example.com"}`.
  Add `"port_id": <id>` to bind it to one of the project's extra ports instead
  of its primary HTTP port. The hostname must be a bare, valid name; it may not
  be the admin panel's own domain, another project's domain (409 `conflict`)
  or a host routed to a tunnel. The new domain is not primary, so it serves the
  app directly. Add `"tls": true` to serve it over HTTPS at once; that needs a
  certificate that already covers the hostname (a wildcard for its zone or one
  named for it), otherwise the answer is 422 and nothing is added. The answer
  (status 201) has the new `domain` and `nginx`, a short text that says how the
  nginx update went.
- `PATCH /api/v1/projects/{slug}/domains/{id}` with `{"tls": true}` or
  `{"tls": false}` — switch HTTPS for an existing domain, with the same
  certificate check. The answer has the updated `domain` and `nginx`.
- `DELETE /api/v1/projects/{slug}/domains/{id}` — the `id` comes from
  `GET .../domains`. The answer has `deleted`, `was_canonical` (redirects to it
  are now off) and `nginx`.
- `POST /api/v1/projects/{slug}/domains/{id}/certificate` — get a certificate
  for the domain, as the **Issue a certificate** card on the domain's SSL page
  does, then switch HTTPS on. Fields (all optional):

  | Field | Meaning |
  |---|---|
  | `source` | `letsencrypt` (default), `internal_ca`, `self_signed` or `upload` |
  | `scope` | `single` or `wildcard`, for the other sources. Blank: the zone's apex gets a wildcard, a subdomain a cert of its own. Let's Encrypt always makes a cert for this one host (HTTP-01). |
  | `tls` | `false` stores the certificate but leaves HTTPS off; switch it on later with `PATCH .../domains/{id}` |
  | `cert_pem`, `key_pem` | for `upload`: the full chain and the private key |

  Let's Encrypt checks the domain over plain HTTP on port 80, so its DNS must
  already point at this server (directly or through a CDN that forwards
  `/.well-known/acme-challenge/`). The server itself also needs outbound HTTPS
  to Let's Encrypt. The call waits for Let's Encrypt, usually 5–30 seconds.
  The answer (status 201) has the updated `domain`, the `certificate` (`name`,
  `wildcard`, `not_after`) and `nginx`. If Let's Encrypt refuses, the answer is
  422 with its reason, and nothing changes. The same 422 comes after 15 seconds
  if the server can't reach Let's Encrypt at all ("could not reach Let's
  Encrypt"), or after 50 seconds if it doesn't finish, which is before nginx or
  a CDN in front of the panel gives up at 60.

  **Behind a CDN:** with HTTPS on, nginx redirects plain HTTP to HTTPS. If the
  CDN reaches this server over plain HTTP, that becomes a redirect loop. Send
  `"tls": false`, switch the CDN to HTTPS towards the server, then turn HTTPS
  on with `PATCH .../domains/{id}`.

The primary domain, wildcard certificates from Let's Encrypt (DNS-01) and the
Let's Encrypt account settings are still managed in the admin panel.

```bash
curl -X POST -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"domain":"shop.example.com"}' "$API/projects/shop/domains"
curl -X POST -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"source":"letsencrypt"}' "$API/projects/shop/domains/12/certificate"
```

### Ports

Each port has default URLs and proxy tuning, as on the **Ports** card of the
**Settings** tab and the **Proxy tuning** panel of the **Domains** tab. They
change nginx, so they need an **admin** key.

- `PATCH /api/v1/projects/{slug}/ports/{port}` — `{port}` is `primary` (the
  project's HTTP port) or an extra port's `id` from `GET .../ports`. Send only
  what you want to change:

  | Field | Meaning |
  |---|---|
  | `subdomain` | `true`/`false`: the default subdomain `<slug>.<base domain>` |
  | `path` | `true`/`false`: the default path `<base domain>/<slug>/`. Turning it on is refused when another project uses that path (409 `conflict`), when the port has no HTTP port, or in single-domain mode without a base path (422). |
  | `nginx` | proxy tuning: `max_body_mb` (largest upload in MB), `proxy_timeout_s` (1–86400), `websockets` (`true` forwards WebSocket and SSE upgrades), `extra` (raw lines added to the proxy `location` block). Fields you leave out stay as they are; `null` goes back to nginx's default. |

  Everything is checked before anything changes. The answer has the updated
  `port` (as in `GET .../ports`) and `nginx`, a short text that says how the
  nginx update went. As in the panel, the new values are saved even when
  nginx refuses them (for example a typo in `extra`): the note then starts with
  `nginx apply failed`, nginx keeps its previous config, and you can send a
  fix.

```bash
curl -X PATCH -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"subdomain":false,"nginx":{"websockets":true}}' "$API/projects/shop/ports/primary"
```

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
- `status_callback_url` decides where the agent sends HTTP requests, so only
  admin keys can set it.
- A project's **webhook URL** is as powerful as an admin key for that project.
  Whoever has it can pass their own `git_url` or `zip_url` and deploy any code.
  Keep it as secret as an admin key, and rotate it if it leaks.
- After 10 wrong keys in a minute, an address is blocked for a minute. All
  programs on the server itself share the address `127.0.0.1`, so one script
  with a wrong key can briefly block the others.
- More in [Security](./security.md).
