# Hermes + 1Password CLI Guide

How to use 1Password with Hermes Agent for secure credential management.

## Setup

Hermes uses a 1Password Service Account token stored in `~/.hermes/.env`:

```bash
ONE_PASSWORD_API_KEY=[REDACTED]
```

The `op` CLI (v2+) reads this as `OP_SERVICE_ACCOUNT_TOKEN`.

## Using op CLI with Hermes

Set the token before running commands:

```bash
export OP_SERVICE_ACCOUNT_TOKEN=$(grep ONE_PASSWORD_API_KEY ~/.hermes/.env | cut -d= -f2-)

# List vaults
op vault list

# List items
op item list --vault <vault-name>

# Get a specific credential
op item get "Item Name" --vault <vault-name> --fields credential

# Get full item as JSON
op item get "Item Name" --vault <vault-name> --format json
```

## Useful Patterns

### Store an API key

```bash
op item create --category "API Credential" \
  --title "My API Key" \
  --vault myvault \
  "credential=the_secret_key" \
  "username=my-service"
```

### Retrieve and use an API key

```bash
# In a script
API_KEY=$(op item get "My API Key" --vault myvault --fields credential)
curl -H "Authorization: Bearer $API_KEY" https://api.example.com/endpoint
```

### Get all fields from an item

```bash
op item get "Item Name" --vault myvault --format json | jq -r '.fields[] | select(.value) | "\(.label): \(.value)"'
```

## Troubleshooting

### "op: command not found"

Install via Homebrew: `brew install op`

### Authentication errors

- Check `ONE_PASSWORD_API_KEY` in `.env`
- Service account tokens start with `ops_`
- Check the token has access to the vault you're querying

### Token not being read

Make sure there are no trailing spaces or newlines in the `.env` value.
