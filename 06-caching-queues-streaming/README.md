[← back to the index](../README.md)

# Field 6 — Caching, Queues & Event Streaming

Redis and Kafka questions are near-certain for a senior backend role. Legend: `B` `I` `A` `X`.

---

## Part A — Caching

#### Q01 · Why and where to cache
`B` · Requires: — · Unlocks: Q02, Q09
- Key: caching trades **staleness for latency/throughput/cost**; the layers — client, CDN/edge,
  API gateway, in-process, distributed (Redis), database buffer pool, materialised view; locality and
  hit ratio economics (a 90% hit rate means the backend still sees 10% — size for the miss path);
  cache only what is expensive and reused; the hardest part is always invalidation.
- Q: "Where would you add a cache in this architecture and what breaks when it's stale?"
- Q: "Your cache hit rate is 99% and the DB still falls over on deploys. Why?" (cold cache /
  thundering herd — Q04).

#### Q02 · Caching strategies
`I` · Requires: Q01 · Unlocks: Q03, Q04, F15, SD05
- Key: **cache-aside/lazy loading** (app reads cache, on miss loads DB and populates — the default,
  resilient to cache failure, first request is slow, risk of stale after write), **read-through**
  (cache library owns the load), **write-through** (write both, consistent, slower writes),
  **write-behind/write-back** (fast, risk of data loss), **refresh-ahead**; negative caching for
  misses (and the unbounded-key attack it prevents); TTL vs explicit invalidation.
- Q: "Cache-aside vs write-through — which and why for a product catalogue? For a user session?"
- Q: "In cache-aside, what is the race between a concurrent read and write, and how do you fix it?"
  (read loads stale value after the write deletes it → delete-after-write + short TTL, versioned
  keys, or write-through/CDC; be able to draw the interleaving).

#### Q03 · Invalidation & key design
`A` · Requires: Q02 · Unlocks: Q04, SD05
- Key: TTL as the safety net always; explicit delete on write; **versioned keys**
  (`user:123:v7`, bump the version to invalidate a whole family) and key prefixes/namespaces;
  tenant/locale/permission in the key (or you leak data between users); never `KEYS *` in production
  (`SCAN`); cache stampede on invalidation; tag-based invalidation; the two hard problems joke, said
  properly.
- Q: "One user's data appears in another user's response. What was wrong with the cache key?"
- Q: "How do you invalidate everything derived from a product when its price changes?"

#### Q04 · Stampede, hot keys, and cache failure modes
`A` · Requires: Q02, Q03 · Unlocks: SD11, SD05
- Key: **thundering herd / cache stampede** on expiry → per-key lock/mutex ("only one loader"),
  request coalescing/singleflight, probabilistic early expiration (XFetch), stale-while-revalidate,
  never expire + background refresh; synchronised TTLs → add jitter; hot key overwhelming one shard
  → client-side local cache, key replication (`key:1..N`), consistent hashing; cache penetration
  (queries for non-existent keys) → negative cache or Bloom filter; **cache avalanche** when Redis
  dies → is your DB sized to survive it? circuit breaker to serve degraded.
- Q: "A celebrity's profile is 40% of your traffic and lives on one Redis shard. Fix it."
- Q: "Redis goes down entirely. What happens to your service?" (this is the question — answer:
  degrade, don't cascade; cache failures must be non-fatal; measure the DB's cold capacity).

#### Q05 · Redis fundamentals
`I` · Requires: DB30, Q01 · Unlocks: Q06, Q07, Q08, A08
- Key: in-memory, **single-threaded command execution** (so every command is atomic, and one slow
  command blocks everything — no `KEYS`, no big `SORT`, beware large `DEL` → `UNLINK`); data
  structures as the API: string/bitmap, hash, list, set, sorted set (skip list — leaderboards,
  delayed queues by score), HyperLogLog, geo, stream, bitmap; expiry semantics (lazy + sampled
  active expiry); `SETNX`/`SET NX PX` for locks; keyspace notifications.
- Q: "Which Redis structure for: a leaderboard / rate limiting / a session / a job queue / unique
  daily visitors?" (zset / string counter or zset / hash / list or stream / HyperLogLog).
- Q: "Redis is single-threaded — how does it do 100k ops/sec?" (in-memory, no locks, epoll event
  loop, pipelining; and Redis 6+ threads only I/O, not execution).

#### Q06 · Redis persistence, eviction & memory
`A` · Requires: Q05 · Unlocks: Q07
- Key: RDB snapshots (fork + copy-on-write, memory spike, point-in-time, fast restart) vs AOF
  (append-only, `appendfsync everysec` default, rewrite/compaction) vs both; you can still lose
  writes — Redis is not a durable system of record; `maxmemory` + eviction policies (`noeviction`,
  `allkeys-lru`, `volatile-lru`, `allkeys-lfu` — LFU is usually right for caches); memory
  fragmentation, `maxmemory-policy` interacting with locks/sessions (evicting a lock is a bug —
  separate cache and non-cache instances).
- Q: "You used the same Redis for the cache and for your distributed locks. What can go wrong?"
- Q: "`noeviction` and you hit maxmemory — what does the client see?" (write errors — decide whether
  that's better or worse than eviction for that dataset).

#### Q07 · Redis HA & clustering
`A` · Requires: Q06 · Unlocks: SD05
- Key: replication (async — failover can lose writes), Sentinel (monitoring + automatic failover for
  a single primary) vs **Cluster** (16384 hash slots, client-side slot routing, `MOVED`/`ASK`
  redirects); **multi-key operations must be in one slot** → hash tags `{user:123}:profile`;
  resharding; no cross-slot transactions or Lua across slots; client library support; managed
  options (ElastiCache/MemoryDB — MemoryDB is durable).
- Q: "Your `MGET` started failing with CROSSSLOT after moving to cluster. Fix?"
- Q: "Sentinel or Cluster for a 20GB cache? For a 500GB one?"

#### Q08 · Redis advanced patterns
`A` · Requires: Q05 · Unlocks: A08, C11, Q21
- Key: **Lua scripts** for atomic multi-step ops (rate limiters, conditional locks — scripts are
  atomic because of the single thread; keep them fast); `MULTI/EXEC` + `WATCH` (optimistic, not a
  rollback-capable transaction); pipelining to remove RTTs; pub/sub (fire and forget, no
  persistence, no consumer groups — do not use it as a queue); **Streams** (`XADD`/`XREADGROUP`,
  consumer groups, `XACK`, pending entries list, `XAUTOCLAIM` for dead consumers — an actual queue);
  distributed locks (see C11 — fencing tokens, Redlock caveats); `SET key val NX PX ttl` + a random
  token checked in Lua on release.
- Q: "Write an atomic token-bucket rate limiter in Redis. Why does it need Lua?"
- Q: "Why is `DEL lock` on release a bug?" (you may delete someone else's lock after your TTL
  expired — compare-and-delete in Lua).

#### Q09 · HTTP, CDN & in-process caching
`I` · Requires: Q01, A04 · Unlocks: SD05
- Key: CDN edge caching, cache keys and `Vary`, purging vs versioned URLs, origin shielding,
  `stale-while-revalidate`/`stale-if-error` at the edge; in-process caches (LRU with a **bounded**
  size — the #1 Node memory leak), per-pod inconsistency with N replicas, two-level cache (local +
  Redis) and the invalidation broadcast problem (pub/sub to evict locally).
- Q: "You added a 60s in-memory cache and users see data flip-flop between old and new. Why?"

---

## Part B — Queues & Streaming

#### Q10 · Messaging fundamentals
`I` · Requires: M03, C02 · Unlocks: Q11, Q12, Q20, F13, M15
- Key: why async — decoupling, load levelling/buffering, retries, fan-out, spikes; queue (work
  distribution, one consumer per message) vs topic/pub-sub (fan-out) vs log (replayable, many
  independent readers); delivery semantics: at-most-once, **at-least-once** (the practical default →
  consumers must be idempotent), exactly-once (only within a closed system); ack/nack, redelivery,
  visibility timeout, ordering guarantees and their scope; the message is a contract (M24).
- Q: "Your consumer crashes after doing the work but before acking. What happens and how do you make
  it safe?"
- Q: "Queue or event log for this use case?" (work distribution vs multiple independent consumers
  needing history/replay).

#### Q11 · RabbitMQ / classic brokers
`A` · Requires: Q10 · Unlocks: Q16, Q20
- Key: exchanges (direct, topic, fanout, headers) → bindings → queues; routing keys; durable queues +
  persistent messages + publisher confirms (all three needed for durability); **prefetch/QoS** as the
  fairness and backpressure knob; manual ack; dead-letter exchanges with TTL for delayed retry;
  ordering only per queue with one consumer; mirrored/quorum queues; the broker holds state so a
  deep queue is a broker problem; competing consumers pattern.
- Q: "Messages pile up and RabbitMQ memory alarms trigger. What do you tune first?" (prefetch,
  consumer count, lazy queues, TTL/DLQ, and fix the slow consumer).
- Q: "How do you implement a 5-minute delayed retry in RabbitMQ?" (TTL queue + DLX, or the delayed
  message plugin).

#### Q12 · Kafka fundamentals
`A` · Requires: Q10 · Unlocks: Q13, Q14, Q15, Q17, Q18
- Key: distributed **append-only log**; topic → partitions → ordered immutable records with offsets;
  brokers, leaders and followers per partition; producers choose the partition (key hash or
  round-robin); consumer **groups** (one partition to one consumer in a group — so partition count
  caps parallelism); offsets stored in `__consumer_offsets`; retention by time/size independent of
  consumption (replay!); pull-based consumers; zero-copy and sequential I/O explain the throughput;
  KRaft replacing ZooKeeper.
- Q: "Why can't you have more consumers than partitions in a group?"
- Q: "Kafka vs RabbitMQ — pick one for order events and one for sending emails, and justify."

#### Q13 · Kafka durability & delivery guarantees
`X` · Requires: Q12, DB12 · Unlocks: Q15, Q16
- Key: replication factor, **ISR** (in-sync replicas), `acks=0/1/all`, `min.insync.replicas`
  (RF=3 + acks=all + min.isr=2 is the standard durable config), unclean leader election = data loss,
  leader epoch and log truncation; producer retries + `enable.idempotence=true` (dedup by producer
  id + sequence number, prevents duplicates from retries) and `max.in.flight<=5` for ordering;
  transactions (`transactional.id`, read-process-write atomicity, `read_committed` consumers) and
  their scope — **inside Kafka only**.
- Q: "Give the exact producer/broker config for 'we must never lose an order event' and say what it
  costs." (latency, and unavailability when a broker is down with min.isr=2).
- Q: "Explain Kafka's exactly-once and where it stops working." (the moment you call an external API
  or a non-Kafka DB → idempotency).

#### Q14 · Partitioning, keys & ordering
`A` · Requires: Q12, M21, M32 · Unlocks: Q15, Q17, SD06
- Key: ordering is **per partition**, so the key determines both ordering scope and parallelism;
  key by aggregate id (order id, user id) to keep an entity's events ordered; hot partitions from
  skewed keys (a big tenant) → composite keys/salting + re-aggregation; partition count is hard to
  increase (it changes key→partition mapping and breaks ordering) so over-provision modestly;
  rebalancing cost; null key = round robin = no ordering.
- Q: "Events for one order arrive out of order. Diagnose." (no key / key changed / partition count
  changed / consumer parallelism inside the handler).
- Q: "One tenant produces 80% of the traffic. Fix the hot partition."

#### Q15 · Consumer semantics & operations
`A` · Requires: Q12, Q13, Q14, M30, C16 · Unlocks: Q16, Q17
- Key: poll loop, `max.poll.records`/`max.poll.interval.ms` (slow processing → the consumer is
  kicked out → rebalance → duplicates: the classic bug), auto-commit vs manual commit **after**
  processing (at-least-once) vs before (at-most-once); rebalance protocols (eager vs cooperative
  sticky, static membership to survive restarts); **consumer lag** as the primary SLI; scaling
  consumers up to partition count then out via more partitions; parallelising inside a consumer
  breaks ordering unless you key-partition the work; poison messages.
- Q: "Your consumer processes each message in 5 seconds and the group rebalances constantly. Fix."
- Q: "How do you monitor a streaming pipeline?" (lag per partition, processing time, DLQ rate,
  end-to-end event latency).

#### Q16 · Retries, DLQ & replay
`A` · Requires: Q11, Q13, Q15, M09, M16 · Unlocks: Q21, SD10
- Key: classify errors — transient (retry with backoff) vs poison (DLQ immediately) vs
  downstream-down (pause/circuit-break, don't burn retries); retry **topics** with increasing delay
  in Kafka (you cannot nack a single offset without blocking the partition); DLQ must carry the
  original payload, headers, error and attempt count; the DLQ needs an owner, an alert and a
  documented replay procedure; replay safety requires idempotent consumers (M16); avoid infinite
  redelivery loops.
- Q: "Design retry + DLQ for a Kafka consumer that calls a flaky third-party API."
- Q: "Your DLQ has 40k messages from an outage. How do you replay them safely?"

#### Q17 · Stream processing
`X` · Requires: Q12, Q14, Q15 · Unlocks: SD12
- Key: stateless (map/filter) vs **stateful** (aggregations, joins) with a local state store +
  changelog topic; event time vs processing time, windows (tumbling, hopping, sliding, session),
  **watermarks** and late/out-of-order data, allowed lateness; stream-table duality; KTable and
  compacted topics; Kafka Streams vs Flink vs ksqlDB; exactly-once via transactions within the
  topology; reprocessing by resetting offsets.
- Q: "Count orders per merchant per hour, with events arriving up to 10 minutes late. Design it."
- Q: "Event time vs processing time — give a bug caused by using the wrong one."

#### Q18 · Log compaction, CDC and the outbox
`A` · Requires: Q12, M15 · Unlocks: Q19, DB32
- Key: compacted topics keep the latest value per key (a durable changelog/snapshot — the basis of
  KTables and of publishing reference data); tombstones for deletes (and GDPR); **CDC with
  Debezium** reading the WAL/binlog → topics, with snapshot + streaming phases; outbox pattern via
  CDC (M15); schema of the change event (before/after/op); ordering guarantees from the DB log.
- Q: "Compare outbox-with-poller and Debezium CDC for publishing order events."
- Q: "How does a compacted topic let a new consumer bootstrap its state?"

#### Q19 · Event schema & evolution
`A` · Requires: Q12, A17, M17, M24 · Unlocks: SD14
- Key: schema registry, Avro/protobuf/JSON-schema, compatibility modes (BACKWARD — new consumer
  reads old data, FORWARD — old consumer reads new data, FULL, TRANSITIVE) and which one you need
  depending on who upgrades first; add optional fields with defaults, never remove/rename/retype;
  event envelope (id, type, version, occurred_at, correlation/causation id, tenant, payload); topic
  naming and ownership; documenting events as a product.
- Q: "Producers deploy before consumers. Which compatibility mode do you need and why?" (BACKWARD:
  the new consumer must read old messages — but if producers go first with new data and old
  consumers are still running, you need FORWARD too → FULL in practice).

#### Q20 · Choosing the right async tool
`I` · Requires: Q10, Q11, Q12 · Unlocks: Q21, SD06
- Key: task queue (BullMQ/Celery/Sidekiq — jobs, retries, scheduling, per-job visibility, easy) vs
  broker (RabbitMQ/SQS — routing, decoupling, at-least-once) vs log (Kafka/Kinesis/Pulsar — replay,
  many consumers, high throughput, ordering per key, stream processing) vs DB-backed queue
  (`FOR UPDATE SKIP LOCKED` — no new infrastructure, transactional with your data, scales far further
  than people think) vs workflow engine (Temporal — durable execution, long-running sagas, retries
  and state built in); SQS standard vs FIFO (message groups, dedup id, 300 tps/group).
- Q: "You need background email sending for a Nest app with 200 rps. What do you use and why not
  Kafka?"
- Q: "When does a Postgres-backed queue stop being enough?"

#### Q21 · Job design, scheduling and idempotency
`A` · Requires: Q08, Q16, Q20, F13, F14 · Unlocks: SD10
- Key: jobs carry ids not snapshots; idempotency key/dedup table per job; make jobs small and
  restartable; visibility timeout > max processing time (or you get duplicates); max attempts + DLQ;
  priority and fairness across tenants (per-tenant queues/weighted consumption to stop one tenant
  starving others); delayed jobs (zset by score, SQS delay, DLX-TTL); scheduled jobs at scale
  (leader election / K8s CronJob / Temporal); batch windows and backfills with throttling; monitoring
  (queue depth, age of the oldest message — **age beats depth as an alert**).
- Q: "One tenant enqueues 2M jobs and everyone else's emails stop. Redesign."
- Q: "Which single metric would you alert on for a job queue?" (oldest-message age / lag, not depth).

---

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

Cross-field parents: `A04` HTTP caching, `A08` rate limiting, `A17` serialisation, `C02/C16`
I/O and backpressure, `DB12` WAL, `DB30` Redis, `F13/F14` jobs, `M03/M09/M15/M16/M17/M21/M24/M30/M32`.

**Certain questions:** cache-aside race and stampede (Q02/Q04), "Redis dies — what happens" (Q04),
at-least-once + idempotent consumers (Q10/M16), Kafka partition/key/ordering (Q14), consumer lag and
rebalance storms (Q15), retry/DLQ design (Q16).
