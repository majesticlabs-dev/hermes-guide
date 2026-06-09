# Agent-Native Toolbelt Implementation Plan

> **For Hermes:** Use this plan when converting repeated agent workflows into deterministic CLIs, local mirrors, skills, and compound commands. The goal is lower drift, lower API/token waste, and more reliable recurring work.

**Source:** Adapted from Matt Rice's X article, “Printing Press Made CLIs Feel Like Agent Superpowers,” and the linked Printing Press workflow concept.

**Goal:** Build a private Hermes toolbelt where repeated workflows are exposed as agent-native CLIs instead of raw APIs, ad-hoc scripts, or giant JSON dumps.

**Architecture:** Each workflow gets a deterministic CLI with job-shaped commands, a local SQLite/cache layer when payloads are bulky, explicit budget/rate guardrails, and a Hermes skill that teaches agents when and how to use it. Hermes remains the operator surface; specialist agents or builders create the CLI when implementation is needed.

**Tech Stack:** Go or Python CLIs, SQLite, JSON/Markdown output contracts, Hermes skills, cron where useful, profile-local auth, optional MCP surface only after the CLI core is stable.

---

## Principles

1. **Job-shaped commands beat endpoint wrappers.** Build commands around the recurring business action, not around API resource names.
2. **Deterministic core first.** Same inputs should produce the same output shape. The LLM decides what to do; the CLI handles plumbing.
3. **Local state beats context flooding.** Store bulky raw data in SQLite/cache and return compact summaries or IDs to the agent.
4. **Budget guardrails are product requirements.** Every paid API CLI needs dry-run, max-cost, cache-first, and usage-report behavior.
5. **Skills are part of the deliverable.** A CLI is not agent-native until Hermes knows when to call it, what commands are safe, and what output means.
6. **Compound commands are the moat.** The useful artifact is not `api get`; it is `score-link-target`, `compare-serp`, `extract-value`, or `audit-cron-health`.

---

## Phase 0 — Toolbelt Audit

**Objective:** Rank candidate workflows by leverage before building anything.

**Inputs:**
- Hermes profiles and skills already in use
- Existing recurring workflows, crons, and manual asks
- APIs/SaaS touched repeatedly
- Workflows with high token, API-credit, or babysitting cost

**Steps:**
1. List the agents used weekly: Hermes/default Atlas, Codex, Claude Code, OpenClaw, specialist Hermes profiles, Pi lanes.
2. List recurring business workflows currently run through agents.
3. List personal workflows where dashboards/apps are still being opened manually.
4. List repeated APIs, dashboards, SaaS tools, and local data sources.
5. Score each workflow 1–5 for frequency, cost pain, drift risk, implementation ease, and business leverage.
6. Rank by total score, then choose one CLI to build first.

**Deliverable:** `toolbelt-audit.md` with:
- install-now existing CLIs
- print-next missing CLIs
- skip-for-now distractions
- first compound commands

**Verification:** Each top candidate names the repeated workflow, current pain, expected savings, and exact first command.

---

## Phase 1 — First Build: DataForSEO CLI

**Objective:** Reduce paid API-credit waste and agent drift in SEO/link-building workflows.

**Why first:** It has the clearest cost signal: repeated API calls, giant JSON payloads, expensive credits, and obvious recurring jobs.

**Commands to design:**
- `dataforseo-cli keyword-metrics --keywords <file|list> --location <code> --max-cost <amount>`
- `dataforseo-cli serp-compare --keyword <kw> --domains <a,b,c> --date-range <range>`
- `dataforseo-cli competitor-snapshot --domain <domain> --markets <codes>`
- `dataforseo-cli link-target-score --url <url> --criteria <profile>`
- `dataforseo-cli budget-report --since <date>`
- `dataforseo-cli cache-status`

**Local mirror:** SQLite tables for requests, responses, normalized keyword metrics, SERP snapshots, domain scores, link-target scores, and spend estimates.

**Guardrails:**
- dry-run mode
- cache-first default
- max-cost required for paid pulls
- per-run summary of estimated and actual credits
- refusal on unbounded keyword/domain inputs

**Hermes skill:** `dataforseo-cli` skill with:
- when to use the CLI
- safe commands
- examples for link target scoring and competitor snapshots
- budget rules
- verification steps

**Verification:**
- smoke against one tiny fixture or mocked response
- one cache hit test proving no second paid pull
- one budget-limit test proving refusal
- one Hermes task using the CLI without raw API calls

---

## Phase 2 — Social/X Value Extraction CLI

**Objective:** Turn “extract value from this post” into a reusable capture pipeline instead of a one-off browser scrape.

**Commands to design:**
- `x-value extract <url>` — fetch post/article/thread and produce structured insight JSON
- `x-value summarize <id|url>` — concise thesis, useful pattern, caveats
- `x-value save-insight <id> --dest <kb|guide|drafts>`
- `x-value linkedin-draft <id> --voice <profile>`
- `x-value compare-priorities <id>` — map to active work without forcing unrelated anchoring

**Local mirror:** SQLite tables for source URL, author, captured text, extracted claims, reusable patterns, destination artifacts, and follow-up status.

**Guardrails:**
- read-only by default
- no posting without explicit approval
- preserve source URL
- label unverified claims from social posts

**Hermes skill:** `x-value-extraction` skill with standalone-analysis behavior and destination rules.

**Verification:**
- run on a public X article URL
- output includes source URL, thesis, reusable pattern, and recommended action
- no unrelated project anchoring unless the user asks

---

## Phase 3 — KB Janitor CLI

**Objective:** Make repeated KB cleanup safe, deterministic, and cheap.

**Commands to design:**
- `kb-janitor dirty-status`
- `kb-janitor classify-change --path <path>`
- `kb-janitor revert-metadata-only --path <path>`
- `kb-janitor commit-real-content --path <path>`
- `kb-janitor push-verified`
- `kb-janitor queue-report`

**Local mirror:** Optional SQLite run log for classified changes, commits, pushes, blockers, and recovered metadata residue.

**Guardrails:**
- never delete real content without explicit approval
- metadata-only residue can be reverted
- real content must be committed and pushed only after verification
- report `HEAD == origin` evidence when pushed

**Hermes skill:** extend the existing KB ops guidance rather than creating a noisy duplicate skill unless the workflow becomes broad enough.

**Verification:**
- run on a fixture repo or dry-run mode first
- verify diff classification before any revert
- verify git status after cleanup

---

## Phase 4 — Hermes Ops CLI

**Objective:** Give Atlas/Janitor deterministic commands for Hermes health, profile state, cron audits, and skill drift.

**Commands to design:**
- `hermes-ops profile-status --all`
- `hermes-ops cron-audit`
- `hermes-ops skill-drift --profile <name>`
- `hermes-ops toolbelt-gap`
- `hermes-ops guide-check`
- `hermes-ops weekly-health --json`

**Local mirror:** SQLite or JSONL run history for health checks, failures, recurring warnings, and accepted fixes.

**Guardrails:**
- read-only default
- config/auth/routing/cost/runtime changes are report-and-approval only
- auto-fix only low-risk reversible housekeeping with backups

**Hermes skill:** update existing Hermes-agent/weekly-health-audit guidance if the CLI becomes canonical.

**Verification:**
- command output is compact enough for agent context
- no secrets or tokens in output
- one weekly cron can consume the CLI output without model escalation

---

## Phase 5 — Pi Pack / Majestic AI Pack CLI

**Objective:** Preserve and inspect active Pi pack/plugin state without repeatedly loading ad-hoc context.

**Commands to design:**
- `pi-pack sort-hat --candidate <path|url>`
- `pi-pack status`
- `pi-pack preserve-check`
- `pi-pack supported-map`
- `pi-pack diff-supported --before <file> --after <file>`

**Local mirror:** Supported-pack map, preserved plugin metadata, candidate decisions, and audit history.

**Guardrails:**
- preserve actions by default
- never modify active packs without explicit approval
- additions run `sort-hat` first

**Hermes skill:** likely a project-specific skill or Guide appendix only after the workflow stabilizes.

**Verification:**
- dry-run candidate classification
- no active pack mutation during status/audit commands

---

## Implementation Order

1. Run Phase 0 audit.
2. Build the smallest DataForSEO CLI slice: `budget-report`, `cache-status`, and one job command.
3. Add the Hermes skill for that CLI.
4. Run one real or mocked workflow end to end.
5. Measure before/after: API calls, credits, token/context size, manual babysitting.
6. Only then build the X value extraction CLI.
7. Promote patterns that prove reusable into Hermes Guide and skills.

---

## Definition of Done for Any Toolbelt CLI

A CLI is done only when all of these exist:

- deterministic command contract
- compact machine-readable output
- local cache/mirror where useful
- explicit budget/rate limits for paid APIs
- safe dry-run or read-only mode
- Hermes skill with examples and warnings
- smoke tests or fixture tests
- one verified real workflow run
- rollback/removal notes

---

## What Not to Do

- Do not build a general API wrapper and call it agent-native.
- Do not expose raw giant JSON payloads to the model by default.
- Do not create MCP first if the deterministic CLI core is not proven.
- Do not install every possible CLI into every profile.
- Do not automate posting, deleting, spending, auth, routing, or config changes without explicit approval.
- Do not turn the audit into research theater. Pick one costly recurring workflow and ship the first command.

---

## First Concrete Next Action

Create `toolbelt-audit.md`, rank the candidate workflows, and choose the first DataForSEO command. If the DataForSEO credentials or billing constraints are unavailable, start with the Social/X value extraction CLI because it can be built read-only with lower risk.
