[← back to the field index](README.md)

# Frameworks · Part 4 — Performance, Docs, Security & Choosing a Stack

Nodes `F24`–`F28`.

---

## F24 · Framework-level performance and resource management

`Advanced` · Requires: `F09`, `DB21`, `C03` · Unlocks: `F28`, `C14`

### Preface

When a service is slow, the framework is rarely the cause. The usual culprits, in order: database
queries, downstream calls, serialisation of large payloads, and resource exhaustion — the pool, the
event loop, or memory.

The skill being tested is knowing where to look, and knowing that "CPU is at 40% and latency is
terrible" points at waiting, not computing.

### Details

#### 1. Pools and limits multiply by pod count

**Theory.** Every per-process limit — database connections, HTTP sockets, worker concurrency — is
multiplied by the number of instances. The database and every downstream service see the total.

**Example.** The arithmetic to do before scaling: `pods x db_pool` against `max_connections`
(`DB21`); `pods x http_agent_maxSockets` against what the downstream service can accept;
`pods x worker_concurrency` against the queue's and database's capacity. Autoscaling makes this
dynamic — scaling out under load can take down the database precisely when it is already struggling,
which is a genuinely counter-intuitive failure worth being able to describe.

**Advanced.** The mitigation is that limits should be derived from the shared resource, not from the
pod: a global budget divided by the maximum replica count. Or introduce a shared limiter — a
connection pooler in front of the database (`DB21`), a concurrency limit enforced at the gateway.
Otherwise the autoscaler's reaction to load makes the incident worse.

#### 2. Connection reuse

**Theory.** Opening a new TCP connection plus a TLS handshake per outbound request costs one to three
round trips before any data flows. Reusing connections removes that entirely.

**Example.** In Node, the default global HTTP agent historically did not keep connections alive
(modern versions do, but many libraries create their own agent). Configure it explicitly:

```js
const agent = new https.Agent({ keepAlive: true, maxSockets: 50, keepAliveMsecs: 30_000 });
```

Watch `maxSockets`: too low and requests queue inside your process invisibly; too high and you
overwhelm the downstream. The same applies to database pools, Redis clients and gRPC channels.

**Advanced.** Keep-alive interacts with load balancing (`M07`): a long-lived connection stays pinned
to one backend, so a client that keeps connections for hours will not discover newly scaled
instances. Set a maximum connection age or a maximum number of requests per connection so the fleet
re-balances periodically. This is a subtle, real problem that shows up as uneven pod CPU.

#### 3. Serialisation cost

**Theory.** Turning objects into JSON is pure CPU, and at scale it is frequently the largest single
consumer in a Node service — ahead of your business logic.

**Example.** Practical mitigations: return fewer fields (a response DTO rather than the whole entity,
`F06`); paginate rather than returning large arrays; avoid deeply nested structures; use a
schema-based serialiser (Fastify compiles one per route, which is much faster than generic
reflection); and consider a binary format for internal service-to-service traffic (`A14`).

**Advanced.** `JSON.parse` and `JSON.stringify` are **synchronous and blocking** in Node — a 50MB
payload stalls the event loop for hundreds of milliseconds, stalling every other in-flight request in
that process (`C03`, `C04`). The fix is not a faster parser; it is not moving 50MB through a single
request. Stream it, paginate it, or hand it off to object storage (`F17`).

#### 4. Finding the bottleneck

**Theory.** Use signals that distinguish waiting from computing. High latency with low CPU means
waiting: on a downstream, on the database, on a lock, or in a queue for a resource.

**Example.** The checklist for "p99 is 2s and CPU is 40%":
1. **Event loop delay** — `perf_hooks.monitorEventLoopDelay()`; if it is high, something synchronous
   is blocking (`C03`).
2. **Pool saturation** — pool wait time and in-use count; if requests queue for a connection, the
   fix is faster queries or a bigger pool (`DB21`).
3. **Downstream latency** — per-dependency histograms; the slow one is usually obvious.
4. **Database** — `pg_stat_statements` for the actual cost (`DB39`).
5. **GC pauses** — for the JVM especially; in Node, check heap growth (`C12`).

**Advanced.** Instrument these four as standard metrics in every service — event loop delay, pool
utilisation, per-dependency latency, and GC or memory — so the checklist is a dashboard rather than
an investigation. Being able to list them unprompted is exactly what "senior" sounds like in a
performance question (`O14`).

### Interview questions

- "Your Nest service holds p99 at 2s under load but CPU is 40%. Where do you look?"
- "Why can scaling out make a database problem worse?"
- "What is usually the top CPU consumer in a Node API?"
- "How does HTTP keep-alive interact with load balancing?"

---

## F25 · API documentation and code generation

`Intermediate` · Requires: `F02`, `F06` · Unlocks: `A22`

### Preface

An API that nobody can discover costs the same to build and is worth much less. OpenAPI (formerly
Swagger) is the standard description format for HTTP APIs, and both generating it from code and
generating code from it are viable.

The real question an interviewer is asking is: how do you stop the documentation from drifting away
from reality?

### Details

#### 1. Code-first

**Theory.** Annotate your controllers and DTOs; the framework generates the specification at
startup. The documentation lives with the code, so it changes when the code changes.

**Example.** Nest's `@nestjs/swagger` reads the DTO decorators — with the CLI plugin it infers most
of the schema from TypeScript types, so you annotate only what it cannot infer (`@ApiProperty` for
descriptions and examples). Django has drf-spectacular; Spring has springdoc-openapi.

**Advanced.** Code-first drifts in the parts the generator cannot see: error responses, authorisation
requirements, rate limits, and the meaning of fields. It will document that a field is a string and
not that it must be an ISO currency code. Document error codes and business rules deliberately —
they are what integrators actually need.

#### 2. Spec-first

**Theory.** Write the OpenAPI document first, review it, then generate server stubs and client SDKs
from it. The specification is the contract and the source of truth.

**Example.** Spec-first wins when several teams or external partners depend on the API: the frontend
can generate a typed client and build against a mock server before the backend exists, and the
contract is reviewed as a design artefact rather than discovered after implementation.

**Advanced.** The trade is that generated server code is often awkward to live with, and the
specification becomes a second thing to keep in sync with reality. A pragmatic middle path, common in
practice: code-first generation, plus a CI check that the generated specification has not changed
incompatibly (`oasdiff`), plus a linter for house style (Spectral). You get code-first ergonomics with
spec-first discipline.

#### 3. Keeping documentation true

**Theory.** Documentation is only trustworthy if something verifies it against behaviour.

**Example.** Mechanisms that work: generate the specification in CI and fail the build on an
unreviewed breaking change; run contract tests against the specification (`A22`); validate real
responses against the schema in your integration tests; and publish the specification as a versioned
artefact so consumers can diff it.

**Advanced.** For internal service-to-service APIs, a `.proto` file (`A14`) is a stronger contract
than OpenAPI because it is compiled into both sides — a breaking change fails the build rather than
appearing in a document nobody re-read. Where you control both ends, prefer the contract that a
compiler enforces.

#### 4. Generated clients

**Theory.** Generating a typed client from the specification removes an entire class of integration
bug and saves consumers work.

**Example.** Publish a versioned SDK per language your consumers use, generated in CI on every
release. For internal TypeScript consumers, generating types from the OpenAPI document
(`openapi-typescript`) gives compile-time safety across service boundaries at almost no cost —
a high-value, low-effort improvement worth proposing in an interview.

**Advanced.** Generated clients have a hidden coupling: they encode your API's shape into the
consumer's build, so a regenerated client after a breaking change fails at compile time — which is
exactly what you want, provided you have a deprecation process (`A06`). Without one, you have simply
moved the breakage earlier, which is still better.

### Interview questions

- "Code-first or spec-first OpenAPI? When does spec-first win?"
- "How do you stop your documentation from drifting?"
- "What does OpenAPI fail to capture?"
- "Would you generate clients for consumers? What does that couple?"

---

## F26 · Web security in the framework

`Advanced` · Requires: `F03`, `F11`, `S06` · Unlocks: `S06`, `S08`

### Preface

Frameworks ship with security features that are only as good as their configuration. The common
failures are not exotic: CORS opened too wide, CSRF misunderstood, mass assignment left enabled,
rate limiting absent, and raw SQL built by string concatenation.

Full treatment is in the security field; this node is the framework-level checklist.

### Details

#### 1. CORS, configured properly

**Theory.** CORS is a **browser** mechanism that controls which origins may read responses from your
API. It is not server-side security — it does nothing against curl or a compromised client (`A19`).

**Example.** The configuration that matters:

```ts
app.enableCors({
  origin: ['https://app.example.com'],   // explicit list, never true/'*' with credentials
  credentials: true,
  maxAge: 86400,
});
```

Reflecting the request's `Origin` header back is the dangerous shortcut — it means *any* origin is
allowed, which with `credentials: true` lets any website make authenticated requests as your logged-in
user. Django's `CORS_ALLOW_ALL_ORIGINS` and Spring's `allowedOrigins("*")` have the same hazard.

**Advanced.** Be able to state plainly that CORS protects **your users' browsers**, not your API, and
that authorisation must be enforced regardless. Candidates who describe CORS as an API security
control are revealing a gap.

#### 2. CSRF, and when it does not apply

**Theory.** CSRF works because browsers attach cookies automatically to cross-site requests. If your
API authenticates with a **cookie**, you need CSRF protection. If it authenticates with an
`Authorization: Bearer` header, you do not — because a cross-site page cannot make the browser attach
a header it does not know.

**Example.** So: a Django or Rails server-rendered application needs CSRF tokens (both ship them by
default). A Nest API consumed by a SPA using bearer tokens does not. A Nest API using session cookies
does. `SameSite=Lax` on cookies mitigates most CSRF by default in modern browsers, and is a useful
additional layer rather than a complete substitute.

**Advanced.** The full cookie recipe: `HttpOnly` (JavaScript cannot read it, limiting XSS damage),
`Secure` (HTTPS only), `SameSite=Lax` or `Strict`, a sensible `Path`, and a short lifetime with
rotation. Know the trade: cookies are vulnerable to CSRF but protected from XSS theft when
`HttpOnly`; localStorage tokens are immune to CSRF and stealable by any XSS. Cookies plus CSRF
protection is generally the stronger position (`S03`).

#### 3. Rate limiting and resource limits

**Theory.** Without limits, one client can exhaust your capacity accidentally or deliberately.
Limits are needed on request rate, body size, query complexity and execution time.

**Example.** `@nestjs/throttler`, DRF's throttle classes, bucket4j for Spring. In practice, put the
coarse rate limit at the gateway or CDN where it is cheap (`M06`), and keep application-level limits
for expensive specific endpoints (login, export, search). Also set: maximum request body size,
maximum upload size (`F17`), `statement_timeout` on the database (`DB21`), and a per-request timeout.

**Advanced.** Watch out for resource exhaustion that is not rate-based: a regular expression with
catastrophic backtracking pinning a CPU (**ReDoS**), an unbounded GraphQL query (`A15`), a
pagination parameter of `limit=1000000`, or a zip bomb. Validate and cap every numeric parameter, and
avoid user-controlled regular expressions entirely (`S13`).

#### 4. The rest of the checklist

**Theory.** Security headers, mass assignment, SSRF, injection, and information leakage.

**Example.** A practical audit list for any service:
- Security headers via helmet or equivalent: `Content-Security-Policy`, `Strict-Transport-Security`,
  `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `X-Frame-Options`.
- Mass assignment: `ValidationPipe` with `whitelist: true` (`F06`).
- SSRF: user-supplied URLs must be validated against an allow-list and blocked from private ranges
  and the cloud metadata endpoint (`S08`).
- Injection: parameterised queries only; audit every raw SQL call for string interpolation (`S07`).
- Leakage: `DEBUG=False`, no stack traces to clients, generic errors for authentication (`F07`).

**Advanced.** The highest-value habit is making the **safe path the easy path**: a service template
with all of this configured, a lint rule that flags raw SQL with template literals, a global
validation pipe enabled in the template, and protected-by-default authentication (`F11`). Security
that depends on every developer remembering will fail; security that is the default will mostly hold
(`S17`).

### Interview questions

- "You use JWT in an `Authorization` header. Do you need CSRF protection? Why?"
- "Your endpoint fetches a user-supplied URL for link previews. What must you defend against?"
- "Does CORS protect your API?"
- "What is on your security checklist for a new service?"

---

## F27 · Choosing and migrating stacks

`Advanced` · Requires: `F18`, `F21`, `F22`, `F23` · Unlocks: `SD15`

### Preface

"Which framework should we use?" is rarely a technical question with a technical answer. The
decisive factors are usually the team's expertise, the ecosystem for the problem domain, and what the
organisation can operate.

For your situation specifically — strong in Nest, interviewing where another stack may be primary —
the most valuable thing to demonstrate is that you know the *concepts* transfer, and can say exactly
what would take you time to learn.

### Details

#### 1. The decision criteria, in order

**Theory.** A defensible ordering: (1) team expertise and the local hiring pool; (2) ecosystem fit
for the domain — machine learning means Python, enterprise integration means Java, infrastructure
means Go; (3) operational maturity — do you already have deployment, monitoring and libraries for
it; (4) performance profile against the actual workload; (5) long-term maintenance and the
ecosystem's health.

**Example.** Applied: a five-person team, all strong in TypeScript, building a standard SaaS backend.
Node with Nest is right, even if Go would use less memory, because delivery speed and the ability to
maintain it dominate. The same team building a machine-learning inference service should use Python,
because the ecosystem is not optional there.

**Advanced.** The criterion people underweight is **operability**: a stack nobody on call understands
is a liability at 3am. Adding a second language to an organisation costs a second set of build
tooling, security patching, dependency management, observability integration, and on-call
expertise. That cost is real, recurring, and usually larger than the performance difference that
motivated the change.

#### 2. Performance honestly

**Theory.** For typical I/O-bound backend work — call a database, call a service, serialise JSON —
the runtime is rarely the limiting factor. Differences appear under CPU-bound work, at very high
concurrency, and in memory footprint and start-up time.

**Example.** Rough characteristics worth knowing: Node is excellent at I/O concurrency and poor at
CPU-bound work (one thread per process, `C04`). The JVM is fast once warm, uses more memory, and
starts slowly (unless natively compiled). Go is fast, starts instantly, and uses little memory.
Python is the slowest per operation and has the richest data ecosystem.

**Advanced.** Start-up time and memory matter more than throughput in two situations: serverless,
where cold start is user-visible latency (`O12`), and dense container deployment, where memory per
pod determines how many fit on a node and therefore your bill. Those are often the real deciding
factors rather than requests per second.

#### 3. Migrating between stacks

**Theory.** Never rewrite a working system in a new language. Move one service at a time, behind a
stable interface, with the old system running until the new one is proven — the strangler fig
(`M28`).

**Example.** A credible plan: pick one new, low-risk service to build in the new stack; build the
operational scaffolding for it properly (CI, logging, metrics, tracing, deployment, on-call runbook);
run it for a quarter; then decide whether to continue. The first service in a new stack costs three
times what it should, because you are building the platform as well as the service — budget for that
explicitly.

**Advanced.** The most common failure is adopting a second stack for one service and never
completing the migration, leaving the organisation permanently paying for two of everything. Decide
in advance what "we are committing" or "we are reverting" looks like, and set a date to decide. Being
the person who says this is a strong senior signal.

#### 4. Becoming productive in a new framework

**Theory.** The concepts transfer; the vocabulary and the ecosystem are the cost. Map what you know
onto the new names, and then learn the parts that genuinely differ — the concurrency model, the ORM,
and the framework's characteristic traps.

**Example.** The honest answer to "how long until you are productive in Spring Boot": productive in
two to three weeks, fluent in a few months. The concepts — dependency injection, the filter chain,
transactions, an ORM with a persistence context, validation at the boundary — are ones you already
use in Nest. What takes time is the ecosystem (which library for what), the build tooling, the
idioms, and the traps: proxy-based annotations, `open-in-view`, JPA's lazy loading.

**Advanced.** Give that answer with **specifics** — naming `@Transactional` self-invocation and
`open-in-view` proves you have already done the mapping rather than merely claiming transferable
skills. Then say what you would read first and what you would build first. That converts "I could
learn it" into "I have a plan", which is the difference between a hopeful answer and a convincing
one.

### Interview questions

- "We are a Node shop considering Go for a new service. Walk me through the decision."
- "How long until you are productive in Spring Boot?"
- "What does adding a second language to an organisation actually cost?"
- "When is a rewrite justified?"

---

## F28 · Runtime and deployment model

`Advanced` · Requires: `F01`, `F19`, `F24` · Unlocks: `O03`, `O06`, `C03`, `C07`

### Preface

How your framework runs — one process or many, one thread or many, how it starts, how it stops —
determines how you scale it and whether deploys drop requests.

The single most valuable thing in this node is **graceful shutdown**. Getting it wrong means every
deploy produces a handful of failed requests, which teams often accept as normal and which is
entirely avoidable.

### Details

#### 1. Process and threading models

**Theory.** **Node** — one thread executing your JavaScript per process; scale with more processes
(`cluster`) or more containers. **Python/Django** — one request per worker process (sync workers);
scale with more workers and more containers. **JVM/Spring MVC** — many threads in one process; scale
with a bigger thread pool, then more containers.

**Example.** The consequence for sizing: a Node pod with 1 CPU runs one useful JavaScript thread,
so giving it 4 CPUs mostly wastes three unless you use `cluster` or worker threads. A JVM pod
benefits from multiple cores directly. A Django pod needs `workers ≈ 2 x cores + 1`. Getting this
wrong means paying for capacity you cannot use.

**Advanced.** `cluster` inside a container versus more containers: more containers is usually right,
because the orchestrator already does the scheduling, you get per-instance isolation and independent
restarts, and metrics are per-pod. `cluster` wins when there is significant fixed per-process
overhead (a large in-memory cache you would rather not duplicate) or when you have a large machine
and want to reduce per-pod overhead. Say which and why rather than asserting one (`O06`).

#### 2. Graceful shutdown — the full recipe

**Theory.** When a pod is terminating, requests are still arriving from clients whose routing caches
have not updated. The server must keep serving briefly, then drain, then exit.

**Example.** The complete sequence:
1. Kubernetes marks the pod terminating and begins removing it from endpoints; it sends `SIGTERM`
   and runs `preStop`.
2. `preStop` sleeps 5-15 seconds — **the pod must keep serving during this**, because endpoint
   removal has not yet propagated to every proxy and client (`M05`).
3. The application's `SIGTERM` handler makes the readiness probe fail and stops accepting new work
   (for a worker: stop pulling jobs).
4. In-flight requests finish; long-lived connections are closed with `Connection: close`.
5. Close the database pool, the broker connection, and flush logs and metrics.
6. Exit 0.
7. `terminationGracePeriodSeconds` must exceed steps 2-5, or the pod is `SIGKILL`ed mid-request.

**Advanced.** In Nest this needs `app.enableShutdownHooks()` plus `onModuleDestroy` implementations —
and it is **off by default** (`F19`). In Docker, use the exec form of `CMD` so your process is PID 1
and actually receives `SIGTERM`; the shell form makes the shell PID 1 and it does not forward signals
(`O03`). Both are single-line fixes for a problem teams often live with for years.

#### 3. Startup

**Theory.** Startup must be fast enough for the orchestrator's probes and must not accept traffic
before it is ready.

**Example.** Distinguish the probes: **startup probe** tolerates a slow boot (a JVM taking 40
seconds) without the liveness probe killing it; **readiness** controls traffic; **liveness** restarts
a wedged process. Readiness should check that the application can serve — dependencies it cannot
function without — while **liveness must not check dependencies**, or a database blip restarts your
whole fleet and turns a degradation into an outage (`O06`).

**Advanced.** Cold start matters in three places: serverless (user-visible latency, `O12`),
autoscaling responsiveness (a pod that takes 60 seconds to start cannot absorb a traffic spike), and
rolling deploys (slow start means a long window at reduced capacity). Node starts in a second or two;
the JVM in tens of seconds unless you use CDS, AOT or GraalVM native image.

#### 4. Statelessness

**Theory.** Horizontal scaling requires that any instance can serve any request. Anything held in one
process's memory — sessions, uploads in progress, in-memory caches, WebSocket connections,
scheduler state — breaks that assumption.

**Example.** The audit: sessions → Redis or signed cookies; uploads → object storage (`F17`);
caches → shared, or accept per-pod inconsistency deliberately (`F15`); WebSockets → a pub/sub
backplane (`F16`); scheduled jobs → a lock or external scheduler (`F14`); local file writes → object
storage, because container filesystems are ephemeral.

**Advanced.** The twelve-factor principles that matter most here: processes are stateless and
share-nothing; backing services are attached resources addressed by configuration; disposability
means fast startup and graceful shutdown; and logs go to stdout as a stream rather than to files the
application manages. Those four, stated in your own words with concrete examples, are a complete
answer to "what makes a service cloud-native".

### Interview questions

- "Your pods drop requests on every deploy. Fix it end to end."
- "Should you run Node `cluster` inside a container, or more containers?"
- "Why must a liveness probe not check the database?"
- "What state in a typical service prevents horizontal scaling?"
