# VPS Bot Deploy

Скилл для Claude Code, позволяющий деплоить Telegram-ботов на VPS одной командой.

## Что это

Единый скилл `/tgbot`, который автоматизирует полный цикл развёртывания Telegram-ботов: от подготовки репозитория и настройки сервера до управления запущенными ботами.

**Два метода деплоя:**
- **systemd** (по умолчанию) — бот работает напрямую на сервере как systemd-сервис. Не требует Docker. Идеален для polling-ботов с SQLite.
- **Dokploy** (опционально) — бот работает в Docker-контейнере через Dokploy. Для webhook-ботов с Traefik или ботов, которым нужен PostgreSQL/Redis.

## Быстрый старт

### 1. Установи скилл

Скопируй папку `skills/tgbot` в `~/.claude/skills/`:

```bash
cp -R skills/tgbot ~/.claude/skills/
```

### 2. Задеплой бота

```
/tgbot deploy https://github.com/username/my-bot
```

Claude попросит IP сервера и root-пароль (только при первом запуске), проанализирует репозиторий, определит язык, базу данных и тип бота, и задеплоит его.

### 3. Управляй

```
/tgbot manage status          # статус всех ботов
/tgbot manage logs my-bot     # логи
/tgbot manage redeploy my-bot # обновить из git
```

### 4. Подготовь репо (если нужно)

```
/tgbot prepare https://github.com/username/my-bot
```

Добавит Dockerfile, `.env.example`, вынесет хардкод в переменные окружения.

## Команды

| Команда | Назначение |
|---------|-----------|
| `/tgbot deploy <repo>` | Настройка сервера + деплой бота |
| `/tgbot manage status` | Статус всех ботов |
| `/tgbot manage logs <name>` | Логи конкретного бота |
| `/tgbot manage redeploy <name>` | Обновить из git и перезапустить |
| `/tgbot manage env <name>` | Просмотр/редактирование переменных окружения |
| `/tgbot manage stop <name>` | Остановить бота |
| `/tgbot manage start <name>` | Запустить бота |
| `/tgbot manage delete <name>` | Удалить бота полностью |
| `/tgbot manage list` | Список всех ботов |
| `/tgbot prepare <repo>` | Подготовка репозитория к деплою |
| `/tgbot` | Интерактивное меню |

## Как выбирается метод деплоя

| База данных | Тип бота | Метод |
|-------------|----------|-------|
| SQLite / нет | polling | **systemd** |
| SQLite / нет | webhook | dokploy |
| PostgreSQL | любой | dokploy |

Можно переопределить вручную при деплое.

## Что автоматизировано

- SSH-ключ: пароль используется один раз, потом только ключ
- Анализ репозитория: язык, фреймворк, база данных, точка входа
- Определение секретов: из `.env.example`, README, исходного кода
- Выбор метода деплоя: на основе характеристик бота
- Установка зависимостей: pip, npm, go build
- Настройка сервиса: systemd unit или Dokploy app
- PostgreSQL/Redis: создаются только когда бот реально их использует

## Структура

```
skills/
└── tgbot/
    ├── SKILL.md                          # единый скилл (deploy + manage + prepare)
    └── references/
        ├── preflight-checks.md           # предварительные проверки
        ├── server-bootstrap.md           # SSH + базовые пакеты
        ├── systemd-deployment.md         # деплой через systemd
        ├── dokploy-setup.md              # установка Dokploy
        ├── dokploy-api.md                # API-справочник (с исправлениями)
        ├── secrets-detection.md          # поиск переменных окружения
        ├── dockerfile-templates.md       # шаблоны Dockerfile
        ├── error-recovery.md             # восстановление после ошибок
        ├── readiness-checks.md           # проверки готовности репо
        └── migration-guides.md           # миграции и рефакторинг
```

## Миграция с отдельных скиллов

Если вы использовали `/deploy-bot`, `/manage-bot`, `/prepare-bot` — они по-прежнему доступны в `skills/deploy-bot/`, `skills/manage-bot/`, `skills/prepare-bot/`. Единый `/tgbot` полностью их заменяет:

| Было | Стало |
|------|-------|
| `/deploy-bot <repo>` | `/tgbot deploy <repo>` |
| `/manage-bot status` | `/tgbot manage status` |
| `/manage-bot logs my-bot` | `/tgbot manage logs my-bot` |
| `/prepare-bot <repo>` | `/tgbot prepare <repo>` |

## Требования

- [Claude Code](https://claude.ai/claude-code)
- VPS (Ubuntu 20.04+, минимум 1 GB RAM для systemd, 2 GB для Dokploy)
- [GitHub CLI](https://cli.github.com/) (`gh auth status` должен быть залогинен)

## Документация

- [GUIDE.md](GUIDE.md) — пошаговая методичка с примерами
- [docs/PRD.md](docs/PRD.md) — продуктовые требования и user stories
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — техническая архитектура, data flows, security model
- [docs/DECISIONS.md](docs/DECISIONS.md) — архитектурные решения (13 ADR)
