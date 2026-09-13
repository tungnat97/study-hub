[← back to the field index](README.md)

# Microservices · Part 3 — Events, Data Ownership & Coordination

Nodes `M17`–`M23`.

---

## M17 · Event-driven architecture

`Intermediate` · Requires: `M02`, `M03` · Unlocks: `M18`, `M19`, `M24`, `Q10`, `Q19`

### Preface

In an event-driven system, a service announces what happened and does not care who listens.
`OrderPlaced` goes out; Email, Analytics, Inventory and Loyalty each react on their own.

The benefit is that adding a fifth listener requires no change to the publisher. The cost is that
no single place describes the whole flow, so understanding "what happens when an order is placed"
means reading five services.

### Details

#### 1. Events are facts, not instructions

**Theory.** An event states something that already happened, in the past tense, and it is
immutable. A command tells a specific service to do something. Publishing an event named after an
action ("SendEmail") is a command in disguise and rebuilds the coupling you were removing.

**Example.** Good: `OrderPlaced`, `PaymentAuthorised`, `StockReserved`, `UserRegistered`. Bad:
`SendWelcomeEmail`, `UpdateInventory`, `ProcessOrder`. With `UserRegistered`, the email service
decides to send a welcome email; later, marketing adds a listener without telling anyone. With
`SendWelcomeEmail`, the publisher has decided what the consumer does, and every new behaviour means
changing the publisher.

**Advanced.** The naming rule also protects you from a subtle ownership problem. Since the producer
owns the event schema, an event named after a consumer's action makes the producer responsible for
the consumer's concerns. Domain events named after business facts stay stable for years; action
events change every time a downstream requirement changes.

#### 2. Thin events versus fat events

**Theory.** A **thin** event (notification) carries just an identifier: `{"orderId": "o1"}`.
Consumers call back for details. A **fat** event (event-carried state transfer) carries the data
consumers need, so they never call back.

**Example.** Thin: Email receives `OrderPlaced{orderId}` and calls `GET /orders/o1` for the
details. Simple and always current, but it creates a synchronous dependency back on Order — if
Order is down, consumers cannot process, so you reintroduced coupling. Fat:
`OrderPlaced{orderId, customerEmail, items, total}`. Email works even if Order is offline.

**Advanced.** Three things to say about fat events. First, they age — the event captured the email
address at that moment, which may since have changed; for some use cases that is correct (an
invoice should show the address at the time) and for others wrong. Second, they leak your internal
model to every consumer, so design the event as a deliberate public contract, not a dump of your
entity. Third, they can carry data a consumer should not see, which makes events a privacy surface
(see `S12`). A common middle ground is a fat event with the fields consumers genuinely need, plus
an id for anything else.

#### 3. What decoupling actually removes

**Theory.** Events remove **temporal** coupling — the consumer need not be running when the
producer publishes. They do not remove **semantic** coupling — the consumer still depends on the
meaning and shape of the event.

**Example.** You rename `total` to `totalAmount` in the event. No compiler complains, no HTTP call
fails, and four consumers silently start reading `undefined`. Schema-less events feel decoupled
right up to the moment they break everything at once. This is why a schema registry with
compatibility checks (see `Q19`) is not optional at scale.

**Advanced.** There is also a hidden coupling in *when* events are published. If a consumer assumes
`PaymentAuthorised` always arrives after `OrderPlaced`, you have an ordering dependency that the
broker only guarantees within a partition and only if both use the same key (see `Q14`). Consumers
that tolerate any order — by storing what they receive and acting when they have enough — are far
more robust than consumers that assume a sequence.

#### 4. Keeping an event-driven system understandable

**Theory.** The main long-term risk is that nobody can answer "what happens when X occurs?". The
countermeasures are documentation and tooling, not architecture.

**Example.** What mature teams have: an **event catalogue** (every event, its schema, its owner,
its consumers — generated from the registry, not hand-written); **distributed tracing** that
propagates through the broker so one trace shows the full chain (see `O15`); **correlation and
causation ids** on every message, so you can reconstruct a flow from logs; and orchestration for
business-critical flows so at least those are explicit (see `M14`).

**Advanced.** A practical rule that keeps systems sane: use **choreography for notifications**
(things that are nice to happen: emails, analytics, cache warming) and **orchestration for business
transactions** (things that must complete correctly: payment, fulfilment). This gives the
flexibility of events where flexibility is valuable, and explicit state where correctness matters.

### Interview questions

- "Thin or fat events — which and why?"
- "Why is publishing an event called `SendWelcomeEmail` a mistake?"
- "How do you stop an event-driven system from becoming impossible to reason about?"
- "Events removed coupling — which coupling exactly, and which remains?"

---

## M18 · Event sourcing

`Expert` · Requires: `M15`, `M17` · Unlocks: `M19`

### Preface

Normally you store the current state: the order row says `status = SHIPPED`. With event sourcing
you store the *sequence of changes* instead — `OrderCreated`, `ItemAdded`, `OrderPaid`,
`OrderShipped` — and compute the current state by replaying them.

The log becomes the source of truth. You gain a perfect audit trail and the ability to answer
questions about the past you did not anticipate. You pay with significant complexity, and the bill
arrives years later when you need to change an old event's shape.

It is a specialised tool. Knowing when *not* to use it is the senior answer.

### Details

#### 1. How it works

**Theory.** Writes append events to a per-entity stream; they are never updated or deleted. Reading
an entity means loading its events and folding them into current state. To avoid replaying
thousands of events, you periodically store a **snapshot** and replay only events after it.

**Example.** A bank account with events `Deposited(100)`, `Withdrew(30)`, `Deposited(50)`. Balance
is computed as 120. After 1,000 events you store `Snapshot(balance=120, version=1000)` and future
loads start there. Concurrency is handled with an expected-version check on append: if the stream
is at version 1000 and you try to append expecting 999, you get a conflict — optimistic locking
built into the model.

**Advanced.** Two rules keep it correct. An event must be **self-contained**: it records what
happened with all data needed to apply it, never "the price changed to what the price service
currently says". And event handlers must be **pure and deterministic** — no clocks, no random
values, no external calls — or replaying produces a different state than the first run, which
defeats the entire model.

#### 2. What you actually gain

**Theory.** A complete, ordered history; the ability to build new views of the past; temporal
queries ("what was the state on 3 March?"); debugging by replaying exactly what happened; and an
audit trail that is the system of record rather than a parallel log that can drift.

**Example.** Product asks: "how many customers added an item, then removed it, then bought it
later?" With current-state storage the data is simply gone. With event sourcing you write a new
projection over the existing log and answer it. This ability to answer unanticipated questions is
the strongest argument for the pattern.

**Advanced.** Fixing bugs gains a new option: if a projection was computed wrongly for six months,
you fix the code and rebuild the projection from the log — the true history was never lost. But
note the limit: if the **event data itself** was wrong (a bug wrote the wrong amount), replay
faithfully reproduces the wrong value. You then have to append a correcting event, never edit
history.

#### 3. The costs, honestly

**Theory.** Event versioning is the real long-term cost. A five-year-old event still has to be
readable by today's code. You need **upcasting** — transforming old event shapes into new ones on
read — and that logic accumulates. Beyond that: every query needs a projection, so ad-hoc querying
disappears; debugging requires tooling; and most developers have never used it, so onboarding is
slow.

**Example.** `OrderPlaced v1` had `price`; `v2` split it into `netPrice` and `tax`. Your upcaster
must derive v2 from v1 using the tax rate that applied at that time — information the old event may
not contain. These migrations are genuinely hard and cannot be avoided by "just updating the old
events", because the log is immutable.

**Advanced.** Deleting personal data collides directly with immutability, which matters under GDPR.
The accepted technique is **crypto-shredding**: encrypt personal fields with a per-subject key held
outside the log, and delete the key to erase the data. The events remain, but the personal fields
become permanently unreadable. Be ready to say this — it is the standard answer and interviewers
use it to separate people who have read about event sourcing from people who have operated it.

#### 4. When to use it and when not to

**Theory.** Use it where history *is* the product: accounting and ledgers, trading, insurance
claims, anything audited by regulators, and collaborative or versioned documents. Avoid it for
ordinary CRUD, reference data, and anywhere the team lacks experience with it.

**Example.** A reasonable hybrid that most systems land on: event-source the one or two aggregates
where history matters (the payments ledger), and use ordinary tables for everything else
(user profiles, product catalogue). You are not obliged to adopt it system-wide, and doing so is
usually a mistake.

**Advanced.** Clear up two common confusions, because interviewers plant them. Event sourcing does
**not** require CQRS, although it pairs naturally with it because you need projections to query.
And using Kafka does **not** mean you are event sourcing — Kafka is a transport with retention;
event sourcing is a storage model where the log is authoritative and entities are rebuilt from it.
Kafka's default retention deletes old messages, which would destroy an event-sourced system.

### Interview questions

- "You need to fix a bug in how state was derived six months ago. How does event sourcing help,
  and how does it hurt?"
- "How do you delete a user's personal data from an immutable log?"
- "Does using Kafka mean you are doing event sourcing?"
- "What is an upcaster and why will you eventually need one?"

---

## M19 · CQRS and read models

`Advanced` · Requires: `M12`, `M17`, `M18` · Unlocks: `M23`, `SD05`, `SD12`

### Preface

CQRS means Command Query Responsibility Segregation: use one model for writing and a different one
for reading.

The reason is that the two have opposite needs. Writing wants a normalised model that enforces
rules. Reading wants exactly the shape the screen needs, pre-joined, so one query serves it. Trying
to make one model do both is where slow, heavily-joined queries come from.

The cost is that read models are built asynchronously, so they lag behind writes.

### Details

#### 1. The basic split

**Theory.** Commands change state and return little. Queries read state and change nothing. At the
lightest level this is just separate code paths in one service and one database. At the heavy end,
writes go to one store and reads to a different store kept up to date by events.

**Example.** Light CQRS, which is where most teams should stop: the write path loads an `Order`
aggregate through the ORM and enforces invariants; the read path runs a hand-written SQL query
returning exactly the twelve columns the order list screen shows, bypassing the ORM entirely. One
database, two models, no lag, most of the benefit.

**Advanced.** Full CQRS with separate stores is the version that costs real money: a second data
store to run, a projection pipeline to build and monitor, and a replay mechanism for when it breaks.
Adopt it when the read and write workloads genuinely differ — for example writes at a few hundred
per second into Postgres and reads at fifty thousand per second served from Elasticsearch or a
denormalised cache.

#### 2. Projections

**Theory.** A projection listens to events and maintains a read-optimised view. It must be
**rebuildable from scratch**, because you will change it, and it will break.

**Example.** An `order_summary` table with customer name, item count, total and status, updated by
handlers for `OrderPlaced`, `OrderPaid` and `OrderShipped`. The order list screen becomes a single
indexed select with no joins. When you add a column, you rebuild the projection from the event
history rather than writing a migration that backfills by querying five services.

**Advanced.** Operational requirements people forget: projections need a **version** so you can run
old and new side by side and switch atomically; they need **lag monitoring**, because a stalled
projection shows users stale data with no error anywhere; they need **idempotent handlers**, since
events are delivered at least once; and rebuilding must be possible without downtime — build into a
new table, verify, then swap.

#### 3. Projection lag and the user experience

**Theory.** After a write, the read model has not caught up. Naively, the user saves something and
does not see it. Same problem as `M12`, and the same fixes.

**Example.** Three practical options, best first: **return the created entity from the write
endpoint** so the UI updates without re-reading; **optimistic UI** insert on the client; or **wait
for the version** — the write returns a version token, and the read endpoint waits until the
projection has reached it (with a short timeout falling back to the write store).

**Advanced.** The honest cost of full CQRS is that this problem appears on every screen, not just
one. That is the reason to keep the projection pipeline's lag well under a second and to alert on
it. Interviewers often push here — the answer they want is that eventual consistency is a
**product** decision made per screen, not a blanket technical choice.

#### 4. When CQRS is the wrong answer

**Theory.** It is overkill when reads and writes have similar shape and volume, when the team is
small, and when a well-designed index or a materialised view would do.

**Example.** A CRUD admin panel with 40 screens does not need CQRS. The reflexive senior answer to
"our list query is slow" is: check the index, check for N+1, add a covering index, consider a
materialised view refreshed every minute — and only then discuss a separate read store.

**Advanced.** State the alternatives ladder explicitly: indexes and query rewriting → denormalised
columns maintained by triggers or application code → materialised views → a separate read table
maintained by events → a separate read *store*. Each step costs more operationally. Reaching for
the last one first is the mistake CQRS is famous for.

### Interview questions

- "After a write the list endpoint does not show the new row. Fixes?"
- "Where is CQRS overkill? What would you do instead?"
- "How do you change the shape of a read model in production?"
- "What do you monitor on a projection?"

---

## M20 · Consensus, quorums and leader election

`Expert` · Requires: `M08`, `M11`, `M13` · Unlocks: `M31`, `M33`, `DB35`

### Preface

Sometimes several machines must agree on one value: who is the leader, what the configuration is,
whether a transaction committed. Consensus algorithms make that agreement safe even when machines
crash and messages are lost.

You will almost never implement one. You will use something that does — etcd, ZooKeeper, Consul,
or a database that embeds Raft. What you need is the intuition, the vocabulary, and knowledge of
where consensus belongs in a design.

### Details

#### 1. Quorums: why a majority

**Theory.** A quorum is the number of nodes that must agree for a decision to count. With a
majority (more than half), any two quorums necessarily share at least one node — so a new decision
always overlaps with the previous one and cannot contradict it. That overlap is the whole trick.
With 2f+1 nodes you tolerate f failures.

**Example.** Five nodes, quorum of three. A network split of 3 and 2: the side with three can still
make decisions; the side with two cannot, so it cannot contradict them. When the split heals, the
minority learns what it missed. With four nodes you still need three for a majority, so you
tolerate exactly one failure — the same as three nodes, at higher cost and latency. Hence odd
numbers.

**Advanced.** In replicated data stores you also see tunable quorums: W + R > N guarantees a read
sees the latest write (some replica in the read set has it). Cassandra and DynamoDB expose this per
request. Note it guarantees *overlap*, not linearizability on its own — without repair mechanisms
you can still read an older value from a lagging replica in the overlap, which is why read repair
and anti-entropy exist (see `M33`).

#### 2. Raft in one paragraph

**Theory.** Time is divided into **terms**. Nodes start as followers; if a follower hears nothing
from a leader it becomes a candidate and requests votes for a new term; a candidate with a majority
becomes leader. All writes go through the leader, which appends to its log and replicates to
followers; once a majority has stored an entry it is **committed** and can be applied. Elections
include a check that prevents a node missing recent entries from winning.

**Example.** A three-node etcd cluster. The leader takes all writes. If it crashes, followers time
out after a randomised interval (randomised precisely to avoid simultaneous candidacies), one wins
the vote, and the cluster continues after a brief unavailability of a few seconds. That short
unavailability is the visible cost of consistency during failover.

**Advanced.** Two safety details worth naming: randomised election timeouts prevent repeated split
votes; and **terms act as fencing tokens** — a message from an older term is rejected, so a
recovered old leader cannot do damage. That is the same fencing idea that makes distributed locks
safe (see `C11`) and is the answer to "what if two nodes think they are leader".

#### 3. Where consensus belongs in a design

**Theory.** Consensus is expensive — every decision costs at least one round trip to a majority. Use
it for **metadata and coordination**, which is low-volume: leader election, cluster membership,
configuration, distributed locks, shard assignment. Do not put it on the per-request data path
unless a system designed for that (a distributed SQL database) is doing it for you.

**Example.** Kafka uses consensus (KRaft, formerly ZooKeeper) to elect a controller and store
metadata, while the actual message traffic flows through partition leaders without a consensus
round per message. That separation is why it can be both consistent about metadata and very fast
for data.

**Advanced.** If you need to elect a leader among your own workers — for a singleton scheduled job,
say — use a lease in etcd, Consul, or even a database row with an expiry, rather than writing an
election. And plan for the window where the lease has expired but the old leader has not noticed:
the work must be safe to do twice, or protected by a fencing token checked at the point of effect.

#### 4. What consensus does not solve

**Theory.** It gives agreement among a known set of nodes, assuming non-malicious behaviour. It does
not give you low latency, does not work if a majority is unreachable, and does not make your
application logic correct.

**Example.** During a network split, the minority side is *unavailable by design*. If your service
needs the consensus store to serve requests, the minority region goes down even though its machines
are healthy. Systems that need to survive that use **static stability** (see `M31`): cache the last
known configuration and keep serving with it when the control plane is unreachable.

**Advanced.** FLP impossibility is the theoretical backdrop: in a fully asynchronous system with
even one faulty process, no algorithm guarantees consensus in bounded time. Real algorithms escape
it with timeouts and randomness — they guarantee safety always and liveness only when the network
behaves. Worth a sentence if the interviewer pushes into theory; do not lead with it.

### Interview questions

- "Explain Raft leader election in 90 seconds."
- "Why 3 or 5 nodes and never 4?"
- "Two nodes both believe they are leader. What prevents corruption?"
- "Where would you put consensus in a system you designed, and where would you keep it out?"

---

## M21 · Time, clocks and ordering

`Advanced` · Requires: `M08` · Unlocks: `M33`, `DB35`, `Q14`

### Preface

Machines disagree about the time. Clocks drift, get corrected in jumps by NTP, and can even move
backwards. So "which of these two events happened first?" cannot be answered by comparing
timestamps from two machines.

The practical rule: use wall-clock time for displaying things to humans, and something else —
sequence numbers, versions, or logical clocks — whenever correctness depends on order.

### Details

#### 1. Why wall clocks lie

**Theory.** A wall clock (`Date.now()`, `System.currentTimeMillis()`) reports civil time and is
periodically corrected by NTP. Corrections can step the clock forwards or backwards. Typical drift
between synchronised servers is milliseconds, but tens of seconds happens when NTP fails, a VM is
migrated, or a machine boots with a bad battery.

**Example.** Measuring a request's duration with two wall-clock readings can produce a negative
number if NTP steps the clock between them. The correct tool is a **monotonic clock** —
`performance.now()` in Node, `System.nanoTime()` in Java, `time.monotonic()` in Python — which only
moves forward and is unaffected by corrections. Use monotonic for durations and timeouts, wall clock
for "when did this happen" in logs.

**Advanced.** Timeouts based on wall clocks are a genuine production hazard: a backwards step can
make a lease appear valid long after it expired, or make a cache entry immortal. Most standard
library timers already use monotonic time; the bug appears in hand-written expiry checks that
compare stored wall-clock timestamps.

#### 2. Last-write-wins and lost updates

**Theory.** If two servers write the same record and the conflict is resolved by "higher timestamp
wins", then the server with the faster clock always wins — including when it should lose. Data
disappears silently, with no error and no log line.

**Example.** Service A (clock 5 seconds fast) and Service B both update a user record. B's update
is genuinely later but carries an earlier timestamp, so it is discarded. The user's change vanishes.
This is a real and common bug in multi-writer systems and in caches populated from several sources.

**Advanced.** The fixes, in order of preference: have a single writer per record (partition by key
so conflicts cannot occur); use a database-assigned sequence or version and reject stale updates
(`WHERE version = $expected`); or use a logical clock so ordering does not depend on physical time.
If you must use timestamps, take them from **one** source — the database's `now()` — not from
application servers.

#### 3. Logical and vector clocks

**Theory.** A **Lamport clock** is a counter each node increments on every event and attaches to
every message; on receipt, a node sets its counter to `max(local, received) + 1`. It guarantees: if
A happened before B, then `L(A) < L(B)`. The converse does not hold — a smaller number does not
prove causality. **Vector clocks** keep one counter per node, which lets you distinguish "A before
B" from "A and B are concurrent", at the cost of size growing with the number of nodes.

**Example.** Vector clocks let a store detect a genuine conflict rather than silently overwriting:
Dynamo-style databases return both versions ("siblings") and ask the application to merge them,
which is the right behaviour for a shopping cart where losing an item is worse than showing a
merged one.

**Advanced.** **Hybrid logical clocks (HLC)** are the practical modern choice: they combine physical
time with a logical counter, so values are close to real time (useful for humans and for range
queries) while still guaranteeing causal ordering. CockroachDB uses them. Spanner takes the opposite
approach with **TrueTime**: expensive hardware (atomic clocks and GPS) bounds clock uncertainty,
and a transaction waits out that uncertainty before committing — trading a few milliseconds of
latency for globally consistent ordering.

#### 4. Ordering in practice

**Theory.** Most systems do not need a global order. They need a **per-entity** order — all events
for order 123 in sequence — which is far cheaper to obtain.

**Example.** In Kafka, set the message key to the aggregate id. All events for that order go to one
partition and are consumed in order. Across different orders there is no ordering, and you do not
need one. This is the single most practical ordering technique in event-driven systems (see `Q14`).

**Advanced.** Where a total order really is required, funnel the writes through a single point that
assigns sequence numbers — a database sequence, a single-partition topic, or a consensus group.
Recognise that this point is a scalability limit by construction, and keep it out of the hot path if
you can. Being able to say "I would use per-key ordering because global ordering does not scale" is
exactly the judgement interviewers look for.

### Interview questions

- "Two services write the same row using `updated_at` from their own clocks. What goes wrong?"
- "How would you order events across services without a global clock?"
- "Which clock do you use for a timeout, and why?"
- "What do vector clocks give you that Lamport clocks do not?"

---

## M22 · Data ownership: database per service

`Intermediate` · Requires: `M01`, `M02` · Unlocks: `M23`, `M25`, `SD04`, `DB25`

### Preface

The rule that makes microservices work: **one service owns its data, and nobody else touches it
directly.** Other services ask through an API or listen to events.

Break this rule and you have a shared database, which means shared schema, coordinated deploys, and
no way to know who will break when you change a column. That is the fastest route back to a
monolith with extra network calls.

### Details

#### 1. Why a shared database defeats the purpose

**Theory.** If two services read and write the same tables, then your schema is a public API with
unknown consumers. You cannot change a column safely, cannot deploy independently, cannot enforce
invariants (another service can write invalid data), and performance problems have no owner.

**Example.** The reporting team adds a query against the orders table. It is unindexed and runs
hourly, holding long transactions. Your service's write latency doubles and autovacuum stops
reclaiming space, and nothing in *your* code changed. Nobody knew that query existed. This is the
everyday reality of a shared database.

**Advanced.** The compliance angle is also real: with a shared database, "who can read customer
personal data" has no meaningful answer, and access reviews become impossible. Ownership gives you
a boundary you can audit, which matters for SOC 2 and GDPR (see `S12`).

#### 2. What ownership actually means

**Theory.** The owning service is the only one that reads or writes its tables. It exposes data
through a synchronous API for queries that need to be current, and through events for others to
maintain their own copies. Other services may **cache or replicate** the parts they need, and are
responsible for keeping their copies fresh.

**Example.** Orders needs the customer's name to display. Options: call Customers per request
(simple, adds a dependency and latency); keep a local copy of `customer_id → name`, updated by
`CustomerRenamed` events (fast, resilient, eventually consistent); or store the name on the order at
creation time (a deliberate historical snapshot — often exactly right for an order). Which you
choose depends on whether you want the current value or the value at the time.

**Advanced.** Duplicating data across services feels wrong to anyone trained on normalisation, and
it is correct here. The difference is that each copy has **one owner and a defined refresh path**,
which is not the same as uncontrolled duplication. Say this explicitly in an interview; it is a
frequent point of confusion and handling it well shows you understand the trade rather than
repeating a rule.

#### 3. Separating a shared database gradually

**Theory.** You rarely get to start clean. The migration path: identify which tables belong to which
capability → stop cross-module SQL by routing all access through a module's code → move tables into
separate schemas with separate database users and permissions → move to separate database instances
→ finally, split the service out.

**Example.** In Postgres you can enforce the boundary early by giving each module its own schema and
its own role, and granting no permissions on other schemas. The moment a query crosses the line, it
fails in development rather than becoming an invisible dependency. That single change does most of
the work of the whole migration.

**Advanced.** The hardest part is joins and foreign keys across the new boundary. You lose
database-enforced referential integrity between services, permanently. Replacements: validate on
write via the owning service; tolerate dangling references in read paths (show "unknown customer"
rather than crashing); and run a periodic reconciliation job that reports orphans. Accepting that
there is no cross-service foreign key is part of the deal.

#### 4. Reporting and analytics

**Theory.** Analytics queries should never run against a service's operational database — different
access patterns, different load profile, and they create an unmanaged dependency. Data belongs in a
warehouse, fed by an explicit pipeline.

**Example.** Change data capture (Debezium) streams row changes from each service's database into
Kafka and then into BigQuery, Snowflake or ClickHouse. Analysts query the warehouse. The service
owner controls what is exported and can change internal tables as long as the exported contract
holds.

**Advanced.** CDC from internal tables does recreate schema coupling with the data team, more
politely. The stronger version is to publish **domain events** for analytics as well as for
services, so the analytics contract is a deliberate event schema rather than your table layout.
Most organisations land in between: CDC for speed, with a documented agreement that certain tables
are contracts.

### Interview questions

- "Another team asks for read-only access to your Postgres for a report. Your answer?"
- "How do you handle a foreign key between two services?"
- "Isn't duplicating customer names across services just denormalisation gone wrong?"
- "Walk me through separating a shared database without downtime."

---

## M23 · Querying across services

`Advanced` · Requires: `M12`, `M19`, `M22` · Unlocks: `SD12`

### Preface

Once each service owns its data, a screen that needs data from three services becomes a problem
that SQL used to solve with a join.

There are three answers, and picking between them is a common interview exercise: call each service
and combine the results in code; maintain a pre-joined read model; or export everything to a
warehouse for analytics.

### Details

#### 1. API composition

**Theory.** The caller (a BFF or gateway) queries each service and combines the results in memory.
Simple, no new infrastructure, always reasonably current. It breaks down as soon as you need to
filter, sort or paginate across services.

**Example.** An order detail page calls Orders, then Customers and Payments in parallel with the
ids from the first response, and merges. Fine. Now try "all orders from customers in Germany, sorted
by value, page 7" — Orders does not know the country, Customers does not know the value. You would
have to fetch *all* German customers and *all* their orders into memory and sort there, which stops
working the moment the data is large.

**Advanced.** Two further hazards. **N+1 across the network**: fetching 50 orders then calling
Customers 50 times. Fix it with a batch endpoint (`GET /customers?ids=...`) or a DataLoader-style
batcher — this is the network equivalent of the ORM N+1 problem. And **partial failure**: if one of
three services fails, decide in advance whether the page degrades or errors, per field.

#### 2. Materialised read model

**Theory.** Maintain a table that already contains the joined data, updated by events from each
owning service. Queries become a single indexed select, and filtering, sorting and pagination work
normally. The cost is an extra store, a pipeline, and eventual consistency.

**Example.** An `order_search` table holding order id, total, status, customer name, customer
country and created date — populated by `OrderPlaced`, `OrderPaid`, `CustomerUpdated` events. The
admin screen queries it directly. This is CQRS applied across services (see `M19`).

**Advanced.** Two practical requirements. It must be **rebuildable**: keep the ability to replay
events or re-import from the owning services, because the projection will get out of sync and you
need a repair path that is not manual SQL. And it needs an **owner** — usually the team that owns
the screen, not the teams that own the source data, so that adding a field does not require three
teams to agree.

#### 3. The warehouse for analytics

**Theory.** Analytical questions — aggregates over long periods, joins across everything, ad-hoc
exploration — should not be served by operational stores at all. Export to a columnar warehouse and
query there.

**Example.** "Revenue by country by month for two years, split by acquisition channel" is a
warehouse query. Trying to serve it from a materialised view in your operational database competes
with production traffic and will not perform, because row stores are the wrong shape for it (see
`DB32`).

**Advanced.** Draw the line explicitly in an interview: **operational queries** (a user is waiting,
small result, needs recent data) go to a read model; **analytical queries** (nobody is waiting,
large scans, minutes of staleness fine) go to the warehouse. Candidates who conflate the two end up
proposing Elasticsearch for reporting or a warehouse for a product screen.

#### 4. What never to do

**Theory.** Do not join across services in the database — federated queries, foreign data wrappers,
or one service reading another's tables. It reintroduces every coupling problem of a shared
database, and adds unpredictable performance.

**Example.** A Postgres foreign data wrapper pointing at another service's database looks elegant
and makes the other team's schema change your outage. Similarly, do not give a "reporting" service
direct read credentials to five databases; it becomes the thing that blocks every schema change.

**Advanced.** The one acceptable close relative is a **read replica dedicated to an export
pipeline**, owned by the service team, feeding CDC into the warehouse. The difference is that the
contract is the exported stream, not live SQL access, and the owning team controls it.

### Interview questions

- "The admin page must filter orders by customer country, sort by order value and paginate.
  Customer and Order are separate services. Design it."
- "When is API composition good enough?"
- "How do you avoid an N+1 across service boundaries?"
- "Where does the boundary between a read model and a data warehouse sit?"
