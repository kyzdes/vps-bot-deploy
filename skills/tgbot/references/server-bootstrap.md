# Server Bootstrap Reference

SSH key setup and base package installation. This is the minimal server preparation — applies to ALL deploy methods.

For Dokploy-specific setup (install, admin, project), see `dokploy-setup.md`.

---

## SSH Key Setup

### Install sshpass (macOS)

```bash
which sshpass || brew install hudochenkov/sshpass/sshpass
```

If brew is not available or sshpass fails to install, use the fallback approach below.

### Generate SSH key

```bash
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```

### Copy key to server

```bash
sshpass -p 'ROOT_PASSWORD' ssh-copy-id -o StrictHostKeyChecking=no root@SERVER_IP
```

### Verify key-based login

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 root@SERVER_IP "echo ok"
```

If this returns "ok", SSH key auth is working. The password is no longer needed.

### Fallback (no sshpass)

Ask the user to run one command manually:

```bash
ssh-copy-id root@SERVER_IP
```

They'll enter their password once. After that, key auth works.

---

## Base Package Installation

Install essential packages needed for any deploy method:

```bash
ssh root@SERVER_IP "apt-get update && apt-get install -y python3 python3-pip python3-venv git curl jq"
```

These are needed because:
- `python3`, `python3-pip`, `python3-venv` — most bots are Python; also useful for scripting
- `git` — cloning repos (systemd method clones directly on server)
- `curl` — API calls, health checks
- `jq` — parsing JSON responses

### Additional packages (install on demand)

Node.js (if deploying a Node.js bot via systemd):
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs
```

Go (if deploying a Go bot via systemd):
```bash
wget -q https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
ln -sf /usr/local/go/bin/go /usr/bin/go
rm go1.22.0.linux-amd64.tar.gz
```

---

## Server Health Checks

Quick checks to run on a new server:

### Disk space
```bash
ssh root@SERVER_IP "df -h /"
```
Minimum: 2GB free for systemd, 5GB free for Dokploy (Docker images).

### Memory
```bash
ssh root@SERVER_IP "free -m"
```
Minimum: 512MB free for a simple bot.

### OS version
```bash
ssh root@SERVER_IP "cat /etc/os-release | grep PRETTY_NAME"
```
Ubuntu 20.04+ or Debian 11+ recommended.
