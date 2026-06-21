# Hermes Dreaming Plugin

Hermes Dreaming is a third-party Hermes plugin for background memory consolidation. It periodically reviews recent sessions and existing prompt-visible memory, then proposes or applies a very small number of high-confidence memory operations.

Source: [alejandroiglesias/hermes-dreaming](https://github.com/alejandroiglesias/hermes-dreaming)

## What it does

Hermes keeps durable memory small and prompt-visible. That makes memory valuable but expensive: every extra character is paid on future sessions.

Hermes Dreaming adds a curator loop:

1. **Light** — scan recent sessions for candidate facts, preferences, corrections, and workflow lessons.
2. **Deep** — find repeated patterns, contradictions, stale entries, and superseded memories.
3. **REM** — score candidates by future usefulness per character, then apply only a few high-confidence changes.

A successful run may make zero memory changes. That is normal. The goal is better memory, not more memory.

## Install

```bash
hermes plugins install alejandroiglesias/hermes-dreaming
```

After installing, start a fresh Hermes session or restart the gateway so plugin commands are available.

## Commands

```bash
/dreaming review    # dry-run; proposes operations without changing memory
/dreaming run       # full consolidation cycle
/dreaming status    # last run, candidate counts, memory usage
/dreaming compact   # merge duplicates and remove obsolete entries; no new adds
```

CLI equivalents:

```bash
hermes dreaming review
hermes dreaming run
hermes dreaming status
hermes dreaming compact
hermes dreaming install-cron
```

## Recommended operating pattern

Start with review mode:

```bash
hermes dreaming review
```

Only run mutation mode after the proposed operations look right:

```bash
hermes dreaming run
```

Use `compact` when memory is bloated or contradictory and you want cleanup without adding new entries:

```bash
hermes dreaming compact
```

## State and audit files

Runtime state lives under:

```text
~/.hermes/dreaming/
├── DREAMS.md
├── state.json
├── candidates.jsonl
├── decisions.jsonl
├── promotions.jsonl
├── runs/
└── backups/
```

`DREAMS.md` is the human-readable audit diary. The JSONL files provide machine-readable sidecars for staged candidates, decisions, and promotions.

## Safety notes

- Treat this as a memory-mutating plugin, not just a passive skill.
- Use `/dreaming review` before `/dreaming run` on a new install.
- Do not promote project status, temporary TODOs, or stale session outcomes into durable memory.
- Prefer skills for reusable procedures and memory for compact, durable facts/preferences.
- Backups under `~/.hermes/dreaming/backups/` are part of the rollback story; verify they exist before relying on automated runs.

## When to use it

Use Hermes Dreaming when:

- the same user preference or correction appears across sessions;
- durable memory is full, duplicated, or contradictory;
- important workflow lessons are being lost in session history;
- you want a scheduled curator for `MEMORY.md` and `USER.md`.

Do not use it as:

- a replacement for a knowledge base;
- a vector database;
- a daily summary generator;
- a dumping ground for raw transcripts.
