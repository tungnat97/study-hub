[← back to the index](../README.md)

# Field 6 — Caching, Queues & Event Streaming

21 nodes. Redis and Kafka questions are near-certain for a senior backend role.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Caching and Redis](1-caching.md) | `Q01`–`Q09` | Why cache, strategies, invalidation, stampedes and hot keys, Redis fundamentals, persistence and eviction, clustering, Lua and streams, CDN and in-process caching |
| [2 — Queues and Kafka](2-queues-and-kafka.md) | `Q10`–`Q16` | Messaging fundamentals, RabbitMQ, Kafka fundamentals, durability, partitioning and ordering, consumer operations, retries and DLQ |
| [3 — Streaming and job design](3-streaming-and-job-design.md) | `Q17`–`Q21` | Stream processing and watermarks, log compaction and CDC, event schemas, choosing the right tool, job design and fairness |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `Q01` | Why and where to cache | Beginner | — | Q02, Q09 |
| `Q02` | Caching strategies | Intermediate | Q01 | Q03, Q04, F15, SD05 |
| `Q03` | Invalidation and key design | Advanced | Q02 | Q04, SD05 |
| `Q04` | Stampede, hot keys, cache failure | Advanced | Q02, Q03 | SD11, SD05 |
| `Q05` | Redis fundamentals | Intermediate | DB30, Q01 | Q06, Q07, Q08, A08 |
| `Q06` | Redis persistence and eviction | Advanced | Q05 | Q07 |
| `Q07` | Redis HA and clustering | Advanced | Q06 | SD05 |
| `Q08` | Redis advanced patterns | Advanced | Q05 | A08, C11, Q21 |
| `Q09` | HTTP, CDN and in-process caching | Intermediate | Q01, A04 | SD05 |
| `Q10` | Messaging fundamentals | Intermediate | M03, C02 | Q11, Q12, Q20, F13, M15 |
| `Q11` | RabbitMQ and classic brokers | Advanced | Q10 | Q16, Q20 |
| `Q12` | Kafka fundamentals | Advanced | Q10 | Q13, Q14, Q15, Q17, Q18 |
| `Q13` | Kafka durability | Expert | Q12, DB12 | Q15, Q16 |
| `Q14` | Partitioning, keys, ordering | Advanced | Q12, M21, M32 | Q15, Q17, SD06 |
| `Q15` | Consumer semantics and operations | Advanced | Q12, Q13, Q14, M30, C16 | Q16, Q17 |
| `Q16` | Retries, DLQ and replay | Advanced | Q11, Q13, Q15, M09, M16 | Q21, SD10 |
| `Q17` | Stream processing | Expert | Q12, Q14, Q15 | SD12 |
| `Q18` | Log compaction, CDC, outbox | Advanced | Q12, M15 | Q19, DB32 |
| `Q19` | Event schemas and evolution | Advanced | Q12, A17, M17, M24 | SD14 |
| `Q20` | Choosing the right async tool | Intermediate | Q10, Q11, Q12 | Q21, SD06 |
| `Q21` | Job design and fairness | Advanced | Q08, Q16, Q20, F13, F14 | SD10 |

## Topological order (study waves)

```
Wave 0  Q01
Wave 1  Q02  Q05  Q10
Wave 2  Q03  Q06  Q08  Q09  Q11  Q12  Q20
Wave 3  Q04  Q07  Q13  Q14  Q18  Q19
Wave 4  Q15
Wave 5  Q16  Q17
Wave 6  Q21
```

Cross-field parents: `A04` HTTP caching, `A08` rate limiting, `A17` serialisation, `C02`/`C16` I/O
and backpressure, `DB12` WAL, `DB30` Redis, `F13`/`F14` jobs, and `M03`, `M09`, `M15`, `M16`, `M17`,
`M21`, `M24`, `M30`, `M32`.

**Certain questions:** the cache-aside race and stampede (`Q02`, `Q04`); "Redis dies — what happens"
(`Q04`); at-least-once plus idempotent consumers (`Q10`, `M16`); Kafka partition, key and ordering
(`Q14`); consumer lag and rebalance storms (`Q15`); retry and DLQ design (`Q16`).
