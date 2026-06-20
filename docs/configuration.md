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
redeployable_runs_per_project = 3
log_retention_per_project = 50
pull_policy = "never"
notify_hook = ""
secret_key_file = "/etc/ci-agent/secret.key"
zip_url_allowlist = []
```

| Key | Meaning |
|---|---|
| `listen` | bind address. The packaged config ships `0.0.0.0:8044` for first boot; the setup wizard rewrites it to `127.0.0.1:8044` once the admin domain is verified behind nginx. A non-loopback value locks the UI to the wizard (see [Security](./security.md)) |
| `data_dir` | SQLite DB, workspaces, run snapshots + logs, uploads, backups |
| `max_concurrent_runs` | global cap on simultaneously *executing* runs; runs for the same project are always serial |
| `run_timeout_secs` | hard wall-clock limit per run (fetch→deploy); the process tree is killed on expiry. Per-project override available in the UI |
| `min_free_disk_bytes` | pre-flight: a run refuses to start when free disk (data dir or Docker root) is below this |
| `zip_max_bytes` | ZIP download/upload size cap |
| `deploy_healthcheck_secs` | how long `up -d` is health-gated before declaring success/failure |
| `redeployable_runs_per_project` | how many recent successful runs keep prune-protected images |
| `log_retention_per_project` | run logs kept per project (older logs are deleted nightly; history rows remain) |
| `pull_policy` | `never` (offline default) / `missing` / `always` — whether builds may pull base images |
| `notify_hook` | executable invoked with one JSON argument per event; empty = disabled |
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

## Environment overrides

`CI_AGENT_LISTEN`, `CI_AGENT_DATA_DIR`, `CI_AGENT_SECRET_KEY_FILE`,
`CI_AGENT_MAX_CONCURRENT_RUNS`, `CI_AGENT_NOTIFY_HOOK` override the file.

## CLI

```
ci-agent serve                [--config /etc/ci-agent/config.toml]   # default
ci-agent set-password         [--config ...]                         # reset admin password
ci-agent reset-2fa            [--config ...]                         # clear two-factor auth
ci-agent apply-staged-update                                         # internal; run as root by systemd
ci-agent --version | -V                                              # print the running build and exit
ci-agent --help | -h                                                 # usage summary
```

`reset-2fa` clears the two-factor (2FA) secret and backup codes — use it if
you lose your authenticator phone and your backup codes. The next login
starts a fresh 2FA enrollment. See [Security](./security.md).

`apply-staged-update` is invoked by the systemd unit's privileged pre-start
step to swap in a binary staged by the Maintenance page's
[self-update](./ui.md); you don't run it by hand.
