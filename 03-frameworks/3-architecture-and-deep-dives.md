[← back to the field index](README.md)

# Frameworks · Part 3 — Architecture & Framework Deep Dives

Nodes `F18`–`F23`. This is the part that answers "you know Nest — could you work in Django or
Spring?"

---

## F18 · Application architecture inside one service

`Advanced` · Requires: `F04`, `F05`, `F12` · Unlocks: `F19`, `F21`, `F22`, `F27`

### Preface

Even a single service needs internal structure, or it becomes a folder of files that all import each
other.

The most useful idea is the **dependency rule**: your business logic should not depend on the
framework, the database or the outside world. Those depend on it. Then you can test the logic without
booting anything, and replace the database without rewriting the rules.

The equally important idea is not to over-apply it. A CRUD endpoint wrapped in four layers of
abstraction is worse than a controller that queries a table.

### Details

#### 1. Layered, hexagonal, clean

**Theory.** **Layered** — controller → service → repository → database; simple and familiar, and the
domain usually ends up depending on the ORM. **Hexagonal (ports and adapters)** — the domain defines
interfaces (ports); adapters implement them for HTTP, the database, the message broker; dependencies
point inward. **Clean architecture** — the same idea with more prescribed layers (entities, use
cases, interface adapters, frameworks).

**Example.** Concretely, hexagonal means your `OrderService` depends on an
`OrderRepository` **interface** it defines, and the TypeORM implementation lives in an
infrastructure folder and is injected (`F04`). The service imports nothing from TypeORM, so a unit
test needs no database and a switch to Prisma touches one file.

**Advanced.** The honest cost is indirection: an extra interface, an extra mapping between the
persistence model and the domain model, and more files. It earns its keep when the domain logic is
genuinely complex, when the application will live for years, or when you expect the infrastructure to
change. For a thin CRUD service it is ceremony. Being able to say *when* rather than always is the
senior answer.

#### 2. Organise by feature, not by layer

**Theory.** Folders named `controllers/`, `services/`, `repositories/` scatter every feature across
the codebase. Folders named `orders/`, `payments/`, `catalogue/` keep a feature's code together and
make the boundaries visible.

**Example.**

```
src/
  orders/
    order.controller.ts
    order.service.ts
    order.repository.ts
    order.entity.ts
    dto/
    orders.module.ts
  payments/
  shared/
```

You can see the modules, and you can see when `orders` starts importing from inside `payments` —
which is exactly the signal you want.

**Advanced.** This is the modular monolith (`M01`): each folder is a module with a public interface
(what the module exports) and private internals. Enforce it with tooling — `eslint-plugin-boundaries`,
Nx module boundaries, ArchUnit in Java, `import-linter` in Python — so a violating import fails the
build. Conventions that are not enforced decay; that is the practical lesson worth stating.

#### 3. Rich versus anaemic domain models

**Theory.** An **anaemic** model is data classes with getters and setters, and all behaviour in
services. A **rich** model puts behaviour with the data: `order.cancel()` enforces the rules about
when cancelling is allowed.

**Example.** With a rich model, "an order cannot be cancelled after dispatch" lives in one place and
cannot be bypassed. With an anaemic model, the check lives in whichever service remembered it — and
the second service to cancel orders will forget. The rich model also reads better:
`order.addLine(product, qty)` versus `orderService.addLine(order, product, qty)`.

**Advanced.** The counterpoint: rich models fight ORMs. An entity with a private constructor,
invariants and no setters is awkward for a framework that wants to instantiate it from a row and
mutate it field by field. The usual resolution is separate persistence models and domain models with
mapping between them — more code, cleaner domain. Choose based on how much genuine business logic
there is; if your "domain" is CRUD with validation, an anaemic model plus services is honest and
fine.

#### 4. Keeping it from decaying

**Theory.** Structure erodes under deadline pressure unless something enforces it.

**Example.** What actually works: enforced import rules in CI; one owning team per module
(`CODEOWNERS`); a module's public surface declared explicitly (Nest `exports`, an `index.ts` that is
the only legal import path); no shared tables between modules (`M22`); and a periodic look at the
dependency graph to spot cycles. Cycles between modules are the clearest early sign of decay.

**Advanced.** The strongest signal that boundaries are real: you could extract a module into its own
service with a mechanical change — replacing function calls with HTTP or message calls — and nothing
else would need to change. If that is not true, the boundary is decorative. That test is also
exactly what makes a future split feasible (`M28`).

### Interview questions

- "Show me the folder structure of a service you would build and defend each boundary."
- "How do you stop a modular monolith from decaying into a big ball of mud?"
- "When is hexagonal architecture not worth it?"
- "What is an anaemic domain model, and is it always wrong?"

---

## F19 · NestJS deep dive

`Advanced` · Requires: `F04`, `F18` · Unlocks: `F20`, `F28`

### Preface

Nest is an opinionated layer over Express or Fastify that brings Angular-style modules and dependency
injection to the backend. Its value is structure and consistency; its cost is indirection and a fair
amount of framework-specific knowledge.

Since this is your primary stack, expect questions to go deeper here than anywhere else: modules,
scopes, dynamic modules, lifecycle hooks and the execution pipeline.

### Details

#### 1. Modules and their public surface

**Theory.** A module groups related providers and controllers. `providers` registers them;
`exports` declares which are visible to modules that import this one; `imports` brings in other
modules. A provider not exported is private, which is the mechanism for module boundaries (`F18`).

**Example.**

```ts
@Module({
  imports: [TypeOrmModule.forFeature([Order]), PaymentModule],
  controllers: [OrderController],
  providers: [OrderService, OrderRepository],
  exports: [OrderService],          // only this is visible to importers
})
export class OrderModule {}
```

`@Global()` makes a module's exports available everywhere without importing — convenient for
configuration and logging, and a boundary-destroyer if used for anything else.

**Advanced.** Circular module imports need `forwardRef` on both sides and usually indicate that the
modules should be merged, or that one should publish an event instead of calling the other (`F04`).
Because Nest resolves the graph at bootstrap, these failures are startup errors — read the
"Nest can't resolve dependencies of X (?, Y)" message carefully: the `?` marks the parameter that
could not be resolved, which is nearly always a missing `exports` or a missing provider.

#### 2. Dynamic modules

**Theory.** A module that is configured by its consumer. `forRoot()` configures it once for the
application; `forFeature()` configures a slice; `forRootAsync()` allows the configuration to be
computed asynchronously, typically from injected config.

**Example.**

```ts
@Module({})
export class PaymentModule {
  static forRootAsync(options: AsyncOptions): DynamicModule {
    return {
      module: PaymentModule,
      imports: options.imports,
      providers: [
        { provide: PAYMENT_OPTIONS, useFactory: options.useFactory, inject: options.inject },
        { provide: PAYMENT_GATEWAY, useClass: StripeGateway },
      ],
      exports: [PAYMENT_GATEWAY],
    };
  }
}
```

This is how every Nest library you use is built — `ConfigModule.forRoot()`,
`TypeOrmModule.forRootAsync()`, `BullModule.forRoot()`. `ConfigurableModuleBuilder` generates most of
this boilerplate for you in recent versions.

**Advanced.** Know why `forRootAsync` exists: `forRoot` needs its values at module-definition time,
which is before `ConfigService` exists. The async form defers construction until the dependency graph
can supply the configuration. It is a good example of DI ordering constraints becoming a visible API
shape.

#### 3. Provider scopes

**Theory.** `Scope.DEFAULT` — singleton, one per application, and the right choice almost always.
`Scope.REQUEST` — a new instance per request. `Scope.TRANSIENT` — a new instance per consumer.

**Example.** The cost of request scope is the important part: a request-scoped provider makes
everything that depends on it request-scoped too, **up the whole chain**, including the controller.
Nest must then instantiate that subtree per request, which measurably reduces throughput and
increases garbage collection. It also means those providers cannot be injected into anything that is
constructed once, such as a global interceptor registered outside the container.

**Advanced.** For request data — the current user, the tenant, the trace id — use `AsyncLocalStorage`
(`nestjs-cls`) instead: everything stays a singleton and reads the ambient context. This is the
recommended pattern for multi-tenancy in Nest and is a strong answer, because it shows you know the
performance consequence rather than reaching for the obvious feature.

#### 4. Lifecycle hooks and shutdown

**Theory.** In order: `onModuleInit` → `onApplicationBootstrap` → (running) → `onModuleDestroy` →
`beforeApplicationShutdown` → `onApplicationShutdown`. Shutdown hooks only run if you call
`app.enableShutdownHooks()`.

**Example.** What belongs where: `onModuleInit` for work needing this module's dependencies
(warming a cache, connecting a client); `onApplicationBootstrap` for work needing the whole
application to be ready (starting a consumer); `onModuleDestroy` for stopping consumers and closing
pools. The correct shutdown order matters — stop accepting work before closing the resources that
in-flight work needs (`F28`).

**Advanced.** `enableShutdownHooks()` is off by default and its absence is the most common cause of
dropped requests and unacknowledged jobs during a deploy. Note also that it registers process signal
listeners, which has a small performance cost in Node — that is why it is opt-in. Turn it on in every
production service.

#### 5. The rest of the surface worth knowing

**Theory.** Custom decorators with `createParamDecorator` and `SetMetadata` plus `Reflector`;
global providers via the `APP_GUARD`, `APP_INTERCEPTOR`, `APP_FILTER`, `APP_PIPE` tokens (which give
you DI, unlike `app.useGlobalX`); `@nestjs/cqrs` for command and query buses;
`@nestjs/microservices` for Kafka, RabbitMQ, gRPC and TCP transports; `@nestjs/swagger` for
generated OpenAPI (`F25`).

**Example.** A typical custom decorator:

```ts
export const CurrentUser = createParamDecorator(
  (_: unknown, ctx: ExecutionContext) => ctx.switchToHttp().getRequest().user,
);
```

Note `ExecutionContext` — the abstraction that lets the same guard or interceptor work for HTTP,
WebSockets and RPC (`F20`).

**Advanced.** `@nestjs/cqrs` provides a command bus, query bus and event bus in-process. It is useful
for structure and easy to over-apply — a bus for a CRUD service adds indirection and no benefit.
Reach for it when you have genuinely complex flows with many handlers, or when you want an in-process
event bus as a stepping stone toward extracting services later (`M28`).

### Interview questions

- "What does `Scope.REQUEST` do to the rest of your injection graph and to throughput?"
- "How do you write a module that other teams configure asynchronously?"
- "How do you implement multi-tenancy in Nest?"
- "Why register a global guard with `APP_GUARD` instead of `useGlobalGuards`?"

---

## F20 · NestJS internals

`Expert` · Requires: `F19` · Unlocks: —

### Preface

Knowing how Nest works underneath explains several behaviours that otherwise look like magic: how it
knows what to inject given that TypeScript types disappear at compile time, why circular imports
explode at startup, and what changes when you swap Express for Fastify.

### Details

#### 1. Decorators and reflect-metadata

**Theory.** TypeScript's `emitDecoratorMetadata` makes the compiler emit the **runtime types** of a
decorated class's constructor parameters, stored via the `reflect-metadata` library under
`design:paramtypes`. Nest reads that array to know what to inject.

**Example.** This is exactly why interfaces cannot be injected (`F04`): an interface has no runtime
representation, so `design:paramtypes` contains `Object` and Nest cannot resolve it. Classes and
abstract classes work because they are real runtime values. It is also why `reflect-metadata` must be
imported once at the entry point and why those two `tsconfig` flags are mandatory.

**Advanced.** This ties Nest to the **legacy** TypeScript decorator implementation
(`experimentalDecorators`), because the standardised ECMAScript decorators do not emit type metadata.
That is a real constraint on the framework's evolution and a good thing to know if someone asks about
Nest's future or about SWC/esbuild build setups, which need explicit plugin support for this
metadata.

#### 2. The IoC container at bootstrap

**Theory.** At startup Nest scans modules, builds a dependency graph of every provider, and
instantiates them in dependency order. Everything is resolved before the server starts listening.

**Example.** The practical consequences: dependency errors are **startup** failures, not runtime
ones — good, because a misconfigured deployment crashes immediately and never receives traffic. And
circular dependencies cannot be lazily resolved, so they must be broken explicitly with
`forwardRef`. It also means a large application has a measurable bootstrap time, which matters for
serverless cold starts (`O12`).

**Advanced.** `ModuleRef` lets you resolve providers imperatively at runtime (`moduleRef.get(Token)`,
or `moduleRef.resolve()` for scoped providers). It is the escape hatch for genuinely dynamic
resolution — choosing a strategy by name at runtime — and it is also a way to hide dependencies from
the constructor, which makes them invisible to readers and to tests. Use it deliberately and rarely.

#### 3. The platform adapter

**Theory.** Nest does not implement HTTP. `HttpAdapter` abstracts Express or Fastify, so Nest's
routing, pipes and filters work on either.

**Example.** Switching to Fastify (`NestFactory.create(AppModule, new FastifyAdapter())`) typically
gives roughly twice the throughput on JSON-heavy endpoints — mainly from Fastify's schema-compiled
serialisation and faster routing. What breaks: Express-specific middleware, code using `@Res()` with
Express APIs, and some ecosystem packages that assume Express request and response objects.

**Advanced.** The `@Res()` decorator is worth knowing precisely: using it puts Nest into "manual
response mode", so interceptors that transform the response, and the serialisation layer, are
bypassed. That surprises people who add `@Res()` just to set a header — use `@Res({ passthrough:
true })` for that, so Nest still handles the response. This one detail causes a disproportionate
number of "my interceptor is not running" questions.

#### 4. ExecutionContext and transport independence

**Theory.** `ExecutionContext` abstracts over transports so the same guard, interceptor or filter can
run for an HTTP request, a WebSocket message or a microservice event.

**Example.** `ctx.getType()` returns `'http' | 'ws' | 'rpc'`, and you call `switchToHttp()`,
`switchToWs()` or `switchToRpc()` to get the underlying objects. A global logging interceptor that
assumes HTTP will throw when a Kafka message arrives — so check the type, or register transport-
specific components.

**Advanced.** This abstraction is what allows `@nestjs/microservices` to reuse the whole pipeline for
message handlers, which is genuinely useful: the same validation pipes and exception filters apply to
a Kafka consumer as to a controller. It is also the reason Nest's abstractions occasionally feel
indirect — they are designed for more than HTTP.

### Interview questions

- "How does Nest know the types to inject if TypeScript types are erased?"
- "When would you switch the adapter to Fastify and what breaks?"
- "Why did adding `@Res()` stop my interceptor from running?"
- "Why are Nest dependency errors startup errors rather than runtime errors?"

---

## F21 · Django deep dive

`Advanced` · Requires: `F18`, `DB40` · Unlocks: `F23`, `F27`

### Preface

Django's philosophy is the opposite of Nest's: batteries included rather than assembly required. You
get an ORM, migrations, an admin interface, authentication, sessions, forms and security defaults on
day one, with very little wiring.

What you give up is a dependency injection container, compile-time types, and explicit module
boundaries. Coming from Nest, the ORM's laziness and the sync/async boundary are the two things that
will actually catch you out.

### Details

#### 1. The ORM and lazy querysets

**Theory.** A queryset is lazy: building it issues no SQL. It is evaluated when you iterate it, slice
it with a step, call `len()`, `list()`, `bool()` or `repr()`. Querysets are also **chainable** and
cache their results after evaluation.

**Example.** The consequences are frequently surprising:

```python
qs = Order.objects.filter(status='open')    # no query yet
if qs:                                      # query runs — fetches ALL rows
    print(len(qs))                          # cached, no second query
print(qs.count())                           # a NEW query: SELECT COUNT(*)
```

Use `qs.exists()` rather than `if qs` when you only need to know whether anything matches, and
`qs.count()` rather than `len(qs)` when you only need the number.

**Advanced.** `qs.iterator(chunk_size=2000)` disables the result cache and streams with a server-side
cursor, which is the correct way to loop over a million rows without loading them into memory. Also
know `only()`/`defer()` for narrowing columns, `values()`/`values_list()` to skip model instantiation
entirely (much faster for read-only work), and `bulk_create`/`bulk_update` for batched writes
(`DB20`).

#### 2. select_related versus prefetch_related

**Theory.** Both exist to avoid N+1 (`DB40`), and they work differently.
`select_related` performs a **SQL join** and is for forward foreign keys and one-to-one relations.
`prefetch_related` runs a **second query** with an `IN` clause and joins in Python; it is for
many-to-many and reverse foreign keys, where a join would multiply rows.

**Example.**

```python
# one query with a JOIN
Order.objects.select_related('customer')

# two queries: orders, then all their items in one IN query
Order.objects.prefetch_related('items')

# combine, and narrow the prefetch
Order.objects.select_related('customer').prefetch_related(
    Prefetch('items', queryset=OrderItem.objects.select_related('product'))
)
```

**Advanced.** Choosing between them is the same trade as `DB20`: a join returns duplicated parent
columns for every child row, so 100 orders x 50 items is 5,000 wide rows; two queries return 100 +
5,000 narrow rows. For one-to-one and small forward relations, join. For collections, prefetch. Being
able to explain the row multiplication is what makes this a senior answer rather than a memorised
rule.

#### 3. Sync, async, WSGI and ASGI

**Theory.** Django was synchronous for most of its life. It now supports async views under **ASGI**,
but the ORM is still fundamentally synchronous — async ORM methods exist (`aget`, `acreate`,
`afilter`) and wrap the sync implementation in a thread.

**Example.** The trap: calling a synchronous ORM method inside an async view raises
`SynchronousOnlyOperation`, and calling a blocking library (like `requests`) inside an async view
silently blocks the event loop and ruins concurrency for everything on that worker (`C07`). Use
`sync_to_async` for the former and an async HTTP client (`httpx`) for the latter.

**Advanced.** Deployment shapes differ accordingly: gunicorn with sync workers (process per request,
simple, memory-hungry), gunicorn with gevent (monkey-patched cooperative concurrency, effective and
occasionally surprising), or uvicorn with ASGI for genuinely async code. For a mostly-synchronous
Django application, more processes is usually the right scaling answer rather than partial async
adoption — and saying that plainly is better than claiming async everywhere.

#### 4. The rest of the framework

**Theory.** Migrations are generated from model diffs and are Python, so data migrations
(`RunPython`) live alongside schema changes. Signals (`post_save`, `pre_delete`) provide hooks.
Django REST Framework adds serializers, viewsets, routers, permissions, throttling and pagination.
The admin gives you a working back office for free.

**Example.** The admin is a genuine, underrated advantage: an internal tool that would take a
sprint in Nest exists automatically and is customisable. For internal operations teams this is often
the single biggest reason to choose Django.

**Advanced.** **Signals are the maintainability trap.** They create action at a distance: saving a
model triggers code in a file you have never opened, they make tests slow and surprising, and they
execute inside the save's transaction whether you wanted that or not. Prefer explicit service
functions; keep signals for genuinely cross-cutting concerns. When asked "what would you change about
Django", overuse of signals is a credible, experienced answer.

### Interview questions

- "`select_related` versus `prefetch_related` — show the SQL each generates."
- "Django is sync-first. How do you serve 5,000 concurrent connections?"
- "Coming from Nest, what would you miss in Django and what would you gain?"
- "Why are signals a maintainability risk?"

---

## F22 · Spring Boot deep dive

`Advanced` · Requires: `F18`, `DB40` · Unlocks: `F23`, `F27`

### Preface

Spring Boot is the enterprise-grade end of this spectrum: a mature dependency injection container,
auto-configuration that wires sensible defaults from what is on the classpath, and an enormous
ecosystem.

Almost every famous Spring gotcha comes from one fact: **the annotations work by wrapping your bean
in a proxy.** Understand the proxy and you can answer four separate interview questions with one
explanation.

### Details

#### 1. Auto-configuration

**Theory.** Spring Boot inspects the classpath and the existing beans, and configures what is
missing. Add a JDBC driver and a datasource URL, and you get a configured `DataSource`, a
`JdbcTemplate` and a transaction manager, with no XML and no code.

**Example.** The mechanism is `@Conditional` annotations: `@ConditionalOnClass`,
`@ConditionalOnMissingBean`, `@ConditionalOnProperty`. Auto-configuration classes are listed in
`META-INF/spring/...AutoConfiguration.imports` (formerly `spring.factories`). Defining your own bean
of the same type disables the automatic one, which is how you override anything.

**Advanced.** When it does something unexpected, run with `--debug` to print the **condition
evaluation report**: every auto-configuration considered, applied or skipped, and why. That single
tool converts "Spring magic" into a readable list, and mentioning it signals real experience rather
than tutorial familiarity.

#### 2. Proxies — the origin of the famous traps

**Theory.** `@Transactional`, `@Async`, `@Cacheable`, `@PreAuthorize` and `@Retryable` are all
implemented by wrapping the bean in a proxy (a JDK dynamic proxy if it implements an interface, a
CGLIB subclass otherwise). The proxy runs the extra behaviour and then calls your method.

**Example.** Therefore, all of the following silently do nothing:
- calling the annotated method from **another method of the same class** (self-invocation — the call
  does not go through the proxy);
- annotating a **private** or `final` method (the proxy cannot intercept it);
- calling a method on an object you created with `new` rather than one injected by Spring.

And `@Transactional` rolls back only on unchecked exceptions unless you set `rollbackFor`.

**Advanced.** The fixes: inject the bean into itself, extract the method into a separate bean (the
cleanest), or use `TransactionTemplate` programmatically. Being able to say "these four annotations
fail the same way for the same reason" is a much stronger answer than knowing one of them (`F10`).

#### 3. JPA and Hibernate

**Theory.** The **persistence context** is an identity map and a unit of work: managed entities are
tracked, changes are detected automatically at flush, and a lazily-loaded association triggers a
query on first access (`DB40`).

**Example.** The three problems that define working with Hibernate:
- **N+1** — fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.
- **`LazyInitializationException`** — accessing a lazy association after the session closed,
  typically during JSON serialisation in the controller.
- **`open-in-view`** — Spring Boot's default of `true` keeps the persistence context open for the
  whole request, which makes the previous problem disappear and is why it is on. It also holds a
  database connection for the entire request including view rendering, and triggers lazy queries from
  the serialisation layer where you cannot see them.

**Advanced.** The consensus recommendation is `spring.jpa.open-in-view=false`, and then fetching what
you need explicitly in the service layer and returning DTOs rather than entities. It exposes the
lazy-loading problems you already had, which is the point. Being able to explain *why* the default
exists and *why* you would still turn it off is exactly the level of nuance interviewers look for.

#### 4. MVC, WebFlux and virtual threads

**Theory.** **Spring MVC** is thread-per-request on a servlet container: simple, blocking, and
limited by thread count. **WebFlux** is reactive and non-blocking on Netty, using Project Reactor
(`Mono`, `Flux`) — high concurrency with few threads, and it only works if the **entire** chain is
non-blocking. **Virtual threads** (Java 21, `spring.threads.virtual.enabled=true`) make blocking code
cheap again by decoupling Java threads from OS threads (`C06`).

**Example.** The practical guidance for a new I/O-bound service in 2026: MVC with virtual threads.
You write straightforward blocking code, get reactive-level concurrency, keep stack traces and
debuggers that work, and avoid the reactive learning curve. WebFlux remains justified for streaming,
backpressure-sensitive pipelines, and teams already fluent in Reactor.

**Advanced.** Two caveats to know. A single blocking call in a WebFlux chain blocks an event loop
thread and destroys throughput — the same failure as blocking a Node event loop (`C03`), and the
reason "half-reactive" code performs worse than plain blocking code. And virtual threads **pin** to
their carrier thread inside `synchronized` blocks and native calls, so legacy code with synchronized
sections around I/O does not benefit until it is changed to use `ReentrantLock`.

### Interview questions

- "Why did adding `@Transactional` to a method change nothing?"
- "`open-in-view = true` — what does it do and why is it a production problem?"
- "WebFlux or virtual threads for a new I/O-bound service? Defend it."
- "How do you debug unexpected auto-configuration?"

---

## F23 · Awareness of other stacks

`Intermediate` · Requires: `F21`, `F22` · Unlocks: `F27`

### Preface

You will not be asked to write Go in a Node interview. You may well be asked "why would you choose Go
over Node for this service?", and a thoughtful answer signals that you choose tools rather than
defend habits.

What follows is enough to have that conversation credibly.

### Details

#### 1. Go

**Theory.** Compiled to a single static binary, starts in milliseconds, low and predictable memory
use. Concurrency via **goroutines** (cheap user-space threads) and channels, with a `context` for
cancellation and deadlines. Errors are explicit return values, not exceptions. No DI culture — you
wire dependencies by hand.

**Example.** Where it wins: infrastructure and network services (proxies, gateways, agents),
CPU-bound work that would block a Node event loop, anything needing very high concurrency with
predictable latency, and CLI tools. Kubernetes, Docker, Terraform and Prometheus are all Go for these
reasons.

**Advanced.** The honest trade: verbose error handling, a deliberately minimal standard approach to
abstraction, and less batteries-included framework support than Django or Spring — you assemble more
yourself. Goroutines are cheap but not free; leaking them (a goroutine blocked on a channel nobody
writes to) is Go's equivalent of a memory leak, and `context` cancellation is how you avoid it.

#### 2. FastAPI and modern Python

**Theory.** Async-first, built on Starlette and pydantic. Type hints drive validation, serialisation
and generated OpenAPI. It is the Python framework closest to the Nest experience: typed DTOs at the
boundary, dependency injection (via `Depends`), and generated docs.

**Example.** Choose it when the service needs the Python ecosystem — machine learning, data science,
scientific libraries — with a modern async HTTP layer. Choose Django instead when you want the ORM,
admin, auth and migrations as a package, especially for a product with a back office.

**Advanced.** FastAPI is deliberately thin: no ORM, no migrations, no admin. You assemble SQLAlchemy
plus Alembic yourself. That is the same assembly-required trade as Nest, and it means the comparison
"Nest is to Express as FastAPI is to Starlette" is a fair one to draw in conversation.

#### 3. Rails, Laravel and .NET

**Theory.** **Rails** and **Laravel** are convention-over-configuration frameworks optimised for
speed of delivery, both built on Active Record. **ASP.NET Core** has first-class dependency
injection and async, excellent performance, and strong typing — structurally the closest mainstream
framework to Nest.

**Example.** Rails remains extremely productive for CRUD-heavy products and is still the fastest way
to get a complete web application running. Its trade-offs are runtime performance and the looseness
that comes with heavy metaprogramming. .NET is a strong choice in organisations already on Microsoft
infrastructure and is genuinely fast.

**Advanced.** The pattern worth naming: **Active Record** (the model knows how to save itself — Rails,
Laravel, Django, TypeORM's active-record mode) versus **Data Mapper** (a separate layer maps objects
to rows — Hibernate, Doctrine, TypeORM's data-mapper mode). Active Record is faster to write and
couples the domain to the database; Data Mapper is more ceremony and keeps the domain clean
(`F18`). Recognising that one distinction lets you talk about half a dozen frameworks coherently.

#### 4. Elixir and the BEAM

**Theory.** Elixir runs on the Erlang VM, built for telecoms: millions of lightweight processes,
message passing, supervision trees that restart failed processes, and hot code reloading. Phoenix is
its web framework, with LiveView for server-rendered realtime UI.

**Example.** It is exceptional at massive numbers of concurrent stateful connections — chat,
presence, live dashboards, IoT. Discord and WhatsApp are the famous examples. The fault-tolerance
model ("let it crash", supervised restart) is a genuinely different way of thinking about resilience
that maps neatly onto the patterns in `M10`.

**Advanced.** The trade is a smaller ecosystem and hiring pool, and it is not fast at raw
number-crunching. Worth knowing as a reference point: when someone describes a system holding
millions of WebSocket connections, the BEAM is the platform designed for exactly that, and knowing
so is a useful signal even if you never write Elixir.

### Interview questions

- "You are starting a new high-throughput service tomorrow. Node, Go, Java or Python — pick and
  defend."
- "What is Active Record versus Data Mapper, and which do you prefer?"
- "When would Go beat Node for a backend service?"
- "Why is Elixir good at holding a million connections?"
