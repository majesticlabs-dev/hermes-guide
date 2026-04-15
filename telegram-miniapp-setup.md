# Hermes Telegram Mini App — Setup Guide

A complete walkthrough for deploying the Hermes Telegram Mini App. This guide covers everything from scratch, including every error we hit and how we fixed it.

## What You'll End Up With

- A terminal-style web interface inside Telegram
- Streaming chat with your Hermes agent
- Cron job management
- System monitoring
- Ed25519 authentication (no API key exposed to clients)

---

## Prerequisites

- Hermes Agent v0.8.0+ running as a gateway
- A Telegram bot (created via @BotFather)
- `cloudflared` CLI installed and authenticated
- A domain/subdomain you control (for Cloudflare Tunnel)
- Python package `pynacl` for Ed25519 verification

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

The Hermes gateway runs on `localhost:8642`. Telegram requires HTTPS.

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
    service: http://localhost:8642
  - service: http_status:404
```

Start the tunnel:
```bash
cloudflared tunnel run hermes
```

**Quick test** (temporary random URL, no setup needed):
```bash
cloudflared tunnel --url http://localhost:8642
```

---

## Step 6 — Apply Gateway Code Changes

The stock Hermes API server needs modifications to support the mini app. All changes go in `gateway/platforms/api_server.py`.

### 6A. Update CORS Headers (~line 235)

The mini app sends custom headers that the default CORS config doesn't allow:

```python
_CORS_HEADERS = {
    "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
    "Access-Control-Allow-Headers": "Authorization, Content-Type, Idempotency-Key, X-Telegram-Init-Data, X-Hermes-Session-Id",
}
```

**What changed:**
- Added `PATCH` to allowed methods
- Added `X-Telegram-Init-Data` (Telegram auth payload)
- Added `X-Hermes-Session-Id` (session continuity)

### 6B. Add Telegram initData Authentication (~line 467)

The server's `_check_auth()` method only supported Bearer tokens. The mini app authenticates via Telegram's Ed25519-signed `initData`. We need a dual auth flow.

Modified `_check_auth()`:
```python
def _check_auth(self, request):
    # 1. Try Telegram initData validation first
    init_data = request.headers.get("X-Telegram-Init-Data", "")
    if init_data:
        user_id = self._verify_telegram_init_data(init_data)
        if user_id is not None:
            return None  # Telegram auth OK

    # 2. Fall back to Bearer token (existing behavior)
    if not self._api_key:
        return None
    auth_header = request.headers.get("Authorization", "")
    if auth_header.startswith("Bearer "):
        token = auth_header[7:].strip()
        if hmac.compare_digest(token, self._api_key):
            return None  # Bearer auth OK

    return web.json_response(
        {"error": {"message": "Invalid API key", "type": "invalid_request_error", "code": "invalid_api_key"}},
        status=401,
    )
```

New method `_verify_telegram_init_data()`:
- Parses initData URL parameters
- Builds the signed payload in Telegram's required format
- Tries Ed25519 verification with Telegram's published public key
- Falls back to classic HMAC-SHA256 verification with `TELEGRAM_BOT_TOKEN`
- Extracts user ID and checks against `TELEGRAM_OWNER_ID` / `TELEGRAM_ALLOWED_USERS`
- Returns user ID on success, None on failure

```python
def _verify_telegram_init_data(self, init_data):
    # Parse all query parameters first
    params = dict(parse_qsl(init_data, keep_blank_values=True))
    user_json = params.get("user", "")
    if not user_json:
        return None

    # Build Telegram's canonical data string
    data_parts = []
    for k, v in sorted(params.items()):
        if k not in ("hash", "signature"):
            data_parts.append(f"{k}={v}")
    data_check_string = "\n".join(data_parts)

    bot_token = os.getenv("TELEGRAM_BOT_TOKEN", "")
    bot_id = bot_token.split(":", 1)[0].strip()
    if not bot_id or not bot_token:
        return None

    validated = False

    # 1. Try Telegram third-party Ed25519 verification
    signature_b64 = params.get("signature", "")
    if signature_b64:
        try:
            verify_key = VerifyKey(bytes.fromhex(
                "e7bf03a2fa4602af4580703d88dda5bb59f32ed8b02a56c187fe7d34caed242d"
            ))
            padding = "=" * (-len(signature_b64) % 4)
            signature = base64.urlsafe_b64decode(signature_b64 + padding)
            signed_payload = f"{bot_id}:WebAppData\n{data_check_string}".encode("utf-8")
            verify_key.verify(signed_payload, signature)
            validated = True
        except Exception:
            pass

    # 2. Fallback to HMAC with the bot token
    if not validated:
        received_hash = params.get("hash", "")
        if received_hash:
            secret_key = hmac.new(b"WebAppData", bot_token.encode("utf-8"), hashlib.sha256).digest()
            expected_hash = hmac.new(secret_key, data_check_string.encode("utf-8"), hashlib.sha256).hexdigest()
            validated = hmac.compare_digest(expected_hash, received_hash)

    if not validated:
        return None

    user = json.loads(unquote(user_json))
    user_id = int(user.get("id", 0))
    allowed_ids = {uid.strip() for uid in os.getenv("TELEGRAM_ALLOWED_USERS", "").split(",") if uid.strip()}
    owner = os.getenv("TELEGRAM_OWNER_ID", "")
    if owner:
        allowed_ids.add(owner)
    return user_id if str(user_id) in allowed_ids else None
```

### 6C. Add Static File Serving

The mini app HTML needs to be served from `~/.hermes/miniapp/`. The gateway has no built-in static file serving.

```python
async def _handle_miniapp(self, request):
    filename = request.match_info.get("filename", "index.html")
    try:
        from hermes_constants import get_hermes_home
        miniapp_dir = get_hermes_home() / "miniapp"
    except ImportError:
        miniapp_dir = os.path.expanduser("~/.hermes/miniapp")

    file_path = os.path.join(miniapp_dir, filename)
    if not os.path.isfile(file_path):
        return web.Response(status=404, text="Not Found")

    content_types = {
        ".html": "text/html", ".js": "application/javascript",
        ".css": "text/css", ".json": "application/json",
        ".png": "image/png", ".jpg": "image/jpeg",
        ".svg": "image/svg+xml", ".ico": "image/x-icon",
    }
    ext = os.path.splitext(filename)[1].lower()
    ct = content_types.get(ext, "application/octet-stream")
    with open(file_path, "rb") as f:
        data = f.read()
    return web.Response(body=data, content_type=ct)

async def _handle_miniapp_index(self, request):
    return await self._handle_miniapp(
        type("Req", (), {"match_info": {"filename": "index.html"}})()
    )
```

### 6D. Add Missing API Endpoints

The mini app expects endpoints that don't exist in the stock gateway:

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/commands` | GET | No | List available slash commands |
| `/api/command` | POST | Yes | Execute a slash command |
| `/api/model-info` | GET | Yes | Current model, provider, context length |
| `/api/processes` | GET | Yes | Running background processes |

**GET /api/commands** — Returns the command registry:
```python
async def _handle_api_commands(self, request):
    from hermes_cli.commands import COMMAND_REGISTRY
    commands = []
    for cmd in COMMAND_REGISTRY:
        if cmd.gateway_only:
            continue
        commands.append({
            "name": cmd.name,
            "description": cmd.description,
            "category": cmd.category,
            "aliases": list(cmd.aliases),
        })
    return web.json_response({"commands": commands})
```

**POST /api/command** — Executes commands like `/model`, `/status`:
```python
async def _handle_api_command(self, request):
    auth_err = self._check_auth(request)
    if auth_err:
        return auth_err
    body = await request.json()
    command = body.get("command", "").strip()
    # Dispatch to command handler...
    return web.json_response({"output": result})
```

**GET /api/model-info** — Model metadata:
```python
async def _handle_api_model_info(self, request):
    auth_err = self._check_auth(request)
    if auth_err:
        return auth_err
    return web.json_response({
        "model": self._model_name,
        "provider": provider,
        "context_length": context_length,
    })
```

**GET /api/processes** — Background processes mapped to the shape the mini app expects:
```python
async def _handle_api_processes(self, request):
    auth_err = self._check_auth(request)
    if auth_err:
        return auth_err

    from tools.process_registry import process_registry
    processes = []
    for proc in process_registry.list_sessions():
        command = str(proc.get("command", "") or "")
        display_name = command.split()[0] if command else (proc.get("pid") or proc.get("session_id") or "?")
        processes.append({
            "id": proc.get("session_id", ""),
            "name": os.path.basename(display_name),
            "command": command,
            "pid": proc.get("pid"),
            "running": proc.get("status") == "running",
            "status": proc.get("status", "unknown"),
            "started_at": proc.get("started_at", ""),
            "uptime_seconds": proc.get("uptime_seconds"),
            "output_preview": proc.get("output_preview", ""),
            "cpu": None,
            "mem": None,
        })

    return web.json_response({"processes": processes})
```

### 6E. Enrich `/health` for the Status Dashboard

The mini app status panel expects more than a plain `{status, platform}` payload. It wants CPU, memory, disk, uptime, and load average.

```python
async def _handle_health(self, request):
    payload = {"status": "ok", "platform": "hermes-agent"}
    try:
        import psutil
        payload["cpu_percent"] = round(psutil.cpu_percent(interval=None), 1)
        payload["memory_percent"] = round(psutil.virtual_memory().percent, 1)
        payload["disk_percent"] = round(psutil.disk_usage("/").percent, 1)
        payload["uptime"] = max(0, int(time.time() - psutil.boot_time()))
    except Exception:
        # Fallback when psutil is not installed
        import shutil
        du = shutil.disk_usage("/")
        payload["disk_percent"] = round(((du.total - du.free) / du.total) * 100, 1)
        if sys.platform == "darwin":
            out = subprocess.check_output(["sysctl", "-n", "kern.boottime"], text=True, timeout=2)
            m = re.search(r"sec\s*=\s*(\d+)", out)
            if m:
                payload["uptime"] = max(0, int(time.time()) - int(m.group(1)))
        elif os.path.exists("/proc/uptime"):
            with open("/proc/uptime") as f:
                payload["uptime"] = int(float(f.read().split()[0]))

    load1, load5, load15 = os.getloadavg()
    payload["load_avg"] = [round(load1, 2), round(load5, 2), round(load15, 2)]
    return web.json_response(payload)
```

This matters because otherwise the dashboard shows nonsense like `Uptime N/A` even when the gateway is perfectly alive.

### 6F. Register All New Routes

In the `connect()` method, add:

```python
# Mini app API endpoints
self._app.router.add_get("/api/commands", self._handle_api_commands)
self._app.router.add_post("/api/command", self._handle_api_command)
self._app.router.add_get("/api/model-info", self._handle_api_model_info)
self._app.router.add_get("/api/processes", self._handle_api_processes)
# Mini app static files
self._app.router.add_get("/miniapp/{filename}", self._handle_miniapp)
self._app.router.add_get("/miniapp/", self._handle_miniapp_index)
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
# Restart the gateway
hermes gateway restart

# Test the tunnel
curl -s https://hs.yourdomain.com/health
# Should return: {"status":"ok","platform":"hermes-agent"}

# Test the mini app
curl -s https://hs.yourdomain.com/miniapp/
# Should return the HTML page

# Test API commands
curl -s https://hs.yourdomain.com/api/commands
# Should return JSON list of commands
```

Then open Telegram, find your bot, tap the menu button. The mini app should load with Telegram Ed25519 auth.

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

**Fix:** Update `_CORS_HEADERS` in `api_server.py` (see Step 6A).

### 404 on /miniapp/ or /api/* Endpoints

**Cause:** Routes not registered in the aiohttp router. The stock gateway doesn't serve static files or have the API endpoints the mini app expects.

**Fix:** Add route registrations and handler methods (see Steps 6C–6F).

### 401 Unauthorized on Authenticated Endpoints

**Cause:** Mini app sends Telegram `initData` in `X-Telegram-Init-Data` header, but `_check_auth()` originally only looked for Bearer tokens.

**Fix:** Add `_verify_telegram_init_data()` and update `_check_auth()` to support Telegram auth plus Bearer fallback (see Step 6B).

### "Invalid API key" on Cron/Status Tab

**Cause:** The mini app sends Telegram `initData`, but the original auth code only understood Bearer tokens. We also initially implemented Telegram verification wrong — wrong public key format / wrong signed payload assumptions — so signature validation silently failed and the request fell through to Bearer auth.

**Fix:** Use a dual-auth path:
- try Telegram Ed25519 verification first using the proper signed payload
- if that fails, fall back to HMAC verification using `TELEGRAM_BOT_TOKEN`

That fallback saved the day when `pynacl` or the Ed25519 path was missing/broken.

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
    │
    ▼
Telegram WebApp (mini app HTML/JS in Telegram's browser)
    │
    ├── X-Telegram-Init-Data header (Ed25519 signed)
    ├── Authorization: Bearer <token> (fallback)
    │
    ▼
Cloudflare Tunnel (HTTPS → localhost:8642)
    │
    ▼
Hermes Gateway API Server (aiohttp)
    │
    ├── CORS middleware (validates Origin)
    ├── Auth middleware (Telegram initData OR Bearer token)
    │
    ├── /miniapp/*          → Static file serving
    ├── /api/commands       → Slash command registry
    ├── /api/command        → Command execution
    ├── /api/model-info     → Model metadata
    ├── /api/processes      → Process list
    ├── /api/jobs/*         → Cron management (built-in)
    ├── /v1/chat/completions → Streaming chat (built-in)
    └── /health             → Health check (built-in)
```

## Auth Flow

```
1. User opens mini app in Telegram
2. Telegram SDK generates signed initData
3. Mini app sends initData in X-Telegram-Init-Data header
4. Server tries Ed25519 verification using Telegram's public key
5. If Ed25519 is unavailable or fails, server falls back to HMAC verification using TELEGRAM_BOT_TOKEN
6. Server extracts user ID, checks against TELEGRAM_ALLOWED_USERS
7. If valid → request proceeds
8. If Telegram auth fails entirely → falls back to Bearer token auth
9. If neither works → 401
```

---

## Files Modified Summary

| File | Change |
|------|--------|
| `gateway/platforms/api_server.py` | CORS headers, Telegram dual auth (Ed25519 + HMAC fallback), static serving, mini app API endpoints, enriched `/health`, route registration |
| `~/.hermes/.env` | Added `API_SERVER_CORS_ORIGINS=*`, `API_SERVER_KEY`, `TELEGRAM_OWNER_ID`, `TELEGRAM_ALLOWED_USERS` |
| `~/.cloudflared/config.yml` | Tunnel routing to localhost:8642 |
| `~/Library/LaunchAgents/com.cloudflare.cloudflared.plist` | Fixed macOS LaunchAgent to run `cloudflared tunnel --config ... run hermes` instead of bare `cloudflared` |
| `~/.hermes/miniapp/index.html` | Session ID normalization, improved dashboard rendering for long values, status panel compatibility fixes |

## Files Created

| File | Description |
|------|-------------|
| `~/.hermes/miniapp/index.html` | Mini app frontend (~2084 lines, single file) |

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
