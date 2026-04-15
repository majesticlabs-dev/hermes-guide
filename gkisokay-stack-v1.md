# The Gkisokay LLM Model Stack
**Graeme @gkisokay — April 2026**

14 models · 4 tiers · Hermes · OpenClaw · Claude Code · Codex  
A multi-model routing guide for agent builders — local to frontier

---

## Tier 1 — Frontier
> Complex Reasoning · Strategy · Planning · External Dev Only

| Model | Cost (per 1M) | Context | Avg Run Cost | Key Specs | Top Benchmarks | Use It For | Why This Model |
|-------|--------------|---------|--------------|-----------|----------------|------------|----------------|
| **Claude Opus 4.6** *(Anthropic, Feb 2026)* | $5in / $25out *(Fast mode: $30/$150)* | 1M | External dev only. Fast in internal runtime. | Adaptive thinking · Agent Teams · context compaction · 76% long-ctx @ 1M · 4 effort levels | SWE-Verified: **80.8%** / Terminal B2: **65.4%** / ARC-AGI-2: **68.8%** / HLE w/ tools: **53.0%** | Complex external dev via Claude Code · multi-file refactoring · deep agentic terminal coding | #1 agentic terminal coding. Catches own errors earlier over long sessions. API only — no subscription for agents. |
| **GPT-5.4** *(OpenAI, Mar 2026)* ⭐ *Graeme @gkisokay* | $2.50in / $15out *(2x above 272K ctx)* | 1.05M | External Codex dev only. Not in internal runtime. | Absorbs GPT-5.3-Codex · dynamic MCP tool search · steerable computer use · planner + judge in Codex · superhuman desktop control | SWE-Pro: **57.7%** / DSWorld: **75.0%** / GPQA Diamond: **92.8%** | Complex features externally via Codex · not in Hermes/OpenClaw runtime | Multi-hour autonomous execution with real planning. Only worth the premium when task complexity genuinely needs frontier judgment. |
| **GLM-5.1** *(Z.AI, Apr 2026)* | $1.40in / $4.40out *(MIT license · open weights)* | 200K | — | 744B total / 40B active MoE · DSA sparse attention · 8-hour autonomous execution · experiments-analyze-optimize loop · Huawei chips | SWE-Pro: **58.4%** / AIME 2026: **95.3%** / GPQA Diamond: **86.2%** / CyberGym: **68.7%** | Long-horizon agentic coding · sustained optimization loops · complex engineering tasks over hours | #1 SWE-Pro globally. MIT license. Sustains 100s of optimization rounds without human intervention. 3x cheaper than Opus on input. |

---

## Tier 2 — Agent Execution
> Tool Calls · Long Task Chains · Multi-Step Pipelines

| Model | Cost (per 1M) | Context | Avg Run Cost | Key Specs | Top Benchmarks | Use It For | Why This Model |
|-------|--------------|---------|--------------|-----------|----------------|------------|----------------|
| **MiniMax M2.7** *(MiniMax, Mar 2026)* ⭐ *Graeme @gkisokay* | $0.30in / $1.20out *($10/mo = 1500 calls/5h)* | 205K | OpenClaw / Hermes tasks. Not in subconscious loop. | Self-evolving (100+ RL cycles) · 97% skill adherence on 40+ skills · multi-agent teams · 86.2% PinchBench · proprietary API | SWE-Pro: **56.2%** / PinchBench: **86.2%** / Terminal B2: **57.0%** / DDPoI-AA ELO: **1495** | OpenClaw execution backbone · Hermes high-volume agent tasks | Best price-to-agent-capability in the stack. 97% skill adherence critical for OpenClaw ecosystem. $10 plan is absurdly good value. |
| **Kimi K2.5** *(Moonshot, Feb 2026)* | $0.60in / $3.00out *(Watch token verbosity)* | 256K | — | 1T params / 32B active · 384 experts · MLA attention · agent swarm · first open-weight trained for parallel agentic work | HLE w/ tools: **50.2%** / BrowseComp: **78.4%** / SWE-Verified: **76.8%** | Long-horizon task chains · research agents · multi-source browsing | Best long-context stability for extended tool chains. Caution: ~6+ more output tokens than peers — budget carefully. |
| **Grok 4.20** *(xAI, Mar 2026)* | $2.00in / $6.00out | 2M | — | 4-agent parallel system (Grok/Harper/Benjamin/Lucas) · 199 t/s · lowest hallucination rate on market · real-time X data · tool calling + web search | IFBench: **83.0%** / I²-Bench: **97.0%** / Omniscience: **78% ✓** / AI Index: **48/100** | Hallucination-sensitive pipelines · long-context research · multi-agent parallel execution | Lowest hallucination rate of any tested model. 2M context window. Multi-agent API. 199 t/s — fastest in this tier. |
| **DeepSeek V3.2** *(DeepSeek, Dec 2025)* | $0.27in / $0.41out | 164K | — | 685B total / sparse active · DSA sparse attention · long-context + tool-use optimised · open weights · MIT license | SWE-Verified: **70.0%** / Aider polyglot: **74.2%** | Open-weight execution · high-volume coding pipelines · cost-floor frontier reasoning | 90% of GPT-5.4 performance at 1/50th the cost. Best price-performance in this tier via OpenRouter. |

---

## Tier 3 — Balanced
> Content · Code · Research · Day-to-Day Tasks

| Model | Cost (per 1M) | Context | Avg Run Cost | Key Specs | Top Benchmarks | Use It For | Why This Model |
|-------|--------------|---------|--------------|-----------|----------------|------------|----------------|
| **Claude Sonnet 4.6** *(Anthropic, Feb 2026)* | $3in / $15out *(API only)* | 1M | 64k out | 40–60 t/s · adaptive thinking · computer use 94% accuracy · OfficeQA matches Opus · prompt injection resistance on par with Opus | SWE-Verified: **79.6%** / Computer use: **94.0%** / AI Index: **52/100** | Daily coding · content automation · Anthropic ecosystem agents | 98% of Opus coding at 1/5 the cost. API only — no $10/mo plan exists for this model. |
| **GPT-5.4 mini** *(OpenAI, Feb 2026)* ⭐ *Graeme @gkisokay* | $0.75in / $4.50out *(OAuth via subscription)* | 400K | ~$0.075 avg *($0.054 min – $0.288 max / QA: ~$0.072 · LP: ~$0.25)* | 2x faster than GPT-5 mini · native computer use · text + image · function calling · web search · sub-agent optimised | SWE-Pro: **54.4%** / live call: —/ DSWorld: **72.1%** / MCP Atlas: **57.7%** | Hermes conscious layer · debates Qwen ideas · (6–9 turns/run) · builds final output after debate | Smart enough to run your entire system. 93.4% tool-call reliability. ChatGPT OAuth = no API billing needed. |
| **Gemini 3.1 Pro** *(Google, Feb 2026)* | $2in / $12out | 1M | 64k out | Native multimodal (text/image/video/audio) · 3 thinking levels · 136 t/s · single API call for all modalities | ARC-AGI-2: **77.1%** / GPQA Diamond: **94.3%** / SWE-Verified: **80.6%** | Multimodal agents · doc-heavy pipelines · best native video+audio option at Tier 3 | 7.5x cheaper than Opus on input. Strongest multimodal in this tier. Native video+audio = no extra pipeline needed. |
| **Qwen3.6 Plus** *(Alibaba · OpenRouter free tier)* 🟢 **FREE NOW** | $0in / $0out *(Via OpenRouter free tier)* | 1M | — | Hybrid linear attn + sparse MoE · built-in reasoning · major leap over Qwen 3.5 on agentic coding + front-end dev | SWE-Verified: **78.8%** | Agentic coding · front-end dev · any Tier 3 task at $0 | Best free model available. 1M context. Near-frontier coding. Use it now — free until preview window closes. |
| **Llama 4 Maverick** *(Meta, 2026)* | $0.19–$0.49 *($0 self-hosted)* | 1M | — | 400B total / 17B active · open weights · multimodal · Apache 2.0 · data sovereignty · 9–23x price-perf vs GPT-4o | MMLU: **85.5%** / SWE-Verified: **~68%** | Self-hosted Tier 3 · EU/data residency · zero marginal cost at scale | Only serious open-weight option at this level. Self-host = zero ongoing cost. Best for data sovereignty needs. |
| **Mistral Small 4** *(Mistral, Mar 2026)* | $0.15in / $0.60out *(Apache 2.0 · self-hostable)* | 256K | — | 11B total / 6.5B active · 128 experts · unifies Magistral + Pixtral + Devstral · configurable reasoning_effort · 144 t/s · vision | GPQA Diamond: **71.2%** / MMLU-Pro: **78.0%** / LiveCodeBench: **>GPT-OSS** | Budget-conscious agents · long-context research · one model replacing three specialist pipelines | One model replacing Magistral + Pixtral + Devstral. 75% less output verbosity than comparable models = real cost savings. Apache 2.0. |

---

## Tier 4 — Local / Micro
> Summaries · Routing · Classification · Always-On Loops · $0 Cost

| Model | Cost (per 1M) | Context | Avg Run Cost | Key Specs | Top Benchmarks | Use It For | Why This Model |
|-------|--------------|---------|--------------|-----------|----------------|------------|----------------|
| **Qwen3.5-9B** *(Alibaba, Feb 2026)* ⭐ *Graeme @gkisokay* | $0.00 *(Local · LM Studio · ~1M w/ YaRN)* | 262K | $0.00 *(2–6 calls/run · always free)* | Thinking toggle · multimodal (text/image/video) · 201 languages · 5GB VRAM @ 4-bit · tool use native · Qwen-Next architecture | GPQA Diamond: **81.7%** / DocBench: **70.1%** / DocBench: **87.7%** / MMMU-U: **81.2%** | Always-on subconscious ideation loop inside Hermes · curation pass before cloud debate | Only model that costs nothing. Runs 24/7. Beats GPT-OSS-120B at 13x the size. Volume is unlimited. |
| **Qwen3.5-27B** *(Alibaba, Feb 2026)* | $0.00 *(Local · 32GB RAM)* | 262K | $0.00 | 27B dense · multimodal · 201 languages · stronger instruction following than 9B · reasoning by default | GPQA Diamond: **80.6%** / MMLU-Pro: **~75%** / LiveCodeBench: **77.6%** | Better local summarisation · routing decisions · complex micro classification | Step up from 9B with 32GB RAM. Still $0. Noticeably better instruction following for complex micro tasks. |
| **Gemma 4 (31B)** *(Google, Apr 2026)* | $0.00 *(Local · Apache 2.0)* | 256K | — | 31B dense · Apache 2.0 · QAT quant checkpoints · hybrid sliding-window + global attention · #3 Arena open leaderboard | AIME 2026: **89.2%** / LiveCodeBench: **80.0%** / GPQA Diamond: **85.7%** / MMMU-Pro: **76.9%** | Local agentic sub-tasks · commercial deployment · math-heavy reasoning tasks | Dramatic leap over Gemma 3. Apache 2.0 for commercial use. Stronger reasoning than Qwen3.5-27B. Tradeoff: slower inference. |
| **DeepSeek R1 distill** *(DeepSeek, Jan 2026)* | $0.00 *(OpenRouter free tier · 9× $0.27/M via API)* | 128K | $0.00 *(Free tier rate limits apply)* | 32B dense distilled from R1 · Qwen2.5 base · RL chain-of-thought · outperforms o1-mini · MIT license | MATH-500: **94.3%** / AIME 2024: **72.6%** / GPQA Diamond: **62.1%** / SWE-V: **57.2%** | Reasoning-heavy micro tasks · math · logic-based classification · free async fallback | Best reasoning at $0 via OpenRouter. 94.3% MATH-500. MIT license ~5B t/s. |
| **GLM-4.5-Air** *(ZhiPu, 2026)* | Low *(via SiliconFlow API)* | 128K | — | MoE · purpose-built for agent tool use + web browsing · designed from ground up for agentic orchestration · not a trimmed general model | Agent tasks: **#1 OSS** / Tool use: **top tier** / Web browsing: **top OSS** | Lightweight agentic sub-tasks · web browsing agents · tool call routing | Purpose-built for agent tool use from scratch. Best OSS option for this job. Run via SiliconFlow for low-cost access. |

---

## Hermes Routing Policy

**Default lane:** GLM-5.1 (zai/glm-5.1, reasoning_effort=high)  
**Fallback:** GPT-5.4 (openai-codex/gpt-5.4)

| Signal | Route To |
|--------|----------|
| Complex backend, debugging, infra, terminal-heavy | GPT-5.4 |
| Marketing, messaging | GPT-5.4 |
| Budget engineering, repo-wide refactors | MiniMax M2.7 |
| Cheap fast execution, triage, grunt work | MiniMax M2.5 |
| External dev, multi-file refactoring | Claude Opus 4.6 (via Claude Code) |
| Complex features externally | GPT-5.4 (via Codex) |
| Long-horizon agentic coding | GLM-5.1 |
| Daily coding, content automation | Sonnet 4.6 |
| Hermes conscious layer | GPT-5.4 mini |
| Multimodal, doc-heavy | Gemini 3.1 Pro |
| Always-on subconscious loop | Qwen3.5-9B (local) |
| Free-tier tasks | Qwen3.6 Plus / DeepSeek R1 distill |

---

> *Prices as of April 2026 · ⭐ = Active in Graeme's Hermes/OpenClaw stack · Free-tier OpenRouter: 20 req/min, 200 req/day*  
> *Qwen3.5-9B benchmarks from live stack testing · GLM-5.1 benchmarks self-reported by Z.AI (independent verification pending) · Benchmarks from official model cards & Artificial Analysis*

— **Graeme @gkisokay · gkisokay.com**
