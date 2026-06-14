# Security

## Model

Single admin user; argon2id-hashed password; HttpOnly `SameSite=Lax`
session cookie; login rate-limited (5/min); every mutating action is
CSRF-protected and written to the audit log.

Webhooks authenticate with a 256-bit random URL token (constant-time
compare; unknown slug and bad token are both `404` so projects can't be
enumerated) — see [Webhooks](./webhooks.md).

Project env vars and GitHub personal access tokens are encrypted at rest
(XChaCha20-Poly1305, key file `0600`) and masked in run logs and the UI.

## Network exposure

The agent speaks **plain HTTP** by design (TLS belongs to your nginx), and
the first-run wizard enforces the safe end state: the admin panel must be
bound to a domain and proven reachable through host nginx before the agent
rewrites `listen` to `127.0.0.1` and restarts itself. Verification is
spoof-resistant — a request only counts as "via nginx" when its `Host`
matches the bound domain **and** the TCP peer is loopback **and**
`X-Forwarded-For` is present, so a forged Host header on the public port
cannot trigger a premature lockdown (or fake one to lock the admin out
before nginx works). If `listen` is later edited back to a public address,
the admin UI locks to the wizard again until the listener is back on
loopback.

- Steady state: `listen` on `127.0.0.1`, access through the wizard's nginx
  site or an SSH tunnel. For TLS, extend the generated server block:

```nginx
server {
    listen 443 ssl;
    server_name ci.example.internal;
    # ssl_certificate ...; ssl_certificate_key ...;
    location / {
        proxy_pass http://127.0.0.1:8044;
        proxy_set_header Host $host;
        # SSE live logs need buffering off:
        proxy_buffering off;
        proxy_read_timeout 1h;
    }
}
```

- The packaged config binds `0.0.0.0:8044` for first boot only, so the
  wizard is reachable before nginx exists. On an untrusted network that
  window sends your password in cleartext — set
  `listen = "127.0.0.1:8044"` before first start and run the wizard
  through an SSH tunnel instead.
- Set the admin password **immediately after first start**; the first-run
  setup page is open until then.

## Accepted trade-offs (by design)

- The agent's user is in the `docker` group, which is root-equivalent on
  the host. Inherent to the job — it deploys containers.
- Builds are **not sandboxed**: a registered repo's Dockerfile runs with
  full daemon access. Only admins can register projects; don't register
  repos you don't trust.
- ZIP-URL fetches default to allow-all (`zip_url_allowlist = []`) because
  on an air-gapped network internal hosts are the point. On a connected
  network, set the allowlist.
- One global login rate limit (not per-IP) — single-admin tool.

## Managing nginx and TLS

One-click nginx apply shells out via `sudo -n` to a **narrow NOPASSWD
allowlist** (installed by the package at `/etc/sudoers.d/`) limited to exactly
`tee` the project's conf file, `nginx -t`, and `systemctl reload nginx`. No
interactive prompt; if the rule is absent the call fails fast and the manual
copy-paste commands still work. TLS cert/key PEMs are written by the
unprivileged agent into its own data dir (nginx's master reads them), so cert
issuance needs no `sudo`. See [Domains & TLS](./domains.md).

## Self-update integrity

The Maintenance page's signed-upload update verifies the uploaded `.deb`
**offline** against the release public key embedded in the binary, pinned to a
single fingerprint — a valid signature from any other key is rejected — and
refuses downgrades. Uploading the `ci-agent-offline-*.tar.gz` bundle is the same
check: the `.deb` and its detached signature are unpacked from it and the
bundle's own bundled public key is ignored, so trust always rests on the
embedded key, never one that travels with the upload. The privileged binary swap runs only from systemd's root
pre-start step, and a crash-looping new binary is rolled back automatically.
See [the UI guide](./ui.md).

## systemd hardening

The shipped unit applies `NoNewPrivileges`, `PrivateTmp`,
`ProtectSystem=strict`, `ProtectHome=read-only`, with writes limited to
`/var/lib/ci-agent` and `/etc/ci-agent` (plus `/etc/nginx`, so the sudo'd
config write lands on a writable mount). The privileged `apply-staged-update`
pre-start step is the only component that writes `/usr/bin`.
