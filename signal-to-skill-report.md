# Signal-to-Skill Report

A Signal-to-Skill Report turns repeated Hermes tool-use patterns into concrete skill candidates. It is a weekly operating loop: scan recent sessions, rank recurring workflows, create or patch the top skills, then verify they load.

## What It Produces

A concise report like this:

```markdown
## Signal-to-Skill Report — Top 5 Candidates

**Scan:** <sessions> sessions · <profiles> profiles · <tool_calls> tool calls · last 14 days

### 1. <Candidate Name>
- **Pattern:** `<tool_a> → <tool_b> → ...` — <plain-English workflow>
- **Occurrences:** <count> across <session_count> sessions
- **Suggested skill:** `<skill-name>`
- **Why it matters:** <why this repeated pattern deserves codification>

...

**Verdict: worth building** — candidates #<n>–#<m> are high-frequency, bounded workflows that would benefit from structured skills.
```

## Weekly Cron Prompt

Use this as the scheduled job prompt:

```text
Run the signal-to-skill pipeline and deliver the top results. Execute: python3 ~/.hermes/scripts/signal-to-skill.py — capture the human-readable summary. In your response, list only the TOP 5 most promising skill candidates with their pattern description, occurrence count, and a suggested skill name. Skip trivial patterns like standalone terminal or read_file calls. Focus on multi-tool combinations that represent real workflows worth automating. End with a one-line verdict: "worth building" or "nothing actionable this week."
```

Recommended schedule:

```bash
hermes cron create '0 11 * * 1' \
  --name 'Signal-to-Skill Weekly Scan' \
  --model '<cheap-reliable-model>'
```

If using the Hermes `cronjob` tool, set:

```yaml
name: Signal-to-Skill Weekly Scan
schedule: 0 11 * * 1
prompt: <weekly cron prompt above>
model: <cheap reliable model for summarization>
deliver: origin
```

## Pipeline Requirements

The script should inspect recent Hermes session data and count ordered tool sequences.

Minimum behavior:

1. Read the last 14 days of sessions from:
   - `~/.hermes/state.db`
   - `~/.hermes/profiles/*/state.db`
   - legacy `~/.hermes/sessions/*.json*` if present
2. Extract assistant `tool_calls` in chronological order.
3. Count repeated adjacent tool combinations, ideally windows of 2–4 tools.
4. Track both total occurrences and distinct session spread.
5. Exclude low-value standalone patterns.
6. Save raw JSON under:
   - `~/.hermes/scorecards/signal-reports/`
7. Print a human-readable Top 5 report.

## Candidate Selection Rules

A candidate is worth building when it is:

- **Frequent** — appears across many sessions, not one long loop.
- **Bounded** — has a clear trigger and finish condition.
- **Procedural** — can be written as steps, checks, pitfalls, and verification.
- **Non-trivial** — more than a single obvious tool call.
- **Reusable** — useful across profiles/projects.

Skip or de-prioritize:

- Single tools such as `terminal`, `read_file`, or `web_search` alone.
- Patterns already covered by a good skill unless the report shows friction.
- Patterns caused by one broken task retrying endlessly.
- Broad concepts that are better handled by profile rules or Kanban.

## Apply-All Procedure

When David says “apply all” to a Signal-to-Skill Report:

1. **Recover the exact report**
   - Read the cron output, not the abbreviated Telegram preview.
   - Example path: `~/.hermes/profiles/<profile>/cron/output/<job_id>/<date>.md`.
2. **Check existing skills first**
   - Use `skills_list` and targeted file search for duplicate names/concepts.
3. **Create or patch skills**
   - Create missing skills with the suggested names when they are clear.
   - Patch existing skills when the concept already exists but needs hardening.
4. **Keep skills compact**
   - Trigger, workflow, pitfalls, verification checklist.
   - Do not create giant doctrine from a simple repeated loop.
5. **Verify each skill**
   - Load with `skill_view` or verify via `hermes skills list` in the target profile.
6. **Update the Guide if the operating loop itself improved**
   - Add public-safe process lessons here, not private task transcripts.

## Skill Template for Candidates

```markdown
---
name: <skill-name>
description: Use when <trigger>. <one-sentence behavior and outcome>.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [<tag>, <tag>]
    related_skills: [<existing-skill>]
---

# <Title>

## Overview

<What the workflow is and why it exists.>

## When to Use

- <trigger>
- <trigger>

Do not use this when <counter-trigger>.

## Workflow

1. <step>
2. <step>
3. <step>

## Pitfalls

- <mistake and fix>
- <mistake and fix>

## Verification Checklist

- [ ] <evidence>
- [ ] <evidence>
- [ ] <evidence>
```

## Example: June 22, 2026 Apply-All

Report scan: 733 sessions · 12 profiles · 8,461 tool calls · last 14 days.

Created skills:

| Candidate | Pattern | Skill |
|---|---|---|
| Codebase Reconnaissance | `search_files → read_file → read_file → search_files` | `codebase-recon` |
| Test-Then-Fix Loop | `terminal → read_file → patch → terminal` | `test-fix-loop` |
| Run-Then-Inspect Debugging | `terminal → read_file/terminal` | `run-inspect-debug` |
| Search-Then-Read Investigation | `search_files → read_file` | `search-read-investigation` |
| Write-Then-Verify Pattern | `write_file → terminal/execute_code` | `write-verify` |

The important move is not the report; it is the follow-through. A weekly report that keeps naming the same workflows without creating or improving skills is just operational theater.

## Verification Checklist

- [ ] Cron job exists and runs weekly on a cheap reliable model.
- [ ] Pipeline scans all relevant profiles, not only default sessions.
- [ ] Raw JSON reports are saved for audit.
- [ ] Delivered report includes only the Top 5 and a verdict.
- [ ] “Apply all” results in real skill creation/patches.
- [ ] Created skills are verified with `skill_view` or `hermes skills list`.
