# Hermes Agent — Community Guide

Practical guides for setting up and extending Hermes Agent.

## Operator, Not Builder

Hermes is not a Builder. Hermes is an Operator.

- **Builders** create one-off artifacts: dashboards, static sites, prototypes, one-shot scripts.
- **Operators** run recurring work: briefings, alerts, scheduled reports, monitoring, memory maintenance, skill execution, escalation handling.

The pattern: a builder creates the surface, Hermes operates the workflow. Example: Claude builds a dashboard — Hermes runs daily analysis against it and sends the brief. Another agent writes a script — Hermes schedules it, monitors it, and escalates on failure.

If the task is "make me a thing," route to a builder. If the task is "keep this running," "watch for this," or "tell me when," that's Hermes.

## Start Here

- [LLM Start Here](LLM-START-HERE.md) — Machine-oriented bootstrap contract for interpreting this repository: reading order, authority hierarchy, evidence states, discovery rules, safety boundaries, and the standard architecture-handoff shape.
- [New Hermes Instance Operator Guide](new-instance-operator-guide.md) — The canonical setup sequence for a fresh Hermes instance or profile: install, model/provider config, toolsets, skills/proficiencies, memory, profile setup, gateway, cron, Kanban, verification, and privacy rules.
- [SOUL.md Example](soul-md-example.md) — Public-safe templates for profile-level agent behavior: role, tone, tool rules, approval gates, quality bar, and the boundary between SOUL.md, AGENTS.md, memory, and skills.
- [Copy-Ready Public SOUL.md](SOUL.md) — Compact, sanitized operator profile that can be copied into a Hermes installation and adapted without exposing private runtime details.
- [Multi-Agent Profiles](multi-agent-profiles.md) — How to run multiple specialized Hermes agents with isolated profiles, dedicated SOUL.md files, shared AGENTS.md, and separate invocation via `hermes -p NAME` or wrapper scripts.
- [GBrain Memory Plugin](gbrain-memory-plugin.md) — Local-first SQLite/FTS5 memory provider for Hermes with deterministic entity extraction, graph-style note linking, the `gbrain_note` tool, setup commands, privacy notes, test checklist, and hosting recommendation.
- [Hermes Dreaming Plugin](hermes-dreaming-plugin.md) — Background memory consolidation for Hermes, including `hermes plugins install alejandroiglesias/hermes-dreaming`, review/run commands, audit files, and safety notes for memory-mutating workflows.
- [Installing Skills from a URL](installing-skills-from-url.md) — How to install a Hermes skill directly from any `.md` URL, including Gists and raw hosted skill files, then refresh it later with `hermes skills update`.
- [Recommended Skills to Install](skills-to-install.md) — Profile-specific list of external/community skills worth adding, including `last30days` and its target profiles.
- [Skill Bundles](skill-bundles.md) — How to group repeatable skill clusters into one slash command with `hermes bundles create`.

## Operations Guides

- [Workspace Auto-Start & Auth](workspace-autostart.md) — How Hermes Workspace auto-starts via launchd after reboot, the gateway/workspace plist arrangement, and the HERMES_API_TOKEN wiring needed to make `/jobs` and other authenticated endpoints work.
- [Message Queuing, Telegram Batching & Rich Messages](message-queuing.md) — How `display.busy_input_mode` (`interrupt` vs `queue`) controls what happens when the agent is busy, plus Telegram-specific text/media batching and native rich-message rendering.
- [Cloudflare Tunnel](cloudflare-tunnel.md) — How to expose your local Hermes gateway to the internet using Cloudflare Tunnel.
- [1Password CLI](1password-cli.md) — Using 1Password with Hermes Agent for secure credential management.
- [Python Environment with uv + venv](python-uv-venv.md) — How to run Hermes from a uv-managed virtualenv, fix `hermes doctor` warnings about global Python, and make shell `hermes` resolve to the venv.
- [Telegram Mini App Setup](telegram-miniapp-setup.md) — Full walkthrough for deploying the Hermes Telegram Mini App with standalone proxy architecture, Cloudflare Tunnel, auth, and the gateway API.
- [Mini App Standalone Proxy Plan](miniapp-standalone-proxy-plan.md) — Detailed implementation plan for a standalone mini-app proxy that decouples mini-app support from Hermes core.
- [Extending the Gateway API](extending-gateway-api.md) — How to add new endpoints, middleware, and features to the Hermes API server.

## Daily Operator Surfaces

These are worth checking on any serious Hermes setup because they change daily operation, not just setup polish:

- **Context-compression indicator** — watch compression count as a session-length signal; compression keeps work moving but repeated compression means you may need to split or persist context.
- **`/browser connect` / browser automation** — use it when logged-in browser state or real UI interaction matters; prefer deterministic readers/scripts for repeatable extraction when available.
- **Web Dashboard** — use it for administration surfaces like config, sessions, logs, credentials, cron jobs, skills, gateway, webhooks, and memory.
- **Desktop App** — use the native app when a visual/operator workspace is better than pure CLI or chat, especially for streaming tool output, files, profiles, cron, skills, settings, and voice.

Official docs: [Browser Automation](https://hermes-agent.nousresearch.com/docs/user-guide/features/browser), [Web Dashboard](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard), [Desktop App](https://hermes-agent.nousresearch.com/docs/user-guide/desktop), [Context Compression](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching).

## Routing, Models, and Coordination

- [Model Tier Rankings](model-tier-rankings.md) — Best LLM models for Hermes Agent by role (orchestrator, executor, auxiliary), based on community testing and fleet experience.
- [Tiered Model Routing](tiered-model-routing.md) — How to configure delegate-first routing with high-reasoning, budget, specialist, local, and fallback lanes.

## Research, Memory, and Context Extraction

- [Daily Personal-Context Questions](daily-context-questions.md) — A daily Hermes cron pattern that asks one thoughtful personal question per day, files the answer into durable memory, and accumulates rich user context over weeks.
- [Q100 Taste Interview](q100-taste-interview.md) — A 100-question structured extraction protocol for capturing writing voice, taste, aesthetic boundaries, and structural preferences.
- [PageIndex Evaluation](pageindex-evaluation.md) — Technical evaluation of VectifyAI/PageIndex, a reasoning-based RAG system with architecture notes and integration tradeoffs.
- [xurl — X API CLI for Hermes](xurl.md) — How to install, authenticate, configure apps/users/redirects/auth modes, safely verify, and use the official X Developer Platform CLI from Hermes.
- [Lessons Learned](lessons-learned.md) — Hard-won Hermes operating rules: PRD-first workflows, circuit breakers, self-scoring, quality gates, explicit handoffs, and bounded second-order improvement loops.
- [Agent Honesty and Cost Controls](agent-honesty-and-cost-controls.md) — Practical profile/skill/cron guardrails for uncertainty, verify-before-done behavior, hook-style checks, independent verification, push-memory hygiene, and token-cost controls.
- [Agent-Native Toolbelt Implementation Plan](agent-native-toolbelt-plan.md) — Plan for converting repeated agent workflows into deterministic CLIs with local mirrors, budget guardrails, Hermes skills, and compound commands.
- [Signal-to-Skill Report](signal-to-skill-report.md) — Weekly loop for scanning Hermes tool-use patterns, ranking Top 5 reusable workflow candidates, applying them as skills, and verifying the result.

## Operational Monitoring

Hermes runs automated weekly jobs that measure and maintain operational health. Data is collected from:

- `~/.hermes/skill-health.log` — skill health check results
- `~/.hermes/cost-log.csv` — per-task cost tracking (schema defined, auto-population blocked — see Lessons Learned)
- `~/.hermes/daily-logs/failures.md` — documented failures
- `~/.hermes/scorecards/` — historical scorecard results

### Weekly Cron Schedule

| Time | Job | Script |
|------|-----|--------|
| Monday 10:00 | Hermes Scorecard (10 dimensions, 0–5 scoring, max 50) | `~/.hermes/skills/hermes-scorecard/scripts/score.py` |
| Monday 11:00 | Signal-to-Skill Pipeline | `~/.hermes/scripts/signal-to-skill.py` |
| Monday 11:30 | Project Burial Scanner | `~/.hermes/scripts/project-burial-scan.py` |

See [Telegram Mini App Setup](telegram-miniapp-setup.md) for full scorecard documentation. See [Lessons Learned](lessons-learned.md) for the process discipline rules that underpin these systems.

## Contributing

These guides document real setup sessions — errors included. If you hit something not covered here, open an issue or PR.
