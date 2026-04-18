# Message Queuing & Telegram Batching

How Hermes handles incoming messages when the agent is already busy, and how Telegram batches rapid messages to avoid accidental interrupts.

---

## `display.busy_input_mode`

Controls what happens when a new user message arrives while the agent is still processing a previous one.

**Config location:** `display.busy_input_mode` in `~/.hermes/config.yaml`

```yaml
display:
  busy_input_mode: queue   # or interrupt
```

### Modes

| Mode | Behavior |
|------|----------|
| `interrupt` | **Default.** Kills the current agent run immediately and starts processing the new message. Any in-progress work from the interrupted run is lost. |
| `queue` | Waits for the current agent run to finish, then processes the queued message sequentially. No work is lost. |

### Current Setting

```yaml
display:
  busy_input_mode: queue
```

This is set to `queue` because rapid Telegram messages (especially on mobile) would otherwise interrupt long agent runs mid-task. With `queue`, messages are held and processed one at a time in order.

### When to Use Each Mode

- **`queue`** — recommended for Telegram and other chat platforms where users tend to send multiple short messages in quick succession. Prevents accidental interrupts from typing ahead.
- **`interrupt`** — useful when you want the latest instruction to take priority immediately (e.g., "stop what you're doing and answer this instead"). Good for CLI or single-message interfaces where the user is deliberately overriding.

---

## Telegram Batching

Telegram has built-in message batching that coalesces rapid messages before they reach the agent. This works alongside `busy_input_mode: queue` to handle the common pattern of users sending multiple messages in a burst.

### Text Batching

When multiple text messages arrive in quick succession, Hermes waits briefly to merge them into a single input:

- **Default window:** 0.6 seconds
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
HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS=0.6

# Text batch split delay — additional delay between batch segments
HERMES_TELEGRAM_TEXT_BATCH_SPLIT_DELAY_SECONDS=0.0

# Media batching — how long to wait for more photos before processing (seconds)
HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS=0.8
```

Increase the delays if you find messages aren't being grouped (user types slowly). Decrease if the agent feels sluggish to respond after a single message.

---

## How They Work Together

```
User sends message 1 → agent starts processing
User sends message 2 (0.3s later) → text batch window absorbs it
User sends message 3 (0.5s later) → text batch window absorbs it
  └─ 0.6s after message 1: batch closes, merged input queued
Agent finishes current run → queued batch is processed next
```

The combination means:

1. **Batching** merges rapid-fire messages into coherent single inputs.
2. **Queue mode** ensures the agent finishes what it's doing before picking up the next batch.
3. Neither feature loses messages — everything gets processed eventually.

---

## Verification

```bash
# Check the current busy_input_mode setting
grep busy_input_mode ~/.hermes/config.yaml

# Check env vars for batch timing
grep -i batch ~/.hermes/.env

# Test: send 3 quick messages via Telegram, verify they're merged into one agent turn
```

---

## Files

| File | Purpose |
|------|---------|
| `~/.hermes/config.yaml` | `display.busy_input_mode` setting |
| `~/.hermes/.env` | `HERMES_TELEGRAM_TEXT_BATCH_DELAY_SECONDS`, `HERMES_TELEGRAM_TEXT_BATCH_SPLIT_DELAY_SECONDS`, `HERMES_TELEGRAM_MEDIA_BATCH_DELAY_SECONDS` |
