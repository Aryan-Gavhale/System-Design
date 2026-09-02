# Progress file format

Lives at `~/.cursor/system-design-course/progress.md`, outside the skill directory so a skill
update never overwrites it. Create the directory if it does not exist. Copy the template
below at placement, then update it after every chapter.

Keep it short. It is a working file read at the start of every session, not a transcript.

---

```markdown
# System Design Course — Progress

Track: <full course | interview sprint | named track from curriculum.md>
Goal: <interviews | a real design at work | depth>
Session length: <15 | 30 | 60> min
Started: <YYYY-MM-DD>   Last session: <YYYY-MM-DD>

## Current
Chapter: <N — title>
Stopped at: <beat number and what was in progress>
Resume with: <the one thing to pick up>

## Completed
| Ch | Title | Date | Exit check | Confidence |
|----|-------|------|-----------|------------|
| 1 | The path of one request | 2026-09-02 | pass | strong |
| 2 | Latency, and why distance is the boss | 2026-09-02 | pass on 2nd try | shaky |

## Weak spots
Oldest first. One is asked at the start of every session. Retire only after two
correct recalls in separate sessions.
| Ch | The specific gap | Added | Correct recalls |
|----|------------------|-------|-----------------|
| 2 | Could not connect p99 to fan-out request count | 2026-09-02 | 0 |

## Learner's systems
Anchor at least one example per chapter here.
- <domain, stack, the systems they actually work on>
- <the part they said they were least sure about, from placement>

## Notes
- <pacing preference, topics to skip, an interview date, anything that changes delivery>
```

---

## Update rules

**Confidence** is honest, not encouraging: `strong` means they could explain it to someone
else, `ok` means they got it with one nudge, `shaky` means they passed but the reasoning
wobbled. A `shaky` chapter also gets a weak-spot row.

**Exit check** records what happened: `pass`, `pass on 2nd try`, or `deferred` if the session
ran out. A chapter with a deferred exit check is not complete and stays as Current.

**Weak spots** hold the specific gap, not the topic. "Chapter 19" is useless six weeks later;
"picked a shard key without asking about the query pattern" is a question you can re-ask.

**Learner's systems** is the highest-value section. A concept explained through something
they maintain lands harder than the same concept explained through Twitter's timeline.
