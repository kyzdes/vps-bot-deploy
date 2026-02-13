# Architecture Decisions

## ADR-1: systemd как метод деплоя по умолчанию

**Статус:** Принято

**Контекст:** Первая версия использовала только Dokploy (Docker). На практике большинство Telegram-ботов — простые polling-боты с SQLite или без БД. Docker добавляет overhead: 500+ MB RAM на базовую инфраструктуру, зависимость от Docker Hub (rate limits), DNS-проблемы в Docker Swarm.

**Решение:** systemd по умолчанию. Dokploy — опция для webhook-ботов и ботов с PostgreSQL.

**Последствия:**
- (+) Минимальный VPS (1 GB) теперь работает: 5-8 ботов
- (+) Нет зависимости от Docker Hub → нет rate limit проблем
- (+) Нет Docker DNS проблем (git clone на хосте)
- (+) Быстрее стартует, проще дебажить (journalctl)
- (-) Нет контейнерной изоляции (все боты под root)
- (-) Webhook-боты требуют ручной настройки reverse proxy
- (-) Обновление рантайма (Python/Node) влияет на все боты

**Матрица выбора:**

| БД бота | Тип | Метод |
|---------|-----|-------|
| SQLite / нет | polling | **systemd** |
| SQLite / нет | webhook | Dokploy (Traefik) |
| PostgreSQL | любой | Dokploy (shared PG) |

---

## ADR-2: Markdown для конфигурации вместо JSON/YAML

**Статус:** Принято

**Контекст:** Нужно хранить состояние между сессиями Claude Code: IP сервера, API keys, список ботов. Варианты: JSON, YAML, TOML, Markdown.

**Решение:** Markdown файл `deploy-config.md` в Claude memory directory.

**Обоснование:**
- Claude Code нативно читает и пишет Markdown (формат его "памяти")
- Человеко-читаемо: пользователь может открыть и проверить/отредактировать
- Нет зависимости на парсеры (jq для JSON, yq для YAML)
- Секции с заголовками `##` + ключ-значение через `- Key: value` — достаточно структурировано
- Совместимо с Claude's memory system (`memory/` directory)

**Альтернативы:**
- JSON: машинно-парсим, но менее читаем, Claude может ошибиться с запятыми
- YAML: хорош для конфигурации, но нет гарантии парсинга на стороне Claude
- SQLite: оверкилл для 10-20 строк конфигурации

---

## ADR-3: Dokploy API вместо прямого Docker CLI

**Статус:** Принято

**Контекст:** Для Docker-метода можно работать через Docker CLI напрямую (`docker run`, `docker compose`) или через Dokploy API.

**Решение:** Dokploy API (tRPC).

**Обоснование:**
- Dokploy управляет Traefik, SSL, деплоем из GitHub — не нужно реализовывать это вручную
- Web UI для пользователей, которые хотят посмотреть визуально
- API для автоматизации (наши скиллы)
- Manages Docker Swarm networking (dokploy-network)

**Gotchas (задокументированы в dokploy-api.md):**
- tRPC-style routing: `POST /api/router.procedure`, не REST
- GET-запросы через `?input=<url-encoded-json>`
- `environmentId` обязателен, но не очевиден — нужно достать из `project.one`
- `appName` в ответе всегда с суффиксом (рандомные 6 символов)
- `githubId: ""` обязателен для публичных репо
- `dockerContextPath` и `dockerBuildStage` обязательны
- `saveEnvironment` заменяет все переменные (не merge)

**Альтернативы:**
- Прямой Docker CLI: больше контроля, но нужно руками настраивать Traefik, SSL, networking
- Docker Compose: проще Dokploy, но нет UI, нет deploy-from-GitHub
- Coolify: более тяжёлый, больше overhead

---

## ADR-4: SSH-ключ через sshpass (одноразовый пароль)

**Статус:** Принято

**Контекст:** Для первого подключения к VPS нужен пароль. Варианты: просить пароль каждый раз, хранить пароль, настроить SSH-ключ.

**Решение:** Использовать sshpass для однократного `ssh-copy-id`, затем только ключ.

**Обоснование:**
- Пароль используется ровно один раз
- После `ssh-copy-id` пароль не нужен и не хранится
- `sshpass` — стандартная утилита для автоматизации SSH
- Fallback: если sshpass недоступен, пользователь может сам запустить `ssh-copy-id`

**Последствия:**
- (+) Безопасно: пароль не сохраняется
- (+) Автоматизировано: не требует ручных шагов
- (-) Требует `sshpass` (установка через brew)
- (-) Пароль виден в истории процессов на момент выполнения

---

## ADR-5: Обнаружение секретов из исходного кода

**Статус:** Принято

**Контекст:** Нужно знать, какие переменные окружения требует бот. Можно просить пользователя перечислить, или определить автоматически.

**Решение:** Многоуровневое сканирование: `.env.example` → README → docker-compose.yml → исходный код.

**Приоритет источников:**
1. `.env.example` — лучший источник: автор сам перечислил переменные
2. `README.md` — часто содержит секцию "Configuration"
3. `docker-compose.yml` — `environment:` секция
4. Исходный код — `os.getenv()`, `process.env.`, `os.Getenv()`

**Классификация переменных:**
- **Secrets** (спросить): `*TOKEN*`, `*API_KEY*`, `*SECRET*`, `*PASSWORD*`
- **Infrastructure** (авто): `DATABASE_URL`, `REDIS_URL` — только если бот использует
- **Defaults** (авто): `PYTHONUNBUFFERED=1`, `NODE_ENV=production`
- **Unknown** (показать): всё, что не попало в категории выше

**Критичное правило:** Infrastructure-переменные заполняются ТОЛЬКО если бот реально использует эту БД. SQLite-бот не получает DATABASE_URL с PostgreSQL.

---

## ADR-6: Shared PostgreSQL/Redis вместо per-bot

**Статус:** Принято

**Контекст:** Для Dokploy-деплоя, если бот использует PostgreSQL или Redis — выделять ли отдельную БД каждому боту?

**Решение:** Один shared PostgreSQL и один shared Redis на все боты в проекте. Каждый бот получает свою базу данных внутри shared PostgreSQL.

**Обоснование:**
- Экономия RAM: один PG-контейнер вместо нескольких
- Проще управлять: один бэкап, один connection pool
- Для 5-10 ботов shared PG справляется легко

**Последствия:**
- (+) Экономия ~200 MB RAM на каждый дополнительный бот
- (-) Единая точка отказа для всех ботов
- (-) Нет изоляции данных между ботами (разные БД, но один сервер)
- Пользователь может запросить отдельную БД при деплое

---

## ADR-7: Polling по умолчанию (не webhook)

**Статус:** Принято

**Контекст:** Telegram боты работают через polling (getUpdates) или webhook (Telegram шлёт HTTP на URL бота). Нужно выбрать дефолт.

**Решение:** Polling по умолчанию. Webhook — только если код явно его использует.

**Обоснование:**
- Polling работает без домена, без SSL, без reverse proxy
- 90%+ indie-ботов используют polling
- Webhook требует: домен → DNS → Cloudflare → Traefik → SSL → порт 8080
- Для webhook-бота всё это поднимается автоматически (через Dokploy)

**Детекция из кода:**
- Polling: `start_polling`, `run_polling`, `getUpdates`, `bot.polling`
- Webhook: `set_webhook`, `webhookCallback`, `express`, `fastapi`, `aiohttp.web`

---

## ADR-8: Claude Code Skill вместо CLI-утилиты

**Статус:** Принято

**Контекст:** Инструмент деплоя можно реализовать как:
- CLI-утилиту (bash/Python скрипт)
- GitHub Action
- Terraform/Ansible конфигурацию
- Claude Code Skill (Markdown-инструкции)

**Решение:** Claude Code Skill — набор Markdown-файлов, интерпретируемых Claude.

**Обоснование:**
- **Адаптивность:** Claude понимает контекст. Если бот нестандартный, Claude адаптирует шаги. Bash-скрипт просто сломается.
- **Error recovery:** Claude читает ошибку и находит решение. Скрипт показывает exit code 1.
- **Интерактивность:** Claude спрашивает секреты в диалоге, маскирует, объясняет что делает.
- **Zero dependencies:** Markdown-файл. Не нужен pip install, npm install, или отдельный runtime.
- **Maintainability:** Обновление — изменить текст в Markdown. Нет компиляции, нет пакетного менеджера.

**Компромиссы:**
- (-) Зависимость от Claude Code (нельзя использовать без него)
- (-) Нет детерминизма: Claude может выполнить шаги немного по-разному
- (-) Скорость: Claude "думает" между шагами; скрипт выполнил бы быстрее
- (-) Нет программных тестов (нельзя написать unit-тесты на Markdown)

---

## ADR-9: Три раздельных скилла вместо одного

**Статус:** Принято

**Контекст:** Можно сделать один скилл `/bot` с подкомандами (`/bot deploy`, `/bot manage`, `/bot prepare`) или три отдельных.

**Решение:** Три отдельных скилла: `/deploy-bot`, `/manage-bot`, `/prepare-bot`.

**Обоснование:**
- Каждый скилл имеет чёткую ответственность и свой набор reference-файлов
- Claude Code загружает только нужный SKILL.md → экономия контекста
- `/prepare-bot` — опциональный, не все его используют
- Именование через глагол (`deploy`, `manage`, `prepare`) интуитивнее

**Жизненный цикл:**
```
/prepare-bot  →  /deploy-bot  →  /manage-bot
(опционально)    (один раз)      (повседневно)
```

---

## ADR-10: `appName` с суффиксом — всегда читать из ответа

**Статус:** Принято (learned lesson)

**Контекст:** При создании ресурсов в Dokploy (`application.create`, `postgres.create`, `redis.create`) мы указываем `appName: "my-bot"`. Dokploy автоматически добавляет случайный 6-символьный суффикс: `my-bot-a1b2c3`. Этот actual appName используется как:
- Docker container name
- Internal DNS hostname в `dokploy-network`
- Часть connection URL для PostgreSQL/Redis

**Решение:** ВСЕГДА читать `appName` из response после create-запроса. Никогда не использовать запрошенное имя для hostname или connection URL.

**Пример ошибки:**
```
# WRONG: используем запрошенное имя
DATABASE_URL=postgresql://user:pass@pg-telegram-bots:5432/bots

# RIGHT: используем actual appName из ответа
DATABASE_URL=postgresql://user:pass@pg-telegram-bots-a1b2c3:5432/bots
```

---

## ADR-11: Docker Hub rate limit → fallback на systemd

**Статус:** Принято

**Контекст:** Docker Hub ограничивает анонимные pull-запросы до 100 за 6 часов на IP. VPS-хостеры часто делят IP между клиентами, что ускоряет достижение лимита. Dokploy-деплой зависит от Docker Hub (базовые образы).

**Решение:** При pre-flight check проверять rate limit. Если лимит достигнут — предупредить и рекомендовать systemd.

**Детекция:**
```bash
docker pull hello-world 2>&1 | grep -q "toomanyrequests"
```

**Recovery options (в порядке приоритета):**
1. Переключиться на systemd (нет Docker-зависимости)
2. Подождать 6 часов (лимит сбрасывается)
3. Залогиниться в Docker Hub (увеличивает лимит до 200)
4. Настроить Docker registry mirror

---

## ADR-12: Pre-flight — все проблемы сразу

**Статус:** Принято

**Контекст:** Можно проверять условия по одному (fail fast) или все разом (fail together).

**Решение:** Собрать все проверки в Phase 0 и показать все проблемы одним списком.

**Обоснование:**
- Пользователь видит полную картину: "SSH OK, Docker rate limited, repo OK"
- Может принять решение зная все факторы (например, выбрать systemd из-за rate limit)
- Не нужно проходить цикл "ошибка → исправить → следующая ошибка"

**Формат вывода:**
```
Pre-flight check results:

  [OK] SSH connection to 185.x.x.x
  [OK] Repository user/bot accessible
  [WARN] Docker Hub rate-limited
  [INFO] Bot uses SQLite — systemd recommended

Recommendation: Deploy via systemd.
```

---

## ADR-13: `environmentId` — обязательное поле

**Статус:** Принято (learned lesson)

**Контекст:** Документация Dokploy не акцентирует внимание на `environmentId`, но API его требует для `application.create`, `postgres.create`, `redis.create`. Без него запрос фейлится.

**Решение:** Задокументировать как CRITICAL. Получать через `project.one` → `environments[0].environmentId`.

**Последовательность:**
```
project.create → projectId
project.one(projectId) → environmentId
application.create(projectId, environmentId) → applicationId
```

Если environments пустой — создать через `environment.create`.
