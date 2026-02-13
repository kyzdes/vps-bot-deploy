# Dokploy API Reference (Corrected)

## Authentication

All requests require header: `x-api-key: <token>`

Base URL: `<PROTOCOL>://<SERVER_IP>:3000/api/`

**CRITICAL:** Protocol can be HTTP or HTTPS. Always detect first (see `dokploy-setup.md`). For HTTPS, always use `-k` (self-signed certs).

All endpoints use **tRPC-style** routing: `POST /api/<router>.<procedure>` with JSON body.

**Important:** GET endpoints use query params with `input` parameter (URL-encoded JSON). POST endpoints use JSON body.

---

## Critical Corrections (vs naive assumptions)

| Issue | Wrong | Correct |
|-------|-------|---------|
| Protocol | Always `https://` | Can be `http://` or `https://` — detect first |
| `environmentId` | Not included | **Required** for `postgres.create`, `redis.create`, `application.create` |
| `githubId` | Not included | **Required** for `application.saveGithubProvider` — use `""` for public repos |
| `dockerContextPath` | Not included | **Required** for `application.saveBuildType` — use `"/"` |
| `dockerBuildStage` | Not included | **Required** for `application.saveBuildType` — use `""` |
| `appName` in response | Same as requested | Dokploy appends random 6-char suffix. ALWAYS read from response |
| Internal hostname | Uses requested appName | Uses actual appName (with suffix) |

---

## Project Management

### project.create
**POST** `/api/project.create`
```json
{
  "name": "telegram-bots",
  "description": "Telegram bots infrastructure"
}
```
Response: `{ "result": { "data": { "projectId": "...", "name": "..." } } }`

### project.all
**GET** `/api/project.all`
Returns array of all projects with their services.

### project.one
**GET** `/api/project.one?input={"json":{"projectId":"..."}}`
Returns single project details including `environments` array.

**CRITICAL:** Use this to extract `environmentId`:
```bash
ENVIRONMENT_ID=$(echo "$RESPONSE" | jq -r '.result.data.environments[0].environmentId // .environments[0].environmentId')
```

### project.remove
**POST** `/api/project.remove`
```json
{ "projectId": "..." }
```

---

## Environment

### environment.create
**POST** `/api/environment.create`
```json
{
  "projectId": "...",
  "name": "production",
  "description": "Production environment"
}
```
Response includes `environmentId`.

---

## Application (Bot Deployment)

### application.create
**POST** `/api/application.create`
```json
{
  "name": "my-bot",
  "appName": "my-bot",
  "projectId": "...",
  "environmentId": "...",
  "description": "My Telegram Bot"
}
```

**CRITICAL fields:**
- `environmentId` — **Required.** Get from `project.one` → `environments[0].environmentId`
- Response `appName` will have a random suffix (e.g., `my-bot-a1b2c3`). **Always read the actual appName from the response.**

Response: `{ "result": { "data": { "applicationId": "...", "appName": "my-bot-XXXXXX" } } }`

### application.one
**GET** `/api/application.one?input={"json":{"applicationId":"..."}}`
Full application details including status, env, domains.

### application.saveGithubProvider
**POST** `/api/application.saveGithubProvider`
```json
{
  "applicationId": "...",
  "repository": "https://github.com/owner/repo",
  "branch": "main",
  "owner": "owner",
  "buildPath": "/",
  "githubId": ""
}
```

**CRITICAL:** `githubId` is **required**. Use `""` (empty string) for public repos cloned without a GitHub App.

### application.saveBuildType
**POST** `/api/application.saveBuildType`
```json
{
  "applicationId": "...",
  "buildType": "dockerfile",
  "dockerfile": "Dockerfile",
  "dockerContextPath": "/",
  "dockerBuildStage": ""
}
```

**CRITICAL:** `dockerContextPath` and `dockerBuildStage` are **required**.
- `dockerContextPath`: `"/"` for root-level Dockerfile
- `dockerBuildStage`: `""` for single-stage builds, or stage name for multi-stage

Valid buildTypes: `dockerfile`, `heroku_buildpacks`, `paketo_buildpacks`, `nixpacks`, `static`

### application.saveEnvironment
**POST** `/api/application.saveEnvironment`
```json
{
  "applicationId": "...",
  "env": "BOT_TOKEN=xxx\nDATABASE_URL=postgresql://...\nREDIS_URL=redis://..."
}
```
Environment variables as newline-separated KEY=VALUE pairs.

**Important:** This REPLACES all env vars. Always include the full set.

### application.deploy
**POST** `/api/application.deploy`
```json
{ "applicationId": "..." }
```

### application.redeploy
**POST** `/api/application.redeploy`
```json
{ "applicationId": "..." }
```

### application.stop
**POST** `/api/application.stop`
```json
{ "applicationId": "..." }
```

### application.start
**POST** `/api/application.start`
```json
{ "applicationId": "..." }
```

### application.reload
**POST** `/api/application.reload`
```json
{ "applicationId": "..." }
```

### application.delete
**POST** `/api/application.delete`
```json
{ "applicationId": "..." }
```

### application.readLogs
**POST** `/api/application.readLogs`
```json
{ "applicationId": "..." }
```

### application.readAppMonitoring
**POST** `/api/application.readAppMonitoring`
```json
{ "applicationId": "..." }
```

---

## Deployment

### deployment.all
**GET** `/api/deployment.all?input={"json":{"applicationId":"..."}}`
Returns array of deployments with status, dates, logs.

Deployment statuses: `running`, `done`, `error`

---

## PostgreSQL

### postgres.create
**POST** `/api/postgres.create`
```json
{
  "name": "pg-telegram-bots",
  "appName": "pg-telegram-bots",
  "projectId": "...",
  "environmentId": "...",
  "databaseName": "bots",
  "databaseUser": "botuser",
  "databasePassword": "secure-password-here",
  "dockerImage": "postgres:16"
}
```

**CRITICAL fields:**
- `environmentId` — **Required**
- Response `appName` will have suffix: `pg-telegram-bots-XXXXXX`
- **Internal hostname** is the actual appName WITH suffix, not the name you requested

Response includes `postgresId` and actual `appName`.

### postgres.deploy
**POST** `/api/postgres.deploy`
```json
{ "postgresId": "..." }
```

### postgres.one
**GET** `/api/postgres.one?input={"json":{"postgresId":"..."}}`

### postgres.remove
**POST** `/api/postgres.remove`
```json
{ "postgresId": "..." }
```

### postgres.saveEnvironment
**POST** `/api/postgres.saveEnvironment`
```json
{
  "postgresId": "...",
  "env": "POSTGRES_INITDB_ARGS=--encoding=UTF8"
}
```

### postgres.saveExternalPort
**POST** `/api/postgres.saveExternalPort`
```json
{
  "postgresId": "...",
  "externalPort": 5432
}
```

---

## Redis

### redis.create
**POST** `/api/redis.create`
```json
{
  "name": "redis-telegram-bots",
  "appName": "redis-telegram-bots",
  "projectId": "...",
  "environmentId": "...",
  "databasePassword": "secure-password-here",
  "dockerImage": "redis:7"
}
```

**CRITICAL:** `environmentId` is **required**. Response `appName` will have suffix.

Response includes `redisId` and actual `appName`.

### redis.deploy
**POST** `/api/redis.deploy`
```json
{ "redisId": "..." }
```

### redis.one
**GET** `/api/redis.one?input={"json":{"redisId":"..."}}`

### redis.remove
**POST** `/api/redis.remove`
```json
{ "redisId": "..." }
```

---

## Domain Management

### domain.create
**POST** `/api/domain.create`
```json
{
  "applicationId": "...",
  "host": "botname.bots.domain.com",
  "https": true,
  "port": 8080,
  "certificateType": "letsencrypt"
}
```

### domain.byApplicationId
**GET** `/api/domain.byApplicationId?input={"json":{"applicationId":"..."}}`

### domain.delete
**POST** `/api/domain.delete`
```json
{ "domainId": "..." }
```

### domain.generateDomain
**POST** `/api/domain.generateDomain`
```json
{ "applicationId": "..." }
```

---

## Docker Network

All services within a Dokploy project share the `dokploy-network` Docker network.

**Internal hostnames** use the **actual appName** (with random suffix), NOT the name you requested.

Example: If you created PostgreSQL with `appName: "pg-telegram-bots"` and the response returned `appName: "pg-telegram-bots-a1b2c3"`, it's reachable at:
- `postgresql://user:pass@pg-telegram-bots-a1b2c3:5432/dbname`

**Always read the actual appName from the create response and use that in connection URLs.**

---

## Settings / Admin

### admin.one
**GET** `/api/admin.one`
Returns admin user info and server details.

### settings.reloadServer
**POST** `/api/settings.reloadServer`
Restarts Dokploy.

---

## Auth (Initial Setup)

### auth.createAdmin
**POST** `/api/auth.createAdmin`
```json
{
  "email": "admin@bot.local",
  "password": "secure-password"
}
```
Only works before any admin is created. Returns error if admin already exists.

### auth.login
**POST** `/api/auth.login`
```json
{
  "email": "admin@bot.local",
  "password": "secure-password"
}
```
Returns session cookie.

### auth.generateToken
**POST** `/api/auth.generateToken`
Requires session cookie from login. Returns API token.

---

## Common Patterns

### Full app creation flow:
1. `project.create` (or find existing)
2. Get `environmentId` from `project.one`
3. `application.create` with `projectId` AND `environmentId`
4. Read actual `appName` from response
5. `application.saveGithubProvider` with `githubId: ""`
6. `application.saveBuildType` with `dockerContextPath: "/"` and `dockerBuildStage: ""`
7. `application.saveEnvironment` — set env vars
8. `domain.create` — for webhook bots only
9. `application.deploy` — trigger first deploy

### Check deployment status:
1. `application.deploy` returns immediately
2. Poll `deployment.all` to check latest deployment status
3. Look for `status: "done"` or `status: "error"`

### Error debugging:
1. `application.readLogs` — container logs
2. `deployment.all` — deployment history and build logs
