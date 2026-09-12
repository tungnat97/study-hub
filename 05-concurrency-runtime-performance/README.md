[← back to the index](../README.md)

# Field 5 — Concurrency, Runtimes & Performance

Where "senior" is actually tested for a Node engineer: they will ask about the event loop, and then
whether you understand the *other* models well enough to move stacks. Legend: `B` `I` `A` `X`.

---

## Nodes

#### C01 · Process, thread, coroutine
`B` · Requires: — · Unlocks: C02, C06, C07, C09
- Key: process (own address space, expensive, isolated) vs thread (shared memory, cheap-ish, needs
  synchronisation) vs coroutine/green thread (user-space scheduling, thousands-to-millions, no
  preemption unless the runtime provides it); context switch cost; concurrency ≠ parallelism
  (Rob Pike's line — be ready to define both).
- Q: "Define concurrency vs parallelism with a real example from your service."

#### C02 · Blocking vs non-blocking I/O
`I` · Requires: C01, A12 · Unlocks: C03, C06, C07, C19
- Key: the kernel-level picture — blocking, non-blocking polling, I/O multiplexing (`select`/`poll`/
  **`epoll`**/`kqueue`/IOCP), signal-driven, async I/O (`io_uring`); thread-per-connection (C10k
  problem) vs event loop; why "async" only helps I/O-bound work; syscalls and user/kernel transitions.
- Q: "Why does an event loop serve 10k connections on one thread when a thread-per-connection server
  can't?"
- Q: "Is async always faster?" (no — CPU-bound work and low concurrency get no benefit and pay
  complexity).

#### C03 · The Node.js event loop
`A` · Requires: C02 · Unlocks: C04, C05, C12, F24, F28
- Key: libuv phases in order — **timers → pending callbacks → idle/prepare → poll → check
  (setImmediate) → close callbacks**, with the **microtask queue** (`process.nextTick` first, then
  promise jobs) drained between every phase and between each callback; the loop is single-threaded
  for *your JS*, but libuv has a **thread pool (default 4, `UV_THREADPOOL_SIZE`)** used by fs, dns
  (`lookup`), zlib and crypto — not by network I/O, which is epoll; event loop lag as the key metric;
  one long synchronous function stalls every request in the process.
- Q: "Order the output of a snippet with `setTimeout(0)`, `setImmediate`, `process.nextTick`, a
  resolved promise and a sync log." (practise this — near-guaranteed question)
- Q: "Which Node operations use the thread pool and which use the OS event notification?"
- Q: "How do you detect and alert on event loop blocking in production?"
  (`perf_hooks.monitorEventLoopDelay`, `eventLoopUtilization`).

#### C04 · CPU-bound work in Node
`A` · Requires: C03 · Unlocks: C13, F24
- Key: options — move it to a worker (`worker_threads`, `SharedArrayBuffer`/transfer for zero-copy),
  a separate service/language, a queue + worker fleet, or chunk it with yields; `cluster`/pm2 vs
  multiple containers; native addons; why JSON.parse/stringify of a 50MB payload is a production
  incident; `crypto.pbkdf2`/bcrypt async variants using the thread pool.
- Q: "Report generation takes 8 seconds of CPU. Where do you put it and why?"

#### C05 · Promises & async/await semantics
`I` · Requires: C03 · Unlocks: C16, C17
- Key: microtask scheduling, `await` = "yield to the loop", sequential awaits in a loop as an
  accidental N x latency bug, `Promise.all` vs `allSettled` vs `race` vs `any`, concurrency limiting
  (`p-limit`, semaphore) to avoid hammering a downstream or exhausting the pool, unhandled rejection
  behaviour (process crash in modern Node), errors in `forEach(async ...)`, `try/finally` for
  cleanup, `AbortController`/`AbortSignal` for cancellation and timeouts, async stack traces.
- Q: "`for (const id of ids) await fetchOne(id)` over 500 ids. What's wrong and what are the two
  fixes?" (batch/parallel with a concurrency cap — and why unbounded `Promise.all` is also wrong).
- Q: "How do you cancel an in-flight HTTP call when the client disconnects?"

#### C06 · JVM concurrency
`A` · Requires: C01, C02 · Unlocks: C08, C09, C12, F22
- Key: platform threads = OS threads (~1MB stack, thousands max) → thread pools and `ExecutorService`,
  sizing (CPU-bound ≈ cores+1; I/O-bound ≈ cores x (1 + wait/service) — Little's Law again),
  `CompletableFuture`, `ForkJoinPool`/work stealing, `ThreadLocal` (and its leak in pooled threads);
  **virtual threads (Loom, Java 21)** — cheap, mounted/unmounted on carriers, blocking is fine again,
  pinning by `synchronized` blocks and native frames; structured concurrency.
- Q: "How do you size a thread pool?"
- Q: "What problem do virtual threads solve, and what do they *not* solve?" (I/O concurrency, not
  CPU parallelism; and they don't fix `ThreadLocal`-heavy or `synchronized`-pinning code).

#### C07 · Python concurrency
`A` · Requires: C01, C02 · Unlocks: C12, F21
- Key: the **GIL** — one thread executes Python bytecode at a time, so threads help I/O-bound work
  only; multiprocessing for CPU (with IPC/pickling costs); `asyncio` event loop (same model as Node,
  explicit `async def`), blocking a coroutine blocks everything (`run_in_executor`/`to_thread`);
  WSGI (sync, worker processes) vs **ASGI** (async, uvicorn); gunicorn worker types (sync/gthread/
  gevent/uvicorn) and worker count ≈ 2 x cores + 1; free-threaded CPython (3.13+ no-GIL builds) as
  the direction of travel.
- Q: "Why doesn't adding threads speed up your Python CPU work, and what do you do instead?"
- Q: "You called `requests.get()` inside an async Django view. What happened?"

#### C08 · Memory model, visibility & atomics
`X` · Requires: C06 · Unlocks: C09, C10
- Key: instruction reordering and cache coherency, happens-before, `volatile` (visibility, not
  atomicity), atomic classes/CAS, false sharing, memory barriers, double-checked locking done right,
  immutability as the simplest correct answer. Mostly a JVM/Go/Rust topic — Node's single thread
  makes it moot until `SharedArrayBuffer`.
- Q: "Two threads increment a shared counter with `volatile`. Is it correct?" (no — needs atomic/CAS)

#### C09 · Locks & coordination primitives
`A` · Requires: C01, C06, C08, DB09 · Unlocks: C10, C11
- Key: mutex, reentrant lock, read-write lock, semaphore, condition variable, latch/barrier;
  deadlock's four conditions and prevention (consistent ordering, timeouts, lock-free);
  livelock and starvation; lock granularity vs contention; Amdahl's law — the serialised section
  caps your speedup.
- Q: "Name the four deadlock conditions and which one you break in practice."

#### C10 · Lock-free, immutability, actors
`X` · Requires: C08, C09 · Unlocks: C11
- Key: CAS loops and the ABA problem, lock-free vs wait-free, why lock-free is rarely worth
  hand-writing; share-nothing designs; actor model (Erlang/Akka) and message passing; Go channels
  ("share memory by communicating"); single-writer principle and per-key serialisation (a queue per
  entity is often the pragmatic distributed answer).
- Q: "Remove the lock from a hot counter without breaking correctness." (sharded counters/atomics,
  then aggregate).

#### C11 · Concurrency across machines
`A` · Requires: C09, C10, Q08, M21 · Unlocks: SD10
- Key: a distributed lock is not a mutex — you need a lease, a **fencing token**, and the
  understanding that a GC pause can make a lock-holder lose its lease while still running (the
  Redlock debate: Kleppmann vs antirez — know both positions). Prefer designs that don't need a
  distributed lock: partition by key so one owner exists, conditional writes/CAS in the DB,
  optimistic concurrency, single-writer per aggregate.
- Q: "Is Redis a safe distributed lock?" (for efficiency yes, for correctness no — use a fencing
  token and the DB's own conditional write for correctness).
- Q: "Two workers process the same job. Where did your lock fail and how do you make it not matter?"

#### C12 · Memory management & GC
`A` · Requires: C03, C06, C07 · Unlocks: C13, O06
- Key: stack vs heap, generational hypothesis; **V8** — young (scavenger, semi-space) and old
  (mark-sweep-compact, incremental/concurrent), `--max-old-space-size` must sit under the container
  limit or you OOMKill before GC pressure shows; **JVM** — G1 vs ZGC/Shenandoah, pause targets, heap
  sizing vs container limits (`MaxRAMPercentage`), allocation rate as the real driver; Python
  refcounting + cycle collector; memory **leak** patterns (unbounded caches/maps, listeners not
  removed, closures holding request data, `ThreadLocal` in pools, global arrays); heap snapshots and
  diffing; RSS vs heap vs native memory (buffers, threads).
- Q: "Node pod is OOMKilled at 512MB but the heap shows 200MB. Explain." (native/Buffer/RSS,
  fragmentation, `max-old-space-size` mis-set).
- Q: "How do you find a memory leak in production?" (three heap snapshots, compare retained sets;
  correlate with a deploy; bound every cache).

#### C13 · Profiling & benchmarking
`A` · Requires: C04, C12 · Unlocks: C14, O17
- Key: measure before optimising; CPU profile vs **flame graph** (on-CPU) vs off-CPU analysis
  (waiting — usually where backend latency lives); `--prof`/`--cpu-prof`/clinic.js/0x, async-profiler
  /JFR, py-spy; sampling vs instrumentation overhead; microbenchmark traps (JIT warm-up, dead code
  elimination, GC noise, unrealistic data); profiling in production with continuous profilers.
- Q: "The endpoint is slow but the CPU profile is flat. Where is the time?" (waiting on I/O, locks,
  the pool, or GC — go to traces and off-CPU).

#### C14 · Latency, percentiles, queueing theory
`A` · Requires: C13, DB21, M07 · Unlocks: C15, C16, O14, SD02
- Key: p50/p95/**p99**/p99.9 and why averages lie; **you cannot average percentiles** across
  instances (use histograms); tail latency amplification (a request fanning out to 100 services hits
  p99 on *some* of them nearly always); **Little's Law** L = λW (concurrency = arrival rate x latency
  — derive pool sizes from it); utilisation vs latency (the hockey stick past ~70-80%); queueing
  delay vs service time; coordinated omission in load tests.
- Q: "Each service has a 10ms p99. You call 50 of them in parallel. What's your p99?"
- Q: "Your box is at 80% CPU and latency doubled. Why non-linear?"
- Q: "You need to handle 500 rps with 200ms latency. How many concurrent workers?" (Little's Law:
  500 x 0.2 = 100).

#### C15 · Load testing & capacity planning
`I` · Requires: C14 · Unlocks: SD02, O18
- Key: k6/Gatling/Locust/wrk; open vs closed workload models (closed models hide overload —
  coordinated omission); realistic data and cache-cold vs cache-warm; ramp to find the knee, not just
  a pass/fail; soak tests for leaks; test the dependency's limits too (you will DoS your own staging
  DB); headroom targets and autoscaling thresholds derived from the knee.
- Q: "Design a load test for the checkout flow. What do you measure and when do you stop?"

#### C16 · Backpressure end to end
`A` · Requires: C05, C14, M30 · Unlocks: F16, Q15
- Key: every queue must be bounded; Node stream backpressure (`write()` returning false, `pipeline`,
  async iterators), reactive streams `request(n)`, HTTP/2 flow control, consumer-driven pull (Kafka)
  vs push (RabbitMQ prefetch); shed load at the edge rather than queueing internally; propagate
  failure fast rather than buffering.
- Q: "You pipe a DB cursor to an HTTP response and memory explodes. What did you skip?"

#### C17 · Network-level efficiency patterns
`I` · Requires: C05 · Unlocks: SD05
- Key: connection reuse/keep-alive and pooling (avoid a TCP+TLS handshake per call), batching and
  pipelining (Redis pipeline, Kafka batching, GraphQL DataLoader), request coalescing/singleflight,
  parallel fan-out with a concurrency cap, caching, compression, moving the computation to the data
  (do the filter in SQL, not in JS).
- Q: "Your service makes 200 sequential internal calls per request. Give three fixes in order of
  impact."

#### C18 · Data structures & algorithmic cost in backend code
`I` · Requires: — · Unlocks: C13, DB33
- Key: the small set that actually shows up — hash map vs sorted structure, O(n²) hidden in nested
  loops or `array.includes` inside a loop (use a Set), sorting cost, LRU implementation, heaps for
  top-k and schedulers, tries/prefix search, ring buffers, bit sets, string concatenation in loops;
  big-O of the thing you just wrote over a 100k-row result.
- Q: "Here's a service function over 50k records; find the accidental O(n²)."
- Q: "Implement an LRU cache." (still asked — hash map + doubly linked list, O(1)).

#### C19 · Streaming & zero-copy
`A` · Requires: C02, A20 · Unlocks: F17
- Key: streams vs buffering whole payloads; `pipeline()` with error propagation; `sendfile`/zero-copy
  (why Kafka is fast); Buffer pooling and `SharedArrayBuffer`; chunked parsing (NDJSON, CSV);
  transforming without materialising; the memory-per-connection budget.
- Q: "Export 10M rows to CSV over HTTP without OOM. Design it."

---

## Topological order (study waves)

```
Wave 0  C01  C18
Wave 1  C02
Wave 2  C03  C06  C07  C19
Wave 3  C04  C05  C08  C12
Wave 4  C09  C13
Wave 5  C10  C14  C17
Wave 6  C11  C15  C16
```

Cross-field parents: `A12` TCP, `A20` payloads, `DB09` locking, `DB21` pooling, `M07` load
balancing, `M21` clocks, `M30` backpressure, `Q08` Redis atomics.

**The four you must nail:** the event loop output-ordering question (C03), Little's Law + percentile
reasoning (C14), distributed locks and fencing (C11), and "how would you find this in production"
for a leak or a latency spike (C12/C13).
