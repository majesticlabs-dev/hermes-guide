# Tiered Model Routing for Hermes

How to run Hermes in delegate-first mode with a default GLM lane, GPT fallback, and named routing rules for GPT, GLM, and MiniMax models.

## What This Setup Does

This setup bakes in three things:

1. The top-level Hermes agent behaves like an orchestrator instead of trying to do all the work itself.
2. The default delegated lane is `zai / glm-5.1`.
3. If a delegated task fails, stalls, or turns into backend / infra / terminal-heavy work, the preferred fallback lane is `openai-codex / gpt-5.4`.

Important: Hermes does not have first-class Tier 1 / Tier 2 / Tier 3 routing in `config.yaml`. There is only one global `delegation` block.

So the actual pattern is:
- use `delegation` for the default lane
- use `agent.system_prompt` to tell the orchestrator how to choose lanes
- spawn explicit child Hermes processes with `--provider` and `-m` when you need a specific tier

## Current Routing Policy

### Default / unclear task
- Primary: `zai / glm-5.1`
- Fallback: `openai-codex / gpt-5.4`

### Clear task routing
- Backend, debugging, infrastructure, terminal-heavy work, incident response
  - `openai-codex / gpt-5.4`
- Hard general coding, long-horizon agentic work, product shaping
  - `zai / glm-5.1`
- Serious budget engineering, repo-wide refactors
  - `minimax / MiniMax-M2.7`
- Medium-cost general implementation
  - `openai-codex / gpt-5.4-mini`
- Cheap fast execution, triage, repetitive grunt work, first-pass ideation
  - `minimax / MiniMax-M2.5`
- Marketing and messaging
  - `openai-codex / gpt-5.4`
- Ideation deepening
  - `minimax / MiniMax-M2.7`

## Config Changes

These are the key config choices:

```yaml
fallback_model:
  provider: openai-codex
  model: gpt-5.4

agent:
  system_prompt: |-
    You are the Hermes orchestrator. Your default behavior is triage, route, delegate, review, and synthesize.
    Do not do substantive task execution yourself unless delegation is impossible or obviously unnecessary.
    Prefer delegate_task for the default lane, and when a specific tier is required,
    spawn a child Hermes process with explicit --provider and -m flags.
    ...

delegation:
  provider: zai
  model: glm-5.1
  reasoning_effort: high

prefill_messages_file: ~/.hermes/prefill-orchestrator-routing.json
```

Why:
- `delegation` fixes the broken default child lane and makes GLM-5.1 the first-choice delegated model.
- `fallback_model` gives a clean GPT-5.4 escape hatch.
- `agent.system_prompt` teaches the orchestrator which lane to pick.
- `prefill_messages_file` adds few-shot routing examples that bias the main model toward delegation-first behavior without changing Hermes source code.

## Why Not Put Every Tier in Config?

Because Hermes cannot do that natively.

`delegate_task` does not expose a per-call `provider` + `model` override. The global `delegation` block is the only built-in model override for subagents.

That means there are two practical ways to get true tiers:

1. Default lane via `delegate_task`
2. Explicit child processes for model-specific lanes

## Child Process Pattern for Specific Tiers

Use the Hermes CLI itself from the terminal tool when you need a non-default lane.

### GPT-5.4 lane
```bash
hermes chat --provider openai-codex -m gpt-5.4 -q "<task>"
```

### GPT-5.4-mini lane
```bash
hermes chat --provider openai-codex -m gpt-5.4-mini -q "<task>"
```

### GLM-5.1 lane
```bash
hermes chat --provider zai -m glm-5.1 -q "<task>"
```

### MiniMax M2.7 lane
```bash
hermes chat --provider minimax -m MiniMax-M2.7 -q "<task>"
```

### MiniMax M2.5 lane
```bash
hermes chat --provider minimax -m MiniMax-M2.5 -q "<task>"
```

For long jobs, run those in the background or inside `tmux`.

## Verification

### 1. Check the configured default delegated lane
```bash
hermes config | grep -A8 '^delegation:'
```

Expected shape:
```yaml
delegation:
  provider: zai
  model: glm-5.1
  reasoning_effort: high
```

### 2. Check the fallback model
```bash
hermes config | grep -A4 '^fallback_model:'
```

Expected shape:
```yaml
fallback_model:
  provider: openai-codex
  model: gpt-5.4
```

### 3. Check the orchestrator prompt exists
```bash
hermes config | grep -A20 '^agent:'
```

You should see the delegate-first routing instructions under `agent.system_prompt`.

## Practical Notes

- Keep the top-level model for orchestration and judgment.
- Use `delegate_task` as the default GLM lane.
- Escalate to GPT-5.4 when a task becomes terminal-heavy, infra-heavy, or ugly to debug.
- Use M2.5 for cheap volume.
- Use M2.7 when you want a cheaper but more agentic engineer.
- This is the strongest no-code version of orchestrator-only behavior: system prompt + prefill examples + routing defaults. It improves delegation compliance but does not hard-enforce it at the Hermes runtime level.
- Existing Hermes sessions may keep the old delegation model in memory. Start a new session, run `/reset`, or restart the gateway / CLI process to pick up the new routing config.

## Files Touched in This Setup

- `~/.hermes/config.yaml`
- `~/Documents/hermes-guide/tiered-model-routing.md`
- `~/Documents/hermes-guide/README.md`

## Rollback

Restore the backup copy of `~/.hermes/config.yaml` created before editing, then restart Hermes / gateway.
