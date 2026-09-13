[← back to the field index](README.md)

# Microservices · Part 5 — Scale, Isolation & Reliability

Nodes `M29`–`M35`.

---

## M29 · Multi-tenancy

`Advanced` · Requires: `M02`, `DB25` · Unlocks: `SD13`, `S05`

### Preface

Multi-tenancy means several customers share one system. The core question is how much they share:
separate databases, separate schemas, or the same tables with a `tenant_id` column.

The answer depends almost entirely on how many tenants you have and how large they are. Fifty
enterprise customers and five hundred thousand self-serve accounts need different designs, and
saying why is the interview answer.

### Details

#### 1. The three models

**Theory.**
- **Silo** — one database (or one whole stack) per tenant. Strongest isolation, easiest compliance
  and per-tenant restore, most expensive per tenant, and migrations must run N times.
- **Bridge** — one database, one schema per tenant. Decent isolation, shared connections, still N
  migrations, and connection pooling gets awkward with thousands of schemas.
- **Pool** — shared tables with a `tenant_id` column on everything. Cheapest, one migration,
  best resource use, and isolation depends entirely on your code being correct.

**Example.** Fifty enterprise tenants with contractual data-isolation requirements: silo or bridge,
because the cost per tenant is small relative to the contract value and you can restore one
customer's data alone. Five hundred thousand free-tier users: pool, because half a million
databases is not operable and most are nearly empty.

**Advanced.** Hybrids are common and worth mentioning: pool by default, and silo the few tenants who
pay for it or whose size hurts everyone else. That needs the application to route by tenant from the
start — a per-tenant connection resolver — even if there is only one database on day one. Retrofitting
that routing later is expensive, so it is a cheap thing to build early.

#### 2. Isolation in the pooled model

**Theory.** In a pooled model, **one missing `WHERE tenant_id = ?` is a data breach.** Isolation
cannot rest on developers remembering it; it must be structural.

**Example.** Layered defences: hold the tenant in a request-scoped context (`AsyncLocalStorage` in
Node, a context variable in Python, a `ThreadLocal` or request scope in Spring); make the data
access layer add the tenant filter automatically rather than each query doing it; add Postgres
**row-level security** as a backstop so even a query that forgets returns nothing; and write tests
that attempt cross-tenant access and assert failure.

**Advanced.** Row-level security is the strongest of these because it is enforced by the database
regardless of application bugs, but it needs care: the session must set the tenant
(`SET LOCAL app.tenant_id`), which interacts badly with transaction-mode connection poolers that do
not preserve session state (see `DB21`); policies add planning overhead; and the application's
database user must not be able to bypass RLS. Mention these caveats — they show operational
experience.

#### 3. Noisy neighbours

**Theory.** In a shared system, one tenant's behaviour degrades everyone else's experience: a huge
import, a pathological query, a burst of API calls, a hot cache key.

**Example.** A tenant uploads a 2 million row CSV. Without controls it saturates the worker pool and
every other tenant's background jobs stall for an hour. Controls: per-tenant rate limits and
quotas; per-tenant job queues or weighted fair consumption so one tenant cannot occupy every worker;
limits on request and payload size; and per-tenant metrics so you can identify who is causing a
problem in minutes rather than hours.

**Advanced.** The strong structural answer is **shuffle sharding** (see `M31`): assign each tenant a
small random subset of workers rather than the whole pool. One abusive tenant then affects only the
handful of workers it shares, and the probability of two tenants overlapping entirely is tiny. This
is how large platforms limit blast radius without giving every tenant dedicated capacity.

#### 4. Tenant-aware everything

**Theory.** Once tenancy exists, it must appear in every layer: cache keys, search indexes, file
storage paths, logs, metrics, background jobs, and events.

**Example.** A cache key of `user:123` is a bug waiting to happen if user ids are only unique within
a tenant; it must be `tenant:abc:user:123`. The same applies to Elasticsearch (index or filter per
tenant), S3 prefixes, and log fields. Every metric should have a tenant label — but be careful,
because high-cardinality labels break Prometheus (see `O14`); use tracing or logs for per-tenant
detail and keep metrics aggregated with only the largest tenants labelled.

**Advanced.** Migrations in a silo or bridge model become a distributed problem: applying a change to
5,000 schemas takes hours, can partially fail, and needs to be resumable with per-tenant status
tracking. Teams typically build a migration runner with concurrency, retries and a dashboard. This
operational cost is the strongest practical argument for the pooled model.

### Interview questions

- "Pick a model for 50 enterprise tenants; now for 500,000 self-serve tenants. Why different?"
- "How do you make cross-tenant data leakage structurally impossible?"
- "One tenant is slowing down everyone. What do you do, short and long term?"
- "What breaks when you have 5,000 schemas?"

---

## M30 · Backpressure and flow control

`Advanced` · Requires: `M08`, `M10` · Unlocks: `C16`, `Q15`

### Preface

Backpressure is a fast producer being told to slow down because the consumer cannot keep up.

Without it, work piles up in a queue. Queues look like they are helping — nothing is failing — but
latency grows until every request times out anyway, and memory grows until the process dies. The
key insight: **an unbounded queue converts a throughput problem into a total outage.**

### Details

#### 1. Why unbounded queues are dangerous

**Theory.** A queue absorbs a *temporary* mismatch between arrival rate and service rate. If the
mismatch is sustained, the queue grows without limit. Every item waits longer, so by the time the
consumer reaches an item, its requester has usually given up — the work is wasted, but it still
consumes capacity, which makes the backlog worse.

**Example.** An API accepting requests into an in-memory queue processed by workers. At 1,000
requests per second arriving and 800 processed, 200 accumulate every second. After five minutes the
queue holds 60,000 items and the newest item waits 75 seconds — long after the 30-second client
timeout. The system is doing full work and serving nobody, and memory is climbing toward a crash.

**Advanced.** Little's Law makes this precise: queue length = arrival rate x wait time. For a fixed
service rate, wait time grows linearly with queue length, so a bounded queue is also a **latency
bound**. Choosing a queue size is really choosing a maximum acceptable wait: if you serve 800/s and
accept at most 1 second of queueing, the queue should hold about 800 items. That is a much better way
to pick the number than guessing (see `C14`).

#### 2. Push versus pull

**Theory.** In a **push** system the producer sends whenever it wants and the consumer must cope. In
a **pull** system the consumer asks for work when it has capacity, so backpressure is automatic.

**Example.** Kafka consumers pull: they call `poll()` and receive at most `max.poll.records`. A slow
consumer simply polls less often and lag grows — visible, measurable, and harmless to the broker.
RabbitMQ pushes by default, which is why **prefetch** exists: it limits how many unacknowledged
messages a consumer may hold. Without a prefetch limit, the broker floods a consumer with thousands
of messages it cannot process, and they sit in its memory rather than in the broker where they are
safe and countable.

**Advanced.** Prefer pull, or push with a strict limit, at every layer. In HTTP, the equivalents are
HTTP/2 flow control (per-stream windows) and TCP's receive window. In Node streams, `write()`
returning `false` is a backpressure signal you must honour — `pipeline()` does it for you; a manual
loop that ignores the return value buffers the whole source in memory (see `C16`).

#### 3. Signals: what to measure

**Theory.** You need a number that says "we are falling behind", available before users notice.
Good signals: consumer lag (messages not yet processed), queue depth, and most usefully **age of the
oldest item**, concurrent in-flight requests, and event loop delay or thread pool saturation.

**Example.** Alert on **age of the oldest unprocessed message**, not on queue depth. Depth of 10,000
is fine if you drain it in 20 seconds; depth of 50 is an emergency if the oldest has been waiting 20
minutes because the consumer is stuck. Age directly expresses user impact; depth does not.

**Advanced.** Distinguish a **growing** backlog from a **large** one. A queue that is large but
draining at a steady rate is healthy. Alert on the trend — is arrival rate above service rate for
several minutes — rather than on an absolute threshold, which either pages during normal bursts or
fires too late.

#### 4. Reject rather than buffer

**Theory.** When you cannot keep up, the choices are: buffer (queue grows), block (slow the
producer, which may be a user), or reject (fail fast). For user-facing traffic, rejecting quickly is
usually correct — the client can retry with backoff, and the system stays responsive for everyone
else.

**Example.** A bounded queue of 1,000 items; when full, return 503 with `Retry-After: 2`
immediately. Users see an honest error in 5ms rather than a timeout after 30 seconds, and the
system recovers as soon as load drops instead of grinding through a useless backlog.

**Advanced.** Push the rejection **as far upstream as possible**. Rejecting at the gateway costs
almost nothing; rejecting after authentication, database lookups and half the business logic means
you paid for work you threw away. Under overload, the cost of rejecting is what determines whether
you survive — this is why admission control belongs at the edge, and why the check must be cheap
(see `M10`).

### Interview questions

- "Queue depth grows for 20 minutes and then everything falls over. Explain and fix."
- "Why is an unbounded queue worse than rejecting requests?"
- "What is prefetch in RabbitMQ and why does it matter?"
- "Which single metric would you alert on for a backlog?"

---

## M31 · Blast radius: cells, shuffle sharding, static stability

`Expert` · Requires: `M07`, `M10`, `M20` · Unlocks: `SD07`, `SD13`

### Preface

At a certain scale the question stops being "how do we prevent failure" and becomes "when it fails,
how few customers notice".

The techniques are all about partitioning: split the system into independent pieces so a failure is
contained in one piece, and make sure the pieces do not share anything that can fail globally.

### Details

#### 1. Cell-based architecture

**Theory.** A cell is a complete, independent copy of the stack — services, database, cache — serving
a subset of customers. A thin routing layer maps a customer to a cell. Cells share no state, so a
failure in one cannot spread. The router must be simple enough to be nearly infallible.

**Example.** Ten cells, each serving 10% of tenants. A bad deploy, a poisoned cache, or a corrupted
database affects one cell: 10% of customers, not everyone. Deployments go cell by cell, so a problem
is caught at 10% exposure. Cells also cap scale problems — you know a cell handles N tenants, so
growth means more cells rather than an unbounded single system.

**Advanced.** The hard parts are the router (it must not become a shared point of failure — keep its
logic static and its data tiny), any genuinely global data such as unique usernames across cells,
and moving a tenant between cells when one becomes too large. Cell migration is essentially the data
migration problem of `M28` and needs to be built before you need it.

#### 2. Shuffle sharding

**Theory.** Instead of giving each customer the whole pool or one dedicated shard, give each
customer a **random small subset** of workers. Two customers rarely share their entire subset, so
one customer's poison workload only affects the few workers they share with others.

**Example.** Eight workers, each customer assigned two at random. There are 28 possible pairs, so
the chance that another specific customer has exactly your pair is about 1 in 28. If your requests
crash workers, most customers lose at most one of their two workers and continue at reduced capacity
with retries. Compare with a shared pool, where one poison request pattern takes down everyone.

**Advanced.** The combinatorics improve rapidly: 100 workers with 5 per customer gives about 75
million combinations, so complete overlap becomes vanishingly unlikely. This is how AWS protects
shared services. The requirement is that clients retry on a different member of their subset, and
that assignment is stable per customer (hash the customer id) so the isolation actually holds.

#### 3. Static stability

**Theory.** A system is statically stable if it keeps working when its control plane — the thing
that makes changes — is unavailable. The data plane should never need to consult the control plane
to serve existing traffic.

**Example.** A load balancer keeps routing to the instances it already knows about when the service
registry is down; it just cannot learn about new ones. A pod keeps running when the Kubernetes API
server is unavailable. A service caches its configuration and feature flags locally with a long
fallback, so a config service outage does not become a total outage.

**Advanced.** The canonical application is capacity: pre-provision enough capacity to survive a zone
failure **without needing to scale up**, because autoscaling depends on a control plane that may
itself be impaired during the event, and because everyone else is trying to scale at the same moment
in the remaining zones. Being "statically stable" means the failure requires no action to survive.
This is one of the highest-signal concepts to bring up in a reliability discussion.

#### 4. Avoiding global changes

**Theory.** Anything applied everywhere at once can break everything at once: a config push, a
feature flag, a certificate rotation, a DNS change, a schema migration.

**Example.** The largest cloud outages of the past decade have mostly been caused by configuration
changes applied globally, not by hardware. The mitigation is to treat config exactly like code:
version it, review it, deploy it progressively (one cell, one region, one percentage at a time),
validate it before applying, and be able to roll it back quickly.

**Advanced.** Certificate and credential expiry deserve a special mention: they are scheduled global
outages that you know about in advance and still miss. Automate rotation, alert on approaching
expiry with generous lead time, and stagger expiry dates so everything does not expire at once.
Mentioning expiry monitoring in a reliability answer shows you have been on call.

### Interview questions

- "One customer sends a payload that crashes workers. How do you limit the damage?"
- "What is static stability and why does it matter during a zone failure?"
- "Explain shuffle sharding and why the maths works."
- "Why are global configuration pushes dangerous?"

---

## M32 · Partitioning and consistent hashing

`Advanced` · Requires: `M07` · Unlocks: `M33`, `DB25`, `Q14`

### Preface

To scale beyond one machine you split data across several. Partitioning is how you decide which
machine holds which piece.

The naive method, `hash(key) % N`, works until N changes — then almost every key moves. Consistent
hashing exists to make adding and removing nodes cheap.

### Details

#### 1. Why modulo hashing fails

**Theory.** With `hash(key) % N`, changing N changes the result for nearly every key. Going from 4
nodes to 5 relocates roughly 80% of the data.

**Example.** A cache of 4 nodes; you add a fifth. Almost every key now hashes to a different node,
so almost every lookup misses. The cache is effectively empty and the whole miss load lands on the
database at once — a self-inflicted outage from a routine capacity change.

**Advanced.** The same problem applies to sharded databases, where it is far worse: instead of a
cold cache you must physically move 80% of the data while serving traffic. This is why the shard key
and partitioning scheme are among the hardest decisions to change later.

#### 2. Consistent hashing

**Theory.** Map both keys and nodes onto a circle (hash space). A key belongs to the first node
clockwise from it. Adding a node captures only the keys between it and its predecessor; removing a
node hands its keys to the next node. Only about K/N keys move.

**Example.** Four nodes evenly placed on the ring; adding a fifth takes over one arc. Roughly 20% of
keys move, and every other key stays where it was. This is what Redis Cluster, Cassandra, DynamoDB
and most consistent-hash load balancers use.

**Advanced.** Plain consistent hashing distributes unevenly because random node positions leave arcs
of very different sizes. The fix is **virtual nodes**: place each physical node at many points on
the ring (100-500 is typical). Distribution smooths out, and removing a node spreads its keys across
many others rather than dumping them all on one neighbour. If asked to design consistent hashing,
mentioning virtual nodes unprompted is the marker of having actually implemented it.
**Rendezvous hashing** is a simpler alternative with similar properties: for each key compute
`hash(key, node)` for every node and pick the highest — no ring, no virtual nodes, at the cost of
O(N) per lookup.

#### 3. Range versus hash partitioning

**Theory.** **Hash partitioning** spreads keys evenly and destroys ordering, so range queries must
hit every partition. **Range partitioning** keeps order, so range scans hit one partition, but
distribution follows your data and is easily skewed.

**Example.** Time-series data partitioned by time range: "last 24 hours" reads one partition —
excellent. But all *writes* go to the newest partition, so one node takes the entire write load — a
hot spot. Hash-partition by device id instead and writes spread evenly, but "everything in the last
hour" now queries every partition. Real systems often combine them: hash by device, range by time
within that.

**Advanced.** This trade-off shows up everywhere: Kafka partition keys, DynamoDB partition and sort
keys, Cassandra's partition and clustering keys, Postgres declarative partitioning. The general
shape is the same: **the partition key controls distribution, the sort key controls in-partition
range queries.** Being able to state it that generically transfers across every one of those
systems.

#### 4. Hot partitions

**Theory.** Even with good hashing, real workloads are skewed: one customer is a hundred times
larger, one product is viral, one key is read constantly. That key's partition becomes a bottleneck
and cannot be split, because a single key lives in one place.

**Example.** Mitigations, in increasing order of complexity: **salting** — split the key into
`product:123:0` through `product:123:9` and aggregate on read, trading read complexity for write
distribution; **caching in front** of the hot key so most reads never reach the partition;
**dedicated capacity** for known large tenants; and **client-side aggregation** — buffer updates
locally and flush periodically rather than sending every increment.

**Advanced.** Detecting hot partitions requires per-key or per-partition metrics, which most people
only add after the incident. Instrument the top-N keys by request count (a count-min sketch is the
cheap way — see `DB33`) so you can identify a hot key during an outage rather than guessing. See
also `SD09`, where this is the celebrity problem.

### Interview questions

- "Derive why consistent hashing moves only about K/N keys when a node joins or leaves."
- "Why do you need virtual nodes?"
- "Range or hash partitioning for time-series data? What breaks either way?"
- "One key gets 40% of your traffic. Fix it."

---

## M33 · Replication, failover and split-brain

`Advanced` · Requires: `M11`, `M20`, `M21`, `M32` · Unlocks: `DB23`, `SD07`

### Preface

Replication keeps copies of your data on several machines, for durability (a machine dies, the data
survives), for availability (failover to a copy), and for read scaling.

The core decision is whether a write must reach the copies before you acknowledge it. Waiting costs
latency; not waiting means a failover can lose recently acknowledged writes.

### Details

#### 1. The three topologies

**Theory.**
- **Single leader** — all writes go to one node, which replicates to followers. Simple, no write
  conflicts, and the leader is a bottleneck and a failover problem. This is Postgres, MySQL, MongoDB
  and Kafka partitions.
- **Multi leader** — several nodes accept writes and replicate to each other. Good for multi-region
  write latency and offline clients, and you must resolve conflicts.
- **Leaderless** — any replica accepts writes; consistency comes from quorums and repair. Dynamo,
  Cassandra, Riak.

**Example.** Single leader is the default and the right answer in most interviews. Multi-leader
appears when you need local writes in several regions and can define conflict resolution — and
conflict resolution is the part people underestimate: two regions editing the same record need a
business rule, not a technical one.

**Advanced.** Leaderless systems trade the leader bottleneck for read repair, hinted handoff and
anti-entropy — background processes that reconcile replicas. They also require tuning W and R per
request, which pushes consistency decisions into application code. That is power and a burden;
teams often set quorum reads and writes everywhere and lose the performance benefit they adopted it
for.

#### 2. Synchronous versus asynchronous

**Theory.** **Synchronous**: the leader waits for a replica to confirm before acknowledging. No data
loss on failover; every write pays the replication round trip; if the replica is slow or down,
writes stall. **Asynchronous**: acknowledge immediately, replicate in the background. Fast, and a
failover loses whatever had not replicated.

**Example.** Postgres `synchronous_commit` with `synchronous_standby_names` gives you a middle
ground: `ANY 1 (replica_a, replica_b)` waits for whichever confirms first, so one slow replica does
not stall writes while still guaranteeing the write exists on two machines. Fully synchronous
replication to a single named standby couples your availability to that standby.

**Advanced.** **Semi-synchronous** is the common production compromise: wait for the replica to
*receive* the write (in its memory or its log) but not to *apply* it. That bounds the loss window to
almost nothing while avoiding the cost of waiting for replay. Know that "acknowledged" can mean
received, flushed to disk, or applied — and that these differ substantially in both durability and
latency. Ask which one a system means before trusting its guarantee.

#### 3. Failover, split-brain and fencing

**Theory.** When the leader fails, something promotes a follower. The danger: the old leader may not
be dead, only unreachable. If it keeps accepting writes, you have two leaders — **split-brain** —
and divergent data that cannot be automatically reconciled.

**Example.** The classic sequence: the leader has a long garbage-collection pause; the failover
system sees it as dead and promotes a follower; the old leader wakes up and, believing it is still
leader, continues writing. Two nodes now accept conflicting writes to the same rows.

**Advanced.** The defences: **fencing tokens** — each leadership term gets an increasing number, and
the storage layer rejects writes carrying an old number, so a revived old leader is refused;
**STONITH** ("shoot the other node in the head") — forcibly power off or network-isolate the old
leader before promoting; and **quorum-based promotion** so a minority partition cannot elect a
leader at all. Fencing is the important one to be able to explain, because the same idea makes
distributed locks safe (`C11`) and is what Raft's terms provide (`M20`).

#### 4. Replication lag and its consequences

**Theory.** Followers are behind, by milliseconds normally and by minutes under load or during a
large write. Reads from a follower are reads from the past, which breaks read-your-writes and
monotonic reads (`M12`).

**Example.** Common causes of lag spikes: a large batch update or bulk import; a long-running query
on the replica blocking replay (Postgres replay conflicts); single-threaded replay unable to keep up
with a highly parallel write workload on the primary; and network saturation between regions. Lag is
also invisible to application error handling — it produces wrong answers, not exceptions, which is
why it must be monitored and alerted on explicitly.

**Advanced.** A replica does not help a write-heavy workload at all: it must apply every write the
primary does, so it uses comparable resources and adds no write capacity. Read replicas help
read-heavy workloads only. And beware the failure mode where lag grows until the replica is useless
for reads, you route reads back to the primary, and the primary — already the bottleneck — falls
over. Routing logic should degrade deliberately rather than by accident.

### Interview questions

- "An async replica is promoted after the primary crashes. What did you lose and how do you detect
  it?"
- "What is a fencing token and which bug does it prevent?"
- "When does a read replica *not* help?"
- "Walk through split-brain and three ways to prevent it."

---

## M34 · SLIs, SLOs and error budgets

`Intermediate` · Requires: `M08`, `O13` · Unlocks: `O16`, `SD15`

### Preface

You cannot make a system perfectly reliable, and trying to is enormously expensive. So you decide
how reliable it needs to be, measure whether you are meeting that, and spend the difference on
shipping features.

An **SLI** is a measurement. An **SLO** is the target. An **SLA** is a contract with a customer,
usually with money attached. The error budget is the gap between your SLO and 100% — the failure you
are allowed to have, and can deliberately spend.

### Details

#### 1. Choosing good indicators

**Theory.** An SLI must measure what users experience, not what your servers report. The standard
shapes: availability (proportion of successful requests), latency (proportion of requests faster
than a threshold), and quality or correctness (proportion of results that were complete).

**Example.** Bad SLI: "CPU below 80%" — users do not care. Weak SLI: "the server returned 200" — it
misses the case where it returned 200 with an empty body. Good SLI: "proportion of `POST /checkout`
requests that returned 2xx within 1 second, measured at the load balancer". Measured at the edge,
because that is closer to the user than inside your service.

**Advanced.** Latency SLOs are better expressed as a **proportion under a threshold** than as a
percentile target. "99% of requests under 300ms" is easy to compute, aggregate across instances and
alert on; "p99 under 300ms" cannot be averaged across instances or over time windows (see `O14`).
This is a small technical point that signals real familiarity.

#### 2. Setting the target

**Theory.** The target comes from what users need and what the business will pay for, not from a
round number. Each additional nine costs roughly ten times more. Your SLO must also be lower than
the combined reliability of your dependencies, or it is unachievable by construction.

**Example.** 99.9% is about 43 minutes of downtime a month; 99.99% is about 4 minutes. Four minutes
means no human can respond in time, so everything must fail over automatically — a different and
much more expensive architecture. If your cloud provider's managed database offers 99.95%, you
cannot promise 99.99% on top of it without multi-region redundancy.

**Advanced.** Two refinements. Set SLOs on **user journeys**, not endpoints: "a customer can
complete checkout" spans several services, and that is what matters. And consider whether an SLO
should be *lowered*: if you are consistently at 99.99% with a 99.9% target, you are over-investing
in reliability and could be shipping faster — genuinely the intended use of the framework, and a
point that surprises interviewers pleasantly.

#### 3. Error budgets

**Theory.** If the SLO is 99.9%, the error budget is 0.1% of requests. Spend it on risk: releases,
migrations, experiments. When the budget is exhausted, stop shipping features and fix reliability.
When it is underspent, you can take more risk.

**Example.** A policy that actually works: budget consumed above 50% mid-month → no risky deploys on
Fridays; budget exhausted → feature work pauses and the team works on reliability until the window
rolls. The value is that it turns "should we ship this?" from an argument into a number both
engineering and product agreed on in advance.

**Advanced.** Error budgets only work if the organisation genuinely honours them; otherwise they
become a dashboard nobody looks at. Also decide in advance how **planned** maintenance and
dependency outages count, or every incident becomes a debate about whether it was really your fault.
Writing that policy down before you need it is the mark of a team that has done this seriously.

#### 4. Alerting on budget burn

**Theory.** Alerting on a raw error rate produces noise (brief spikes) and misses slow burns (a
0.5% error rate for two days quietly consumes a month's budget). Alert instead on **burn rate** —
how fast you are consuming the budget relative to the window.

**Example.** The standard multi-window, multi-burn-rate setup: page if the budget is being consumed
14x faster than sustainable over the last hour *and* the last 5 minutes (a fast burn that would
exhaust the month in about two days); raise a ticket if it is burning 3x over the last 6 hours (a
slow burn). Requiring both a long and a short window prevents both false alarms from momentary
spikes and slow leaks going unnoticed.

**Advanced.** This directly implements "alert on symptoms, not causes" (see `O16`): you are alerted
when users are being affected at a rate that matters, regardless of which component is at fault.
Cause-based alerts (high CPU, disk filling) belong as tickets or dashboards, not pages.

### Interview questions

- "Set an SLO for a checkout API and derive the alert from it."
- "What is an error budget and what do you do when it is exhausted?"
- "Why alert on burn rate instead of error rate?"
- "Your SLO is 99.99%. What does that force about your architecture?"

---

## M35 · Testing distributed systems

`Advanced` · Requires: `M24`, `M26`, `F12` · Unlocks: `SD07`

### Preface

The classic test pyramid assumes one process. Across services, the expensive middle — spinning up
every service to test an interaction — does not scale: it is slow, flaky, and owned by nobody.

The shift is: test each service thoroughly in isolation, verify the *contracts* between them
automatically, and test the real system in production with controlled experiments.

### Details

#### 1. The reshaped pyramid

**Theory.** Per service: many unit tests; a solid layer of integration tests against real
dependencies (a real database and broker via containers, not mocks); contract tests to verify the
boundaries; and a small number of end-to-end tests over critical user journeys only.

**Example.** For an Orders service: unit tests for pricing rules; integration tests running against
a real Postgres and a real Kafka in Testcontainers, covering repository code, migrations and
consumers; contract tests proving it satisfies what Billing expects; and exactly one end-to-end test
that places an order through the real stack.

**Advanced.** The argument for Testcontainers over mocks or in-memory substitutes is that the
interesting bugs live in the real thing: constraint violations, transaction and isolation behaviour,
migration correctness, SQL dialect differences, serialisation edge cases. An in-memory H2 or SQLite
substitute tests your code against a database you do not run in production, which is why those
suites pass while production fails.

#### 2. Contract tests instead of cross-service integration tests

**Theory.** End-to-end tests across many services are slow, flaky, and fail for reasons unrelated to
the change. Contract tests give most of the confidence at a fraction of the cost by verifying each
side of each boundary independently (see `M24`).

**Example.** With Pact: the consumer's test suite records the requests it makes and the responses it
needs; the provider replays those expectations in its own pipeline. Neither service needs the other
running. A provider that breaks a consumer finds out in its own build, in seconds, with the
consumer's name in the failure message.

**Advanced.** For asynchronous systems the equivalent is schema-registry compatibility checks in CI
(see `Q19`) plus consumer tests against recorded example messages. A useful extra is a test that
replays a captured sample of **real production messages** through the consumer — it catches the
shapes your hand-written fixtures never contain.

#### 3. Fault injection and chaos engineering

**Theory.** Your resilience code — timeouts, retries, circuit breakers, fallbacks — is the least
tested code you have, and it only runs during incidents. Chaos engineering tests it deliberately:
form a hypothesis about steady-state behaviour, inject a failure, and check whether the hypothesis
holds.

**Example.** A proper experiment: "if the recommendation service returns 500 for all requests, the
product page still renders within 500ms with no recommendations, and checkout is unaffected". Inject
the failure in staging, or in production for a small percentage of traffic, and verify. Test cases
worth covering: a dependency being slow (usually worse than being down), a dependency returning
errors, a broker being unavailable, a replica lagging, a pod being killed mid-request.

**Advanced.** Run experiments with a defined blast radius and an abort condition, during working
hours, with the owning team watching — "game days". Chaos in production with no plan is not
engineering. Note that **slow** is the most valuable fault to inject and the most commonly skipped:
most systems handle a clean error well and handle a 30-second delay catastrophically.

#### 4. Testing in production

**Theory.** Some properties cannot be tested anywhere else — real traffic shapes, real data volumes,
real third-party behaviour. The techniques: shadow traffic, canaries with automated analysis,
feature flags with staged rollout, and synthetic probes that continuously exercise the critical
journey.

**Example.** A synthetic check that performs a real checkout with a test account every minute from
three regions. It catches broken journeys that per-service monitoring misses entirely — every
service reporting healthy while the flow between them is broken.

**Advanced.** Testing distributed correctness deterministically is the frontier: deterministic
simulation testing (FoundationDB, Antithesis) runs the whole system in a simulated world where
message delays, reordering and crashes are controlled by a seed, so a failure is exactly
reproducible. Worth naming as the state of the art. For everyday work, the achievable version is
property-based testing of message handlers — generate random orderings and duplicates of events and
assert the final state is always correct, which is the cheapest way to find idempotency and ordering
bugs (`M16`).

### Interview questions

- "How do you test that your saga compensates correctly when service B dies mid-flow?"
- "Why not just write end-to-end tests?"
- "What would you inject in a chaos experiment, and what would you assert?"
- "Your test suite uses in-memory SQLite. What bugs does that hide?"
