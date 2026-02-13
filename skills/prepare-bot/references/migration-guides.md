# Migration Guides Reference

Patterns for making repos deployment-ready: adding Dockerfiles, extracting config, migrating databases.

---

## Guide 1: Add Dockerfile to Python Bot

### Before (no Dockerfile)
```
repo/
  bot/
    __main__.py
    handlers.py
  requirements.txt
```

### After
```
repo/
  bot/
    __main__.py
    handlers.py
  requirements.txt
  Dockerfile        ← new
  .dockerignore     ← new
```

### Steps
1. Detect package manager: `requirements.txt` → pip, `poetry.lock` → poetry, `uv.lock` → uv
2. Detect entry point: `bot/__main__.py` → `CMD ["python", "-m", "bot"]`
3. Select template from `/deploy-bot/references/dockerfile-templates.md`
4. Write Dockerfile and .dockerignore

### Choosing the CMD
| Entry point | CMD |
|------------|-----|
| `bot/__main__.py` | `["python", "-m", "bot"]` |
| `main.py` | `["python", "main.py"]` |
| `app.py` | `["python", "app.py"]` |
| `src/bot/__main__.py` | `["python", "-m", "src.bot"]` |
| Unclear | Ask user |

---

## Guide 2: Add Dockerfile to Node.js Bot

### Steps
1. Check `package.json` for `main` or `scripts.start`
2. Detect if TypeScript (`tsconfig.json` present)
3. Detect package manager: `bun.lockb` → bun, `package-lock.json` → npm
4. Select template accordingly

### Choosing the CMD
| package.json | CMD |
|-------------|-----|
| `"main": "src/index.js"` | `["node", "src/index.js"]` |
| `"start": "ts-node src/index.ts"` | Use TypeScript build template |
| `"start": "bun run src/index.ts"` | Use Bun template |

---

## Guide 3: Extract Hardcoded SQLite Path

### Before
```python
# bot/db.py
import aiosqlite

DB_PATH = "./data/bot.db"

async def get_db():
    return await aiosqlite.connect(DB_PATH)
```

### After
```python
# bot/db.py
import os
import aiosqlite

DB_PATH = os.getenv("DATABASE_PATH", "./data/bot.db")

async def get_db():
    return await aiosqlite.connect(DB_PATH)
```

### Also add to .env.example
```
# Database path (default: ./data/bot.db)
# DATABASE_PATH=./data/bot.db
```

### Systemd note
With systemd deployment, `WorkingDirectory=/opt/BOT_NAME` makes relative paths like `./data/bot.db` resolve to `/opt/BOT_NAME/data/bot.db`. This usually works without changes.

---

## Guide 4: Extract Hardcoded Config Values

### Pattern: API URL
```python
# Before
API_URL = "https://api.openai.com/v1"

# After
API_URL = os.getenv("API_URL", "https://api.openai.com/v1")
```

### Pattern: Log configuration
```python
# Before
logging.basicConfig(filename="./logs/bot.log", level=logging.INFO)

# After
log_file = os.getenv("LOG_FILE", "./logs/bot.log")
log_level = os.getenv("LOG_LEVEL", "INFO")
logging.basicConfig(filename=log_file, level=getattr(logging, log_level))
```

### Pattern: Bot configuration
```python
# Before
ADMIN_IDS = [12345, 67890]

# After
ADMIN_IDS = [int(x) for x in os.getenv("ADMIN_IDS", "12345,67890").split(",")]
```

---

## Guide 5: Create .env.example

### Structure
```bash
# ============================================
# Bot Configuration
# ============================================

# Telegram bot token (get from @BotFather)
BOT_TOKEN=

# ============================================
# API Keys (if needed)
# ============================================

# OpenAI API key
# OPENAI_API_KEY=

# ============================================
# Database (auto-configured by deploy-bot)
# ============================================

# Uncomment if using PostgreSQL:
# DATABASE_URL=postgresql://user:pass@host:5432/db

# SQLite path (default is fine for most setups):
# DATABASE_PATH=./data/bot.db

# ============================================
# Optional Settings
# ============================================

# Log level: DEBUG, INFO, WARNING, ERROR
# LOG_LEVEL=INFO

# Python: see logs immediately in docker/journalctl
PYTHONUNBUFFERED=1
```

### Rules
- Required secrets: no default value, just `KEY=`
- Optional with defaults: commented out with `# KEY=default`
- Infrastructure vars: commented out, marked as "auto-configured"
- Group by purpose with section headers

---

## Guide 6: SQLite to PostgreSQL Migration

**When to migrate:** Only if the user explicitly wants it. SQLite works fine for single-instance bots.

### Step 1: Add PostgreSQL dependency

Python:
```bash
# Add asyncpg or psycopg2 to requirements.txt
asyncpg==0.29.0
# or
psycopg2-binary==2.9.9
```

### Step 2: Abstract database layer

Create a database abstraction that supports both:

```python
import os

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./data/bot.db")

if DATABASE_URL.startswith("sqlite"):
    # Use aiosqlite
    pass
elif DATABASE_URL.startswith("postgresql"):
    # Use asyncpg
    pass
```

### Step 3: Schema migration

If the bot uses raw SQL, the schema likely needs adjustment:
- `AUTOINCREMENT` → `SERIAL` or `GENERATED ALWAYS AS IDENTITY`
- `TEXT` → `TEXT` (same)
- `INTEGER` → `INTEGER` (same)
- `REAL` → `DOUBLE PRECISION`
- `BLOB` → `BYTEA`

### Step 4: Data migration (if needed)

```bash
# Export from SQLite
sqlite3 bot.db .dump > dump.sql

# Adjust SQL syntax for PostgreSQL
# Import to PostgreSQL
psql DATABASE_URL < dump.sql
```

**Recommendation:** For most Telegram bots, SQLite is perfectly fine. Only migrate if:
- Bot needs to scale to multiple instances
- Bot needs concurrent writes from many users
- User specifically requests PostgreSQL
