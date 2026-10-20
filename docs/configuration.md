# Configuration reference

Config file: `/etc/ci-agent/config.toml` (override path with
`ci-agent serve --config <path>`). All keys are optional; defaults shown.

```toml
listen = "127.0.0.1:8044"
data_dir = "/var/lib/ci-agent"
max_concurrent_runs = 1
run_timeout_secs = 1800
min_free_disk_bytes = 2147483648
zip_max_bytes = 524288000
deploy_healthcheck_secs = 30
redeployable_runs_per_project = 1
log_retention_per_project = 50
pull_policy = "never"
notify_hook = ""
backup_hook = ""
secret_key_file = "/etc/ci-agent/secret.key"
zip_url_allowlist = []
```

| Key | Meaning |
|---|---|
| `listen` | bind address. The packaged config ships `0.0.0.0:8044` for first boot; the setup wizard rewrites it to `127.0.0.1:8044` once the admin domain is verified behind nginx. A non-loopback value locks the UI to the wizard (see [Security](./security.md)) |
| `data_dir` | SQLite DB, workspaces, run snapshots + logs, uploads, backups |
| `max_concurrent_runs` | global cap on simultaneously *executing* runs; runs for the same project are always serial. Used only until you save **Parallel deploys** in the **Maintenance** card on the **Settings** page (range 1–16, no restart). After that, the saved value wins over this key *and* over `CI_AGENT_MAX_CONCURRENT_RUNS`. Values above 16 are treated as 16 |
| `run_timeout_secs` | hard wall-clock limit per run (fetch→deploy); the process tree is killed on expiry. Per-project override available in the UI |
| `min_free_disk_bytes` | pre-flight: a run refuses to start when free disk (data dir or Docker root) is below this. A daily check also fires a `disk_low` event (see `notify_hook`) when free space in the data dir drops below it |
| `zip_max_bytes` | size cap for ZIP downloads and uploads. It also caps single uploads in the File Manager and the terminal |
| `deploy_healthcheck_secs` | how long `up -d` is health-gated before declaring success/failure |
| `redeployable_runs_per_project` | how many recent successful runs keep prune-protected images (and stay redeployable). Used only until you save **Redeployable runs / project** in the **Maintenance** card on the **Settings** page (1–50); the saved value then wins. Auto-prune removes each project's images beyond this count |
| `log_retention_per_project` | run logs kept per project (older logs are deleted nightly; history rows remain) |
| `pull_policy` | `never` (offline default) / `missing` / `always` — whether builds may pull base images |
| `notify_hook` | executable invoked with one JSON argument per event; empty = disabled |
| `backup_hook` | executable run after each successful backup, receiving the backup file's path as its single argument. Use it to copy backups off-box (rsync/scp to a mounted share) so a single-disk failure can't lose them. Empty = disabled |
| `secret_key_file` | 32-byte key for env-var encryption; auto-generated `0600` on first start |
| `zip_url_allowlist` | URL prefixes allowed for ZIP-URL fetches; empty = allow all (air-gapped default) |

Credentials for cloning private repos are **not** configured here — register
them on the **Git credentials** page in the UI (encrypted at rest with
`secret_key_file`). See [Projects](./projects.md).

Domains, certificates and the Let's Encrypt account (email + staging) are
likewise managed in the UI, not in `config.toml` — see
[Domains & TLS](./domains.md).

Failed-deployment **email alerts** (SMTP host, port, credentials) are also set
in the UI, on the Settings page — see [Operations](./operations.md).

## Checks when the agent starts

The agent reads the file once at start. It stops with an error if:

- `listen` is not an `IP:port` pair. Use `127.0.0.1:8044`, not
  `localhost:8044` (host names are not accepted).
- `pull_policy` is not `never`, `missing` or `always`.
- `max_concurrent_runs` is `0`.

An **unknown key** does not stop the agent. It only logs a warning such as
`unknown config key (ignored — typo, or written for a different ci-agent version?)`.

> **What this means.** A typo in a key name is silently ignored except for that
> log line. If a setting seems to have no effect, check
> `journalctl -u ci-agent` for this warning.

The **Effective configuration** card at the bottom of the **Settings** page
shows the values the running agent actually uses.

## Environment overrides

`CI_AGENT_LISTEN`, `CI_AGENT_DATA_DIR`, `CI_AGENT_SECRET_KEY_FILE`,
`CI_AGENT_MAX_CONCURRENT_RUNS`, `CI_AGENT_NOTIFY_HOOK`, `CI_AGENT_BACKUP_HOOK`
override the file.

- While `CI_AGENT_LISTEN` is set, the setup wizard, **Direct port access**,
  `panel-port` and `remove-domain` can't rewrite `listen`. Remove the variable
  from the service environment first.
- `CI_AGENT_MAX_CONCURRENT_RUNS` is ignored once **Parallel deploys** is saved
  on the **Settings** page (see the table above).
- `RUST_LOG` sets how much the agent logs. The default is `info`. For example,
  `RUST_LOG=debug` logs more detail.

## Settings managed in the UI (not in `config.toml`)

These settings are stored in the agent's database, not in the config file.
They are included in backups.

| Setting | Where |
|---|---|
| **Parallel deploys**, **Redeployable runs / project**, **Auto-prune old images & build cache** | **Maintenance** card on the **Settings** page |
| **Shared environment** — `KEY=VALUE` variables merged into every project's build and deploy (a per-project key of the same name wins) | **Settings** page. See [Projects](./projects.md) |
| **Default subdomain** (`<slug>.<base>` URLs) | **Settings** page. See [Domains & TLS](./domains.md) |
| **Failed-deployment email alerts** | **Settings** page. See [Operations](./operations.md) |
| **Branding** (server name and logo) and **Two-factor authentication** devices | **Settings** page. See [Admin UI guide](./ui.md) |
| Theme | Theme picker at the bottom of the sidebar. See [Admin UI guide](./ui.md) |
| **API keys** — keys for scripts and AI agents, each with its own access level | **API keys** page. See [API & AI agents](./api.md) |
| Git credentials | **Git credentials** page |
| Domains, certificates, Let's Encrypt account | See [Domains & TLS](./domains.md) |
| Tunnel mode | **Tunnel** page. See [Reverse tunnel](./tunnel.md) |

## CLI

Run these commands with `sudo`. They need to read `/etc/ci-agent`. When they finish, any
files they created are given back to the `ci-agent` user, so the service can still use them.

```
ci-agent serve                [--config /etc/ci-agent/config.toml]   # default
ci-agent set-password         [--config ...]                         # reset admin password
ci-agent reset-2fa            [--config ...]                         # clear two-factor auth
ci-agent panel-port <on|off|status> [--config ...]                   # emergency direct IP:port access
ci-agent remove-domain        [--config ...]                         # unbind the panel domain, back to IP:port
ci-agent apply-staged-update                                         # internal; run as root by systemd
ci-agent reconcile-sudoers                                           # internal; run by apply-staged-update
ci-agent fs-helper | ufw-helper ...                                  # internal; privileged helpers, run via sudo
ci-agent --version | -V                                              # print the running build and exit
ci-agent --help | -h                                                 # usage summary
```

`set-password` asks for a new admin password (at least 8 characters). The
password is visible while you type it, so make sure nobody is watching.

`reset-2fa` removes every enrolled two-factor (2FA) device and the backup
codes — use it if you lose your authenticator phone and your backup codes. The
next login starts a fresh 2FA enrollment. See [Security](./security.md).

> **After `set-password` or `reset-2fa`:** browsers that are already logged in
> stay logged in until the service restarts. Run
> `sudo systemctl restart ci-agent` afterwards, as both commands remind you.

`panel-port` re-opens the panel by IP address and port when your domain or DNS
is down. `on` (or `enable`) binds `0.0.0.0` on the panel's port (`8044` by
default), opens that port in the firewall, and restarts, so the panel also
answers at `http://<server-ip>:8044`; `off` (or `disable`) returns to
`127.0.0.1` on the same port (reachable only through nginx) and closes the
port; `status` (the default with no argument) just prints the current state. Access still needs
your password and two-factor code. This is the server-shell counterpart of the
Host page's **Direct port access** switch — see [Admin UI → Host](./ui.md) and
[Security](./security.md).

`remove-domain` unbinds the panel's admin domain for good and goes back to
direct IP:port access: it clears the stored domain, binds `0.0.0.0` on the
panel's port (`8044` by default), opens that port in the firewall, and restarts. Unlike `panel-port on` (which keeps the domain
for when nginx recovers), this is the permanent way back — after it, each login
shows a skippable reminder to bind a domain again. The nginx site file
`/etc/nginx/conf.d/ci-agent.conf` is left in place for you to delete. The same
action is on the Host page's **Admin domain** card.

`apply-staged-update` is invoked by the systemd unit's privileged pre-start
step to swap in a binary staged by the Maintenance page's
[self-update](./ui.md); you don't run it by hand. The other internal commands
(`reconcile-sudoers`, `fs-helper`, `ufw-helper`) are also not for manual use:
the agent calls them itself (see [Security](./security.md)).
