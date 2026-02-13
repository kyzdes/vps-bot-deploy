# Readiness Checks Reference

What makes a repository deployment-ready, and how to check each criterion.

---

## Check 1: Dockerfile

```bash
ls /tmp/REPO_NAME/Dockerfile 2>/dev/null && echo "OK" || echo "MISSING"
```

Also check for `Dockerfile.prod`, `docker/Dockerfile`, `Dockerfile.production`.

**If missing:** Generate using templates from `/deploy-bot/references/dockerfile-templates.md`.

---

## Check 2: .env.example

```bash
ls /tmp/REPO_NAME/.env.example 2>/dev/null || \
ls /tmp/REPO_NAME/.env.sample 2>/dev/null || \
ls /tmp/REPO_NAME/.env.template 2>/dev/null || \
ls /tmp/REPO_NAME/env.example 2>/dev/null || \
echo "MISSING"
```

**If missing:** Generate by scanning source code for env var access patterns.

---

## Check 3: Dependency Lock File

| Language | Expected lock file | Status if missing |
|----------|-------------------|-------------------|
| Python (pip) | `requirements.txt` | WARN — deps may not be pinned |
| Python (poetry) | `poetry.lock` | WARN — should run `poetry lock` |
| Python (uv) | `uv.lock` | WARN — should run `uv lock` |
| Node.js (npm) | `package-lock.json` | WARN — builds may be non-deterministic |
| Node.js (bun) | `bun.lockb` | WARN |
| Go | `go.sum` | WARN |
| Rust | `Cargo.lock` | WARN |

---

## Check 4: Entry Point

The entry point must be detectable. Check:

### Python
```bash
# Check for __main__.py
find /tmp/REPO_NAME -name "__main__.py" -not -path "*/.venv/*" 2>/dev/null

# Check for common entry files
for f in main.py app.py run.py bot.py; do
  [ -f "/tmp/REPO_NAME/$f" ] && echo "Found: $f"
done
```

### Node.js
```bash
# Check package.json for main or start script
cat /tmp/REPO_NAME/package.json | jq -r '.main // .scripts.start // empty'
```

### Go
```bash
[ -f "/tmp/REPO_NAME/main.go" ] && echo "Found: main.go"
find /tmp/REPO_NAME/cmd -name "main.go" 2>/dev/null
```

**If unclear:** Ask the user what the entry point is.

---

## Check 5: Hardcoded Paths

Scan for hardcoded file paths that won't work in a container or different deployment:

### Python
```bash
grep -rn "sqlite:///\|\.db\"\|/data/\|/tmp/\|/home/" /tmp/REPO_NAME --include="*.py" | grep -v ".venv" | grep -v "__pycache__"
```

### Node.js
```bash
grep -rn "sqlite\|\.db\"\|/data/\|/tmp/\|/home/" /tmp/REPO_NAME --include="*.ts" --include="*.js" | grep -v "node_modules"
```

Common patterns that need extraction to env vars:
- `sqlite:///./data/bot.db` → should use env var for path
- `open("./logs/bot.log")` → should use env var for log dir
- `Path("/home/user/bot/data")` → absolute path, won't work in container

---

## Check 6: Hardcoded Secrets

Scan for potential secrets committed to the repo:

```bash
grep -rn "sk-\|ghp_\|token.*=.*['\"]" /tmp/REPO_NAME --include="*.py" --include="*.ts" --include="*.js" --include="*.go" | grep -v ".venv" | grep -v "node_modules" | grep -v ".env.example"
```

**If found:** CRITICAL warning — secrets should be in env vars, not in code.

---

## Check 7: .gitignore

Check that sensitive and generated files are ignored:

```bash
cat /tmp/REPO_NAME/.gitignore 2>/dev/null
```

Should include:
- `.env` (not `.env.example`)
- `*.db` or `data/` (if using SQLite)
- `__pycache__/`, `*.pyc` (Python)
- `node_modules/` (Node.js)
- `.venv/` (Python)

---

## Check 8: Database Configuration

Determine what database the bot uses:

### SQLite indicators
```bash
grep -rn "sqlite\|aiosqlite\|better-sqlite3\|sql\.js" /tmp/REPO_NAME --include="*.py" --include="*.ts" --include="*.js" --include="*.toml" --include="*.json" | grep -v ".venv" | grep -v "node_modules"
```

### PostgreSQL indicators
```bash
grep -rn "asyncpg\|psycopg\|DATABASE_URL\|postgresql://\|pg\." /tmp/REPO_NAME --include="*.py" --include="*.ts" --include="*.js" --include="*.toml" --include="*.json" | grep -v ".venv" | grep -v "node_modules"
```

### Redis indicators
```bash
grep -rn "aioredis\|redis\.\|ioredis\|REDIS_URL" /tmp/REPO_NAME --include="*.py" --include="*.ts" --include="*.js" --include="*.toml" --include="*.json" | grep -v ".venv" | grep -v "node_modules"
```

### No database
If none of the above match, the bot likely stores no persistent data or uses in-memory state only.

---

## Summary Report Template

```
Readiness Report for OWNER/REPO:

[OK/MISSING] Dockerfile
[OK/MISSING] .env.example
[OK/WARN] Dependency lock file
[OK/WARN] Entry point detectable
[OK/WARN] No hardcoded paths
[OK/CRITICAL] No hardcoded secrets
[OK/WARN] .gitignore complete
[INFO] Database: sqlite/postgresql/redis/none

Overall: READY / NEEDS PREPARATION
Actions needed: (list of specific fixes)
```
