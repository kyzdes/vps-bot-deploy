# Error Recovery Reference

Common deployment failures and how to recover from them.

---

## SSH Failures

### "Connection timed out" / "No route to host"

**Cause:** Server is down, IP is wrong, or firewall blocks port 22.

**Recovery:**
1. Verify the IP address with the user
2. Ask user to check server is powered on
3. Check if a firewall rule blocks SSH: some VPS providers have a web console

### "Permission denied (publickey)"

**Cause:** SSH key not accepted, key-based auth required but key not copied.

**Recovery:**
1. Check if the key exists: `ls ~/.ssh/id_ed25519`
2. Re-copy the key: need the password again
3. Fallback: ask user to run `ssh-copy-id root@SERVER_IP` manually

### "Host key verification failed"

**Cause:** Server was reinstalled and has a new host key.

**Recovery:**
```bash
ssh-keygen -R SERVER_IP
```
Then retry the connection.

---

## Docker Hub Rate Limit

### "toomanyrequests: You have reached your pull rate limit"

**Cause:** Docker Hub limits unauthenticated pulls to 100/6h per IP.

**Recovery options:**
1. **Switch to systemd** (recommended) — no Docker needed for the bot
2. **Wait** — rate limit resets after 6 hours
3. **Authenticate** — `docker login` with Docker Hub account increases limit to 200/6h
4. **Use a mirror** — configure Docker daemon to use a registry mirror

```bash
# Check current rate limit status
ssh root@SERVER_IP "curl -s 'https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull' | jq -r .token | xargs -I {} curl -sI -H 'Authorization: Bearer {}' 'https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest' 2>&1 | grep -i ratelimit"
```

---

## Dokploy Installation Failures

### "Port 3000 already in use"

**Cause:** Dokploy (or another service) is already running on port 3000.

**Recovery:**
1. Check what's on port 3000: `ssh root@SERVER_IP "ss -tlnp | grep :3000"`
2. If it's Dokploy → skip install, use existing instance
3. If it's something else → ask user what to do

### Dokploy installer hangs / times out

**Cause:** Slow network, Docker pull issues, low disk space.

**Recovery:**
1. Check disk space: `ssh root@SERVER_IP "df -h /"`
2. Check if Docker is installed: `ssh root@SERVER_IP "docker --version"`
3. Try installing Docker separately first:
```bash
ssh root@SERVER_IP "curl -fsSL https://get.docker.com | sh"
```
4. Then retry Dokploy install

---

## Dokploy API Failures

### 401 Unauthorized

**Cause:** Invalid API key.

**Recovery:**
1. Ask user to regenerate API key from Dokploy dashboard
2. URL: `${PROTOCOL}://SERVER_IP:3000` → Settings → Profile → API

### "HTTPS request failed" / SSL errors

**Cause:** Dokploy may be running on HTTP, not HTTPS.

**Recovery:**
1. Try HTTP: `curl -sf http://SERVER_IP:3000/api/admin.one -H "x-api-key: KEY"`
2. If HTTP works, update config to use `http://` protocol
3. For HTTPS with self-signed cert, always use `-k` flag

### "fetch failed" / DNS resolution error in Dokploy container

**Cause:** Docker Swarm network DNS issues — container can't resolve external hosts.

**Recovery:**
1. This affects `git clone` inside Dokploy container
2. Check Docker DNS: `ssh root@SERVER_IP "docker exec \$(docker ps -q -f name=dokploy) cat /etc/resolv.conf"`
3. Fix by adding DNS to Docker daemon config:
```bash
ssh root@SERVER_IP "echo '{\"dns\":[\"8.8.8.8\",\"1.1.1.1\"]}' > /etc/docker/daemon.json && systemctl restart docker"
```
4. **Better alternative:** switch to systemd method (git clone runs directly on host, no Docker DNS issues)

---

## Git Clone Failures

### Inside Dokploy (Docker network)

**Cause:** Docker Swarm DNS can't resolve github.com.

**Recovery:** See DNS fix above, or switch to systemd.

### On host (systemd method)

**Cause:** git not installed, or network issues.

```bash
ssh root@SERVER_IP "which git || apt-get install -y git"
ssh root@SERVER_IP "git ls-remote https://github.com/OWNER/REPO.git HEAD"
```

If git works but clone fails:
1. Check disk space: `df -h /opt`
2. Check if `/opt/BOT_NAME` already exists (use `git pull` instead)

---

## Build / Dependency Failures

### Python: "ModuleNotFoundError"

**Cause:** Dependencies not installed, or wrong Python version.

**Recovery:**
1. Check venv exists: `ls /opt/BOT_NAME/.venv/bin/python`
2. Reinstall deps: `.venv/bin/pip install -r requirements.txt`
3. Check Python version matches requirements

### Python: "No module named 'bot'"

**Cause:** Entry point wrong. The module name in `ExecStart` doesn't match the actual package.

**Recovery:**
1. Check what's in the repo: `ls /opt/BOT_NAME/`
2. Look for `__main__.py`: `find /opt/BOT_NAME -name "__main__.py"`
3. Update the systemd unit with the correct entry point

### Node.js: "Cannot find module"

**Cause:** `npm ci` failed or `node_modules` missing.

**Recovery:**
1. Check: `ls /opt/BOT_NAME/node_modules`
2. Reinstall: `cd /opt/BOT_NAME && rm -rf node_modules && npm ci --production`

### Dokploy build: "Dockerfile not found"

**Cause:** No Dockerfile in repo, or wrong path.

**Recovery:**
1. Use nixpacks build type instead of dockerfile
2. Or run `/prepare-bot` to add a Dockerfile to the repo

---

## Runtime Failures

### Bot crashes immediately (systemd)

```bash
ssh root@SERVER_IP "journalctl -u BOT_NAME -n 50 --no-pager"
```

Common causes:
- Missing env vars → check `.env` file
- Wrong entry point → check `ExecStart` in unit file
- Database not accessible → check connection string
- Bot token invalid → verify with user

### Bot crashes immediately (Dokploy)

```bash
curl -sk -X POST "${PROTOCOL}://SERVER_IP:3000/api/application.readLogs" \
  -H "x-api-key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"applicationId":"APP_ID"}'
```

Common causes:
- Same as systemd above
- Docker network issues → container can't reach external APIs
- Port conflict → another container using the same port

### Bot runs but doesn't respond to messages

1. Check bot token is valid: user should verify with @BotFather
2. Check for polling conflict: only one instance can poll at a time
3. Check logs for "Conflict: terminated by other getUpdates request"
4. Solution: stop any other running instances of the same bot

---

## Resume After Interruption

If a deployment was interrupted mid-way, check the deploy config for the last completed phase.

### Determine current state

```bash
# Server accessible?
ssh -o ConnectTimeout=5 root@SERVER_IP "echo ok"

# Bot directory exists?
ssh root@SERVER_IP "ls /opt/BOT_NAME 2>/dev/null && echo 'exists'"

# Service exists?
ssh root@SERVER_IP "systemctl cat BOT_NAME 2>/dev/null && echo 'service-exists'"

# Service running?
ssh root@SERVER_IP "systemctl is-active BOT_NAME 2>/dev/null"

# Dokploy app exists?
curl -sk "${PROTOCOL}://SERVER_IP:3000/api/application.one?input=..." -H "x-api-key: KEY"
```

### Resume logic

| State | Action |
|-------|--------|
| Server not accessible | Start from Phase 1 (server access) |
| Server OK, no bot dir | Start from Phase 3 (setup environment) |
| Bot dir exists, no service | Create service and start (Phase 3, step 6+) |
| Service exists, not running | Check logs, fix issue, start |
| Service running | Deployment complete, verify |
