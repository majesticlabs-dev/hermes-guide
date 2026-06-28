# Agent Honesty and Cost Controls

Use this checklist when configuring a Hermes profile, writing a skill, or reviewing an agent workflow. The goal is not more process. The goal is fewer fake completions, fewer fabricated details, and lower token waste.

## 1. Give the agent permission to say “I don’t know”

Add explicit uncertainty behavior to profile rules and task handoffs:

- State uncertainty plainly when it changes the decision.
- Never invent paths, commands, APIs, test results, job IDs, or external facts.
- If evidence is missing, say what evidence is missing and take the safe partial path.
- Prefer “blocked because X is unverified” over a plausible but ungrounded answer.

This matters because agents often optimize for sounding helpful. A good profile makes honesty the helpful behavior.

## 2. Tool-call-shaped text must mean real tool use

Do not let agents print tool-call-shaped text as if it executed. This is a common failure when an agent knows the right action but emits it as prose instead of using the runtime tool.

Invalid final output:

```text
delegate_task(goal="Fix the failing backend test", toolsets=["terminal", "file"])
```

Valid behavior:

- Call the real `delegate_task` tool in the tool channel.
- Or say `BLOCKED` and name the one missing prerequisite, such as the repo path or job ID.
- Never use placeholders like `<path>`, `<cmd>`, or `<job_id>` in a claimed handoff.

A simple review rule: if the final response contains `delegate_task(`, `cronjob(`, `terminal(`, or similar tool syntax, verify that a real tool call happened. If not, treat the answer as failed, not merely verbose.

## 3. Verify before writing and before reporting done

For file or code work:

- Read the existing artifact before editing.
- After editing, re-read or diff the touched artifact.
- Run the smallest relevant check: syntax, lint, unit test, smoke test, link check, or `git status`.
- Report only checks that actually ran. Never write “tests pass” unless the test command returned success.

For research or market work:

- Prefer primary sources and official docs before summaries.
- Label unverified social claims as unverified.
- Preserve the source URL or source file path behind the claim.

## 4. Use hooks where the runtime supports them

Some agent runtimes can run hooks after tool use or before stop. When available, use them as enforcement, not decoration:

- After file writes: run typecheck/lint/format checks for the touched surface.
- Before completion: run the acceptance command or verify the artifact exists.
- Feed hook output back into the agent so failures are corrected or reported.

Hermes profiles should still encode the same behavior even when hooks are unavailable: explicit tool use, verification commands, and a clear done definition.

## 5. Add an independent verification pass for high-risk claims

Use a reviewer profile, Janitor verifier, or second-pass checklist when the claim is easy to fake and costly to trust:

- “fixed” for code or runtime failures
- “deployed” or “pushed”
- “configured” for auth, tools, cron, gateway, MCP, or routing
- “pricing/current facts verified”
- “KB/Guide clean and pushed”

The verifier should inspect the artifact or command output, not just trust the worker summary.

## 6. Control token cost before adding tools

Before installing a compression proxy or routing layer, apply the cheap controls first:

- Keep profile SOUL files and skills role-specific.
- Avoid dumping huge logs, database rows, or source files when a targeted search/read is enough.
- Use cheaper models for triage, extraction, and bounded execution.
- Keep `auxiliary.compression.provider` set to `auto` unless there is a verified reason to override it.
- Track expensive workflows by job/profile so cost can be optimized at the source.

## 7. Keep push memory tiny and route lessons to skills

Always-loaded memory, SOUL.md, CLAUDE.md, and AGENTS.md are expensive because they are injected into every session. Treat them as a hot steering layer, not an archive.

Use four buckets during memory review:

- **Delete / archive:** shipped-work records, transient project status, old PRs, one-off logs, and anything git/session history already preserves.
- **Move to a skill:** workflow-specific lessons, recurring tool gotchas, profile-lane rules, or anything learned the hard way about a repeated procedure.
- **Move to cron:** recurring audits, reminders, watchdogs, scheduled briefs, and deterministic maintenance checks. Prefer `no_agent` script jobs when no reasoning is needed.
- **Keep in push memory:** cross-cutting preferences, approval boundaries, safety scar tissue, canonical paths, and facts that change behavior across many skills or profiles.

The rule of thumb: if a line does not change a future decision, remove it from push memory. If it changes one workflow, patch that workflow's skill. If it is a recurring check or timed follow-up, make it cron. If it needs recall but not every-turn steering, store it in pull memory such as GBrain, profile-local notes, or session-searchable history.

## 8. Evaluate compression tools before wiring them in

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
