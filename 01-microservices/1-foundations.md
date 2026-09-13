[← back to the field index](README.md)

# Microservices · Part 1 — Foundations & Communication

Nodes `M01`–`M07`.

---

## M01 · Why services at all: monolith → modular monolith → microservices

`Beginner` · Requires: — · Unlocks: `M02`, `M03`, `M22`, `M28`

### Preface

A monolith is one program, one deployment. Microservices split that program into many small
programs that each deploy on their own and talk over the network.

The split does not make your code better. It makes your *teams* independent: a team can ship
without waiting for anyone else. You pay for that with network calls that fail, data spread
across many databases, and far more operating work.

The basic idea to hold on to: **microservices are a way to scale an organisation, not a way to
scale a program.** If you have one team and no deployment pain, you do not need them.

### Details

#### 1. What a monolith actually is (and is not)

**Theory.** A monolith is a single deployable unit. Inside, it can be well organised or a mess —
that is a separate question. People confuse "monolith" with "badly structured", but a monolith
with clear internal modules is a normal, healthy system. Calls inside it are function calls: fast,
reliable, and able to share one database transaction.

**Example.** A typical NestJS application with `UsersModule`, `OrdersModule` and `BillingModule`,
one Postgres database, deployed as one container. When an order is placed, the code writes the
order row and the payment row in a single transaction. If anything fails, the database rolls both
back. There is no partial state to clean up.

**Advanced.** The real limit of a monolith is rarely performance — one server handles far more
traffic than most people assume. The limits that actually bite are: build and test time growing
until a change takes an hour to verify; one team's bad deploy stopping everyone else's release;
and one memory-hungry feature forcing the whole application onto a bigger machine. Notice all
three are about *people and process*, not about code speed.

#### 2. The modular monolith — the step most teams skip

**Theory.** A modular monolith keeps one deployment but enforces hard boundaries inside it: each
module owns its tables, exposes a small public interface, and may not reach into another module's
internals. You get clear ownership without network calls.

**Example.** In a Nest monolith, `BillingModule` exports only a `BillingService` with three
methods. Other modules import the module, never the repository or the entity. An import-rule
linter (for example `eslint-plugin-boundaries`, or Nx module boundary rules) fails the build if
`OrdersModule` imports anything from `billing/internal/*`. Each module has its own database
schema, and cross-module reads go through the exported service.

**Advanced.** A modular monolith is also the correct *preparation* for splitting later. If your
modules already have clean interfaces and separate tables, extracting one into a service becomes
mechanical: replace a function call with an HTTP or message call. If they do not, splitting will
just spread the mess over a network. This is why the common advice is "monolith first" — you
usually do not understand your domain boundaries well enough to draw them on day one, and
boundaries are much cheaper to move inside one codebase.

#### 3. What microservices buy and what they cost

**Theory.** The benefits: independent deployment, independent scaling, team autonomy, technology
choice per service, and failure isolation *if* you design for it. The costs: every in-process call
becomes a network call that can be slow, fail, or be delivered twice; a single transaction becomes
a distributed workflow; debugging spans many systems; and you now need service discovery, tracing,
centralised logging, per-service CI/CD, and an on-call story.

**Example.** In a monolith, "place order" is one transaction. As services it becomes: Order
service writes the order, calls Payment over HTTP, publishes an event that Inventory consumes.
Now you must answer: what if Payment times out but actually succeeded? What if Inventory never
receives the event? What does the customer see meanwhile? None of these questions existed before.

**Advanced.** Availability multiplies downward. If a request needs four services that are each up
99.9% of the time, the end-to-end availability is 0.999⁴ ≈ 99.6% — roughly 35 hours of downtime a
year instead of 9. To get back the lost availability you must add caching, fallbacks, retries and
asynchronous handoffs, all of which add complexity. This is the arithmetic behind "distributed
systems are not free".

#### 4. The distributed monolith — the failure mode to name in an interview

**Theory.** A distributed monolith is a set of services that must be deployed together. You paid
every cost of distribution and received none of the benefit.

**Example.** Signs you have one: a release checklist that says "deploy user-service first, then
order-service"; a shared database that three services write to; a change to one API forcing
simultaneous changes in four repositories; services that cannot start unless five others are
already running; a shared library containing domain models that every service imports and must
upgrade in lockstep.

**Advanced.** The cure is not more services, it is stronger contracts: backward-compatible API and
event changes so old and new versions run side by side (see `M24`), separate data ownership so no
two services write the same table (see `M22`), and asynchronous communication so a downstream
service being briefly absent is tolerable (see `M03`). Practical detection: if you cannot deploy
any single service to production alone, on a Friday, without coordinating, you have one.

#### 5. Conway's law and the organisational angle

**Theory.** Conway's law observes that a system's structure tends to copy the communication
structure of the organisation that built it. If three teams build a compiler, you get a
three-pass compiler. The "inverse Conway manoeuvre" is deliberately shaping teams to get the
architecture you want.

**Example.** A company with one backend team of six that splits its system into twelve services
usually ends up with every engineer touching every service — the boundaries provide no autonomy
and only add friction. The same company with three teams of eight, each owning two or three
services end to end including on-call, gets real independence.

**Advanced.** A useful rule of thumb is that a service should have exactly one owning team, and a
team should own a number of services it can keep in its head. Two teams sharing one service leads
to unclear ownership of incidents and slow decisions. This is why "team size" is a legitimate
technical answer to "how big should a service be?".

### Interview questions

- "When would you *not* use microservices?" — small team, unclear domain boundaries, no CI/CD or
  observability maturity, low traffic, or a product still changing shape weekly.
- "Your team of six has one Nest monolith deploying three times a day with no pain. Why split it?"
  The honest answer is: you probably should not. Name the specific trigger that would change your
  mind — a component with wildly different scaling needs, a compliance boundary, or team growth.
- "What is a distributed monolith and how would you detect you have one?"
- "You are asked to migrate to microservices by a manager. What do you ask first?" — what problem
  are we solving, and can we solve it with modules first.

---

## M02 · Service decomposition: bounded contexts and aggregates

`Intermediate` · Requires: `M01` · Unlocks: `M17`, `M22`, `M24`, `M29`

### Preface

Once you decide to split, the only hard question is *where the lines go*. Draw them wrong and
every feature will touch five services.

The rule that works: split by **business capability**, not by technical layer and not by database
table. A service should own a complete piece of the business — "everything about payments" — so
that a typical change stays inside it.

The vocabulary comes from Domain-Driven Design (DDD): a **bounded context** is a part of the
business where words have one precise meaning, and an **aggregate** is a cluster of data that must
stay consistent together.

### Details

#### 1. Bounded context: the same word means different things

**Theory.** A bounded context is a boundary inside which a term has a single, clear definition.
The same real-world thing appears in several contexts with different shapes, and that is correct,
not duplication to be removed.

**Example.** "Customer" in three contexts of an e-commerce system:
- Sales: name, company, discount tier, account manager.
- Shipping: name, address, delivery instructions, phone.
- Billing: legal entity, tax id, payment methods, credit limit.

Trying to build one shared `Customer` service that serves all three produces a huge model that
every team must change and no team owns. Three contexts, each with its own customer record keyed
by a shared customer id, works far better.

**Advanced.** How contexts relate is itself a design decision — DDD calls this context mapping.
The useful ones in practice: **anti-corruption layer** (you translate the other context's model
into your own at the boundary, so their changes do not leak into your code — this is what you do
around a legacy system or a third-party API); **shared kernel** (a small model both share and both
must agree to change — keep it tiny or avoid it); **conformist** (you accept their model as-is,
usually because you have no leverage, like a payment provider's webhook format).

#### 2. Aggregates: the unit of consistency

**Theory.** An aggregate is a small group of objects treated as one unit for changes. It has a
root, and outside code may only reference the root. The key rule: **one transaction changes one
aggregate.** Anything crossing aggregates becomes eventual consistency.

**Example.** `Order` is an aggregate root; `OrderLine` items live inside it. You never load or
change an order line independently — you load the order, change it, save it, and the invariant
"total equals the sum of lines" is enforced in one place, in one transaction. `Customer` is a
separate aggregate. So "place order and deduct customer loyalty points" spans two aggregates and
cannot be one transaction if they live in different services.

**Advanced.** Aggregate size is a real trade-off. Large aggregates make invariants easy but create
lock contention — if `Warehouse` is one aggregate containing every item, two concurrent
item updates conflict. Small aggregates scale but push invariants into asynchronous processes. A
common fix for a contended aggregate is to shrink it and accept an eventually-consistent check:
allow both updates, detect the violation afterwards, and compensate. Designing aggregate
boundaries is effectively designing where you are willing to be eventually consistent.

#### 3. Decompose by capability, not by layer or entity

**Theory.** Three common wrong splits: by technical layer (an "API service", a "business logic
service", a "database service" — every feature touches all three); by entity (a "User service", an
"Order service", a "Product service" — sounds tidy, but features cut across them); and by team
convenience without regard to data.

**Example.** Good split for an e-commerce system, by capability: Catalogue (browse, search),
Ordering (cart, checkout, order lifecycle), Payments (charges, refunds, reconciliation), Inventory
(stock levels, reservations), Fulfilment (picking, shipping, tracking), Identity (accounts, auth).
Test it with a feature: "add gift wrapping at checkout" touches Ordering and maybe Fulfilment —
two services, not six. Compare with an entity split, where the same feature touches Order, Product,
Price, Customer and Shipment services.

**Advanced.** A practical technique is **event storming**: put every business event on a wall in
time order ("order placed", "payment authorised", "stock reserved", "parcel dispatched"), then
group events that tend to change together and are triggered by the same people. The groups are
your candidate contexts. Another signal is data gravity: if two pieces of data are always read and
written together in the same transaction, they belong in the same service.

#### 4. Ubiquitous language

**Theory.** Inside a context, the code, the database, the API and the conversations with the
business all use the same words. If the business says "reservation" and the code says
`TempOrderHold`, every conversation needs translation and misunderstandings become bugs.

**Example.** If the domain experts distinguish "authorisation" (money reserved on a card) from
"capture" (money actually taken), your code must have both words, and your table must not be
called `payments` with a `status` column that blurs them. Interviewers notice when your naming
reflects a real domain.

**Advanced.** When one word means two things to two groups of people, that is almost always a
signal you have found a context boundary, not that someone is using the word wrongly. "Shipment"
to the warehouse means a box; to finance it means a billable event. Two contexts.

#### 5. How big is too small

**Theory.** There is no line count. The working tests are: can one team own it; does a typical
change stay inside it; does it own its own data; can it be deployed and rolled back alone; is it
worth a separate pipeline, dashboard and on-call page.

**Example.** A "currency formatting service" fails every test — it holds no data, changes rarely,
and adds a network hop to every price display. It should be a library. A "notification service"
passes — it owns templates and delivery state, has its own scaling profile, and its failure should
not stop checkout.

**Advanced.** "Nanoservices" cost more than they look: each one needs a repository, pipeline,
dashboards, alerts, dependency upgrades, security patching and an on-call owner. Multiply that
fixed cost by the number of services before splitting. Where a shared capability really is needed
by everyone (validation rules, currency maths), a versioned library is usually the right answer —
accepting that a library update requires each service to redeploy, which is exactly the coupling
you avoid only if the library is small and stable.

### Interview questions

- "Split an e-commerce monolith. Where do the seams go and why?" Talk capabilities and walk one
  feature through your split to prove it holds.
- "Order and Inventory both need `Product`. How do you avoid a shared Product service everyone
  blocks on?" — each context keeps the fields it needs, kept up to date by events; translate at
  the boundary with an anti-corruption layer.
- "How do you size a service? What is too small?"
- "What is an aggregate and why does it matter for microservices?" — it is the transaction
  boundary, and therefore the boundary beyond which you must accept eventual consistency.

---

## M03 · Communication styles: synchronous vs asynchronous

`Beginner` · Requires: `M01` · Unlocks: `M04`, `M06`, `M08`, `M17`, `Q10`

### Preface

Two services can talk in two ways. **Synchronous**: A calls B and waits for the answer, like a
phone call. **Asynchronous**: A leaves a message and carries on, like sending an email.

The choice is not about speed. It is about what happens when the other side is down. With a
synchronous call, if B is down, A fails too. With a message, if B is down, the message waits and
B catches up later.

The basic guidance: use synchronous calls when you need the answer to continue, and messages for
everything else.

### Details

#### 1. Availability couples through synchronous calls

**Theory.** A synchronous dependency means your uptime cannot exceed the uptime of everything you
call. Availability multiplies along the chain.

**Example.** Checkout calls Payment, which calls Fraud, which calls a third-party score API. Each
is up 99.9%. Combined: 0.999³ ≈ 99.7%. Add a fourth hop and you are under 99.6%. Customers see
every one of those failures as "checkout is broken".

**Advanced.** Asynchronous calls break the chain: if "send confirmation email" is a message rather
than a call, the email service being down for an hour does not fail a single checkout. The
practical design move is to identify the **minimum set of things that must be true before you can
answer the user** — usually "the order is safely recorded and the money is authorised" — and make
everything else asynchronous.

#### 2. Commands, events and queries

**Theory.** Three different kinds of message, often confused:
- **Command**: "do this" — one intended handler, and you generally care whether it succeeded.
  `ChargeCard`.
- **Event**: "this happened" — a fact about the past, broadcast; the publisher does not know or
  care who listens. `OrderPlaced`.
- **Query**: "tell me something" — no side effects, needs an answer now.

**Example.** Checkout sends a *command* to Payment ("charge this card") because it needs the
outcome. After the order is saved it publishes an *event* `OrderPlaced`; Email, Analytics,
Inventory and Loyalty each react on their own. Adding a fifth consumer later requires no change to
Checkout at all — that is the payoff of events.

**Advanced.** Naming matters more than it looks. Publishing an event called `SendConfirmationEmail`
is a command wearing an event's clothes: it names what the consumer should do, which recreates the
coupling you were trying to remove. Name events after facts in the past tense (`OrderPlaced`,
`PaymentAuthorised`) and let consumers decide what to do.

#### 3. What asynchronous actually costs

**Theory.** You trade immediate consistency for availability. After an asynchronous handoff the
system is *temporarily wrong* — the order exists but the invoice does not yet. You must design
the user experience and the error handling around that window.

**Example.** A user uploads a profile photo; processing happens asynchronously. If the UI shows
the old photo after upload, users re-upload and complain. The fixes are all product decisions:
show the local file optimistically, show a "processing" state, or block until done for this one
case. The technical choice creates a user-experience obligation.

**Advanced.** Asynchronous systems are also harder to debug: there is no stack trace across the
boundary, ordering is not guaranteed, messages can be delivered twice, and a failure surfaces
minutes later in a different service. That is why the infrastructure around messages —
correlation ids in every message, tracing across the broker, dead-letter queues, and idempotent
consumers — is not optional extra work. It is the cost of admission. See `Q10`, `Q16` and `M16`.

#### 4. Choosing per interaction, not per system

**Theory.** This is not an architectural style you pick once. Each interaction gets its own answer,
based on whether the user is waiting for the result and what happens if it is delayed.

**Example.** Checkout, decided step by step:
| Step | Style | Why |
|---|---|---|
| Validate cart | sync, in-process | needed to respond |
| Reserve stock | sync call | must not sell what we do not have |
| Authorise payment | sync call | user must know now if the card failed |
| Save order | sync, local transaction | the source of truth |
| Send confirmation email | async event | user does not wait for it; email being down must not fail checkout |
| Update analytics | async event | no user impact |
| Start fulfilment | async event | warehouse works in minutes, not milliseconds |

**Advanced.** A common senior-level refinement is to shrink the synchronous part further. Some
large systems do not even authorise payment synchronously: they accept the order, return "we are
processing", and handle the payment asynchronously with a notification. That maximises
availability and conversion at the cost of a more complex user experience and a cancellation path
when payment later fails. Being able to argue both sides is the point.

### Interview questions

- "Checkout must reserve stock, charge a card and send an email. Which calls are synchronous,
  which asynchronous, and why?"
- "If A → B → C → D are all synchronous at 99.9% each, what is the effective availability?"
  About 99.6%.
- "What is the difference between an event and a command? Why does it matter?"
- "What do you lose by making something asynchronous?" — immediate consistency, simple debugging,
  and guaranteed ordering; you gain availability and decoupling.

---

## M04 · Service-to-service protocols: REST, gRPC, messaging

`Intermediate` · Requires: `M03`, `A01` · Unlocks: `M24`, `M26`

### Preface

Once you have decided a call is synchronous, you choose how it travels. The realistic options are
HTTP with JSON (REST), gRPC with Protocol Buffers, or a message broker for asynchronous work.

Rough guidance: **REST at the edge** where many different clients and humans need to use it,
**gRPC inside** where you control both sides and want speed and strict contracts, and
**messaging** whenever you do not need an answer immediately.

### Details

#### 1. REST over HTTP/JSON

**Theory.** Text-based, readable, works with every language, every proxy, every browser, and every
debugging tool. No code generation needed. The cost is size and parsing speed, and that the
"contract" is only as strong as your discipline — JSON has no built-in schema.

**Example.** A partner integration endpoint, `POST /v1/orders`, documented with OpenAPI. A partner
can test it with `curl` in thirty seconds. That accessibility is worth more than any efficiency
gain at the edge.

**Advanced.** JSON parsing and serialising is frequently the single largest CPU cost in a Node
service under load, ahead of your business logic. If a hot internal path moves hundreds of
thousands of messages, that cost is real and measurable — it is one of the honest arguments for a
binary format internally. Measure before assuming.

#### 2. gRPC and Protocol Buffers

**Theory.** You define messages and service methods in a `.proto` file, and generate client and
server code for each language. Messages travel in a compact binary format over HTTP/2. You get a
machine-checked contract, smaller payloads, faster parsing, streaming in both directions, and
deadlines that propagate through the call chain.

**Example.** A `.proto` shared between a Nest service and a Go service:

```proto
service Pricing {
  rpc GetPrice(GetPriceRequest) returns (Price);
}
message GetPriceRequest {
  string sku = 1;
  string currency = 2;
}
```

Both sides generate typed clients. If someone removes `currency`, the build breaks rather than
production.

**Advanced.** The compatibility rules are strict and worth memorising, because breaking them
corrupts data silently rather than failing loudly: never change a field's number, never change a
field's type, never reuse a number from a deleted field (mark it `reserved`). All fields are
optional on the wire, so a consumer must handle absence. Also note gRPC's load-balancing problem:
it holds one long-lived HTTP/2 connection, so a simple TCP (layer 4) load balancer pins all
requests to one server — you need a layer 7 proxy or client-side balancing (see `M07`).

#### 3. Deadlines instead of timeouts

**Theory.** A timeout says "I give up after 2 seconds". A deadline says "this work is worthless
after 14:05:03.200" and travels with the call, so every service downstream knows how much time
remains and stops work that can no longer be used.

**Example.** The gateway sets a 3-second deadline. It spends 1 second, then calls Orders with
2 seconds remaining. Orders spends 0.5s and calls Pricing with 1.5s remaining. If Pricing is slow,
it gives up at the right moment instead of doing work whose result will be thrown away. In gRPC
this is built in; over HTTP you propagate a header yourself and convert it to a local timeout.

**Advanced.** Without deadline propagation you get the classic waste pattern: the user has already
seen a timeout error, but five services are still busy computing an answer for them, consuming
capacity that live requests need. Under overload this turns a slowdown into a collapse. Deadline
propagation is one of the cheapest resilience wins available.

#### 4. Choosing between them

**Theory.** Decide on: who the consumer is, how strict the contract must be, how much traffic there
is, whether you need streaming, and how debuggable it must be.

**Example.** A realistic mixed setup: browser and mobile apps talk REST/JSON to a gateway; the
gateway talks gRPC to internal services; internal services publish events to Kafka for anything
that does not need an answer; a partner-facing API is REST with OpenAPI and an API key.

**Advanced.** GraphQL sits in a different place: it is for clients that need to pick their own
shape of data, not for service-to-service calls. Using GraphQL between backend services usually
adds the resolver N+1 problem and loses gRPC's type safety without giving you anything you need.
When asked "which protocol", the strongest answer names the consumer first.

### Interview questions

- "Why is gRPC often better inside the mesh but rarely at the public edge?"
- "How do deadlines propagate in gRPC and why does that beat per-hop timeouts?"
- "What are the rules for evolving a protobuf message safely?"
- "Your gRPC traffic is all landing on one pod. Why?"

---

## M05 · Service discovery

`Intermediate` · Requires: `M03` · Unlocks: `M07`, `M26`

### Preface

Services move. Containers restart with new IP addresses, instances scale up and down, machines
die. So a caller cannot hard-code an address. Service discovery is how a caller finds a currently
healthy instance of the service it wants.

In Kubernetes this is mostly solved for you: you call a stable name like `http://orders`, and the
platform keeps the list of healthy addresses behind it up to date.

### Details

#### 1. Client-side vs server-side discovery

**Theory.** **Client-side**: the caller asks a registry for the list of instances and picks one
itself. It can balance intelligently, but every client needs the logic. **Server-side**: the caller
sends to a fixed address (a load balancer or the platform's virtual IP) which forwards to a healthy
instance. Simpler for callers, and the default in Kubernetes.

**Example.** Server-side: in Kubernetes, `orders.default.svc.cluster.local` resolves to a stable
virtual IP; kube-proxy (or its replacement) forwards to one of the pods currently listed as ready.
Client-side: a Spring application using Eureka fetches the instance list and applies its own
balancing.

**Advanced.** Client-side balancing is making a comeback for gRPC precisely because server-side
layer-4 balancing distributes long-lived HTTP/2 connections badly. A service mesh (see `M26`)
gives you client-side balancing without writing it into every application, by putting a proxy
next to each instance.

#### 2. The registry and health checking

**Theory.** A registry holds "which instances exist and which are healthy". Instances register on
start and deregister on shutdown; health is checked actively (the registry probes) or passively
(the instance sends heartbeats). Unhealthy instances must be removed quickly, but not so eagerly
that a brief hiccup empties the pool.

**Example.** In Kubernetes, the readiness probe is the health signal. A pod that fails readiness is
removed from the Service's endpoint list and stops receiving traffic — without being restarted.
That is the important difference from the liveness probe, which restarts the container.

**Advanced.** Removing unhealthy instances has a dangerous edge case. If a downstream dependency is
slow and every instance's health check fails at once, you remove *all* instances and cause a total
outage from a partial problem. Real registries guard against this with a "panic threshold": if
more than a set share of instances look unhealthy, ignore health and send to everyone, on the
grounds that degraded service beats none. This is why health checks must test *your own* health,
never your dependencies' (see `O06`).

#### 3. Stale addresses and shutdown

**Theory.** Discovery information is cached in several places — DNS caches, client connection
pools, proxy tables. When a pod stops, those caches take time to update, so traffic keeps arriving
at an instance that is shutting down.

**Example.** The standard sequence that avoids dropped requests when a pod terminates:
1. Kubernetes marks the pod as terminating and starts removing it from endpoints.
2. In parallel, it runs the `preStop` hook and sends `SIGTERM`.
3. The `preStop` hook sleeps a few seconds — this is the crucial step, giving the endpoint removal
   time to propagate to every proxy and client.
4. The app stops accepting new requests, finishes in-flight ones, closes its database pool, exits.
5. `terminationGracePeriodSeconds` must be longer than steps 3 and 4 together, or the pod is
   killed mid-request.

**Advanced.** Note the counter-intuitive part: the pod must keep serving during the `preStop`
sleep. Requests are still arriving from clients whose caches have not updated. Closing the server
immediately on `SIGTERM` is the most common cause of errors during deploys. Also watch HTTP
keep-alive: a client with an open connection will keep using it regardless of discovery, so
servers should send `Connection: close` during draining or set a maximum connection age.

#### 4. DNS as discovery, and its limits

**Theory.** DNS is the simplest discovery mechanism and is universally supported, but it carries
no health or load information and clients cache it — sometimes ignoring your TTL entirely.

**Example.** A classic production incident: a database failover updates a DNS record to point at
the new primary, but the application keeps connecting to the old address for minutes because the
JVM cached the resolution (historically forever, controlled by `networkaddress.cache.ttl`), or a
connection pool simply never re-resolves for existing connections.

**Advanced.** In Kubernetes, DNS resolution is also a hidden latency source. The `ndots:5` setting
in the default `resolv.conf` means a name like `orders` is tried against several search domains
before resolving, so a single lookup can cost multiple round trips. Fully-qualified names ending
with a dot, or tuning `ndots`, removes it. Worth knowing as an example of "the platform default
was the performance bug".

### Interview questions

- "A pod is terminating but still receives traffic. Walk through why and how to fix it."
- "What is the difference between a readiness probe and a liveness probe?"
- "Why should a health check not check the database?"
- "Client-side or server-side discovery — when does the choice matter?"

---

## M06 · API gateway and backend-for-frontend

`Intermediate` · Requires: `M03` · Unlocks: `M26`, `A08`, `S13`

### Preface

A gateway is the single front door to your services. Outside clients talk to it; it forwards to the
right service inside.

It exists so that concerns every request needs — TLS, authentication, rate limiting, routing,
logging — live in one place instead of being reimplemented by every service. The danger is that it
gradually absorbs business logic and becomes a monolith that every team must change.

A **backend-for-frontend (BFF)** is a variation: instead of one general-purpose gateway, each type
of client gets its own tailored backend.

### Details

#### 1. What belongs in a gateway

**Theory.** Cross-cutting, request-shaped concerns that are identical for all services: TLS
termination, routing by path or host, authentication (verifying the token) and passing on identity,
coarse rate limiting, request size limits, request ids, access logging, CORS, and protocol
translation (outside REST to inside gRPC).

**Example.** A request arrives with a JWT. The gateway verifies the signature against the identity
provider's public keys, rejects it if invalid with a 401, and forwards the request to the Orders
service with a header carrying the verified user id and tenant. Orders never needs the signing
keys, but still decides for itself whether *this* user may read *this* order.

**Advanced.** Note the split: authentication (who are you) at the gateway, authorisation (may you
do this) in the service. Business authorisation depends on data only the service has, so pushing
it into the gateway either duplicates data or forces the gateway to make calls. A defence-in-depth
point too: services should not blindly trust an identity header, because anything inside the
network could set it. Use mutual TLS between the gateway and services, or pass the signed token
through and verify it again.

#### 2. What must never go in a gateway

**Theory.** Business rules, per-feature logic, and anything requiring frequent change by product
teams. Every such addition makes the gateway a shared bottleneck — a component many teams must
change and no team owns, with the highest blast radius in the system.

**Example.** A "temporary" discount rule added to the gateway because it was quick. Six months
later the gateway holds pricing logic, forty routes with special cases, and three teams queueing to
change the riskiest component in production.

**Advanced.** Gateway configuration deserves the same treatment as code: version control, review,
staged rollout, and the ability to roll back. A bad gateway config push is a total outage, unlike
a bad deploy of one service. This is why teams that run heavy gateway logic end up building canary
deployment for gateway config too — at which point a plain service would have been simpler.

#### 3. Backend-for-frontend

**Theory.** Different clients need different data shapes. A mobile app on a slow network wants one
small aggregated response; a web dashboard wants many fields; a partner wants a stable contract.
One API serving all three becomes a compromise that suits none. A BFF is a thin service per client
type, owned by the team that builds that client.

**Example.** The mobile BFF exposes `GET /home`, which internally calls Orders, Recommendations and
Promotions and returns one small payload with exactly the fifteen fields the home screen shows. The
web BFF exposes richer endpoints. Each evolves at the pace of its own client, without negotiating
with the other.

**Advanced.** BFFs can duplicate logic across clients — that is an accepted cost, and it is
contained because the duplicated part is presentation shaping, not business rules. The deeper risk
is a BFF quietly growing business logic; it must stay a composition and shaping layer. GraphQL is
one way to implement a BFF where clients choose fields, at the cost of query complexity and
authorisation being harder (see `A15`).

#### 4. Aggregation: latency and failure

**Theory.** When a gateway or BFF calls several services to build one response, it inherits all of
their failure modes and the latency of the slowest one.

**Example.** `GET /home` calls three services in parallel. Latency is the maximum of the three, not
the sum — so call them concurrently, never in a loop. For failure, decide per dependency: if
Recommendations fails, return the page without recommendations (degrade); if Orders fails, the page
is meaningless, so return an error. Each dependency needs its own timeout, shorter than the overall
deadline, and a defined fallback.

**Advanced.** This is where partial responses and tail latency meet. If each of three services has
a 1% chance of exceeding 500ms, the chance that *at least one* does is close to 3% — so the
aggregated endpoint's p99 is far worse than any single service's. Mitigations: aggressive
per-dependency timeouts with fallbacks, caching the slow-changing parts, and precomputing the
aggregate rather than assembling it per request (see `M19`, `SD05`).

### Interview questions

- "What belongs in the gateway and what must never go there?"
- "You aggregate six downstream calls in the gateway — what is your failure and latency story?"
- "Where do you do authentication and where authorisation? Why the split?"
- "When is a BFF worth the extra service?"

---

## M07 · Load balancing

`Intermediate` · Requires: `M05` · Unlocks: `M10`, `M31`, `C14`

### Preface

A load balancer spreads requests across several instances so that no single one is overwhelmed,
and so a dead instance stops receiving traffic.

Two layers matter. **Layer 4** balances TCP connections — fast and protocol-ignorant. **Layer 7**
understands HTTP, so it can route by path or header, retry failed requests, and balance individual
requests rather than whole connections.

### Details

#### 1. Layer 4 versus layer 7

**Theory.** L4 picks a backend when the connection is opened and every byte on that connection
goes to the same place. L7 parses HTTP and can make a decision per request, which enables
path-based routing, header-based canaries, retries, and per-request balancing.

**Example.** An AWS Network Load Balancer (L4) in front of a service that uses HTTP/1.1 with
keep-alive: each client connection sticks to one backend for its lifetime, which is usually fine
because there are many short-lived connections. An Application Load Balancer or Envoy (L7) can send
`/api/v2/*` to the new service and 5% of requests carrying a specific header to a canary.

**Advanced.** The failure case to remember: **HTTP/2 and gRPC use one long-lived connection that
carries many requests**. Under an L4 balancer, one connection means one backend, so a client sends
thousands of requests to a single pod while others idle. Symptoms are wildly uneven CPU across
pods. Fixes: an L7 proxy that balances per request, client-side balancing that opens connections to
all backends, or a maximum connection age that forces periodic re-balancing.

#### 2. Balancing algorithms

**Theory.** **Round robin** — next in line; simple, assumes all requests and all servers are equal.
**Least connections** — send to whoever has fewest in flight; adapts to uneven request costs.
**Least response time / EWMA** — track observed latency per backend and prefer the fast ones.
**Power of two choices** — pick two backends at random and send to the less loaded one; almost as
good as tracking everything, at a fraction of the cost and with no herd effect.
**Consistent hashing** — the same key always goes to the same backend, for cache locality or
session affinity (see `M32`).

**Example.** A service where most requests take 5ms but exports take 3 seconds. Round robin will
hand a second export to a pod already running one, while an idle pod waits. Least connections fixes
this without any configuration of request types.

**Advanced.** Choosing "the least loaded backend" globally has a herd problem: many balancers all
detect the same idle server and flood it simultaneously, so it becomes the slowest, and they all
move away together — load oscillates. Power-of-two-choices solves this by adding randomness. If an
interviewer asks why round robin is not enough, the answer is heterogeneous request cost and
heterogeneous servers; if they push further, oscillation is the advanced answer.

#### 3. Health checks and outlier detection

**Theory.** A balancer must stop sending to instances that are failing. Active checks probe an
endpoint; passive checks (outlier detection) watch real traffic for errors or slow responses and
temporarily eject a backend.

**Example.** Envoy's outlier detection: if a backend returns five consecutive 5xx responses, eject
it for 30 seconds, then let one request through to test it. That catches "instance is up but
broken" — a state active health checks often miss because `/health` still returns 200.

**Advanced.** Ejecting too aggressively can remove the whole pool during a shared incident, so
implementations cap the proportion that may be ejected (typically around 50%). This is the same
"panic mode" idea as in discovery — partial service is better than none.

#### 4. Sticky sessions and why to avoid them

**Theory.** Stickiness pins a user to one instance, usually via a cookie or a hashed IP. It is
needed when the server holds per-user state in memory. It also undermines even load distribution,
makes deploys disruptive, and prevents scaling from helping the users already pinned to a busy
instance.

**Example.** A WebSocket chat server holds connections in memory, so a user must stay on one pod.
The better design is not stickiness but externalising state: a shared Redis pub/sub backplane so
any pod can deliver a message to any user's connection, with stickiness only for the connection
itself (see `F16`).

**Advanced.** Stickiness also interacts badly with autoscaling: scaling out adds capacity that
existing sessions never use, so the busy instances stay busy. And removing an instance drops its
sessions. If you must have affinity, prefer consistent hashing over cookies — when an instance
disappears, only its share of keys moves, instead of reshuffling everyone.

### Interview questions

- "Round robin gives you an unbalanced fleet. Why, and what do you switch to?"
- "Why does gRPC behind a layer 4 load balancer distribute traffic badly?"
- "How does a load balancer detect an instance that is up but broken?"
- "What breaks when you enable sticky sessions?"
