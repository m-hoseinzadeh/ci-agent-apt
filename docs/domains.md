# Domains & TLS

Each project is reached through host nginx. The agent manages the nginx server
block and (optionally) the TLS certificates for you; nginx itself stays the
plain host service you installed.

## Domains per project

A project can serve **many hostnames**. Add them on the project page; each row
has two toggles:

- **Canonical** — at most one domain per project. When a canonical is set, the
  other domains **301-redirect** to it (good for `www → apex`, or retiring an
  old name). With no canonical set, every domain serves the app directly.
- **HTTPS** — opt this hostname into TLS. It only takes effect once a
  certificate that covers the host exists (see below); until then the toggle is
  armed but the site stays HTTP.

## Default subdomain (`<slug>.<base>`)

Set a global **base domain** under **Settings → Default subdomain** and every
project automatically gets a `<slug>.<base>` hostname (e.g. project `blog` with
base `example.lan` → `blog.example.lan`) — no per-project typing. Each project
page shows the generated host with its own **enable** toggle (on by default);
unchecking it stops serving the project there. If a **wildcard certificate**
for the zone exists, the default subdomain is served over HTTPS automatically;
otherwise it stays plain HTTP. Leaving the base domain blank disables the
feature for all projects. Explicitly-added domains keep working alongside it.

## Per-project nginx tuning

The project's edit form exposes a few knobs that are baked into its generated
server block:

| Knob | nginx directive | Default |
|---|---|---|
| **Max upload size** | `client_max_body_size` | 1 MB |
| **Proxy timeout** | `proxy_connect/send/read_timeout` | 60 s |
| **WebSockets** | adds `Upgrade`/`Connection` headers + HTTP/1.1 | off |
| **Extra directives** | raw lines added to the proxy `location` block | — |

Raise the upload size for large file uploads, the proxy timeout for slow or
long-running requests (avoids spurious 504s), and enable WebSockets for
long-lived WS/SSE connections. Changes are validated with `nginx -t` on save and
rolled back if invalid — a typo in **Extra directives** can otherwise break the
vhost until corrected.

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
