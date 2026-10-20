# Webhooks

A **webhook** is just a URL you send a small HTTP request to. When the agent
receives that request, it starts a deployment. This is how your git server (or
any script) tells the agent "new code is ready — deploy it".

Every project gets two webhook URLs. You find them on the project's
**Deployments** tab, in the **Webhooks** card:

```
GET|POST /hooks/<slug>/<token>/git
GET|POST /hooks/<slug>/<token>/zip
```

Both methods work. **POST** with a JSON body is the usual way (and what git
servers send). **GET** is there for convenience — you can paste the URL into a
browser or a `curl` with no body to trigger a default deploy, and you can pass
the same overrides as query parameters (see below). A GET carries no body, so
everything comes from the query string or the project's own configuration.

The `<token>` is a long random secret built into the URL. To change it, click
**Rotate token** in the same **Webhooks** card. The old URL stops working
immediately.

> **Important:** that secret token is the **only** thing protecting the
> webhook. Anyone who has the full URL can trigger a deploy. They can also
> send their own `git_url` or `zip_url` and deploy **any code** they like.
> So treat the URL like an admin password: keep it private and rotate it if
> it ever leaks.

## Git endpoint

Body is optional JSON:

| Field | Meaning |
|---|---|
| `git_url` | override the project's configured repo for this run |
| `branch` | override the configured branch |
| `commit` | build an exact commit (fetched on top of the shallow clone); if it can't be fetched or checked out, the run **fails** rather than deploying branch `HEAD` |

Plain **GitHub / GitLab / Gitea push payloads are accepted as-is**: the
commit is read from `after`/`checkout_sha`; anything missing falls back to the
project's configuration. Unknown fields are ignored. Point the git server's
webhook at the `/git` URL with content type **`application/json`** (a
form-encoded body is rejected with `422`).

A git server fires its webhook for more than the deploys you want, so the
agent **filters** what it receives — each check runs only when its signal is
present, so a plain `curl` or script call is never filtered:

- **Event:** if the request has an `X-GitHub-Event`, `X-Gitea-Event`,
  `X-Gogs-Event` or `X-Gitlab-Event` header, only a push (`push` /
  `Push Hook`) deploys. Anything else — including the `ping` GitHub sends when
  you create the webhook — is skipped.
- **Branch:** if the body has a `ref`, it must be `refs/heads/<branch>` where
  `<branch>` is the project's configured branch (or the `branch` override).
  Pushes to other branches and tag pushes are skipped.

A skipped hit still answers `202`, with `{"skipped": "<reason>"}`, and shows on
the **Audit** page. The payload's `repository.*` URL is used only when the
project has no repo configured — a push to a mirror never redirects which repo
the project builds from. When it is used, the HTTPS form (`clone_url` /
`git_http_url`) is preferred, and an SSH form (`git@host:owner/repo.git` or
`ssh://…`) is rewritten to `https://` — the agent clones over HTTPS only (see
[Git access](./projects.md)).

After first-run setup the agent listens only on `127.0.0.1`, so call webhooks
through your **admin domain** (the same nginx site that serves the panel), not
the raw `:8044` port:

```bash
# simplest: deploy the configured repo/branch
curl -X POST https://ci.example.internal/hooks/my-app/<token>/git

# specific branch (POST body)
curl -X POST https://ci.example.internal/hooks/my-app/<token>/git \
  -H 'content-type: application/json' -d '{"branch":"release"}'

# same thing with a plain GET — overrides go in the query string
curl 'https://ci.example.internal/hooks/my-app/<token>/git?branch=release'
```

For a GET, pass `git_url`, `branch`, and/or `commit` as query parameters; each
falls back to the project's configuration when omitted. A POST body always wins
over a query parameter for the same field.

## ZIP endpoint

```bash
# POST body
curl -X POST https://ci.example.internal/hooks/my-app/<token>/zip \
  -H 'content-type: application/json' \
  -d '{"zip_url":"http://artifacts.internal/app-1.2.3.zip"}'

# GET with the URL as a query parameter
curl 'https://ci.example.internal/hooks/my-app/<token>/zip?zip_url=http://artifacts.internal/app-1.2.3.zip'
```

The ZIP endpoint always needs a `zip_url` — from the POST body or the
`?zip_url=` query parameter. Without one it returns `422`.

The archive may have the compose file at its root or inside a single
top-level directory.

Limits and safety checks:

- **Size:** the download may be at most `zip_max_bytes` (default 500 MB). The
  unpacked files may be at most twice that, and the archive may hold at most
  50,000 entries.
- **Time:** the download must finish within 10 minutes.
- **Allowed URLs:** if you set `zip_url_allowlist` in the
  [Configuration](./configuration.md), the URL must start with one of its
  entries (URL prefixes, for example `https://artifacts.internal/`). An empty
  list allows any URL.
- **Blocked hosts:** the agent refuses to download from loopback
  (`127.0.0.1`, `localhost`, `::1`) and link-local (`169.254.*`, `fe80::`)
  addresses, also when a host name resolves to one. Serve the ZIP from another
  address, for example the server's LAN IP.
- **Unsafe files:** an archive with symlinks, absolute paths or `..` in a path
  is rejected.

> **What this means.** The ZIP URL comes from whoever calls the webhook. These
> checks stop a caller from using the agent to reach services that listen
> only on the server itself, or to fill the disk.

## Responses

| Code | Meaning |
|---|---|
| `202` | accepted; body `{"run_id": N}` — or `{"skipped": "<reason>"}` when the event/branch filter (or a deploy pause) dropped it |
| `404` | unknown project, bad token, **or** archived project (all look the same on purpose) |
| `413` | request body too large (the webhook body cap is 1 MiB) |
| `422` | body present but unusable (bad JSON, missing `zip_url`, no git URL anywhere) |
| `500` | the agent could not queue the run; see the ci-agent log |

`run_id` is the run's **global id** — unique across all projects. It is not the
per-project `#number` shown in the run list (both appear on the run's page, as
**Run ID** and in the title). Keep `run_id` if you want status updates: the
[status callback](#status-callback) sends it back with every report.

## Status callback

A webhook answers right away with a `run_id`, but the deploy takes longer. If
another system started the deploy (a CD tool, a release script), it usually
wants to know how it ended. Instead of polling, give the project a **status
callback URL**: the agent then sends an HTTP `POST` to that URL every time the
run changes status.

> **What this means.** You call the agent to start a deploy; the agent calls
> *you* back to say how it is going. The `run_id` in each report is the same
> number the webhook returned, so you can match reports to your requests.

### Which runs are reported

Only runs that a caller **requested and got a `run_id` for**:

- runs started by the `/git` or `/zip` webhook URL, and
- runs started through the [API](./api.md) (deploy, redeploy, recreate, or
  create with `deploy: true`).

These runs are **not** reported — nobody outside is waiting for them:

- runs you start from the admin panel;
- runs started by **Auto-redeploy every (hours)**;
- runs started by an env change, in the panel or through the API
  (`PUT`/`PATCH /api/v1/projects/{slug}/env`).

### Set it up

1. Open the project and go to the **Settings** tab.
2. Below **Auto-redeploy every (hours)**, fill in **Status callback URL** — for example
   `https://cd.example.internal/ci-agent/status`. It must start with `http://`
   or `https://`.
3. Optional: fill in **Callback token**. The agent sends it in the
   `X-CiAgent-Token` header, so your receiver can check the request really
   came from this server.
4. Click **Save callback**.

To turn it off, clear the URL and save. The token is stored encrypted and is
shown in clear text on the Settings tab, like env values.

![screenshot: Settings tab with the Status callback URL and Callback token fields](TODO)

### What the agent sends

```bash
curl -X POST "<your callback URL>" \
  -H "Content-Type: application/json" \
  -H "X-CiAgent-Token: <your token>" \
  -d '{"status": "failed", "run_id": 123, "error": "build failed: …"}'
```

A successful run sends:

```json
{ "status": "success", "run_id": 123 }
```

| Field | Meaning |
|---|---|
| `status` | the run's new status (table below) |
| `run_id` | the `run_id` the webhook or API returned |
| `error` | only for `failed`, `interrupted` and `timeout`: why it stopped. Secrets (project and shared env values, git credentials) are replaced with `*****`; long text is cut at 2000 characters |

The header is left out when no token is set.

**Statuses, in the order they happen:**

| Status | Kind | Meaning |
|---|---|---|
| `queued` | in progress | the run is waiting its turn |
| `fetching` | in progress | downloading the code (git clone or ZIP) |
| `building` | in progress | building images |
| `testing` | in progress | running the `ci-test` service (only if the project has one) |
| `deploying` | in progress | starting the new containers and checking health |
| `success` | final | deployed and healthy |
| `failed` | final | something went wrong; see `error` |
| `timeout` | final | the run hit its time limit |
| `interrupted` | final | the agent restarted while the run was in progress. Sent after the restart; `error` is `ci-agent restarted while the run was in progress` |
| `cancelled` | final | someone cancelled it |
| `superseded` | final | a newer webhook deploy replaced it before it ran (see [Queue behavior](#queue-behavior)) |

A redeploy or recreate skips `fetching` and `building`. Each run reports
`queued` first and one final status last (a final status can arrive twice —
see [Delivery and retries](#delivery-and-retries)).

> **Restart while a run is still queued.** Some runs have not started yet when
> the agent restarts. Webhook runs and API deploys from git or a ZIP URL can't
> be resumed, because the agent did not save their source. They end as
> `failed` with the error `lost at restart`. Call the webhook (or the API)
> again to retry. Queued redeploys, recreates and image/inline deploys are put
> back in the queue and still run.

### Delivery and retries

- Reports for one run arrive **in order**. Different runs are sent
  independently, so a slow receiver never holds up other runs.
- Any `2xx` answer counts as delivered. The agent ignores the response body.
- **In-progress** reports are tried **once** (10-second timeout).
- The **final** report is retried when the receiver can't be reached or
  answers `408`, `429` or `5xx`: up to 5 tries, waiting 5 s, 30 s, 2 min and
  10 min between them. Any other answer (for example `404` for a run your
  receiver doesn't know) is not retried.
- If the agent stops before the final report is confirmed, it sends it again
  after it starts. So your receiver may see the **same final report twice** —
  treat repeats as "already recorded".
- The run's page has a **Status callback** row (only for reported runs). It
  shows:

  | Value | Meaning |
  |---|---|
  | `sending progress updates` | the run is still going |
  | `pending` | the run ended; the final report is being sent |
  | `delivered (HTTP 200)` | your receiver accepted the final report |
  | `failed (…)` | the final report did not arrive; the brackets show the HTTP code or error, for example `failed (HTTP 404)` or `failed (cannot decrypt callback token)` |
  | `skipped (no status callback URL set)` | the project had no callback URL when the run ended |
  | `not sent` | the run ended but no report was recorded |

> **Note.** A webhook that answers `{"skipped": …}` (filtered push, or deploys
> paused) created **no run**, so no callback follows. Treat a `202` without
> `run_id` as "not deployed".

### Common problems and fixes

| Problem | Fix |
|---|---|
| No reports arrive | Check that the run was started by a webhook or the API (panel deploys, auto-redeploys and env changes are not reported), and that the **Status callback URL** is set on that project |
| The run page says `failed (…)` | The detail shows the HTTP code or connection error. Make sure the agent's server can reach the URL. A `4xx` other than 408/429 is not retried — fix the receiver and redeploy |
| Saving the URL says the host "is not allowed" | Link-local addresses (`169.254.*`, `fe80::`), `0.0.0.0`, `::` and `255.255.255.255` are blocked. Loopback (`127.0.0.1`) and LAN addresses are allowed |
| The receiver gets the same final status twice | Expected after a retry or an agent restart. Ignore repeats |
| The webhook answers `404` but the URL is right | The project is archived. Unarchive it on the **Settings** tab |
| A ZIP deploy fails with "disallowed loopback/link-local address" | The `zip_url` points at this server itself (`127.0.0.1`, `localhost`). Use the server's LAN address or another host |

## Queue behavior

Webhook runs **collapse**. When a new webhook run is queued, every older
webhook run of the same project that is still waiting in the queue is marked
`superseded`. Only the newest one runs. A run that has already started is not
touched.

Runs from the admin panel and the API (deploys, redeploys, recreates) never
collapse, and they never replace a webhook run.

> **What this means.** If you push five commits quickly, the agent builds the
> one that is running now and then only the newest commit. It does not build
> the three in between.

## Paused and archived projects

**Paused.** On the **Settings** tab, click **Pause deploys** to freeze a
project. The running stack stays up. Then:

- webhook calls answer `202` with `{"skipped": "deploys paused"}` and start
  no run;
- deploys from the admin panel still work;
- API deploy, redeploy and recreate answer `409` with the code
  `deploys_paused`, unless you send `"force": true`.

Click **Resume deploys** to let webhooks through again.

**Archived.** A webhook call to an archived project answers `404`, the same
as a wrong token. Click **Unarchive** on the **Settings** tab to turn the
webhook back on.

See [Operations](./operations.md).
