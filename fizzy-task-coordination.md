# Fizzy Task Coordination — Agent-to-Agent Task Board

How Hermes integrates with Fizzy (fizzy.do) for multi-agent task coordination. This guide covers the CLI, the board layout, the lifecycle, and the coordination patterns that keep a fleet of agents and a human on the same page.

---

## What Fizzy Is

Fizzy is a project management tool at [fizzy.do](https://fizzy.do) with a Ruby CLI gem (`fizzy`) that AI agents use to track tasks on a shared board. It's the coordination layer when Hermes spins up subagents and needs to track who's doing what, what's blocked, and what's done.

---

## Setup

**Install the gem:**

```bash
gem install fizzy-cli
```

Source: the Fizzy CLI repository provided by your Fizzy workspace.

**Auth:** Token stored in 1Password. The CLI reads it from your 1Password vault at runtime. Make sure your 1Password CLI is unlocked before running fizzy commands.

**Verify it works:**

```bash
fizzy boards list
```

---

## Board Layout

A Fizzy board has 4 real columns and 2 card states. Tasks flow left-to-right through columns, then close or defer:

| Position | Name | Type | Meaning |
|----------|------|------|---------|
| 1 | `inbox` | Column | New tasks, not yet picked up |
| 2 | `doing` | Column | Agent actively working |
| 3 | `blocked` | Column | Needs human input or decision |
| 4 | `in_review` | Column | Work complete, awaiting human review |
| — | `done` | Card state | Completed — `fizzy cards close NUMBER` |
| — | `deferred` | Card state | Postponed — `fizzy cards not-now NUMBER` |

Get your column IDs once and reuse them:

```bash
fizzy columns list --board BOARD_ID
```

---

## Core Lifecycle — CLI Commands

### Create a card

```bash
fizzy cards create "Fix auth token refresh" --board BOARD_ID --body "See session-start check section below for body template" --json
```

Returns the card number. You'll use this number for everything else.

### Assign to an agent

```bash
fizzy cards assign NUMBER USER_ID
```

### Move to a column (triage)

```bash
fizzy cards triage NUMBER --column DOING_COLUMN_ID
```

### Add steps (subtasks)

```bash
fizzy steps create "Investigate token expiry logic" --card NUMBER
```

### Complete a step

```bash
fizzy steps update STEP_ID --card NUMBER --completed
```

### Comment (audit trail)

```bash
fizzy comments create "Token refresh reproduces on staging. Root cause: expired client secret." --card NUMBER
```

### Close (done)

```bash
fizzy cards close NUMBER
```

### Defer (not now)

```bash
fizzy cards not-now NUMBER
```

---

## The Gotcha: Steps API Field Name

Before v0.6.2, `fizzy steps create` used a `description` field internally. The API expects `content`. If you're on an older gem, step creation returns a **400 Bad Request** with no useful error message.

**Fix:** Upgrade to v0.6.2+.

```bash
gem update fizzy-cli
```

If you can't upgrade, the workaround is to skip steps and use comments for subtask tracking.

---

## Agent Coordination Pattern

The coordination model is orchestrator → subagent → human review:

### 1. Orchestrator creates and delegates

The orchestrator agent creates a card, assigns it to a subagent, and triages it to `doing`:

```bash
fizzy cards create "Implement rate limiter for gateway API" \
  --board BOARD_ID \
  --body "Goal: Add rate limiting to all /api/ endpoints
Context: Production gateway is unprotected, need 100 req/min per token
Files: ~/.hermes/gateway/server.rb, ~/.hermes/gateway/middleware/
Acceptance: 429 response after limit exceeded, headers include X-RateLimit-Remaining" \
  --json

fizzy cards assign 47 agent-worker-03
fizzy cards triage 47 --column DOING_COLUMN_ID
```

### 2. Subagent works the card

The subagent adds steps, comments progress, and moves the card through the lifecycle:

```bash
# Break it down
fizzy steps create "Add Rack::Attack gem" --card 47
fizzy steps create "Configure throttle rules" --card 47
fizzy steps create "Write tests for 429 response" --card 47

# Work through steps
fizzy steps update STEP_ID --card 47 --completed
fizzy comments create "Rack::Attack configured. 100 req/min throttle per API token." --card 47
```

### 3. Blocked → escalate to human

When the subagent hits a wall, it moves the card to `blocked`, assigns it to the human, and leaves a comment:

```bash
fizzy cards triage 47 --column BLOCKED_COLUMN_ID
fizzy cards assign 47 HUMAN_USER_ID
fizzy comments create "BLOCKED: Rack::Attack needs Redis for distributed rate limiting. Should I use the existing Redis instance or spin up a separate one?" --card 47
```

### 4. Done → send for review

When work is complete, move to `in_review` and assign to the human:

```bash
fizzy cards triage 47 --column IN_REVIEW_COLUMN_ID
fizzy cards assign 47 HUMAN_USER_ID
fizzy comments create "DONE: Rate limiter live on staging. All tests passing. Ready for review." --card 47
```

### 5. Human reviews and closes

The human (or whoever owns the board) reviews and either closes the card or sends it back:

```bash
fizzy cards close 47
# or send back:
fizzy cards triage 47 --column DOING_COLUMN_ID
fizzy cards assign 47 agent-worker-03
fizzy comments create "Needs adjustment: throttle should be 200 req/min for premium tokens." --card 47
```

---

## Subagent Handoff Template

When the orchestrator delegates a task to a subagent via `delegate_task`, include this context string so the subagent knows how to update the board:

```
Fizzy card #NUMBER tracks this task. Use fizzy CLI to update:
- Move to doing: fizzy cards triage NUMBER --column DOING_COLUMN_ID
- Add steps: fizzy steps create "desc" --card NUMBER
- Comment progress: fizzy comments create "update" --card NUMBER
- If blocked: fizzy cards triage NUMBER --column BLOCKED_COLUMN_ID
  fizzy cards assign NUMBER HUMAN_USER_ID
  fizzy comments create "BLOCKED: reason" --card NUMBER
- When done: fizzy cards triage NUMBER --column IN_REVIEW_COLUMN_ID
  fizzy cards assign NUMBER HUMAN_USER_ID
```

Replace `NUMBER`, column IDs, and `HUMAN_USER_ID` with actual values.

---

## Session-Start Check

At the beginning of every session, check the `blocked` and `in_review` columns for actionable items. Three CLI calls keep nothing slipping through cracks:

```bash
# 1. Check blocked cards assigned to you (the human)
fizzy cards list --board BOARD_ID --column BLOCKED_COLUMN_ID --json

# 2. Check in_review cards awaiting your approval
fizzy cards list --board BOARD_ID --column IN_REVIEW_COLUMN_ID --json

# 3. Refresh the local board cache in case other agents moved things
fizzy boards sync
```

Any cards returned need a decision: unblock, approve, send back, or defer.

---

## When to Use Fizzy vs Internal Todo

Not every task belongs on Fizzy. Use this decision matrix:

| Condition | Use Fizzy | Use Internal Todo |
|-----------|-----------|-------------------|
| Multi-step (3+ subtasks) | Yes | No |
| Spans multiple sessions | Yes | No |
| Involves multiple agents | Yes | No |
| Might get blocked on human input | Yes | No |
| Uncertain failure modes | Yes | No |
| Quick one-shot task | No | Yes |
| Single agent, single session | No | Yes |
| Purely internal to one agent | No | Yes |

Promote to Fizzy when coordination overhead is justified. Keep it on internal todo for fast, local work.

---

## Card Body Template

Every card should have a structured body. Use this template:

```
Goal: [One sentence — what this task accomplishes]
Context: [Why this task exists, what triggered it]
Files: [Paths to relevant files, separated by commas]
Acceptance: [What done looks like — testable criteria]
```

**Example:**

```
Goal: Add rate limiting to the gateway API
Context: Production gateway has no request throttling. High-volume users can monopolize resources.
Files: ~/.hermes/gateway/server.rb, ~/.hermes/gateway/middleware/rate_limit.rb
Acceptance: 429 response after 100 req/min per token, X-RateLimit-Remaining header present, tests pass
```

---

## Important Details

- **Cards use numbers** (integers like `47`). Everything else uses IDs (base36 UUIDs like `abc123xyz`). Don't mix them up.
- **Use `--json`** for any command you're parsing programmatically. The human-readable output format can change.
- **`fizzy cards get NUMBER --json`** returns steps inline — you don't need a separate call to list steps.
- **Comments are the audit trail.** If it happened and isn't commented, it didn't happen.
- **`fizzy boards sync`** refreshes the local board cache. Run it when you suspect stale data.
- **`fizzy columns list --board BOARD_ID`** shows column IDs. You need these for triage commands.

---

## Full Workflow Example

Putting it all together — a complete task lifecycle from creation to close:

```bash
# 1. Orchestrator creates the task
fizzy cards create "Add health check endpoint to gateway" \
  --board b7k9x2 \
  --body "Goal: Add /health endpoint returning 200
Context: Load balancer needs a health check target
Files: ~/.hermes/gateway/server.rb
Acceptance: GET /health returns 200, no auth required" \
  --json
# Returns card #52

# 2. Assign to subagent and start work
fizzy cards assign 52 agent-worker-03
fizzy cards triage 52 --column doing_col_id

# 3. Subagent adds steps
fizzy steps create "Add GET /health route" --card 52
fizzy steps create "Add integration test" --card 52

# 4. Subagent works and comments
fizzy steps update step_abc123 --card 52 --completed
fizzy comments create "Route added. Working on test." --card 52

# 5. Subagent finishes
fizzy steps update step_def456 --card 52 --completed
fizzy comments create "All done. Test passing. /health returns 200 unauthenticated." --card 52
fizzy cards triage 52 --column in_review_col_id
fizzy cards assign 52 HUMAN_USER_ID

# 6. Human reviews at session start
fizzy cards list --board b7k9x2 --column in_review_col_id --json
# Sees card #52, reviews the code, approves

# 7. Close
fizzy cards close 52
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `fizzy steps create` returns 400 Bad Request | Old gem using `description` field | `gem update fizzy-cli` (need v0.6.2+) |
| Auth error on every command | 1Password CLI locked or token missing | Unlock 1Password, verify token in vault |
| Stale card list | Local cache out of date | `fizzy boards sync` |
| Column not found | Wrong column ID | `fizzy columns list --board BOARD_ID` to get current IDs |
| Card not found | Using an ID instead of a number | Cards are referenced by integer number, not UUID |

---

*Last updated: April 2026. Fizzy CLI v0.6.2+.*
