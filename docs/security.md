# Security

## Model

There is **one admin user**. Login is in two steps:

1. **Password** — stored only as an argon2id hash (never in plain text).
2. **Two-factor code (2FA)** — a 6-digit code from an authenticator app
   (Google Authenticator, and any app that follows the same standard).

After a successful login you get an HttpOnly `SameSite=Lax` session cookie
(also flagged `Secure` when you reached the panel over HTTPS). Logins are
rate-limited **per client IP** (5 tries a minute, with a global backstop).
Changing the admin password **signs out all other sessions**. Every action that
changes something is CSRF-protected and written to the audit log.

> **What is 2FA?** "Two-factor" means you need two things to log in:
> something you *know* (the password) and something you *have* (your phone
> with the authenticator app). If someone steals the password, they still
> can't get in.

### Two-factor authentication (TOTP)

- **Mandatory.** After you first set the password and log in, the panel
  sends you to an enrollment page and stays locked until you finish.
- **Offline-friendly.** The setup page shows a QR code drawn by the agent
  itself (no internet needed). Scan it with your authenticator app, or type
  the shown secret in by hand.
- **Standard settings** (cannot be changed): SHA1, 6 digits, a new code
  every 30 seconds. The agent accepts the code from the step before and
  after the current one, so a small clock difference between phone and
  server is fine. A code that was **already used** is rejected, so a
  captured code can't be replayed while it's still valid.
- **Backup codes.** When you enable 2FA the agent shows **10 one-time backup
  codes**. Save them somewhere safe — each one logs you in once if you lose
  your phone.
- **The secret and backup codes are encrypted at rest** with the same key
  that protects project secrets (`secret_key_file`), never stored in plain
  text.
- **Lost your phone?** Use a backup code on the code-entry page, or run
  `ci-agent reset-2fa` on the server to clear 2FA — the next login starts a
  fresh enrollment. See the [Admin UI guide](./ui.md) for day-to-day 2FA
  management (regenerate backup codes, change the secret).

Webhooks authenticate with a 256-bit random URL token (constant-time
compare; unknown slug and bad token are both `404` so projects can't be
enumerated) — see [Webhooks](./webhooks.md).

The optional **automation API** (`/api/*`) is guarded by a single
admin-generated **bearer token** (constant-time compare, stored encrypted at
rest). The whole surface is **disabled until a token is set**, and it is
**read-only** — it exposes health, project and run status but cannot trigger
deploys or change anything. Set or revoke it on the **Settings** page; see
[Operations](./operations.md).

Project env vars and stored **git credentials** (tokens or passwords for private
clones) are encrypted at rest (XChaCha20-Poly1305, key file `0600`) and masked in
run logs and the UI.

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
  network, set the allowlist. Either way, git and ZIP fetches are blocked
  from reaching **loopback / link-local** addresses (e.g. `127.0.0.1`,
  `169.254.*`, `::1`), so a supplied URL can't be pointed back at the host
  itself; private LAN ranges stay reachable on purpose.
- A webhook that **pins an exact commit** must be able to fetch and check that
  commit out — if it can't, the run **fails** rather than quietly deploying
  branch `HEAD` instead.

## Managing nginx and TLS

One-click nginx apply shells out via `sudo -n` to a **narrow NOPASSWD
allowlist** (installed by the package at `/etc/sudoers.d/`) limited to exactly
`tee` a `/etc/nginx/conf.d/*.conf` file, `nginx -t`, and `systemctl reload
nginx`. The same allowlist backs the **Host page** editors (the admin-panel
vhost and the global http snippet are just more conf.d files), its **service
controls** — a fixed set of `systemctl restart nginx|docker|ci-agent` and
`journalctl -u <those>` — and the **Firewall page**, which is limited to the
`ufw` status/allow/deny/reject/delete/enable/disable/reload commands (the
arguments are validated in-process). Nothing arbitrary. No interactive prompt;
if the rule is absent the call fails fast and the manual copy-paste commands
still work. The agent re-checks and repairs this allowlist on every boot, so it
self-heals if it drifts. TLS cert/key PEMs are written by the unprivileged agent
into its own data dir (nginx's master reads them), so cert issuance needs no
`sudo`. See [Domains & TLS](./domains.md).

The **host file manager** (`/files`) and the **server shell** run as the agent's
own user inside the systemd sandbox below — the host filesystem is **read-only**
apart from the agent's own data dirs, and `sudo` can't authenticate as the
passwordless agent account, so neither is a general root console. But the
agent's Docker access is root-equivalent on the host (e.g. `docker run
--privileged`), so treat their blast radius as root SSH access all the same. For
privileged, filesystem-mutating host work the server shell offers an **Open host
SSH session** button that logs in as a real admin account via `sshd`, outside
the sandbox (see [the UI guide](./ui.md)). The container file manager and shell
are scoped to containers carrying the project's compose label, the same boundary
as `docker cp` / `docker exec`.

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
