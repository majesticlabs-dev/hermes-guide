# Extending the Hermes Gateway API

How to add new endpoints, middleware, and features to the Hermes API server.

## File Location

`gateway/platforms/api_server.py` — the aiohttp-based web server.

## Adding a New Endpoint

Three steps: write a handler, register the route, add auth if needed.

### 1. Write the Handler

Handler methods live on the `ApiServerAdapter` class. They take a `web.Request` and return a `web.Response`.

```python
async def _handle_my_endpoint(self, request: "web.Request") -> "web.Response":
    """GET /api/my-endpoint — does something useful."""
    # Require authentication
    auth_err = self._check_auth(request)
    if auth_err:
        return auth_err

    # Your logic here
    data = {"status": "ok", "result": "..."}

    return web.json_response(data)
```

For POST endpoints with a JSON body:

```python
async def _handle_my_post(self, request: "web.Request") -> "web.Response":
    auth_err = self._check_auth(request)
    if auth_err:
        return auth_err

    try:
        body = await request.json()
    except Exception:
        return web.json_response({"error": "Invalid JSON"}, status=400)

    result = do_something(body.get("param"))
    return web.json_response({"output": result})
```

### 2. Register the Route

In the `connect()` method (search for `router.add_get`):

```python
self._app.router.add_get("/api/my-endpoint", self._handle_my_endpoint)
self._app.router.add_post("/api/my-post", self._handle_my_post)
```

### 3. Auth or No Auth

- `_check_auth()` validates Bearer token or Telegram initData
- Leave it out for public endpoints (like `/health` or `/api/commands`)
- Always add it for endpoints that modify state or expose private data

## Auth System

The gateway uses a dual auth flow in `_check_auth()`:

1. **Telegram initData** — Mini App auth from Telegram WebApp
   - first try Ed25519 verification using Telegram's public key
   - if that fails, fall back to HMAC-SHA256 verification using `TELEGRAM_BOT_TOKEN`
2. **Bearer token** — Standard `Authorization: Bearer <API_SERVER_KEY>`

Both are checked. Telegram auth is tried first. If neither validates, return 401.

## CORS

CORS is handled by `cors_middleware` in `api_server.py`.

**Configuration:** Set `API_SERVER_CORS_ORIGINS` in `.env`:
- `*` — allow all origins (needed for Telegram Mini App)
- `https://example.com` — allow specific origin
- Empty — block all browser requests (non-browser clients still work)

**Custom headers:** If your endpoint uses custom headers, add them to `_CORS_HEADERS`:
```python
_CORS_HEADERS = {
    "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
    "Access-Control-Allow-Headers": "Authorization, Content-Type, Idempotency-Key, X-Telegram-Init-Data, X-Hermes-Session-Id, X-My-Custom-Header",
}
```

For Telegram Mini Apps, `X-Telegram-Init-Data` and `X-Hermes-Session-Id` are not optional. If you forget them, the browser kills your request at preflight and you get 403s that look like auth problems but aren't.

## Static File Serving

The `_handle_miniapp` handler shows the pattern for serving static files from disk:

```python
async def _handle_static(self, request):
    filename = request.match_info.get("filename", "index.html")
    file_path = os.path.join(static_dir, filename)
    if not os.path.isfile(file_path):
        return web.Response(status=404, text="Not Found")
    # Determine content type from extension
    with open(file_path, "rb") as f:
        data = f.read()
    return web.Response(body=data, content_type=ct)
```

## Available Imports

The gateway can import from:
- `hermes_cli.commands` — Command registry (`COMMAND_REGISTRY`, `resolve_command`)
- `hermes_cli.config` — Configuration loading
- `agent.model_metadata` — Model context lengths
- `gateway.run` — Gateway config helpers (`_load_gateway_config`, `_resolve_gateway_model`)
- `cron.jobs` — Cron job management
- `tools.process_registry` — Background process tracking

## Path References

Always use `get_hermes_home()` for file paths — never hardcode `~/.hermes`:
```python
from hermes_constants import get_hermes_home
miniapp_dir = get_hermes_home() / "miniapp"
```

This respects the profile system where each profile has its own home directory.

## Testing

After changes, verify syntax:
```bash
cd hermes-agent
source venv/bin/activate
python3 -c "import py_compile; py_compile.compile('gateway/platforms/api_server.py', doraise=True)"
```

Test with curl:
```bash
# No auth needed
curl http://localhost:8642/health

# With auth
curl -H "Authorization: Bearer YOUR_KEY" http://localhost:8642/api/model-info

# Via tunnel
curl https://hs.yourdomain.com/api/commands
```

If you are debugging the mini app dashboard specifically, inspect the exact JSON shape the frontend gets:
```bash
curl -s http://localhost:8642/health | jq
curl -s -H "Authorization: Bearer YOUR_KEY" http://localhost:8642/api/processes | jq
```

## Common Pitfalls

- **Don't forget the route registration** — a handler without a route is dead code
- **CORS blocks custom headers** — if your client sends custom headers, add them to `_CORS_HEADERS`
- **PATCH method not in CORS** — add it to `Access-Control-Allow-Methods` if you use it
- **Static file paths must use `get_hermes_home()`** — hardcoding `~/.hermes` breaks profiles
- **Mini app auth is easy to botch** — Telegram auth needs the right signed payload; keep the HMAC fallback because it's far less fragile
- **Frontend and backend data shapes must match** — if the dashboard expects `name`, `running`, `uptime`, and you return something else, the UI will look broken even though the request succeeded
- **CloudStorage can hand you garbage** — normalize values before rendering or you'll end up displaying `[object Object]`
- **aiohttp is synchronous-ish** — handlers are async but the agent loop inside `_handle_chat_completions` runs synchronously in a thread
- **Score your process** — run the Hermes Scorecard weekly (`~/.hermes/skills/hermes-scorecard/scripts/score.py`), track trends in `~/.hermes/scorecards/`, and don't celebrate until scores move. See the scorecard skill definition at `~/.hermes/skills/hermes-scorecard/SKILL.md` for the 10 dimensions and scoring methodology.
