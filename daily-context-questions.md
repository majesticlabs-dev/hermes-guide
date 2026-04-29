# Daily Personal-Context Questions

A recurring Hermes pattern: one personal question per day, filed to durable memory on the next run. Over weeks, this daily drip captures small details users never think to volunteer — routines, coffee preference, family context, old hobbies, places lived, sports fandom — building richer context than any one-shot intake interview.

---

## How It Works

1. **AI sends one question** via the user's primary chat channel (Telegram DM, Slack, etc.)
2. **User answers whenever** — typically ~30 seconds, no pressure
3. **Next run, AI processes yesterday's answer**, updates the right file, then asks a new question

The cycle repeats every morning. One question, one answer, filed to the right place, every day.

---

## Why This Beats an Initial Interview

- Users don't know what's worth volunteering. A structured interview captures the obvious stuff. Daily questions catch the long tail of small personal details that turn out to matter.
- Questions are contextual — the AI reads what it already knows and finds gaps. Not generic like "What's your favorite color?" More like "You mentioned your daughter is into stained glass. How'd she get into that?"
- Low effort for the user (~30 sec/day) but compound value over weeks and months.

---

## Setup Recipe

### File Structure

```
~/.hermes/personal-memory/
├── daily-context.md      # Accumulated answers, organized by topic
└── question-log.md       # History: date, question asked, whether answered
```

### Cron Job

Create a Hermes cron job that runs daily (e.g., 9:00 AM local time):

```bash
hermes cron create \
  --name "daily-context-question" \
  --schedule "0 9 * * *" \
  --prompt "Daily personal-context question"
```

### Cron Prompt Template

Use this as the cron job's system prompt:

```
You are running a daily personal-context routine.

1. Read ~/.hermes/personal-memory/daily-context.md and ~/.hermes/personal-memory/question-log.md.
2. Check if yesterday's question was answered. If yes, file the answer into the right section of daily-context.md.
3. Log what happened in question-log.md (date, question, answered yes/no).
4. Find a gap — something you don't know about the user that would be useful context. Look at what's already recorded and pick something adjacent, personal, and specific. NOT generic.
5. Ask exactly one question. Send it to the user via their chat channel.
6. Stop. Do not ask more than one question per day.

Question quality examples:
  GOOD: "You mentioned your daughter is into stained glass. How'd she get into that?"
  GOOD: "What's your morning routine like — coffee, exercise, anything consistent?"
  BAD:  "What's your favorite color?"
  BAD:  "Tell me about yourself."
```

### Model Choice

A local model (e.g., `Carnice-9b`) or the cheapest reliable local executor is sufficient for this task. It's simple read/file/write/ask work — no heavy reasoning needed.

---

## What Gets Captured

After a few weeks of daily questions, the accumulated context typically includes:

- Family details (spouse, kids, parents, siblings)
- Daily routines and habits
- Food and drink preferences
- Hobbies past and present
- Places lived and traveled
- Sports and entertainment fandom
- Work style and communication preferences
- Home and living situation

Topics like health, pets, car, or neighborhood details may come up organically but are boundary-dependent — the routine respects whatever the user volunteers.

None of this comes from one big interview. It comes from showing up every day and asking one good question.

---

## Source

Pattern captured from David's workflow, inspired by Peter Yang / @Shpigford's X post about personal-context AI routines.
