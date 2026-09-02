# Design walkthrough

For "design YouTube", "design Uber", and every Part 9 case study.

**The goal is that the learner could design the next system alone.** That means depth on the
few decisions that define the system, and the reusable rule behind each one — not coverage
of everything that could be said. A learner who can recite your architecture but cannot pick
a datastore for a system you did not cover has learned nothing portable.

## Deliver it in four stages

One stage per message. Between stages, a single line offering the next — never a quiz
question that blocks progress. They asked for a design; do not make them earn it.

If they say "all of it", give all four, still separated by headings.

| Stage | Content | Ends with |
|-------|---------|-----------|
| 1 | Requirements, estimation, **the one number** | What that number forces |
| 2 | **Storage decisions** — every store, why, and what was rejected | The reusable rule |
| 3 | Architecture — write path and read path, drawn separately | The request paths in prose |
| 4 | Failures, trade-offs, monitoring, follow-ups | An offer to drill |

## Depth over breadth

**Cut ruthlessly.** A component whose purpose is obvious from its name gets one line, or no
line. Spend the space on the three or four decisions someone could plausibly get wrong.

Every system has one central insight. Say it explicitly and early:

- YouTube — the write path is a pipeline, the read path is a CDN, and egress dictates both
- Uber — matching is a state machine that must never double-book, over a geospatial index
- WhatsApp — the hard part is not delivery, it is knowing which server holds the socket
- Payments — never charge twice, which makes idempotency the whole architecture

If the learner remembers only that sentence, the chapter worked.

---

## Stage 1 — Requirements and the one number

**Functional requirements.** Five or six capabilities a user would recognise. Then what is
**explicitly out of scope** — naming the boundary is what separates a scoped design from a
vague one.

**Non-functional requirements.** Latency, availability, consistency, durability, cost — each
with a number, and split per path where they differ. "Highly available" is not a
requirement; "99.95% on the watch path, 99.9% on upload" is.

**Estimation with the arithmetic shown.** An unsupported number cannot be checked. Work
through daily volume, per-second rate, peak, storage, bandwidth.

**End with the one number that drives the architecture, and what it forces.** This is the
most important sentence in the whole walkthrough.

## Stage 2 — Storage decisions

**The section that matters most, and the one usually missing.** Every distinct kind of data
gets a row, and every row names the rejected alternative:

| Data | Access pattern | Choice | Why this | Why not the obvious alternative |
|------|----------------|--------|----------|---------------------------------|

Rules for this table:

- **Every store named with real technology and a size.** "A database" is not a decision.
- **The rejected option must be the one a reasonable person would have picked**, and the
  reason must be a real constraint, not a preference. "Not MySQL for the video bytes,
  because megabyte blobs evict the buffer pool and make backups unrestorable."
- **Include derived stores** and say they are rebuildable — search indexes, caches,
  aggregates. That is a design property worth stating.
- **Name the shard key**, and the query it makes expensive.

Close the stage with the **reusable rule**, drawn from `decision-rules.md`, in one sentence.
Then ask them to apply it somewhere else in the same design. That transfer question is where
understanding forms.

## Stage 3 — Architecture

**Two diagrams, always separate**: write path, read path. They are different systems with
different bottlenecks, and merging them is what makes a diagram unreadable.

Numbered hops, real component names and sizes, prose walkthrough underneath. Conventions:
`diagram-kit.md`.

Then only the components whose existence is non-obvious: what it does, why it exists, what
breaks without it. Skip the ones the diagram already explains.

## Stage 4 — Failures, trade-offs, monitoring

**Failures** — what dies, what the user sees, how it degrades. Include the partial-failure
case, because that is where real systems corrupt: the row committed before the downstream
call timed out.

**Trade-offs** — a table whose final column is **the condition that flips the decision**.
That column is the entire point; without it the table is a list of opinions.

**Monitoring** — four or five signals, as user-visible symptoms rather than machine stats.

**Follow-ups** — five questions this design invites, one line each.

---

## Closing

A short menu, never a blocking question: `deeper` on a component, `numbers` to redo the
estimation, `break it` for a failure drill, `mock` to defend it under pressure, or
`design <something else>` to apply the same rules cold.

The best sign the walkthrough worked is the learner designing the next system without help.
Offer that explicitly: "want to try designing Instagram yourself, and I'll only push back
where the reasoning slips?"
