# Hermes Telegram Mini App — Setup Guide

A complete walkthrough for deploying the Hermes Telegram Mini App. This guide covers everything from scratch, including every error we hit and how we fixed it.

## What You'll End Up With

- A terminal-style web interface inside Telegram
- Streaming chat with your Hermes agent
- Cron job management
- System monitoring
- Ed25519 authentication (no API key exposed to clients)

## Architecture (Current — Standalone Proxy)

The mini app runs through a **standalone proxy on port 8643** that sits between Cloudflare and the stock Hermes gateway. Hermes is never modified and can be updated freely.

```
Telegram Client
    |
    v
Telegram WebApp (mini app HTML/JS in Telegram's browser)
    |
    |  X-Telegram-Init-Data header (Ed25519 signed)
    |  X-Hermes-Session-Id header
    |  Authorization: Bearer *** (fallback for browser access)
    |
    v
Cloudflare Tunnel (hs.yourdomain.com -- HTTPS)
    |
    v
Mini App Proxy (localhost:8643)            <-- standalone service
    |
    |  Validates Telegram initData (Ed25519 + HMAC)
    |  Translates to Bearer token for Hermes
    |  Serves static files from ~/.hermes/miniapp/
    |  Provides missing endpoints locally
    |  Enriches /health with system metrics
    |
    v
Hermes Gateway (localhost:8642)            <-- UNMODIFIED STOCK
    |
    |  /v1/chat/completions  (streaming chat)
    |  /api/jobs/*           (cron CRUD)
    |  /v1/models            (model list)
    |  /health               (basic health)
```

> **Why this architecture?** Previously the mini app required patching `gateway/platforms/api_server.py` with custom routes, CORS headers, and Telegram auth. Hermes updates overwrote those modifications, causing 404s on all mini-app endpoints. The standalone proxy makes mini-app support survive Hermes updates because Hermes core is never touched.

---

## Prerequisites

- Hermes Agent v0.8.0+ running as a gateway on port 8642
- A Telegram bot (created via @BotFather)
- `cloudflared` CLI installed and authenticated
- A domain/subdomain you control (for Cloudflare Tunnel)
- Python package `pynacl` for Ed25519 verification (already in Hermes venv)

---

## Step 1 — Get the Mini App Frontend

```bash
mkdir -p ~/.hermes/miniapp
curl -o ~/.hermes/miniapp/index.html \
  https://raw.githubusercontent.com/clawvader-tech/hermes-telegram-miniapp/main/index.html
```

That's it. One HTML file, ~75KB, no build step.

---

## Step 2 — Create a Telegram Bot

1. Open [@BotFather](https://t.me/BotFather)
2. Send `/newbot`, pick a name and username
3. Save the bot token (looks like `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)

Get your numeric user ID from [@userinfobot](https://t.me/userinfobot) — send `/start`, it replies with a number.

---

## Step 3 — Configure Environment Variables

Add these to `~/.hermes/.env`:

```bash
# Required
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_OWNER_ID=your_numeric_user_id
TELEGRAM_ALLOWED_USERS=your_numeric_user_id
API_SERVER_KEY=$(python3 -c "import secrets; print(secrets.token_urlsafe(32))")

# CRITICAL — without this, the mini app gets 403 on every request
API_SERVER_CORS_ORIGINS=*
```

**Common mistake:** `TELEGRAM_OWNER_ID` is a number, not your username.

Generate the API key:
```bash
echo "API_SERVER_KEY=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')"
```

---

## Step 4 — Optional: Install pynacl

The server can verify Telegram auth two ways:

1. Ed25519 verification with Telegram's public key (`pynacl` required)
2. HMAC verification with your bot token (fallback, no extra package)

Install `pynacl` if you want the Ed25519 path available:

```bash
pip install pynacl
```

If you skip it, the mini app can still work via the HMAC fallback as long as `TELEGRAM_BOT_TOKEN` is set correctly.

---

## Step 5 — Set Up Cloudflare Tunnel

Telegram requires HTTPS. The tunnel points to the **mini app proxy on port 8643**, not to Hermes directly.

```bash
# Create the tunnel
cloudflared tunnel create hermes

# Route your domain
cloudflared tunnel route dns hermes hs.yourdomain.com
```

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: hermes
credentials-file: ~/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: hs.yourdomain.com
    service: http://localhost:8643
  - service: http_status:404
```

> **Important:** The tunnel targets port **8643** (the proxy), not 8642 (Hermes). The proxy handles mini-app specific endpoints and passes everything else through to Hermes on 8642.

Start the tunnel:
```bash
cloudflared tunnel run hermes
```

**Quick test** (temporary random URL, no setup needed):
```bash
cloudflared tunnel --url http://localhost:8643
```

---

## Step 6 — Deploy the Standalone Mini App Proxy

Instead of patching Hermes core files (which get overwritten on every update), the mini app runs through a standalone proxy on port 8643. The proxy:

- Serves static files from `~/.hermes/miniapp/`
- Validates Telegram initData (Ed25519 + HMAC fallback)
- Translates Telegram auth to Bearer tokens for Hermes
- Provides endpoints the mini app needs (`/api/commands`, `/api/model-info`, `/api/processes`, etc.)
- Enriches `/health` with system metrics
- Passes all other requests through to Hermes on 8642

### 6A. Create the Proxy

The proxy is a single Python file using aiohttp. It uses the Hermes venv Python which already has all required dependencies.

```bash
mkdir -p ~/.hermes/miniapp-proxy
```

Create `~/.hermes/miniapp-proxy/proxy.py` — the full implementation (~350 lines). Key components:

1. **Telegram auth validation** — `verify_telegram_init_data()` parses initData URL params, builds the canonical data-check string, tries Ed25519 verification with Telegram's public key, falls back to HMAC-SHA256 with `TELEGRAM_BOT_TOKEN`, checks user ID against `TELEGRAM_ALLOWED_USERS`.

2. **Auth middleware** — On every request: check `X-Telegram-Init-Data` header, validate it, replace with `Authorization: Bearer <API_SERVER_KEY>` before proxying to Hermes. If no Telegram header, pass through original auth as-is.

3. **Static file serving** — `GET /miniapp/` and `GET /miniapp/{filename}` serve files from `~/.hermes/miniapp/` with correct MIME types.

4. **Local API endpoints:**

   | Endpoint | Method | Auth | Purpose |
   |----------|--------|------|---------|
   | `/api/commands` | GET | Yes | Slash command registry (imports from `hermes_cli.commands`) |
   | `/api/command` | POST | Yes | Execute a slash command |
   | `/api/model-info` | GET | Yes | Current model, provider, context length |
   | `/api/processes` | GET | Yes | Background process list from `process_registry` |
   | `/health` | GET | No | Health + CPU, memory, disk, uptime, load metrics |

5. **CORS handling** — Always emits full CORS headers including `PATCH` method, `X-Telegram-Init-Data`, and `X-Hermes-Session-Id`.

6. **Proxy fallback** — Any unmatched route forwards to `http://127.0.0.1:8642` with translated auth headers, streaming SSE responses back chunk-by-chunk (critical for `/v1/chat/completions`).

> See `miniapp-standalone-proxy-plan.md` in this guide directory for the full implementation plan, including the complete handler code.

### 6B. Install as a launchd Service

Create `~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.hermes.miniapp-proxy</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/<user>/.hermes/hermes-agent/venv/bin/python</string>
        <string>/Users/<user>/.hermes/miniapp-proxy/proxy.py</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/<user>/.hermes/miniapp-proxy</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/<user>/.hermes/logs/miniapp-proxy.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/<user>/.hermes/logs/miniapp-proxy.error.log</string>
</dict>
</plist>
```

Load it:
```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist
```

### 6C. Verify the Proxy

```bash
# 1. Proxy is listening
lsof -i :8643

# 2. Static files served
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8643/miniapp/
# Expected: 200

# 3. Health enrichment
curl -s http://127.0.0.1:8643/health | python3 -m json.tool
# Expected: has cpu_percent, memory_percent, disk_percent, uptime, load_avg

# 4. Commands endpoint
curl -s -H "Authorization: Bearer $API_SERVER_KEY" http://127.0.0.1:8643/api/commands
# Expected: JSON with commands list

# 5. Chat pass-through (streaming)
curl -s -N -H "Authorization: Bearer $API_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"ping"}],"stream":true}' \
  http://127.0.0.1:8643/v1/chat/completions
# Expected: SSE stream

# 6. Hermes gateway untouched
curl -s http://127.0.0.1:8642/miniapp/
# Expected: 404 (Hermes still doesn't serve miniapp -- proxy does)
```

---

## Step 7 — Configure BotFather

In [@BotFather](https://t.me/BotFather):

1. Send `/setmenubutton`
2. Pick your bot
3. Send your URL: `https://hs.yourdomain.com/miniapp/index.html`

Users tap the menu button in the chat to launch the mini app.

---

## Step 8 — Restart and Test

```bash
# Start the proxy (if not already running via launchd)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist

# Reload Cloudflare tunnel (now pointing to 8643)
launchctl bootout gui/$(id -u)/com.cloudflare.cloudflared 2>/dev/null || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist

# Test the tunnel
curl -s https://hs.yourdomain.com/health
# Should return: {"status":"ok","platform":"hermes-agent",...metrics...}

# Test the mini app
curl -s https://hs.yourdomain.com/miniapp/
# Should return the HTML page

# Test API commands
curl -s https://hs.yourdomain.com/api/commands
# Should return JSON list of commands
```

Then open Telegram, find your bot, tap the menu button. The mini app should load with Telegram Ed25519 auth.

## Post-Reboot Checklist

After a macOS reboot, this stack should come back automatically at login via launchd. The fastest sanity check is:

```bash
~/.hermes/scripts/check-miniapp-stack.sh
```

That script verifies:
- `ai.hermes.gateway` is loaded
- `ai.hermes.miniapp-proxy` is loaded
- `com.cloudflare.cloudflared` is loaded
- port `8642` is listening
- port `8643` is listening
- `https://hs.yourdomain.com/health` returns `200`

Expected result:
```bash
RESULT: ALL PASS
```

If it fails, inspect the relevant logs:

```bash
# Mini-app proxy logs
cat ~/.hermes/logs/miniapp-proxy.log
cat ~/.hermes/logs/miniapp-proxy.error.log

# Gateway logs
cat ~/.hermes/logs/gateway.log
cat ~/.hermes/logs/gateway.error.log
```

---

## Troubleshooting

### 403 Forbidden on Every Request

**Cause:** CORS middleware rejects requests when `API_SERVER_CORS_ORIGINS` is not set. The Telegram WebApp sends an `Origin` header, and without configured origins, the server blocks it.

**Fix:** Add to `.env`:
```bash
API_SERVER_CORS_ORIGINS=*
```

### 403 on OPTIONS Preflight

**Cause:** CORS `Access-Control-Allow-Headers` doesn't include `X-Telegram-Init-Data` and `X-Hermes-Session-Id`. Browser blocks the actual request after preflight.

**Fix:** The proxy sets full CORS headers including these. If seeing this, the proxy on 8643 may not be running — check `lsof -i :8643`.

### 404 on /miniapp/ or /api/* Endpoints

**Cause:** The proxy on port 8643 is not running, or the Cloudflare tunnel is pointing to Hermes (8642) instead of the proxy (8643).

**Fix:** Check that the proxy is running (`lsof -i :8643`), and verify `~/.cloudflared/config.yml` targets `http://localhost:8643`.

### 401 Unauthorized on Authenticated Endpoints

**Cause:** Mini app sends Telegram `initData` in `X-Telegram-Init-Data` header. The proxy should validate this and translate it to a Bearer token. If the proxy's Telegram verification fails, the request falls through to Bearer auth which won't have the right token.

**Fix:** Check the proxy logs (`~/.hermes/logs/miniapp-proxy.error.log`) for Telegram verification errors. Verify `TELEGRAM_BOT_TOKEN` is set correctly in `~/.hermes/.env`. The dual-auth path (Ed25519 first, HMAC fallback) must work.

### "Invalid API key" on Cron/Status Tab

**Cause:** The proxy's Telegram verification is failing, so auth falls through to Bearer token. This can happen if Ed25519 payload format is wrong (must be `<bot_id>:WebAppData\n<sorted key=value lines>`) or if `TELEGRAM_BOT_TOKEN` is wrong.

**Fix:** Check proxy logs. Verify the env vars. The HMAC fallback should catch most cases when Ed25519 fails.

### Session ID Shows `[object Object]`

**Cause:** Telegram CloudStorage can return values in shapes that are not plain strings. The mini app was reading them back and rendering the raw object directly.

**Fix:** Normalize CloudStorage values before use. Treat `[object Object]`, `null`, `undefined`, and object wrappers as invalid and create a fresh session ID instead.

### Uptime Shows `N/A`

**Cause:** The status dashboard expected `/health` to return uptime, but the backend only returned a minimal health response. On top of that, `psutil` was not installed in the gateway venv.

**Fix:** Enrich `/health` to return metrics and add a no-`psutil` fallback:
- macOS: derive boot time via `sysctl kern.boottime`
- Linux: read `/proc/uptime`

### Dashboard Data Missing or Half Blank

**Cause:** The status UI expected CPU, memory, disk, uptime, load average, and properly-shaped process objects. The backend was returning too little data or the wrong process format.

**Fix:**
- expand `/health`
- map `/api/processes` to the fields the frontend actually renders (`name`, `running`, `pid`, `status`, etc.)
- adjust the mini app CSS so long values like session IDs wrap sanely instead of wrecking the row layout

### Cloudflare Error 1033

**Cause:** The tunnel DNS record existed, but `cloudflared` was not running.

**Fix:** Start it with:
```bash
cloudflared tunnel run hermes
```

### Cloudflare launchd Service Installed but Broken

**Cause:** `cloudflared service install` created a LaunchAgent plist that only ran `cloudflared` with no `tunnel run` arguments. So it crash-looped forever.

**Fix:** Edit `~/Library/LaunchAgents/com.cloudflare.cloudflared.plist` to run:
```xml
/opt/homebrew/bin/cloudflared tunnel --config /Users/<user>/.cloudflared/config.yml run hermes
```
Then reload it with `launchctl`.

### Chat Works but Commands Return "Not Available"

**Cause:** The command dispatch only handles a subset (help, status, model, provider, new, profile). Commands like `/cron` are not wired up for API dispatch.

**Fix:** Extend `_dispatch_api_command()` to handle more commands, or send them as chat messages instead.

---

## Architecture Overview

```
Telegram Client
    |
    v
Telegram WebApp (mini app HTML/JS in Telegram's browser)
    |
    |-- X-Telegram-Init-Data header (Ed25519 signed)
    |-- Authorization: Bearer *** (fallback)
    |
    v
Cloudflare Tunnel (HTTPS -> localhost:8643)
    |
    v
Mini App Proxy (port 8643)
    |
    |-- CORS headers (full: PATCH, Telegram headers)
    |-- Auth translation (Telegram initData -> Bearer token)
    |-- /miniapp/*          -> Static file serving
    |-- /api/commands       -> Slash command registry
    |-- /api/command        -> Command execution
    |-- /api/model-info     -> Model metadata
    |-- /api/processes      -> Process list
    |-- /health             -> Health + system metrics
    |
    v (pass-through for unmatched routes)
Hermes Gateway (port 8642, UNMODIFIED STOCK)
    |
    |-- /v1/chat/completions -> Streaming chat
    |-- /api/jobs/*          -> Cron management
    |-- /v1/models           -> Model list
    |-- /health              -> Basic health (stock)
```

### Legacy Architecture (Deprecated — Do Not Use)

The old approach patched `gateway/platforms/api_server.py` with custom routes, CORS headers, and Telegram auth logic. This was fragile because **Hermes updates overwrite the file**, causing 404s on all mini-app endpoints. The standalone proxy architecture replaced this entirely. Do not modify `api_server.py` for mini-app support.

## Auth Flow

```
1. User opens mini app in Telegram
2. Telegram SDK generates signed initData
3. Mini app sends initData in X-Telegram-Init-Data header
4. Proxy tries Ed25519 verification using Telegram's public key
5. If Ed25519 is unavailable or fails, proxy falls back to HMAC verification using TELEGRAM_BOT_TOKEN
6. Proxy extracts user ID, checks against TELEGRAM_ALLOWED_USERS
7. If valid -> proxy replaces auth header with Bearer <API_SERVER_KEY> and forwards to Hermes
8. If Telegram auth fails entirely -> proxy passes through original Bearer token
9. If neither works -> 401
```

---

## Files Summary

| File | Action | Description |
|------|--------|-------------|
| `~/.hermes/miniapp-proxy/proxy.py` | CREATE | Standalone proxy server (~350 lines) |
| `~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist` | CREATE | launchd service for proxy on port 8643 |
| `~/.cloudflared/config.yml` | MODIFY | Tunnel target changed from 8642 to 8643 |
| `~/.hermes/miniapp/index.html` | CREATE | Mini app frontend (~2084 lines, single file) |
| `~/.hermes/.env` | MODIFY | Telegram env vars, API_SERVER_KEY, CORS origins |
| `gateway/platforms/api_server.py` | NO CHANGE | Stock Hermes left untouched |

---

## Dashboard Scorecard & Operational Monitoring

The Hermes Scorecard is a self-assessment system that measures operational health across 10 dimensions on a 0–5 scale, run weekly.

### What It Measures

| # | Dimension | What It Tracks |
|---|-----------|---------------|
| 1 | Task Completion | Did the agent finish what was asked? |
| 2 | First-Attempt Accuracy | Did it work right the first time, or require rework? |
| 3 | Context Retention | Did the agent hold context across a session? |
| 4 | Skill Health | Are skills loading, running, and passing checks? |
| 5 | Cost Efficiency | Are we spending wisely per task? |
| 6 | Routing Accuracy | Is the right model being used for the right task? |
| 7 | Error Recovery | When something breaks, does the agent recover gracefully? |
| 8 | Proactive Value | Did the agent anticipate needs or just wait for instructions? |
| 9 | Communication | Are responses clear, honest, and appropriately concise? |
| 10 | Process Discipline | Is the agent following its own rules, PRDs, and handoff protocols? |

Each dimension is scored 0–5, giving a maximum of 50. The first score came in at 13/50 — honest low scores, especially where data was missing. That baseline is the point: you can't improve what you don't measure.

### Data Collection Files

| File | Purpose |
|------|---------|
| `~/.hermes/skill-health.log` | Skill load/run status and health check results |
| `~/.hermes/cost-log.csv` | Per-task cost tracking |
| `~/.hermes/daily-logs/failures.md` | Documented failures and what went wrong |

### Scorecard History

Scorecard results are stored in:

```
~/.hermes/scorecards/
```

Each run produces a timestamped file so you can track trends over time.

### Scoring Script & Skill Definition

- **Scoring script:** `~/.hermes/skills/hermes-scorecard/scripts/score.py`
- **Skill definition:** `~/.hermes/skills/hermes-scorecard/SKILL.md`

### Weekly Cron Job

The scorecard runs automatically every Monday at 10am via cron. This ensures consistent measurement without relying on manual discipline.

### How to Run Manually

```bash
python3 ~/.hermes/skills/hermes-scorecard/scripts/score.py
```

Results are printed to stdout and saved to `~/.hermes/scorecards/`.

### Philosophy

The scorecard exists because self-congratulation is a trap. The first score of 13/50 was a signal that most operational data wasn't being collected yet — not a failure, but a starting point. The goal is to watch the number move, not to celebrate early.

---

## Operational Scripts & Weekly Cron Jobs

Beyond the scorecard itself, two automated scripts run on a weekly schedule to keep the system healthy.

### Signal-to-Skill Pipeline

**Script:** `~/.hermes/scripts/signal-to-skill.py`
**Cron:** Every Monday at 11am (runs 1 hour after the scorecard)

This script scans operational signals (failure logs, skill health output, cost data) and converts them into actionable skill improvement suggestions. It's the automation behind the "collect → score → act" loop described in the Lessons Learned Signal-to-Skill Pipeline section.

### Project Burial Scanner

**Script:** `~/.hermes/scripts/project-burial-scan.py`
**Cron:** Every Monday at 11:30am (runs 30 minutes after signal-to-skill)

Flags projects that haven't been touched in 7+ days for archival. It doesn't delete anything — it surfaces stale projects so they can be moved out of the active workspace or consciously picked back up.

### Cron Schedule Summary

| Time | Job | Script |
|------|-----|--------|
| Monday 10:00 | Hermes Scorecard | `~/.hermes/skills/hermes-scorecard/scripts/score.py` |
| Monday 11:00 | Signal-to-Skill Pipeline | `~/.hermes/scripts/signal-to-skill.py` |
| Monday 11:30 | Project Burial Scanner | `~/.hermes/scripts/project-burial-scan.py` |
