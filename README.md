# ci-agent apt repository

Binary-only apt repository for [ci-agent](https://github.com/m-hoseinzadeh/ci-agent)
(source repository is private). Packages are built and signed automatically by CI.

## Use

```bash
sudo curl -fsSL https://m-hoseinzadeh.github.io/ci-agent-apt/ci-agent.gpg \
  -o /usr/share/keyrings/ci-agent.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ci-agent.gpg] https://m-hoseinzadeh.github.io/ci-agent-apt ./" \
  | sudo tee /etc/apt/sources.list.d/ci-agent.list
sudo apt-get update
sudo apt-get install -y ci-agent
```

Upgrades arrive with regular `sudo apt-get upgrade`.

Signing key fingerprint: `A80F7309CF26133DBE88E2EC9C5489AAE86BF159`
