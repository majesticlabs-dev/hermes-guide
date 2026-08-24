# New Hermes Instance Operator Guide

A practical setup guide for bringing up a new Hermes instance or profile without leaking private context, mixing roles, or giving an agent instructions it cannot execute.

Use this as the starting point before the narrower guides:

- [`soul-md-example.md`](soul-md-example.md) for profile identity templates.
- [`multi-agent-profiles.md`](multi-agent-profiles.md) for multiple specialized profiles.
- [`gbrain-memory-plugin.md`](gbrain-memory-plugin.md) for local-first memory.
- [`installing-skills-from-url.md`](installing-skills-from-url.md) for installing individual skills.
- [`workspace-autostart.md`](workspace-autostart.md) and [`cloudflare-tunnel.md`](cloudflare-tunnel.md) for always-on gateway/API setups.

---

## Mental model: three layers

A Hermes instance is not just a model plus tools. Treat it as three separate layers.

### 1. Identity: who the agent is

Stored mainly in:

```text
~/.hermes/SOUL.md
~/.hermes/profiles/<profile>/SOUL.md
```

Identity defines:

- role and mission
- communication style
- approval boundaries
- what the agent should refuse or escalate
- action contract: what counts as real progress
- routing rules to other profiles or queues

Keep identity lean. If a rule is a repeatable procedure, it probably belongs in a skill. If it is project-specific factual context, it belongs in `AGENTS.md`. If it is a private user preference, it belongs in memory.

### 2. Knowledge: what the agent knows

Stored mainly in:

```text
~/.hermes/memories/USER.md
~/.hermes/memories/MEMORY.md
~/.hermes/sessions/
<project>/AGENTS.md
optional external memory provider
```

Knowledge includes:

- durable user preferences
- environment facts
- project/repo facts
- prior session history
- long-lived notes from a memory provider

Do not store task progress, one-off outputs, PR numbers, chat IDs, credentials, or stale status in memory. Use tickets, Kanban, cron output, or session history for those.

### 3. Tools, skills, and proficiencies: what the agent can do

Stored/configured mainly in:

```text
~/.hermes/config.yaml
~/.hermes/.env
~/.hermes/skills/
~/.hermes/profiles/<profile>/config.yaml
~/.hermes/profiles/<profile>/.env
~/.hermes/profiles/<profile>/skills/
```

Tools are capabilities: terminal, file, browser, web, memory, cron, delegation, gateway adapters, MCP, etc.

Skills are reusable procedures. Some are narrow tool recipes. Others are broad **proficiencies**: workspace hygiene, publishing safety, skill-building discipline, human SOP creation. Install proficiencies on profiles that need that operating behavior.

Hard rule: the identity layer must match the tool layer. Do not tell a profile to process X posts with `x_search` if `x_search` is disabled and no specialist handoff is configured. Do not tell a profile to create Kanban cards if it lacks the relevant tool/CLI path.

---

## Day 0 install checklist

### 1. Install Hermes

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Then start setup:

```bash
hermes setup
hermes doctor
```

### 2. Choose the default model/provider

Use the interactive picker first:

```bash
hermes model
```

Or set config values directly:

```bash
hermes config set model.provider <provider>
hermes config set model.default <model-name>
```

Examples of provider categories:

- high-reasoning model for orchestration, debugging, and hard judgment
- cheaper reliable model for ordinary operations and summaries
- specialized model/provider for a specific modality or data source
- fallback provider for outages or quota exhaustion

Do not hardcode private API keys into `config.yaml`. Put secrets in `.env` or provider auth storage.

### 3. Add credentials safely

Use the provider login/auth flow when available:

```bash
hermes login --provider <provider>
hermes auth add
hermes auth list
```

For env-based providers, write placeholders like this in docs and templates:

```bash
OPENROUTER_API_KEY=[REDACTED]
ANTHROPIC_API_KEY=[REDACTED]
GOOGLE_API_KEY=[REDACTED]
```

Never put real tokens, cookies, webhook URLs, chat IDs, customer data, or private account identifiers in a guide, SOUL.md example, skill, or committed repo.

### 4. Enable the minimum useful toolsets

Inspect tool availability:

```bash
hermes tools list
```

Common baseline for an operator profile:

```bash
hermes tools enable terminal
hermes tools enable file
hermes tools enable web
hermes tools enable skills
hermes tools enable memory
hermes tools enable session_search
hermes tools enable todo
```

Add only when needed:

```bash
hermes tools enable browser      # browser automation, higher privacy/cost surface
hermes tools enable cronjob      # scheduled jobs
hermes tools enable delegation   # subagent work
hermes tools enable image_gen    # media generation
hermes tools enable tts          # voice output
```

After tool changes, start a fresh session or reset the current one so the prompt/tool schema reloads.

### 5. Set safer defaults

Recommended defaults for most operator setups:

```bash
hermes config set approvals.mode smart
hermes config set display.busy_input_mode queue
hermes config set auxiliary.compression.provider auto
hermes config set session_reset.mode none
```

Why:

- `approvals.mode smart`: low-risk actions can proceed; risky/destructive actions still require approval.
- `display.busy_input_mode queue`: new messages wait instead of interrupting in-flight work.
- `auxiliary.compression.provider auto`: avoids stale cross-provider compression model mismatches.
- `session_reset.mode none`: prevents surprise daily or idle session resets that make the agent lose active context. Use manual `/reset` or `/new` when you intentionally want a clean session.

For gateways or production-like usage, restart after config changes:

```bash
hermes gateway restart
```

For CLI sessions, exit and start a new session.

---

## What to install first

Install only what the instance needs for its role. More skills and tools are not automatically better.

### Core proficiencies

For an operator or maintainer profile, install broad discipline skills first:

- workspace hygiene: route files before writing, check git, avoid orphan artifacts
- publishing safety: prevent private data from entering shared/public repos
- skill/tool SOP builder: turn repeated procedures into maintainable skills
- human playbook/SOP creator: turn messy operating knowledge into human-followable SOPs

If installing from a raw skill URL:

```bash
hermes skills install https://example.com/path/to/SKILL.md
hermes skills list
```

If installing a skill pack from a tap or repository, inspect before installing when possible:

```bash
hermes skills inspect <skill-or-pack-id>
hermes skills install <skill-or-pack-id>
hermes skills list
```

### Role-specific skills

Use a narrow set per profile.

Good examples:

- researcher: web research, citation discipline, source comparison, browser only if needed
- engineer: repo inspection, targeted patch, testing, code review handoff
- reviewer: read-only QA, test/lint commands, browser validation
- devops: deployment, logs, monitoring, cron/gateway/MCP operations
- librarian/intake: link capture, source triage, value extraction, KB routing
- marketing/copy: positioning, campaign ideation, humanization, channel-specific templates

Bad pattern: clone a general profile, leave every skill enabled, and only change the SOUL.md. That creates role mismatch. The profile sounds specialized but still has a kitchen-sink execution surface.

### Verify installed skills

```bash
hermes skills list
hermes -p <profile> skills list
```

Check the filesystem when profile-local skills matter:

```bash
find ~/.hermes/profiles/<profile>/skills -name SKILL.md -maxdepth 5
```

If a profile uses `.no-bundled-skills`, copy or install needed skills into that profile's own `skills/` tree. Do not assume root/default skills are visible.

---

## Configure memory

Hermes has several memory layers. Configure them intentionally. Keep native `MEMORY.md` and `USER.md` as small hot caches for every-turn steering. Put durable but low-frequency profile context into GBrain instead of creating a profile-local overflow file.

### Native memory

Native memory is the default starting point.

Typical files:

```text
~/.hermes/memories/USER.md     # user preferences and stable personal/context facts
~/.hermes/memories/MEMORY.md   # environment facts, conventions, lessons
```

Named profiles keep their hot memory in profile-local files:

```text
~/.hermes/profiles/<profile>/memories/USER.md
~/.hermes/profiles/<profile>/memories/MEMORY.md
```

These files are the prompt-visible cache, not the archive. Durable low-frequency profile context belongs in a profile-specific GBrain page, for example `hermes/profile-memory-overflow/<profile>`.

Use native memory for:

- stable preferences
- durable environment facts
- recurring corrections
- compact rules that reduce future steering

Do not use native memory for:

- raw session transcripts
- temporary task progress
- issue/PR numbers
- one-off project status
- secrets or credentials
- facts likely stale within a week

### Memory commands

```bash
hermes memory status
hermes memory setup
hermes memory off
```

For a named profile:

```bash
hermes -p <profile> memory status
hermes -p <profile> memory setup
```

### GBrain / pull memory

Use GBrain for searchable, inspectable durable context that should not be injected into every turn. The active GBrain source is the canonical owner. Discover it instead of assuming a per-profile SQLite database:

```bash
gbrain sources list
hermes memory status
```

Enable GBrain through Hermes when needed:

```bash
hermes memory setup
# choose: gbrain, if available
hermes config set memory.provider gbrain
hermes -p <profile> config set memory.provider gbrain
```

For hot-memory compaction, keep only every-turn steering in `MEMORY.md` and `USER.md`. Write the durable low-frequency remainder to a profile-specific `hermes-memory` page:

```text
hermes/profile-memory-overflow/<profile>
```

Use valid YAML frontmatter. Quote a title that contains a colon:

```markdown
---
title: "Hermes memory overflow: <profile>"
type: hermes-memory
profile: <profile>
tags:
  - hermes
  - memory
  - profile-<profile>
---

Durable low-frequency context goes here.
```

Write and verify the page:

```bash
gbrain put hermes/profile-memory-overflow/<profile> \
  --content "$(cat /tmp/profile-memory-<profile>.md)"
gbrain get hermes/profile-memory-overflow/<profile>
```

A successful write must report `status: created_or_updated` and `chunks > 0`. A zero exit code alone is not proof of success. If the result is `status: error` or `chunks: 0`, keep the local hot-memory source intact, inspect the frontmatter, and do not delete the local copy.

Read the page back from GBrain and verify the canonical source before removing any temporary local copy. Do not create `~/.hermes/personal-memory/hot-memory-overflow.md` as a second archive layer.

After changing the provider, restart the CLI or gateway.

### Backfill memory

If a provider supports backfill, dry-run first:

```bash
python scripts/gbrain_backfill.py --dry-run
python scripts/gbrain_backfill.py
```

Backfill should read existing memory and write deduplicated provider notes. It should not mutate the source memory markdown files.

### Memory quality rules

Use this decision rule:

```text
Every-turn steering -> MEMORY.md / USER.md
Durable low-frequency profile context -> profile-specific GBrain hermes-memory page
Repeatable procedure -> skill
Project facts -> AGENTS.md or project docs
Durable task across profiles -> Kanban
Scheduled recurring output -> cron
Temporary progress -> session/todo, not memory
```

Memory should be declarative, not imperative:

```text
Good: User prefers concise status reports with verification evidence.
Bad: Always answer concisely and always include verification.
```

Imperative memory can become accidental system-prompt policy. Keep policy in SOUL.md or skills.

---

## Configure a new profile

Profiles are isolated Hermes homes with their own identity, config, memory, sessions, logs, cron, and skills.

### 1. Decide the role before creating it

Write a one-line role first:

```text
This profile owns <domain>. It may <actions>. It must not <red lines>.
```

Examples:

```text
researcher owns evidence gathering and synthesis; it may use web/browser; it must not edit source code.
reviewer owns QA and verification; it may run tests and browser checks; it must not modify source code.
librarian owns source intake and KB routing; it may process links and create KB handoffs; it must not fake retrieval or save weak signals by default.
```

### 2. Create the profile

```bash
hermes profile create <profile> --clone
```

Use `--clone` for config, `.env`, and SOUL.md only. Avoid `--clone-all` unless you intentionally want to copy sessions, memory, logs, and state.

Show the result:

```bash
hermes profile show <profile>
```

### 3. Rewrite SOUL.md immediately

```bash
$EDITOR ~/.hermes/profiles/<profile>/SOUL.md
```

Include:

- role and scope
- default workflow
- tool-use contract
- handoff rules
- approval gates
- red lines
- verification requirements
- reporting format

Remove cloned identity text that no longer fits. A profile with a new name but old SOUL.md is not a new profile; it is a confused clone.

### 4. Configure toolsets to match the role

Inspect what the profile actually has:

```bash
hermes -p <profile> tools list
```

Enable only what the role needs:

```bash
hermes -p <profile> tools enable web
hermes -p <profile> tools enable skills
hermes -p <profile> tools enable memory
```

Disable risky or irrelevant toolsets:

```bash
hermes -p <profile> tools disable homeassistant
hermes -p <profile> tools disable image_gen
hermes -p <profile> tools disable browser
```

If SOUL.md says the profile should do something, verify the tool exists. If the tool belongs to another specialist, write a real handoff command into the SOP.

Correct handoff shape:

```bash
hermes -p <specialist> chat -q "<self-contained task with source, constraints, expected output, and verification>"
```

Wrong shape:

```bash
hermes profile run <specialist> ...
```

### 5. Configure model/provider deliberately

```bash
hermes -p <profile> model
```

Or:

```bash
hermes -p <profile> config set model.provider <provider>
hermes -p <profile> config set model.default <model-name>
```

Do not change routing/cost defaults blindly across profiles. A writing profile, code profile, and intake profile may need different models.

Set compression safely:

```bash
hermes -p <profile> config set auxiliary.compression.provider auto
```

Avoid `provider: auto` paired with a stale explicit model from another provider.

### 6. Configure memory per profile

```bash
hermes -p <profile> memory status
hermes -p <profile> memory setup
```

Decide whether the profile should have:

- its own private memory only
- shared project context via `AGENTS.md`
- shared team context files
- a local-first provider like GBrain
- no memory for sensitive/stateless tasks

Do not copy another profile's `USER.md` wholesale unless the role truly needs those facts. Prefer a small role-specific memory seed.

### 7. Install only role-matching skills

```bash
hermes -p <profile> skills list
hermes -p <profile> skills install <skill-id-or-url>
```

If using profile-local skills, verify actual files:

```bash
find ~/.hermes/profiles/<profile>/skills -name SKILL.md -maxdepth 5
```

A profile should not inherit broad unrelated capabilities just because the source profile had them.

### 8. Configure cron/gateway only after the profile works in CLI

Smoke-test CLI first:

```bash
hermes -p <profile> chat -q "State your role, tool limits, and one thing you must not do."
```

Then configure scheduled jobs or gateway bindings.

Cron is profile-scoped by Hermes home. Check both root and profile stores when debugging missing jobs:

```bash
hermes cron list
hermes -p <profile> cron list
```

Gateway changes require restart:

```bash
hermes gateway restart
```

For a profile-specific gateway/service, verify which `HERMES_HOME` the running process uses before assuming config changes are live.

---

## Intake-profile caution: the Librarian failure pattern

This section generalizes a real class of failures: a profile was told to process content, but its config/tools/SOP did not fully define how to process the content object.

### What goes wrong

Common failure chain:

1. SOUL.md says “route X/Twitter links to the specialist.”
2. The profile lacks the native X tool itself.
3. Browser/web fallback is available, so the agent tries browser extraction first.
4. The source blocks browser/web extraction.
5. The agent reports “needs specialist” instead of invoking the specialist it was already instructed to use.
6. The output summarizes failure mechanics instead of extracting value for the user.

This is a config/SOP mismatch, not just a model mistake.

### How to prevent it

For every intake profile, define source-specific processing rules with:

- source type detection
- preferred retrieval path
- fallback path
- specialist handoff command
- what counts as enough content
- output schema
- no-save/archive criteria
- verification step

Example X/Twitter rule:

```text
For X/Twitter URLs, treat the source as a content object, not a single text blob.
Inspect: single post vs thread, quote/repost/reply context, embedded posts, linked sources, media, screenshots/charts/video claims, and high-signal replies.
Use the X-native specialist first when X-native context is required.
Use browser/web extraction only after the X-native path fails or for linked non-X pages.
If the specialist can be invoked from this profile, invoke it in-turn and integrate the result.
Use Kanban only when the work must outlive the turn, is blocked, or requires review.
```

Example specialist handoff:

```bash
hermes -p <x-specialist-profile> chat -q "Process this X URL as a full content object: <url>. Inspect thread, quote/embedded posts, linked pages, media claims, and high-signal replies. Return: full-read status, content map, core thesis, why it matters, durable value, no-save/save recommendation, and unresolved context. Do not use browser as first path unless X-native retrieval fails."
```

Example Reddit rule:

```text
For Reddit URLs, try normal extraction. If incomplete, try the `.json` endpoint with a normal User-Agent. If JSON is blocked or incomplete, use browser/CDP only when a rendered/authenticated context is available. Do not conclude inaccessible until the JSON path has been tested.
```

### Profile configuration checklist for intake agents

Before declaring an intake profile ready:

```bash
hermes -p <profile> tools list
hermes -p <profile> skills list
hermes -p <profile> config
hermes -p <profile> chat -q "For an X URL, what exact retrieval path do you use first, and what do you do if it fails?"
```

Check the answer against actual tool availability. If it names a tool that is disabled, fix the config or rewrite the handoff rule.

### Required output schema for content intake

Use a stable shape so weak summaries do not hide missing retrieval:

```text
Status: read | partial | blocked | delegated
Source type: post | thread | article | page | repo | document | unknown
Content map: what was inspected
Core thesis:
Why it matters:
User relevance:
Durable value:
Actions taken:
Recommendation: save | archive | reply | delegate | ignore
Unresolved:
```

If blocked, say exactly which retrieval paths were tried and what the next executable action is. Do not tell the user to do a specialist handoff that the profile can perform itself.

---

## Configure project context

Project context belongs in the project, not in a profile memory blob.

At the root of a repo/workspace:

```text
AGENTS.md
```

Include:

- project purpose
- repo layout
- setup commands
- test commands
- coding conventions
- deployment/runtime notes
- important paths
- safety boundaries

Do not include:

- API keys or tokens
- personal data
- customer secrets
- temporary task state
- instructions that only apply to one profile identity

Every profile working inside that project should read the same `AGENTS.md`.

---

## Configure gateway / messaging

Use gateway setup for Telegram, Slack, Discord, email, API server, webhooks, and similar adapters.

```bash
hermes gateway setup
hermes gateway status
hermes gateway run
hermes gateway install
hermes gateway start
hermes gateway restart
```

Rules:

- Keep platform tokens in `.env` or secure auth storage.
- Do not commit tokens, chat IDs, webhook URLs, or private channel names to docs.
- Record only generic setup shape in public guides.
- After changing gateway config, restart the gateway.
- Verify the live gateway is running under the expected Hermes home/profile.

For profile-specific bots, verify:

```bash
hermes -p <profile> config
hermes -p <profile> tools list
hermes -p <profile> chat -q "What profile are you and what channel behavior do you own?"
```

If callbacks/buttons/actions are used, smoke-test the full action, not just UI acknowledgment. A button disappearing does not prove the backend created the card, saved the note, or ran the action.

---

## Configure cron jobs

Use Hermes cron for recurring work.

```bash
hermes cron list
hermes cron create "0 9 * * *"
hermes cron run <job-id>
hermes cron pause <job-id>
hermes cron resume <job-id>
hermes cron remove <job-id>
```

Profile-specific cron:

```bash
hermes -p <profile> cron list
hermes -p <profile> cron create "every monday 9am"
```

Cron rules:

- Prompt must be self-contained. Future jobs do not inherit the chat context.
- Use `no_agent` scripts for deterministic watchdogs that should only speak on changes/errors.
- Use scripts for data collection; use the agent for judgment/synthesis.
- Keep no-op jobs silent.
- Deliver only useful states: start if needed, blocked, finished, changed.
- For authenticated browser work, include login recovery inside the script; do not rely on a current interactive session.
- Verify which profile owns the job before editing or removing it.

Cron output and status are task evidence, not durable memory.

---

## Configure Kanban / durable multi-agent work

Use Kanban when work:

- survives beyond the current chat turn
- needs multiple profiles
- needs review or verification
- has dependencies
- benefits from an audit trail

Common commands:

```bash
hermes kanban boards list
hermes kanban --board <board> create "Task title" --assignee <profile>
hermes kanban --board <board> show <task-id>
hermes kanban --board <board> comment <task-id> "Evidence or update"
hermes kanban --board <board> dispatch --max 1
```

Card descriptions should include:

- goal
- relevant paths/URLs
- constraints and red lines
- expected output
- verification steps
- dependencies
- what not to touch

Workers must read the card description and all comments before executing. Completion must include evidence, not just “done.”

Use in-session delegation for quick bounded subtasks. Use Kanban for durable work.

---

## Public/private hygiene rules

Before sharing or pushing a guide, skill, SOUL example, or setup instructions, scan for:

- real user names or family details
- emails, phone numbers, addresses, account IDs
- private domains and private repo names when not intentionally public
- local usernames in absolute paths
- chat IDs, platform IDs, bot tokens, webhook URLs
- API keys, cookies, bearer tokens, OAuth secrets
- customer names or customer data
- private memory content
- raw session logs
- cost/routing details that are private operational policy

Use placeholders:

```text
<profile>
<project>
<repo>
<provider>
<model-name>
<chat-id>
[REDACTED]
```

Never use placeholders in a live operational handoff where the agent is claiming action. Placeholders are fine in public documentation examples.

---

## Verification checklist for a new instance

Run these before calling setup complete.

### Base instance

```bash
hermes doctor
hermes config check
hermes status --all
hermes tools list
hermes skills list
hermes memory status
hermes chat -q "State your role, enabled memory mode, and the safest next diagnostic command if tools fail."
```

Expected:

- doctor has no blocking errors
- model/provider works
- secrets are not printed
- tools match intended role
- memory provider status is clear
- agent can state its role and boundaries

### Profile

```bash
hermes profile show <profile>
hermes -p <profile> config
hermes -p <profile> tools list
hermes -p <profile> skills list
hermes -p <profile> memory status
hermes -p <profile> chat -q "State your role, red lines, and handoff target for work outside your lane."
```

Expected:

- SOUL.md matches the role
- no cloned stale identity remains
- tools match SOUL.md claims
- role-specific skills are installed
- memory hot cache is profile-local unless intentionally shared; durable low-frequency profile context belongs in a profile-specific GBrain page
- handoff command uses `hermes -p <profile> chat -q`, not an invalid profile command

### Gateway

```bash
hermes gateway status
hermes gateway restart
```

Then send a real test message from the target platform. Verify not just the reply, but the side effect if the message is supposed to create a card, save a note, run a job, or invoke a specialist.

### Cron

```bash
hermes cron list
hermes cron run <job-id>
```

Expected:

- job is in the intended profile store
- prompt is self-contained
- script paths exist
- output delivery target is correct
- no-op behavior is intentional

### Repo/docs publishing

```bash
git status --short
git diff --check
git grep -nE 'AKIA|sk-[A-Za-z0-9]|xox[baprs]-|ghp_|github_pat_|BEGIN (RSA|OPENSSH|PRIVATE) KEY|Bearer [A-Za-z0-9._-]+'
```

Also scan for local paths and personal identifiers:

```bash
git grep -nE '/Users/[^/ ]+|/home/[^/ ]+|chat_id|telegram|phone|email|token|secret|password|cookie|webhook'
```

Review findings manually. Some generic docs will mention words like `token`; that is fine if they do not contain real values.

---

## Quick setup sequence

For a brand-new operator instance:

```bash
# install and health check
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
hermes setup
hermes doctor

# choose model and safe defaults
hermes model
hermes config set approvals.mode smart
hermes config set display.busy_input_mode queue
hermes config set auxiliary.compression.provider auto

# inspect and enable baseline tools
hermes tools list
hermes tools enable terminal
hermes tools enable file
hermes tools enable web
hermes tools enable skills
hermes tools enable memory
hermes tools enable session_search
hermes tools enable todo

# memory
hermes memory setup
hermes memory status

# profile example
hermes profile create researcher --clone
$EDITOR ~/.hermes/profiles/researcher/SOUL.md
hermes -p researcher tools list
hermes -p researcher skills list
hermes -p researcher chat -q "State your role and tool limits."

# final check
hermes config check
hermes status --all
```

For a new specialist profile, do not stop at creation. A profile is ready only after identity, tools, skills, memory, handoffs, and a smoke test all match.

---

## Maintenance cadence

Weekly:

- review failed/blocked cron jobs
- check profile-specific gateway logs
- inspect Kanban blocked/review-required cards
- prune stale memories that no longer represent durable truth
- patch skills when the same failure repeats
- check whether profile SOUL.md grew into a junk drawer

Monthly:

- audit every active profile's toolsets against its role
- verify compression/model routing config
- review public docs for private-data drift
- test important specialist handoffs end-to-end
- archive or disable profiles that no longer own a live lane

The aim is not more automation. The aim is fewer silent mismatches between what the agent is told to do and what it can actually do.
