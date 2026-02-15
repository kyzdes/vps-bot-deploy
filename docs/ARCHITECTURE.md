# Architecture: VPS Bot Deploy

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Developer's Machine (macOS)                                │
│                                                             │
│  Claude Code                                                │
│  ├── /tgbot ─── unified SKILL.md + 10 reference docs       │
│  │   ├── deploy  (server setup + bot deployment)            │
│  │   ├── manage  (status, logs, redeploy, env, stop/start)  │
│  │   └── prepare (Dockerfile, .env.example, refactoring)    │
│  └── memory/deploy-config.md  ─── persistent state          │
│                                                             │
│  Tools: SSH, gh CLI, curl, sshpass                          │
└──────────────┬──────────────────────┬───────────────────────┘
               │ SSH                  │ GitHub API (gh)
               ▼                     ▼
┌──────────────────────┐  ┌─────────────────────┐
│  VPS (Ubuntu 20.04+) │  │  GitHub              │
│                       │  │  └── user/bot-repo   │
│  Method A: systemd    │  └─────────────────────┘
│  ├── /opt/bot-name/   │
│  │   ├── .venv/       │
│  │   ├── .env         │
│  │   └── (code)       │
│  ├── bot-name.service │
│  └── journalctl logs  │
│                       │
│  Method B: Dokploy    │
│  ├── Dokploy (:3000)  │
│  │   └── tRPC API     │
│  ├── Traefik (proxy)  │
│  ├── [PostgreSQL]     │
│  ├── [Redis]          │
│  ├── bot containers   │
│  └── dokploy-network  │
└───────────────────────┘
```

## Component Model

### 1. Skill (Claude Code Instructions)

Skill — единый Markdown-файл с тремя flows (deploy, manage, prepare), который Claude Code интерпретирует как инструкции. Никакого исполняемого кода, CI/CD или агентов. Claude читает SKILL.md, роутит по подкоманде, следует шагам, вызывает bash-команды через SSH/curl.

```
skills/
└── tgbot/
    ├── SKILL.md                    # Единый скилл: deploy + manage + prepare
    └── references/
        ├── preflight-checks.md     # Deploy: валидация перед стартом
        ├── server-bootstrap.md     # Deploy: SSH + базовые пакеты
        ├── systemd-deployment.md   # Deploy: systemd-деплой
        ├── dokploy-setup.md        # Deploy: Dokploy-деплой
        ├── dokploy-api.md          # Deploy+Manage: Dokploy API справочник
        ├── secrets-detection.md    # Deploy: обнаружение секретов
        ├── dockerfile-templates.md # Deploy+Prepare: шаблоны Dockerfile
        ├── error-recovery.md       # Deploy: восстановление после ошибок
        ├── readiness-checks.md     # Prepare: проверки готовности
        └── migration-guides.md     # Prepare: паттерны миграции
```

### 2. State Management

Единственный файл состояния — `deploy-config.md`:

```
~/.claude/projects/<project>/memory/deploy-config.md
```

Формат — Markdown с фиксированной структурой:

```markdown
# Deploy Configuration

## Server
- IP: 185.x.x.x
- SSH: key-based

## Dokploy (optional)
- Protocol: http|https
- URL: <protocol>://<ip>:3000
- API Key: trpc_xxx
- Project ID: uuid
- Environment ID: uuid

## Shared PostgreSQL (optional)
- Postgres ID: uuid
- Actual App Name: pg-telegram-bots-a1b2c3
- Internal URL: postgresql://botuser:pass@pg-telegram-bots-a1b2c3:5432/bots

## Shared Redis (optional)
- Redis ID: uuid
- Actual App Name: redis-telegram-bots-d4e5f6
- Internal URL: redis://:pass@redis-telegram-bots-d4e5f6:6379

## Deployed Bots

### my-bot
- Repo: https://github.com/user/my-bot
- Branch: main
- Method: systemd
- Type: polling
- Service: my-bot.service, Path: /opt/my-bot
- Database: sqlite
- Deployed: 2024-01-15
```

Зачем Markdown, а не JSON/YAML:
- Claude Code нативно читает/пишет Markdown
- Человеко-читаемо, можно редактировать вручную
- Нет зависимостей на парсеры
- Совместимо с Claude memory system

### 3. Communication Channels

```
Developer ──── Claude Code ──── VPS
                  │
                  ├── SSH (commands)
                  │   ├── apt-get, git clone, systemctl
                  │   ├── Read/write files
                  │   └── Docker commands
                  │
                  ├── curl (Dokploy tRPC API)
                  │   ├── POST /api/router.procedure (mutations)
                  │   └── GET /api/router.procedure?input={...} (queries)
                  │
                  └── gh CLI (GitHub API)
                      ├── Repo metadata
                      ├── File contents (base64)
                      └── Branch info
```

## Data Flow: Deploy Bot

```
Phase 0: Pre-Flight
───────────────────
  Read deploy-config.md
  ├── exists? → verify SSH, check Dokploy, check Docker Hub
  └── missing? → first-time setup

Phase 1: Server Access
──────────────────────
  Ask: IP + root password
  ├── sshpass + ssh-copy-id → key-based auth
  ├── apt-get install python3 git curl jq
  └── Password discarded

Phase 2: Analyze Repo
─────────────────────
  gh API → fetch file tree, dependency files, source code
  ├── Detect: language, framework, entry point
  ├── Detect: database type (SQLite/PG/Redis/none)
  ├── Detect: polling vs webhook
  ├── Detect: secrets from .env.example, README, source
  ├── Classify: secrets / infrastructure / defaults / unknown
  └── Recommend: deploy method (systemd or Dokploy)

Phase 3A: Deploy (systemd)                Phase 3B: Deploy (Dokploy)
───────────────────────────                ───────────────────────────
  Install runtime (if needed)              Detect/install Dokploy
  git clone → /opt/<name>/                 Detect protocol (HTTP/HTTPS)
  venv + pip install (or npm ci)           Create admin / get API key
  Write .env (chmod 600)                   Create project + environment
  Create systemd unit                      [Create PG/Redis if needed]
  systemctl enable + start                 Create app + GitHub + build
                                           Set env vars
                                           Deploy + poll status

Phase 4: Verify
───────────────
  systemd: systemctl is-active + journalctl
  Dokploy: deployment.all status + readLogs
  ├── Success patterns: "Bot started", "Polling"
  └── Error patterns: "Traceback", "ModuleNotFoundError"

Phase 5: Save Config
────────────────────
  Update deploy-config.md with all IDs, methods, dates
```

## Data Flow: Manage Bot

```
Load Config
────────────
  Read deploy-config.md → extract bot list, methods, IDs

Route by Subcommand + Method
─────────────────────────────
  ┌──────────┬──────────────────────────┬────────────────────────────┐
  │ Command  │ systemd                  │ Dokploy                    │
  ├──────────┼──────────────────────────┼────────────────────────────┤
  │ status   │ systemctl is-active      │ application.one API        │
  │ logs     │ journalctl -u <name>     │ application.readLogs API   │
  │ redeploy │ git pull + deps + restart│ application.redeploy API   │
  │ env      │ read/write .env file     │ application.saveEnvironment│
  │ stop     │ systemctl stop           │ application.stop API       │
  │ start    │ systemctl start          │ application.start API      │
  │ delete   │ rm service + rm -rf code │ application.delete API     │
  └──────────┴──────────────────────────┴────────────────────────────┘
```

## Network Architecture

### systemd Method

```
Internet
  │
  ▼
VPS (bare metal)
  ├── my-bot (systemd service)
  │   ├── Polls Telegram API directly
  │   ├── /opt/my-bot/.env (secrets)
  │   └── /opt/my-bot/data/bot.db (SQLite)
  ├── another-bot (systemd service)
  └── SSH :22 (admin access)
```

Простая модель: каждый бот — отдельный процесс. Нет Docker, нет reverse proxy. Подходит для polling-ботов, которые сами ходят к Telegram API.

### Dokploy Method

```
Internet
  │
  ▼
VPS
  ├── Traefik (reverse proxy, auto-SSL)
  │   ├── *.bots.domain.com → bot containers
  │   └── Let's Encrypt certificates
  ├── Dokploy (:3000, management UI + tRPC API)
  ├── dokploy-network (Docker overlay)
  │   ├── webhook-bot (container, :8080)
  │   ├── pg-telegram-bots-XXXXXX (PostgreSQL)
  │   └── redis-telegram-bots-XXXXXX (Redis)
  └── SSH :22 (admin access)
```

Все контейнеры в `dokploy-network` видят друг друга по hostname = appName (с суффиксом). Traefik маршрутизирует внешний HTTPS на внутренний :8080 контейнера.

## Dokploy API Architecture

Dokploy использует **tRPC** — типизированный RPC поверх HTTP:

```
POST /api/<router>.<procedure>
  Headers: x-api-key: <token>, Content-Type: application/json
  Body: { "field": "value" }

GET /api/<router>.<procedure>?input=<url-encoded-json>
  Headers: x-api-key: <token>
```

### Critical API Gotchas

| Проблема | Решение |
|----------|---------|
| HTTP vs HTTPS | Детектировать перед первым вызовом; HTTPS + `-k` для self-signed |
| `environmentId` обязателен | Получить через `project.one` → `environments[0].environmentId` |
| `githubId` обязателен | `""` для публичных репо без GitHub App |
| `dockerContextPath` обязателен | `"/"` для корневого Dockerfile |
| `appName` в ответе != запросу | Dokploy добавляет 6-символьный суффикс. Всегда читать из response |
| Internal hostname = actual appName | Использовать appName С суффиксом в connection URLs |
| `saveEnvironment` заменяет всё | Всегда отправлять полный набор переменных |

### API Call Sequence (Application Deploy)

```
1. project.create / project.all        → projectId
2. project.one                          → environmentId
3. application.create                   → applicationId, actual appName
4. application.saveGithubProvider       (+ githubId: "")
5. application.saveBuildType            (+ dockerContextPath, dockerBuildStage)
6. application.saveEnvironment
7. [domain.create]                      (webhook bots only)
8. application.deploy
9. deployment.all (poll)                → status: done/error
```

## Security Model

```
Secrets Lifecycle:
──────────────────
  Root password → sshpass → ssh-copy-id → DISCARDED (never stored)
  SSH key → ~/.ssh/id_ed25519 (local machine only)
  Bot tokens → .env file (chmod 600) or Dokploy env vars
  PG/Redis passwords → openssl rand → deploy-config.md + env vars
  Dokploy API key → deploy-config.md (local only)

Display:
────────
  Secrets masked: "sk-proj-a1b2..." → "sk-p..."
  deploy-config.md lives in Claude memory dir (not committed to git)
```

## Error Recovery Architecture

Каждая фаза имеет детерминированную стратегию восстановления:

```
Error Category          │ Detection                    │ Recovery
────────────────────────┼──────────────────────────────┼─────────────────────────
SSH timeout             │ "Connection timed out"       │ Verify IP, check firewall
SSH key rejected        │ "Permission denied"          │ Re-copy key with password
Docker Hub rate limit   │ "toomanyrequests"            │ Switch to systemd
Dokploy port conflict   │ "port 3000 already in use"   │ Use existing Dokploy
DNS in Docker           │ "fetch failed"               │ Fix Docker DNS or systemd
Missing module          │ "ModuleNotFoundError"        │ Reinstall deps, check venv
Wrong entry point       │ "No module named 'bot'"      │ Detect correct entry point
Build fail (no Docker)  │ "Dockerfile not found"       │ Use nixpacks or /tgbot prepare
Bot polling conflict    │ "terminated by getUpdates"   │ Stop other bot instances
Interrupted deploy      │ Check current state          │ Resume from last phase
```

## Scalability Characteristics

| Resource | systemd | Dokploy |
|----------|---------|---------|
| RAM per bot | 30-80 MB | 80-200 MB (Docker overhead) |
| Base overhead | ~0 | ~500 MB (Dokploy + Traefik + [PG + Redis]) |
| Min server | 1 GB RAM | 2 GB RAM |
| Bots on 2 GB | 10-15 | 5-8 |
| Bots on 4 GB | 20+ | 10-15 |
| Docker dependency | No | Yes |
| SSL support | No (manual) | Yes (Traefik + Let's Encrypt) |
| DB isolation | File-level (SQLite) | Container-level (shared PG) |
