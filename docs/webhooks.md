# Webhooks

A **webhook** is just a URL you send a small HTTP request to. When the agent
receives that request, it starts a deployment. This is how your git server (or
any script) tells the agent "new code is ready — deploy it".

Every project gets two webhook URLs, shown on its detail page:

```
POST /hooks/<slug>/<token>/git
POST /hooks/<slug>/<token>/zip
```

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
| `commit` | build an exact commit (fetched on top of the shallow clone) |

Plain **GitHub / GitLab / Gitea push payloads are accepted as-is**: the
branch is read from `ref: "refs/heads/..."`, the commit from
`after`/`checkout_sha`, the repo URL from `repository.*` — anything missing
falls back to the project's configuration. Unknown fields are ignored.

```bash
# simplest: deploy the configured repo/branch
curl -X POST http://ci-host:8044/hooks/my-app/<token>/git

# specific branch
curl -X POST http://ci-host:8044/hooks/my-app/<token>/git \
  -H 'content-type: application/json' -d '{"branch":"release"}'
```

## ZIP endpoint

```bash
curl -X POST http://ci-host:8044/hooks/my-app/<token>/zip \
  -H 'content-type: application/json' \
  -d '{"zip_url":"http://artifacts.internal/app-1.2.3.zip"}'
```

The archive may have the compose file at its root or inside a single
top-level directory. Downloads are size-capped (`zip_max_bytes`) and the
optional `zip_url_allowlist` restricts which hosts may be fetched.

## Responses

| Code | Meaning |
|---|---|
| `202` | accepted; body `{"run_id": N}` |
| `404` | unknown project **or** bad token (indistinguishable on purpose) |
| `422` | body present but unusable (bad JSON, missing `zip_url`, no git URL anywhere) |

## Queue behavior

Webhook-triggered runs **collapse**: if several pushes arrive while a build
is in progress, intermediate queued runs are marked `superseded` and only
the newest runs. Manual runs and redeploys never collapse.
