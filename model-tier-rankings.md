# Historical Model Evaluations — April 2026

> **Archived snapshot, not current setup advice.** The rankings, prices, provider availability, and community anecdotes below were recorded in April 2026 and have not been re-benchmarked. They are not Hermes defaults, a current price list, or the operator's active model policy. Capability guarantees and vendor-regression theories are unverified community opinions. Use [current routing guidance](tiered-model-routing.md) and official provider information before making a model or cost decision.

**Last updated:** April 2026  
**Source:** BoxminingAI community testing (1 month real-world Hermes usage), plus local fleet experience.

Models ranked by role in a multi-agent setup — not raw benchmarks. Role specialization matters more than general chat performance.

---

## Tier Definitions

| Role | Purpose | Key Traits |
|------|---------|------------|
| **Orchestrator** | The brain — reasoning, planning, critical thinking | Strong chain-of-thought, consistency across long sessions, context recovery |
| **Executor** | Efficient task execution — coding, debugging, tool use | Fast, reliable tool calling, agentic-native, stays on task |
| **Auxiliary** | Support role — niche tasks, not everyday driver | Good at one specific thing (search grounding, image analysis, HTML one-shots) |

---

## Orchestrators (Brain)

### GPT-5.4 — "The New King"
The strongest overall brain model. Native agentic design pairs powerfully with Hermes routing. Good at everything — planning, reasoning, multi-turn consistency. If you're picking one model and one model only, this is it.

**Use for:** Primary orchestrator, complex problem-solving, backend/debug escalation.

### Qwen 3.6 Plus — "Always-On Reasoning"
Unique always-on reasoning trace (no toggle). Preserves chain-of-thought across *all* prior turns in a session, not just the current one. Produces fewer contradictions in long-horizon tasks. Great starting model for new Hermes setups.

**Use for:** Fallback orchestrator, long-horizon agentic tasks, planning sessions where consistency across 10+ turns matters.

### Kimi K2.5 — "Swarm Orchestrator"
Versatile enough to be both orchestrator and executor. Native image input for front-end/UI reasoning. Swarm agents feature — up to 100 parallel sub-agents, 1500 tool calls without predefined workflow. Compound effect inside Hermes for research and extraction workflows.

**Use for:** Front-end generation from screenshots, research workflows needing parallel sub-agents, UI tasks.

**Caveat:** Managing the swarm effectively requires clear communication with your Hermes agent setup.

### Gemini 3.1 Pro Preview — "Multimodal Brain"
Strongest Google model. Native video and audio input. Good for extracting structured data from visual dashboards, processing audio instructions, screen recording analysis. Not quite as smart as GPT-5.4 but multimodal edge is real.

**Use for:** Visual data extraction, audio/video processing tasks, multimodal reasoning.

---

## Executors (Worker / Subagent)

### MiMo-V2-Pro — "High Volume King"
Most-used model on OpenRouter for Hermes this month. Purpose-trained for agentic use cases — tool calls integrate cleanly with Hermes skill registry. Great for testing and building skill workflows. Skills generated carry over to future sessions and survive model switches.

**Pricing:** $1/M input, $3/M output (OpenRouter)

**Use for:** High-volume document processing, long agentic workflows (hours), skill building and testing, low-cost execution.

### MiniMax M2.7 — "Agentic-Native Executor"
Trained on the OpenClaw agent harness framework (same lineage as Hermes). Thinks in agentic terms natively — no heavy system prompt scaffolding needed. Official partner with Nous Research. Give it a plan and it executes. Don't ask it to plan.

**Use for:** Budget engineering, repo-wide refactors, predefined task execution.

**Caveat:** Do NOT use as orchestrator. Give it a plan, not a blank slate.

### GLM 5.1 — "Context Recovery Champion"
The original evaluation reported strong coding and post-compaction context recovery, including a successful game-generation anecdote. These are historical observations, not a controlled comparison. Hermes configuration is profile-local `config.yaml`; it does not provide an automatic fleet-wide model switch.

**Use for:** Complex coding, server fleet management, long sessions with compaction, default fleet executor.

### Nemotron 3 Super — "Open-Weight Dev Specialist"
The original evaluation recommended it for coding agents and self-hosted workflows. Self-hosting can avoid a hosted provider's quota, but does not remove hardware limits, tool/network exposure, or data privacy risks.

**Use for:** Pro developers who know what they're doing, self-hosted pipelines, privacy-sensitive workflows.

**Caveat:** Not for beginners. Requires developer expertise.

### Step 3.5 Flash — "RL Synergy"
Natively integrates a scalable RL framework. When paired with Hermes' Atropos RL training environment, can act as both the acting agent *and* improve over a rollout. Open-source.

**Use for:** RL self-improvement loops, agent training rollouts.

### DeepSeek V3.2 — "Cron Reliability Specialist" 💰
The original evaluation recommended reasoning/tool-use behavior for cron workflows. No model eliminates tool-call errors; scheduled work still requires task-level execution and delivery verification.

**Pricing:** $0.259/M input, $0.42/M output (OpenRouter) — cheapest executor by far.

**Use for:** Cron jobs, daily automations, scheduled tasks, cost-optimized tool-heavy workflows.

### GPT-5.4 Mini — "Narrow Sub-Agent"
Great for sub-agent parallel task execution with narrow, well-defined scopes. NOT suitable as a primary model or solo driver.

**Use for:** Defined sub-agent roles in multi-agent pipelines.

**⚠️ Cost trap:** In some configurations, requests silently fall back to full GPT-5.4 even when Mini is configured. Check your logs. Unexpectedly ramps costs.

---

## Auxiliary (Support)

### Gemini 3 Flash Preview — "Built-In Search Grounding"
Built-in Google search grounding and URL context reading. Hermes agent can cite live web data without a separate browsing tool. Good alternative if you don't want the Nous Research $10/mo subscription for built-in tools.

**Use for:** Web grounding, search-augmented tasks, citation-heavy workflows.

### Gemini 2.5 Flash — "Default Auxiliary"
This was described as the auxiliary default in the original notes. It is not a current universal default. Current Hermes resolves auxiliary providers/models by task, configuration, credentials, and fallback policy; inspect the selected version rather than assuming Gemini 2.5 Flash is in use. See [Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration).

**Use for:** Side tasks, image/screenshot analysis, web page summarization.

### Trinity Large Preview Free — "Long Tool Chain Support"
Open-weight, natively agent-tuned. Handles complex tool chains with long prompts (50-60 tool calls per session). Good at reasoning-heavy workloads (math, coding, multi-step agent workflows). Surprisingly decent at creative writing/storytelling. Free on OpenRouter.

**Use for:** Low-stakes support tasks in multi-agent setups, long tool-chain sessions, academic/writing tasks.

### MiMo-V2-Flash — "HTML One-Shot"
Lightweight version of MiMo-V2-Pro. Only genuinely good at one thing: making HTML webpages in one shot. Hybrid thinking toggle (thinking vs instant answer) shows no meaningful difference.

**Use for:** Quick HTML page generation. Nothing else.

---

## Avoid / Uncertain

### Claude Opus 4.6 & 4.7 — "Nerfed"
Severe regressions. Failed simple one-shot prompts (couldn't build a basic video game). Intelligence may be redirected to training Mythos (next super-model). Could theoretically function as orchestrator if you can get past the nerfs, but situation is murky as of April 2026.

**Status:** Avoid until Anthropic reverses the regression or ships Mythos.

### Claude Sonnet 4.6 — "Regressed"
Clear regression in structured task execution. Failed to follow explicit instructions for HTML presentation generation that previously worked. Ignores system prompt skill instructions.

**Status:** Avoid for agentic workflows.

### MiniMax M2.5 — "Superseded"
M2.7 exists. No reason to use M2.5 anymore.

**Status:** Use M2.7 instead.

---

## Quick Reference: Role → Model

| Role | Primary Pick | Budget Pick | Specialty Pick |
|------|-------------|-------------|----------------|
| Orchestrator | GPT-5.4 | Qwen 3.6 Plus | Kimi K2.5 (swarm), Gemini 3.1 Pro Preview (multimodal) |
| Executor | GLM 5.1 | DeepSeek V3.2 ($0.26/$0.42) | MiMo-V2-Pro ($1/$3), M2.7 (agentic-native) |
| Auxiliary | Gemini 2.5 Flash (historical default claim) | Gemini 3 Flash Preview (search) | MiMo-V2-Flash (HTML only) |

---

## Historical Fleet Routing (Retired)

The following table records one former deployment's choices. None of its lanes, subscription names, local endpoints, or fallback rules should be installed or applied as current policy.

|| Lane | Model | Role |
|------|-------|------|
| Architect / primary orchestrator | GPT-5.4 | Plans, reasons, decides |
| Fallback orchestrator | Qwen 3.6 Plus (OpenRouter, fallback-only) | Takes over when 5.4 is unavailable or cost-optimized pass |
| Long-horizon coding / execution / fleet | GLM-5.1 | Context recovery, complex coding |
| Budget engineering / refactors | MiniMax-M2.7 | Agentic-native executor with a plan |
| Cheap fast / triage / grunt work | MiMo-V2-Pro | Low-cost high volume |
| Cron / scheduled automation | DeepSeek V3.2 | Thinks inside tool calls, cheapest executor |
| Research / parallel sub-agents | Kimi K2.5 (direct API) | Swarm agents, screenshot→UI |
| Local / simple tasks | local-carnice (Carnice-9b) | Short queries, cheap local inference on localhost:8080 |
| Medium-cost general (narrow scope only) | GPT-5.4-mini | ⚠️ Cost trap — sub-agent roles only |
| Marketing / messaging | GPT-5.4 | Quality writing |
| Ideation deepening | MiniMax-M2.7 | Volume ideation |

> **Historical fleet constraint only:** this deployment used OpenRouter as fallback. Upstream Hermes supports OpenRouter as a primary provider too; this is not a product restriction.

---

## Hot-Swapping Models

Since Hermes v0.8, you can hot-swap models mid-session:

- Type `/model` in Discord, Telegram, or any active session
- Session continues with the new model — no restart needed
- Typical pattern: start with orchestrator for planning, swap to executor for implementation

`/model` changes the active session's model; profile defaults and separately pinned jobs must be handled explicitly. Editing one profile's `config.yaml` does not update every profile. See [Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) and [Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron).

---

## Original Evaluation Conclusions (Historical, Not Recommendations)

1. **No single best model** — role-based routing beats raw benchmarks every time.
2. **Chinese models dominate agentic roles** — Qwen, GLM, Kimi, MiMo, DeepSeek, MiniMax all outperform Western counterparts for specific roles, often at better value.
3. **GPT-5.4 is the overall king** but expensive — pair with budget executors for cost efficiency.
4. **Claude has regressed badly** — avoid for agentic workflows until further notice.
5. **Context recovery matters** — GLM-5.1's post-compaction recovery is a real operational advantage.
6. **Tool-call reasoning was valued** — reliability still needs measured workflow evidence; it never eliminates cron errors.
7. **DeepSeek V3.2 is the cost king** — $0.259/$0.42 makes it the cheapest reliable executor on the market.

---

*Sources: BoxminingAI "Top AI Models for Hermes Agent" (April 17, 2026), local fleet experience, OpenRouter pricing. Update quarterly or after major model releases.*
