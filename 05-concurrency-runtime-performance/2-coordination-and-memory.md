[← back to the field index](README.md)

# Concurrency & Performance · Part 2 — Coordination, Memory & Profiling

Nodes `C09`–`C13`.

---

## C09 · Locks and coordination primitives

`Advanced` · Requires: `C01`, `C06`, `C08`, `DB09` · Unlocks: `C10`, `C11`

### Preface

When two threads must not touch something at the same time, you need mutual exclusion. The
primitives are few, and the failure modes — deadlock, starvation, contention — are well catalogued.

The practical wisdom is that the best lock is the one you did not need: partition the data so each
piece has one owner, or make the operation atomic, and the coordination problem disappears.

### Details

#### 1. The primitives

**Theory.**
- **Mutex** — one holder at a time.
- **Reentrant lock** — the same thread may acquire it again without deadlocking itself.
- **Read-write lock** — many readers or one writer; helps when reads vastly outnumber writes.
- **Semaphore** — a counter permitting N concurrent holders; the natural way to express a
  concurrency limit or a bulkhead (`M10`).
- **Condition variable** — wait until another thread signals a state change.
- **Latch / barrier** — wait until N things have happened.

**Example.** A semaphore is the one that appears most in application code: limiting concurrent
outbound calls to a downstream service (`p-limit` in Node, `Semaphore` in Java) is a bulkhead, and it
prevents one slow dependency from consuming all your capacity.

**Advanced.** A read-write lock is not automatically better than a mutex: it has higher overhead per
operation, and under a write-heavy workload it performs worse. It also risks **writer starvation** if
readers keep arriving — fair implementations fix that at further cost. Measure rather than assume.

#### 2. Deadlock

**Theory.** Four conditions must all hold: **mutual exclusion**, **hold and wait**, **no
preemption**, and **circular wait**. Break any one and deadlock is impossible.

**Example.** The one you break in practice is **circular wait**, by imposing a global ordering: always
acquire locks in a consistent order, such as by ascending id. This is the same fix as for database
deadlocks (`DB09`), and it generalises to any resource acquisition.

**Advanced.** The second most practical is breaking **hold and wait** with timeouts: `tryLock` with a
deadline, and back off and retry on failure. That converts a permanent hang into a transient error
you can observe and retry — always preferable, because a deadlocked thread pool produces a service
that is up, healthy by its own health check, and serving nothing.

#### 3. Contention and granularity

**Theory.** A lock serialises access, so throughput on the protected resource is capped by how long
each holder keeps it. Coarse locks are simple and contended; fine-grained locks are faster and much
easier to get wrong.

**Example.** The progression for a shared map: one lock for the whole map (simple, contended) → a
lock per bucket (Java's older `ConcurrentHashMap`) → lock-free reads with CAS on writes (the modern
implementation). Each step trades complexity for throughput.

**Advanced.** Amdahl's law bounds the payoff: if 5% of your work is serialised, the maximum speedup is
20x regardless of core count. That is why reducing the **critical section** — doing the expensive work
outside the lock and holding it only for the update — usually beats clever locking. "Compute outside,
mutate inside" is the practical rule.

#### 4. Starvation and fairness

**Theory.** A **fair** lock grants access in request order; an **unfair** one lets whoever is
scheduled next take it. Unfair locks have much higher throughput (no handoff cost) and can starve a
thread indefinitely.

**Example.** Java's `ReentrantLock` is unfair by default, and that is usually correct: throughput
matters more than ordering, and starvation is rare in practice. Where fairness genuinely matters — a
per-tenant queue where one tenant must not monopolise — enforce it explicitly at a higher level with
a fair queue, rather than relying on lock fairness (`Q21`).

**Advanced.** **Livelock** is the subtler cousin: threads are actively doing work and making no
progress, each politely backing off in response to the other. Retry loops without randomised backoff
produce it (`M09`). Adding jitter breaks the symmetry, which is the same fix as for retry storms — a
nice connection to make.

### Interview questions

- "Name the four deadlock conditions and which one you break in practice."
- "When is a read-write lock worse than a mutex?"
- "What is Amdahl's law and what does it imply for locking?"
- "What is livelock and how do you fix it?"

---

## C10 · Lock-free techniques, immutability and actors

`Expert` · Requires: `C08`, `C09` · Unlocks: `C11`

### Preface

Locks are one way to make concurrent access safe. The alternatives avoid the problem: make the data
immutable so there is nothing to race on; give each piece of data a single owner; or use atomic
operations instead of blocking.

For application work, the enormously valuable idea here is the **single-writer principle** — partition
by key so only one thing ever writes a given item — because it eliminates coordination rather than
managing it.

### Details

#### 1. Lock-free and compare-and-swap

**Theory.** A lock-free algorithm guarantees that **some** thread makes progress; wait-free
guarantees **every** thread does. Both are built on CAS loops (`C08`) rather than blocking.

**Example.** The everyday benefits are in the libraries you already use: `ConcurrentHashMap`,
`AtomicLong`, concurrent queues, and Go's channels internally. The value of knowing about lock-free
programming is mostly knowing **not to write it yourself** — the ABA problem, memory reclamation and
memory ordering make hand-written lock-free structures notoriously hard to get right.

**Advanced.** The **ABA problem**: a thread reads value A, another changes it to B and back to A, and
the first thread's CAS succeeds even though the world changed underneath it. Solutions attach a
version counter to the value (a tagged pointer, or Java's `AtomicStampedReference`). Naming ABA is a
reliable signal that you have read seriously about this rather than skimmed.

#### 2. Immutability

**Theory.** If data never changes after construction, it can be shared freely with no synchronisation
at all. Updates produce a new value rather than mutating the old one.

**Example.** This is the simplest correct answer to a lot of concurrency problems, and it is why
functional languages have an easier time with concurrency. In practice: return copies rather than
internal collections; use persistent data structures (Immutable.js, Java records, frozen objects)
where sharing matters; and make configuration objects immutable so a reload swaps a reference rather
than mutating fields underneath readers.

**Advanced.** The cost is allocation pressure — creating new objects instead of mutating increases
garbage collection work (`C12`). Persistent data structures mitigate this by sharing unchanged
sub-structures, so updating one element of a large map copies only the path to it, not the whole
thing. The trade is usually worth it for correctness, and worth measuring on a hot path.

#### 3. The single-writer principle

**Theory.** Coordination is only needed when several things write the same data. Partition the data
so each item has exactly one writer, and no locks are required.

**Example.** This is the most transferable idea in this node, and it appears at every scale: a Kafka
partition is consumed by one consumer in a group (`Q12`); a database row is updated by one worker
because jobs are keyed by entity id; an in-memory cache is sharded so each shard has one owner; an
actor processes its own mailbox sequentially. In each case, concurrency comes from having **many
independent items**, not from many writers per item.

**Advanced.** The design question this poses is what to do about operations spanning partitions —
which is where you are forced back into coordination, and it is exactly the same problem as
cross-shard transactions (`DB25`) and sagas (`M14`). Recognising that partitioning pushes the
difficulty to the boundaries, rather than removing it, is the honest version of this principle.

#### 4. Actors and channels

**Theory.** The **actor model** gives each actor private state and a mailbox; actors communicate only
by messages and process them one at a time, so there is no shared mutable state. **CSP** (Go's
channels) has independent goroutines communicating over typed channels — "do not communicate by
sharing memory; share memory by communicating".

**Example.** Erlang and Elixir build supervision trees on top: when an actor crashes, its supervisor
restarts it in a known state. "Let it crash" is a coherent fault-tolerance strategy because state is
isolated, and it maps directly onto the resilience patterns in `M10`.

**Advanced.** The practical distributed-systems version of a mailbox is a per-entity queue: route all
commands for order 123 through one partition, and the handler processes them sequentially with no
locking (`Q14`). That gives you actor semantics using infrastructure you already run, and it is a
strong answer to "how would you avoid a distributed lock" (`C11`).

### Interview questions

- "Remove the lock from a hot counter without breaking correctness."
- "What is the ABA problem?"
- "What is the single-writer principle and where have you applied it?"
- "Why does immutability simplify concurrency, and what does it cost?"

---

## C11 · Concurrency across machines

`Advanced` · Requires: `C09`, `C10`, `Q08`, `M21` · Unlocks: `SD10`

### Preface

A mutex works because one process controls memory. Across machines there is no such guarantee: a
process can hold a "lock" while being paused by garbage collection, partitioned from the network, or
declared dead by everyone else — and it does not know.

So a distributed lock is not a mutex. It is a **lease** with an expiry, and it can be wrong. Designs
that need correctness must not rely on it alone.

### Details

#### 1. Why distributed locks are not locks

**Theory.** The lock is granted for a time (a lease). If the holder is slow, the lease expires, and
another process acquires it — while the first still believes it holds the lock and continues working.
Both now act as the exclusive owner.

**Example.** The canonical sequence: client A acquires a 10-second lock; a garbage collection pause
stops A for 15 seconds; the lock expires and client B acquires it; A wakes up with no idea anything
happened and writes to the shared resource. Nothing was faulty — no network partition, no crash — and
the invariant is broken.

**Advanced.** This is Martin Kleppmann's critique of Redlock, and it is worth being able to state both
sides. Redlock's author argues the algorithm is fine for its intended use; Kleppmann's point is that
**no** lease-based lock can guarantee mutual exclusion without cooperation from the resource being
protected. Both are right about different things, and the resolution is fencing tokens.

#### 2. Fencing tokens

**Theory.** Each lock grant carries a monotonically increasing number. Every write to the protected
resource includes its token, and the resource **rejects any token lower than the highest it has
seen**. A revived old holder is refused.

**Example.**

```
A acquires lock, token = 33 → pauses
B acquires lock, token = 34 → writes with token 34; storage records 34
A resumes, writes with token 33 → storage rejects: 33 < 34
```

The essential requirement is that the **storage layer** checks the token. A lock service alone cannot
provide safety, because it is not in the path of the write.

**Advanced.** In practice you often already have a fencing mechanism: a version column with an
optimistic-concurrency check (`DB10`) is exactly this. `UPDATE ... WHERE version = $expected` rejects
a stale writer without any lock service at all. That observation — that the database's conditional
write is the real safety mechanism — is the strongest answer to this question.

#### 3. Avoiding distributed locks entirely

**Theory.** The best designs do not need one. Partition so a single owner exists; use conditional
writes; make operations idempotent so duplicate execution is harmless.

**Example.** The alternatives, in order of preference:
1. **Partition by key** — route all work for an entity to one consumer (`Q14`, `C10`), so there is
   never a second writer.
2. **Conditional write / CAS** — let the database arbitrate (`DB10`).
3. **Idempotency** — accept that two workers may do the work and make the second a no-op (`M16`).
4. **Database row lock** — `SELECT ... FOR UPDATE` if the work is short and database-bound (`DB09`).
5. **Lease plus fencing** — only when the above cannot apply.

**Advanced.** Distinguish locks for **efficiency** from locks for **correctness**. "Do not let two
workers both regenerate this expensive cache entry" is efficiency: a rare duplicate costs money, not
correctness, and Redis is fine. "Do not charge the card twice" is correctness, and needs fencing or
an idempotency key at the provider (`A09`). Making that distinction explicitly is the answer
interviewers are looking for.

#### 4. Implementing a lease properly

**Theory.** If you must, the requirements are: atomic acquisition, a TTL so a dead holder releases
automatically, safe release (only the owner may release), and renewal for long work.

**Example.**

```
acquire:  SET lock:resource <random-token> NX PX 30000
release:  Lua script — if GET lock:resource == token then DEL else 0
```

Release **must** be the compare-and-delete script, not a plain `DEL` (`Q08`): after your lease
expired, a plain `DEL` would delete somebody else's lock. And the TTL must exceed the expected work
duration, with a renewal (watchdog) thread for long jobs.

**Advanced.** Use a system designed for it where correctness matters: etcd and ZooKeeper provide
leases with monotonic revision numbers usable as fencing tokens, built on consensus (`M20`). Redis is
not a consensus system — its replication is asynchronous, so a failover can lose the lock entirely.
That is the technically precise reason Redis locks are unsafe for correctness, and it is a better
answer than "Redlock is controversial".

### Interview questions

- "Is Redis a safe distributed lock?"
- "Two workers process the same job. Where did your lock fail and how do you make it not matter?"
- "What is a fencing token and who must check it?"
- "How would you design this so no distributed lock is needed?"

---

## C12 · Memory management and garbage collection

`Advanced` · Requires: `C03`, `C06`, `C07` · Unlocks: `C13`, `O06`

### Preface

Managed runtimes free memory for you, which removes a class of bugs and introduces two others:
**pauses** while the collector works, and **leaks** where you keep references to things you no longer
need.

In containers there is a third: the runtime's idea of available memory and the container's limit are
set separately, and when they disagree the process is killed with no stack trace.

### Details

#### 1. Generational collection

**Theory.** The generational hypothesis: most objects die young. So collectors split the heap into a
small **young** generation, collected frequently and cheaply, and an **old** generation collected
rarely and expensively. Objects surviving several young collections are promoted.

**Example.** V8: a small young generation collected by a copying "scavenger" (fast, proportional to
**surviving** objects, so short-lived garbage is nearly free), and an old generation collected by a
mark-sweep-compact collector that runs incrementally and concurrently to limit pauses. The JVM's G1
divides the heap into regions and collects the ones with most garbage first; ZGC and Shenandoah aim
for sub-millisecond pauses on very large heaps.

**Advanced.** The practical implication is that **allocation rate** matters more than total heap size
for pause frequency. A request handler creating thousands of short-lived objects triggers frequent
young collections. They are cheap, and at high throughput they add up — which is why reducing
allocations (reusing buffers, avoiding unnecessary intermediate arrays and string concatenation in
loops) is a real optimisation on hot paths.

#### 2. Containers and the OOM killer

**Theory.** The container has a memory limit enforced by cgroups (`O04`). The runtime has its own
heap limit. If the runtime's limit is higher than the container's, the process is killed by the
kernel before the collector ever feels pressure.

**Example.** A Node pod with a 512MB container limit and no `--max-old-space-size` uses a default heap
cap that may exceed it. The heap grows toward its own limit, total memory passes 512MB, and the pod is
**OOMKilled with exit code 137** — no exception, no stack trace, nothing in the application log. Set
`--max-old-space-size` to roughly 75-80% of the container limit, leaving room for buffers, native
allocations and thread stacks. The JVM equivalent is `-XX:MaxRAMPercentage=75`, which is container-aware
by default in modern versions.

**Advanced.** The gap between heap and RSS is the part people miss: `Buffer` allocations in Node are
outside the V8 heap; the JVM has metaspace, thread stacks, code cache and direct byte buffers. So a
process reporting 200MB of heap can legitimately use 400MB of RSS. When a pod is OOMKilled while heap
metrics look healthy, native memory is the answer (`O04`).

#### 3. Leaks in managed runtimes

**Theory.** A garbage collector frees only what is unreachable. A leak is therefore a **reference you
forgot** — memory that is still reachable and will never be used again.

**Example.** The recurring patterns:
- An **unbounded cache**: a `Map` keyed by user id, never evicted (`F15`). The commonest Node leak.
- **Event listeners** added per request and never removed, so the emitter accumulates closures.
- **Closures** capturing large objects: a callback that references the whole request body keeps it
  alive as long as the callback is reachable.
- **`ThreadLocal` in a pooled thread** that is never cleared (`C06`).
- **Timers and intervals** never cleared, keeping their closures alive.
- A **global array** used as a buffer that is appended to and never trimmed.

**Advanced.** The diagnostic method: take three heap snapshots — a baseline, after load, and after
more load — and compare retained sizes between them. Anything growing monotonically across all three
is the leak, and the retainer path shows exactly what is holding it. In Node use
`--heapsnapshot-signal` or the inspector; in the JVM use a heap dump plus Eclipse MAT. Correlating the
start of the growth with a deploy usually identifies the change responsible.

#### 4. Tuning, and when not to

**Theory.** Most services need very little garbage-collection tuning. Set the heap size correctly
relative to the container, choose a collector appropriate to your latency requirements, and stop.

**Example.** When it is genuinely worth tuning: p99 latency is dominated by pauses (visible as
periodic spikes correlating with collection events); the heap is very large, where a low-pause
collector helps; or allocation rate is extreme and reducing it is the real fix. Before tuning,
confirm the pauses are actually the problem by exporting GC metrics (`O14`).

**Advanced.** The most common real fix is not a collector flag but reducing garbage: streaming instead
of buffering (`C19`), avoiding large intermediate arrays, reusing buffers, and not serialising more
data than you send. Framing it as "reduce allocation before tuning collection" is the right instinct
to demonstrate.

### Interview questions

- "A Node pod is OOMKilled at 512MB but the heap shows 200MB. Explain."
- "How do you find a memory leak in production?"
- "What is the generational hypothesis and why does it make collection cheap?"
- "What would you tune before touching GC flags?"

---

## C13 · Profiling and benchmarking

`Advanced` · Requires: `C04`, `C12` · Unlocks: `C14`, `O17`

### Preface

The rule is: measure first. Intuition about what is slow is wrong often enough that acting on it
wastes days.

The key distinction is **on-CPU** versus **off-CPU** time. A CPU profile shows where cycles are spent.
Backend services usually spend most of their time **waiting** — for the database, for a downstream, for
a lock, for a connection from the pool — and a CPU profile of a waiting service looks empty.

### Details

#### 1. CPU profiles and flame graphs

**Theory.** A sampling profiler interrupts the program periodically and records the stack. Aggregating
thousands of samples shows which stacks consume the most time. A **flame graph** draws this: width is
time, and vertical stacking is call depth.

**Example.** How to read one: look for the **widest** frames, not the tallest. Width is cost; height
is only nesting depth. A wide plateau near the top is where time is actually spent. Tools: `--cpu-prof`
or clinic.js or 0x for Node; async-profiler or JFR for the JVM; py-spy for Python.

**Advanced.** Sampling profilers have blind spots. Node's older `--prof` attributes a lot of time to
opaque C++ frames; async-profiler avoids the JVM's safepoint bias that made older Java profilers
misattribute time. And a sampling profiler cannot see anything that happens while the thread is not
running — which is exactly the waiting you most want to understand.

#### 2. Off-CPU analysis

**Theory.** Off-CPU profiling measures where threads are **blocked**: on I/O, on locks, on a
connection pool, on a semaphore. For an I/O-bound service this is where nearly all latency lives.

**Example.** The practical substitute for most teams is **distributed tracing** (`O15`): a trace of
one slow request shows exactly which span consumed the time, whether it was your code or a downstream
or the database. That is off-CPU analysis for practical purposes, and it is far easier to deploy than
kernel-level tooling.

**Advanced.** Cheap, targeted alternatives: measure the time to acquire a connection from the pool
separately from the query time (`DB21`); measure event loop delay in Node (`C03`); instrument lock
wait times in the JVM. Each of these turns invisible waiting into a metric, and they are usually
enough without eBPF-based off-CPU profiling.

#### 3. Benchmarking traps

**Theory.** Microbenchmarks lie in well-known ways, and a benchmark that measures the wrong thing is
worse than none because it justifies the wrong decision.

**Example.** The traps:
- **No warm-up** — JIT compilation, cache population and connection pools mean the first N
  iterations are unrepresentative.
- **Dead code elimination** — the compiler removes work whose result is unused, so you measure
  nothing. Consume the result.
- **Unrealistic data** — a benchmark over 100 rows tells you nothing about 10 million; data
  distribution matters as much as size.
- **Everything cached** — a benchmark running the same query repeatedly measures the cache.
- **Measuring the client** — the load generator itself becomes the bottleneck.

**Advanced.** **Coordinated omission** is the subtlest and matters most for latency measurement: if
your load generator waits for a response before sending the next request, then during a slow period it
simply sends fewer requests — so the slow period is under-represented and your reported p99 is far
better than reality. Tools with an open workload model (constant arrival rate regardless of response
times) avoid it; the term is worth knowing by name (`C15`).

#### 4. Profiling in production

**Theory.** Staging does not reproduce production's data, traffic mix or concurrency. Continuous
profiling runs a low-overhead sampling profiler in production all the time.

**Example.** Tools such as Pyroscope, Parca or a cloud provider's profiler sample at around 100Hz for
roughly 1-2% overhead, and let you compare flame graphs across deploys — "which function got slower
after yesterday's release" becomes a diff rather than an investigation.

**Advanced.** Combine the signals: metrics tell you **that** something is slow, traces tell you
**where** in the system, and profiles tell you **which code**. A team with all three resolves
performance regressions in minutes; a team with only metrics guesses. Saying that chain out loud
answers "how do you approach a performance problem" better than any single tool name (`O17`).

### Interview questions

- "The endpoint is slow but the CPU profile is flat. Where is the time?"
- "How do you read a flame graph?"
- "What is coordinated omission?"
- "Walk me through diagnosing a performance regression after a deploy."
