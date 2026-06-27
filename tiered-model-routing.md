# Tiered Model Routing for Hermes

Use tiered routing when one Hermes profile needs to coordinate work across models with different cost, context, and reliability tradeoffs.

Important constraint: Hermes currently has one global `delegation` block per profile. `delegate_task` does not accept a per-task provider/model override. True tiering requires either:

1. the profile's default `delegation` lane,
2. explicit child Hermes/Pi processes with verified `--provider` and `-m` model IDs, or
3. separate specialist profiles with their own configs.

Do not pretend a lane exists just because it is named in prose. Smoke-test the exact provider/model path first.

## Current Routing Pattern

| Lane | Example model/provider | Use when | Notes |
|---|---|---|---|
| Deterministic/no-agent | script-only cron/job | check-ins, backups, watchdogs, exact fixed messages | Best cost is no LLM. |
| Free/local cheap lane | `beelink / Qwopus3.6-27B-Coder-Compat-MTP-Q6_K` | tiny/simple prompts, smart-model-routing cheap model, mechanical low-risk work | Free when the Beelink endpoint is online; smoke-test before depending on it. |
| Trivial / low-stakes hosted lane | `kimi-coding / kimi-k2.7-code` verified; other direct Kimi lanes include Kimi K2.6 / Kimi-for-coding | formatting, simple edits, extraction, boilerplate, cheap motion | Hermes direct smoke passed; Pi 0.73.1 may not list this lane. |
| High-context / high-IQ leverage | `zai / glm-5.2` | repo-scale context, second review, agentic coding, frontend/design, long-context synthesis | Good default delegation lane when leverage matters. |
| Premium fallback | `openai-codex / gpt-5.5` | hard failures, production/security risk, final arbitration | Keep scarce; don't use as first reflex. |
| Social/X signal | Grok / xAI profile | social scans, buyer language, X intelligence | Keep social work in the social profile/lane. |

## Recommended Default Shape

```yaml
delegation:
  provider: zai
  model: glm-5.2
  reasoning_effort: high
```

Why:

- GLM-5.2 gives a large context window and strong reasoning at lower cost than premium fallback models.
- It is good for delegation where context depth, repo understanding, or second-review quality matters.
- Trivial work should be routed away from GLM-5.2 rather than weakening the GLM lane.

## Explicit Child Process Pattern

Use a spawned Hermes process when you need a specific non-default lane.

### GLM-5.2 leverage lane

```bash
hermes chat --provider zai -m glm-5.2 -q "<task>"
```

### GPT-5.5 premium fallback lane

```bash
hermes chat --provider openai-codex -m gpt-5.5 -q "<task>"
```

### Kimi trivial/front-end lane

Use the verified direct Hermes lane:

```bash
hermes chat --provider kimi-coding -m kimi-k2.7-code -q "<task>"
```

Other direct Kimi lanes may include `kimi-coding/kimi-k2.6`, `kimi-coder/kimi-k2.6`, and `kimi-coding/kimi-for-coding`; verify with the runtime before configuring.

If checking through Pi, note that Pi 0.73.1 may not list `kimi-k2.7-code` even when Hermes can run it directly:

```bash
set -a
[ -f "$HOME/.hermes/.env" ] && . "$HOME/.hermes/.env"
set +a
pi --list-models kimi
```

Do not substitute a paid OpenRouter Kimi route unless the user explicitly approved OpenRouter for that run.

## Routing Rules

1. **No-agent first** for deterministic recurring work.
2. **Beelink/Qwopus for free local cheap motion** when the endpoint is online and the task is tiny/simple.
3. **Kimi for stronger hosted cheap motion** through verified `kimi-coding/kimi-k2.7-code`.
4. **GLM-5.2 for leverage**: large context, repo-scale reasoning, agentic coding, second review, design/frontend generation.
5. **GPT-5.5 for premium fallback**: hard debug, high blast radius, final arbitration.
6. **Penalize verbosity**. A cheaper per-token model can be more expensive if it produces unnecessary output.
7. **Approval-gate config/cost changes**. Model defaults, fallback chains, cron models, gateway config, and provider spend changes need explicit approval plus rollback.

## Verification Checklist

Before claiming a routing change works:

```bash
hermes profile list
hermes config | grep -A12 '^delegation:'
hermes chat --provider beelink -m Qwopus3.6-27B-Coder-Compat-MTP-Q6_K -q 'Output exactly BEELINK_OK' -Q
hermes chat --provider kimi-coding -m kimi-k2.7-code -q 'Output exactly KIMI_OK' -Q
hermes chat --provider zai -m glm-5.2 -q 'Output exactly GLM_OK' -Q
hermes chat --provider openai-codex -m gpt-5.5 -q 'Output exactly GPT_OK' -Q
pi --list-models kimi
```

If using a profile-specific lane:

```bash
hermes -p <profile> chat -q 'Output exactly PROFILE_OK' -Q
```

For config changes, also verify:

- backup exists,
- diff is exactly the intended routing change,
- new session or gateway restart/reset happened when required,
- fallback path is known,
- no paid provider was introduced silently.

## Fake Tool-Call Guard

Routing instructions must become real execution. A response that prints this is a failure:

```text
delegate_task(goal="...", toolsets=["terminal"])
```

The agent must either call the real tool or say what prerequisite is missing. Do not use placeholders like `<path>`, `<cmd>`, or `<job_id>` in an actual handoff.

## Rollback

For config-backed routing changes:

1. Restore the backup copy of `~/.hermes/config.yaml` or profile-local `config.yaml`.
2. Restart the affected CLI/gateway session or run a fresh session.
3. Re-run the relevant smoke tests.
4. Confirm no cron jobs still pin the reverted model/provider.
