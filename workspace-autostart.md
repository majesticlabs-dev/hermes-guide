# Hermes Workspace — Auto-Start & Gateway Auth Setup

> **Status: legacy reference.** This operator setup now uses [nesquena/hermes-webui](https://github.com/nesquena/hermes-webui) as its selected browser interface. The local WebUI checkout is launched at login by `com.parantoux.hermes-webui`; the repository's `start.sh` starts the Python/vanilla-JS application against the existing Hermes installation. The Hermes Workspace setup below remains documented only for historical troubleshooting and should not drive current UI setup.

How the Hermes Workspace web UI starts automatically on this machine, connects to the gateway, and authenticates API requests (including /jobs).

---

## Architecture

Two launchd services start at login:

| Service | Plist | Port | Role |
|---------|-------|------|------|
| Hermes Gateway | `~/Library/LaunchAgents/ai.hermes.gateway.plist` | 8642 | Agent API server |
| Hermes Workspace | `~/Library/LaunchAgents/ai.hermes.workspace.plist` | 8612 | React/Vite web UI |

The gateway starts first (it has no dependency on workspace). The workspace connects to the gateway's HTTP API at `http://127.0.0.1:8642`.

---

## Launchd Configuration

### Gateway (`ai.hermes.gateway.plist`)

- **Binary**: `~/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main gateway run --replace`
- **Working directory**: `~/.hermes/hermes-agent`
- **Logs**: `~/.hermes/logs/gateway.log` / `gateway.error.log`
- **KeepAlive**: restarts on non-zero exit
- **RunAtLoad**: true

### Workspace (`ai.hermes.workspace.plist`)

- **Binary**: `pnpm dev` (dev server, not the built `.output/server/index.mjs`)
- **Working directory**: `~/hermes-workspace`
- **Logs**: `~/.hermes/logs/workspace.log` / `workspace.error.log`
- **KeepAlive**: restarts on non-zero exit
- **RunAtLoad**: true
- **Environment**:
  - `PORT=8612`
  - `HERMES_API_URL=http://127.0.0.1:8642`
  - `HERMES_API_TOKEN=<set to the same value as API_SERVER_KEY in ~/.hermes/.env>`

---

## Token Wiring (the /jobs 401 fix)

The workspace needs to authenticate against the gateway for protected endpoints like `/api/hermes-jobs`. Here's the chain:

1. **Gateway** reads `API_SERVER_KEY` from `~/.hermes/.env` and expects requests to carry `Authorization: Bearer <API_SERVER_KEY>`.
2. **Workspace** reads `HERMES_API_TOKEN` from its environment and sends it as a Bearer token to the gateway.

**Key insight**: the env var is called `HERMES_API_TOKEN` on the workspace side, NOT `HERMES_BEARER_TOKEN`. The value must match `API_SERVER_KEY` from `~/.hermes/.env`.

The workspace plist sets `HERMES_API_TOKEN` directly in the `EnvironmentVariables` dict so it survives reboots without relying on shell dotfiles.

**To rotate the token**: update `API_SERVER_KEY` in `~/.hermes/.env`, then update `HERMES_API_TOKEN` in `~/Library/LaunchAgents/ai.hermes.workspace.plist` to match, then `launchctl unload` + `launchctl load` both plists.

---

## Why `pnpm dev` Instead of `pnpm start`

The skill originally documented `pnpm start` which runs `.output/server/index.mjs`, but that built server path did not work on this machine (missing build artifacts, mismatched runtime expectations). Using `pnpm dev` with the Vite dev server works reliably and is what the launchd plist runs.

If a production build is needed later, run `pnpm build` first and verify `.output/server/index.mjs` exists and starts cleanly before switching the plist.

---

## Verification Commands

```bash
# Check both services are loaded
launchctl list | grep ai.hermes

# Check ports are listening
lsof -i :8642   # gateway
lsof -i :8612   # workspace

# Test gateway health
curl -s http://127.0.0.1:8642/health

# Test workspace serves UI
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8612

# Test authenticated endpoint (jobs page uses this)
curl -s -w "\n%{http_code}" http://127.0.0.1:8612/api/hermes-jobs
# Should return 200, not 401

# Reload a plist after editing
launchctl unload ~/Library/LaunchAgents/ai.hermes.workspace.plist
launchctl load ~/Library/LaunchAgents/ai.hermes.workspace.plist
```

---

## Troubleshooting

### /jobs returns 401 after reboot

**Cause**: `HERMES_API_TOKEN` in the workspace plist doesn't match `API_SERVER_KEY` in `~/.hermes/.env`, or the env var is missing from the plist entirely.

**Fix**: Ensure both values match. The plist must explicitly set `HERMES_API_TOKEN` — it is not inherited from `~/.hermes/.env`.

### Workspace not starting

**Cause**: launchd PATH may not include pnpm or node. The plist uses the full mise path to pnpm.

**Fix**: Check `workspace.error.log` in `~/.hermes/logs/`. If pnpm isn't found, update the `ProgramArguments` path in the plist to the absolute pnpm binary path (e.g., `/Users/<user>/.local/share/mise/installs/pnpm/<version>/pnpm`).

### Gateway not starting

**Cause**: Python venv path changed, or `~/.hermes/.env` is missing `API_SERVER_KEY`.

**Fix**: Verify the venv binary exists at the path in `ai.hermes.gateway.plist`. Check `gateway.error.log`.
