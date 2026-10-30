# Domains & TLS

Each project is reached through host nginx. The agent manages the nginx server
block and (optionally) the TLS certificates for you; nginx itself stays the
plain host service you installed.

## Domains per project

A project can serve **many hostnames**. Add them on the project's **Domains**
tab. Type a bare hostname (for example `app.example.lan`) in **Add domain** and
click **Add**. You can also add a wildcard such as `*.example.com` (see
*Wildcard domains* below).

Every change on this tab is written to nginx and nginx is reloaded
**automatically**. You do not need to apply anything by hand.

![screenshot: the Domains tab with the domain table and the Add domain form](TODO)

Each domain is one row in the table. The columns are:

- **Primary** — pick at most one primary domain per port. The other domains on
  that port then **301-redirect** to it (good for `www → apex`, or for retiring
  an old name). Pick the last row, **None — every domain serves the app
  directly**, to turn redirects off. A wildcard domain can't be the primary.
- **Domain** — the hostname. Edit it in place and click **Rename** to change it.
  The primary domain shows a **primary** chip.
- **Port** — only shown when the project has extra ports. Pick another port from
  the list to move the domain there.
- **HTTPS** — the certificate state of this hostname (see below), plus a
  **Manage** or **Add SSL** button that opens the domain's SSL page.
- **Maintenance** — tick it to serve a "Down for maintenance" page for this
  domain instead of the app. The container keeps running. The row then shows an
  **offline** chip. Untick it to bring the site back.
- **×** — delete the domain. nginx is re-applied.

If the project has **extra ports**, the **Add domain** form also has a port
list, so you can point different hostnames at different services in the same
project.

### What the HTTPS column shows

| Chip / button | Meaning |
|---|---|
| **HTTPS** | HTTPS is on for this domain. Hover to see the certificate source and expiry date. |
| **covered** | A wildcard certificate covers this host, but HTTPS is not switched on yet. |
| **ready** | A certificate covers this host, but HTTPS is not switched on yet. |
| **renew** | The certificate is within its renewal window (30 days before expiry). |
| **Add SSL** | No certificate covers this host yet. Click it to get one. |

To switch HTTPS on or off, open the domain's SSL page (**Manage**) and use the
**Serve HTTPS** switch. See *The per-domain SSL page* below.

> **What this means.** A domain only gets HTTPS when two things are true: a
> certificate covers it, **and** HTTPS is switched on for it. You cannot switch
> HTTPS on before a certificate exists — ci-agent refuses and tells you to
> create one first.

## Wildcard domains

A wildcard domain sends **every subdomain of a zone** to one project. Type it in
**Add domain** as `*.` plus the zone, for example `*.example.com`. Then
`shop.example.com`, `blog.example.com` and so on all reach this project.

Rules:

- Only the first label can be `*`, and at least two labels must follow it.
  `*.example.com` works; `*.com`, `*example.com` and `a.*.example.com` do not.
- Each wildcard belongs to one project, like any other domain.
- A host that **another project names exactly** still goes to that project.
  For example, with `*.example.com` on project `sites` and `ci.example.com` on
  project `ci`, `ci.example.com` goes to `ci`. nginx always prefers an exact
  name. Default URLs (`<slug>.<base>`) are exact names too, so they keep
  working.
- `*.example.com` does not match `example.com` itself. Add `example.com` as
  its own domain if you want it.
- A wildcard can't be the **Primary** domain, because it is not one host. It
  can still redirect to another primary domain.

**HTTPS for a wildcard** needs a **wildcard certificate** for the zone (it
covers `example.com` and `*.example.com`). On the domain's SSL page:

- **Let's Encrypt** — use **Start wildcard order** (DNS-01). HTTP-01 can't
  issue a wildcard, so it is not offered. Pick a DNS provider so it renews
  automatically.
- **Internal / Self-signed** and **Upload** — only the **Wildcard** scope is
  offered.

With HTTPS on, plain HTTP is redirected to HTTPS on the same host the visitor
asked for.

> **What this means.** A wildcard certificate covers one level only:
> `shop.example.com`, but not `a.shop.example.com`. nginx still sends the deeper
> name to the project, but over HTTPS the browser shows a certificate error.

## Catch-all: any other host

Normally a request for a host that no project has gets a **404 page** ("Nothing
is here yet"). You can send these requests to **one project** instead. This is
for platforms where customers point **their own domains** at your server (for
example through Cloudflare for SaaS): the app sees the customer's host in the
`Host` header and decides what to show.

On the project's **Domains** tab, tick **Receive requests for any other host
(catch-all)**. nginx is updated at once.

- Only **one project** on the server can have the catch-all. If another project
  has it, the box is greyed out and names that project; turn it off there
  first.
- The project needs an **HTTP port**. Requests go to its **primary port**, with
  that port's **Proxy tuning**.
- Hosts that any project names (exact or wildcard), default URLs, and the admin
  panel's domain keep going where they went before. Only unknown hosts reach
  the catch-all.
- `/.well-known/acme-challenge/` still goes to ci-agent, so Let's Encrypt
  HTTP-01 keeps working for new domains.
- **HTTPS:** the catch-all answers on port 443 only when a wildcard certificate
  covers the base domain (the same rule as the 404 page). It then uses that
  certificate, which does not match customers' domains. Behind Cloudflare,
  reach the server over plain HTTP (the Flexible SSL mode), or use a Cloudflare
  mode that does not check the origin certificate.
- Untick the box to go back to the 404 page.

ci-agent writes this into `/etc/nginx/conf.d/00-ci-agent-catchall.conf`. The
`00-` name makes it nginx's default server. If another nginx file declares
`default_server` on port 80 (for example the stock
`/etc/nginx/sites-enabled/default`), that file wins and the catch-all gets no
requests. Remove that default site.

## Default URLs: subdomain and path

Set a global **base domain** on the main **Settings** page (sidebar), in the
**Default subdomain** card. With one set, each project **port** can be reached
automatically in two independent ways. You switch them per port in the
**Default URL** column of the **Ports** card, on the project's **Settings** tab:

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

### Optional base path prefix

By default path routes sit at the apex root (`<base>/<slug>/`). Set an optional
**Base path** in the same **Default subdomain** card to nest them all under a
shared prefix instead — e.g. `/p` makes project `a-b` reachable at
`<base>/p/a-b/`. It applies globally to every path route (subdomains are
unaffected), so the apex root and other paths on the host stay free — handy when
ci-agent shares the hostname with something else or sits behind another front
proxy that routes only `/p/*` to it. Blank serves path routes at the root.

### Single-domain mode: panel and projects on one host

You can run **everything on one domain** — the admin panel at the root and your
projects on paths under it. Set the **Base domain** (Settings → Default
subdomain) to the **same hostname your admin panel uses**. ci-agent then serves:

- the **admin panel** at `https://<domain>/`, and
- each path-enabled project at `https://<domain>/<base-path>/<slug>/`.

**What this means:** one hostname, one certificate, no extra DNS records. Good
for a small box where you do not want a separate domain per app.

**You must set a base path first.** Because the panel lives at the root, projects
need a prefix so their URLs cannot clash with the panel's own pages. Set a **Base
path** (e.g. `/p`) under Settings → Default subdomain **before** turning on Path
for any project. If you try to expose a project on a path without a base path,
ci-agent refuses with:

> Set a base path in Settings before exposing a project on a path — the admin
> panel shares this domain, so projects need a path prefix (e.g. /p) to avoid
> colliding with it

**How to turn it on:**

1. On the main **Settings** page, in **Default subdomain**, set **Base path** to
   `/p` (or any prefix you like) and save.
2. In the same card, set **Base domain** to your admin panel's hostname and save.
3. Open the project, go to its **Settings** tab, and in the **Ports** card tick
   **Path** for the port you want to publish.

The project is now at `https://<domain>/p/<slug>/`, and the panel stays at
`https://<domain>/`.

**No manual nginx work.** ci-agent folds the project path routes into the admin
panel's own nginx config (a single server block for the shared host), so the two
never collide. Toggling Path on a project updates it automatically.

**HTTPS on the shared domain.** ci-agent adds the HTTPS part to the shared
config itself. It does this only when **all** of these are true:

- a **Base path** is set;
- at least **one** project port has **Path** ticked;
- a certificate covers the domain — a wildcard for the zone, or a dedicated
  certificate named exactly for the domain;
- no raw admin config is set (see the note below).

If any of these is missing, the panel uses its normal config, and the shared
domain stays plain HTTP.

> **Note — Advanced (raw) admin config.** If you use the **Advanced — edit the
> full config (can lock you out)** editor for the admin panel on the **Host**
> page, ci-agent cannot fold path routes into your hand-written block, so **path
> routing is turned off** while a raw override is set. Remove the raw override to
> use single-domain mode, or give each project its own subdomain instead.

## Ports

A project always has one **primary HTTP port**. You can add **extra ports** in
the **Ports** card on the project's **Settings** tab — useful when one project
exposes more than one service (for example an app on one port and its metrics on
another). You attach domains to a port on the **Domains** tab.

Each port gets:

- its own **slug** (a short DNS-safe name) that names both of its default URLs
  (the `<slug>.<base>` subdomain and the `<base>/<slug>/` path — see above);
- its own **domains** (the host port they proxy to); and
- its own **proxy tuning** (the next section).

## Per-port nginx tuning

On the **Domains** tab, each port has a collapsible **Proxy tuning** panel. The
knobs are baked into the generated server block and apply to **every** domain
that proxies to that port (including its default URL):

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
**Extra directives** can otherwise break the vhost until corrected. A port with
custom settings shows a **tuned** chip.

> **What this means.** "Per-port" tuning is set once per port and reused by all
> of that port's domains, because every hostname on a port reaches the same
> backend service. If you have extra ports, tune each one separately.

## The nginx guide page

ci-agent applies nginx for you on every domain, port, or HTTPS change. You only
need this page to check things or to repair a config by hand.

Click **nginx guide** at the top of the project page. It shows:

- a **Status** card: whether nginx is installed and running, whether the file
  `/etc/nginx/conf.d/<slug>.conf` is present, and **matches expected** — `yes`,
  or **no (drift or missing)** when the file on disk differs from what ci-agent
  generates;
- the **Current config on disk**; and
- the **Expected server block**.

Two ways to apply it:

- **Apply config & reload** — writes the expected server block to disk, runs
  `nginx -t`, and reloads nginx. A failed test rolls the file back and skips the
  reload. It uses the narrow `sudo -n` allowlist installed by the package (see
  [Security](./security.md)). No password prompt; if the allowlist is missing
  the call fails fast and the manual commands still work.
- **Manual steps (no agent privileges needed)** — the same steps as commands, for
  hosts where you'd rather run them by hand.

## Certificates

A certificate is keyed by a **base domain** and is either:

- **Wildcard** — covers `{base, *.base}` (e.g. `example.lan` and
  `app.example.lan`) and is **shared** by every project whose domains fall under
  that base. One wildcard typically covers a whole air-gapped zone.
- **Dedicated** — covers exactly one hostname.

Cert and key PEMs live under the agent's data dir and are read directly by
nginx's master process, so issuing a cert needs **no** `sudo` (unlike writing
the nginx *config*).

### Sources

| Source | Use it for | Trust |
|---|---|---|
| **Internal CA** | air-gapped zones — issue trusted wildcards offline | install the CA root (`/ca.crt`) on clients once; browsers then trust every cert it issues |
| **Self-signed** | quick internal use | browsers show a warning until the cert is trusted |
| **Upload** | a cert+key pair you already have | whatever issued it |
| **Let's Encrypt** | internet-facing hosts | publicly trusted, no client setup |

For an air-gapped zone, the internal CA is the easiest choice: download its root
once from **`/ca.crt`**, install it on the machines that browse your services,
and every certificate the agent issues from it is trusted with no per-cert
steps.

### The per-domain SSL page

Most of the time you get a certificate from the domain itself. On the
**Domains** tab, click **Add SSL** (or **Manage**) on the domain's row. This
opens the page **SSL · <domain>**.

![screenshot: the SSL page of one domain, with the Certificate card and the Issue a certificate tabs](TODO)

The **Certificate** card shows the current state when a certificate covers the
host:

- **Status** — **HTTPS on**, or **certificate present — HTTPS off**.
- **Source**, **Covers**, and **Expires** (shown in red when within the renewal
  window).
- **Serve HTTPS** — the switch that turns HTTPS on or off for this domain.
  With HTTPS on, plain HTTP requests are redirected to HTTPS.
- **Renew now** — re-issues the certificate right away. Shown for certificates
  that renew automatically.
- For uploads and manual DNS-01 wildcards you see instead: *Renews manually —
  re-issue before expiry (uploads, or manual DNS-01 orders).*
- For a wildcard issued through a DNS provider you see: *Renews automatically —
  DNS records published through* (the provider's name).

If the host is covered by a **wildcard** that belongs to its zone, the card says
so. You manage that certificate from its base domain or on the **Certificates**
page.

The **Issue a certificate** card has three tabs. **Let's Encrypt** opens first.

1. **Let's Encrypt** — for domains with public DNS that the internet can reach.
   - **Issue via HTTP-01** — a dedicated certificate for this host, validated
     over HTTP on port 80. **Renews automatically.**
   - **Start wildcard order** — a wildcard for `*.<base>` by DNS-01. In
     **TXT records**, pick a DNS provider and ci-agent adds the DNS TXT records
     itself, so the wildcard **renews automatically**. Or pick **Manual** to add
     them by hand; then it renews manually (see *Let's Encrypt* below).
2. **Internal / Self-signed** — generated on the server, fully offline, and
   renewed automatically. Pick the **Source** (Internal CA or Self-signed) and the
   **Scope** (**Wildcard** for the zone and all its subdomains, or **Dedicated**
   for just this host), then click **Issue & enable HTTPS**.
3. **Upload** — paste a certificate you got elsewhere (**Certificate (fullchain
   PEM)** and **Private key (PEM)**), pick the **Scope**, and click **Upload &
   enable HTTPS**. Not renewed automatically — upload a new one before expiry.

Issuing a certificate from this page also **switches HTTPS on** for this domain.
Other domains under a new wildcard show **covered** on the Domains tab; switch
each one on with **Serve HTTPS** on its own SSL page.

### The Certificates page

The **Certificates** page (sidebar) is the global overview.

![screenshot: the Certificates page with the Internal CA card and the Managed certificates table](TODO)

- **Internal CA** — shows **root present** once the CA exists, with **Download
  root CA (root.crt)**. The CA is created automatically the first time you issue
  an Internal CA certificate.
- **Managed certificates** — every certificate with its **Name**, **Covers**,
  **Source**, and **Expires** date. A **manual renew** chip marks certificates
  that do not renew by themselves. An **auto · <provider>** chip marks a
  wildcard that renews through a DNS provider. **Renew** re-issues a certificate
  now (only for ones that renew automatically). **×** deletes it; the domains it
  covered go back to HTTP.
- **Let's Encrypt** — the account settings (see the next section), plus
  **Wildcard for a zone (DNS-01)**: enter a **Base domain**, pick who adds the
  **TXT records**, and click **Start wildcard order**.
- **DNS providers** — API access to your DNS host, so wildcard orders add their
  TXT records themselves (see *DNS providers* below).
- **Add an offline certificate** — create an Internal CA, Self-signed, or
  uploaded certificate for any **Name**, with **Wildcard** or **Dedicated**
  scope, then click **Create certificate**.

## Let's Encrypt

On the **Certificates** page, set the **Account email** (optional, used for
expiry notices). Turn on **Use staging environment** while testing, to avoid
production rate limits. Click **Save Let's Encrypt settings**. The account key is
created on first use and stored under the data dir. Two challenge types:

- **HTTP-01** (single host, fully automatic) — the agent answers the challenge
  at `/.well-known/acme-challenge/<token>`, which nginx proxies to it for every
  project domain. Issuance **and renewal** need no operator action; daily
  housekeeping renews these within 30 days of expiry and re-applies nginx.
- **DNS-01 through a DNS provider** (wildcard, fully automatic) — pick the
  provider in **TXT records** and click **Start wildcard order**. ci-agent adds
  the TXT records with the provider's API, waits until the zone's nameservers
  serve them, has Let's Encrypt check them, and then deletes them. Issuance
  **and renewal** need no operator action, like HTTP-01. If the order is not
  done after 45 seconds, the page says it is still running; reload in a minute
  or two. If it fails, the reason is in the **Audit log**.
- **DNS-01 by hand** (wildcard, manual) — pick **Manual** in **TXT records**.
  After you click **Start wildcard order**, the page shows a **Finish the
  wildcard order (DNS-01)** card with the TXT record(s) to add. Publish them at
  your DNS host, wait for them to spread, then click **Validate & issue**. A
  wildcard needs **two** TXT records with the same name and different values —
  add **both**; do not replace one with the other. This path is **not**
  auto-renewable, so plan to renew these wildcards by hand.

> Let's Encrypt only works when the host is reachable from Let's Encrypt's
> servers (HTTP-01) or you control public DNS (DNS-01), and when this server can
> reach Let's Encrypt over HTTPS. If it can't (a firewall or a national network
> blocks it), issuing stops after 15 seconds with "could not reach Let's
> Encrypt". On an air-gapped or blocked network, use the **internal CA**, or
> bring a public certificate over with *Offline certificates* below.

### DNS providers

A DNS provider lets ci-agent add the `_acme-challenge` TXT records for a
Let's Encrypt wildcard itself, so the wildcard renews automatically 30 days
before it expires. Two are supported: **Cloudflare** and **ArvanCloud**. The
server must reach the provider's API and Let's Encrypt over HTTPS.

1. Get an API token:
   - **Cloudflare** — **My Profile → API Tokens → Create Token**. Give it
     **Zone → Zone → Read** and **Zone → DNS → Edit**, for your zone or all zones.
   - **ArvanCloud** — **Profile → API keys** (a machine user's key). You can
     paste it with or without the leading `Apikey`.
2. On the **Certificates** page, in **DNS providers**, pick the **Provider**,
   give it a **Name** (optional), paste the **API token**, and click
   **Save & test**. ci-agent checks the token and lists the zones it can see.
   Saving an existing name replaces its token.
3. Start a wildcard order and pick the provider in **TXT records**.

**Test** checks a saved token again. **×** deletes a provider; wildcards that
renewed through it then renew manually. A wildcard you already have from a
manual order keeps renewing manually until you issue it again through a
provider: start a new wildcard order for the same zone and pick the provider.

## Offline certificates

Use this when an **air-gapped** server needs a **publicly trusted** certificate.
You need a second ci-agent server that can reach the internet, and you must
control the domain's public DNS. Open **Offline certs** in the sidebar on both
servers.

![screenshot: the Offline certificates page with Issue & export, Import a bundle, and the Offline certificates table](TODO)

**On the online server:**

1. In **Issue & export (online server)**, enter the **Domain**, pick the
   **Scope** (**Wildcard** or **Dedicated**) and who adds the **TXT records**.
   Click **Start order**.
2. With a DNS provider, ci-agent finishes the order itself. With **Manual**,
   the **Finish the order (DNS-01)** card shows the TXT record(s). Add them at
   your DNS host, wait for them to spread, then click **Validate & issue**.
3. In the **Offline certificates** table, click **Download** on the new row. You
   get a file named `ci-agent-cert-<domain>.zip` (the certificate, its key, and a
   manifest).
4. Carry the file to the air-gapped server (USB stick, `scp` through a jump
   host, and so on).

**On the air-gapped server:**

1. In **Import a bundle (offline server)**, choose the `.zip` as the **Bundle
   file** and click **Import bundle**.
2. ci-agent checks the certificate, stores it, and re-applies nginx. It is listed
   as **Uploaded**.
3. If a domain is not on HTTPS yet, open its SSL page and switch on **Serve
   HTTPS**.

**Renewing:**

1. On the **online** server: a certificate issued through a DNS provider renews
   by itself (marked **auto · <provider>**). For a manual one, click **Renew** on
   the row; this starts a fresh DNS-01 order, which you finish as above. Then
   **Download** the new bundle.
2. On the **air-gapped** server, import the new bundle. A newer bundle for the
   same domain replaces the old one in place.

> **What this means.** The air-gapped server never talks to Let's Encrypt. The
> online server does the public part; you carry the finished certificate over by
> hand. You repeat this before each expiry.

The table on this page lists uploads and Let's Encrypt DNS-01 certificates.
**×** deletes one; its domains go back to HTTP.

## Renewal

Generated certs (internal CA, self-signed), LE HTTP-01 certs, and LE DNS-01
wildcards issued through a DNS provider are re-issued automatically by daily
housekeeping once they fall within **30 days** of expiry. Uploaded and imported
certs and LE DNS-01 wildcards from a manual order are manual. The
Certificates page shows each cert's expiry, and the Domains tab shows a **renew**
chip, so you can renew before it lapses.

For those manual certificates, ci-agent also sends a `cert_expiring` alert (once
per expiry date) when one enters the 30-day window. Set up the alert hook as
described in the **Notifications** section of [Operations](./operations.md).

## With the API

An **admin** API key can do most of this page without the panel (see
[API & AI agents](./api.md)):

| In the panel | With the API |
|---|---|
| **Add domain** (also `*.zone`), **×** on the Domains tab | `POST` and `DELETE /api/v1/projects/{slug}/domains…` |
| **Receive requests for any other host (catch-all)** | `PATCH /api/v1/projects/{slug}` with `{"catch_all": true}` |
| **Serve HTTPS** switch | `PATCH /api/v1/projects/{slug}/domains/{id}` with `{"tls": true}` |
| **Issue a certificate** card: Let's Encrypt HTTP-01, Let's Encrypt DNS-01 through a DNS provider, Internal CA, Self-signed, Upload | `POST /api/v1/projects/{slug}/domains/{id}/certificate` |
| **Default URL** column (Subdomain, Path) of the Ports card | `PATCH /api/v1/projects/{slug}/ports/{port}` with `subdomain` / `path` |
| **Proxy tuning** panel | `PATCH /api/v1/projects/{slug}/ports/{port}` with `nginx` |

Picking the **Primary** domain, manual Let's Encrypt DNS-01 orders, DNS
providers, the Certificates page, and adding or removing ports stay in the panel.

## The admin panel's own TLS

The steps above cover your *projects*. Once a domain is bound, the admin panel
is served by host nginx from `/etc/nginx/conf.d/ci-agent.conf`. ci-agent
generates this file, and the generated version has **no HTTPS part** (except in
single-domain mode — see above).

Do **not** edit that file by hand. ci-agent rewrites it — for example when you
click **Apply & reload** in the **Admin panel nginx** card, or when single-domain
mode is in use — and your edits are lost.

To put the panel on HTTPS, use the supported editor:

1. Get a certificate that covers the panel's domain (for example on the
   **Certificates** page).
2. Open the **Host** page, card **Admin panel nginx**.
3. Open **Advanced — edit the full config (can lock you out)**. Start from the
   text under **What will be written**, add a `listen 443 ssl` server block with
   your `ssl_certificate` and `ssl_certificate_key` paths, and click **Apply raw
   config**.
4. A syntax error is caught by `nginx -t` and rolled back. A wrong `proxy_pass`
   or `server_name` still passes the test and can cut you off — recover over SSH
   or click **Reset to generated default**.

Remember that a raw override turns off single-domain path routing (see the note
above). See also [Security](./security.md).

## Common problems and fixes

| Problem | Fix |
|---|---|
| "No certificate covers … — create one (base …) on the Certificates page first" | You tried to turn on HTTPS without a certificate. Issue one on the domain's SSL page first. |
| A domain shows **covered** or **ready**, but the site is still HTTP | A certificate exists, but HTTPS is off for this domain. Open **Manage** and switch on **Serve HTTPS**. |
| "Domain already in use" | Another project already has this hostname. Remove it there first. |
| "Invalid domain — a bare hostname … or a wildcard like *.example.com" | Type only the host: no `http://`, no port, no path. A wildcard is `*.` plus at least two labels. |
| "A wildcard domain can't be the primary …" | Pick an exact host as the primary, or **None**. |
| "Let's Encrypt can't issue a wildcard over HTTP-01 …" | Use **Start wildcard order** (DNS-01) for a wildcard domain. |
| "Project '…' already receives the catch-all; turn it off there first" | Only one project can have the catch-all. Untick it on the other project. |
| Customers' domains still get the 404 page with the catch-all on | Another nginx file is the default server (for example `/etc/nginx/sites-enabled/default` with `default_server`). Remove it and reload nginx. |
| "'…' is routed to tunnel '…' — remove that route first" | This box is a tunnel relay and that hostname goes down a tunnel. Delete the route on the **Tunnel** page first — see [Reverse tunnel](./tunnel.md). |
| "Set a base path in Settings before exposing a project on a path …" | Single-domain mode needs a **Base path** (e.g. `/p`) before any project can use **Path**. |
| An app under a path loads without styles or redirects to `/` | The app uses absolute URLs. Give it a configurable base path, or use **Subdomain** instead. |
| 504 errors on slow requests, or uploads fail | Raise **Proxy timeout (s)** or **Max upload (MB)** in the port's **Proxy tuning**. |
| "Imported certificates renew by re-issuing on the online server and re-importing the bundle." | You clicked **Renew** on an imported cert. Renew it on the online server and import the new bundle. |
| "Could not import bundle: …" | The file is not a ci-agent bundle, or it is damaged. Download it again on the online server. |
