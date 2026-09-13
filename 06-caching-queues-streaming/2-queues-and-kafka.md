[← back to the field index](README.md)

# Caching, Queues & Streaming · Part 2 — Queues, Brokers and Kafka

Nodes `Q10`–`Q16`.

---

## Q10 · Messaging fundamentals

`Intermediate` · Requires: `M03`, `C02` · Unlocks: `Q11`, `Q12`, `Q20`, `F13`, `M15`

### Preface

A message broker lets one service hand work to another without waiting. That buys decoupling,
buffering during spikes, retries, and the ability to add new consumers without changing the producer.

It costs you: eventual consistency, harder debugging, no guaranteed ordering by default, and — the
one that shapes everything — **messages are delivered at least once**, so every consumer must be safe
to run twice.

### Details

#### 1. Queue, topic and log

**Theory.** Three shapes, often confused.
- **Queue (work distribution)** — many consumers compete; each message is processed by exactly one of
  them. Used to spread work.
- **Topic (pub/sub)** — each subscriber gets a copy. Used for fan-out.
- **Log** — an ordered, retained sequence; consumers track their own position and can replay. Kafka.

**Example.** "Send a welcome email" is a queue: one worker should do it. "User registered" is a topic
or log: email, analytics and CRM each react independently. Kafka gives you both — a consumer *group*
competes for partitions (queue semantics), while separate groups each receive everything (topic
semantics).

**Advanced.** The distinguishing feature of a log is **retention independent of consumption**: the
message stays for its retention period whether or not anyone read it, so you can add a consumer that
reprocesses the last week, or reset a consumer to reprocess after a bug. A queue deletes on
acknowledgement, so that history is gone. If replay matters, you need a log.

#### 2. Delivery semantics

**Theory.**
- **At most once** — send and forget; messages can be lost. Acknowledge before processing.
- **At least once** — retry until acknowledged; duplicates are possible. Acknowledge after
  processing. **The practical default.**
- **Exactly once** — impossible as a delivery guarantee (`M16`); achievable as an *effect* through
  idempotent consumers.

**Example.** The interleaving that forces this on you: the consumer processes the message, writes to
the database, and crashes before acknowledging. The broker cannot tell how far it got, so it
redelivers. Nothing was faulty — the duplicate is a correct consequence of the protocol.

**Advanced.** The consequence to state clearly: **choosing at-least-once means committing to
idempotent consumers** (`M16`). That is not an extra safeguard, it is the price of the guarantee. A
system that assumes single delivery will eventually double-charge, double-ship or double-send, and
usually in the incident where you can least afford it.

#### 3. Acknowledgement and redelivery

**Theory.** The consumer tells the broker when it is done. Until then, the message is either invisible
to others (a visibility timeout) or held unacknowledged. If the consumer dies, the message becomes
available again.

**Example.** The mechanisms: SQS has a **visibility timeout** — the message is hidden for N seconds
and reappears unless deleted. RabbitMQ holds it unacknowledged and requeues on channel closure. Kafka
tracks a committed **offset** per partition per group.

**Advanced.** The visibility timeout **must exceed the maximum processing time**, or the message
reappears while you are still working on it and a second consumer starts the same job — producing
duplicates that look like a broker fault and are actually a configuration error. Either extend the
timeout while working (SQS supports changing it mid-flight) or split long jobs into shorter ones
(`Q21`).

#### 4. Ordering

**Theory.** Ordering is guaranteed only within a narrow scope, if at all: per queue with a single
consumer, per partition in Kafka, per message group in SQS FIFO. Across those scopes there is none.

**Example.** The practical approach is to need ordering only **per entity**: all events for order 123
in sequence, with no ordering requirement between different orders. Key by the entity id and you get
that cheaply (`Q14`). A global total order means a single consumer and no horizontal scaling.

**Advanced.** Better still, write consumers that do not require ordering: include a version or
timestamp in the event and ignore anything older than what you have applied (`M16`). That makes the
consumer robust to reordering **and** duplication at once, and removes a constraint from the whole
pipeline. It is the single most valuable consumer-design habit.

### Interview questions

- "Your consumer crashes after doing the work but before acking. What happens and how do you make it
  safe?"
- "Queue or event log for this use case?"
- "Why is exactly-once delivery impossible?"
- "How do you get ordering without limiting yourself to one consumer?"

---

## Q11 · RabbitMQ and classic brokers

`Advanced` · Requires: `Q10` · Unlocks: `Q16`, `Q20`

### Preface

RabbitMQ is a traditional message broker: producers publish to an **exchange**, which routes to
**queues** according to bindings, and consumers take messages from queues.

Its strength is flexible routing and per-message handling — acknowledge, reject, requeue, dead-letter,
each individually — which Kafka cannot do. Its weakness is that the broker holds the state, so a deep
queue is the broker's problem.

### Details

#### 1. Exchanges and routing

**Theory.** Four exchange types: **direct** (route by exact routing key), **topic** (route by pattern,
`order.*.created`), **fanout** (to every bound queue), and **headers** (by message attributes).

**Example.** A typical layout: a topic exchange `events`; queues `email`, `analytics` and `inventory`
bound with patterns `order.*`, `#` and `order.placed` respectively. Publishing `order.placed` reaches
all three; `order.cancelled` reaches email and analytics. Adding a consumer means adding a queue and
a binding — no producer change.

**Advanced.** The routing lives in the broker, which is powerful and makes the broker a piece of
configuration that must be version-controlled and deployed (`O10`). Undocumented bindings created by
hand are a classic source of "why is this service receiving these messages?". Declare topology in
code or infrastructure-as-code, never in the management UI.

#### 2. Durability — all three parts

**Theory.** For a message to survive a broker restart, **three** things must be true: the queue is
declared durable, the message is published as persistent, and the publisher uses **publisher
confirms** to know the broker accepted it.

**Example.** Missing any one loses messages silently. A durable queue with non-persistent messages
loses the messages. Persistent messages with no confirms means the publisher believes it sent
something the broker never received. This trio is a common interview question precisely because
partial configuration looks like it works.

**Advanced.** Persistence costs throughput — every message is written to disk. Classic mirrored
queues had known failure modes during network partitions; **quorum queues** (Raft-based) are the
modern replacement and are the correct default for anything important. Being able to say "quorum
queues, not mirrored" is a current, practical detail.

#### 3. Prefetch — the backpressure knob

**Theory.** `basic.qos(prefetch_count)` limits how many unacknowledged messages a consumer may hold.
Without it, RabbitMQ pushes as fast as it can and the consumer buffers them in memory.

**Example.** With `prefetch=1`, a consumer gets one message at a time: perfectly fair distribution,
and a round trip per message limits throughput. With `prefetch=1000`, one consumer can grab a
thousand while another idles, and if it dies all thousand are redelivered. Typical values are 10-100,
tuned by processing time — fast messages want a higher prefetch to amortise round trips, slow ones a
lower one for fairness.

**Advanced.** This is backpressure made explicit (`C16`): prefetch is the bound on your in-memory
buffer. An unbounded prefetch moves the backlog from the broker — where it is durable, visible and
countable — into your process memory, where it is none of those. "Set prefetch deliberately" is the
RabbitMQ-specific version of "bound every queue".

#### 4. Dead-lettering and delays

**Theory.** A rejected message (`basic.nack` with `requeue=false`), an expired message, or one that
overflows a queue length limit can be routed to a **dead-letter exchange**. RabbitMQ has no native
delayed delivery, so the standard trick uses TTL plus dead-lettering.

**Example.** Delayed retry without a plugin: publish the failed message to a `retry.30s` queue that
has a message TTL of 30 seconds and a dead-letter exchange pointing back at the main queue. The
message sits there, expires, and is routed back — a 30-second delay. Chain several such queues for
increasing backoff (`Q16`). The delayed-message-exchange plugin does this more directly.

**Advanced.** Watch out for **head-of-line blocking** in a TTL queue: messages expire in order of
insertion, so a message with a long TTL at the head delays shorter-TTL messages behind it. Use one
queue per delay tier rather than per-message TTLs in a shared queue — a real and surprising
behaviour.

### Interview questions

- "Messages pile up and RabbitMQ memory alarms trigger. What do you tune first?"
- "How do you implement a five-minute delayed retry in RabbitMQ?"
- "What three things are needed for a message to survive a broker restart?"
- "What does prefetch do and how do you choose it?"

---

## Q12 · Kafka fundamentals

`Advanced` · Requires: `Q10` · Unlocks: `Q13`, `Q14`, `Q15`, `Q17`, `Q18`

### Preface

Kafka is not a queue. It is a **distributed, append-only log**: producers append, consumers read at
their own pace by tracking an offset, and messages stay for a retention period regardless of who has
read them.

That design gives enormous throughput, replay, and many independent consumers of the same data. It
also means the unit of parallelism and ordering is the **partition**, and almost every Kafka question
comes back to that.

### Details

#### 1. Topics, partitions and offsets

**Theory.** A topic is split into partitions. Each partition is an ordered, immutable sequence of
records, each with a monotonically increasing **offset**. Ordering is guaranteed **within** a
partition and not across them. Each partition has a leader broker and replicas.

**Example.** A topic with 12 partitions can be consumed by up to 12 consumers in a group, each owning
some partitions. Records with the same key always go to the same partition, so all events for one
order are ordered relative to each other. A 13th consumer in the group sits idle — **partition count
caps parallelism** (`Q14`).

**Advanced.** Increasing the partition count later is possible and **breaks key-to-partition
mapping**: `hash(key) % partitions` changes, so a key that was in partition 3 may move to partition
7, and its new events can be processed before its old ones. Over-provision modestly at the start
(more partitions than consumers you expect) because it is much easier than increasing later.

#### 2. Consumer groups

**Theory.** Consumers with the same `group.id` share the partitions of a topic — each partition is
assigned to exactly one consumer in the group. Different groups each receive **all** the messages
independently, with their own offsets.

**Example.** This is how one topic serves both queue and topic semantics (`Q10`): the `email-service`
group and the `analytics-service` group each read every message, while within each group the
partitions are divided among its instances for parallelism.

**Advanced.** Offsets are committed to an internal topic (`__consumer_offsets`) and represent "the
next offset to read". Committing **before** processing gives at-most-once; committing **after** gives
at-least-once. Auto-commit does it on a timer, which means both duplicates and gaps are possible,
depending on when the timer fires relative to your processing — which is why manual commit after
processing is the correct default for anything that matters (`Q15`).

#### 3. Why it is fast

**Theory.** Three design choices: append-only **sequential** disk writes (fast even on spinning
disks); the **page cache** rather than an application-level cache, so recent data is served from
memory the OS manages; and **zero-copy** transfer from page cache to socket (`C19`), because messages
are stored in the same format they are sent.

**Example.** Kafka does not deserialise messages to serve them — it sends the bytes it stored. That is
why a broker can saturate a network interface with modest CPU, and why a single partition handles tens
of megabytes per second.

**Advanced.** The corollary is that Kafka is happiest when consumers are **caught up**, reading recent
data that is still in the page cache. A consumer replaying from the beginning reads from disk and
competes for I/O with live traffic — so a large backfill can degrade the whole cluster. Schedule
replays deliberately, and consider a separate cluster or throttling for large ones.

#### 4. Cluster structure

**Theory.** Brokers hold partitions; each partition has a leader (which handles all reads and writes)
and followers that replicate. Metadata and controller election used ZooKeeper and now use **KRaft**,
Kafka's own Raft implementation (`M20`).

**Example.** With replication factor 3, each partition exists on three brokers. If a leader fails, one
of the in-sync replicas is promoted; producers and consumers discover the new leader through metadata
requests. Clients are cluster-aware and handle this automatically.

**Advanced.** Note that reads go to the **leader** by default, not to replicas — so replicas are for
durability and failover, not read scaling (unlike a database read replica, `DB23`). Follower fetching
exists for cross-rack locality, and its motivation is reducing network cost rather than adding read
capacity. That is a distinction interviewers sometimes probe.

### Interview questions

- "Why can't you have more consumers than partitions in a group?"
- "Kafka versus RabbitMQ — pick one for order events and one for sending emails, and justify."
- "What happens if you increase the partition count?"
- "Why is Kafka so fast?"

---

## Q13 · Kafka durability and delivery guarantees

`Expert` · Requires: `Q12`, `DB12` · Unlocks: `Q15`, `Q16`

### Preface

Kafka's durability is configurable, and the defaults are not the safest. The settings interact, so the
answer to "we must never lose an event" is a specific combination, not one flag.

You should be able to give that combination and say what it costs — because the cost is real:
latency, and unavailability when brokers are down.

### Details

#### 1. Replication and in-sync replicas

**Theory.** Each partition has `replication.factor` copies. The **ISR** (in-sync replica set) is the
subset currently caught up with the leader. A write is considered committed when all ISR members have
it.

**Example.** The durable configuration to memorise:

```
replication.factor = 3        (topic)
min.insync.replicas = 2       (topic)
acks = all                    (producer)
```

This means: three copies; a write must reach at least two before being acknowledged; the producer
waits for that. You survive one broker failure with no data loss. If two brokers are down, the
partition **stops accepting writes** — which is the correct behaviour and is the cost.

**Advanced.** `acks=all` with `min.insync.replicas=1` is a trap: `all` means "all in-sync replicas",
and if only the leader is in sync, that is one copy — you get the latency of waiting with none of the
safety. The two settings must be read together, and this is a favourite interview detail.

#### 2. Unclean leader election

**Theory.** If every in-sync replica is unavailable, Kafka can either wait (unavailable but
consistent) or promote an out-of-sync replica (available, and it **loses** the records the old leader
had that the new one does not).

**Example.** `unclean.leader.election.enable=false` is the default in modern Kafka and is correct for
anything important: prefer unavailability over silent data loss. Setting it true is a deliberate
availability-over-consistency choice for data where loss is acceptable — metrics, logs — and it is a
clean illustration of CAP as a per-topic configuration (`M11`).

**Advanced.** Related: the **leader epoch** prevents a returning old leader from serving stale data;
followers truncate their logs to the correct point on rejoining. This is fencing (`M33`) inside Kafka,
and it is worth naming because it shows the same distributed-systems primitive appearing again.

#### 3. Idempotent producers and transactions

**Theory.** `enable.idempotence=true` (the default in recent versions) gives each producer an id and
each record a sequence number, so the broker deduplicates retries — a retry caused by a lost
acknowledgement does not produce a duplicate. **Transactions** (`transactional.id`) let a producer
atomically write to several partitions and commit consumer offsets in the same transaction.

**Example.** Transactions enable exactly-once **read-process-write within Kafka**: consume from topic
A, produce to topic B, commit the offset — all atomically, so a crash cannot leave the output written
without the offset committed. Consumers set `isolation.level=read_committed` to skip aborted records.

**Advanced.** State the boundary precisely: this is exactly-once **inside Kafka only**. The moment the
processing writes to a database, calls an HTTP API or sends an email, the guarantee does not extend to
it — you need idempotency there (`M16`). Candidates who claim Kafka gives exactly-once end-to-end are
revealing a gap, and candidates who draw this boundary crisply stand out.

#### 4. Ordering and in-flight requests

**Theory.** With retries enabled and several requests in flight, a failed-and-retried batch can be
written **after** a later batch that succeeded — reordering records within a partition.

**Example.** With idempotence enabled, Kafka handles this: it tracks sequence numbers and preserves
order for up to five in-flight requests per connection. Without idempotence, you would need
`max.in.flight.requests.per.connection=1` to guarantee ordering, at a significant throughput cost.
This is a good example of a default (idempotence on) that quietly fixes a subtle correctness problem.

**Advanced.** The full producer configuration for "durable and ordered":
`acks=all`, `enable.idempotence=true`, `max.in.flight.requests.per.connection<=5`,
`retries=Integer.MAX_VALUE` with `delivery.timeout.ms` bounding the total, and
`min.insync.replicas=2` on the topic. Being able to list that set, and explain each, is a strong
answer to the standard "configure Kafka for no data loss" question.

### Interview questions

- "Give the exact producer and broker config for 'we must never lose an order event' and say what it
  costs."
- "`acks=all` with `min.insync.replicas=1` — what is wrong?"
- "Explain Kafka's exactly-once and where it stops working."
- "What is unclean leader election and when would you enable it?"

---

## Q14 · Partitioning, keys and ordering

`Advanced` · Requires: `Q12`, `M21`, `M32` · Unlocks: `Q15`, `Q17`, `SD06`

### Preface

The partition key is the most consequential decision in a Kafka design. It determines three things at
once: which records are ordered relative to each other, how much parallelism you can have, and whether
load is spread evenly.

Get it right and everything else is straightforward. Get it wrong and you have either lost ordering or
created a hot partition, and both are expensive to fix later.

### Details

#### 1. Key, partition, order

**Theory.** The producer computes `hash(key) % partition_count`. The same key always goes to the same
partition, and a partition is consumed in order by one consumer in a group. Therefore: **same key ⇒
same partition ⇒ ordered processing**.

**Example.** Keying `OrderPlaced`, `OrderPaid` and `OrderShipped` by `orderId` guarantees they are
processed in order for that order. Events for different orders are processed in parallel with no
ordering between them — which is exactly what the business needs and what scales.

**Advanced.** A **null key** means round-robin (or sticky batching in newer clients), so there is no
ordering at all. If events arrive out of order, the first thing to check is whether the key is set,
whether it is the same for all events of that entity, and whether some producer path forgets it. That
is the most common cause by far.

#### 2. Choosing the key

**Theory.** The key should be the entity whose events must be ordered, and should have high enough
cardinality to spread evenly across partitions.

**Example.** Good keys: `orderId`, `userId`, `deviceId`, `accountId`. Bad keys: `country` (a handful
of values, so most partitions are empty and a few are overloaded), `eventType` (same problem),
`timestamp` (all current traffic to one partition), and a constant (everything in one partition).

**Advanced.** Note the tension with multi-tenancy: keying by `tenantId` gives per-tenant ordering and
guarantees a hot partition when one tenant is much larger than the others (`M29`). Keying by a finer
entity (`orderId`) spreads better and loses cross-order ordering per tenant — which is usually
acceptable. Ask what ordering the business actually requires; it is almost always narrower than the
first answer given.

#### 3. Hot partitions

**Theory.** Skew means one partition receives far more traffic than the others. Its consumer becomes
the bottleneck, lag grows on that partition only, and adding consumers does not help because a
partition cannot be split.

**Example.** Mitigations: **composite keys** — `tenantId:orderId` spreads a large tenant's traffic
while keeping per-order ordering; **salting** — append a small random suffix, spreading load and
losing ordering, with re-aggregation downstream if needed; or a **dedicated topic** for the largest
tenants, consumed by their own consumers.

**Advanced.** Diagnose it with per-partition lag (`Q15`), not aggregate lag — aggregate lag looks
mildly elevated while one partition is hours behind. Per-partition lag and per-partition throughput
should be on the dashboard for any serious Kafka deployment, and asking for them is a good sign in an
interview.

#### 4. Ordering beyond one partition

**Theory.** If you genuinely need a global order, you need a single partition, which means a single
consumer and no scaling. That is almost never the right requirement.

**Example.** The alternatives: narrow the ordering requirement to per-entity (usually sufficient);
include a **version or sequence number** in the event and have the consumer ignore anything older
than what it has applied (`M16`), which makes ordering unnecessary; or use timestamps with a
watermark in a stream processor and handle late data explicitly (`Q17`).

**Advanced.** Ordering is also broken by parallelism **inside** the consumer: consuming a partition in
order and then handing each record to a thread pool destroys it. If you parallelise within a consumer,
partition the work by key again so each key is handled sequentially (`C10`'s single-writer principle
applied inside the process). This is a genuinely common bug and a good thing to raise.

### Interview questions

- "Events for one order arrive out of order. Diagnose."
- "One tenant produces 80% of the traffic. Fix the hot partition."
- "Do you need a global order? What would it cost?"
- "You parallelised your consumer for throughput. What did you break?"

---

## Q15 · Consumer semantics and operations

`Advanced` · Requires: `Q12`, `Q13`, `Q14`, `M30`, `C16` · Unlocks: `Q16`, `Q17`

### Preface

Most Kafka production problems are consumer problems: rebalance storms, growing lag, duplicates after
a crash, and a stuck consumer nobody notices.

Almost all of them trace back to one thing — **processing taking longer than the consumer's
configured allowance** — and the remedies follow from understanding the poll loop.

### Details

#### 1. The poll loop

**Theory.** A consumer repeatedly calls `poll()`, which returns a batch of records and also serves as
its **liveness signal** to the group coordinator. If the consumer does not call `poll()` again within
`max.poll.interval.ms` (five minutes by default), the coordinator assumes it has died and triggers a
rebalance.

**Example.** The classic failure: `max.poll.records=500` and each record takes one second to process.
The consumer takes 500 seconds between polls, exceeds the five-minute interval, is removed from the
group, a rebalance occurs, its partitions are reassigned, the messages are redelivered to another
consumer — which also takes too long. The group rebalances continuously and throughput collapses.

**Advanced.** The fixes: reduce `max.poll.records` so each batch fits comfortably in the interval;
raise `max.poll.interval.ms` if processing is genuinely slow; or move the slow work off the poll
thread and manage offsets manually. The first is usually right. Also note that heartbeats are sent by
a **background thread** (since Kafka 0.10.1), so `session.timeout.ms` and `max.poll.interval.ms` are
separate concerns — a consumer can be heartbeating happily while stuck processing.

#### 2. Offset commits

**Theory.** Committing after processing gives at-least-once (a crash before committing means
redelivery). Committing before gives at-most-once (a crash after committing means loss). Auto-commit
commits on a timer and gives you neither guarantee cleanly.

**Example.** The correct pattern for at-least-once with manual commits:

```
records = poll()
for record in records: process(record)     // must be idempotent
commitSync()                               // commit the whole batch after processing
```

Use `commitAsync()` in the loop for throughput with a final `commitSync()` on shutdown, and accept
that an async commit may fail silently — so log failures.

**Advanced.** Committing offsets **and** writing results in one atomic step is only possible either
within Kafka (transactions, `Q13`) or by storing the offset in the **same database** as the results,
in the same transaction, and seeking to it on startup. The latter is a legitimate and
under-used pattern for consumers whose output is a database — it gives genuine exactly-once effect
without Kafka transactions.

#### 3. Rebalancing

**Theory.** When a consumer joins or leaves, partitions are reassigned. The classic ("eager") protocol
**stops all consumers**, revokes everything and reassigns — a stop-the-world pause.

**Example.** Improvements worth knowing: **cooperative sticky** assignment (the default in recent
versions) revokes only the partitions that must move, so most consumers keep working. **Static
membership** (`group.instance.id`) means a consumer restarting within `session.timeout.ms` keeps its
partitions, so a rolling deploy does not trigger a rebalance per pod — a significant operational
improvement for stateful consumers.

**Advanced.** Rebalances cause duplicates: partitions are reassigned with the last committed offset,
so anything processed but uncommitted is reprocessed. This is a further reason idempotency is not
optional (`M16`), and it explains why a rebalance storm produces a flood of duplicate side effects,
which is often the visible symptom before anyone looks at the lag graph.

#### 4. Lag, and what to monitor

**Theory.** **Consumer lag** is the difference between the latest offset and the consumer's committed
offset — how far behind you are. It is the single most important streaming metric.

**Example.** Monitor: lag **per partition** (not just aggregate, `Q14`); the **time** equivalent of
lag (how old is the oldest unprocessed record — more meaningful than a count); processing time per
record; rebalance frequency; and dead-letter rate (`Q16`).

**Advanced.** Alert on **lag in time, and on its trend**, not on an absolute count. A lag of 100,000
is fine if you drain it in 30 seconds; a lag of 500 is an emergency if the consumer is stuck and the
oldest record is an hour old. And a lag that is **growing steadily** means arrival rate exceeds
processing rate — you will not recover without adding capacity or shedding load (`M30`). That
distinction between a large backlog and a growing one is the operationally useful one.

### Interview questions

- "Your consumer processes each message in 5 seconds and the group rebalances constantly. Fix."
- "How do you monitor a streaming pipeline?"
- "Where exactly do duplicates come from in Kafka?"
- "How do you get exactly-once effect when the output is a database?"

---

## Q16 · Retries, dead-letter queues and replay

`Advanced` · Requires: `Q11`, `Q13`, `Q15`, `M09`, `M16` · Unlocks: `Q21`, `SD10`

### Preface

Messages fail. The design question is how to distinguish a **transient** failure (retry) from a
**poison** message (give up) from a **downstream outage** (stop and wait), because treating them the
same is how a pipeline either stalls or discards data.

Kafka makes this harder than a traditional broker, because you cannot reject a single message without
blocking its partition.

### Details

#### 1. Classify the failure

**Theory.** Three categories, three responses:
- **Transient** — a timeout, a 503, a deadlock. Retry with backoff (`M09`).
- **Poison** — malformed data, a validation failure, a missing referenced record. Retrying will never
  work; send it to a dead-letter destination immediately.
- **Downstream outage** — the dependency is down for everything. Do not burn retries per message;
  pause the consumer or open a circuit breaker (`M10`) and resume when it recovers.

**Example.** The difference matters: retrying a poison message a hundred times wastes capacity and
delays everything behind it; dead-lettering a transient failure discards work that would have
succeeded on the second attempt; and retrying every message individually during a full outage turns a
dependency failure into a backlog you cannot drain.

**Advanced.** In code, this means mapping errors to a decision rather than catching broadly. A
`DeserializationError` is always poison; a `ConnectionTimeout` is transient; a 4xx from a downstream
is poison and a 5xx is transient. Making that mapping explicit and testable is the practical
implementation of this node.

#### 2. Retry topics in Kafka

**Theory.** You cannot nack one record without stalling its partition, so retries are implemented by
**republishing** the failed record to a separate retry topic with a delay, and moving on.

**Example.** The standard ladder: main topic → on failure, publish to `orders.retry.5s` → a consumer
of that topic waits until the record's timestamp plus five seconds, then republishes to the main topic
→ on repeated failure, escalate to `orders.retry.1m`, then `orders.retry.10m`, then
`orders.DLT`. Carry the attempt count in a header.

**Advanced.** The alternative is **pausing the partition** and retrying in place, which preserves
ordering (important for entity-keyed streams) at the cost of blocking everything behind it. Choose by
whether ordering matters more than throughput: for an entity's state changes, pause; for independent
work items, retry topics. Spring Kafka's `@RetryableTopic` and the non-blocking retry pattern
implement the second. Being able to state the trade is the senior answer.

#### 3. The dead-letter queue

**Theory.** A destination for messages that cannot be processed. It must contain enough to diagnose
and reprocess: the original payload, the original headers and key, the error, the stack trace, the
attempt count, and the timestamp.

**Example.** A dead-letter queue needs three things or it is a bin: an **owner**, an **alert** (any
message arriving is an incident, or at minimum a ticket), and a **documented replay procedure**.
Without them, teams discover 40,000 messages six months later and cannot safely do anything with
them.

**Advanced.** Replaying safely requires: consumers to be idempotent (`M16`), because some of those
messages may have been partially processed; rate limiting the replay, so you do not overwhelm the
downstream that was the original cause; and the ability to replay a **subset** (by time range, by
error type, by key) rather than everything. Build the replay tooling when you build the DLQ, not
during the incident.

#### 4. Preventing infinite loops

**Theory.** A message that fails, is retried, fails, and is republished forever consumes capacity
indefinitely and can crowd out real work.

**Example.** Bound it: a maximum attempt count in a header, checked before republishing; a maximum
age, after which the message is dead-lettered regardless; and a check that the retry topic's consumer
cannot republish to itself without incrementing the counter. Also guard against a **poison pill** that
crashes the consumer on deserialisation — use an error-handling deserialiser so the record goes to the
DLQ instead of killing the consumer, which otherwise restarts, re-reads the same record and crashes
again in a loop.

**Advanced.** The deserialisation poison pill is worth naming specifically: it is one of the few
failures that takes down a consumer group completely and looks like a crash-loop rather than a data
problem. `ErrorHandlingDeserializer` in Spring Kafka, or an explicit try/catch around deserialisation,
is the fix, and knowing it exists prevents a very confusing outage.

### Interview questions

- "Design retry and DLQ for a Kafka consumer that calls a flaky third-party API."
- "Your DLQ has 40,000 messages from an outage. How do you replay them safely?"
- "How do you retry in Kafka without blocking the partition, and what do you lose?"
- "A malformed message crashes your consumer on startup, every time. What is happening?"
