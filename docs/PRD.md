# PRD: VPS Bot Deploy

## Problem

Деплой Telegram-ботов на VPS — рутинный, но трудоёмкий процесс. Разработчик тратит 30-60 минут на каждого бота: настройка сервера, SSH, установка зависимостей, systemd/Docker, переменные окружения, проверка логов. При этом 90% шагов одинаковы для всех ботов.

Существующие решения (PaaS вроде Railway/Render) стоят $5-20/мес за бота. VPS за $3-5/мес может держать 10+ ботов, но требует ручной настройки.

## Target Users

**Indie-разработчики Telegram-ботов**, которые:
- Пишут ботов на Python (aiogram, python-telegram-bot), Node.js (grammY, Telegraf) или Go
- Имеют VPS, но не хотят тратить время на DevOps
- Используют Claude Code как основной инструмент разработки
- Хотят деплоить одной командой, управлять без SSH

## Solution

Набор из трёх Claude Code skills, которые превращают деплой в диалог:

| Skill | Назначение | Частота использования |
|-------|-----------|----------------------|
| `/deploy-bot` | Настройка сервера + деплой бота | При каждом новом боте |
| `/manage-bot` | Статус, логи, редеплой, env, start/stop, delete | Повседневное управление |
| `/prepare-bot` | Подготовка репо (Dockerfile, .env.example) | Одноразово перед первым деплоем |

## User Stories

### US-1: Первый деплой (новый сервер)
> Как разработчик, я хочу задеплоить бота на чистый VPS одной командой, чтобы не настраивать сервер вручную.

**Acceptance Criteria:**
- Пользователь вводит `/deploy-bot https://github.com/user/bot`
- Claude спрашивает только IP и root-пароль
- SSH-ключ настраивается автоматически (пароль больше не нужен)
- Claude анализирует репо: язык, БД, polling/webhook, секреты
- Claude выбирает метод деплоя (systemd/Dokploy) на основе анализа
- Пользователь предоставляет только значения секретов (BOT_TOKEN и т.д.)
- Бот запущен и отвечает в Telegram
- Конфигурация сохранена для будущих деплоев

### US-2: Последующие деплои (сервер уже настроен)
> Как разработчик, я хочу добавить ещё одного бота на тот же сервер без повторной настройки.

**Acceptance Criteria:**
- Claude читает сохранённую конфигурацию сервера
- Проверяет SSH-доступ (pre-flight check)
- Пропускает настройку сервера, сразу анализирует репо
- Деплоит на тот же сервер рядом с другими ботами

### US-3: Просмотр статуса
> Как разработчик, я хочу видеть, работают ли мои боты, одной командой.

**Acceptance Criteria:**
- `/manage-bot` или `/manage-bot status` показывает таблицу всех ботов
- Для каждого бота: имя, метод (systemd/dokploy), статус, тип, дата деплоя
- Работает одинаково для systemd и Dokploy ботов

### US-4: Просмотр логов
> Как разработчик, я хочу быстро посмотреть логи бота при проблемах.

**Acceptance Criteria:**
- `/manage-bot logs my-bot` показывает последние 100 строк логов
- systemd: `journalctl`, Dokploy: API `readLogs`
- Claude комментирует ошибки, если видит их в логах

### US-5: Редеплой после обновления кода
> Как разработчик, я хочу обновить бота на сервере после пуша в GitHub.

**Acceptance Criteria:**
- `/manage-bot redeploy my-bot` обновляет код и перезапускает
- systemd: `git pull` + переустановка зависимостей + `systemctl restart`
- Dokploy: API `application.redeploy` + ожидание завершения
- Верификация: бот запустился без ошибок

### US-6: Управление переменными окружения
> Как разработчик, я хочу менять секреты/настройки без ручного SSH.

**Acceptance Criteria:**
- `/manage-bot env my-bot` показывает текущие переменные (маскируя секреты)
- Можно изменить значения, Claude обновит файл/.env/API
- После обновления — предложение перезапустить бота

### US-7: Удаление бота
> Как разработчик, я хочу полностью убрать бота с сервера.

**Acceptance Criteria:**
- `/manage-bot delete my-bot` с обязательным подтверждением
- Полная очистка: остановка, удаление файлов/контейнера, удаление из конфигурации
- systemd: stop + disable + rm unit + rm -rf /opt/name
- Dokploy: stop + delete app + delete domain (если есть)

### US-8: Подготовка репозитория
> Как разработчик, я хочу узнать, что нужно добавить в репо перед деплоем.

**Acceptance Criteria:**
- `/prepare-bot https://github.com/user/bot` клонирует и анализирует
- Показывает отчёт: что есть, чего не хватает
- Предлагает: добавить Dockerfile, создать .env.example, вынести хардкод
- Коммитит и пушит изменения (с подтверждением)

## Functional Requirements

### FR-1: Анализ репозитория
- Определение языка по файлам зависимостей (requirements.txt, package.json, go.mod, Cargo.toml)
- Определение фреймворка по зависимостям (aiogram, grammy, telegraf и др.)
- Определение точки входа (\_\_main\_\_.py, main.py, package.json main/start)
- Определение типа БД: SQLite, PostgreSQL, Redis, нет
- Определение polling vs webhook по исходному коду
- Обнаружение секретов из .env.example, README, docker-compose.yml, исходного кода

### FR-2: Выбор метода деплоя
- Автоматическая рекомендация на основе матрицы:

| БД | Тип | Рекомендация |
|----|-----|-------------|
| SQLite / нет | polling | systemd |
| SQLite / нет | webhook | Dokploy |
| PostgreSQL | любой | Dokploy |

- Пользователь всегда может переопределить

### FR-3: Настройка сервера (первый раз)
- SSH-ключ: `sshpass` + `ssh-copy-id` (пароль одноразово)
- Базовые пакеты: python3, git, curl, jq
- Рантайм по языку бота: Python venv, Node.js, Go

### FR-4: Деплой через systemd
- Клонирование в `/opt/<bot-name>/`
- Виртуальное окружение + установка зависимостей
- `.env` файл (chmod 600)
- Systemd unit file с автозапуском и Restart=always
- Логи через journalctl

### FR-5: Деплой через Dokploy
- Установка Dokploy (если нет)
- Создание admin / получение API key
- Создание project, получение environmentId
- Создание application + настройка GitHub + build type + env vars
- Создание shared PG/Redis (только если бот их использует)
- Деплой + polling статуса

### FR-6: Классификация секретов
- **Secrets** (спросить у пользователя): токены, API-ключи, пароли
- **Infrastructure** (заполнить автоматически): DATABASE_URL, REDIS_URL — только если бот их использует
- **Defaults** (заполнить автоматически): PYTHONUNBUFFERED, PORT, NODE_ENV
- **Unknown** (показать пользователю): всё остальное

### FR-7: Управление ботами
- Status: таблица всех ботов
- Logs: последние записи (journalctl / API)
- Redeploy: git pull + deps + restart / API redeploy
- Env: чтение + редактирование + маскировка секретов
- Stop/Start: systemctl / API
- Delete: полная очистка + удаление из конфигурации

### FR-8: Pre-flight проверки
- SSH-доступ к серверу
- Docker Hub rate limit (для Dokploy)
- Доступ к репозиторию (через gh CLI)
- Состояние Dokploy (если уже установлен)
- Все проблемы отображаются разом, а не по одной

### FR-9: Персистентная конфигурация
- Хранится в `~/.claude/projects/<project>/memory/deploy-config.md`
- Содержит: IP сервера, метод, Dokploy URL/API key, project/environment ID, список ботов
- Обновляется после каждого деплоя/удаления

## Non-Functional Requirements

### NFR-1: Безопасность
- Пароль используется единожды для SSH key setup, не хранится
- `.env` файлы: chmod 600
- Секреты маскируются при отображении (первые 4 символа + `...`)
- API key Dokploy хранится только в локальном конфиге
- Пароли PG/Redis генерируются через `openssl rand`

### NFR-2: Идемпотентность
- Повторный запуск `/deploy-bot` на тот же сервер не ломает существующее
- Dokploy не переустанавливается, если уже стоит
- Бот обновляется (git pull + restart), а не дублируется
- Shared PG/Redis не пересоздаются

### NFR-3: Восстанавливаемость
- Прерванный деплой можно продолжить (проверка текущего состояния)
- Для каждой типичной ошибки есть recovery-стратегия
- При неудаче Dokploy — fallback на systemd
- При Docker Hub rate limit — предупреждение + альтернатива

### NFR-4: Производительность
- Systemd-бот: 30-80 MB RAM
- Сервер 1 GB: 5-8 polling-ботов
- Сервер 2 GB: 10-15 polling или 5-8 Dokploy
- Сервер 4 GB: 20+ polling или 10-15 Dokploy

### NFR-5: Поддерживаемые стеки
- Python: pip, poetry, uv
- Node.js: npm, bun
- Go: go build
- Rust: cargo build (Dockerfile only)
- Фреймворки: aiogram, python-telegram-bot, Telethon, grammY, Telegraf, node-telegram-bot-api, telebot (Go)

## Out of Scope (v2)

- Автоматический CI/CD (webhook на push в GitHub)
- Мониторинг и алерты (Telegram-уведомления о падении)
- Мультисерверный деплой (распределение по нескольким VPS)
- Backup/restore SQLite данных
- Автоскейлинг
- Webhook бот через systemd + Caddy/nginx (без Dokploy)
- Windows-серверы
