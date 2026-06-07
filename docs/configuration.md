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
github_token = ""
```

| Key | Meaning |
|---|---|
| `listen` | bind address. Keep on loopback and front with nginx/SSH when exposed (see [Security](./security.md)) |
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
| `github_token` | optional GitHub token used for: (1) cloning private `https://github.com/...` project repos, (2) the Maintenance page's update check, (3) ci-agent self-update downloads while this repo is private. Empty on air-gapped servers. Protect the config file (`chmod 640`, owner `ci-agent`) |

## Environment overrides

`CI_AGENT_LISTEN`, `CI_AGENT_DATA_DIR`, `CI_AGENT_SECRET_KEY_FILE`,
`CI_AGENT_MAX_CONCURRENT_RUNS`, `CI_AGENT_NOTIFY_HOOK` override the file.

## CLI

```
ci-agent serve         [--config /etc/ci-agent/config.toml]   # default
ci-agent set-password  [--config ...]                         # reset admin password
```
