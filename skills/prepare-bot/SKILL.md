# /prepare-bot — Make a Repo Deployment-Ready

## Description

Analyzes a GitHub repository and prepares it for deployment: adds a Dockerfile if missing, creates `.env.example`, extracts hardcoded config to environment variables. Separate from `/deploy-bot` — run this first if the repo isn't deployment-ready.

## Usage

- `/prepare-bot` — interactive: asks for repo URL
- `/prepare-bot https://github.com/owner/repo` — prepare this specific repo

## Instructions

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
- Node.js: `package.json` → `main` field or `scripts.start`
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

Use templates from `/deploy-bot/references/dockerfile-templates.md`.

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
- Hardcoded SQLite path → `os.getenv("DATABASE_PATH", "./data/bot.db")`
- Hardcoded API URLs → `os.getenv("API_URL", "https://...")`
- Hardcoded log paths → `os.getenv("LOG_DIR", "./logs")`

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
> Your repo is now deployment-ready! Run `/deploy-bot https://github.com/OWNER/REPO` to deploy it.

---

## Decisions

| Decision | Default | Override |
|----------|---------|---------|
| Dockerfile template | Auto-detect from language | User can specify |
| .env.example vars | All discovered vars | User can add/remove |
| Config extraction | Suggest, don't force | User approves each change |
| Commit message | Standard message | User can customize |

## Reference Files

- `references/readiness-checks.md` — what makes a repo deploy-ready
- `references/migration-guides.md` — how to fix common issues (SQLite path, Dockerfile, env extraction)
