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
> Once installed, the agent updates *itself* from the Maintenance page using
> the `github_token` config key — no manual API calls needed.

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

```bash
sudo apt-get install -y nginx
sudo systemctl enable --now nginx
nginx -v
```

### Verify everything

```bash
docker --version && docker compose version && nginx -v && git --version && curl --version | head -1
```

## 2. Install ci-agent

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

## 3. First login

The agent listens on `127.0.0.1:8044` by default. From your workstation:

```bash
ssh -L 8044:127.0.0.1:8044 you@your-server
```

Open <http://127.0.0.1:8044> and **set the admin password immediately** —
until you do, anyone who can reach the port can claim the instance.

The env-encryption key is generated at `/etc/ci-agent/secret.key` on first
start; **back it up** together with the database (Maintenance page).

To expose the UI publicly, put nginx + TLS in front — config in
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

(Requires the Docker apt repository configured on the connected machine —
see section 1.)

### ci-agent package (connected machine)

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

## 5. Upgrading ci-agent

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
built-in self-update flow for non-apt installs.

## 6. Building from source (alternative)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env
git clone https://github.com/m-hoseinzadeh/ci-agent.git && cd ci-agent
cargo build --release
sudo install -m 0755 target/release/ci-agent /usr/bin/ci-agent
```
