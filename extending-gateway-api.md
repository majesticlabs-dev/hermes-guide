# Extending the Hermes Gateway API

How to add new endpoints, middleware, and features to the Hermes API server.

> **Scope:** Current stock API extension points, not the old locally patched Mini App server. See [official API Server docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server). Audited against upstream `4429693a62f784b5f2bd3c7f773c154be97b0d98`; inspect your installed version before editing internals.

## File Location

`gateway/platforms/api_server.py` — the aiohttp-based web server.

## Adding a New Endpoint

Three steps: write a handler, register the route, add auth if needed.

### 1. Write the Handler

Handler methods live on the `APIServerAdapter` class. They take a `web.Request` and return a `web.Response`.

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

Add rows in `_http_route_table()`, which `connect()` uses to register routes:

```python
("GET", "/api/my-endpoint", self._handle_my_endpoint),
("POST", "/api/my-post", self._handle_my_post),
```

### 3. Auth or No Auth

- `_check_auth()` validates the stock API Bearer token, not Telegram Mini App initData
- `/health` is public; `/api/commands` was a custom Mini App route, not a stock public endpoint
- Always add it for endpoints that modify state or expose private data

## Auth System

The stock `_check_auth()` compares the HTTP Authorization header's Bearer value with the configured `API_SERVER_KEY`. The previous Ed25519/HMAC description applied to custom Mini App code, not current upstream Hermes. A separate proxy must implement Telegram validation and user authorization itself before translating to a Bearer token. Never expose the upstream token in frontend assets. Some API surfaces have additional scoped permission checks; follow their existing authorization contract.

## CORS

CORS is handled by `cors_middleware` in `api_server.py`.

**Configuration:** Set `API_SERVER_CORS_ORIGINS` in `.env`:
- `*` — allow all origins; not inherently required by Telegram Mini Apps and not an auth mechanism
- `https://example.com` — allow specific origin
- Empty — block all browser requests (non-browser clients still work)

**Custom headers:** If your endpoint uses custom headers, add them to `_CORS_HEADERS`:
```python
_CORS_HEADERS = {
    "Access-Control-Allow-Methods": "GET, POST, DELETE, PATCH, OPTIONS",
    "Access-Control-Allow-Headers": "Authorization, Content-Type, Idempotency-Key, X-Telegram-Init-Data, X-Hermes-Session-Id, X-My-Custom-Header",
}
```

The block above is a custom extension example, not the current stock header set. Stock allows `GET, POST, DELETE, OPTIONS` and `Authorization, Content-Type, Idempotency-Key, X-Hermes-Session-Id`. `PATCH` and `X-Telegram-Init-Data` need explicit handling for a custom browser client; the session header is already present. Prefer an explicit origin allowlist. Server-to-server clients generally do not need CORS.

## Static File Serving

There is no stock `_handle_miniapp` handler. Use a traversal-safe static-file mechanism and review security headers before adding a UI. The previous unchecked path-join example is not an acceptable public-file boundary.

## Available Imports

The old custom integration used imports such as these. They are version-coupled implementation details, not a stable public extension API:
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

After changes, verify syntax, then run relevant gateway tests and exercise the new endpoint with authenticated/unauthenticated requests, invalid inputs, profile boundaries, and browser preflight where applicable. Syntax alone is not acceptance evidence:
```bash
cd hermes-agent
# Use the selected development environment; see python-uv-venv.md.
python3 -c "import py_compile; py_compile.compile('gateway/platforms/api_server.py', doraise=True)"
```

Test with curl:
```bash
# No auth needed
curl http://localhost:8642/health

# With auth
curl -H "Authorization: Bearer YOUR_KEY" http://localhost:8642/v1/models

# Via tunnel
curl https://hs.yourdomain.com/health
```

If you are debugging the mini app dashboard specifically, inspect the exact JSON shape the frontend gets:
```bash
curl -s http://localhost:8642/health | jq
# /api/processes is custom, not a stock API check.
```

## Common Pitfalls

- **Don't forget the route registration** — a handler without a route is dead code
- **CORS blocks custom headers** — if your client sends custom headers, add them to `_CORS_HEADERS`
- **PATCH method not in CORS** — add it to `Access-Control-Allow-Methods` if you use it
- **Static file paths must use `get_hermes_home()`** — hardcoding `~/.hermes` breaks profiles
- **Mini app auth is easy to botch** — custom Telegram auth must verify signature, freshness, user allowlist, and failure behavior independently of stock Bearer auth
- **Frontend and backend data shapes must match** — if the dashboard expects `name`, `running`, `uptime`, and you return something else, the UI will look broken even though the request succeeded
- **CloudStorage can hand you garbage** — normalize values before rendering or you'll end up displaying `[object Object]`
- **aiohttp is synchronous-ish** — handlers are async but the agent loop inside `_handle_chat_completions` runs synchronously in a thread
- **Score your process** — run the Hermes Scorecard weekly (`~/.hermes/skills/hermes-scorecard/scripts/score.py`), track trends in `~/.hermes/scorecards/`, and don't celebrate until scores move. See the scorecard skill definition at `~/.hermes/skills/hermes-scorecard/SKILL.md` for the 10 dimensions and scoring methodology.
