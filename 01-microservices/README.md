[← back to the index](../README.md)

# Field 1 — Microservices & Distributed Systems

**How to read this file:** every node has an ID, a level, `Requires` (parents — learn first), and
`Unlocks` (children — they need this node). The *Topological Order* section at the bottom is the
study sequence: everything in Wave N depends only on Waves < N, so a whole wave can be learned in
parallel.

Legend: `B` beginner · `I` intermediate · `A` advanced · `X` expert/staff-level.

---

## Nodes

#### M01 · Why services at all: monolith → modular monolith → microservices
`B` · Requires: — · Unlocks: M02, M03, M22, M28
- Key: microservices buy **independent deployability + team autonomy**, and cost you network
  failure, data consistency and operational overhead. They are an *organisational* solution
  (Conway's law) before a technical one. "Modular monolith first" is the correct default answer.
- Distributed monolith = worst case: services that must be deployed together.
- Q: "When would you *not* use microservices?" (small team, unclear domain boundaries, no CI/CD or
  observability maturity, low traffic).
- Q: "Your team of 6 has one Nest monolith deploying 3x a day with no pain. Why split it?"
- Q: "What is a distributed monolith and how would you detect you have one?"

#### M02 · Service decomposition: DDD, bounded contexts, aggregates
`I` · Requires: M01 · Unlocks: M17, M22, M24, M29
- Key: decompose by **bounded context / business capability**, not by technical layer or entity.
  Aggregate = transactional consistency boundary; one aggregate = one local transaction.
  Ubiquitous language; context mapping (shared kernel, anti-corruption layer, conformist).
- Smells: chatty services, shared tables, a "god" entity service, every feature touching 5 services.
- Q: "Split an e-commerce monolith. Where do the seams go and why?"
- Q: "Order and Inventory both need `Product`. How do you avoid a shared Product service everyone
  blocks on?" (data duplication + events + anti-corruption layer).
- Q: "How do you size a service? What is too small?"

#### M03 · Communication styles: synchronous vs asynchronous
`B` · Requires: M01 · Unlocks: M04, M06, M08, M17, Q10
- Key: sync request/response couples **availability** (your uptime ≈ product of your dependencies')
  and latency; async messaging decouples time and availability but buys eventual consistency and
  harder debugging. Distinguish commands, events and queries.
- Q: "Checkout must reserve stock, charge a card and send an email. Which calls are sync, which
  async, and why?"
- Q: "If A→B→C→D are all sync at 99.9% each, what is the effective availability?" (~99.6%).

#### M04 · Service-to-service protocols: REST vs gRPC vs messaging
`I` · Requires: M03, A01 · Unlocks: M24, M26
- Key: gRPC/protobuf for internal high-volume, strict contracts, streaming and deadlines; REST/JSON
  for the edge and partners; messaging for fire-and-forget and fan-out.
- Q: "Why is gRPC often better *inside* the mesh but rarely at the public edge?"
- Q: "How do deadlines propagate in gRPC and why does that beat per-hop timeouts?"

#### M05 · Service discovery & registry
`I` · Requires: M03 · Unlocks: M07, M26
- Key: client-side (Consul/Eureka + client LB) vs server-side (K8s Service, ALB); DNS TTL and stale
  entries; active vs passive health checks; register on start / deregister on shutdown; K8s
  Endpoints/EndpointSlice as the modern default.
- Q: "A pod is terminating but still receives traffic. Walk through why and how to fix it."
  (readiness flip + endpoint propagation lag + `preStop` sleep + graceful shutdown).

#### M06 · API Gateway & Backend-for-Frontend
`I` · Requires: M03 · Unlocks: M26, A08, S13
- Key: gateway does auth, TLS termination, routing, rate limiting, aggregation, protocol
  translation. BFF per client type avoids one over-generalised API. Risk: the gateway becomes a
  business-logic monolith and a deployment bottleneck.
- Q: "What belongs in the gateway and what must never go there?"
- Q: "You aggregate 6 downstream calls in the gateway — what is your failure and latency story?"

#### M07 · Load balancing
`I` · Requires: M05 · Unlocks: M10, M31, C14
- Key: L4 (TCP, fast, no HTTP semantics) vs L7 (routing, retries, header-based canary); algorithms:
  round-robin, least-connections, **EWMA / least-loaded**, power-of-two-choices, consistent hashing;
  sticky sessions and why they undermine scaling; HTTP/2 connection pinning skews L4 balancing.
- Q: "Round robin gives you an unbalanced fleet. Why, and what do you switch to?"
- Q: "Why does gRPC behind an L4 load balancer distribute traffic badly?"

#### M08 · Failure modes & the fallacies of distributed computing
`I` · Requires: M03 · Unlocks: M09, M11, M20, M21, M30
- Key: the network is not reliable / zero-latency / infinite / secure / homogeneous; topology
  changes; **partial failure** is the defining problem. Failure taxonomy: crash-stop,
  crash-recovery, omission, timing, Byzantine. Grey failure ("sick but not dead"). Cascading failure.
- Q: "You got a timeout from a payment service. Did the payment happen?" (unknowable — hence
  idempotency keys + reconciliation).
- Q: "What is a grey failure and why do health checks miss it?"

#### M09 · Timeouts, retries, backoff, idempotency
`I` · Requires: M08 · Unlocks: M10, M16, SD10, Q16
- Key: every remote call needs a timeout smaller than the caller's deadline; **deadline
  propagation** beats per-hop timeouts; retry only idempotent/safe operations; exponential backoff
  with **full jitter**; retry budgets (3 retries x 3 hops = 27x load); retry storms; idempotency
  keys for non-idempotent writes; hedged requests for tail latency.
- Q: "Where does exponential backoff alone still take the system down?" (synchronised retries —
  hence jitter; retry amplification across layers — hence budgets and retrying at one layer only).
- Q: "Design idempotency for `POST /payments`."

#### M10 · Resilience patterns: circuit breaker, bulkhead, shedding, fallback
`A` · Requires: M07, M09 · Unlocks: M26, M30, M31, SD07
- Key: circuit breaker states (closed/open/half-open) and why breakers misfire on low-traffic
  dependencies; bulkhead = isolated pool per dependency so one slow dependency cannot exhaust your
  threads/event loop; load shedding, admission control, priority queues; graceful degradation and
  static fallbacks; brownout.
- Q: "The recommendation service is slow and now the whole API times out. Diagnose and fix."
- Q: "Circuit breaker vs retry vs rate limit — when is each the wrong tool?"

#### M11 · CAP, PACELC, and what they actually say
`I` · Requires: M08 · Unlocks: M12, M20, M33
- Key: CAP applies **only during a partition**: choose availability or linearizability. PACELC adds
  the normal-case trade: Else Latency vs Consistency. Real systems choose per operation, not per
  system. CAP is not "pick 2 of 3".
- Q: "Is Postgres CP or AP?" (single node makes it a trick question; with replication + failover it
  becomes a design choice — discuss quorum and split-brain).
- Q: "Name a feature in your product where you would choose availability over consistency, and one
  where you would not."

#### M12 · Consistency models
`A` · Requires: M11 · Unlocks: M14, M19, M23, SD08, DB23
- Key: linearizable > sequential > causal > read-your-writes / monotonic reads > eventual. Session
  guarantees matter: without them "eventual consistency" breaks UX ("I saved it and it's gone").
  Convergence, CRDTs, last-write-wins and its lost-update hazard.
- Q: "A user updates their profile, reloads, and sees the old value. Three fixes?" (read from
  primary after write / sticky to the same replica / version token — i.e. read-your-writes).
- Q: "What is causal consistency and when is it enough?"

#### M13 · Distributed transactions: 2PC/3PC and why they are avoided
`A` · Requires: M12, DB06 · Unlocks: M14, M20
- Key: 2PC = prepare + commit with a coordinator; blocks if the coordinator dies while participants
  hold locks in the prepared state; does not survive partitions; XA support is patchy; couples the
  availability of every participant. 3PC is non-blocking only under assumptions partitions violate.
- Q: "Why doesn't everyone just use 2PC across services?"
- Q: "When is 2PC actually fine?" (one DB with several resource managers; short, low-volume work).

#### M14 · Saga pattern & compensation
`A` · Requires: M12, M13 · Unlocks: M15, M16, SD10
- Key: split a distributed transaction into local transactions plus **compensating actions**.
  Orchestration (central state machine — explicit, debuggable, coupling in the orchestrator) vs
  choreography (events — loose, but emergent flows are hard to reason about). Sagas give ACD, not I:
  you need semantic locks / commutative updates / pessimistic ordering to avoid lost updates and
  dirty reads. Compensations are not rollbacks — you cannot un-send an email, you send an apology.
- Q: "Design order placement across Order/Payment/Inventory as a saga. What if the compensation
  itself fails?" (retry forever + DLQ + human-in-the-loop; make compensations idempotent).
- Q: "Orchestration or choreography for a 7-step branching flow? Why?"

#### M15 · Transactional outbox, inbox, and CDC
`A` · Requires: M14, DB06, Q10 · Unlocks: M16, M18, Q18
- Key: you cannot atomically write the DB and publish to Kafka (dual-write problem). Outbox = write
  the event row in the **same local transaction**, relay via poller or CDC (Debezium reading the
  WAL/binlog); inbox table on the consumer side for dedup. Ordering per aggregate; at-least-once.
- Q: "Why not just publish after commit?" (crash between commit and publish loses the event;
  publishing before commit creates a phantom event).
- Q: "Outbox poller vs CDC: trade-offs?" (latency, DB load, ops complexity, schema coupling).

#### M16 · Idempotent consumers, dedup, and the exactly-once myth
`A` · Requires: M09, M14, M15 · Unlocks: SD10, Q15, Q16
- Key: exactly-once *delivery* is impossible; exactly-once *effect* is achievable via idempotency —
  natural keys, dedup store with TTL, conditional writes, version/sequence numbers, upserts. Side
  effects that leave your system (email, payment) need an idempotency key at the provider.
- Q: "Kafka advertises exactly-once semantics. What does that actually mean?" (transactional
  read-process-write **within Kafka**, not for external side effects).
- Q: "How large does your dedup table get and how do you expire it safely?"

#### M17 · Event-driven architecture
`I` · Requires: M02, M03 · Unlocks: M18, M19, M24, Q10, Q19
- Key: event notification (thin: "OrderPlaced id=1") vs event-carried state transfer (fat: removes
  the callback but duplicates and ages data) vs event sourcing. Events are facts (past tense),
  commands are requests. The producer owns the schema; consumers must tolerate unknown fields.
  Temporal coupling disappears, semantic coupling does not.
- Q: "Thin or fat events — which and why?"
- Q: "How do you stop an event-driven system from becoming impossible to reason about?" (event
  catalogue, tracing, orchestration for business flows, choreography for notifications).

#### M18 · Event sourcing
`X` · Requires: M15, M17 · Unlocks: M19
- Key: the append-only log of state changes is the source of truth; rebuild by replay; snapshots for
  performance; event versioning/upcasting is the real long-term cost; GDPR deletion is genuinely
  hard (crypto-shredding). Not required for CQRS and not implied by using Kafka.
- Q: "You must fix a bug in how state was derived 6 months ago. How does event sourcing help, and
  how does it hurt?"
- Q: "How do you delete a user's PII from an immutable log?"

#### M19 · CQRS and read models
`A` · Requires: M12, M17, M18 · Unlocks: M23, SD05, SD12
- Key: separate write model (normalised, enforces invariants) from read models (denormalised, one
  per query). Projections are eventually consistent, so the UI must handle **projection lag**
  (return the written entity, optimistic UI, version token). Projections must be rebuildable. Do not
  apply CQRS globally.
- Q: "After a write the list endpoint does not show the new row. Fixes?"
- Q: "Where is CQRS overkill?"

#### M20 · Consensus, quorums, leader election
`X` · Requires: M08, M11, M13 · Unlocks: M31, M33, DB35
- Key: Raft (terms, leader election, log replication, commit index) vs Paxos vs ZAB; quorum
  W + R > N; 2f+1 nodes tolerate f failures; leases and their clock assumptions; use
  etcd/ZooKeeper/Consul rather than writing your own; consensus belongs on the metadata and
  coordination path, not the data path.
- Q: "Explain Raft leader election in 90 seconds."
- Q: "Why 3 or 5 nodes and never 4?"
- Q: "Two nodes both believe they are leader. What prevents corruption?" (terms/epochs + fencing).

#### M21 · Time, clocks and ordering
`A` · Requires: M08 · Unlocks: M33, DB35, Q14
- Key: wall clocks drift and jump (NTP steps, leap seconds) — never use them for ordering or
  correctness; monotonic clocks for durations; Lamport clocks (happens-before), vector clocks
  (detect concurrency, grow with participants), hybrid logical clocks; Spanner TrueTime + commit wait.
- Q: "Two services write the same row using `updated_at` from their own clocks. What goes wrong?"
- Q: "How would you order events across services without a global clock?"

#### M22 · Data ownership: database-per-service
`I` · Requires: M01, M02 · Unlocks: M23, M25, SD04, DB25
- Key: a shared database is the classic anti-pattern (schema coupling, no independent deploy, hidden
  writers). Each service owns its schema and exposes it only through API or events. Migration path:
  logical separation → separate schema → separate instance. Analytics goes to a warehouse via CDC,
  never by querying another team's OLTP database.
- Q: "Another team asks for read-only access to your Postgres for a report. Your answer?"

#### M23 · Querying across services
`A` · Requires: M12, M19, M22 · Unlocks: SD12
- Key: API composition (simple, but N+1, partial failure, and in-memory joins that cannot paginate)
  vs CQRS materialised view (fast, eventually consistent, extra infrastructure) vs a data lake for
  analytics. Never join across services in the database.
- Q: "The admin page must filter orders by customer country, sort by order value and paginate.
  Customer and Order are separate services. Design it."

#### M24 · Contracts, versioning, backward compatibility
`A` · Requires: M02, M04, M17 · Unlocks: M27, A22, Q19
- Key: additive and optional-only changes; never repurpose a field or renumber a protobuf tag;
  expand → migrate → contract; support N-1 during rolling deploys (your own service runs two
  versions at once); consumer-driven contract tests (Pact); schema-registry compatibility modes.
- Q: "You must rename a field used by 12 consumers. Sequence the change."
- Q: "During a rolling deploy v1 and v2 consume the same topic. What does that constrain in your
  event schema *and* your DB migration?"

#### M25 · Distributed tracing & correlation
`I` · Requires: M22, O13 · Unlocks: M26, O15
- Key: trace/span/parent; W3C `traceparent` propagated through HTTP **and** message headers and
  async job boundaries; sampling (head vs tail); correlation IDs in logs; reading a waterfall.
- Q: "A request is slow 1% of the time. How do you find where?" (tail-based sampling, exemplars).

#### M26 · Service mesh & sidecars
`A` · Requires: M04, M05, M06, M10, M25 · Unlocks: M27, S15
- Key: moves retries/timeouts/mTLS/traffic-splitting/telemetry out of the app into a sidecar
  (Envoy) or ambient/eBPF layer; control plane vs data plane; costs are a latency hop, memory per
  pod and operational complexity; library vs mesh trade-off; workload identity (SPIFFE).
- Q: "What does a mesh give you that a library (resilience4j, Nest interceptors) does not?"
- Q: "Retries are configured in both the mesh and the app. What is the bug?" (multiplicative retries)

#### M27 · Deployment & release strategies
`I` · Requires: M24, M26 · Unlocks: M28, O09, SD14
- Key: rolling, blue/green, canary with automated analysis, shadow/dark traffic; **decouple deploy
  from release** with feature flags; DB migrations must be compatible with the previous version; the
  rollback plan must include data.
- Q: "How do you roll back a deploy that already ran a destructive migration?" (you cannot — hence
  expand/contract).

#### M28 · Strangler fig: migrating off a monolith
`A` · Requires: M01, M22, M27 · Unlocks: SD14
- Key: put a façade in front, move one capability at a time, dual-write or CDC the data, reconcile,
  cut reads over, then delete. Branch-by-abstraction. Parity/shadow testing. Never a big-bang rewrite.
- Q: "Walk me through extracting notifications from a Nest monolith with zero downtime."

#### M29 · Multi-tenancy
`A` · Requires: M02, DB25 · Unlocks: SD13, S05
- Key: silo (DB per tenant) vs bridge (schema per tenant) vs pool (shared tables + `tenant_id`);
  noisy neighbours; per-tenant quotas; row-level security; tenant-aware cache keys; migration cost at
  10k tenants; data residency.
- Q: "Pick a model for 50 enterprise tenants; now for 500,000 self-serve tenants. Why different?"

#### M30 · Backpressure & flow control
`A` · Requires: M08, M10 · Unlocks: C16, Q15
- Key: bounded queues everywhere (an unbounded queue is a latency bomb and an OOM); reject early
  rather than queue; TCP and HTTP/2 flow control; reactive streams `request(n)`; consumer lag as the
  signal; queueing delay vs service time (Little's Law).
- Q: "Queue depth grows for 20 minutes and then everything falls over. Explain and fix."

#### M31 · Blast radius: cells, shuffle sharding, static stability
`X` · Requires: M07, M10, M20 · Unlocks: SD07, SD13
- Key: cell-based architecture (independent full stacks behind a thin router); shuffle sharding to
  shrink the set of customers a poison workload can hit; static stability (keep serving when the
  control plane is down); regional isolation; avoid global config pushes.
- Q: "One customer sends a payload that crashes workers. How do you limit the damage?"

#### M32 · Partitioning & consistent hashing
`A` · Requires: M07 · Unlocks: M33, DB25, Q14
- Key: modulo hashing breaks on resize; consistent hashing ring + virtual nodes; rendezvous hashing;
  range vs hash partitioning; hot partitions and key salting; rebalancing cost.
- Q: "Derive why consistent hashing moves only ~K/N keys when a node joins or leaves."

#### M33 · Replication, failover, split-brain
`A` · Requires: M11, M20, M21, M32 · Unlocks: DB23, SD07
- Key: single-leader / multi-leader / leaderless (quorum, read repair, hinted handoff, anti-entropy);
  sync vs async replication and the durability/latency trade; failover hazards — lost writes, split
  brain, fencing tokens, STONITH; replication lag and monotonic reads.
- Q: "An async replica is promoted after the primary crashes. What did you lose and how do you
  detect it?"
- Q: "What is a fencing token and which bug does it prevent?"

#### M34 · SLIs, SLOs, error budgets
`I` · Requires: M08, O13 · Unlocks: O16, SD15
- Key: SLI (measured), SLO (target), SLA (contract with penalties); error budget governs release
  pace; multi-window multi-burn-rate alerting; measure a *user journey*, not a host.
- Q: "Set an SLO for a checkout API and derive the alert from it."

#### M35 · Testing distributed systems
`A` · Requires: M24, M26, F12 · Unlocks: SD07
- Key: the pyramid shifts — consumer-driven contract tests replace most cross-service integration
  tests; testcontainers for real DB/broker; fault injection and chaos engineering (steady-state
  hypothesis, blast radius, game days); deterministic simulation; testing in production (shadow,
  canary analysis).
- Q: "How do you test that your saga compensates correctly when service B dies mid-flow?"

---

## Topological order (study waves)

```
Wave 0  M01
Wave 1  M02  M03
Wave 2  M04  M05  M06  M08  M17  M22
Wave 3  M07  M09  M11  M21  M24  M25  M32  M34
Wave 4  M10  M12  M29  M30
Wave 5  M13  M20  M23  M26  M33
Wave 6  M14  M27  M31  M35
Wave 7  M15  M28
Wave 8  M16  M18
Wave 9  M19
```

Cross-field parents referenced above: `A01` HTTP, `DB06` transactions, `DB25` sharding, `Q10` queue
basics, `O13` observability, `F12` testing — see the other field files.

**If you only have 3 days for this field:** M01 → M02 → M03 → M08 → M09 → M10 → M11 → M12 → M14 →
M15 → M16 → M24 → M33. Those thirteen carry most of the senior microservices interview.
