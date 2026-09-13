[← back to the field index](README.md)

# Caching, Queues & Streaming · Part 3 — Stream Processing, CDC & Job Design

Nodes `Q17`–`Q21`.

---

## Q17 · Stream processing

`Expert` · Requires: `Q12`, `Q14`, `Q15` · Unlocks: `SD12`

### Preface

Consuming messages one at a time is simple. Computing something **across** messages — a count per
hour, a join between two streams, a running total — needs state, and state in a distributed
stream processor raises questions a simple consumer never faces.

The two ideas that matter most: the difference between **event time** and **processing time**, and
**watermarks**, which are how a system decides it has waited long enough for late data.

### Details

#### 1. Stateless versus stateful

**Theory.** Stateless operations (map, filter, enrich from a lookup) process each record
independently, so a restart just resumes from the last offset. Stateful operations (count, aggregate,
join, deduplicate) accumulate state that must survive restarts and be redistributed when partitions
move.

**Example.** Kafka Streams keeps state in a local RocksDB store and mirrors every change to a
**changelog topic** in Kafka. If the instance dies, another rebuilds the state by replaying the
changelog. That is the general pattern: local state for speed, a durable log for recovery.

**Advanced.** State makes rebalancing expensive: reassigning a partition means rebuilding potentially
gigabytes of state before processing resumes. Mitigations are **standby replicas** (a warm copy on
another instance) and **static membership** (`Q15`) so a restarting pod keeps its partitions. For a
stateful processor, a rolling deploy without these can mean minutes of unavailability per instance.

#### 2. Event time versus processing time

**Theory.** **Event time** is when the thing happened, recorded in the event. **Processing time** is
when your code sees it. They differ by network delay, retries, consumer lag, and a mobile device being
offline for an hour.

**Example.** "Orders per hour" computed by processing time is wrong whenever there is lag: a backlog
drained at 10:00 puts 09:00's orders into the 10:00 bucket, and a replay puts a week of history into
one minute. Computed by event time, the result is correct and **reproducible** — reprocessing gives
the same answer, which is the property you need for anything anyone will act on.

**Advanced.** Event time requires the producer's clock, which you cannot trust (`M21`). A device with
a wrong clock produces events timestamped next year, which can push a watermark far into the future
and cause every subsequent event to be treated as late. Defences: clamp timestamps to a sane range,
use a server-assigned ingestion timestamp alongside the event timestamp, and monitor the distribution
of the difference.

#### 3. Windows and watermarks

**Theory.** A window groups events by time. **Tumbling** windows are fixed and non-overlapping
(every hour). **Hopping** windows overlap (every hour, advancing every 15 minutes). **Session**
windows group activity separated by a gap of inactivity. A **watermark** is the system's assertion
that no more events older than time T are expected.

**Example.** "Count orders per merchant per hour, with events up to 10 minutes late": tumbling
one-hour windows on event time, with a watermark lagging the maximum observed timestamp by 10
minutes. The 09:00-10:00 window is emitted at 10:10. Events arriving after that are **late** and you
must choose: drop them, update the already-emitted result, or send them to a side output for
reconciliation.

**Advanced.** The trade is explicit and worth stating: a longer allowed lateness means more correct
results and more delay before any result appears, plus more state held open. There is no right answer
— it is a product decision about freshness versus completeness. The mature design emits an early
result and **corrects** it later (a retraction or an updated value), which is what a
"materialised view that converges" means in practice.

#### 4. The tools

**Theory.** **Kafka Streams** is a library embedded in your application — no separate cluster, scales
by running more instances, Kafka-only. **Flink** is a distributed processing engine with a cluster of
its own, richer windowing and state, better support for very large state, and genuine exactly-once
across many sinks. **ksqlDB** offers SQL over streams.

**Example.** Choose Kafka Streams when the processing is part of a service you already run and the
state is modest — it is much less operational overhead. Choose Flink when the job is a first-class
data pipeline, state is large, you need sophisticated event-time handling, or you are joining several
sources.

**Advanced.** The **stream-table duality** is the concept to be able to explain: a stream of changes
can be folded into a table (the current state), and a table's changes can be emitted as a stream.
Kafka expresses this as a KStream versus a KTable over a **compacted** topic (`Q18`). It is the same
insight as event sourcing (`M18`) and CQRS (`M19`) — the log is primary and the table is a derived
view — and connecting those three is a strong senior observation.

### Interview questions

- "Count orders per merchant per hour, with events arriving up to 10 minutes late. Design it."
- "Event time versus processing time — give a bug caused by using the wrong one."
- "What is a watermark and what does a longer one cost?"
- "Why is a rolling deploy of a stateful stream processor slow?"

---

## Q18 · Log compaction, change data capture and the outbox

`Advanced` · Requires: `Q12`, `M15` · Unlocks: `Q19`, `DB32`

### Preface

Two features turn Kafka from a transport into a place state can live. **Log compaction** keeps the
latest value per key forever, so a topic becomes a durable snapshot rather than a rolling window.
**Change data capture** reads a database's replication log and turns every row change into an event.

Together they are how data gets reliably from an operational database into everything else — search
indexes, caches, warehouses, other services — without dual writes (`M15`).

### Details

#### 1. Log compaction

**Theory.** A compacted topic retains, for each key, at least the **most recent** value, deleting
older versions in the background. Retention is by key rather than by time, so the topic is bounded by
the number of distinct keys, not by the volume of updates.

**Example.** A `customers` topic keyed by customer id, compacted: a new consumer reads from the
beginning and receives the current state of every customer — a full snapshot — then continues with
live updates. That is how a service bootstraps a local copy of reference data without an API call per
record, and it is the foundation of a KTable (`Q17`).

**Advanced.** A **tombstone** is a record with a key and a `null` value: compaction removes the key
entirely after `delete.retention.ms`. This is how deletes propagate, and it is also how you satisfy a
deletion request in a compacted topic (`S12`) — one of the few ways to remove data from a log. Note
that compaction is asynchronous and best-effort, so consumers may still see older versions of a key
for a while; it guarantees convergence, not immediacy.

#### 2. Change data capture

**Theory.** CDC reads the database's own replication log — the Postgres WAL (`DB12`) or the MySQL
binlog — and emits an event per committed row change. Because it reads the log, it captures every
change regardless of which application made it, in commit order, with no polling.

**Example.** Debezium, running on Kafka Connect, produces messages with `before` and `after` images
and an operation type (`c`, `u`, `d`, `r` for a snapshot read). It begins with a consistent snapshot
of existing rows and then streams changes. Downstream: a search indexer, a cache updater, a warehouse
loader (`DB32`), or another service's read model (`M19`).

**Advanced.** The operational hazard is the **replication slot** (`DB12`): the database retains WAL
until the connector has consumed it, so a stopped or lagging connector fills the primary's disk and
takes the database down. Monitor slot lag, alert on it early, and set `max_slot_wal_keep_size` so the
database sacrifices the slot rather than itself. This is a genuinely common and severe incident, and
naming it is a strong signal.

#### 3. CDC on business tables versus the outbox

**Theory.** You can capture changes from your ordinary tables, or you can write purpose-built events
to an `outbox` table in the same transaction and capture only that (`M15`).

**Example.** The trade:
- **CDC on business tables** — no application change, captures everything, and the event shape **is
  your schema**, so every consumer is coupled to your table structure and a column rename breaks them.
- **Outbox** — you write a deliberate domain event (`OrderPlaced` with the fields consumers need) in
  the same transaction as the data, and CDC ships it. Your internal schema stays free to change.

The outbox is better wherever other services consume the events; direct CDC is fine for a
warehouse pipeline you own on both sides.

**Advanced.** The outbox also solves the **granularity** mismatch: one business event may span several
table writes, and CDC on business tables emits several unrelated row changes that consumers must
reassemble — including working out which ones belonged to the same transaction. An outbox row is one
event with the correct boundary, which is why it is worth the extra write.

#### 4. Ordering and delivery from CDC

**Theory.** CDC preserves the database's commit order within a table's stream. To preserve per-entity
ordering downstream, set the message key to the entity id so all its changes land in one partition
(`Q14`).

**Example.** Delivery is **at least once**: a connector restart can re-emit changes since its last
committed position, so consumers must be idempotent (`M16`). Debezium includes the log position in
each message, which gives consumers a natural version for "ignore anything older than what I have
applied".

**Advanced.** During the initial **snapshot** phase, records are emitted with an operation type of
`r` and may interleave with live changes depending on the snapshot mode. Consumers must handle a
snapshot record for a row they have already seen updated — again solved by version-based idempotency
rather than by ordering. Knowing that the snapshot is a distinct phase with its own semantics is a
practical detail that only comes from having run it.

### Interview questions

- "Compare outbox-with-poller and Debezium CDC for publishing order events."
- "How does a compacted topic let a new consumer bootstrap its state?"
- "An inactive Debezium connector took down the database. Explain the mechanism."
- "How do you delete data from a Kafka topic?"

---

## Q19 · Event schemas and evolution

`Advanced` · Requires: `Q12`, `A17`, `M17`, `M24` · Unlocks: `SD14`

### Preface

An event is a contract with consumers you may not know about, stored for days or years, and read by
code deployed at different times.

That makes schema evolution harder than an API: you cannot simply coordinate a release, because old
messages already exist and old consumers are already running. A **schema registry** enforces the
rules automatically.

### Details

#### 1. The schema registry

**Theory.** Producers register a schema; the registry assigns an id and **rejects incompatible
changes** according to a configured mode. Messages carry only the small schema id, and consumers fetch
the schema to deserialise.

**Example.** Two benefits beyond compatibility: messages are smaller (no field names on the wire,
unlike JSON), and a consumer written against schema v1 can read a message written with v3 because
Avro resolves the two schemas — filling in defaults for fields it does not know.

**Advanced.** The registry is on the critical path for producers and consumers, so it needs caching
(clients cache schema-id-to-schema mappings) and high availability. A registry outage should not stop
a consumer that has already cached what it needs — verify that behaviour rather than assuming it.

#### 2. Compatibility modes

**Theory.**
- **BACKWARD** — a new **consumer** can read data written with the previous schema. Consumers upgrade
  first. (Allows deleting fields and adding optional ones.)
- **FORWARD** — an old **consumer** can read data written with the new schema. Producers upgrade
  first. (Allows adding fields and deleting optional ones.)
- **FULL** — both.
- **TRANSITIVE** variants check against **all** previous versions, not just the last.

**Example.** Reasoning it out for a rolling deploy: both old and new producers, and old and new
consumers, are running simultaneously. Old consumers will see new messages (needs forward
compatibility) and new consumers will see old messages (needs backward). So a rolling deploy requires
**FULL**. And because messages are retained for days, a consumer may encounter a schema several
versions old — so **FULL_TRANSITIVE** is the safe default for long-retention topics.

**Advanced.** In practice the safe change set is small and worth memorising: **add a field with a
default**, or **remove a field that has a default**. Anything else — renaming, changing a type,
adding a required field, changing an enum's meaning — is breaking. Renaming is done as add-new,
dual-write, migrate consumers, remove-old: expand and contract (`M24`) applied to events.

#### 3. The event envelope

**Theory.** Separate the metadata every event needs from the payload specific to the event type. A
consistent envelope makes generic tooling possible — routing, tracing, deduplication, auditing.

**Example.** A reasonable envelope:

```json
{
  "eventId": "01J...",            // unique; the dedup key (M16)
  "eventType": "order.placed",
  "eventVersion": 2,
  "occurredAt": "2026-09-13T10:00:00Z",
  "producer": "orders-service",
  "traceId": "4bf92f...",         // propagates tracing through the broker (O15)
  "correlationId": "...",         // ties a business flow together
  "causationId": "...",           // the event or command that caused this one
  "tenantId": "acme",
  "data": { ... }
}
```

**Advanced.** `correlationId` and `causationId` together let you reconstruct a complete causal chain
across services from logs alone — invaluable when tracing has expired or was sampled away (`M25`).
And `tenantId` in the envelope lets infrastructure route, filter and audit by tenant without parsing
the payload, which matters for data residency (`SD13`).

#### 4. Events as a product

**Theory.** If other teams consume your events, they are a public interface and deserve the same
treatment as an API: documentation, ownership, versioning, and a deprecation process.

**Example.** What good looks like: an event catalogue generated from the registry showing every event,
its schema, its owner and its consumers; naming conventions (`<domain>.<entity>.<past-tense-verb>`);
a review step for new event types; and consumer registration so you know who to contact.

**Advanced.** The most common organisational failure is events that leak internal schema — because
they were generated from tables (`Q18`) or built by copying an entity. Then every internal refactor
becomes a cross-team negotiation, which is exactly the coupling microservices were meant to remove
(`M17`). Designing the event as a deliberate contract, distinct from the internal model, is the
discipline that keeps event-driven architecture workable over years.

### Interview questions

- "Producers deploy before consumers. Which compatibility mode do you need and why?"
- "What is the complete set of safe changes to an Avro schema?"
- "What goes in an event envelope and why?"
- "How do you rename a field in an event consumed by six services?"

---

## Q20 · Choosing the right async tool

`Intermediate` · Requires: `Q10`, `Q11`, `Q12` · Unlocks: `Q21`, `SD06`

### Preface

Kafka is not the answer to every asynchronous problem, and reaching for it by default is a common
mistake that costs an operational commitment out of proportion to the need.

The honest hierarchy runs from "a database table" through task queues and brokers to event logs and
workflow engines. Pick the least powerful thing that solves the problem.

### Details

#### 1. A database-backed queue

**Theory.** A table with a status column, claimed with `SELECT ... FOR UPDATE SKIP LOCKED` (`DB09`).
No new infrastructure, and the job claim is **transactional with your data**.

**Example.** Why it is better than people assume: you can enqueue a job in the same transaction as the
business write (no dual-write problem, `M15`), query jobs with SQL, and see everything in one place.
Postgres handles thousands of jobs per second comfortably. Ideal for background work in a system that
already has a database and no other messaging.

**Advanced.** Where it stops: very high throughput (tens of thousands per second), where the table
becomes a hot spot and vacuum struggles with the churn (`DB08`); fan-out to many independent
consumers; and replay of historical events. Until then it is the simplest thing that works, and
saying so confidently is a mark of judgement rather than inexperience.

#### 2. Task queues

**Theory.** BullMQ (Redis), Celery, Sidekiq, RQ. Designed for *jobs*: retries, scheduling, priorities,
progress, and a UI to inspect them.

**Example.** Best for application background work — emails, thumbnails, imports, report generation.
They give you per-job visibility, delayed and repeatable jobs, and concurrency control out of the
box, which a raw broker does not. BullMQ on Redis is the natural choice for a Nest application
already using Redis (`F13`).

**Advanced.** Their weakness is durability inherited from the store: BullMQ on Redis loses jobs if
Redis loses data (`Q06`), so for jobs that must not be lost, either accept the risk with an
appropriate persistence configuration, or reconcile from the database (the job is derived from a row
whose state you can re-scan). Celery with RabbitMQ is more durable, and the same reconciliation
advice applies.

#### 3. Brokers and logs

**Theory.** **RabbitMQ/SQS** for decoupling services with flexible routing and per-message handling.
**Kafka/Kinesis/Pulsar** for high throughput, replay, many independent consumers, per-key ordering and
stream processing.

**Example.** SQS deserves a mention for its simplicity: fully managed, effectively infinite scale, at
least once with a visibility timeout, and a FIFO variant offering ordering per **message group id**
with deduplication — limited to 300 transactions per second per group, which is the constraint to
remember. For most teams on AWS, SQS plus SNS covers what RabbitMQ would, with no operations.

**Advanced.** The cost of Kafka is the part to be honest about: brokers to run (or a managed service
to pay for), partition and retention planning, consumer group operations, schema management, and a
team that understands lag and rebalancing. That is a real, ongoing commitment. Adopt it when you need
replay, very high throughput, or many consumers of the same stream — not because it is what large
companies use.

#### 4. Workflow engines

**Theory.** Temporal, AWS Step Functions and similar make a **long-running, multi-step process**
durable: the engine records progress, retries individual steps, handles timeouts, and resumes after a
crash exactly where it stopped.

**Example.** This is the natural home for orchestrated sagas (`M14`): the workflow is written as
ordinary code with awaits, and the engine guarantees that if the process dies mid-flow it resumes with
the same state. Compared with hand-rolling a state machine over a queue, you get history, visibility
and retry semantics for free.

**Advanced.** The trade is another substantial system to run and a programming model with real
constraints (workflow code must be deterministic, because it is replayed to rebuild state — so no
`Date.now()` or random values outside activities). For a handful of multi-step processes it is a lot;
for a business built on them — order fulfilment, onboarding, claims — it is transformative. Naming
Temporal as the alternative to a hand-built saga orchestrator is a strong answer to `M14`.

### Interview questions

- "You need background email sending for a Nest app with 200 rps. What do you use and why not
  Kafka?"
- "When does a Postgres-backed queue stop being enough?"
- "What does adopting Kafka actually cost?"
- "When would you reach for a workflow engine?"

---

## Q21 · Job design, scheduling and fairness

`Advanced` · Requires: `Q08`, `Q16`, `Q20`, `F13`, `F14` · Unlocks: `SD10`

### Preface

Once you have a queue, the design of the jobs themselves determines whether the system is operable.

The rules are short: jobs carry ids not data, jobs are idempotent, jobs are small and restartable, and
no single tenant may consume all the workers. Each of these prevents a specific incident that is
otherwise inevitable.

### Details

#### 1. Job payload design

**Theory.** Send an identifier and let the worker load current data. A payload that embeds a snapshot
is stale by the time it runs, and it is large.

**Example.** `{ orderId: "o1" }` rather than the whole order object. The worker reads the order and
sees its current state — which matters when the job was delayed by a backlog, or retried an hour
later. The exception is when you deliberately want the value **as it was** (an invoice line), and then
it should be a comment in the code explaining why.

**Advanced.** Payloads must also be **version-tolerant**: during a rolling deploy, old workers consume
jobs enqueued by new code and vice versa (`M24`). Adding a required field breaks the old worker. Keep
payloads minimal and additive, exactly like an event schema (`Q19`).

#### 2. Idempotency and claiming

**Theory.** At-least-once delivery means every job may run twice. The job must detect that its work is
already done, or be naturally repeatable (`M16`).

**Example.** Patterns that work: a unique constraint on the effect (`UNIQUE (order_id, notification_type)`
so a second send fails and is caught); a status check at the start (`if (order.status === 'CONFIRMED')
return`); an idempotency key passed to the external provider (`A09`); or a `processed_jobs` table
written in the same transaction as the effect (`M15`).

**Advanced.** The **visibility timeout must exceed the maximum processing time**, or the job reappears
while it is still running and a second worker starts it — producing duplicates that look like a broker
fault (`Q10`). Either extend the lease while working, or split the job so each piece is short. This
single configuration error causes a surprising share of duplicate-processing incidents.

#### 3. Fairness across tenants

**Theory.** A single queue is first-come-first-served. One tenant enqueuing two million jobs occupies
every worker, and everyone else waits behind them.

**Example.** The options:
- **A queue per tenant** with workers round-robining across queues. Fair, and unwieldy beyond a few
  hundred tenants.
- **Priority or weighted queues** — bulk work in a low-priority queue, interactive work in a high one.
  Simple and effective, and the most common answer.
- **Per-tenant concurrency limits** — a tenant may occupy at most N workers, enforced with a counter
  in Redis. Fair and precise.
- **Shuffle sharding** (`M31`) — each tenant is assigned a random subset of workers, so one tenant's
  flood affects only the few they share.

**Advanced.** Separate **bulk** from **interactive** work regardless of tenancy: a user waiting for a
password-reset email must not queue behind a 500,000-row import. Two queues with separate worker pools
is the simplest effective form of the bulkhead pattern (`M10`), and it is usually the first thing to
do when a job system becomes unpredictable.

#### 4. Monitoring and scheduling at scale

**Theory.** The metric that matters is the **age of the oldest unprocessed job**, not the queue depth
(`M30`). Depth without throughput tells you nothing.

**Example.** The dashboard: oldest-job age per queue, throughput, failure rate, retry rate, DLQ
arrivals, and worker utilisation. Alert on age and on a **growing** backlog, not on an absolute depth.
Scale workers on age or depth via a custom metric rather than on CPU (`O06`) — a worker waiting on I/O
shows low CPU while the backlog grows.

**Advanced.** For per-entity scheduled work at scale ("remind each of a million users 24 hours
before"), do not create a million timers. Either use delayed jobs backed by a sorted set scored by due
time (`Q05`), or a table with a `due_at` index scanned by a poller claiming rows with
`FOR UPDATE SKIP LOCKED` (`DB09`). The second handles cancellation and rescheduling naturally, because
the data is the source of truth. And add jitter to bulk due times so a million reminders at 09:00 do
not become a self-inflicted thundering herd (`M09`).

### Interview questions

- "One tenant enqueues two million jobs and everyone else's emails stop. Redesign."
- "Which single metric would you alert on for a job queue?"
- "A job runs twice. Is that a bug?"
- "How do you schedule a reminder for each of a million appointments?"
