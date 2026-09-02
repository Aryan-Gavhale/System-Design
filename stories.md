# Story library

Beat 1 of every chapter opens with one of these. A named company, an era, a number, a
consequence. Tell it in three or four sentences — the story is the hook, not the lesson.

`Guess` is the guess-the-number prompt for that chapter. Ask before revealing.
`Seen it when` is the product moment where the learner has already met this component
without knowing its name.

Round numbers and hedge them out loud ("roughly", "about"). Never invent precision.

---

## Part 0 — Ground floor

### Ch 1 — The path of one request
**Story.** Amazon ran an experiment years ago that has shaped web engineering ever since:
they deliberately slowed pages down. Every extra 100 milliseconds cost them roughly 1% in
sales. Google ran a similar test on search and reported that around half a second of extra
delay cut searches by about 20%. Nobody clicked "leave" — they just quietly did less.
**Guess.** How much money is 1% of Amazon's annual sales?
**Seen it when.** Any time a page felt slow and you gave up on it without thinking.

### Ch 2 — Latency
**Story.** Light in fibre travels roughly 200,000 km per second, so a round trip from Mumbai
to a server in Virginia costs about 250 ms *before* anything is computed. Netflix decided
this was unwinnable and stopped fighting it — instead of faster servers, they shipped
physical boxes to internet providers around the world and put the movies inside them.
**Guess.** How far away is the Netflix server that streams to your house?
**Seen it when.** Netflix starts playing in under two seconds, but a video call to another
continent still lags.

### Ch 3 — Throughput, concurrency and queues
**Story.** 15 November 2022, Ticketmaster's Taylor Swift Eras sale. Ticketmaster later
reported about 3.5 billion system requests, roughly four times their previous record, and
over 14 million people trying at once. The site did not just get slow — the public sale was
cancelled entirely. Nothing was broken. Everything was full.
**Guess.** How many of those 14 million people could actually be served at once?
**Seen it when.** Any queue-to-buy page, and every airport security line.

### Ch 4 — Back-of-envelope estimation
**Story.** When Facebook bought WhatsApp in 2014, WhatsApp was carrying tens of billions of
messages a day for hundreds of millions of users. The engineering team was about 32 people.
They got there by being ruthless about arithmetic — knowing exactly how many connections one
server could hold before buying the next one.
**Guess.** How many engineers did WhatsApp have at that scale?
**Seen it when.** Every "we need Kubernetes" argument that nobody has done the maths on.

### Ch 5 — How to draw an HLD
**Story.** Around 2002 Jeff Bezos sent a memo that reshaped Amazon: all teams must expose
their data only through service interfaces, with no back doors, no shared databases, no
exceptions — and anyone who did not comply would be fired. It was brutal, and it is why AWS
exists at all. Drawing the boxes and the lines between them was the whole point.
**Seen it when.** Every AWS service you have ever used is a descendant of that memo.

---

## Part 1 — The request path

### Ch 6 — DNS and service discovery
**Story.** 4 October 2021, Facebook vanished. A configuration change withdrew the network
routes to their own DNS servers, so nothing could look up `facebook.com`, `instagram.com` or
`whatsapp.com` for about six hours. The worst part: their internal tools ran on the same
names, and the badge readers on the doors stopped working, so engineers could not physically
get into the building holding the servers.
**Guess.** How long until someone could open the door?
**Seen it when.** A site is "down" but works fine on your phone's mobile data.

### Ch 7 — Connections: TCP, TLS, HTTP/1.1 → 2 → 3
**Story.** Google measured that a cold HTTPS connection burns two to three network round
trips on handshakes before a single byte of your page arrives. On a mobile connection to
another continent that is a quarter of a second of pure ceremony. So Google built a new
protocol on UDP, shipped it in Chrome to a billion users, and it became HTTP/3.
**Guess.** What fraction of a slow page load is handshakes rather than data?
**Seen it when.** The first page on a site is slow and every page after it is instant.

### Ch 8 — Load balancers
**Story.** Google got tired of buying expensive load-balancing hardware that could not be
deployed or scaled like the rest of their fleet, so they wrote one in software and ran it on
ordinary servers. It is called Maglev, and it front-ends Google Search. Meanwhile Stack
Overflow, serving a large slice of every programmer's working day, ran on a famously tiny
fleet behind HAProxy — they needed the failover far more than the scale.
**Guess.** How many web servers did Stack Overflow run on?
**Seen it when.** A site deploys new code in the middle of the day and you notice nothing.

### Ch 10 — Statelessness and sessions
**Story.** The reason "log out of all devices" exists as a button, and the reason it
sometimes does not work, is a design decision about where your login lives. If the server
holds it, logout is instant. If the token holds it, the server has to be *told* not to trust
a token it already signed — and plenty of systems forget to build that part.
**Seen it when.** You change your password and a stale tab stays logged in anyway.

### Ch 11 — Rate limiting
**Story.** Stripe published how their rate limiters work, and the interesting part is that
they do not treat all traffic equally. When the system is under pressure they shed
low-priority requests first, so a payment goes through while a bulk report request waits.
The limiter is not a bouncer, it is a triage nurse.
**Seen it when.** An API returns 429, or a game shows "you're doing that too much".

### Ch 12 — API design and idempotency
**Story.** Your card payment times out. Did it go through? The app cannot tell — a timeout
means the answer was lost, not that nothing happened. Stripe's answer was the idempotency
key: the client invents a unique string, and Stripe promises that the same key never charges
twice no matter how many times you retry. Every double-charge bug in history is a missing
version of this.
**Seen it when.** You pressed "Pay" twice in a panic and were only charged once.

---

## Part 2 — Storage

### Ch 13 — What a database does with your row
**Story.** Databases learned decades ago that disks hate random writes and love sequential
ones. So almost every database writes your change twice: first appended to a log as fast as
the disk can take it, then later organised into the real data pages. If the power dies in
between, the log is what saves you. That trick is why your commit returns in milliseconds.
**Seen it when.** A database restarts after a crash and says "recovering".

### Ch 14 — LSM trees
**Story.** Discord's messages outgrew MongoDB, then outgrew Cassandra, and in 2023 they
published how they moved trillions of messages to ScyllaDB. The reason all three of those
later choices look alike underneath: they buffer writes in memory and flush sorted files to
disk, because a chat app writes constantly and reads only the recent slice.
**Guess.** How many messages has Discord stored?
**Seen it when.** Discord loads a channel's recent history instantly but takes a moment to
jump to a message from 2019.

### Ch 15 — Indexes
**Story.** Every engineering team has this incident exactly once: a query is slow, someone
adds an index, the query gets fast, and then write throughput quietly halves — because now
every insert has to update that sorted copy too. An index is not free speed, it is a loan
against your write path.
**Seen it when.** An admin search page that used to time out suddenly works.

### Ch 16 — Transactions and isolation
**Story.** Two people withdraw from the same account at the same moment, both see a balance
of 100, both withdraw 80. Each transaction was individually correct and the database
happily committed both. This is write skew, and it survives at isolation levels most teams
assume are safe. Banks have known about it longer than computers have existed.
**Seen it when.** Two people book the last seat and both get a confirmation email.

### Ch 18 — Replication
**Story.** 21 October 2018, GitHub lost network connectivity between two east-coast data
centres for 43 seconds. Automatic failover promoted a second database, but the original one
had accepted writes nobody else had seen. Both were now "the primary" with different data.
Untangling it took over 24 hours of degraded service, from 43 seconds of network trouble.
**Guess.** How long does 43 seconds of network loss take to recover from?
**Seen it when.** A site is up but shows you stale data, or your own comment vanishes.

### Ch 19 — Partitioning and sharding
**Story.** Instagram had 30 million users and 13 employees when Facebook bought them in
2012, all on PostgreSQL. Their trick was to split the data into thousands of *logical*
shards spread across a handful of machines, so growing meant moving shards rather than
moving rows. Pinterest went further and simply banned joins.
**Guess.** How many engineers did Instagram have at 30 million users?
**Seen it when.** Your profile loads fast while someone with 40 million followers loads slow.

### Ch 20 — Object storage
**Story.** Dropbox stored everyone's files on Amazon S3 until they got big enough that the
bill justified building their own. In 2016 they moved roughly half an exabyte of user data
onto their own hardware, a project called Magic Pocket, and their public filings later put
the saving at around 75 million dollars over two years.
**Guess.** At what scale does it become cheaper to build your own storage than rent it?
**Seen it when.** A file upload shows a progress bar that survives losing your wifi.

---

## Part 3 — Caching and delivery

### Ch 23 — The cache hierarchy
**Story.** Facebook published a paper on their memcache layer describing billions of
requests per second across trillions of cached items. The striking part is what it implies:
the database is not what serves Facebook. The cache is. The database is the thing the cache
falls back to when it misses, and it could not survive being asked directly.
**Guess.** What percentage of Facebook's reads ever reach a database?
**Seen it when.** A viral post loads instantly for millions of people at once.

### Ch 25 — Invalidation and stampede
**Story.** The most reliable way to take down your own database is to clear your cache
during peak traffic. Everything that was being served from memory now arrives at a database
sized for 5% of that load, all within the same second. Facebook had to invent a lease
mechanism specifically so a thousand simultaneous misses on one hot key become one query.
**Seen it when.** A site is fine, someone deploys, and it dies for ten minutes.

### Ch 27 — CDN and the edge
**Story.** Netflix does not stream from a datacentre. They build their own appliances,
ship them to internet providers, and plug them in inside the provider's building — so when
you hit play, the movie is often already in the same city as you. At peak hours Netflix has
accounted for a large share of all downstream internet traffic in some countries, and almost
none of it crosses an ocean.
**Guess.** How much of Netflix's traffic comes from inside your own ISP?
**Seen it when.** 4K Netflix plays flawlessly while a 1080p video call stutters.

---

## Part 4 — Scaling

### Ch 28 — Vertical, horizontal and the scale cube
**Story.** Pokémon GO launched in July 2016 and hit traffic roughly fifty times what Niantic
had planned for as their worst case. Google's engineers ended up helping rewrite parts of it
live. On the other side of the same coin, Stack Overflow spent years serving enormous
traffic on about a dozen well-tuned machines, because they optimised instead of multiplying.
**Guess.** How many times over their worst-case estimate did Pokémon GO land?
**Seen it when.** A game launch where nobody can log in for three days.

### Ch 29 — Service decomposition
**Story.** Segment split their monolith into microservices, grew to over a hundred of them,
and then in 2018 published a blog post called "Goodbye Microservices" explaining why they
merged them all back into one. Not because microservices are wrong — because they had 140
separate deploy pipelines and dependency sets for a team that could not maintain them.
**Seen it when.** Any company where a one-line change needs four repositories.

### Ch 30 — Timeouts, retries, circuit breakers
**Story.** 2 July 2019, Cloudflare deployed one new firewall rule containing a regular
expression. That regex backtracked catastrophically, consumed 100% of CPU on every machine
in their global network simultaneously, and took down a large fraction of the internet's
front door for about half an hour. One line, no bad intent, no timeout on the CPU.
**Guess.** How long does it take one bad regex to break a global network?
**Seen it when.** Half the internet shows a 502 page with a cloud logo on it.

---

## Part 5 — Data in motion

### Ch 33 — Queues versus streams
**Story.** LinkedIn built Kafka because they had a growing mess of systems all needing the
same events, and every new consumer meant a new integration. By 2019 they reported it
carrying around seven trillion messages a day. The insight was small and enormous: stop
delivering messages, start keeping a log that anyone can read from wherever they like.
**Guess.** How many messages a day does LinkedIn push through Kafka?
**Seen it when.** You change a profile detail and it updates in search a moment later.

### Ch 35 — Delivery semantics and idempotency
**Story.** "Exactly once" delivery is the unicorn of distributed systems. The network can
always lose the acknowledgement after the work is done, so the sender cannot know whether to
retry. Every real system settles for at-least-once delivery plus operations that are safe to
repeat — which is why your bank has an idempotency key and not a promise.
**Seen it when.** A notification arrives twice, or an order confirmation email duplicates.

---

## Part 6 — Distributed systems truths

### Ch 39 — CAP, honestly
**Story.** Amazon's 2007 Dynamo paper made a decision that sounds insane until you see the
business behind it: adding to a shopping cart must *never* fail, even if datacentres cannot
talk to each other. The consequence is that carts sometimes had to be merged afterwards, and
a deleted item could reappear. Amazon decided a slightly wrong cart beats a cart that
refuses your money.
**Guess.** Which would you rather: a cart that occasionally resurrects an item, or a cart
that is sometimes unavailable?
**Seen it when.** An item you removed from your Amazon cart shows up again.

### Ch 41 — Time and clocks
**Story.** Google wanted globally consistent transactions across continents, which requires
knowing what happened first. Ordinary server clocks drift, so Google installed GPS receivers
and atomic clocks in their datacentres and built TrueTime — an API that returns time as an
*interval* rather than a number. Spanner literally waits out the uncertainty before
committing.
**Guess.** How wrong is a normal server's clock?
**Seen it when.** Two chat messages appear in the wrong order.

### Ch 42 — Consensus
**Story.** In October 2021 Roblox went down for 73 hours. The trigger was a newly enabled
feature in Consul, the system their services used to find each other, which caused
contention that fed on itself. The site stayed down long after the original trigger was
understood, because the coordination layer everything depended on could not get healthy
while everything was hammering it.
**Guess.** What is the longest outage a major consumer service has had?
**Seen it when.** Kubernetes, which stores its entire cluster state in a consensus system.

---

## Part 7 — Reliability and operations

### Ch 45 — Availability maths and SLOs
**Story.** Google's SRE teams ended the eternal fight between "ship features" and "keep it
stable" with arithmetic. If your target is 99.9%, you have about 43 minutes of downtime a
month to spend. Spend it and feature launches stop until you have earned it back. It turned
reliability from an argument about feelings into a budget.
**Guess.** How much downtime a month does 99.9% allow?
**Seen it when.** Any status page claiming four nines.

### Ch 46 — How systems really fail
**Story.** 28 February 2017, an AWS engineer running a routine playbook to debug S3 billing
mistyped a command and removed far more capacity than intended. S3 in North Virginia went
down for about four hours, taking a large slice of the web with it. The detail everyone
remembers: AWS could not update their own status dashboard, because the dashboard was hosted
on S3.
**Guess.** How much of the internet was hosted in that one region?
**Seen it when.** Unrelated apps all break at the same time on the same afternoon.

### Ch 47 — Resilience patterns
**Story.** Netflix's answer to "what if a server dies" was to stop asking and start killing
them on purpose. Chaos Monkey terminates random production instances during working hours,
so failures happen when engineers are awake rather than at 3 a.m. And when their
recommendation engine fails, Netflix does not show an error — it shows you a generic popular
row, and most people never notice.
**Seen it when.** Netflix's homepage looks slightly generic for a few minutes.

### Ch 49 — Shipping change safely
**Story.** 1 August 2012, Knight Capital deployed new trading code to seven of their eight
servers. The eighth ran old code that reused a configuration flag for a different purpose.
For 45 minutes it fired unintended orders into the market. The loss was around 440 million
dollars, roughly four times the company's annual profit, and Knight did not survive as an
independent firm.
**Guess.** How much can a partial deploy cost in 45 minutes?
**Seen it when.** Any feature flag, canary release, or staged rollout you have ever used.

### Ch 50 — Disaster recovery
**Story.** 31 January 2017, a GitLab engineer fighting a database problem at 11 p.m. ran a
directory deletion against the wrong server and wiped roughly 300 GB of production data.
Then they discovered that five separate backup and replication mechanisms had all failed or
were not working. They recovered from a six-hour-old staging snapshot that existed by luck,
and lost six hours of user data. They live-streamed the recovery.
**Guess.** How many of their five backup methods worked?
**Seen it when.** Every "have you tested your restore?" question you have ever been asked.

---

## Part 9 — Case studies

### Ch 60 — News feed
**Story.** Twitter's timeline works by writing a copy of every tweet into every follower's
timeline in advance, so reading is one cheap lookup. Then someone with 100 million followers
tweets, and one write becomes 100 million. The celebrity problem forced Twitter into a
hybrid: precompute for normal accounts, fetch celebrities live at read time and merge.
**Guess.** How many rows does one tweet from a huge account create?
**Seen it when.** A celebrity tweet takes a moment longer to show up than a friend's.

### Ch 61 — Chat and presence
**Story.** In 2012 WhatsApp published a post titled "1 million is so 2011" about holding
2 million simultaneous connections on a single server using Erlang and FreeBSD. Delivering
a message is not the hard part — knowing which of thousands of servers is currently holding
your friend's open socket is.
**Guess.** How many live connections can one well-tuned server hold?
**Seen it when.** A single grey tick, then two, then two blue ones.

### Ch 62 — Video platform
**Story.** When you upload one video to YouTube, it is not stored as one video. It is
transcoded into a ladder of resolutions and bitrates, chopped into segments of a few seconds
each, and those segments are what get cached worldwide. Netflix went further with per-title
encoding — a cartoon and an action film get different treatment, because flat settings waste
bandwidth on one and starve the other.
**Guess.** How many copies of your video exist after upload?
**Seen it when.** Video quality drops mid-scene and recovers without pausing.

### Ch 64 — Payments and the ledger
**Story.** The most reliable technology in modern payments was invented by Italian merchants
in the 1300s. Double-entry bookkeeping never updates a balance — it appends two matching
entries and derives the balance by adding them up. That is an append-only immutable log,
which is exactly what distributed systems reinvented six centuries later.
**Seen it when.** A refund appears as a new line in your statement, not as a corrected one.

### Ch 68 — Flash-sale commerce
**Story.** Amazon Prime Day is a self-inflicted traffic spike of a scale most companies
never see, and AWS has published that their own DynamoDB peaked at over 100 million requests
per second serving it. Ticketmaster, facing the same shape of problem for a Taylor Swift
sale, buckled. The difference is not talent — it is that one of them designed for it and one
of them was surprised by it.
**Guess.** How many database requests per second does Prime Day need?
**Seen it when.** A "you're in the queue" page that is actually protecting the system.

---

## Using a chapter with no entry here

Build one from the `In the wild` line in `curriculum.md`, but keep the shape: a named
company, a moment in time, a number, and something that went right or wrong as a result.
"Uber uses H3" is trivia. "Uber needed to find drivers near you without comparing your
coordinates to every driver in the city, so they carved the world into hexagons" is a story.
