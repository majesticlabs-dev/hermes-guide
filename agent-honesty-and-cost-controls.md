# Agent Honesty and Cost Controls

Use this checklist when configuring a Hermes profile, writing a skill, or reviewing an agent workflow. The goal is not more process. The goal is fewer fake completions, fewer fabricated details, and lower token waste.

## 1. Give the agent permission to say “I don’t know”

Add explicit uncertainty behavior to profile rules and task handoffs:

- State uncertainty plainly when it changes the decision.
- Never invent paths, commands, APIs, test results, job IDs, or external facts.
- If evidence is missing, say what evidence is missing and take the safe partial path.
- Prefer “blocked because X is unverified” over a plausible but ungrounded answer.

This matters because agents often optimize for sounding helpful. A good profile makes honesty the helpful behavior.

## 2. Verify before writing and before reporting done

For file or code work:

- Read the existing artifact before editing.
- After editing, re-read or diff the touched artifact.
- Run the smallest relevant check: syntax, lint, unit test, smoke test, link check, or `git status`.
- Report only checks that actually ran. Never write “tests pass” unless the test command returned success.

For research or market work:

- Prefer primary sources and official docs before summaries.
- Label unverified social claims as unverified.
- Preserve the source URL or source file path behind the claim.

## 3. Use hooks where the runtime supports them

Some agent runtimes can run hooks after tool use or before stop. When available, use them as enforcement, not decoration:

- After file writes: run typecheck/lint/format checks for the touched surface.
- Before completion: run the acceptance command or verify the artifact exists.
- Feed hook output back into the agent so failures are corrected or reported.

Hermes profiles should still encode the same behavior even when hooks are unavailable: explicit tool use, verification commands, and a clear done definition.

## 4. Add an independent verification pass for high-risk claims

Use a reviewer profile, Janitor verifier, or second-pass checklist when the claim is easy to fake and costly to trust:

- “fixed” for code or runtime failures
- “deployed” or “pushed”
- “configured” for auth, tools, cron, gateway, MCP, or routing
- “pricing/current facts verified”
- “KB/Guide clean and pushed”

The verifier should inspect the artifact or command output, not just trust the worker summary.

## 5. Control token cost before adding tools

Before installing a compression proxy or routing layer, apply the cheap controls first:

- Keep profile SOUL files and skills role-specific.
- Avoid dumping huge logs, database rows, or source files when a targeted search/read is enough.
- Use cheaper models for triage, extraction, and bounded execution.
- Keep `auxiliary.compression.provider` set to `auto` unless there is a verified reason to override it.
- Track expensive workflows by job/profile so cost can be optimized at the source.

## 6. Evaluate compression tools before wiring them in

Tools like Headroom are promising because they compress tool outputs, logs, RAG chunks, and agent context before the LLM sees them. Do not wire them into Hermes globally by default.

Evaluate first:

- Does compression preserve the exact facts needed for code/debug/research tasks?
- Can originals be retrieved when compressed context loses detail?
- Does it work with the active Hermes providers and profile-local auth without leaking secrets?
- Does it reduce cost after accounting for install, runtime, and failure overhead?
- Can it be enabled per profile or per job instead of globally?

Default recommendation: treat compression proxies as an evaluation item, not a live routing change, until a small benchmark proves savings without quality loss.

## Minimal SOUL.md clause

```markdown
## Honesty and Verification

- Say “I don’t know” when evidence is missing. Do not invent paths, commands, APIs, IDs, citations, or test results.
- Verify before writing and verify again before reporting done.
- Report only checks that actually ran, with the command or artifact that proves them.
- If verification fails or is unavailable, say blocked and name the missing evidence.
```
