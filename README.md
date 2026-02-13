# VPS Bot Deploy

Набор скиллов для Claude Code, позволяющий деплоить Telegram-ботов на VPS одной командой.

## Что это

Три скилла (`/deploy-bot`, `/manage-bot`, `/prepare-bot`), которые автоматизируют полный цикл развёртывания Telegram-ботов: от настройки сервера до управления запущенными ботами.

**Два метода деплоя:**
- **systemd** (по умолчанию) — бот работает напрямую на сервере как systemd-сервис. Не требует Docker. Идеален для polling-ботов с SQLite.
- **Dokploy** (опционально) — бот работает в Docker-контейнере через Dokploy. Для webhook-ботов с Traefik или ботов, которым нужен PostgreSQL/Redis.

## Быстрый старт

### 1. Установи скиллы

Скопируй папки из `skills/` в `~/.claude/skills/`:

```bash
cp -R skills/deploy-bot ~/.claude/skills/
cp -R skills/manage-bot ~/.claude/skills/
cp -R skills/prepare-bot ~/.claude/skills/
```

### 2. Задеплой бота

```
/deploy-bot https://github.com/username/my-bot
```

Claude попросит IP сервера и root-пароль (только при первом запуске), проанализирует репозиторий, определит язык, базу данных и тип бота, и задеплоит его.

### 3. Управляй

```
/manage-bot status          # статус всех ботов
/manage-bot logs my-bot     # логи
/manage-bot redeploy my-bot # обновить из git
```

## Скиллы

| Скилл | Назначение |
|-------|-----------|
| `/deploy-bot` | Настройка сервера + деплой бота |
| `/manage-bot` | Статус, логи, редеплой, переменные, старт/стоп, удаление |
| `/prepare-bot` | Подготовка репозитория: Dockerfile, .env.example, рефакторинг конфига |

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
├── deploy-bot/
│   ├── SKILL.md                          # основной скилл (5 фаз)
│   └── references/
│       ├── preflight-checks.md           # предварительные проверки
│       ├── server-bootstrap.md           # SSH + базовые пакеты
│       ├── systemd-deployment.md         # деплой через systemd
│       ├── dokploy-setup.md              # установка Dokploy
│       ├── dokploy-api.md                # API-справочник (с исправлениями)
│       ├── secrets-detection.md          # поиск переменных окружения
│       ├── dockerfile-templates.md       # шаблоны Dockerfile
│       └── error-recovery.md             # восстановление после ошибок
├── manage-bot/
│   ├── SKILL.md                          # управление ботами
│   └── references/
│       └── dokploy-api.md                # API для manage-операций
└── prepare-bot/
    ├── SKILL.md                          # подготовка репозитория
    └── references/
        ├── readiness-checks.md           # проверки готовности
        └── migration-guides.md           # миграции и рефакторинг
```

## Требования

- [Claude Code](https://claude.ai/claude-code)
- VPS (Ubuntu 20.04+, минимум 1 GB RAM для systemd, 2 GB для Dokploy)
- [GitHub CLI](https://cli.github.com/) (`gh auth status` должен быть залогинен)

## Документация

Подробная инструкция с примерами: [GUIDE.md](GUIDE.md)
