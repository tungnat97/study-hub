[← back to the index](../README.md)

# Field 9 — System Design & Senior Signals (the capstone)

Every node here consumes the other eight fields. This is where the offer is decided: the design
round tests whether you can *use* the knowledge under ambiguity, and the behavioural round tests
whether you can be trusted with scope. Legend: `B` `I` `A` `X`.

---

## Part A — The design interview

#### SD01 · Requirements, scoping and the first 5 minutes
`I` · Requires: — · Unlocks: SD02, SD03, SD15
- Key: never start drawing. Establish: who are the users and what are the top 2-3 use cases;
  functional scope (explicitly cut things — "I'll leave analytics out unless you want it");
  **non-functional** — scale (DAU, RPS read/write, data size, growth), latency targets, consistency
  requirements, availability target, retention, compliance; constraints (team, existing stack,
  build vs buy). Write them on the board and get agreement. Restate before designing.
- Q: "Design Twitter." → your first move is 6-8 clarifying questions, then a written requirement list.
- Signal: interviewers score "drove the conversation and set the scope" heavily.

#### SD02 · Estimation
`I` · Requires: SD01, C14, C15 · Unlocks: SD03, SD04, SD06
- Key: DAU → requests/day → **average RPS** (÷86400 ≈ ÷100k) → peak (x2-10); storage = rows/day x
  bytes x retention x replication; bandwidth = RPS x payload; memory for a cache = hot set x size;
  round aggressively and say so; sanity anchors worth memorising — a modern server handles ~10-50k
  simple rps, one Postgres primary comfortably does low thousands of simple writes/sec, one Kafka
  partition ~10MB/s, one Redis ~100k ops/sec, disk seek vs SSD vs memory vs network latencies
  (memory ~100ns, SSD ~100µs, same-DC RTT ~0.5ms, cross-continent ~100ms).
- Q: "1M DAU posting 2 items a day with 100:1 read ratio. Size the system." (do this out loud)

#### SD03 · High-level design
`I` · Requires: SD01, SD02, M01, M03 · Unlocks: SD04, SD05, SD06, SD07
- Key: a box diagram with clients → edge/CDN → gateway/LB → services → data stores → async
  pipeline; name each component's responsibility and the protocol on each arrow; keep it simple
  first and *then* add complexity where a requirement forces it; state your default stack and be
  ready to defend it; walk the primary use case end-to-end through the diagram before optimising.
- Q: "Walk the write path, then the read path." (say this yourself — it structures the whole round)

#### SD04 · Data model & storage choice
`A` · Requires: SD03, DB01, DB26, M22 · Unlocks: SD05, SD06, SD08
- Key: define the entities, the access patterns **first**, then choose the store per access pattern
  (polyglot but justified); schema sketch with the key indexes; where the source of truth lives and
  what is derived (cache, search index, warehouse) and how derived stores stay in sync (CDC/outbox);
  hot vs cold data and retention/archival.
- Q: "Why Postgres here and DynamoDB there?" (answer in terms of access patterns, scale and
  operational familiarity — not fashion)

#### SD05 · Scaling reads
`A` · Requires: SD04, DB23, Q02, Q04, Q09, A04, M19 · Unlocks: SD09, SD12
- Key: the ladder in order — index/query fix → connection pooling → read replicas → application
  cache (cache-aside + stampede protection) → CDN/edge for cacheable responses → denormalised read
  models/materialised views → precomputation. Each step's consistency cost. Read-your-writes
  handling at each level.
- Q: "Reads are 100x writes and the DB is at 90% CPU. Give me your ordered plan."

#### SD06 · Scaling writes
`A` · Requires: SD02, SD04, DB25, Q14, Q20, DB32 · Unlocks: SD09, SD10
- Key: batching and bulk writes, async ingestion through a queue/log (accept fast, process later —
  and what that means for the user's expectation of "saved"), write coalescing, partitioning/
  sharding by a well-chosen key, avoiding hot rows (counters → sharded counters or a stream +
  periodic aggregation), append-only designs, separating the OLTP write path from analytics,
  backpressure and shedding when ingest exceeds capacity.
- Q: "1M writes/sec of telemetry. Design the ingest path." (LB → stateless collectors → Kafka
  partitioned by device → stream processor → columnar store; talk about ordering, retention, cost)
- Q: "A single 'likes' counter on a viral post. Fix the hot row."

#### SD07 · Availability & failure design
`A` · Requires: SD03, M10, M31, M33, M35, O20 · Unlocks: SD13, SD15
- Key: enumerate failure modes per component and say what happens to the user in each; redundancy
  (N+1, multi-AZ), removing single points of failure, graceful degradation and feature-level kill
  switches, timeouts/retries/circuit breakers at each hop, idempotency so retries are safe, bounded
  queues, health checks and automatic replacement, chaos testing, and the availability maths of a
  dependency chain.
- Q: "Walk through what happens to this design when the cache dies / a region dies / the payment
  provider is down for 2 hours."

#### SD08 · Consistency decisions
`A` · Requires: SD04, M12, DB07 · Unlocks: SD10, SD13
- Key: decide **per feature**, not per system: what must be strongly consistent (money, inventory
  decrement, uniqueness of a username) versus what can be eventually consistent (feed, counters,
  search index, analytics); how the UI hides lag (optimistic updates, return the written entity,
  version tokens); where you need a transaction versus a saga; where you accept duplicates and
  reconcile.
- Q: "Which parts of your design are eventually consistent and how would a user notice?"

#### SD09 · Skew, hot spots and fan-out
`A` · Requires: SD05, SD06, Q14, M32 · Unlocks: SD12
- Key: the **celebrity problem**; fan-out on write (precompute each follower's feed — fast reads,
  expensive for huge followings) vs fan-out on read (cheap writes, slow reads) vs the hybrid
  (push for normal users, pull for celebrities at read time — the standard answer); hot partition
  mitigation (salting, dedicated shards, local caching); tenant skew; power-law distributions being
  the norm, not the exception.
- Q: "Design a news feed for 300M users where some have 100M followers."

#### SD10 · Correctness in user-facing flows (payments, inventory, bookings)
`X` · Requires: SD06, SD08, M14, M16, A09, DB38, Q21, C11 · Unlocks: SD15
- Key: idempotency keys end to end, exactly-once *effect* via dedup + conditional writes, saga with
  compensations across the payment provider, reservation/hold with TTL for inventory and seats,
  optimistic vs pessimistic locking for the decrement, double-entry ledger and reconciliation jobs,
  handling the unknown outcome (timeout on a charge → query the provider, never blind-retry),
  webhooks from the provider being out of order and at-least-once, audit trail.
- Q: "Design ticket booking for a 50,000-seat venue with 500k people hitting refresh." (queue/waiting
  room, hold with TTL, atomic seat claim via a unique constraint or `SELECT FOR UPDATE`, idempotent
  payment, release on abandonment, oversell policy)
- Q: "Your charge request timed out. What do you do next?"

#### SD11 · Rate limiting & multi-tenant fairness at scale
`A` · Requires: A08, Q04, S13, M29 · Unlocks: SD13
- Key: where to enforce (edge vs gateway vs service), distributed counters and their accuracy/cost
  trade (local buckets + periodic sync vs central Redis + Lua), per-tenant quotas and weighted fair
  queueing, burst allowance, what the client sees (429 + Retry-After + headers), and protecting
  shared resources (DB connections, worker pools) from a single noisy tenant.
- Q: "Design a rate limiter for a public API with 10k customers on different plans."

#### SD12 · Canonical design patterns to have pre-solved
`A` · Requires: SD05, SD09, M23, A15, A16, A21, DB31, Q17 · Unlocks: SD16
- Key: **feed/timeline** (fan-out hybrid, ranking, pagination cursors); **chat** (WS gateway +
  presence + per-conversation ordering + fanout + offline delivery + read receipts); **notification
  system** (multi-channel, templating, preferences, dedup, digest, per-provider fallback,
  rate limits); **search** (indexing pipeline via CDC, relevance, filters, pagination);
  **file storage** (presigned upload, chunking, metadata service, dedup by hash, CDN);
  **URL shortener** (id generation, redirect latency, analytics pipeline);
  **metrics/analytics pipeline** (ingest → stream → columnar → query); **scheduler/cron at scale**;
  **ride-hailing/geo matching** (geohash/quadtree, proximity, state machine).
- Q: any of the above, cold. Have the 2-minute skeleton for each memorised.

#### SD13 · Geography, tenancy and regulation
`X` · Requires: SD07, SD08, SD11, M29, M31, O11, O20, S15 · Unlocks: SD15
- Key: latency-driven placement (edge, read replicas, regional caches), active-passive vs
  active-active and the write-conflict problem, data residency/sovereignty forcing regional
  partitioning by tenant, cross-region replication lag, global uniqueness without a global
  transaction, regional failover and DNS/anycast, and the fact that "multi-region active-active" is
  a consistency decision before it is an infrastructure one.
- Q: "EU customer data must stay in the EU. How does that change your design?"

#### SD14 · Migration, rollout and backfill
`A` · Requires: SD03, DB22, M27, M28, O09, Q19 · Unlocks: SD15
- Key: you are almost always changing a live system, and interviewers love this: expand/contract,
  dual writes with a read-from-old → compare → read-from-new sequence, shadow traffic and parity
  checking, throttled backfills with resumable checkpoints, feature flags per cohort, rollback at
  each stage, and how you *verify* the migration (row counts, checksums, reconciliation job).
- Q: "Move this table from Postgres to DynamoDB with zero downtime."
- Q: "Split the user service out of the monolith while shipping features weekly."

#### SD15 · Communicating like a senior
`A` · Requires: SD01, SD07, SD10, SD13, SD14, O14, O17, O18, M34, F27 · Unlocks: SD17
- Key: the loop — clarify → estimate → propose the simple design → walk the main flow →
  identify the bottleneck → address it → **state the trade-off you just made and what you'd measure
  to know if it was right**. Say "I'd start simpler and here's the signal that would make me add
  X". Give numbers. Admit unknowns and say how you'd find out. Drive the whiteboard, check in
  ("does that level of detail work, or should I go deeper on the storage?"), and manage the clock.
  Name the risks unprompted. Talk about cost and about who operates it.
- Anti-patterns interviewers explicitly downgrade: jumping to microservices/Kafka unprompted,
  buzzword stacking, ignoring the stated requirements, no numbers, defensive responses to pushback,
  designing for 100x scale that was never asked for, and silence.
- Q: "What would you do differently if you had 10x the traffic? 1/10th the team?"

#### SD16 · Practice problem set (do these out loud, timed at 45 min)
`A` · Requires: SD12 · Unlocks: —
1. URL shortener with analytics · 2. Rate limiter as a service · 3. News feed · 4. WhatsApp/chat ·
5. Uber matching · 6. Ticketmaster booking · 7. Dropbox/file sync · 8. Payment/wallet ledger ·
9. Notification service · 10. Metrics & alerting pipeline · 11. Job scheduler · 12. E-commerce
checkout with inventory · 13. Multi-tenant SaaS API platform · 14. Live leaderboard ·
15. Document collaboration (and where you'd stop and say "this needs CRDTs/OT").
- For each, write: requirements, estimates, diagram, data model, bottleneck, failure story, and the
  one trade-off you'd flag to a staff engineer.

---

## Part B — Senior behavioural signals (do not skip this)

#### SD17 · The stories you must have ready
`A` · Requires: SD15 · Unlocks: —
- Format: **STAR** (Situation, Task, Action, Result) with numbers in the result, and "what I'd do
  differently" on at least half of them. 90 seconds each, not 6 minutes.
- Prepare one story for each: (1) a system you designed end to end and its trade-offs; (2) a
  production incident you led — detection, mitigation, root cause, prevention; (3) a technical
  disagreement you lost gracefully, and one you won with evidence; (4) something you shipped that
  was wrong and what it cost; (5) mentoring or levelling up a teammate; (6) pushing back on scope or
  a deadline with data; (7) a migration/refactor you sequenced safely; (8) working across teams to
  land something you didn't control; (9) a time you chose the boring/simple solution; (10) how you
  handle an ambiguous, under-specified task.
- Senior-vs-mid signals interviewers listen for: you talk about *why* and about trade-offs rather
  than tools; you mention users, cost and operations; you say "we" for the team and "I" for your own
  actions; you quantify; you describe influencing without authority; you show judgement about what
  *not* to build.
- Questions to ask them (this is scored): "What does the on-call rotation look like?" · "How do
  decisions like a framework choice get made here?" · "What's the biggest piece of technical debt
  a new senior would meet?" · "How do you measure a senior engineer's impact in the first 6 months?"
  · "What broke most recently and what changed afterwards?"

---

## Topological order (study waves)

```
Wave 0  SD01
Wave 1  SD02
Wave 2  SD03
Wave 3  SD04
Wave 4  SD05  SD06  SD07  SD08  SD11
Wave 5  SD09  SD10  SD14
Wave 6  SD12  SD13
Wave 7  SD15
Wave 8  SD16  SD17
```

Every node here also requires nodes from Fields 1-8; the cross-field master order is in the README.
