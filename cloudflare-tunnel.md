# Hermes + Cloudflare Tunnel Guide

How to expose your local Hermes gateway to the internet using Cloudflare Tunnel.

## Why Cloudflare Tunnel?

Your Hermes gateway runs on `localhost:8642`. Telegram webhooks and mini apps need HTTPS to reach it. Cloudflare Tunnel gives you:

- No port forwarding (firewall stays sealed)
- No public IP exposed
- Free HTTPS with TLS handled by Cloudflare
- DDoS absorption at the edge
- Zero Trust auth if you want it

The alternative is port forwarding + dynamic DNS + reverse proxy + cert management. Don't do that.

## Install cloudflared

```bash
# macOS
brew install cloudflared

# Linux
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

## Authenticate

```bash
cloudflared tunnel login
```

This opens a browser to link your Cloudflare account. It saves a `cert.pem` to `~/.cloudflared/`.

## Quick Test (Temporary URL)

```bash
cloudflared tunnel --url http://localhost:8642
```

This gives you a random `*.trycloudflare.com` URL. Good for testing, but it changes on every restart.

## Production Setup (Stable URL)

### 1. Create a tunnel

```bash
cloudflared tunnel create hermes
```

Save the tunnel ID from the output. Credentials are written to `~/.cloudflared/<tunnel-id>.json`.

### 2. Route DNS

```bash
cloudflared tunnel route dns hermes hs.yourdomain.com
```

This creates a CNAME record in your Cloudflare DNS.

### 3. Configure

Create `~/.cloudflared/config.yml`:

```yaml
tunnel: hermes
credentials-file: ~/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: hs.yourdomain.com
    service: http://localhost:8642
  - service: http_status:404
```

The catch-all `http_status:404` at the end is required.

### 4. Run

```bash
cloudflared tunnel run hermes
```

### 5. Run as a Service (Recommended)

On macOS, `cloudflared service install` may create a broken LaunchAgent that only runs bare `cloudflared` with no tunnel arguments. That's useless.

What the LaunchAgent actually needs to run is:

```xml
<array>
  <string>/opt/homebrew/bin/cloudflared</string>
  <string>tunnel</string>
  <string>--config</string>
  <string>/Users/<user>/.cloudflared/config.yml</string>
  <string>run</string>
  <string>hermes</string>
</array>
```

Full plist path:
`~/Library/LaunchAgents/com.cloudflare.cloudflared.plist`

Reload it after editing:

```bash
launchctl bootout gui/$(id -u)/com.cloudflare.cloudflared 2>/dev/null || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl list | grep cloudflared
```

If it shows a PID and exit code `0`, the tunnel is finally behaving.

Linux systemd is still the sane option there:

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

## Multiple Services

If you want to route different paths to different services:

```yaml
ingress:
  - hostname: hs.yourdomain.com
    path: /api/.*
    service: http://localhost:8642
  - hostname: hs.yourdomain.com
    service: http://localhost:8080
  - service: http_status:404
```

## Troubleshooting

### "Cannot determine default origin certificate path"

You haven't authenticated. Run `cloudflared tunnel login`.

### "Port already in use"

Check if another tunnel is running: `ps aux | grep cloudflared`

### Cloudflare Error 1033

Cloudflare can resolve your hostname but cannot reach the tunnel. Translation: DNS is fine, `cloudflared` is dead.

Check:
```bash
cloudflared tunnel list
ps aux | grep '[c]loudflared'
```

If the process is not running:
```bash
cloudflared tunnel run hermes
```

### launchd Service Installed but Crash-Looping

If `launchctl list | grep cloudflared` shows exit code `1`, your plist is probably wrong. The common failure is a LaunchAgent that only runs `/opt/homebrew/bin/cloudflared` with no `tunnel run ...` arguments.

Inspect it:
```bash
cat ~/Library/LaunchAgents/com.cloudflare.cloudflared.plist
launchctl print gui/$(id -u)/com.cloudflare.cloudflared | head -20
```

### Tunnel connects but site doesn't load

- Check the `service` URL in config matches your app's port
- Check your app is actually running on that port: `curl http://localhost:8642/health`
- Check DNS propagation: `dig hs.yourdomain.com`
