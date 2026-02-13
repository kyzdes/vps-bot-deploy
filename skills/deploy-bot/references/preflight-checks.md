# Pre-Flight Checks Reference

Validate everything before starting deployment. Report ALL issues at once so the user can decide how to proceed.

---

## Check 1: Configuration File

Read `~/.claude/projects/<current-project>/memory/deploy-config.md`.

- **Config exists** → extract server IP, method, Dokploy info (if any)
- **Config missing** → first-time setup, proceed to server access phase

---

## Check 2: SSH Connectivity

If config exists with a server IP:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 root@SERVER_IP "echo ok" 2>&1
```

Possible outcomes:
- `ok` → SSH works, server is reachable
- `Permission denied` → SSH key not accepted, need to re-setup
- `Connection timed out` / `No route to host` → server unreachable
- `Connection refused` → SSH not running on server

---

## Check 3: Docker Hub Rate Limit (Dokploy method only)

Only check if the configured or chosen method is `dokploy`:

```bash
ssh root@SERVER_IP "docker pull hello-world 2>&1"
```

Parse output for:
- `toomanyrequests` or `rate limit` → Docker Hub is rate-limited
- `pull access denied` → different issue (auth)
- Success → Docker Hub accessible

If rate-limited:
- Warn the user: "Docker Hub is rate-limited on this server. Dokploy deployments may fail."
- Suggest switching to `systemd` method (no Docker dependency for the bot itself)
- If user insists on Dokploy, suggest configuring a Docker Hub login or mirror

---

## Check 4: Repository Access

If a repo URL is provided:

```bash
gh api repos/OWNER/REPO --jq '.default_branch' 2>&1
```

Possible outcomes:
- Returns branch name → repo accessible
- `Not Found` → repo doesn't exist or is private and `gh` isn't authenticated
- `gh: command not found` → GitHub CLI not installed

Also check repo readiness indicators:
```bash
gh api repos/OWNER/REPO/contents/ --jq '.[].name' 2>&1
```

Look for: `Dockerfile`, `.env.example`, `requirements.txt`, `package.json`, `go.mod`, etc.

---

## Check 5: Dokploy Status (if method=dokploy)

If config says method is dokploy, or server already has Dokploy:

```bash
ssh root@SERVER_IP "docker ps --format '{{.Names}}' 2>/dev/null | grep -q dokploy && echo 'running' || echo 'not-running'"
```

If Dokploy is running, also check API access:
```bash
# Try HTTP first (more common), then HTTPS
curl -sf -o /dev/null -w "%{http_code}" "http://SERVER_IP:3000/api/admin.one" -H "x-api-key: API_KEY" --max-time 5 2>/dev/null
```

- `200` → API works
- `401` → API key invalid
- `000` / timeout → try HTTPS:
```bash
curl -sfk -o /dev/null -w "%{http_code}" "https://SERVER_IP:3000/api/admin.one" -H "x-api-key: API_KEY" --max-time 5 2>/dev/null
```

---

## Check 6: Existing Dokploy Detection (fresh server)

On a server without config, detect if Dokploy is already installed:

```bash
ssh root@SERVER_IP "docker ps --format '{{.Names}}' 2>/dev/null | grep dokploy"
```

If Dokploy container found:
- Do NOT try to install Dokploy again (port 3000 conflict)
- Check if admin exists: try `POST /api/auth.createAdmin` — if 401/error, admin already exists
- Ask user for the existing API key
- Detect protocol (HTTP vs HTTPS) — see Check 5

---

## Report Format

Present all findings at once:

```
Pre-flight check results:

  [OK] SSH connection to 72.56.93.172
  [OK] Repository kyzdes/open-llm-chatbot-psy accessible
  [WARN] Docker Hub rate-limited on this server
  [INFO] Dokploy already installed on server
  [INFO] Bot uses SQLite — systemd recommended

Recommendation: Deploy via systemd (no Docker dependency needed).
Proceed? Or switch to Dokploy?
```

---

## Decision Matrix

| Bot DB | Bot Type | Docker Hub | Dokploy Present | Recommended Method |
|--------|----------|------------|-----------------|-------------------|
| SQLite | polling | any | any | systemd |
| SQLite | webhook | OK | yes | dokploy |
| SQLite | webhook | limited | any | systemd + manual webhook |
| PostgreSQL | polling | OK | yes | dokploy (for shared PG) |
| PostgreSQL | polling | limited | any | systemd + local PG |
| PostgreSQL | webhook | OK | yes | dokploy |
| none | polling | any | any | systemd |

The user can always override the recommendation.
