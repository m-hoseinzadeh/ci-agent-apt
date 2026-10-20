# Security

## Model

There is **one admin user**. Login is in two steps:

1. **Password** — stored only as an argon2id hash (never in plain text).
2. **Two-factor code (2FA)** — a 6-digit code from an authenticator app
   (Google Authenticator, and any app that follows the same standard).

After a successful login you get an HttpOnly `SameSite=Lax` session cookie
(also flagged `Secure` when you reached the panel over HTTPS). Logins are
rate-limited **per client IP** (5 tries a minute, with a global backstop of 50
tries a minute). Every action that changes something is CSRF-protected and
written to the audit log.

Sessions work like this:

- A session lasts **7 days from your last visit**. Each page you open restarts
  the 7 days.
- Sessions are kept **in memory only**. Restarting the service (or the server)
  logs everyone out. So does a self-update.
- When you enter the 2FA code, the agent gives you a **new** session cookie. A
  cookie stolen during the password step is useless after that.
- Changing the password on the **Settings** page **signs out all other
  sessions**. The `ci-agent set-password` and `ci-agent reset-2fa` commands do
  not — restart the service afterwards to sign everyone out.

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
- **Several devices at once.** You can enrol more than one authenticator —
  a phone and a laptop, say — and a code from **any** of them logs you in.
  Each device is listed on the **Settings** page and can be removed on its
  own; the others keep working. Adding one never disables the others. The
  **last** remaining device can't be removed, so you can't accidentally lock
  yourself out — add the replacement first.
- **Backup codes.** When you enable 2FA the agent shows **10 one-time backup
  codes**. Save them somewhere safe — each one logs you in once if you lose
  your phone. They belong to the account, not to a device, so adding or
  removing a device leaves them alone.
- **Device secrets and backup codes are encrypted at rest** with the same key
  that protects project secrets (`secret_key_file`), never stored in plain
  text.
- **Lost your phone?** Sign in with a backup code, then remove the lost device
  on the **Settings** page and add a new one. With no codes left either, run
  `ci-agent reset-2fa` on the server to clear every device — the next login
  starts a fresh enrollment. See the [Admin UI guide](./ui.md) for day-to-day
  2FA management.

Webhooks authenticate with a 256-bit random URL token (constant-time
compare; unknown slug, bad token and archived project are all `404` so
projects can't be enumerated) — see [Webhooks](./webhooks.md).

**Treat a webhook URL like an admin password.** Whoever has it can start a
deploy, and the request can also name *what* to build. On the `/git` hook, a
`git_url` (in the JSON body or `?git_url=`) replaces the project's repository
for that run. The `/zip` hook always builds the archive at the `zip_url` the
caller sends. That is the same power as an **admin** API key's code choice. Only an
admin can see the URL (on the project page, or through an admin API key). If it
leaks, click **Rotate token** on the project's **Deployments** tab — the old
URL stops working at once.

The **automation API** (`/api/*`) only accepts **API keys** created on the
**API keys** page. It answers nothing until a key exists.

> **Upgraded from an older version?** Older releases had a single API token.
> On upgrade, the agent turns it into a **read**-level key named `legacy`, so
> existing monitoring keeps working. You'll see it on the **API keys** page.
> Revoke it if nothing uses it any more.

- Each key is 256-bit random. Only its SHA-256 hash is stored, and it is
  compared in constant time. The key is shown once.
- Each key has a level: **read**, **operator** or **admin**. **Operator** and
  **admin** keys can read every project's environment variables in cleartext.
- Only **admin** keys can choose what code runs: a git repository, a zip URL,
  an inline compose file, or a project's categories (which decide which other
  projects it can reach). A compose file can ask for a privileged container or
  mount host folders, so an admin key means **full access to the server**.
  Admin keys can also delete projects and run commands inside containers.
- **Operator** keys deploy only what an admin set up, plus image-mode projects
  (whose compose file ci-agent writes itself). Use **read** keys for agents
  that only need to look.
- Keys **skip the password and 2FA login**, so give each agent its own key with
  the lowest level it needs, and set an expiry. Revoking a key takes effect on
  the next request.
- Every change made with a key is in the audit log as `api:<key name>`. Each
  key's last use (time and IP) shows on the API keys page.
- After 10 wrong keys within a minute, an address gets `429` for a minute.
- Unlike the admin panel, the API answers under any host name that reaches
  ci-agent, not only the admin domain. That lets agents on the server use
  `http://127.0.0.1:<port>/api/v1`. Other machines should use the admin
  domain over HTTPS, so keys are never sent in cleartext.
- Container logs are not masked, so a read key can see anything an app prints.

See [API & AI agents](./api.md).

Project env vars and stored **git credentials** (tokens or passwords for private
clones) are encrypted at rest (XChaCha20-Poly1305, key file `0600`) and masked
(`*****`) in run logs. The admin panel itself shows env values in **cleartext**,
so anyone logged in to the panel can read them.

## Network exposure

The agent speaks **plain HTTP** by design (TLS belongs to your nginx), and
the first-run wizard enforces the safe end state: the admin panel must be
bound to a domain and proven reachable through host nginx before the agent
rewrites `listen` to `127.0.0.1` and restarts itself. Verification is
spoof-resistant — a request only counts as "via nginx" when its `Host`
matches the bound domain **and** the TCP peer is loopback **and**
`X-Forwarded-For` is present, so a forged Host header on the public port
cannot trigger a premature lockdown (or fake one to lock the admin out
before nginx works). If `listen` is later set back to a public address
(by hand, **Direct port access** or `panel-port on`), each login shows a
**lock to loopback** reminder. You can skip it for that login.

After setup, the panel only answers on its own admin domain. Any other domain
pointed at the agent gets a plain **Under construction** page. Webhooks
(`/hooks/…`), the API (`/api/…`) and Let's Encrypt challenges are the
exceptions — they answer on any host name.

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
- **Emergency direct access:** if your domain or DNS later goes down and you
  can't reach the panel through nginx, use the **Direct port access** switch on
  the [Host page](./ui.md) (or `ci-agent panel-port on` on the server) rather
  than editing `listen` by hand. It binds `0.0.0.0:8044`, opens the firewall
  port, and restarts, so the panel answers at `http://<server-ip>:8044` — the
  sanctioned inverse of the loopback lock. Login still requires your password
  and two-factor code, but this is **plain HTTP over the open network**: use it
  only to recover, then turn it off (**Disable — lock to loopback** /
  `ci-agent panel-port off`) once nginx is healthy again. While the port is
  open, every login shows a skippable **lock to loopback** reminder.
- **Going back to IP:port for good:** to drop the domain entirely (not just
  bridge an outage), use **Remove domain** on the [Host page](./ui.md) or
  `ci-agent remove-domain` on the server. It clears the stored domain, binds
  `0.0.0.0:8044`, opens the firewall port, and restarts. The same plain-HTTP
  caveat applies permanently; each login then shows a skippable reminder to
  bind a domain again.
- Set the admin password **immediately after first start**; the first-run
  setup page is open until then.

## Accepted trade-offs (by design)

- The agent's user is in the `docker` group, which is root-equivalent on
  the host. Inherent to the job — it deploys containers.
- Builds are **not sandboxed**: a registered repo's Dockerfile runs with
  full daemon access. Only admins (and admin API keys) can register projects
  or change their source — but a webhook URL can name a source too (see
  *Model* above). Don't register repos you don't trust.
- ZIP-URL fetches default to allow-all (`zip_url_allowlist = []`) because
  on an air-gapped network internal hosts are the point. On a connected
  network, set the allowlist. Either way, git and ZIP fetches are blocked
  from reaching **loopback / link-local** addresses (e.g. `127.0.0.1`,
  `169.254.*`, `::1`), so a supplied URL can't be pointed back at the host
  itself; private LAN ranges stay reachable on purpose.
- A webhook that **pins an exact commit** must be able to fetch and check that
  commit out — if it can't, the run **fails** rather than quietly deploying
  branch `HEAD` instead.
- A project's **status callback** makes the agent send HTTP `POST` requests to
  a URL the admin chooses (only the admin, or an admin API key, can set it).
  Loopback and LAN addresses are allowed on purpose, because the receiver often
  runs on the same server or network. Link-local (`169.254.*`, `fe80::`),
  `0.0.0.0` and broadcast addresses are refused when written as an IP address
  in the URL (a host name is not looked up for this check). The callback token is stored encrypted and passed to
  `curl` through standard input, so it never appears in the process list. The
  `error` text it sends has project/shared env values and git credentials
  replaced with `*****`. See [Webhooks → Status callback](./webhooks.md).

## Managing nginx and TLS

The agent runs as the unprivileged `ci-agent` user. For the few jobs that need
root, it uses `sudo -n` with a **NOPASSWD allowlist** that the package installs
at `/etc/sudoers.d/ci-agent-nginx`. The list has no wildcards. It allows:

- `nginx -t` and `systemctl reload nginx` (applying nginx config);
- `systemctl restart nginx|docker|ci-agent` and `journalctl -u <those> -n 200
  --no-pager` (the **Host** page service controls);
- `ufw status` (plain, `verbose`, `numbered`), `ufw --force enable`,
  `ufw disable` and `ufw reload` (the **Firewall** page);
- the agent's own binary, `/usr/bin/ci-agent`, with any arguments. This is how
  the agent runs its two privileged helpers, `fs-helper` and `ufw-helper`.

The helpers handle everything that takes input from you:

- `ufw-helper` adds and deletes firewall rules. It checks every rule field
  itself.
- `fs-helper` reads, writes, deletes and sizes files **anywhere on the host**,
  as root. The File Manager, the disk reports, cleanups and the nginx config
  editors (project sites, the admin-panel site, the global http snippet) all
  use it.

Both helpers refuse to run unless they really are root, and they never pass
your input through a shell.

> **What this means.** The allowlist is narrow in *which programs* it allows,
> but `fs-helper` is full root access to files by design — the File Manager
> needs it. So the honest summary is: **the `ci-agent` account can act as root
> on this server.** The limit is who can log in to the panel, not the sudo
> rules. Protect the admin password and your 2FA devices accordingly.

If the sudo rule is missing, the privileged button fails fast and the page tells
you so. The manual copy-paste commands still work. The agent re-checks and
repairs the allowlist on every start, so it heals itself if the file drifts.
TLS cert/key PEMs are written by the unprivileged agent into its own data dir
(nginx's master reads them), so cert issuance needs no `sudo`. See
[Domains & TLS](./domains.md).

### Server shell and File Manager

Both the **host File Manager** (`/files`) and the **Server shell** are
**root-equivalent**. Use them as you would a root SSH login:

- The **File Manager** works as the `ci-agent` user first. When it hits a file
  it may not touch, it retries through `fs-helper` as root. So it can read,
  edit and delete any file on the host.
- The **Server shell** runs as the `ci-agent` user. That account can run
  `sudo /usr/bin/ci-agent …` without a password (see above), and it is in the
  `docker` group, which is root-equivalent on its own (e.g. `docker run
  --privileged`).
- For normal admin work, the shell's **Open host SSH session** button logs you
  in as a real admin account through `sshd`, with your own password. See
  [the UI guide](./ui.md).
- The **container** File Manager and shell are limited to containers that carry
  the project's compose label — the same boundary as `docker cp` /
  `docker exec`.

## Self-update integrity

The Maintenance page's update verifies the `.deb` **offline** against the
release public key embedded in the binary, pinned to a single fingerprint — a
valid signature from any other key is rejected — and refuses downgrades. The
online buttons (**Update now**, **Download & stage latest**) download from the
public apt repository but go through the exact same check, so the network is
never trusted. Uploading the `ci-agent-offline-*.tar.gz` bundle is the same
check: the `.deb` and its detached signature are unpacked from it and the
bundle's own bundled public key is ignored, so trust always rests on the
embedded key, never one that travels with the upload. The privileged binary swap runs only from systemd's root
pre-start step, and a crash-looping new binary is rolled back automatically.
See [the UI guide](./ui.md).

## systemd sandbox

The shipped unit runs the agent as `User=ci-agent`, `Group=docker`, and sets
`PrivateTmp=yes` (a private `/tmp`). Other common sandbox options are
**deliberately not set**:

- **`NoNewPrivileges`** — not set, because the agent's privileged pages work
  through `sudo` (see *Managing nginx and TLS* above), and `sudo` needs to gain
  privileges.
- **`ProtectSystem` / `ProtectHome`** — set to `false`. A `sudo` child inherits
  the service's read-only mounts, so with them even the root `fs-helper` could
  not write outside a short list of folders. The File Manager, disk reports and
  cleanups need the whole filesystem, so the helper's own checks are the real
  boundary instead.

On every start the agent also writes a small drop-in,
`/etc/systemd/system/ci-agent.service.d/10-ci-agent.conf`. It re-applies these
settings on boxes that were updated from the panel, and adds
`AmbientCapabilities=CAP_NET_BIND_SERVICE`. That lets the agent listen on ports
80 and 443 itself — used only by the tunnel relay in **Online** mode (see
[Reverse tunnel](./tunnel.md)); with tunnelling off it changes nothing.

The privileged `apply-staged-update` pre-start step is the only component that
writes `/usr/bin`.

## Common problems and fixes

| Problem | Fix |
|---|---|
| Everyone was logged out | Normal after a service restart, server reboot or self-update — sessions live in memory. Log in again. |
| A webhook URL leaked | Click **Rotate token** on the project's **Deployments** tab, then update the URL wherever you use it. |
| Lost every 2FA device and all backup codes | Run `sudo ci-agent reset-2fa`, then `sudo systemctl restart ci-agent`, and enroll again at the next login. |
| A Host, Firewall or File Manager action fails with a `sudo` / "password is required" error | The sudo allowlist is missing or was rejected. Restart the service (it repairs the file on start), then check `journalctl -u ci-agent` for a `visudo` error. |
| An API key leaked | Click **Revoke** next to it on the **API keys** page. It stops working on the next request. |
