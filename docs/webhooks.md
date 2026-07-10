# Webhooks

A **webhook** is just a URL you send a small HTTP request to. When the agent
receives that request, it starts a deployment. This is how your git server (or
any script) tells the agent "new code is ready — deploy it".

Every project gets two webhook URLs, shown on its detail page:

```
GET|POST /hooks/<slug>/<token>/git
GET|POST /hooks/<slug>/<token>/zip
```

Both methods work. **POST** with a JSON body is the usual way (and what git
servers send). **GET** is there for convenience — you can paste the URL into a
browser or a `curl` with no body to trigger a default deploy, and you can pass
the same overrides as query parameters (see below). A GET carries no body, so
everything comes from the query string or the project's own configuration.

The `<token>` is a long random secret built into the URL. Rotate it any time
from the project page (the old URL stops working immediately).

> **Important:** that secret token is the **only** thing protecting the
> webhook. Anyone who has the full URL can trigger a deploy, so keep it
> private and rotate it if it ever leaks.

## Git endpoint

Body is optional JSON:

| Field | Meaning |
|---|---|
| `git_url` | override the project's configured repo for this run |
| `branch` | override the configured branch |
| `commit` | build an exact commit (fetched on top of the shallow clone); if it can't be fetched or checked out, the run **fails** rather than deploying branch `HEAD` |

Plain **GitHub / GitLab / Gitea push payloads are accepted as-is**: the
branch is read from `ref: "refs/heads/..."`, the commit from
`after`/`checkout_sha`, the repo URL from `repository.*` — anything missing
falls back to the project's configuration. Unknown fields are ignored.

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
top-level directory. Downloads are size-capped (`zip_max_bytes`) and the
optional `zip_url_allowlist` restricts which hosts may be fetched.

## Responses

| Code | Meaning |
|---|---|
| `202` | accepted; body `{"run_id": N}` |
| `404` | unknown project **or** bad token (indistinguishable on purpose) |
| `413` | request body too large (the webhook body cap is 1 MiB) |
| `422` | body present but unusable (bad JSON, missing `zip_url`, no git URL anywhere) |

## Queue behavior

Webhook-triggered runs **collapse**: if several pushes arrive while a build
is in progress, intermediate queued runs are marked `superseded` and only
the newest runs. Manual runs and redeploys never collapse.

## Paused and archived projects

A project with **deploys paused** (Settings → *Pause deploys*) or one that is
**archived** ignores webhook triggers — the running stack stays up, and manual
deploys still work on a paused project. Resume or unarchive to let webhooks
through again. See [Operations](./operations.md).
