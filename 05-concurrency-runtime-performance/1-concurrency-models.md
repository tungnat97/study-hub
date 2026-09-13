[← back to the field index](README.md)

# Concurrency & Performance · Part 1 — Concurrency Models and Runtimes

Nodes `C01`–`C08`.

---

## C01 · Process, thread and coroutine

`Beginner` · Requires: — · Unlocks: `C02`, `C06`, `C07`, `C09`

### Preface

Three ways to run more than one thing at a time, in increasing order of cheapness and decreasing
order of isolation.

A **process** has its own memory — safe, expensive. A **thread** shares memory with its siblings —
cheaper, and now you must coordinate access. A **coroutine** is scheduled by your program rather than
the operating system — extremely cheap, and it only helps when the work is waiting rather than
computing.

### Details

#### 1. What each costs

**Theory.** A process: its own address space, typically megabytes of overhead, and communication
requires explicit inter-process mechanisms. A thread: shares the address space, around 1MB of stack
by default, and switching costs a kernel transition. A coroutine or green thread: a few kilobytes,
switched in user space without involving the kernel.

**Example.** Rough capacities on an ordinary server: thousands of processes, tens of thousands of
threads before scheduling overhead dominates, and **millions** of coroutines (goroutines, Erlang
processes, Java virtual threads). That difference in scale is why the C10k problem was solved by
event loops and green threads rather than by more OS threads.

**Advanced.** Context switching is the hidden cost. An OS thread switch flushes CPU caches and the
translation lookaside buffer, so the new thread starts cold — typically a few microseconds of direct
cost plus indirect cache-miss cost. A coroutine switch is a function call and a stack pointer change,
so it is measured in nanoseconds. That ratio is the entire argument for user-space scheduling.

#### 2. Concurrency versus parallelism

**Theory.** **Concurrency** is structuring a program so several things are in progress at once.
**Parallelism** is actually executing several things at the same instant, which needs multiple cores.
Concurrency is a design property; parallelism is an execution property.

**Example.** Node is highly concurrent and not parallel for your JavaScript: one thread makes progress
on thousands of in-flight requests by switching whenever one waits for I/O. Go is both: goroutines
give concurrency, and the scheduler distributes them over several OS threads for parallelism.

**Advanced.** The practical consequence is which problem each solves. Concurrency solves **waiting** —
if your service spends 95% of its time waiting for the database, concurrency lets you handle far more
requests with the same CPU. Parallelism solves **computing** — if you are hashing passwords or
resizing images, only more cores help. Diagnosing which one you need is the first step in any
performance conversation (`C14`).

#### 3. Choosing a model

**Theory.** The models map onto workloads: process-per-request for isolation and simplicity;
thread-per-request for a balance; event loop or coroutines for very high I/O concurrency.

**Example.** How real systems land: PostgreSQL uses a process per connection (isolation, and the
reason connections are expensive, `DB21`). Spring MVC uses a thread per request. Node and Nginx use an
event loop. Go and modern Java use scheduled green threads, which look like thread-per-request in
code and behave like an event loop underneath.

**Advanced.** The trend is clear: give developers the simple blocking programming model, and make the
runtime schedule cheaply underneath. Goroutines, Java virtual threads and Python's asyncio all do
this in different ways. It is worth saying that async/await was a workaround for expensive threads,
and that virtual threads partially remove the need for it — a genuinely current observation.

### Interview questions

- "Define concurrency versus parallelism with a real example from your service."
- "Why can you have a million goroutines but not a million OS threads?"
- "Your service is CPU-bound. Does async help?"
- "Why is a Postgres connection expensive?"

---

## C02 · Blocking versus non-blocking I/O

`Intermediate` · Requires: `C01`, `A12` · Unlocks: `C03`, `C06`, `C07`, `C19`

### Preface

When your code reads from a socket, either it waits until data arrives (**blocking**) or it returns
immediately and you are told later (**non-blocking**).

Blocking is simple: one thread per connection, and the thread sleeps while waiting. Non-blocking is
efficient: one thread can watch thousands of connections and do work for whichever is ready. Every
high-concurrency server is built on the second.

### Details

#### 1. The I/O models

**Theory.** The classical taxonomy: **blocking** (the call waits); **non-blocking polling** (the call
returns "not ready" and you try again — wasteful); **I/O multiplexing** (`select`, `poll`, **`epoll`**
on Linux, `kqueue` on BSD — ask the kernel to tell you which of many descriptors are ready);
**signal-driven**; and **asynchronous I/O** (`io_uring` — the kernel performs the operation and
notifies you when it is done).

**Example.** `epoll` is what makes event loops work. You register thousands of sockets; one call
returns the handful that have data; you process them and go back to waiting. Its cost is proportional
to the number of **ready** descriptors, not the total — unlike `select`, which scans everything and
is why it does not scale.

**Advanced.** `io_uring` is the significant recent change: a shared ring buffer between the
application and the kernel that removes the syscall per operation, and it handles file I/O
asynchronously — something `epoll` never did properly, which is why runtimes use a thread pool for
files (`C03`). Adoption is growing in Node, Rust and database engines.

#### 2. The C10k problem

**Theory.** With thread-per-connection, ten thousand concurrent connections means ten thousand
threads — roughly 10GB of stack and a scheduler spending most of its time switching. The event-loop
model handles the same load with one thread and a few megabytes.

**Example.** This is the architectural origin of Nginx, Node and Netty. It also explains the shape of
their APIs: callbacks, then promises, then async/await — all ways to express "continue when this is
ready" without a thread waiting.

**Advanced.** The modern answer is that you can have both: virtual threads (`C06`) and goroutines
give you blocking-style code on top of non-blocking I/O underneath. The runtime parks the green
thread and reuses the OS thread. So "blocking is bad" is outdated — **blocking an OS thread** is bad,
and cheap threads make the distinction disappear for the programmer.

#### 3. Where the model leaks

**Theory.** Non-blocking runtimes assume your code never blocks. One synchronous operation ruins the
model for everything sharing that thread.

**Example.** The same mistake in three runtimes: `fs.readFileSync` in a Node request handler; a
blocking `requests.get()` inside an async Django view; a `Thread.sleep` or a synchronous JDBC call in
a WebFlux chain. In each case one operation blocks the event loop and every other in-flight request
stalls — throughput collapses and the cause is invisible in a CPU profile, because the thread is
idle, not busy.

**Advanced.** This is why "half-async" code is often slower than fully synchronous code: you pay the
complexity of the asynchronous model and keep the blocking behaviour. Either commit to non-blocking
all the way down, or use a model where blocking is cheap (virtual threads, processes). The middle is
the worst place to be.

#### 4. File I/O is different

**Theory.** `epoll` works for sockets and pipes, and historically not for regular files — a read from
disk always blocks. So event-loop runtimes fake it with a thread pool.

**Example.** Node's libuv uses a thread pool (default size four) for `fs` operations, DNS lookups via
`getaddrinfo`, some crypto, and zlib (`C03`). Heavy file work saturates those four threads and delays
everything else that uses the pool — including DNS resolution, which produces bewildering connection
latency under load.

**Advanced.** `UV_THREADPOOL_SIZE` can be raised (up to 1,024), and doing so is a reasonable fix for a
file-heavy service. The better long-term answer is `io_uring`, which makes file I/O genuinely
asynchronous. Knowing that the thread pool exists, what uses it, and how to observe its saturation is
a strong practical Node answer.

### Interview questions

- "Why does an event loop serve 10,000 connections on one thread when thread-per-connection can't?"
- "Is async always faster?"
- "Which Node operations use the thread pool and which use epoll?"
- "Why is half-async code often slower than fully synchronous code?"

---

## C03 · The Node.js event loop

`Advanced` · Requires: `C02` · Unlocks: `C04`, `C05`, `C12`, `F24`, `F28`

### Preface

Node runs your JavaScript on **one thread**. The event loop is the mechanism that keeps that thread
busy: it waits for I/O to complete, then runs the callbacks.

Two things follow, and both come up in interviews. The **ordering** of different kinds of callbacks
is well defined and frequently asked. And any synchronous work in your code blocks **every** request
in that process — so "the event loop is blocked" is the diagnosis for a whole class of production
problems.

### Details

#### 1. The phases

**Theory.** Each iteration of the loop passes through phases in order:
1. **timers** — callbacks from `setTimeout` and `setInterval` whose time has come.
2. **pending callbacks** — some system-level callbacks, such as certain TCP errors.
3. **idle / prepare** — internal.
4. **poll** — retrieve new I/O events and run their callbacks; this is where the loop blocks waiting
   if there is nothing else to do.
5. **check** — `setImmediate` callbacks.
6. **close callbacks** — `socket.on('close')` and similar.

**Example.** `setImmediate` runs in the check phase, so inside an I/O callback (poll phase) it runs
**before** a `setTimeout(fn, 0)`, which must wait for the next iteration's timer phase. At the top
level, their relative order is non-deterministic, because it depends on how long the process took to
start relative to the timer threshold.

**Advanced.** `setTimeout(fn, 0)` is really `setTimeout(fn, 1)` — the minimum is clamped — and timers
fire *at or after* their time, never before, and can be arbitrarily late if the loop is busy. A timer
that must be accurate is the wrong tool; for precise scheduling, you need a different mechanism
entirely.

#### 2. Microtasks

**Theory.** Between **every** callback, and between phases, Node drains the microtask queues:
`process.nextTick` callbacks first (a Node-specific queue with higher priority), then resolved
promise callbacks.

**Example.** The canonical ordering exercise — be able to do this from memory:

```js
console.log('1');
setTimeout(() => console.log('2'), 0);
setImmediate(() => console.log('3'));
Promise.resolve().then(() => console.log('4'));
process.nextTick(() => console.log('5'));
console.log('6');
// 1, 6, 5, 4, 2, 3
```

Synchronous code first (1, 6); then `nextTick` (5); then promises (4); then the timer (2); then
`setImmediate` (3).

**Advanced.** A recursive `process.nextTick` **starves the event loop entirely** — the queue is
drained completely before the loop proceeds, so I/O never gets a turn. A recursive promise chain has
the same effect. This is a real way to hang a server with no CPU-heavy code in sight, and it is why
`setImmediate` exists as the "yield to the loop" primitive.

#### 3. What actually blocks

**Theory.** Anything synchronous occupies the thread until it finishes. During that time no
callbacks run, no requests are served, and no timers fire.

**Example.** The usual offenders: `JSON.parse` or `JSON.stringify` on a large payload (synchronous,
and often the top CPU cost in a Node service); synchronous `fs` calls; `crypto.pbkdf2Sync` and
synchronous bcrypt; a regular expression with catastrophic backtracking (`S13`); a loop over a large
array; and `console.log` to a pipe, which is synchronous.

**Advanced.** Measure it rather than guessing: `perf_hooks.monitorEventLoopDelay()` gives a histogram
of how late the loop is running, and `performance.eventLoopUtilization()` gives the fraction of time
the loop was busy rather than waiting. Export both as metrics from every Node service (`O14`). A p99
event-loop delay above about 100ms means some request somewhere waited that long for no reason.

#### 4. The thread pool, again

**Theory.** libuv maintains a thread pool (default four) for operations the OS cannot do
asynchronously on a socket: file system calls, `dns.lookup`, `zlib`, and some `crypto` (`C02`).

**Example.** The practical consequence: heavy file or compression work saturates all four threads, and
then DNS lookups queue behind them. The symptom is connection latency to downstream services that has
nothing obviously to do with the network. Raising `UV_THREADPOOL_SIZE` helps; moving the work out of
the process helps more.

**Advanced.** Note the distinction people get wrong: network I/O does **not** use the thread pool — it
uses epoll and is genuinely asynchronous. Only the operations listed above use the pool. Being able to
state which is which is a good discriminator for real Node knowledge.

### Interview questions

- "Order the output of a snippet with `setTimeout`, `setImmediate`, `process.nextTick`, a promise and
  a sync log." (practise this — near-guaranteed)
- "Which Node operations use the thread pool and which use the OS event notification?"
- "How do you detect and alert on event loop blocking in production?"
- "How can you hang a Node server without any CPU-heavy code?"

---

## C04 · CPU-bound work in Node

`Advanced` · Requires: `C03` · Unlocks: `C13`, `F24`

### Preface

Node is excellent at waiting and poor at computing. A single CPU-heavy operation stops the entire
process from serving anyone.

The options are: move it off the main thread, move it out of the process, or break it into pieces
that yield. Which one depends on how heavy it is and how often it happens.

### Details

#### 1. Worker threads

**Theory.** `worker_threads` gives you real OS threads inside the Node process, each with its own V8
isolate and event loop. They communicate by message passing, and can share memory explicitly via
`SharedArrayBuffer`.

**Example.**

```js
const worker = new Worker('./resize.js', { workerData: { buffer } });
worker.on('message', result => { /* ... */ });
```

Suitable for: image processing, compression, cryptography, large parsing, heavy computation. Use a
**pool** of workers (`piscina`) rather than creating one per task — thread creation costs tens of
milliseconds, which dominates for short tasks.

**Advanced.** Message passing **copies** data by default (structured clone), so sending a 100MB buffer
to a worker costs a 100MB copy — sometimes more than the computation you were trying to offload. Use
`transferList` to transfer ownership of an `ArrayBuffer` without copying, or `SharedArrayBuffer` for
genuinely shared access. Getting this wrong makes workers slower than doing the work inline.

#### 2. Separate processes and services

**Theory.** For heavy or long-running work, a separate process — or a separate service in a language
better suited to it — is cleaner than a worker thread.

**Example.** The progression by weight: a few milliseconds — just do it inline; tens to hundreds of
milliseconds and frequent — worker thread pool; seconds — a background job on a worker fleet
(`F13`); heavy and continuous — a separate service, possibly in Go or Rust. Report generation and
video transcoding belong at the far end.

**Advanced.** The advantage of a queue plus separate workers over in-process threads is operational:
independent scaling, independent failure, visibility into backlog, retries, and the ability to give
the work its own resource limits. The advantage of worker threads is latency — no queue round trip.
Choose by whether the user is waiting.

#### 3. Cluster and multiple containers

**Theory.** `cluster` forks several Node processes sharing a listening socket, so one machine uses
several cores. The alternative is running more containers and letting the orchestrator schedule them.

**Example.** More containers is usually right in Kubernetes: the platform already handles scheduling,
you get independent restarts and per-pod metrics, and one crashed process does not take others with
it. `cluster` wins when there is significant fixed per-process overhead you would rather not
duplicate — a large in-memory cache — or on a big machine where per-pod overhead matters (`F28`).

**Advanced.** `cluster` distributes connections either by the operating system (the default on Linux,
which can be uneven) or round-robin by the primary process. Neither accounts for how busy each worker
is, so a worker stuck on a heavy request keeps receiving new ones. That is a real weakness compared
with an external load balancer that can use least-connections (`M07`).

#### 4. Chunking and yielding

**Theory.** If the work can be split, process it in pieces and yield to the event loop between them,
so other requests make progress.

**Example.**

```js
async function processAll(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
    if (i % 100 === 0) await new Promise(r => setImmediate(r));   // yield
  }
}
```

This turns a 2-second block into 2 seconds of work interleaved with other requests. Total time is
slightly longer; p99 latency for everyone else is dramatically better.

**Advanced.** `setImmediate` is the correct yield primitive because it schedules for the **check**
phase, after the poll phase has had a chance to process I/O. `process.nextTick` does not yield at all
— it runs before the loop continues, so a loop yielding with `nextTick` still starves I/O (`C03`).
Knowing which of the two actually yields is a precise, high-signal detail.

### Interview questions

- "Report generation takes 8 seconds of CPU. Where do you put it and why?"
- "When is a worker thread slower than doing the work inline?"
- "`cluster` or more containers?"
- "How do you yield to the event loop, and why not `process.nextTick`?"

---

## C05 · Promises and async/await semantics

`Intermediate` · Requires: `C03` · Unlocks: `C16`, `C17`

### Preface

`async`/`await` makes asynchronous code look sequential, which is its strength and its trap: code
that *looks* like a loop over a list is a loop over N network round trips.

The two skills that matter are recognising accidental sequencing, and bounding concurrency so you do
not fire a thousand simultaneous requests at a service that can handle fifty.

### Details

#### 1. Accidental sequencing

**Theory.** `await` inside a loop waits for each iteration before starting the next. N items means N
times the latency.

**Example.**

```js
// 500 items x 50ms = 25 seconds
for (const id of ids) results.push(await fetchOne(id));

// all at once: 50ms — and 500 simultaneous connections
const results = await Promise.all(ids.map(fetchOne));

// bounded: fast and kind to the downstream
const limit = pLimit(20);
const results = await Promise.all(ids.map(id => limit(() => fetchOne(id))));
```

**Advanced.** The middle option is the one people reach for and it is also wrong at scale: 500
simultaneous requests can exhaust your connection pool, overwhelm the downstream service, and trigger
its rate limiter — turning a latency problem into an outage (`M09`). Always bound concurrency. The
better answer still is a **batch endpoint** so 500 items become one request (`C17`).

#### 2. The combinators

**Theory.** `Promise.all` rejects as soon as any input rejects, and the others keep running.
`Promise.allSettled` waits for all and reports each outcome. `Promise.race` settles with the first to
settle, including rejections. `Promise.any` settles with the first **fulfilment**.

**Example.** Use `allSettled` when partial success is acceptable — an aggregation endpoint where one
failed section degrades rather than fails (`M06`). Use `all` when any failure invalidates the whole
operation. Use `race` for a timeout wrapper, and `any` for "whichever replica answers first".

**Advanced.** With `Promise.all`, a rejection means the other promises continue in the background;
if one later rejects with nobody listening, you get an unhandled rejection that can crash the
process. And the promises started executing when created, not when awaited — so
`const p = fetch(...)` has already begun. That eagerness surprises people coming from lazy
abstractions.

#### 3. Error handling and cancellation

**Theory.** An unhandled promise rejection terminates the process in modern Node by default.
`async` callbacks passed to APIs that ignore return values (`forEach`, some event emitters) swallow
errors entirely.

**Example.**

```js
items.forEach(async item => { await save(item); });   // errors vanish; the function returns before saves finish
await Promise.all(items.map(item => save(item)));     // correct
```

For cancellation, use `AbortController` — pass its `signal` to `fetch`, to timers, and to your own
functions. When a client disconnects, aborting the downstream work stops you computing an answer
nobody wants (`M04`).

**Advanced.** Cancellation is cooperative: your code must check the signal or pass it to something
that does. A CPU-bound loop ignores it entirely. And `AbortSignal.timeout(ms)` is the clean modern way
to express a request timeout, replacing the `Promise.race` with a timer pattern — which leaked timers
unless you cleared them.

#### 4. Async context

**Theory.** Because execution jumps between callbacks, there is no call stack tying related
operations together. `AsyncLocalStorage` provides a context that follows the asynchronous chain.

**Example.** It is how request ids and tenant context reach every log line without being passed
explicitly (`F03`, `F08`). It works across `await`, promises and most callback APIs, and it can be
lost across some manually-detached callbacks and some native modules.

**Advanced.** There is a small performance cost (the runtime tracks context transitions), which was
significant in older Node versions and is modest now. Async stack traces (`--async-stack-traces`, on
by default in recent versions) reconstruct a useful stack across `await` boundaries, which is
invaluable for debugging — worth knowing exists, because the default assumption is that async traces
are useless.

### Interview questions

- "`for (const id of ids) await fetchOne(id)` over 500 ids. What is wrong and what are the two fixes?"
- "Why is unbounded `Promise.all` also wrong?"
- "How do you cancel an in-flight HTTP call when the client disconnects?"
- "`items.forEach(async ...)` — what happens to the errors?"

---

## C06 · JVM concurrency

`Advanced` · Requires: `C01`, `C02` · Unlocks: `C08`, `C09`, `C12`, `F22`

### Preface

The JVM's traditional model is one platform thread per request, which is simple to write and limited
by how many OS threads you can afford.

Java 21's **virtual threads** change that: threads become cheap, so you can keep blocking code and
still handle enormous concurrency. For an interview, the valuable knowledge is how to size a
traditional thread pool and what virtual threads do and do not solve.

### Details

#### 1. Platform threads and pools

**Theory.** A platform thread maps to an OS thread: about 1MB of stack, a kernel-level context
switch, and a practical limit in the low thousands. So you pool them and queue work.

**Example.** Sizing, which is a classic question:
- **CPU-bound**: `threads ≈ cores + 1`. More threads only add switching.
- **I/O-bound**: `threads ≈ cores x (1 + waitTime / serviceTime)`. If a request waits 100ms on the
  database and computes for 10ms, that is `cores x 11`.

The second formula is Little's Law in disguise (`C14`), and deriving it rather than quoting a number
is what impresses.

**Advanced.** `ThreadPoolExecutor` has a subtlety worth knowing: with an **unbounded** queue, the
maximum pool size is never reached, because work is queued rather than causing new threads to be
created. An unbounded queue also hides overload until memory runs out (`M30`). Use a bounded queue
with an explicit rejection policy, so the pool behaves as configured and load shedding is visible.

#### 2. Virtual threads

**Theory.** A virtual thread is scheduled by the JVM onto a small pool of carrier threads. When it
blocks on I/O, the JVM **unmounts** it from the carrier and runs something else. Creating millions is
feasible; blocking is no longer a mistake.

**Example.** For a Spring Boot service, `spring.threads.virtual.enabled=true` makes each request run
on a virtual thread. Blocking JDBC calls no longer consume a scarce resource, so a service that could
handle 200 concurrent requests handles thousands — with no code change and with stack traces and
debuggers that still work (`F22`).

**Advanced.** Two important limits. **Pinning**: inside a `synchronized` block, a virtual thread
cannot unmount, so it holds its carrier thread while blocked — which can deadlock a small carrier pool.
Replace `synchronized` with `ReentrantLock` in code that blocks. And virtual threads do nothing for
**CPU-bound** work: you still have only as many cores as you have. They solve waiting, not computing
(`C01`).

#### 3. CompletableFuture and reactive

**Theory.** Before virtual threads, non-blocking on the JVM meant `CompletableFuture` chains or a
reactive library (Project Reactor, RxJava) with `Mono` and `Flux`.

**Example.** Reactive gives high concurrency with few threads, plus built-in backpressure (`C16`) and
composable operators. The costs are a steep learning curve, stack traces that do not show the logical
flow, debuggers that are much less useful, and the requirement that the **entire** chain be
non-blocking — one blocking call ruins it (`C02`).

**Advanced.** The current guidance for a new I/O-bound service is virtual threads with blocking code:
simpler, debuggable, and comparable throughput. Reactive remains justified for genuine streaming with
backpressure, and for teams already fluent in it. Being able to give that recommendation with reasons
is a strong, current answer (`F22`).

#### 4. ThreadLocal and context

**Theory.** `ThreadLocal` stores per-thread state, and is how logging context (MDC), security context
and transaction context are traditionally propagated in Spring.

**Example.** It breaks whenever work moves to another thread: a thread pool, `@Async`, a
`CompletableFuture` continuation, or reactive code. Symptoms are missing or **wrong** user ids in log
lines — wrong being worse than missing, because a pooled thread retains the previous request's value
if it is not cleared.

**Advanced.** Virtual threads make `ThreadLocal` usable again for request-scoped data, since each
request has its own thread — though a `ThreadLocal` holding a large object across millions of virtual
threads is now a memory concern rather than a correctness one. `ScopedValue` (a newer JDK feature) is
the intended replacement: immutable, explicitly scoped, and cheaper. The equivalent problem and
solution in Node is `AsyncLocalStorage` (`C05`).

### Interview questions

- "How do you size a thread pool?"
- "What problem do virtual threads solve, and what do they not solve?"
- "What is pinning and how do you avoid it?"
- "Why does your log line show the wrong user id in async code?"

---

## C07 · Python concurrency

`Advanced` · Requires: `C01`, `C02` · Unlocks: `C12`, `F21`

### Preface

Python's concurrency story is dominated by one fact: the **Global Interpreter Lock** means only one
thread executes Python bytecode at a time, per process.

So threads help with waiting and not with computing. For CPU work you use processes; for I/O
concurrency you use threads or `asyncio`; and for web serving you run many worker processes.

### Details

#### 1. The GIL

**Theory.** CPython uses reference counting for memory management, and making that thread-safe
without a global lock would require fine-grained locking everywhere, which historically made
single-threaded code slower. The GIL is released during I/O operations and inside C extensions that
opt out, which is why threads still help for I/O.

**Example.** The consequence: two threads computing sums take the same wall-clock time as one thread
doing both (plus switching overhead). Two threads each waiting on a database do overlap, because the
GIL is released while waiting. So `ThreadPoolExecutor` is useful for I/O and useless for computation.

**Advanced.** NumPy, pandas and similar libraries release the GIL inside their C routines, so
numerical work **does** parallelise across threads — which is why "Python cannot use multiple cores"
is too strong a statement. And CPython 3.13 introduced an optional free-threaded build without the
GIL; it is experimental, costs single-threaded performance, and requires extensions to be updated. It
is the direction of travel and not yet the default.

#### 2. Multiprocessing

**Theory.** For CPU-bound work, use processes: each has its own interpreter and its own GIL.
Communication requires serialisation (`pickle`), which has a cost.

**Example.** `ProcessPoolExecutor` for parallel computation. The cost to watch is data transfer:
sending a large DataFrame to each worker can exceed the computation. Mitigations: `fork` on Linux
shares memory copy-on-write so read-only data need not be copied; or use shared memory
(`multiprocessing.shared_memory`) explicitly.

**Advanced.** `fork` is the default start method on Linux and is **unsafe in a process with threads**
— the child inherits a copy of the memory including locks held by threads that do not exist in the
child, which produces deadlocks. This bites when forking from inside a web server that already uses
threads. Python 3.14 changes the default to `forkserver` on Linux for this reason; knowing that this
hazard exists is a good, specific detail.

#### 3. asyncio

**Theory.** `asyncio` is an event loop like Node's, with explicit `async def` and `await`. One thread
handles many concurrent I/O operations.

**Example.** The same rule as Node applies (`C02`): everything in the path must be non-blocking. A
synchronous `requests.get()` or a blocking database driver inside a coroutine blocks the loop and
destroys the concurrency you were trying to gain. Use `httpx`/`aiohttp`, `asyncpg`, and
`asyncio.to_thread()` for unavoidable blocking calls.

**Advanced.** Django's async support is partial: async views work, and the ORM is synchronous
underneath, with async methods (`aget`, `acreate`) wrapping it in a thread via `sync_to_async`. So an
async Django view doing ORM work is not truly async — it is using a thread pool. Being precise about
this is the strongest Django-concurrency answer (`F21`).

#### 4. Serving Python in production

**Theory.** Because of the GIL, a Python web application scales with **processes**. WSGI servers run
several worker processes; ASGI servers do the same for async applications.

**Example.** Configurations and when to use them:
- **gunicorn sync workers** — one request per worker at a time; `workers ≈ 2 x cores + 1`; simple and
  memory-hungry.
- **gunicorn + gevent** — monkey-patches the standard library to make I/O cooperative; high
  concurrency for blocking code, with occasional surprises from libraries that do not cooperate.
- **uvicorn (ASGI)** — for genuinely async applications.

**Advanced.** Memory per worker matters: each worker is a full interpreter with its own copies of
loaded modules, so a Django application at 200MB per worker with 9 workers needs 1.8GB. Preloading the
application (`--preload`) lets `fork` share memory copy-on-write and cuts this substantially — at the
cost of the fork-safety hazard above. That trade-off is a real production decision.

### Interview questions

- "Why doesn't adding threads speed up your Python CPU work, and what do you do instead?"
- "You called `requests.get()` inside an async Django view. What happened?"
- "How many gunicorn workers, and why?"
- "Is Django's async ORM genuinely async?"

---

## C08 · Memory model, visibility and atomics

`Expert` · Requires: `C06` · Unlocks: `C09`, `C10`

### Preface

On a multi-core machine, each core has its own caches and the compiler and CPU reorder instructions
for speed. So a value written by one thread may not be visible to another, and operations may appear
to happen in a different order than written.

A memory model defines the rules for when one thread is guaranteed to see another's writes. This is
JVM, Go and Rust territory; Node's single thread makes it irrelevant until you use
`SharedArrayBuffer`.

### Details

#### 1. Visibility and reordering

**Theory.** A write may sit in a core's store buffer or cache and not be visible to other cores.
Compilers and CPUs also reorder instructions when the reordering is not observable *by that thread*.
A memory model specifies the synchronisation points that force visibility and ordering.

**Example.** The classic broken pattern:

```java
boolean running = true;     // not volatile
// thread A: while (running) { ... }
// thread B: running = false;
```

Thread A may loop forever: the compiler is entitled to hoist the read out of the loop, since nothing
in that thread changes it. Marking `running` as `volatile` forces a read from main memory each time
and forbids the reordering.

**Advanced.** `volatile` gives **visibility and ordering**, not **atomicity**. `volatile int count;
count++` is still broken, because the increment is read-modify-write — three operations, and two
threads can interleave. Atomicity needs `AtomicInteger` (compare-and-swap) or a lock. Confusing the
two is the most common misunderstanding in this area.

#### 2. Happens-before

**Theory.** The memory model is expressed as a **happens-before** relation: if action A happens-before
action B, then A's effects are visible to B. Releasing a lock happens-before acquiring it; writing a
volatile happens-before reading it; starting a thread happens-before anything it does; anything a
thread does happens-before another thread sees it terminate.

**Example.** This is what makes correct locking work at all: everything done before releasing a lock
is visible to whoever acquires it next. It is also why "I used a lock for the write but read without
one" is broken — the reader has no happens-before edge and may see a stale value.

**Advanced.** **Safe publication** is the practical rule that follows: an object shared between
threads must be published through a synchronisation point — a volatile field, a final field set in
the constructor, a concurrent collection, or under a lock. Publishing via a plain field means another
thread can observe a partially-constructed object. This is why double-checked locking needs
`volatile` to be correct.

#### 3. Atomics and compare-and-swap

**Theory.** CPUs provide an atomic compare-and-swap instruction: "if this location holds X, set it to
Y, atomically". All lock-free algorithms build on it.

**Example.** `AtomicInteger.incrementAndGet()` loops: read the value, compute the new one, attempt a
CAS, and retry if another thread won. Under low contention this is much faster than a lock, because
there is no blocking and no context switch.

**Advanced.** Under **high** contention CAS degrades: many threads retry repeatedly, burning CPU
without progress — this is why `LongAdder` exists, spreading increments across several internal cells
and summing on read (the same sharded-counter idea as `M32`). The crossover between "CAS is faster"
and "a lock is faster" is a real, measurable point, and knowing it exists is more useful than the
number.

#### 4. False sharing

**Theory.** Caches operate on **cache lines** of 64 bytes. Two variables in the same line are
invalidated together, so two threads updating adjacent independent variables force each other's
caches to reload constantly.

**Example.** An array of per-thread counters is the classic case: logically independent, physically
adjacent, and therefore slow. The fix is padding so each counter occupies its own cache line —
Java's `@Contended` annotation does this.

**Advanced.** This is mostly relevant when writing high-performance infrastructure — queues,
counters, ring buffers — rather than application code. It is a good example of the general lesson
that at this level, **physical layout affects correctness-adjacent performance** in ways the source
code does not show, which is worth a sentence if the conversation reaches that depth.

### Interview questions

- "Two threads increment a shared counter with `volatile`. Is it correct?"
- "What does `volatile` actually guarantee?"
- "Explain happens-before in one sentence."
- "When does compare-and-swap perform worse than a lock?"
