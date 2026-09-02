# Curriculum — 72 chapters, 10 parts

Each entry is a one-session chapter. `Needs` lists prerequisite chapters; warn once if the
learner jumps past one, then teach anyway if they insist. `In the wild` names the systems
that must appear in beat 7 — add more, never fewer.

Chapter numbers are stable. Never renumber; the progress file references them.

---

## Part 0 — Ground floor (1-5)

**1 · The path of one request** — what actually happens between a click and a pixel?
Components: browser · DNS resolver · TLS handshake · CDN edge · load balancer · app server · cache · database · disk
Mechanism: every hop's job, what state lives where, where the milliseconds go, and which hops can fail independently
In the wild: trace a static asset path against a logged-in dashboard path and compare which hops each one touches
Needs: —

**2 · Latency, and why distance is the boss**
Components: CPU cache · RAM · SSD · disk · same-rack, same-DC, cross-region and cross-ocean network
Mechanism: the latency table every engineer carries; tail latency and why p99 not p50 defines user experience; how a fan-out of 100 calls makes your p99 the common case
Numbers: RAM read ~100 ns · SSD read ~100 µs · same-DC round trip ~0.5 ms · cross-country ~40 ms · transatlantic ~80 ms
In the wild: Google's tail-at-scale work · the reason a CDN exists at all
Needs: 1

**3 · Throughput, concurrency and queues**
Components: thread pool · connection pool · request queue · admission control
Mechanism: Little's Law (L = λW) and using it to size a pool; the utilization-versus-latency curve and why latency explodes past roughly 70-80% utilization; queueing delay as most of your p99; coordinated omission hiding it from your benchmark
In the wild: connection-pool exhaustion, probably the single most common production outage · Netflix's adaptive concurrency limits
Needs: 2

**4 · Back-of-envelope estimation**
Components: DAU → QPS → peak QPS → storage per day → bandwidth → cache size → machine count
Mechanism: 86,400 seconds a day; peak is 2-10x average; read/write ratio drives the whole architecture; the 80/20 hot set sizes your cache; identify the single number that decides the design
In the wild: sizing a feed's fanout writes · sizing a video platform's egress bill, which is usually the real constraint
Needs: 2, 3

**5 · How to draw an HLD**
Components: actors · trust boundaries · services · datastores · queues · external dependencies
Mechanism: three zoom levels — context, container, component; drawing the read path and write path separately; marking single points of failure and blast radius; what belongs in the diagram versus the prose
In the wild: what a design review looks for · what an interviewer scores on the whiteboard
Needs: 1

---

## Part 1 — The request path (6-12)

**6 · DNS, anycast and service discovery**
Components: recursive resolver · authoritative nameserver · TTL · anycast IP · service registry · health-based deregistration
Mechanism: the resolution chain; TTL as a failover lever and a failover trap; GeoDNS versus anycast; client-side versus server-side discovery; why DNS is a poor load balancer but is used as one anyway
In the wild: Route 53 latency and failover routing · Cloudflare anycast · Airbnb's SmartStack · Consul and etcd registries · the Facebook 2021 outage where a BGP withdrawal made the DNS servers unreachable and locked staff out of their own tools
Needs: 1

**7 · Connections: TCP, TLS, HTTP/1.1 → 2 → 3**
Components: three-way handshake · TLS handshake · connection pool · keep-alive · ALPN
Mechanism: why a cold HTTPS connection costs 2-3 round trips before any data; head-of-line blocking in HTTP/1.1 and again in TCP itself; HTTP/2 multiplexing; QUIC over UDP in HTTP/3; session resumption and 0-RTT
Numbers: a cold TLS connection across an ocean spends roughly 240 ms on handshakes alone, which is why pooling is not an optimisation but a requirement
In the wild: Google's QUIC rollout · gRPC riding on HTTP/2 · mobile apps holding connections open
Needs: 2, 6

**8 · Load balancers**
Components: L4 NLB · L7 ALB/Envoy/NGINX · health checker · target group · connection draining · slow start
Mechanism: L4 forwards by 4-tuple and knows nothing about your request; L7 parses HTTP and routes by path, header or cookie; round-robin versus least-connections versus consistent hash versus power-of-two-choices; TLS termination and where the certificate lives; active versus passive health checks and the flapping problem
In the wild: Google Maglev, built on commodity servers with ECMP and consistent hashing · Cloudflare Unimog · Envoy, created at Lyft · HAProxy fronting Stack Overflow's famously small fleet
Needs: 7

**9 · Gateways, proxies and the BFF**
Components: API gateway · reverse proxy · forward proxy · backend-for-frontend · sidecar
Mechanism: pulling cross-cutting concerns out of every service — auth, rate limits, quotas, transformation, protocol translation, API composition; the gateway as a deploy chokepoint and a shared failure domain; when a BFF per client beats one universal API
In the wild: Netflix Zuul and its async rewrite · Kong and Apigee · Envoy serving as both edge proxy and sidecar · GraphQL BFFs at Netflix
Needs: 8

**10 · Statelessness, sessions and identity on the wire**
Components: stateless app tier · session store · cookie · JWT · refresh token
Mechanism: why in-process state kills horizontal scaling and complicates every deploy; sticky sessions and what they cost you; server-side sessions in Redis versus self-contained JWTs; the revocation problem a JWT creates; token lifetime as a trade of security against lookup load
In the wild: session-in-Redis as the safe default · JWTs for service-to-service calls · the logout that does not actually log anyone out
Needs: 8

**11 · Rate limiting and throttling**
Components: counter store · limiter middleware · quota configuration · `Retry-After` response
Mechanism: fixed window and its boundary burst; sliding window log versus sliding window counter; token bucket for burst tolerance; leaky bucket for smoothing; local approximate counters versus a central store and the round trip that costs; what to do when the limiter itself is down
In the wild: Stripe's published rate-limiter design and its load shedding by request class · Cloudflare edge limits · GitHub's `X-RateLimit` headers · AWS burst credits
Needs: 10, 26

**12 · API design: REST, gRPC, GraphQL, streaming**
Components: endpoint · schema or IDL · pagination cursor · idempotency key · version
Mechanism: resource modelling; cursor versus offset pagination and why offset collapses at depth; idempotency keys so a retry is safe; backward-compatible versioning; when a binary contract beats JSON; GraphQL's N+1 and query-cost problems; WebSocket versus SSE versus long polling
In the wild: Stripe's idempotency keys and dated API versions · gRPC between Google services · GraphQL at GitHub and Shopify · SSE carrying LLM token streams
Needs: 7

---

## Part 2 — Storage (13-22)

**13 · What a database does with your row**
Components: page · buffer pool · B+ tree · write-ahead log · checkpoint · fsync
Mechanism: rows live in fixed-size pages; reads pull pages into the buffer pool; writes hit the log first and the data pages later; `fsync` is the actual durability boundary; random versus sequential IO, and why an append-only log is the fastest thing on disk
Numbers: a B+ tree three or four levels deep indexes hundreds of millions of rows, so a point lookup is a few page reads, nearly all of them cached
In the wild: Postgres heap plus WAL · MySQL InnoDB's clustered index and redo log
Needs: 2

**14 · LSM trees and write-optimised stores**
Components: memtable · immutable SSTable · commit log · compaction · bloom filter
Mechanism: buffer writes in memory, flush sorted runs to disk, merge them in the background; the write / read / space amplification triangle where you pick two; bloom filters to avoid reading SSTables that cannot contain the key; compaction as the hidden source of latency spikes and IO cost
In the wild: RocksDB and LevelDB · Cassandra and ScyllaDB · the storage layer beneath CockroachDB and TiDB
Needs: 13

**15 · Indexes**
Components: clustered/primary index · secondary index · composite index · covering index
Mechanism: an index is a sorted copy that buys read speed with write cost and space; the leftmost-prefix rule for composite indexes; selectivity and cardinality deciding whether the planner uses it at all; covering indexes avoiding the table lookup; every index taxing every write
In the wild: the "we added an index and write throughput halved" incident every team has once
Needs: 13

**16 · Transactions, isolation and MVCC**
Components: transaction manager · lock manager · version chain · snapshot
Mechanism: ACID stated precisely; read-uncommitted → read-committed → repeatable-read → serializable, and the specific anomaly each still permits — dirty read, non-repeatable read, phantom, write skew; MVCC giving readers a snapshot instead of a lock; optimistic versus pessimistic concurrency; deadlock detection
In the wild: Postgres MVCC and the vacuum it requires · MySQL gap locks · the write-skew bug that survives repeatable read and silently breaks a business invariant
Needs: 13

**17 · SQL versus NoSQL, as a decision procedure**
Components: relational · document · key-value · wide-column · graph
Mechanism: start from access patterns, never from the data shape; what needs a transaction, what needs ad-hoc querying, how it will partition, and what the largest single entity is; normalisation versus denormalisation as a read/write trade; schema-on-write versus schema-on-read and who pays for the flexibility
In the wild: Postgres carrying far more load than people assume · DynamoDB single-table design driven entirely by access patterns · Neo4j when the traversal is the query
Needs: 15, 16

**18 · Replication**
Components: leader · follower · replication log · lag monitor · failover controller
Mechanism: single-leader, multi-leader and leaderless; synchronous versus asynchronous versus semi-synchronous, and exactly what data you lose on failover in each; replication lag producing read-your-writes violations and stale reads; read replicas as a scaling tool and the ceiling they hit
In the wild: Postgres streaming replication · Amazon Aurora replicating the log rather than the pages · the GitHub 2018 outage where a 43-second partition caused a split-brain and a full day of degraded service
Needs: 13

**19 · Partitioning and sharding**
Components: shard key · router · shard map · rebalancer
Mechanism: range versus hash versus directory partitioning; consistent hashing with virtual nodes, and why plain modulo forces a full reshuffle; hot partitions and celebrity keys; local versus global secondary indexes; resharding without downtime; the price of any query that crosses shards
In the wild: Instagram's logical shards on Postgres · Vitess sharding MySQL under YouTube · Pinterest's sharded MySQL with joins simply forbidden · DynamoDB adaptive capacity fighting hot partitions
Needs: 18

**20 · Object storage and the blob path**
Components: bucket · object · multipart upload · presigned URL · storage class · lifecycle policy
Mechanism: objects are immutable blobs with metadata, not files; erasure coding for durability; presigned URLs so the bytes never pass through your app servers; hot, warm and cold tiers and their retrieval cost; the metadata in a database, the bytes in the store, and keeping those two consistent
Numbers: S3 advertises eleven nines of durability; egress and requests, not storage, are usually the bill
In the wild: Facebook Haystack for photos and f4 for warm blobs · Dropbox's Magic Pocket migration off S3 once its scale justified the move
Needs: 4

**21 · Search and the inverted index**
Components: analyser · inverted index · segment · shard and replica · relevance scorer
Mechanism: documents become tokens become posting lists, answering "which documents contain this" in the direction a B+ tree cannot; TF-IDF and BM25 scoring; near-real-time indexing via segment flush and merge; n-grams and fuzzy matching for typos; search as a derived store you must be able to rebuild from scratch
In the wild: Lucene under Elasticsearch and OpenSearch · Twitter's Earlybird · search indexes fed by change data capture rather than dual writes
Needs: 15

**22 · Specialised stores**
Components: time-series · geospatial · graph · vector · append-only ledger
Mechanism: time-series compresses timestamp/value pairs and downsamples history; geospatial indexes such as geohash, S2 or H3 flatten 2D proximity into a 1D range scan; graph stores index the edge rather than the row; vector stores need approximate nearest neighbour (HNSW, IVF) because exact search does not scale; ledgers append and derive balances instead of updating them
In the wild: Facebook Gorilla's in-memory TSDB · Prometheus · Uber's H3 hexagonal grid · pgvector and Pinecone · double-entry ledgers in payments
Needs: 17

---

## Part 3 — Caching and delivery (23-27)

**23 · The cache hierarchy**
Components: browser cache · CDN edge · gateway cache · application cache · in-process cache · database buffer pool
Mechanism: each layer's hit rate, staleness window, and blast radius when it holds something wrong; the closer to the user, the cheaper the hit and the harder the invalidation; cache hit ratio as the number that decides how big your database has to be
In the wild: a static asset served from an edge PoP versus a personalised page that can only be cached per user
Needs: 2

**24 · Cache patterns**
Components: cache-aside · read-through · write-through · write-behind · refresh-ahead
Mechanism: who owns the miss, who owns the write, and what a crash loses under each; cache-aside as the default, and its race between a read miss and a concurrent write; write-behind's durability gap; negative caching so misses are not free to repeat
In the wild: cache-aside with Redis as the industry default · write-through inside CDN origin shields
Needs: 23

**25 · Invalidation, eviction, stampede and hot keys**
Components: TTL · explicit invalidation · versioned key · lease · eviction policy
Mechanism: TTL versus event-driven invalidation, and why mature systems use both; LRU versus LFU versus TTL-only; the stampede when a hot key expires and a thousand requests reach the database at once, solved with per-key locks, leases or probabilistic early expiry; hot keys defeating your sharding; versioned keys turning invalidation into a write rather than a delete
In the wild: Facebook's memcache leases from the Scaling Memcache paper · mcrouter · the classic "we flushed the cache and took down the database" incident
Needs: 24

**26 · Redis and memcached in production**
Components: single-threaded event loop · data structures · 16,384 hash slots · replica · AOF and RDB persistence
Mechanism: why single-threaded is fast, and why one slow command stalls everything; what each data structure is actually for — sorted sets for leaderboards, hashes for objects, streams for queues; cluster resharding; persistence options and the honest conclusion that a cache should not be a source of truth; `KEYS` as a production-ending command
In the wild: Redis for sessions, counters, rate limits and leaderboards · Netflix EVCache built on memcached across zones · timeline caches at Twitter
Needs: 25

**27 · CDN and the edge**
Components: PoP · edge cache · origin shield · origin · cache key · purge API · edge compute
Mechanism: anycast routing to the nearest PoP; cache key design, and how one stray query parameter destroys your hit rate; `Cache-Control`, `stale-while-revalidate` and ETag revalidation; origin shield collapsing the miss stampede; purge versus versioned URLs; dynamic content at the edge; adaptive bitrate video as ordinary cacheable segments
In the wild: Netflix Open Connect appliances placed inside ISP networks · Cloudflare Workers · Fastly's instant purge · HLS and DASH segments
Needs: 23

---

## Part 4 — Scaling the architecture (28-32)

**28 · Vertical, horizontal and the scale cube**
Components: bigger machine · more machines · functional split · data split
Mechanism: the scale cube — clone on X, split by function on Y, split by data on Z; vertical scaling being the correct first answer more often than anyone admits; autoscaling on the right signal, which is queue depth or concurrency rather than CPU; the minutes an autoscaler takes that you may not have
In the wild: Stack Overflow serving enormous traffic on a handful of tuned machines · Kubernetes HPA on custom metrics
Needs: 3, 4

**29 · Service decomposition and data ownership**
Components: service boundary · privately owned datastore · anti-corruption layer
Mechanism: split by business capability and rate of change, not by technical layer; one service owns its data and nobody else touches it directly; the distributed monolith you get when two services share a database; the bill you accept — network calls, partial failure, distributed debugging, versioned contracts; when a monolith is still the right answer
In the wild: Amazon's API mandate and two-pizza teams · Segment's well-documented migration from microservices back to a monolith
Needs: 12

**30 · Talking between services without falling over**
Components: timeout · retry with exponential backoff and jitter · circuit breaker · bulkhead · service mesh
Mechanism: every remote call needs a timeout shorter than its caller's; retries multiply load exactly when the system is already struggling; the circuit breaker's closed, open and half-open states; bulkheads so one slow dependency cannot eat the whole thread pool; the mesh moving all of this out of application code
In the wild: Netflix Hystrix and its successors · Envoy and Istio sidecars · the Cloudflare 2019 outage where one unbounded regex consumed every CPU core globally
Needs: 29

**31 · Multi-region and geo-distribution**
Components: global load balancer · regional stack · cross-region replication · data residency boundary
Mechanism: active-passive versus active-active and the write-conflict problem the second creates; latency-based and geo routing; read-local, write-global as the common compromise; replication lag across oceans; data residency as a hard architectural constraint rather than a legal footnote; failover you have actually rehearsed
In the wild: Spanner buying global consistency with TrueTime commit-wait · DynamoDB global tables settling conflicts by last-writer-wins · GDPR forcing EU-resident data
Needs: 18, 27

**32 · Cells, blast radius and shuffle sharding**
Components: cell · cell router · shuffle-sharded assignment · quarantine
Mechanism: partition the entire stack, not just the data, so a failure takes one cell instead of the service; shuffle sharding gives each tenant a random subset of workers, so no two tenants share an identical set and one poisoned tenant's blast radius collapses; per-cell deploys as an implicit canary
Numbers: with 8 workers and 2 assigned per tenant there are 28 distinct combinations, so one abusive tenant fully overlaps roughly 1 in 28 others
In the wild: AWS cell-based architecture and Route 53's shuffle sharding · Slack and Shopify pod models
Needs: 19, 29

---

## Part 5 — Data in motion (33-38)

**33 · Queues versus streams**
Components: producer · broker · queue or topic · consumer · visibility timeout · offset
Mechanism: a task queue deletes the message once handled and lets you scale consumers freely; a log retains the record and lets many independent consumers replay it; competing consumers versus consumer groups; which ordering guarantee you actually get; when the honest answer is neither and you should just call the service
In the wild: SQS and RabbitMQ for work dispatch · Kafka as the durable log, built at LinkedIn · SQS FIFO's throughput ceiling as the price of ordering
Needs: 12

**34 · Kafka internals**
Components: topic · partition · offset · leader and follower replica · in-sync replica set · consumer group · coordinator
Mechanism: a partition is an append-only file, which is the whole reason it is fast; ordering exists per partition and nowhere else, making the partition key a design decision; partition count caps consumer parallelism; `acks=all` plus ISR defines durability; rebalancing pauses consumption; retention versus log compaction; consumer lag as the one health metric that matters
In the wild: Kafka as the change-data backbone of most modern data platforms · compacted topics as a changelog you can rebuild state from
Needs: 33

**35 · Delivery semantics and idempotency**
Components: idempotency key · dedup store · offset commit · dead-letter queue
Mechanism: at-most-once, at-least-once, and why exactly-once is really at-least-once plus idempotent effects or a transactional boundary; committing the offset before or after processing, and the duplicate or the loss each choice causes; natural idempotency keys; dedup windows and their storage cost; poison messages and DLQ replay; ordering versus parallelism as a direct trade
In the wild: Stripe's idempotency key on every mutating call · Kafka's idempotent producer and transactions · the double-charge bug, which is always a missing idempotency key
Needs: 34

**36 · Outbox, CDC and saga**
Components: outbox table · relay or CDC connector · event bus · saga orchestrator · compensating action
Mechanism: the dual-write problem, because you cannot atomically commit a row and publish an event; the outbox pattern making the event part of the same transaction; CDC reading the replication log instead, producing events with no application change; sagas replacing a distributed transaction with local commits plus compensations; orchestration versus choreography and who owns the failure path
In the wild: Debezium reading the MySQL binlog and the Postgres WAL · order/payment/inventory sagas · search and cache indexes fed by CDC rather than dual writes
Needs: 16, 35

**37 · Stream processing**
Components: source · operator · state store · window · watermark · sink
Mechanism: stateful computation over an unbounded stream; tumbling, sliding and session windows; event time versus processing time, and the late-data problem watermarks exist to bound; stream-stream and stream-table joins; checkpointing to make state exactly-once; backpressure through the topology
In the wild: Flink carrying Alibaba's peak sale traffic · Kafka Streams · the lambda architecture, and Kappa's argument against maintaining the same logic twice
Needs: 34

**38 · Batch, OLAP and the analytics plane**
Components: warehouse · lake · lakehouse · columnar file format · orchestrator · MPP query engine
Mechanism: row stores and column stores as opposite designs for opposite questions; columnar layout plus compression plus predicate pushdown; separating storage from compute so they scale independently; ETL versus ELT; the small-file problem in a lake; keeping analytics off the transactional database
In the wild: Snowflake and BigQuery separating storage and compute · Parquet and Iceberg · Airflow orchestration · the analyst query that takes down production, which is a boundary failure not a query failure
Needs: 20

---

## Part 6 — Distributed systems truths (39-44)

**39 · CAP and PACELC, honestly**
Components: partition · availability · consistency
Mechanism: CAP applies only during a partition, and the real question is what a node does when it cannot reach its peers — refuse the write, or accept it and reconcile later; PACELC adds the part that matters every day, that even without a partition you trade latency against consistency; why CP or AP is a per-operation decision and not a label on a database
In the wild: one system running a CP path for payments and an AP path for likes, which is what real architectures look like
Needs: 18

**40 · Consistency models**
Components: linearizable · sequential · causal · read-your-writes · monotonic reads · eventual
Mechanism: each model as a rule about what a reader is allowed to observe; eventual consistency as a promise about the end that says nothing about the middle; the session guarantees that fix real user-visible bugs; quorum reads and writes with R + W > N; read repair and anti-entropy; which model costs an extra round trip and where
In the wild: "I posted a comment and it vanished on refresh" as a missing read-your-writes guarantee · Dynamo-style tunable quorums
Needs: 39

**41 · Time, clocks and ordering**
Components: wall clock · monotonic clock · NTP · Lamport clock · vector clock · hybrid logical clock
Mechanism: clock skew makes latest-timestamp-wins quietly lossy; monotonic clocks for durations, wall clocks for display, and neither for ordering across machines; logical clocks capturing happens-before; vector clocks detecting concurrency instead of hiding it; TrueTime turning uncertainty into a bounded interval and simply waiting it out
In the wild: silent last-writer-wins data loss in Cassandra · Spanner's TrueTime commit-wait · CockroachDB's hybrid logical clocks
Needs: 40

**42 · Consensus and quorums**
Components: leader · follower · replicated log · term or epoch · quorum · lease
Mechanism: making N machines agree on one ordered log despite failures; Raft's leader election, log replication and majority commit; needing 2f+1 nodes to survive f failures, and why an even cluster size buys nothing; consensus being expensive enough that you use it for metadata and coordination, not for every write; leases as cheap, short-lived agreement
In the wild: etcd under Kubernetes · ZooKeeper's ZAB under HBase and older Kafka · Google Chubby · Consul
Needs: 41

**43 · Distributed transactions, and why you avoid them**
Components: coordinator · participant · prepare phase · commit phase · compensating transaction
Mechanism: two-phase commit gives atomicity across services but blocks every participant if the coordinator dies, while holding locks across the network; three-phase commit does not really fix it; the practical answer is to move the boundary so the transaction becomes local, and use sagas plus idempotency for whatever is left; TCC as the middle ground
In the wild: XA transactions in older enterprise stacks, remembered mainly for their operational pain · order-to-payment sagas as the modern default
Needs: 36, 42

**44 · Failure detection, split-brain, fencing and locks**
Components: heartbeat · phi-accrual detector · gossip · quorum · fencing token · lease
Mechanism: a slow node and a dead node are indistinguishable, so every detector trades false positives against detection time; gossip spreading membership without a central registry; split-brain when two nodes both believe they lead, which is why a write needs a fencing token the storage layer can reject when stale; why a plain Redis lock is a performance optimisation and not a correctness mechanism
In the wild: Cassandra's gossip and phi-accrual detector · the GitHub 2018 split-brain · the Redlock debate, which is really an argument about fencing
Needs: 42

---

## Part 7 — Reliability and operations (45-50)

**45 · Availability maths and SLOs**
Components: SLI · SLO · SLA · error budget · dependency chain
Mechanism: nines converted into real downtime; components in series multiplying, so every added dependency drags availability down — the actual mathematical argument against gratuitous microservices; redundancy in parallel; error budgets turning reliability into a spendable number that settles the ship-versus-stabilise argument; choosing SLIs a user can feel
Numbers: 99.9% is roughly 43 minutes a month; five services at 99.9% in series is about 99.5%, or three and a half hours
In the wild: Google SRE's error budget policy · cloud SLAs that refund credits worth far less than the outage
Needs: 3

**46 · How systems really fail**
Components: cascading failure · retry storm · thundering herd · gray failure · metastable state
Mechanism: real outages are a small trigger plus a feedback loop that sustains itself after the trigger is gone, which is why the system stays down after you fix the original cause; queues growing faster than they drain; retries amplifying load at the worst moment; a degraded node that still passes health checks; correlated failure through a shared dependency nobody drew on the diagram
In the wild: the AWS S3 2017 outage from a mistyped capacity command · Facebook's 2021 BGP withdrawal that locked out the tooling needed to recover · Roblox's 73-hour outage driven by a consensus-layer feedback loop · the metastable-failure literature
Needs: 30, 45

**47 · Resilience patterns**
Components: backoff with jitter · load shedding · admission control · backpressure · hedged request · graceful degradation
Mechanism: jitter to break the synchronised retry wave; shedding by priority before the whole system collapses, because a fast 503 is kinder than a slow timeout; backpressure pushing the limit upstream instead of buffering forever; hedged requests trading a little extra work for a much better tail; degradation paths decided in advance, per feature, not improvised during the incident
In the wild: Google's tail-at-scale hedging · Netflix serving a static fallback row when the recommender is down · Stripe shedding low-priority traffic first
Needs: 46

**48 · Observability**
Components: metric · log · trace · exemplar · dashboard · alert · burn-rate rule
Mechanism: metrics say something is wrong, traces say where, logs say why; RED for services and USE for resources; cardinality as the cost driver that ambushes every team; head versus tail sampling; alerting on user-visible symptoms and SLO burn rate instead of on CPU; one correlation ID threaded through every hop
In the wild: Prometheus's pull model and its cardinality limits · OpenTelemetry as the vendor-neutral wire format · Facebook Gorilla trading retention for in-memory speed
Needs: 45

**49 · Shipping change without breaking production**
Components: blue-green · canary · feature flag · expand-contract migration · backfill job · rollback plan
Mechanism: separating deployment from release so a flag turns the feature on after the code is already out; canary with automated metric comparison and automatic rollback; the expand-contract sequence for a schema change — add the column, dual-write, backfill, switch reads, stop writing the old, drop it — because a rename is a distributed transaction spanning code and data; throttled backfills; every migration needing a rehearsed way back
In the wild: Knight Capital losing 440 million dollars in 45 minutes to a partial deploy and a reused flag · gh-ost and pt-online-schema-change for online schema changes
Needs: 16, 29

**50 · Disaster recovery and chaos**
Components: RPO · RTO · backup · point-in-time recovery · runbook · game day
Mechanism: RPO is the data you can afford to lose and RTO is the time you can afford to be down, and together they drive the architecture and its cost; a backup you have never restored is not a backup; PITR from a snapshot plus the log; DR tiers and their honest price; chaos experiments with a hypothesis and a bounded blast radius; a runbook someone can follow at 3 a.m.
In the wild: Netflix Chaos Monkey and the wider Simian Army · the GitLab 2017 incident where several independent backup mechanisms were all found broken at the same time
Needs: 31, 46

---

## Part 8 — Security, cost, judgment (51-54)

**51 · Authentication, authorisation and secrets**
Components: identity provider · OAuth2/OIDC flow · access and refresh token · RBAC or ABAC engine · mTLS · secret manager
Mechanism: keeping authentication and authorisation separate; authorisation-code-with-PKCE as the current default and why the implicit flow died; short-lived access tokens with revocable refresh tokens; coarse authorisation at the gateway and fine-grained inside the service; workload identity and mTLS between services; rotation, and the blast radius when a key leaks
In the wild: OIDC behind most "sign in with" buttons · SPIFFE and SPIRE for workload identity · Vault and cloud KMS
Needs: 10

**52 · Data protection, privacy and multi-tenancy**
Components: TLS in transit · encryption at rest · envelope encryption · tokenisation · PII inventory · tenant isolation boundary
Mechanism: encryption at rest defends against a stolen disk and very little else, so scope the claim honestly; envelope encryption enabling key rotation without re-encrypting everything; tokenising card and identity data to shrink compliance scope; the right-to-erasure problem once data has fanned out into caches, backups, logs and the warehouse; shared-schema versus schema-per-tenant versus database-per-tenant, and the noisy-neighbour and blast-radius trade between them
In the wild: PCI scope reduction through tokenisation · GDPR deletion pipelines · shared-schema SaaS multi-tenancy versus per-tenant databases for enterprise customers
Needs: 51

**53 · Cost engineering**
Components: compute · storage class · network egress · cross-AZ traffic · managed-service premium · idle capacity
Mechanism: cost per request as a first-class design metric; egress and cross-zone traffic as the line items that surprise everyone; storage tiering and lifecycle rules; the real trade between a managed service and the team you would need to run it yourself; over-provisioning as the price of a slow autoscaler; caching justified in dollars as well as milliseconds
In the wild: chatty services chatting across availability zones and outspending their own compute · Dropbox moving off S3 once its scale justified owning the hardware
Needs: 4, 28

**54 · Defending a design**
Components: requirements table · trade-off table · risk register · rollout plan
Mechanism: the design review as an argument you win with reasoning rather than vocabulary; naming the constraint behind every choice; stating what you deliberately did not build; presenting the simplest version first and the scaling path second; answering "why not X" with the condition under which X becomes correct; the red flags reviewers and interviewers listen for
In the wild: what separates a mid-level answer from a staff-level answer on the identical architecture
Needs: Parts 0-8

---

## Part 9 — Case studies (55-68)

Build each one in layers: naive version, break it with a number, add only the component that
fixes it. Never reveal the finished architecture in one shot.

**55 · URL shortener** — the smallest complete system
Focus: ID generation, the redirect path, and a 100:1 read/write ratio
Components: ID generator · key-value store · edge cache · redirect service · async analytics pipeline
Hard parts: base62 encoding versus hashing and collisions, custom aliases, 301 versus 302 and what a cached redirect costs your analytics, expiry, abuse
Needs: 12, 19, 24

**56 · Distributed unique IDs**
Focus: uniqueness without coordination
Components: Snowflake-style generator · machine ID assignment · clock guard · ticket server · UUIDv7
Hard parts: the 64-bit layout of timestamp, machine and sequence; handling clock rollback; monotonicity for index locality; why random UUIDv4 primary keys wreck B+ tree write performance; sortability as a feature
In the wild: Twitter Snowflake · Instagram generating IDs inside Postgres with the shard ID embedded · UUIDv7 as the modern default
Needs: 41, 15

**57 · Rate limiter as a service**
Focus: correctness under distribution, and the cost of the check itself
Components: edge counter · central Redis · Lua script for atomicity · quota configuration store · fail-open policy
Hard parts: accuracy versus added latency, per-user versus per-IP versus per-key, local approximate counters syncing to a central store, and deciding fail-open or fail-closed when the limiter itself dies
Needs: 11, 26

**58 · Web crawler**
Focus: politeness, prioritisation and deduplication at enormous scale
Components: frontier queue · DNS cache · fetcher pool · content-seen store · parser · URL normaliser · robots cache
Hard parts: per-host politeness delays, priority and recrawl scheduling, near-duplicate detection with simhash, crawler traps and infinite URL spaces, bloom filters for the seen set, distributing the frontier without hotspots
Needs: 33, 19

**59 · Typeahead**
Focus: sub-100 ms suggestions under enormous read volume
Components: trie or FST index · precomputed top-k per prefix · edge cache · log-driven update pipeline
Hard parts: precompute versus query-time ranking, index size and sharding by prefix, freshness for trending terms, typo tolerance, and how personalisation destroys a shared cache
Needs: 21, 25

**60 · News feed**
Focus: fanout, the celebrity problem, and ranking
Components: post service · follow graph · fanout workers · per-user timeline cache · ranking service · media CDN
Hard parts: fanout-on-write versus fanout-on-read and the hybrid every real system converges on, one celebrity write becoming a hundred million, pagination over a shifting feed, ranking freshness, backfill when someone follows a new account
In the wild: Twitter's hybrid timeline · Instagram's ranked feed · Facebook TAO fronting MySQL for the social graph
Needs: 19, 25, 33

**61 · Chat and presence**
Focus: long-lived connections, ordering, and delivery you can prove
Components: WebSocket gateway · connection registry · message store · fanout service · offline inbox · push notification bridge
Hard parts: finding which server holds a given user's socket, per-conversation ordering by sequence number rather than timestamp, delivery and read receipts, group fanout, gap recovery after a reconnect, presence accuracy versus its cost, and what end-to-end encryption removes from the server
In the wild: WhatsApp holding millions of connections on a famously small Erlang fleet · Discord's message store moving MongoDB → Cassandra → ScyllaDB · Slack's edge cache for client boot
Needs: 12, 34, 41

**62 · Video platform**
Focus: the write path is a pipeline, the read path is a CDN
Components: resumable upload service · object storage · transcoding farm · manifest generator · CDN · metadata database · view counter
Hard parts: chunked resumable upload, transcoding fanout into an adaptive bitrate ladder and its compute cost, HLS/DASH manifests and segment caching, view-count accuracy against write volume, storage tiering for the long tail nobody watches, and how much tighter live streaming's budget is
In the wild: YouTube on Vitess for metadata · Netflix Open Connect appliances inside ISPs · per-title encoding to cut bitrate
Needs: 20, 27, 33

**63 · Ride-hailing**
Focus: geospatial indexing plus a state machine that must never double-book
Components: location ingest · geospatial index · matching and dispatch service · trip state machine · pricing service · payment integration
Hard parts: driver location updates as a write firehose, proximity search via H3 or S2 rather than coordinate comparison, preventing double assignment of one driver, trip state transitions as the durable core of the system, surge pricing, regional partitioning and the trips that cross a boundary
In the wild: Uber's H3 hexagonal grid and its dispatch design · geosharding by city
Needs: 22, 35, 44

**64 · Payments and the ledger**
Focus: correctness over availability, and never charging twice
Components: payment API with idempotency keys · double-entry ledger · outbox · provider adapters · reconciliation job · settlement batch
Hard parts: the authorise/capture/refund lifecycle, an immutable append-only ledger with derived balances, what happens when your database commits and the provider call then times out, reconciliation against provider statements, currency and rounding, audit and retention
In the wild: Stripe's idempotency keys · double-entry accounting as the oldest and still the best consistency trick in the business
Needs: 16, 35, 36

**65 · Notification platform**
Focus: multi-channel fanout with preferences, deduplication and rate control
Components: event ingest · preference service · template service · per-channel workers · provider adapters · dedup store · digest scheduler
Hard parts: quiet hours and channel preferences, deduplication and bundling so one event does not become five notifications, provider failure and fallback order, retrying without spamming, delivery tracking, and priority lanes so a one-time passcode never queues behind a marketing blast
Needs: 33, 35

**66 · Collaborative editing**
Focus: concurrent edits converging without a lock
Components: document store · operation log · transform or merge engine · presence channel · snapshot compactor
Hard parts: operational transformation versus CRDTs and the honest cost of each, intention preservation, cursor and selection presence, reconciling offline edits, snapshotting so a document does not replay a million operations, and undo inside a shared document
In the wild: Google Docs using OT · Figma's custom last-writer-wins-per-property model with the server as authority · Yjs and Automerge as CRDT implementations
Needs: 40, 41

**67 · Metrics platform at scale**
Focus: ingesting billions of points and still querying fast enough to alert
Components: agent · ingest gateway · time-series store · downsampler · query engine · alert evaluator · long-term object storage
Hard parts: cardinality explosion as the defining problem, delta-of-delta and XOR compression, pull versus push collection, retention tiers and rollups, evaluating alerts without re-querying everything, per-tenant limits
In the wild: Facebook Gorilla's compression · Prometheus with Thanos or Cortex behind it · the label someone set to a user ID that multiplied the bill
Needs: 22, 48

**68 · Flash-sale commerce**
Focus: a 10,000x traffic spike against strictly finite inventory
Components: virtual waiting room · inventory reservation service · queue · idempotent order service · payment integration · cache warmer
Hard parts: never overselling while never blocking on a lock, reservation with a TTL versus decrement-on-order, optimistic concurrency on the stock row, the hot key when the entire internet wants one item, bot and scalper defence, fairness, and queueing gracefully instead of returning 503
In the wild: Ticketmaster-style waiting rooms · Shopify's flash-sale pod architecture · seat locking in booking systems
Needs: 25, 32, 47

---

## Part 10 — Electives (69-72)

**69 · Machine-learning platforms**
Focus: training and serving as two different systems that must agree
Components: offline and online feature store · training pipeline · model registry · serving layer · shadow and A/B infrastructure · drift monitoring
Hard parts: train/serve skew from two implementations of one feature, online feature freshness, batch versus real-time inference, rolling a model back, evaluating in production
Needs: 38, 47

**70 · LLM serving systems**
Focus: why GPU inference breaks ordinary web-serving assumptions
Components: token streaming API · request scheduler · continuous batching · KV cache · vector store · retrieval pipeline · semantic cache
Hard parts: prefill and decode as two different workloads, KV cache memory as the real capacity limit, continuous batching trading per-request latency for throughput, semantic caching, cost per token as an SLO, and queueing when GPUs cannot autoscale in seconds
Needs: 22, 47

**71 · Job scheduling and workflow engines**
Focus: running work later, once, reliably
Components: schedule store · leader-elected dispatcher · execution queue · workflow state machine · retry policy · idempotent tasks
Hard parts: cron at scale without duplicate firing, distributed timers, long-running workflows surviving a deploy, catch-up after downtime, dependency graphs, and visibility into a stuck job
In the wild: Airflow DAGs · Temporal's durable execution · Kubernetes CronJob's duplicate-run edge cases
Needs: 42, 44

**72 · Low-level design mini-track**
Focus: the same rigour one zoom level down, inside a single service
Components: entities · value objects · interfaces · state machines · concurrency primitives
Hard parts: modelling invariants so illegal states cannot be represented, reaching for a pattern only when it removes a real conditional, thread safety around shared mutable state, testing state transitions. Run through parking lot, elevator, rate limiter, a cache with TTL, and a booking system
Needs: 12

---

## Suggested tracks

Pick one at placement and say so; the learner should know the shape of the road.

| Track | Chapters |
|-------|----------|
| Full course | 1 → 72 in order |
| Interview sprint | 4, 5, 8, 12, 15, 17, 18, 19, 23-25, 33, 35, 39, 40, 45, then 55, 60, 61, 62, 63, 64, then 54 |
| My service keeps falling over | 3, 30, 45, 46, 47, 48, 49 |
| We are outgrowing one database | 13, 15, 16, 18, 19, 36, 38 |
| Event-driven and data platform | 33, 34, 35, 36, 37, 38, 40 |
| Interview is tomorrow | 4, 5, 54, plus one case study closest to the company's domain |
