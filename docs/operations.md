# Operations

## Run lifecycle

```
queued → fetching → building → (testing) → deploying → success
                                               ↘ failed (auto-rollback attempted)
other terminal states: cancelled, superseded, interrupted, timeout
```

`testing` happens only when the project has a `ci-test` service. A redeploy or
recreate skips `fetching` and `building` (`queued → deploying`). A run that was
in progress when the agent restarted becomes `interrupted`.

- **Status callback**: runs started by a webhook or the API can report every
  status change to a URL you choose — see
  [Webhooks → Status callback](./webhooks.md).

- **Pre-flight**: a run refuses to start when the data directory or
  `/var/lib/docker` has less than `min_free_disk_bytes` free.
- **Timeout**: each run has a hard limit (`run_timeout_secs` or per-project
  override). On expiry the whole process tree is killed.
- **Cancel**: from the run page; TERM, then KILL after 10 s.
- **Build niceness**: heavy subprocesses run under `nice`/`ionice` so the
  agent doesn't starve the apps it hosts.

## Deploys, health gate, rollback

Every deploy works in this order:

1. **Pull** — `docker compose pull --ignore-buildable` fetches newer copies of
   the `image:` services. If the pull fails (always the case on an air-gapped
   box), the log shows a warning and the deploy uses the images already on the
   server.
2. **Start** — `docker compose up -d --remove-orphans`.
3. **Health gate** — for up to `deploy_healthcheck_secs` the agent checks the
   containers every 2 seconds. The run **fails** when a container exits with a
   **non-zero** code or reports **unhealthy**, or when containers are still
   starting at the end of the wait. A container that exits with code `0` (a
   one-shot setup or migration job) is fine.
4. **Rollback on failure** — the agent automatically redeploys the **last
   successful run's snapshot** (clearly marked in the log). The rollback gets
   its own time limit, so it can finish even if the run timed out.

The brief stop-start window of the `recreate` strategy is an accepted
trade-off on small servers.

> **Note:** "rolling back" here means putting the previous working version
> back, automatically, so a bad deploy doesn't leave your app down. A
> "snapshot" is the exact compose file and image versions the agent saved for
> each successful run.

## Redeploying old versions

Every successful run snapshots its exact compose file + the generated
override (pinned image tags) under `runs/<id>/`. *Redeploy* re-runs `up -d`
from that snapshot — no fetch, no build — after the same pull step as a normal
deploy (a failed pull is ignored on an air-gapped box). It works as long as the run's images still exist, which the
cleanup keeps for the last few successful runs of each project — set by
**Redeployable runs / project** in Settings → Maintenance (default **1**, so by
default only the version that's live now is redeployable; raise it to keep more).

*Recreate* (the **Recreate** button on the **Containers** card of the
**Deployments** tab) is related but different: it rebuilds the containers from
the last successful snapshot using the project's **current env vars and
volumes**, so it's how you apply env/volume edits without a full rebuild.

The agent also acts by itself when you save changed env vars or volumes:

- Most projects get an automatic **recreate**.
- **Dockerfile** projects with a **Git URL** get a full **rebuild** from git
  instead (the message reads *rebuilding image with new build args*). This is
  because env values can be baked into the image as build arguments (`ARG`).
- Nothing happens when the project has never deployed successfully — the next
  deploy uses the new values.

These automatic runs are not reported to the status callback.

## Queueing

Serial per project; at most `max_concurrent_runs` executing overall (editable
live in **Settings → Maintenance → Parallel deploys**); rapid
webhook pushes collapse to the newest (`superseded`).

After a restart:

- Runs that were in progress become `interrupted`.
- Queued **redeploys** and queued **Deploy now** runs of Image / Inline compose
  projects (`manual_deploy`) are queued again automatically.
- Queued git/ZIP runs are failed with the message "lost at restart" — the
  agent does not keep their source. Trigger them again.

## Pausing deploys (change freeze)

Each project's **Settings** tab has a **Pause deploys** button. While a project
is paused:

- **Webhook** deploys are skipped. The webhook still answers `202` with
  `{"skipped": "deploys paused"}`, so your git server doesn't report an error.
- **API** deploys, redeploys and recreates are refused with `409`
  `deploys_paused`, unless the request sends `"force": true` (see
  [API & AI agents](./api.md)).
- Scheduled auto-redeploys (below) are skipped.
- **Manual** deploys and **redeploys** from the panel still work.
- The running stack stays up, and a **deploys paused** chip is shown.

Use it during a change freeze or an incident, then click **Resume deploys** to
let webhooks through again.

## Scheduled auto-redeploy

Each project's **Settings** tab has an **Auto-redeploy every (hours)** field
(0–168, `0` = off). When set, the agent re-runs the project's **last successful
deploy** on that interval. It's handy for refreshing an image pinned to a moving
tag (e.g. `:latest`) and for giving a flaky stack a periodic recreate.

- The agent checks schedules every 10 minutes, so a redeploy can start up to
  about 10 minutes after it is due.
- The **first time** you turn the schedule on, the first redeploy lands one full
  interval later.
- The agent remembers when it last fired. If you turn the schedule off and on
  again, or make the interval shorter, it counts from that last time — so a
  redeploy can start within about 10 minutes of saving.
- A project needs one successful deploy first; until then the schedule waits.
- Scheduled redeploys are skipped while deploys are paused, and they are not
  reported to the status callback.

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

Scripts, monitoring boxes and AI agents can use ci-agent over HTTP with an
**API key**. Create keys on the **API keys** page. A **read** key can poll
health and run status, an **operator** key can also deploy and change projects,
and an **admin** key can do everything. Example:

```bash
curl -H 'Authorization: Bearer cia_…' https://ci.example.internal/api/v1/health
```

The full endpoint list is in [API & AI agents](./api.md).

## Housekeeping (nightly)

Housekeeping first runs about **60 seconds after the agent starts**, then every
**24 hours** from then on — so "nightly" means "once a day", not a fixed clock
time. To run it right now, use **Run housekeeping now** on the **Maintenance**
page. Each pass does these steps:

1. SQLite backup via `VACUUM INTO` → `data_dir/backups/` (newest 7 kept;
   backups made with **Create backup now** count toward the 7).
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

It also sweeps stale scratch files: an abandoned staged update after an hour,
and leftover deploy-ZIP uploads (from runs that were cancelled or lost while
queued) after a day.

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
{"event":"health_degraded","project":"my-app","problems":["…"]}
{"event":"health_recovered","project":"my-app"}
{"event":"cert_expiring","base_domain":"example.internal","source":"upload",
 "not_after":"2026-12-01T00:00:00Z"}
```

| Event | When |
|---|---|
| `run_finished` | a run ends (from the runner: success, failed, cancelled, timeout) |
| `disk_low` | the nightly check finds free disk below `min_free_disk_bytes` |
| `health_degraded` | a project's background health check stays in a problem state (see *Health monitoring & down-alerts* above); `problems` lists what is wrong |
| `health_recovered` | that project is healthy again |
| `cert_expiring` | a certificate that can't renew by itself (an uploaded/imported cert, or a Let's Encrypt DNS-01 wildcard) enters its renewal window; sent once per expiry date |

> **Hook or callback?** `notify_hook` is one program on **this server** that
> hears about **every** project. The per-project
> [status callback](./webhooks.md) is an HTTP `POST` to **another system**,
> only for runs that system requested. You can use both.

The hook can send mail through an internal relay, post to a chat server, run
`wall` — whatever exists on your network. It runs in the background and never
delays or fails anything. A non-zero exit is logged as a warning. A hook that
runs longer than **30 seconds** is stopped.

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
download backups on demand, and restore by uploading a backup. To restore:

1. In the **Restore** card on the Maintenance page, pick the backup file and
   click **Stage restore**. The agent checks that it is a CI Agent database.
2. The restore is now **staged** — nothing has changed yet. To back out, click
   **Cancel staged restore**.
3. Restart the service (`sudo systemctl restart ci-agent`). The backup is put in
   place, and the replaced database is kept in the data directory as
   `ci-agent.db.pre-restore-<timestamp>`.

![screenshot: Maintenance page with a staged restore and the Cancel staged restore button](TODO)

### Copying backups off-box (backup hook)

Backups sitting on the same disk don't help if that disk fails. Set
`backup_hook` in `config.toml` to any executable and the agent runs it after
each successful **housekeeping** backup (the daily one, or **Run housekeeping
now**), passing the backup file's path as the single argument — so you can
rsync or scp it to a mounted share or another host. Empty (the default)
disables it.

- A backup made with **Create backup now** on the Maintenance page does
  **not** run the hook. Copy that file yourself, or click **Run housekeeping now** instead.
- A non-zero exit is logged as a warning. A hook that runs longer than
  **5 minutes** is stopped.

```bash
#!/bin/sh
# backup_hook = "/etc/ci-agent/copy-backup.sh"
rsync -a "$1" /mnt/backup-share/ci-agent/
```
