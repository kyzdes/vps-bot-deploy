# /tgbot — Deploy, Manage & Prepare Telegram Bots on VPS

## Description

Universal skill for the full lifecycle of Telegram bots on a VPS: deploy, manage, and prepare repos for deployment. Combines three workflows into one command with subcommand routing.

**Deploy methods:**
- **systemd** (default) — runs the bot directly on the server as a systemd service. No Docker dependency. Best for polling bots using SQLite.
- **dokploy** — runs the bot in Docker via Dokploy. Best for webhook bots needing Traefik, or bots needing shared PostgreSQL/Redis.

## Usage

- `/tgbot` — interactive: show menu of available actions
- `/tgbot deploy <repo-url>` — deploy a Telegram bot to VPS
- `/tgbot deploy` — deploy interactively (asks for repo)
- `/tgbot manage` — show status of all bots
- `/tgbot manage status` — show status of all bots
- `/tgbot manage logs <name>` — show logs for a specific bot
- `/tgbot manage redeploy <name>` — pull latest code and restart
- `/tgbot manage env <name>` — view/edit environment variables
- `/tgbot manage stop <name>` — stop a bot
- `/tgbot manage start <name>` — start a stopped bot
- `/tgbot manage delete <name>` — remove a bot entirely
- `/tgbot manage list` — list all deployed bots
- `/tgbot prepare <repo-url>` — prepare a repo for deployment
- `/tgbot prepare` — prepare interactively (asks for repo)

## Instructions

### Step 0: Route Subcommand

Parse the arguments passed to the skill.

**If no arguments:** Show interactive menu:
```
What would you like to do?

1. Deploy a bot (/tgbot deploy)
2. Manage deployed bots (/tgbot manage)
3. Prepare a repo for deployment (/tgbot prepare)
```

**Routing rules:**
- First argument is `deploy` → go to **Deploy Flow**
- First argument is `manage` → go to **Manage Flow**
- First argument is `prepare` → go to **Prepare Flow**
- First argument looks like a URL (`https://...`) → treat as `/tgbot deploy <url>`

---

## Deploy Flow

Deploy a Telegram bot to a VPS. On first run, sets up the server. On subsequent runs, skips server setup and deploys the bot.

### Phase 0: Pre-Flight Checks

Follow `references/preflight-checks.md`.

**0.1** Read config from: `~/.claude/projects/<current-project>/memory/deploy-config.md`

**0.2** If config exists:
- Verify SSH works: `ssh -o BatchMode=yes -o ConnectTimeout=5 root@IP "echo ok"`
- If method=dokploy, verify Dokploy API is reachable

**0.3** If method=dokploy (configured or will be chosen):
- Check Docker Hub rate limit: `ssh root@IP "docker pull hello-world 2>&1"` — parse for `toomanyrequests`
- If rate-limited, warn and suggest systemd

**0.4** If repo provided, validate: `gh api repos/OWNER/REPO --jq '.default_branch'`

**0.5** Report ALL issues at once. Let user decide how to proceed.

---

### Phase 1: Server Access

**1.1** If SSH already works (from config or pre-flight) — skip to Phase 2

**1.2** Ask user for:
- Server IP address
- Root password

Do NOT ask for Dokploy API key, Cloudflare token, or any other infrastructure details.

**1.3** Set up SSH key authentication — follow `references/server-bootstrap.md`:
1. Check if `sshpass` is installed, install via brew if not
2. Generate SSH key if `~/.ssh/id_ed25519` doesn't exist
3. Copy key to server via `sshpass -p 'PASSWORD' ssh-copy-id`
4. Verify key-based login: `ssh -o BatchMode=yes root@IP "echo ok"`

**Fallback:** If sshpass unavailable, ask user to run `ssh-copy-id root@IP` manually.

**1.4** Install base packages:
```bash
ssh root@IP "apt-get update && apt-get install -y python3 python3-pip python3-venv git curl jq"
```

**1.5** After SSH is working, **do not store or reuse the password**.

---

### Phase 2: Analyze Repo

**2.1** Get repo URL (from argument or ask user). Extract owner, repo name, default branch:
```bash
gh api repos/OWNER/REPO --jq '.default_branch'
```

Bot name defaults to repo name (lowercased, hyphens for underscores).

**2.2** Detect bot characteristics:
- **Language:** Check for `requirements.txt`, `package.json`, `go.mod`, etc.
- **Framework:** Read dependency files for aiogram, grammy, telegraf, etc.
- **Entry point:** `bot/__main__.py`, `main.py`, `package.json` main, etc.
- **Database type:**
  - SQLite: `sqlite3`, `aiosqlite`, `better-sqlite3` in deps
  - PostgreSQL: `asyncpg`, `psycopg2`, `pg` in deps, or `DATABASE_URL` patterns
  - Redis: `redis`, `aioredis`, `ioredis` in deps
  - None: no database indicators
- **Polling vs webhook:** Check source for `set_webhook`, `express`, `fastapi` (webhook) vs `start_polling`, `run_polling` (polling)

**2.3** Detect secrets — follow `references/secrets-detection.md`:
- Read `.env.example`, `README.md`, `docker-compose.yml`, scan source
- Classify: Secret (ask user), Infrastructure (auto-fill IF bot needs it), Default, Unknown

**IMPORTANT:** Only auto-fill infrastructure vars (DATABASE_URL, REDIS_URL) if the bot actually uses that database. SQLite bots don't need shared PostgreSQL.

**2.4** Check deployment readiness:
- Has Dockerfile? Has `.env.example`? Any hardcoded paths?
- If not ready, suggest running `/tgbot prepare` first — OR proceed with systemd (which doesn't need a Dockerfile)

**2.5** Present analysis and recommend deploy method:

| Bot DB | Bot Type | Recommendation |
|--------|----------|---------------|
| SQLite / none | polling | **systemd** (default) |
| SQLite / none | webhook | dokploy (for Traefik routing) |
| PostgreSQL | any | dokploy (for shared PG container) |

User can always override the recommendation.

**2.6** Collect secret values from user. Present only what needs user input:

```
I analyzed your repo and found these environment variables:

Secrets (need your values):
  - BOT_TOKEN — Telegram bot token
  - OPENAI_API_KEY — OpenAI API key

Auto-configured:
  - PYTHONUNBUFFERED = 1

Please provide values for the secrets listed above.
```

---

### Phase 3: Setup Environment

#### Path A — Systemd (default)

Follow `references/systemd-deployment.md`:

**3A.1** Install runtime if not present:
- Python: `apt-get install -y python3 python3-pip python3-venv`
- Node.js: NodeSource setup + `apt-get install -y nodejs`
- Go: download and install

**3A.2** Create bot directory:
```bash
ssh root@IP "mkdir -p /opt/BOT_NAME"
```

**3A.3** Clone repo:
```bash
ssh root@IP "cd /opt && git clone https://github.com/OWNER/REPO.git BOT_NAME"
```
If directory exists: `cd /opt/BOT_NAME && git fetch origin && git reset --hard origin/BRANCH`

**3A.4** Install dependencies:
- Python: `cd /opt/BOT_NAME && python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`
- Node.js: `cd /opt/BOT_NAME && npm ci --production`
- Go: `cd /opt/BOT_NAME && go build -o bot .`

**3A.5** Write .env file:
```bash
ssh root@IP "cat > /opt/BOT_NAME/.env << 'ENVEOF'
KEY=value
ENVEOF"
ssh root@IP "chmod 600 /opt/BOT_NAME/.env"
```

**3A.6** Create systemd unit file — see `references/systemd-deployment.md` for templates per language.

**3A.7** Enable and start:
```bash
ssh root@IP "systemctl daemon-reload && systemctl enable BOT_NAME && systemctl start BOT_NAME"
```

#### Path B — Dokploy

Follow `references/dokploy-setup.md` and `references/dokploy-api.md`:

**3B.1** Detect if Dokploy already installed:
```bash
ssh root@IP "docker ps --format '{{.Names}}' 2>/dev/null | grep -q dokploy && echo 'installed' || echo 'not-installed'"
```

**3B.2** If not installed — install Dokploy (see `references/dokploy-setup.md`)

**3B.3** Detect protocol (HTTP vs HTTPS):
- Try `http://IP:3000` first, then `https://IP:3000` with `-k`
- Use whichever responds

**3B.4** Check if admin exists:
- Try `POST /api/auth.createAdmin` — if error/401 — admin exists, ask user for API key
- If success — save credentials, generate API token

**3B.5** Check Docker Hub rate limit:
```bash
ssh root@IP "docker pull hello-world 2>&1"
```
If `toomanyrequests` — warn user, suggest switching to systemd

**3B.6** Find or create project. **GET environmentId from project.one response:**
```bash
# Get project detail
curl -sk "PROTOCOL://IP:3000/api/project.one?input=..." -H "x-api-key: KEY"
# Extract: .environments[0].environmentId
```

**3B.7** Create shared services ONLY if bot needs PG/Redis:
- Include `environmentId` in all create calls
- Read actual `appName` (with random suffix) from response
- Use actual appName in connection URLs

---

### Phase 4: Deploy & Verify

#### Systemd

**4.1** Check service is running:
```bash
ssh root@IP "systemctl is-active BOT_NAME"
```

**4.2** Read logs, look for errors:
```bash
ssh root@IP "journalctl -u BOT_NAME -n 30 --no-pager"
```

**4.3** Look for error patterns: `Traceback`, `Error`, `FATAL`, `ModuleNotFoundError`
Look for success patterns: `Bot started`, `Polling`, `Running`

**4.4** If errors found, see `references/error-recovery.md` and fix.

#### Dokploy

**4.1** Create application — include `environmentId`:
```bash
curl -sk -X POST "PROTOCOL://IP:3000/api/application.create" \
  -H "x-api-key: KEY" -H "Content-Type: application/json" \
  -d '{"name":"BOT_NAME","appName":"BOT_NAME","projectId":"PID","environmentId":"EID","description":"BOT_NAME Telegram bot"}'
```

**CRITICAL:** Read actual `appName` from response (Dokploy appends random 6-char suffix).

**4.2** Set source as git — include `githubId`:
```bash
curl -sk -X POST "PROTOCOL://IP:3000/api/application.saveGithubProvider" \
  -H "x-api-key: KEY" -H "Content-Type: application/json" \
  -d '{"applicationId":"AID","repository":"https://github.com/OWNER/REPO","branch":"BRANCH","owner":"OWNER","buildPath":"/","githubId":""}'
```

**4.3** Set build type — include `dockerContextPath` and `dockerBuildStage`:
```bash
curl -sk -X POST "PROTOCOL://IP:3000/api/application.saveBuildType" \
  -H "x-api-key: KEY" -H "Content-Type: application/json" \
  -d '{"applicationId":"AID","buildType":"dockerfile","dockerfile":"Dockerfile","dockerContextPath":"/","dockerBuildStage":""}'
```

If no Dockerfile in repo, use `"buildType":"nixpacks"` instead.

**4.4** Set environment variables:
```bash
curl -sk -X POST "PROTOCOL://IP:3000/api/application.saveEnvironment" \
  -H "x-api-key: KEY" -H "Content-Type: application/json" \
  -d '{"applicationId":"AID","env":"KEY1=val1\nKEY2=val2"}'
```

**4.5** Create domain (webhook bots only)

**4.6** Deploy and poll status (max 5 minutes):
```bash
curl -sk -X POST "PROTOCOL://IP:3000/api/application.deploy" \
  -H "x-api-key: KEY" -H "Content-Type: application/json" \
  -d '{"applicationId":"AID"}'
```

Poll `deployment.all` for `status: "done"` or `status: "error"`.

On error — read build logs, help debug.

---

### Phase 5: Save Config

Update `~/.claude/projects/<current-project>/memory/deploy-config.md`:

```markdown
# Deploy Configuration

## Server
- IP: <ip>
- SSH: key-based

## Dokploy (only if used)
- Protocol: http|https
- URL: <protocol>://<ip>:3000
- API Key: <key>
- Project ID: <id>
- Environment ID: <id>

## Shared PostgreSQL (only if created)
- Postgres ID: <id>
- Actual App Name: pg-telegram-bots-XXXXXX
- Internal URL: postgresql://botuser:<pass>@<actual-app-name>:5432/bots

## Shared Redis (only if created)
- Redis ID: <id>
- Actual App Name: redis-telegram-bots-XXXXXX
- Internal URL: redis://:<pass>@<actual-app-name>:6379

## Deployed Bots

### <bot-name>
- Repo: <url>
- Branch: main
- Method: systemd|dokploy
- Type: polling|webhook
- [systemd] Service: <name>.service, Path: /opt/<name>
- [dokploy] App ID: <id>, Actual App Name: <name-XXXXXX>
- Database: sqlite|postgresql|none
- Deployed: <date>
```

---

## Manage Flow

Manage Telegram bots deployed via the Deploy flow. Check status, view logs, redeploy, update env vars, start/stop bots. Supports both systemd and Dokploy deployment methods.

### Step 0: Load Configuration

Read config from: `~/.claude/projects/<current-project>/memory/deploy-config.md`

If config doesn't exist, tell user to run `/tgbot deploy` first.

Extract: Server IP, and for each bot: name, method (systemd/dokploy), app ID (if dokploy), service name and path (if systemd).

If Dokploy is used: extract Protocol, Dokploy URL, and API Key.

Parse the remaining args to determine which subcommand to run.

---

### Subcommand: `status` (default)

Show status of all deployed bots.

For each bot in the config, check based on its **method**:

**Systemd bots:**
```bash
ssh root@SERVER_IP "systemctl is-active BOT_NAME 2>/dev/null; echo '---'; systemctl show BOT_NAME --property=ActiveEnterTimestamp --no-pager 2>/dev/null"
```

**Dokploy bots:**
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/application.one?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

Display a table:
```
| Bot Name | Method  | Status  | Type    | Last Deploy |
|----------|---------|---------|---------|-------------|
| my-bot   | systemd | active  | polling | 2024-01-15  |
| web-bot  | dokploy | running | webhook | 2024-01-14  |
```

---

### Subcommand: `logs <name>`

Find the bot in config. Check its **method**:

**Systemd:**
```bash
ssh root@SERVER_IP "journalctl -u BOT_NAME -n 100 --no-pager"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.readLogs" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Display the logs, showing the most recent entries. If logs are long, show the last 50 lines.

For Dokploy build/deployment logs:
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/deployment.all?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

---

### Subcommand: `redeploy <name>`

Find the bot in config. Check its **method**:

**Systemd:**
```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git pull origin BRANCH"
```

Then reinstall deps based on language:
- Python: `ssh root@SERVER_IP "cd /opt/BOT_NAME && .venv/bin/pip install -r requirements.txt"`
- Node.js: `ssh root@SERVER_IP "cd /opt/BOT_NAME && npm ci --production"`
- Go: `ssh root@SERVER_IP "cd /opt/BOT_NAME && go build -o bot ."`

Restart:
```bash
ssh root@SERVER_IP "systemctl restart BOT_NAME"
```

Verify:
```bash
ssh root@SERVER_IP "sleep 2 && systemctl is-active BOT_NAME && journalctl -u BOT_NAME -n 5 --no-pager"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.redeploy" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Poll deployment status until done or error (max 3 minutes).

---

### Subcommand: `env <name>`

Find the bot in config. Check its **method**:

**Systemd:**

Read current env:
```bash
ssh root@SERVER_IP "cat /opt/BOT_NAME/.env"
```

Display variables (mask BOT_TOKEN and passwords — show only first 4 chars + `...`).

If user wants to update, write the new env file:
```bash
ssh root@SERVER_IP "cat > /opt/BOT_NAME/.env << 'ENVEOF'
KEY1=val1
KEY2=val2
ENVEOF"
ssh root@SERVER_IP "chmod 600 /opt/BOT_NAME/.env"
```

After updating, ask if user wants to restart for changes to take effect:
```bash
ssh root@SERVER_IP "systemctl restart BOT_NAME"
```

**Dokploy:**

Get current env:
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/application.one?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

Display current environment variables (masked).

If user wants to update:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.saveEnvironment" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "APP_ID",
    "env": "FULL_ENV_STRING"
  }'
```

**Important:** `saveEnvironment` replaces ALL env vars. Always include the full set.

After updating, ask if user wants to redeploy for changes to take effect.

---

### Subcommand: `stop <name>`

**Systemd:**
```bash
ssh root@SERVER_IP "systemctl stop BOT_NAME"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.stop" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Confirm the bot has stopped.

---

### Subcommand: `start <name>`

**Systemd:**
```bash
ssh root@SERVER_IP "systemctl start BOT_NAME"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.start" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Confirm the bot has started.

---

### Subcommand: `delete <name>`

**This is destructive!** Ask for confirmation before proceeding.

**Systemd:**
1. Stop the service: `systemctl stop BOT_NAME`
2. Disable the service: `systemctl disable BOT_NAME`
3. Remove unit file: `rm /etc/systemd/system/BOT_NAME.service`
4. Reload systemd: `systemctl daemon-reload`
5. Remove bot directory: `rm -rf /opt/BOT_NAME`

**Dokploy:**
1. Stop the application (if running)
2. If the bot has a custom domain, remove it:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/domain.delete" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domainId": "DOMAIN_ID"}'
```
3. Delete the application:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.delete" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

**Both methods:** Remove the bot entry from `deploy-config.md` and confirm deletion.

---

### Subcommand: `list`

Simply read and display the `## Deployed Bots` section from `deploy-config.md`.

Show each bot with: name, method, type, repo, status.

---

### Manage Quick Reference: Method Dispatch

| Command | systemd | dokploy |
|---------|---------|---------|
| status | `systemctl is-active` + `show` | `application.one` API |
| logs | `journalctl -u <name> -n 100` | `application.readLogs` API |
| redeploy | `git pull && deps && systemctl restart` | `application.redeploy` API |
| env | read/write `/opt/<name>/.env` | `application.saveEnvironment` API |
| stop | `systemctl stop` | `application.stop` API |
| start | `systemctl start` | `application.start` API |
| delete | stop + disable + rm unit + rm -rf /opt/<name> | `application.delete` API |

---

## Prepare Flow

Analyze a GitHub repository and prepare it for deployment: add a Dockerfile if missing, create `.env.example`, extract hardcoded config to environment variables. Run this before Deploy if the repo isn't deployment-ready.

### Step 1: Get the Repository

If a repo URL was passed as argument, use it. Otherwise ask:
> What GitHub repo do you want to prepare for deployment?

Extract owner and repo name. Clone it locally for analysis:

```bash
cd /tmp && rm -rf REPO_NAME && git clone --depth 1 https://github.com/OWNER/REPO.git REPO_NAME
```

---

### Step 2: Analyze the Repository

Run a comprehensive analysis. Check `references/readiness-checks.md` for all checks.

**Detect language and framework:**

| File | Language | Framework hint |
|------|----------|---------------|
| `requirements.txt` | Python | Check contents for aiogram, python-telegram-bot, etc. |
| `pyproject.toml` | Python | Check deps for framework |
| `package.json` | Node.js | Check deps for grammy, telegraf, etc. |
| `go.mod` | Go | Check for telegram bot libs |
| `Cargo.toml` | Rust | Check for teloxide, etc. |

**Detect entry point:**

- Python: `bot/__main__.py`, `main.py`, `app.py`, `run.py`, `bot.py`
- Node.js: `package.json` — `main` field or `scripts.start`
- Go: `main.go`, `cmd/*/main.go`

**Detect database type:**

- SQLite: `sqlite3`, `aiosqlite`, `better-sqlite3`, `*.db` in `.gitignore`
- PostgreSQL: `asyncpg`, `psycopg2`, `pg`, `DATABASE_URL` with `postgresql://`
- Redis: `redis`, `aioredis`, `ioredis`
- None: no database indicators found

**Detect polling vs webhook:**

- Polling: `start_polling`, `run_polling`, `getUpdates`, `bot.polling`
- Webhook: `set_webhook`, `webhookCallback`, `express`, `fastapi`, `aiohttp.web`

**Detect configuration method:**

- Environment variables: `os.environ`, `os.getenv`, `process.env`, `os.Getenv`
- Config file: `config.py`, `config.json`, `config.yaml`, `.env` loading
- Hardcoded: values directly in source code (problematic for deployment)

---

### Step 3: Present Analysis

Show the user what was found:

```
Repository Analysis: owner/repo

  Language: Python 3.x (aiogram)
  Entry point: bot/__main__.py
  Database: SQLite (aiosqlite)
  Bot type: Polling
  Config: Mix of env vars and hardcoded values

Readiness:
  [OK] Has requirements.txt
  [MISSING] No Dockerfile
  [MISSING] No .env.example
  [WARN] Hardcoded SQLite path: ./data/bot.db (line 15 in bot/db.py)
  [WARN] Hardcoded log path: ./logs/ (line 8 in bot/main.py)

Recommended actions:
  1. Add Dockerfile (Python slim template)
  2. Create .env.example with discovered variables
  3. Extract hardcoded paths to env vars (optional)
```

Ask user which actions to perform.

---

### Step 4: Add Dockerfile (if missing)

Use templates from `references/dockerfile-templates.md`.

Select the right template based on:
- Language detected
- Package manager (pip, poetry, uv, npm, bun)
- Webhook vs polling (webhook needs `EXPOSE`)

Write the Dockerfile to the cloned repo:
```bash
# Write Dockerfile to /tmp/REPO_NAME/Dockerfile
```

Also create `.dockerignore` if missing:
```
.git
.env
*.pyc
__pycache__
node_modules
.venv
```

---

### Step 5: Create .env.example (if missing)

Based on all discovered environment variables from Step 2, create `.env.example`:

```bash
# /tmp/REPO_NAME/.env.example
# Bot Configuration
BOT_TOKEN=your-telegram-bot-token

# OpenAI (if used)
OPENAI_API_KEY=your-openai-api-key

# Database (auto-configured by deploy-bot)
# DATABASE_URL=postgresql://user:pass@host:5432/db

# Defaults
PYTHONUNBUFFERED=1
LOG_LEVEL=info
```

Group variables by purpose with comments. Mark auto-configured ones as comments.

---

### Step 6: Extract Hardcoded Config (optional)

If the user wants, refactor hardcoded values to use environment variables.

See `references/migration-guides.md` for patterns:
- Hardcoded SQLite path — `os.getenv("DATABASE_PATH", "./data/bot.db")`
- Hardcoded API URLs — `os.getenv("API_URL", "https://...")`
- Hardcoded log paths — `os.getenv("LOG_DIR", "./logs")`

Show the user each proposed change before making it.

---

### Step 7: Commit and Push

Show the user all changes:
```bash
cd /tmp/REPO_NAME && git diff && git status
```

Ask for confirmation, then:
```bash
cd /tmp/REPO_NAME
git add Dockerfile .dockerignore .env.example
git commit -m "Add deployment configuration (Dockerfile, .env.example)"
git push
```

Clean up:
```bash
rm -rf /tmp/REPO_NAME
```

---

### Step 8: Next Steps

Tell the user:
> Your repo is now deployment-ready! Run `/tgbot deploy https://github.com/OWNER/REPO` to deploy it.

---

## Decisions Made Automatically (not asked from user)

| Decision | Default | Override |
|----------|---------|---------|
| Deploy method | systemd (polling+SQLite) / dokploy (webhook+PG) | User can override |
| Bot name | GitHub repo name | User can specify |
| Polling vs webhook | Auto-detect from code | User can specify |
| Shared PG/Redis | Only if bot needs it | User can force |
| Dockerfile | Use existing, or nixpacks, or suggest /tgbot prepare | User decides |
| Branch | Default branch from GitHub | User can specify |
| Dockerfile template | Auto-detect from language | User can specify |
| .env.example vars | All discovered vars | User can add/remove |
| Config extraction | Suggest, don't force | User approves each change |
| Commit message | Standard message | User can customize |

## Error Handling

See `references/error-recovery.md` for comprehensive recovery strategies.

**Deploy errors:**
- **SSH fails** — verify IP and password, check server reachability
- **Docker Hub rate limit** — switch to systemd, or wait 6 hours, or authenticate
- **Dokploy already installed** — detect, skip install, ask for API key
- **Dokploy HTTP vs HTTPS** — detect protocol automatically
- **Missing environmentId** — get from project.one response
- **appName mismatch** — always read actual appName from create response
- **Build fails** — check logs, suggest `/tgbot prepare` if no Dockerfile
- **Bot crashes** — read logs, check env vars, verify entry point
- **Git clone fails in Docker** — DNS issue, suggest systemd or fix Docker DNS

**Manage errors:**
- If a bot name is not found in the config, list available bots and ask the user to pick one
- If Dokploy API returns an error, show the error message and suggest troubleshooting steps
- If SSH command fails, check if server is reachable
- If the config file is missing or malformed, suggest running `/tgbot deploy`

**Important notes:**
- Always read the latest config before any operation
- When showing env vars, mask sensitive values (tokens, passwords)
- For Dokploy: `saveEnvironment` replaces ALL vars — always include the full set
- After env changes, a restart/redeploy is needed for changes to take effect
- Deletion is permanent — always confirm with user

## Reference Files

- `references/preflight-checks.md` — all pre-flight validation logic
- `references/server-bootstrap.md` — SSH key setup and base package installation
- `references/systemd-deployment.md` — systemd deploy method (templates, update, rollback)
- `references/dokploy-setup.md` — Dokploy installation, admin, project setup
- `references/dokploy-api.md` — Dokploy API reference (corrected fields)
- `references/secrets-detection.md` — detecting env vars and database needs from repo
- `references/dockerfile-templates.md` — Dockerfile templates per language
- `references/error-recovery.md` — common failures and recovery strategies
- `references/readiness-checks.md` — what makes a repo deploy-ready
- `references/migration-guides.md` — how to fix common issues (SQLite path, Dockerfile, env extraction)
