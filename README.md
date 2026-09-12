# Senior Backend Interview — Knowledge Graph & Study Plan

A dependency graph of ~230 knowledge nodes across 9 fields, each with prerequisites, dependents, the
points you must be able to state, and the questions interviewers actually ask.

## The files

| # | Field | Nodes | Prefix |
|---|-------|-------|--------|
| 1 | [Microservices & Distributed Systems](01-microservices/README.md) | 35 | `M` |
| 2 | [Databases](02-databases/README.md) | 40 | `DB` |
| 3 | [Frameworks — NestJS · Django · Spring Boot · others](03-frameworks/README.md) | 28 | `F` |
| 4 | [API Design, HTTP & Protocols](04-api-and-protocols/README.md) | 22 | `A` |
| 5 | [Concurrency, Runtimes & Performance](05-concurrency-runtime-performance/README.md) | 19 | `C` |
| 6 | [Caching, Queues & Event Streaming](06-caching-queues-streaming/README.md) | 21 | `Q` |
| 7 | [Security](07-security/README.md) | 17 | `S` |
| 8 | [Infrastructure, Observability & Delivery](08-infra-observability-delivery/README.md) | 20 | `O` |
| 9 | [System Design & Senior Signals](09-system-design-and-senior-signals/README.md) | 17 | `SD` |

## Node format

```
#### M09 · Timeouts, retries, backoff, idempotency
`I` · Requires: M08 · Unlocks: M10, M16, SD10, Q16
- Key: <the things you must be able to say>
- Q: <a question you will be asked>
```

`Requires` = parents (learn first). `Unlocks` = children (they need this node). Each file ends with
its **topological waves**: everything in Wave N depends only on Waves < N, so a wave can be learned
in any order or in parallel. Levels: `B` beginner · `I` intermediate · `A` advanced · `X` expert.

---

## Master order across all fields

Phases are strictly ordered. Inside a phase, follow each field file's own wave order.

**Phase 0 — Roots (no prerequisites)**
`A01` HTTP · `A12` TCP · `C01` process/thread/coroutine · `C18` data structures · `DB01` relational
model · `M01` why services · `O01` Linux · `Q01` why cache · `S01` threat modelling · `SD01` scoping

**Phase 1 — Primitives**
`A02` idempotency of methods · `A04` HTTP caching · `A10` HTTP/2-3 · `A13` DNS · `A17` serialisation
· `A19` CORS · `C02` blocking vs non-blocking I/O · `DB02` SQL · `DB05` constraints · `DB26` NoSQL
taxonomy · `F01` request lifecycle · `M02` decomposition · `M03` sync vs async · `Q02` cache
strategies · `S02` authentication · `S06` OWASP

**Phase 2 — Building blocks**
`A11` TLS · `A16` realtime · `A20` payloads · `C03` **Node event loop** · `C06` JVM threads ·
`C07` Python GIL/asyncio · `DB03` advanced SQL · `DB04` types & time · `DB06` **transactions** ·
`DB27` Mongo · `DB30` Redis · `F02` routing · `F03` **middleware pipeline** · `F04` **DI** ·
`F05` config · `M04` protocols · `M05` discovery · `M06` gateway · `M08` failure modes ·
`M17` event-driven · `M22` data ownership · `O02` network debugging · `Q05` Redis fundamentals ·
`Q10` messaging fundamentals · `S03` sessions vs JWT · `S07` injection · `S08` XSS/CSRF/SSRF

**Phase 3 — Core engineering**
`A03` REST modelling · `A14` gRPC · `C04` CPU-bound in Node · `C05` promise semantics ·
`C12` GC & memory · `C19` streaming · `DB07` **isolation levels** · `F06` validation ·
`F09` persistence · `F11` authz integration · `F12` testing · `F15` framework caching ·
`M07` load balancing · `M09` **timeouts & retries** · `M11` CAP · `M21` clocks · `Q11` RabbitMQ ·
`Q12` **Kafka fundamentals** · `S04` OAuth2/OIDC · `S10` secrets · `S14` supply chain

**Phase 4 — Depth**
`A07` error contracts · `A15` GraphQL · `A18` API auth · `C08` memory model · `C13` profiling ·
`DB08` **MVCC** · `DB09` **locking** · `DB10` optimistic concurrency · `DB11` storage ·
`F07` error handling · `F10` **transactions in the framework** · `F16` websockets · `F17` uploads ·
`F18` architecture · `F25` OpenAPI · `M10` **circuit breaker/bulkhead** · `M12` consistency models ·
`M24` contracts & versioning · `M30` backpressure · `M32` consistent hashing · `Q03` invalidation ·
`Q06` Redis persistence · `Q08` Redis Lua/streams · `Q13` Kafka durability · `Q14` **partitions &
ordering** · `Q18` CDC · `Q19` schema evolution · `S05` authz models · `S11` crypto · `S17` SDLC

**Phase 5 — Senior core**
`A05` pagination · `A06` versioning · `A08` **rate limiting** · `A09` **idempotency keys** ·
`A21` async APIs · `C09` locks · `C14` **percentiles & Little's Law** · `DB12` WAL ·
`DB13` **B+trees** · `F08` logging · `F13` background jobs · `F19` **NestJS deep** · `F21` Django ·
`F22` Spring Boot · `F24` framework performance · `M13` 2PC · `M20` consensus · `M26` service mesh ·
`M33` **replication & failover** · `O13` observability · `Q04` **stampede & hot keys** ·
`Q07` Redis cluster · `Q15` **consumer semantics & lag** · `S09` **IDOR/mass assignment** ·
`S13` abuse protection

**Phase 6 — Advanced**
`A22` contract testing · `C10` lock-free · `C15` load testing · `C16` backpressure · `C17` network
efficiency · `DB14` index types · `DB15` **composite indexes** · `DB17` partial indexes ·
`DB18` planner · `DB22` **zero-downtime migrations** · `DB23` replication · `DB38` money & ledgers ·
`F14` scheduling · `F20` Nest internals · `F23` other stacks · `F28` **runtime & graceful
shutdown** · `M14` **saga** · `M15` **outbox** · `M27` deployment · `M35` testing ·
`O03` containers · `O05` Kubernetes · `O08` CI/CD · `O14` metrics · `O15` tracing ·
`Q16` **retry & DLQ** · `Q17` stream processing · `S12` privacy/GDPR · `S16` detection & IR

**Phase 7 — Expert**
`C11` **distributed locks & fencing** · `DB16` covering indexes · `DB19` **EXPLAIN** ·
`DB24` partitioning · `DB28` DynamoDB/Cassandra · `DB29` LSM vs B-tree · `DB31` Elasticsearch ·
`DB32` OLAP · `DB33` probabilistic structures · `DB34` Postgres FTS · `DB36` backups/PITR ·
`DB37` DB security · `F26` framework security · `M16` **idempotent consumers** · `M18` event
sourcing · `M28` strangler fig · `M31` blast radius · `O04` cgroups/OOMKill · `O06` **K8s probes &
zero-downtime** · `O07` stateful workloads · `O09` release · `O10` IaC · `O11` cloud ·
`Q20` choosing async tools · `Q21` job design · `S15` infra identity

**Phase 8 — Integration**
`DB20` **query tuning** · `DB21` **connection pooling** · `DB25` sharding · `DB35` distributed SQL ·
`DB39` DB observability · `DB40` **ORM internals** · `F27` choosing a stack · `M19` CQRS ·
`M23` cross-service queries · `M29` multi-tenancy · `M34` SLOs · `O12` serverless · `O16` alerting ·
`O17` **debugging production** · `O18` cost · `O19` config · `O20` DR

**Phase 9 — Capstone**
All of `SD01`–`SD17`: design rounds and behavioural stories.

---

## The 30 nodes that decide the interview

If you run out of time, these are non-negotiable. Be able to explain each at a whiteboard in two
minutes, unprompted, with a real example from your own work:

`C03` Node event loop · `C14` percentiles + Little's Law · `C11` distributed locks & fencing ·
`DB07` isolation levels & write skew · `DB08` MVCC & bloat · `DB09` locking & `SKIP LOCKED` ·
`DB13` B+tree · `DB15` composite index design · `DB19` reading EXPLAIN · `DB20` N+1 & keyset
pagination · `DB21` pooling · `DB22` zero-downtime migration · `DB40` ORM internals ·
`M02` bounded contexts · `M09` timeouts/retries/backoff · `M10` circuit breaker · `M12` consistency
models · `M14` saga · `M15` outbox · `M16` idempotent consumers · `M24` versioning ·
`M33` replication & failover · `A08` rate limiting · `A09` idempotency keys · `A18` JWT vs sessions ·
`Q04` cache stampede & "Redis is down" · `Q14` Kafka keys & ordering · `Q15` consumer lag ·
`F10` transaction boundaries · `O17` debugging a live latency spike.

---

## Study plans

### 4 weeks (≈2h/day)
- **Week 1 — foundations & databases.** Phases 0-2, then the whole database file to `DB23`. Daily:
  run `EXPLAIN ANALYZE` on real queries from your Nest app and narrate the plan out loud.
- **Week 2 — distributed systems & async.** Fields 1 and 6 (Phases 3-6 of those prefixes). Build a
  small outbox + Kafka/BullMQ consumer with idempotency and a DLQ. This one project answers
  `M15`, `M16`, `Q13`-`Q16`, `Q21`.
- **Week 3 — frameworks & cross-training.** `F19`-`F23`, plus the three-framework exercise at the
  bottom of the frameworks file. Also `C03`-`C14` and `O03`-`O17`.
- **Week 4 — design & delivery.** Field 9 top to bottom: two timed design problems from `SD16` every
  day out loud with a recording, then write the 10 `SD17` stories in STAR form.

### 10 days (crunch)
Day 1-2 the 30 nodes above, skimming only. Day 3 databases deep (`DB07`-`DB22`). Day 4 distributed
(`M09`-`M16`). Day 5 Kafka + Redis (`Q04`-`Q16`). Day 6 Nest internals + one comparison framework
(`F10`, `F19`, and pick **one** of Django/Spring, not both). Day 7 performance & production
(`C03`, `C14`, `O06`, `O17`). Day 8-9 five timed system design problems. Day 10 behavioural stories
and your questions for them.

---

## How to self-assess

Mark each node red / amber / green:
- **Green** — you can explain it unprompted for 2 minutes, give a real example, and name a trade-off.
- **Amber** — you recognise it and could discuss it with hints.
- **Red** — you'd be exposed.

Rule: never study a red node whose parents are red. Walk *up* the `Requires` chain first — that is
the entire point of the graph. And convert amber→green by **saying it out loud**, not by re-reading.

---

## Three mock rounds to rehearse

**Round A — deep technical (60 min).** "Explain what happens when you type a request into our API"
→ `A01`/`A12`/`F01` · "Your endpoint got slow this week, walk me through it" → `O17`/`DB20`/`C14` ·
"Explain isolation levels; what's write skew" → `DB07` · "Design idempotency for a payment endpoint"
→ `A09`/`M16` · "Order these Node outputs" → `C03`.

**Round B — architecture (60 min).** "Design ticket booking" → `SD10` · follow-ups on the seat hold,
the payment timeout, the 500k-user thundering herd, the multi-region question, and "what would you
build first if you had two weeks".

**Round C — ownership & behaviour (45 min).** Incident story · a technical disagreement · a
migration you sequenced · the biggest mistake · what you'd change about your current architecture ·
your questions for them. See `SD17`.

---

## One honest note on framework breadth

Depth in NestJS plus fluency in the *concepts* (DI, middleware chains, ORM/transactions, request
lifecycle) beats shallow familiarity with four frameworks. When asked about Django or Spring, the
strongest answer maps the concept you own onto their vocabulary and names the one thing that differs
— e.g. "Spring's `@Transactional` is proxy-based, so self-invocation silently skips the transaction;
Nest has no equivalent decorator so I wire it through AsyncLocalStorage." That single sentence reads
as senior. Do the three-framework exercise in [the frameworks module](03-frameworks/README.md) and you will have
a dozen of them.
