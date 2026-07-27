# Recommended skills to install

Use this as the lightweight shopping list for third-party or profile-local skills worth installing on Hermes profiles. Install by role, not everywhere.

## First principle: improve the operating system before adding skills

Do not install community skills by default. Treat external skill lists as prompts to inspect local failure modes first.

Use this order:

1. **Patch existing local behavior first** — if the agent already has an umbrella skill, profile rule, guide page, or workflow note that covers the behavior, update that artifact instead of adding another skill.
2. **Encode repeated reminders** — if a human has to remind an agent about the same behavior twice, convert that reminder into a skill/profile rule or Hermes Guide improvement.
3. **Install only for role fit** — add third-party skills only to profiles that actually own the workflow.
4. **Verify before trusting** — inspect the skill contents, scripts, dependencies, permissions, and cost/tool implications before installing it.
5. **Keep Hermes Guide current** — when a broadly useful Hermes operating lesson is relevant to this setup, add the public-safe version to this Guide automatically, unless it touches runtime config, auth, cost/routing, external publishing, or private data.

The goal is compounding behavior, not a larger skill pile.

## Quality bar for reusable or public skills

A useful `SKILL.md` is not automatically a production-ready skill. Before recommending, publishing, or broadly installing a skill, verify that it is:

- **Precisely triggered** — documents both when to load and when not to load.
- **Operational** — changes the agent's process with ordered steps, checkable completion criteria, failure boundaries, and an explicit safety/approval contract.
- **Reproducible** — includes realistic success and boundary/failure examples, with every referenced script, template, and file present.
- **Tested at the contract level** — covers a positive trigger, negative trigger, expected workflow behavior, safety boundary, and any known regression. Structural tests prove packaging consistency; they do not prove real-world outcomes.
- **Public-safe** — contains no credentials, private paths, memories, sessions, databases, logs, caches, customer data, or generated artifacts.
- **Versioned and attributable** — keeps source/license attribution, a clear version, and a machine-readable catalog when publishing a multi-skill repository.

For a public skill repository, add deterministic validation for frontmatter/catalog agreement, JSON validity, local Markdown links, supporting-file existence, common secret patterns, and prohibited private state. Keep bulky references behind progressive-disclosure links instead of inflating every `SKILL.md`.

Treat stars, download counts, and passing structural checks as signals—not evidence that a workflow is safe, useful, or genuinely field-tested. Adapt the narrow rules that improve the local owner skill; do not install an overlapping bundle just because it is well packaged.

Pattern adapted from [Hermes Field Kit](https://github.com/asimons81/hermes-field-kit), Apache-2.0.

## Current recommendations

### `last30days`

- **Source:** https://github.com/mvanhorn/last30days-skill
- **Purpose:** Recent community and market research across Reddit, X, YouTube, TikTok, Hacker News, Polymarket, GitHub, and the web.
- **Install on:** `researcher`, `grok`, `product-strategist`, `steph`.
- **Optional later:** `bob`, only if CTO/product-tech evaluations repeatedly need recent community sentiment.
- **Do not install by default:** implementation/review/ops profiles such as `engineer`, `leon`, `ziv`, `reviewer`, `devops`, plus narrative/curation/dispatch profiles such as `james`, `kb-curator`, `librarian`, `janitor`, `orchestrator`.
- **Install note:** this skill has scripts and support files under `skills/last30days/`; do not install only the raw `SKILL.md` if the runtime expects the engine files. Copy or install the full skill directory into each target profile's local `skills/research/last30days/`.
- **Cost note:** preserve the Hermes Perplexity approval gate. Do not enable or call Perplexity-backed paths unless David explicitly approves Perplexity for that task.

## Profile-selection rule

- Research and market-signal skills belong on research/product/marketing profiles.
- Coding, review, DevOps, and janitor profiles should receive these only when their role actually owns that workflow.
- If a profile has `.no-bundled-skills`, install under that profile's own `~/.hermes/profiles/<profile>/skills/` tree and verify with `hermes -p <profile> skills list`.
