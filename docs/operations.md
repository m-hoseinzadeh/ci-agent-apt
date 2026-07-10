# Operations

## Run lifecycle

```
queued → fetching → (testing) → building → deploying → success
                                              ↘ failed (auto-rollback attempted)
other terminal states: cancelled, superseded, interrupted, timeout
```

- **Pre-flight**: a run refuses to start below `min_free_disk_bytes` free.
- **Timeout**: each run has a hard limit (`run_timeout_secs` or per-project
  override). On expiry the whole process tree is killed.
- **Cancel**: from the run page; TERM, then KILL after 10 s.
- **Build niceness**: heavy subprocesses run under `nice`/`ionice` so the
  agent doesn't starve the apps it hosts.

## Deploys, health gate, rollback

`docker compose up -d --remove-orphans` is followed by a health gate
(`deploy_healthcheck_secs`): any exited container or failing healthcheck
fails the run, and the agent automatically redeploys the **last successful
run's snapshot** (clearly marked in the log). The brief stop-start window of
the `recreate` strategy is an accepted trade-off on small servers.

> **Note:** "rolling back" here means putting the previous working version
> back, automatically, so a bad deploy doesn't leave your app down. A
> "snapshot" is the exact compose file and image versions the agent saved for
> each successful run.

## Redeploying old versions

Every successful run snapshots its exact compose file + the generated
override (pinned image tags) under `runs/<id>/`. *Redeploy* re-runs `up -d`
from that snapshot — no fetch, no build — after first **pulling the latest
image** for any registry-backed service (a failed pull is ignored on an
air-gapped box). It works as long as the run's images still exist, which the
cleanup keeps for the last few successful runs of each project — set by
**Redeployable runs / project** in Settings → Maintenance (default **1**, so by
default only the version that's live now is redeployable; raise it to keep more).

*Recreate* (on the project's Containers card) is related but different: it
rebuilds the containers from the last successful snapshot using the project's
**current env vars and volumes**, so it's how you apply env/volume edits without
a full rebuild. The agent also recreates automatically when you save changed env
vars or volumes.

## Queueing

Serial per project; at most `max_concurrent_runs` executing overall; rapid
webhook pushes collapse to the newest (`superseded`). Queued git/ZIP runs do
not survive a restart (they are failed as "lost at restart"); queued
redeploys are re-queued.

## Pausing deploys (change freeze)

Each project's **Settings** tab has a **Pause deploys** button. While a project
is paused, **webhook-triggered** deploys are suppressed and a **deploys paused**
chip is shown — but the running stack stays up, and **manual** deploys and
**redeploys** still work. Use it during a change freeze or an incident, then
**Resume deploys** to let webhooks through again. Scheduled auto-redeploys (below)
are also suppressed while paused.

## Scheduled auto-redeploy

Each project's **Settings** tab has an **Auto-redeploy every (hours)** field
(0–168, `0` = off). When set, the agent re-runs the project's **last successful
deploy** on that interval. It's handy for refreshing an image pinned to a moving
tag (e.g. `:latest`) and for giving a flaky stack a periodic recreate. The first
run lands one full interval after you save, and scheduled redeploys are skipped
while deploys are paused.

## Health monitoring & down-alerts

Separately from the deploy-time health gate, the agent **continuously monitors**
every non-archived project in the background (about every 90 seconds), using the
same checks the project's **Health** card shows. If a project falls into a
**problem** state and stays there (a short debounce avoids alerting on a brief
blip), the agent fires the **notify hook** *and* sends an **SMTP alert** — and
sends a **recovery** notice when the project comes back. A mere *degraded*
(warning) state does not alert.

This catches a container that crash-loops or is OOM-killed **between** deploys —
something the deploy-time gate can't see. It uses the same SMTP settings as
[email alerts](#email-alerts-on-failed-deployments) below; set those up to
receive the emails. The last-alerted state is remembered across restarts, so a
restart neither re-alerts nor loses a pending recovery.

## Automation API

A small **read-only** HTTP API lets an internal monitoring or cron box poll
status without a browser session or 2FA. Generate a token on the **Settings**
page (**Automation API token → Generate token**) and send it as a bearer header:

```bash
curl -H 'Authorization: Bearer <token>' https://ci.example.internal/api/health
```

| Endpoint | Returns |
|---|---|
| `GET /api/health` | each project's health `status` (`healthy` / `degraded` / `problem`) |
| `GET /api/projects` | the list of projects |
| `GET /api/runs/{id}` | one run's status and metadata |

The API is **disabled until a token is set**, and the token is stored encrypted
at rest. **Triggering** deploys stays on the per-project
[webhook tokens](./webhooks.md) — the API only reads. Revoke or regenerate the
token any time on the Settings page.

## Housekeeping (nightly)

1. SQLite backup via `VACUUM INTO` → `data_dir/backups/` (newest 7 kept).
2. Run-log retention (`log_retention_per_project`).
3. Auto-prune (Settings toggle, **on by default**): removes each project's old
   images beyond the keep count (**Redeployable runs / project**, default 1),
   plus dangling images and build cache. Protected images and images a running
   container uses are never touched. This same cleanup also runs after every
   successful deploy, not only nightly (see *Disk cleanup* below).
4. Low-disk check → `disk_low` notification.
5. Certificate renewal — auto-renewable certs (internal CA, self-signed,
   Let's Encrypt HTTP-01) within 30 days of expiry are re-issued and nginx is
   re-applied. Uploaded certs and LE DNS-01 wildcards are manual. See
   [Domains & TLS](./domains.md).

It also sweeps stale scratch files (e.g. an abandoned staged update) after an
hour.

## Disk cleanup

Every deploy builds a new Docker image and tags it per run. These images can be
large, so old ones must be removed or they fill the disk.

With **auto-prune** on (the default — Settings → Maintenance), cleanup happens
automatically at two times:

- **After every successful deploy** — the agent removes that project's now-old
  images, plus dangling images and build cache.
- **Nightly** — the same cleanup runs across all projects.

**What is kept.** The last *N* successful runs of each project stay redeployable
and keep their image, where *N* is the **Redeployable runs / project** setting
(default **1**, range 1–50). Project-data volumes are always kept too.

**What is safe.** The agent never force-removes an image, so an image a running
container still uses is never deleted, even if the keep count is set low.

Turn auto-prune **off** if you prefer to clean up by hand on the **Storage**
page, where each cleanup button shows how much space it will free. See the
[Admin UI guide](./ui.md) for the buttons.

> **Upgrading from an older version?** Auto-prune used to be off by default and
> only cleaned dangling images and build cache. After this update it is turned
> on automatically (unless you had explicitly turned it off), and it now also
> removes old per-project images. Old images that piled up before are cleared on
> the next nightly pass or the next deploy.

## Notifications

Set `notify_hook` to any executable; it receives one JSON argument:

```json
{"event":"run_finished","run_id":42,"project":"my-app","status":"success",
 "trigger":"webhook_git","duration_s":73,"error":null}
{"event":"disk_low","free_bytes":1500000000,"threshold":2147483648}
```

Exit status and output are logged; the hook can send mail through an
internal relay, post to a chat server, run `wall` — whatever exists on your
network.

## Email alerts on failed deployments

For email specifically, you don't need a hook — the agent has built-in SMTP.
On the **Settings** page, fill in **Failed-deployment email alerts**: the admin
email, the SMTP host and port, the security mode (STARTTLS / implicit TLS /
none), optional username and password, and a From address. Use **Save & send
test** to confirm it works.

When a real deploy fails, the agent emails **both the admin and the commit's
committer** (when a committer email is known). Sending is fire-and-forget: it
never blocks or fails the run, and errors are logged. This runs in addition to
`notify_hook` if you set both. See the [Admin UI guide](./ui.md) for the form.

> **What this means.** *SMTP* is the standard email-sending protocol. Point the
> agent at any mail server it can reach — an internal relay on an air-gapped
> network, or a normal SMTP server elsewhere.

## Backup & restore

Back up `data_dir` (DB + `backups/` + `runs/` snapshots) and the
`secret_key_file`. Restore = put them back, start the service. Note that
images themselves live in Docker; a restored "redeployable" run needs its
images still present (or rebuild by re-running its commit).

The **Maintenance page** covers the database part interactively: create and
download backups on demand, and restore by uploading a backup — the restore
is validated, staged, and applied at the next service restart with the
replaced database kept as a safety copy.

### Copying backups off-box (backup hook)

Backups sitting on the same disk don't help if that disk fails. Set
`backup_hook` in `config.toml` to any executable and the agent runs it after
**each** successful backup, passing the backup file's path as the single
argument — so you can rsync or scp it to a mounted share or another host. Its
exit status and output are logged. Empty (the default) disables it.

```bash
#!/bin/sh
# backup_hook = "/etc/ci-agent/copy-backup.sh"
rsync -a "$1" /mnt/backup-share/ci-agent/
```
