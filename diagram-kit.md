# Diagram kit

Text diagrams, always inside a fenced code block so the alignment survives. Maximum 78
characters wide. A diagram nobody can read teaches nothing, so check the width before
sending.

## Shapes

```
 ┌──────────┐    stateless service or worker — can be killed and replaced
 │ order svc│
 └──────────┘

 ╭──────────╮    traffic layer — load balancer, gateway, router, CDN edge
 │  L7 LB   │
 ╰──────────╯

 ╔══════════╗    durable store — database, object store, log, queue with
 ║ Postgres ║    retention. Double line means the data survives a restart.
 ╚══════════╝

 ┌╌╌╌╌╌╌╌╌╌╌┐    volatile store — cache, in-memory index. Dashed means
 ╎  Redis   ╎    losing it is survivable but expensive.
 └╌╌╌╌╌╌╌╌╌╌┘
```

External systems you do not operate get the shape that fits plus `(3rd party)` in the label.
That distinction matters, because you cannot fix them during an incident.

## Edges

```
 ──▶     synchronous call, caller blocks
 ╌╌▶     asynchronous — event, message, background job
 ◀──▶    long-lived bidirectional connection (WebSocket, gRPC stream)
```

Label every edge with what actually travels on it and what it costs:
`──HTTP/2, p99 8ms──▶`, `╌╌Kafka orders.v1, 12 partitions╌╌▶`,
`──replication, async, ~200 ms lag──▶`.

An unlabelled arrow is a guess wearing a diagram's clothing.

## Rules

- Number the hops `(1) (2) (3)` and follow the diagram with the path written out in prose.
  If a hop needs no prose, it does not need to be on the diagram.
- Put real sizes in the boxes: `Redis Cluster, 6 shards, 64 GB, LRU`, not `cache`.
- Draw the read path and the write path separately when they differ. They usually differ,
  and merging them is how a diagram becomes unreadable.
- Say which zoom level you drew: **L0** actors and one system box, **L1** services and
  stores, **L2** inside one service.
- Build case studies in layers. Draw the naive version, break it with a number, then add
  only the component that fixes it. The finished picture should be the last thing they see,
  not the first.

---

## D1 — Baseline three-tier

The version everything else is a departure from. Draw this first in almost every chapter.

```
 ┌────────┐  (1) HTTPS   ╭───────────╮  (2) HTTP   ┌────────────┐
 │ Client │ ───────────▶ │  L7 LB    │ ──────────▶ │  app  x3   │
 └────────┘              │  Envoy    │             │  stateless │
                         ╰───────────╯             └──────┬─────┘
                                        (3) SQL p99 2ms   │
                                                          ▼
                                              ╔═══════════════════╗
                                              ║ Postgres primary  ║
                                              ║   + 2 replicas    ║
                                              ╚═══════════════════╝
```

## D2 — Cache-aside read path

```
 ┌────────┐   (1) GET    ┌──────────────┐  (2) GET user:42
 │ Client │ ───────────▶ │    app       │ ────────────────▶ ┌╌╌╌╌╌╌╌╌╌╌╌┐
 └────────┘              │              │ ◀──────────────── ╎  Redis    ╎
                         │              │  (3) MISS         ╎  TTL 300s ╎
                         │              │                   └╌╌╌╌╌╌╌╌╌╌╌┘
                         │              │  (5) SETEX             ▲
                         │              │ ───────────────────────┘
                         └──────┬───────┘
                     (4) SELECT │  ~4 ms
                                ▼
                        ╔═══════════════╗
                        ║   Postgres    ║
                        ╚═══════════════╝
```

Hit path: (1) (2) return, about 0.4 ms. Miss path: (1) (2) (3) (4) (5) return, about 6 ms.
At a 95% hit rate the database sees 5% of read traffic; at 90% it sees double that. The hit
rate, not the cache size, is the lever.

## D3 — Async write path with outbox and CDC

```
 ┌────────┐  (1) POST /orders    ┌──────────────┐
 │ Client │ ───────────────────▶ │  order svc   │
 └────────┘ ◀─────────────────── │              │
             (3) 202 Accepted    └──────┬───────┘
                          (2) one txn:  │  order row + outbox row
                                        ▼
                              ╔══════════════════╗
                              ║    Postgres      ║
                              ╚════════╤═════════╝
                       (4) CDC reads   ╎  the WAL — no app change
                                       ▼
                              ╔══════════════════╗
                              ║ Kafka orders.v1  ║
                              ║  12 partitions   ║
                              ╚═══╤══════════╤═══╝
                        (5) ╌╌╌╌╌╌┘          └╌╌╌╌╌╌ (5)
                            ▼                       ▼
                    ┌───────────────┐      ┌────────────────┐
                    │  email wkr    │      │ inventory wkr  │
                    └───────────────┘      └────────────────┘
```

The client gets its answer at (3), before any of the work at (5) has happened. Everything
after the commit is at-least-once, so both workers must be idempotent.

## D4 — Sharded datastore

```
                          ┌──────────────┐
                          │     app      │
                          └──────┬───────┘
             logical_shard = hash(user_id) % 1024
                                 │
                          ╭──────▼───────╮
                          │ shard router │  owns the shard map
                          ╰──┬───┬───┬───╯
              ┌──────────────┘   │   └──────────────┐
              ▼                  ▼                  ▼
      ╔═══════════════╗  ╔═══════════════╗  ╔═══════════════╗
      ║  shard 0-341  ║  ║ shard 342-682 ║  ║shard 683-1023 ║
      ║  pg-a + rep   ║  ║  pg-b + rep   ║  ║  pg-c + rep   ║
      ╚═══════════════╝  ╚═══════════════╝  ╚═══════════════╝
```

1024 logical shards spread over 3 physical nodes. Adding a fourth node moves logical shards,
not individual rows, and that indirection is the entire trick. Any query that does not carry
`user_id` becomes a scatter-gather across every node.

## D5 — Fanout on write versus on read

```
 FANOUT ON WRITE                    FANOUT ON READ
 ┌──────────┐                       ┌──────────┐
 │  post    │ 1 write               │  post    │ 1 write
 └────┬─────┘                       └────┬─────┘
      ▼                                  ▼
 ╔══════════╗                       ╔══════════╗
 ║  posts   ║                       ║  posts   ║   (that's all)
 ╚════╤═════╝                       ╚══════════╝
      ╎ fanout worker
      ╎ N inserts, N = followers    read = query every followee,
      ▼                                    merge, rank, page
 ┌╌╌╌╌╌╌╌╌╌╌╌╌┐
 ╎ timeline   ╎  read = one lookup
 ╎ per user   ╎
 └╌╌╌╌╌╌╌╌╌╌╌╌┘

 write O(followers), read O(1)      write O(1), read O(followees)
```

Real systems run both: fanout-on-write for ordinary accounts, fanout-on-read for the handful
of accounts with tens of millions of followers, merged at query time.

## D6 — CDN and origin shield

```
 ┌────────┐   ┌────────┐   ┌────────┐      each PoP, ~10 ms from user
 │ user EU│   │ user US│   │ user IN│
 └───┬────┘   └───┬────┘   └───┬────┘
     ▼            ▼            ▼
 ╭────────╮   ╭────────╮   ╭────────╮
 │PoP FRA │   │PoP IAD │   │PoP BOM │  edge cache, hit ratio ~95%
 ╰───┬────╯   ╰───┬────╯   ╰───┬────╯
     └────────────┼────────────┘   (1) miss
                  ▼
            ╭──────────────╮
            │ origin shield│  collapses N PoP misses into 1 fetch
            ╰──────┬───────╯
                   ▼  (2) origin fetch, ~120 ms
            ┌──────────────┐        ╔═══════════════╗
            │  origin app  │ ─────▶ ║ object store  ║
            └──────────────┘        ╚═══════════════╝
```

Without the shield, one cache expiry sends every PoP to the origin at the same moment. That
is the stampede of chapter 25, at global scale.

## D7 — Multi-region, active-active reads

```
        ╭─────────────────────────────────────────╮
        │  global LB — latency-based routing      │
        ╰────────────┬───────────────┬────────────╯
                     ▼               ▼
        ┌──────────────────┐  ┌──────────────────┐
        │  REGION eu-west  │  │  REGION us-east  │
        │  ┌────────────┐  │  │  ┌────────────┐  │
        │  │  app tier  │  │  │  │  app tier  │  │
        │  └─────┬──────┘  │  │  └─────┬──────┘  │
        │        ▼         │  │        ▼         │
        │ ╔═════════════╗  │  │ ╔═════════════╗  │
        │ ║ replica (r) ║◀─┼──┼─║ PRIMARY(r/w)║  │
        │ ╚═════════════╝  │  │ ╚═════════════╝  │
        └──────────────────┘  └──────────────────┘
             async replication, ~90 ms lag
```

Reads are local and fast. Writes cross the ocean. A user who writes in EU and immediately
reads in EU hits a replica that has not caught up yet — that is the read-your-writes
violation from chapter 40, and it needs sticky routing or a read-from-primary rule, not a
bigger replica.

## D8 — Failure annotation

Mark the failure on the same diagram rather than describing it in prose.

```
 ┌────────┐      ╭───────────╮      ┌────────────┐
 │ Client │ ───▶ │  L7 LB    │ ───▶ │  app  x3   │
 └────────┘      ╰───────────╯      └──────┬─────┘
                                           ▼
                                   ┌╌╌╌╌╌╌╌╌╌╌╌╌┐
                                   ╎ Redis  ✗   ╎  ← fails at 14:02
                                   └╌╌╌╌╌╌╌╌╌╌╌╌┘
                                           ┆ every read now falls through
                                           ▼
                                  ╔══════════════════╗
                                  ║ Postgres  ⚠ SPOF ║  30k QPS arrives
                                  ║ capacity 2k QPS  ║  at a 2k QPS box
                                  ╚══════════════════╝
                                  ░░ blast radius: all reads, all users ░░
```

`✗` the component that failed, `⚠ SPOF` the one with no redundancy, and state the blast
radius in users or features, never as "some impact".

## D9 — L2 zoom, inside one service

Use when the chapter is about what happens within a process rather than between machines.

```
                     ┌─────────────────────────────────────┐
   inbound ─────────▶│ ╭─────────╮  ┌────────┐  ┌────────┐ │
   HTTP/2            │ │ accept  │─▶│ worker │─▶│ conn   │─┼──▶ DB
                     │ │ queue   │  │ pool   │  │ pool   │ │
                     │ │ depth 512│  │  32   │  │  10    │ │
                     │ ╰─────────╯  └────────┘  └────────┘ │
                     └─────────────────────────────────────┘
      Little's Law: 32 workers ÷ 40 ms per request = 800 req/s max.
      Beyond that the accept queue grows and latency is pure waiting.
```

The connection pool of 10 behind 32 workers is the bug in this picture. Ask the learner to
find it before pointing at it.
