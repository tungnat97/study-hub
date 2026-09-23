[← back to the field index](README.md)

# Concurrency, Runtimes & Performance · Part 4 — Real-life production problems

Nodes `C01`–`C19`, applied.

Parts 1–3 teach the models. This part is about what they look like at 3am: a p99 that doubles while
CPU sits at 20%, a heap that grows 3 MB an hour, a lock that is "held" by a process that was frozen
for eleven seconds. Interviewers at senior level narrate a symptom and listen for whether you have
actually been paged for it — whether you reach for the non-obvious cause, the right tool, and the fix
that does not appear in the documentation.

**How to use this file.** Read the pre-knowledge once, properly. Then work the questions level by
level: read the scenario, answer it **out loud** as if in the room (hypothesis, how you would prove
it, what you would change, what you would watch afterwards), and only then read the direction line.
The direction names the insight and points back to the pre-knowledge section (`§n`) and node IDs.
If you cannot say *how you would confirm it* in production, you have not answered the question.

---

## Pre-knowledge

### 1. The Node event loop under load (`C03`, `C04`)

**One thread runs all your JavaScript.** Every request's callbacks, every `JSON.parse`, every regex,
every template render, every `Array.sort` shares one thread. If one callback takes 200 ms, every other
in-flight request on that process waits 200 ms, whatever its own work costs. That is why a single slow
endpoint in Node raises the p99 of *every* endpoint on the same process — the latency is shared, not
isolated as it is in a thread-per-request server.

**Measuring event-loop lag.**
- `perf_hooks.monitorEventLoopDelay({ resolution: 10 })` returns an HDR histogram of how late a
  periodic timer fired. Export `p50`, `p99` and `max` (in nanoseconds — divide by 1e6) as metrics.
  Healthy services sit at a p99 of a few ms; a p99 above ~50 ms means something is holding the thread.
- `performance.eventLoopUtilization()` (ELU) gives the fraction of time the loop was busy rather than
  idle in the poll phase. ELU near 1.0 means the process is saturated even if container CPU looks
  modest (one core busy out of four is "25% CPU" on the dashboard and 100% on the loop).
- The cheap homemade version: `setInterval` every 100 ms and record `Date.now() - expected`. Crude, but
  it is how most people first see the problem.
- ELU is the right autoscaling and load-shedding signal for Node; CPU% is not, because a Node process
  can only ever use one core for JavaScript.

**The usual blockers, in rough order of how often they bite:**
1. `JSON.parse` / `JSON.stringify` of large payloads. Roughly 1 ms per 1–2 MB is a fair rule of
   thumb on modern hardware, so a 50 MB body or a cached blob stringified for Redis is tens to
   hundreds of ms of blocked loop. Logging a huge object with `JSON.stringify` in a hot path is a
   classic. Fixes: do not build the giant object (paginate, stream with a streaming parser such as
   `stream-json`, or NDJSON), move it to a worker thread, or store pre-serialised bytes.
2. Synchronous APIs left in: `fs.readFileSync`, `crypto.pbkdf2Sync`, `zlib.gzipSync`,
   `child_process.execSync`, `bcrypt.hashSync`. Find them with `node --trace-sync-io` (prints a stack
   trace for each sync I/O call after the first tick).
3. Catastrophic regex backtracking (ReDoS): `/(a+)+$/`-shaped patterns on attacker-controlled input
   can take seconds (`§16`).
4. Big synchronous loops: sorting 500k items, building large arrays, deep clones (`structuredClone`
   of a big object is synchronous too), Lodash `cloneDeep` on request state.
5. Template, markdown and React server-side rendering, which are CPU work.
6. Microtask starvation: a promise chain that keeps resolving (or recursive `process.nextTick`)
   never yields to the poll phase, so I/O callbacks and timers starve (`C05`).

**Yielding.** Chunk long work and yield with `setImmediate` (check phase, lets I/O run), never with
`process.nextTick` (runs before the loop continues, so it yields nothing). Or move it off the loop to
`worker_threads` via a pool such as Piscina — do not spawn a worker per request, since start-up costs
tens of ms and ~10 MB each — or out of process entirely.

**Timers are lower bounds.** `setTimeout(fn, 100)` fires *at least* 100 ms later. Under a blocked
loop, timers you rely on for correctness (request timeouts, lease renewals, heartbeats) fire late —
a subtle source of "the timeout didn't work" and of lost distributed locks (`§13`). Worse, a timeout
measured from the moment the callback finally runs will attribute the wait to the downstream.

**Health checks share the loop.** A liveness probe served by the same process answers late when the
loop is blocked, so Kubernetes kills a process that was merely busy, which moves its load onto the
others, which then block too — a restart cascade. Keep liveness trivially cheap and tolerant (generous
`timeoutSeconds` and `failureThreshold`), and put saturation signals in readiness instead.

### 2. The libuv threadpool (`C02`, `C03`)

Network sockets use the OS's non-blocking mechanism (epoll/kqueue/IOCP) and do not touch the pool.
Everything without a good async OS API goes to the **libuv threadpool**:
- all `fs.*` operations (except `fs.watch` and pipes),
- `dns.lookup` — which is what `http.request`, `net.connect` and nearly every HTTP client use by
  default, because it honours `/etc/hosts` and `nsswitch` by calling `getaddrinfo`,
- `crypto.pbkdf2`, `crypto.scrypt`, async `crypto.randomBytes`, `crypto.generateKeyPair`,
- `zlib` async functions (`gzip`, `deflate`, and therefore compression middleware),
- native addons that choose to use it (`bcrypt`, `sharp` and other image libraries).

**Default size is 4** (`UV_THREADPOOL_SIZE`, maximum 1024). It is process-wide and must be set in the
environment **before** the pool is first used — assigning `process.env.UV_THREADPOOL_SIZE` inside the
app after any fs call has no effect. It is not derived from the number of cores.

**The contention failure mode.** Four slow `getaddrinfo` calls (a resolver timing out at 5 s) or four
`bcrypt` hashes occupy all four threads; every `fs.readFile`, every compression and every *other
outbound HTTP call's DNS lookup* queues behind them. Symptom: event-loop lag is fine, CPU is low, yet
outbound calls and file reads are slow. It is invisible to event-loop metrics because the loop is
idle — it is waiting.

**Unconventional fixes.**
- Raise `UV_THREADPOOL_SIZE` (16–64) for pool-heavy workloads — but it will not fix a hung resolver,
  only postpone the pile-up.
- Stop hitting DNS per request: keep-alive agents reuse sockets (`§15`); cache lookups
  (`cacheable-lookup`); or use `dns.resolve*`, which uses c-ares off the pool but ignores `/etc/hosts`.
- In Kubernetes, `ndots:5` in `resolv.conf` means `api.example.com` (two dots, fewer than five) is
  first tried against each search domain (`.<ns>.svc.cluster.local`, `.svc.cluster.local`,
  `.cluster.local`, plus any node domains) — several failed lookups per resolution, each doubled for
  A and AAAA. A trailing dot (`api.example.com.`) or `dnsConfig` with `ndots: 1` fixes it.
- Move password hashing to a dedicated worker pool so it cannot starve fs and DNS.
- Linux conntrack races on UDP DNS (A and AAAA sent in parallel from the same socket) gave
  intermittent **exactly 5-second** lookups — the resolver's retry timeout. `options
  single-request-reopen`, `use-vc`, or NodeLocal DNSCache are the fixes. A latency histogram with a
  spike at exactly 5 s (or 5 s plus normal latency) is a DNS signature.

### 3. V8 memory, heap limits and GC (`C12`)

**Heap layout.** New space (the young generation: semi-spaces of a few MB to tens of MB, collected by
the **Scavenger**, a fast parallel copying collector) and old space (collected by **Mark-Compact**,
with concurrent and incremental marking but stop-the-world finalisation). Objects surviving two
scavenges are promoted. Large objects go straight to large-object space.

**Heap limits.** `--max-old-space-size` (MB) caps old space. The default is derived from system
memory with a cap, and older versions defaulted to around 1.5–2 GB regardless of the box; only recent
releases (Node 20+) size it from the cgroup limit. A container with a 512 MB limit could therefore let
V8 aim far above what the container allows, and the result is an **OOMKill (exit 137) with no
JavaScript stack** rather than a clean `FATAL ERROR: Reached heap limit`. Rule: set
`--max-old-space-size` explicitly to about 70–80% of the container limit, leaving room for off-heap
memory. Conversely, `--max-semi-space-size` (young generation) raised from the default to 32–64 MB
can cut scavenge frequency dramatically for allocation-heavy services — an unconventional throughput
win of 10–20% on some workloads.

**Off-heap memory counts against the container but not the heap.** `Buffer`s live outside the V8
heap, as do native addon memory, thread stacks, code space and glibc malloc arenas.
`process.memoryUsage()` shows `rss`, `heapTotal`, `heapUsed`, `external` and `arrayBuffers`. RSS
growing while `heapUsed` is flat means the problem is off-heap: retained buffers, a native module, or
**glibc malloc fragmentation** (many threads, each with its own arena). The unconventional fixes are
`MALLOC_ARENA_MAX=2` or running with `jemalloc` via `LD_PRELOAD` — which applies equally to the JVM
and to Python.

**GC pauses in Node.** Scavenges are usually 1–10 ms; they become frequent when the allocation rate is
high. Mark-compact pauses grow with the live heap. As old space approaches the limit, V8 collects more
often and reclaims less each time: the process spends most of its time in GC (the "GC death spiral")
and throughput collapses **minutes before** it finally crashes. Latency rising with heap size, before
the crash, is the tell.

**Observing it.** `--trace-gc` logs every collection and its duration; a `PerformanceObserver` with
`entryTypes: ['gc']` exports GC time as a metric; `v8.getHeapStatistics()` returns the real limit
(`heap_size_limit`) — log it at start-up and you will never again guess what the limit was.

### 4. Finding leaks and hot code in Node (`C12`, `C13`)

**Leak shapes that recur.**
- **Unbounded caches**: a module-level `Map` keyed by user id, URL or query string. A cache keyed on
  something with unbounded cardinality is a leak with a hit rate. Always bound it (LRU with max size
  and TTL).
- **Closures capturing more than you think**: closures created in the same scope share one context
  object, so a small callback kept long-term can pin a large request object another closure used.
- **Event listeners** added per request to a long-lived emitter (`process.on`, a shared socket, a
  Redis client) and never removed. Node warns with `MaxListenersExceededWarning` beyond 10 — the
  warning is a leak detector, not noise to silence with `setMaxListeners(0)`.
- **Timers and intervals** never cleared, holding their closures.
- **Promises that never settle** hold their continuation chain forever; `Promise.race` with a timeout
  does not free the losing promise or what it references.
- **Per-request metric labels** (user id, or a raw path with ids, as a Prometheus label) — a
  cardinality leak living in the metrics client, and in the metrics backend.
- **Module re-requires in tests or hot reload**, and `vm` contexts, which are never collected while
  referenced.

**The three-snapshot technique.** Take a heap snapshot, drive traffic, take a second, drive the same
traffic, take a third. In snapshot 3 look at objects allocated between 1 and 2 that still exist; follow
the **retainers** path up to a GC root. Tools: Chrome DevTools via `--inspect`,
`v8.writeHeapSnapshot()`, `--heapsnapshot-signal=SIGUSR2`, and `--heapsnapshot-near-heap-limit=N` to
capture automatically as the heap nears the limit. Caveats: writing a snapshot is stop-the-world,
needs roughly the heap size again in memory, and can take tens of seconds on a multi-GB heap — take it
on a pod drained from the load balancer, or the snapshot itself OOMKills the pod you are debugging.

**Attaching to a running process.** Sending `SIGUSR1` to a Node process opens the inspector on
`127.0.0.1:9229` without a restart; reach it with `kubectl port-forward` on a drained pod, take the
profile or snapshot, and close it. Better still, export GC time, heap statistics and event-loop delay
as metrics from day one, so the evidence exists before the incident.

**Allocation sampling** (`--heap-prof`, or DevTools "allocation sampling") is cheap enough to run in
production and shows *who allocates*, which pairs with the snapshot's *who retains*.

**CPU profiling.** `--cpu-prof` writes a `.cpuprofile`; `0x` and `clinic flame` produce flame graphs;
`clinic doctor` triages event loop vs I/O vs GC; `clinic bubbleprof` shows async waiting. Linux `perf`
with `node --perf-basic-prof` gives JIT-symbolised stacks including native and kernel frames. Reading a
flame graph: width is time on CPU, so look for wide plateaus; and an on-CPU flame graph shows nothing
about time spent *waiting* (`§17`).

**`async_hooks` cost.** `async_hooks.createHook` with `init`/`before`/`after` callbacks adds work to
every promise; older APM agents built on it cost 10–30% of throughput on promise-heavy code.
`AsyncLocalStorage` has since been made much cheaper (`AsyncContextFrame` in recent Node), but a
profile showing time in promise hooks or APM wrappers is a real finding. Rule out the agent before
blaming the code — disable it on one canary and compare.

### 5. Promises and async pitfalls in production (`C05`)

- **Accidental serialisation**: `for ... of` with `await` inside makes N sequential round trips; a
  page making 40 downstream calls one at a time costs 40 × RTT.
- **Unbounded parallelism**: `Promise.all(items.map(call))` over 50,000 items opens 50,000 sockets or
  queues 50,000 requests at a pool. Bounded concurrency (`p-limit`, `p-map` with `concurrency`) is
  the fix and is itself a backpressure mechanism (`§14`).
- **`Promise.all` fails fast but does not cancel**: the other calls keep running and consuming
  downstream capacity after the caller has already returned a 500. Propagate an `AbortSignal` to
  `fetch` and clients so the work actually stops.
- **Unhandled rejections** crash the process by default since Node 15. A rarely-failing
  fire-and-forget call becomes a crash loop the day its downstream has an outage.
- **Timeouts that do not stop work**: `Promise.race([call(), timeout(1000)])` returns to the caller but
  the call is still in flight, still holding a pool connection — so timeouts under load *raise*
  concurrency at the downstream rather than shedding it.
- **Stampede on cold keys**: 500 concurrent requests miss the same cache key and all compute it.
  Promise coalescing ("single-flight": store the in-flight promise in a map by key, delete it on
  settle) collapses them into one.
- **`await` in a hot loop over already-resolved values** costs a microtask per iteration; a
  synchronous fast path can be several times faster for cache hits.

### 6. JVM garbage collection and safepoints (`C06`, `C12`)

**Collectors.**
- **G1** (default since JDK 9): region-based, with a soft pause target of
  `-XX:MaxGCPauseMillis=200`. Pauses scale with the live data to evacuate and remembered-set work.
  **Humongous allocations** (objects of at least half a region) go straight into old regions and can
  trigger early concurrent cycles or full GCs; raise `-XX:G1HeapRegionSize` or avoid giant arrays.
- **ZGC** (production since JDK 15, generational from JDK 21 and the only mode from JDK 23):
  concurrent compaction, pauses under a millisecond regardless of heap size. The trade-off is
  throughput and headroom: if allocation outruns the concurrent collector you get **allocation
  stalls** — threads blocked waiting for memory — which look like latency spikes with no "pause" in
  the GC log (look for `Allocation Stall` lines).
- **Shenandoah**: similar low-pause goals, from Red Hat.
- **Parallel GC**: best throughput, longest pauses — right for batch, wrong for latency-sensitive APIs.
- **Serial GC**: what the JVM silently selects when it decides the machine is not "server class"
  (fewer than 2 CPUs or under ~1792 MB) — which a container with `cpu: 1` triggers (`§9`).

**Full GC** is G1's failure mode: evacuation failure ("to-space exhausted") when the heap is too full
to copy live objects into, followed by a full compaction taking seconds. Causes: a heap too small for
the live set, humongous objects, or a leak. Tuning pause targets does not fix a heap that is too small.
Starting concurrent marking earlier (`-XX:InitiatingHeapOccupancyPercent`, default 45, adaptive) gives
marking more time to finish before the heap fills. On OOM, `-XX:+ExitOnOutOfMemoryError` makes the JVM
exit immediately so the orchestrator restarts it, instead of limping on with threads that died.

**Reading a GC log line.** Unified logging reports `User`, `Sys` and `Real` times per pause. `Real`
close to `(User + Sys) / GC threads` is normal. `Real` far *above* `User + Sys` means the GC threads
were not running at all — waiting on swap-in, blocked on a log write, or throttled by the CFS quota
(`§9`). `Sys` unusually high points to the kernel: page faults, THP compaction, memory zeroing.

**Safepoints.** Many JVM operations — GC phases, deoptimisation, biased-lock revocation (before JDK 15),
thread dumps, class redefinition by agents — require every Java thread to reach a **safepoint**.
Time-to-safepoint (TTSP) is how long the slowest thread takes to get there, and every other thread is
stopped while it does. A long counted loop from which the JIT removed safepoint polls could hold the
whole JVM for hundreds of ms; loop strip mining (JDK 10+) mitigated this. Diagnose with
`-Xlog:safepoint`, reading the "reaching safepoint" time and not just the operation time. A GC log
showing a 5 ms pause alongside a 400 ms user-visible stall is a TTSP problem.

**Other stop-the-world surprises.**
- `-XX:+HeapDumpOnOutOfMemoryError` writing a 30 GB dump to a slow volume.
- `jmap -histo:live` forces a full GC; running it "to have a look" in production causes the incident.
- GC log writes blocking at safepoint when the disk is saturated (another container's log burst on a
  shared node); log to tmpfs or use asynchronous logging (`-Xlog:async`, JDK 17+).
- Transparent huge pages compaction stalls; many teams set THP to `madvise` or `never`.
- Swap: a heap partly swapped out turns a 50 ms GC into seconds, because GC touches every live page.

**Tooling.** `-Xlog:gc*:file=...` unified logging, read with GCViewer or GCeasy; **JFR**
(`jcmd <pid> JFR.start`) at ~1–2% overhead records GC, lock contention, allocation, I/O and thread
states in production; `jcmd <pid> Thread.print` or `jstack` for thread dumps — take three a few
seconds apart and look for threads stuck in the same frame; **async-profiler** for CPU, allocation,
lock and wall-clock flame graphs without safepoint bias (older JVMTI samplers only sample at
safepoints, so they blame code near the polls rather than the code actually burning CPU). For slow
leaks: trend `jcmd <pid> GC.class_histogram` over days, use JFR's `OldObjectSample` event (which
records where long-lived objects were allocated), and analyse heap dumps with Eclipse MAT's dominator
tree and "path to GC roots" — taken from an instance drained of traffic, since the dump pauses it.

### 7. JVM threads, pools, virtual threads and warm-up (`C06`, `C08`)

**Thread pool starvation.**
- Tomcat's default `server.tomcat.threads.max=200`. If each request blocks 1 s on a downstream,
  200 threads cap throughput at 200 rps whatever the CPU (Little's Law, `§10`).
- `CompletableFuture.supplyAsync` without an executor uses the **common ForkJoinPool**, sized
  cores − 1. Blocking I/O inside it (JDBC, HTTP) starves every other user of the common pool,
  including parallel streams elsewhere in the app. On a 2-CPU container the common pool has **one**
  thread; on 1 CPU `CompletableFuture` falls back to a new thread per task.
- **Nested submission deadlock**: a task on a bounded pool submits a subtask to the same pool and
  blocks on it. When every thread is a parent waiting for a child, nothing progresses — and it only
  happens at full load, so it passes every test. Use separate pools per stage, or never block on the
  pool you run on.
- **Connection pools** (HikariCP's default `maximumPoolSize=10`) smaller than the thread pool: 200
  request threads queue for 10 connections and show as `TIMED_WAITING` in
  `HikariPool.getConnection`. The right size is small — on the order of DB cores × 2 — not "match the
  threads" (`DB21`). A long transaction holding a connection while making an HTTP call is the usual
  root cause of pool exhaustion, not the pool size. Hikari's `leakDetectionThreshold` logs the stack
  of any connection held longer than the threshold — the fastest way to find who holds them.

**Virtual threads (JDK 21).** Millions of cheap threads mounted on a small carrier pool (a
ForkJoinPool sized to cores). A virtual thread blocking on I/O unmounts from its carrier — *unless it
is pinned*:
- blocking inside a `synchronized` block or method (fixed in JDK 24 by JEP 491),
- blocking inside native code or a foreign call.
A pinned virtual thread holds its carrier; with 8 carriers, 8 pinned threads stall the whole
application while CPU sits near zero. Diagnose with `-Djdk.tracePinnedThreads=full` (before JDK 24)
or the JFR event `jdk.VirtualThreadPinned`. Fixes: replace `synchronized` with `ReentrantLock` on the
blocking path, upgrade libraries (older JDBC drivers and pools used `synchronized` heavily), or
upgrade the JDK. Two further traps: virtual threads remove the thread-pool bulkhead, so the bottleneck
moves to the DB pool or downstream and you need an explicit `Semaphore`; and per-thread caches in
`ThreadLocal` (buffers, formatters) now multiply by a million threads.

**Container awareness.** Since JDK 10 (and 8u191) the JVM reads cgroup limits.
`-XX:MaxRAMPercentage` defaults to **25**, so a 4 GB container gets a 1 GB heap — too small for most
services; set 70–75%. The CPU count comes from the CPU quota and sizes GC threads, the common pool and
JIT compiler threads; `cpu: 1` gives Serial GC, a tiny common pool and starved JIT.
`-XX:ActiveProcessorCount=N` overrides it. Non-heap memory — metaspace, code cache, thread stacks
(1 MB each by default), direct buffers (Netty), GC structures — means `-Xmx` equal to the container
limit guarantees an OOMKill. Native Memory Tracking (`-XX:NativeMemoryTracking=summary`,
`jcmd <pid> VM.native_memory`) accounts for it. Direct buffers default to a cap equal to `-Xmx`, so a
Netty service can use twice its heap; set `-XX:MaxDirectMemorySize` and watch the pooled allocator's
metrics.

**JIT warm-up.** A fresh JVM starts interpreted, then compiles with C1, then C2 once methods are hot
(thousands to ~10,000 invocations). The first minutes of a pod's life are slower and burn CPU on
compilation, giving CPU and p99 spikes on every deploy and scale-out — worst under a tight CPU limit,
where JIT threads compete with request threads and get throttled (`§9`). Mitigations: synthetic
warm-up traffic before readiness (replaying captured requests), load-balancer slow start (Envoy
`slow_start_config`, ALB slow-start duration), a higher CPU limit during start-up (in-place pod resize
or a start-up CPU boost), CDS/AppCDS archives for class loading, CRaC checkpoint/restore, or GraalVM
native image (no warm-up, lower peak throughput). Deoptimisation storms — a new type seen at a
monomorphic call site, a feature flag flipping a code path — can cause mid-life CPU spikes too. Slow
start-up under a small CPU request also trips liveness and readiness probes into a restart loop; a
Kubernetes `startupProbe` gives start-up its own generous budget before the other probes begin.

### 8. Python: the GIL, asyncio and pre-fork servers (`C07`)

**The GIL.** One thread executes Python bytecode at a time per interpreter; the running thread is
asked to drop it every `sys.getswitchinterval()` = 5 ms. Threads help for I/O (the GIL is released
during blocking syscalls) and for C extensions that release it (NumPy, hashlib and zlib on large
buffers). A CPU-heavy thread in a web process adds up to 5 ms to every other thread's GIL
re-acquisition, and because I/O threads must re-acquire the GIL after every syscall, it can convoy
them. Free-threaded builds (PEP 703, `3.13t`) exist, but most production Python still has the GIL.

**asyncio blocking calls.** One blocking call — `requests.get`, `time.sleep`, a sync DB driver, a big
`json.loads`, sync file I/O — stalls the loop for everyone, exactly as in Node. Diagnose with
`PYTHONASYNCIODEBUG=1` or `loop.set_debug(True)`, which logs callbacks slower than
`loop.slow_callback_duration` (default 100 ms). Fixes: async clients (`httpx`, `asyncpg`), or
`await asyncio.to_thread(fn)` / `run_in_executor` — whose default executor is bounded at
`min(32, cpu + 4)` threads, a hidden pool that can itself saturate. Django's `sync_to_async` defaults to
`thread_sensitive=True`, which runs all such calls on **one shared thread** — an accidental global
serialisation point for sync ORM calls made from async views.

**Fire-and-forget tasks.** `asyncio.create_task` returns a task the loop references only weakly; if
nothing else holds it, it can be garbage-collected before it finishes (the docs say to keep a
reference). Creating a task per message with no bound is also unbounded concurrency. Keep tasks in a
set and discard them on completion, bound concurrency with a `Semaphore`, or use `asyncio.TaskGroup`
(3.11+) so failures and cancellation propagate.

**Gunicorn workers.** Sync workers handle one request at a time each, so concurrency equals worker
count; a slow downstream directly caps throughput, and the default `timeout=30` kills workers
mid-request (`WORKER TIMEOUT`). `gthread` adds threads per worker; Uvicorn workers serve ASGI.
`--max-requests` with `--max-requests-jitter` recycles workers — a legitimate leak mitigation, and the
jitter matters, otherwise every worker restarts at the same moment. Gunicorn's documented starting
point is `2 × cores + 1` workers for I/O-bound sync apps; with threads on top, total runnable threads
should still be sized against the CPU limit, not the node (`§9`).

**Task queues.** Celery's `worker_prefetch_multiplier` (default 4) makes each worker process reserve
`concurrency × 4` messages in advance. With a mix of long and short tasks, short ones sit trapped behind
long ones on a busy worker while other workers are idle. For long tasks use prefetch 1 with
`acks_late`, and route long and short task types to separate queues and worker pools.

**Pre-fork and copy-on-write.** `--preload` imports the app in the master and then forks, so workers
share pages copy-on-write. In theory, large savings. In practice CPython's **reference counting writes
to the header of every object it touches**, and the cyclic GC writes to object headers as it walks
generations, so shared pages are copied page by page and each worker's private memory climbs towards a
full copy within hours. Fixes: `gc.freeze()` (3.7+) just before forking, which moves every existing
object into a permanent generation the collector ignores — Instagram's approach, combined with
`gc.disable()` in the master during import; immortal objects (PEP 683, 3.12) for some builtins; or
moving large read-only data out of Python objects into a memory-mapped file, a NumPy array or shared
memory. Measure with PSS/USS (`smem`, `/proc/<pid>/smaps_rollup`), not RSS, because RSS counts shared
pages in every process. `--preload` also breaks anything opened before the fork: DB connections and
sockets shared between workers interleave protocol bytes and corrupt streams; open them in `post_fork`.

**Python memory.** pymalloc keeps freed memory in arenas, and fragmentation stops arenas being
returned, so RSS rarely goes back down after a spike — a worker that once parsed a 500 MB payload
stays big. `tracemalloc` snapshot diffs find Python allocations; `memray` also sees native ones;
`py-spy top`/`record`/`dump` sample a live process without restarting it, and `py-spy dump` prints
every thread's stack — the Python equivalent of `jstack`.

**Python's cyclic GC.** Reference counting frees most objects immediately; the generational cycle
collector handles cycles, triggered by allocation counts (`gc.get_threshold()`, default
`(700, 10, 10)`). A generation-2 collection walks every tracked container object, so a process holding
millions of long-lived objects gets pauses of tens to hundreds of ms. Measure with `gc.callbacks`
(start/stop timing), then raise thresholds, avoid building cycles in hot paths, or `gc.freeze()` the
long-lived objects after start-up so collections skip them. Recent releases rework the collector's
generations, so check the version's behaviour before tuning.


### 9. Containers, CPU quotas and memory accounting (`C01`, `C12`, `O01`)

**CFS quota throttling.** A Kubernetes CPU limit becomes a CFS quota: `cpu: 2` means 200 ms of CPU
time per **100 ms period**, summed across all threads. A multi-threaded process on an 8-core node can
burn the whole quota in the first 25 ms of a period and is then **frozen for the remaining 75 ms**.
Average CPU over a minute can read 30% while individual requests stall 50–100 ms — the classic "p99
spikes at low average CPU". Evidence: the ratio `container_cpu_cfs_throttled_periods_total` /
`container_cpu_cfs_periods_total`, or `nr_throttled` and `throttled_usec` in the cgroup's `cpu.stat`.
More than a few per cent of periods throttled on a latency-sensitive service is a real problem.

Why it bites harder than people expect:
- Runtimes size themselves from the node's cores rather than the quota: dozens of GC threads, JIT
  threads, `GOMAXPROCS` threads or pool workers all burn the quota together in a burst. Match thread
  counts to the limit (`-XX:ActiveProcessorCount`, `automaxprocs` in Go, explicit worker counts).
- GC bursts: a parallel GC phase with many threads empties the quota and the application is throttled
  for the rest of the period — the GC log reports a short pause, users see a long one.
- Older kernels had a bug (fixed around 5.4) that throttled even when the quota was not used.
- Metrics sampled every 15–60 s average away 100 ms bursts entirely; only the throttling counters
  reveal them.

The unconventional fix many teams land on: **remove CPU limits** for latency-sensitive services and
rely on CPU *requests* (which set CFS weight, a proportional guarantee under contention) plus node
capacity planning. Alternatives: a limit well above the request, fewer threads, the kubelet CPU
manager's `static` policy for exclusive cores (Guaranteed QoS with integer CPUs), or cgroup v2
`cpu.max.burst`. Memory limits, by contrast, should stay — memory is not compressible.

**Memory in cgroups.**
- Container memory usage includes **page cache** charged to the cgroup. `container_memory_usage_bytes`
  counts it; the kernel reclaims cache before killing. Alert on `container_memory_working_set_bytes`,
  not usage, or you page on harmless file cache.
- An OOMKill shows as exit code **137** (SIGKILL), `reason: OOMKilled`, and in `dmesg` as
  `Memory cgroup out of memory: Killed process`. There is no application log, no stack and no heap
  dump — the runtime never got a chance.
- tmpfs (`emptyDir: { medium: Memory }`) and `/dev/shm` count against the memory limit: a service
  writing temp files there OOMs with a flat heap.
- In a multi-process container (gunicorn, `cluster`), the OOM killer picks the largest process — often
  a worker, sometimes the master, which takes every worker with it.

**Base images and libc.** Alpine uses musl rather than glibc. musl's allocator is slower under
multi-threaded allocation-heavy load, its DNS resolver handles search domains and parallel A/AAAA
queries differently (and ignores glibc `resolv.conf` options such as `single-request-reopen`), and
Python wheels may be built from source without the optimisations of manylinux builds. An image change
that makes a service slower with no code change is worth benchmarking on allocator and resolver first.

**Noisy neighbours.** Pods on a node share L3 cache, memory bandwidth, disk I/O and network. A batch job
landing on your node degrades p99 with no change in your own metrics. The symptoms correlate with the
node, not with your version — slice latency by node and availability zone before blaming code.

**Swap on nodes.** Kubernetes historically required swap off, but swap support (`NodeSwap`, stable in
recent releases) and zram on some node images mean a pod can be paged out rather than killed. For a
latency-sensitive service that is worse than an OOMKill: every touched page is a major fault, and GC or
Python's refcounting touches pages constantly. Check `si/so` in `vmstat` and the node's swap
configuration; set `LimitedSwap`/no swap for such workloads. Under memory pressure without swap, the
kernel instead evicts page cache and code pages, which shows as rising major faults and disk reads.

**Steal time** (`%st` in `top`, `st` in `vmstat`) on VMs means the hypervisor gave your vCPU to
someone else. Burstable instances (AWS `t3`) that exhaust CPU credits drop to a baseline — a cliff
that arrives after hours of normal operation.

**NUMA and big boxes.** On multi-socket hosts, memory attached to the other socket is slower, and
threads migrating between sockets lose cache. Mostly relevant to large JVMs and databases; `numactl`
and `-XX:+UseNUMA` exist for it.

### 10. Queueing theory for the on-call engineer (`C14`)

**Little's Law**: L = λ × W. Concurrency in a system equals arrival rate times time in the system. It
holds for any stable system and is the fastest sanity check in an incident:
- 2,000 rps × 50 ms = 100 requests in flight. If latency rises to 500 ms at the same rate you need
  1,000 in flight — more than your 200 threads or 10 DB connections, so a queue forms somewhere.
- It sizes pools: a DB pool of 20 with 5 ms queries supports at most 4,000 queries/s; beyond that
  requests wait.
- It locates the bottleneck: the stage whose in-flight count grows while its throughput stays flat.

**Utilisation and the knee.** For an M/M/1 queue, mean waiting time is service time × ρ / (1 − ρ). At
50% utilisation the queueing delay equals one service time; at 80% it is 4×; at 90% 9×; at 95% 19×.
Latency is flat until roughly 70–80% and then climbs almost vertically. Consequences:
- A service at 85% CPU is not "15% from full" — it is past the knee.
- Small load increases near the knee cause large latency increases, which cause retries, which
  increase load (`§18`).
- Variability matters as much as load. Kingman's approximation: wait ≈ ρ/(1−ρ) × (Ca² + Cs²)/2 × service
  time, where Ca and Cs are the coefficients of variation of arrivals and service. A few very slow
  requests among fast ones make queues far worse at the same average; isolating slow work in its own
  queue or pool reduces everyone's wait.
- Many servers sharing one queue (M/M/c) beat one queue per server at the same utilisation — why
  "least outstanding requests" or "power of two choices" balancing beats round-robin, and why a shared
  worker pool beats pinning work to workers.

**Tail latency amplification with fan-out.** If a request fans out to N backends in parallel and waits
for all of them, its latency is the maximum of N samples. The probability that at least one exceeds
its p99 is 1 − 0.99^N: about 10% for N = 10 and 63% for N = 100. A backend's p99 becomes the
frontend's median at high fan-out. Hence:
- **Hedged requests**: send to one replica; if there is no reply by the backend's p95, send a second
  to another replica and take whichever answers first. Costs ~5% extra load and cuts the tail
  sharply (Dean and Barroso, "The Tail at Scale"). Only for idempotent reads; cancel the loser.
- **Tied requests**: enqueue on two replicas at once, each aware of the other; whichever starts first
  cancels its twin.
- Reduce fan-out, return partial results at a deadline, or answer from a good-enough subset.
- Micro-partitioning and replicating hot shards so a slow node can shed its share.

**Percentiles cannot be averaged.** The mean of per-pod p99s is not the fleet p99. Aggregate
histograms (Prometheus buckets summed then `histogram_quantile`, or merged HDR histograms), never
percentiles. A user action that makes 20 sequential calls experiences the tail of each.

**Latency shapes and signatures.** A bimodal histogram means two populations (cache hit or miss, one
bad pod, one AZ, GC or no GC). Spikes at exact values mean timers and retries: 1 s and 3 s (TCP SYN
retransmits; initial RTO was 3 s on old kernels, 1 s since RFC 6298), 5 s (DNS retry), 40 ms or
200 ms (Nagle with delayed ACK, `§15`), 30 s or 60 s (a default timeout somewhere). Knowing these
signatures is often the whole diagnosis.

### 11. Load testing, coordinated omission and capacity (`C15`)

**Coordinated omission** (Gil Tene). A closed-loop load generator — N virtual users, each sending the
next request only after the previous reply — slows down when the server slows down. During a 2 s stall
each user records one slow request and sends nothing else, whereas real users arriving at a constant
rate would have produced hundreds of slow requests. The reported p99 can be 10–100× better than
reality. Fixes:
- Use an **open-model**, constant-arrival-rate generator: `wrk2` (`-R` rate), k6's
  `constant-arrival-rate` executor, Gatling's open injection profiles, Vegeta.
- Measure latency from the *intended* send time, not the actual send time (HdrHistogram has a
  correction for this).
- Treat a test in which the load generator itself hit 100% CPU as invalid.

**Other load-test lies.**
- A warm cache with a tiny key set, while production has a long tail of cold keys.
- One client IP, which a load balancer hashes to one backend, and connection reuse that hides TLS
  handshake cost.
- Too short: leaks, old-generation growth, log rotation, connection churn and JIT deoptimisation only
  show after hours. A soak test runs at realistic load for 8–24 h.
- Synthetic payloads that compress perfectly or skip the expensive branch.
- A staging database at 1% of production's size, so query plans differ.
- Ramps faster or slower than real traffic, so autoscaling is never exercised realistically.

**Finding the real capacity.** Step the arrival rate up and plot throughput and latency against offered
load. Throughput rises linearly, flattens (saturation) and, in a system with contention, *falls*
(retrograde scaling). Gunther's Universal Scalability Law, C(N) = N / (1 + α(N−1) + βN(N−1)), models it:
α is contention (serialised fraction, Amdahl), β is coherency cost (crosstalk between nodes), and any
β > 0 means that beyond a point adding nodes or threads reduces throughput. Define capacity as the load
at which the latency SLO is breached, not where throughput peaks, and plan to run at about 60–70% of it.

**Capacity from production.** Shadow traffic, replaying captured logs, and **squeeze testing** —
shifting live traffic onto fewer instances with load-balancer weights until latency degrades — tell
you more than any synthetic test. Keep N+1 headroom per availability zone so losing one zone does not
push the others past the knee.

### 12. Locks, contention and scheduling pathologies (`C08`, `C09`, `C10`)

**False sharing.** Two threads updating different variables on the same 64-byte cache line make the
line bounce between cores (MESI invalidations). A per-thread counter array `long[threads]` scales
*negatively* as threads are added. Fixes: padding (`@Contended` in the JDK, which needs
`-XX:-RestrictContended` for application classes), `LongAdder` instead of a hot `AtomicLong` (it
stripes and pads cells), or per-thread state combined at read time. Evidence: `perf c2c` reports HITM
events (loads that hit a line modified in another core's cache).

**Lock convoys.** Under contention threads queue on a lock; each release wakes one waiter, which must be
scheduled, run, and release, while new arrivals join the back. Throughput collapses to roughly one
context switch per critical section even though the critical section is tiny. Signs: a high
context-switch rate (`vmstat` `cs`, `pidstat -w`), low CPU, many threads `BLOCKED` on the same monitor
in thread dumps, JFR `jdk.JavaMonitorEnter` events. Fixes: shrink or remove the critical section,
stripe the lock, use a concurrent or lock-free structure, or batch work per acquisition. A *fair* lock
makes convoys worse, because it forbids barging and forces a context switch on every handover.

**Hidden global locks** are the ones that surprise people:
- Synchronous logging appenders, or a logger writing synchronously to a stdout pipe whose reader (the
  log collector) is slow — every thread that logs blocks. Log4j2 async loggers and Logback's
  `AsyncAppender` exist for this, with the trade-off of dropping or blocking when their queue fills.
  In Node, a logger writing with `fs.writeSync` to stdout blocks the loop the same way; pino's
  worker-thread transports move formatting and I/O off the loop, at the cost of losing buffered lines
  if the process crashes.
- `java.util.Random` shared across threads (CAS contention on the seed — use `ThreadLocalRandom`).
- `SecureRandom` blocking on `/dev/random` entropy in old JDKs and fresh VMs.
- Class loading and static initialisers; `Hashtable`, `Collections.synchronizedMap`, `StringBuffer`.
- Pool-internal locks in older connection pools and HTTP clients.
- In Python the GIL and the import lock; in Node the single thread *is* the lock.

**Thundering herds.** Many waiters woken by one event when only one can proceed: `accept()` on a shared
socket (solved with `SO_REUSEPORT` or `EPOLLEXCLUSIVE`), `notifyAll` on a condition, every client
reconnecting when a server restarts, every cache entry expiring together, every cron job at
`0 * * * *`, every retry firing at the same backoff instant. Fixes: jitter everything (TTLs, backoffs,
schedules), single-flight, staggered reconnects, `notify` rather than `notifyAll` when one waiter is
enough.

**Priority inversion.** A high-priority task waits for a lock held by a low-priority task that is itself
pre-empted by medium-priority work — the Mars Pathfinder bug, fixed with priority inheritance. In
backend terms: health checks or admin endpoints queued behind bulk traffic on the same pool; a
latency-critical request waiting for a connection held by a batch job; consumer heartbeats on the same
thread as slow processing; lease renewal starved by the work the lease protects. Fixes: separate pools
and connections per priority class, and keep the control plane (health, heartbeats, lease renewal) on
its own thread or process — Spring Boot's `management.server.port` serves actuator endpoints from a
separate connector and thread pool for exactly this reason.

**Read-write locks and config swaps.** A read-write lock around hot shared state stalls badly when a
writer arrives: in writer-preferring implementations new readers queue behind the waiting writer,
which itself waits for every in-flight reader, so one config reload freezes all requests for as long
as the slowest reader. For read-mostly state, publish **immutable snapshots** and swap an atomic
reference (`AtomicReference`, a `volatile` field, or plain reassignment in Node) — readers never block.

**Concurrent maps are not atomic transactions.** Replacing `Collections.synchronizedMap` with
`ConcurrentHashMap` removes the global lock, but `if (!map.containsKey(k)) map.put(k, v)` is now a race;
use `computeIfAbsent`, `putIfAbsent` or `merge`. A slow function inside `computeIfAbsent` holds the lock
on that bin, blocking other keys that hash nearby — keep it short or cache a future instead.

**Shared versus partitioned queues.** Per-worker queues with hash assignment preserve per-key ordering
and cache affinity but inherit the worse queueing of separate M/M/1 servers (`§10`); work stealing
(`ForkJoinPool`) lets idle workers take from busy ones when ordering does not matter.

**Deadlocks that only happen in production.** Lock ordering across two code paths; the nested thread
pool deadlock (`§7`); a DB row lock and an in-process lock taken in opposite orders; blocking on a
result on the very thread that must produce it (`.get()` on an event-loop thread,
`asyncio.run_coroutine_threadsafe(...).result()` from inside the loop). JVM thread dumps report monitor
deadlocks explicitly ("Found one Java-level deadlock"); pool deadlocks and DB-plus-app deadlocks they
do not. Single-threaded runtimes deadlock too: two promises awaiting each other through an in-process
mutex (`async-mutex`) or a queue whose worker needs a lock the awaiting request holds. Nothing is
"blocked", the loop is idle, and only request timeouts and async stack traces (`--async-stack-traces`,
on by default) reveal the two waits. Never hold an in-process lock across an `await` of work that may
need the same lock.

**Spinning without progress.** A CAS retry loop under heavy contention burns CPU and makes little
progress; a busy-wait with no backoff looks like "high CPU, low throughput". Spin briefly, then park,
or back off exponentially on CAS failure.

### 13. Distributed locks, leases and time (`C11`)

**A lock is a lease.** Across processes you cannot know the holder is alive, so locks carry a TTL. The
holder may be paused — by a GC pause, CFS throttling, VM live migration, SIGSTOP, a page-fault storm,
a blocked event loop — for longer than the TTL. When it resumes it still *believes* it holds the lock
and writes, while the lease has expired and another process is also writing. Checking "do I still
hold it?" before the write does not close the gap: the pause can fall between the check and the write.

**Fencing tokens.** The lock service issues a monotonically increasing token with each grant (an etcd
revision, a ZooKeeper sequence number, a DB sequence). Every write to the protected resource carries
the token, and **the resource rejects writes with a token lower than the highest it has seen**
(`UPDATE ... SET ..., fence = :token WHERE id = :id AND fence < :token`, or a conditional write).
Correctness then lives in the storage layer, not in the lock. If the resource cannot check tokens, the
lock is an efficiency optimisation (avoiding duplicate work), not a correctness guarantee — so design
the work to be idempotent.

**Redlock and its critique.** Redlock acquires the lock on a majority of N independent Redis nodes with
a TTL. Martin Kleppmann's critique: its safety depends on bounded clock drift, bounded pauses and
bounded network delay, and it yields no fencing token, so a paused client breaks mutual exclusion.
Antirez replied that the timing assumptions are reasonable in practice. The senior position: for
efficiency locks, a single Redis `SET key <random> NX PX 30000` with a Lua compare-and-delete release
is fine; for correctness, use a consensus system (etcd, ZooKeeper) or the database itself (`SELECT ...
FOR UPDATE`, a transaction-scoped Postgres advisory lock) and fence at the resource. Redis failover with
asynchronous replication can also lose a freshly acquired lock outright.

**Lease renewal pitfalls.**
- A renewal timer on a blocked event loop or a GC-paused JVM fires late (`§1`, `§6`), so the lease
  expires mid-job although the code "renews every 10 s".
- Renew at a third of the TTL, check each renewal's result, and **abort the work** when a renewal
  fails rather than carrying on.
- Choose TTLs above your worst *observed* pause — measure GC logs, throttling counters and event-loop
  max — and remember a longer TTL means slower failover.

**Clocks.** Wall clocks jump (NTP step corrections, differing leap-second smears, VM resume). Use
monotonic clocks (`process.hrtime.bigint`, `performance.now`, `System.nanoTime`, `time.monotonic`)
for durations, timeouts and TTL arithmetic. Ordering events from two machines by wall-clock
timestamps is unsafe without bounds (TrueTime-style uncertainty intervals or hybrid logical clocks,
`M21`).

**Leader election** has the same problem: a paused leader wakes and acts as leader. Epochs and
generations are fencing tokens by another name — Kubernetes Lease objects, Kafka's controller epoch,
the consumer-group generation id that fences zombie consumers, and the transactional producer epoch
that fences zombie producers. ZooKeeper leaders have the same gap: a session can expire while the
process is paused, and the process learns of it only when it next hears from ZooKeeper — the storage
must reject its writes (use the znode's sequence number or `zxid` as the token). Epoch fencing covers
writes to systems that check it; external side effects a zombie already performed are not undone.

**Atomicity in the lock store itself.** Many "lock" bugs are non-atomic sequences: `INCR` followed by
`EXPIRE` leaves an immortal key if the client dies between them; "GET then DEL" to release a lock deletes
someone else's lock if yours expired in between. Use single commands (`SET key val NX EX 30`), Lua
scripts (atomic on the server), or optimistic `WATCH`/`MULTI`/`EXEC`. And ask whether the lock is needed
at all: an atomic read-modify-write (a Lua script, `INCRBY`, a conditional SQL `UPDATE`) avoids the
lock and its expiry problem entirely.

**Postgres advisory locks.** `pg_advisory_lock` / `pg_try_advisory_lock` are **session-level**: held
by the connection until released or disconnected. With a connection pool the job may release a lock on
a different connection than it took it on (so the release fails and the lock leaks), and behind
PgBouncer in transaction mode session state is meaningless. Use `pg_try_advisory_xact_lock`, which is
released at transaction end, inside the transaction that does the work — or a job table with
`SELECT ... FOR UPDATE SKIP LOCKED`.

**Failure detection.** A fixed "dead after 3 missed heartbeats" rule declares healthy but busy nodes
dead — especially when heartbeats run on the same thread or loop as the work (priority inversion,
`§12`). Put heartbeats on a dedicated thread, use accrual detectors (phi accrual, as in Cassandra and
Akka) that adapt to observed heartbeat variance, and rate-limit reassignment so one false positive
does not overload the survivors.

**Idempotency beats locking.** For "do this exactly once" side effects — a daily report, a charge, an
email — a unique key enforced by storage (a unique constraint on `(report_date)`, an idempotency key at
the payment provider) makes duplicates harmless and needs no lock for correctness. Hold row locks only
for short local work, never across a remote call: record an intent, commit, call out with the key,
record the result.

### 14. Backpressure and streams (`C16`, `C19`)

**Backpressure** means a slow consumer slows the producer, instead of an unbounded buffer absorbing the
difference until memory runs out. Every unbounded queue is a latent OOM and a latency bomb: work waits
in it long after the client gave up, so the server spends its capacity on requests nobody is waiting for.

**Node streams.** `writable.write()` returns `false` when the internal buffer exceeds `highWaterMark`
(16 KiB for byte streams, 16 objects in object mode, 64 KiB for `fs.createReadStream`). Ignoring the
return value — writing in a loop without waiting for `'drain'` — buffers everything in memory. `.pipe()`
handles backpressure but not errors or clean-up (a failed destination leaves the source open, leaking
file descriptors); **`stream.pipeline()`** (or `stream/promises`) handles both and destroys every stream
on error. `for await (const chunk of readable)` respects backpressure on the read side. Streaming a
response to a slow client: when `res.write` returns `false` the source must pause, or one slow mobile
client makes the server buffer an entire export.

**The kernel's own queues overflow silently.** When the accept queue is full (the application is not
calling `accept()` fast enough — a blocked event loop, say) or the conntrack table is full
(`nf_conntrack: table full, dropping packet` in `dmesg`), SYNs are dropped and clients retransmit after
1 s, then 2 s, 4 s. `nstat -az | grep -i listen` shows `ListenOverflows` and `ListenDrops`; `ss -lnt`
shows `Recv-Q` (current backlog) against `Send-Q` (the limit, the lower of the app's backlog and
`net.core.somaxconn`).

**Where queues hide.** Socket send buffers; the kernel accept queue (`somaxconn`, listen backlog); the
load balancer's queue; the executor queue (`Executors.newFixedThreadPool` uses an unbounded
`LinkedBlockingQueue` and never rejects); pending event-loop callbacks; Kafka consumer lag; HTTP/2 flow
control windows; the connection-pool wait queue; retries inside client libraries.

**Load shedding.** Bound every queue and reject early when it is full (fast 503/429 with
`Retry-After`) — a rejected request costs microseconds, a timed-out one cost its whole budget.
- **Adaptive concurrency limits** (Netflix `concurrency-limits`, Envoy adaptive concurrency): treat the
  server like a TCP window and shrink the limit when latency rises (AIMD, gradient algorithms) instead
  of a fixed limit that is always wrong for somebody.
- **CoDel-style queue management**: if queueing delay has stayed above a target for an interval, drop
  requests or switch from FIFO to LIFO — the newest requests are the ones whose clients are still
  waiting (Facebook's adaptive LIFO).
- **Deadline propagation**: pass the remaining budget downstream (gRPC deadlines, a header) and drop
  work whose deadline has already passed before starting it.
- **Priority shedding**: drop batch, prefetch and analytics traffic before interactive traffic.
- **Rejection policies are backpressure choices.** Java's `ThreadPoolExecutor` offers `AbortPolicy`
  (throw — fast rejection), `CallerRunsPolicy` (the submitting thread runs the task — natural
  backpressure for producers, a disaster when the submitter is a request thread, because the front door
  then does background work), and the discard policies (silent loss). Choose per queue deliberately.
- **Per-subscriber buffers.** A push fan-out (websockets, gRPC server streaming, SSE) needs a bounded
  buffer per subscriber and a policy for slow ones: drop, conflate to the latest value, or disconnect.
  gRPC Java exposes readiness (`ServerCallStreamObserver.isReady()` and `setOnReadyHandler`) so the
  server writes only when HTTP/2 flow control allows. For push *sources* you cannot slow, the choices
  are the same: pause reading the socket (TCP backpressure), drop, or conflate.
- **Global limits across pods.** A downstream's quota is global; per-pod limiters of quota ÷ N break
  with uneven load, autoscaling and retries. Use a shared token bucket (Redis script) or a central
  gateway, honour `Retry-After`, and propagate the limit back to callers rather than queueing forever.

**Queue consumers.** Pull-based consumers get backpressure naturally, but a Kafka consumer that takes
longer than `max.poll.interval.ms` (default 5 minutes) between polls is removed from the group; the
rebalance hands its partitions and uncommitted batch to another consumer, which is also slow — a
rebalance storm with duplicate processing. Fixes: smaller `max.poll.records`, `pause()`ing partitions
while processing, bounded asynchronous processing that commits only completed offsets. Partitions cap
parallelism — consumers beyond the partition count sit idle — so the lever for more throughput within
a partition is keyed parallelism: process records for different keys concurrently while preserving
order per key (Confluent's parallel consumer does this), committing only the contiguous prefix of
completed offsets.

**Zero-copy and syscalls.** `sendfile`/`splice` avoid copying file bytes through user space
(`FileChannel.transferTo`, Kafka's log serving). TLS defeats it unless kernel TLS is used. Many tiny
`write` syscalls — one per log line or small chunk — cost more than the bytes; buffer and flush.

### 15. Network efficiency and connection management (`C17`, `A12`)

**Keep-alive and ephemeral ports.** Node's global `http.Agent` defaulted to `keepAlive: false` until
Node 19, so each outbound request paid DNS, a TCP handshake and a TLS handshake. At high rates this
also exhausts **ephemeral ports**: the side that closes first holds the socket in `TIME_WAIT` for 60 s
on Linux, and with the default range `32768–60999` (~28,000 ports) one client can open only about
470 new connections per second to a single destination IP and port. Symptom: `EADDRNOTAVAIL` or
connect timeouts under load, with the service itself healthy. Fix: keep-alive pools with sensible
`maxSockets` (the agent's default is `Infinity`, so a keep-alive agent under a burst opens one socket
per concurrent request and then keeps them all); not `tcp_tw_recycle`, which was removed in Linux 4.12 because it broke clients behind NAT.

**Idle-timeout mismatches.** A pooled keep-alive connection closed by the server — or silently by a load
balancer, NAT gateway or firewall — while idle gives `ECONNRESET` or "socket hang up" on the next
request. AWS ALB's idle timeout defaults to 60 s; an AWS NAT gateway drops idle flows after 350 s
without telling either side; Node's HTTP server `keepAliveTimeout` defaults to 5 s. Rules: **the
client's idle timeout must be shorter than the server's**, and a server behind a load balancer must keep
connections open longer than the balancer does (Node behind an ALB: `keepAliveTimeout` ~65 s with
`headersTimeout` slightly above it). Retry once on a reset for idempotent requests; TCP keepalive
probes keep NAT mappings alive for long-lived idle connections.

**Nagle and delayed ACK.** Nagle's algorithm holds a small write until the previous segment is
acknowledged; delayed ACK holds the ACK for up to ~40 ms on Linux (200 ms on some stacks). A request
written as two small writes (headers, then body) can stall 40–200 ms each time. `TCP_NODELAY`
(`socket.setNoDelay`; Node's HTTP client and server default to it since Node 18) or coalescing into one
write fixes it.

**HTTP/2 and gRPC.** One TCP connection multiplexes many streams, so a lost packet stalls all of them
(TCP head-of-line blocking), and connection-level (L4) balancing — including a Kubernetes `ClusterIP`
service via kube-proxy — pins all of a client's calls to one backend, so new pods get no traffic after
a scale-out. Fixes: L7 balancing (Envoy, a mesh), client-side balancing with periodic re-resolution
(headless services), or `MAX_CONNECTION_AGE` on the server so clients reconnect and redistribute. A
connection's `MAX_CONCURRENT_STREAMS` (often 100) queues extra calls silently.

**The cluster network is part of your latency.** kube-proxy in iptables mode rewrites large rule sets
on every Service or endpoint change, and on big clusters those updates take seconds and cost CPU on
every node; IPVS or eBPF data planes (Cilium) scale better. Conntrack tables fill under connection churn
and drop packets. CoreDNS load rises with every scale event and every pod's `ndots` expansion (`§2`).
Mesh sidecars add a hop and a config push per change. Latency spikes that coincide with cluster
events — node additions, large deploys — rather than with your traffic point here.

**Chattiness.** N+1 calls, per-item RPCs in loops and serial dependent calls dominate latency more than
CPU does. Batch (the DataLoader pattern: collect keys within one tick, issue one call), pipeline
(Redis pipelining, HTTP/2 multiplexing) and parallelise independent calls with a bound. Compression
costs CPU: gzip at level 6 on a hot internal path can cost more than the bandwidth it saves inside one
data centre; skip it for small bodies and prefer cheap codecs or none internally.

**Payload size.** Big JSON costs serialisation CPU on both ends, GC churn and memory spikes. Field
selection, pagination and binary encodings (Protobuf) reduce all three.

### 16. Data structures and algorithmic cost in production (`C18`)

- **Accidental quadratic**: `array.includes` or `find` inside a loop, `Array.prototype.shift` in a
  loop (O(n) each), string concatenation in a loop where strings are immutable (Java without
  `StringBuilder`), `list.remove` / `list.pop(0)` in Python, and spreading in a reduce
  (`acc = { ...acc, [k]: v }` is O(n²)). Invisible at 100 items, fatal at 100,000 — typically when the
  biggest customer onboards.
- **Hash flooding**: keys chosen to collide degrade a hash table to a list. Java 8+ `HashMap` turns long
  buckets into trees (O(log n)); V8 and Python seed their string hashes; parsers of query strings,
  JSON and form bodies are the attack surface, so cap parameter counts and body sizes.
- **ReDoS**: nested quantifiers `(a+)+`, overlapping alternations `(a|a)*`, and `.*` followed by
  something that fails, applied to input you do not control. V8's Irregexp backtracks. Fixes: rewrite
  the pattern, limit input length, use RE2 (linear time, the `re2` package), or V8's experimental
  linear engine. One request pinning a Node process at 100% for 30 s is the signature.
- **Big-O ignores constants and memory layout**: a linear scan of a contiguous array often beats a tree
  or hash map up to thousands of elements thanks to cache locality; Java's `LinkedList` is almost always
  slower than `ArrayList`. Boxed collections (`List<Long>`, Python lists of ints) cost several times the
  memory of primitive arrays and load the GC.
- **V8 shapes**: objects built with consistent property order share hidden classes and stay fast;
  `delete obj.x` or many dynamic keys push an object into dictionary mode and call sites megamorphic.
  Use `Map` for dictionaries with dynamic keys.
- **Pagination and sorting**: `OFFSET 100000` is linear in the offset; keyset pagination is constant.
  Fetching everything to sort in memory costs O(n log n) plus the memory for n.
- **Top-k with a bounded heap** is O(n log k) time and O(k) memory, versus sorting everything.
- **Probabilistic structures** give fixed memory at scale: HyperLogLog for cardinality, Bloom filters
  for membership, Count-Min sketches for frequency.

### 17. Diagnosis methodology and tooling (`C13`)

**Method before tool.**
- **USE** (Brendan Gregg): for every resource — CPU, memory, disk, network, *and software resources such
  as pools, locks and queues* — check **U**tilisation, **S**aturation (queue length, time waiting) and
  **E**rrors. Saturation is the one people skip, and it is where latency lives.
- **RED** for services: rate, errors, duration.
- **Off-CPU analysis**: a request that is slow while CPU is idle is waiting — on a lock, a pool, I/O, a
  throttle, a page fault. On-CPU flame graphs cannot show it; wall-clock profiling (async-profiler
  `-e wall`, `py-spy record --idle`, `clinic bubbleprof`) and off-CPU flame graphs (BCC `offcputime`)
  can.
- **Differential diagnosis**: good pod against bad pod, before against after the deploy, node against
  node, AZ against AZ. The difference is usually found faster than the code is read.
- **Change correlation**: most incidents follow a change — a deploy, a config flag, a traffic-mix shift,
  a dependency's deploy, a certificate rotation, a base image or kernel update, a data-size threshold.

**The 60-second Linux checklist** (Gregg): `uptime`, `dmesg -T | tail` (OOM kills, TCP drops),
`vmstat 1` (`r` run queue, `cs` context switches, `si/so` swap, `st` steal), `mpstat -P ALL 1` (one
hot core means a single-threaded bottleneck), `pidstat 1`, `iostat -xz 1` (`await`, `%util`),
`free -m`, `sar -n DEV 1`, `sar -n TCP,ETCP 1` (retransmits), `top`. Linux load average counts threads
in uninterruptible sleep (D state, usually disk or NFS), so a load of 40 on four cores with idle CPU
means blocked I/O, not CPU.

**Production-safe samplers.** async-profiler and JFR (JVM), `py-spy` (attaches to a PID without a
restart), `perf` with `--perf-basic-prof` (Node), `pprof` (Go), and continuous profilers (Pyroscope,
Parca, vendor agents) that keep flame graphs over time so you can diff yesterday against now. eBPF tools
— `bpftrace`, BCC's `runqlat` (scheduler run-queue latency), `tcpretrans`, `biolatency`, `execsnoop`
— answer kernel questions without code changes. `perf stat -e cycles,instructions,cache-misses` gives
instructions per cycle (IPC): the same code running at a much lower IPC on one node than another is
evidence of cache or memory-bandwidth contention from a neighbour, which no per-pod CPU metric shows.

**Microbenchmark traps**: dead-code elimination, missing warm-up, constant folding, timer resolution.
Use JMH on the JVM, `mitata` or `tinybench` in Node, `pyperf` in Python, and report distributions.

**Observer effects.** Heap dumps and `jmap` stop the world; `strace` can slow the traced process 10–100×
(prefer `perf trace` or eBPF); debug logging at high rates becomes the bottleneck; APM agents add
overhead (`§4`); `--inspect` left enabled in production is a remote-code-execution port.

### 18. Overload, retries and metastable failure (`C14`, `C16`, `M09`)

**Metastable failure** (Bronson et al., HotOS 2021): a system stable at a load it handled yesterday is
pushed into a bad state by a trigger (a brief spike, a cache flush, a GC pause, a deploy) and **stays**
bad after the trigger has gone, because a feedback loop sustains the overload. Common loops:
- **Retry amplification**: three layers each retrying three times turn one failing call into 3³ = 27
  calls at the bottom. Fix: retry at one layer, retry budgets (retries capped at ~10% of requests, as
  gRPC and Finagle do), exponential backoff with **full jitter**, circuit breakers.
- **Timed-out work still done**: clients time out but servers keep processing, so goodput falls to zero
  while throughput stays high. Deadline propagation and dropping requests that have waited too long fix
  it.
- **Cache-miss storms**: a flush or cold start sends everything to the database, whose latency rises,
  which slows cache refill. Fix: request coalescing, stale-while-revalidate, warming, and rate-limiting
  cache-miss traffic to the database.
- **Connection storms**: slow dependencies make clients open more connections, each new TLS handshake
  costs CPU on an already-overloaded server, and reconnection after an outage arrives as a synchronised
  herd.
- **Memory and GC**: overload raises in-flight requests, which raises the live heap, which raises GC
  time, which lowers throughput, which raises in-flight requests.

The defining property is that recovery needs load well *below* the trigger level — shed load, drain
queues, or restart and ramp traffic back gradually.

**Autoscaling traps.** Scaling a Node or I/O-bound service on CPU scales late or never. New pods arrive
cold (JIT, empty caches, connection set-up) and add load while contributing little. Scaling the app
multiplies database connections (50 pods × a pool of 20 = 1,000 connections, far past what Postgres
runs well). Scale on saturation (ELU, queue depth, in-flight requests), cap maximum replicas by the
downstream's capacity, and put PgBouncer in transaction mode between the two.

**Graceful shutdown.** After SIGTERM a pod keeps receiving traffic for a few seconds while endpoint
removal propagates. Sleep in a `preStop` hook (5–15 s), stop accepting, drain in-flight requests, close
keep-alive connections (`Connection: close`), and exit before `terminationGracePeriodSeconds` (default
30 s). Node as PID 1 gets no default signal handling, so an unhandled SIGTERM does nothing until the
SIGKILL — use `tini`/`--init` or handle SIGTERM explicitly. Without all this every deploy shows a burst
of 502s.

---

## Questions

### Level 1 — Everyday latency and a blocked runtime

The incidents every backend engineer meets in the first year on call; the bar is a fast, evidence-led diagnosis.

1. Our Node API's p99 went from 80 ms to 900 ms after we added an "export my data" endpoint, but the export endpoint itself is rarely called and the others did not change. CPU on the pods averages 30%. What is going on?
   > **Direction:** One thread serves every request, so a synchronous serialisation in the export blocks every other request on that process; prove it with `monitorEventLoopDelay` and a `--cpu-prof` capture, then stream or move the export to a worker (§1, `C03`, `C04`).

2. A Node service's latency is fine for 10 minutes, then every 10 minutes it spikes for about 300 ms across all endpoints. Nothing in the logs. Where do you look?
   > **Direction:** A periodic synchronous job (a cache refresh doing a big `JSON.parse`, a timer rebuilding an in-memory index) shares the loop; correlate event-loop max lag with the timer's schedule and chunk it with `setImmediate` or move it off-process (§1, `C03`).

3. We log every request body with `JSON.stringify` for audit. At 5k rps latency is fine; after a customer started sending 4 MB payloads the whole service slowed down. Why does one customer affect everyone?
   > **Direction:** `JSON.stringify` of a multi-MB body is several ms of blocked loop per request, shared by all tenants on the process; log a truncated or size-capped representation and enforce body limits (§1, §15, `C03`).

4. Our Kubernetes liveness probe on a Node service keeps failing under peak load, the pods restart, and the outage gets worse each time. The probe just returns `200 OK`. How can a trivial endpoint fail?
   > **Direction:** The probe shares the blocked event loop so it answers late; restarts shift load to survivors and cascade — make liveness tolerant, move saturation into readiness, and fix the blocking (§1, §18, `C03`).

5. A Python FastAPI service does 200 rps comfortably, but a single slow third-party call (sometimes 3 s) makes every endpoint slow while it is in progress. The code uses `async def` everywhere. What did we miss?
   > **Direction:** A sync client (`requests`) inside an `async def` blocks the asyncio loop; enable asyncio debug mode to log slow callbacks, then use `httpx.AsyncClient` or `asyncio.to_thread` (§8, `C07`).

6. After a refactor our checkout endpoint went from 120 ms to 1.4 s. The profiler shows almost no CPU. The code now does `for (const item of cart.items) { await priceService.get(item.id) }`. Explain the regression and fix it without overloading the price service.
   > **Direction:** Accidental serialisation — N round trips in sequence; use bounded concurrency (`p-map` with a limit) or a batch endpoint via a DataLoader-style collector (§5, §15, `C05`, `C17`).

7. A Node job processor does bcrypt hashing for sign-ups and reads templates from disk. During a sign-up campaign, template reads and outbound HTTP calls slowed from 5 ms to 2 s, but event-loop lag stayed under 10 ms. Explain.
   > **Direction:** Async bcrypt, `fs` and `dns.lookup` all share the libuv threadpool of 4, so hashing starves file reads and DNS for outbound calls while the loop sits idle; isolate hashing and size `UV_THREADPOOL_SIZE` via the environment (§2, `C02`, `C03`).

8. We set `process.env.UV_THREADPOOL_SIZE = 32` at the top of our `app.js` and saw no improvement. Why?
   > **Direction:** The pool is created on first use and some imported module had already touched `fs` or DNS; the variable must be set in the process environment before start (container env or `node` wrapper) (§2, `C03`).

9. A Java Spring Boot service gets slower under load but CPU is at 25% and the DB reports idle. Thread dumps show 200 threads. What is your next step and what are you expecting to see?
   > **Direction:** Take three thread dumps a few seconds apart and look for threads parked in the same frame — typically `HikariPool.getConnection` or a blocking HTTP client — then apply Little's Law to the pool sizes (§7, §10, §17, `C06`, `C14`).

10. Our endpoint p50 is 40 ms and p99 is 2.1 s. The team wants to optimise the SQL query because "that's the slow part". How would you decide whether that is right?
    > **Direction:** A 50× p99/p50 ratio usually means waiting (GC, a pool, a lock, a retry, a timeout) rather than slow work; inspect the histogram shape and exact spike values, and trace the slow requests specifically before optimising the median path (§10, §17, `C13`, `C14`).

11. A new endpoint that renders a PDF with a JavaScript library blocks for 2 s each time. Product wants it on the main API. What are your options and which do you pick?
    > **Direction:** Keep CPU work off the request loop — a worker-thread pool (Piscina) for moderate volumes, or a separate service/queue so it scales and fails independently; never a worker per request (§1, `C04`).

12. We have a latency histogram for outbound calls with a clear second peak at exactly 5 seconds. What does that tell you?
    > **Direction:** An exact 5 s mode is the resolver retry timeout: lost UDP DNS packets (conntrack race or `ndots` search-list expansion); fix with NodeLocal DNSCache, `single-request-reopen`, trailing-dot FQDNs, or connection reuse (§2, §10, `A13`).

13. Some requests to our service take 40 ms longer than they should, but only for one specific internal client written in a different language. Server-side timing shows 2 ms. How do you investigate?
    > **Direction:** A fixed ~40 ms (or 200 ms) extra is Nagle interacting with delayed ACK when the client writes headers and body in separate small writes; confirm with `tcpdump` and fix with `TCP_NODELAY` or a single write (§15, `A12`, `C17`).

14. Every morning at 09:00 our error rate spikes for two minutes with timeouts, then recovers. Traffic rises but only by 20%. Where do you start?
    > **Direction:** Look for synchronised work at the top of the hour — cron jobs, cache TTLs expiring together, batch reports — plus cold caches meeting a traffic ramp; jitter schedules and TTLs and pre-warm (§12, §18, `C14`).

15. After upgrading a logging library our Node service's throughput dropped 30%. The flame graph shows a wide block in `writeSync`. What is happening and what are the trade-offs of fixing it?
    > **Direction:** Synchronous writes to stdout (a pipe to a slow collector) block the loop; use an asynchronous transport (pino with a worker-thread transport) and accept the risk of losing buffered lines on crash (§1, §12, `C03`, `C13`).

16. We deployed and the p99 of every endpoint doubled for 15 minutes, then went back to normal — on every deploy. The service is Java. What do you think it is?
    > **Direction:** JIT warm-up (interpreted then C1 then C2) competing with request threads, worsened by a tight CPU limit; use load-balancer slow start, warm-up traffic before readiness, a start-up CPU boost, or AppCDS (§7, §9, `C06`).

17. A gunicorn Django service logs `WORKER TIMEOUT` a few times an hour and those users get 502s. The endpoints involved call a payment provider. What is going on and what would you change first?
    > **Direction:** Sync workers each hold one request, and the default 30 s timeout kills a worker stuck on a slow provider call; set explicit client timeouts well under the worker timeout, and move slow calls to async workers or a job (§8, `C07`, `M09`).

18. A developer "fixed" a CPU-heavy loop by adding `await Promise.resolve()` every 1,000 iterations, but other requests still stall. Why, and what is the correct yield?
    > **Direction:** Awaiting a resolved promise only queues a microtask, which runs before the loop returns to I/O; yield with `setImmediate` (check phase) or move the work to a worker (§1, §5, `C03`, `C05`).

19. Our Node service handles 3k rps on a 4-vCPU pod and autoscales on CPU at 70%, but it never scales and latency is terrible at peak. Why?
    > **Direction:** A single Node process can only use about one core for JavaScript, so pod CPU caps near 25% while the loop is saturated; scale on event-loop utilisation or in-flight requests, or run one process per core (§1, §18, `C03`, `C04`).

20. A single request to our search endpoint with a crafted query parameter pinned a Node process at 100% CPU for 40 seconds. What happened and how do you stop it happening again?
    > **Direction:** Catastrophic regex backtracking (ReDoS) on user input; rewrite the pattern, cap input length, or use a linear-time engine such as RE2 (§1, §16, `C18`).

### Level 2 — Memory growth, leaks and OOMKills

Almost every senior interview includes one; the signal is knowing which memory is growing and how to capture evidence without making the incident worse.

1. Our Node service's memory grows about 50 MB an hour until Kubernetes kills it every two days. Restarting "fixes" it. Walk me through finding the leak in production.
   > **Direction:** Check whether `heapUsed` or only RSS grows; then use the three-snapshot technique on a drained pod and follow retainers to a GC root, pairing with allocation sampling (§3, §4, `C12`, `C13`).

2. Pods are being OOMKilled with exit code 137, but our Node heap metrics show `heapUsed` flat at 300 MB of a 1 GB limit. How can that be?
   > **Direction:** The growth is off-heap — Buffers, native addons or glibc malloc arena fragmentation — which RSS and the cgroup count but the heap does not; compare `rss` with `external`/`arrayBuffers` and try `MALLOC_ARENA_MAX=2` or jemalloc (§3, §9, `C12`).

3. We found a module-level `Map` caching user permission lookups. The developer says "it's fine, we only have 200k users". Is it?
   > **Direction:** An unbounded cache keyed by unbounded cardinality is a leak with a hit rate, and it never sees revocations; bound it with an LRU plus TTL and consider memory per entry × keys (§4, `C12`, `Q01`).

4. We see `MaxListenersExceededWarning: Possible EventEmitter memory leak detected` in logs. A colleague proposes `emitter.setMaxListeners(0)`. What do you say?
   > **Direction:** The warning is a leak detector — something is adding a listener per request to a long-lived emitter without removing it; find the call site from `--trace-warnings` and use `once` or remove on completion (§4, `C12`).

5. Our Node pods restart with OOMKilled but never print `FATAL ERROR: Reached heap limit`. Why no JavaScript error, and what would you change?
   > **Direction:** V8's heap limit was above the container limit, so the kernel killed the process before V8 hit its own ceiling; set `--max-old-space-size` to ~75% of the limit and enable `--heapsnapshot-near-heap-limit` (§3, §9, `C12`).

6. We took a heap snapshot of a 3 GB Node process in production to find a leak, and the pod died. What went wrong and how would you do it next time?
   > **Direction:** Snapshots stop the world and need memory comparable to the heap, so the pod hit its limit; drain it from the load balancer, raise its memory temporarily, or capture near the limit automatically, and prefer allocation sampling on live pods (§4, `C12`, `C13`).

7. A Python gunicorn service with `--preload` and 8 workers used 1.2 GB total at start-up, and 6 GB after a day, with no leak visible in `tracemalloc`. What is happening?
   > **Direction:** Copy-on-write is being undone by reference-count and GC header writes, so each worker gradually privatises the shared pages; measure PSS, call `gc.freeze()` before fork, and move big read-only data out of Python objects (§8, `C07`, `C12`).

8. A Java service on a 4 GB container has `-Xmx4g` and is OOMKilled every few hours with no `OutOfMemoryError` and no heap dump. Explain.
   > **Direction:** Non-heap memory (metaspace, code cache, thread stacks, direct buffers, GC structures) pushes RSS past the limit; size the heap at ~70–75% with `MaxRAMPercentage` and account for the rest with Native Memory Tracking (§7, §9, `C06`, `C12`).

9. The same Java service, with no JVM flags at all, runs on a 4 GB container and hits `OutOfMemoryError: Java heap space` at only 1 GB of heap. Why?
   > **Direction:** The default `MaxRAMPercentage` is 25%, giving a 1 GB heap on a 4 GB container; set it explicitly (§7, `C06`).

10. After a single large import, a Python worker's RSS went from 300 MB to 2 GB and never came down, even though the import finished and objects were freed. Is this a leak?
    > **Direction:** Not necessarily — pymalloc and glibc keep freed memory in fragmented arenas; confirm with `tracemalloc` that live objects dropped, then use worker recycling (`--max-requests` with jitter) or process the import in a separate short-lived process (§8, §3, `C07`, `C12`).

11. Our Prometheus metrics endpoint on a Node service grew to 80 MB and scraping takes 10 s; the process memory grows with it. What would you look for?
    > **Direction:** A label cardinality leak — user ids, raw URLs with ids, or error messages as label values; normalise routes and remove unbounded labels (§4, `C12`, `O06`).

12. A Node service's memory grows only when a downstream is slow. Once the downstream recovers, memory stays high for an hour and then falls. What is being held?
    > **Direction:** In-flight promises and their closures, buffered request/response bodies and queued work accumulate with latency (Little's Law); memory tracks concurrency, so bound in-flight work and add timeouts that actually cancel (§5, §10, §14, `C05`, `C16`).

13. Heap snapshots show thousands of retained `IncomingMessage` objects, retained through a closure in a `setInterval` inside our metrics middleware. How can one closure retain so much?
    > **Direction:** Closures created in the same scope share a context object, so a long-lived interval callback pins every variable any sibling closure captured — including the request; clear the interval and avoid capturing request scope in long-lived callbacks (§4, `C12`).

14. A service using `Promise.race([fetchData(), timeout(2000)])` leaks memory steadily when the upstream hangs. Why doesn't the timeout free anything?
    > **Direction:** The losing promise and its socket keep running and holding references; the timeout only stopped the caller waiting — abort the request with an `AbortSignal` so it is torn down (§4, §5, `C05`).

15. Our JVM service's heap usage after full GC climbs slowly for a week, then the service falls over. We can't reproduce it in staging. What is your plan?
    > **Direction:** Use JFR's old-object sample and a class histogram trend in production, or a heap dump from a drained instance analysed with dominator trees in Eclipse MAT; watch for caches, ThreadLocals and listener lists (§6, `C12`, `C13`).

16. A container is OOMKilled although the app's own memory is stable. It writes uploaded files to `/tmp` before processing. What is the likely cause?
    > **Direction:** `/tmp` or an `emptyDir` backed by memory (tmpfs) is charged to the container's memory cgroup; use a disk-backed volume or stream uploads (§9, §14, `O01`).

17. Our memory alert fires on `container_memory_usage_bytes` every night during a batch export, but nothing is ever killed. Is the alert wrong?
    > **Direction:** Usage includes reclaimable page cache from writing files; alert on `container_memory_working_set_bytes`, which the OOM killer effectively acts on (§9, `O01`).

18. We added `AsyncLocalStorage` for request-scoped logging context, and memory grew and throughput dropped 15%. How do you tell whether it is our code or the context propagation?
    > **Direction:** Context propagation adds per-promise cost and can retain large context objects on long-lived resources; A/B it on a canary, keep only small values in the store, and check the Node version's implementation (§4, `C05`, `C12`).

19. The Java team switched to virtual threads and memory usage doubled, though request rate is unchanged. What is the first thing you would check?
    > **Direction:** Per-thread state multiplies with thread count — `ThreadLocal` buffers and caches that were bounded by a 200-thread pool now exist per virtual thread; move to shared or scoped values (§7, `C06`).

20. We use `sharp` for image resizing in Node. RSS climbs to the container limit under load while the V8 heap is small, and it only happens on the image service. What do you try?
    > **Direction:** Native allocations from libvips across multiple threads fragment glibc arenas; set `MALLOC_ARENA_MAX`, use jemalloc, cap `sharp.concurrency`, and bound concurrent resizes (§2, §3, `C12`).

### Level 3 — Pools, threads and saturation

Where throughput caps out before CPU does; the signal is reasoning with Little's Law and knowing which pool is actually full.

1. Our Spring Boot service tops out at exactly 200 rps no matter how many CPU cores we give it. Average latency is about 1 s because of a slow downstream. Why exactly 200?
   > **Direction:** Little's Law: 200 Tomcat threads ÷ 1 s per request = 200 rps; the cap is concurrency, not CPU — raise concurrency safely (virtual threads, async client) or cut the downstream latency, and bound the downstream separately (§7, §10, `C06`, `C14`).

2. We raised Tomcat's `threads.max` from 200 to 2,000 to handle a traffic spike and the service got slower and then fell over. Explain.
   > **Direction:** More threads moved the queue to the 10-connection Hikari pool and the downstream, added memory for stacks and context switching, and pushed the database past its knee; size pools from the bottleneck outward and shed load instead (§7, §10, §14, `C06`, `DB21`).

3. Thread dumps show 180 of 200 request threads in `TIMED_WAITING` at `HikariPool.getConnection`, yet the database reports only 10 active queries, all fast. What is holding the connections?
   > **Direction:** Connections are checked out but idle — held across a remote HTTP call or long in-app work inside a `@Transactional` method; enable Hikari's `leakDetectionThreshold`, and shorten transactions to exclude remote calls (§7, §17, `C06`, `DB21`).

4. A service uses `CompletableFuture.supplyAsync(() -> jdbcCall())` without an executor. It works in development, but in production on 2-CPU pods everything that uses parallel streams stalls too. Why?
   > **Direction:** The default is the common ForkJoinPool, sized cores − 1 = 1 thread in the pod, shared by the whole JVM; blocking JDBC starves it — always pass a dedicated, bounded executor for blocking work (§7, `C06`).

5. At full load only, our batch service hangs completely with CPU at zero. Thread dumps show every worker thread waiting on a `Future.get()`. There is no "Java-level deadlock" in the dump. What happened?
   > **Direction:** A nested submission deadlock: tasks on a bounded pool submitted subtasks to the same pool and blocked on them, so all threads are parents waiting for children; use separate pools per stage or non-blocking composition (§7, §12, `C06`, `C09`).

6. We migrated to Java 21 virtual threads. Under load the service now stalls for seconds with almost no CPU, and it never did that on platform threads. What would you check first?
   > **Direction:** Carrier-thread pinning — blocking inside `synchronized` (common in older JDBC drivers and pools) holds one of the few carriers; find it with `jdk.tracePinnedThreads` or JFR `VirtualThreadPinned`, then use `ReentrantLock`, upgrade the library, or move to JDK 24+ (§7, `C06`).

7. After switching to virtual threads, the database started rejecting connections and the team says "but virtual threads are supposed to make us scale". What changed?
   > **Direction:** Virtual threads removed the implicit 200-thread bulkhead, so tens of thousands of requests now reach the connection pool and downstreams at once; add an explicit `Semaphore` or bounded pool in front of scarce resources (§7, §14, `C06`, `M10`).

8. A Python service with gunicorn sync workers, 4 workers per pod, handles 40 rps and the team wants to add more pods. Downstream calls take ~100 ms on average. What would you do instead?
   > **Direction:** 4 sync workers × (1 ÷ 0.1 s) is about 40 rps — concurrency is the cap, not CPU; move to `gthread` or an async worker class for I/O-bound work, after checking with Little's Law (§8, §10, `C07`).

9. We moved sync Django ORM calls into async views with `sync_to_async`. Under load the async views are slower than the old sync ones. Why?
   > **Direction:** `sync_to_async` defaults to `thread_sensitive=True`, running all those calls on one shared thread — a global serialisation point; use a native async driver path or `thread_sensitive=False` where safe (§8, `C07`).

10. An asyncio service offloads blocking work with `loop.run_in_executor(None, fn)`. On a 64-core box it performs fine; on 2-CPU pods it queues badly at modest load. Why?
    > **Direction:** The default executor is `min(32, cpu + 4)` threads — 6 on a 2-CPU pod; size a dedicated executor for the I/O it actually blocks on, or use async clients (§8, `C07`, `C14`).

11. Our Node service calls an internal API over HTTPS at 3k rps. Under load we see `EADDRNOTAVAIL` and connection timeouts, while the target reports low load. What is exhausted?
    > **Direction:** Ephemeral ports stuck in `TIME_WAIT` because every request opens a new connection (no keep-alive agent before Node 19); use a keep-alive agent with bounded `maxSockets` (§15, `C17`, `A12`).

12. We enabled keep-alive on our Node HTTP client and now get a steady trickle of `ECONNRESET` / "socket hang up" errors on the first request after quiet periods. Why, and how do you fix it without disabling keep-alive?
    > **Direction:** The server or an intermediary closes idle connections before the client stops reusing them; set the client idle timeout shorter than the server's (and the server's longer than the LB's), and retry idempotent requests once on reset (§15, `C17`).

13. Behind an AWS ALB, our Node service returns intermittent 502s at low traffic, with nothing in the app logs. What default is biting you?
    > **Direction:** Node's `server.keepAliveTimeout` (5 s) is shorter than the ALB idle timeout (60 s), so the ALB reuses a connection Node just closed; set `keepAliveTimeout` to ~65 s and `headersTimeout` just above it (§15, `C17`).

14. A consumer service reads from SQS with 10 concurrent workers per pod. Throughput plateaus at 50 messages/s per pod while CPU is 15%. Processing each message calls an API with 200 ms latency. What do you tell the team?
    > **Direction:** 10 ÷ 0.2 s = 50 msg/s — Little's Law caps it; raise in-flight concurrency per pod (async processing with a bound) before adding pods, checking the API's own limits (§10, §14, `C14`, `C16`).

15. We use a `ThreadPoolExecutor` from `Executors.newFixedThreadPool(50)` for outbound calls. During a downstream slowdown the service's memory grew to the limit and latency went to minutes. Why was nothing ever rejected?
    > **Direction:** `newFixedThreadPool` uses an unbounded `LinkedBlockingQueue`, so it queues forever instead of rejecting; use a bounded queue with a rejection policy and a timeout so overload fails fast (§7, §14, `C06`, `C16`).

16. A Node service does a bcrypt hash with cost 12 on login. Login latency at peak is 3 s even though each hash takes 250 ms. The pods have 4 vCPUs. What is the arithmetic, and what is the fix?
    > **Direction:** The threadpool of 4 allows 4 concurrent hashes = 16 hashes/s per process; a queue forms beyond that — size the pool or a worker pool to the cores, scale on queue length, and rate-limit login attempts (§2, §10, `C04`).

17. A gRPC client in Node sends 5k rps to a backend over a single channel. Latency grows under load even though the backend is idle. What might limit you?
    > **Direction:** One HTTP/2 connection with `MAX_CONCURRENT_STREAMS` (often 100) queues excess calls client-side, and one TCP connection also serialises on a single core and a single loss window; use several channels/sub-channels (§15, `C17`, `A10`).

18. Our Python Celery workers use `prefetch_multiplier=4` and `concurrency=8`. Short tasks queue behind long ones and some workers sit idle while others have a backlog. Why?
    > **Direction:** Each worker reserves 32 messages up front, so long tasks trap short ones behind them while idle workers have nothing to take; set prefetch to 1 with late acks for long tasks, or route long and short tasks to separate queues (§8, §10, `C07`, `Q15`).

19. A Java service has 200 Tomcat threads and a 20-connection DB pool. The team wants to set the pool to 200 "so threads never wait". What is your answer?
    > **Direction:** The DB's throughput peaks at a small number of active connections (roughly cores × 2); 200 connections per pod × N pods oversubscribes it and slows every query — keep the pool small and let requests queue briefly, or shed them (§7, §10, `DB21`).

20. After a Node upgrade our outbound HTTP calls got faster but a downstream team complains we now hold 4,000 open connections to them. What changed?
    > **Direction:** Node 19+ made the global agent keep-alive by default, with `maxSockets` unlimited, so every concurrent request keeps its own pooled socket; set `maxSockets`/`maxFreeSockets` and an idle timeout deliberately (§15, `C17`).

### Level 4 — Containers, CPU quotas and resource accounting

Kubernetes-era incidents where the application is fine and the platform is lying to you; the signal is knowing how cgroups account for CPU and memory.

1. Our Java service's average CPU is 35% of its 2-CPU limit, but p99 latency has 100 ms spikes every few seconds. There is no GC problem in the logs. What is your first hypothesis and how do you prove it?
   > **Direction:** CFS quota throttling — bursts of many threads exhaust the 100 ms period's quota and the pod freezes for the remainder; check `cfs_throttled_periods` / `cfs_periods` and `cpu.stat`, then cut thread counts or remove the limit (§9, `C01`).

2. A platform engineer proposes removing CPU limits from all latency-sensitive services. The security team says that is dangerous. Make the argument either way.
   > **Direction:** Limits cause throttling at low average CPU while requests (CFS weight) already guarantee a fair share under contention; the risks are noisy neighbours and unbounded bursts, mitigated by right-sized requests, node capacity and keeping memory limits (§9, `O01`).

3. The same JVM image runs fine on 4-CPU pods, but on 1-CPU pods it has multi-second pauses and much lower throughput. Nothing else differs. Why?
   > **Direction:** With one CPU the JVM treats itself as a non-server-class machine and picks Serial GC, a single-thread common pool and minimal JIT threads; set the GC explicitly or `-XX:ActiveProcessorCount`, or give it 2 CPUs (§6, §7, §9, `C06`).

4. We moved a thread-heavy Java service from 16-core to 96-core Kubernetes nodes and throttling got much worse at the same CPU limit. Why would bigger nodes hurt?
   > **Direction:** Thread pools and GC thread counts are sized from visible cores, so more threads burn the quota in a shorter burst each period; pin counts with `ActiveProcessorCount` and explicit pool sizes (§7, §9, `C06`).

5. Our p99 shows periodic spikes that align with GC in the JVM logs, but the logged pauses are 15 ms while users see 90 ms. What bridges the gap?
   > **Direction:** A parallel GC phase with many threads empties the CFS quota and the whole container is then throttled for the rest of the period; fewer GC threads (`ParallelGCThreads`/`ConcGCThreads`), a higher limit, or no limit (§6, §9, `C12`).

6. A Node pod with a 1 GB memory limit is OOMKilled under load. `--max-old-space-size` is not set. The team says V8 "knows" the container limit. Does it?
   > **Direction:** Only recent Node versions derive the heap limit from the cgroup, and even then off-heap memory is outside it; log `v8.getHeapStatistics().heap_size_limit` at start-up and set the flag to ~75% explicitly (§3, §9, `C12`).

7. Our Grafana panel shows a pod at 98% of its memory limit for days, but it is never OOMKilled. Another pod at 70% was killed an hour ago. Explain both.
   > **Direction:** The first panel includes reclaimable page cache; the second pod's working set spiked quickly (a large allocation or burst) between scrapes; use working-set metrics and container-level OOM events, not averages (§9, `O01`).

8. A gunicorn container with 9 workers is OOMKilled, and after the restart all nine workers come back cold at the same time and latency is terrible for 5 minutes. What happened and what would you change?
   > **Direction:** In a multi-process container the kernel kills one process — possibly the master, taking all workers — and the restart is cold; size memory for workers × peak, recycle workers with jitter, and use `--preload` plus `gc.freeze` to share memory (§8, §9, `C07`).

9. After moving from EC2 `m5` to `t3` instances to save money, the service runs well all morning and then latency doubles every afternoon. No deploys. What is the cause?
   > **Direction:** Burstable instances exhaust CPU credits and drop to baseline performance; check CPU credit balance and steal time, and use unlimited mode or fixed-performance instances for latency-critical work (§9, `O01`).

10. Latency is bad only on some pods, and the bad set changes after each reschedule. The code and version are identical. How do you find the cause?
    > **Direction:** Slice by node — noisy neighbours competing for cache, memory bandwidth, disk or network, or a node-level issue (throttling, steal, a sick kernel); use differential diagnosis good-pod vs bad-pod and node-level metrics (§9, §17, `C13`).

11. Our Node service's CPU usage in the container shows 100% of one core and the dashboard says 25% of the 4-CPU limit. The team wants to reduce the limit to 1 CPU to save money. What do you say?
    > **Direction:** A single-process Node service only uses one core for JavaScript, but libuv threads, GC helper threads and compression use more in bursts; at 1 CPU those bursts get throttled — measure throttling and ELU before cutting, or run one process per core (§1, §2, §9, `C03`).

12. We set `-Xmx` to 75% of the container limit, yet a Netty-based service is still OOMKilled during traffic spikes. What memory are we not counting?
    > **Direction:** Direct (off-heap) buffers used by Netty, plus thread stacks and metaspace; cap `-XX:MaxDirectMemorySize`, enable Native Memory Tracking, and watch pooled allocator metrics (§7, §9, `C12`, `C19`).

13. Our Java service's latency is great for a week after deploys, then degrades steadily, and a reboot of the node (not the pod) fixes it. Where would you look?
    > **Direction:** Node-level state: transparent huge page compaction stalls, fragmentation, or swap on the host; check `khugepaged`, THP settings (`madvise`), and major page faults (§6, §9, §17).

14. On a 4-core pod with a 4-CPU limit, a Python service running 8 gunicorn workers with 4 threads each shows heavy throttling and poor latency at modest load. Why, and what would you set?
    > **Direction:** 32 runnable threads contending for a 4-CPU quota burst through it and get throttled, and CPU-bound threads also fight over each worker's GIL; size workers to cores (roughly 2 × cores + 1 for I/O-bound sync) and cap threads (§8, §9, `C07`).

15. The service gets throttled even when `kubectl top` says it uses 20% of its limit. The platform team says throttling "can't happen below the limit". Who is right?
    > **Direction:** Quota is per 100 ms period, so bursts throttle while the minute average is low; older kernels also had a throttling bug — the throttling counters, not averages, are the evidence (§9, `O01`).

16. Our JVM starts in 40 s on 0.5 CPU but in 8 s on 2 CPUs, and on 0.5 CPU the readiness probe fails and the pod is restarted forever. How do you fix it without permanently paying for 2 CPUs?
    > **Direction:** Start-up (class loading, JIT) is CPU-bound and throttled; use a startup probe with a long budget, a start-up CPU boost or in-place resize, AppCDS or CRaC, and a lower steady-state request (§7, §9, `C06`).

17. Our pods run with `emptyDir: { medium: Memory }` for a scratch directory used by an image-conversion step. Memory alerts fire with a flat heap and OOMKills follow. Explain.
    > **Direction:** tmpfs pages are charged to the container's memory cgroup, so scratch files count as memory; switch to disk-backed `emptyDir`, stream the conversion, or raise the limit consciously (§9, §14).

18. We see `%st` at 15% on our VM-based services during business hours. What does it mean for your latency, and what can you do as an application team?
    > **Direction:** Steal time is CPU the hypervisor gave to other tenants; your threads are runnable but not running, which appears as latency with no code cause — move to dedicated or larger instance types, and account for it in capacity headroom (§9, §17).

19. An asyncio Python service is at 60% CPU but the event loop lags badly. `py-spy top` shows a thread that isn't the event loop consuming most of the time. What might that be?
    > **Direction:** A background CPU-bound thread (metrics aggregation, a compression step, a C extension holding the GIL) competes for the GIL and delays the loop thread by switch intervals; move it to a process or release-the-GIL code (§8, §17, `C07`).

20. After a base image change from Debian to Alpine, a Node and a Python service both became slower and used more memory. Nothing in the app changed. What would you suspect?
    > **Direction:** musl libc's allocator and DNS resolver behave differently from glibc — slower malloc under threaded load, different resolver behaviour (no `single-request-reopen`, different search handling), and wheels built from source without optimisations; benchmark allocator and resolver, or move back to a glibc image (§2, §3, §9).

### Level 5 — Garbage collection and runtime pauses

Pauses that no application log explains; the signal is reading GC and safepoint evidence rather than guessing flags.

1. A Java API with a 16 GB G1 heap has occasional 3–4 second pauses. The GC log shows "to-space exhausted" just before each one. What is happening and what do you change?
   > **Direction:** Evacuation failure followed by a full GC — the heap is too full to copy live objects; raise headroom (heap size or lower `InitiatingHeapOccupancyPercent`), find the live-set growth or leak, and check for humongous allocations (§6, `C12`).

2. Our G1 logs show frequent "humongous allocation" entries and concurrent cycles starting early. The service parses large JSON documents into byte arrays. What would you do?
   > **Direction:** Objects of half a region or more go straight into old regions; increase `G1HeapRegionSize`, stream the parsing instead of buffering whole documents, or reuse buffers (§6, §14, `C12`, `C19`).

3. We switched a latency-sensitive service from G1 to ZGC. GC pauses in the log dropped to under 1 ms, but p99 latency got worse under peak load. How is that possible?
   > **Direction:** ZGC pauses are tiny but it needs headroom; when allocation outruns the concurrent collector threads hit allocation stalls, which are not pauses — look for `Allocation Stall` in the log and give it more heap or reduce allocation (§6, `C12`).

4. A user-visible stall of 600 ms happens every few minutes on a JVM service, but no GC pause in the log exceeds 20 ms. Where else can the JVM stop the world?
   > **Direction:** Time-to-safepoint — one thread slow to reach a safepoint stalls all others; enable `-Xlog:safepoint` and compare "reaching" time with operation time, and look for long counted loops, agents, or disk-blocked GC logging (§6, `C06`, `C12`).

5. Every time an engineer runs a diagnostic on a production JVM "to have a look", that pod has a latency spike. What are they probably running, and what should they use instead?
   > **Direction:** `jmap -histo:live` and heap dumps force full GCs or stop the world; use JFR (`jcmd JFR.start`) and async-profiler, which are designed for production (§6, §17, `C13`).

6. A JVM service shows multi-second pauses only on one noisy node, and the GC log shows `real` time much higher than `user + sys` during those pauses. What does that tell you?
   > **Direction:** Wall time far above CPU time means the GC threads were not running — waiting on swap, blocked writing the GC log to a saturated disk, or throttled; check swap, disk I/O on the node, and throttling, and use asynchronous GC logging (§6, §9, `C12`).

7. Our Node service's p99 correlates with heap size: it is fine after restart and degrades over 6 hours to 5× worse, then the pod crashes. What is the mechanism?
   > **Direction:** As old space nears the limit V8 runs mark-compact more often and reclaims less — the GC death spiral; confirm with `--trace-gc` or GC performance entries, then find the leak (§3, §4, `C12`).

8. A high-throughput Node service spends 15% of its CPU in scavenges according to the profiler. Heap size is stable and there is no leak. What unconventional knob would you try, and what's the trade-off?
   > **Direction:** Raise `--max-semi-space-size` so the young generation is larger and scavenges less often (fewer objects promoted too), at the cost of more memory and slightly longer individual scavenges; also cut allocation in the hot path (§3, `C12`).

9. The Java team set `-XX:MaxGCPauseMillis=10` on G1 to cut latency. Throughput fell and the pauses did not meet 10 ms. Why?
   > **Direction:** The pause target is a soft goal G1 meets by collecting less per cycle, so it collects more often and falls behind; a live set that needs more evacuation cannot hit 10 ms — use ZGC for that goal or reduce the live set (§6, `C12`).

10. A batch-processing JVM uses ZGC because "it's the modern one". The job takes 30% longer than on the old cluster. What would you suggest?
    > **Direction:** Batch jobs care about throughput, not pauses — Parallel GC is often faster; choose the collector per workload (§6, `C12`).

11. Our OOM handling flag `-XX:+HeapDumpOnOutOfMemoryError` is set. After an OOM, the pod hung for 4 minutes before being killed and the dump was truncated. What went wrong?
    > **Direction:** Writing a heap-sized dump to a slow or small volume stops the world and can exceed the grace period or disk; write to a sized persistent volume, use `ExitOnOutOfMemoryError` for fast restart, and rely on JFR for evidence (§6, §9, `C12`).

12. A Python service shows regular 200 ms pauses on requests, and they correlate with no external event. The service builds large object graphs per request. What would you check?
    > **Direction:** The cyclic garbage collector's generation-2 collections scale with the number of tracked objects; check with `gc.callbacks` timing, reduce cycles and object counts, raise thresholds, or `gc.freeze()` long-lived objects (§8, `C07`, `C12`).

13. A service's p99 went up after a heavily used feature flag was switched on, though the new code is cheap. CPU rose by 20% for 10 minutes and then settled. What might the JIT be doing?
    > **Direction:** A new type or path invalidated optimised code (deoptimisation), so hot methods recompiled; watch JIT compilation events in JFR and expect a transient, or warm the new path (§6, §7, `C06`).

14. A Java service using Kafka consumers suffers regular rebalances, and each correlates with a GC pause of about 12 seconds in the logs. What is the chain of events and what would you fix?
    > **Direction:** A long pause stops polling and heartbeats; the consumer exceeds its session or poll interval, is evicted, and the rebalance redistributes work — fix the GC pause (heap sizing, collector) and keep heartbeats and processing time within the timeouts (§6, §14, `C12`, `Q15`).

15. A Node service uses a large in-memory LRU with 5 million entries for performance. Each major GC takes 400 ms. The cache hit rate is excellent. What do you do?
    > **Direction:** A large live set of small objects makes mark-compact expensive; move the cache off-heap (a Buffer-based store or Redis/local shared memory), store compact serialised values, or shrink it — trade some hit rate for pause time (§3, §16, `C12`, `C18`).

16. A JVM service with 2 CPUs and a 2 GB heap uses G1. Under load, GC concurrent marking can't keep up and full GCs happen. More memory is not available. What levers remain?
    > **Direction:** Concurrent GC threads compete with application threads under a small quota; start marking earlier (`InitiatingHeapOccupancyPercent`), reduce allocation rate with an allocation profile, or trade to Parallel GC if pauses are acceptable (§6, §9, §17, `C12`).

17. During an incident you want to know whether a Node process is spending time in GC right now, without a restart or new flags. How?
    > **Direction:** Use a `PerformanceObserver` for `gc` entries already exported as a metric, or attach the inspector with `SIGUSR1` on a drained instance; plan ahead by exporting GC time and event-loop delay continuously (§3, §4, `C13`).

18. A Java service's GC log is written to a volume that is shared with a noisy application log. Sometimes GC pauses of 5 ms become 2 s. Explain.
    > **Direction:** Synchronous GC log writes happen at the safepoint, so a disk blocked by other writers extends the pause; use `-Xlog:async` or log to tmpfs (§6, `C12`).

19. The team disabled transparent huge pages on database hosts years ago but not on the JVM application nodes. You see sporadic stalls with high `sys` time. How would you investigate?
    > **Direction:** THP compaction by `khugepaged` or direct compaction during allocation stalls the process in the kernel; check `/sys/kernel/mm/transparent_hugepage` settings and compaction stats, and set it to `madvise` (§6, §17).

20. A Python gunicorn service with `--preload` shows memory growing across workers after every full GC run, even though no objects leak. Why would the garbage collector itself cause memory growth?
    > **Direction:** The cyclic collector writes to object headers as it scans, dirtying copy-on-write pages shared from the master, so each worker privatises pages; `gc.freeze()` before forking stops the collector touching them (§8, `C07`, `C12`).

### Level 6 — Queueing, tail latency and load testing

Where averages hide the problem; the signal is quantitative reasoning — utilisation, fan-out, and honest measurement.

1. A service runs at 85% CPU and the team says it has "15% headroom". Traffic is expected to grow 10% next month. What do you tell them?
   > **Direction:** Queueing delay grows as ρ/(1−ρ), so going from 85% to 94% roughly triples queueing delay; plan capacity to stay around 60–70% at peak (§10, §11, `C14`, `C15`).

2. Our load test says p99 is 120 ms at 5k rps, but production shows 900 ms at 4k rps. The test uses 200 virtual users in a loop. What is wrong with the test?
   > **Direction:** A closed-loop generator suffers coordinated omission — it stops sending during stalls and under-records slow requests; use a constant-arrival-rate (open-model) tool and measure from intended send time (§11, `C15`).

3. A page that calls 30 backend services in parallel has p50 of 400 ms, while each backend's p50 is 20 ms and p99 is 350 ms. Explain the page's median.
   > **Direction:** With fan-out the page waits for the slowest of 30; 1 − 0.99³⁰ ≈ 26% of pages hit at least one p99 and most hit a p90+, so backend tails become the page's median — hedge, reduce fan-out, or use deadlines with partial results (§10, `C14`).

4. Engineering wants to add hedged requests to a payment API to reduce tail latency. What do you say?
   > **Direction:** Hedging is only safe for idempotent reads; for payments it risks double execution unless every call carries an idempotency key the server enforces — and hedging must be bounded (e.g. after p95) to cap extra load (§10, `C14`, `M09`).

5. The dashboard shows a service's p99 as the average of each pod's p99, and it looks fine. Users complain. What is wrong with the metric?
   > **Direction:** Percentiles cannot be averaged — aggregate histogram buckets across pods then compute the quantile; one bad pod's tail disappears in a mean of p99s (§10, `C14`, `O14`).

6. Our latency histogram is bimodal: one peak at 15 ms and a second at 180 ms. What are the first three explanations you would test?
   > **Direction:** Two populations: cache hit versus miss, a bad pod/AZ/node, or a periodic pause (GC, throttling) — split the histogram by pod, node, cache status and route to find which (§9, §10, §17, `C14`).

7. A service with 10 workers each taking one request at a time handles 99% of requests in 10 ms, but 1% take 2 s (report exports). Overall p99 is terrible even for fast requests. Why, and what is the fix that doesn't make exports faster?
   > **Direction:** High service-time variance makes queues far worse (Kingman); fast requests queue behind slow ones — isolate slow work into a separate pool or queue so the fast path has its own capacity (§10, `C14`, `M10`).

8. We run 20 instances behind a round-robin load balancer. p99 is poor although average utilisation is 50%. Switching to "least outstanding requests" improved it a lot. Why?
   > **Direction:** Round-robin ignores per-instance queue length, so slow instances accumulate queues; load-aware balancing approximates a shared queue (M/M/c), which cuts waiting (§10, `C14`, `M07`).

9. A 30-minute load test showed the service handles 8k rps comfortably. It fell over at 6k rps on the second day in production. What didn't the test exercise?
   > **Direction:** Duration and realism — leaks, old-generation growth, cache cardinality, connection churn, log rotation and warm-cache effects only appear in soak tests with production-like data and key distributions (§11, `C15`).

10. As we add application instances, total throughput increases up to 12 instances and then drops. The database is not saturated. How can adding instances lower throughput?
    > **Direction:** Retrograde scaling from coherency costs (USL β > 0) — lock contention, shared-row updates, cache invalidation traffic or connection churn grows with N; find the shared coordination point and remove or partition it (§11, §12, `C15`).

11. You're asked to size a new service for 3,000 rps with a p99 SLO of 200 ms. Each request takes ~40 ms of processing and 1 core. How do you estimate instance count, and what margin do you add?
    > **Direction:** 3,000 × 0.04 = 120 cores busy at 100%; target ~60–70% utilisation for queueing headroom (~175–200 cores), then add N+1 per AZ and verify with a step-load test against the SLO, not peak throughput (§10, §11, `C14`, `C15`).

12. Our latency distribution has a spike at exactly 1 s and another at exactly 3 s on connection-heavy paths. What is the kernel telling you?
    > **Direction:** TCP SYN retransmission timeouts — connects are being dropped (full accept queue, conntrack table, SYN backlog, a security group); look at `ListenOverflows`, `somaxconn` and conntrack, and use connection reuse (§10, §14, §15, `A12`).

13. A search service fans out to 50 shards and waits for all of them. The p99 of the search is 5× the p99 of any shard. You cannot make the shards faster. What can you do?
    > **Direction:** Return partial results at a deadline, hedge or tie requests to shard replicas, and replicate hot or slow shards — accepting slightly less complete or slightly more expensive answers for a much better tail (§10, `C14`).

14. Our k6 test ramps from 0 to 10k VUs in 30 seconds and the service falls over at 3k. Product says real traffic never ramps that fast. Is the test useless?
    > **Direction:** It tests the cold-start and autoscaling path, not steady-state capacity — useful for flash-crowd readiness, but capacity should come from a step-wise constant-arrival-rate test with warm-up (§7, §11, §18, `C15`).

15. We added a cache in front of a slow service. Average latency halved but p99 got worse. How can a cache make the tail worse?
    > **Direction:** Misses now take the slow path plus the cache round trip, misses can stampede on the same key, and the traffic mix left for the backend is the expensive tail; coalesce misses and look at the miss-path latency separately (§5, §10, `C14`, `Q01`).

16. A queue-based worker system processes 1,000 jobs/s. The average job time is 50 ms and each worker runs one job at a time. How many workers do you need, and what happens at 950 jobs/s with 50 workers?
    > **Direction:** 1,000 × 0.05 = 50 busy workers is 100% utilisation — unstable; at 950 jobs/s utilisation is 95% and waiting grows to ~19× service time, so provision ~70–80 workers (§10, `C14`).

17. Our production load test "squeezes" traffic onto fewer pods to find capacity. The SRE team is nervous. How do you run it safely and what do you learn that synthetic tests don't tell you?
    > **Direction:** Shift weights gradually with automatic rollback on SLO breach, one AZ at a time; you learn real capacity with the true request mix, cache behaviour and dependencies, which synthetic tests miss (§11, `C15`).

18. The latency panel shows p99 at 200 ms but users report multi-second page loads. Each page makes 25 sequential API calls. Reconcile.
    > **Direction:** Per-call percentiles understate per-page experience: 25 sequential calls hit the tail often and sum; measure the user-facing operation end to end and cut sequential calls (§10, §15, `C14`, `C17`).

19. A load test on a Node service shows throughput plateaus at 60% CPU on the load test box. The team concludes the service is the limit. What do you check first?
    > **Direction:** The load generator — a saturated or single-threaded generator (or its event loop) produces coordinated omission and false plateaus; check generator CPU and ELU, and distribute the load (§1, §11, `C15`).

20. A service's latency SLO is met at 95th percentile but not 99.9th. The business wants 99.9th improved. What class of techniques changes the far tail, as opposed to the median?
    > **Direction:** The far tail is dominated by rare events — GC pauses, throttling, retries, cold caches, slow replicas — so fixes are isolation, hedging, removing pauses and timeout/retry tuning rather than making the median path faster (§6, §9, §10, `C14`).

### Level 7 — Backpressure, streams and data flow

Less common, higher signal: the candidate has built pipelines that survived a slow consumer.

1. A Node service exports a 2 GB CSV from Postgres to S3. It works for small exports, but large ones OOM the pod at 1.5 GB. The code reads rows with a cursor and calls `upload.write(row)` in a loop. What is wrong?
   > **Direction:** The loop ignores `write()` returning `false`, so rows buffer in memory faster than S3 accepts them; use `stream.pipeline()` with a transform so backpressure flows from S3 back to the cursor (§14, `C16`, `C19`).

2. A streaming download endpoint works fine on office Wi-Fi but memory spikes when mobile users on slow connections download large files. Why do slow clients cost memory?
   > **Direction:** If the handler writes to `res` without honouring `false`/`'drain'`, the response is buffered in memory at the server's read speed; pipe with backpressure so a slow client slows the source (§14, `C16`, `C19`).

3. We replaced `.pipe()` chains with `stream.pipeline()` during a refactor and a slow file-descriptor leak disappeared. Why?
   > **Direction:** `.pipe()` does not destroy the other streams when one errors, so a failed destination left source file handles open; `pipeline` propagates errors and destroys all streams (§14, `C19`).

4. During a downstream brownout, our service's queue of pending jobs grows to millions and, after the downstream recovers, it takes hours to catch up — processing jobs whose users have long since given up. What design change would you make?
   > **Direction:** Bound the queue, attach deadlines and drop expired work, and consider LIFO when overloaded so fresh requests are served; unbounded queues turn overload into hours of useless work (§14, §18, `C16`).

5. An API gateway returns 504 after 10 s, but the backend keeps processing those requests for up to 60 s. During a traffic spike the backend gets slower and slower. What is the mechanism and the fix?
   > **Direction:** Timed-out work is still performed, so goodput collapses; propagate deadlines, cancel work when the client disconnects (`AbortSignal` on `req` close), and drop queued requests past their deadline (§5, §14, §18, `C16`).

6. A Kafka consumer group keeps rebalancing every few minutes during peak. Each consumer processes batches of 500 records that call a slow API. Consumer lag grows and some messages are processed twice. What's happening?
   > **Direction:** Processing a batch exceeds `max.poll.interval.ms`, so the consumer is kicked out and its uncommitted batch is redelivered — a rebalance storm; lower `max.poll.records`, pause partitions, bound concurrency and commit completed offsets (§14, `C16`, `Q15`).

7. We fixed a slow endpoint by adding a fixed concurrency limit of 50. It works on Monday, rejects too much on Tuesday when the downstream is fast, and is too high on Wednesday when it is slow. What better approach exists?
   > **Direction:** An adaptive concurrency limit (AIMD or gradient on observed latency, like TCP congestion control) finds the current capacity automatically (§14, `C16`).

8. A Node service consumes a websocket feed at 20k messages/s and writes to Postgres. Memory grows steadily during market open and drops afterwards. How do you apply backpressure to a push-based source?
   > **Direction:** Push sources can't be slowed directly, so bound the in-memory buffer, batch inserts, and choose explicitly between pausing the socket (TCP backpressure to the producer), dropping, or conflating messages — whichever the domain allows (§14, `C16`).

9. `Promise.all(users.map(sendEmail))` over 80,000 users crashed the email service and our process ran out of memory. What should the code look like instead?
   > **Direction:** Bounded concurrency (`p-map` with a limit, or a queue), ideally streaming the user list rather than loading it — so memory and downstream load are both bounded (§5, §14, `C05`, `C16`).

10. Our Node service reads a large JSON upload with `await req.json()` and then processes it. Memory spikes at 3–4× the body size and the loop blocks. What is the alternative?
    > **Direction:** Buffering then `JSON.parse` holds the raw bytes, the string and the object graph at once, and parses synchronously; use NDJSON or a streaming parser and process records incrementally, with a body size limit (§1, §14, `C19`).

11. After adding gzip compression middleware, our Node service's throughput halved and latency rose, even though the responses are only a few KB. Why?
    > **Direction:** Compression is CPU and threadpool work per response; for small bodies inside a data centre it costs more than it saves — set a size threshold, a lower level, or compress at the edge (§2, §15, `C17`).

12. A Java service's executor queue is bounded at 1,000 with `CallerRunsPolicy`. Under overload, request-handling threads started running background tasks themselves and HTTP latency exploded. Explain the trade-off.
    > **Direction:** `CallerRunsPolicy` applies backpressure by making the submitter do the work, which here is the request thread — it blocks the front door; for request paths prefer fast rejection with a 503 and a separate pool for background work (§7, §14, `C06`, `C16`).

13. An ingestion service reads from S3 and writes to a database. Throughput is 30% of what either side can do alone. CPU is low. The pipeline is: read file → parse → write, one file at a time. How would you speed it up?
    > **Direction:** Sequential stages leave each resource idle while the other works; pipeline with bounded buffers between stages and bounded parallelism per stage, sized by Little's Law (§10, §14, `C16`, `C19`).

14. A service serving large static files from disk uses 40% CPU at 2 Gbps. A colleague says "that's just what TLS costs". What could reduce it?
    > **Direction:** Copying file bytes through user space costs CPU; `sendfile`-style zero-copy helps only without user-space TLS, so offload TLS to kernel TLS or the load balancer, or serve from a CDN (§14, §15, `C19`).

15. Our logging pipeline sidecar goes down for 10 minutes and our application's p99 goes up 20×, even though logging is "fire and forget". Why?
    > **Direction:** Logs written synchronously to stdout block when the pipe's reader stops draining; use async logging with a bounded buffer and an explicit drop policy (§12, §14, `C09`, `C16`).

16. A gRPC streaming service sends updates to thousands of subscribers. One slow subscriber's buffer grew to 800 MB. What is missing from the design?
    > **Direction:** Per-subscriber flow control — honour HTTP/2 flow-control readiness (`isReady`/`onReady`), bound each subscriber's buffer, and drop or conflate for slow consumers, or disconnect them (§14, §15, `C16`).

17. After scaling a consumer group from 12 to 48 consumers, throughput didn't change. The topic has 12 partitions. What else is wrong beyond the obvious?
    > **Direction:** Partitions cap consumer parallelism, so 36 consumers are idle; beyond that, per-partition ordering serialises work — process within a partition with bounded parallelism keyed by entity, committing only contiguous completed offsets (§14, `C16`, `Q15`).

18. A Python asyncio service creates a task per incoming message with `asyncio.create_task` and never awaits them. Memory grows and sometimes tasks disappear before finishing. Explain both symptoms.
    > **Direction:** Unbounded task creation has no backpressure, and the event loop keeps only weak references to tasks, so unreferenced tasks can be garbage-collected mid-flight; keep references in a set, bound concurrency with a semaphore or `TaskGroup` (§8, §14, `C07`, `C16`).

19. A downstream's rate limit is 100 requests/s. Our 20 pods each use a local limiter of 5 rps. We still get 429s. Why?
    > **Direction:** Per-pod limiters only sum correctly when traffic is even and the pod count is fixed; bursts, autoscaling and retries exceed the global budget — use a shared token bucket, honour `Retry-After`, and apply backpressure to callers (§14, §18, `C16`).

20. A batch job reads 10 million rows with `OFFSET`/`LIMIT` pagination in pages of 1,000. The first pages take 20 ms and the last ones take 8 s. What is the cost model and the fix?
    > **Direction:** Offset pagination is linear in the offset, so total cost is quadratic; use keyset pagination (`WHERE id > last_id`) or a server-side cursor streamed with backpressure (§14, §16, `C18`).

### Level 8 — Contention, locks and scheduling

Rarer, and a strong senior signal: diagnosing contention when every resource looks idle.

1. Adding threads to a Java metrics aggregator made it slower: 4 threads handled 2M events/s and 16 threads handled 1.2M. Each thread increments its own slot in a shared `long[]`. What's going on?
   > **Direction:** False sharing — adjacent slots share a cache line that bounces between cores; pad the slots, use `@Contended` or `LongAdder`, and confirm with `perf c2c` (§12, `C08`, `C10`).

2. A service has low CPU, high context switches per second (500k) and threads mostly `BLOCKED` on the same monitor in thread dumps. The critical section is a few microseconds. Why is throughput so poor?
   > **Direction:** A lock convoy — each handover costs a context switch, so throughput is bounded by scheduling rather than work; shrink or stripe the lock, batch work per acquisition, or avoid a fair lock (§12, `C09`).

3. We made a lock fair (`new ReentrantLock(true)`) to stop some requests starving. Overall throughput dropped by 10×. Why?
   > **Direction:** Fair locks forbid barging, forcing a handoff and context switch on every release; fairness costs throughput — prefer reducing contention, or bounded unfairness (§12, `C09`).

4. A profiler shows 30% of CPU in `java.util.Random.next` in a service that generates request ids. What happened and how do you fix it in one line?
   > **Direction:** A shared `Random` uses CAS on one seed, so threads contend and retry; use `ThreadLocalRandom.current()` (or per-thread generators) (§12, `C08`, `C10`).

5. Our service started up fine in the data centre but after moving to fresh cloud VMs it sometimes takes 3 minutes to start, hanging while creating TLS contexts. What is the classic cause?
   > **Direction:** `SecureRandom` blocking on `/dev/random` entropy on a freshly booted VM in old JDKs and libraries; use the non-blocking source (`urandom`) and modern JDKs (§12, `C09`).

6. A Java service deadlocks once a week under peak load. Thread dumps say "Found one Java-level deadlock" between two locks, one in the cache layer and one in the audit logger. How do you fix it structurally?
   > **Direction:** Break circular wait with a global lock ordering, or restructure so no code calls out (logging, callbacks) while holding a lock (§12, `C09`).

7. A Node service's "rare" deadlock: a request awaits a result from a background queue, but the queue worker runs on the same process and awaits a lock held by the request. It only happens under load. What class of bug is this, and how do you find it?
   > **Direction:** An async deadlock — circular waiting between promises, invisible to thread dumps; find it with request timeouts and async stack traces showing both waits, and fix by never holding an in-process lock across an await of dependent work (§5, §12, `C05`, `C09`).

8. Health checks on a busy Java service time out under load, the orchestrator restarts pods, and the outage gets worse. The health endpoint is served by the same Tomcat pool. What's the principle and the fix?
   > **Direction:** Priority inversion — the control plane waits behind bulk traffic; serve health on a separate port and pool (Spring's management port), and make liveness independent of load (§1, §12, §18, `C09`).

9. After a Redis restart, all 400 application pods reconnected at once and the Redis primary spent a minute at 100% CPU doing TLS handshakes while clients timed out and reconnected again. How do you avoid this next time?
   > **Direction:** A thundering herd of reconnections sustained by timeouts; add jittered, exponential reconnect backoff, connection pooling with limits, and longer connect timeouts during recovery (§12, §18, `C09`).

10. A popular cache key expires every 60 seconds, and each time the database sees a spike of 2,000 identical queries. What is the minimal-code fix, and what is the more robust one?
    > **Direction:** Single-flight / promise coalescing in each process is minimal; stale-while-revalidate with a background refresh, jittered TTLs, or a distributed lock for recompute is more robust across pods (§5, §12, `C09`, `Q01`).

11. A service handles 90% of CPU in `Collections.synchronizedMap` get/put under load. The team proposes `ConcurrentHashMap`. Is that enough, and what could still go wrong?
    > **Direction:** It removes the global lock, but compound operations (check then put) are no longer atomic — use `computeIfAbsent`/`merge` — and a long computation inside `computeIfAbsent` blocks that bin (§12, `C09`, `C10`).

12. Our CPU is 100% but throughput is 30% of expected. The profiler shows most time in a `compareAndSet` retry loop on a shared counter. Why is a lock-free algorithm spinning uselessly?
    > **Direction:** Under heavy contention CAS retries waste work (livelock-like); stripe the state (`LongAdder`), batch updates, or add backoff — lock-free is not contention-free (§12, `C10`).

13. A Python service uses threads for a mix of CPU-heavy JSON processing and I/O calls. Adding threads made I/O-bound requests slower. Why?
    > **Direction:** CPU-bound threads hold the GIL for switch intervals and every I/O thread must re-acquire it after each syscall — a GIL convoy; move CPU work to processes or C code that releases the GIL (§8, §12, `C07`).

14. We rewrote a service on Java virtual threads, one thread per request. It works well, except a `synchronized` in-memory rate limiter now caps throughput at 2k rps. What do you do?
    > **Direction:** A global lock in the hot path serialises everything, and under virtual threads `synchronized` can also pin carriers; partition or stripe the limiter, use atomics or `ReentrantLock`, or move it to per-key structures (§7, §12, `C06`, `C09`).

15. Every hour on the hour, our cluster's CPU spikes and job latency doubles. All services schedule a cleanup job with `0 * * * *`. What would you change?
    > **Direction:** A cron thundering herd; add random jitter to schedules, spread by hash of the instance id, or use a single coordinator (§12, §18).

16. A service's p99 got worse after a well-meant change to log with a global `synchronized` formatter so lines don't interleave. What would you suggest?
    > **Direction:** A hidden global lock in the logging path; use per-thread formatters and an async appender with a bounded queue, or structured logging that writes whole lines atomically (§12, `C09`).

17. A worker pool with one queue per worker (tasks assigned by hash) has uneven latency: some workers have long queues while others are idle. Switching to a shared queue helped a lot. Why, in queueing terms, and when would you still keep per-worker queues?
    > **Direction:** A shared queue is M/M/c, which beats c separate M/M/1 queues; keep per-worker (or per-key) queues only when ordering or cache affinity per key matters, and then use work stealing (§10, §12, `C10`, `C14`).

18. We read-lock a large configuration map on every request and write-lock it when config changes every minute. Under load the service stalls for seconds after each config change. Why?
    > **Direction:** The writer waits for all readers and blocks new ones (or starves), creating a stall; publish immutable snapshots with an atomic reference swap instead of a read-write lock (§12, `C09`, `C10`).

19. A batch job holds a database row lock while calling an external API; meanwhile the web tier needs the same row. Web latency spikes during the batch. What is this pattern and how do you fix it?
    > **Direction:** Priority inversion across systems: a low-priority holder blocks high-priority requests; do not hold locks across remote calls — reorder to call first then lock briefly, or use optimistic concurrency (§12, `C09`, `DB09`).

20. A high-throughput Node service writes metrics to a UDP StatsD agent. At 50k rps the service spends 20% CPU in `send` syscalls. What would you change?
    > **Direction:** One syscall per metric is expensive; aggregate in-process and flush periodically, batch several metrics per packet, or use a pull model — many small syscalls cost more than the data (§14, §15, `C17`).

### Level 9 — Distributed coordination, leases and time

Rarely asked well, and a clear senior separator: correctness when processes pause and clocks lie.

1. We use a Redis lock (`SET NX PX 30000`) so only one worker processes each invoice. Twice this month an invoice was charged twice. Logs show worker A took the lock, then a 35-second gap, then A wrote the charge after worker B had already done so. What happened, and why doesn't "check the lock before writing" fix it?
   > **Direction:** A paused (GC, throttling, blocked loop) past the TTL and resumed believing it held the lock; any check can be followed by another pause — fence at the resource with a monotonically increasing token or make the charge idempotent with a unique key (§13, `C11`).

2. The team proposes switching to Redlock across five Redis nodes to make the invoice lock "safe". What is your response?
   > **Direction:** Redlock still relies on bounded pauses and clock drift and gives no fencing token, so it does not fix a paused holder; for correctness use consensus-backed leases with fencing or a DB constraint, and keep Redis locks for efficiency only (§13, `C11`).

3. How would you implement fencing when the protected resource is a Postgres table and the lock comes from etcd?
   > **Direction:** Use the etcd lease/key revision as the token, store the last token per resource row, and write with `UPDATE ... WHERE fence < :token` so stale holders' writes affect zero rows (§13, `C11`, `DB09`).

4. A Node worker holds a lease on a job and renews it every 10 s with a 30 s TTL. Occasionally two workers run the same job. The Node process sometimes parses 200 MB files. Connect the two facts.
   > **Direction:** Parsing blocks the event loop for longer than the TTL, so the renewal timer fires late and the lease expires; move parsing to a worker thread, check renewal results and abort on failure (§1, §13, `C03`, `C11`).

5. Our leader-elected scheduler occasionally runs a job twice across two instances right after a network blip. The election uses Kubernetes Lease objects. What's going on?
   > **Direction:** The old leader acted on a stale belief after losing the lease (renewal deadline passed during the blip); stop work when renewal fails and fence side effects with the lease's generation or an idempotency key (§13, `C11`).

6. A service computes timeouts with `Date.now()` differences. After an NTP correction, a batch of requests timed out instantly and some never timed out. Why?
   > **Direction:** Wall clocks can step backwards and forwards; use a monotonic clock (`performance.now`, `process.hrtime`, `System.nanoTime`, `time.monotonic`) for durations (§13, `C11`, `M21`).

7. We order events from two services by their wall-clock timestamps and sometimes a "cancel" appears before the "create" it cancels. What's the right way to reason about this?
   > **Direction:** Clocks across machines are not ordered; use causal ordering — a sequence from one source, logical or hybrid logical clocks, or version numbers — and never rely on timestamps for correctness (§13, `M21`).

8. A Kafka consumer paused for 40 s by a GC pause wakes up and commits offsets for a partition that has already been reassigned to another consumer. What protects you, and what doesn't?
   > **Direction:** The group generation id fences the stale commit (it is rejected), but side effects the zombie already performed are not fenced — make processing idempotent or transactional (§13, §14, `C11`, `Q15`).

9. Our distributed lock TTL is 10 s. SRE wants to raise it to 5 minutes to "stop the double processing". What are the consequences?
   > **Direction:** A longer TTL reduces expiry-under-pause but slows failover when a holder really dies and still does not guarantee safety; measure worst pauses, renew with abort-on-failure, and fence for correctness (§13, `C11`).

10. A cron-like job runs on every pod but should run once. The team uses a Postgres advisory lock taken with `pg_try_advisory_lock` on a pooled connection. Sometimes it runs twice, sometimes never. Why?
    > **Direction:** Session-level advisory locks belong to a connection; with pooling (especially PgBouncer transaction mode) the lock can be released or held by a connection the job no longer uses — use transaction-scoped `pg_try_advisory_xact_lock` inside the job's transaction or a job table with row locks (§13, `C11`, `DB09`).

11. Your lock service is etcd. During a leader election in etcd lasting 5 s, several application jobs stalled and then two ran concurrently. What should the application have done?
    > **Direction:** When the lock service is unavailable, renewals fail — the holder must stop work at its local deadline (computed from a monotonic clock minus a safety margin) rather than assume it still holds the lease; fencing catches the rest (§13, `C11`).

12. A payment worker uses a DB row lock (`SELECT ... FOR UPDATE`) to serialise charges for an account, then calls the payment provider inside the transaction. Under load, lock waits spike. How do you keep correctness but release the lock?
    > **Direction:** Don't hold row locks across remote calls: record an intent with an idempotency key, commit, call the provider with that key, then record the result — idempotency replaces the long lock (§12, §13, `C11`, `DB09`).

13. A VM live migration froze a Java process for 8 seconds. What classes of bugs does that expose in a system that seemed correct under normal operation?
    > **Direction:** Lease and lock expiry while paused, missed heartbeats causing false failovers, timers firing in bursts, and wall-clock jumps; anything relying on bounded pauses needs fencing, monotonic clocks and idempotency (§6, §13, `C11`).

14. A rate limiter keyed in Redis uses `INCR` then `EXPIRE` as two commands. Under load, some keys never expire and users are blocked permanently. Why?
    > **Direction:** The two commands are not atomic, so a crash or failure between them leaves a key with no TTL; use a Lua script or `SET` with `EX` and `NX`, or `INCR` in a `MULTI` with the expiry (§13, `C11`, `Q08`).

15. After a Redis primary failover, two workers both believe they hold the same lock. Nothing paused. How?
    > **Direction:** Replication is asynchronous, so the lock write had not reached the replica that was promoted; Redis locks are not safe across failover — fence at the resource or use a consensus store (§13, `C11`).

16. Two instances of a stateful stream processor both processed the same partition for 30 seconds after a deploy. How do systems like Kafka Streams or Flink prevent a zombie instance from committing results?
    > **Direction:** Epoch fencing — transactional producer epochs and group generations reject writes from the old instance; your own external side effects need the same epoch or idempotency (§13, `C11`, `Q15`).

17. Our job scheduler has a "heartbeat every 5 s, dead after 15 s" rule. During peak load healthy workers get declared dead and their jobs reassigned, overloading the others. What's the fix?
    > **Direction:** Heartbeats on the same thread or loop as heavy work are starved (priority inversion), causing false failures and a cascade; put heartbeats on a dedicated thread, use accrual failure detection, and make reassignment rate-limited (§12, §13, §18, `C11`).

18. A system uses a lock around "read balance, compute, write balance" in Redis with a TTL of 2 s. Latency to Redis spiked to 3 s during an incident. What went wrong and what would you do instead?
    > **Direction:** The lock expired mid-operation, so two writers interleaved; replace lock-read-write with an atomic operation (a Lua script or `WATCH`/`MULTI` optimistic transaction), or move the invariant to a store with transactions (§13, `C10`, `C11`, `Q08`).

19. You need exactly one instance to send a daily report email. The team proposes a distributed lock. What would you suggest that is simpler and more robust?
    > **Direction:** Make the side effect idempotent with a unique key per report date (a DB unique constraint or a job table row), so duplicates are harmless and no lock is needed for correctness (§13, `C11`).

20. We use ZooKeeper ephemeral nodes for leader election. A leader with a long GC pause lost its session, and a new leader started. When the old one resumed, it kept writing to shared storage for 2 seconds. How do you stop that?
    > **Direction:** The old leader cannot know it lost the session until it hears from ZooKeeper; the storage must reject its writes via a fencing token (the znode's sequence number or `zxid`), and the leader should check session state before side effects (§13, `C11`).

### Level 10 — Metastable failures and cross-layer mysteries

The rarest and hardest; the answer spans runtime, kernel, network and system dynamics, and recovery is part of the question.

1. A 30-second database blip happened at 14:00. The database recovered at 14:00:30, but our service stayed at 100% errors for 40 minutes, until we cut traffic by half. What kind of failure is this, and what sustained it?
   > **Direction:** A metastable failure: retries, timed-out-but-still-running work and cold caches kept load above capacity after the trigger ended; recovery needed load well below normal — add retry budgets, deadlines and load shedding (§18, `C14`, `C16`).

2. Three layers of services each retry failed calls three times with 100 ms backoff. A brief slowdown in the bottom service turned into a full outage. Quantify and fix.
   > **Direction:** 3³ = 27× amplification at the bottom plus synchronised backoff; retry at one layer only, with retry budgets, exponential backoff with full jitter and circuit breakers (§18, `M09`).

3. After a cache cluster restart, the database was overwhelmed; the cache never refilled because every query timed out before it could populate the cache. How do you recover now, and prevent it in future?
   > **Direction:** A cache-miss metastable loop; recover by shedding traffic and warming the cache gradually, prevent it with request coalescing, stale-while-revalidate, rate-limited cache-miss traffic to the DB and warm restarts (§5, §18, `Q01`).

4. We autoscale on CPU. During a spike new pods came up, and latency got worse; the database hit `max_connections` and started rejecting all connections, including existing pods'. Explain the chain and the design fix.
   > **Direction:** Each new pod adds a full connection pool and cold-start load, multiplying DB connections past its capacity; cap replicas by downstream capacity, pool via PgBouncer in transaction mode, and scale on saturation signals (§7, §18, `DB21`).

5. A Node service behind a Kubernetes Service has three pods. After scaling to six, the new pods get almost no traffic from a gRPC client service, while the old ones stay hot. Why, and what are the options?
   > **Direction:** HTTP/2 long-lived connections are balanced at connection time by L4 kube-proxy, so existing connections stay on old pods; use L7 balancing (a mesh or Envoy), client-side balancing with a headless service, or `MAX_CONNECTION_AGE` to force reconnection (§15, `C17`, `M07`).

6. Every deploy causes a burst of about 2% 502 errors for 20 seconds. The service handles SIGTERM "correctly" by closing the server. What is still wrong?
   > **Direction:** Endpoints are removed asynchronously, so traffic arrives after shutdown starts, and keep-alive clients reuse connections the server is closing; add a `preStop` sleep, keep serving while draining, send `Connection: close`, and check the process actually receives SIGTERM as PID 1 (§15, §18).

7. A Java service's p99 degrades only on nodes that also run a particular batch job, but CPU and memory for our pods look unchanged. The batch job is memory-bandwidth heavy. How would you prove the interference and what would you do?
   > **Direction:** Noisy-neighbour contention for L3 cache and memory bandwidth is invisible in per-pod CPU; correlate p99 by node with co-scheduling, check hardware counters (cache misses, IPC via `perf stat`), and isolate with anti-affinity, dedicated node pools or exclusive CPUs (§9, §17).

8. Our service's latency looks healthy on the dashboard (p99 150 ms) but the synthetic monitor in another region shows 1.2 s for the same calls. Nothing is queued at our pods. Where could the missing second be?
   > **Direction:** Outside the server's timer: accept-queue waits, TLS and TCP handshakes with retransmits, DNS, load-balancer queueing, and Nagle; measure from the client and at the LB, and check `ListenOverflows` and SYN retransmits (§2, §10, §14, §15, `A12`).

9. A Node service works fine until a partner sends larger webhooks. Then event-loop lag rises, liveness probes fail, pods restart, the partner retries all failed webhooks at once, and the whole fleet goes down. Break the cycle at as many points as you can.
   > **Direction:** Bound body size and parse off the loop (§1), make liveness tolerant (§1), accept-then-queue webhooks and respond fast (§14), shed and rate-limit retries with `Retry-After` (§18); each link in the feedback loop is a place to cut it (`C03`, `C16`).

10. Our service's latency is fine, but every 6 hours for about 90 seconds it doubles, across all pods simultaneously. Pods were started at different times. What systemic, time-based causes would you investigate?
    > **Direction:** Things synchronised by time rather than pod age: certificate or credential refresh, DNS TTLs, cache TTLs set at the same moment by a deploy, cron on the node, log rotation, a dependency's own batch; correlate with the calendar, not uptime (§12, §17, §18).

11. A service that uses HTTP keep-alive to a partner API through an AWS NAT gateway gets occasional requests that hang for exactly the client timeout (30 s), only after quiet periods overnight. What's happening?
    > **Direction:** The NAT gateway silently dropped idle flows after 350 s, so the client writes into a dead connection and waits for its timeout; set pool idle timeouts under 350 s or enable TCP keepalive probes, and retry idempotent requests (§15, `C17`, `A12`).

12. After enabling an APM agent on a Node fleet, throughput dropped 25% and memory rose 30%. The vendor says the overhead is 2%. How do you settle it with evidence?
    > **Direction:** Run an A/B canary with identical traffic and compare throughput, ELU and heap; profile with and without to find promise hooks and wrappers, then tune sampling or instrumentations — observer effects are real (§4, §17, `C13`).

13. We moved a CPU-heavy Node service from 2 pods × 4 vCPU to 8 pods × 1 vCPU for "better packing". p99 doubled even though total CPU is the same. Why?
    > **Direction:** With 1 CPU, libuv threads, GC helper threads and JIT compete with the main thread for one quota and bursts are throttled; each pod has less burst headroom and smaller pods have worse queueing (fewer servers per queue) — measure throttling and ELU (§2, §3, §9, §10).

14. A Python service on 16 gunicorn workers per pod was moved to a node with half the memory per core. It started swapping and latency went to seconds. Swap is supposed to be off in Kubernetes. What happened and how do you design for memory safety?
    > **Direction:** Node-level swap (or zram) enabled for the node pool, or page cache thrashing under memory pressure, made pages slow to fault in; with pre-fork copy-on-write unwinding, worker memory grew; size by PSS, freeze GC before fork, recycle workers and keep limits honest (§8, §9, `C07`).

15. A fleet of JVM services on Kubernetes shows p99 spikes every time the cluster autoscaler adds nodes. The new nodes aren't even running our pods yet. What could link the two?
    > **Direction:** Cluster-level side effects: DNS and kube-proxy reprogramming (iptables rule updates with large rule sets), conntrack churn, service mesh config pushes, or CoreDNS load; correlate latency with control-plane events and look at the network path, not the app (§2, §15, §17).

16. A Node service's p99 has a steady 40 ms floor for one downstream but not for others. The downstream is a Java service. `tcpdump` shows the request body arriving 40 ms after the headers. Walk through the diagnosis and the fix.
    > **Direction:** Nagle on the sender plus delayed ACK on the receiver: headers and body are separate small writes and the second waits for an ACK; enable `TCP_NODELAY` on the client and write the request in one go (§15, `A12`, `C17`).

17. After a successful load test, the production launch fell over at 30% of the tested load. The load test used 10,000 synthetic users hitting 50 product pages. Production had 2 million distinct products. What went wrong across layers?
    > **Direction:** Cache key cardinality — the test ran from warm caches and hot DB pages, while production's long tail missed every layer; coordinated omission hid tail latency too — test with realistic key distributions and open-model load (§11, §16, `C15`).

18. A system handles its normal peak, but after a 5-minute regional failover sent it 2× traffic, it never recovered even after traffic returned to normal. The heap was full and GC was running continuously. Why would GC be part of a metastable loop?
    > **Direction:** Overload raises in-flight requests and live heap, GC time rises and throughput falls, keeping in-flight high; recovery needs shedding to drain the heap — admission control keyed on in-flight count or GC time breaks the loop (§3, §6, §18, `C12`).

19. Your company's "golden" Node service has had every incident type in this file. The CTO asks you what five signals you would put on its main dashboard so the next incident is diagnosed in five minutes. What are they and why?
    > **Direction:** Saturation signals over averages: event-loop delay and ELU, threadpool and connection-pool queue waits, GC time and heap against its limit, CFS throttled ratio and working-set memory, and latency histograms per downstream with retry and timeout counts (§1, §2, §3, §9, §17, `C13`).

20. An outage post-mortem shows five contributing factors: a GC pause, a lease expiry, a retry storm, CPU throttling on the lock service, and a liveness probe restart loop. The team wants a "root cause". How do you frame the analysis and which fixes do you prioritise?
    > **Direction:** Metastable and cross-layer failures have feedback loops rather than one root cause; map the loop, then prioritise fixes that break it structurally — fencing and idempotency, retry budgets, load shedding, isolation of control-plane work and removing throttling — over tuning any single trigger (§12, §13, §18, `C11`, `C14`).
