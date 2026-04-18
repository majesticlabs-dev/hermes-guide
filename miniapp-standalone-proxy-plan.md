# Mini App Standalone Proxy — Implementation Plan

## Problem

Hermes updates overwrite `gateway/platforms/api_server.py`, wiping out custom mini-app routes (static serving, Telegram auth, `/api/commands`, `/api/model-info`, etc.). The current gateway on 8642 returns 404 for every mini-app endpoint except `/health` and the built-in `/v1/*` and `/api/jobs/*` routes.

## Architecture

```
Telegram Client
    |
    v
Telegram WebApp (index.html in Telegram's embedded browser)
    |
    |  X-Telegram-Init-Data header (Ed25519 signed)
    |  X-Hermes-Session-Id header
    |  Authorization: Bearer *** (fallback for browser access)
    |
    v
Cloudflare Tunnel (hs.majesticlabs.dev -- HTTPS)
    |
    v
Mini App Proxy (localhost:8643)          <-- NEW SERVICE
    |
    |  Validates Telegram initData (Ed25519 + HMAC)
    |  Translates to Bearer token for Hermes
    |  Serves static files from ~/.hermes/miniapp/
    |  Provides missing endpoints locally
    |  Enriches /health with system metrics
    |
    v
Hermes Gateway (localhost:8642)          <-- UNMODIFIED STOCK
    |
    |  /v1/chat/completions  (streaming chat)
    |  /api/jobs/*           (cron CRUD)
    |  /v1/models            (model list)
    |  /health               (basic health)
```

The proxy sits between Cloudflare and the stock Hermes gateway. Hermes stays completely unpatched and can be updated freely.

---

## Frontend Endpoint Inventory

The mini app (`~/.hermes/miniapp/index.html`, line 825: `const API = window.location.origin`) calls these endpoints relative to its own origin:

### Endpoints the proxy handles LOCALLY (not in stock Hermes)

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/miniapp/{filename}` | GET | No | Static file serving (index.html) |
| `/miniapp/` | GET | No | Redirect to index.html |
| `/api/commands` | GET | Yes | Slash command registry |
| `/api/command` | POST | Yes | Execute a slash command |
| `/api/model-info` | GET | Yes | Current model, provider, context length |
| `/api/processes` | GET | Yes | Background process list |
| `/health` (enriched) | GET | No | Health + CPU, memory, disk, uptime, load |

### Endpoints the proxy PASS-THROUGHS to Hermes 8642

| Endpoint | Method | Auth | Handler |
|----------|--------|------|---------|
| `/v1/chat/completions` | POST | Yes | SSE streaming chat (stock) |
| `/v1/models` | GET | Yes | Model list (stock) |
| `/api/jobs` | GET, POST | Yes | Cron job list/create (stock) |
| `/api/jobs/{id}` | GET, PATCH, DELETE | Yes | Cron job CRUD (stock) |
| `/api/jobs/{id}/run` | POST | Yes | Trigger job (stock) |
| `/api/jobs/{id}/pause` | POST | Yes | Pause job (stock) |
| `/api/jobs/{id}/resume` | POST | Yes | Resume job (stock) |
| `/v1/responses*` | various | Yes | Responses API (stock) |
| `/v1/runs*` | various | Yes | Run streaming (stock) |
| `/health/detailed` | GET | No | Rich status (stock) |

**Note on PATCH**: The stock CORS config only allows `GET, POST, DELETE, OPTIONS` and does NOT list `PATCH`. The proxy must add PATCH to CORS Allow-Methods for `/api/jobs` PATCH calls to work.

---

## Auth Strategy

The frontend sends auth two ways:

1. **Telegram mode**: `X-Telegram-Init-Data` header with Ed25519-signed initData
2. **Browser mode**: `Authorization: Bearer <token>` (user enters API key manually)

The proxy's auth pipeline:

```
1. Read X-Telegram-Init-Data header
2. If present:
   a. Parse query parameters from initData
   b. Build canonical data-check string (sorted key=value pairs, newline-separated)
   c. Try Ed25519 verification with Telegram's public key
      - Signed payload: "<bot_id>:WebAppData\n<data_check_string>"
      - Public key: e7bf03a2fa4602af4580703d88dda5bb59f32ed8b02a56c187fe7d34caed242d
   d. If Ed25519 fails, fall back to HMAC-SHA256 with TELEGRAM_BOT_TOKEN
      - secret_key = HMAC-SHA256("WebAppData", bot_token)
      - expected_hash = HMAC-SHA256(secret_key, data_check_string)
   e. Extract user ID, check against TELEGRAM_ALLOWED_USERS / TELEGRAM_OWNER_ID
   f. If valid: replace auth header with Bearer <API_SERVER_KEY> before proxying
3. If no Telegram initData:
   a. Pass through the original Authorization: Bearer header as-is
   b. Hermes validates it with its own _check_auth()
```

This way Hermes always sees a standard Bearer token. The Telegram auth complexity lives entirely in the proxy.

---

## Files to Create

### 1. `~/.hermes/miniapp-proxy/proxy.py` (~350 lines)

The standalone proxy server. Single-file Python using aiohttp.

**Structure:**

```python
#!/usr/bin/env python3
"""
Hermes Mini App Proxy — standalone service on port 8643.

Serves the mini app UI, validates Telegram auth, enriches /health,
provides missing API endpoints, and proxies everything else to the
stock Hermes gateway on 8642.

No modifications to Hermes core required.
"""

import asyncio
import base64
import hashlib
import hmac
import json
import logging
import os
import re
import shutil
import subprocess
import sys
import time
from pathlib import Path
from urllib.parse import parse_qsl, unquote

import aiohttp
from aiohttp import web

# --- Configuration ---
HERMES_URL = "http://127.0.0.1:8642"
PROXY_HOST = "127.0.0.1"
PROXY_PORT = 8643
MINIAPP_DIR = Path.home() / ".hermes" / "miniapp"
ENV_FILE = Path.home() / ".hermes" / ".env"

# Telegram Ed25519 public key (hex)
TELEGRAM_PUBLIC_KEY_HEX = (
    "e7bf03a2fa4602af4580703d88dda5bb59f32ed8b02a56c187fe7d34caed242d"
)

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [miniapp-proxy] %(levelname)s %(message)s",
)
logger = logging.getLogger(__name__)


def load_env():
    """Load key=value pairs from .env into os.environ (don't overwrite existing)."""
    if not ENV_FILE.exists():
        return
    with open(ENV_FILE) as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith("#") or "=" not in line:
                continue
            key, _, value = line.partition("=")
            key = key.strip()
            value = value.strip().strip('"').strip("'")
            if key and key not in os.environ:
                os.environ[key] = value


load_env()

BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "")
ALLOWED_USERS = {
    uid.strip()
    for uid in os.getenv("TELEGRAM_ALLOWED_USERS", "").split(",")
    if uid.strip()
}
OWNER_ID = os.getenv("TELEGRAM_OWNER_ID", "")
if OWNER_ID:
    ALLOWED_USERS.add(OWNER_ID)
API_KEY = os.getenv("API_SERVER_KEY", "")


# --- Telegram Auth ---

def verify_telegram_init_data(init_data: str) -> int | None:
    """Verify Telegram WebApp initData. Returns user ID or None."""
    params = dict(parse_qsl(init_data, keep_blank_values=True))
    user_json = params.get("user", "")
    if not user_json:
        return None

    # Build canonical data string
    data_parts = []
    for k, v in sorted(params.items()):
        if k not in ("hash", "signature"):
            data_parts.append(f"{k}={v}")
    data_check_string = "\n".join(data_parts)

    bot_id = BOT_TOKEN.split(":", 1)[0].strip() if BOT_TOKEN else ""
    if not bot_id or not BOT_TOKEN:
        return None

    validated = False

    # Ed25519 path
    signature_b64 = params.get("signature", "")
    if signature_b64:
        try:
            from nacl.signing import VerifyKey
            verify_key = VerifyKey(bytes.fromhex(TELEGRAM_PUBLIC_KEY_HEX))
            padding = "=" * (-len(signature_b64) % 4)
            signature = base64.urlsafe_b64decode(signature_b64 + padding)
            signed_payload = f"{bot_id}:WebAppData\n{data_check_string}".encode()
            verify_key.verify(signed_payload, signature)
            validated = True
        except Exception:
            pass

    # HMAC fallback
    if not validated:
        received_hash = params.get("hash", "")
        if received_hash:
            secret_key = hmac.new(
                b"WebAppData", BOT_TOKEN.encode(), hashlib.sha256
            ).digest()
            expected_hash = hmac.new(
                secret_key, data_check_string.encode(), hashlib.sha256
            ).hexdigest()
            validated = hmac.compare_digest(expected_hash, received_hash)

    if not validated:
        return None

    user = json.loads(unquote(user_json))
    user_id = int(user.get("id", 0))
    return user_id if str(user_id) in ALLOWED_USERS else None


# --- Auth Middleware ---

async def auth_middleware(app, handler):
    """Validate Telegram initData and translate to Bearer token for Hermes."""
    request = web.Request  # actual request passed by aiohttp
    # ... (full implementation in the actual file)
```

**Key handler functions to implement:**

| Function | Route | What It Does |
|----------|-------|-------------|
| `handle_static()` | `GET /miniapp/{filename}` | Serve files from `~/.hermes/miniapp/` with correct MIME types |
| `handle_static_index()` | `GET /miniapp/` | Serve index.html |
| `handle_root_redirect()` | `GET /` | 302 redirect to `/miniapp/` |
| `handle_health_enriched()` | `GET /health` | Proxy to Hermes, then enrich with psutil/shell metrics |
| `handle_api_commands()` | `GET /api/commands` | Import Hermes command registry, return JSON |
| `handle_api_command()` | `POST /api/command` | Dispatch slash command, return output |
| `handle_api_model_info()` | `GET /api/model-info` | Return current model/provider/context_length |
| `handle_api_processes()` | `GET /api/processes` | List background processes from process_registry |
| `handle_proxy()` | `* /**` | Pass-through to Hermes with translated auth |

**Auth middleware flow (applied to all requests):**
1. Check for `X-Telegram-Init-Data` header
2. If present, validate with `verify_telegram_init_data()`
3. If valid, strip the Telegram header and inject `Authorization: Bearer <API_SERVER_KEY>`
4. If invalid, return 401
5. If no Telegram header, pass through original auth as-is
6. Always add CORS headers including `PATCH`, `X-Telegram-Init-Data`, `X-Hermes-Session-Id`

### 2. `~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist`

launchd service definition to run the proxy on port 8643 at login.

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
        <string>/Users/clawbot/.hermes/hermes-agent/venv/bin/python</string>
        <string>/Users/clawbot/.hermes/miniapp-proxy/proxy.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/clawbot/.hermes/miniapp-proxy</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/clawbot/.hermes/hermes-agent/venv/bin:/opt/homebrew/bin:/usr/bin:/bin</string>
        <key>HERMES_HOME</key>
        <string>/Users/clawbot/.hermes</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/clawbot/.hermes/logs/miniapp-proxy.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/clawbot/.hermes/logs/miniapp-proxy.error.log</string>
</dict>
</plist>
```

Uses the Hermes venv Python (`/Users/clawbot/.hermes/hermes-agent/venv/bin/python`) which already has aiohttp 3.13.5 and pynacl installed. No extra pip installs needed.

---

## Files to Modify

### 3. `~/.cloudflared/config.yml`

Change the tunnel target from 8642 (Hermes direct) to 8643 (proxy).

**Before:**
```yaml
tunnel: hermes
credentials-file: /Users/clawbot/.cloudflared/353d6632-0c7a-437d-8ebf-9df2c8750d7d.json

ingress:
  - hostname: hs.majesticlabs.dev
    service: http://localhost:8642
  - service: http_status:404
```

**After:**
```yaml
tunnel: hermes
credentials-file: /Users/clawbot/.cloudflared/353d6632-0c7a-437d-8ebf-9df2c8750d7d.json

ingress:
  - hostname: hs.majesticlabs.dev
    service: http://localhost:8643
  - service: http_status:404
```

One line change: `8642` → `8643`.

---

## Execution Order

### Task 1: Create the proxy directory

```bash
mkdir -p ~/.hermes/miniapp-proxy
mkdir -p ~/.hermes/logs
```

### Task 2: Write `~/.hermes/miniapp-proxy/proxy.py`

Create the full proxy server. Key implementation details:

- **Imports**: aiohttp, aiohttp.web, standard library + nacl (from Hermes venv)
- **Config**: Read from `.env` at startup. HERMES_URL=http://127.0.0.1:8642, listen on 127.0.0.1:8643
- **CORS**: Always emit `Access-Control-Allow-Origin: *`, `Allow-Methods: GET, POST, DELETE, PATCH, OPTIONS`, `Allow-Headers: Authorization, Content-Type, Idempotency-Key, X-Telegram-Init-Data, X-Hermes-Session-Id`
- **Static serving**: Map `/miniapp/` and `/miniapp/{filename}` to `~/.hermes/miniapp/`. Also serve `/` → redirect to `/miniapp/`
- **Health enrichment**: Forward `/health` to Hermes, then patch the response JSON with cpu_percent, memory_percent, disk_percent, uptime, load_avg (same logic from the old setup guide)
- **`/api/commands`**: Import `hermes_cli.commands.COMMAND_REGISTRY` (add `~/.hermes/hermes-agent` to sys.path), serialize CommandDef objects to JSON
- **`/api/command`**: Parse command name from POST body, dispatch to the gateway command handler
- **`/api/model-info`**: Read model info from Hermes gateway state or proxy to `/v1/models` and extract
- **`/api/processes`**: Import `tools.process_registry.process_registry` (add to sys.path), list sessions
- **Proxy fallback**: For any unmatched route, use `aiohttp.ClientSession` to forward the request to `http://127.0.0.1:8642{path}` with translated auth headers, streaming the response back (critical for SSE on `/v1/chat/completions`)
- **Graceful shutdown**: Handle SIGTERM, close ClientSession

### Task 3: Install the launchd plist

```bash
launchctl load ~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist
```

### Task 4: Update Cloudflare tunnel config

Edit `~/.cloudflared/config.yml`: change port 8642 → 8643, then reload:

```bash
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl load ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
```

### Task 5: Verify

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

# 5. Chat completions pass-through (streaming)
curl -s -N -H "Authorization: Bearer $API_SERVER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"hermes-agent","messages":[{"role":"user","content":"ping"}],"stream":true}' \
  http://127.0.0.1:8643/v1/chat/completions
# Expected: SSE stream

# 6. Jobs pass-through
curl -s -H "Authorization: Bearer $API_SERVER_KEY" http://127.0.0.1:8643/api/jobs
# Expected: JSON (may be empty list)

# 7. Through tunnel
curl -s https://hs.majesticlabs.dev/miniapp/ | head -5
# Expected: HTML head of index.html

# 8. Hermes gateway untouched
curl -s http://127.0.0.1:8642/health
# Expected: {"status":"ok","platform":"hermes-agent"}
curl -s http://127.0.0.1:8642/miniapp/
# Expected: 404 (Hermes still doesn't serve miniapp — proxy does)
```

### Task 6: Test in Telegram

Open the bot in Telegram, tap the menu button. The mini app should load, authenticate via Telegram initData, and work.

---

## Rollback Steps

If something goes wrong, restore the original state:

```bash
# 1. Point tunnel back to Hermes directly
#    Edit ~/.cloudflared/config.yml: change 8643 back to 8642

# 2. Reload Cloudflare
launchctl unload ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl load ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist

# 3. Stop the proxy
launchctl unload ~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist

# 4. Verify Hermes is still running
curl -s http://127.0.0.1:8642/health
```

No Hermes files were modified, so rollback is trivial.

---

## Key Design Decisions

### Why not patch api_server.py?

Hermes updates overwrite the file. We've confirmed this breaks the mini app (404s on all custom endpoints). The proxy is a separate process that Hermes updates cannot touch.

### Why port 8643 instead of replacing Hermes on 8642?

Hermes on 8642 still serves the workspace UI and other API clients directly. The proxy on 8643 only handles external traffic from the Cloudflare tunnel. Internal tools can still hit 8642 directly.

### Why use the Hermes venv Python?

The Hermes venv (`~/.hermes/hermes-agent/venv/bin/python`) already has aiohttp 3.13.5 and pynacl. No additional pip installs. The proxy can also import Hermes modules (hermes_cli.commands, tools.process_registry) directly.

### What about the `/api/commands` and `/api/model-info` imports?

The proxy adds `~/.hermes/hermes-agent` to `sys.path` so it can import:
- `hermes_cli.commands.COMMAND_REGISTRY` for the commands list
- `tools.process_registry.process_registry` for the processes list

These are stable internal APIs that rarely change between Hermes versions. If they do, only the proxy needs updating — not the core gateway.

### What about `/api/command` dispatch?

This is the trickiest endpoint. The proxy needs to execute slash commands (like `/model`, `/status`). Two approaches:

**Option A (recommended):** Import and call the command handler directly from Hermes modules. This is synchronous code running in the proxy's event loop — use `asyncio.to_thread()` to avoid blocking.

**Option B (simpler fallback):** Send the command as a chat message through `/v1/chat/completions` with a system hint. The agent processes it naturally. Less structured but always works.

Start with Option A, fall back to B if imports break after an update.

### CORS details

The stock gateway's `_CORS_HEADERS` (line 236-239 of api_server.py) lacks:
- `PATCH` in Allow-Methods (breaks `/api/jobs/{id}` PATCH calls)
- `X-Telegram-Init-Data` in Allow-Headers (breaks Telegram auth)
- `X-Hermes-Session-Id` in Allow-Headers (breaks session continuity)

The proxy sets its own CORS headers that include all of these. For proxied requests, the proxy strips its CORS headers and lets the response pass through, but also patches in the correct CORS headers if Hermes's are insufficient.

---

## File Summary

| Action | Path | Description |
|--------|------|-------------|
| CREATE | `~/.hermes/miniapp-proxy/proxy.py` | Standalone proxy server (~350 lines) |
| CREATE | `~/Library/LaunchAgents/ai.hermes.miniapp-proxy.plist` | launchd service on port 8643 |
| MODIFY | `~/.cloudflared/config.yml` | Change tunnel target from 8642 → 8643 |
| NO CHANGE | `~/.hermes/hermes-agent/gateway/platforms/api_server.py` | Leave stock Hermes alone |
| NO CHANGE | `~/.hermes/miniapp/index.html` | Frontend already works with `window.location.origin` |

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Hermes module imports break after update | `/api/commands`, `/api/processes` return errors | Wrap imports in try/except, return empty lists with a warning. Fall back to chat-based dispatch. |
| Proxy process crashes | Mini app goes offline, but Hermes stays up | launchd KeepAlive restarts it. Add logging. |
| Session ID tracking | Session continuity lost between proxy restarts | Session IDs are stored in Telegram CloudStorage by the frontend. The proxy just passes `X-Hermes-Session-Id` through. |
| SSE streaming for chat | Response buffering breaks streaming | Use `aiohttp.ClientSession.get()` with chunked reading, flush SSE chunks immediately. Don't buffer. |
| Double CORS headers | Browser rejects duplicate headers | Proxy strips Hermes CORS headers on proxied responses, sets its own. |
