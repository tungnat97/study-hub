[← back to the field index](README.md)

# Frameworks · Part 2 — Persistence, Transactions, Jobs & Realtime

Nodes `F09`–`F17`.

---

## F09 · The persistence layer

`Intermediate` · Requires: `F04`, `DB02` · Unlocks: `F10`, `F13`, `DB40`

### Preface

Every framework needs a way to get data in and out of the database, own the connection pool, and
manage schema changes.

The choice is between a full ORM (objects, change tracking, relations), a lightweight query builder
(typed SQL), and raw SQL. Most real systems use two of the three: an ORM for writes and object
graphs, and SQL for reads that matter.

### Details

#### 1. The options in the Node ecosystem

**Theory.** They differ in how much they hide.

**Example.**
- **TypeORM** — a full ORM with both Active Record and Data Mapper styles, decorator-based entities,
  lazy relations, and a migration system. Flexible, mature, and has a reputation for surprising
  edge cases in complex queries.
- **Prisma** — schema-first: you write a `schema.prisma`, generate a fully typed client. Excellent
  type safety and migrations, **no lazy loading** (you declare `include`/`select`, which prevents
  accidental N+1), and less suited to very complex SQL, where you drop to `$queryRaw`.
- **Drizzle / Kysely** — SQL-first query builders with full TypeScript types. You write what is
  essentially SQL and get compile-time checking. Increasingly the preferred choice for teams who
  know SQL and want no hidden behaviour.

**Advanced.** The setting that must never be true in production is TypeORM's `synchronize: true`,
which alters your schema to match the entities on startup. It will drop columns. Use generated
migrations, review them (`DB40`), and run them as a deliberate pipeline step (`O09`).

#### 2. The repository boundary

**Theory.** A repository exposes domain-meaningful operations (`findActiveByCustomer`) rather than
leaking query-builder objects across the codebase. It keeps persistence concerns in one layer and
makes the data access testable and reviewable.

**Example.** The useful form is a thin one:

```ts
@Injectable()
export class OrderRepository {
  constructor(private readonly db: DataSource) {}

  findOpenForCustomer(customerId: string, limit: number): Promise<Order[]> { /* ... */ }
  save(order: Order): Promise<void> { /* ... */ }
}
```

Services depend on this, not on the ORM. Swapping a slow ORM query for hand-written SQL then happens
in one file and changes nothing else.

**Advanced.** The common objection — "the ORM's repository is already a repository" — is fair for
simple CRUD. The value appears when queries get complex, when you want them all in one reviewable
place, and when you want to test services without a database. Do not build a repository per table out
of habit; build one per aggregate (`M02`), which is the boundary that actually means something.

#### 3. Migrations

**Theory.** Schema changes belong in version control, applied in order, and reviewable. Every
framework has a migration system; the important part is not the tool but the discipline of reading
what it generates (`DB22`).

**Example.**
- **Django** — `makemigrations` generates from model diffs; the migration is Python, so `RunPython`
  lets you write data migrations. Set `atomic = False` when you need `CREATE INDEX CONCURRENTLY`.
- **TypeORM/Prisma** — generate SQL from the entity or schema diff; edit it before running.
- **Spring** — Flyway or Liquibase, usually hand-written SQL, versioned files applied in order.

**Advanced.** Two rules. Generated migrations are *correct* but not necessarily *safe* — a generated
`ALTER COLUMN TYPE` will lock a large table (`DB22`); always read before merging. And migrations
should run as a **separate pipeline step**, not at application startup, because concurrent pods race
each other, a long migration delays the readiness probe until the pod is killed, and a failure
becomes a crash loop (`O09`).

#### 4. Connection pooling

**Theory.** The framework owns a pool of connections shared by all requests in that process. Size it
deliberately (`DB21`), and remember that the total across all pods is what the database sees.

**Example.** In Nest with TypeORM, the pool is configured in the `DataSource` options
(`extra: { max: 10 }`). In Prisma it is the `connection_limit` parameter in the database URL. In
Django it is `CONN_MAX_AGE` — and note that Django historically opened a connection **per request**
unless this is set, which is a common and expensive default. Spring uses HikariCP with
`maximumPoolSize`.

**Advanced.** The pool must be closed on shutdown, after in-flight work finishes, or you leave
connections hanging until the database times them out — which, during a rolling deploy of many pods,
can exhaust `max_connections` (`F28`). Nest's `enableShutdownHooks()` plus `onModuleDestroy` is where
that belongs.

### Interview questions

- "Repository pattern with an ORM that already is a repository — worth it?"
- "TypeORM versus Prisma, honestly."
- "Why is `synchronize: true` dangerous?"
- "Where do migrations run in your deployment, and why not at startup?"

---

## F10 · Transactions in the framework

`Advanced` · Requires: `F09`, `DB06` · Unlocks: `F13`, `M15`

### Preface

The database gives you transactions; the framework decides where they begin and end, and how the
transaction handle reaches the code that needs it.

The rule: **one transaction per use case**, opened in the application/service layer, containing only
database work — never a network call (`DB06`).

This node contains the single most-asked framework trap: Spring's `@Transactional` silently doing
nothing.

### Details

#### 1. Where the boundary goes

**Theory.** Not in the controller (it knows nothing about invariants) and not in the repository
(each method would be its own transaction, so two writes could not be atomic). It belongs in the
service method that represents one business operation.

**Example.** "Place order" writes the order, the order lines, and an outbox row (`M15`) — one
transaction. If it also charges a card, that call happens **outside** the transaction, with the
result recorded in a second transaction.

**Advanced.** A useful test for whether your boundary is right: if the transaction were to roll back,
would the resulting state be sensible? If rolling back would leave a charged card with no order, the
boundary is wrong — and the fix is usually a saga (`M14`) plus an outbox, not a bigger transaction.

#### 2. Spring's `@Transactional` and the proxy trap

**Theory.** Spring implements `@Transactional` with a **proxy**: the bean you inject is a wrapper
that opens a transaction, calls the real method, then commits. Anything bypassing the proxy bypasses
the transaction.

**Example.** The three classic failures, all silent:

```java
@Service
public class OrderService {
    @Transactional
    public void create(Order o) { ... }

    public void bulk(List<Order> os) {
        for (Order o : os) this.create(o);   // self-invocation: NO transaction
    }

    @Transactional
    private void helper() { ... }            // private: proxy cannot intercept, NO transaction
}
```

Plus: by default Spring rolls back only on **unchecked** exceptions, so a checked exception commits
the transaction unless you set `rollbackFor`.

**Advanced.** The fixes: call through an injected reference to the proxy (self-injection, or
`AopContext.currentProxy()`), move the annotated method to another bean, or use
`TransactionTemplate` programmatically. The same proxy mechanism explains why `@Async`, `@Cacheable`
and `@PreAuthorize` also silently do nothing on self-invocation — it is one fact that answers four
interview questions (`F22`).

#### 3. Django and Nest

**Theory.** Django wraps with `transaction.atomic()` (a decorator or a context manager); nested
blocks become savepoints. Nest has **no** built-in `@Transactional` — you either pass a transaction
manager explicitly or wire one through `AsyncLocalStorage`.

**Example.** Django's `on_commit` is the important feature to know:

```python
with transaction.atomic():
    order = Order.objects.create(...)
    transaction.on_commit(lambda: send_order_placed.delay(order.id))
```

The task is only enqueued **after** the transaction commits. Without `on_commit`, the worker can pick
up the job and query for an order that is not yet visible — the classic "job cannot find the row"
bug.

**Advanced.** In Nest, the explicit approach is passing an `EntityManager` or a Prisma transaction
client down through the call chain, which is verbose but honest. The implicit approach uses
`nestjs-cls` with a transactional plugin to store the transaction in request context so repositories
pick it up automatically — closer to Spring's ergonomics, with the same risk of it being unclear
which code participates. Say which you prefer and why; both are defensible.

#### 4. Propagation

**Theory.** What happens when a transactional method calls another one. `REQUIRED` (default) joins
the existing transaction. `REQUIRES_NEW` suspends it and opens a separate one, on a **second
connection**. `NESTED` uses a savepoint. `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER` cover the
remaining cases.

**Example.** The legitimate use of `REQUIRES_NEW` is an audit or failure record that must persist
even when the main transaction rolls back. The hazard: it takes a second connection from the pool
while the first is still held, so under load you can exhaust the pool — and it can deadlock against
its own parent if both touch the same rows.

**Advanced.** Understand that "one logical operation, one transaction" keeps this simple: if you find
yourself reaching for complex propagation, the operation is probably doing too much. And note that a
transaction spanning two datasources is a distributed transaction (`M13`) with all its problems —
the outbox pattern exists precisely to avoid it.

### Interview questions

- "Why does calling `this.doWork()` from another method in the same Spring bean skip the
  transaction?"
- "You enqueue a job inside a transaction and the worker can't find the row. Explain."
- "Nest has no `@Transactional`. How do you handle transactions?"
- "When would you use `REQUIRES_NEW`, and what does it cost?"

---

## F11 · Authentication and authorisation in the framework

`Intermediate` · Requires: `F03`, `S02` · Unlocks: `S05`, `F26`

### Preface

Authentication answers "who are you"; authorisation answers "may you do this". They belong in
different places: authenticate once at the edge, authorise close to the data.

The mistake that produces real breaches is stopping at route-level checks. A guard proving the user
is a logged-in customer says nothing about whether *this* order belongs to them.

### Details

#### 1. Authentication in the pipeline

**Theory.** A guard or filter reads the credential (a bearer token or a session cookie), verifies it,
loads the principal, and attaches it to the request context. Everything downstream reads the
principal rather than re-parsing the token.

**Example.**
- **Nest**: a `JwtAuthGuard` extending `AuthGuard('jwt')` from Passport, registered globally with
  `APP_GUARD`, and a `@Public()` decorator plus `Reflector` to opt specific routes out. Registering
  globally and opting out is safer than opting in — a new endpoint is protected by default.
- **Django**: `AuthenticationMiddleware` populates `request.user`; DRF adds authentication classes.
- **Spring**: the Spring Security filter chain, configured as a `SecurityFilterChain` bean, populates
  `SecurityContextHolder`.

**Advanced.** "Secure by default" is the design principle to state: the framework should require an
explicit decision to make an endpoint public. Opt-in protection means the endpoint someone forgot to
annotate is open, and that is exactly the endpoint that gets found.

#### 2. Route-level authorisation

**Theory.** Declarative checks on roles or permissions, expressed near the handler.

**Example.**

```ts
@Roles('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
@Delete(':id')
remove(@Param('id') id: string) { ... }
```

The `RolesGuard` reads the metadata with `Reflector` and compares with the principal's roles. Spring:
`@PreAuthorize("hasRole('ADMIN')")`. DRF: permission classes.

**Advanced.** Spring's `@PreAuthorize` accepts expressions that can reference method arguments and
call your beans — `@PreAuthorize("@orderAuth.canEdit(#id, authentication)")` — which moves
object-level checks into the declarative layer. It is powerful and easy to overuse; complex SpEL
expressions are untested, unrefactorable strings. Keep them simple and put real logic in a named
component.

#### 3. Object-level authorisation — the one that matters

**Theory.** The check that most matters is whether this principal may act on **this record**. It
needs data, so it cannot live in the pipeline.

**Example.** The failure and the fix:

```ts
// vulnerable: the guard proved they are logged in, nothing more
const order = await this.orders.findById(id);
return order;

// safe: ownership is part of the query
const order = await this.orders.findByIdForTenant(id, ctx.tenantId);
if (!order) throw new OrderNotFoundError(id);   // 404, not 403 — do not confirm existence
```

**Advanced.** Make it structural rather than remembered: scope every query by tenant or owner in the
repository layer, using the ambient request context; add Postgres row-level security as a backstop
(`DB37`); and write tests that attempt cross-tenant access and assert failure. Relying on each
developer to remember a `WHERE` clause is how IDOR vulnerabilities ship (`S09`).

#### 4. Service-to-service authentication

**Theory.** Internal calls need identity too. Options: mutual TLS with workload identity, a signed
token propagated from the edge, or a service-specific API key.

**Example.** Propagating the user's token to downstream services preserves the user's identity for
authorisation there, and means a downstream service can do no more than the user could. The
alternative — services calling each other with a powerful service account — means a compromised
service can read everything. The token-propagation model is stronger; the practical caveat is token
lifetime on long chains and background work, where you need a token-exchange flow instead (`S04`).

**Advanced.** Never trust an `X-User-Id` header from inside the network unless the network is
genuinely closed and the header is set by a component you control and cannot be spoofed (`M06`).
Defence in depth says verify the signed token at each service, or use mTLS so you know which
workload the call came from (`S15`).

### Interview questions

- "Route guard passes but the user reads another tenant's record. What was missing?"
- "Where do you enforce authorisation: controller, service or repository?"
- "How do you make an endpoint protected by default?"
- "How does service B know who the original user was?"

---

## F12 · Testing

`Intermediate` · Requires: `F04` · Unlocks: `F18`, `M35`, `O08`

### Preface

Tests exist to let you change code confidently. The shape that achieves that: many fast unit tests
for logic, a solid layer of integration tests against a **real** database, and a few end-to-end tests
for critical journeys.

The two most common failures are over-mocking (tests that pass while production breaks) and
under-testing the data layer (where most real bugs live).

### Details

#### 1. Unit tests and dependency injection

**Theory.** DI exists partly so a class can be tested with fakes in place of its collaborators. The
test constructs the class directly, or uses the framework's testing module, and substitutes what it
needs.

**Example.**

```ts
const moduleRef = await Test.createTestingModule({ providers: [OrderService, OrderRepository] })
  .overrideProvider(OrderRepository).useValue(fakeRepo)
  .compile();
```

Spring's `@MockBean` in a `@SpringBootTest` does the same. In Django, without a container, you patch
or pass collaborators explicitly.

**Advanced.** Over-mocking is the standard failure: mocking the repository means you are testing that
your service calls a method, not that the behaviour is correct — and a mock that returns whatever you
told it will happily encode a wrong assumption about the real thing. Prefer a **fake** (a working
in-memory implementation of the same interface) over a mock, and prefer a real database over both
when the logic involves queries at all.

#### 2. Integration tests against real infrastructure

**Theory.** The most valuable tests run your code against a real Postgres, a real Redis and a real
broker, started in containers for the test run.

**Example.** Testcontainers (available for Node, Python and Java) starts a Postgres container, runs
your migrations, and gives you a connection string. The tests then exercise repositories, migrations,
constraints and queries against the real engine. This catches: constraint violations, transaction
behaviour, SQL that is valid in SQLite and not in Postgres, migration errors, and index assumptions.

**Advanced.** An in-memory substitute (H2, SQLite) is the classic false economy: different SQL
dialect, different transaction semantics, no `jsonb`, no partial indexes, different locking. Tests
pass and production fails. If someone tells you their suite uses SQLite against a Postgres
production database, that is a finding.

#### 3. Test isolation

**Theory.** Tests must not depend on each other's data. The usual approaches: wrap each test in a
transaction and roll it back; truncate tables between tests; or give each test its own schema or
database.

**Example.** Django's `TestCase` wraps each test in a transaction and rolls back — fast and clean.
Spring's `@Transactional` on a test does the same.

**Advanced.** The catch worth naming: rolling back **hides commit-time behaviour**. Deferred
constraints, `on_commit` hooks (so your enqueued jobs never fire in tests), triggers on commit, and
anything depending on data actually being visible to another connection. Django provides
`TransactionTestCase` for exactly this, at the cost of speed. Knowing what the fast path hides is the
senior detail.

#### 4. End-to-end and contract tests

**Theory.** End-to-end tests exercise the whole application over HTTP. Keep them few — they are slow
and brittle — and reserve them for critical journeys. Between services, prefer contract tests
(`M35`).

**Example.** Nest: `supertest` against the compiled application with real infrastructure and stubbed
third parties. Spring: `MockMvc` for the web layer alone, or `@SpringBootTest(webEnvironment =
RANDOM_PORT)` with a real server. Django: the test `Client`.

**Advanced.** Stub third parties at the **HTTP boundary** (WireMock, `nock`, `responses`) rather than
mocking your own client class. That way you are testing your client code — including retries,
timeouts, error mapping and deserialisation — which is exactly the code that fails in production.
Mocking your own wrapper tests nothing about how you actually talk to Stripe.

### Interview questions

- "How do you test a service that calls Stripe and writes to Postgres?"
- "Your test suite rolls back transactions — what class of bug does that hide?"
- "Your integration tests use in-memory SQLite. What do you think?"
- "When is a mock the wrong test double?"

---

## F13 · Background jobs

`Advanced` · Requires: `F09`, `F10`, `Q10` · Unlocks: `F14`, `Q21`, `SD10`

### Preface

Anything that does not need to finish before you answer the user should not happen inside the
request: emails, reports, image processing, third-party calls, exports.

Moving work to a queue makes the API fast and resilient. It also means the work now happens
at-least-once, possibly out of order, possibly on a different deployment version — so jobs must be
**idempotent** and **restartable**.

### Details

#### 1. The tooling

**Theory.** A job system needs a queue, workers, retries with backoff, a dead-letter destination, and
visibility into what is stuck.

**Example.**
- **Nest**: BullMQ on Redis (`@nestjs/bullmq`), with `@Processor` classes, per-queue concurrency,
  delayed and repeatable jobs, and a UI (Bull Board) for inspection.
- **Django**: Celery with Redis or RabbitMQ; or RQ / Django-Q for something simpler.
- **Spring**: `@Async` with a `TaskExecutor` for fire-and-forget within the process (no durability),
  Spring Batch for chunked processing, `@KafkaListener` or `@RabbitListener` for real queues.

**Advanced.** `@Async` and similar in-process mechanisms are **not** background jobs: if the process
dies, the work is gone, and there is no retry, no visibility and no backpressure. They are fine for
truly optional work (a cache warm) and wrong for anything that must happen. Say this distinction
clearly; interviewers use it to check whether you understand durability.

#### 2. Jobs must be idempotent

**Theory.** Every queue redelivers. A worker that crashes after doing the work but before
acknowledging causes the job to run again. Design for it rather than trying to prevent it (`M16`).

**Example.** Techniques: a unique constraint on the effect (`UNIQUE (order_id, type)` on a
notifications table, so the second send fails and is caught); a status check at the start (`if
order.status === 'CONFIRMED' return`); an idempotency key passed to the external provider; or
recording the processed job id (`M15`).

**Advanced.** For BullMQ specifically, a deterministic `jobId` gives deduplication at enqueue time —
adding a job with an existing id is ignored. That prevents duplicates from the producer side; it does
not prevent redelivery after a crash, so you still need consumer-side idempotency. Both, not either.

#### 3. Payloads: ids, not snapshots

**Theory.** A job payload should carry the minimum needed to find the data — usually an id — rather
than a copy of it. By the time the job runs, a snapshot may be stale.

**Example.** The bug: the job payload includes `user.email`, the job is delayed an hour by a backlog,
the user changed their address in the meantime, and the email goes to the old one. With just
`userId`, the worker reads the current value. The exception is when you deliberately want the value
**as it was** — an invoice must show the address at the time of purchase — and then storing it is
correct and should be a comment in the code.

**Advanced.** Payload size matters: queues charge for it, Redis holds it in memory, and large
payloads slow everything. Keep payloads small and put large data in object storage with a reference.
Also version your payloads — during a rolling deploy, old workers consume jobs enqueued by new code
and vice versa (`M24`), so a new required field breaks the old worker.

#### 4. Running workers

**Theory.** Workers should be a separate deployment from the API: different scaling profile,
different resource needs, and a slow job must not affect API latency.

**Example.** Two Kubernetes deployments from one image, with the entrypoint selecting API or worker
mode. Scale the workers on **queue depth or oldest-message age** rather than CPU (`Q21`, `O06`). Set
concurrency per worker deliberately — too high and you exhaust the database pool (`DB21`).

**Advanced.** Graceful shutdown matters more for workers than for the API: on `SIGTERM` a worker must
stop taking new jobs, finish the current one, and acknowledge it — otherwise the job is redelivered
and re-executed (which is safe if idempotent, and still wasteful). `terminationGracePeriodSeconds`
must exceed your longest job, or split long jobs into resumable chunks (`F28`).

### Interview questions

- "A job runs twice. Is that a bug?"
- "The job payload holds the full user object and the email goes to the old address. Fix?"
- "Why is `@Async` not a background job system?"
- "How do you scale workers, and on what signal?"

---

## F14 · Scheduling and cron

`Intermediate` · Requires: `F13` · Unlocks: `Q21`

### Preface

Scheduled work — nightly reports, cleanup, reminders — is simple until you run more than one replica.
Then every replica fires the same job at the same moment.

The fix is to ensure only one instance actually runs it: a distributed lock, leader election, or an
external scheduler.

### Details

#### 1. The multi-replica problem

**Theory.** In-process schedulers (`@Cron` in Nest, `@Scheduled` in Spring, Celery beat) run inside
each application instance. With three pods, the job runs three times.

**Example.** The symptom is unmistakable: customers receive three copies of the nightly summary
email. It appears the moment someone scales the deployment, often long after the job was written, and
the cause is not obvious from the job's code.

**Advanced.** Duplicate execution is not always harmless-but-embarrassing; concurrent runs of the
same job can corrupt data (two processes both calculating and writing the same aggregate) or
multiply load (three full table scans at once). Treat "exactly one runner" as a correctness
requirement, not a tidiness one.

#### 2. Three fixes

**Theory.** Take a **lock** before running; elect a **leader** that owns scheduled work; or move
scheduling **outside** the application.

**Example.**
- **Lock**: `SELECT pg_try_advisory_lock(id)` or Redis `SET key val NX PX ttl` at the start of the
  job; skip if not acquired. ShedLock does this for Spring; a small wrapper does it for Nest.
- **Leader election**: one instance holds a lease (etcd, Consul, a database row with an expiry) and
  only it runs scheduled work.
- **External**: a Kubernetes `CronJob` starts a one-off pod that runs the task and exits — the
  scheduler is the platform, and the job is a separate process with its own resources and its own
  failure reporting.

**Advanced.** The external option is usually best for anything substantial: it gets its own CPU and
memory limits, its failures are visible as a failed Job rather than a log line, and it cannot affect
API latency. Keep in-process scheduling for lightweight, frequent tasks where a pod per run would be
wasteful. Note also that a lock with a TTL can expire while the job is still running — so the job
must be safe to run concurrently anyway, or renew its lease.

#### 3. Time, missed runs and overlap

**Theory.** Three issues every scheduler faces: what timezone the schedule is in; what happens to a
run that was missed because the system was down; and what happens if a run is still going when the
next is due.

**Example.** "Every day at 09:00" in a business context means local time, which moves relative to UTC
twice a year. Cron in UTC will run an hour early or late for half the year. Either specify the
timezone explicitly (most schedulers support it) or accept UTC and be clear that it is UTC. And
decide the overlap policy: Kubernetes `CronJob` has `concurrencyPolicy: Forbid` / `Replace` /
`Allow`; in-process schedulers usually need you to guard it yourself.

**Advanced.** Missed runs need a decision: after four hours of downtime, should the hourly job run
four times, once, or not at all? The robust design does not rely on the schedule for correctness —
the job looks at the data and processes whatever is outstanding, so a missed run simply means the
next run has more to do. That makes the schedule a trigger rather than a source of truth, which is
much more resilient.

#### 4. Scheduling at scale

**Theory.** "Send a reminder 24 hours before each of a million appointments" is not a cron problem —
it is a per-entity scheduling problem.

**Example.** Two workable designs: a **delayed queue** (enqueue the job with a delay when the
appointment is created — BullMQ delayed jobs, SQS delay, a Redis sorted set keyed by due time); or a
**polling scan** of a table with a `due_at` index, claiming rows with `FOR UPDATE SKIP LOCKED`
(`DB09`). The second handles cancellation and rescheduling much more naturally, because the data is
the source of truth.

**Advanced.** Add jitter to bulk scheduled work (`M09`): if a million reminders are all due at 09:00,
you create a thundering herd on yourself and on the email provider. Spread them across a window. The
same applies to anything triggered by a shared clock — a good instinct to demonstrate.

### Interview questions

- "You scaled to 3 pods and the nightly report is now sent 3 times. Fix it three ways."
- "Your hourly job missed four runs during an outage. What should happen?"
- "How do you schedule a reminder for each of a million appointments?"
- "What timezone is your cron expression in?"

---

## F15 · Caching at the framework layer

`Intermediate` · Requires: `F03`, `Q02` · Unlocks: `SD05`

### Preface

Frameworks offer easy caching — a decorator on a method, an interceptor on a route. That convenience
hides two traps: an in-process cache is per-pod, so users see different data depending on which
instance they hit; and an automatically generated cache key rarely includes the tenant or the user.

Full caching theory is in `Q01`–`Q09`; this node is about the framework's own mechanisms.

### Details

#### 1. The mechanisms

**Theory.** Three levels: HTTP response caching (headers, handled by CDN or browser); route-level
caching inside the app; and method-level caching of arbitrary results.

**Example.**
- **Nest**: `CacheModule` with a Redis store, `CacheInterceptor` for routes, and `cacheManager`
  injected for manual use. Route caching keys on the URL by default.
- **Django**: `cache_page` decorator, the low-level cache API, `cached_property` for per-instance
  memoisation.
- **Spring**: `@Cacheable`, `@CachePut`, `@CacheEvict` over a pluggable `CacheManager` (Caffeine
  locally, Redis distributed).

**Advanced.** Spring's cache annotations are proxy-based, exactly like `@Transactional` — so
self-invocation bypasses the cache silently (`F22`). Same mechanism, same trap, and a nice thing to
connect out loud in an interview.

#### 2. Cache keys must include the context

**Theory.** A generated key based on the method arguments or the URL omits everything ambient: the
authenticated user, the tenant, the locale, the permission set.

**Example.** The breach: `@Cacheable("user")` on `getProfile(userId)` is fine, but
`CacheInterceptor` on `GET /me` keys on the URL — which is identical for every user. The first
user's profile is served to everyone. Always include tenant and user in the key when the response
depends on them, and prefer an explicit key factory over the default.

**Advanced.** The same problem exists at the HTTP layer with `Vary` (`A04`): a CDN caching a response
that depended on the `Authorization` header without `Vary: Authorization` will serve one user's data
to another. The lesson generalises — **anything that varies the response must be in the cache key**,
including things that are not in the URL.

#### 3. In-process versus distributed

**Theory.** An in-process cache is fastest and is private to one pod. A distributed cache is shared
and consistent across pods, at the cost of a network round trip.

**Example.** With three pods and a 60-second in-process cache, a user refreshing sees values flipping
between old and new depending on which pod answers — confusing and hard to reproduce. Use a shared
cache (Redis) for anything user-visible; keep in-process caching for immutable or slow-changing
reference data (configuration, feature flags, currency rates).

**Advanced.** A two-level cache (local + Redis) gives you speed and shared state, and needs an
invalidation broadcast — publish an eviction message on Redis pub/sub so every pod drops its local
copy. This is a well-known pattern and a well-known source of subtle bugs; only reach for it when the
Redis round trip is genuinely the bottleneck (`Q09`).

#### 4. Bounded caches

**Theory.** An unbounded in-memory cache is a memory leak with a friendly name. It must have a
maximum size with an eviction policy, or a TTL, or both.

**Example.** A `Map` keyed by user id, populated on every request, with no eviction: memory grows
until the pod is OOMKilled (`O04`). Use an LRU implementation with a size limit (`lru-cache`,
Caffeine) — and remember the limit should be in **bytes or entries you have measured**, not a number
someone guessed.

**Advanced.** In Node this is one of the most common memory-leak shapes (`C12`), and it is
particularly insidious because it works fine in development and in staging where traffic is low.
Alert on container memory as a proportion of the limit, and take a heap snapshot when it climbs.

### Interview questions

- "`@Cacheable` on a method that takes a user id — what is the risk in a multi-tenant app?"
- "You added a 60-second in-memory cache and users see data flip-flopping. Why?"
- "Why does `@Cacheable` sometimes appear to do nothing in Spring?"
- "What must every in-memory cache have?"

---

## F16 · Realtime: WebSockets and server-sent events

`Advanced` · Requires: `F03`, `A16` · Unlocks: `SD12`, `C16`

### Preface

A normal HTTP request is short-lived and stateless, which is why scaling it is easy. A WebSocket is a
long-lived, stateful connection pinned to one process — which breaks nearly every assumption behind
horizontal scaling.

The central problem to solve: a message for user B may arrive at a pod that does not hold B's
connection.

### Details

#### 1. Choosing the transport

**Theory.** **Long polling** — repeated requests; simple, wasteful. **Server-sent events (SSE)** —
one-way server-to-client over plain HTTP, with automatic reconnection and event ids built in.
**WebSockets** — bidirectional, lowest overhead per message, and you implement reconnection,
heartbeats and auth yourself.

**Example.** For a live dashboard, a notification feed or a progress indicator — all one-way — SSE is
usually the better choice: it works through proxies, uses ordinary HTTP infrastructure, reconnects on
its own, and supports resuming with `Last-Event-ID`. Choose WebSockets when the client genuinely
sends frequent messages too: chat, collaborative editing, games.

**Advanced.** SSE's practical limit is the browser's per-domain connection cap under HTTP/1.1 (six),
which multiple tabs can exhaust — HTTP/2 removes it via multiplexing (`A10`). WebSockets bypass the
cap but are more likely to be interfered with by corporate proxies. Both need a plan for what happens
when the connection cannot be established at all: fall back to polling.

#### 2. Scaling across instances

**Theory.** Connections live in one process's memory. To deliver a message from anywhere, instances
must share a channel.

**Example.** The standard solution is a pub/sub backplane: when pod A needs to send to user B, it
publishes to Redis; every pod subscribes; the pod holding B's connection delivers it. In Nest this is
`@socket.io/redis-adapter`; in Django it is Channels with a Redis channel layer; in Spring it is an
external STOMP broker relay (RabbitMQ) rather than the simple in-memory broker.

**Advanced.** Redis pub/sub is **fire and forget** — if the target pod is momentarily disconnected,
the message is lost. For messages that must not be lost, persist them (a per-user inbox in the
database or a Redis stream) and have the client fetch anything missed on reconnect, using a cursor.
"Real-time delivery plus a durable inbox" is the design that actually works (`SD12`).

#### 3. Authentication and lifetime

**Theory.** A WebSocket authenticates once at the handshake and then stays open for hours — longer
than the access token's lifetime.

**Example.** Practical approach: authenticate during the handshake (a token in the query string is
common but logs the token in access logs; a subprotocol header or a short-lived ticket obtained over
HTTP is better); record the principal on the connection; and periodically re-validate — either by
requiring the client to send a refreshed token over the socket, or by checking a revocation list on a
timer and closing the connection if access has been withdrawn.

**Advanced.** Also check authorisation **per subscription**, not only at connect: a client asking to
subscribe to `conversation:123` must be shown to be a participant. A connection-level check is the
same mistake as route-level-only authorisation (`F11`), and it is easy to miss because the
subscription request does not look like an HTTP endpoint.

#### 4. Resources and backpressure

**Theory.** Each connection costs memory and a file descriptor, and a client that reads slowly makes
the server buffer.

**Example.** Budget for it: tens of thousands of connections per pod is realistic for Node with
small per-connection state, and you must raise the file descriptor limit (`O01`). Send heartbeats
(ping/pong) to detect connections that are dead but not closed, or they accumulate. And cap the
outbound buffer per connection — if a client cannot keep up, drop it rather than growing memory
without limit (`C16`).

**Advanced.** Deploys are the operational pain: rolling a deployment disconnects every client at
once, and they all reconnect simultaneously — a thundering herd against your own service and the
backplane. Mitigate with reconnect jitter on the client, and by draining connections gradually rather
than closing them all at `SIGTERM` (`F28`).

### Interview questions

- "Chat app scaled to 5 pods; users on different pods can't see each other's messages. Fix."
- "SSE or WebSocket for a live dashboard?"
- "The access token expires while the socket is open. What now?"
- "What happens to 50,000 connections during a rolling deploy?"

---

## F17 · File upload, download and streaming

`Intermediate` · Requires: `F02`, `C19` · Unlocks: `S14`

### Preface

Files are big and user-controlled, which makes them both a memory risk and a security risk.

The rule that solves most of it: **do not let the bytes go through your API.** Give the client a
presigned URL so it uploads straight to object storage, and let downloads come from storage or a CDN.
Your service handles metadata and permissions only.

### Details

#### 1. Presigned uploads

**Theory.** Your API authorises the upload and returns a time-limited URL that permits a `PUT` to a
specific object key in your bucket. The client uploads directly. Storage notifies you (an event) or
the client confirms.

**Example.** The flow: client requests an upload → API checks quota and permissions, generates a key
(`tenant/{tenantId}/uploads/{uuid}`), returns a presigned URL valid for five minutes with a
content-type and size condition → client PUTs the bytes → S3 emits an event → a worker validates,
scans, transcodes and records the file as ready.

**Advanced.** Constrain the presigned URL as tightly as you can: expiry, exact key, content-type, and
maximum size (S3 POST policies support a content-length range; plain presigned PUTs are harder to
bound, so verify size after the fact). Never let the client choose the key freely — path traversal
and overwriting another tenant's object both follow. Generate the key server-side and store the
mapping.

#### 2. When files do go through your API

**Theory.** Sometimes you must accept the upload directly (small files, a legacy client). Then the
critical rule is to **stream, not buffer**: never load the whole file into memory.

**Example.** In Nest, `FileInterceptor` with multer buffers to memory by default — fine for a 1MB
avatar, fatal for a 2GB video with several concurrent uploads. Configure disk storage or a streaming
handler, and always set `limits: { fileSize }`. The equivalent in Django is chunked upload handlers;
in Spring, `MultipartFile` with `spring.servlet.multipart.max-file-size` (and note Spring writes to
disk above a threshold by default).

**Advanced.** Size limits must exist at **every** layer: the load balancer or ingress
(`nginx client_max_body_size`), the framework, and the storage policy. A limit in only one place is
bypassed the moment traffic takes another path, and an unbounded upload endpoint is a trivial
denial-of-service (`S13`).

#### 3. Validating what arrived

**Theory.** The filename and the declared content type are attacker-controlled. Validate by content,
not by claim.

**Example.** The checklist: never trust the extension (check magic bytes); never use the
client-supplied filename as a path (generate your own; strip directory components); re-encode images
rather than storing them as received (this strips embedded payloads and EXIF data, which also has
privacy implications); virus-scan anything that will be downloaded by other users; and serve user
content from a **separate domain** so a malicious HTML file cannot run script in your application's
origin.

**Advanced.** The separate-domain rule is the one most often missed and the one that turns a stored
file into stored XSS (`S08`): if `app.example.com` serves user-uploaded HTML, that script runs with
access to your cookies and local storage. Serve from `usercontent-example.net`, set
`Content-Disposition: attachment` where appropriate, and send
`X-Content-Type-Options: nosniff`.

#### 4. Downloads and streaming responses

**Theory.** For downloads, redirect to a presigned URL or serve through a CDN. For generated content
(a large export), stream it rather than building it in memory.

**Example.** A CSV export of ten million rows: do not build a string. Stream from a database cursor
through a transform to the response, so memory stays flat (`C19`). In Nest, return a
`StreamableFile` or pipe to the response; in Django, `StreamingHttpResponse`; in Spring,
`StreamingResponseBody`. Better still for very large exports: generate asynchronously into object
storage and email a link (`A21`) — an HTTP request held open for four minutes is fragile regardless
of memory.

**Advanced.** Streaming has a real drawback to acknowledge: once you have sent a 200 and started the
body, you cannot change the status code if the query fails halfway. You end up with a truncated file
and a successful-looking response. For anything important, the async-job-plus-link pattern is more
robust, and it is the answer that shows you have hit this in production.

### Interview questions

- "Users upload 2GB videos. Design the flow."
- "An export endpoint OOMs the pod. Fix it."
- "Why serve user-uploaded files from a different domain?"
- "What do you validate about an uploaded file, and what do you never trust?"
