# Multi-Agent Profiles for Hermes

How to run multiple specialized Hermes agents — each with its own personality, memory, and workspace — by cloning profiles, writing dedicated SOUL.md files, sharing AGENTS.md, and invoking them separately.

## Why Isolated Profiles Beat Context Stuffing

The temptation is to cram everything into one agent: "You're a writer AND a VP of Marketing AND a debug engineer." That works for about five minutes, then the context gets muddy. The agent loses coherence because it's trying to hold contradictory mandates in one head.

Isolated profiles solve this cleanly:

- **Each profile gets its own SOUL.md.** The writer agent doesn't need to think about CAC projections. The marketing VP doesn't need to care about code style.
- **Each profile gets its own memory, sessions, and logs.** No cross-contamination between agent workspaces.
- **Each profile gets its own config.yaml.** You can route different profiles to different model providers (e.g., writer on GLM-5.1, vpmktg on GPT-5.4).
- **AGENTS.md is shared across profiles.** Project-level context — repo layout, API surface, deployment details — stays in one file that every profile reads. No duplication, no drift.

The pattern is simple: one Hermes install, multiple profiles, each a specialist. You invoke the right one for the right job.

## The 4-Step Setup

### Step 1: Clone a Profile

Create a new profile by cloning your existing one. This copies `config.yaml`, `.env`, and `SOUL.md` from the active profile as a starting point.

```bash
hermes profile create writer --clone
hermes profile create vpmktg --clone
```

Flags:
- `--clone` copies config.yaml, .env, and SOUL.md from the active profile.
- `--clone-all` does a full state copy (memories, sessions, logs, everything).
- `--clone-from SOURCE` lets you pick which profile to clone from instead of the active one.
- `--no-alias` skips the wrapper script creation if you don't want one.

### Step 2: Write a Dedicated SOUL.md

Edit the new profile's SOUL.md to define that agent's personality, expertise, and operating rules. This is the key differentiator between profiles — it's what makes the writer different from the marketing VP.

```bash
vim ~/.hermes/profiles/writer/SOUL.md
vim ~/.hermes/profiles/vpmktg/SOUL.md
```

The SOUL.md should define:
- **Identity and tone** — who this agent is, how it communicates
- **Domain expertise** — what this agent specializes in
- **Operating principles** — how it approaches tasks, what it avoids
- **Process discipline** — any hard rules for this role

### Step 3: Share AGENTS.md

AGENTS.md lives at the project/repo level and describes the codebase, API, deployment details, and anything all agents need to know. Every profile reads the same AGENTS.md — no per-profile copy needed.

```
your-project/
  AGENTS.md        <-- shared context, all profiles read this
  src/
  ...
```

AGENTS.md typically covers:
- Project structure and key files
- API endpoints and data models
- Deployment and environment details
- Coding conventions and patterns

This is where you put knowledge that's project-level, not agent-level. If you update the API surface, you update AGENTS.md once and every profile picks it up.

### Step 4: Invoke Separately

There are two ways to invoke a specific profile:

**Via the `-p` flag:**
```bash
hermes -p writer "Draft a blog post about our new API"
hermes -p vpmktg "Write a pricing page headline for the enterprise tier"
```

**Via wrapper scripts** (created automatically during `profile create` unless `--no-alias` is used):
```bash
writer "Draft a blog post about our new API"
vpmktg "Write a pricing page headline for the enterprise tier"
```

The wrapper scripts live at `~/.local/bin/<profile-name>` and are simple one-liners:

```sh
#!/bin/sh
exec hermes -p writer "$@"
```

They're just convenience aliases — functionally identical to `hermes -p NAME`.

## Directory Structure

After setup, the filesystem looks like this:

```
~/.hermes/
  profiles/
    writer/
      SOUL.md          <-- writer-specific personality and rules
      config.yaml      <-- writer-specific config (model, routing, etc.)
      .env             <-- credentials (shared or per-profile)
      memories/        <-- writer's own memory
      sessions/        <-- writer's session history
      logs/            <-- writer's logs
      skills/          <-- writer's installed skills
      plans/
      workspace/
      ...
    vpmktg/
      SOUL.md          <-- VP Marketing personality and rules
      config.yaml      <-- can use a different model provider
      .env
      memories/        <-- separate memory from writer
      sessions/
      logs/
      skills/
      plans/
      workspace/
      ...

~/.local/bin/
  writer               <-- wrapper: exec hermes -p writer "$@"
  vpmktg               <-- wrapper: exec hermes -p vpmktg "$@"

your-project/
  AGENTS.md            <-- shared project context (all profiles read this)
```

## Real Examples: writer and vpmktg

We set up two profiles following this exact pattern.

### writer Profile

Purpose: Long-form content, copywriting, editing, tone-shaping.

```bash
hermes profile create writer --clone
```

Then customized `~/.hermes/profiles/writer/SOUL.md` with writing-specific instructions:
- Strong opinions, no hedging, no corporate tone
- Brevity as default, depth only when requested
- Sharp wit, no filler
- Process discipline: PRD gate for multi-file tasks, circuit breaker on repeated failures, failure logging

Wrapper at `~/.local/bin/writer`:
```sh
#!/bin/sh
exec hermes -p writer "$@"
```

### vpmktg Profile

Purpose: Marketing strategy, campaign hooks, pricing page copy, positioning.

```bash
hermes profile create vpmktg --clone
```

Then customized `~/.hermes/profiles/vpmktg/SOUL.md` with marketing-specific instructions (same base personality as writer but tuned for marketing domain).

Wrapper at `~/.local/bin/vpmktg`:
```sh
#!/bin/sh
exec hermes -p vpmktg "$@"
```

### Using Them Side by Side

```bash
# Draft long-form content with the writer
writer "Write a 2000-word deep dive on why context windows matter"

# Generate campaign hooks with the marketing VP
vpmktg "Brainstorm 30 campaign hooks for the product launch"

# Or use the -p flag explicitly
hermes -p writer "Edit this draft to be more punchy"
hermes -p vpmktg "Give me three pricing page headline variants"
```

Both profiles share the same AGENTS.md at the project level, so they both understand the product, API, and codebase. But they bring different personalities and expertise to the conversation.

## Shared Context: Cross-Agent Knowledge Sharing

Isolated profiles are great for specialization, but agents still need to agree on fundamentals: writing style rules, market intel, and corrections. Duplicating these into every SOUL.md means drift. The shared-context pattern solves this.

### What It Is

A single directory, `~/.hermes/shared-context/`, containing three files that every profile reads:

| File | Purpose |
|------|---------|
| `THESIS.md` | Shared beliefs and positioning: writing rules, marketing principles, decision framework |
| `SIGNALS.md` | Reference intel: market signals, content performance data, competitive intelligence, the voice decision matrix |
| `FEEDBACK-LOG.md` | Style corrections and evolving conventions. Logged by any agent, enforced by all |

These files live outside any single profile. Every profile references them from the same path. Update once, every agent picks it up.

### Why It Matters

Without shared context, each SOUL.md has to duplicate style rules and conventions independently. That breaks fast: one profile gets updated, the other doesn't, they start disagreeing about whether em dashes are allowed.

The shared-context pattern gives you:

- **Single source of truth for cross-cutting rules.** Writing conventions, banned words, voice decision matrix, all in one place.
- **Live learning across agents.** When a reviewer catches a style violation, they log it in FEEDBACK-LOG.md. Every agent reads that file and treats it as a hard rule. One agent's correction becomes every agent's default.
- **FEEDBACK-LOG entries override SOUL.md defaults.** This is the key mechanism. If a feedback entry contradicts a personality trait, output protocol step, or guardrail, the feedback log wins. It's a living errata sheet that takes precedence over static instructions.
- **Market intel that any agent can append.** SIGNALS.md is append-only. Any agent that observes something worth noting (a competitor move, a content performance result, a new convention) adds an entry.

### How to Wire It Into SOUL.md

Add this block to every profile's SOUL.md (exact text, same in every profile):

```
On startup and when relevant tasks arrive, read these files:
- `~/.hermes/shared-context/THESIS.md` — shared beliefs and positioning
- `~/.hermes/shared-context/SIGNALS.md` — reference intel and market signals
- `~/.hermes/shared-context/FEEDBACK-LOG.md` — style corrections and learnings

Treat every entry in FEEDBACK-LOG as a hard rule that overrides defaults in this file.
If a feedback entry contradicts a Core Convention or voice rule, the feedback log wins.
```

That's it. Six lines of instruction plus a blank separator. Both the writer and vpmktg profiles have this exact block at the top of their SOUL.md files, right after the identity section.

### The Cleanup: Kill Redundant Skills

When you move shared knowledge into the shared-context directory, check whether any skills are now redundant. In our setup, the `style-writer` skill duplicated conventions that THESIS.md and FEEDBACK-LOG.md now cover. We deleted it:

```bash
hermes skill delete style-writer
```

The rule: if a skill's entire purpose is now handled by a shared-context file, kill the skill. SOUL.md + shared-context should be enough for style and convention rules. Skills should handle procedural workflows (how to deploy, how to run tests), not static reference material.

### Updated Directory Structure

With shared context added, the filesystem looks like this:

```
~/.hermes/
  shared-context/
    THESIS.md          <-- shared beliefs (all profiles read this)
    SIGNALS.md         <-- market intel (any agent can append)
    FEEDBACK-LOG.md    <-- style corrections (overrides SOUL.md defaults)
  profiles/
    writer/
      SOUL.md          <-- includes shared-context wiring block
      config.yaml
      .env
      memories/
      sessions/
      logs/
      skills/
      plans/
      workspace/
      ...
    vpmktg/
      SOUL.md          <-- includes shared-context wiring block
      config.yaml
      .env
      memories/
      sessions/
      logs/
      skills/
      plans/
      workspace/
      ...

~/.local/bin/
  writer               <-- wrapper: exec hermes -p writer "$@"
  vpmktg               <-- wrapper: exec hermes -p vpmktg "$@"

your-project/
  AGENTS.md            <-- shared project context (all profiles read this)
```

## Operator Layer: Keeping the Team Coherent Past Day 30

Setup gets you to day one. The operator layer gets you to day thirty. Multi-profile teams degrade in predictable ways if you don't actively maintain boundaries, handoffs, and memory hygiene. Below is the minimum viable operator runbook.

### Profiles Isolate More Than Tone

A profile isn't just a personality swap — it walls off seven independent state dimensions:

| Dimension | What breaks if shared |
|-----------|----------------------|
| configuration | Wrong model routing, incorrect provider keys |
| sessions | Cross-contaminated conversation history |
| memory | Research notes bleed into copy drafts |
| skills | Writer accidentally runs deploy workflows |
| personality | Role-specific tone and decision heuristics |
| cron state | Scheduled jobs fire for the wrong profile |
| gateway state | Messaging channels route to wrong agent |

Specialization stays durable only when all seven remain separated. If you're seeing role bleed, check which dimension leaked — it's almost always memory or personality.

### Handoff Contracts

When one profile's output feeds another profile's input, write an explicit handoff contract. These live at:

```
~/.hermes/team/handoffs/<from>-to-<to>.md
```

Each contract contains four fields:

1. **Input shape** — what the receiving profile expects (format, required fields, naming conventions)
2. **Output shape** — what the sending profile must deliver
3. **Failure action** — what happens when the contract is violated (reject and re-queue, escalate to orchestrator, halt)
4. **Verification gate** — how the receiver confirms the handoff landed correctly (checksum, schema check, smoke test)

Example: `writer-to-vpmktg.md` might specify that the writer delivers a draft in Markdown with H2 sections, the vpmktg profile validates that all H2s map to the campaign outline from SIGNALS.md, and on mismatch it escalates back to the writer with the specific missing sections.

Contracts don't need to be long. A five-line markdown file is enough. The point is making the interface explicit so both profiles can be tested against it independently.

### Weekly Memory-KPI Audit

Once a week, check each profile's memory health. The key metric is **stale notes ratio**: notes that haven't been referenced or updated in 7+ days as a percentage of total notes.

```
hermes -p writer memory stats
hermes -p vpmktg memory stats
```

Rules of thumb:

- **Stale ratio under 15%** — healthy, no action needed.
- **Stale ratio 15–30%** — schedule a `brain-resolve` pass on that profile to consolidate and prune.
- **Stale ratio above 30%** — the profile is accumulating junk memory. Full audit: delete irrelevant notes, merge near-duplicates, archive anything older than 30 days that isn't actively referenced.

This takes five minutes per profile and prevents the slow drift where an agent starts citing month-old context that's no longer accurate.

### Policy Ceilings by Role

Not every profile should have the same permissions. Define a risk class per profile and enforce it:

| Risk Class | Profiles | Permissions |
|------------|----------|-------------|
| **safe** | Research, writing | Read-only outside their own output directories |
| **review** | Building, debugging | Read repo, write feature branches, run sandboxed tests — no direct merges |
| **critical** | Orchestrator only | Approve merges, widen permissions, authorize spend above budget |

The orchestrator is the only profile that should touch production state, approve merges, or change shared configuration. This isn't about trust — it's about blast radius containment.

### The Four Failure Modes

These are the predictable ways a multi-profile setup degrades. Check for them monthly.

**1. Profile drift** — SOUL.md edits accumulate over weeks, roles blur. The writer starts sounding like the marketing VP; the researcher starts giving strategic recommendations.
- *Patch:* Diff each SOUL.md against its original intent weekly. If it's drifted more than ~20% from the original role definition, rewrite it.

**2. Handoff rot** — Contracts exist on disk but aren't actually enforced. Agents start passing freeform text instead of the agreed format.
- *Patch:* Wire handoff contract validation into the receiving profile's SOUL.md as a hard rule. If the input doesn't match the contract shape, reject it.

**3. SOUL.md bloat** — Edge cases and one-off instructions accrete until the file is 800+ words and the agent has lost its original identity.
- *Patch:* Cap SOUL.md at 400 words. If you need more, move procedural rules into a skill and reference material into shared context files.

**4. Cron collision** — Scheduled tasks across profiles fire at the same time or compete for shared resources.
- *Patch:* Maintain a shared `~/.hermes/team/cron-schedule.md` with staggered windows. No two profiles should run heavy jobs in the same 15-minute block.

## Practical Notes

- **Cloning copies everything except state.** `--clone` gives you config.yaml, .env, and SOUL.md from the source profile. `--clone-all` copies sessions, memories, and logs too. Usually `--clone` is what you want — clean state, inherited config.
- **Edit SOUL.md after cloning.** The cloned SOUL.md is a copy of the source profile's personality. Rewrite it immediately to match the new role.
- **Config can differ per profile.** The writer might run on GLM-5.1 while vpmktg runs on GPT-5.4. Edit each profile's config.yaml independently.
- **AGENTS.md is the shared brain.** Keep project-level knowledge in AGENTS.md, not in SOUL.md. SOUL.md is for identity, AGENTS.md is for context.
- **Wrapper scripts are optional.** If you prefer, just use `hermes -p NAME` directly. The wrappers are convenience, not requirement.
- **Memory is isolated.** Each profile has its own memories/ directory. The writer won't accidentally reference the marketing VP's session history. This is a feature, not a bug.
- **Queue busy input by default.** When Hermes is busy processing a message, incoming messages are dropped by default. Flip this to queue instead so nothing gets lost — add to `config.yaml`:
  ```yaml
  display:
    busy_input_mode: queue
  ```
  This is especially useful in multi-profile setups where orchestrator dispatches may arrive while a profile is mid-task.

## Verification

### List all profiles
```bash
hermes profile list
```

### Show profile details
```bash
hermes profile show writer
hermes profile show vpmktg
```

### Check wrapper scripts exist
```bash
ls -la ~/.local/bin/writer ~/.local/bin/vpmktg
```

### Test invocation
```bash
writer "ping"
vpmktg "ping"
```

Each should respond with its own personality and context.

## Files in This Setup

- `~/.hermes/shared-context/THESIS.md` — shared beliefs and positioning (cross-agent)
- `~/.hermes/shared-context/SIGNALS.md` — market intel and reference data (cross-agent)
- `~/.hermes/shared-context/FEEDBACK-LOG.md` — style corrections, overrides SOUL.md (cross-agent)
- `~/.hermes/profiles/writer/SOUL.md` — writer personality (includes shared-context wiring block)
- `~/.hermes/profiles/writer/config.yaml` — writer config
- `~/.hermes/profiles/vpmktg/SOUL.md` — marketing VP personality (includes shared-context wiring block)
- `~/.hermes/profiles/vpmktg/config.yaml` — marketing VP config
- `~/.local/bin/writer` — wrapper script
- `~/.local/bin/vpmktg` — wrapper script
- `AGENTS.md` (project root) — shared project context
