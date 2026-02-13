# /deploy-bot — Deploy a Telegram Bot

## Description

Deploys a Telegram bot to a VPS. Supports two methods:
- **systemd** (default) — runs the bot directly on the server as a systemd service. No Docker dependency. Best for polling bots using SQLite.
- **dokploy** — runs the bot in Docker via Dokploy. Best for webhook bots needing Traefik, or bots needing shared PostgreSQL/Redis.

On first run, sets up the server. On subsequent runs, skips server setup and deploys the bot.

## Usage

- `/deploy-bot` — interactive: asks for everything needed
- `/deploy-bot https://github.com/owner/repo` — deploy this specific repo

## Instructions

### Phase 0: Pre-Flight Checks

Follow `references/preflight-checks.md`.

**0.1** Read config from: `~/.claude/projects/<current-project>/memory/deploy-config.md`

**0.2** If config exists:
- Verify SSH works: `ssh -o BatchMode=yes -o ConnectTimeout=5 root@IP "echo ok"`
- If method=dokploy, verify Dokploy API is reachable

**0.3** If method=dokploy (configured or will be chosen):
- Check Docker Hub rate limit: `ssh root@IP "docker pull hello-world 2>&1"` → parse for `toomanyrequests`
- If rate-limited, warn and suggest systemd

**0.4** If repo provided, validate: `gh api repos/OWNER/REPO --jq '.default_branch'`

**0.5** Report ALL issues at once. Let user decide how to proceed.

---

### Phase 1: Server Access

**1.1** If SSH already works (from config or pre-flight) → skip to Phase 2

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
- If not ready, suggest running `/prepare-bot` first — OR proceed with systemd (which doesn't need a Dockerfile)

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

**3B.2** If not installed → install Dokploy (see `references/dokploy-setup.md`)

**3B.3** Detect protocol (HTTP vs HTTPS):
- Try `http://IP:3000` first, then `https://IP:3000` with `-k`
- Use whichever responds

**3B.4** Check if admin exists:
- Try `POST /api/auth.createAdmin` — if error/401 → admin exists, ask user for API key
- If success → save credentials, generate API token

**3B.5** Check Docker Hub rate limit:
```bash
ssh root@IP "docker pull hello-world 2>&1"
```
If `toomanyrequests` → warn user, suggest switching to systemd

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

On error → read build logs, help debug.

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

## Decisions Made Automatically (not asked from user)

| Decision | Default | Override |
|----------|---------|---------|
| Deploy method | systemd (polling+SQLite) / dokploy (webhook+PG) | User can override |
| Bot name | GitHub repo name | User can specify |
| Polling vs webhook | Auto-detect from code | User can specify |
| Shared PG/Redis | Only if bot needs it | User can force |
| Dockerfile | Use existing, or nixpacks, or suggest /prepare-bot | User decides |
| Branch | Default branch from GitHub | User can specify |

## Error Handling

See `references/error-recovery.md` for comprehensive recovery strategies.

Quick reference:
- **SSH fails** → verify IP and password, check server reachability
- **Docker Hub rate limit** → switch to systemd, or wait 6 hours, or authenticate
- **Dokploy already installed** → detect, skip install, ask for API key
- **Dokploy HTTP vs HTTPS** → detect protocol automatically
- **Missing environmentId** → get from project.one response
- **appName mismatch** → always read actual appName from create response
- **Build fails** → check logs, suggest /prepare-bot if no Dockerfile
- **Bot crashes** → read logs, check env vars, verify entry point
- **Git clone fails in Docker** → DNS issue, suggest systemd or fix Docker DNS

## Reference Files

- `references/preflight-checks.md` — all pre-flight validation logic
- `references/server-bootstrap.md` — SSH key setup and base package installation
- `references/systemd-deployment.md` — systemd deploy method (templates, update, rollback)
- `references/dokploy-setup.md` — Dokploy installation, admin, project setup
- `references/dokploy-api.md` — Dokploy API reference (corrected fields)
- `references/secrets-detection.md` — detecting env vars and database needs from repo
- `references/dockerfile-templates.md` — Dockerfile templates per language
- `references/error-recovery.md` — common failures and recovery strategies
