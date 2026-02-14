# Systemd Deployment Reference

Deploy a bot as a systemd service — no Docker dependency. This is the default and simplest method.

---

## Overview

The bot runs directly on the server as a systemd service:
- Code lives in `/opt/<bot-name>/`
- Virtual environment (Python) or node_modules in the same directory
- `.env` file for configuration
- systemd unit file for process management (auto-restart, logs via journalctl)

---

## Step 1: Install Runtime

Detect language and install the required runtime if not present.

### Python

```bash
ssh root@SERVER_IP "python3 --version 2>/dev/null || (apt-get update && apt-get install -y python3 python3-pip python3-venv)"
```

### Node.js

```bash
ssh root@SERVER_IP "node --version 2>/dev/null || (curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && apt-get install -y nodejs)"
```

### Go

Go bots are compiled — no runtime needed on server. Build locally or use `go build` on server:

```bash
ssh root@SERVER_IP "go version 2>/dev/null || (wget -q https://go.dev/dl/go1.22.0.linux-amd64.tar.gz && tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz && ln -sf /usr/local/go/bin/go /usr/bin/go && rm go1.22.0.linux-amd64.tar.gz)"
```

---

## Step 2: Clone Repository

```bash
ssh root@SERVER_IP "mkdir -p /opt && cd /opt && git clone https://github.com/OWNER/REPO.git BOT_NAME"
```

If directory already exists (redeployment):
```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git fetch origin && git reset --hard origin/BRANCH"
```

---

## Step 3: Install Dependencies

### Python (requirements.txt)

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt"
```

### Python (pyproject.toml + poetry)

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && python3 -m venv .venv && .venv/bin/pip install poetry && .venv/bin/poetry config virtualenvs.create false && .venv/bin/poetry install --no-dev"
```

### Python (pyproject.toml + uv)

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && pip install uv && uv venv .venv && uv sync --frozen --no-dev"
```

### Node.js

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && npm ci --production"
```

### Node.js (bun)

```bash
ssh root@SERVER_IP "which bun || (curl -fsSL https://bun.sh/install | bash) && cd /opt/BOT_NAME && bun install --production"
```

### Go

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && go build -o bot ."
```

---

## Step 4: Write .env File

```bash
ssh root@SERVER_IP "cat > /opt/BOT_NAME/.env << 'ENVEOF'
BOT_TOKEN=xxx
OTHER_VAR=yyy
ENVEOF"
```

Set restrictive permissions:
```bash
ssh root@SERVER_IP "chmod 600 /opt/BOT_NAME/.env"
```

---

## Step 5: Create Systemd Unit File

### Python Bot

```bash
ssh root@SERVER_IP "cat > /etc/systemd/system/BOT_NAME.service << 'EOF'
[Unit]
Description=BOT_NAME Telegram Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/BOT_NAME
EnvironmentFile=/opt/BOT_NAME/.env
ExecStart=/opt/BOT_NAME/.venv/bin/python -m ENTRY_MODULE
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF"
```

**Detecting ENTRY_MODULE:** Look for the entry point:
- `bot/__main__.py` exists → module is `bot`
- `main.py` exists → use `/opt/BOT_NAME/.venv/bin/python main.py` instead of `-m`
- `app.py` exists → use `/opt/BOT_NAME/.venv/bin/python app.py`
- `run.py` exists → use `/opt/BOT_NAME/.venv/bin/python run.py`

### Node.js Bot

```bash
ssh root@SERVER_IP "cat > /etc/systemd/system/BOT_NAME.service << 'EOF'
[Unit]
Description=BOT_NAME Telegram Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/BOT_NAME
EnvironmentFile=/opt/BOT_NAME/.env
ExecStart=/usr/bin/node ENTRY_FILE
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF"
```

**Detecting ENTRY_FILE:** Check `package.json` → `main` or `scripts.start`:
- `"main": "src/index.js"` → use `src/index.js`
- `"start": "node dist/index.js"` → use `dist/index.js`
- Default: `src/index.js` or `index.js`

### Go Bot

```bash
ssh root@SERVER_IP "cat > /etc/systemd/system/BOT_NAME.service << 'EOF'
[Unit]
Description=BOT_NAME Telegram Bot
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/BOT_NAME
EnvironmentFile=/opt/BOT_NAME/.env
ExecStart=/opt/BOT_NAME/bot
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF"
```

---

## Step 6: Enable and Start

```bash
ssh root@SERVER_IP "systemctl daemon-reload && systemctl enable BOT_NAME && systemctl start BOT_NAME"
```

---

## Verification

### Check service is running

```bash
ssh root@SERVER_IP "systemctl is-active BOT_NAME"
```

Expected: `active`

### Check recent logs

```bash
ssh root@SERVER_IP "journalctl -u BOT_NAME -n 20 --no-pager"
```

Look for:
- Error patterns: `Traceback`, `Error`, `FATAL`, `ModuleNotFoundError`, `ImportError`
- Success patterns: `Bot started`, `Polling`, `Running`, `Listening`

### Check service status details

```bash
ssh root@SERVER_IP "systemctl status BOT_NAME --no-pager"
```

---

## Update / Redeploy

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git pull origin BRANCH && .venv/bin/pip install -r requirements.txt && systemctl restart BOT_NAME"
```

For Node.js:
```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git pull origin BRANCH && npm ci --production && systemctl restart BOT_NAME"
```

---

## Rollback

```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git log --oneline -5"
# Pick a commit hash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git checkout COMMIT_HASH && systemctl restart BOT_NAME"
```

---

## Remove / Delete

```bash
ssh root@SERVER_IP "systemctl stop BOT_NAME && systemctl disable BOT_NAME && rm /etc/systemd/system/BOT_NAME.service && systemctl daemon-reload && rm -rf /opt/BOT_NAME"
```

---

## SQLite Considerations

Many bots use SQLite for local storage. With systemd:
- SQLite DB file lives alongside the bot code in `/opt/BOT_NAME/`
- No external database service needed
- Backups: `ssh root@SERVER_IP "cp /opt/BOT_NAME/data.db /opt/BOT_NAME/data.db.bak"`
- If the bot has hardcoded paths like `./data/bot.db`, it works because `WorkingDirectory` is set

### Detecting SQLite usage

Look for in the source code:
- Python: `sqlite3`, `aiosqlite`, `sqlalchemy` with `sqlite:///`
- Node.js: `better-sqlite3`, `sqlite3`, `sql.js`
- Go: `mattn/go-sqlite3`, `modernc.org/sqlite`
