[← back to the field index](README.md)

# Concurrency & Performance · Part 3 — Latency, Throughput & Data Flow

Nodes `C14`–`C19`.

---

## C14 · Latency, percentiles and queueing theory

`Advanced` · Requires: `C13`, `DB21`, `M07` · Unlocks: `C15`, `C16`, `O14`, `SD02`

### Preface

This is the most valuable node in the field. Three ideas — **percentiles**, **Little's Law**, and the
relationship between **utilisation and latency** — let you reason quantitatively about a system
instead of guessing.

They also produce the two answers that mark out a senior candidate: why averages are useless, and why
a system at 80% utilisation is much slower than one at 40%.

### Details

#### 1. Percentiles, and why averages lie

**Theory.** The average hides the tail. A service where 99% of requests take 10ms and 1% take 5
seconds has an average of about 60ms — which describes nobody's experience. p50 is the typical user;
p99 and p99.9 are the users who are suffering.

**Example.** Why the tail matters more than it seems: a user action that makes 100 backend requests
hits the p99 on at least one of them with probability `1 − 0.99^100 ≈ 63%`. So **most page loads
experience your p99**. That arithmetic is the single most persuasive argument for caring about tail
latency, and it is worth memorising.

**Advanced.** **You cannot average percentiles.** The p99 of two instances is not the average of their
p99s, and the p99 of an hour is not the average of the 60 per-minute p99s. Computing either produces
a number that is simply wrong — usually optimistic. The correct approach is to aggregate **histograms**
and compute the percentile from the merged distribution, which is why Prometheus histograms and
`histogram_quantile` exist (`O14`). Stating this confidently is a strong signal.

#### 2. Little's Law

**Theory.** **L = λW**: the average number of items in a system equals the arrival rate multiplied by
the average time each spends there. It holds for any stable system, with no assumptions about
distributions.

**Example.** It answers sizing questions directly:
- 500 requests per second, 200ms each → `500 x 0.2 = 100` concurrent requests in flight. That is how
  many workers, threads or connections you need.
- A connection pool of 20 with 5ms queries serves `20 / 0.005 = 4,000` queries per second (`DB21`).
- A queue holding 10,000 messages, consumed at 500 per second → each message waits 20 seconds
  (`M30`).

**Advanced.** The rearrangement `W = L / λ` is the one to use during an incident: queue depth divided
by throughput gives the current wait time, which is what users actually experience. It also shows why
a bounded queue is a latency bound — choosing a maximum queue length is choosing a maximum wait
(`M30`). Deriving a pool size from Little's Law in an interview, rather than quoting a rule of thumb,
is exactly the level expected.

#### 3. Utilisation and the hockey stick

**Theory.** Queueing theory gives an approximate wait time of `W ≈ S x ρ / (1 − ρ)`, where `S` is
service time and `ρ` is utilisation. As utilisation approaches 1, wait time goes to infinity —
non-linearly.

**Example.** With a 10ms service time:
| Utilisation | Approximate wait |
|---|---|
| 50% | 10ms |
| 70% | 23ms |
| 80% | 40ms |
| 90% | 90ms |
| 95% | 190ms |

Going from 70% to 90% doubles the load and quadruples the wait. This is why a system runs fine for
months and then falls over after a modest traffic increase — and why capacity planning targets 60-70%
rather than 90% (`C15`).

**Advanced.** The variability of service times makes it worse: the formula assumes randomness, and a
workload mixing 5ms and 3-second requests queues far more badly than the average suggests. That is the
quantitative argument for separating workloads — putting slow reports on their own pool or fleet
(`M10` bulkheads) — because one long request delays everything queued behind it.

#### 4. RED, USE and what to measure

**Theory.** **RED** for services: **R**ate (requests per second), **E**rrors, **D**uration
(distribution). **USE** for resources: **U**tilisation, **S**aturation (queue depth), **E**rrors.
Together they cover "is the service healthy" and "is the resource the cause".

**Example.** Applied: for the API, RED per endpoint. For the database connection pool, USE —
utilisation (connections in use), saturation (requests waiting for a connection), errors (acquisition
timeouts). Pool saturation is a leading indicator that predicts latency before it becomes visible to
users (`F24`).

**Advanced.** **Saturation is the most useful and least instrumented** of these. Utilisation tells you
how busy something is; saturation tells you how much work is waiting, which is what predicts the
hockey stick. Queue depth, pool wait time, event loop delay and consumer lag are all saturation
metrics, and a dashboard with all four answers most "why is it slow" questions immediately (`O14`).

### Interview questions

- "Each service has a 10ms p99. You call 50 of them in parallel. What is your p99?"
- "Your box is at 80% CPU and latency doubled. Why non-linear?"
- "You need 500 rps with 200ms latency. How many concurrent workers?"
- "Why can't you average p99 across pods?"

---

## C15 · Load testing and capacity planning

`Intermediate` · Requires: `C14` · Unlocks: `SD02`, `O18`

### Preface

A load test answers two questions: how much can this handle, and what breaks first. The second is
more useful — knowing the bottleneck tells you what to fix and what to monitor.

The most common mistake is testing until it passes rather than testing until it breaks. A test that
confirms you can handle today's traffic tells you nothing about the margin.

### Details

#### 1. Open versus closed workload models

**Theory.** A **closed** model has N virtual users, each waiting for a response before sending the
next request — so when the system slows, load automatically decreases. An **open** model sends at a
fixed **arrival rate** regardless of how fast responses come back, which is how real traffic behaves.

**Example.** This is where **coordinated omission** comes from (`C13`): in a closed model, a 5-second
stall means far fewer requests were sent during the stall, so it barely registers in the percentiles.
Your reported p99 is much better than reality. Use tools with an open model or an arrival-rate
executor — k6's `constant-arrival-rate`, Gatling's injection profiles, wrk2.

**Advanced.** A closed model is the right choice when you genuinely have a fixed number of clients
that behave that way — an internal batch integration with 10 workers. For a public API, real users
keep arriving whether or not you are keeping up, so open is correct. Choosing the model deliberately,
and saying why, is a strong detail.

#### 2. Finding the knee, not the pass mark

**Theory.** Ramp load gradually and plot throughput and latency against offered load. Throughput rises
linearly, then flattens; latency is flat, then rises sharply. The point where they diverge is the
**knee** — your real capacity.

**Example.** What the shape tells you: if throughput flattens while CPU is well below 100%, you are
blocked on something else — the connection pool, a downstream service, a lock (`C14`). If latency
rises and throughput *falls*, you have passed into overload, usually because of queueing and retries
(`M30`). Running to the knee gives you both the number and the bottleneck.

**Advanced.** Test the **dependencies** too, and be careful about it: load-testing against a shared
staging database will take out everyone else's environment, and load-testing against a third-party
sandbox may get you rate-limited or banned. Isolate the environment, and mock external providers with
realistic latency — mocking them as instant produces a test that exercises nothing real.

#### 3. Realistic tests

**Theory.** The test must resemble production in traffic mix, data size, cache state and concurrency
distribution.

**Example.** What makes tests useless: hitting one endpoint when production has a mix; using 100 test
rows when production has 50 million (`DB18` — plans change with data size); repeatedly requesting the
same id so everything is cached; and no think time, which produces an unrealistic arrival pattern.
Derive the traffic mix from production access logs, and use a production-sized dataset.

**Advanced.** **Soak tests** are the ones people skip and the ones that find the expensive problems: a
moderate load for several hours reveals memory leaks (`C12`), connection leaks, log or disk growth,
and slow degradation from table bloat (`DB08`). A 10-minute peak test finds none of these. Run a soak
before any significant launch.

#### 4. Capacity planning

**Theory.** Convert the measured knee into a provisioning decision: target utilisation well below it,
size for peak rather than average, and leave headroom for failure.

**Example.** The arithmetic: peak is 3x average; you want to survive losing one of three availability
zones, so 1.5x again; and you target 60% utilisation, so divide by 0.6. If average load is 1,000 rps
and one instance handles 500 rps at the knee, you need
`1000 x 3 x 1.5 / 0.6 / 500 ≈ 15` instances. Show the working — that is the answer, not the number.

**Advanced.** Autoscaling reduces but does not remove the need for headroom: scaling takes time (pod
start, warm-up, JIT), the signal lags, and during a zone failure everyone else is scaling at the same
moment and capacity may not be available. **Static stability** (`M31`) says provision to survive the
failure without needing to scale. That is the sophisticated version of capacity planning and a good
point to raise unprompted.

### Interview questions

- "Design a load test for the checkout flow. What do you measure and when do you stop?"
- "What is the difference between an open and a closed workload model?"
- "Why is a soak test worth running?"
- "How many instances do you need? Show your working."

---

## C16 · Backpressure end to end

`Advanced` · Requires: `C05`, `C14`, `M30` · Unlocks: `F16`, `Q15`

### Preface

Backpressure is the signal that travels upstream saying "slow down". Without it, a fast producer and
a slow consumer fill a buffer until memory runs out or latency becomes meaningless.

The rule to carry: **every buffer must be bounded**, and when it is full you must make a deliberate
choice — block the producer, or reject the work. Silently growing is never an option.

### Details

#### 1. Node streams

**Theory.** `writable.write()` returns `false` when its internal buffer is over the high-water mark.
That is the backpressure signal, and honouring it means stopping until the `drain` event.

**Example.** The wrong and right versions of the same job:

```js
// wrong: ignores the return value; the whole source ends up in memory
for await (const row of cursor) res.write(format(row));

// right: pipeline handles backpressure and error propagation
await pipeline(cursor, formatTransform, res);
```

`pipeline()` also propagates errors and destroys the streams on failure, which manual `.pipe()`
chains do not — a leaked stream after an error is a common source of file descriptor exhaustion.

**Advanced.** Async iterators (`for await`) handle backpressure naturally on the **read** side: the
source is not asked for more until you are ready. The write side still needs `pipeline` or an explicit
`drain` wait. Mixing the two — reading with an async iterator and writing without checking the return
value — is the most common way to reintroduce the problem while believing it is handled.

#### 2. Reactive streams and request(n)

**Theory.** Reactive Streams formalises backpressure: the consumer calls `request(n)` to say how many
items it can accept, and the producer sends no more than that. It is **pull-based** demand over a
push-based transport.

**Example.** In Project Reactor or RxJava, operators propagate demand upstream. A slow database writer
requests fewer items, so the HTTP client reads more slowly, so TCP flow control slows the sender.
Backpressure travels the whole chain automatically — which is the strongest argument for the reactive
model, and the reason a single blocking call in the chain breaks it (`C06`).

**Advanced.** Note what happens when the source **cannot** be slowed — a Kafka topic, a UDP stream, a
WebSocket from a client you do not control. Then your only options are to buffer (bounded), drop, or
sample. Reactor exposes exactly these as `onBackpressureBuffer`, `onBackpressureDrop` and
`onBackpressureLatest`. Choosing among them is a product decision: for live telemetry, dropping old
values is usually right; for orders, it never is.

#### 3. Backpressure at the protocol level

**Theory.** TCP has a receive window: the receiver advertises how much it can accept and the sender
stops when it is full. HTTP/2 adds per-stream flow control on top.

**Example.** So backpressure is already present at the transport layer — but only if your application
stops reading. A server that reads everything into memory as fast as it arrives defeats TCP flow
control entirely, because the kernel buffer drains immediately into your heap. Honouring application
backpressure is what makes the transport's mechanism effective.

**Advanced.** For message consumers, backpressure is expressed as **pull** or **prefetch limits**:
Kafka consumers pull at their own pace (`Q15`), and RabbitMQ requires an explicit prefetch count or
the broker floods the consumer (`Q11`). In both cases an unbounded consumer moves the backlog from
the broker — where it is safe, durable and measurable — into your process memory, where it is none of
those things.

#### 4. When to reject instead

**Theory.** Backpressure slows the producer. When the producer is a user who will not wait, slowing
them is equivalent to failing, but slower and more expensively. Then rejection is the right answer.

**Example.** For synchronous user-facing traffic: bounded queue, and return 503 with `Retry-After`
when it is full (`M30`). For asynchronous pipelines: real backpressure, because the producer is a
system that can genuinely wait. The distinction is whether the upstream has somewhere to put the work
while it waits.

**Advanced.** The general principle: **backpressure for systems, load shedding for users.** And shed
as early and as cheaply as possible — rejecting at the gateway costs almost nothing, while rejecting
after authentication and three database queries means you paid for work you threw away (`M10`). That
sentence is a compact, complete answer to a common design question.

### Interview questions

- "You pipe a database cursor to an HTTP response and memory explodes. What did you skip?"
- "What is prefetch in RabbitMQ and why does it matter?"
- "What do you do when the source cannot be slowed down?"
- "Backpressure or load shedding — how do you choose?"

---

## C17 · Network-level efficiency patterns

`Intermediate` · Requires: `C05` · Unlocks: `SD05`

### Preface

Most backend latency is round trips. A service making 200 sequential calls per request is slow no
matter how fast each call is.

The techniques for fixing it are few and reliably effective: reuse connections, batch, parallelise
with a bound, coalesce duplicate work, and move the computation to where the data already is.

### Details

#### 1. Batching

**Theory.** Replace N requests with one request carrying N items. The saving is N−1 round trips plus
N−1 sets of per-request overhead.

**Example.** Everywhere this applies: `WHERE id IN (...)` instead of N queries (`DB20`); a batch
endpoint instead of N HTTP calls; Redis pipelining instead of N round trips (`Q08`); Kafka producer
batching; DataLoader for GraphQL resolvers (`A15`). In each case the mechanism is the same and the
improvement is usually an order of magnitude.

**Advanced.** Batching trades latency for throughput when it involves **waiting** to accumulate a
batch — a Kafka producer with `linger.ms=10` waits up to 10ms to fill a batch. For request-scoped
batching (DataLoader), the wait is one event-loop tick, so the trade is negligible. Know which kind
you are doing: adding artificial delay to a user-facing path needs justification.

#### 2. Parallelism with a bound

**Theory.** Independent calls should run concurrently, so total latency is the maximum rather than the
sum. But unbounded concurrency overwhelms the downstream (`C05`).

**Example.** Three sequential 50ms calls take 150ms; run concurrently they take 50ms. Five hundred
concurrent calls, however, exhaust your connection pool and trip the downstream's rate limiter. The
answer is always "concurrent, with a limit" — and the limit should come from the downstream's
capacity divided by your instance count (`F24`).

**Advanced.** Note the effect on tail latency: with a fan-out, your latency is the **maximum** of the
calls, so you are exposed to each downstream's p99 (`C14`). Mitigations: a timeout with a fallback per
call so one slow dependency cannot dominate (`M10`), or hedged requests for idempotent reads (`M09`).

#### 3. Request coalescing

**Theory.** When many concurrent requests ask for the same thing, do the work once and share the
result. Known as singleflight or request collapsing.

**Example.** A hundred requests arrive for an uncached popular item. Without coalescing, a hundred
identical database queries (`Q04` — the stampede). With coalescing, the first starts the query and the
other ninety-nine await the same promise. In Node this is a `Map` from key to in-flight promise; in Go
it is `singleflight`.

**Advanced.** This is the in-process half of cache stampede protection; the distributed half is a lock
or a probabilistic early refresh in the shared cache (`Q04`). Both are worth having: coalescing removes
the per-instance amplification for free, and the distributed protection handles the N-instances case.

#### 4. Move the computation to the data

**Theory.** Transferring data to compute over it is usually more expensive than computing where it
already lives.

**Example.** Filtering in the application what the database could filter; loading 100,000 rows to sum
a column; fetching a whole document to read one field. Each is a network transfer, a deserialisation
and a memory cost that a `WHERE`, a `SUM`, or a projection would avoid entirely (`DB20`).

**Advanced.** The same principle at larger scale: a materialised view or read model computes the
answer once at write time rather than per read (`M19`); a CDN moves the data to the user; an edge
function moves computation to the data's location. The general rule — **do the work once, close to
the data, as early as possible** — connects this node to caching, CQRS and CDN design in one sentence.

### Interview questions

- "Your service makes 200 sequential internal calls per request. Give three fixes in order of impact."
- "What is request coalescing and where does it help?"
- "Why is unbounded parallelism a problem?"
- "When does batching hurt latency?"

---

## C18 · Data structures and algorithmic cost in backend code

`Intermediate` · Requires: — · Unlocks: `C13`, `DB33`

### Preface

You will not be asked to invert a binary tree in a senior backend interview. You may well be shown a
function and asked to find the accidental quadratic loop.

The set of structures that actually matter in backend work is small, and the mistakes are repetitive:
a linear search inside a loop, sorting when you needed the top few, and string concatenation building
a large result.

### Details

#### 1. The accidental quadratic

**Theory.** A linear operation inside a loop over the same data is O(n²). It is invisible in code
review at small scale and catastrophic at large scale.

**Example.** The pattern to recognise instantly:

```js
// O(n * m): includes() scans the array each time
const missing = allIds.filter(id => !existingIds.includes(id));

// O(n + m)
const existing = new Set(existingIds);
const missing = allIds.filter(id => !existing.has(id));
```

With 10,000 and 10,000 items that is 100 million comparisons versus 20,000. The same shape appears as
`array.find()` inside a loop, and as a repeated `indexOf`.

**Advanced.** The general fix is to build a lookup structure once before the loop — a `Set` or `Map`
keyed by whatever you are matching on. This is also the in-memory equivalent of a database index, and
the equivalent of a hash join versus a nested loop join (`DB19`) — the same idea at three different
layers, which is a nice observation to make.

#### 2. The structures worth knowing

**Theory.** For backend work: **hash map** (O(1) lookup, no order), **sorted structure** — a tree or
skip list (ordered, O(log n), range queries), **heap** (top-k and schedulers), **trie** (prefix
search, autocomplete), **ring buffer** (fixed-size recent history), **bit set** (dense boolean sets in
tiny space), **LRU** (hash map plus doubly-linked list).

**Example.** Where each shows up: hash map everywhere; sorted structures for leaderboards and range
queries (Redis sorted sets, `Q05`); heaps for "the next 10 jobs by due time" and for top-k; tries for
autocomplete; ring buffers for recent-events windows; bit sets for feature flags or membership across
millions of ids.

**Advanced.** "Implement an LRU cache" is still asked, and the answer is a hash map for O(1) lookup
plus a doubly-linked list for O(1) recency updates — move a node to the head on access, evict from the
tail. Be able to write it. It is also worth knowing that a real implementation needs a size **in
bytes** rather than entries for memory safety (`C12`).

#### 3. Cost at your actual scale

**Theory.** Big-O matters when n is large or growing. For small, fixed n, constant factors and
clarity dominate — an O(n²) loop over 10 items is fine.

**Example.** The judgement to apply: ask what n is in production and what it will be in two years. A
linear scan over 50 configuration entries is fine forever. A linear scan over "all orders for this
customer" is fine until one customer has 200,000. The bugs come from data that grows without anyone
revisiting the code.

**Advanced.** The backend-specific point is that **the database is usually the n that matters**. An
O(n²) in memory over 1,000 rows is a few milliseconds; an unindexed query over the same table is a
sequential scan repeated per request. Profile before optimising algorithms (`C13`) — in most services,
the algorithmic win is in the query plan, not in the JavaScript.

#### 4. Memory as well as time

**Theory.** Complexity has a space dimension, and in a container memory is a hard limit (`O04`).

**Example.** Loading a million rows to compute a sum uses hundreds of megabytes and may OOM the pod;
`SUM()` in the database uses none. Building an array of every result before sending uses memory
proportional to the result; streaming uses constant memory (`C19`). Deduplicating with a `Set` over
ten million ids is roughly a gigabyte in JavaScript — a Bloom filter is 12MB (`DB33`).

**Advanced.** The habit worth demonstrating: for any operation over a collection, ask "what is the
maximum size this can be, and what happens then?" Most memory incidents are an operation that was
bounded in practice becoming unbounded because of a new customer, a missing filter, or a retry loop.
Adding an explicit limit — and an error when it is exceeded — turns a silent OOM into a clear failure.

### Interview questions

- "Here is a service function over 50,000 records; find the accidental O(n²)."
- "Implement an LRU cache."
- "When does big-O not matter?"
- "How much memory does deduplicating ten million ids take, and what is the alternative?"

---

## C19 · Streaming and zero-copy

`Advanced` · Requires: `C02`, `A20` · Unlocks: `F17`

### Preface

Buffering means holding the whole thing in memory. Streaming means processing it in pieces, so memory
stays constant regardless of size.

For anything whose size is controlled by data or by users — exports, uploads, imports, reports — this
is the difference between a service that scales and one that dies at an unpredictable moment.

### Details

#### 1. Streaming instead of buffering

**Theory.** A stream processes chunks as they arrive. Memory is proportional to the chunk size and the
buffer high-water mark, not to the total.

**Example.** An export of ten million rows: read from a database cursor, transform each row to CSV,
write to the response — constant memory, and the first bytes reach the client immediately. The
buffered version builds an array of ten million objects, serialises it to a several-gigabyte string,
and the pod dies. The code difference is small; the operational difference is total.

**Advanced.** The subtlety is that the database driver must stream too. Most drivers **buffer the
entire result set by default** — `pg` in Node requires `pg-cursor` or `pg-query-stream`; JDBC requires
setting a fetch size and disabling auto-commit for the cursor to be server-side. Streaming your
application code while the driver buffers everything achieves nothing, and this is a commonly missed
half of the fix.

#### 2. Chunked formats

**Theory.** A format that can be produced and consumed incrementally is required for streaming.
JSON arrays are awkward (a consumer must parse the whole document); **NDJSON** (one object per line),
CSV and length-prefixed binary records are natural.

**Example.** For a streaming API, NDJSON with `Content-Type: application/x-ndjson`: each line is a
complete JSON object, so the consumer processes as it reads and can stop early. It is also resilient —
a truncated stream loses the final partial line, not the whole document.

**Advanced.** Streaming's weakness is error handling: once you have sent a 200 and part of the body,
you cannot retract it (`A20`). Mitigations: emit a final sentinel record the consumer checks for, or
use trailers, or avoid the problem by generating to object storage and returning a link (`A21`). For
anything a customer depends on, the asynchronous approach is more robust.

#### 3. Zero-copy

**Theory.** Normally, sending a file means copying it from disk into kernel memory, into user-space
memory, back into kernel socket buffers, then out. `sendfile` lets the kernel do it without the
user-space round trip.

**Example.** This is a substantial part of why Kafka is fast: it stores messages in the same format
it sends them, so serving a consumer is a `sendfile` from the page cache straight to the socket — no
deserialisation, no copies. Nginx serves static files the same way.

**Advanced.** Application-level equivalents: in Node, `stream.pipeline` with a file stream avoids
loading into your heap (though not a true kernel `sendfile` unless the runtime uses it underneath);
`Buffer` slices share the underlying memory rather than copying, so `buf.subarray()` is free while
`Buffer.from(buf)` copies; and `SharedArrayBuffer` with worker threads transfers ownership without
copying (`C04`). Each is a case of the same idea: avoid moving bytes you do not need to move.

#### 4. Memory budgets per connection

**Theory.** If you serve 1,000 concurrent connections and each buffers 1MB, that is 1GB. Concurrency
multiplies per-connection memory, and the limit is what determines how many connections a pod can
hold.

**Example.** The budget to calculate: per-connection memory x expected concurrency must fit
comfortably inside the container limit (`O04`), leaving room for everything else. Streaming with a
64KB high-water mark makes 10,000 connections cost 640MB of buffers rather than 10GB — which is the
difference between feasible and not.

**Advanced.** Under backpressure this matters even more (`C16`): a slow client that cannot consume
what you send causes your buffer to grow. With a bounded high-water mark, the stream stops producing
and memory stays flat; without one, a handful of slow clients can exhaust your heap. "Bound every
buffer and honour backpressure" is the one-sentence summary of both nodes.

### Interview questions

- "Export ten million rows to CSV over HTTP without OOM. Design it."
- "Your code streams but the pod still runs out of memory. What did you miss?"
- "Why is Kafka able to serve consumers so cheaply?"
- "How much memory does 10,000 concurrent streaming connections cost?"
