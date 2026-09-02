# Decision rules

The transferable part. An architecture memorised is worth little; these procedures let the
learner derive one for a system nobody has taught them.

Teach the rule alongside the decision, every time. "We chose Redis here" is trivia. "We
chose Redis because 200,000 counter increments per second would lock-contend a MySQL row,
and the count may lag by a minute" is a rule they can reuse tomorrow.

---

## Picking a datastore

Ask these five, in order. The answers pick the store; the store does not pick itself.

1. **What is the access pattern?** Point lookup by key, range scan, full-text search,
   aggregate over millions of rows, or traverse relationships. This is the single most
   important question and most people skip it.
2. **Does anything need a transaction across rows?** If yes, that pushes hard toward
   relational.
3. **How large is one item, and how many are there?** Megabyte-plus items do not belong in
   a database at all.
4. **What is the read:write ratio?** 1000:1 and 1:1 are different systems.
5. **Can it be rebuilt from somewhere else?** If yes, you may choose something fast and
   lossy, because losing it is an inconvenience rather than an incident.

### The default, and the pressure that moves you off it

**Start at PostgreSQL or MySQL.** Move only when a named pressure appears. If nobody can
name the pressure, the answer is still Postgres.

| Pressure | Move to | Because |
|---|---|---|
| Items are megabytes or larger | Object storage (S3) | Blobs evict everything useful from the buffer pool and make backups unrestorable |
| One row takes thousands of writes/sec | Redis counter, batch-flushed | Row locks serialise; memory does not |
| Full-text or ranked search | Elasticsearch / OpenSearch | An inverted index answers "which docs contain X"; a B+ tree cannot |
| Writes exceed one primary | Shard it (Vitess), then consider Cassandra | Sharded SQL keeps your query language; wide-column trades it for write throughput |
| Append-heavy, read-recent (chat, events) | Cassandra / ScyllaDB (LSM) | LSM buffers writes in memory; B+ trees do random IO per insert |
| "Near me" queries | Geospatial index — H3, S2, geohash | Comparing coordinates is a full scan; a grid cell is a range scan |
| Scans over millions of rows for analytics | Columnar warehouse (Snowflake, BigQuery) | Row stores read whole rows to answer one column |
| Similarity over embeddings | Vector store with ANN (HNSW) | Exact nearest-neighbour does not scale |
| Relationship traversal is the query | Graph store | Indexing the edge beats joining the row repeatedly |

**Choosing NoSQL before answering question 1 is the most common design-interview red flag.**

## When to cache

Both conditions, or do not cache:

- The read:write ratio for **that specific field** is high
- Staleness is tolerable for **that specific field**

Then state four things or the cache is not designed: the **key**, the **TTL**, the
**invalidation trigger**, and **what happens when it is cold**. That last one is where
outages come from — a cache flush at peak sends full traffic to a database sized for 5%.

Cache-aside is the default. Reach for anything else only with a reason.

## When to make work asynchronous

Async when both hold: the user does not need the result to continue, and the work is safe
to retry. Anything retried must be idempotent, which usually means an idempotency key.

Keep it synchronous when the user cannot act without the answer — a payment
authorisation, a login, a seat reservation.

## When to shard

When one primary cannot hold the write rate or the dataset. Not before — sharding costs you
joins, transactions, and every operational procedure you had.

**Choosing the key:** take the key present in your highest-QPS query. Then immediately name
the query that becomes a scatter-gather across all shards, and decide whether to denormalise
a second table to serve it. A shard key with no named victim query means the analysis was
not done.

Watch for the celebrity key — one value taking a disproportionate share of traffic defeats
the whole scheme.

## Queue or log

- **One logical consumer, delete after handling** → task queue (SQS, RabbitMQ)
- **Several independent consumers, or you need replay** → log (Kafka)

Ordering exists per partition and nowhere else, so the partition key is a design decision,
not a default.

## Strong or eventual consistency

Decide **per operation**, never once for the whole system. The same product runs both.

- **Strong:** money, inventory, seat and slot booking, anything a user can exploit twice
- **Eventual:** view counts, feeds, presence, recommendations, search freshness

"The database is eventually consistent" is not an answer. "The balance read is strong, the
transaction list is eventual" is.

## Sizing anything

Daily volume → per second → peak multiplier (2-10x) → storage per day and per year →
bandwidth. 86,400 seconds in a day. 1 Mbps sustained for an hour is about 450 MB.

Finish with the one number that drives the architecture. Every system has exactly one, and
naming it is most of the design.

---

## How to use these in a lesson

After any technology choice, say the rule out loud in one sentence. After a design, ask the
learner to apply one rule to a **different** system — that is the moment the knowledge
becomes portable.

> "We used Redis for view counts because a hot counter row lock-contends. Where else in
> this design does that same rule apply?"
