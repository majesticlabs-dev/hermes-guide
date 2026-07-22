# Lessons Learned — AGI Experiment Review for Hermes

Hard-won rules from AGI experiment reviews, applied directly to the Hermes agent system. These came from shipping, breaking, and honestly scoring real work.

---

## 1. PRD-Before-Coding for Complex Tasks

Before writing implementation code on any multi-step task, write a one-page PRD (Problem, Approach, Deliverables, Acceptance Criteria). This applies to agent workflows too — if a task involves more than one file or more than two steps, the agent should draft a plan before touching code.

The five minutes spent on the PRD saves hours of rework and scope creep.

---

## 2. Circuit Breaker Rule

**Max 2 retries, then escalate.** If something fails twice in the same way, stop retrying and either:
- Ask the user for clarification
- Switch to a different approach
- Escalate to a stronger model

Blind retries burn tokens, waste time, and produce the same broken output. The circuit breaker exists to prevent the agent from spinning in circles.

---

## 3. Signal-to-Skill Pipeline

Raw signals (logs, errors, metrics) mean nothing until they flow through a pipeline:

1. **Collect** — raw data lands in structured files (e.g., `skill-health.log`, `cost-log.csv`, `failures.md`)
2. **Score** — the scorecard converts raw data into comparable numbers
3. **Trend** — historical scorecard files show direction
4. **Act** — if a dimension drops, investigate and fix

Without the pipeline, you have feelings about quality. With it, you have evidence.

---

## 4. Self-Scoring Beats Self-Congratulation

The first Hermes Scorecard was 13/50. That's not a failure — it's an honest baseline. Most dimensions scored low because the data collection wasn't in place yet. The point of self-scoring isn't to feel good; it's to:

- Establish where you actually are
- Track whether changes move the number
- Catch regressions before they compound

Don't celebrate until scores move. And don't game the scoring — low honesty now beats inflated scores that hide problems.

---

## 5. Document Failures with Same Energy as Wins

Most documentation covers what worked. The failures are where the actual learning lives. The `failures.md` log exists because:

- Future-you (or future-agent) will hit the same wall
- Debugging sessions are cheaper when you can grep for "last time this broke"
- Patterns of failure reveal systemic issues that individual fixes don't

If you spent an hour debugging something, spend five minutes writing down what happened.

---

## 6. Cost Tracking Is a Blind Spot (Backlogged)

Agents spend tokens fast. Without per-task cost logging, you only notice the bill at the end of the month. The `cost-log.csv` file is designed to capture:

- Which task cost what
- Which model was used
- Whether the spend was justified

**Status:** Backlogged as a future improvement. Currently blocked — there is no hook mechanism in the gateway to intercept requests and log costs automatically. The cost-log.csv schema exists, but population requires either a gateway middleware hook or manual discipline, neither of which is wired up yet.

If you don't track cost, you can't optimize routing. And bad routing is the biggest waste in a tiered model system.

---

## 7. Project Burial Rule

**If a project hasn't been touched in 1 week, flag it for archive.** This doesn't mean delete it — it means:

- Move it out of active workspace
- Note what state it's in
- Make it retrievable but not distracting

Active projects deserve attention. Dead projects deserve a clean burial so they stop cluttering the mental model of what's "in progress."

---

## 8. Explicit Handoff Artifacts for Subagent Delegation

When delegating work to a subagent (or a different model tier), don't just say "fix the bug." Provide:

- **Context:** What the system does, what went wrong
- **Constraints:** What not to touch, what the acceptance criteria are
- **Files:** Exact paths to read and modify
- **Verification:** How to check that the fix worked

Vague handoffs produce vague results. Explicit handoffs produce targeted ones.

**Implementation:** The `productivity/handoff-template` skill formalizes this into a reusable 5-section format. When a task is delegated to a subagent, the delegating agent should fill out the handoff template to ensure no context is lost. The skill lives at `~/.hermes/skills/productivity/handoff-template/`.

---

## 9. Quality Gates Over Raw Autonomy

More autonomy is not better autonomy. Quality gates are checkpoints that prevent cascading failures:

- **Syntax check** before declaring a code edit done
- **Test run** before marking a task complete
- **Scorecard** before calling a session successful
- **Cost check** before approving a heavy operation

Each gate catches a class of error that would otherwise propagate. The agent that passes all gates on the first try is better than the agent that moves fastest and breaks things.

---

## 10. Honesty Rules Beat Helpful-Sounding Fabrication

Agents need explicit permission to say “I don’t know.” Without it, they tend to invent paths, commands, APIs, citations, or test results because confident completion looks helpful.

Every serious profile or skill should encode the rule directly:

- Say uncertainty plainly when evidence is missing.
- Do not invent artifact IDs, file paths, citations, or command results.
- Verify before writing and before claiming done.
- If verification fails, report the blocker instead of smoothing over it.

For high-risk claims, use an independent verifier or review profile. A worker summary is not evidence.

---

## 11. Token Waste Starts in Tool Output

Most agent cost waste is not the user’s question. It is oversized context: full logs, broad file dumps, giant database results, repeated source snippets, and irrelevant tool output.

Apply cheap controls before installing another routing layer:

- Search/read targeted ranges instead of dumping whole files.
- Summarize or filter tool output before feeding it back to the model.
- Use budget models for bounded extraction and triage.
- Keep profile skills role-specific so every session does not load a kitchen sink.
- Evaluate compression proxies with small benchmarks before wiring them into live Hermes routing.

Compression tools can be useful, but only after they prove they preserve facts and reduce total cost in the actual workflow.

---

## 12. Add a Second-Order Improvement Loop

A reliable workflow is only the first loop: receive a task, gather context, act, verify, and record the outcome. Mature Hermes operations also need a bounded second loop that improves the first.

Recommended pattern:

1. **Record structured outcomes with reasons.** `failed` or `no reply` is weak evidence; record what failed, why, and which rule, skill, prompt, or tool path was involved.
2. **Keep judgment reviewable.** Put routing rules, acceptance criteria, prompts, and scorecards in small files humans can inspect instead of burying every decision in code or one giant system prompt.
3. **Propose one conceptual change at a time.** This preserves attribution and makes rollback straightforward.
4. **Run a fixed eval before accepting the change.** Include successes, regressions, bad-fit cases, and edge cases. A plausible explanation is not evidence.
5. **Reject flat or worse changes.** Revert them rather than rationalizing them.
6. **Require human review before promotion.** The improver may propose and test; it should not silently promote its own policy changes.
7. **Tune on a cadence, not after every outcome.** Weekly or threshold-based review reduces overfitting to one loud event.

**Why:** without this loop, corrections disappear into session history and the same failures recur. With it, outcomes become small, testable improvements to skills, prompts, evals, or operating rules.

**Guardrail:** do not optimize a convenient proxy such as task count, reply rate, or scorecard points in isolation. Require minimum sample sizes where relevant and pair throughput metrics with quality, downstream value, safety, and regression checks. Never weaken the eval gate with an unconditional success path such as `test_command || true` when later steps depend on that result.

For recurring tool-use patterns, the [Signal-to-Skill Report](signal-to-skill-report.md) is one implementation of this principle. Other workflows can use the same structure without creating a new skill: outcome log → bounded proposal → eval → human review → promotion.

*Pattern adapted from a public article about a self-improving outbound workflow. Specific performance claims were not independently verified.*

---

## The Meta-Lesson

All of these lessons came from running the scorecard honestly and reviewing what went wrong. The scorecard didn't create these insights — the data did. The scorecard just made sure the data was collected and reviewed regularly.

That's the loop: collect, score, review, improve. Everything else follows.

---

## Process Discipline — Hard Rules (soul.md)

The following five hard rules are codified in `~/.hermes/soul.md` under the Process Discipline section. They are not aspirational — they are enforced rules that the agent must follow:

1. **PRD Gate:** No implementation code on multi-step tasks without a one-page PRD first.
2. **Circuit Breaker:** Max 2 retries on the same failure, then escalate or change approach.
3. **Failure Logging:** Every debugging session worth more than 15 minutes must be documented in `daily-logs/failures.md`.
4. **Skill Health Tracking:** All skills must pass health checks. Failures are logged to `skill-health.log` and surfaced in the weekly scorecard.
5. **Handoff Standard:** Every subagent delegation must use the handoff template (5-section format) to prevent context loss.

These rules were added to soul.md so the agent treats them as core behavioral constraints, not optional guidelines.
