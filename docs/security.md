# Security

## Model

Single admin user; argon2id-hashed password; HttpOnly `SameSite=Lax`
session cookie; login rate-limited (5/min); every mutating action is
CSRF-protected and written to the audit log.

Webhooks authenticate with a 256-bit random URL token (constant-time
compare; unknown slug and bad token are both `404` so projects can't be
enumerated). Optional per-project HMAC signatures harden this further —
see [Webhooks](./webhooks.md).

Project env vars are encrypted at rest (XChaCha20-Poly1305, key file
`0600`) and masked in run logs and the UI.

## Network exposure

The agent speaks **plain HTTP** by design (TLS belongs to your nginx):

- Best: keep `listen` on `127.0.0.1` and access the UI through an SSH
  tunnel, or publish it via host nginx with TLS:

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

- Binding to `0.0.0.0` on an untrusted network sends your password, session
  cookie and submitted secrets in cleartext — acceptable for a quick test
  on a trusted LAN, not for the internet.
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

## systemd hardening

The shipped unit applies `NoNewPrivileges`, `PrivateTmp`,
`ProtectSystem=strict`, `ProtectHome=read-only`, with writes limited to
`/var/lib/ci-agent` and `/etc/ci-agent`.
