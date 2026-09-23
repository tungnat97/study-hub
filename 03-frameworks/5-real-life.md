[← back to the field index](README.md)

# Frameworks · Part 5 — Real-life production problems

Parts 1–4 teach the frameworks as they are documented. This part is about how they behave at 3am:
the timeout that is 5 seconds on one side and 60 on the other, the decorator that silently does
nothing, the provider scope that quietly makes every request allocate a hundred objects, the worker
that picks the same job up twice.

**How to use it.** Read the pre-knowledge once, properly. Then work through the questions level by
level. For each one, cover the direction line and say your answer out loud as you would to an
interviewer — symptom, hypothesis, how you would prove it, the fix, and the guard-rail that stops it
coming back. Only then read the direction. A senior answer names the mechanism, not just the remedy:
"raise the timeout" is a junior answer; "the Node keep-alive timeout is shorter than the load
balancer's idle timeout, so the LB reuses a socket Node has just closed" is the senior one.

Node IDs in `code` point back to Parts 1–4 and to other fields.

---

## Pre-knowledge

### 1. Timeouts along the path, and why 502s happen

Every hop in `client → CDN → load balancer → ingress/nginx → app server → app → pool → database`
has its own idle, read and total timeouts. Most production 502/504 mysteries are two adjacent hops
disagreeing (`F01`, `F28`, `A01`).

**The keep-alive race (the classic 502).** A load balancer keeps a pool of idle keep-alive
connections to each backend and reuses them. If the **backend closes idle connections sooner than
the LB does**, there is a window where the LB sends a new request down a socket the backend has just
closed. The backend responds with a TCP RST, and the LB returns **502**. Symptoms: a low, steady rate
of 502s (0.01–0.5%), often worse at low-to-medium traffic (more connections sit idle long enough), and
no error in the app logs because the app never saw the request.

Real defaults to know:

| Component | Setting | Default |
|---|---|---|
| Node `http.Server` | `keepAliveTimeout` | 5 000 ms |
| Node `http.Server` | `headersTimeout` | 60 000 ms (Node 18+; older: 40 000) |
| Node `http.Server` | `requestTimeout` | 300 000 ms (Node 18+; 0 before) |
| AWS ALB | idle timeout | 60 s |
| GCP external HTTP(S) LB | backend keep-alive | 600 s (backend service timeout 30 s) |
| nginx | `keepalive_timeout` (client side) | 75 s |
| nginx | `proxy_read_timeout` | 60 s |
| gunicorn | `--keep-alive` | 2 s |
| gunicorn | `--timeout` (silent worker killed) | 30 s |
| undici / Node `fetch` | client keep-alive timeout | 4 s (or the server's `Keep-Alive` hint) |
| Tomcat | `keepAliveTimeout` | = `connectionTimeout` (connector default 60 s; stock `server.xml` 20 s) |

**Rule:** each hop's idle timeout must be *longer* than the hop in front of it. Behind a 60 s ALB, set
Node `keepAliveTimeout` to about 65 s and `headersTimeout` slightly above that (older Node versions
had a bug where `headersTimeout < keepAliveTimeout` caused its own spurious disconnects):

```ts
const app = await NestFactory.create(AppModule);
const server = app.getHttpServer();
server.keepAliveTimeout = 65_000;
server.headersTimeout = 66_000;
```

**The mirror image, outbound.** When your service calls another over a pooled keep-alive agent, you
are now the "LB". If your agent keeps sockets idle longer than the upstream's keep-alive, you reuse a
dead socket and get `ECONNRESET` / `socket hang up` on the *first* request after an idle gap. Fix: set
the client's idle timeout *shorter* than the server's (for example 4 s against a 5 s server), and
retry idempotent requests once on `ECONNRESET` for a reused socket.

**Unconsumed response bodies leak sockets.** With undici (Node's `fetch`), a response whose body is
never read or cancelled keeps its socket busy until garbage collection gets round to it. Code that
checks `res.status` and returns on an error path without `await res.text()` or `res.body.cancel()`
slowly exhausts the connection pool: outbound calls start queueing, and `ss` shows sockets stuck
open. Every path must consume or cancel the body. (The Java equivalent is not closing a
`CloseableHttpResponse` or not consuming the entity with Apache HttpClient, which leaves the pooled
connection leased until the pool is empty.)

**Duplicate side effects from timeouts.** A 504 does not mean the upstream did nothing — it usually
finished the work after the proxy gave up. A client (or an SDK, or a gateway) that retries a `POST`
on 502/504 creates a second order or a second charge. Non-idempotent operations need an idempotency
key stored with a unique constraint (`M09`, `SD10`); proxies should not retry non-idempotent
requests (nginx has not, by default, since 1.9.13).

**504 versus 502.** 504 means the proxy waited and gave up (upstream slow); 502 means the upstream
connection broke or returned garbage (reset, crash, protocol error). A request that consistently
fails at exactly 60 s is a proxy read timeout; at exactly 30 s it is often gunicorn killing a sync
worker (`WORKER TIMEOUT` in its log), a Hikari acquire wait or a GCP backend timeout; at 300 s it is
Node `requestTimeout`. **Round-number latencies are timeouts** — always ask which component owns that
exact number.

**Timeout budgets.** The outer timeout must exceed the inner ones plus retries, or the client gives
up while the server is still working (wasted work, and a retry storm on top). Propagate a deadline
rather than stacking fixed timeouts (`M09`). An inner retry loop of 3 × 10 s under a 15 s gateway
timeout means the client always sees 504 while the server keeps hammering the dependency.

### 2. The request pipeline: order, scope and what silently does not run

**NestJS order:** middleware → guards → interceptors (before) → pipes → handler → interceptors
(after, via RxJS) → exception filters. Within a stage: global → controller → method (`F03`, `F19`).
Consequences that bite:

- **Middleware runs before the route handler is resolved into an `ExecutionContext`**, so it cannot
  read `@Roles()` or `@Public()` metadata; guards and interceptors can (via `Reflector`).
- **Guards run before pipes**, so a guard sees the raw, unvalidated, untransformed body and params.
  `req.params.id` in a guard is a string even if the route has `ParseIntPipe`.
- An interceptor's `catchError` sees exceptions *before* exception filters; an interceptor that
  swallows or remaps errors changes what the filter (and your error tracker) sees. A timing or
  logging interceptor that only uses `tap()` never records failed requests.
- `app.useGlobalGuards/Pipes/Filters()` registered in `main.ts` sit outside the DI module graph:
  they cannot inject dependencies, and they do **not** apply to hybrid-app microservice transports.
  Register with `APP_GUARD` / `APP_PIPE` / `APP_FILTER` providers instead.
- `@Res()` switches the route to library-specific mode: interceptors that `map()` the response are
  bypassed, `ClassSerializerInterceptor` does nothing, and if you forget to send, the request hangs.
  Use `@Res({ passthrough: true })` when you only need to set a header or cookie.
- Route and middleware path syntax changed in Nest 11 (Express 5 / path-to-regexp v8: `*` must be
  named, e.g. `*splat`). Middleware or guards silently not matching after an upgrade is a real
  incident class — and when it is auth middleware, a security one.
- Exception filters that catch only `HttpException` let a TypeORM `QueryFailedError` or a Prisma
  `P2002` through as a generic 500; unique-violation should become 409.
- Streaming responses (`StreamableFile`, SSE): once headers are sent, a later error cannot change the
  status code; a filter that tries throws `ERR_HTTP_HEADERS_SENT`.

**Express specifics.** Express 4 does not catch rejected promises from async handlers or
middleware: the request hangs until a timeout, and the rejection may be unhandled (Node 15+ crashes
the process on an unhandled rejection by default). Express 5 forwards rejections to `next(err)`. Nest
wraps its own handlers, but *raw Express middleware* mounted with `app.use()` is still exposed.
Express's `json()` body limit is 100 kB by default — large payloads get 413 from the parser, before
any Nest code runs.

**Django:** middleware is an onion in `MIDDLEWARE` order — top to bottom on the request, bottom to
top on the response. `SecurityMiddleware` and CORS middleware must be high; a middleware that returns
early short-circuits everything below it. `ATOMIC_REQUESTS=True` wraps the *view* in a transaction,
not the middleware, and not streaming response generation.

**Spring:** servlet `Filter`s (ordered by `@Order` / `FilterRegistrationBean`; Spring Security's chain
is one filter at order −100) → `DispatcherServlet` → `HandlerInterceptor.preHandle` → argument
resolution and `@Valid` → controller, through AOP proxies (`@Transactional`, `@Cacheable`,
`@PreAuthorize`, each an advisor with its own order) → `@ControllerAdvice` for exceptions. Errors
thrown in a filter never reach `@ControllerAdvice`; they go to the container's `/error` handling.

### 3. Dependency injection footguns

**Nest request scope bubbles up (`F04`, `F19`, `F20`).** If provider A is `Scope.REQUEST`, every
provider that injects A — and every provider that injects *those*, up to and including the
controller — becomes request-scoped too. One innocent `@Inject(REQUEST)` in a logger or tenant
service deep in the graph turns a whole subtree into per-request instantiation: dozens of objects
constructed and garbage-collected per request, `onModuleInit` never called for them, and throughput
dropping 20–50% with more GC time and a worse p99. It is invisible in code review because the
consumer files do not change. Fixes:

- Replace request scope with **AsyncLocalStorage** (`nestjs-cls`): singleton services read the
  per-request context from ALS. This is almost always the right answer.
- Use **durable providers** (`durable: true` plus a `ContextIdStrategy`) to get one subtree per
  *tenant* rather than per request.
- Inject `ModuleRef` and resolve request-scoped things only where they are needed.

Request-scoped providers also break in contexts without an HTTP request — cron jobs, queue
consumers, WebSocket gateways — where `REQUEST` is `undefined` or the scope is not supported.

**Circular dependencies.** Two kinds. *Module/provider cycles* need `forwardRef(() => X)` on both
sides. *File import cycles* (often via barrel `index.ts` files) cause a class to be `undefined` at
the moment decorator metadata is emitted, giving `Nest can't resolve dependencies of FooService
(?)` — the `?` means the token was `undefined` — or, worse, an `undefined` injected in one build
but not another because import order differs between `ts-node`/`swc` in dev and compiled output.
Diagnose with `madge --circular`. Fix by breaking the cycle (extract a third provider, use events),
not by sprinkling `forwardRef`.

**Singletons holding request state.** A default-scope provider with a field like `this.currentUser`
is shared by every concurrent request; under load, users see each other's data. The same bug exists
in Spring (singleton bean with an instance field) and in Django (module-level mutable state, or
class attributes mutated per request).

**Lifecycle hook order.** Nest: `onModuleInit` (per module, dependency order) →
`onApplicationBootstrap` → listening. Shutdown (only if `app.enableShutdownHooks()` was called):
`onModuleDestroy` → `beforeApplicationShutdown(signal)` → connections closed →
`onApplicationShutdown(signal)`. An async `onModuleInit` that awaits a slow external call delays
startup and can blow a startup or readiness probe; one that throws kills boot.

**Spring.** A prototype bean injected into a singleton is created once (use `ObjectProvider` or
lookup methods). Circular references are refused by default since Boot 2.6
(`spring.main.allow-circular-references`). `@Transactional` on a `@PostConstruct` method does not
start a transaction (the proxy is not in the call path). Bean overriding is disabled by default since
Boot 2.1. Two beans of the same *type* without `@Primary` / `@Qualifier` fail with
`NoUniqueBeanDefinitionException` — which often appears only when a new dependency brings its own
`ObjectMapper` or `DataSource` and switches off Boot's auto-configured one via
`@ConditionalOnMissingBean`, silently dropping your Jackson or pool settings.

**Self-invocation (Spring, and any proxy-based AOP).** `@Transactional`, `@Cacheable`, `@Async`,
`@Retryable` and `@PreAuthorize` work through a proxy around the bean. `this.otherMethod()` inside
the same class bypasses the proxy, so the annotation does nothing. Neither do annotations on
`private` methods, or on a `final` class/method with CGLIB proxies. Nest's interceptors are route
based, so the equivalent trap is calling a decorated method directly from another service.

### 4. Async context propagation

**Node AsyncLocalStorage.** ALS follows the async call graph through promises, timers and most
callbacks. It is *lost* or, worse, *wrong* when a library queues callbacks and runs them later from a
different context:

- Connection pools and batchers (older `generic-pool`, some Redis and Mongo drivers, DataLoader-style
  batchers) run the callback in the context of whoever triggered the flush, so request B's log lines
  carry request A's trace ID.
- `EventEmitter` listeners run in the context of the `emit()` call, not the `on()` call.
- Custom thenables and some callback-to-promise shims drop context.

Fixes: `AsyncResource.bind(fn)` / `AsyncLocalStorage.bind(fn)` at the boundary; create the context
with `als.run()` rather than `enterWith()` (which leaks into the caller's continuation); in Nest,
start the context in middleware or with `nestjs-cls` *before* guards run. Test the invariant: fire
N concurrent requests with distinct IDs and assert every log line's ID matches its own request.

**OpenTelemetry in Node** instruments by patching modules at `require` time, so the SDK must load
*before* anything else (`node --require ./tracing.js`, or `--import` for ESM). Initialise it in
`main.ts` after `AppModule` has pulled in `pg`, `http` and `express`, and spans are missing or
traces are disconnected.

**Java.** `ThreadLocal`-based context (MDC, `SecurityContextHolder`, the Hibernate session,
`@Transactional`'s bound connection) does not cross to `@Async`, `CompletableFuture.supplyAsync`,
`parallelStream()` or a custom executor. You need a `TaskDecorator` that copies MDC and security
context, or Micrometer context-propagation. A method that is both `@Async` and `@Transactional`
runs its transaction on the new thread, independent of the caller's. In WebFlux, use Reactor
`Context`, not `ThreadLocal`. With virtual threads (JDK 21), `synchronized` blocks pinned the
carrier thread until JDK 24, and per-thread caches become per-task allocations.

**Python.** `threading.local` does not follow `asyncio`; `contextvars` do. Django's
`sync_to_async(thread_sensitive=True)` — the default — runs all such calls on **one shared thread**,
so an async view doing sync ORM calls is effectively serialised through a single thread. Celery tasks
do not inherit request context; pass IDs explicitly in task headers.

### 5. Graceful shutdown and deploys

The full sequence in Kubernetes (`F28`, `O03`, `O06`): pod marked Terminating → *in parallel*, the
endpoint is removed from Services and ingress (which takes seconds to propagate to kube-proxy, the
ingress controller and cloud LB target groups) **and** the `preStop` hook runs, then SIGTERM is sent
→ the app must stop accepting, drain in-flight work and close pools → after
`terminationGracePeriodSeconds` (default 30 s, which *includes* preStop time) SIGKILL.

Because deregistration and SIGTERM race, an app that exits promptly on SIGTERM still receives new
requests for a few seconds, and they fail with 502 or connection refused. The unglamorous fix is a
**`preStop` sleep** (5–15 s; native `sleep` action since k8s 1.29) so the app keeps serving while
routing catches up, then shuts down normally. With AWS ALB target groups, `deregistration_delay`
(default 300 s) also matters.

Specific traps:

- **PID 1 and signals.** The Linux kernel does not apply default signal actions to PID 1. If Node is
  PID 1 and has no SIGTERM handler, SIGTERM is *ignored* and the pod dies by SIGKILL after 30 s on
  every deploy. `CMD npm start` or shell-form `CMD node main.js` puts `npm` or `sh` at PID 1, and
  they may not forward the signal. Use exec form `CMD ["node","dist/main.js"]`, plus `tini` or
  `--init` to reap zombies.
- **Nest** only runs shutdown hooks after `app.enableShutdownHooks()`. `app.close()` calls
  `server.close()`, which stops accepting but waits for keep-alive connections to go idle; Node
  18.2+ has `server.closeIdleConnections()`, and Node 19+ closes idle ones on `close()`. Busy
  keep-alive connections still need `Connection: close` on their next response.
- **Teardown order** matters: stop the HTTP server and queue consumers first, let in-flight work
  finish, *then* close the DB pool and Redis. Closing Prisma or TypeORM in `onModuleDestroy` while
  requests are in flight produces a burst of "pool is closed" errors on every deploy.
- **Spring Boot**: `server.shutdown=graceful` (default since Boot 3.4) with
  `spring.lifecycle.timeout-per-shutdown-phase` (30 s). Readiness flips to `REFUSING_TRAFFIC`.
- **gunicorn**: SIGTERM is graceful within `--graceful-timeout` (30 s); SIGINT/SIGQUIT are fast.
  `--max-requests` recycling also exercises this path.
- **Queue workers**: BullMQ `worker.close()` waits for active jobs; Celery does a warm shutdown on
  SIGTERM. If the grace period is shorter than the longest job, the job is killed mid-flight and
  re-run (§11) — so jobs must be idempotent and the grace period must fit the work.
- **Rolling deploy capacity**: `maxUnavailable` / `maxSurge` plus slow startup (Spring 20–60 s, cold
  JIT, empty caches) means a deploy under peak load briefly runs with fewer, colder instances — the
  "every deploy causes a latency spike" problem. Warm up before reporting ready.

**Probes.** Liveness answers "is this process wedged?" and must not check dependencies — a liveness
probe that pings the database restarts *every* pod when the database blips, turning a partial outage
into a full one. Readiness may check critical dependencies but should be cheap and cached. Startup
probes stop slow-booting JVMs from being killed by liveness before they finish starting. A liveness
probe served on the same event loop as the traffic fails whenever the loop is blocked (§6), so a
CPU-heavy request can get healthy pods killed in a cascade.

### 6. The event loop, the thread pool and CPU work

A Node process has one JS thread. Anything CPU-bound on it — `JSON.stringify` of a 20 MB object,
`bcrypt.hashSync`, a catastrophic regex, `class-transformer` over 50 000 objects, sorting large
arrays, synchronous `zlib`, `fs.readFileSync` — stalls *every* in-flight request (`C03`, `F24`).
Symptoms: p99 explodes while p50 looks fine; one core pinned at 100% while the pod shows 25% of
four; health checks time out and the pod is restarted; timeouts on *unrelated* endpoints.

Measure it: `perf_hooks.monitorEventLoopDelay()` (export p99 loop lag as a metric; sustained >100 ms
is a problem), `--cpu-prof`, Clinic Doctor/Flame, `0x`. Shed load on lag (Fastify's
`@fastify/under-pressure`) rather than letting the queue grow.

**The libuv thread pool** (default `UV_THREADPOOL_SIZE=4`, max 1024) runs `fs`, `dns.lookup`, async
`crypto.pbkdf2/scrypt/randomBytes`, async `zlib` and native addons such as `bcrypt`. Four concurrent
bcrypt hashes saturate it, and then *file reads and DNS lookups queue behind password hashing* — an
outbound HTTP call appears to take 2 s because `dns.lookup` waited for a thread. Raise the pool size
(set it before the pool is first used — as an env var, not mid-program), cache DNS, or move hashing
to a worker thread or a separate service.

**Microtask starvation.** A loop of already-resolved promises or recursive `process.nextTick` never
yields to I/O. `setImmediate` yields; `await` on a resolved value does not.

**Serialisation is CPU.** Nest's `ClassSerializerInterceptor` walks every object with reflection;
`plainToInstance` under `ValidationPipe({ transform: true })` does too. On list endpoints returning
thousands of entities it can dominate the request. Return plain objects shaped by the query, or use
Fastify's schema-compiled serialiser (`fast-json-stringify`).

**stdout is synchronous on Linux pipes and files** in Node. `console.log` of large objects (or a
debug logger left on) blocks the event loop whenever the log collector reading the pipe is slow.
Use an async logger (pino with a transport or `sonic-boom` async), log less, never log whole bodies.

**Containers and CPU.** `os.cpus().length` reports the *host's* CPUs, not the cgroup quota, so
"one worker per CPU" in a 1-CPU pod on a 64-core node spawns 64 processes. The CFS quota is
enforced per 100 ms period: a process with a 0.5 CPU limit that uses 50 ms of CPU across several
threads (GC threads, the libuv pool) is throttled for the rest of the period — latency spikes in
exact ~100 ms steps with low average CPU. Check `container_cpu_cfs_throttled_periods_total`.
The JVM sizes its parallel GC threads, JIT compiler threads and the common `ForkJoinPool` from the
CPU count it detects; a JVM that sees many CPUs (no CPU limit set, or an old JDK) but runs under a
small quota burns the whole period's budget in a burst of GC threads and is then frozen. Pin it with
`-XX:ActiveProcessorCount=<n>` and size GC threads to the quota. Python's `os.cpu_count()` and
`multiprocessing.cpu_count()` also report host CPUs, which is how gunicorn configs computing
`2 × CPU + 1` spawn 129 workers in a 2-CPU pod.

**Python and Java equivalents.** The GIL means CPU work in one thread starves others in the process;
an async (ASGI) view doing CPU or sync I/O blocks the whole loop. gunicorn sync workers handle one
request each, so worker count *is* concurrency, and a slow client holding a sync worker is a DoS
unless a buffering proxy sits in front. A blocking call on a Netty event-loop thread in WebFlux
(`reactor-http-nio-*`) freezes that loop; BlockHound detects it.

### 7. Memory: leaks, limits and fragmentation

**Node leaks come from references that outlive the request:** module-level `Map` caches with no
bound; listeners added per request to a long-lived emitter (`MaxListenersExceededWarning` is the
tell); closures captured in intervals never cleared; request objects stored in singletons;
`prom-client` metrics labelled with raw paths or user IDs (unbounded cardinality); ALS contexts
retained by a long-lived promise; durable-provider subtrees cached per tenant that are never
evicted (§3) — memory then grows with the number of tenants ever seen. A subtle one: a
`Promise.race([work, sleep(60_000)])` timeout helper that never calls `clearTimeout` keeps every
timer — and the closure, request and response it captured — alive for 60 s even though the work
finished in 20 ms; at 5 000 requests per second that is 300 000 retained requests. Diagnose with **three heap snapshots** (baseline, after load,
after more load) and look at what keeps growing; the retainer path shows who holds it. Take snapshots
on a pod pulled out of the load balancer — a snapshot pauses the process and can need roughly the
heap size again in memory. `node --heapsnapshot-signal=SIGUSR2` and
`--heapsnapshot-near-heap-limit=1` capture in production.

**Heap limit versus container limit.** V8's old-space limit and the container memory limit are
different numbers. If the heap may grow beyond the container limit, the kernel OOM-kills (exit code
137, `OOMKilled`, no JS stack trace). If the container is bigger than the heap limit, V8 dies with
`FATAL ERROR: Reached heap limit` while the pod has spare memory. Set `--max-old-space-size` to
roughly 70–80% of the container limit, leaving room for buffers, native memory and thread stacks.
`Buffer`s and native addons live outside the JS heap: RSS growing while `heapUsed` is flat means
off-heap growth (buffers, image libraries, unclosed gzip streams, glibc arena fragmentation — often
cured by `MALLOC_ARENA_MAX=2` or jemalloc).

**Java.** The default `MaxRAMPercentage` is 25% of the container limit — a 2 GiB pod gets a 512 MiB
heap and the rest sits unused. Or the team sets `-Xmx` equal to the limit and gets OOM-killed,
because metaspace, thread stacks, direct buffers and the code cache are on top. With fewer than 2
CPUs or less than 1792 MB, the JVM silently selects SerialGC. Tools: `jcmd <pid> GC.heap_info`, JFR,
`-XX:NativeMemoryTracking=summary`, `-XX:+HeapDumpOnOutOfMemoryError`. Also: an exposed
`/actuator/heapdump` hands the whole heap — secrets included — to anyone who can reach it.

**Python.** RSS rarely shrinks after a peak (pymalloc arenas and fragmentation), so a worker that
once built a 500 MB report stays at 500 MB. `DEBUG=True` makes Django append every SQL query to
`connection.queries` — a genuine leak in long-running Celery workers and management commands.
`tracemalloc` snapshots find Python-level growth. Mitigation that is also legitimate engineering:
gunicorn `--max-requests 1000 --max-requests-jitter 100`, Celery `worker_max_tasks_per_child` /
`worker_max_memory_per_child`. The jitter matters: without it, all workers restart together.

**Fork and copy-on-write.** gunicorn `--preload` loads the app once and forks, sharing pages — but
CPython's reference counting writes to every object header, so pages get copied anyway;
`gc.freeze()` after preload helps. Anything opened before the fork (DB connections, gRPC channels,
Kafka producers, random seeds) is shared across workers and gets corrupted: open it post-fork.

### 8. Connection pools

**Sizing is arithmetic** (`DB21`, `F24`): total = replicas × processes per replica × pool size.
Ten pods × 4 gunicorn workers × persistent connections, or 20 pods × Prisma's default pool, easily
exceeds Postgres `max_connections` (default 100; each connection is a backend process using
megabytes). Autoscaling adds pods exactly when the database is struggling. Size from the database
down: a small pool per process is usually *faster* than a big one. HikariCP's guidance of roughly
`cores × 2 + effective spindles` is for the *database server's* total, not per pod. Little's law:
connections needed ≈ throughput × average time a connection is held.

Defaults and their traps:

| Library | Default size | Waiting for a free connection |
|---|---|---|
| node-postgres (`pg.Pool`, used by TypeORM) | `max: 10` | `connectionTimeoutMillis: 0` = **waits forever** |
| Prisma | `num_physical_cpus × 2 + 1` | `pool_timeout` 10 s, then P2024 |
| HikariCP | `maximumPoolSize` 10 | `connectionTimeout` 30 s |
| Django | no pool; `CONN_MAX_AGE=0` = connect per request | 5.1+: `OPTIONS: {"pool": ...}` with psycopg 3 |
| SQLAlchemy | `pool_size` 5 + `max_overflow` 10 | `pool_timeout` 30 s |

**Pool starvation looks like slowness, not errors.** When requests wait for connections, latency
rises everywhere, CPU is low, the database is mostly idle, and nothing logs. Because node-postgres
waits forever by default, TypeORM apps just hang. Always set an acquire timeout and export pool
metrics (active, idle, waiting). In Java, a thread dump full of threads parked in
`HikariPool.getConnection` is the signature.

**What holds connections too long:** a transaction that calls an HTTP API or sends email in the
middle; `@Transactional` on a controller or orchestration method; Spring **open-session-in-view**
(`spring.jpa.open-in-view=true` by default, with a startup warning) holding the connection for the
whole request including JSON serialisation of lazy collections; streaming a large result set to a
slow client.

**Pool deadlock.** If a request holds one connection and needs a *second* before releasing the
first — `REQUIRES_NEW`, a TypeORM transaction plus a call through the injected repository instead of
the transaction's `manager`, Prisma `$transaction` plus a call on `prisma` instead of `tx` — then
with pool size N and N concurrent such requests, every request holds one connection and waits
forever for another. It passes tests (low concurrency) and deadlocks at exactly pool-size
concurrency. Beyond the deadlock, the stray call runs *outside* the transaction: it cannot see
uncommitted rows, and it is not rolled back.

**Stale connections.** Firewalls, NAT gateways and cloud load balancers drop idle TCP flows silently
(AWS NAT gateway: 350 s; many enterprise firewalls: an hour). The pool hands out a dead connection
and the first query after a quiet period hangs until TCP retransmission gives up (minutes), or fails
with a reset. Fixes: pool `maxLifetime` / `idleTimeout` below the middlebox timeout (Hikari's
`maxLifetime`, 30 min by default, should be a little less than any database or infrastructure
limit), TCP keepalive (`keepaliveTime` in Hikari, `keepAlive: true` in pg), and health checks on
borrow (Django 4.1+ `CONN_HEALTH_CHECKS`).

**PgBouncer in transaction mode** multiplexes many client connections onto a few server
connections, but session state does not survive between transactions: session-level `SET`,
advisory locks, `LISTEN/NOTIFY`, temp tables and (before PgBouncer 1.21's `max_prepared_statements`)
named prepared statements all break. Prisma needed `?pgbouncer=true`; JDBC's `prepareThreshold`
produces `prepared statement "S_1" already exists`. Use `SET LOCAL` inside the transaction for
per-request settings such as an RLS tenant ID — a plain `SET` leaks the tenant to whoever gets that
server connection next.

**Attribute connections.** Put `application_name=orders-api-<pod>` in each connection string;
`pg_stat_activity` then shows which service and pod holds the `idle in transaction` sessions. Set
`idle_in_transaction_session_timeout` and a per-role `statement_timeout` as the database's own
guard-rails. `sqlcommenter`-style SQL comments carrying the route or trace ID map a slow query in
`pg_stat_statements` back to an endpoint.


### 9. Transactions in the framework

**Spring `@Transactional` rules that surprise people (`F10`, `F22`):**

- Self-invocation and `private` methods bypass the proxy (§3), so there is no transaction at all.
- By default it rolls back only on unchecked exceptions (`RuntimeException`, `Error`). A checked
  exception thrown out of the method **commits** the work done so far. Use `rollbackFor`.
- Catching an exception thrown by an inner `@Transactional(REQUIRED)` method does not save you: the
  inner proxy has already marked the shared transaction rollback-only, and the outer commit throws
  `UnexpectedRollbackException: Transaction silently rolled back because it has been marked as
  rollback-only`.
- `REQUIRES_NEW` suspends the outer transaction and takes a *second* connection (§8, pool
  deadlock); it also means the inner commit survives an outer rollback — correct for audit logs, a
  bug for anything that should be atomic.
- `readOnly = true` is a hint: Hibernate skips dirty checking and flush (a real saving on large
  reads), and with a routing data source it can send the query to a replica — which then shows
  stale data to a user who has just written.
- Hibernate flushes at commit and before queries that touch dirty entities. A managed entity loaded
  "just to read" and then modified in memory (for example by a mapper setting a field) is silently
  written back at commit.

**Django.** Autocommit by default; nested `transaction.atomic()` blocks become savepoints.
`ATOMIC_REQUESTS=True` makes every view one transaction, which holds locks and a connection for the
whole view, slow external calls included. `select_for_update()` must run inside `atomic()` —
outside it raises `TransactionManagementError`, except under `TestCase`, which wraps every test in
a transaction and hides the bug. With `select_related` it locks the joined rows too unless you pass
`of=("self",)`. `skip_locked=True` turns a table into a work queue; `nowait=True` fails fast.

**The enqueue-before-commit race** (the most common framework-level data bug, `F10`, `F13`). Code
saves a row and enqueues a job (Celery, BullMQ, an event) inside the transaction. The worker is fast,
picks the job up before the commit, and reads nothing (`DoesNotExist`) or the old state. It appears
only under load or with fast queues, and retries "fix" it intermittently. Fixes: Django
`transaction.on_commit(lambda: task.delay(id))` (Celery 5.4+ also offers `task.delay_on_commit()`),
Spring `@TransactionalEventListener(phase = AFTER_COMMIT)`, or — for guaranteed delivery — a
transactional **outbox** (`M15`). Django `post_save` signals run synchronously *inside* the
transaction, so a signal handler that enqueues has the same race. The reverse bug: enqueuing *after*
commit without an outbox loses the job if the process dies between the two.

**Node ORMs.**

- **TypeORM**: inside `dataSource.transaction(async (manager) => ...)` only calls made through
  `manager` are in the transaction. Calls through injected `@InjectRepository` repositories use a
  different pooled connection (§8). `QueryRunner` transactions must be released in `finally`, or the
  connection never goes back to the pool.
- **Prisma**: `$transaction([...])` (batch) runs the operations atomically;
  `$transaction(async (tx) => ...)` (interactive) has a default `timeout` of 5 s and `maxWait` of
  2 s — a slow external call inside it fails with "Transaction already closed" (P2028). Use `tx`,
  not `prisma`, inside.
- **Propagation across services** in Nest usually needs ALS (`@nestjs-cls/transactional`) so that
  nested service calls pick up the current transaction without passing `manager` everywhere.

**Idle in transaction.** A transaction opened and then abandoned (an exception path that never
rolls back, a `QueryRunner` not released, a debugger breakpoint in production) holds locks and
blocks `VACUUM` cleanup; an `ALTER TABLE` then queues behind it, and every later query queues behind
the `ALTER` — a whole table frozen by one idle session. `idle_in_transaction_session_timeout`, and
`lock_timeout` on migrations, are the guard-rails.

**Isolation.** Postgres and most ORMs default to READ COMMITTED, so read-then-write logic ("check
stock, then decrement") races. Use `SELECT … FOR UPDATE`, an atomic
`UPDATE … SET stock = stock - 1 WHERE stock > 0`, optimistic `@Version` columns, or SERIALIZABLE
with retries (and then the framework code must retry on SQLSTATE `40001`).

**Retries and transactions compose badly.** A retry must wrap the *whole* transaction from outside:
retrying a statement inside a transaction that Postgres has already aborted just gets `current
transaction is aborted, commands ignored until end of transaction block`. In Spring, `@Retryable`
and `@Transactional` on the same method are two advisors, and whether the retry sits outside or
inside the transaction depends on their order — put the retry on a caller method in a different
bean, or set explicit advisor order so each attempt gets a fresh transaction.

### 10. ORM query behaviour

**N+1 everywhere (`DB40`, `F09`, `F21`).** Lazy relations accessed in a loop — in a serializer, a
template, a DTO mapper, a GraphQL resolver — issue one query per row. It hides in code that looks
innocent: DRF `SerializerMethodField`, a Nest GraphQL `@ResolveField`, Jackson serialising a lazy JPA
collection under open-session-in-view. Detect it with query counts per request
(django-debug-toolbar, `assertNumQueries`, Hibernate statistics, ORM query logging), not wall-clock
time.

**Django querysets are lazy and cached.** Nothing runs until iteration, `len()`, `bool()`, slicing
with a step, or `list()`. `if qs:` loads *all* rows; use `.exists()`. `len(qs)` loads everything;
`.count()` then iterating runs two queries. An evaluated queryset is cached, but `qs.filter(...)`
creates a new, unevaluated one. `.iterator(chunk_size=2000)` streams without caching and uses
server-side cursors on Postgres, which break under PgBouncer transaction mode unless
`DISABLE_SERVER_SIDE_CURSORS=True`. `prefetch_related` runs a second query with `IN (...)` — with
100 000 parents that is a huge IN list. `.update()` and `bulk_create()` bypass `save()` and signals.

**JPA / Hibernate.**

- `JOIN FETCH` of a collection with pagination: Hibernate cannot paginate in SQL, so it fetches
  **all** rows and paginates in memory, logging `HHH000104` (Hibernate 5) / `HHH90003004` (6). An
  endpoint that is fine on 1 000 rows goes OOM on 1 000 000. Paginate IDs first, then fetch by IDs.
- Two `List` collections fetch-joined together throw `MultipleBagFetchException`; "fixing" it with
  `Set` gives a cartesian product instead.
- `GenerationType.IDENTITY` disables JDBC batch inserts; `SEQUENCE` with an allocation size plus
  `hibernate.jdbc.batch_size` enables them.
- The persistence context grows with every entity loaded in a transaction; a batch job reading a
  million rows in one transaction slows down on dirty checking and then OOMs. Flush and `clear()`
  every N rows, or use a `StatelessSession`.
- `@ManyToOne` is **eager** by default in JPA; `@OneToMany` is lazy.

**TypeORM.** `synchronize: true` in production drops and recreates a column on a rename — data loss
on deploy. `eager: true` relations load on every `find`. `save()` issues a `SELECT` first to decide
between insert and update, and cascades through relations. `find({ relations: [...] })` across
several one-to-many relations builds one big join — a cartesian explosion; `relationLoadStrategy:
'query'` splits it. Large `In([...])` arrays hit Postgres's 65 535 bind-parameter limit.

**Prisma.** `include` of a relation is a *separate* query by default, not a join (5.x added
`relationLoadStrategy: 'join'`). `findMany` without `take` on a growing table is a slow OOM.
`$queryRaw` tagged templates are parameterised; `$queryRawUnsafe` with interpolation is SQL
injection. Instantiating `new PrismaClient()` per request (or per hot reload in dev, or per
serverless invocation) creates a pool each time and exhausts `max_connections`.

**Types coming back from the driver.** node-postgres returns `bigint` (`int8`) and `numeric` as
**strings** — `COUNT(*)` is `"42"`, and `"42" + 1` is `"421"`. `timestamp without time zone` is
parsed in the *process's* local time zone, so a container in UTC and a laptop in Europe/London
disagree by an hour in summer. Store `timestamptz`; set `TZ=UTC` explicitly in images.

### 11. Background jobs and scheduling

**At-least-once is the only honest guarantee (`F13`, `Q10`).** Every queue redelivers under some
failure, so jobs must be idempotent (an idempotency key, a unique constraint, a state check under a
lock). The interesting part is knowing *which* mechanism redelivers.

**BullMQ.** A worker holds a lock on each active job (`lockDuration` 30 s), renewed from the event
loop. If the loop is blocked by CPU work for longer than that, the lock expires, the stalled-job
checker (`stalledInterval` 30 s) moves the job back to waiting, and **another worker runs it while
the first is still running** — duplicate side effects. After `maxStalledCount` (default 1) it fails
with "job stalled more than allowable limit". Fixes: keep CPU work off the loop (sandboxed
processors run in a child process), raise `lockDuration` for long jobs, make the job idempotent.
Other traps: the ioredis connection for workers must use `maxRetriesPerRequest: null`; completed and
failed jobs are kept unless `removeOnComplete` / `removeOnFail` are set, so Redis memory grows until
eviction starts deleting *queue keys* — a Redis used for queues must run `maxmemory-policy
noeviction`. Retries default to none (`attempts: 1`); backoff must be configured.

**Celery.**

- `task_acks_late=False` (default): the message is acknowledged when the worker *receives* it, so a
  crash or OOM-kill mid-task loses the task. `acks_late=True` acknowledges after completion, so the
  task is redelivered after a crash — and must therefore be idempotent. Add
  `task_reject_on_worker_lost=True`, or a SIGKILLed child's task is still acknowledged.
- **Visibility timeout** (Redis and SQS brokers): an unacknowledged message becomes visible again
  after `visibility_timeout` (Redis default 1 hour). A task that runs longer than that, or an
  ETA/countdown task scheduled further ahead than that, is **delivered again to another worker** —
  the classic "long task runs twice" or "reminder email sent three times" bug. Raise the timeout
  above the longest ETA and runtime, or keep long-dated scheduling in the database, not the broker.
- `worker_prefetch_multiplier` (default 4) × concurrency tasks are reserved per worker. With long
  tasks, one worker hoards messages while others sit idle, and those reserved tasks wait behind a
  slow one. Use `-O fair` and a multiplier of 1 for long tasks.
- Arguments are serialised at enqueue time; passing a model instance instead of an ID sends stale
  data (and pickle is a security hole). Pass IDs and re-read.
- `task_always_eager` in tests hides the transaction race (§9) and serialisation problems.
- Prefetched (reserved) messages are already unacknowledged from the broker's point of view, so with
  `acks_late` on Redis/SQS the visibility clock is ticking while they wait in a worker's buffer
  behind a slow task. Short tasks can be redelivered too if they sat reserved for longer than the
  visibility timeout.
- **Poison messages.** With `acks_late` and `task_reject_on_worker_lost`, a task that crashes its
  worker (OOM on one huge input, a segfault in a native library) is redelivered, crashes the next
  worker, and so on — a crash loop across the fleet from one message. Count deliveries (a header or
  a database row) and park the message in a dead-letter place after N attempts. The same applies to
  BullMQ stalls and to any broker with redelivery.

**Spring.** `@Scheduled` runs on Boot's auto-configured scheduler with **one thread**
(`spring.task.scheduling.pool.size=1`), so one slow job delays every other scheduled job. `@Async`
uses Boot's `ThreadPoolTaskExecutor` with core size 8 and an **unbounded queue**, so `max-size` is
never reached and work queues silently in memory (outside Boot, `SimpleAsyncTaskExecutor` makes a
new thread per call). `@Async` methods returning `void` swallow exceptions unless an
`AsyncUncaughtExceptionHandler` is set.

**Cron in a replicated service.** `@nestjs/schedule` `@Cron`, Spring `@Scheduled` and in-process
APScheduler run in *every* replica: scale to 6 pods and the nightly invoice job runs 6 times.
Options: a Kubernetes `CronJob` (with `concurrencyPolicy: Forbid`; it can still, rarely, run twice
or be skipped), a distributed lock (ShedLock, Redis `SET NX PX`, Postgres `pg_try_advisory_lock`),
or a queue's repeatable jobs (BullMQ `repeat`, or Celery beat — but only *one* beat process). A lock
whose TTL is shorter than the job lets a second run start while the first is still going; the robust
version records "run for date X" in a row with a unique constraint, or uses a fencing token.

**Time.** Cron in local time meets daylight saving: in Europe/London a job at 01:30 runs twice in
October and not at all in March. Schedule in UTC or outside 01:00–02:00. Container time zones
default to UTC; laptops do not.

### 12. Validation, serialisation and data types

**Nest `ValidationPipe` (`F06`).**

- Without `whitelist: true`, properties not in the DTO pass straight through to the service — mass
  assignment (`{ "role": "admin" }`) when the body is spread into an entity.
  `forbidNonWhitelisted: true` returns 400 instead of silently stripping.
- Nested objects need both `@ValidateNested()` and `@Type(() => Child)`; without `@Type` the nested
  object is never validated, and nothing tells you so.
- `transform: true` with `enableImplicitConversion: true` converts by TypeScript type:
  `?active=false` becomes `Boolean("false") === true`. Use an explicit `@Transform` for booleans.
- An array of DTOs as the body (`@Body() items: ItemDto[]`) is not validated element-wise, because
  generic type information is erased; wrap it in a DTO or use `ParseArrayPipe`.
- Validation and transformation are reflection-heavy; at high RPS on large bodies they show up in
  CPU profiles.

**JSON and numbers.** JavaScript numbers are IEEE doubles: integers above 2^53 − 1
(9 007 199 254 740 991) lose precision. Snowflake or `BIGINT` IDs serialised as JSON numbers by a
Java or Python backend arrive corrupted in the browser or in a Node service — `JSON.parse` rounds
silently. Serialise them as strings (Jackson `ToStringSerializer`, a DRF `CharField`). In Node,
`JSON.stringify` of a `BigInt` throws `TypeError: Do not know how to serialize a BigInt`.

**Jackson.** `ObjectMapper` is thread-safe and expensive to build — creating one per request costs
CPU and defeats its caches. Spring Boot's auto-configured mapper disables
`FAIL_ON_UNKNOWN_PROPERTIES` and registers the Java time module; a hand-made `new ObjectMapper()`
does neither, so behaviour differs between the controller and your own code. Serialising a lazy JPA
entity triggers loading (N+1, or `LazyInitializationException` with OSIV off), and bidirectional
relations recurse infinitely. Serialise DTOs, not entities.

**Dates and time zones.** `Date` serialises to UTC with `Z`; `LocalDateTime` has no zone at all;
Django with `USE_TZ=False` stores naive times. Mixed conventions produce one-hour bugs twice a year.

**DRF.** `ModelSerializer` is slow for large lists (field machinery per instance); `many=True` over
10 000 rows can take seconds of pure Python. Use `.values()` and plain dicts for hot list endpoints,
and paginate. A `SerializerMethodField` that queries is N+1 (§10).

### 13. Caching, rate limiting and realtime

**Framework caches (`F15`, `Q02`).**

- Nest `CacheInterceptor` caches only `GET` and keys by URL by default — not by user. Put it on an
  endpoint that returns user-specific data and one user's response is served to everyone requesting
  that URL. Override `trackBy()` or do not cache personalised responses at this layer.
- Nest 10 moved to `cache-manager` v5, where `ttl` is in **milliseconds**, not seconds: a `ttl: 60`
  that meant one minute now means 60 ms (cache effectively off, database load up), and the reverse
  mistake caches for 16 hours.
- Spring `@Cacheable` has the self-invocation trap (§3), caches `null` unless told
  `unless = "#result == null"`, and keys by the method arguments — two methods sharing a cache name
  with the same arguments collide.
- An in-process cache (a `Map`, Caffeine, Django `LocMemCache`) is per process: N pods give N
  inconsistent views after a write, and each gunicorn worker holds its own copy.
- **Stampede.** A popular key expires and hundreds of requests recompute it at once. Use request
  coalescing (a single in-flight promise per key), a short lock, early probabilistic refresh, or
  stale-while-revalidate. Jitter TTLs so keys written together do not expire together.

**Rate limiting.** `@nestjs/throttler` with its default in-memory storage limits *per pod*: with 8
pods, "100 requests per minute" is 800. Behind a proxy, `req.ip` is the proxy's address unless
`trust proxy` is set, so every user shares one bucket and the whole site gets 429s; trusting
`X-Forwarded-For` blindly lets clients spoof their IP. Trust exactly the number of proxy hops you
have.

**WebSockets and SSE (`F16`, `A16`).**

- Idle connections are closed by the LB idle timeout (ALB 60 s) unless something is sent;
  application-level pings every 20–30 s keep them open. socket.io pings by default (`pingInterval`
  25 s); a raw `ws` server does not.
- socket.io's HTTP long-polling transport needs **sticky sessions**; without them the handshake lands
  on pod A and the next poll on pod B ("Session ID unknown", 400). Broadcasting across pods needs the
  Redis (or similar) adapter; otherwise an event emitted on pod A never reaches clients on pod B.
- A deploy drops every connection and they all reconnect at once — a thundering herd that can
  overwhelm auth and the database. Clients need exponential backoff with jitter; servers can drain
  gradually.
- SSE through compression middleware or nginx buffering arrives in bursts or not at all: disable
  compression on that route, and set `X-Accel-Buffering: no` or `proxy_buffering off`.
- Each connection holds memory and a file descriptor; a `ulimit -n` of 1024 caps a process at about
  a thousand connections.
- On Google Cloud's external HTTP(S) load balancer, the backend service timeout (30 s by default)
  is also the **maximum lifetime of a WebSocket connection**, regardless of traffic on it — sockets
  close at exactly 30 s even with pings. Raise the backend timeout for the WebSocket backend and make
  clients reconnect cleanly anyway.

### 14. Security footguns behind proxies and in frameworks

- **Scheme and host behind TLS termination.** Django's `SECURE_SSL_REDIRECT` behind a load balancer
  that talks HTTP to the app causes an infinite redirect loop unless `SECURE_PROXY_SSL_HEADER` is set
  (and the proxy overwrites any client-supplied `X-Forwarded-Proto`). Spring needs
  `server.forward-headers-strategy`; without it, redirects and OAuth callback URLs point at
  `http://` or the internal host name. Express needs `trust proxy` for `req.secure` and secure
  cookies.
- **CSRF and origins.** Django 4.0+ requires `CSRF_TRUSTED_ORIGINS` entries to include the scheme
  (`https://app.example.com`); upgrades break form posts with 403. `SameSite=None` cookies require
  `Secure`.
- **Host header.** Django's `ALLOWED_HOSTS` rejects unknown hosts with 400 — including a load
  balancer health check that uses the pod IP as the host. Password-reset links built from the `Host`
  header are a poisoning vector.
- **Mass assignment** (§12), **prototype pollution** from deep-merging request bodies into objects
  (`__proto__` keys), and **ReDoS** from badly written regexes running on the event loop (§6) are
  the Node-specific classics (`F26`, `S06`).
- **Actuator and debug endpoints.** Spring `/actuator/env`, `/heapdump` and `/threaddump`, Django
  `DEBUG=True` error pages (settings and stack traces), and Swagger UI left open in production are
  standard findings. Put management endpoints on a separate port.
- **Body and upload limits.** `multer` memory storage buffers whole files in RAM (OOM on large
  uploads); nginx `client_max_body_size` (default 1 MB) rejects with 413 before the app sees
  anything — so "uploads fail at exactly 1 MB" is an ingress setting, not an app bug (`F17`).

### 15. Configuration, upgrades and changed defaults

- **Environment variables are strings.** `if (process.env.FEATURE_ENABLED)` is true for `"false"`;
  `Number("")` is `0`; `parseInt("30s")` is `30`. Validate config at startup with a schema
  (`@nestjs/config` with Joi or zod, Spring `@ConfigurationProperties` with `@Validated`,
  pydantic-settings) and fail fast (`F05`).
- **Config precedence.** Spring's property sources (command line > environment > profile files >
  `application.yml`) and relaxed binding (`SPRING_DATASOURCE_HIKARI_MAXIMUMPOOLSIZE`) mean a stray
  environment variable in the deployment overrides the file you are reading. `/actuator/env`
  (safely exposed) or logging resolved values at startup settles it.
- **Upgrades that change defaults** are a recurring incident class: Node 18 added the 300 s
  `requestTimeout` (long uploads and long-poll endpoints started failing at five minutes); Node 19
  made the global agent keep-alive; Nest 10's millisecond cache TTLs (§13); Nest 11's path syntax
  (§2); Spring Boot 3.4's graceful shutdown by default; Hibernate 6's ID-generation and query
  changes; Django 4.0's CSRF origin scheme. Read migration guides for *default* changes, not just
  breaking API changes.
- **Hibernate 6 / Boot 3 sequences.** `@GeneratedValue(strategy = AUTO)` stopped sharing one
  `hibernate_sequence` and now expects a sequence per entity (`<entity>_seq`), with the pooled
  optimiser and `allocationSize` 50. If the database sequence still increments by 1, each instance
  reserves "blocks" of 50 that overlap other instances' blocks — duplicate primary keys under
  concurrency, or inserts failing because the expected sequence does not exist. Make the sequence's
  `INCREMENT BY` match `allocationSize`, and restart it above the current maximum ID.
- **`NODE_ENV`.** Unset means some libraries run in development mode (Express caches views only in
  production; some loggers pretty-print synchronously). Spring profiles selected by an environment
  variable that one environment forgot produce "works in staging".
- **Config reloads.** Hot-reloaded config that recreates a pool or client on each change leaks the
  old one unless it is closed.

### 16. Diagnostic toolkit and unconventional techniques

**Node.** `--inspect` (attached through a port-forward, never exposed), `kill -USR1 <pid>` to
enable the inspector on a running process, `--cpu-prof` / `--heap-prof`, `--heapsnapshot-signal`,
`process.report.writeReport()` for a JSON dump of the JS stack, heap summary, handles and libuv
state, `--trace-warnings`, `why-is-node-running` for a process that will not exit,
`monitorEventLoopDelay` for lag, and `process.getActiveResourcesInfo()` for what keeps the loop
alive.

**JVM.** Three thread dumps 5–10 s apart (`jstack` or `jcmd <pid> Thread.print`) — threads stuck in
the same frame across all three are the answer (`HikariPool.getConnection`, a `synchronized`
monitor, a socket read with no timeout). async-profiler for CPU, allocation and lock flame graphs,
JFR for continuous low-overhead recording, `jcmd VM.native_memory`, GC logs (`-Xlog:gc*`).

**Python.** `py-spy dump --pid` (no restart, no code change) shows every thread's stack in a hung
gunicorn worker; `py-spy top` / `record` for profiles; `faulthandler` on a signal; `tracemalloc`;
django-debug-toolbar and django-silk for query counts; `celery inspect active` / `reserved` for what
workers hold.

**Database side.** `pg_stat_activity` (`state`, `wait_event`, `xact_start`, `application_name`),
`pg_blocking_pids()` to find the blocker, `pg_stat_statements` for top queries by total time,
PgBouncer `SHOW POOLS` (`cl_waiting` is your queue).

**Network.** `ss -tanp` to count connections by state (thousands in `CLOSE_WAIT` means *your* side
is not closing; thousands in `TIME_WAIT` means you are opening and closing too often), `tcpdump` on
the pod to see who sends the RST (it settles every keep-alive argument, §1), and load balancer
access logs with backend status codes (an ALB 502 with `target_status_code` of `-` means the target
never answered).

**Unconventional techniques experienced engineers reach for:**

- **Read the round number.** 100 ms, 2 s, 5 s, 30 s, 60 s and 300 s latencies map to specific
  defaults (§1, §6, §8) and shorten an investigation from days to minutes.
- **Take one pod out of rotation** (make its readiness fail or remove its Service label) and debug
  it live — heap snapshots, profilers, inspectors — without user impact and without losing the
  evidence to a restart.
- **Correlate with deploys, cron and traffic shape** before reading code: a problem at :00 every hour
  is a scheduled job; one after every deploy is shutdown or startup (§5); one at low traffic but not
  high is usually keep-alive or idle-connection related (§1, §8).
- **Invariant load tests.** Fire concurrent requests with unique markers and assert every response
  and log line carries its own marker — this catches singleton state and ALS leaks (§3, §4) that unit
  tests never will.
- **Count, do not time.** Query counts per request, connections per pod, job executions per run:
  counts reveal N+1, pool deadlocks and duplicate cron runs that timings hide.
- **Restart as mitigation, not as the fix.** Worker recycling, pod restarts and larger limits buy
  time; the senior move is to buy time *and* capture evidence (a heap snapshot, a thread dump,
  `py-spy dump`) before the restart destroys it.
- **Guard-rails in the shared resource.** `statement_timeout`, `idle_in_transaction_session_timeout`,
  PgBouncer in front of the database, Redis `noeviction`, per-tenant concurrency limits — the shared
  resource protects itself from every service at once, including the next one with the same bug.
- **Make failure loud.** Acquire timeouts instead of waiting forever, `forbidNonWhitelisted` instead
  of silent stripping, fail-fast config validation, alerts on `MaxListenersExceededWarning`, on
  event-loop lag, and on `HHH000104` in the logs.

---

## Questions

### Level 1 — Everyday incidents: 502s, timeouts and slow endpoints

The problems every on-call engineer meets in their first year; the interviewer wants the mechanism,
not "we added more pods".

1. "Our NestJS API sits behind an AWS ALB. Around 0.1% of requests get a 502, spread evenly through
   the day and slightly worse overnight. The app logs show nothing for those requests, and CPU and
   memory are fine. What is going on?"
   > **Direction:** Keep-alive race — Node's 5 s `keepAliveTimeout` is shorter than the ALB's 60 s idle timeout, so the ALB reuses sockets Node has just closed; set `keepAliveTimeout` above 60 s and `headersTimeout` above that (§1, `F28`, `F01`).

2. "Users report that uploading a PDF works for small files but fails for anything above about a
   megabyte with a 413. There is no log line in the Nest service at all, and the upload code has no
   size check. Where is the limit coming from?"
   > **Direction:** A 413 with no app log is rejected before the app — nginx ingress `client_max_body_size` defaults to 1 MB, and Express's JSON parser to 100 kB for JSON bodies; raise it at the right hop and stream rather than buffer (§14, §2, `F17`).

3. "A Django report endpoint fails at exactly 30 seconds every time with a 502 from nginx. In the
   gunicorn log you see `WORKER TIMEOUT (pid:…)`. The team wants to set `--timeout 300`. What do you
   say?"
   > **Direction:** A round 30 s is gunicorn killing a silent sync worker; raising it lets one slow report hold a whole worker (worker count is concurrency) — move the report to a Celery job with a download link, and fix the query (§1, §6, `F21`, `F13`).

4. "Our service calls an internal pricing API through a keep-alive HTTP agent. Every morning, and
   after any quiet spell, the first few calls fail with `ECONNRESET: socket hang up`, then everything
   is fine. Retrying manually always works. Explain it and fix it properly."
   > **Direction:** The mirror-image keep-alive race — our agent keeps sockets idle longer than the upstream's keep-alive, so it reuses dead sockets; set the client idle timeout below the server's and retry idempotent calls once on a reused-socket reset (§1, `F24`).

5. "An export endpoint builds a 30 MB JSON array for large customers. While it runs, p99 latency on
   *every* endpoint of that pod goes to several seconds, although p50 is fine and the pod shows only
   30% CPU across its four cores. Why do unrelated endpoints suffer?"
   > **Direction:** One CPU-bound `JSON.stringify`/serialisation blocks the single JS thread, so every request on that process waits; 30% of four cores is one pinned core — stream the export, move it to a worker or job, and alert on event-loop lag (§6, `C03`, `F24`).

6. "A NestJS service with TypeORM started 'hanging' under a traffic spike: requests never return, no
   errors are logged, the database's CPU is at 10% and `pg_stat_activity` shows ten idle-ish
   connections from this service. What happened, and why is there no error?"
   > **Direction:** Pool starvation; node-postgres's default `connectionTimeoutMillis: 0` means requests wait forever for one of the 10 pooled connections — find what holds connections, and set an acquire timeout plus pool metrics so it fails loudly (§8, `DB21`, `F09`).

7. "A DRF list endpoint returning 50 orders takes 3 seconds in production but 80 ms locally. The
   database is healthy, and the slow-query log shows nothing slow. What do you check first?"
   > **Direction:** Count queries, not time — a `SerializerMethodField` or nested serializer doing N+1 produces hundreds of fast queries that never appear in a slow log; fix with `select_related` / `prefetch_related` and pin it with `assertNumQueries` (§10, §16, `F21`, `DB40`).

8. "A Spring Boot order service starts throwing `Connection is not available, request timed out
   after 30000ms` every Black Friday. The database is barely loaded. The `placeOrder` method is
   `@Transactional` and calls the payment provider in the middle. What is wrong?"
   > **Direction:** The transaction holds a Hikari connection for the whole payment call, so throughput is capped by pool size ÷ payment latency (Little's law); move the external call outside the transaction (or split into two short transactions with a state machine) (§8, §9, `F10`, `F22`).

9. "When a user registers with an email that already exists, our Nest API returns a 500 with an
   internal error, and the alert fires. The service checks for an existing user first. Why does
   this still happen, and what should the response be?"
   > **Direction:** Check-then-insert races under concurrency, and the unique violation surfaces as a TypeORM/Prisma error that an `HttpException`-only filter maps to 500 — rely on the unique constraint and translate the driver error code to 409 in a filter (§2, §9, `F07`).

10. "In one Nest service, a custom Express middleware mounted with `app.use()` calls an async token
    introspection endpoint. When the introspection service is down, affected requests hang until the
    ALB times them out at 60 s, and occasionally the process restarts. Why?"
    > **Direction:** Raw Express 4 middleware does not catch rejected promises: the request never gets `next(err)`, and the unhandled rejection crashes Node 15+ — wrap with try/catch or `next(err)`, add a timeout, or move it into a Nest guard (§2, §1, `F03`).

11. "After upgrading Node from 16 to 20, a long-running CSV import endpoint that used to take six or
    seven minutes now fails at exactly five minutes. Nothing else changed. What happened?"
    > **Direction:** Node 18 introduced a 300 s default `server.requestTimeout`; an exact round number is a default — raise it for that server (or better, make the import async with a job and a status endpoint) and read upgrade notes for changed defaults (§1, §15, `F28`).

12. "Our gateway has a 15-second timeout. The service behind it calls a flaky dependency with three
    retries of 10 seconds each. During a dependency slowdown, users see 504s and the dependency's
    load triples. Walk me through what is happening."
    > **Direction:** Timeout budgets are inverted — the client gives up at 15 s while the server keeps retrying for 30 s, multiplying load; propagate a deadline, make inner timeout × retries fit the outer budget, and use backoff and a retry budget (§1, `M09`).

13. "A Nest endpoint `GET /users?active=false` returns only active users. The DTO has
    `active?: boolean` and the global `ValidationPipe` uses `transform: true` with implicit
    conversion. Explain the bug."
    > **Direction:** Implicit conversion calls `Boolean("false")`, which is `true`; use an explicit `@Transform` for boolean query params and test with string inputs (§12, `F06`).

14. "Our frontend occasionally fetches the wrong record: the ID in the URL is `1234567890123456800`
    but the database row is `1234567890123456789`. The backend is Spring Boot with snowflake IDs. How
    did the ID change?"
    > **Direction:** 64-bit IDs serialised as JSON numbers exceed 2^53 and are rounded by `JSON.parse` in the browser; serialise them as strings (Jackson `ToStringSerializer`) end to end (§12, `A17`).

15. "A dashboard widget shows '421 orders' where it should show 43 — the database has 42. The Nest
    service adds 1 to the result of a raw `SELECT COUNT(*)` query via node-postgres. What is going
    on?"
    > **Direction:** node-postgres returns `int8`/`numeric` as strings, so `"42" + 1` concatenates; cast in SQL (`::int`), parse explicitly, or register a type parser (§10, `F09`).

16. "A Django view that shows 'you have pending invoices' became slow as customers grew; one large
    customer's page takes 8 seconds. The code is `if invoices: show_banner()`. What is wrong?"
    > **Direction:** Truth-testing a queryset evaluates and loads every row; use `.exists()`, and know which operations evaluate a lazy queryset (§10, `F21`).

17. "We moved our Nest service behind a new load balancer and suddenly every user is getting 429 Too
    Many Requests at lunchtime. The throttler config did not change. Why?"
    > **Direction:** Without `trust proxy`, `req.ip` is the load balancer's address, so all users share one rate-limit bucket; trust exactly the right number of hops (and use shared storage across pods) (§13, §14, `F26`).

18. "When one tenant triggers a heavy PDF generation, Kubernetes restarts the pod, killing the
    other 200 in-flight requests. The liveness probe is `GET /health` on the same port. Explain the
    chain of events."
    > **Direction:** CPU work blocks the event loop, so the health endpoint cannot answer, liveness fails and the pod is killed — move CPU work off the loop (worker thread or job), and make liveness tolerant rather than a hair trigger (§5, §6, `F28`).

19. "A Spring Boot service in a 2 GiB container dies with `OutOfMemoryError: Java heap space`, but
    the container metrics show it never used more than 900 MB. No `-Xmx` is set. What is going on?"
    > **Direction:** The container-aware JVM defaults `MaxRAMPercentage` to 25%, giving a ~512 MiB heap; set `MaxRAMPercentage` to ~70–75% and leave headroom for non-heap memory (§7, `F22`, `C07`).

20. "Order timestamps shown in the admin are one hour off in summer, but only in production — on
    developers' laptops they are correct. The column is `timestamp` and the service is Nest with
    node-postgres. Why?"
    > **Direction:** `timestamp without time zone` is parsed in the process's local time zone, and containers run in UTC while laptops do not; store `timestamptz`, serialise UTC, and set `TZ` explicitly (§10, §12).

### Level 2 — Connection pools and database access

Pools are where most "the database is slow" incidents actually live; the interviewer wants
arithmetic and an understanding of who holds what.

1. "We scaled our Django API from 5 to 30 pods for a marketing campaign. At peak, Postgres started
   rejecting connections with `too many clients already`, and the site went down — when it had been
   fine with fewer pods. Why did more capacity cause an outage?"
   > **Direction:** Connections multiply — pods × gunicorn workers × connections each — against `max_connections` (100 by default); size from the database down and put PgBouncer or a small per-process pool in front (§8, `DB21`, `F24`).

2. "Our Nest API runs on AWS Lambda with Prisma. Under load we get `P2024: Timed out fetching a new
   connection from the connection pool`, and the RDS instance hits its connection limit. Each
   invocation is short. What is happening?"
   > **Direction:** Each concurrent Lambda instance gets its own Prisma pool (default `cpus × 2 + 1`), so thousands of pools exhaust the database; set `connection_limit=1`, reuse the client across invocations, and put RDS Proxy or PgBouncer in front (§8, §10).

3. "Every Monday morning the first requests to our Spring service hang for about 15 minutes and then
   fail, then everything is normal. The service talks to a database in another VPC through a NAT
   gateway. Weekends are quiet. What is going on?"
   > **Direction:** The NAT gateway (350 s) or firewall silently drops idle flows, so the pool hands out dead connections and TCP retransmits for minutes — set `maxLifetime`/`idleTimeout` below the middlebox limit and enable TCP keepalive (`keepaliveTime`) (§8, `DB21`).

4. "We put PgBouncer in transaction mode in front of Postgres to cut connections. Immediately our
   Spring services started throwing `ERROR: prepared statement "S_1" already exists`
   intermittently. Explain."
   > **Direction:** Named server-side prepared statements are session state, and transaction pooling hands the next transaction a different server connection; disable server-side prepares (`prepareThreshold=0`) or use PgBouncer 1.21+ `max_prepared_statements` (§8).

5. "We use Postgres row-level security with the tenant set via `SET app.tenant_id = …` at the start
   of each request. After we introduced PgBouncer, a customer saw another tenant's invoices. How is
   that possible?"
   > **Direction:** A session-level `SET` survives on the server connection, which PgBouncer's transaction mode gives to someone else next; use `SET LOCAL` (or `set_config(..., true)`) inside each transaction so the setting dies with it (§8, `S05`).

6. "A NestJS checkout flow works perfectly in tests and in staging. In production, at about 10
   concurrent checkouts, the whole service freezes with no errors. The code uses
   `dataSource.transaction()` and inside it calls `this.ordersRepository.save()`. What is wrong?"
   > **Direction:** Pool deadlock — each transaction holds one of the 10 connections and the injected repository asks for a second, so ten requests wait on each other forever (and that write is outside the transaction); use the transaction's `manager` (or ALS-propagated transactions) (§8, §9, `F10`).

7. "A Spring service writes an audit record with `@Transactional(propagation = REQUIRES_NEW)` from
   inside the main transaction. Under load, threads pile up and requests time out after 30 s with
   Hikari errors. With low traffic it is fine. Why?"
   > **Direction:** `REQUIRES_NEW` takes a second connection while the outer one is still held; at concurrency equal to the pool size everyone holds one and waits for another — size the pool for two per request, use a separate pool for audit, or write the audit after commit (§8, §9, `F22`).

8. "Our Spring API's Hikari pool shows all connections active, yet the database shows most of them
   idle and the controller methods themselves are fast. Many clients are on slow mobile networks.
   What might hold the connections?"
   > **Direction:** Open-session-in-view (on by default) keeps the Hibernate session and its connection bound for the whole request, including serialisation and writing to slow clients; set `spring.jpa.open-in-view=false` and fetch what the DTO needs inside the service (§8, §12, `F22`).

9. "A Django API spends 15–25 ms per request just connecting to the database, and Postgres logs
   show thousands of connections per minute. Someone suggests `CONN_MAX_AGE=600`. What are the
   trade-offs, and what would you actually do?"
   > **Direction:** The default `CONN_MAX_AGE=0` connects per request; persistent connections fix latency but multiply held connections per worker/thread and can go stale — combine them with `CONN_HEALTH_CHECKS`, or use Django 5.1's psycopg pool or PgBouncer (§8, `F21`).

10. "To fix slow requests, a teammate raised the Hikari pool from 10 to 80 per pod across 12 pods.
    Throughput went *down* and database CPU went up. Explain why a bigger pool was slower."
    > **Direction:** Past a small multiple of database cores, more active connections add contention and context switching rather than throughput; 12 × 80 is ~1 000 backends — size from database capacity with Little's law and queue in the app (§8, `DB21`).

11. "During a database slowdown, our HPA scaled the API from 10 to 40 pods on CPU and latency. The
    database, which was struggling, then fell over completely. Why did autoscaling make it worse,
    and how do you design against this?"
    > **Direction:** Every new pod brings a full pool, so scaling out multiplies connections and load exactly when the shared resource is weakest; derive pool size from a global budget ÷ max replicas, cap replicas, and use a pooler as the shared limiter (§8, `F24`).

12. "Our Spring service logs `HikariPool-1 - Failed to validate connection … Possibly consider using
    a shorter maxLifetime value` a few times an hour, followed by occasional query failures. The
    database is MySQL. What is going on?"
    > **Direction:** The server (MySQL `wait_timeout`) or a proxy closes connections before Hikari retires them; set `maxLifetime` a little below the smallest server/infrastructure limit and enable keepalive (§8).

13. "Postgres shows 40 sessions `idle in transaction` for over an hour, blocking autovacuum. Six
    services share the database and all use the same DB user. How do you find the culprit quickly,
    and how do you stop it happening again?"
    > **Direction:** Set `application_name` per service and pod in connection strings so `pg_stat_activity` attributes sessions, and enforce `idle_in_transaction_session_timeout` as the database's own guard-rail (§8, §9, §16).

14. "After enabling `--preload` on gunicorn to save memory, our Django app intermittently raises
    `SSL error: decryption failed or bad record mac` and `server closed the connection unexpectedly`.
    What did preload change?"
    > **Direction:** A connection opened during app import before the fork is shared by all workers, which interleave bytes on one socket; make sure nothing connects at import time and open connections post-fork (§7, §8, `F21`).

15. "A migration that added a column with a default froze the whole orders table for ten minutes:
    every query on it timed out. The migration itself is instant on staging. What happened?"
    > **Direction:** The `ALTER` waited for an `ACCESS EXCLUSIVE` lock behind a long `idle in transaction` session, and every query queued behind the `ALTER`; set `lock_timeout` on migrations, retry, and kill idle transactions (§9, `DB22`).

16. "A handful of customers downloading big CSV exports seem to take down our Django service. Each
    export streams rows from a queryset to the response. The database is fine. What resource is
    running out?"
    > **Direction:** Each streaming response to a slow client holds a sync worker and a database connection/cursor for its full duration; generate exports in a background job to object storage and hand out a link (§6, §8, `F17`).

17. "In development, after a few hours of hot reloading, our Nest app with Prisma fails with
    `too many clients already`. It never happens in production. Why, and why does it matter for
    production anyway?"
    > **Direction:** Each reload creates a new `PrismaClient` and pool without closing the old one; cache the client on `globalThis` in dev and ensure exactly one client per process — the same bug appears in serverless and in code that creates clients per request (§10, §8).

18. "We turned on PgBouncer transaction pooling and a Django management command that uses
    `.iterator()` over a large table started failing with `cursor "_django_curs_…" does not exist`.
    Why?"
    > **Direction:** `.iterator()` uses a named server-side cursor, which lives in a session that transaction pooling does not preserve between statements; set `DISABLE_SERVER_SIDE_CURSORS=True` for that database or use a session-mode pool for batch work (§10, §8).

19. "After we routed `@Transactional(readOnly = true)` methods to a read replica, users occasionally
    update their profile and then see the old values on the next page. Explain and fix."
    > **Direction:** Replica lag breaks read-your-writes; route to the primary for a short window after a user's write (sticky primary, or pass a write marker), or read critical data from the primary (§9, `DB23`).

20. "`pg_stat_statements` shows one query consuming 40% of database time, but it is generated by an
    ORM and five services could issue it. How do you tie it to an endpoint without guessing?"
    > **Direction:** Tag SQL with comments carrying the route, service and trace ID (sqlcommenter-style) and set `application_name`, so the database's own views name the caller (§8, §16, `O13`).

### Level 3 — Transactions and ORM behaviour

The framework hides transaction boundaries and SQL; these questions test whether you know exactly
where they are.

1. "A Django view creates an order and calls `send_confirmation.delay(order.id)`. About 2% of the
   Celery tasks fail with `Order.DoesNotExist`, and when they retry they succeed. The view uses
   `ATOMIC_REQUESTS`. What is happening?"
   > **Direction:** Enqueue-before-commit — the fast worker reads before the view's transaction commits; enqueue with `transaction.on_commit` (or `delay_on_commit`), or use an outbox for guaranteed delivery (§9, §11, `F10`, `F13`, `M15`).

2. "A Spring service method annotated `@Transactional` inserts an invoice, then its lines, then
   calls a validator that throws a checked `InvalidInvoiceException`. The API returns an error, yet
   half-written invoices appear in the database. Why?"
   > **Direction:** By default Spring rolls back only on unchecked exceptions; a checked exception commits the work done so far — use `rollbackFor = Exception.class` or unchecked domain exceptions (§9, `F22`).

3. "A developer wrapped a call to `inventoryService.reserve()` in a try/catch so that a failed
   reservation would not fail the order. Both methods are `@Transactional`. Now some orders fail at
   the very end with `UnexpectedRollbackException: Transaction silently rolled back because it has
   been marked as rollback-only`. Explain."
   > **Direction:** The inner `REQUIRED` method joins the outer transaction and its proxy marks it rollback-only on the exception, so catching it cannot save the commit; use `REQUIRES_NEW`/`noRollbackFor` deliberately, or do not make the inner call transactional (§9, `F10`).

4. "In a Spring `OrderService`, `placeOrder()` calls `this.saveWithAudit()`, which is annotated
   `@Transactional`. When the audit insert fails, the order row stays committed. There is no
   `@Transactional` on `placeOrder`. Why did it not roll back?"
   > **Direction:** Self-invocation bypasses the AOP proxy, so `saveWithAudit` runs with no transaction at all; move it to another bean, put the boundary on the public entry method, or inject the proxy (§3, §9, `F22`).

5. "A read-only-looking endpoint in a Spring app — `GET /orders/{id}` — shows up in the database's
   write statistics and occasionally deadlocks with the order-update job. The code loads the entity
   and maps it to a DTO. How can a GET write?"
   > **Direction:** A mapper or getter mutated the managed entity, and Hibernate's dirty checking flushed an `UPDATE` at commit; map from a detached copy or projection, use `readOnly = true`, and look for unexpected `UPDATE`s in SQL logs (§9, §10, `DB40`).

6. "A paginated admin endpoint in Spring Data JPA — 20 customers per page, with their orders
   fetch-joined — was fine for years, then started OOM-ing the pod as data grew. The query has
   `LIMIT` in the code. What is Hibernate doing?"
   > **Direction:** Fetch-joining a collection with pagination makes Hibernate load all rows and paginate in memory (`HHH000104` / `HHH90003004`); page the IDs first, then fetch by IDs, and alert on that warning (§10, §16, `DB40`).

7. "A nightly Spring Batch-style job reads two million rows in one `@Transactional` method, updates a
   field on each and saves. It starts fast, slows to a crawl after about 200 000 rows and dies with
   an OOM. Why is it getting slower as well as bigger?"
   > **Direction:** The persistence context holds every loaded entity and dirty-checks all of them on each flush; process in chunks with `flush()`/`clear()`, a `StatelessSession`, or set-based SQL (§10, `F22`).

8. "Importing 100 000 products with `saveAll()` in Spring Data takes 20 minutes although batching is
   configured with `hibernate.jdbc.batch_size=50`. The SQL log shows one `INSERT` per round trip.
   Why is batching not happening?"
   > **Direction:** `GenerationType.IDENTITY` forces Hibernate to insert row by row to get each generated key, which disables JDBC batching; switch to a pooled `SEQUENCE`, or use `COPY`/bulk SQL for imports (§10).

9. "We renamed a column in a TypeORM entity from `name` to `fullName`. After the deploy, every
   user's name was empty. Nobody ran a migration. What happened, and what should the setup be?"
   > **Direction:** `synchronize: true` in production dropped the old column and created a new empty one; disable it, generate and review migrations, and do renames as expand/contract (§10, §15, `DB22`).

10. "A Nest service uses Prisma's interactive `$transaction(async (tx) => …)` to create a booking and
    call the payment provider inside. Under load, some bookings fail with `Transaction already
    closed` / P2028, even though the payment succeeded. What went wrong?"
    > **Direction:** Interactive transactions default to a 5 s timeout (and 2 s `maxWait`), and a slow external call inside blows it after the charge has happened; never call external services inside a transaction — reserve, commit, charge, then confirm with idempotency (§9, `F10`, `SD10`).

11. "During a flash sale we sold 130 units of a product with a stock of 100. The code reads the
    product, checks `stock > 0`, then decrements and saves, all inside a transaction. How did
    it oversell, and what are your options?"
    > **Direction:** READ COMMITTED lets concurrent transactions read the same stock before either writes; use an atomic conditional `UPDATE … WHERE stock > 0`, `SELECT … FOR UPDATE`, or a version column with retry (§9, `DB09`).

12. "Our Django tests for a wallet debit using `select_for_update()` all pass. In production the same
    code throws `TransactionManagementError: select_for_update cannot be used outside of a
    transaction`. How can the tests have missed it?"
    > **Direction:** `TestCase` wraps every test in a transaction, hiding that production code runs in autocommit; wrap the code in `transaction.atomic()` and use `TransactionTestCase` for concurrency tests (§9, `F12`, `F21`).

13. "After we added `select_for_update()` to the order-processing view, lock waits and deadlocks
    appeared on the `customers` table, which the code only reads. The query uses
    `select_related('customer')`. Why is the customer locked?"
    > **Direction:** `select_for_update` locks every row in the join unless you pass `of=("self",)`, so concurrent orders for the same customer contend on the customer row (§9, `F21`).

14. "We use a Postgres table as a job queue. Five workers poll with `SELECT … FOR UPDATE LIMIT 1`.
    Throughput does not improve past one worker and there are constant lock waits. How do you fix it
    without adding a broker?"
    > **Direction:** All workers block on the same first row; use `FOR UPDATE SKIP LOCKED` (`select_for_update(skip_locked=True)`) so each worker takes the next free row (§9, `DB09`).

15. "A Django `post_save` signal on `Payment` publishes a `payment.completed` event to Kafka. Now
    and then, a downstream service receives the event for a payment that is later rolled back and
    never exists. Explain."
    > **Direction:** Signals run synchronously inside the transaction, so the event leaves before the commit and survives a rollback; publish from `on_commit` or, for guaranteed consistency, via a transactional outbox (§9, `M15`).

16. "A TypeORM `find()` for 50 projects with `relations: ['tasks', 'members', 'tags']` takes 12
    seconds and the Node process's memory jumps by 1 GB. Each project has a few hundred tasks and
    members. What is happening?"
    > **Direction:** Joining several one-to-many relations in one query produces a cartesian product (tasks × members × tags rows per project); use `relationLoadStrategy: 'query'` or separate queries (§10, `DB40`).

17. "We turned off open-session-in-view in Spring. Now several endpoints throw
    `LazyInitializationException` during JSON serialisation, and one previously returned a 50 MB
    response. What does this tell you, and how do you fix it properly?"
    > **Direction:** Jackson was serialising entities and lazily loading relations outside the service layer — the 50 MB response was an entity graph; return DTOs fetched explicitly (fetch joins or entity graphs) and keep OSIV off (§8, §12, `F22`).

18. "Products created by the new bulk import do not appear in search, although products created
    through the admin do. Search indexing is triggered by a `post_save` signal. The import uses
    `bulk_create`. What is going on?"
    > **Direction:** `bulk_create` and `.update()` skip `save()` and signals; trigger indexing explicitly for bulk paths or use change data capture instead of ORM hooks (§10, `M15`).

19. "A Nest endpoint that deletes items by ID works for small batches, but a request with 70 000 IDs
    fails with a Postgres protocol error about bind parameters. How do you handle this?"
    > **Direction:** Postgres allows at most 65 535 bind parameters per statement, and `In([...])` uses one per value; chunk the list, or pass a single array parameter (`= ANY($1)`) (§10).

20. "We switched a financial reconciliation service to SERIALIZABLE isolation to fix a race. Now
    about 1% of requests fail with `could not serialize access due to read/write dependencies`, and
    retrying in the service did not help. What did the retry get wrong?"
    > **Direction:** Serialisation failures (40001) must be retried by re-running the whole transaction from outside; a retry inside the aborted transaction, or an advisor inside the transactional proxy, just fails again (§9, `DB06`).

### Level 4 — The pipeline, DI and framework lifecycle

Framework magic that works until it silently does not; the interviewer wants to hear that you know
the order of things and what the container actually builds.

1. "After someone added a tenant-aware logger that injects `REQUEST`, our Nest service's throughput
   dropped by 40% and GC time tripled. The logger itself is trivial. What happened?"
   > **Direction:** Request scope bubbles up: everything injecting the logger, up to the controllers, became request-scoped and is rebuilt per request; move per-request data to AsyncLocalStorage (`nestjs-cls`) and keep providers singletons (§3, `F04`, `F19`, `F20`).

2. "Under load, a user of our Nest app briefly saw another user's shopping basket. The
   `BasketService` sets `this.userId` in a guard-called method and reads it later. It never happens
   in manual testing. Explain."
   > **Direction:** Default-scope providers are singletons shared by all concurrent requests, so an instance field is a race; keep per-request state in ALS or method arguments, and prove it with a concurrent invariant test (§3, §16, `F04`).

3. "A Nest guard checks that the user owns the resource: `req.params.id === user.accountId`. The
   route has `ParseIntPipe` on `id`, `accountId` is a number, and the guard always denies. Why?"
   > **Direction:** Guards run before pipes, so the guard sees the raw string param; convert in the guard or move the ownership check after validation (into an interceptor or the service) (§2, `F03`, `F11`).

4. "We registered a global auth guard with `app.useGlobalGuards(new AuthGuard())` in `main.ts`. It
   cannot get `ConfigService`, and in our hybrid app the RabbitMQ message handlers are not
   protected at all. Why?"
   > **Direction:** Guards instantiated in `main.ts` live outside the DI container and are not applied to hybrid microservice transports; register it as `APP_GUARD` in a module so it is injected and bound properly (§2, `F19`).

5. "An endpoint that returns a user started including the `passwordHash` field after a refactor,
   although the entity has `@Exclude()` and `ClassSerializerInterceptor` is global. The refactor
   switched the handler to `@Res()` to set a cookie. Why did the exclusion stop working?"
   > **Direction:** `@Res()` puts the route in library-specific mode, bypassing response-mapping interceptors; use `@Res({ passthrough: true })` and serialise response DTOs rather than relying on entity decorators (§2, `F06`, `F26`).

6. "Our Nest latency dashboard looks excellent, but customers complain of failures. The metrics come
   from a timing interceptor using `tap()` on the observable. What is the dashboard missing?"
   > **Direction:** `tap(next)` fires only on success, so errored requests are never recorded; use `finalize()` or handle the error branch, and record status codes (§2, `F08`, `O13`).

7. "Our Nest app boots fine with `nest start --watch` but the production build crashes with `Nest
   can't resolve dependencies of PaymentsService (?)`. Nothing changed in the module definitions.
   What does the `?` tell you?"
   > **Direction:** The injected class was `undefined` when decorator metadata was emitted, typically from a circular file import through barrel files that resolves differently in the compiled output; find it with `madge --circular` and break the cycle rather than adding `forwardRef` (§3, `F20`).

8. "After upgrading to Nest 11, the admin audit middleware bound with `forRoutes('admin/*')` stopped
   running on nested routes, and nobody noticed for a week. Why, and how do you stop this class of
   bug?"
   > **Direction:** Nest 11 moved to Express 5 / path-to-regexp v8, where wildcards must be named (`*splat`), so the old pattern stopped matching silently; test that security middleware actually runs on each route and read upgrade notes for default and syntax changes (§2, §15).

9. "We added an observability starter to our Spring Boot app. Suddenly the API started failing on
   unknown JSON fields and serialising dates as numbers. We never touched Jackson config. What
   happened?"
   > **Direction:** The starter defined its own `ObjectMapper` bean, so Boot's `@ConditionalOnMissingBean` auto-configured mapper (with its lenient settings and your `spring.jackson.*` properties) backed off; customise via `Jackson2ObjectMapperBuilderCustomizer` instead of replacing the bean (§3, §12, `F22`).

10. "A Spring method is annotated `@Cacheable("prices")` but the cache is never hit, and a
    `@Transactional` method in the same class does not roll back. Both are `private` helpers called
    from a public method. Explain."
    > **Direction:** Proxy-based AOP only intercepts external calls to public methods; private methods and self-invocation bypass it — move them to separate beans (§3, `F15`, `F22`).

11. "A Spring singleton `ReportService` injects a prototype-scoped `ReportBuilder` that keeps state.
    Reports started mixing data from different users under load. Prototype should mean a new one
    each time — so why is it shared?"
    > **Direction:** A prototype injected into a singleton is created once, at injection; use `ObjectProvider<ReportBuilder>`/lookup to get a new instance per use, or make the builder stateless (§3, `F04`).

12. "An exception thrown in our custom Spring servlet filter returns an HTML 'Whitelabel Error Page'
    instead of our JSON problem format, even though we have a global `@ControllerAdvice`. Why?"
    > **Direction:** Filters run before the `DispatcherServlet`, so `@ControllerAdvice` never sees their exceptions; write the error response in the filter, or delegate to a `HandlerExceptionResolver` (§2, `F07`).

13. "We added a custom Django middleware that reads `request.user` to set a tenant, and it raises
    `AttributeError: 'WSGIRequest' object has no attribute 'user'` in production only. The
    middleware list was merged from two branches. What is wrong?"
    > **Direction:** Middleware order is the onion — it must come after `SessionMiddleware` and `AuthenticationMiddleware`; review the whole `MIDDLEWARE` list order, not just the new entry (§2, `F21`).

14. "Our Nest service crash-loops whenever a feature-flag service is down, even though flags are
    optional. The flag client loads flags in `onModuleInit`. How do you make startup resilient?"
    > **Direction:** An awaited, failing `onModuleInit` blocks or kills boot, and startup probes turn it into a crash loop; start with defaults, load flags in the background with timeouts, and keep optional dependencies out of the boot path (§3, §5, `F19`).

15. "A scheduled cleanup job in our Nest app throws `Cannot read properties of undefined (reading
    'headers')` inside a service that works fine when called from controllers. What is different
    about the cron context?"
    > **Direction:** The service (or something it injects) depends on the request-scoped `REQUEST`, which does not exist in cron or queue contexts; decouple via ALS with an explicit context per job, not `REQUEST` (§3, §4, `F14`).

16. "Our global exception filter sometimes logs `ERR_HTTP_HEADERS_SENT` and the client receives a
    truncated file instead of an error. It happens on the endpoint that streams exports with
    `StreamableFile`. What is going on?"
    > **Direction:** Once headers are sent the status is committed, so a mid-stream error cannot become a JSON error response; validate before streaming, abort the connection on failure, and make clients verify completeness (length/checksum) (§2, `F17`).

17. "Sentry shows almost no errors from our Nest service, yet users see 500s. A response-wrapping
    interceptor was added last month. What would you check?"
    > **Direction:** An interceptor's `catchError` runs before exception filters and can swallow or remap errors before the filter that reports to Sentry; report in one place and ensure interceptors rethrow (§2, `F07`, `F08`).

18. "After upgrading Spring Boot from 2.5 to 2.7, the application refuses to start with 'The
    dependencies of some of the beans in the application context form a cycle'. It worked for
    years. Should we set `spring.main.allow-circular-references=true`?"
    > **Direction:** Boot 2.6 made circular references fail by default; the flag is a stopgap — break the cycle (extract a collaborator, use events or constructor injection with a redesign) (§3, §15, `F22`).

19. "A developer moved JWT validation from a guard into a Nest middleware 'for performance'. Now
    endpoints decorated `@Public()` require a token, and the health check started failing. Why?"
    > **Direction:** Middleware runs before the handler's metadata is available through `ExecutionContext`/`Reflector`, so it cannot see `@Public()`; authentication that depends on route metadata belongs in a guard (§2, `F03`, `F11`).

20. "A Spring app seeds reference data in a `@PostConstruct` method annotated `@Transactional`. When
    one insert fails halfway, the other rows stay committed and the app starts with partial data.
    Why did the annotation not help?"
    > **Direction:** `@PostConstruct` is called on the raw bean, not through the transactional proxy, so there is no transaction; seed in an `ApplicationRunner` / `@EventListener(ApplicationReadyEvent)` calling a proxied bean, or better, via migrations (§3, §9, `F22`).

### Level 5 — Background jobs, cron and queues

Work that runs outside a request fails in its own ways: twice, never, or on every replica at once.

1. "We scaled our Nest monolith from one pod to six. The next morning, every customer got six copies
   of the daily digest email. The job uses `@Cron('0 7 * * *')` from `@nestjs/schedule`. What
   happened and what are your options?"
   > **Direction:** In-process cron runs in every replica; move it to a Kubernetes `CronJob`, a single repeatable queue job, or guard it with a lock plus a unique "run for date X" record so a duplicate run is a no-op (§11, `F14`).

2. "A BullMQ worker that generates PDFs occasionally processes the same job twice — two invoices
   emailed, both 'completed'. Jobs take about 45 seconds of mostly CPU work. The logs mention
   stalled jobs. Explain the mechanism."
   > **Direction:** CPU work blocks the event loop so the 30 s job lock is not renewed, the stalled checker hands the job to another worker while the first is still running; use sandboxed processors or worker threads, raise `lockDuration`, and make the job idempotent (§11, §6, `F13`).

3. "Our Django app schedules a reminder with `send_reminder.apply_async(countdown=3*3600)`. Some
   users get the reminder three times, all at roughly the right time. The broker is Redis. What is
   going on?"
   > **Direction:** ETA/countdown tasks sit unacknowledged in a worker beyond Redis's 1-hour `visibility_timeout`, so the broker redelivers them to other workers; raise the visibility timeout above the longest ETA, or store long-dated schedules in the database and enqueue when due (§11, `Q10`).

4. "When a Celery worker is OOM-killed during a big export task, the task simply disappears — no
   failure, no retry, and the user waits forever. How do you make this failure visible and
   recoverable?"
   > **Direction:** With default early acknowledgement the message is gone once received; use `acks_late=True` with `task_reject_on_worker_lost=True` for redelivery, make the task idempotent, and track job state in the database with a timeout sweeper (§11, `F13`).

5. "After enabling `acks_late` and `task_reject_on_worker_lost`, one malformed upload made every
   Celery worker in the fleet crash, one after another, for an hour. What is this failure mode and
   how do you contain it?"
   > **Direction:** A poison message redelivered after each crash walks through the fleet; count delivery attempts and park the message in a dead-letter queue or failed state after N, and bound input size before processing (§11, `Q10`).

6. "Our Celery queue has a backlog of 2 000 tasks, yet `celery inspect active` shows only two of the
   eight workers busy; the idle ones have nothing to do. Tasks take 1–10 minutes. What is going
   on?"
   > **Direction:** Prefetch (multiplier 4 × concurrency) lets busy workers reserve tasks that then wait behind long ones; use `-O fair`, `worker_prefetch_multiplier=1` and `acks_late` for long tasks, and check `inspect reserved` (§11, §16).

7. "Jobs in our BullMQ queues started vanishing: producers get job IDs back, but workers never see
   them. Redis memory is at its limit, and the Redis is shared with the cache. What is happening?"
   > **Direction:** Completed/failed jobs are retained unless `removeOnComplete`/`removeOnFail` are set, memory fills, and an LRU eviction policy deletes queue keys; give queues their own Redis with `noeviction` and trim job history (§11, `Q02`).

8. "In a Spring Boot app, a nightly reconciliation `@Scheduled` job that takes 40 minutes causes
   every other scheduled task — cache refresh, heartbeats — to stop running while it runs. Why?"
   > **Direction:** Boot's default scheduler has a single thread (`spring.task.scheduling.pool.size=1`); give it more threads, or run long jobs on a dedicated executor or as separate workloads (§11, `F14`, `F22`).

9. "A Spring service sends notifications via an `@Async` method. During a slow provider incident,
   heap usage climbed steadily until the pod died, although the thread count stayed at eight.
   Explain."
   > **Direction:** Boot's default `ThreadPoolTaskExecutor` has core size 8 and an unbounded queue, so tasks pile up in memory and `max-size` is never reached; bound the queue, choose a rejection policy, and use a durable queue for work that must survive (§11, §7).

10. "We protect a nightly settlement job with a Redis lock (`SET NX PX 600000`). One night the job
    took 14 minutes and settlements were paid twice. What went wrong and what is a robust design?"
    > **Direction:** The lock's TTL expired while the job was still running, so a second instance acquired it; extend the lock with a heartbeat, and make the work itself idempotent with a unique "settlement for date X" row or a fencing token (§11, `C09`).

11. "A billing job scheduled at 01:30 Europe/London ran twice one Sunday in October and not at all
    one Sunday in March. Explain and fix."
    > **Direction:** Daylight saving repeats and skips the 01:00–02:00 hour; schedule in UTC or outside the transition hour, and make runs idempotent per business date (§11).

12. "A Celery task receives a `User` object as an argument and emails the user their updated plan.
    Sometimes the email shows the *old* plan, although the task runs seconds after the change. Why?"
    > **Direction:** Arguments are serialised at enqueue time, so the task works on a stale snapshot (and pickle is dangerous); pass IDs and re-read inside the task, after commit (§11, §9).

13. "All our Celery tests pass with `task_always_eager=True`. In production, the same flows fail
    with missing rows and `kombu` serialisation errors. What did eager mode hide?"
    > **Direction:** Eager execution runs tasks inline inside the caller's transaction with no serialisation, hiding the enqueue-before-commit race and non-serialisable arguments; test with a real broker and worker (§11, §9, `F12`).

14. "Our new BullMQ worker crashes at startup with an ioredis error about `maxRetriesPerRequest`, and
    when it does run, a Redis blip makes it stop processing permanently until restart. What is the
    configuration issue?"
    > **Direction:** BullMQ workers need an ioredis connection with `maxRetriesPerRequest: null` so blocking commands retry indefinitely rather than failing; use separate connection settings for queues and workers (§11).

15. "Every deploy, a few long import jobs end up half-done: some rows imported twice, some not at
    all. Workers run in the same deployment as the API with the default grace period. Explain the
    chain and the fix."
    > **Direction:** SIGKILL after the 30 s grace period cuts jobs mid-flight and they are redelivered; make the grace period fit the longest job (or checkpoint the job), close the worker gracefully first, and make imports idempotent (§5, §11, `F28`).

16. "Our Celery workers' memory grows from 300 MB to 2.5 GB over three days, then the node starts
    OOM-killing. Profiling locally shows no leak. What do you do now, and later?"
    > **Direction:** Buy time with `worker_max_tasks_per_child`/`worker_max_memory_per_child` (fragmentation means RSS rarely drops), then find the growth with `tracemalloc` snapshots on a production worker — and check `DEBUG` is off (§7, §16).

17. "A long-running Django management command that processes events for hours slowly uses more and
    more memory, though the loop processes one event at a time. `DEBUG` is `True` in that
    environment by mistake. Why does that matter?"
    > **Direction:** With `DEBUG=True` Django records every SQL query in `connection.queries`, an unbounded list in a long-lived process; turn off `DEBUG` (or call `reset_queries()`) and iterate with `.iterator()` (§7, §10, `F21`).

18. "We moved our nightly job to a Kubernetes `CronJob`. Occasionally it still runs twice, and once
    it did not run at all after a cluster upgrade. We set `concurrencyPolicy: Forbid`. Why?"
    > **Direction:** `CronJob` scheduling is best-effort — missed schedules (controller downtime, `startingDeadlineSeconds`) and rare double creation both happen; keep the job idempotent per business date and alert on "no successful run" (§11, `O06`).

19. "Logs from our BullMQ workers have no correlation ID and no tenant, so we cannot trace a failed
    job back to the request that created it. The API uses `nestjs-cls` for context. Why is the
    context missing, and how do you fix it?"
    > **Direction:** ALS context does not travel through Redis; put the correlation and tenant IDs in the job data or headers at enqueue time and start a new ALS context with them in the processor (§4, §11, `F08`).

20. "Periodic tasks started running twice after we moved Celery to Kubernetes. We run `celery worker
    --beat` in each of our two worker pods. What is wrong?"
    > **Direction:** Each embedded beat schedules independently; run exactly one beat process (its own single-replica deployment) or a lock-based scheduler, and keep tasks idempotent (§11).

### Level 6 — Deploys, shutdown and running in containers

The problems that appear only at the edges of a process's life, or because a container is not the
machine the runtime thinks it is.

1. "Every deploy of our Nest service produces a burst of about 200 502s over five seconds, then
   everything is fine. We have graceful shutdown with `enableShutdownHooks()`. What is still
   missing?"
   > **Direction:** Endpoint removal propagates to kube-proxy/ingress/LB after SIGTERM, so new requests still arrive at a pod that has stopped listening; add a `preStop` sleep, keep serving during it, then drain (§5, `F28`, `O06`).

2. "Our pods always take exactly 30 seconds to terminate, and in-flight requests are cut off at the
   end. The Node app has no signal handling, and the Dockerfile ends with `CMD npm run start:prod`.
   Explain."
   > **Direction:** npm (or a shell) is PID 1 and may not forward SIGTERM, and a PID 1 Node process without a handler ignores it, so the pod waits for SIGKILL; use exec-form `CMD ["node","dist/main.js"]` with `tini`/`--init` and handle SIGTERM (§5, `O03`).

3. "Each deploy produces a burst of `Can't reach database` / 'pool is closed' errors from the old
   pods in their last second. The Prisma client is disconnected in `onModuleDestroy`. What is the
   ordering problem?"
   > **Direction:** The pool is closed while requests are still in flight; stop accepting and drain HTTP and consumers first, then close the database in `onApplicationShutdown` or after the server has closed (§5, §3, `F19`).

4. "We added `onApplicationShutdown` logic to flush buffered metrics and audit events in our Nest
   service. The events from the last few seconds before every deploy are always missing, and a log
   line we put in the hook never appears — locally or in the cluster. Why?"
   > **Direction:** Nest calls lifecycle shutdown hooks only after `app.enableShutdownHooks()`; without it the process simply exits on the signal — enable it, and confirm the process really receives SIGTERM (PID 1) (§5, §3, `F19`).

5. "After our graceful shutdown work, old pods still receive requests for the full grace period on
   existing connections, and those requests are killed by SIGKILL. `server.close()` is called
   immediately on SIGTERM. What is happening?"
   > **Direction:** `server.close()` stops accepting new connections but busy keep-alive connections stay open and the LB keeps using them; send `Connection: close` on responses once draining, call `closeIdleConnections()`, and bound the drain time (§5, §1).

6. "When our Postgres primary failed over — a 20-second blip — every pod of every service restarted,
   and the recovery took 15 minutes instead of 20 seconds. Our liveness probe calls `/health`, which
   runs `SELECT 1`. Explain."
   > **Direction:** Liveness must not depend on shared dependencies; a database blip failed liveness everywhere, restarting all pods and causing a cold-start stampede — liveness checks only the process, readiness may check dependencies (§5, §8, `O06`).

7. "A Spring Boot service takes 70 seconds to start in the cluster. After an upgrade that added a
   few beans, pods never become ready: they get killed and restarted in a loop. What is going on?"
   > **Direction:** The liveness probe starts failing before startup completes, so the kubelet kills the pod; add a `startupProbe` with enough budget, and reduce startup time (lazy init, fewer eager connections) (§5, `F22`).

8. "Every deploy at peak causes a five-minute p99 spike to two seconds, even though no request
   fails. The service is Spring Boot, rolled out with `maxUnavailable: 25%`. Why, and what can you
   do?"
   > **Direction:** New pods are cold (JIT, class loading, empty caches and pools) and capacity drops by a quarter during rollout; warm up before reporting ready, use `maxSurge` instead of `maxUnavailable`, and deploy outside peak (§5, `F24`).

9. "Our Node service uses the `cluster` module with one worker per `os.cpus().length`. In
   Kubernetes with a 1-CPU limit on 64-core nodes, pods OOM-kill at startup. Why?"
   > **Direction:** `os.cpus()` reports the host's cores, not the cgroup quota, so it forks 64 workers; derive the count from the quota or configuration — and usually run one process per pod and scale pods (§6, `O03`).

10. "Our API's p99 jumps in 100 ms steps, although average CPU is 40% of the limit. Profiling shows
    nothing slow in the code. The pod has a CPU limit of 1. What is happening?"
    > **Direction:** CFS quota throttling — bursts of work across several threads use the 100 ms period's budget early and the container is frozen for the rest; check `cfs_throttled_periods`, raise or remove the limit, and reduce parallel threads (§6, `O03`).

11. "A Nest service's pods restart with exit code 137 and `OOMKilled`, but there is no JavaScript
    stack trace or heap error in the logs. The container limit is 1 GiB and no Node flags are set.
    What is going on?"
    > **Direction:** The kernel killed the process because V8's heap limit (plus off-heap memory) exceeded the container limit; set `--max-old-space-size` to ~75% of the limit so V8 GCs harder or fails with a diagnosable heap error, and investigate growth (§7, `C03`).

12. "To 'fix' Java heap OOMs, a teammate set `-Xmx2g` in a 2 GiB container. Now pods are
    `OOMKilled` by Kubernetes instead. Why is this worse, and what is the right sizing?"
    > **Direction:** Metaspace, thread stacks, direct buffers and the code cache live outside the heap, so heap = limit guarantees a kernel kill without a heap dump; use `MaxRAMPercentage` around 70–75% and measure with Native Memory Tracking (§7, `C06`).

13. "A small Spring service with a 1-CPU, 1 GiB limit has occasional 800 ms pauses. GC logs show
    `Pause Full (Allocation Failure)` with the Serial collector. Nobody chose Serial GC. Why is it in
    use?"
    > **Direction:** The JVM picks SerialGC automatically for fewer than 2 CPUs or under 1792 MB; choose the collector explicitly (G1) and size CPU/memory with it in mind (§7, §6).

14. "Our gunicorn config sets `workers = multiprocessing.cpu_count() * 2 + 1`. After moving to
    Kubernetes, each pod runs 129 workers, uses 6 GB of RAM and opens 129 database connections. Why?"
    > **Direction:** `cpu_count()` reports host CPUs, not the cgroup limit; set worker count explicitly from the pod's CPU allocation (§6, §8, `C07`).

15. "Each deploy of our chat service drops 50 000 WebSocket connections, and they all reconnect in
    the same second, overwhelming auth and the database. What can you do on both sides?"
    > **Direction:** A deploy is a thundering herd; clients reconnect with exponential backoff and jitter, and servers drain connections gradually (close in batches during a longer grace period) rather than all at once (§13, §5, `F16`).

16. "Our deploys on AWS take over five minutes per batch, and old pods keep getting requests long
    after they stopped being ready. The ALB target group has default settings. What is slowing it
    down?"
    > **Direction:** The target group's `deregistration_delay` defaults to 300 s; set it to match the real drain time and align it with the preStop sleep and the pod's grace period (§5).

17. "We added a 20-second `preStop` sleep and the app needs 20 seconds to drain. Now shutdown
    reliably ends in SIGKILL with jobs cut off. The grace period is the default. Why?"
    > **Direction:** `terminationGracePeriodSeconds` (30 s) includes the preStop hook, leaving 10 s for draining; set the grace period to preStop + drain + margin (§5).

18. "Our gunicorn workers restart after 1 000 requests to contain a slow leak. Every so often,
    latency spikes for a few seconds across the pod. Why, and how do you smooth it?"
    > **Direction:** Without `--max-requests-jitter` all workers hit the limit together and restart at once; add jitter so recycling is staggered — and still find the leak (§7, §5).

19. "After upgrading to Spring Boot 3.4, our deploys became slower and some batch pods stay in
    Terminating for 30 seconds. Nothing in our code changed. What did the upgrade change?"
    > **Direction:** Boot 3.4 enabled graceful shutdown by default, so the server now waits up to `timeout-per-shutdown-phase` (30 s) for in-flight requests; tune the phase timeout and ensure long-lived requests or consumers stop promptly (§5, §15).

20. "Traffic peaks at 09:00. The HPA scales from 6 to 20 pods, but new pods take 90 seconds to become
    useful and users see errors in the first minutes. What would you change?"
    > **Direction:** Reactive scaling lags a cold start; schedule scale-up before a known peak, keep headroom, reduce startup time and warm pools and caches before readiness — and check the added pods do not overload the database (§5, §8, `F24`).

### Level 7 — Memory leaks and the event loop

Slow leaks and stalls that only show up after hours of production traffic; the interviewer wants
evidence-gathering, not guesses.

1. "Our Nest service logs `MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
   11 close listeners added`, and memory grows by about 200 MB a day. Where would you look, and how
   would you confirm it?"
   > **Direction:** Something adds a listener per request to a long-lived emitter (a shared socket, client or `process`) without removing it; run with `--trace-warnings` to get the stack, then confirm with heap snapshot diffs showing growing listener arrays and closures (§7, §16).

2. "Memory on our metrics-heavy Nest service grows steadily, and `/metrics` responses have grown to
   40 MB. We use `prom-client` with a histogram labelled by `req.url`. What is the leak?"
   > **Direction:** Raw URLs with IDs and query strings create unbounded label cardinality, each series held forever; label with the route template, bound label values, and alert on series count (§7, `O13`).

3. "A Nest image-processing service's RSS climbs to the container limit, but `process.memoryUsage()
   .heapUsed` stays flat at 150 MB. Heap snapshots show nothing growing. What kind of memory is it,
   and what do you try?"
   > **Direction:** Off-heap memory — `Buffer`s, native libraries such as libvips, unclosed streams, or glibc arena fragmentation; check `external`/`arrayBuffers`, close streams, limit native concurrency, and try `MALLOC_ARENA_MAX=2` or jemalloc (§7).

4. "An engineer took a heap snapshot on a production Node pod to debug a leak. The pod froze for 40
   seconds, was killed by liveness and the snapshot was lost. How do you do this safely?"
   > **Direction:** A snapshot pauses the process and needs extra memory; take the pod out of rotation first (fail readiness), relax liveness, ensure memory headroom, then capture with `--heapsnapshot-signal` and take three snapshots to compare (§7, §16).

5. "A Nest service keeps a `Map` of currency rates keyed by `${from}-${to}-${date}` in a singleton to
   avoid API calls. Memory grows until OOM every four days. The cache seemed harmless. What is
   wrong?"
   > **Direction:** An unbounded in-process cache keyed on an unbounded space is a leak; use an LRU with a maximum size and TTL (or an external cache), and remember each pod holds its own copy (§7, §13).

6. "When many users log in at once, our Node service's calls to an internal API suddenly take two
   seconds, although the API itself answers in 20 ms. Login hashes passwords with the async `bcrypt`
   library. How are these connected?"
   > **Direction:** Async bcrypt and `dns.lookup` share the libuv thread pool (4 threads by default), so DNS lookups for outbound calls queue behind hashes; raise `UV_THREADPOOL_SIZE` at startup, cache DNS, or move hashing off the process (§6, `C03`).

7. "A single request to our Nest signup endpoint with a crafted 50 KB email address pinned the CPU
   of a pod for 30 seconds and timed out every other request on it. The email is validated with a
   custom regex. What is this, and how do you defend against it?"
   > **Direction:** ReDoS — catastrophic backtracking in a regex running on the event loop; use linear-time validators (or RE2), cap input length before validation, and watch event-loop lag (§6, §14, `F26`).

8. "A Nest batch endpoint processes 100 000 in-memory items with `for (const x of items) await
   transform(x)`, where `transform` is synchronous code wrapped in `async`. During it, health checks
   fail even though every step `await`s. Why does awaiting not yield?"
   > **Direction:** Awaiting already-resolved promises only runs microtasks, which never let the event loop reach I/O; yield with `setImmediate` every N items, or move the work to a worker thread or job (§6).

9. "Our API latency spiked every time the logging pipeline (Fluent Bit) fell behind, although the
   service itself was not busy. The service logs with `console.log(JSON.stringify(...))`. What is
   the connection?"
   > **Direction:** In Node, stdout to a pipe or file is synchronous on Linux, so a slow log reader blocks the event loop; use an async logger (pino with async destination or transport), log less, and never log whole bodies (§6, `F08`).

10. "A Nest endpoint returning 5 000 products takes 900 ms, of which the database query is 60 ms.
    A CPU profile shows most of the time in `class-transformer`. The global
    `ClassSerializerInterceptor` is on. What would you do?"
    > **Direction:** Reflection-based serialisation of thousands of class instances is CPU on the event loop; return plain objects shaped by the query (or schema-compiled serialisation on Fastify), paginate, and keep the interceptor off hot list endpoints (§6, §12, `F24`).

11. "After generating one large report, a gunicorn worker stays at 1.2 GB of RSS for the rest of its
    life, although the report objects are gone. Is this a leak, and what do you do?"
    > **Direction:** Not a classic leak — CPython and glibc rarely return freed memory to the OS after a peak; avoid the peak (stream, chunk, move to a job) and recycle workers with `--max-requests` plus jitter (§7, `C07`).

12. "We moved a Django API to ASGI with async views for performance. Throughput went *down*. The
    views `await sync_to_async(...)` around ORM calls. Why is it slower?"
    > **Direction:** `sync_to_async(thread_sensitive=True)`, the default, runs sync ORM calls on one shared thread, serialising them; stay on WSGI for ORM-heavy views (Django's async ORM methods still wrap the sync ORM), and use async views only where they await genuinely async I/O (§4, §6, `F21`).

13. "A Spring WebFlux service freezes completely for seconds at a time; thread dumps show all
    `reactor-http-nio` threads inside a JDBC call made by a newly added library. What happened?"
    > **Direction:** A blocking call on the Netty event-loop threads stalls every connection served by them; move blocking work to `Schedulers.boundedElastic()`, or use a reactive driver, and run BlockHound in tests (§6, `C06`, `F22`).

14. "A heap snapshot comparison shows hundreds of thousands of retained `IncomingMessage` objects,
    each retained by a `Timeout`. Our code wraps every downstream call in a `withTimeout()` helper
    based on `Promise.race` with a 60-second timer. Explain."
    > **Direction:** The timer is never cleared when the work finishes first, so every request's closure lives for 60 s — at high RPS that is a massive "leak" that drains slowly; clear the timer in `finally` or use `AbortSignal.timeout` (§7, §16).

15. "A Node service crashes with `FATAL ERROR: Reached heap limit Allocation failed - JavaScript
    heap out of memory` at about 2 GB, while its container has an 8 GiB limit and plenty of free
    memory. Is this a leak?"
    > **Direction:** Not necessarily — V8's heap limit is separate from the container limit; set `--max-old-space-size` deliberately (about 75% of the limit), then check whether usage plateaus or keeps growing (§7).

16. "Heap snapshots of our Nest service show request contexts from hours ago retained through
    `AsyncLocalStorage` stores. Nothing in our code references them. How can ALS keep them alive?"
    > **Direction:** Any long-lived async resource created during a request (a never-resolving promise, a pooled resource, `enterWith` on a shared continuation) keeps the store it captured alive; find the retainer path, and use `run()` rather than `enterWith()` (§4, §7).

17. "A Spring Boot service's container memory keeps growing while heap usage after GC is flat. It
    uses Netty-based HTTP clients heavily. How do you find where the memory goes?"
    > **Direction:** Non-heap growth — direct buffers, threads, metaspace or native allocations; enable `-XX:NativeMemoryTracking=summary` and compare `jcmd VM.native_memory` over time, and check buffer and client lifecycle (§7, §16).

18. "We enabled gunicorn `--preload` to share memory across 8 workers, but total RSS barely dropped
    after a few hours. Why doesn't copy-on-write help much in Python?"
    > **Direction:** Reference counting (and GC) write to object headers, so shared pages are copied as soon as objects are touched; call `gc.freeze()` after preload, and make sure nothing that must be per-process is opened before the fork (§7).

19. "Our Nest API degrades badly under overload: latency climbs to 30 seconds and everything times
    out, instead of some requests failing fast. What would you add at the framework level?"
    > **Direction:** Load shedding driven by event-loop lag and in-flight counts (for example `@fastify/under-pressure`) returns fast 503s when the process is saturated, keeping the rest of the traffic healthy (§6, `F24`).

20. "We moved a Spring Boot service to JDK 21 virtual threads. Under load it stalls intermittently,
    and thread dumps show virtual threads pinned while holding monitors in an older JDBC driver.
    What is going on?"
    > **Direction:** Before JDK 24, blocking inside `synchronized` pins the carrier thread, so a few blocked calls exhaust the small carrier pool; upgrade the JDK or driver, replace `synchronized` with `ReentrantLock`, and remember the connection pool still bounds concurrency (§4, §8, `C06`).

### Level 8 — Context propagation, observability, caching and realtime

The cross-cutting machinery — context, caches, connections that stay open — where one wrong
assumption leaks data or hides everything.

1. "Under load, our logs show the wrong trace ID on some lines: a line from request B carries
   request A's ID, always around Redis calls. We use `nestjs-cls`. How is that possible?"
   > **Direction:** The Redis client queues commands and runs callbacks in the context of whoever triggered the flush, so ALS context is inherited wrongly; bind callbacks with `AsyncResource.bind` or upgrade the driver, and prove the fix with a concurrent invariant test (§4, §16).

2. "We added OpenTelemetry to our Nest app in `main.ts`. HTTP spans appear, but there are no
   Postgres or Redis spans, and traces from downstream services are disconnected. Why?"
   > **Direction:** Auto-instrumentation patches modules at `require` time, so the SDK must load before `pg`, `ioredis` and `http` are imported — start it with `--require`/`--import` of a tracing file (§4, `O13`).

3. "Our Nest app emits a domain event with `EventEmitter2`, and a listener writes an audit log. The
   audit records the wrong user — the user of a different, concurrent request — about 1% of the
   time. What would you look at?"
   > **Direction:** Listeners run in the context of the `emit()`, and anything that defers or batches emission (or a singleton caching the user) mixes contexts; pass the actor in the event payload explicitly rather than reading ambient context in listeners (§4, §3).

4. "In our Spring service, log lines from `@Async` methods have no `traceId` or `userId` in the MDC,
   so they cannot be correlated. Why, and how do you fix it everywhere at once?"
   > **Direction:** MDC is `ThreadLocal` and does not cross to executor threads; register a `TaskDecorator` (or Micrometer context propagation) on the executors that copies and clears MDC (§4, `C06`, `F08`).

5. "A Spring method uses `CompletableFuture.supplyAsync()` to call two services in parallel. The
   calls fail with 401 because the security context is empty on those threads. How do you fix it
   without passing tokens around manually?"
   > **Direction:** `SecurityContextHolder` is thread-local; use a context-propagating executor (`DelegatingSecurityContextExecutor` or a `TaskDecorator`), and never run such work on the common `ForkJoinPool` (§4, `F11`).

6. "After adding `CacheInterceptor` to speed up `GET /me/profile` in our Nest API, a user reported
   seeing someone else's profile. How did caching cause a data leak?"
   > **Direction:** Nest's `CacheInterceptor` keys by URL, not by user, so the first user's response is served to everyone at that URL; override `trackBy()` to include the user or do not cache personalised responses at this layer (§13, `F15`, `S05`).

7. "After upgrading from Nest 9 to Nest 10, database load on the catalogue service tripled, although
   cache hit metrics are still reported and nothing else changed. The config says `ttl: 300`. What
   happened?"
   > **Direction:** `cache-manager` v5 takes TTL in milliseconds, so 300 means 0.3 s and the cache is effectively off; convert TTLs and treat default and unit changes as part of upgrades (§13, §15).

8. "Every hour on the hour our homepage latency spikes and the database CPU hits 100% for 30 seconds.
   The homepage aggregates are cached in Redis with a one-hour TTL. What is happening?"
   > **Direction:** Cache stampede — the key expires and every request recomputes it at once; coalesce with single-flight or a short lock, refresh early or serve stale-while-revalidate, and jitter TTLs (§13, `Q02`).

9. "In Spring, `@Cacheable("products")` on `findProduct(id)` caused a bug: products created after
   someone looked them up return 404 for an hour. Why?"
   > **Direction:** The cache stored the `null` result of the earlier miss; add `unless = "#result == null"` and evict or update on writes (§13, `F15`).

10. "An admin changes a feature price, and for the next five minutes customers see a mix of old and
    new prices depending on which request hits which pod. Prices are cached in a Caffeine cache.
    What is the design issue?"
    > **Direction:** In-process caches are per pod (and per worker), so invalidation on one does not reach others; use a shared cache, broadcast invalidations, or accept and bound staleness with short TTLs (§13, `Q02`).

11. "Our public API is limited to 100 requests per minute per key with `@nestjs/throttler`, but a
    partner is clearly making 700 per minute without being throttled. We run 8 pods. Why?"
    > **Direction:** The default in-memory storage counts per pod, so the real limit is multiplied by the replica count; use shared storage (Redis) for the throttler or enforce limits at the gateway (§13, `F26`).

12. "Our socket.io chat worked on one pod. After scaling to three, clients fail to connect with
    `Session ID unknown` and fall into reconnect loops. What is going on?"
    > **Direction:** The long-polling handshake and subsequent polls land on different pods; enable sticky sessions at the load balancer (or force the WebSocket transport) (§13, `F16`).

13. "After scaling our Nest WebSocket gateway to multiple pods, users only receive some chat
    messages — roughly one in three. Connections are stable. What is missing?"
    > **Direction:** An emit only reaches sockets on the local pod; use the Redis (or similar) adapter or a pub/sub fan-out so broadcasts reach every pod (§13, `F16`, `A16`).

14. "Our raw `ws` WebSocket connections behind an AWS ALB drop at exactly 60 seconds of inactivity,
    and on GCP they drop at exactly 30 seconds even while messages flow. Explain both."
    > **Direction:** On ALB the 60 s idle timeout closes quiet connections, fixed with application pings; on GCP's external HTTP(S) LB the backend service timeout (30 s default) caps WebSocket lifetime regardless of traffic, fixed by raising that timeout (§13, §1, `F16`).

15. "Our SSE endpoint for live order updates works locally, but in production events arrive in
    bursts every minute or so, or only when the stream closes. What is buffering them?"
    > **Direction:** Compression middleware or proxy buffering holds the stream; disable compression for `text/event-stream`, set `X-Accel-Buffering: no` / `proxy_buffering off`, and send periodic comments as heartbeats (§13, `F16`).

16. "After a deploy of our realtime service, the auth service fell over: 80 000 clients reconnected
    within two seconds and each validated its token. What do you change so deploys are boring?"
    > **Direction:** A reconnect storm is a thundering herd; clients back off with jitter, servers drain gradually, and reconnect validation is made cheap (short-lived signed tokens verified locally, not a network call per reconnect) (§13, §5).

17. "Our WebSocket service stops accepting new connections at almost exactly 1 000 per pod with
    `EMFILE`, although CPU and memory are low. What limit are we hitting?"
    > **Direction:** The per-process file descriptor limit (`ulimit -n`, often 1024); raise it in the container, and count connections per pod in capacity planning (§13).

18. "You suspect our Nest app leaks tenant context between concurrent requests, but you cannot
    reproduce it by hand. How would you prove it — and later guard against regressions?"
    > **Direction:** Build an invariant load test: fire many concurrent requests with unique markers and assert every response and log line carries its own marker; keep it in CI (§4, §16, `F12`).

19. "A Django project running under ASGI stores the current tenant in a `threading.local` set by
    middleware. Since the move to ASGI, some queries use the wrong tenant. Why did ASGI break it?"
    > **Direction:** Under asyncio many requests share a thread, so `threading.local` leaks between them; use `contextvars` (which asgiref propagates through `sync_to_async`), or pass the tenant explicitly (§4, `F21`).

20. "Two Spring methods, `getUserSettings(Long id)` and `getUserLimits(Long id)`, both use
    `@Cacheable("user")`. Occasionally the limits screen shows settings data and throws a
    `ClassCastException`. Why?"
    > **Direction:** The default key is the method arguments, so both methods write the same key in the same cache; use separate cache names or keys that include the method (§13, `F15`).

### Level 9 — Security, configuration and upgrades that bite

Less frequent, higher stakes: framework defaults, proxies and upgrades that quietly change what the
application trusts.

1. "A pentest found that any user can make themselves an admin by adding `"role": "admin"` to the
   body of `PATCH /users/me`. The DTO has no `role` field and the global `ValidationPipe` is on.
   How did it get through?"
   > **Direction:** Without `whitelist: true` unknown properties pass through, and the service spreads the body into the entity — mass assignment; enable `whitelist` and `forbidNonWhitelisted`, and map DTO fields explicitly (§12, `F06`, `F26`).

2. "Our `CreateOrderDto` has an `address: AddressDto` field with `@ValidateNested()`. Invalid
   addresses — missing postcode, a 5 MB street name — are saved without complaint. What is missing?"
   > **Direction:** Without `@Type(() => AddressDto)` class-transformer never instantiates the nested class, so its validators never run; add `@Type` and length limits, and test that invalid nested input is rejected (§12, `F06`).

3. "A bulk endpoint takes `@Body() items: CreateItemDto[]`. Validation works for single items
   elsewhere, but here anything is accepted, including objects with no fields at all. Why?"
   > **Direction:** TypeScript erases the array's element type, so `ValidationPipe` cannot validate elements; wrap the array in a DTO with `@ValidateNested({ each: true })` and `@Type`, or use `ParseArrayPipe` (§12, `F06`).

4. "We turned on `SECURE_SSL_REDIRECT` in Django behind our load balancer, which terminates TLS. Now
   every page is an infinite redirect loop. What is missing — and what is the security trap in the
   fix?"
   > **Direction:** Django sees plain HTTP from the LB and keeps redirecting; set `SECURE_PROXY_SSL_HEADER`, but only if the proxy always overwrites `X-Forwarded-Proto`, or clients can spoof HTTPS (§14, `F21`).

5. "Our Spring Boot app behind an ingress generates OAuth redirect URIs and `Location` headers with
   `http://` and the internal hostname, so logins fail. How do you fix it properly?"
   > **Direction:** The app does not trust forwarded headers by default; set `server.forward-headers-strategy` (native or framework) and make sure only the trusted proxy can set those headers (§14, `F22`, `F11`).

6. "After upgrading from Django 3.2 to 4.2, every form POST from our admin domain fails with 403
   CSRF verification failed, although the tokens are present. What changed?"
   > **Direction:** Django 4.0 checks the `Origin` header against `CSRF_TRUSTED_ORIGINS`, whose entries must now include the scheme (`https://admin.example.com`); fix the setting, and read upgrade notes for default changes (§14, §15).

7. "After a Django deploy, the load balancer marked every pod unhealthy and the site went down.
   Health checks return 400. Nothing changed in the health view. What would you check?"
   > **Direction:** The health check sends the pod IP as the `Host` header, which `ALLOWED_HOSTS` rejects; exempt the health path or allow the internal host deliberately, rather than setting `ALLOWED_HOSTS = ['*']` (§14, `F21`).

8. "A security researcher downloaded a 400 MB file from `https://api.example.com/actuator/heapdump`
   and found database passwords and JWT signing keys in it. How did this happen, and what is the
   right setup?"
   > **Direction:** Actuator endpoints were exposed on the public port; expose only health and metrics, run management on a separate internal port, and treat heap and thread dumps as secrets (§14, §7, `F22`).

9. "A request with a body containing `{"__proto__": {"isAdmin": true}}` made every subsequent user
   an admin on that pod until it restarted. Our Nest service deep-merges the body into a settings
   object with a utility function. Explain."
   > **Direction:** Prototype pollution — the merge wrote to `Object.prototype`, affecting every object in the process; use safe merges that reject `__proto__`/`constructor`, whitelist DTO fields, and prefer `Object.create(null)` maps (§14, §12, `S06`).

10. "We set `FEATURE_NEW_CHECKOUT=false` in production to disable a feature, and it was enabled for
    everyone. The code reads `if (process.env.FEATURE_NEW_CHECKOUT)`. What is the wider lesson?"
    > **Direction:** Environment variables are strings, and `"false"` is truthy; parse and validate all config with a schema at startup and fail fast on invalid values (§15, `F05`).

11. "We changed Hikari's `maximum-pool-size` in `application.yml` from 10 to 30 and redeployed, but
    metrics show the pool still at 10. The file in the image is correct. Where else would you look?"
    > **Direction:** A higher-precedence property source — an environment variable like `SPRING_DATASOURCE_HIKARI_MAXIMUMPOOLSIZE` via relaxed binding, a profile file or a command-line arg — overrides it; check resolved values via `/actuator/env` or startup logging (§15, `F05`).

12. "A Nest service is noticeably slower in production than in staging on identical hardware, and
    its logs are pretty-printed and colourised. What single setting would you check first?"
    > **Direction:** `NODE_ENV` is not `production`, so libraries run in development mode (no view caching, synchronous pretty logging, extra checks); set it explicitly in the image and assert it at startup (§15, §6).

13. "A client's security team found our Swagger UI and a Django debug error page, with settings and
    stack traces, on the public production domain. Which framework defaults and practices led here?"
    > **Direction:** Debug and docs features are on by default in many setups; gate Swagger by environment or auth, ensure `DEBUG=False` via validated config, and scan production for debug endpoints in CI (§14, §15, `F25`).

14. "Our IP allowlist for the admin API and our rate limiter both use `X-Forwarded-For`. A tester
    bypassed both by sending the header themselves. How should client IP be determined?"
    > **Direction:** Take the address added by your own trusted proxies — trust exactly the number of hops you control (`trust proxy` with a hop count or subnets) — and never the leftmost client-supplied value (§13, §14, `F26`).

15. "A code review found `` prisma.$queryRawUnsafe(`SELECT * FROM orders WHERE status = '${status}'`) ``
    in a reporting endpoint. The author says the status comes from a dropdown. What do you say?"
    > **Direction:** It is SQL injection regardless of the UI; use the tagged-template `$queryRaw` (parameterised) or `Prisma.sql`, and validate the value against an enum (§10, §12, `S06`).

16. "Attackers took our Nest upload service down by sending a few 2 GB multipart uploads in
    parallel. The pods OOM-killed each time. What made this so easy?"
    > **Direction:** `multer` memory storage buffers whole files in RAM; set file size limits at the ingress and in multer, stream to disk or object storage (or presigned direct uploads), and limit concurrent uploads (§14, `F17`).

17. "A customer received a password reset email whose link pointed to an attacker's domain. The
    email was sent by our own app. How is that possible?"
    > **Direction:** The reset link was built from the request's `Host` header, which the attacker controlled; build absolute URLs from configuration, and validate hosts (`ALLOWED_HOSTS`, trusted proxy headers) (§14, `S06`).

18. "Our Nest service reloads configuration from a remote store every minute. Over a week, Postgres
    connections from this service grow from 20 to 900. Nothing else changed. What is leaking?"
    > **Direction:** Each reload creates a new pool or client without closing the previous one; reuse clients and only rebuild — with explicit teardown — when connection settings actually change (§15, §8).

19. "A Celery setup accepts the `pickle` serialiser for 'compatibility'. Someone with write access to
    the Redis broker — which has no auth, inside the VPC — could do what?"
    > **Direction:** Unpickling a crafted message executes arbitrary code on every worker; use JSON serialisation only (`accept_content=['json']`), pass IDs not objects, and secure the broker (§11, `S06`).

20. "After upgrading a service to Spring Boot 3 / Hibernate 6, inserts started failing intermittently
    with duplicate primary key errors under concurrency. IDs use `@GeneratedValue` with defaults.
    What changed?"
    > **Direction:** Hibernate 6 expects a per-entity sequence with a pooled optimiser and `allocationSize` 50, but the database sequence still increments by 1, so instances hand out overlapping ID blocks; align the sequence increment with the allocation size and restart it above the max ID (§15, `F22`).

### Level 10 — Multi-layer failures and the rarest war stories

Incidents where two or three correct-looking behaviours combine into something no single document
describes; the interviewer wants to see you hold several layers in your head at once.

1. "A mobile client retries on 504. During a slow database period, some customers were charged twice
   and got two orders. The gateway times out at 30 s, our Nest service keeps working after that, and
   the order endpoint is a plain `POST`. Walk me through the chain and the fix."
   > **Direction:** The 504 did not cancel the work — the order completed after the gateway gave up, and the client's retry created a second one; require idempotency keys stored with a unique constraint, propagate deadlines so the server stops wasted work, and make retries safe by design (§1, §9, `M09`, `SD10`).

2. "Our multi-tenant GraphQL API on Nest uses DataLoader for batching and `nestjs-cls` for tenant
   context, with RLS set per query from that context. A customer saw another tenant's projects in a
   response. How could the batching layer cause a cross-tenant leak?"
   > **Direction:** A DataLoader (or its batch function) shared beyond one request runs the batch in the context of the first caller, mixing keys from several tenants under one tenant's context; create loaders per request, carry the tenant in the key, and enforce the tenant in the query itself (§4, §8, `S05`).

3. "Our BullMQ payment worker does CPU-heavy fraud scoring and then calls the payment provider.
   During a traffic spike, some customers were charged twice. The provider has no idempotency on
   our calls. Explain how the event loop, the queue and the provider combined."
   > **Direction:** Scoring blocked the loop past `lockDuration`, the job stalled and ran again on another worker, and the non-idempotent charge executed twice; send an idempotency key per job to the provider, move CPU work to a sandboxed processor, and record the charge before side effects are repeated (§11, §6, `M09`).

4. "A Nest service uses durable request-scoped providers keyed by tenant, introduced to fix a
   request-scope performance problem. Six months later, memory grows linearly with the number of
   distinct tenants seen since the last deploy, and we now have 40 000 tenants. Why?"
   > **Direction:** Durable subtrees are cached per context ID and never evicted, so every tenant ever seen keeps its own provider graph; move tenant data to ALS with singleton providers, or bound and evict the per-tenant state yourself (§3, §7, `F20`).

5. "A Spring service in a pod with a 2-CPU limit on 96-core nodes has 300 ms pauses several times a
   minute. GC pauses in the logs are short, but CPU throttling metrics are high. How do these
   connect?"
   > **Direction:** The JVM sized GC, JIT and ForkJoin threads from the CPU count it saw, and bursts of many threads exhaust the CFS quota and freeze the container until the next period; set `-XX:ActiveProcessorCount`, tune GC threads, or remove the CPU limit (§6, §7, `C06`).

6. "Our outbound calls to a healthy partner API time out during login peaks, the circuit breaker
   opens, and checkout fails. The partner confirms no slow requests. Login uses async `bcrypt` and
   the partner calls use hostnames. Connect the dots."
   > **Direction:** Hashing saturates the libuv pool, `dns.lookup` for the partner host queues behind it, the call times out before a socket even opens, and the breaker counts it as the partner's failure; enlarge the pool, cache DNS, isolate hashing, and measure connect/DNS time separately from response time (§6, §16, `M09`).

7. "Under load, a Spring order-history endpoint started deadlocking with the order-update job. The
   endpoint is a GET that loads orders and maps them to DTOs with a mapper; it has no explicit
   writes. Where would you look?"
   > **Direction:** The mapper mutates managed entities, Hibernate flushes `UPDATE`s at commit, and those row locks are taken in a different order from the job's; make reads `readOnly`/projections, stop mutating entities in mappers, and check SQL logs for unexpected updates (§9, §10, `DB09`).

8. "We use a Postgres advisory lock to elect a leader for our Nest cron jobs. After introducing
   PgBouncer in transaction mode, jobs sometimes run on two pods at once, and sometimes on none for
   hours. Why?"
   > **Direction:** Session advisory locks belong to a server connection that transaction pooling reassigns, so the lock is held by the wrong client or released unpredictably; use transaction-scoped locks inside one transaction, a direct (non-pooled) connection for leadership, or a lease row with a unique constraint (§8, §11).

9. "After tuning keep-alive (Node `keepAliveTimeout` 65 s behind a 60 s ALB) to fix 502s, our
   deploys started producing a different error: requests on old pods cut off at SIGKILL. The preStop
   sleep is 10 s and the grace period 30 s. What is the new interaction?"
   > **Direction:** Longer keep-alive means the ALB keeps sending on existing connections to a draining pod, and `server.close()` waits for them; during drain respond with `Connection: close`, close idle sockets, and align deregistration delay, preStop and grace period (§1, §5).

10. "Our Celery tasks are short — two seconds each — yet some are executed twice, several hours
    apart. We use Redis, `acks_late=True` and the default prefetch multiplier, and queues back up
    for hours during the nightly import. How can short tasks be redelivered?"
    > **Direction:** Prefetched messages are unacknowledged while they wait in a busy worker's buffer; if they wait longer than the 1-hour visibility timeout, Redis redelivers them elsewhere; use prefetch multiplier 1, a larger visibility timeout, separate queues for bulk work, and idempotent tasks (§11).

11. "Our Kafka consumer in the same Spring app as our REST API fails to deserialise events with dates
    and unknown fields, while the REST controllers handle the same JSON fine. Both use Jackson. Why
    does the same JSON behave differently?"
    > **Direction:** The consumer builds its own `new ObjectMapper()`, which lacks Boot's lenient `FAIL_ON_UNKNOWN_PROPERTIES` setting and JavaTime module; inject Boot's configured mapper everywhere (and never create one per message) (§12, §3, `A17`).

12. "An endpoint started leaking one database connection every time a downstream call returned 404.
    After a few days, the pool was exhausted. The code checks `res.status` from `fetch` and throws.
    The pool is Postgres — how is a `fetch` response related to it?"
    > **Direction:** The unconsumed `fetch` body kept the socket busy, and the handler threw from inside a transaction path whose error branch never released the `QueryRunner`, leaving an idle-in-transaction connection; consume or cancel bodies and release query runners in `finally` (§1, §9, §8).

13. "We run Postgres RLS where the tenant is set on each connection in a pool `connect` hook, using
    the tenant from AsyncLocalStorage. Under concurrency, queries occasionally run with another
    tenant's ID. Nothing is shared in our code. Explain."
    > **Direction:** The pool hook runs once per physical connection, in the context of whichever request caused the connection to open, and the connection is later reused by other tenants; set the tenant per transaction with `SET LOCAL` from the current request's context, never per connection (§4, §8, `S05`).

14. "When our database failed over for 30 seconds, every service's pods reconnected simultaneously,
    the new primary hit `max_connections`, and it took 20 minutes to recover. Liveness probes do
    not touch the database. What amplified the outage, and how do you design for recovery?"
    > **Direction:** Every pool refilled at once (minimum-idle settings, retries without jitter, request queues retrying), a connection storm against a cold primary; use jittered backoff on connect, small minimum-idle, a pooler in front, and load shedding while pools recover (§8, §5, `DB21`).

15. "Spring's `@Retryable` and `@Transactional` are both on `transferFunds()`. On a serialisation
    failure, the logs show three retry attempts, all failing immediately with `current transaction
    is aborted`. Explain the ordering problem."
    > **Direction:** The retry advisor sits inside the transactional one, so every attempt reuses the already-aborted transaction; move the retry to an outer bean or set advisor order so each attempt starts a fresh transaction (§9, §3, `F10`).

16. "Our Nest app's cold start in Lambda is 4 seconds, and under a burst the database gets
    thousands of connections. Someone proposes provisioned concurrency plus a larger Prisma pool.
    What would you actually do?"
    > **Direction:** Each concurrent environment has its own client and pool, so a bigger pool multiplies the problem; use `connection_limit=1` with RDS Proxy or PgBouncer, reuse the client across invocations, and trim module initialisation — or reconsider Lambda for this workload (§8, §10, `F27`).

17. "An import endpoint in our Django app (gunicorn sync workers behind nginx, where someone set
    `proxy_request_buffering off` to speed up another endpoint) returns 502s only for uploads over
    about 100 MB, only from slow client networks, and gunicorn logs show `WORKER TIMEOUT`. Faster
    clients uploading the same file succeed. What is happening?"
    > **Direction:** With request buffering off, a slow upload trickles straight into a sync worker, which blocks reading the body past gunicorn's 30 s timeout and is killed, so nginx reports 502; buffer uploads at the proxy (or upload directly to object storage) so sync workers only see complete requests (§1, §6, §14, `F17`).

18. "Our Nest service calls a partner API through a keep-alive agent with a 120-second idle
    timeout. We see sporadic `ECONNRESET`s, but only on the first call after a pause of between one
    and two minutes — never after shorter pauses, never under steady load. The partner says their
    nginx keeps connections for 75 s. Which hop is resetting, and how do you prove it?"
    > **Direction:** The pause window points at an idle timeout of about 60 s on the path — typically an ALB or proxy in front of the partner's nginx — not at nginx itself; confirm with `tcpdump` who sends the RST, then set our idle timeout below the smallest one on the path and retry idempotent calls once (§1, §16).

19. "A Spring `OrderService.create()` is `@Transactional` and, at the end, calls an `@Async`
    `notificationService.sendConfirmation(orderId)`. About 5% of confirmations fail with 'order not
    found', more under load. Both beans are correct in isolation. Explain."
    > **Direction:** The `@Async` method runs on another thread with its own transaction and can start before the caller's transaction commits — the in-process version of enqueue-before-commit; publish an event handled by `@TransactionalEventListener(phase = AFTER_COMMIT)` (plus `@Async`), or use an outbox (§4, §9, §11, `F10`).

20. "Three things happened in one week: after a Nest 11 upgrade, our tenant-audit middleware
    silently stopped matching; the new cache TTLs were in the wrong unit; and the Node upgrade added
    a request timeout that broke long uploads. As the senior engineer, what process change do you
    propose, beyond fixing each bug?"
    > **Direction:** Treat upgrades as changes to *defaults*, not just APIs: read migration notes for default changes, add tests that assert security middleware runs on each route, assert cache TTL units and timeouts in config tests, canary upgrades with metrics on 4xx/5xx and latency, and make failures loud (§15, §2, §16).

