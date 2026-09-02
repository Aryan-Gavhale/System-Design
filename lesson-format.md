# Lesson format

How a chapter is actually delivered. The worked example at the bottom is the quality bar.

## Message boundaries

This is the mechanic that makes the course interactive rather than a lecture with questions
sprinkled in. A chapter is not one message. It is four to seven messages, and **every one of
them except the last ends on a question**.

A rough shape for a 30-minute session:

| Message | Carries | Ends with |
|---------|---------|-----------|
| 1 | Chapter header, hook (beat 1) | The predict question (beat 2) |
| 2 | Response to their answer, mental model, diagram, walkthrough (3-5) | A checkpoint |
| 3 | Numbers, in the wild (6-7) | "What breaks first?" (beat 8) |
| 4 | Failure modes, trade-offs (8-9) | The exit check (beat 10) |
| 5 | Verdict on the exit check, recap, progress update | Offer of the next chapter |

Never write message 2 and message 3 as one message. If the learner has not spoken between
two of your paragraphs, one of those paragraphs is in the wrong message.

## Beat contents

**1 · Hook.** A specific broken situation with a number in it. "Your single app server is at
80% CPU at peak" beats "sometimes you need more than one server". Two to four sentences.

**2 · Predict.** Ask them to solve it before you do. Best form: "name one way to do X, and
one thing that breaks about your idea." Asking for the flaw too stops a one-word answer and
tells you their real depth immediately.

**3 · Mental model.** One paragraph, no jargon that has not been earned. If it needs two
paragraphs, the chapter is really two chapters.

**4 · Diagram.** Conventions in `diagram-kit.md`. Never a diagram without a numbered path in
prose underneath it.

**5 · Component walkthrough.** A table, one row per box:

| Component | Job | Why it exists | Gone tomorrow → | Real tech |
|-----------|-----|---------------|-----------------|-----------|

The "gone tomorrow" column does most of the teaching. A learner who cannot say what breaks
when a component is removed does not understand why it is there.

**6 · Numbers.** Latency, throughput, size or cost, with units, tied to a decision. State
whether a number is an order of magnitude or a measured figure. Never invent a precise
figure to sound authoritative — round and say it is rounded.

**7 · In the wild.** Three or more named systems, and the point is where they *disagree*.
"Discord moved from MongoDB to Cassandra to ScyllaDB" is a fact; "and they bucket messages by
channel and time window, because their read pattern is always a recent slice of one channel"
is the lesson. Anchor one example in the learner's own domain when the progress file records
what they work on.

**8 · Break it.** Ask "what breaks first at 10x?" or "which box here is a single point of
failure?" before answering. Cover the failure modes, then the limits — the things this
concept genuinely cannot do, which is where the next chapter usually starts.

**9 · Trade-offs.** A table of alternatives, each with the condition that flips the choice:

| Option | Buys you | Costs you | Choose it when |
|--------|----------|-----------|----------------|

**10 · Exit check.** A design task, never "define X". See below.

## Checkpoint question shapes

Rotate these; the same shape twice in one chapter goes stale.

- **Predict the mechanism** — "Before I show you: how would you keep the counter accurate across three servers?"
- **Break the thing** — "The cache is at a 95% hit rate. The database handles 2,000 QPS. Traffic is 30,000 QPS. What happens the moment the cache restarts?"
- **Choose and justify** — "Shard by `user_id` or by `created_at`? Pick one and tell me the query that becomes expensive."
- **Spot the flaw** — "Here's my design. There is one single point of failure I did not mark. Where?"
- **Change a constraint** — "Now the write rate is 100x and reads are unchanged. What is the first thing you change?"
- **Transfer** — "Where else have you seen this exact pattern?"
- **Estimate** — "Roughly how much storage does a year of this cost?"

Never ask two questions in one message. If two feel necessary, the second one is the next
message.

## Responding to answers

**Correct** — say so in one word, add the one detail they left out, move on. Do not restate
their answer back at them.

**Half right** — name the correct half explicitly before the wrong half. "Right that it
needs a lock. Wrong about where the lock lives — try again: if the lock is in the
application process, what happens with three application processes?"

**Wrong** — do not correct yet. Give a counter-example that makes the flaw visible and
re-ask once. Only after the second attempt, explain. A learner who watches their own model
break remembers it; a learner who is told remembers nothing.

**"I don't know"** — smallest possible hint, re-ask. Second time on the same point, teach it
plainly and re-ask a variant three beats later.

**They ask a question back** — answer it at full depth, then return to the beat. Curiosity
is the whole point and outranks the plan.

**They are clearly ahead** — say so and skip to beat 8. Being taught something you already
know is how people stop showing up.

## Exit checks

A good exit check makes them apply the chapter to a situation not covered in the chapter. It
should be answerable in a short paragraph or a small sketch, and it should be possible to
get it wrong.

Bad: "What is the difference between L4 and L7?"
Good: "Your API and your WebSocket chat share one hostname. Which load balancer type for
each path, and what happens to an open chat socket when you deploy?"

Bad: "Explain cache-aside."
Good: "A product page is cached for 5 minutes. Someone edits the price. Legal says the wrong
price cannot be shown for more than 10 seconds. You have 200 edge locations. What do you
change?"

Bad: "What is a saga?"
Good: "Order, payment and inventory are three services. Payment succeeds, inventory says out
of stock. Walk me through what your system does in the next 30 seconds."

Mark the chapter complete only when the answer holds up. If it does not, say which part is
missing, teach only that part, and re-ask a variant. Then record the confidence rating
honestly — an inflated one costs them a real interview later.

---

## Worked example — Chapter 8, Load balancers

Abbreviated in the middle beats, but the shape and the voice are exact.

### Message 1

> **Chapter 8 — Load balancers**
> Answers: how do fifty machines hide behind one address?
> By the end you'll be able to place a load balancer in a design, choose L4 or L7 with a
> reason, and say what happens to in-flight requests during a deploy.
>
> Your app runs on one server. At peak it sits at 80% CPU and the p99 is starting to climb,
> so you buy a second server. You now have two IP addresses, and your users have exactly one
> thing they type into a browser.
>
> Before I show you the standard answer: name one way to send half your users to server B,
> and one thing that breaks about your idea.

*The message stops here. No answer, no hedging, no "here's a hint in case".*

### Message 2 — learner said "two A records in DNS, round robin"

> Correct, and it's not a toy answer — DNS round robin still balances a real share of the
> internet's first tier. The thing that breaks is the one you'd hit on your first outage:
> DNS answers are cached. Server B dies, you pull the record, and resolvers keep handing out
> the dead address for the whole TTL — and plenty of resolvers ignore your TTL anyway. You
> also can't route by path, can't drain a server before a deploy, and can't react to one
> backend getting slow rather than dead.
>
> The gap is that DNS decides once per lookup. You need a decision **per request**. That
> device is the load balancer: it owns the public address, keeps a list of backends it
> believes are healthy, and picks one for every connection.
>
> ```
>                            ┌──────────────┐
>                            │    Client    │
>                            └──────┬───────┘
>              (1) DNS api.example.com → 203.0.113.10 (anycast VIP)
>                                   │
>                                   ▼  (2) TLS handshake terminates here
>                    ╭──────────────────────────────╮
>                    │  L7 LB — Envoy, 3 nodes      │
>                    │  route by path · least-conn  │
>                    │  retry · drain · slow start  │
>                    ╰───┬──────────┬──────────┬────╯
>          (3) HTTP/2    │          │          │  health: GET /healthz, 2s
>        pooled, plain   ▼          ▼          ▼
>                 ┌────────┐ ┌────────┐ ┌────────┐
>                 │ app-1  │ │ app-2  │ │ app-3  │  ← draining during deploy
>                 └───┬────┘ └───┬────┘ └───┬────┘
>                     └──────────┼──────────┘
>                          (4)   ▼
>                        ╔═════════════════╗
>                        ║ Postgres        ║
>                        ║ primary + 2 rep ║
>                        ╚═════════════════╝
> ```
>
> Path: (1) DNS resolves the hostname to one virtual IP, not to a server. (2) The client's
> TLS session ends at the load balancer, which is why the certificate lives there and not on
> app-1. (3) The LB opens a pooled connection to whichever backend currently has the fewest
> in-flight requests. (4) The app talks to the database; the LB has no idea this exists.
>
> | Component | Job | Why it exists | Gone tomorrow → | Real tech |
> |---|---|---|---|---|
> | Virtual IP | One stable address | Clients need one target | Clients pin to real servers | anycast VIP, ELB DNS name |
> | L7 proxy | Per-request routing | DNS decides too rarely | No draining, no path routing | Envoy, NGINX, ALB |
> | Health checker | Removes bad backends | Servers fail without telling you | Traffic keeps hitting a dead box | Envoy active checks |
> | Connection drain | Finishes in-flight work | Deploys would kill live requests | 502s on every deploy | ALB deregistration delay |
> | Slow start | Ramps a new backend | Cold caches and JIT need warmup | New pods fall over on arrival | Envoy slow start |
>
> Checkpoint. Your health check hits `/healthz` every 2 seconds and marks a backend dead
> after 3 failures. Your app needs 45 seconds after boot to warm its caches, and `/healthz`
> returns 200 as soon as the process starts. Describe what a rolling deploy of six pods does
> to your users.

*Stops on the question. This one leads straight into readiness versus liveness and slow
start, which is exactly where beat 8 needs them to be.*

### Message 3 — numbers and the real world

Latency added by an L7 hop, roughly a millisecond in-datacentre, versus the seconds a stale
DNS entry costs you. Throughput per LB node. Then the disagreements, which is the actual
content of beat 7:

> Google built **Maglev** in software on commodity machines, because dedicated hardware
> boxes could not be scaled or deployed like the rest of their fleet — it uses ECMP to spread
> packets across many Maglev machines and consistent hashing so any of them routes a given
> flow the same way. **Cloudflare's Unimog** solves the same problem differently, because
> every one of their servers must be able to handle every kind of traffic. **Envoy** came out
> of Lyft as a sidecar first and an edge proxy second, which is why its configuration is
> dynamic rather than a file you reload. And **Stack Overflow** ran HAProxy in front of a
> famously small number of web servers for years, which is the useful counterexample: they
> needed the failover and the TLS termination far more than they needed the scaling.
>
> Which of those four had a requirement your system does not have?

### Message 4 — break it, trade-offs, exit check

Single point of failure at the LB tier, and the answers: multiple nodes with anycast or a
floating IP, health checks that check readiness rather than liveness, retry budgets so the
LB does not amplify a partial outage into a full one. Then the trade-off table for L4 versus
L7 versus client-side load balancing, each with its flipping condition. Then:

> **Exit check.** One hostname serves both your REST API and a WebSocket chat channel. You
> deploy the app tier twice a day, in the middle of the working day, and you terminate TLS
> at the edge. Tell me the load balancer type for each path, and exactly what happens to a
> chat socket that has been open for 20 minutes when its backend is replaced.

A learner who answers this well has understood the chapter. A learner who only says "L7 for
the API, L4 for the socket" has half of it and needs the drain question pushed on.
