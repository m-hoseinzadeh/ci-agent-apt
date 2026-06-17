# Troubleshooting & FAQ

This page lists common problems and short answers to common questions. If a
fix mentions a config key, see [Configuration](./configuration.md). If it
mentions a UI page, see the [Admin UI guide](./ui.md).

## Common problems

| Problem | Likely cause | Fix |
|---|---|---|
| The whole panel is stuck on the setup wizard | `listen` was changed back to a public address (e.g. `0.0.0.0`), so the agent re-locks itself for safety | Set `listen = "127.0.0.1:8044"` in `/etc/ci-agent/config.toml`, restart the service, and reach the panel through nginx or an SSH tunnel |
| Login asks for a code but I lost my phone | You no longer have your authenticator app | Use one of your **backup codes** on the code screen. No codes left? Run `ci-agent reset-2fa` on the server, then log in and enroll again |
| "Check for updates" says **unreachable** | The server is offline / air-gapped (this is normal) | Nothing to fix. Update with the **signed offline upload** on the Maintenance page, or by `apt` / `scp` |
| "Apply & restart" runs but **System & versions** still shows the old release | The service unit predates self-update and lacks the `ExecStartPre` apply step (the in-app update swaps only the binary, never the unit) | Add the step once via a drop-in — see the note under [Installation → Upgrading](./install.md). Confirm with `systemctl cat ci-agent \| grep ExecStartPre` |
| Updating the agent fails with **413 Request Entity Too Large** | The admin nginx vhost caps the upload body (default 1 MB) and the release bundle is larger | Add `client_max_body_size 0;` inside the `server { … }` block of `/etc/nginx/conf.d/ci-agent.conf`, then `sudo nginx -t && sudo systemctl reload nginx` |
| A run fails right away with a disk message | Free disk is below `min_free_disk_bytes` (default 2 GB) | Free up space (use the **Storage** cleanup page) or lower the threshold in the config |
| Build fails with "image not found" / "pull access denied" | `pull_policy = "never"` (the offline default) and the base image is not on the server | Preload it: `docker save img:tag -o img.tar` on a connected machine, copy it over, then `docker load -i img.tar`. Or set `pull_policy = "missing"` on a connected server |
| Cloning a private GitHub repo fails | No token registered for that GitHub account | Add a personal access token on the **GitHub tokens** page (one per GitHub username) |
| Webhook returns **404** | Wrong project slug **or** wrong token (both return 404 on purpose) | Re-copy the exact webhook URL from the project page; rotate the token if needed |
| The **nginx apply** button fails | The package's sudo rule is missing (e.g. manual install) | Use the copy-paste commands shown on the same page, or reinstall via the `.deb`/apt which installs the rule |
| **Redeploy** button is greyed out | The run's Docker images were pruned | Only the last few successful runs keep their images (`redeployable_runs_per_project`). Trigger a fresh run instead |
| I forgot the admin password | — | Run `ci-agent set-password` on the server |
| After a restore, env vars look wrong/empty | The `secret.key` does not match the restored database | Restore the **matching** `secret.key` too — encrypted values can only be read with the key that wrote them |
| Queued runs disappeared after a restart | Queued git/ZIP runs do not survive a restart (they are marked "lost at restart") | Trigger them again; queued **redeploys** are re-queued automatically |

> **Tip:** the **Audit** page lists recent actions and webhook hits, and each
> run has a full live log on its run page — both are good first places to look
> when something behaves unexpectedly.

## Frequently asked questions

**Does the agent need an internet connection?**
No. It is built for offline / air-gapped networks. Nothing is pulled by
default, there is no telemetry, and the docs and QR codes are generated
locally.

**Can it run several deployments at the same time?**
Runs for the same project are always one at a time. Across all projects, the
limit is `max_concurrent_runs` (default `1`, good for small servers).

**What happens if a deployment breaks?**
The agent health-checks the new containers. If they fail, it automatically
rolls back to the last working version and marks this in the run log.

**Where is my data stored?**
In `data_dir` (default `/var/lib/ci-agent`): the database, run snapshots,
backups, workspaces and uploads. The encryption key is `secret.key`
(default `/etc/ci-agent/secret.key`).

**How do I back up?**
Back up `data_dir` **and** `secret.key`. The Maintenance page can create and
download database backups for you; a nightly backup also runs automatically.

**How do I update the agent?**
Three ways: `apt` on a connected server, the **self-update** on the
Maintenance page (upload the signed release), or copy the new binary over and
restart. See [Installation](./install.md).

**Can the admin panel use HTTPS?**
Yes. The agent speaks plain HTTP and your host nginx adds TLS. Extend the
generated nginx server block with a `listen 443 ssl;` block — example in
[Security](./security.md).

**Can I turn off two-factor auth?**
No — it is required. You can change the secret (new phone) or regenerate
backup codes from the **Settings** page, and clear it with `ci-agent reset-2fa`
if you are locked out.
