# Installation

Every command needed to go from a bare Ubuntu server to a running ci-agent.
Two paths: **online server** (sections 1–3) and **air-gapped server**
(section 4). Commands assume Ubuntu 22.04/24.04 x86_64 and a sudo-capable
user.

Releases are built automatically from every commit on `main`:
**<https://github.com/m-hoseinzadeh/ci-agent/releases>** — the
`…-x86_64-linux-musl` binary is fully static and runs on any x86_64 Linux.

> **Private repository note:** the plain `curl -fsSLO …/releases/latest/download/…`
> commands below only work if the repo is public. While it is private, download
> with a token via the API instead:
>
> ```bash
> TOKEN=ghp_yourtoken
> API=https://api.github.com/repos/m-hoseinzadeh/ci-agent/releases/latest
> URL=$(curl -fsS -H "Authorization: Bearer $TOKEN" $API \
>   | grep -o '"url": *"[^"]*assets/[0-9]*"' | head -1 | grep -o 'https[^"]*')
> # pick the asset you need by listing names first:
> curl -fsS -H "Authorization: Bearer $TOKEN" $API | grep '"name"'
> curl -fsSL -H "Authorization: Bearer $TOKEN" -H "Accept: application/octet-stream" \
>   -o ci-agent-x86_64-linux-musl.tar.gz "$URL"
> ```
>
> Once installed, the agent updates *itself* from the Maintenance page — the
> update check reads the latest version from the public apt repository, so no
> token is needed.

## 1. Install requirements

### Docker Engine + Compose plugin

```bash
# prerequisites + Docker's apt repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl git
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# enable + verify
sudo systemctl enable --now docker
sudo docker run --rm hello-world
docker compose version
```

### nginx

Current stable nginx from the official nginx.org apt repository (the distro
package lags behind), following
[nginx.org/en/linux_packages.html](https://nginx.org/en/linux_packages.html#Ubuntu):

```bash
# prerequisites
sudo apt-get install -y curl gnupg2 ca-certificates lsb-release ubuntu-keyring

# import the nginx signing key
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg > /dev/null

# verify the key — the output must contain the fingerprint
# 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
gpg --dry-run --quiet --no-keyring --import --import-options import-show \
  /usr/share/keyrings/nginx-archive-keyring.gpg

# add the stable repository and pin it above the distro package
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list
printf "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
  | sudo tee /etc/apt/preferences.d/99nginx

# install
sudo apt-get update
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
nginx -v
```

nginx.org packages load sites from `/etc/nginx/conf.d/*.conf` and have **no**
`sites-available`/`sites-enabled`. All commands the agent generates (setup
wizard, per-project nginx guide) write to `conf.d`, which Ubuntu's distro
package also includes — both layouts work.

### Verify everything

```bash
docker --version && docker compose version && nginx -v && git --version && curl --version | head -1
```

## 2. Install CI Agent

### Option A — apt (recommended for online servers)

Every `main` commit is published as a signed package to the public apt
repository (binaries only — the source stays private):

```bash
# trust the signing key + add the repository (one time)
sudo curl -fsSL https://m-hoseinzadeh.github.io/ci-agent-apt/ci-agent.gpg \
  -o /usr/share/keyrings/ci-agent.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ci-agent.gpg] https://m-hoseinzadeh.github.io/ci-agent-apt ./" \
  | sudo tee /etc/apt/sources.list.d/ci-agent.list

# install — creates the ci-agent user, installs the systemd unit and
# /etc/ci-agent/config.toml (kept as a conffile across upgrades)
sudo apt-get update
sudo apt-get install -y ci-agent
sudo systemctl start ci-agent
systemctl status ci-agent --no-pager
```

Signing key fingerprint: `A80F7309CF26133DBE88E2EC9C5489AAE86BF159`.

### Option B — manual tarball

```bash
# download the latest release (binary + systemd unit + example config)
curl -fsSLO https://github.com/m-hoseinzadeh/ci-agent/releases/latest/download/ci-agent-x86_64-linux-musl.tar.gz
tar xzf ci-agent-x86_64-linux-musl.tar.gz

# binary
sudo install -m 0755 ci-agent/ci-agent /usr/bin/ci-agent
ci-agent --help

# dedicated user (in the docker group) + directories
sudo useradd -r -s /usr/sbin/nologin -G docker ci-agent
sudo mkdir -p /etc/ci-agent /var/lib/ci-agent

# configuration (defaults are sane; see /docs/configuration)
sudo cp ci-agent/deploy/config.example.toml /etc/ci-agent/config.toml
sudo chown -R ci-agent: /var/lib/ci-agent /etc/ci-agent

# systemd service
sudo cp ci-agent/deploy/ci-agent.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ci-agent
systemctl status ci-agent --no-pager
```

## 3. First-run setup

The packaged config listens on `0.0.0.0:8044` so the setup wizard is
reachable at `http://<server-ip>:8044` right after install (the built-in
default without a config file stays `127.0.0.1:8044`). The wizard enforces,
in order:

1. **Set the admin password** — do this immediately; until you do, anyone
   who can reach the port can claim the instance.
2. **Bind a domain** for the admin panel — a LAN DNS name or `/etc/hosts`
   entry (e.g. `ci.example.internal`) works on air-gapped networks.
3. **Install the generated nginx site** — copy-paste commands; the agent
   never touches nginx itself.
4. **Open `http://<your-domain>/`** — when the first request arrives through
   nginx (Host matches, peer is loopback, `X-Forwarded-For` present), the
   agent rewrites `listen` to `127.0.0.1:8044` in its config and restarts.
   From then on the panel is unreachable from outside the server except
   through nginx; the rest of the UI stays locked until this completes.
5. **Set up two-factor authentication (2FA)** — on your first login the panel
   shows a QR code. Scan it with an authenticator app (e.g. Google
   Authenticator) and enter the 6-digit code to finish. **Save the 10 backup
   codes** it shows — each one logs you in once if you lose your phone. 2FA is
   required; the panel stays locked until you enroll.

> **Note:** the QR code is drawn by the agent itself, so 2FA setup works fully
> offline. Lost your phone with no backup codes left? Run `ci-agent reset-2fa`
> on the server and log in to enroll again. See [Security](./security.md).

Prefer to never expose the port, even briefly? Set
`listen = "127.0.0.1:8044"` *before* the first start and run the wizard
through an SSH tunnel:

```bash
ssh -L 8044:127.0.0.1:8044 you@your-server
```

The nginx step still applies — verification happens via a local request
through the proxy.

The env-encryption key is generated at `/etc/ci-agent/secret.key` on first
start; **back it up** together with the database (Maintenance page).

For TLS, extend the generated server block — config in
[Security](./security.md).

## 4. Air-gapped server

Do section 1's downloads on a **connected machine with the same Ubuntu
release**, transfer, install offline.

### Docker + nginx as .deb files (connected machine)

```bash
mkdir debs && cd debs

# resolve and download the full dependency closure
sudo apt-get update
apt-get download $(apt-cache depends --recurse --no-recommends --no-suggests \
  --no-conflicts --no-breaks --no-replaces --no-enhances \
  docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin nginx git curl \
  | grep "^\w" | sort -u)

cd .. && tar czf debs.tar.gz debs
scp debs.tar.gz you@airgapped-server:
```

(Requires the Docker **and nginx.org** apt repositories configured on the
connected machine — see section 1; the pin in `99nginx` makes the closure
resolve to nginx.org's nginx.)

### CI Agent package (connected machine)

```bash
# easiest: the .deb (installs user + unit + config in one step)
curl -fsSL https://m-hoseinzadeh.github.io/ci-agent-apt/Packages \
  | grep -m1 ^Filename | awk '{print "https://m-hoseinzadeh.github.io/ci-agent-apt/" $2}' \
  | xargs curl -fsSLO
scp ci-agent_*.deb you@airgapped-server:
```

### Base images your projects need (connected machine)

```bash
docker pull nginx:1.31            # whatever your compose files use
docker save nginx:1.31 -o images.tar
scp images.tar you@airgapped-server:
```

### On the air-gapped server

```bash
# packages
tar xzf debs.tar.gz && cd debs && sudo dpkg -i *.deb; cd ..
sudo systemctl enable --now docker nginx

# base images
sudo docker load -i images.tar

# ci-agent
sudo dpkg -i ci-agent_*.deb
sudo systemctl start ci-agent
```

The default `pull_policy = "never"` means deployments never try to reach a
registry — they use exactly the images you loaded.

## 5. Upgrading CI Agent

apt installs:

```bash
sudo apt-get update && sudo apt-get install --only-upgrade ci-agent
```

Manual installs:

```bash
curl -fsSLO https://github.com/m-hoseinzadeh/ci-agent/releases/latest/download/ci-agent-x86_64-linux-musl
sudo install -m 0755 ci-agent-x86_64-linux-musl /usr/bin/ci-agent
sudo systemctl restart ci-agent
```

Air-gapped: fetch the new `.deb` on the connected machine, `scp`, `sudo dpkg
-i`. All state survives upgrades (`/etc/ci-agent/config.toml` is a conffile);
the running release is shown on the Maintenance page, which also offers a
built-in self-update flow for non-apt installs. There you can upload the whole
`ci-agent-offline-*.tar.gz` release bundle (the agent pulls the `.deb` and its
signature out of it), or the `.deb` and its detached `.asc` separately; either
way the signature is verified offline against the embedded release key before
the binary is staged.

> **Self-update needs one line in the service unit.** Applying a staged
> binary is done by a root pre-start step, `ExecStartPre=+-/usr/bin/ci-agent
> apply-staged-update`, in `ci-agent.service`. Packages built before this
> shipped don't have it, and the self-update only ever swaps the binary — it
> never rewrites the unit file — so an already-deployed box keeps the old
> unit even after updating. If "Apply & restart" leaves you on the old
> version (check **System & versions** on the Maintenance page), add the step
> once with a drop-in:
>
> ```bash
> sudo mkdir -p /etc/systemd/system/ci-agent.service.d
> printf '[Service]\nExecStartPre=+-/usr/bin/ci-agent apply-staged-update\n' \
>   | sudo tee /etc/systemd/system/ci-agent.service.d/self-update.conf
> sudo systemctl daemon-reload
> # apply the binary already staged by the failed attempt, then restart:
> sudo /usr/bin/ci-agent apply-staged-update
> sudo systemctl restart ci-agent
> ```
>
> Confirm with `systemctl cat ci-agent | grep ExecStartPre`. The drop-in lives
> outside the packaged unit, so it survives both self-updates and later `dpkg`
> upgrades. Fresh installs already include the step and need none of this.

## 6. Building from source (alternative)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env
git clone https://github.com/m-hoseinzadeh/ci-agent.git && cd ci-agent
cargo build --release
sudo install -m 0755 target/release/ci-agent /usr/bin/ci-agent
```
