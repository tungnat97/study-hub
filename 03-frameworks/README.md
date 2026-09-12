[← back to the index](../README.md)

# Field 3 — Backend Frameworks (NestJS deep · Django · Spring Boot · others)

Strategy for the interview: **know the concepts framework-agnostically, then name the three
implementations.** A senior candidate who says "this is the DI container; Nest calls it providers,
Spring calls it beans, Django mostly doesn't have one and here's what that costs" outranks a
candidate who only knows Nest. Every node below has a *concept* line and a *per-framework* line.

Legend: `B` beginner · `I` intermediate · `A` advanced · `X` expert.

---

## Nodes

#### F01 · What a framework is doing: request lifecycle
`B` · Requires: A01 · Unlocks: F02, F03, F28
- Concept: socket accept → parse HTTP → route match → middleware chain → handler → serialise →
  response; where the framework sits relative to the HTTP server and the process model.
- Nest: `NestFactory` wraps Express (default) or Fastify; `main.ts` bootstrap. Django: WSGI/ASGI
  callable, `get_wsgi_application()`, middleware stack, URLconf resolution, view, response.
  Spring: servlet container (Tomcat) → `DispatcherServlet` → HandlerMapping → HandlerAdapter →
  controller → `ViewResolver`/`HttpMessageConverter`.
- Q: "Trace a request from TCP accept to JSON response in your framework."
- Q: "What is the DispatcherServlet / what is WSGI / what does NestFactory actually create?"

#### F02 · Routing, controllers, params, binding
`B` · Requires: F01 · Unlocks: F06, F07, F25, A03
- Concept: path/query/body/header binding, route precedence and ambiguity, path params vs query,
  method routing, versioned routes, wildcard ordering.
- Nest: `@Controller('users')`, `@Get(':id')`, `@Param/@Query/@Body/@Headers`, route order matters
  (`:id` before `me` shadows it). Django: `path()`/`re_path()`, DRF `ViewSet` + `DefaultRouter`.
  Spring: `@RestController`, `@GetMapping`, `@PathVariable`, `@RequestParam`, `@RequestBody`.
- Q: "`GET /users/me` returns a 'user not found' for id=me. Why?"

#### F03 · The middleware / interceptor pipeline
`I` · Requires: F01 · Unlocks: F07, F08, F11, F15, F26
- Concept: cross-cutting concerns as a chain; ordering; pre vs post processing; short-circuiting;
  error propagation through the chain; where to put auth vs logging vs transactions.
- **Nest execution order (classic interview question):** middleware → guards → interceptors (pre) →
  pipes → handler → interceptors (post) → exception filters. Know it cold.
- Django: middleware `__call__` with request going down and response coming back up,
  `process_view`/`process_exception` hooks. Spring: servlet Filter → Interceptor (`preHandle`,
  `postHandle`, `afterCompletion`) → `@ControllerAdvice`; plus AOP `@Around` advice.
- Q: "Order of guards, interceptors and pipes in Nest, and why is a guard before a pipe?"
- Q: "Where would you implement request timing, and where tenant resolution? Why different layers?"

#### F04 · Dependency injection & IoC
`I` · Requires: F01 · Unlocks: F09, F12, F18, F19, F22
- Concept: inversion of control, constructor injection, interfaces over implementations,
  lifetimes/scopes (singleton/request/transient), circular dependency resolution, why DI exists at
  all (testability, swap implementations, lifecycle management) and its costs (indirection, startup
  graph, magic).
- Nest: providers, `@Injectable()`, module `providers`/`exports`/`imports`, custom providers
  (`useClass/useValue/useFactory/useExisting`), injection tokens for interfaces (TS interfaces do not
  exist at runtime — a classic question), `forwardRef` for circular deps.
  Spring: beans, `@Component/@Service/@Bean`, `@Autowired` (prefer constructor), scopes,
  `@Qualifier`/`@Primary`, `@Conditional`. Django: no container — you import modules and use
  settings, which is simpler and less testable; DI arrives via pytest fixtures/manual wiring.
- Q: "Why can't you inject a TypeScript interface in Nest, and what do you do instead?"
- Q: "Field vs constructor injection — why is field injection discouraged?"

#### F05 · Configuration, environments, secrets
`B` · Requires: F01 · Unlocks: F18, O19, S10
- Concept: 12-factor config in the environment, typed and **validated at boot** (fail fast), no
  secrets in the repo or image, per-environment overrides, feature flags vs config, reloadable vs
  static config.
- Nest: `@nestjs/config` + Joi/zod schema validation, `ConfigService`, namespaced config factories.
  Django: `settings.py` split per environment, `django-environ`, `DEBUG=False` checklist.
  Spring: `application.yml` + profiles, `@ConfigurationProperties` (typed, validated),
  relaxed binding, config server.
- Q: "Where does a secret live from developer laptop to production pod?"
- Q: "Why validate config at startup rather than on first use?"

#### F06 · Validation & serialisation
`I` · Requires: F02 · Unlocks: F07, F25, S09
- Concept: never trust input; validate at the boundary into a typed DTO; separate the wire model
  from the domain model from the persistence model; whitelist, not blacklist; output serialisation
  must not leak fields (password hash, internal ids).
- Nest: `ValidationPipe` with `class-validator`/`class-transformer`, `whitelist: true` +
  `forbidNonWhitelisted` (mass-assignment defence), `transform: true`, `@Exclude()`/
  `ClassSerializerInterceptor`; or zod + a custom pipe. Django: DRF serializers (validate,
  `read_only_fields`), forms. Spring: `@Valid` + Jakarta Bean Validation, Jackson `@JsonIgnore`/
  `@JsonView`, DTO mapping (MapStruct).
- Q: "A user POSTs `{ "role": "admin" }` to your update endpoint. What stops them?"
- Q: "Where do you validate: DTO, service, or database? Argue for all three."

#### F07 · Error handling
`I` · Requires: F02, F03, F06 · Unlocks: F08, A07
- Concept: expected (domain) vs unexpected errors; map domain errors to HTTP codes at the boundary
  only; a single consistent error contract (RFC 7807); never leak stack traces or SQL; correlation id
  in every error response; retryable vs terminal errors.
- Nest: `HttpException`, exception filters, a global filter for domain→HTTP mapping.
  Django: DRF `exception_handler`, custom exceptions. Spring: `@ControllerAdvice` +
  `@ExceptionHandler`, `ProblemDetail` (Spring 6).
- Q: "Should the service layer throw `HttpException`/`ResponseStatusException`? Why not?"
- Q: "Design your API's error format and explain each field."

#### F08 · Logging
`I` · Requires: F03, F07 · Unlocks: O13, M25
- Concept: structured JSON logs, levels, request-scoped context (correlation id, user id, tenant)
  without threading it through every function, sampling, never log PII/secrets/tokens, log once per
  error at the boundary (not at every layer).
- Nest: pino/winston, `AsyncLocalStorage` (`nestjs-cls`) for request context.
  Django: `logging` config dicts, contextvars. Spring: SLF4J/Logback, MDC (thread-local, and the
  reactive/virtual-thread caveat).
- Q: "How do you attach a request id to every log line in an async call stack without passing it
  everywhere?" (AsyncLocalStorage / contextvars / MDC — and what breaks it).

#### F09 · Persistence layer integration
`I` · Requires: F04, DB02 · Unlocks: F10, F13, DB40
- Concept: repository pattern, mapping rows to objects, migrations owned by the app, connection
  pool owned by the framework, query composition, avoiding leaking ORM entities into controllers.
- Nest: TypeORM (DataSource, repositories, entities, `synchronize: true` is a production footgun) or
  Prisma (schema-first, generated client, migrations, no lazy loading, `$transaction`) or Drizzle/
  Kysely (SQL-first). Django: models + queryset API + built-in migrations (`makemigrations` is not
  the same as a safe migration!). Spring: Spring Data JPA repositories, derived query methods,
  `@Query`, JPQL vs native, `EntityManager`; or jOOQ/JdbcTemplate.
- Q: "Repository pattern with an ORM that already is a repository — worth it?" (yes at the boundary
  of the domain; no as ceremony).
- Q: "TypeORM vs Prisma, honestly." (Prisma: better DX/typing/migrations, weaker for complex SQL and
  long transactions historically; TypeORM: flexible, buggy edge cases, active-record vs data-mapper).

#### F10 · Transactions in the framework
`A` · Requires: F09, DB06 · Unlocks: F13, M15
- Concept: transaction boundary = one use case, started in the application/service layer; propagation
  (REQUIRED, REQUIRES_NEW, NESTED/savepoints); read-only transactions; no network calls inside;
  passing the transaction handle without polluting every signature.
- Nest: TypeORM `QueryRunner`/`dataSource.transaction()`, Prisma `$transaction` (interactive
  transactions and their timeout), transaction propagation via `AsyncLocalStorage`
  (`@Transactional` decorators are not built in — a known gap vs Spring).
  Django: `transaction.atomic()` (nested = savepoints), `on_commit` hooks (the right place to
  enqueue jobs), autocommit default. Spring: `@Transactional` — proxy-based, so **self-invocation
  does not open a transaction**, `rollbackFor` only unchecked exceptions by default, and
  `@Transactional` on a private method does nothing. These three are classic interview traps.
- Q: "Why does calling `this.doWork()` from another method in the same Spring bean skip the
  transaction?"
- Q: "You enqueue a job inside a transaction and the worker can't find the row. Explain." (worker
  read before commit → enqueue in `on_commit` / after commit / use the outbox).

#### F11 · Authentication & authorisation in the framework
`I` · Requires: F03, S02 · Unlocks: S05, F26
- Concept: authenticate at the edge, authorise at the resource; principal in request context;
  declarative permission checks; object-level (not just route-level) authorisation.
- Nest: Guards + Passport strategies (`JwtStrategy`), `@UseGuards`, custom `@Roles()` decorator +
  `Reflector`, `CanActivate`. Django: auth middleware, `request.user`, permissions/DRF permission
  classes, object permissions. Spring: Spring Security filter chain, `SecurityFilterChain` bean,
  `@PreAuthorize("hasRole()")`/`@PostAuthorize`, method security, `SecurityContextHolder`.
- Q: "Route guard passes but the user reads another tenant's record. What was missing?" (object-level
  authz / row scoping — IDOR).

#### F12 · Testing
`I` · Requires: F04 · Unlocks: F18, M35, O08
- Concept: pyramid (many unit, fewer integration, few e2e); DI makes unit tests possible; test
  doubles (stub/mock/fake/spy) and over-mocking as a smell; integration tests against a **real**
  database via testcontainers; deterministic time and ids; fixtures/factories; flaky test causes.
- Nest: `Test.createTestingModule()`, `overrideProvider`, supertest for e2e, `@nestjs/testing`.
  Django: `TestCase` (transaction-wrapped, rolled back), `pytest-django`, factory_boy, `Client`.
  Spring: `@SpringBootTest` vs slices (`@WebMvcTest`, `@DataJpaTest`), MockMvc, `@MockBean`,
  Testcontainers, `@Transactional` rollback in tests (and why it hides commit-time bugs).
- Q: "How do you test a service that calls Stripe and writes to Postgres?"
- Q: "Your test suite rolls back transactions — what class of bug does that hide?" (constraint checks
  deferred to commit, `on_commit` hooks, trigger behaviour).

#### F13 · Background jobs & async work
`A` · Requires: F09, F10, Q10 · Unlocks: F14, Q21, SD10
- Concept: get work off the request path; at-least-once delivery so jobs must be **idempotent**;
  retries with backoff, max attempts, DLQ; job payload should carry an id, not a whole object
  (stale data); visibility timeout; isolate worker fleet from the API fleet; graceful shutdown to
  finish in-flight jobs.
- Nest: BullMQ (`@nestjs/bullmq`) on Redis, processors, `@Processor`/`@Process`, concurrency,
  repeatable jobs; or `@nestjs/microservices` with Kafka/RabbitMQ transports.
  Django: Celery (broker + result backend, `acks_late`, `task_acks_on_failure_or_timeout`,
  prefetch multiplier), or Django-Q/RQ. Spring: `@Async` + `TaskExecutor`, Spring Batch for chunked
  jobs, `@KafkaListener`, Quartz.
- Q: "A job runs twice. Is that a bug?" (no — design for it; then show the dedup strategy).
- Q: "Job payload holds the full user object and the email goes out with the old address. Fix?"

#### F14 · Scheduling & cron
`I` · Requires: F13 · Unlocks: Q21
- Concept: in-process schedulers break with >1 replica (every pod fires) → distributed lock, leader
  election, or an external scheduler (K8s CronJob, Temporal); missed runs after downtime; timezone
  and DST for business schedules; overlapping long runs.
- Nest: `@nestjs/schedule` `@Cron` + a Redis lock. Django: `celery beat` (single beat process or
  `RedBeat`). Spring: `@Scheduled` + ShedLock.
- Q: "You scaled to 3 pods and the nightly report is now sent 3 times. Fix it three ways."

#### F15 · Caching at the framework layer
`I` · Requires: F03, Q02 · Unlocks: SD05
- Concept: HTTP caching vs application cache vs query cache; where the cache decorator lies to you
  (per-instance memory cache with N pods); cache keys must include tenant/user/locale/version;
  invalidation on write.
- Nest: `CacheModule` + `CacheInterceptor` (+ Redis store), custom key factories.
  Django: cache framework, `cache_page`, per-view/low-level, `cached_property`.
  Spring: `@Cacheable/@CacheEvict/@CachePut`, cache manager abstraction, Caffeine + Redis two-level.
- Q: "`@Cacheable` on a method that takes a user id — what's the risk in a multi-tenant app?"

#### F16 · Realtime: WebSockets, SSE, long polling
`A` · Requires: F03, A16 · Unlocks: SD12, C16
- Concept: stateful connections break horizontal scaling (sticky routing or a shared pub/sub
  backplane), auth on the handshake and re-auth on token expiry, heartbeats, reconnect with
  resume/cursor, backpressure to a slow client, connection limits and memory per connection.
- Nest: `@WebSocketGateway`, socket.io vs ws adapter, Redis adapter for multi-instance.
  Django: Channels + ASGI + channel layer (Redis). Spring: STOMP over WebSocket, `@MessageMapping`,
  external broker relay; or SSE with `SseEmitter`/`Flux`.
- Q: "Chat app scaled to 5 pods; users in different pods can't see each other's messages. Fix."
- Q: "SSE vs WebSocket for a live dashboard?" (SSE: simpler, HTTP, auto-reconnect, one-way).

#### F17 · File upload, download, streaming
`I` · Requires: F02, C19 · Unlocks: S14
- Concept: never buffer large uploads in app memory; **presigned S3 URLs** so bytes bypass your API;
  multipart parsing limits, content-type sniffing and extension validation, virus scanning,
  streaming responses and range requests, generated download links with expiry.
- Nest: `FileInterceptor`/multer limits, `StreamableFile`. Django: `FileField`/storages, chunked
  upload handlers. Spring: `MultipartFile`, `StreamingResponseBody`, `Resource`.
- Q: "Users upload 2GB videos. Design the flow." (presigned upload → S3 event → queue → worker
  transcodes → callback; API never touches the bytes).

#### F18 · Application architecture inside the service
`A` · Requires: F04, F05, F12 · Unlocks: F19, F21, F22, F27
- Concept: layered vs hexagonal/ports-and-adapters vs clean architecture; the dependency rule
  (domain depends on nothing); module boundaries inside a monolith (modular monolith); anaemic vs
  rich domain model; where mapping lives; pragmatism — do not put four layers around a CRUD endpoint.
- Nest: modules as the boundary, `exports` as the public API, feature modules, shared module smell,
  domain/application/infrastructure folder split. Django: apps, fat models vs service layer debate.
  Spring: packages by feature not by layer, `@Service` as the application layer.
- Q: "Show me the folder structure of a service you'd build and defend each boundary."
- Q: "How do you stop a modular monolith from decaying into a big ball of mud?" (enforced import
  rules/linting, module public API, one owner per module, no shared tables).

#### F19 · NestJS deep dive
`A` · Requires: F04, F18 · Unlocks: F20, F28
- Key: module system (`imports/providers/controllers/exports`), **dynamic modules**
  (`forRoot`/`forRootAsync`, `ConfigurableModuleBuilder`), global modules, provider **scopes**
  (`DEFAULT` singleton, `REQUEST` — bubbles up the whole injection chain and hurts performance,
  `TRANSIENT`), `@Inject(TOKEN)`, lifecycle hooks (`onModuleInit`, `onApplicationBootstrap`,
  `onModuleDestroy`, `beforeApplicationShutdown`, `enableShutdownHooks`), custom decorators +
  `Reflector`, pipes/guards/interceptors/filters at global/controller/handler scope,
  `@nestjs/cqrs`, `@nestjs/microservices` transports, `@nestjs/swagger`.
- Q: "What does `Scope.REQUEST` do to the rest of your injection graph and to throughput?"
- Q: "How do you write a module that other teams configure asynchronously?" (`forRootAsync` with
  `useFactory` + `inject`).
- Q: "How do you implement multi-tenancy in Nest?" (request-scoped tenant context via ALS, not
  request-scoped providers everywhere).

#### F20 · NestJS internals
`X` · Requires: F19 · Unlocks: —
- Key: decorators + `reflect-metadata` + `emitDecoratorMetadata` produce the DI metadata; the IoC
  container resolves the dependency graph at bootstrap (which is why circular imports explode at
  startup, not runtime); the platform adapter abstraction (Express vs **Fastify**, and what you gain/
  lose); `ExecutionContext` abstraction over HTTP/WS/RPC; `APP_GUARD`-style global provider tokens;
  why Nest is a framework *over* Express rather than a server.
- Q: "How does Nest know the types to inject if TypeScript types are erased?"
- Q: "When would you switch the adapter to Fastify and what breaks?" (Express-specific middleware,
  `@Res()` usage, some libs; gain ~2x throughput on JSON-heavy paths).

#### F21 · Django deep dive (your cross-training target)
`A` · Requires: F18, DB40 · Unlocks: F23, F27
- Key: MTV, the ORM's **lazy querysets** (evaluated on iteration/slicing/len — a common perf bug),
  `select_related` (SQL join, FK/one-to-one) vs `prefetch_related` (second query + Python join,
  M2M/reverse FK), `only`/`defer`/`values`, `annotate`/`aggregate`, `F()`/`Q()` expressions,
  `bulk_create/bulk_update`, `iterator()` + `chunk_size` for large scans, `select_for_update`;
  migrations (autogenerated, `RunPython`, `atomic = False` for concurrent index creation);
  middleware; signals (and why they are a maintainability trap); DRF (serializers, viewsets,
  routers, permissions, throttling, pagination); WSGI vs **ASGI**, gunicorn workers vs uvicorn,
  the sync/async ORM boundary (`sync_to_async`); Celery; the admin as a genuine advantage;
  settings/secrets; "batteries included" vs Nest's assembly-required.
- Q: "`select_related` vs `prefetch_related` — show the SQL each generates."
- Q: "Django is 'sync-first'. How do you serve 5k concurrent connections?" (ASGI + async views, or
  more processes; explain the GIL and worker model — see C07).
- Q: "Coming from Nest, what would you miss in Django and what would you gain?" (miss: DI, typed
  DTOs at the boundary, first-class module structure; gain: ORM+migrations+admin+auth out of the
  box, batteries, less wiring, mature ecosystem).

#### F22 · Spring Boot deep dive (your cross-training target)
`A` · Requires: F18, DB40 · Unlocks: F23, F27
- Key: **auto-configuration** (`@EnableAutoConfiguration`, `@ConditionalOnClass/OnMissingBean`,
  starters, `spring.factories`/`AutoConfiguration.imports`) and how to debug it
  (`--debug` condition report); bean lifecycle and scopes; **AOP proxies** (JDK dynamic proxy vs
  CGLIB) which explains the `@Transactional`/`@Async`/`@Cacheable` self-invocation trap;
  Spring Data JPA + Hibernate (persistence context, dirty checking, `LazyInitializationException`,
  `open-in-view` — turn it off and know why, N+1 and `@EntityGraph`/`join fetch`, batch size,
  optimistic locking with `@Version`); Spring Security filter chain; Spring MVC (thread per request)
  vs **WebFlux** (reactive, Netty, Project Reactor, non-blocking all the way or it is pointless) vs
  **virtual threads** (Java 21+ — largely removes the reason to use WebFlux for I/O-bound apps);
  Actuator (health, metrics, Micrometer); GraalVM native image + AOT.
- Q: "Why did adding `@Transactional` to a method change nothing?" (self-invocation / private / not
  a Spring-managed bean).
- Q: "`open-in-view = true` — what does it do and why is it a production problem?" (session held for
  the whole request → connection held during rendering + lazy loads at serialisation time).
- Q: "WebFlux or virtual threads for a new I/O-bound service in 2026? Defend it."

#### F23 · Awareness of other stacks
`I` · Requires: F21, F22 · Unlocks: F27
- Key: **Go** (net/http, goroutines + channels, context cancellation, no DI culture, explicit errors,
  single static binary, fast cold start — ideal for infra services and high-concurrency proxies);
  **FastAPI** (pydantic validation, async-first, OpenAPI generated, thin — closest Python analogue to
  Nest's DTO ergonomics); **Rails/Laravel** (convention over configuration, fastest CRUD velocity,
  ActiveRecord pattern); **.NET** (ASP.NET Core minimal APIs, first-class DI and async, very strong
  performance); **Express/Fastify** raw; **Elixir/Phoenix** (BEAM, actors, LiveView, exceptional at
  massive concurrent connections).
- Q: "You're starting a new high-throughput service tomorrow. Node, Go, Java or Python — pick and
  defend." (the right answer names team expertise and operational maturity first, then the workload
  profile: CPU-bound → Go/Java; I/O fan-out → Node/Go; ML/data adjacency → Python; strict domain
  modelling and long-lived enterprise → Java/.NET).

#### F24 · Framework-level performance & resource management
`A` · Requires: F09, DB21, C03 · Unlocks: F28, C14
- Key: connection pool sizing vs worker count vs DB max connections (multiply by pod count!);
  HTTP keep-alive and agent maxSockets; response compression (and when it is wasted on already-
  compressed payloads); JSON serialisation cost at scale (it is often the top CPU consumer in Node);
  avoiding synchronous work on the event loop; DTO mapping cost; N+1 at the HTTP level; payload size
  and pagination as a performance feature; `--max-old-space-size`/JVM heap vs container limit.
- Q: "Your Nest service holds p99 at 2s under load but CPU is 40%. Where do you look?" (event loop
  lag, pool exhaustion, downstream latency, GC pauses, queueing — see C14).

#### F25 · API documentation & codegen
`I` · Requires: F02, F06 · Unlocks: A22
- Key: OpenAPI generated from code (Nest Swagger decorators, drf-spectacular, springdoc) vs
  spec-first with generated servers/clients; keeping docs true (contract tests, CI diff);
  client SDK generation; examples and error documentation; gRPC/proto as the spec instead.
- Q: "Code-first or spec-first OpenAPI? When does spec-first win?" (multiple teams/consumers, public
  API, parallel frontend work).

#### F26 · Web security in the framework
`A` · Requires: F03, F11, S06 · Unlocks: S06, S08
- Key: CORS configured correctly (not `*` with credentials), CSRF (only relevant for cookie auth —
  explain why token-in-header is immune), security headers (helmet/CSP/HSTS), mass assignment
  (whitelist DTOs), SSRF on user-supplied URLs, rate limiting (`@nestjs/throttler`, DRF throttling,
  bucket4j), safe file handling, ORM injection escape hatches (raw queries with interpolation),
  dependency and `DEBUG=False`/stack-trace hygiene.
- Q: "You use JWT in an `Authorization` header. Do you need CSRF protection? Why/why not?"
- Q: "Your endpoint fetches a user-supplied URL for link previews. What must you defend against?"

#### F27 · Choosing & migrating stacks
`A` · Requires: F18, F21, F22, F23 · Unlocks: SD15
- Key: decision criteria in order — team skill and hiring pool, ecosystem for your domain,
  operational maturity (observability, deployment, libraries), performance profile, long-term
  maintenance; polyglot cost is real (tooling, on-call, security patching x N); migration by
  strangler fig, service by service, never a rewrite; how to become productive in a new framework
  fast (map the concepts you already own — this file's structure is the answer).
- Q: "We're a Node shop considering Go for a new service. Walk me through the decision."
- Q: "How long until you're productive in Spring Boot?" (honest, structured answer: the concepts
  transfer — DI, middleware chain, ORM, transactions — the syntax and the ecosystem are the cost;
  name what you'd read first).

#### F28 · Runtime & deployment model of the framework
`A` · Requires: F01, F19, F24 · Unlocks: O03, O06, C03, C07
- Key: process model — Node single-threaded per process + `cluster`/multiple pods; Python
  gunicorn workers (sync vs gevent vs uvicorn) and the GIL; JVM one thread per request (MVC) with a
  large heap and slow start; cold start and JIT warm-up; **graceful shutdown** (stop accepting,
  drain in-flight, close pool, honour `SIGTERM` and `terminationGracePeriodSeconds`); health vs
  readiness endpoints; statelessness as a requirement for horizontal scaling; 12-factor.
- Q: "Should you run Node `cluster` inside a container, or more containers?" (usually more
  containers — the orchestrator already does it and gets you isolation; cluster wins on
  per-pod fixed overhead and a large machine).
- Q: "Your pods drop requests on every deploy. Fix it end to end." (SIGTERM handler → readiness
  false → preStop sleep → drain keep-alive connections → close DB pool → exit).

---

## Topological order (study waves)

```
Wave 0  F01
Wave 1  F02  F03  F04  F05
Wave 2  F06  F09  F11  F12  F15  F17
Wave 3  F07  F10  F16  F18  F25  F26
Wave 4  F08  F13  F19  F21  F22  F24
Wave 5  F14  F20  F23  F28
Wave 6  F27
```

Cross-field parents: `A01` HTTP, `A16` realtime protocols, `DB02` SQL, `DB06` transactions,
`DB21` pooling, `DB40` ORM internals, `Q02` caching strategies, `Q10` queues, `C03/C07` runtimes,
`S02/S06` security.

**The five questions you will definitely be asked:** Nest pipeline order (F03); DI scopes and
why interfaces need tokens (F04/F19); N+1 and how your ORM causes it (DB40/F21); transaction
boundaries and the enqueue-before-commit bug (F10); graceful shutdown (F28).

**Cross-training plan (2 weeks, evenings):** build the *same* small service three times — a
`POST /orders` with validation, a transaction, an outbox row, a background job and one integration
test — in Nest, Django/DRF and Spring Boot. You will then be able to answer any comparative
question from experience rather than from reading.
