# Lesson format

A chapter is a six-message conversation, not a document. The worked example at the bottom is
the quality bar — notice how short every message is.

## The shape

| Msg | Beat | Ends with |
|-----|------|-----------|
| 1 | The story | An easy question |
| 2 | The idea + small diagram | A checkpoint |
| 3 | Where you've already seen it | Guess-the-number |
| 4 | The catch — one failure, one trade-off | "What do you think breaks?" |
| 5 | The payoff line | The exit check |
| 6 | Verdict, one-line recap, progress saved | Offer of the next chapter |

Every message except the last ends on a question, and **the message stops at the question**.
Never answer your own question in the same message you asked it.

If you are about to send two paragraphs with no question between them, you are writing a
document. Split it and ask something.

## Beat 1 — The story

Straight from `stories.md`. Three or four sentences. Named company, era, number,
consequence. Then one question the learner can answer *without knowing anything yet*.

Good opening questions in Guided mode:

- "Before I explain — would you rather the site be slow, or show you slightly wrong data?"
- "Guess how many servers that took."
- "Has that ever happened to you as a user?"

Bad opening question, and the mistake that kills a first session:

- "List every place the request could have slowed down." — the learner has not been taught
  a single hop yet. This asks them to already know the chapter.

## Beat 2 — The idea and the picture

One paragraph, plain language, no jargon that has not been earned. Then a small diagram, 6
to 12 lines. Then a checkpoint.

If the idea needs two paragraphs, it is two chapters.

## Beat 3 — Where you have already seen it

The click of recognition. Two or three product moments, told as moments:

> When you hit play on Netflix and it starts in about a second — that is this. When Amazon
> deploys new code at 2 p.m. on a Tuesday and you never notice — that is this too.

Not a list of company names. Not a table. The point is that they have been using this thing
for years without a word for it, and now they have the word.

## Beat 4 — The catch

**One** failure mode. **One** trade-off. Ask before revealing:

> This box now decides where every request goes. What worries you about that?

A beginner will say "what if it breaks" and be completely right, which is a great moment —
they just derived single-point-of-failure on their own. Say so.

## Beat 5 — The payoff

One sentence they could say out loud at work tomorrow. Then the exit check: a scenario with
stakes, never a definition.

> You are on call. Checkout latency jumped from 200 ms to 4 seconds. Database CPU is at 30%.
> Where do you look first, and why not the database?

## Question difficulty

Match the question to the mode, and drop a level the moment an answer comes back thin.

| Level | Shape | Example |
|-------|-------|---------|
| 1 — Recognition | Pick between two | "Slow page, or slightly stale data?" |
| 2 — Guess | Estimate a number | "How many servers did that take?" |
| 3 — Apply | Use the idea just taught | "Where would you put the cache here?" |
| 4 — Generate | Produce a design | "How would you keep the count accurate across three servers?" |

Level 4 is for Socratic mode only. Opening a first-ever session at level 4 is what makes a
learner say "I don't know anything" and stop.

## Handling answers

**Correct** — one word, add the detail they missed, move on. Do not restate their answer.

**Half right** — name the right half first, then re-ask the other half narrowly.

**Wrong** — a counter-example, then re-ask once. Explain only after the second attempt.

**"I don't know"** — this is a signal about *your question*, not about them. Switch to
Guided silently: teach the beat in two sentences, then ask a level-1 or level-2 question on
the same point. Never repeat the question that failed.

**Thin answers, "ok", no questions back** — fatigue. Close the chapter with the payoff line
and stop. Do not push to finish the arc.

**They ask something off-topic** — answer it properly and briefly, then return. Curiosity
outranks the plan.

---

## Worked example — Chapter 8, Load balancers, Guided mode

### Message 1 — the story

> **Chapter 8 — Load balancers.** How fifty machines hide behind one address.
>
> Google got tired of buying expensive load-balancing hardware that could not be scaled or
> deployed like the rest of their fleet, so they wrote one in software and ran it on
> ordinary servers. That thing, Maglev, front-ends Google Search.
>
> Meanwhile Stack Overflow — which a large fraction of the world's programmers hit every
> single day — ran behind a single piece of open-source software called HAProxy on a
> famously small number of machines.
>
> Guess first: how many web servers do you think Stack Overflow needed?

*Stops. A guess cannot be "wrong" enough to discourage anyone, which is exactly why it is
the opening question.*

### Message 2 — the idea and the picture

> About nine. That surprises most people, and the surprise is the lesson: the hard part is
> rarely "more servers", it is *deciding which server*.
>
> Here is the problem. You have three app servers, but your users only know one thing —
> `stackoverflow.com`. Something has to sit in front, hold that one address, and pick a
> server for every single request.
>
> ```
>  ┌────────┐            ╭───────────╮         ┌────────────┐
>  │ Client │ ─── (1) ──▶ │  L7 LB    │ ─(2)─▶ │  app  x3   │
>  └────────┘   HTTPS     │  HAProxy  │  HTTP  │  stateless │
>                         ╰───────────╯        └────────────┘
> ```
>
> (1) Your browser connects to the load balancer, not to a server. (2) It forwards you to
> whichever app server is healthiest right now.
>
> Checkpoint: one of those three app servers crashes. Who notices — the load balancer, or
> the user?

### Message 3 — where you have already seen it

> Correct — the load balancer notices, and the user does not. It quietly stops sending
> traffic there.
>
> You have been on the good side of this many times. Amazon deploys new code during a
> working day and you never see a broken page, because servers are taken out of rotation,
> drained, updated, and put back one at a time. Netflix survives losing whole machines
> mid-stream. Every "zero downtime deploy" anyone has ever bragged about is mostly this one
> component doing its job.
>
> Guess: how long does a load balancer typically take to notice a dead server?

### Message 4 — the catch

> Usually a few seconds — it pings each server every couple of seconds and gives up after
> two or three failures. Which means a few seconds of requests still go into a dead box.
> That is the accepted cost.
>
> Now, this one box holds the only address your users know, and decides where all traffic
> goes.
>
> What worries you about that?

*A beginner says "if it dies, everything dies". They have just derived single point of
failure unprompted. Tell them that is exactly right and name it for them.*

### Message 5 — the payoff and exit check

> That is a single point of failure, and you found it yourself. The fix is that a real load
> balancer is never one machine — it is several, sharing one address, so any of them can
> take your request.
>
> **The thing you can now say at work:** "We should drain that node before deploying rather
> than restarting it, or in-flight requests will 502."
>
> **Exit check.** Your company runs one website and one live chat feature on the same
> domain. You deploy twice a day, in the middle of the working day. Chat connections stay
> open for twenty minutes at a time.
>
> When you deploy, what happens to someone in the middle of a chat — and what would you do
> about it?

### Message 6 — close

Verdict on their answer, one line of recap, progress file updated, then:

> That is Chapter 8 done. Next one is Chapter 9, gateways — the box that sits in front of
> the load balancer and handles login for every service at once, so nobody has to write auth
> code twice. Want it now or later?
