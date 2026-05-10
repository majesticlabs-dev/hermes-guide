# Q100 Taste Interview — Structured Voice Extraction Protocol

A 100-question protocol for extracting a person's writing taste, voice, aesthetic boundaries, and structural preferences. Run as a structured interview or drip-fed over sessions. The output is a reusable voice/taste profile that any AI agent can load to match the subject's style exactly.

---

## Overview

The Q100 is designed to produce a complete voice profile — not just "how do you sound" but the deeper layer of *what you would never do*, *what you secretly admire*, and *what patterns you fall back on under pressure*. It is deliberately adversarial: the questions are sharp, specific, and designed to surface real preferences instead of polite non-answers.

**7 categories, 100 questions total:**

| Category | Count | Purpose |
|----------|-------|---------|
| Beliefs & worldview | 15 | Core values, taste philosophy, what matters |
| Writing mechanics | 20 | Sentence-level habits, diction, rhythm choices |
| Aesthetic crimes | 15 | Lines that should never be crossed |
| Voice & personality | 15 | Tone, humor, warmth, edge, persona boundaries |
| Structure & format | 15 | How the subject organizes and presents ideas |
| Hard nos | 10 | Absolute dealbreakers — things they will never accept |
| Red flags | 10 | Patterns that signal "this is wrong" immediately |

---

## Hard Rules

1. **All 100 questions must be answered.** Skipping is not allowed. "I don't know" is a valid answer — record it as-is.
2. **No hedging.** "It depends" is not an answer. Pick a side. If both sides are true, explain the contradiction.
3. **Speed matters more than depth on first pass.** Gut answers are more honest than reasoned ones. Go fast, then refine.
4. **Record the exact words.** Paraphrasing destroys the signal. Quote the subject directly.
5. **If the subject contradicts themselves, flag it but don't resolve it.** Contradictions are data.
6. **No coaching.** The interviewer does not suggest answers, give examples, or lead.
7. **Stop after 45 minutes.** Fatigue answers are unreliable. Resume in the next session.

---

## Output Profile Schema

After the interview is complete, compile answers into this profile structure:

```yaml
# Voice/Taste Profile — [Subject Name / Identifier]
# Generated from Q100 Taste Interview

voice:
  tone: ""                    # e.g., "dry, direct, slightly irreverent"
  humor: ""                   # e.g., "deadpan, sparing, never slapstick"
  warmth: ""                  # e.g., "low — professionalism over friendliness"
  edge: ""                    # e.g., "sharp but controlled, never cruel"
  persona: ""                 # e.g., "experienced operator, no-nonsense"

mechanics:
  sentence_length: ""         # e.g., "short-to-medium, punchy"
  diction: ""                 # e.g., "plain English, avoids jargon"
  rhythm: ""                  # e.g., "staccato statements, sparse commas"
  punctuation: ""             # e.g., "em dashes over semicolons, no exclamation marks"

structure:
  format_preference: ""       # e.g., "bullet lists, no long paragraphs"
  opening_style: ""           # e.g., "lead with the conclusion"
  closing_style: ""           # e.g., "single-line takeaway, no fluff"
  section_breaks: ""          # e.g., "headers only when section > 3 paragraphs"

beliefs:
  core_values: []             # e.g., ["clarity", "brevity", "honesty"]
  taste_philosophy: ""        # e.g., "if it can be cut, cut it"

boundaries:
  hard_nos: []                # e.g., ["no emoji in professional writing"]
  aesthetic_crimes: []        # e.g., ["stock photo aesthetics"]
  red_flags: []               # e.g., ["hedging language in claims"]
```

---

## Storage Template

```
~/.hermes/voice-profiles/
├── <subject-name>-taste-profile.md      # Compiled profile (schema above)
├── <subject-name>-raw-answers.md        # Raw Q&A transcript
└── <subject-name>-contradictions.md     # Flagged contradictions for review
```

---

## The 100 Questions

### Category 1: Beliefs & Worldview (15 questions)

1. What is the single most important quality in writing — the one thing you cannot forgive the absence of?
2. Name a writer whose work you admire but would never try to imitate. Why?
3. What belief about communication do you hold that most people would disagree with?
4. If you could eliminate one common writing convention from existence, what would it be?
5. What's more important: being understood or being respected for your intellect?
6. Describe the relationship between truth and style. Which serves which?
7. What's a position you used to hold about writing/communication that you've since reversed?
8. What does "good taste" mean to you in three words?
9. When you read something that's clearly competent but still feels wrong, what's usually the cause?
10. What's the most overrated quality people praise in writing?
11. What's the most underrated quality people ignore in writing?
12. If your writing voice had a political leaning (not party, but disposition), what would it be?
13. What idea or principle do you return to every time you're unsure about a draft?
14. What's a "rule" of good writing that you break regularly and deliberately?
15. What do you want people to *feel* after reading something you wrote — and what do you definitely not want them to feel?

### Category 2: Writing Mechanics (20 questions)

16. Long sentences or short sentences — and what's the exception?
17. Do you use semicolons? If yes, when? If no, why not?
18. What's your relationship with the em dash?
19. How do you feel about exclamation marks?
20. Parentheses — useful tool or crutch?
21. When is a paragraph too long? What's your personal threshold?
22. Do you prefer active voice exclusively, or is passive voice sometimes the right call?
23. What's your stance on sentence fragments — grammatically incorrect or stylistically valid?
24. How do you handle transitions between ideas — smooth connectors or hard cuts?
25. Oxford comma: always, never, or context-dependent?
26. What's your go-to sentence rhythm — do you build momentum or vary deliberately?
27. How do you feel about starting sentences with "And" or "But"?
28. What's your relationship with adjectives — lean, moderate, or generous?
29. Do you think about syllable count or word weight when choosing words?
30. How do you handle quotes — embed them, block them, or paraphrase?
31. What's your rule for using numbers — spell them out or use digits?
32. When you reread your own writing, what mechanical habit do you most often edit?
33. How do you handle lists — numbered, bulleted, or inline?
34. What punctuation mark do you overuse? Be honest.
35. What's your approach to headlines or titles — descriptive, provocative, or minimal?

### Category 3: Aesthetic Crimes (15 questions)

36. What's a word or phrase that makes you immediately distrust the writer?
37. What font or typography choice signals "I don't care about craft"?
38. What's the worst common design choice in business documents?
39. Name a perfectly grammatical sentence you'd still cut from any document.
40. What's the laziest opening line you see people use?
41. What writing habit signals "I'm trying to sound smart" rather than "I am clear"?
42. What's a visual or formatting choice that makes you stop reading immediately?
43. What's the most annoying thing people do with bullet points?
44. What cliché would you ban from all professional communication forever?
45. Describe the worst PowerPoint slide you've ever seen. What made it bad?
46. What's an aesthetic choice that's popular but that you personally can't stand?
47. What's the worst way to end an email?
48. What "professional" writing convention is actually just bad writing?
49. What's a word people think sounds impressive but actually means nothing?
50. What layout or formatting sin is unforgivable in a document longer than one page?

### Category 4: Voice & Personality (15 questions)

51. If your writing voice were a person at a party, how would you describe them?
52. How much of your actual personality shows up in professional writing?
53. Are you funnier in writing than in person, or the other way around?
54. What's your relationship with sarcasm in professional contexts?
55. How do you express warmth without sounding fake?
56. What's the gap between how you sound and how you want to sound?
57. Do you write differently when you're angry? How?
58. How do you handle writing about topics you find boring?
59. What's your default register — casual, neutral, or formal?
60. When is informality the right choice?
61. What aspect of your voice do people most often misinterpret?
62. How do you adjust your voice for different audiences — or do you?
63. What's the hardest emotion to convey in writing?
64. How do you signal confidence without sounding arrogant?
65. What does your inner monologue sound like versus your written output?

### Category 5: Structure & Format (15 questions)

66. How do you decide where to start a piece of writing?
67. What's your approach to organizing a complex argument — outline first or discover as you go?
68. How long should an introduction be? What's the maximum before you lose patience?
69. When do you use headers and when do you let the text flow?
70. What's the ideal length for a paragraph in your writing?
71. How do you handle conclusions — summarize, call to action, or just stop?
72. What's your preferred document length — do you write long and cut, or write short and expand?
73. How do you signal a change in topic without using a header?
74. What's your relationship with footnotes or endnotes?
75. When is it appropriate to break a structural rule?
76. How do you handle evidence or data — inline or separated?
77. What's the right way to use bold or emphasis in professional writing?
78. How do you feel about executive summaries?
79. What structural pattern do you fall back on when you're unsure?
80. What's the most important sentence in any document, and where should it go?

### Category 6: Hard Nos (10 questions)

81. What's a word or phrase you would absolutely never use in your writing?
82. What's a formatting choice you refuse to accept in any document you're associated with?
83. What tone or register is permanently off-limits for you?
84. What's a common writing practice you consider ethically wrong?
85. Name a structural pattern you would veto in any draft you review.
86. What's a type of humor you would never use in professional writing?
87. What's an opening strategy you would never use?
88. What's a closing strategy you would never use?
89. What word or phrase would make you reject a piece of writing regardless of its other qualities?
90. What's a design or typography choice that would make you dismiss the entire document?

### Category 7: Red Flags (10 questions)

91. When reviewing someone else's writing, what's the first thing you check?
92. What single signal tells you "this person can't write" within the first sentence?
93. What pattern, when you spot it, tells you the writer didn't revise?
94. What's a tell that someone is hiding behind jargon instead of saying something real?
95. What's the fastest way to identify that a piece of writing doesn't respect the reader's time?
96. What's a sign that a writer is padding length instead of adding substance?
97. What's the most common mistake smart people make in their writing?
98. What's a subtle sign that a writer doesn't actually understand the topic?
99. What pattern reveals that feedback was incorporated mechanically rather than thoughtfully?
100. What's the one thing that, if present, makes you trust a writer immediately?

---

## Example Sharp Prompts for Running the Interview

Use these to keep the session moving when the subject is hedging or slow:

- *"Don't think about it — first answer that comes to mind."*
- *"You can only pick one. Which is it?"*
- *"If someone who knows you well read your answer, would they agree? If not, try again."*
- *"That's a reasonable answer. Now give me the honest one."*
- *"You just said two things. Pick the one you actually believe."*
- *"I don't need the explanation — just the answer."*
- *"Say it in five words."*

---

## Usage Notes

- **Drip mode:** Ask 5–10 questions per session. File answers immediately. Resume next session. Complete in 10–20 sessions over 2–4 weeks.
- **Single-session mode:** Block 45 minutes. Power through all 100. Expect quality to degrade after question 60 — flag those answers for re-ask.
- **AI self-interview:** An agent can run this on itself by answering from its system prompt, SOUL.md, or accumulated memory. Useful for bootstrapping a voice profile from behavioral data.
- **Multi-agent handoff:** After compilation, any agent loading the resulting profile should be able to match the voice within one prompt cycle. If it can't, the profile is incomplete — flag the weak category and re-ask.

---

## Source

Protocol designed for Hermes Agent voice-extraction workflows. Inspired by structured intake interviews, the Proust Questionnaire, and adversarial red-teaming of taste preferences.
