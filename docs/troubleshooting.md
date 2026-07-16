# Troubleshooting & FAQ

This page lists common problems and short answers to common questions. If a
fix mentions a config key, see [Configuration](./configuration.md). If it
mentions a UI page, see the [Admin UI guide](./ui.md).

## Common problems

| Problem | Likely cause | Fix |
|---|---|---|
| Every login lands on a "lock the listener to loopback" reminder | `listen` is on a public address (e.g. `0.0.0.0`) — either **Direct port access** is on, or the config was edited by hand | If the open port is intentional, click **Skip for now** — the reminder returns each login while the port is open. Otherwise lock down: click **Lock to 127.0.0.1 and restart**, or set `listen = "127.0.0.1:8044"` in `/etc/ci-agent/config.toml` and restart |
| Can't reach the panel — the domain or DNS is down | nginx/DNS is broken, so the domain URL no longer works, but the server itself is up | On the server run `ci-agent panel-port on` — the panel re-opens at `http://<server-ip>:8044` (still needs your password + 2FA). Fix nginx/DNS, then `ci-agent panel-port off` to lock back to loopback. You can also toggle this from the **Host** page's **Direct port access** card if you can still reach it — see [Security](./security.md) |
| Want to stop using a domain and go back to `http://<server-ip>:8044` for good | The domain was a bad fit (moved network, no DNS, starting over) | Run `ci-agent remove-domain` on the server, or use **Remove domain** on the **Host** page's **Admin domain** card. The panel restarts on the open port; a skippable reminder to bind a domain shows at each login. Delete `/etc/nginx/conf.d/ci-agent.conf` when convenient — see [Security](./security.md) |
| Login asks for a code but I lost my phone | You no longer have your authenticator app | Use one of your **backup codes** on the code screen. No codes left? Run `ci-agent reset-2fa` on the server, then log in and enroll again |
| "Check for updates" says **unreachable** | The server is offline / air-gapped (this is normal) | Nothing to fix. Update with the **signed offline upload** on the Maintenance page, or by `apt` / `scp` |
| "Apply & restart" runs but **System & versions** still shows the old release | The service unit predates self-update and lacks the `ExecStartPre` apply step (the in-app update swaps only the binary, never the unit) | Add the step once via a drop-in — see the note under [Installation → Upgrading](./install.md). Confirm with `systemctl cat ci-agent \| grep ExecStartPre` |
| Updating the agent fails with **413 Request Entity Too Large** | The admin nginx vhost caps the upload body (default 1 MB) and the release bundle is larger | Add `client_max_body_size 0;` inside the `server { … }` block of `/etc/nginx/conf.d/ci-agent.conf`, then `sudo nginx -t && sudo systemctl reload nginx` |
| A run fails right away with a disk message | Free disk is below `min_free_disk_bytes` (default 2 GB) | Free up space (use the **Storage** cleanup page) or lower the threshold in the config |
| Build fails with "image not found" / "pull access denied" | `pull_policy = "never"` (the offline default) and the base image is not on the server | Preload it: `docker save img:tag -o img.tar` on a connected machine, copy it over, then `docker load -i img.tar`. Or set `pull_policy = "missing"` on a connected server |
| Cloning a private repo fails | No credential registered for that git host | Add a credential on the **Git credentials** page (GitHub: host + owner + token; other hosts: host + username + secret) |
| Failed-deploy emails never arrive | SMTP settings wrong, or the mail server is unreachable | On **Settings → Failed-deployment email alerts**, click **Save & send test** and check the error. Confirm host, port and security mode match your mail server |
| Two projects can't reach each other over the network | They share **no category** (network-isolated) | Give them a **category in common** (Project → Settings) — they then share a private Docker network and can reach each other by hostname. Takes effect as soon as you save |
| Webhook returns **404** | Wrong project slug **or** wrong token (both return 404 on purpose) | Re-copy the exact webhook URL from the project page; rotate the token if needed |
| The **nginx apply** button fails | The package's sudo rule is missing (e.g. manual install) | Use the copy-paste commands shown on the same page, or reinstall via the `.deb`/apt which installs the rule |
| **Redeploy** button is greyed out | The run's Docker image was cleaned up | Only the last few successful runs keep their images — set by **Redeployable runs / project** in Settings → Maintenance (default **1**, so by default only the latest version is redeployable). Raise it to keep more, or trigger a fresh run |
| I forgot the admin password | — | Run `ci-agent set-password` on the server |
| After a restore, env vars look wrong/empty | The `secret.key` does not match the restored database | Restore the **matching** `secret.key` too — encrypted values can only be read with the key that wrote them |
| Queued runs disappeared after a restart | Queued git/ZIP runs do not survive a restart (they are marked "lost at restart") | Trigger them again; queued **redeploys** are re-queued automatically |
| Webhook push does nothing, no run starts | The project has **deploys paused** (or is archived) | Click **Resume deploys** (or unarchive) on the project's Settings tab — see [Operations](./operations.md) |
| A **Local** tunnel box won't connect to the relay | Wrong relay address/token/fingerprint, or port **7000** blocked on the Online box | Check **Online server** (`host:7000`), token and fingerprint on the Local box; open port **7000** on the Online box's **Firewall** page — see [Reverse tunnel](./tunnel.md) |
| A tunnel **route** shows "no backend" | The Local box for that tunnel hasn't connected yet | Bring the Local box online and check its **Connection status** — see [Reverse tunnel](./tunnel.md) |

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
