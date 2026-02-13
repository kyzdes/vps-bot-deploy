# Dockerfile Templates for Telegram Bots

## Python (aiogram / python-telegram-bot / Telethon)

### Standard Python Bot
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["python", "-m", "bot"]
```

### Python Bot with Poetry
```dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install poetry
COPY pyproject.toml poetry.lock ./
RUN poetry config virtualenvs.create false && poetry install --no-dev --no-interaction
COPY . .
CMD ["python", "-m", "bot"]
```

### Python Bot with uv
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
CMD ["uv", "run", "python", "-m", "bot"]
```

### Python Webhook Bot (aiohttp/FastAPI)
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "-m", "bot"]
```

---

## Node.js (grammY / Telegraf / node-telegram-bot-api)

### Standard Node.js Bot
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
CMD ["node", "src/index.js"]
```

### Node.js Bot with TypeScript
```dockerfile
FROM node:20-slim AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src/ ./src/
RUN npm run build

FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

### Node.js Bot with Bun
```dockerfile
FROM oven/bun:1
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --production
COPY . .
CMD ["bun", "run", "src/index.ts"]
```

### Node.js Webhook Bot
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 8080
CMD ["node", "src/index.js"]
```

---

## Go

### Go Bot
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o bot .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
WORKDIR /app
COPY --from=builder /app/bot .
CMD ["./bot"]
```

---

## Rust

### Rust Bot
```dockerfile
FROM rust:1.77-slim AS builder
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs && cargo build --release && rm -rf src
COPY src/ ./src/
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /app/target/release/bot .
CMD ["./bot"]
```

---

## Detection Logic

To auto-detect which template to use, check for these files in the repo:

| File | Language | Template |
|------|----------|----------|
| `requirements.txt` | Python | Standard Python |
| `pyproject.toml` + `poetry.lock` | Python | Poetry |
| `pyproject.toml` + `uv.lock` | Python | uv |
| `package.json` | Node.js | Standard Node.js |
| `tsconfig.json` + `package.json` | Node.js | TypeScript |
| `bun.lockb` | Node.js | Bun |
| `go.mod` | Go | Go |
| `Cargo.toml` | Rust | Rust |

## Webhook Port Convention

All webhook bots should expose port **8080** inside the container. The bot framework should listen on `0.0.0.0:8080`. Dokploy/Traefik routes external HTTPS traffic to this port.

## Health Check (optional)

For webhook bots, add a health endpoint:
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1
```

## .dockerignore (recommended)

Create alongside Dockerfile:
```
.git
.env
*.pyc
__pycache__
node_modules
.venv
```
