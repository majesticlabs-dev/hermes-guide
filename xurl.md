# xurl — X API CLI for Hermes

`xurl` is the official X Developer Platform CLI for authenticated X API requests. Use it when Hermes needs real X API operations: reading posts, searching, timelines, mentions, bookmarks, likes, follows, DMs, media uploads, webhooks, or raw `/2/...` API calls.

For David's normal X URL/article capture workflow, prefer the `grok` profile first. Use `xurl` when the task explicitly needs X API behavior or when API access/cost is intentional.

## Current local status

Verified locally on 2026-05-21:

```bash
command -v xurl
# /opt/homebrew/bin/xurl

xurl version
# xurl 1.1.1
```

Authentication depends on which `HOME` Hermes is running with:

- David's normal shell home (`HOME=/Users/clawbot`) has `hermes-majestic` authenticated as `dpaluy`.
- The Atlas profile tool sandbox home (`HOME=/Users/clawbot/.hermes/profiles/atlas/home`) has no `xurl` apps registered.

The working verification command from an Atlas/Hermes tool call is:

```bash
HOME=/Users/clawbot xurl auth status
HOME=/Users/clawbot xurl whoami
```

Do not copy or read `~/.xurl` into the profile home from an agent session.

## Install

macOS Homebrew:

```bash
brew install --cask xdevplatform/tap/xurl
```

Shell installer:

```bash
curl -fsSL https://raw.githubusercontent.com/xdevplatform/xurl/main/install.sh | bash
```

Verify:

```bash
xurl version
xurl auth status
```

## One-time X Developer setup

Do this outside an agent session. Do not paste secrets into Hermes chat.

1. Open the X Developer Portal: https://developer.x.com/en/portal/dashboard
2. Create or open an app.
3. In User Authentication Settings, use app type **Web app, automated app or bot**.
4. Set redirect URI:

```text
http://localhost:8080/callback
```

5. Copy the app's Client ID and Client Secret.
6. Register the app locally:

```bash
xurl auth apps add my-app --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET
```

7. Start OAuth 2.0 PKCE login:

```bash
xurl auth oauth2 --app my-app YOUR_USERNAME
```

Passing `YOUR_USERNAME` is useful because X can fail the post-OAuth `/2/users/me` lookup with `UsernameNotFound` or 403.

8. Set the default app and user:

```bash
xurl auth default my-app YOUR_USERNAME
```

9. Verify:

```bash
xurl auth status
xurl whoami
```

## What else can be configured

### Registered apps

You can keep multiple X API apps locally:

```bash
xurl auth apps add prod --client-id ... --client-secret ...
xurl auth apps add staging --client-id ... --client-secret ...
xurl auth apps list
xurl auth apps update prod --client-id ... --client-secret ...
xurl auth apps remove staging
```

Use this for production vs testing apps, separate billing projects, or different permission scopes.

### Default app and default user

Set the global default:

```bash
xurl auth default prod alice
```

Override per request:

```bash
xurl --app staging /2/users/me
```

If an app has multiple authenticated users, select one per request:

```bash
xurl --app prod --username alice /2/users/me
```

### Redirect URI

The stored redirect URI can be inspected or changed:

```bash
xurl auth apps redirect-uri get --app prod
xurl auth apps redirect-uri set --app prod http://localhost:8080/callback
```

Keep this matched to the X Developer Portal setting.

### Auth mode

`xurl` supports different auth modes depending on the endpoint:

```bash
xurl --auth oauth2 /2/users/me
xurl --auth oauth1 /2/users/me
xurl --auth app /2/tweets/search/recent?query=hermes
```

Default for most personal-account operations should be OAuth 2.0.

### Raw API requests

Shortcut commands cover common work. Raw mode covers any X API v2 endpoint:

```bash
xurl /2/users/me
xurl -X POST /2/tweets -d '{"text":"Hello from xurl"}'
xurl -H "Content-Type: application/json" /2/users/me
xurl https://api.x.com/2/users/me
```

### Streaming

Known streaming endpoints auto-stream, or force streaming:

```bash
xurl /2/tweets/search/stream --auth app
xurl -s /2/users/me
```

### Webhooks

`xurl` includes webhook management:

```bash
xurl webhook --help
```

Use this for account activity/webhook workflows after the X app and environment support it.

### Shell completion

Generate shell completions:

```bash
xurl completion zsh
xurl completion bash
```

## Common commands

Read and search:

```bash
xurl read 1234567890
xurl read https://x.com/user/status/1234567890
xurl search "from:XDevelopers xurl" -n 10
xurl user @XDevelopers
xurl timeline -n 20
xurl mentions -n 20
```

Engagement:

```bash
xurl like 1234567890
xurl unlike 1234567890
xurl repost 1234567890
xurl unrepost 1234567890
xurl bookmark 1234567890
xurl unbookmark 1234567890
```

Posting:

```bash
xurl post "Hello world"
xurl reply 1234567890 "Agreed"
xurl quote 1234567890 "My take"
xurl delete 1234567890
```

Media:

```bash
xurl media upload image.png
xurl media status MEDIA_ID
xurl post "Image attached" --media-id MEDIA_ID
```

Social graph and DMs:

```bash
xurl follow @XDevelopers
xurl unfollow @XDevelopers
xurl followers -n 20
xurl following -n 20
xurl dm @someuser "Message text"
xurl dms -n 10
```

## Agent safety rules

Inside Hermes/agent sessions:

- Do not read, print, or summarize `~/.xurl`.
- Do not use `--verbose` / `-v`; it can expose auth headers.
- Do not pass inline secrets in commands.
- Do not ask the user to paste Client IDs, Client Secrets, bearer tokens, access tokens, or refresh tokens into chat.
- Verify only with safe probes:

```bash
command -v xurl
xurl version
xurl auth status
xurl whoami
```

## Common setup failures

- **OAuth succeeds but commands fail**: token saved to the wrong app. Re-run `xurl auth oauth2 --app my-app YOUR_USERNAME`, then `xurl auth default my-app YOUR_USERNAME`.
- **`unauthorized_client`**: X app type is probably wrong. Use **Web app, automated app or bot**, not Native App.
- **403 / `UsernameNotFound` after OAuth**: pass the username explicitly: `xurl auth oauth2 --app my-app YOUR_USERNAME`.
- **`client-forbidden` / `client-not-enrolled`**: X Developer project/package issue. Check billing/enrollment in the X Developer Portal.
- **`CreditsDepleted`**: API credits are exhausted. Add credits in the X Developer Portal.
- **X Article body not returned**: `/article/...` URLs are not normal status reads. Use Grok or authenticated browser extraction for article bodies.

## Cost boundary

The CLI is free/open-source. X API usage may be paid and plan-limited. Treat read/search/engagement automation as cost-sensitive; check current Developer Portal pricing before high-volume use.
