[← back to the field index](README.md)

# Microservices · Part 4 — Evolution, Observability & Delivery

Nodes `M24`–`M28`.

---

## M24 · Contracts, versioning and backward compatibility

`Advanced` · Requires: `M02`, `M04`, `M17` · Unlocks: `M27`, `A22`, `Q19`

### Preface

The moment another service depends on your API or your events, you cannot change them freely. But
you also cannot stop changing them.

The way out is **backward-compatible change**: add things, never remove or repurpose them, and
retire old fields only after you can prove nobody uses them. Done well, you almost never need a
version 2.

A point people miss: during a rolling deploy, *your own service* runs two versions at once. Every
change must work with both.

### Details

#### 1. What is and is not a breaking change

**Theory.** Safe: adding an optional field to a response; adding an optional field to a request;
adding a new endpoint; adding a new enum value *if* consumers are built to ignore unknown values.
Breaking: removing a field; renaming a field; changing a type; making an optional field required;
tightening validation; changing the meaning of an existing value; changing default behaviour.

**Example.** The sneaky ones are semantic. Changing `status: "PENDING"` to mean something slightly
different, or changing pagination from returning all results to a default limit of 100, breaks
consumers without any schema change. So does tightening validation — a field that accepted 200
characters now rejecting anything over 100 breaks callers who were within the old rule.

**Advanced.** Adding an enum value is the classic argument. It is safe only if consumers follow the
**tolerant reader** principle: ignore what you do not understand, do not fail on unknown values.
Many generated clients throw on unknown enums. So either document that consumers must tolerate new
values from day one, or treat new enum values as breaking. Decide it before you need it, and write
it in the API guidelines.

#### 2. Expand, migrate, contract

**Theory.** The safe sequence for any change that looks breaking, applied to APIs, events and
database schemas alike:
1. **Expand** — add the new thing alongside the old; write both; read the old.
2. **Migrate** — move consumers to the new one; backfill data; switch reads.
3. **Contract** — once nothing uses the old thing, remove it.

Each step is independently deployable and reversible.

**Example.** Renaming `name` to `fullName` in an API: (1) return both fields, accept both on
input; (2) update consumers to use `fullName`, monitor usage of `name`; (3) when usage is zero for
long enough, remove `name`. It may take a month. That is the correct speed.

**Advanced.** The same shape applies to database columns (`DB22`), to event schemas, and to queue
message formats. The only hard requirement is **measuring** step 2: you need per-consumer usage
metrics on the old field, or you are guessing. Without that data, teams either never remove
anything or remove it and break someone. Logging field-level access, or at minimum per-client
request counts, is what makes contraction possible.

#### 3. Versioning when you must break

**Theory.** If a breaking change is unavoidable, run both versions in parallel. Options: URL path
(`/v1/`, `/v2/`) — visible and easy to route; header or media type — cleaner in theory, harder to
test and debug; and for events, a new topic or a version field.

**Example.** A practical policy: support N-1 versions; announce deprecation with a date; emit
`Deprecation` and `Sunset` headers; track usage per client; contact the remaining users directly;
then apply a **brownout** — making the old version fail for short periods (say one minute an hour)
so remaining consumers notice before the final shutdown. Brownouts are much kinder than a surprise
removal.

**Advanced.** Internally, prefer evolution over versioning: two versions means double the
maintenance, double the tests, and confusion about which is current. Externally — a public or
partner API where you cannot make clients upgrade — versioning is unavoidable and you should plan
for supporting an old version for years. Mobile apps are the extreme case: old versions live in
users' pockets indefinitely, so the server must support them essentially forever.

#### 4. Both versions run at once

**Theory.** During a rolling deploy there is a window where old and new instances of *your* service
handle traffic, consume the same topics and share the same database. Every change must be safe in
that mixed state.

**Example.** You add a `NOT NULL` column and deploy. Old instances, still running, insert rows
without it and fail. The correct sequence: add the column nullable → deploy code that writes it →
backfill → add the constraint → deploy code that requires it. Four steps, each safe with both
versions live. The same applies to events: the new version publishes a new field while old
consumers still read the message, so the field must be optional.

**Advanced.** This is the strongest practical argument for expand/contract and for the discipline
of never doing a schema change and a code change that depend on each other in one deploy. If an
interviewer asks about zero-downtime deploys, connecting migration safety to "two versions run
simultaneously" is the insight they are listening for.

#### 5. Consumer-driven contract testing

**Theory.** Integration-testing every consumer against every provider does not scale. Instead, each
consumer writes down what it actually uses, and the provider runs those expectations in its own
pipeline. The provider learns it broke someone *before* deploying.

**Example.** With Pact: the Orders service (consumer) writes a test declaring "when I call
`GET /customers/1`, I need `id` and `email`". That produces a contract file. The Customers service
(provider) runs all consumer contracts in CI. If a change removes `email`, the provider's build
fails, naming the consumer.

**Advanced.** The real value is that contracts capture only fields that are genuinely used, so the
provider learns what it can safely change. The real cost is organisational — consumers must
maintain their contracts, and someone must run a broker. For events, the equivalent is a schema
registry with compatibility checking (see `Q19`), which is cheaper to adopt and should be the
default for any message-based system.

### Interview questions

- "You must rename a field used by 12 consumers. Sequence the change."
- "During a rolling deploy v1 and v2 both consume the same topic. What does that constrain?"
- "Is adding a new enum value a breaking change?"
- "How do you know when it is safe to delete a deprecated field?"

---

## M25 · Distributed tracing and correlation

`Intermediate` · Requires: `M22`, `O13` · Unlocks: `M26`, `O15`

### Preface

In a monolith, an error gives you a stack trace. Across services, one user action becomes twenty
log lines in eight systems with nothing linking them.

Tracing fixes this by attaching an id to the request at the edge and carrying it everywhere,
including through queues and background jobs. Then one query shows the whole journey and where the
time went.

### Details

#### 1. Traces, spans and context

**Theory.** A **trace** is one end-to-end request. A **span** is one unit of work inside it (an HTTP
handler, a database query, a queue publish), with a start time, duration, attributes, and a parent.
Spans form a tree; drawn on a timeline it is a waterfall showing exactly where time was spent.

**Example.** A checkout trace: `POST /checkout` (820ms) → validate (5ms) → `SELECT cart` (12ms) →
call Payments (600ms) → inside Payments, call the card provider (580ms) → publish `OrderPlaced`
(8ms). One glance tells you the card provider owns the latency and your own code is not the problem.

**Advanced.** Span attributes are where the value is: tenant id, user id, endpoint, SKU, cache hit
or miss, retry count. They let you ask "are slow requests concentrated in one tenant?" — a question
metrics cannot answer because you cannot put high-cardinality values in metric labels (see `O14`).
This is the core argument for traces existing alongside metrics.

#### 2. Propagation

**Theory.** The trace context must travel with the work. The standard is the W3C `traceparent`
header, carrying trace id, span id and flags. Most HTTP libraries handle it automatically once
instrumented; the gaps are elsewhere.

**Example.** The three places propagation usually breaks: **message queues** (you must put
`traceparent` in the message headers and restore it in the consumer); **background jobs** (the job
payload must carry it); and **manual thread or async boundaries** (a `setTimeout`, a worker thread,
a Java thread pool where the context does not follow). A trace that stops at the queue is the most
common complaint about tracing setups.

**Advanced.** For asynchronous work, the right model is often a **span link** rather than a
parent-child relation: the consumer's span is causally related to the producer's but is not a child,
because the parent has already ended. OpenTelemetry supports links for exactly this. Using links
keeps the queue latency visible without producing a trace that appears to last four hours because a
message sat in a queue.

#### 3. Sampling

**Theory.** Tracing everything is expensive in bandwidth, storage and money. **Head sampling**
decides at the start of the request (keep 1%) — cheap and simple, but it throws away most errors and
slow requests, which are the ones you wanted. **Tail sampling** buffers the whole trace and decides
after it completes — keep all errors, all traces over 1 second, and 1% of the rest.

**Example.** A sensible production configuration: tail sampling in the OpenTelemetry collector with
rules — keep 100% of traces containing an error, 100% over the p99 latency threshold, 100% for a
named debug header, and 1% of everything else. You keep what is useful at a fraction of the cost.

**Advanced.** Tail sampling requires all spans of a trace to reach the same collector instance,
which needs load balancing by trace id — an operational detail that surprises teams. And sampling
must be **consistent** across services: if each service samples independently, you get partial
traces that are worse than none. The sampling decision travels in the `traceparent` flags for
exactly this reason.

#### 4. Correlating logs, metrics and traces

**Theory.** The three signals are useful together and weak apart. Put the trace id in every log
line; attach **exemplars** (sample trace ids) to metrics, so you can click from a latency spike on a
graph to a real slow trace.

**Example.** The debugging path you want to be possible: alert fires on p99 latency → open the
dashboard → click an exemplar on the spike → land in a real slow trace → see the slow span → open
that service's logs filtered by trace id → read the error. That path turns a 40-minute
investigation into a 4-minute one, and it only works if the ids are wired through everywhere.

**Advanced.** Also propagate a **business** correlation id (order id, tenant id) alongside the
trace id. Trace ids expire with your retention; a support ticket about an order from three weeks ago
needs an identifier that lives in your own data. Mature systems log both.

### Interview questions

- "A request is slow 1% of the time. How do you find where?"
- "How do you propagate a trace through Kafka and through a background job?"
- "Head sampling or tail sampling? What does each cost?"
- "How do you get from a latency graph to the actual slow request?"

---

## M26 · Service mesh and sidecars

`Advanced` · Requires: `M04`, `M05`, `M06`, `M10`, `M25` · Unlocks: `M27`, `S15`

### Preface

Every service needs the same networking behaviour: retries, timeouts, TLS between services, load
balancing, metrics, traffic splitting for canaries. Implementing all of it in every service, in
every language, and keeping them consistent, is a lot of duplicated work.

A service mesh moves that behaviour out of your application into a proxy that runs beside it.
Your service makes a plain HTTP call to `localhost`; the proxy does the rest.

The trade is uniformity and central control against another moving part, extra latency and real
operational complexity.

### Details

#### 1. Data plane and control plane

**Theory.** The **data plane** is the set of proxies (usually Envoy) that carry the actual traffic —
one sidecar container per pod, or a per-node agent. The **control plane** (Istio, Linkerd) configures
them: it knows the service registry and pushes routing rules, certificates and policies.

**Example.** Your Nest service calls `http://payments`. The request never leaves the pod
unencrypted: the sidecar intercepts it, resolves a healthy Payments instance, opens a mutual-TLS
connection, applies a timeout and retry policy, records metrics, and forwards it. Your code contains
no TLS, no retry logic and no service discovery.

**Advanced.** Sidecars cost real resources — typically 50-100MB of memory and some CPU per pod, plus
around 1ms of added latency per hop (two hops per call: outbound sidecar and inbound sidecar). At a
thousand pods that is tens of gigabytes of memory spent on proxies. This is why newer designs
(Istio's ambient mode, Cilium's eBPF approach) move the work to a per-node agent or into the kernel,
removing the per-pod overhead.

#### 2. What a mesh gives you

**Theory.** Uniform mutual TLS with automatic certificate rotation and workload identity; retries,
timeouts and circuit breaking configured centrally; traffic splitting by percentage or header for
canaries; consistent metrics and traces for every call without touching application code; and
policy — which service may call which.

**Example.** Canary release without a deploy: tell the mesh to send 5% of traffic to v2, watch error
rate and latency for v2 specifically, then increase. No code change, no separate load balancer
configuration, and instant rollback by setting the split back to 0%.

**Advanced.** The security value is often the strongest justification, more than the traffic
features. Mutual TLS everywhere with short-lived, automatically rotated certificates and identity
per workload (SPIFFE) is genuinely difficult to implement per service and per language, and it turns
"anything on the network can call anything" into an explicit policy (see `S15`).

#### 3. Mesh versus library

**Theory.** The alternative is a shared library in each service (resilience4j, Polly, an
interceptor in Nest). Libraries have no network overhead and are easy to reason about, but must be
written per language, and upgrading them means redeploying every service — which is precisely the
coupling you wanted to avoid.

**Example.** A single-language shop (all TypeScript) gets most of the value from a shared internal
package with a configured HTTP client. A polyglot shop with Node, Java, Python and Go has four
implementations that drift, and a mesh starts paying for itself. Team size matters too: a mesh needs
someone who understands it when it misbehaves.

**Advanced.** The classic bug when adopting a mesh is **double retries**: the application retries
three times and the mesh retries three times, producing nine attempts and an amplified load storm.
When adopting a mesh, remove retry logic from applications, or explicitly disable it in the mesh.
Pick one layer (see `M09`).

#### 4. Is it worth it

**Theory.** It is a platform investment. It pays off with many services, several languages, and a
team to operate it. It does not pay off with five services in one language.

**Example.** A staged, honest path: start with a shared HTTP client library that has timeouts and
retries; add OpenTelemetry for tracing; if you outgrow those and need mutual TLS everywhere and
per-service traffic policy, adopt a lightweight mesh (Linkerd is markedly simpler than Istio);
consider ambient or eBPF modes to avoid per-pod overhead.

**Advanced.** The failure mode to be honest about: a mesh adds a component that can itself break, in
the path of **every** request, and debugging it requires understanding Envoy configuration. Teams
have had outages caused solely by mesh upgrades or misapplied policies. If asked "would you adopt a
service mesh?", the strong answer starts with "what problem are we solving, and do we have someone
to own it?".

### Interview questions

- "What does a mesh give you that a library does not?"
- "Retries are configured in both the mesh and the app. What is the bug?"
- "What does a sidecar cost?"
- "When would you not adopt a service mesh?"

---

## M27 · Deployment and release strategies

`Intermediate` · Requires: `M24`, `M26` · Unlocks: `M28`, `O09`, `SD14`

### Preface

Independent deployment is the reason to have services at all, so how you deploy matters as much as
what you build.

The central idea is to **separate deploying code from releasing behaviour**. Deploy the new binary
with the feature switched off; turn it on afterwards, gradually, and turn it off instantly if
something is wrong. A rollback then costs a config change, not a redeploy.

### Details

#### 1. The strategies

**Theory.**
- **Rolling** — replace instances a few at a time. The default; both versions run together during
  the roll, which constrains compatibility.
- **Blue/green** — run a complete second environment, switch traffic at once, switch back to roll
  back. Fast rollback, double the resources, and the database is still shared.
- **Canary** — send a small percentage of real traffic to the new version, watch, then increase.
  Catches problems that only real traffic reveals.
- **Shadow** — send a copy of production traffic to the new version and discard the responses.
  Zero user risk, but you must prevent side effects.

**Example.** A realistic combination: rolling deploys for routine changes; canary with automated
analysis for anything risky; shadow traffic when rewriting a critical component, comparing outputs
before switching.

**Advanced.** Blue/green's weakness is data. The two environments share a database, so a schema
change must work with both versions anyway — which is the same constraint as rolling. Blue/green
buys fast traffic rollback, not data rollback. Any migration you cannot reverse (a dropped column)
makes rollback impossible regardless of strategy.

#### 2. Canary analysis

**Theory.** A canary is only useful if something is watching. Compare the canary against the
baseline on error rate, latency percentiles, and key business metrics, for long enough to be
statistically meaningful, and roll back automatically on regression.

**Example.** Route 1% of traffic to v2 for 10 minutes. Compare v2's 5xx rate and p99 latency with
v1 over the same window — comparing against v1 concurrently, not against yesterday, so you are not
fooled by a traffic pattern change. If error rate is worse by a defined margin, roll back
automatically. Then 5%, 25%, 50%, 100% with the same checks.

**Advanced.** Two traps. **Sample size**: 1% of traffic for two minutes may contain zero occurrences
of the bug, so a clean canary proves little — for rare paths you need longer or a higher percentage.
**Canary-specific bias**: if your canary receives only new sessions, or only one region's traffic,
it is not a representative sample. Where a bug only appears for one customer's data shape, canaries
do not help and feature flags with per-tenant targeting do.

#### 3. Feature flags

**Theory.** A flag separates "the code is deployed" from "the behaviour is on". You can deploy at
any time, enable for internal users first, then a percentage, then everyone, and disable instantly
without a deploy.

**Example.** New pricing logic ships behind `newPricingEngine`, defaulting to off. Enable it for
your own accounts, then 1% of traffic, and compare computed prices with the old engine in shadow
mode before trusting it. When it misbehaves at 3am, the fix is flipping a flag, not an emergency
release.

**Advanced.** Flags accumulate and become a liability: each one doubles the paths through the code,
and combinations become untestable. Discipline required: an owner and an expiry date per flag, a
periodic sweep to delete resolved flags, and a rule that flags are short-lived release tools, not
permanent configuration. Also remember flags are a **production dependency** — if your flag service
is unreachable, the SDK must fall back to sane defaults and cached values rather than failing.

#### 4. Migrations in the deployment pipeline

**Theory.** Schema changes are the part of a deploy that cannot be rolled back by redeploying the
previous image. They must be applied separately, before the code that needs them, and must remain
compatible with the version currently running.

**Example.** The safe order: (1) run the additive migration (add nullable column, create index
concurrently) while the old code is live; (2) deploy code that writes both old and new; (3) backfill
in throttled batches; (4) deploy code that reads the new; (5) later, remove the old column. Steps 1
and 5 are separate releases, days or weeks apart.

**Advanced.** Running migrations automatically at application startup is a common pattern with real
hazards: several pods start at once and race (you need an advisory lock), a long migration delays
the readiness probe until the pod is killed, and a failing migration puts the deployment into a
crash loop. Prefer a separate migration step in the pipeline — a Kubernetes Job or a pipeline stage —
that must succeed before the rollout begins. See `DB22` and `O09`.

### Interview questions

- "How do you roll back a deploy that already ran a destructive migration?"
- "Blue/green solves rollback — what does it not solve?"
- "How do you decide a canary is healthy?"
- "Where do database migrations run in your pipeline, and why not at startup?"

---

## M28 · Strangler fig: migrating off a monolith

`Advanced` · Requires: `M01`, `M22`, `M27` · Unlocks: `SD14`

### Preface

Named after a vine that grows around a tree and eventually replaces it. You put a proxy in front of
the monolith, move one capability at a time to a new service, and route that capability's traffic to
the new place. The monolith shrinks until it disappears.

The alternative — a big-bang rewrite — fails reliably: it takes far longer than planned, the old
system keeps changing while you work, and you cannot ship anything until the end.

### Details

#### 1. The mechanics

**Theory.** Put a routing layer in front (a gateway, reverse proxy, or the monolith itself
forwarding). Extract one capability: build the new service, move the data, route its endpoints to
it, verify, then delete the old code. Repeat.

**Example.** Extracting notifications from a Nest monolith:
1. Add a facade inside the monolith: all notification sending goes through `NotificationFacade`,
   still implemented in-process. (This alone is valuable and low-risk.)
2. Build the new service with the same interface.
3. Change the facade to call the new service, behind a feature flag, for 1% of traffic.
4. Increase to 100%, watching error rates and delivery metrics.
5. Move the templates and delivery-history data.
6. Delete the monolith's implementation.

**Advanced.** Step 1 is **branch by abstraction** and it is the step that makes the rest safe: you
introduce a seam inside the old code first, so switching implementations is a flag, not a rewrite.
Teams that skip it end up doing the extraction and the switchover as one irreversible change.

#### 2. Choosing what to extract first

**Theory.** Pick something with clear boundaries, limited data coupling, and real value from being
separate. Early extractions should build confidence and tooling, not prove heroism.

**Example.** Good first candidates: notifications, file or image processing, search, reporting,
PDF generation, third-party integrations — leaf capabilities that mostly consume data and produce
side effects. Bad first candidates: user management (everything depends on it), the core domain
entity (orders in an order system), anything sharing tables with five modules.

**Advanced.** Sometimes the correct first move is the opposite: extract the part that is causing
the most operational pain, even if it is hard, because that is where the return is. The judgement
call is between "prove the pattern cheaply" and "solve the real problem now". State the trade-off in
an interview rather than giving one rule.

#### 3. Moving the data

**Theory.** The hardest part, always. The data must move without downtime and without losing writes
while both systems are live. Options: dual writes; change data capture; or a read-only period.

**Example.** A typical sequence: (1) new service reads from the monolith's database temporarily —
ugly but it unblocks progress; (2) copy data to the new store and keep it in sync with CDC; (3)
switch reads to the new store and verify by comparing results; (4) switch writes to the new service,
which becomes the owner; (5) stop syncing and remove the old tables.

**Advanced.** **Dual writes are dangerous** — writing to both stores is the same dual-write problem
as `M15`: one write succeeds, the other fails, and the stores diverge silently. Prefer one
authoritative writer plus CDC replication to the other. If you must dual-write, you need a
reconciliation job comparing the two continuously and an accepted procedure for resolving
differences. Never assume they stay in sync.

#### 4. Verifying with parity testing

**Theory.** Before switching, run both implementations on real traffic and compare their outputs.
Differences you did not expect are bugs — often in your understanding of the old system, which
usually has undocumented behaviour nobody remembers.

**Example.** Shadow mode: for each request, call the old path (serve its result) and the new path
(discard its result), and record any mismatch with the input. Run it for a week. You will find
behaviour in the monolith that nobody knew about — a rounding rule, a special case for one customer,
a quirk relied on by a partner. GitHub's `scientist` library is the canonical implementation.

**Advanced.** Be careful with side effects in shadow mode: the new path must not send emails, charge
cards, or write to shared tables. That usually means a "dry run" mode in the new service, which is
itself worth building because it is useful for testing. And budget for the mismatches being
*correct* — sometimes the old behaviour is a bug, and the migration is the moment someone finally
decides what the rule should be.

### Interview questions

- "Walk me through extracting notifications from a Nest monolith with zero downtime."
- "How do you move the data without losing writes?"
- "What would you extract first and why?"
- "How do you know the new implementation behaves identically?"
