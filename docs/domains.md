# Domains & TLS

Each project is reached through host nginx. The agent manages the nginx server
block and (optionally) the TLS certificates for you; nginx itself stays the
plain host service you installed.

## Domains per project

A project can serve **many hostnames**. Add them on the **Domains & Ports** tab
of the project page. Each domain has:

- **Canonical** — at most one canonical domain per port. When a canonical is
  set, the other domains on that port **301-redirect** to it (good for
  `www → apex`, or retiring an old name). With no canonical set, every domain
  serves the app directly.
- **HTTPS** — opt this hostname into TLS. It only takes effect once a
  certificate that covers the host exists (see below); until then the toggle is
  armed but the site stays HTTP.

If the project has **extra ports**, the add-domain form also asks **which port**
the domain should proxy to, so you can point different hostnames at different
services in the same project.

Use **Rename** to change a hostname, **Manage** / **Add SSL** to handle its
certificate, and the **×** button to remove it.

## Default URLs: subdomain and path

Set a global **base domain** under **Settings → Default subdomain**. With one
set, each project **port** can be reached automatically in two independent ways,
toggled per port in the **Default URL** column of the **Ports** card:

- **Subdomain** — `<slug>.<base>` (e.g. project `blog` with base `example.lan`
  → `blog.example.lan`). **On by default.**
- **Path** — `<base>/<slug>/` (e.g. `example.lan/blog/`). **Off by default.**

The primary port's segment is the project slug; each extra port uses its own
port slug. Tick either box — or both — per port; each change is applied to nginx
immediately. If a **wildcard certificate** for the zone exists, both forms are
served over HTTPS automatically; otherwise they stay plain HTTP. Leaving the base
domain blank disables default URLs for every project. Explicitly-added domains
keep working alongside them.

### Choosing subdomain vs path

- **Subdomain** gives each app its **own origin**, so the browser isolates
  cookies, `localStorage`, and CORS between projects. This is the safer default.
- **Path** puts the app on the **shared apex origin** (`<base>`), so every
  path-routed app shares one cookie jar and `localStorage`. Use it when you want
  a single hostname, but know that path-routed apps can see each other's
  browser-stored data — which is why it is opt-in per port.
- Path routing **strips the `/<slug>` prefix** before forwarding and sends an
  `X-Forwarded-Prefix` header. An app works under a path only if it uses
  **relative URLs** or supports a configurable base path; apps that hardcode `/`
  for assets or redirects will break. Subdomain mode has no such constraint.
- Each path segment must be **unique across all projects** (the apex path
  namespace is shared) — enabling one that another project already claims is
  refused.

## Ports

A project always has one **primary HTTP port**. You can add **extra ports** on
the **Domains & Ports** tab — useful when one project exposes more than one
service (for example an app on one port and its metrics on another).

Each port gets:

- its own **slug** (a short DNS-safe name) that names both of its default URLs
  (the `<slug>.<base>` subdomain and the `<base>/<slug>/` path — see above);
- its own **domains** (the host port they proxy to); and
- its own **proxy tuning** (the next section).

## Per-port nginx tuning

On the **Domains & Ports** tab, each port has a collapsible **Proxy tuning**
panel. The knobs are baked into the generated server block and apply to **every**
domain that proxies to that port (including its default URL):

| Knob | nginx directive | Default |
|---|---|---|
| **Max upload (MB)** | `client_max_body_size` | 256 MB |
| **Proxy timeout (s)** | `proxy_connect/send/read_timeout` | 300 s |
| **Forward WebSocket / SSE upgrades** | adds `Upgrade`/`Connection` headers + HTTP/1.1 | off |
| **Extra directives** | raw lines added to the proxy `location` block | — |

Raise the upload size for large file uploads, the proxy timeout for slow or
long-running requests (avoids spurious 504s), and turn on the WebSocket / SSE
toggle for long-lived connections. Click **Save proxy settings**. Changes are
validated with `nginx -t` on save and rolled back if invalid — a typo in
**Extra directives** can otherwise break the vhost until corrected.

> **What this means.** "Per-port" tuning is set once per port and reused by all
> of that port's domains, because every hostname on a port reaches the same
> backend service. If you have extra ports, tune each one separately.

## Applying the nginx config

The project's **nginx** page shows the generated server block, the live service
status, and a **drift badge** comparing what's generated against what's on
disk. Two ways to apply it:

- **One-click apply** — the agent writes the config, runs `nginx -t`, and
  reloads, via a narrow `sudo -n` allowlist (`tee` the conf, `nginx -t`,
  `systemctl reload nginx`) installed by the package. No password prompt; if the
  allowlist is missing the call fails fast and the manual commands still work.
- **Copy-paste** — the same commands are shown for hosts where you'd rather run
  them by hand.

Re-apply whenever you add a domain, change the canonical, or toggle HTTPS — the
drift badge tells you when the on-disk config is stale.

## Certificates

TLS certs are managed on the **Certificates** page (and per-domain SSL pages).
A cert is keyed by a **base domain** and is either:

- **Wildcard** — covers `{base, *.base}` (e.g. `example.lan` and
  `app.example.lan`) and is **shared** by every project whose domains fall under
  that base. One wildcard typically covers a whole air-gapped zone.
- **Single-host** — covers exactly one hostname, for maximum specificity.

Cert and key PEMs live under the agent's data dir and are read directly by
nginx's master process, so issuing a cert needs **no** `sudo` (unlike writing
the nginx *config*).

### Sources

| Source | Use it for | Trust |
|---|---|---|
| **Internal CA** | air-gapped zones — issue trusted wildcards offline | install the CA root (`/ca.crt`) on clients once; browsers then trust every cert it issues |
| **Self-signed** | quick internal use | clients see a warning unless you import the leaf |
| **Upload** | a cert+key pair you already have | whatever issued it |
| **Let's Encrypt** | internet-facing hosts | publicly trusted, no client setup |

The internal CA is the offline-friendly default: download its root once from
**`/ca.crt`**, install it on the machines that browse your services, and every
wildcard the agent issues from it is trusted with no per-cert steps.

## Let's Encrypt

Set the account **email** (and optionally toggle the **staging** environment
for testing) on the Certificates page. Account credentials are created on first
use and stored per-environment under the data dir. Two challenge types:

- **HTTP-01** (single host, fully automatic) — the agent answers the challenge
  at `/.well-known/acme-challenge/<token>`, which nginx proxies to it for every
  project domain. Issuance **and renewal** need no operator action; nightly
  housekeeping renews these within 30 days of expiry and re-applies nginx.
- **DNS-01** (wildcard, manual) — the agent computes the `_acme-challenge` TXT
  record value(s); you publish them at your DNS provider and click **Continue**.
  This path is **not** auto-renewable (housekeeping skips it and the UI flags
  it), so plan to renew wildcard LE certs by hand.

> Let's Encrypt only works when the host is reachable from Let's Encrypt's
> servers (HTTP-01) or you control public DNS (DNS-01). On an air-gapped
> network, use the **internal CA** instead.

## Renewal

Generated certs (internal CA, self-signed, LE HTTP-01) are re-issued
automatically by nightly housekeeping once they fall within **30 days** of
expiry. Uploaded certs and LE **DNS-01** wildcards are manual — the
Certificates page shows each cert's expiry so you can renew before it lapses.

## The admin panel's own TLS

The steps above cover your *projects*. The admin panel itself is bound to
`127.0.0.1` behind nginx; to put it on HTTPS, extend its server block (created
by the setup wizard) with a certificate — see [Security](./security.md).
