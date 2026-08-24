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

## 3. Teach the multi-subset delegation pattern

Hermes can fan work out across subagents, but the feature being enabled is not enough. The profile/SOUL prompt must teach the LLM when and how to split the work.

Copyable LLM instruction:

```markdown
## Multi-subset delegation

When a task contains multiple independent subsets of work, do not handle them sequentially in the main thread. Split the job into bounded subsets and run them in parallel with real subagent/tool calls.

Use this pattern when the subsets have no ordering dependency and no overlapping write surface, for example:
- code review + security audit + test coverage review
- frontend implementation + backend implementation + reviewer validation
- source extraction + market scan + synthesis/fact-check
- repo reconnaissance + failing-test investigation + docs/config inspection

Before delegating, define each subset with:
- objective
- exact inputs: repo/path/URL/files/context
- constraints and side-effect boundary
- expected output format
- verification criteria

For consequential fan-out, also freeze the dispatch contract:
- bind every subset to the same canonical goal artifact plus a version or digest
- assign an explicit, non-overlapping write surface and owner to each subset
- keep cost-, permission-, or risk-sensitive model/executor choices blocked until resolved instead of silently using a default
- name the reviewer/integration owner; parallel workers do not auto-merge their own output

Then execute the real delegation call, such as `delegate_task(tasks=[...])`, or create real Kanban/profile handoffs for durable work. Do not print tool-call-shaped text as prose. Do not use placeholders like `<path>`, `<cmd>`, or `<job_id>`; discover the real value or ask one targeted question.

After the subsets return, the main agent is the judge: compare outputs against the original request, resolve conflicts, run/inspect verification, and merge into one final answer. Workers do not grade themselves.

Do not use this pattern when the steps are inherently sequential, when two workers would edit the same files, or when a deterministic local command/script can answer the question faster than an LLM worker.
```

Runtime checks still matter:

```bash
hermes tools list | grep delegation
python3 - <<'PY'
from pathlib import Path
import yaml
cfg = yaml.safe_load((Path.home() / '.hermes/config.yaml').read_text())
print(cfg.get('delegation', {}))
PY
```

Look for `delegation` enabled, `max_concurrent_children > 1`, `max_async_children > 1` if present, and a valid delegation model/provider. But if the agent only prints `delegate_task(...)` text, the problem is not config — the LLM instruction failed.

## 4. Verify before writing and before reporting done

For file or code work:

- Read the existing artifact before editing.
- After editing, re-read or diff the touched artifact.
- Run the smallest relevant check: syntax, lint, unit test, smoke test, link check, or `git status`.
- Report only checks that actually ran. Never write “tests pass” unless the test command returned success.

Keep completion claims on an evidence ladder:

- **Prepared:** the plan, contract, handoff, or artifact exists.
- **Observed:** the runtime recorded that an action ran and produced a result.
- **Verified:** the matching test, readback, review, or acceptance gate passed.

Track review, CI, merge-readiness, and merge as separate receipts. A worker summary or exit code zero does not prove any later state.

*Evidence-state and dispatch-integrity vocabulary adapted selectively from [rlaope/oh-my-hermes](https://github.com/rlaope/oh-my-hermes/tree/ac0b5b5ee7a4503cde370d09cb92525457e444bb) (MIT). Its runtime, plugin, router, state store, memory provider, and role pack are not part of this Guide recommendation.*

For research or market work:

- Prefer primary sources and official docs before summaries.
- Label unverified social claims as unverified.
- Preserve the source URL or source file path behind the claim.

## 5. Use hooks where the runtime supports them

Some agent runtimes can run hooks after tool use or before stop. When available, use them as enforcement, not decoration:

- After file writes: run typecheck/lint/format checks for the touched surface.
- Before completion: run the acceptance command or verify the artifact exists.
- Feed hook output back into the agent so failures are corrected or reported.

Hermes profiles should still encode the same behavior even when hooks are unavailable: explicit tool use, verification commands, and a clear done definition.

## 6. Add an independent verification pass for high-risk claims

Use a reviewer profile, Janitor verifier, or second-pass checklist when the claim is easy to fake and costly to trust:

- “fixed” for code or runtime failures
- “deployed” or “pushed”
- “configured” for auth, tools, cron, gateway, MCP, or routing
- “pricing/current facts verified”
- “KB/Guide clean and pushed”

The verifier should inspect the artifact or command output, not just trust the worker summary.

## 7. Control token cost before adding tools

Before installing a compression proxy or routing layer, apply the cheap controls first:

- Keep profile SOUL files and skills role-specific.
- Avoid dumping huge logs, database rows, or source files when a targeted search/read is enough.
- Use cheaper models for triage, extraction, and bounded execution.
- Keep `auxiliary.compression.provider` set to `auto` unless there is a verified reason to override it.
- Track expensive workflows by job/profile so cost can be optimized at the source.

## 8. Keep push memory tiny and route lessons to skills

Always-loaded memory, SOUL.md, CLAUDE.md, and AGENTS.md are expensive because they are injected into every session. Treat them as a hot steering layer, not an archive.

Use four buckets during memory review:

- **Delete / archive:** shipped-work records, transient project status, old PRs, one-off logs, and anything git/session history already preserves.
- **Move to a skill:** workflow-specific lessons, recurring tool gotchas, profile-lane rules, or anything learned the hard way about a repeated procedure.
- **Move to cron:** recurring audits, reminders, watchdogs, scheduled briefs, and deterministic maintenance checks. Prefer `no_agent` script jobs when no reasoning is needed.
- **Keep in push memory:** cross-cutting preferences, approval boundaries, safety scar tissue, canonical paths, and facts that change behavior across many skills or profiles.

The rule of thumb: if a line does not change a future decision, remove it from push memory. If it changes one workflow, patch that workflow's skill. If it is a recurring check or timed follow-up, make it cron. If it needs recall but not every-turn steering, store it in pull memory, normally a profile-specific GBrain `hermes-memory` page or session-searchable history. Do not create a profile-local overflow file as a second memory layer.

## 9. Evaluate compression tools before wiring them in

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
