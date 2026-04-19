# Hermes Agent — Community Guide

Practical guides for setting up and extending Hermes Agent.

## Guides

- [Telegram Mini App Setup](telegram-miniapp-setup.md) — Full walkthrough for deploying the Hermes Telegram Mini App with standalone proxy architecture (port 8643), Cloudflare Tunnel, Ed25519 auth, and the gateway API. Includes scorecard system documentation, operational scripts, and cron schedule.
- [Mini App Standalone Proxy Plan](miniapp-standalone-proxy-plan.md) — Detailed implementation plan for the standalone mini-app proxy on port 8643 that decouples mini-app support from Hermes core (survives updates).
- [Cloudflare Tunnel](cloudflare-tunnel.md) — How to expose your local Hermes gateway to the internet using Cloudflare Tunnel.
- [1Password CLI](1password-cli.md) — Using 1Password with Hermes Agent for secure credential management.
- [Extending the Gateway API](extending-gateway-api.md) — How to add new endpoints, middleware, and features to the Hermes API server.
- [Multi-Agent Profiles](multi-agent-profiles.md) — How to run multiple specialized Hermes agents with isolated profiles, dedicated SOUL.md files, shared AGENTS.md, and separate invocation via `hermes -p NAME` or wrapper scripts. Includes shared-context pattern (THESIS.md, SIGNALS.md, FEEDBACK-LOG.md) for cross-agent knowledge sharing, plus writer and vpmktg examples.
- [Workspace Auto-Start & Auth](workspace-autostart.md) — How Hermes Workspace auto-starts via launchd after reboot, the gateway/workspace plist arrangement, and the HERMES_API_TOKEN wiring needed to make /jobs and other authenticated endpoints work.
- [Message Queuing & Telegram Batching](message-queuing.md) — How `display.busy_input_mode` (`interrupt` vs `queue`) controls what happens when a new message arrives while the agent is busy, plus Telegram-specific text and media batching windows that coalesce rapid messages.
- [Fizzy Task Coordination](fizzy-task-coordination.md) — How Hermes integrates with Fizzy (fizzy.do) for agent-to-agent task coordination. Covers the Ruby CLI gem, board layout (inbox/doing/blocked/in_review/done/deferred), full card lifecycle, subagent handoff templates, session-start checks, and when to promote tasks to Fizzy vs keeping them on internal todo.
- [Model Tier Rankings](model-tier-rankings.md) — Best LLM models for Hermes Agent by role (orchestrator, executor, auxiliary), based on community testing and fleet experience. Updated April 2026.
- [Lessons Learned](lessons-learned.md) — Hard-won rules from AGI experiment reviews applied to Hermes: PRD-first workflows, circuit breakers, self-scoring, handoff templates, and process discipline hard rules.

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
