[← back to the field index](README.md)

# Microservices · Part 2 — Failure, Resilience & Consistency

Nodes `M08`–`M16`. This is the heart of the field. If you learn only one part, learn this one.

---

## M08 · Failure modes and the fallacies of distributed computing

`Intermediate` · Requires: `M03` · Unlocks: `M09`, `M11`, `M20`, `M21`, `M30`

### Preface

Inside one program, a function call either runs or throws. Across a network, there is a third
outcome: **you do not know**. The request may have been lost, or it may have arrived and done the
work while the reply was lost.

This uncertainty — called partial failure — is the single fact that makes distributed systems
hard. Everything else in this field is a response to it.

### Details

#### 1. The eight fallacies

**Theory.** A famous list of assumptions that feel true and are not: the network is reliable;
latency is zero; bandwidth is infinite; the network is secure; topology does not change; there is
one administrator; transport cost is zero; the network is homogeneous.

**Example.** Each fallacy maps to a real bug. "Latency is zero" becomes a loop calling a service
once per item — fine with 10 items in testing, 40 seconds with 2,000 in production. "The network is
reliable" becomes code with no timeout, which hangs forever when a peer disappears without closing
the connection. "Topology does not change" becomes a cached IP address that survives a failover.

**Advanced.** The subtlest is "bandwidth is infinite". A service returning 2MB of JSON per request
at 500 requests per second is pushing 1GB/s; you will hit network limits, and in cloud
environments cross-zone traffic is also billed. Payload size is a scaling and cost concern, not
only a speed concern.

#### 2. Partial failure and the unknown outcome

**Theory.** When a call times out, three things may have happened: the request never arrived; it
arrived and failed; it arrived and succeeded but the response was lost. The caller cannot tell
these apart. Therefore the only safe design is to make the operation **safe to repeat** and to
have a way to **find out what really happened**.

**Example.** Checkout calls the payment provider and times out after 10 seconds. Was the customer
charged? Blindly retrying risks a double charge; not retrying risks losing a valid order. The
correct handling: send an idempotency key with the original request so a retry is recognised as the
same operation (see `A09`); and on timeout, query the provider for the status of that key rather
than guessing.

**Advanced.** This is why reconciliation jobs exist in every serious payment system. Whatever your
real-time logic, a background job periodically compares your records with the provider's and
resolves differences. Interviewers are impressed by candidates who mention reconciliation
unprompted, because it shows you have operated a system rather than only designed one.

#### 3. The taxonomy of failure

**Theory.** Useful categories: **crash-stop** (a process dies and stays dead); **crash-recovery**
(it dies and comes back, possibly with stale state); **omission** (messages are dropped);
**timing** (things are late enough to be effectively wrong); **Byzantine** (a component behaves
arbitrarily or maliciously — relevant to blockchains, rarely to internal systems).

**Example.** A crash-recovery failure that bites in practice: a consumer reads a message, crashes
before committing its offset, restarts, and reprocesses the message. Nothing was lost; the message
was simply processed twice. Your consumer must tolerate that (see `M16`).

**Advanced.** Timing failures are the ones people design for least. A node that is not dead but
responds in 30 seconds is worse than a dead one: dead nodes are removed from load balancers, slow
nodes keep accepting work and hold your connections and threads. This is the argument for treating
"too slow" as "failed" — hard timeouts, and ejecting slow instances as if they had crashed.

#### 4. Grey failure

**Theory.** A grey failure is a partial, hard-to-detect failure: the component looks healthy from
outside but is failing some work. Health checks pass, dashboards look fine, and users complain.

**Example.** A pod whose disk is full: it still answers `/health` with 200, because that endpoint
only returns a constant, but every write fails. Or one broker in a cluster with a failing network
card that drops 5% of packets — throughput is fine on average, but tail latency is terrible and
some requests fail for reasons no single service's logs explain.

**Advanced.** Detecting grey failure needs signals from the *caller's* perspective, not the
server's: client-observed error rates and latency per instance, which is why outlier detection at
the load balancer and per-instance metrics matter. The general principle: measure what users
experience, not what components report about themselves.

#### 5. Cascading failure

**Theory.** One slow component makes its callers slow; the callers' threads or connections fill up;
they become unresponsive; their callers follow. A small problem becomes a total outage. Retries
make it worse by multiplying load exactly when the system is weakest.

**Example.** The recommendation service gets slow. The product page calls it without a timeout, so
each request holds a connection for 30 seconds. The pool fills. Now the product page cannot serve
*any* request, including the 90% that do not need recommendations. The whole site is down because
of an optional feature.

**Advanced.** Recovery from cascading failure is often harder than prevention, because when you
restart everything, the accumulated backlog of retries and queued work immediately overwhelms the
system again — a "thundering herd on recovery". Real recovery usually requires shedding load first
(reject most traffic), letting the system stabilise, then ramping back up. Prevention is timeouts,
bulkheads, circuit breakers and load shedding (see `M10`).

### Interview questions

- "You got a timeout from a payment service. Did the payment happen?" — unknowable; explain
  idempotency keys, status queries and reconciliation.
- "What is a grey failure and why do health checks miss it?"
- "Walk me through how one slow dependency takes down an entire service."
- "Why is a slow node worse than a dead node?"

---

## M09 · Timeouts, retries, backoff and idempotency

`Intermediate` · Requires: `M08` · Unlocks: `M10`, `M16`, `SD10`, `Q16`

### Preface

These four things are the basic toolkit for surviving an unreliable network, and they must be used
together. A timeout without a retry turns a blip into an error. A retry without backoff turns a
blip into an outage. A retry without idempotency turns a blip into duplicate charges.

### Details

#### 1. Every remote call needs a timeout

**Theory.** Default timeouts in HTTP clients are often infinite or measured in minutes. An
unbounded wait means a resource — a connection, a thread, an event loop task — is held
indefinitely, and under failure your capacity drains away. A timeout must be shorter than your own
caller's deadline, or you produce work nobody is waiting for.

**Example.** Concrete settings for a Node client: connection timeout 1s, total request timeout
2s, when the user-facing budget is 3s. Note the layers — DNS resolution, TCP connect, TLS
handshake, and response body reading each need bounding, and a "request timeout" in some libraries
does not cover the body stream.

**Advanced.** Choose timeouts from measured latency, not from taste: a good starting point is
around the p99.9 of the healthy call, so normal slowness is not cut off but genuine hangs are. Too
short is its own outage — a timeout below the real p99 fails good requests and adds retry load.
Consider also that a timeout does not stop the server working; unless you propagate cancellation
(for example `AbortSignal` in Node, context cancellation in Go, gRPC deadlines), the downstream
keeps burning capacity.

#### 2. Retries, and when not to retry

**Theory.** Retry only when the operation is safe to repeat and the failure looks transient.
Retry on: connection failures, timeouts on idempotent operations, 429, 502, 503, 504. Do not retry
on: 400, 401, 403, 404, 422 — they will fail identically. Never retry a non-idempotent write
without an idempotency key.

**Example.** `GET /orders/42` — retry freely. `POST /payments` — retry only with an
`Idempotency-Key` header so the provider recognises the repeat. `DELETE /orders/42` — safe to
retry; the second call returning 404 is a success, not an error.

**Advanced.** **Retry amplification** is the trap. If each of three layers retries three times,
one user request can become 27 backend calls. Under partial failure this multiplies load precisely
when capacity is lowest. Two rules: retry at **one** layer only, usually the one closest to the
failure that knows the operation is safe; and use a **retry budget** — a limit such as "retries may
not exceed 10% of total requests", enforced globally, so retries stop automatically during a wide
outage.

#### 3. Backoff and jitter

**Theory.** Retrying immediately hammers a struggling service. Exponential backoff waits longer
after each attempt (100ms, 200ms, 400ms…). But if a thousand clients fail at the same moment, they
all retry at the same moments — synchronised waves. **Jitter** adds randomness so retries spread
out.

**Example.** Full jitter, the version AWS recommends and the one to quote:

```js
const delay = Math.random() * Math.min(capMs, baseMs * 2 ** attempt);
```

Not `base * 2^attempt + random(0, 100)` — that is "equal jitter" and still leaves visible waves.
Full jitter picks uniformly from the whole interval and spreads load best.

**Advanced.** The pattern that jitter fixes is a **retry storm** or thundering herd: a service
restarts, every client reconnects at once, the service falls over again, and the cycle repeats. It
appears in reconnect logic, cache expiry (see `Q04`) and scheduled jobs as well as retries. Any
time many clients act on the same clock, add jitter. For scheduled work, that means not running
every tenant's job at exactly midnight.

#### 4. Idempotency as the enabler

**Theory.** An operation is idempotent if doing it twice has the same effect as doing it once. Once
operations are idempotent, retries are safe, at-least-once delivery is acceptable, and most
distributed complexity becomes manageable.

**Example.** Making a create endpoint idempotent: the client sends a unique key; the server stores
the key with the result of the first execution; a repeat with the same key returns the stored
result instead of doing the work again. The insert of the key must be atomic — a unique index on
the key column, catching the duplicate-key error — or two concurrent retries both pass the "does it
exist" check. Full algorithm in `A09`.

**Advanced.** Natural idempotency is better than bolted-on idempotency where you can get it. "Set
status to SHIPPED" is naturally idempotent; "add 1 to attempts" is not. Designing operations as
*declarations of desired state* rather than *increments* removes the problem. Where you cannot,
carry a version or sequence number and reject out-of-order or repeated updates.

#### 5. Hedged requests

**Theory.** For read-only calls where tail latency matters, send a second request to another
replica if the first has not answered within, say, the p95, and use whichever returns first. This
trades a little extra load for a much better p99.

**Example.** Google's "The Tail at Scale" describes this for search. A practical version: if the
call has not answered in 50ms, fire a second one; typical extra load is around 5%, and p99
improves dramatically.

**Advanced.** Only hedge idempotent, side-effect-free calls, and always cancel the loser to avoid
doubling downstream work. Hedging is dangerous under overload — it adds load exactly when the
system is struggling — so it should be disabled automatically when error rates rise, or bounded by
the same retry budget.

### Interview questions

- "Where does exponential backoff alone still take the system down?" — synchronised retries;
  jitter is required. And retry amplification across layers; budgets are required.
- "Design idempotency for `POST /payments`."
- "Which HTTP status codes do you retry and which do you not?"
- "Your service calls three downstreams and one is slow. Stop the cascade."

---

## M10 · Resilience patterns: circuit breaker, bulkhead, load shedding

`Advanced` · Requires: `M07`, `M09` · Unlocks: `M26`, `M30`, `M31`, `SD07`

### Preface

Timeouts and retries handle *individual* failures. These patterns handle *sustained* failure: what
to do when a dependency is not having a blip but is genuinely down for ten minutes.

The common theme is **containment**: stop a broken dependency from consuming your resources, and
stop your own overload from spreading.

### Details

#### 1. Circuit breaker

**Theory.** Wrap calls to a dependency in a state machine. **Closed**: calls pass through, failures
are counted. When failures exceed a threshold, move to **Open**: fail immediately without calling,
for a cooldown period. After the cooldown, move to **Half-open**: allow a few trial calls; if they
succeed, close; if not, open again. The value is twofold — you stop wasting your own resources on
calls that will fail, and you stop hammering a service that is trying to recover.

**Example.** Configuration that behaves sensibly: open when more than 50% of the last 20 calls fail
*and* there were at least 20 calls in the window; stay open 30 seconds; allow 3 trial calls in
half-open. When open, return a fallback — cached recommendations, an empty list, or a clear error.

**Advanced.** The minimum-calls condition matters. Without it, a dependency receiving two requests a
minute opens its breaker after a single failure and a coincidence becomes an outage. Two more
subtleties: breaker state is usually **per process**, so with 50 pods you have 50 independent
breakers, which is generally desirable (each reacts to what it sees) but makes behaviour look
inconsistent in metrics; and the breaker must be **per dependency**, not global, or one bad
downstream blocks calls to healthy ones.

#### 2. Bulkhead

**Theory.** Named after a ship's watertight compartments: give each dependency its own limited pool
of resources, so exhausting one cannot sink the whole service.

**Example.** Your service calls Payments and Recommendations. Without bulkheads, both use the same
HTTP agent and the same worker capacity, so a slow Recommendations fills everything and Payments
calls cannot get through. With bulkheads: a semaphore allowing at most 10 concurrent calls to
Recommendations and 50 to Payments. When Recommendations hangs, the 11th call fails instantly
instead of waiting, and Payments is unaffected.

**Advanced.** In Node, "thread pool exhaustion" is not the failure mode — the event loop is. The
equivalent bulkhead is a concurrency limiter (a counting semaphore, or `p-limit`) plus a hard cap
on pending promises, because unlimited in-flight requests consume memory and event loop time. In
the JVM, use separate thread pools or a semaphore-based limiter. The database connection pool is
also a shared resource worth partitioning: a slow report query can starve every request handler,
which is why long-running queries deserve their own pool.

#### 3. Load shedding and admission control

**Theory.** When demand exceeds capacity, you cannot serve everything. Your choice is between
degrading for everyone (queues grow, latency rises, eventually everything times out and *no*
request succeeds) or rejecting some requests quickly so the rest succeed. Rejecting is better, and
it should be explicit.

**Example.** Measure a saturation signal — event loop delay, queue depth, or concurrent in-flight
requests. Above a threshold, return 503 with `Retry-After` immediately, cheaply, before doing any
work. Prioritise: shed health-check-like or low-value traffic (analytics, prefetch) before
checkout; shed anonymous before logged-in; shed retries before first attempts (you can tell, if
clients mark them).

**Advanced.** The key insight is that **queueing is not free capacity**. A request waiting 30
seconds in a queue for a client that timed out at 3 seconds is pure waste; you did the work and
nobody wanted it. This is why bounded queues plus fast rejection beat large queues (see `M30`).
The most advanced version is adaptive: derive the concurrency limit at runtime from observed
latency, as TCP congestion control does — Netflix's `concurrency-limits` library is the canonical
implementation.

#### 4. Fallbacks and graceful degradation

**Theory.** Decide in advance what a failed dependency means for the user. Options: serve stale
cached data; serve a generic default; omit the feature; queue the work for later; or fail the
request if the data is essential.

**Example.** Product page degradation ladder: recommendations fail → show "popular items" from a
static cache; reviews fail → hide the reviews section; price service fails → **fail the request**,
because showing a wrong price is worse than an error. The point is that the answer differs per
dependency, and somebody must decide it deliberately.

**Advanced.** Stale data as a fallback needs a policy: how stale is acceptable, and do you tell the
user. Serving a six-hour-old price is a business problem, not a technical one. A related pattern is
**brownout** — the service deliberately disables expensive optional features under load (turning
off personalisation, reducing image quality) to protect the core path. Feature flags make this
operable during an incident, which is why kill switches for every non-essential dependency are a
good investment.

### Interview questions

- "The recommendation service is slow and now the whole API times out. Diagnose and fix."
- "Circuit breaker versus retry versus rate limit — when is each the wrong tool?"
- "Why does a circuit breaker need a minimum request count?"
- "Under overload, why is rejecting requests better than queueing them?"

---

## M11 · CAP and PACELC

`Intermediate` · Requires: `M08` · Unlocks: `M12`, `M20`, `M33`

### Preface

CAP is the most quoted and most misquoted idea in distributed systems. It says: when the network
splits so that two parts of your system cannot talk, you must choose between **staying available**
(answer with possibly stale data) and **staying consistent** (refuse to answer).

It does *not* say "pick two of three". Partition tolerance is not optional — networks do split —
so the real choice is only what to do during a split.

### Details

#### 1. What CAP really claims

**Theory.** C is linearizability: every read sees the most recent write, as if there were one copy.
A is availability: every non-failing node answers every request. P is partition tolerance: the
system keeps working when messages between nodes are lost. The theorem: during a partition you
cannot have both C and A.

**Example.** Two data centres, replicating, and the link between them drops. A write arrives in
data centre A. Either A accepts it (available; data centre B now serves stale reads — you chose
availability) or A refuses until it can reach B (consistent; users see errors — you chose
consistency). There is no third option, because A cannot know whether B is dead or just
unreachable.

**Advanced.** The common mistake is applying CAP to a single-node database. A single Postgres has
no partition to tolerate, so CAP says nothing about it; it becomes relevant when you add replicas
and failover. The second common mistake is treating the choice as system-wide. Real systems choose
per operation: the same database might serve stale reads from a replica (available) while requiring
quorum for writes (consistent).

#### 2. PACELC: the part that matters every day

**Theory.** PACELC extends CAP: **if** there is a **P**artition, choose **A** or **C**; **E**lse
(normal operation) choose between **L**atency and **C**onsistency. The second half describes your
system 99.99% of the time, which makes it more practically useful than CAP.

**Example.** Synchronous replication: a write is not acknowledged until a replica confirms it. That
is consistency, paid for with latency on every write — potentially tens of milliseconds across
regions. Asynchronous replication acknowledges immediately and replicates in the background: lower
latency, but a failover can lose recent writes, and replicas serve stale reads.

**Advanced.** Quote the classification when asked: DynamoDB and Cassandra are PA/EL by default
(available and low latency, eventually consistent) but offer stronger modes per request.
Traditional single-leader SQL with synchronous replication is PC/EC. Spanner is PC/EC and pays for
consistency with commit-wait latency bounded by TrueTime. The ability to say "this is a per-request
knob, not a database property" is a senior-level signal.

#### 3. Choosing per feature

**Theory.** The right unit of decision is the user-visible operation. Ask: if this data is a few
seconds stale, does anything bad happen? If yes, pay for consistency; if no, take availability and
speed.

**Example.** In one e-commerce system: product views — stale is fine (availability). Adding to
cart — stale is fine. **Decrementing stock at checkout** — must be consistent, or you sell the last
item twice. Account balance display — stale by seconds is acceptable. **Money transfer** — must be
consistent. Notice that the strict requirements are few and localised; that is typical, and it is
what lets most of a system be fast and available.

**Advanced.** When you need strong consistency in only one narrow place, keep it there rather than
raising the whole system's guarantees. Concretely: keep stock counts in a strongly consistent store
with a proper transaction, and let the catalogue, search index and product page be eventually
consistent caches of it. This "strong core, eventual edge" shape appears in almost every large
system.

### Interview questions

- "Is Postgres CP or AP?" — a single node makes the question ill-formed; with replication and
  failover it becomes a configuration choice. Discuss synchronous versus asynchronous replication
  and split-brain.
- "Name a feature where you would choose availability over consistency, and one where you would
  not."
- "What does the E in PACELC mean and why does it matter more day to day?"

---

## M12 · Consistency models

`Advanced` · Requires: `M11` · Unlocks: `M14`, `M19`, `M23`, `SD08`, `DB23`

### Preface

"Eventually consistent" is not one thing. There is a ladder of guarantees between "always the
latest value" and "some value, eventually". Knowing the rungs lets you pick the weakest guarantee
that still gives users a sane experience — which is usually the fastest and most available option.

The rung that matters most in practice is **read-your-writes**: a user must see their own change
immediately, even if others see it a second later.

### Details

#### 1. The ladder, strongest first

**Theory.**
- **Linearizable** — there is one global order and a read always returns the latest committed
  write. Behaves like a single copy. Most expensive.
- **Sequential** — all nodes see operations in the same order, but that order may lag real time.
- **Causal** — operations that are causally related (B was written after reading A) appear in that
  order everywhere; unrelated operations may appear in any order.
- **Read-your-writes / monotonic reads** — session guarantees: you see your own writes; you never
  see time go backwards.
- **Eventual** — if writes stop, all replicas converge. Says nothing about what you see meanwhile.

**Example.** Eventual consistency without session guarantees produces the classic bug: a user edits
their display name, the write goes to the primary, the page reloads and reads a replica that has
not caught up, and the old name appears. The user believes the save failed and does it again.

**Advanced.** Monotonic reads matter as much as read-your-writes and are forgotten more often. If
successive reads hit different replicas with different lag, a user can see a comment appear, then
disappear, then reappear. Fixing it usually means pinning a user's session to one replica, or
carrying a version marker (an LSN or timestamp) and requiring a replica at least that fresh.

#### 2. Achieving read-your-writes in practice

**Theory.** Three standard techniques: route reads to the primary for a short window after a write;
pin the session to the replica that received the write; or have the client carry a version token
and route to a replica caught up to it.

**Example.** Practical implementation: after a write, set a cookie or cache entry
`lastWriteAt = now` for that user; for the next 5 seconds, that user's reads go to the primary.
Simple, effective, and it only sends a small fraction of reads to the primary. In Postgres you can
be more precise: capture the write's LSN and require a replica whose replay position has passed it.

**Advanced.** An often better answer is to avoid the read entirely: have the write endpoint
**return the updated entity**, so the UI updates from the response rather than re-fetching. This is
free, exact, and removes the problem rather than mitigating it. Interviewers like this answer
because it shows you look for the simplest fix first.

#### 3. Convergence, last-write-wins and CRDTs

**Theory.** When two replicas accept conflicting writes, something must decide the winner.
**Last-write-wins** uses timestamps — simple, and it silently discards data. **CRDTs** (conflict-free
replicated data types) are structures designed so concurrent updates merge deterministically
without losing information: counters that only grow, sets with add/remove tracking, sequences for
collaborative text.

**Example.** Two devices edit a user's settings offline. With last-write-wins on the whole object,
the device that syncs second overwrites the other's changes entirely — one is lost. With per-field
merge or a CRDT map, both changes survive because they touched different fields.

**Advanced.** Last-write-wins is also unsafe because it depends on clocks (see `M21`): a node with a
clock 30 seconds fast wins every conflict, including ones it should lose. If you use LWW, use a
logical clock or a server-assigned sequence, not wall time. CRDTs are the right answer for
collaborative editing and offline-first apps, and overkill for ordinary business data — say so
rather than proposing them everywhere.

#### 4. Making eventual consistency acceptable to users

**Theory.** The user experience question is usually more important than the technical one. Options:
optimistic UI (show the change immediately, reconcile later), explicit pending states, blocking
only the specific view that needs freshness, or returning the authoritative result from the write.

**Example.** Posting a comment: show it immediately in the local list marked as "sending", replace
it with the server's version when confirmed, and show an error with a retry if it fails. The
underlying system is eventually consistent; the user never notices.

**Advanced.** The failure case to plan for is when the optimistic assumption turns out wrong — the
write is rejected after the UI showed success. You need a visible, non-destructive way to undo the
optimistic state. Systems that skip this produce the worst kind of bug: the user believes something
happened that did not, and nothing on screen says otherwise.

### Interview questions

- "A user updates their profile, reloads, and sees the old value. Three fixes?"
- "What is causal consistency and when is it enough?"
- "Why is last-write-wins dangerous?"
- "Explain read-your-writes and monotonic reads, and give a bug caused by each being absent."

---

## M13 · Distributed transactions: two-phase commit

`Advanced` · Requires: `M12`, `DB06` · Unlocks: `M14`, `M20`

### Preface

The obvious way to keep several databases consistent is a transaction across all of them. That is
what two-phase commit (2PC) does, and it works — it is just so costly in availability that most
systems avoid it.

Knowing *why* it is avoided is what makes the saga pattern (`M14`) make sense.

### Details

#### 1. How 2PC works

**Theory.** A coordinator drives two phases. **Prepare**: ask every participant "can you commit?"
Each does the work, takes locks, writes it durably, and answers yes or no — promising it can commit
if asked. **Commit**: if all said yes, tell everyone to commit; if any said no, tell everyone to
abort. The promise in phase one is what makes the protocol correct.

**Example.** Order service and Inventory service, each with its own database, under an XA
transaction manager. Prepare: both write their changes and hold locks. Commit: both finalise. If
Inventory says no, both roll back, and the order never existed.

**Advanced.** The critical detail is that a prepared participant **must** hold its locks until it
is told what to do. It cannot decide alone, because it does not know what the others answered. This
is the source of every problem below.

#### 2. Why it is avoided: the blocking problem

**Theory.** If the coordinator crashes after participants have prepared, they are stuck. They hold
locks and cannot commit or abort. Those rows are unreadable and unwritable until the coordinator
returns. The coordinator is a single point of failure that can freeze several databases at once.

**Example.** The coordinator's host dies at the wrong moment. Inventory rows for a popular product
stay locked. Every checkout touching that product now blocks, then times out. The outage lasts
until someone restarts the coordinator or manually resolves the in-doubt transactions — an
operation most teams have never rehearsed.

**Advanced.** 3PC adds a phase to avoid blocking, but only under the assumption of a synchronous
network with bounded delays — an assumption real networks violate, so 3PC can produce inconsistent
outcomes during a partition. In practice, 2PC's blocking is mitigated by making the coordinator
itself replicated and consistent (via consensus, `M20`), which is what modern distributed databases
do internally. That is a much bigger commitment than "we added XA".

#### 3. The other costs

**Theory.** Availability multiplies: the transaction only succeeds if every participant is up.
Latency is at least two round trips plus durable writes at each step. Locks are held across the
network for that whole time, so throughput on contended rows collapses. And support is patchy —
many modern systems (Kafka, most NoSQL stores, most HTTP APIs) have no XA support at all.

**Example.** Five participants at 99.9% availability each: the distributed transaction succeeds
99.5% of the time at best, before considering the coordinator. And you cannot enrol a third-party
payment API in an XA transaction, which alone rules it out for most real workflows.

**Advanced.** The deeper architectural objection: a distributed transaction re-couples services
that you split apart precisely to decouple. If Order cannot commit without Inventory being
available, you have a distributed monolith with extra latency (see `M01`).

#### 4. When 2PC is actually fine

**Theory.** It is reasonable when participants are few, close, and under your control; the
transaction is short; volume is modest; and the alternative is genuinely worse.

**Example.** A single application writing to one database and one message broker, both in the same
data centre, using a transaction manager — historically common in Java/JEE. But note that the
better modern answer for exactly this case is the **transactional outbox** (`M15`): one local
transaction, no coordinator, no XA.

**Advanced.** 2PC is also alive and well *inside* distributed databases — Spanner, CockroachDB and
others use it across shards, combined with consensus to make the coordinator fault-tolerant and
with careful lock management. So the accurate statement is not "2PC is bad" but "2PC across
independently owned services, with a single coordinator, is a poor trade". Saying it that precisely
scores well.

### Interview questions

- "Why doesn't everyone just use 2PC across services?"
- "What exactly happens if the coordinator dies after prepare?"
- "When is 2PC actually fine?"
- "How do distributed SQL databases use 2PC without the blocking problem?"

---

## M14 · Sagas and compensation

`Advanced` · Requires: `M12`, `M13` · Unlocks: `M15`, `M16`, `SD10`

### Preface

If you cannot use one transaction across services, you break the work into a sequence of local
transactions, each in one service. If a later step fails, you run **compensating actions** to undo
the earlier ones.

That sequence is a saga. The important mental shift: there is no rollback. There is only "do
something that makes up for what we already did".

### Details

#### 1. The shape of a saga

**Theory.** A saga is a series of steps T1, T2, T3, each a local transaction, with compensations
C1, C2, C3. If T3 fails, run C2 then C1. Each step commits immediately, so intermediate states are
visible to the rest of the system — that is the price.

**Example.** Order placement:
| Step | Action | Compensation |
|---|---|---|
| T1 | Create order (status PENDING) | Mark order CANCELLED |
| T2 | Reserve stock | Release reservation |
| T3 | Charge payment | Refund payment |
| T4 | Mark order CONFIRMED | — (last step needs none) |

If T3 fails, run C2 (release stock) and C1 (cancel order). The customer sees a cancelled order,
which is a normal business outcome rather than a technical failure.

**Advanced.** Order the steps so that the most likely failure happens **early** and the
hardest-to-compensate step happens **last**. Charging a card is hard to undo cleanly (refunds cost
money and take days), so validate everything cheap and reversible first. This "cheap and reversible
first" ordering is a design principle worth stating explicitly in an interview.

#### 2. Compensation is not rollback

**Theory.** A database rollback erases history. A compensation is a new, visible business action.
Some effects cannot be undone at all: an email has been read, a physical parcel has left the
building, a third party has been told something.

**Example.** Compensating a sent confirmation email is not deleting it — it is sending a
cancellation email. Compensating a dispatched parcel is starting a return process. The
compensation belongs to the business domain, and a domain expert should define it, not the
engineer.

**Advanced.** Compensations must be **idempotent and must eventually succeed**. If a compensation
fails, you cannot compensate the compensation — you have nowhere to go. The standard handling:
retry forever with backoff, and after N attempts route it to a dead-letter queue with an alert so a
human resolves it. A saga with an unhandled failed compensation leaves money or stock in limbo, so
this path needs monitoring like any user-facing feature.

#### 3. Orchestration versus choreography

**Theory.** **Orchestration**: a central component (the saga orchestrator) holds the state machine
and tells each service what to do next. **Choreography**: no central component; each service reacts
to events and emits its own, and the flow emerges.

**Example.** Choreography: Order publishes `OrderCreated` → Inventory reserves and publishes
`StockReserved` → Payment charges and publishes `PaymentCompleted` → Order marks confirmed. Nobody
owns the flow. Orchestration: an `OrderSaga` component receives the request and issues commands
step by step, recording where it is after each one.

**Advanced.** Practical guidance: choreography for two or three steps with no branching, because it
is simple and adds no component. Orchestration once there is branching, timeouts, or more than
about four steps, because the alternative is a flow nobody can see. The killer argument for
orchestration is debuggability — a support engineer can ask "where is order 123 stuck?" and get an
answer from one table, rather than reconstructing it from five services' logs. Tools like Temporal
make the orchestrator durable so the flow survives restarts.

#### 4. What you lose: isolation

**Theory.** Sagas give you atomicity (eventually), consistency and durability, but **not
isolation**. Because each step commits immediately, other transactions can see and act on
intermediate state. Classic anomalies: **dirty reads** (someone reads state that will later be
compensated), **lost updates** (two sagas interleave and one overwrites the other).

**Example.** Order A reserves the last unit of stock, then its payment fails and stock is released.
Meanwhile Order B saw stock as zero and told the customer "out of stock". Nothing is corrupt, but a
sale was lost. Worse: a customer sees an order appear and then vanish.

**Advanced.** The countermeasures have names and are worth knowing: **semantic lock** — mark the
record with a pending state (`status = PENDING`) so other sagas know it is in flux and can wait or
refuse; **commutative updates** — design operations so order does not matter (add/subtract rather
than set); **pessimistic view** — reorder steps so the risky one happens when less damage is
possible; **re-read value / version check** — verify nothing changed before committing;
**by-value** — route high-value requests through a stricter (even 2PC) path and low-value ones
through the saga. Naming two of these in an interview marks you as having read the literature.

### Interview questions

- "Design order placement across Order, Payment and Inventory as a saga. What if the compensation
  itself fails?"
- "Orchestration or choreography for a seven-step branching flow? Why?"
- "What does a saga give up compared with a database transaction, and how do you manage it?"
- "How do you order the steps of a saga?"

---

## M15 · Transactional outbox, inbox and change data capture

`Advanced` · Requires: `M14`, `DB06`, `Q10` · Unlocks: `M16`, `M18`, `Q18`

### Preface

Almost every event-driven service needs to do two things together: write to its database and
publish an event. You cannot do both atomically — they are different systems. This is the
**dual-write problem**, and it silently corrupts data in a surprising number of production
systems.

The outbox pattern solves it by turning two writes into one.

### Details

#### 1. The dual-write problem

**Theory.** There are only two orderings, and both are broken. Write the database, then publish: if
the process dies in between, the data changed but nobody was told — a **lost event**. Publish, then
write: if the write fails, consumers act on something that never happened — a **phantom event**.
Wrapping them in a database transaction does not help, because the publish is not part of it.

**Example.** The code that looks fine and is not:

```js
await orderRepo.save(order);          // committed
await kafka.send('order.placed', ev); // process dies here
```

The order exists, the warehouse never hears about it, and nothing in your logs looks like an error.
These bugs surface as "a few orders a week never shipped" and are painful to trace.

**Advanced.** Retrying the publish in a `catch` block does not fix it either — the process can die
during the retry, and retrying forever inside a request handler blocks the response. The problem is
structural: you need the intent to publish to be stored durably in the same atomic unit as the
data.

#### 2. The outbox pattern

**Theory.** Add an `outbox` table in the same database. In the same transaction as your business
write, insert a row describing the event. A separate process reads unpublished rows, sends them to
the broker, and marks them sent. One atomic write, then an independent, retryable delivery step.

**Example.**

```sql
BEGIN;
INSERT INTO orders (id, status, total) VALUES ('o1', 'PENDING', 4200);
INSERT INTO outbox (id, aggregate_id, type, payload, created_at)
VALUES ('e1', 'o1', 'OrderPlaced', '{"orderId":"o1"}', now());
COMMIT;
```

A relay then does: select unsent rows ordered by id, publish each, mark sent. If it crashes after
publishing but before marking, the row is published twice — which is fine, because consumers are
idempotent (`M16`).

**Advanced.** Practical details that matter: order by a monotonic id and publish per aggregate in
order, or you deliver `OrderShipped` before `OrderPlaced`; use `FOR UPDATE SKIP LOCKED` so several
relay instances can work in parallel without duplicating (see `DB09`); delete or archive sent rows,
or the table grows without bound and slows the poll query; and set the message key from
`aggregate_id` so per-entity ordering survives in Kafka (see `Q14`).

#### 3. Polling relay versus change data capture

**Theory.** The relay can **poll** the outbox table on a short interval, or you can use **change
data capture (CDC)** — a tool such as Debezium reads the database's replication log (the Postgres
WAL or MySQL binlog) and emits each committed change as a message, with no polling at all.

**Example.** Polling: simple, no extra infrastructure, latency equals the poll interval (say
200ms), and adds constant query load. CDC: latency in milliseconds, no query load, but you now
operate Debezium and Kafka Connect, manage replication slots (a stalled slot makes the database
retain WAL until the disk fills — a real outage mode), and handle schema changes.

**Advanced.** A middle option is to use CDC directly on the business tables and skip the outbox
table entirely, deriving events from row changes. It works, but the events then mirror your
internal schema, which couples consumers to your table structure — exactly what the outbox avoids
by letting you write a proper domain event. Prefer outbox-plus-CDC: CDC for reliable, low-latency
delivery; the outbox table for a clean, stable event contract.

#### 4. The inbox pattern

**Theory.** The mirror image on the consumer side. Before processing a message, record its id in an
`inbox` table within the same transaction as the effects. If the same id arrives again, skip it.
This turns at-least-once delivery into exactly-once *effect*.

**Example.**

```sql
BEGIN;
INSERT INTO inbox (message_id) VALUES ('e1');  -- unique constraint; fails on duplicate
UPDATE inventory SET reserved = reserved + 1 WHERE sku = 'X';
COMMIT;
```

If the insert raises a unique-violation, the message was already handled — acknowledge and move on.

**Advanced.** The inbox table needs a retention policy: rows must live at least as long as the
broker might redeliver (retention period plus the longest possible consumer downtime), then be
deleted. Also note the requirement that makes this work: the effect and the id must be in **the
same transaction and the same database**. If your effect is an external API call, the inbox does not
help — you need an idempotency key at that API instead.

### Interview questions

- "Why not just publish after commit?"
- "Outbox poller versus CDC: trade-offs?"
- "How do you keep per-order event ordering through the outbox?"
- "What is an inbox table and when does it not work?"

---

## M16 · Idempotent consumers and the exactly-once myth

`Advanced` · Requires: `M09`, `M14`, `M15` · Unlocks: `SD10`, `Q15`, `Q16`

### Preface

Message systems promise **at-least-once** delivery: a message will arrive, possibly more than once.
Exactly-once delivery is impossible in general — the sender can never be certain the receiver got
the message, so it must either risk losing it or risk sending it twice.

What you *can* achieve is exactly-once **effect**: the message may arrive five times, but the
result is the same as if it arrived once. That is entirely in the consumer's hands.

### Details

#### 1. Why exactly-once delivery is impossible

**Theory.** The Two Generals problem: two parties communicating over a lossy channel can never both
be certain of agreement. Applied here: the broker sends, the consumer processes, the acknowledgement
is lost. The broker must choose — redeliver (possible duplicate) or not (possible loss). Any
guarantee of "no duplicates and no loss" requires an unbounded chain of acknowledgements.

**Example.** A consumer processes a message, writes to the database, then crashes before
acknowledging. The broker has no way to know how far it got, so it redelivers. The duplicate is a
correct consequence of the protocol, not a bug to be fixed at the broker.

**Advanced.** Kafka's "exactly-once semantics" is real but narrow: with transactions and
`read_committed`, a read-process-write loop **within Kafka** is atomic — offsets and output
messages commit together. The moment your side effect leaves Kafka (a database write, an HTTP
call, an email), that guarantee does not extend to it. Stating this boundary clearly is one of the
strongest signals in a streaming interview.

#### 2. Techniques for idempotent processing

**Theory.** In rough order of preference:
1. **Naturally idempotent operations** — setting a value rather than incrementing.
2. **Conditional writes** — `UPDATE ... WHERE version = ?`, or upserts on a unique key.
3. **Deduplication store** — record processed message ids and skip repeats (the inbox, `M15`).
4. **Sequence numbers** — ignore any message with a sequence not greater than the last applied.

**Example.** Handling `OrderShipped`:
- Bad: `UPDATE orders SET shipment_count = shipment_count + 1` — a duplicate corrupts the count.
- Good: `UPDATE orders SET status = 'SHIPPED', shipped_at = $1 WHERE id = $2 AND status <> 'SHIPPED'`
  — running it twice changes nothing the second time, and it needs no extra table.

**Advanced.** Pay attention to **ordering plus idempotency together**. If `OrderShipped` and
`OrderCancelled` can arrive in either order, naive idempotency still leaves the wrong final state.
Use a version or timestamp from the producer and reject anything older than what you have applied:
`WHERE version < $new_version`. This makes the consumer both idempotent and out-of-order safe,
which is usually what you actually need.

#### 3. Side effects that leave your system

**Theory.** Database effects can be deduplicated inside your own transaction. External effects
cannot — you must push the idempotency to the external system or accept the duplicate.

**Example.** Sending an email on `OrderPlaced`. If the consumer is redelivered, the customer gets
two emails. Options: give the email provider an idempotency key (most support one); record
"email sent for order X" in your database in the same transaction as the send attempt (still
imperfect, but narrows the window to a crash between the call and the commit); or accept it,
because a duplicate email is cheap. For a payment, none of those is acceptable and you must use
the provider's idempotency key.

**Advanced.** The general rule is to order operations so the irreversible one happens last and is
protected by an idempotency key: do the database work, commit, then call the external system with a
key derived from the message id. If you crash before the external call, redelivery repeats it; if
you crash after, the key prevents duplication. You cannot remove the gap, only make it safe.

#### 4. Operating a dedup store

**Theory.** A deduplication table grows forever unless you prune it, and it must be consulted on
the hot path, so its performance matters.

**Example.** Table `processed_messages(message_id PK, processed_at)`, with the insert as the
dedup check (let the unique constraint do the work rather than a read-then-write, which races).
A daily job deletes rows older than the broker's retention plus a safety margin. Partitioning by
day makes pruning a partition drop rather than a mass delete (see `DB24`).

**Advanced.** Redis with a TTL is a tempting dedup store and is acceptable only when losing the
dedup record is tolerable — Redis is not durable in the way your database is, and an eviction under
memory pressure silently turns duplicates back on. If correctness depends on it, keep the record in
the same database and transaction as the effect. This is a good example of choosing storage by the
consequence of losing the data, not by convenience.

### Interview questions

- "Kafka advertises exactly-once semantics. What does that actually mean?"
- "How do you make a consumer idempotent when it sends an email?"
- "A duplicate `OrderShipped` doubles a counter. Rewrite the handler."
- "How large does your dedup table get and how do you expire it safely?"
- "How do you handle messages arriving out of order *and* twice?"
