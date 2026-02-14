# Dokploy Setup Reference

Setting up Dokploy on the server — install, admin creation, project setup. Only used when deploy method is `dokploy`.

---

## Pre-Check: Is Dokploy Already Installed?

```bash
ssh root@SERVER_IP "docker ps --format '{{.Names}}' 2>/dev/null | grep -q dokploy && echo 'installed' || echo 'not-installed'"
```

- `installed` → skip installation, go to admin/API key setup
- `not-installed` → proceed with installation

**CRITICAL:** Do NOT install Dokploy if it's already running. The installer will fail with "port 3000 already in use".

---

## Install Dokploy (fresh server only)

```bash
ssh root@SERVER_IP "curl -sSL https://dokploy.com/install.sh | sh"
```

This takes 2-5 minutes. The installer sets up Docker, Traefik, and Dokploy.

### Wait for Dokploy to be ready

Try HTTP first (more common default), then HTTPS:

```bash
# Try HTTP
ssh root@SERVER_IP "for i in \$(seq 1 60); do curl -sf http://localhost:3000 > /dev/null 2>&1 && echo 'ready-http' && exit 0; sleep 5; done; echo 'timeout'"
```

If HTTP times out, try HTTPS:
```bash
ssh root@SERVER_IP "curl -sfk https://localhost:3000 > /dev/null 2>&1 && echo 'ready-https' || echo 'not-ready'"
```

### Verify Docker container

```bash
ssh root@SERVER_IP "docker ps --format '{{.Names}}' | grep dokploy"
```

---

## Detect Protocol (HTTP vs HTTPS)

**CRITICAL:** Dokploy can run on HTTP or HTTPS. Always detect, never assume.

```bash
# From local machine, try HTTP first
HTTP_CODE=$(curl -sf -o /dev/null -w "%{http_code}" "http://SERVER_IP:3000" --max-time 5 2>/dev/null)
if [ "$HTTP_CODE" -gt 0 ] 2>/dev/null; then
  PROTOCOL="http"
else
  HTTPS_CODE=$(curl -sfk -o /dev/null -w "%{http_code}" "https://SERVER_IP:3000" --max-time 5 2>/dev/null)
  if [ "$HTTPS_CODE" -gt 0 ] 2>/dev/null; then
    PROTOCOL="https"
  else
    echo "Cannot reach Dokploy on port 3000"
  fi
fi
```

Use `PROTOCOL://SERVER_IP:3000` as the base URL for all API calls.

For HTTPS, always use `-k` flag (self-signed certs are common).

---

## Admin Account Setup

### Check if admin already exists

```bash
curl -sf -X POST "${PROTOCOL}://SERVER_IP:3000/api/auth.createAdmin" \
  -H "Content-Type: application/json" \
  -k \
  -d '{"email":"test@test.com","password":"test"}' 2>&1
```

Possible responses:
- Success (200 with user data) → admin was just created (you now have credentials)
- Error with "admin already exists" or 401 → admin exists, need API key from user
- Error with "already registered" → same as above

### If no admin exists → create one

```bash
ADMIN_PASSWORD=$(openssl rand -base64 16)

curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/auth.createAdmin" \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"admin@bot.local\",\"password\":\"$ADMIN_PASSWORD\"}"
```

### Login and get API token

```bash
# Login
curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/auth.login" \
  -H "Content-Type: application/json" \
  -c /tmp/dokploy-cookies.txt \
  -d "{\"email\":\"admin@bot.local\",\"password\":\"$ADMIN_PASSWORD\"}"

# Generate token
API_KEY=$(curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/auth.generateToken" \
  -H "Content-Type: application/json" \
  -b /tmp/dokploy-cookies.txt | jq -r '.result.data // .token // .')

rm -f /tmp/dokploy-cookies.txt
```

### Verify API key

```bash
curl -sk "${PROTOCOL}://SERVER_IP:3000/api/admin.one" \
  -H "x-api-key: $API_KEY"
```

### If admin exists → ask user for API key

Tell the user:
> Dokploy is already set up on this server with an admin account.
> Please provide the API key from: **Dokploy Dashboard → Settings → Profile → API**
> (Open ${PROTOCOL}://SERVER_IP:3000 in your browser)

---

## Project Setup

### Find or create project

First check if project already exists:
```bash
PROJECTS=$(curl -sk "${PROTOCOL}://SERVER_IP:3000/api/project.all" \
  -H "x-api-key: $API_KEY")
```

Look for a project named "telegram-bots" in the response. If found, use its `projectId`.

If not found, create:
```bash
PROJECT=$(curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/project.create" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"telegram-bots","description":"Telegram bots infrastructure"}')

PROJECT_ID=$(echo "$PROJECT" | jq -r '.result.data.projectId // .projectId')
```

### Get Environment ID (CRITICAL)

**Every service creation requires `environmentId`.** Get it from the project:

```bash
PROJECT_DETAIL=$(curl -sk "${PROTOCOL}://SERVER_IP:3000/api/project.one?input=%7B%22json%22%3A%7B%22projectId%22%3A%22${PROJECT_ID}%22%7D%7D" \
  -H "x-api-key: $API_KEY")

ENVIRONMENT_ID=$(echo "$PROJECT_DETAIL" | jq -r '.result.data.environments[0].environmentId // .environments[0].environmentId')
```

If no environment exists, create one:
```bash
ENV_RESULT=$(curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/environment.create" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"projectId\":\"$PROJECT_ID\",\"name\":\"production\",\"description\":\"Production environment\"}")

ENVIRONMENT_ID=$(echo "$ENV_RESULT" | jq -r '.result.data.environmentId // .environmentId')
```

---

## Shared Services (On-Demand Only)

**Only create PostgreSQL/Redis if the bot actually needs them.** See `secrets-detection.md` for database-needs detection.

### Create Shared PostgreSQL

```bash
PG_PASSWORD=$(openssl rand -base64 16 | tr -d '/+=')

PG_RESULT=$(curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/postgres.create" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"pg-telegram-bots\",
    \"appName\": \"pg-telegram-bots\",
    \"projectId\": \"$PROJECT_ID\",
    \"environmentId\": \"$ENVIRONMENT_ID\",
    \"databaseName\": \"bots\",
    \"databaseUser\": \"botuser\",
    \"databasePassword\": \"$PG_PASSWORD\",
    \"dockerImage\": \"postgres:16\"
  }")

PG_ID=$(echo "$PG_RESULT" | jq -r '.result.data.postgresId // .postgresId')
# CRITICAL: Read the actual appName from the response — Dokploy adds a random suffix
PG_ACTUAL_APPNAME=$(echo "$PG_RESULT" | jq -r '.result.data.appName // .appName')
```

Deploy:
```bash
curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/postgres.deploy" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"postgresId\": \"$PG_ID\"}"
```

Wait and verify:
```bash
sleep 15
ssh root@SERVER_IP "docker ps --format '{{.Names}}' | grep pg-telegram-bots"
```

**Internal URL:** `postgresql://botuser:PG_PASSWORD@PG_ACTUAL_APPNAME:5432/bots`

Note: Use `PG_ACTUAL_APPNAME` (with suffix), NOT the name you requested.

### Create Shared Redis

```bash
REDIS_PASSWORD=$(openssl rand -base64 16 | tr -d '/+=')

REDIS_RESULT=$(curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/redis.create" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"redis-telegram-bots\",
    \"appName\": \"redis-telegram-bots\",
    \"projectId\": \"$PROJECT_ID\",
    \"environmentId\": \"$ENVIRONMENT_ID\",
    \"databasePassword\": \"$REDIS_PASSWORD\",
    \"dockerImage\": \"redis:7\"
  }")

REDIS_ID=$(echo "$REDIS_RESULT" | jq -r '.result.data.redisId // .redisId')
# CRITICAL: Read the actual appName
REDIS_ACTUAL_APPNAME=$(echo "$REDIS_RESULT" | jq -r '.result.data.appName // .appName')
```

Deploy:
```bash
curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/redis.deploy" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"redisId\": \"$REDIS_ID\"}"
```

**Internal URL:** `redis://:REDIS_PASSWORD@REDIS_ACTUAL_APPNAME:6379`
