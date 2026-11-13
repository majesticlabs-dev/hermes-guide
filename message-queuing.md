# Message Queuing, Telegram Batching & Rich Messages

How Hermes handles incoming messages when the agent is already busy, how Telegram batches rapid messages to avoid accidental interrupts, and how to enable Telegram-native rich rendering.

---

## `display.busy_input_mode`

Controls what happens when a new user message arrives while the agent is still processing a previous one.

**Config location:** `display.busy_input_mode` in `~/.hermes/config.yaml`

```yaml
display:
  busy_input_mode: queue   # or interrupt (default), or steer
```

### Modes

| Mode | Behavior |
|------|----------|
| `interrupt` | **Default.** On current messaging gateways, redirects the active turn to the follow-up; a foreground terminal command can move to the background rather than being killed. |
| `queue` | Processes follow-ups as later turns after the current task. It is not a durability or eventual-delivery guarantee. |
| `steer` | Injects text into the active run after the next tool call; falls back to queue if the agent has not started. Attachments are queued, not reduced to text steering. |

### Example Setting

```yaml
display:
  busy_input_mode: queue
```

An operator may choose `queue` because rapid Telegram messages (especially on mobile) would otherwise interrupt long agent runs mid-task. With `queue`, messages are held and processed one at a time in order.

### When to Use Each Mode

- **`queue`** — recommended for Telegram and other chat platforms where users tend to send multiple short messages in quick succession. Prevents accidental interrupts from typing ahead.
- **`interrupt`** — useful when you want the latest instruction to take priority immediately (e.g., "stop what you're doing and answer this instead"). Good for CLI or single-message interfaces where the user is deliberately overriding.

---

## Telegram Batching

Telegram has built-in message batching that coalesces rapid messages before they reach the agent. This works alongside `busy_input_mode: queue` to handle the common pattern of users sending multiple messages in a burst.

### Text Batching

When multiple text messages arrive in quick succession, Hermes waits briefly to merge them into a single input:

- **Current adaptive defaults:** short messages use shorter delays; the configured text delay defaults to 1.5 seconds, and near-split continuations use a 2.0-second split delay. These are quiet windows, not a fixed timer from the first message.
- **Env var:** `HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS`
- **Split delay env var:** `HERMES_TELEGRAM_TEXT_BATCH_SPLIT_DELAY_SECONDS`

If you send "fix the bug" followed immediately by "in auth.py", the batching window merges them into one agent turn rather than treating them as two separate inputs.

### Photo / Media Batching

When multiple photos arrive in a burst (e.g., an album upload), Hermes groups them together:

- **Default window:** 0.8 seconds
- **Env var:** `HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS`

This handles the common case of sending multiple photos from the phone's gallery — they arrive as separate Telegram messages but should be processed as a single album.

### Configuring Batch Windows

Add to `~/.hermes/.env`:

```bash
# Text batching — how long to wait for more text messages before processing (seconds)
HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS=1.5

# Longer quiet window for likely split continuations (not an extra inter-segment sleep)
HERMES_TELEGRAM_TEXT_BATCH_SPLIT_DELAY_SECONDS=2.0

# Media batching — how long to wait for more photos before processing (seconds)
HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS=0.8
```

Increase the delays if you find messages aren't being grouped (user types slowly). Decrease if the agent feels sluggish to respond after a single message.

---

## How They Work Together

```
Rapid eligible text chunks → adapter waits for an adaptive quiet window → one merged input
Agent already busy → selected queue / interrupt / steer policy handles the follow-up
Photo burst → album batching → one media input
```

The combination means:

1. **Batching** merges rapid-fire messages into coherent single inputs.
2. **Queue mode** ensures the agent finishes what it's doing before picking up the next batch.
3. Batching is distinct from busy-input handling. Check actual ingress and execution evidence; neither setting proves delivery or protects against every failure/reset.

---

## Telegram Rich Messages

Telegram rich messages let Hermes send final replies through Telegram's native `sendRichMessage` path instead of flattening markdown. This preserves useful operator formats such as headings, tables, task lists, and collapsible `<details>` sections.

Enable it per Hermes profile in that profile's `config.yaml`:

```yaml
gateway:
  platforms:
    telegram:
      extra:
        rich_messages: true
```

For the default profile, edit `~/.hermes/config.yaml`. For named profiles, edit `~/.hermes/profiles/<profile>/config.yaml`.

After changing a running profile, restart that profile's gateway:

```bash
hermes gateway restart
hermes -p <profile> gateway restart
```

### Rich Message Test Prompt

Send this through Telegram after restart:

```text
Summarize this as a Telegram rich table with columns: Task, Owner, Status.
Give me a checklist for the deployment, using completed and incomplete task boxes.
Format this as:
- heading
- short summary
- table
- checklist
- collapsible details section for risks
```

Expected result on a compatible client/API: Telegram renders eligible rich constructs natively. Hermes may fall back to MarkdownV2 on rejection; CJK rich rendering has a separate opt-in. Verify on the actual client. If a Telegram client renders rich messages poorly, roll back by setting `gateway.platforms.telegram.extra.rich_messages: false` and restarting the affected gateway.

---

## Verification

```bash
# Check the current busy_input_mode setting
grep busy_input_mode ~/.hermes/config.yaml

# Check env vars for batch timing
grep -i batch ~/.hermes/.env

# Check rich message config across default + named profiles
python3 - <<'PY'
import yaml
from pathlib import Path
root = Path.home() / '.hermes'
configs = [root / 'config.yaml'] + sorted((root / 'profiles').glob('*/config.yaml'))
for cfg in configs:
    if cfg.exists():
        profile = 'default' if cfg.parent == root else cfg.parent.name
        data = yaml.safe_load(cfg.read_text()) or {}
        enabled = data.get('gateway', {}).get('platforms', {}).get('telegram', {}).get('extra', {}).get('rich_messages')
        print(f'{profile}: {enabled}')
PY

# Test: send 3 quick messages via Telegram, verify they're merged into one agent turn
# Test rich messages: send the rich message test prompt above and verify Telegram renders the table/checklist/details natively
```

---

## Sources

- [Messaging busy-input behavior](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/#queue-vs-interrupt-vs-steer-busy-input-mode)
- [Telegram rendering](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/telegram#rendering-rich-messages-tables-and-link-previews)
- Upstream `plugins/platforms/telegram/adapter.py` (`_TEXT_BATCH_DEFAULT_DELAY_S`, `_TEXT_BATCH_DEFAULT_SPLIT_DELAY_S`, `_text_batch_delay_for`) at `4429693a62f784b5f2bd3c7f773c154be97b0d98`.

## Files

| File | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | `display.busy_input_mode` and default-profile `gateway.platforms.telegram.extra.rich_messages` settings |
| `~/.hermes/profiles/<profile>/config.yaml` | Named-profile `gateway.platforms.telegram.extra.rich_messages` setting |
| `~/.hermes/.env` | `HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS`, `HERMES_TELEGRAM_TEXT_BATCH_SPLIT_DELAY_SECONDS`, `HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS` |
