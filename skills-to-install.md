# Recommended skills to install

Use this as the lightweight shopping list for third-party or profile-local skills worth installing on Hermes profiles. Install by role, not everywhere.

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
