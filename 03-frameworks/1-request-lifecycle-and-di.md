[← back to the field index](README.md)

# Frameworks · Part 1 — Request Lifecycle, DI, Validation & Errors

Nodes `F01`–`F08`. Every node gives the concept first, then how NestJS, Django and Spring Boot each
express it — so you can answer a question about any of the three from one piece of understanding.

---

## F01 · What a framework is doing: the request lifecycle

`Beginner` · Requires: `A01` · Unlocks: `F02`, `F03`, `F28`

### Preface

A web framework sits between a socket and your function. It accepts a TCP connection, parses the
HTTP text into an object, decides which of your functions should handle it, runs some shared steps
before and after, and turns your return value back into an HTTP response.

Knowing this sequence lets you answer "where should this logic live?" and "why is this slow?"
without guessing — and it is the same sequence in every framework, with different names.

### Details

#### 1. From socket to handler

**Theory.** The steps: accept the connection → read bytes → parse the request line, headers and body
→ match a route → run the pre-processing chain → call your handler → serialise the result → write
the response → keep the connection alive or close it.

**Example.** The same sequence, three vocabularies:
- **Nest**: `NestFactory.create()` wraps an HTTP adapter (Express by default, Fastify optionally).
  The adapter parses; Nest's router matches; then middleware → guards → interceptors → pipes →
  your controller method.
- **Django**: the web server (gunicorn/uvicorn) calls a **WSGI or ASGI** callable; Django builds a
  `HttpRequest`, runs the middleware chain, resolves the URLconf to a view, calls it, and passes the
  `HttpResponse` back out through the middleware.
- **Spring**: the servlet container (Tomcat) hands the request to the `DispatcherServlet`, which
  consults `HandlerMapping` to find the controller, runs filters and interceptors, invokes the
  method through a `HandlerAdapter`, and converts the return value with an `HttpMessageConverter`.

**Advanced.** The framework is not the server. Nest does not listen on a socket — Express or Fastify
does. Django does not either — gunicorn or uvicorn does. This matters because several behaviours you
might attribute to the framework belong to the server: keep-alive handling, header size limits,
request timeouts, the number of worker processes, and how `SIGTERM` is handled during shutdown
(`F28`).

#### 2. Where the time goes

**Theory.** For a typical JSON API the time splits between: connection and TLS setup (avoided by
keep-alive), body parsing, routing (negligible), your handler (usually dominated by I/O to the
database or another service), and serialisation.

**Example.** A practical instinct: if a handler takes 200ms, it is almost never the framework. Look
at the database queries first (`DB20`), then downstream HTTP calls, then serialisation of a large
payload. Framework overhead is typically well under a millisecond per request; JSON serialisation of
a large response can be tens of milliseconds and is frequently the top CPU consumer in Node (`F24`).

**Advanced.** Frameworks differ more at very high request rates: Fastify is roughly twice as fast as
Express on JSON-heavy paths, largely because it compiles schema-based serialisers instead of using
generic `JSON.stringify`. That matters at tens of thousands of requests per second and is irrelevant
at a few hundred. Say which regime you are in before optimising.

#### 3. Blocking versus non-blocking handlers

**Theory.** The framework's concurrency model determines what a slow handler does to everyone else.
Node runs your JavaScript on one thread, so a synchronous computation blocks every other request in
that process. The JVM's Spring MVC gives each request its own thread, so one slow request occupies a
thread but others proceed. Django with sync workers occupies a whole worker process.

**Example.** The same mistake has different consequences: a 500ms synchronous loop in Nest stalls
every in-flight request in that pod; in Spring MVC it consumes one thread of two hundred; in Django
with four gunicorn workers it consumes 25% of your capacity. See `C03`, `C06`, `C07`.

**Advanced.** This single difference explains most of the architectural divergence between these
frameworks — why Node code is written around async/await and a tiny worker pool, why the JVM tunes
thread pools, and why Python deploys many processes. It is also why virtual threads (Java 21) and
ASGI matter: they move the JVM and Django toward the cheap-concurrency model without rewriting
application code.

### Interview questions

- "Trace a request from TCP accept to JSON response in your framework."
- "What is the `DispatcherServlet` / what is WSGI / what does `NestFactory` actually create?"
- "A handler takes 200ms. Where do you look first?"
- "What happens to other requests when one handler blocks for 500ms?"

---

## F02 · Routing, controllers and parameter binding

`Beginner` · Requires: `F01` · Unlocks: `F06`, `F07`, `F25`, `A03`

### Preface

Routing maps a method and path to one of your functions, and binds parts of the request — path
segments, query string, body, headers — into arguments.

It is mostly mechanical. The parts worth knowing are route precedence (which rule wins when two
match) and the fact that everything arriving here is untrusted text until validated.

### Details

#### 1. Declaring routes

**Theory.** Routes are declared with decorators or annotations near the handler, or in a central
routing table. Either way the framework builds a lookup structure — usually a prefix tree — and
matches incoming requests against it.

**Example.**
- **Nest**: `@Controller('users')` + `@Get(':id')` → `GET /users/:id`; parameters via `@Param('id')`,
  `@Query()`, `@Body()`, `@Headers()`.
- **Django**: `path('users/<int:pk>/', views.detail)` in `urls.py`; DRF adds `ViewSet` +
  `DefaultRouter`, which generates the standard REST routes for you.
- **Spring**: `@RestController` + `@GetMapping("/users/{id}")`, binding with `@PathVariable`,
  `@RequestParam`, `@RequestBody`.

**Advanced.** Route **order** matters when a literal and a parameter can both match. In Nest,
`@Get(':id')` declared before `@Get('me')` means a request for `/users/me` binds `id = "me"` and you
get "user not found". The rule: declare specific literal routes before parameterised ones. Django's
URLconf matches in list order with the same consequence; Spring scores by specificity and usually
does the right thing automatically, which is convenient and makes the behaviour less predictable
across frameworks.

#### 2. Binding and type coercion

**Theory.** Everything in a URL or a header is a string. The framework converts to the declared type,
and how it handles a failed conversion is the interesting part.

**Example.** `GET /users/abc` where the handler expects a number. Nest with `ParseIntPipe` returns
400 with a clear message; without it, you get the string `"abc"` and your query silently compares a
text value (which may break an index, `DB20`). Django's `<int:pk>` converter simply does not match
the route, so you get a 404 — arguably wrong, since the resource was requested badly rather than
being absent. Spring returns 400 automatically.

**Advanced.** The security angle: binding directly into an entity is **mass assignment** (`S09`).
`@Body() user: User` where `User` is your database entity lets a client set `role` or `isAdmin`. Bind
into a DTO listing only permitted fields, and enable whitelisting so unexpected fields are stripped
or rejected (`F06`).

#### 3. Versioning and grouping

**Theory.** Routes are grouped by prefix, module or router to keep them organised, and versioned so
that breaking changes can coexist (`A06`).

**Example.** Nest supports URI versioning (`app.enableVersioning()`, then `@Version('2')`), header
versioning and media-type versioning. Django groups with `include()` under `api/v1/`. Spring uses a
`@RequestMapping("/api/v1")` at class level. The mechanism is uninteresting; the **policy** — how
long you support a version and how you retire it — is what an interview is actually asking about.

**Advanced.** Prefer evolving the API additively so you rarely need a second version at all (`M24`).
When you do version, version the whole API rather than individual endpoints: mixed per-endpoint
versions produce a combinatorial mess for clients and for your tests.

### Interview questions

- "`GET /users/me` returns 'user not found' for id=me. Why?"
- "A client posts `{ "role": "admin" }` to your update endpoint. What stops them?"
- "Where do you put the API version, and why?"
- "What type is a query parameter before validation?"

---

## F03 · The middleware and interceptor pipeline

`Intermediate` · Requires: `F01` · Unlocks: `F07`, `F08`, `F11`, `F15`, `F26`

### Preface

Some work applies to every request: logging, authentication, timing, compression, error handling.
Rather than repeating it in every handler, frameworks run it as a chain around the handler.

Each layer can inspect the request on the way in, inspect or transform the response on the way out,
and stop the chain entirely. The important knowledge is **the order**, because it determines what
information each layer has.

### Details

#### 1. Nest's pipeline, in order

**Theory.** The sequence is: **middleware → guards → interceptors (before) → pipes → handler →
interceptors (after) → exception filters**. Know it cold; it is one of the most frequently asked
NestJS questions.

**Example.** What each is for:
- **Middleware** — raw request/response access, framework-agnostic concerns (request id, helmet,
  body parsing). It runs before Nest knows which handler will be used.
- **Guards** — return true or false: may this request proceed? Authentication and authorisation.
  They run before pipes deliberately, so you do not spend time validating a body for a request that
  is about to be rejected with 401.
- **Interceptors** — wrap the handler; they can transform the result, add timing, caching or
  logging, and see both sides.
- **Pipes** — transform and validate the handler's arguments.
- **Exception filters** — catch anything thrown and turn it into a response.

**Advanced.** Guards before pipes has a security consequence worth stating: validation errors cannot
be used to probe an endpoint you are not authorised for, because the guard rejects first. The
reverse order would leak information about the request schema to unauthenticated callers.

#### 2. The equivalents elsewhere

**Theory.** The same responsibilities exist everywhere under different names.

**Example.**
- **Django**: one middleware chain; each middleware is called with the request going down, and gets
  the response coming back up (it wraps `get_response`). Hooks `process_view`, `process_exception`
  and `process_template_response` give finer control. Order in `settings.MIDDLEWARE` is
  significant — `AuthenticationMiddleware` must come after `SessionMiddleware`, for example.
- **Spring**: servlet `Filter`s are outermost (before Spring even sees the request), then
  `HandlerInterceptor` (`preHandle`, `postHandle`, `afterCompletion`), then AOP `@Around` advice
  around the method, and `@ControllerAdvice` for exceptions. Spring Security is itself a chain of
  filters.

**Advanced.** The practical rule for choosing a layer: use the **outermost** layer that has the
information you need. Request id and access logging belong in middleware (they apply even to
requests that match no route). Authentication belongs in a guard or security filter. Business-level
concerns — "does this user own this order" — belong in the service, not in the pipeline, because they
need domain data.

#### 3. Scope and ordering of your own layers

**Theory.** Cross-cutting components can be registered globally, per controller or per handler. Order
among same-type components is usually declaration order.

**Example.** In Nest, global registration via `app.useGlobalGuards()` is simple but the instance is
created outside the DI container, so it cannot inject dependencies. Registering with the
`APP_GUARD` token in a module instead gives you a global guard **with** dependency injection — a
detail that trips people up when they need a global guard that reads configuration or queries a
database.

**Advanced.** Be careful with interceptors that transform every response: a global "wrap everything
in `{ data: ... }`" interceptor breaks file downloads, streaming responses and health checks, and
makes third-party integrations awkward. If you add one, exclude the routes that must return raw
content, and prefer being explicit at the controller level over a global transform.

#### 4. Request context without passing it everywhere

**Theory.** Many layers need to share information — the request id, the authenticated user, the
tenant — without threading it through every function signature.

**Example.** The mechanism differs by runtime: Node uses `AsyncLocalStorage` (the `nestjs-cls`
package wraps it), Python uses `contextvars`, and the JVM uses a `ThreadLocal` (SLF4J's MDC). All
three store a value that is implicitly available for the duration of the request.

**Advanced.** Each has a failure mode. `AsyncLocalStorage` loses context across some callback-style
APIs and manually-detached promises. `ThreadLocal` breaks when work moves to another thread — a
thread pool, `@Async`, or reactive code — which is why WebFlux needs the Reactor context instead and
why MDC-based logging often shows the wrong user id in async code. Know that context propagation is
a runtime concern, not a framework feature.

### Interview questions

- "Order of guards, interceptors and pipes in Nest, and why is a guard before a pipe?"
- "Where would you implement request timing, and where tenant resolution? Why different layers?"
- "How do you attach a request id to every log line without passing it everywhere?"
- "Why would a global guard not be able to inject a service?"

---

## F04 · Dependency injection and inversion of control

`Intermediate` · Requires: `F01` · Unlocks: `F09`, `F12`, `F18`, `F19`, `F22`

### Preface

Instead of a class creating the things it needs, they are handed to it. The framework builds the
object graph and passes each dependency into the constructor.

The point is substitutability: in a test you pass a fake, in production the real thing, and the class
under test does not change. It also gives the framework a place to manage object lifetimes.

The cost is indirection — you cannot see from a class which concrete implementation it will get.

### Details

#### 1. The mechanism

**Theory.** Register implementations with a container; declare dependencies in the constructor; the
container resolves the graph at startup and injects. Depend on abstractions rather than concrete
classes so the implementation can be swapped.

**Example.**
- **Nest**: `@Injectable()` marks a provider; `providers` in a module registers it; `exports` makes
  it available to importing modules. Constructor parameters are resolved by type.
- **Spring**: `@Component`/`@Service`/`@Repository`, or `@Bean` methods in a `@Configuration` class.
  Constructor injection is preferred; `@Qualifier` and `@Primary` disambiguate when several
  candidates implement the interface.
- **Django**: essentially none. You import modules directly and configure through `settings`.
  Substitution happens via `django.test.override_settings`, monkeypatching, or passing collaborators
  explicitly.

**Advanced.** Django's lack of a container is a real trade, not an oversight: less ceremony and less
indirection, at the cost of harder substitution in tests and less explicit wiring. It is worth being
able to discuss it as a trade-off rather than a deficiency — that is the kind of balanced judgement
an interviewer is listening for.

#### 2. Injecting interfaces in TypeScript

**Theory.** TypeScript interfaces are erased at compile time, so there is no runtime value for the
container to match. You must inject against a **token**.

**Example.**

```ts
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');

@Module({
  providers: [{ provide: PAYMENT_GATEWAY, useClass: StripeGateway }],
  exports: [PAYMENT_GATEWAY],
})
export class PaymentModule {}

@Injectable()
export class OrderService {
  constructor(@Inject(PAYMENT_GATEWAY) private readonly gateway: PaymentGateway) {}
}
```

Now swapping Stripe for a fake in a test, or for Adyen in production, is a one-line module change.

**Advanced.** This is why Nest has `useClass`, `useValue`, `useFactory` and `useExisting`.
`useFactory` with `inject` is how you build a provider that depends on configuration — for example
choosing a gateway based on an environment variable resolved at startup. An abstract class also works
as a token, since it exists at runtime, and some teams prefer that for the better type ergonomics.

#### 3. Scopes and lifetimes

**Theory.** **Singleton** — one instance for the whole application (the default, and correct for
almost everything). **Request-scoped** — a new instance per request. **Transient** — a new instance
per injection point.

**Example.** Request scope is tempting for "current user" or "current tenant", and it has a real cost
in Nest: making a provider request-scoped **bubbles up the entire injection chain**, so every
provider and controller that depends on it also becomes request-scoped and is instantiated per
request. Throughput drops measurably.

**Advanced.** The better pattern for request data is `AsyncLocalStorage` (`F03`): keep everything a
singleton and read the ambient context where you need it. Spring has the same issue and solves it
with scoped proxies — a singleton holds a proxy that resolves to the current request's instance —
which is clever and adds its own confusion. The general advice is: singletons plus explicit context,
not request-scoped graphs.

#### 4. Circular dependencies

**Theory.** If A needs B and B needs A, the container cannot decide which to build first.

**Example.** Nest offers `forwardRef(() => OtherService)` on both sides, which works and is a
**design smell**: two services that need each other usually want a third component coordinating
them, an event so one no longer calls the other, or to be one service. Fix the design first;
`forwardRef` is the escape hatch.

**Advanced.** Because Nest resolves the whole graph at bootstrap, dependency errors surface as
startup failures rather than runtime ones — which is good (fail fast) and makes the error messages
important to read carefully: "Nest can't resolve dependencies of X (?, Y)" tells you the position of
the unresolvable parameter with `?`, which is the fastest way to identify a missing `exports` or a
missing provider.

### Interview questions

- "Why can't you inject a TypeScript interface in Nest, and what do you do instead?"
- "Field versus constructor injection — why is field injection discouraged?"
- "What does `Scope.REQUEST` do to the rest of your injection graph?"
- "You have a circular dependency. What does that usually mean?"

---

## F05 · Configuration, environments and secrets

`Beginner` · Requires: `F01` · Unlocks: `F18`, `O19`, `S10`

### Preface

Configuration is everything that differs between environments: database URLs, API keys, feature
flags, limits.

Two rules cover most of it. **Config comes from the environment**, not from files in the repository,
so the same build artifact runs anywhere. And **validate it at startup**, so a missing variable
crashes the deploy immediately rather than throwing at 3am when a rare code path runs.

### Details

#### 1. Config from the environment

**Theory.** The twelve-factor principle: strict separation of config from code. The same container
image is promoted from staging to production and behaves differently purely because of its
environment. Anything in the image is not config.

**Example.** The usual layering: sensible defaults in code → environment variables → secrets injected
by the platform (Kubernetes secrets mounted as env vars or files, or fetched from a secret manager
at startup). `.env` files exist for local development only and must be in `.gitignore`.

**Advanced.** Environment variables have real limits: they are visible in `/proc` and in crash
dumps, they appear in many logging and error-reporting integrations, and they cannot be rotated
without a restart. For high-value secrets, prefer a mounted file (readable once at startup) or
fetching from a secret manager with support for rotation (`S10`). Being aware of the difference
between "config" and "secret" is the senior distinction.

#### 2. Validate at startup

**Theory.** Parse and validate the whole configuration when the process starts. If anything is
missing or malformed, log a clear message and exit non-zero. A crash at boot is caught by your
deployment; a crash on first use is caught by a customer.

**Example.**
- **Nest**: `ConfigModule.forRoot({ validationSchema })` with Joi, or a zod schema parsed in a
  factory provider that returns a typed config object.
- **Django**: explicit checks in `settings.py`, or `django-environ` with required casts; plus the
  deployment checklist (`manage.py check --deploy`).
- **Spring**: `@ConfigurationProperties` classes with Bean Validation annotations, which bind and
  validate at startup and fail the context.

**Advanced.** Validate **types and ranges**, not only presence: a port that is not a number, a
timeout of `0`, a URL missing its scheme. Also produce a typed configuration object rather than
reading `process.env.X` scattered through the code — that gives you one place to see every setting,
one place to document it, and compile-time safety.

#### 3. Per-environment differences

**Theory.** Environments should differ as little as possible. Every difference between staging and
production is a class of bug that staging cannot catch.

**Example.** Differences that matter and should be deliberate: connection pool sizes, log level,
rate limits, whether external services are real or sandboxed, and feature flag defaults. Differences
that should **not** exist: different code paths (`if (env === 'production')` in business logic),
different database engines, or a schema that has drifted.

**Advanced.** `if (process.env.NODE_ENV === 'production')` in business logic is an anti-pattern
because the behaviour you ship is then untested. Express the difference as a configuration **value**
(`emailDeliveryEnabled: boolean`) rather than an environment check, so tests can exercise both
branches and staging can run production's configuration when you want to verify it.

#### 4. Configuration changes as a risk

**Theory.** A config change is a production change with the same blast radius as a deploy, and
usually with less review and no canary.

**Example.** Guard rails worth having: configuration in version control (`O10`), reviewed like code;
validation applied before the change is accepted; staged rollout for anything applied globally
(`M31`); and an audit log of who changed what. A bad value pushed to every instance at once is one
of the most common causes of large outages.

**Advanced.** Distinguish **static** configuration (requires a restart — pool sizes, ports) from
**dynamic** configuration (changeable at runtime — feature flags, rate limits, log level). Dynamic
config is powerful for incident response ("turn off the expensive feature") and needs a safe fallback
when the config service is unreachable: cache the last known good value and keep serving it (`O19`).

### Interview questions

- "Where does a secret live from developer laptop to production pod?"
- "Why validate config at startup rather than on first use?"
- "Why is `if (env === 'production')` in business logic a problem?"
- "A config change caused an outage. What guard rails do you add?"

---

## F06 · Validation and serialisation

`Intermediate` · Requires: `F02` · Unlocks: `F07`, `F25`, `S09`

### Preface

Everything arriving from a client is untrusted. Validation converts it into a known, typed shape at
the boundary, so the rest of your code can assume correctness.

Serialisation is the same problem in reverse: decide explicitly what leaves your system, so you do
not accidentally return a password hash, an internal id, or another tenant's field.

The rule for both: **allow-list, never block-list.**

### Details

#### 1. Validate at the boundary into a DTO

**Theory.** A DTO (data transfer object) describes the wire format. It is separate from your domain
model and from your database entity, because the three change for different reasons and have
different audiences.

**Example.**

```ts
export class CreateOrderDto {
  @IsUUID() customerId: string;
  @IsArray() @ValidateNested({ each: true }) @Type(() => OrderLineDto) lines: OrderLineDto[];
  @IsOptional() @IsString() @MaxLength(500) note?: string;
}

app.useGlobalPipes(new ValidationPipe({
  whitelist: true,             // strip properties with no decorator
  forbidNonWhitelisted: true,  // or reject the request outright
  transform: true,             // instantiate the DTO class and coerce types
}));
```

`whitelist: true` is the setting that prevents mass assignment (`S09`) — without it, extra fields
reach your service and may be passed to `repository.save()`.

**Advanced.** The Django and Spring equivalents: DRF serializers with explicit `fields` and
`read_only_fields`; Spring with `@Valid` plus Jakarta Bean Validation annotations, and a DTO mapped
to the entity by hand or with MapStruct. In all three the same trap exists — binding straight to the
persistence entity is convenient and is how mass-assignment vulnerabilities get shipped.

#### 2. Where validation belongs

**Theory.** Three layers, each with a different job. The **DTO** checks shape and format (is this a
UUID, is this string under 500 characters). The **service** checks business rules that need context
(is this customer allowed to order, is the product still available). The **database** enforces
invariants that must hold regardless of code paths (`DB05`).

**Example.** "Email must be unique" illustrates all three: the DTO checks it looks like an email;
the service may check for a friendly error message; and the **unique index** is what actually
guarantees it, because only the database is free of race conditions (`DB05`). Present all three
layers when asked — candidates who name only one are usually missing the concurrency point.

**Advanced.** Keep framework-specific validation out of the domain. If your domain entity is
decorated with `class-validator` annotations, it is coupled to an HTTP concern and hard to reuse.
Validate the DTO at the edge, construct a domain object from validated data, and let the domain
enforce its own invariants in its constructor or factory.

#### 3. Serialisation: controlling what leaves

**Theory.** Returning an entity directly exposes every column, including ones added later by someone
who did not consider your endpoint. Map explicitly to a response DTO, or annotate exclusions.

**Example.** Nest offers `@Exclude()` on the entity plus `ClassSerializerInterceptor`, which works
but is opt-out — a new sensitive column is exposed by default. An explicit response DTO or a mapping
function is opt-in and safer. Django REST Framework's explicit `fields` list is opt-in by default,
which is a point in its favour.

**Advanced.** Serialisation is also a performance concern: it is often the largest CPU cost in a Node
service (`F24`). Fastify's schema-based serialiser is much faster than generic reflection-based
approaches precisely because it knows the shape in advance. And field-level authorisation belongs
here — some fields should only appear for some roles, which an opt-in mapping expresses naturally
and a global "serialise the entity" approach does not.

#### 4. Error messages

**Theory.** Validation errors should identify the field and the rule, be machine-readable, and be
safe to show. They must not reflect input back unescaped, and must not reveal internal structure.

**Example.** A good shape:
`{ "code": "VALIDATION_ERROR", "errors": [{ "field": "lines[0].quantity", "code": "min", "message": "must be at least 1" }] }`.
The client can highlight the right input; the `code` lets it localise the message. See `A07`.

**Advanced.** Be careful not to leak information through validation: an endpoint that says "this
email is already registered" confirms account existence to an attacker (`S13`). For signup and
password reset flows, respond identically whether or not the account exists, and communicate the
conflict through the email channel instead.

### Interview questions

- "A user posts `{ "role": "admin" }` to your update endpoint. What stops them?"
- "Where do you validate: DTO, service, or database? Argue for all three."
- "How do you stop a newly added column from leaking through an endpoint?"
- "Your validation message says 'email already registered'. Is that a problem?"

---

## F07 · Error handling

`Intermediate` · Requires: `F02`, `F03`, `F06` · Unlocks: `F08`, `A07`

### Preface

Two kinds of error need different treatment. **Expected** errors are part of the domain — not found,
insufficient funds, already cancelled — and deserve a clear, documented response. **Unexpected**
errors are bugs and infrastructure failures, and deserve a generic 500, a log entry with full detail,
and an alert.

The structural rule: the domain throws domain errors, and one place at the boundary translates them
into HTTP.

### Details

#### 1. Keep HTTP out of the service layer

**Theory.** If your service throws `NotFoundException` (an HTTP concept), it can only be used behind
HTTP. The same service called from a queue consumer, a CLI or a gRPC handler has no meaningful
response code to throw.

**Example.**

```ts
// domain layer
export class OrderNotFoundError extends DomainError {
  constructor(readonly orderId: string) { super(`Order ${orderId} not found`); }
}

// boundary: one global exception filter
if (err instanceof OrderNotFoundError)   return res.status(404).json(problem(err, 'ORDER_NOT_FOUND'));
if (err instanceof InsufficientFunds)    return res.status(409).json(problem(err, 'INSUFFICIENT_FUNDS'));
```

One place maps domain errors to status codes. Adding a transport later requires only a new mapper.

**Advanced.** The pragmatic counter-argument is worth acknowledging: for a service that will only
ever be an HTTP API, throwing `NotFoundException` directly is less code and perfectly readable.
State the trade — purity versus ceremony — and note that the translation layer earns its cost as soon
as a second entry point appears (a queue consumer is usually the first).

#### 2. One consistent error contract

**Theory.** Every endpoint should fail in the same shape, with a stable machine-readable code, a
human message, field-level detail where relevant, and a correlation id.

**Example.** RFC 7807 `application/problem+json` (`A07`) plus an application-specific `code`.
Framework support: Nest exception filters, DRF's `EXCEPTION_HANDLER`, Spring's `@ControllerAdvice`
with `ProblemDetail`. Whichever you use, apply it globally so no endpoint can accidentally return a
different shape.

**Advanced.** The `code` must be stable and separate from the message: clients switch on the code,
and the message is for humans and may be reworded or translated at any time. Document the codes
alongside the API. Changing a code is a breaking change (`M24`).

#### 3. Never leak internals

**Theory.** Stack traces, SQL text, file paths, library versions and internal service names must not
reach clients. They help attackers and confuse users.

**Example.** The rule: log the full detail server-side with a correlation id, return the correlation
id to the client, and nothing else. The support conversation then becomes "please give me the
reference from the error message" and you find the exact request in your logs. Make sure `DEBUG` is
false in Django and that Nest's default 500 handler is replaced — the defaults are developer-friendly
and production-hostile.

**Advanced.** Also consider *timing* and *status code* leaks: returning 404 for "not found" but 403
for "exists but forbidden" tells an attacker which ids exist. For sensitive resources, return 404 in
both cases. This is a subtle point that distinguishes candidates who have thought about
authorisation properly (`S09`).

#### 4. Retryable versus terminal

**Theory.** The client needs to know whether trying again might work. Communicate it through the
status code (429 and 503 with `Retry-After` are retryable; 400 and 422 are not) and, where useful,
an explicit flag.

**Example.** This matters most for service-to-service calls, where an automatic retry policy reads
the response (`M09`). Returning 500 for a validation failure causes clients to retry something that
will never succeed, amplifying load during an incident. Getting status codes right is an
availability concern, not a style preference.

**Advanced.** Log once, at the boundary. A common anti-pattern is catching, logging and rethrowing at
every layer, which produces five entries for one error and makes the log unreadable. Let errors
propagate with context attached (wrap with the original as `cause`), and log the complete chain once
where it is handled.

### Interview questions

- "Should the service layer throw `HttpException`/`ResponseStatusException`? Why not?"
- "Design your API's error format and explain each field."
- "Why might you return 404 instead of 403?"
- "Your logs show the same error five times per occurrence. Why?"

---

## F08 · Logging

`Intermediate` · Requires: `F03`, `F07` · Unlocks: `O13`, `M25`

### Preface

Logs are what you have at 3am when something is wrong and you cannot reproduce it.

Two decisions make them useful: log **structured data** (JSON with fields) rather than sentences, so
you can filter and aggregate; and attach **request context** — request id, trace id, user, tenant —
to every line automatically, so you can reconstruct one request's journey.

And one rule: never log secrets or personal data.

### Details

#### 1. Structured logging

**Theory.** A log line should be an object with a message and fields, serialised as JSON. Log
aggregators can then filter by field (`tenantId = X`), aggregate by value, and alert on counts —
none of which works reliably with free text.

**Example.**

```js
logger.info({ orderId, tenantId, amountMinor, durationMs }, 'order placed');
```

rather than `` logger.info(`Order ${orderId} placed for ${amount}`) ``. Use pino in Node (it is
markedly faster than winston because it serialises minimally and can offload work to a worker),
Python's `logging` with a JSON formatter, and Logback with a JSON encoder in Spring.

**Advanced.** Two performance notes. Logging is not free: a synchronous write to stdout on a busy
service can become a bottleneck, and in Node, `console.log` to a pipe is **synchronous** and blocks
the event loop. Use a logger with asynchronous transports. Second, beware of logging large objects —
serialising a 100KB entity per request is real CPU and real storage cost.

#### 2. Levels and what to log

**Theory.** `error` — something failed and needs attention. `warn` — unexpected but handled.
`info` — significant business events. `debug` — detail, off in production by default. The useful
discipline is that `error` should be actionable; if nobody will act on it, it is a `warn`.

**Example.** What to log for a typical request: one access log line at the boundary (method, path,
status, duration, user, request id) and nothing else on the happy path; errors with full context;
and explicit business events worth counting (`order placed`, `payment failed`). Avoid logging on
entry and exit of every function — it costs money and buries the signal.

**Advanced.** Make log level dynamically adjustable at runtime (`O19`). During an incident you want
debug logging for one service or one tenant without a deploy. Per-tenant or per-request debug
logging, triggered by a header, is even better — it is the same mechanism as a debug trace flag and
it is cheap to build.

#### 3. Request context

**Theory.** Every line produced while handling a request should carry the request id and trace id
without the calling code passing them (`F03`).

**Example.**

```ts
// middleware: create the context once
cls.run(() => { cls.set('requestId', req.headers['x-request-id'] ?? randomUUID()); next(); });

// logger: mixin adds it to every line automatically
const logger = pino({ mixin: () => ({ requestId: cls.get('requestId'), traceId: cls.get('traceId') }) });
```

Now every log line in the request, at any depth, is filterable by request id, and joins to your
traces (`M25`).

**Advanced.** Propagate the incoming `x-request-id` if present rather than always generating one, so
that a request id assigned by the gateway ties together every service involved. And put the **trace
id** in logs too — it is the join key between your logs and your traces, and without it you have two
systems that cannot talk to each other (`O13`).

#### 4. What never to log

**Theory.** Passwords, tokens, API keys, card numbers, full personal data, and anything covered by
your privacy policy. Logs are usually stored longer, replicated more widely and protected less
carefully than your database.

**Example.** The common accidents: logging the whole request body on error (which contains the
password on a login endpoint); logging the `Authorization` header; logging a whole user object;
logging a database error that includes the parameter values. Redaction must be automatic — pino's
`redact` option, a Logback masking converter, a DRF filter — because relying on every developer to
remember will fail.

**Advanced.** Treat log retention as part of your data-protection posture (`S12`): set a retention
period, know which fields are personal, and be able to answer "where does personal data live" with
logs included. A deletion request that ignores logs and analytics is not complete. This is a detail
that impresses because most candidates only think about the database.

### Interview questions

- "How do you attach a request id to every log line in an async call stack?"
- "Why structured logs rather than formatted strings?"
- "What would you log for a single successful request, and what would you not?"
- "A deletion request arrives. Do your logs matter?"
