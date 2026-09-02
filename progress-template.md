# Progress file format

Lives at `~/.cursor/system-design-course/progress.md`, outside the skill directory so a skill
update never overwrites it. Create the directory if it does not exist. Copy the template
below at placement, then update it after every chapter.

Keep it short. It is a working file read at the start of every session, not a transcript.

---

```markdown
# System Design Course — Progress

Mode: <guided | socratic>
Products they use: <Netflix, Amazon, WhatsApp... — the examples to teach through>
Pace: <short chapters | longer deep dives>
Started: <YYYY-MM-DD>   Last session: <YYYY-MM-DD>
Chapters done: <n>   Current run: <n sessions in a row>

## Current
Part: <Part 0 — Ground floor>
Chapter: <N — title>  (<i> of <j> in this part)
Stopped at: <beat and what was in progress>

## Completed
| Ch | Title | Date | Exit check | Confidence |
|----|-------|------|-----------|------------|
| 1 | The path of one request | 2026-09-02 | pass | strong |

## Weak spots
One is asked at the start of each session, oldest first. Retire after two correct
recalls in separate sessions.
| Ch | The specific gap | Added | Correct recalls |
|----|------------------|-------|-----------------|

## Stories used
Avoid repeating a story across chapters.
- Ch 1 — Amazon's 100 ms latency experiment

## Notes
- <what engaged them, what bored them, jargon to avoid, anything that changes delivery>
```

---

## Update rules

**Mode** is the most important field. Once it is `guided`, keep teaching before asking until
the learner answers two recognition questions confidently. Flipping back too early is what
makes a session feel like an exam.

**Confidence** is honest, not encouraging: `strong` means they could explain it to someone
else, `ok` means they got it with one nudge, `shaky` means the reasoning wobbled. A `shaky`
chapter also gets a weak-spot row.

**Exit check** records `pass`, `pass on 2nd try`, or `deferred`. A deferred exit check means
the chapter stays as Current.

**Weak spots** hold the specific gap, not the topic. "Chapter 19" is useless six weeks
later; "picked a shard key without asking about the query pattern" is a question you can
re-ask.

**Chapters done and current run** exist to show momentum. Mention the count when it is
encouraging and never mention the 72.

**Stories used** stops the course reusing Knight Capital or the Facebook outage three times.
Every chapter gets a fresh story.

**Notes** is where engagement lives. Record which stories landed and which explanations
fell flat — that is what makes the next session better than the last one.

**Never record or teach through the learner's own employer, product or industry.** Examples
come from famous consumer products only — see "Which examples to use" in `SKILL.md`.
