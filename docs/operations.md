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

## Redeploying old versions

Every successful run snapshots its exact compose file + the generated
override (pinned image tags) under `runs/<id>/`. *Redeploy* re-runs `up -d`
from that snapshot — no fetch, no build. It works as long as the run's
images still exist, which the cleanup tooling guarantees for the last
`redeployable_runs_per_project` successful runs of each project.

## Queueing

Serial per project; at most `max_concurrent_runs` executing overall; rapid
webhook pushes collapse to the newest (`superseded`). Queued git/ZIP runs do
not survive a restart (they are failed as "lost at restart"); queued
redeploys are re-queued.

## Housekeeping (nightly)

1. SQLite backup via `VACUUM INTO` → `data_dir/backups/` (newest 7 kept).
2. Run-log retention (`log_retention_per_project`).
3. Optional auto-prune of dangling images + build cache (Settings toggle,
   off by default; never touches protected images).
4. Low-disk check → `disk_low` notification.
5. Certificate renewal — auto-renewable certs (internal CA, self-signed,
   Let's Encrypt HTTP-01) within 30 days of expiry are re-issued and nginx is
   re-applied. Uploaded certs and LE DNS-01 wildcards are manual. See
   [Domains & TLS](./domains.md).

It also sweeps stale scratch files (e.g. an abandoned staged update) after an
hour.

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

## Backup & restore

Back up `data_dir` (DB + `backups/` + `runs/` snapshots) and the
`secret_key_file`. Restore = put them back, start the service. Note that
images themselves live in Docker; a restored "redeployable" run needs its
images still present (or rebuild by re-running its commit).

The **Maintenance page** covers the database part interactively: create and
download backups on demand, and restore by uploading a backup — the restore
is validated, staged, and applied at the next service restart with the
replaced database kept as a safety copy.
