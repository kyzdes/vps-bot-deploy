# Secrets Detection Reference

How to discover and classify environment variables from a GitHub repository, including database needs detection.

---

## Source Priority

Check these files in order. Stop early if `.env.example` gives a complete picture.

### 1. `.env.example` (best source)

```bash
gh api repos/OWNER/REPO/contents/.env.example -q '.content' | base64 -d
```

Parse each line as `KEY=value` or `KEY=`. Comments often describe what the variable is for.

Also check for `.env.sample`, `.env.template`, `env.example`.

### 2. `README.md`

```bash
gh api repos/OWNER/REPO/contents/README.md -q '.content' | base64 -d
```

Look for sections with these headings (case-insensitive):
- "Environment", "Env", "Configuration", "Config", "Setup", "Installation"
- ".env", "Environment Variables", "Settings"

Extract any variable names mentioned (patterns like `KEY_NAME`, `UPPER_SNAKE_CASE`).

### 3. `docker-compose.yml`

```bash
gh api repos/OWNER/REPO/contents/docker-compose.yml -q '.content' | base64 -d
```

Parse the `environment:` section of each service. Variables may use `${VAR}` or `${VAR:-default}` syntax.

Also check `docker-compose.yaml`, `compose.yml`, `compose.yaml`.

### 4. Source code scan

Get the file tree:
```bash
gh api "repos/OWNER/REPO/git/trees/BRANCH?recursive=1" -q '.tree[].path'
```

Find entry point files and scan for env var access:

**Python:**
```
os.environ["KEY"]
os.environ.get("KEY")
os.getenv("KEY")
config.KEY  (with pydantic-settings or similar)
```

Key files to check: `bot.py`, `main.py`, `config.py`, `settings.py`, `app.py`, `src/config.py`, `bot/config.py`

**Node.js:**
```
process.env.KEY
process.env["KEY"]
```

Key files to check: `src/index.ts`, `src/config.ts`, `config.js`, `src/env.ts`, `.env.validation.ts`

**Go:**
```
os.Getenv("KEY")
viper.GetString("KEY")
```

Key files to check: `main.go`, `cmd/main.go`, `config/config.go`, `internal/config/config.go`

Read these files:
```bash
gh api repos/OWNER/REPO/contents/PATH -q '.content' | base64 -d
```

---

## Database Needs Detection

**CRITICAL:** Before auto-filling infrastructure variables, determine what database the bot actually uses. Do NOT assume every bot needs PostgreSQL and Redis.

### Step 1: Check dependency files

**Python** (`requirements.txt`, `pyproject.toml`):

| Dependency | Database |
|-----------|----------|
| `sqlite3` (stdlib) | SQLite |
| `aiosqlite` | SQLite |
| `asyncpg` | PostgreSQL |
| `psycopg2`, `psycopg2-binary` | PostgreSQL |
| `sqlalchemy` | Check connection string — could be SQLite or PG |
| `databases` | Check connection string |
| `redis`, `aioredis` | Redis |
| `celery` | Likely needs Redis |

**Node.js** (`package.json`):

| Dependency | Database |
|-----------|----------|
| `better-sqlite3`, `sqlite3`, `sql.js` | SQLite |
| `pg`, `postgres`, `@prisma/client` | Could be PG or SQLite — check schema |
| `sequelize` | Check dialect config |
| `redis`, `ioredis` | Redis |
| `bull`, `bullmq` | Needs Redis |

**Go** (`go.mod`):

| Dependency | Database |
|-----------|----------|
| `mattn/go-sqlite3`, `modernc.org/sqlite` | SQLite |
| `jackc/pgx`, `lib/pq` | PostgreSQL |
| `go-redis/redis` | Redis |

### Step 2: Check source code for connection strings

```bash
# SQLite patterns
gh api repos/OWNER/REPO/contents/PATH -q '.content' | base64 -d | grep -i "sqlite\|\.db\""

# PostgreSQL patterns
gh api repos/OWNER/REPO/contents/PATH -q '.content' | base64 -d | grep -i "postgresql://\|postgres://\|asyncpg\|psycopg"

# Redis patterns
gh api repos/OWNER/REPO/contents/PATH -q '.content' | base64 -d | grep -i "redis://\|REDIS_URL\|aioredis"
```

### Step 3: Classify database needs

| Detection | Needs PG | Needs Redis | Action |
|-----------|----------|-------------|--------|
| Only SQLite deps | No | No | Don't create PG/Redis |
| No database deps at all | No | No | Don't create PG/Redis |
| PostgreSQL deps | Yes | No | Create PG if using Dokploy |
| Redis deps | No | Yes | Create Redis if using Dokploy |
| Both PG and Redis deps | Yes | Yes | Create both if using Dokploy |
| SQLAlchemy with no clear dialect | Ask user | No | Check config for dialect |

---

## Variable Classification

After collecting all variable names, classify them:

### Secrets (ask user)

Variables that contain credentials the user must provide:

| Pattern | Examples |
|---------|----------|
| `*TOKEN*` | `BOT_TOKEN`, `TELEGRAM_TOKEN`, `API_TOKEN` |
| `*API_KEY*` | `OPENAI_API_KEY`, `STRIPE_API_KEY` |
| `*SECRET*` | `JWT_SECRET`, `APP_SECRET`, `SECRET_KEY` |
| `*PASSWORD*` (non-DB) | `ADMIN_PASSWORD`, `AUTH_PASSWORD` |
| `*WEBHOOK_URL*` | `TELEGRAM_WEBHOOK_URL` (if webhook bot) |
| `*PRIVATE_KEY*` | `RSA_PRIVATE_KEY` |
| `*_CREDENTIALS*` | `GOOGLE_CREDENTIALS` |

### Infrastructure (auto-fill ONLY if bot needs it)

**Only fill these if the database needs detection confirms the bot uses this database.**

| Pattern | Value from config | Condition |
|---------|-------------------|-----------|
| `DATABASE_URL` | PostgreSQL Internal URL | Bot uses PostgreSQL |
| `DB_HOST` | actual PG appName | Bot uses PostgreSQL |
| `DB_PORT` | `5432` | Bot uses PostgreSQL |
| `DB_NAME` / `DB_DATABASE` | `bots` | Bot uses PostgreSQL |
| `DB_USER` / `DB_USERNAME` | `botuser` | Bot uses PostgreSQL |
| `DB_PASSWORD` / `DB_PASS` | PostgreSQL password | Bot uses PostgreSQL |
| `POSTGRES_*` | Corresponding PG values | Bot uses PostgreSQL |
| `REDIS_URL` | Redis Internal URL | Bot uses Redis |
| `REDIS_HOST` | actual Redis appName | Bot uses Redis |
| `REDIS_PORT` | `6379` | Bot uses Redis |
| `REDIS_PASSWORD` | Redis password | Bot uses Redis |

**For SQLite bots:** Do NOT fill DATABASE_URL or any DB_* vars with PostgreSQL values. If the bot uses a `DATABASE_PATH` or `DB_PATH` variable, set it to a reasonable local path (e.g., `./data/bot.db`).

### Defaults (set automatically)

Common variables with reasonable defaults:

| Variable | Default | Reason |
|----------|---------|--------|
| `PORT` | `8080` | Webhook port convention |
| `HOST` | `0.0.0.0` | Listen on all interfaces |
| `NODE_ENV` | `production` | Standard Node.js |
| `PYTHONUNBUFFERED` | `1` | See Python logs in real-time |
| `LOG_LEVEL` | `info` | Reasonable default |
| `TZ` | `UTC` | Consistent timezone |
| `ENVIRONMENT` / `ENV` | `production` | Standard |

### Unknown (show to user)

Anything not matching the above patterns. Show them to the user with context from where they were found, and ask:
- Is this required?
- What value should it have?

---

## Output Format

Present to the user like this:

### If bot uses PostgreSQL (Dokploy method):
```
I analyzed your repo and found these environment variables:

Secrets (I need values from you):
  - BOT_TOKEN — Telegram bot token (from .env.example)
  - OPENAI_API_KEY — OpenAI API key (from README)

Auto-configured (from server infrastructure):
  - DATABASE_URL = postgresql://botuser:***@pg-telegram-bots-XXXXXX:5432/bots
  - REDIS_URL = redis://:***@redis-telegram-bots-XXXXXX:6379

Defaults:
  - PYTHONUNBUFFERED = 1
  - LOG_LEVEL = info

Please provide values for the secrets listed above.
```

### If bot uses SQLite (systemd method):
```
I analyzed your repo and found these environment variables:

Secrets (I need values from you):
  - BOT_TOKEN — Telegram bot token (from .env.example)
  - OPENAI_API_KEY — OpenAI API key (from config.py)

Defaults:
  - PYTHONUNBUFFERED = 1
  - LOG_LEVEL = info

Note: This bot uses SQLite — no external database needed.

Please provide values for the secrets listed above.
```

Mask infrastructure passwords in the display (show `***`), but use real values when setting env vars.

---

## Edge Cases

- **No env vars found anywhere** → The bot might use command-line args or a config file. Ask the user what configuration the bot needs.
- **`.env.example` has values filled in** → These might be example/placeholder values. Still ask the user to confirm secrets.
- **Monorepo** → Check if there's a subdirectory for the bot. Look for `buildPath` hints in docker-compose or CI config.
- **Multiple `.env` files** → e.g., `.env.development`, `.env.production`. Prefer `.env.production` or `.env.example`.
- **SQLAlchemy without clear dialect** → Check `config.py` or `settings.py` for the connection string pattern. If ambiguous, ask the user.
