# /manage-bot — Manage Deployed Telegram Bots

## Description
Manage Telegram bots deployed via `/deploy-bot`. Check status, view logs, redeploy, update env vars, start/stop bots. Supports both systemd and Dokploy deployment methods.

## Usage
- `/manage-bot` — show status of all bots
- `/manage-bot status` — show status of all bots
- `/manage-bot logs <name>` — show logs for a specific bot
- `/manage-bot redeploy <name>` — trigger manual redeploy
- `/manage-bot env <name>` — view/edit environment variables
- `/manage-bot stop <name>` — stop a bot
- `/manage-bot start <name>` — start a stopped bot
- `/manage-bot delete <name>` — remove a bot entirely
- `/manage-bot list` — list all deployed bots from config

## Instructions

### Step 0: Load Configuration

Read config from: `~/.claude/projects/<current-project>/memory/deploy-config.md`

If config doesn't exist, tell user to run `/deploy-bot` first.

Extract: Server IP, and for each bot: name, method (systemd/dokploy), app ID (if dokploy), service name and path (if systemd).

If Dokploy is used: extract Protocol, Dokploy URL, and API Key.

Parse the args passed to the skill to determine which subcommand to run.

---

### Subcommand: `status` (default)

Show status of all deployed bots.

For each bot in the config, check based on its **method**:

**Systemd bots:**
```bash
ssh root@SERVER_IP "systemctl is-active BOT_NAME 2>/dev/null; echo '---'; systemctl show BOT_NAME --property=ActiveEnterTimestamp --no-pager 2>/dev/null"
```

**Dokploy bots:**
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/application.one?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

Display a table:
```
| Bot Name | Method  | Status  | Type    | Last Deploy |
|----------|---------|---------|---------|-------------|
| my-bot   | systemd | active  | polling | 2024-01-15  |
| web-bot  | dokploy | running | webhook | 2024-01-14  |
```

---

### Subcommand: `logs <name>`

Find the bot in config. Check its **method**:

**Systemd:**
```bash
ssh root@SERVER_IP "journalctl -u BOT_NAME -n 100 --no-pager"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.readLogs" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Display the logs, showing the most recent entries. If logs are long, show the last 50 lines.

For Dokploy build/deployment logs:
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/deployment.all?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

---

### Subcommand: `redeploy <name>`

Find the bot in config. Check its **method**:

**Systemd:**
```bash
ssh root@SERVER_IP "cd /opt/BOT_NAME && git pull origin BRANCH"
```

Then reinstall deps based on language:
- Python: `ssh root@SERVER_IP "cd /opt/BOT_NAME && .venv/bin/pip install -r requirements.txt"`
- Node.js: `ssh root@SERVER_IP "cd /opt/BOT_NAME && npm ci --production"`
- Go: `ssh root@SERVER_IP "cd /opt/BOT_NAME && go build -o bot ."`

Restart:
```bash
ssh root@SERVER_IP "systemctl restart BOT_NAME"
```

Verify:
```bash
ssh root@SERVER_IP "sleep 2 && systemctl is-active BOT_NAME && journalctl -u BOT_NAME -n 5 --no-pager"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.redeploy" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Poll deployment status until done or error (max 3 minutes).

---

### Subcommand: `env <name>`

Find the bot in config. Check its **method**:

**Systemd:**

Read current env:
```bash
ssh root@SERVER_IP "cat /opt/BOT_NAME/.env"
```

Display variables (mask BOT_TOKEN and passwords — show only first 4 chars + `...`).

If user wants to update, write the new env file:
```bash
ssh root@SERVER_IP "cat > /opt/BOT_NAME/.env << 'ENVEOF'
KEY1=val1
KEY2=val2
ENVEOF"
ssh root@SERVER_IP "chmod 600 /opt/BOT_NAME/.env"
```

After updating, ask if user wants to restart for changes to take effect:
```bash
ssh root@SERVER_IP "systemctl restart BOT_NAME"
```

**Dokploy:**

Get current env:
```bash
curl -sk "PROTOCOL://DOKPLOY_URL/api/application.one?input=%7B%22json%22%3A%7B%22applicationId%22%3A%22APP_ID%22%7D%7D" \
  -H "x-api-key: API_KEY"
```

Display current environment variables (masked).

If user wants to update:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.saveEnvironment" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "applicationId": "APP_ID",
    "env": "FULL_ENV_STRING"
  }'
```

**Important:** `saveEnvironment` replaces ALL env vars. Always include the full set.

After updating, ask if user wants to redeploy for changes to take effect.

---

### Subcommand: `stop <name>`

**Systemd:**
```bash
ssh root@SERVER_IP "systemctl stop BOT_NAME"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.stop" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Confirm the bot has stopped.

---

### Subcommand: `start <name>`

**Systemd:**
```bash
ssh root@SERVER_IP "systemctl start BOT_NAME"
```

**Dokploy:**
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.start" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

Confirm the bot has started.

---

### Subcommand: `delete <name>`

**This is destructive!** Ask for confirmation before proceeding.

**Systemd:**
1. Stop the service: `systemctl stop BOT_NAME`
2. Disable the service: `systemctl disable BOT_NAME`
3. Remove unit file: `rm /etc/systemd/system/BOT_NAME.service`
4. Reload systemd: `systemctl daemon-reload`
5. Remove bot directory: `rm -rf /opt/BOT_NAME`

**Dokploy:**
1. Stop the application (if running)
2. If the bot has a custom domain, remove it:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/domain.delete" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domainId": "DOMAIN_ID"}'
```
3. Delete the application:
```bash
curl -sk -X POST "PROTOCOL://DOKPLOY_URL/api/application.delete" \
  -H "x-api-key: API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId": "APP_ID"}'
```

**Both methods:** Remove the bot entry from `deploy-config.md` and confirm deletion.

---

### Subcommand: `list`

Simply read and display the `## Deployed Bots` section from `deploy-config.md`.

Show each bot with: name, method, type, repo, status.

---

## Quick Reference: Method Dispatch

| Command | systemd | dokploy |
|---------|---------|---------|
| status | `systemctl is-active` + `show` | `application.one` API |
| logs | `journalctl -u <name> -n 100` | `application.readLogs` API |
| redeploy | `git pull && deps && systemctl restart` | `application.redeploy` API |
| env | read/write `/opt/<name>/.env` | `application.saveEnvironment` API |
| stop | `systemctl stop` | `application.stop` API |
| start | `systemctl start` | `application.start` API |
| delete | stop + disable + rm unit + rm -rf /opt/<name> | `application.delete` API |

---

## Error Handling

- If a bot name is not found in the config, list available bots and ask the user to pick one
- If Dokploy API returns an error, show the error message and suggest troubleshooting steps
- If SSH command fails, check if server is reachable
- If the config file is missing or malformed, suggest running `/deploy-bot`

## Reference Files
- `references/dokploy-api.md` — Dokploy API reference

## Important Notes
- Always read the latest config before any operation
- When showing env vars, mask sensitive values (tokens, passwords)
- For Dokploy: `saveEnvironment` replaces ALL vars — always include the full set
- After env changes, a restart/redeploy is needed for changes to take effect
- Deletion is permanent — always confirm with user
- Config file is now `deploy-config.md` (was `dokploy-config.md` in v1)
