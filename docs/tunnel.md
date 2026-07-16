# Reverse tunnel

The **tunnel** lets a ci-agent box that has **no public IP** — one behind NAT, a
home router, or a locked-down office network — serve its apps on the public
internet through **another** ci-agent box that does have a public IP.

The private box **never opens an inbound port**. It dials *out* to the public
box and keeps the connection open; visitors reach the public box, and their
traffic is passed straight through to the private box.

> **What this means.** Normally a server can only be reached if the internet can
> connect *to* it. A box behind NAT can't be reached that way. The tunnel flips
> it around: the private box connects *out* to a public helper, and that helper
> forwards visitors back down the same connection. No port-forwarding, no public
> IP on the private box.

## The two boxes (roles)

Every ci-agent install has a **Role**, set on the **Tunnel** page in the
sidebar (`/tunnel`). While tunneling is off, the page shows two cards — pick
the one that matches this box:

| Role | What the box becomes |
|---|---|
| **Off** | A normal ci-agent. No tunneling. (The default.) |
| **Online** (relay) | A public box that is **both** a normal ci-agent **and** a relay for private boxes. |
| **Local** (dial out) | A private/NAT'd box that **dials out** to an Online box. |

Picking a card only opens that role's settings — nothing changes yet. Each
role's screen has an **On/Off switch** at the top: turn it on to activate the
role, turn it off to go back to a plain ci-agent (and back to the two cards).
You can fill in everything **before** switching on; staged tunnels, routes and
connection settings take effect the moment the switch flips. While a role is
active, a link above its screen still opens the **other** role's settings, so
you can stage or tidy them without turning anything off — switching on from
there replaces the active role directly.

- **Online** = the public helper. It keeps hosting its own apps *and* forwards
  traffic for the private boxes that connect to it.
- **Local** = the private box. It reaches the internet only by dialling out to
  an Online box.
- **Off** = tunneling disabled — a plain ci-agent.

You set the role in the UI; the agent applies it. No reinstall and no
command-line flags.

### Important: the relay never decrypts your traffic

TLS (HTTPS) is terminated on the **private (Local)** box, where your
certificates live. The Online box forwards the raw encrypted stream and
**never decrypts it** — it only reads the hostname the visitor asked for, to
decide which tunnel to send them down. The public box needs **no certificate**
for your tunneled apps.

## What Online mode does to the public box

When you switch a box to **Online**, the ci-agent relay takes over the public
ports **443**, **80** and **7000**:

- **443 / 80** — visitor traffic. Requests for a **routed** hostname go down the
  tunnel to the matching private box; **everything else** falls through to this
  box's **own** nginx, so the Online box keeps serving its own apps normally.
- **7000** — the control port that private (Local) boxes dial into.

The box's own nginx is quietly moved to loopback (`127.0.0.1:8443` / `:8080`)
so the relay can own the public front door. Your existing sites keep working;
they are only affected if you deliberately route one of their exact hostnames
down a tunnel (which the UI blocks — see below).

Switching to Online also tries to open UFW ports **7000, 443 and 80** for you.
If port 7000 is still closed, the Tunnel page warns:

> Port 7000 is closed in the firewall — agents can't connect. Open it on the
> Firewall page.

## Set up a tunnel

You work on **both** boxes: mint a tunnel token on the Online box, then paste it
into the Local box.

### On the Online (public) box

1. Pick the **Online** card and turn the switch **on**. (You can also stage
   steps 2–4 first and switch on afterwards.)
2. Under **Add tunnel**, fill:
   - **Id** — a short name the Local box will present (e.g. `box-a`). 1–63
     characters: `a–z`, `0–9`, `-`, `_`, `.`.
   - **Label** (optional) — a human note, e.g. `Warehouse edge box`.
   Click **Add tunnel**.
3. A card appears with three values — **Tunnel id**, **Token**, and **Relay
   fingerprint** — and this warning:

   > Copy these now — the token is shown only once and is never stored in
   > cleartext.

   Copy all three. You need them on the Local box. (If you lose the token, use
   the **Rotate** button on the tunnel's row to mint a new one — the old one
   stops working immediately.)
4. Under **Add route**, tell the relay which hostnames belong to this tunnel:
   - **Hostname** — an exact host (`app.example.com`) or a leftmost wildcard
     (`*.example.com`). It must **not** clash with a hostname this box's own
     apps already serve.
   - **Tunnel** — pick the tunnel you just added.
   - **PROXY protocol** (optional) — see [below](#proxy-protocol).
   Click **Add route**. Add as many routes as you like; anything not routed
   here is served by this box's own nginx.
5. **Point DNS** for each routed hostname at **this** (the Online box's) public
   IP.

### On the Local (private) box

1. Pick the **Local** card. The switch stays disabled until the connection
   details below are saved — fill them in first.
2. In the **Connection settings** card, enter the values from the Online box:
   - **Online server** — the relay's address as `host:port`, e.g.
     `relay.example.com:7000` (port 7000 is the default control port).
   - **Tunnel id** — the id you created (e.g. `box-a`).
   - **Token** — the one-time token. (Leave blank to keep the current one.)
   - **Relay fingerprint** — the SHA-256 fingerprint from the Online box. This
     pins the relay's certificate; the agent refuses to connect if it doesn't
     match, so a fake relay can't impersonate yours.
   - **Ports** (default `443,80`) — an advisory list for the Online operator.
     Keep **80** so per-host HTTPS certificate renewal can work through the
     tunnel. (The relay forwards each visitor to whichever port they arrived on
     regardless.)
3. Click **Save settings**, then turn the switch **on**. The box dials out and
   connects.

A **Connection status** card on the Local box shows **connected** (with the
time since) or **disconnected**, plus a **Reconnect now** button and the last
error if a connection failed.

## Certificates through the tunnel

Your tunneled apps keep their certificates on the **Local** box, exactly like a
normal ci-agent — issue and renew them on the Local box's
[Certificates](./domains.md) page. Because port **80** is relayed down the
tunnel, **Let's Encrypt HTTP-01** renewals keep working automatically. The
Online box needs no certificate for tunneled hostnames.

## Statuses you'll see

- **Role header** (top of the page): the picked role with an `off` or `active`
  chip, next to the On/Off switch.
- **Tunnels table** → *Status*: **connected** or **offline**.
- **Routes table** → *Backend*: **live** (a connected tunnel is behind it) or
  **no backend** (the tunnel hasn't connected yet).
- **Local Connection status**: **connected** / **disconnected**.

## PROXY protocol

By default, a tunneled app sees the connection as coming from the relay, not
from the real visitor. Turn on **PROXY protocol** (per route on the Online box,
and matched in **Local settings**) to have the relay prepend a small header so
the private box can recover the **real visitor IP**. When you enable it on the
Local box you must also set the **Relay IP** (the Online box's public IP), which
is trusted for `set_real_ip_from`.

> **Gotcha.** Once PROXY protocol is on for the Local box, its nginx expects the
> header on **every** connection — so the box is then reachable **only** through
> the tunnel. Direct connections to it will break. Enable it on both ends or
> neither.

## Common problems and fixes

- **Local box won't connect** — check the **Online server** address and port
  (7000 by default), that the token and fingerprint match, and that port
  **7000** is open in the Online box's firewall.
- **"Port 7000 is closed in the firewall"** — open it on the
  [Firewall](./ui.md) page of the Online box.
- **"… is already served by a local app on this box"** — you tried to route a
  hostname the Online box's own app already owns. Remove it from that app first,
  or pick a different hostname.
- **A route shows "no backend"** — the Local box for that tunnel hasn't
  connected yet. Check its Connection status.
- **A mode switch fails** — the agent tests nginx before committing and rolls
  back to the previous working state if the test fails; the reason is shown in
  the flash message.
