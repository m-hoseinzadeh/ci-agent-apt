# Webhooks

Every project gets two endpoints, shown on its detail page:

```
POST /hooks/<slug>/<token>/git
POST /hooks/<slug>/<token>/zip
```

The token is a 256-bit random secret embedded in the URL. Rotate it any time
from the project page (the old URL stops working immediately).

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
| `401` | signature required and missing/invalid |
| `422` | body present but unusable (bad JSON, missing `zip_url`, no git URL anywhere) |

## Signatures (optional hardening)

Set an HMAC secret on the project, then either:

- **GitHub style** — header `X-Hub-Signature-256: sha256=<hex>`, the HMAC of
  the raw request body (GitHub sends this automatically when you configure
  the same secret), or
- **GitLab style** — header `X-Gitlab-Token: <secret>` (plain match).

When a secret is configured, unsigned requests are rejected even with a
valid URL token.

## Queue behavior

Webhook-triggered runs **collapse**: if several pushes arrive while a build
is in progress, intermediate queued runs are marked `superseded` and only
the newest runs. Manual runs and redeploys never collapse.
