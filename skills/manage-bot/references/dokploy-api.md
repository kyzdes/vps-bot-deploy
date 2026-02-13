# Dokploy API Reference (Corrected)

## Authentication

All requests require header: `x-api-key: <token>`

Base URL: `<PROTOCOL>://<SERVER_IP>:3000/api/`

**CRITICAL:** Protocol can be HTTP or HTTPS. Always detect first (see deploy-bot's `dokploy-setup.md`). For HTTPS, always use `-k` (self-signed certs).

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

## Application (Bot Management)

### application.one
**GET** `/api/application.one?input={"json":{"applicationId":"..."}}`
Full application details including status, env, domains.

### application.saveEnvironment
**POST** `/api/application.saveEnvironment`
```json
{
  "applicationId": "...",
  "env": "BOT_TOKEN=xxx\nDATABASE_URL=postgresql://..."
}
```
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

---

## Deployment

### deployment.all
**GET** `/api/deployment.all?input={"json":{"applicationId":"..."}}`
Returns array of deployments with status, dates, logs.

Deployment statuses: `running`, `done`, `error`

---

## Domain Management

### domain.byApplicationId
**GET** `/api/domain.byApplicationId?input={"json":{"applicationId":"..."}}`

### domain.delete
**POST** `/api/domain.delete`
```json
{ "domainId": "..." }
```

---

## Docker Network

All services within a Dokploy project share the `dokploy-network` Docker network.

**Internal hostnames** use the **actual appName** (with random suffix), NOT the name you requested.

Example: If PostgreSQL was created with `appName: "pg-telegram-bots"` and the response returned `appName: "pg-telegram-bots-a1b2c3"`, it's reachable at:
- `postgresql://user:pass@pg-telegram-bots-a1b2c3:5432/dbname`

**Always use the actual appName from deploy-config.md (stored as "Actual App Name") in connection URLs.**
