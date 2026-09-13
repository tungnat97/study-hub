[← back to the field index](README.md)

# API & Protocols · Part 1 — HTTP Semantics and REST Design

Nodes `A01`–`A09`.

---

## A01 · HTTP/1.1 semantics

`Beginner` · Requires: — · Unlocks: `A02`, `A03`, `A04`, `A10`, `A19`, `F01`, `M04`

### Preface

HTTP is a text protocol: a request line, headers, a blank line, and an optional body. The server
replies with a status line, headers, a blank line and a body.

Almost everything else in this field is a refinement of that. What matters for an interview is using
the semantics **correctly** — the right method, the right status code — because clients, proxies,
caches and your own retry logic all behave according to them.

### Details

#### 1. Methods and what they mean

**Theory.** `GET` retrieves and must not change anything. `POST` creates or performs an action.
`PUT` replaces a resource entirely at a known URL. `PATCH` partially updates. `DELETE` removes.
`HEAD` is `GET` without a body. `OPTIONS` asks what is allowed (and drives CORS preflight).

**Example.** The difference between `PUT` and `POST` that actually matters: `PUT /users/42` means
"make the resource at this URL be exactly this", so sending it twice leaves the same state. `POST
/users` means "create a new one", so sending it twice creates two. That is why `PUT` is idempotent
and `POST` is not, and why retry logic treats them differently (`A02`).

**Advanced.** A `GET` with side effects is a real bug, not a style issue: browsers prefetch links,
proxies cache responses, crawlers follow URLs, and retry logic repeats them freely. The classic
disaster is `GET /items/42/delete` — a search engine crawler deletes the catalogue. Use `POST` or
`DELETE` for anything that changes state.

#### 2. Status codes worth knowing precisely

**Theory.** `2xx` success, `3xx` redirection, `4xx` the client's fault, `5xx` the server's fault.

**Example.** The distinctions interviewers actually probe:
- **401 vs 403** — 401 means "you are not authenticated" (or your credentials are invalid), and must
  include `WWW-Authenticate`. 403 means "we know who you are and you may not do this". Sending 401
  for an authorisation failure makes clients try to re-authenticate pointlessly.
- **400 vs 422** — 400 means the request is malformed (bad JSON, missing a required parameter). 422
  means it is well-formed but semantically wrong (a date in the past, an unknown product id). Many
  APIs use 400 for both, which is acceptable if it is consistent.
- **404 vs 410** — 404 means "not found"; 410 means "it existed and is permanently gone", which tells
  caches and crawlers to stop asking.
- **409** — a conflict with current state: a duplicate, or a version mismatch (`DB10`).
- **502 vs 503 vs 504** — 502 means an upstream returned something invalid; 503 means this server is
  unavailable or overloaded (pair it with `Retry-After`); 504 means an upstream timed out. Usually
  the **proxy** emits 502 and 504, and your application emits 503.

**Advanced.** Given a 504, you know the request reached your proxy and the proxy gave up waiting for
the backend — so the backend may well have completed the work. That is a partial failure (`M08`) and
it is exactly why non-idempotent operations need idempotency keys (`A09`). Being able to reason from
a status code to "what might have happened" is the senior skill here.

#### 3. Headers that matter

**Theory.** `Content-Type` describes the body being sent; `Accept` states what the client wants;
`Authorization` carries credentials; `Location` points at a created or moved resource; `Retry-After`
tells a client when to try again; `X-Forwarded-For` and `Forwarded` carry the original client
address through proxies.

**Example.** `X-Forwarded-For` is a list that each proxy appends to, and it is **client-controllable**
— an attacker can send a fake one. If you use it for rate limiting or audit logs, take the correct
entry counting back from the right by the number of proxies you actually trust, configured
explicitly. Blindly using the leftmost value lets anyone spoof their IP address (`S13`).

**Advanced.** Header names are case-insensitive but HTTP/2 requires them lowercase; duplicate headers
have per-header rules for whether they combine; and header size limits (typically 8KB total) are
enforced by servers and proxies — an oversized cookie or a very long JWT produces a 431 or a silent
proxy rejection that is puzzling to debug.

#### 4. Statelessness

**Theory.** Each request carries everything needed to process it; the server keeps no per-client
state between requests. This is what lets any instance serve any request, which is what makes
horizontal scaling simple (`F28`).

**Example.** Sessions do not violate this if the session **data** lives in a shared store and the
request carries only an identifier. They do violate it if the session lives in one server's memory —
which is why that design forces sticky sessions (`M07`).

**Advanced.** The trade is that every request re-does work: parsing and verifying the token, loading
the user, checking permissions. That is the cost of scalability, and it is why token verification
should be cheap (a signature check rather than a database lookup) and why per-request user loads are
a common, cacheable hot path.

### Interview questions

- "401 vs 403? 400 vs 422? 502 vs 504?"
- "A client got a 504. Which component produced it and what does it tell you?"
- "Why is a `GET` with side effects a bug?"
- "Can you trust `X-Forwarded-For`?"

---

## A02 · Safety, idempotency and cacheability

`Beginner` · Requires: `A01` · Unlocks: `A03`, `A09`, `M09`

### Preface

Three properties of HTTP methods drive the behaviour of caches, proxies, browsers and retry logic.

**Safe** — does not change state. **Idempotent** — doing it N times has the same effect as doing it
once. **Cacheable** — the response may be stored and reused.

Getting these right is what makes the rest of the system able to retry safely (`M09`).

### Details

#### 1. The table

**Theory.**

| Method | Safe | Idempotent | Cacheable |
|---|---|---|---|
| GET | yes | yes | yes |
| HEAD | yes | yes | yes |
| OPTIONS | yes | yes | no |
| PUT | no | yes | no |
| DELETE | no | yes | no |
| POST | no | **no** | rarely |
| PATCH | no | **not necessarily** | no |

**Example.** Safe implies idempotent (changing nothing twice still changes nothing), but not the
reverse: `DELETE` is idempotent and not safe.

**Advanced.** Idempotent does **not** mean the same response. `DELETE /orders/42` returns 204 the
first time and 404 the second — different responses, identical resulting state. That is still
idempotent, and confusing the two is a common error. Idempotency is about the effect on state, not
about the bytes returned.

#### 2. Why PATCH is the interesting one

**Theory.** `PATCH` applies a change document. Whether it is idempotent depends entirely on what the
document says.

**Example.** `PATCH {"status": "shipped"}` sets a field to a value — idempotent. `PATCH {"op":
"increment", "field": "views"}` — not idempotent, since applying it twice adds two. JSON Merge Patch
(RFC 7386) is typically idempotent; JSON Patch (RFC 6902) with `add` operations on arrays is not.

**Advanced.** Design your `PATCH` to be idempotent whenever you can, because then retries are safe
without any extra machinery. Prefer declaring desired state over describing deltas — the same
principle as `M09`'s "make operations naturally idempotent". Where a counter genuinely must be
incremented, use `POST` with an idempotency key (`A09`).

#### 3. What proxies and clients do with this

**Theory.** Intermediaries rely on these properties. A proxy may retry a failed `GET` automatically.
A browser will warn before re-submitting a `POST`. A CDN caches `GET` and never `POST`.

**Example.** This is why a retry policy checks the method (`M09`): retrying a `GET` that timed out is
free; retrying a `POST` may charge a card twice. Any retry layer — your HTTP client, a service mesh
(`M26`), a load balancer — must respect it, and misconfiguring a mesh to retry `POST` is a genuine
production hazard.

**Advanced.** `POST` responses **can** be cached if they carry explicit freshness headers, but almost
nothing implements it and you should not rely on it. If you need a cacheable query with a body too
large for a URL, the pragmatic options are a `GET` with a hashed query parameter, or the
`QUERY` method that is being standardised — say so rather than inventing a cacheable `POST`.

#### 4. Making the properties true

**Theory.** These are contracts you must uphold, not properties HTTP enforces. If your `GET` writes
to the database, HTTP does not stop you — it just means everything downstream behaves wrongly.

**Example.** Audit worth doing on any API: does any `GET` write? (Logging a view count is the usual
offender — move it to an async event.) Is `PUT` genuinely a full replacement, or does it merge
(making it a `PATCH` in disguise)? Does `DELETE` twice error, when it should be a no-op?

**Advanced.** "Last read timestamp updated on GET" is the most common and most defensible violation.
The clean solution is to record it asynchronously via an event so the read path stays safe and
cacheable, which also removes a write from your hot read path — a performance win as well as a
correctness one.

### Interview questions

- "Is `PATCH` idempotent?"
- "Why can a proxy retry a `GET` but not a `POST`?"
- "`DELETE` returns 404 the second time. Is it still idempotent?"
- "Your `GET` updates a 'last viewed' timestamp. Is that a problem?"

---

## A03 · REST and resource modelling

`Intermediate` · Requires: `A02`, `F02` · Unlocks: `A05`, `A06`, `A07`, `A21`

### Preface

REST models your system as **resources** identified by URLs, manipulated with the standard HTTP
methods. The benefit is predictability: a client that knows one endpoint can guess the others.

Where it gets interesting is actions that are not create, read, update or delete — "cancel this
order", "refund this payment" — and the honest answer is that REST does not model those elegantly,
so you pick a reasonable convention and stay consistent.

### Details

#### 1. Resources and URL structure

**Theory.** URLs name things (nouns), methods do things (verbs). Plural collections, identifiers for
items, nesting to express containment.

**Example.**

```
GET    /orders                 list
POST   /orders                 create
GET    /orders/42              read
PUT    /orders/42              replace
PATCH  /orders/42              partial update
DELETE /orders/42              delete
GET    /orders/42/lines        sub-collection
```

Be consistent about plurals, use lowercase with hyphens, and keep identifiers opaque to the client.

**Advanced.** Stop nesting at one level. `/customers/1/orders/42/lines/7/product` is unusable: it
hard-codes a hierarchy that will change, makes every URL long, and forces the client to know
relationships it does not need. Prefer `/lines/7` with links or ids to related resources. Use nesting
only where the child genuinely cannot exist without the parent **and** you always access it that way.

#### 2. Modelling actions

**Theory.** Three defensible approaches for "cancel an order":
1. **State transition**: `PATCH /orders/42 {"status": "cancelled"}` — most RESTful, and it hides the
   fact that cancelling is a rich operation with rules and side effects.
2. **Sub-resource**: `POST /orders/42/cancellations` — creates a record of the cancellation, which
   is often genuinely what happens in the domain, and gives you somewhere to attach a reason.
3. **Action endpoint**: `POST /orders/42/cancel` — pragmatic, clear, and not strictly REST.

**Example.** Option 2 is often the best-kept secret: if the business has a concept of a
"cancellation" with a reason, a timestamp and an author, then it *is* a resource, and modelling it as
one is both RESTful and more truthful than a status field.

**Advanced.** Be able to argue all three and then pick one, consistently, for the whole API. The
worst outcome is a mixture, where a client cannot predict anything. And note that option 1 hides
important detail: `PATCH {"status": "cancelled"}` implies any status transition is allowed, which is
not true — the API then needs to reject invalid transitions with a clear error, and document the
state machine.

#### 3. Richardson maturity and HATEOAS

**Theory.** Level 0: one endpoint, everything in the body (RPC over HTTP). Level 1: multiple
resources. Level 2: proper methods and status codes — where nearly all real APIs sit and should.
Level 3: hypermedia (HATEOAS), where responses include links telling the client what it can do next.

**Example.** HATEOAS in practice means an order response includes
`"links": [{"rel": "cancel", "href": "/orders/42/cancel"}]`, and the client shows a Cancel button
only when the link is present. In theory this decouples the client from the state machine; in
practice almost no client is written to consume links dynamically, so the effort is usually wasted.

**Advanced.** The honest position — worth stating plainly — is that Level 2 plus good documentation
is what the industry does, and that HATEOAS's benefit (clients discovering capabilities) is real but
rarely realised. The exception is where the link genuinely encodes authorisation-dependent
availability: including a `cancel` link only when this user may cancel this order saves the client
from duplicating your permission rules, which is a real and underused win.

#### 4. When REST is the wrong shape

**Theory.** REST fits resource-oriented CRUD over HTTP. It fits less well for: chatty clients needing
many resources per screen; real-time updates; high-volume internal calls; and complex workflows.

**Example.** The alternatives and when they win: **GraphQL** when clients need to choose their own
data shape (`A15`); **gRPC** for internal, high-volume, strictly-typed calls (`A14`);
**WebSockets/SSE** for server-pushed updates (`A16`); **async job APIs** for long operations
(`A21`); and a **BFF** when one client type needs an aggregated view (`M06`).

**Advanced.** The strongest answer to "is REST the right choice here?" starts by naming the
consumer: a public partner API should be REST (tooling, debuggability, no code generation required);
an internal service-to-service call should probably be gRPC; a mobile app on a slow network may
justify a BFF or GraphQL. The consumer decides, not the fashion.

### Interview questions

- "Model 'user transfers money to another user' as REST."
- "When is REST the wrong shape?"
- "How deep do you nest URLs, and why stop?"
- "Where would you use HATEOAS, honestly?"

---

## A04 · HTTP caching

`Intermediate` · Requires: `A01` · Unlocks: `A20`, `Q09`, `SD05`

### Preface

HTTP has a complete caching system built in, and using it means requests that never reach your
servers at all — the cheapest possible optimisation.

Two mechanisms: **freshness** (the response may be reused for N seconds without asking) and
**validation** (ask the server "has this changed?", and get a cheap 304 if not).

### Details

#### 1. Cache-Control

**Theory.** The main directives: `max-age=N` (fresh for N seconds in any cache), `s-maxage=N` (same,
for shared caches only — overrides `max-age` for CDNs), `public` (may be stored by shared caches),
`private` (browser only — never a CDN), `no-cache` (may be stored but must be revalidated before
use), `no-store` (never store at all), `must-revalidate`, and
`stale-while-revalidate=N` (serve stale for N seconds while refreshing in the background).

**Example.** Typical settings:
- Static asset with a hashed filename: `public, max-age=31536000, immutable` — cache forever, because
  a change produces a new URL.
- A user's own data: `private, no-store` — never in a shared cache.
- A slow-changing public list: `public, max-age=60, stale-while-revalidate=600` — fresh for a minute,
  then served stale while refreshed, so users never wait for a refresh.

**Advanced.** `no-cache` and `no-store` are the pair people confuse. `no-cache` **stores** the
response and revalidates before each use — so you still get 304 savings. `no-store` forbids storage
entirely and is what you want for genuinely sensitive responses. Using `no-cache` where you meant
`no-store` leaves personal data in caches and on disk.

#### 2. Validation: ETag and Last-Modified

**Theory.** The server sends an `ETag` (an opaque version identifier) or `Last-Modified`. The client
sends it back as `If-None-Match` or `If-Modified-Since`. If unchanged, the server replies **304 Not
Modified** with no body.

**Example.** The saving is bandwidth and rendering, not server work — you still had to determine
whether it changed. So the ETag should be cheap to compute: a version column, an `updated_at`
timestamp, or a hash of a small identifying set. Hashing the full serialised response to produce an
ETag saves the client's bandwidth and costs you the full work anyway.

**Advanced.** Strong versus weak validators: `ETag: "abc"` is strong (byte-identical), `W/"abc"` is
weak (semantically equivalent). Strong validators are required for range requests (`A20`), so if you
support resumable downloads you need a strong one. And an ETag that changes when the response is
compressed differently, or when an unrelated field changes, silently destroys cache effectiveness —
compute it from the data, not from the rendered bytes.

#### 3. Vary — and the leak it prevents

**Theory.** `Vary` lists the request headers that affect the response, so a cache stores a separate
entry per combination.

**Example.** The dangerous omission: an endpoint returning different data per user, cached by a CDN
without `Vary: Authorization`, serves the first user's data to everyone. The correct handling is
usually **not** `Vary: Authorization` — a per-user cache entry in a shared cache is near-useless and
risky — but `Cache-Control: private, no-store` so the CDN never stores it at all.

**Advanced.** `Vary: Accept-Encoding` is standard (gzip and identity are different bytes). Avoid
`Vary: User-Agent`, which effectively disables caching because there are millions of distinct values.
Every additional `Vary` header multiplies the number of cache entries and divides your hit rate, so
each one must earn its place.

#### 4. Conditional writes

**Theory.** The same validators work for writes: `If-Match: "abc"` on a `PUT` means "only apply this
if the resource is still at version abc". A mismatch returns **412 Precondition Failed**.

**Example.** This is optimistic concurrency control over HTTP (`DB10`): return an `ETag` on read,
require `If-Match` on write, and a client editing stale data is rejected rather than silently
overwriting someone else's change. `If-None-Match: *` on a `POST` or `PUT` means "only if it does not
already exist" — a clean way to express create-if-absent.

**Advanced.** This is worth proposing unprompted in an API design question, because it solves the
lost-update problem using standard HTTP rather than a custom `version` field in your payload. Map the
ETag to your row's version column (`DB10`) and the two mechanisms are literally the same thing at
different layers.

### Interview questions

- "Explain `no-cache` versus `no-store`."
- "Two users see each other's data through the CDN. What header was wrong?"
- "How do you prevent a client from overwriting someone else's change, using HTTP?"
- "Why is `Vary: User-Agent` a bad idea?"

---

## A05 · Pagination, filtering and sorting

`Intermediate` · Requires: `A03`, `DB20` · Unlocks: `A21`, `SD05`

### Preface

No list endpoint should ever return everything. The question is which pagination style, and the
answer for anything large or live is **cursor-based**, because offset pagination gets slower with
every page and shows users duplicated or missing rows when the data changes.

### Details

#### 1. Offset versus cursor

**Theory.** **Offset**: `?page=5&per_page=20` — simple, allows jumping to any page, and the database
must generate and discard all preceding rows (`DB20`). **Cursor (keyset)**: `?after=<opaque>` —
constant cost per page, stable under concurrent writes, and no random access.

**Example.** The two failures of offset pagination on live data:
- **Performance**: page 5,000 requires scanning 100,000 rows to return 20.
- **Correctness**: a row inserted at the top while the user is reading shifts everything down by one,
  so page 2 repeats the last item of page 1. Deletion causes items to be skipped entirely.

Cursor pagination has neither problem, because the cursor names a position in a stable sort order
rather than a count.

**Advanced.** The cursor must encode **every** column in the sort order, including a unique
tiebreaker, or rows with equal sort values are duplicated or skipped. So for `ORDER BY created_at
DESC`, the cursor is `(created_at, id)` and the query uses a tuple comparison
`WHERE (created_at, id) < ($1, $2)` with a matching index (`DB20`). Missing the tiebreaker is the
classic subtle bug.

#### 2. Designing the cursor

**Theory.** Cursors should be **opaque** to clients: base64-encoded, and treated as a token to send
back unmodified.

**Example.**

```json
{
  "data": [ ... ],
  "page": {
    "next": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wOS0xMy4uLiIsImlkIjoiYWJjIn0",
    "has_more": true
  }
}
```

Opacity matters for two reasons: you can change the encoding without breaking clients, and clients
cannot construct cursors that skip authorisation or produce inefficient queries.

**Advanced.** Consider signing or encrypting the cursor if it contains anything sensitive or if a
crafted cursor could cause an expensive query — it is user input reaching your `WHERE` clause. At
minimum validate it strictly and fail closed on anything unparseable. Also decide what happens when a
cursor is very old and the row it references has been deleted: the tuple comparison still works
(you are comparing values, not looking up a row), which is another advantage of encoding values
rather than a row id alone.

#### 3. Limits and counts

**Theory.** Always have a default page size and a maximum, and enforce both. Total counts are
expensive (`DB20`) and should be optional.

**Example.** `?limit=1000000` must be clamped, or a single request can exhaust memory and the
database. Default 20, maximum 100 is typical. For totals, offer `?include_total=true` explicitly, or
return an estimate, or simply return `has_more` — which is what infinite scroll actually needs.

**Advanced.** A neat trick for `has_more` without a count: request `limit + 1` rows, and if you get
`limit + 1` back, there is another page — return only `limit` of them. One query, no count, exactly
the information the client needs.

#### 4. Filtering and sorting

**Theory.** Filters and sorts are user input that reaches your query, so both need allow-lists.

**Example.** Design simply: `?status=open&created_after=2026-01-01&sort=-created_at`, with a
documented set of filterable fields and sortable fields. A leading `-` for descending is a common
convention. Validate the sort field against an allow-list — you cannot parameterise a column name in
SQL (`S07`), so an unvalidated sort parameter is an injection vector.

**Advanced.** Resist building a general query language in your API (`?filter=status eq 'open' and
total gt 100`). It looks flexible and it means every client query becomes an arbitrary database query
you must make fast — you have exported your query planner to your users. Expose the filters you can
index and support, and add more deliberately. When a client genuinely needs arbitrary queries, that
is a signal for a reporting export or GraphQL with complexity limits (`A15`).

### Interview questions

- "Design pagination for an infinite feed with items being inserted at the top."
- "Why is your cursor opaque and base64-encoded?"
- "How do you return `has_more` without counting?"
- "A client passes `sort=created_at; DROP TABLE`. What protects you?"

---

## A06 · Versioning and evolution

`Advanced` · Requires: `A03`, `M24` · Unlocks: `A22`

### Preface

The best API version is the one you never had to create. Additive, backward-compatible change lets a
single version live for years.

When you genuinely must break something, you run two versions in parallel and retire the old one with
a documented process — which is expensive, and is why avoiding it is worth real effort (`M24`).

### Details

#### 1. Where the version goes

**Theory.** **URL path** (`/v1/orders`) — visible, easy to route, easy to test with curl, and it
arguably violates the idea that a URL identifies a resource. **Header**
(`Accept: application/vnd.example.v2+json`) — cleaner in theory, invisible in logs and browser
address bars, harder to test. **Query parameter** (`?version=2`) — simple and easily forgotten by
clients.

**Example.** The industry has largely settled on the URL path because it is the most operable: you
can route on it at the gateway, see it in logs and metrics, and a developer can try it in a browser.
Stripe is the notable exception, using a date-based version pinned per account — which is elegant and
requires significant infrastructure to maintain many behavioural variants.

**Advanced.** Version the **whole API**, not individual endpoints. Per-endpoint versions produce a
combinatorial matrix of supported states that nobody can test and clients cannot reason about. One
version number for the surface, changed rarely.

#### 2. Evolving without versioning

**Theory.** Most changes can be made compatibly (`M24`): add optional fields, add endpoints, add
enum values (if clients tolerate unknowns), widen validation. Never remove, rename, retype or
tighten.

**Example.** The rule that makes this work is the **tolerant reader**: clients must ignore fields
they do not recognise and not fail on unexpected values. State it in your API documentation from day
one, because retrofitting it is impossible — you cannot make deployed clients tolerant after the
fact.

**Advanced.** The subtle breaking changes worth naming: changing a default (a list that returned
everything now defaults to 100 items); changing the *meaning* of an existing value; changing
ordering that clients relied on even though you never guaranteed it; and tightening validation.
Hyrum's law applies — every observable behaviour of your API is depended upon by someone, whether or
not you documented it.

#### 3. Deprecation with teeth

**Theory.** Announce, measure, migrate, and only then remove. Without measurement you are guessing,
and you will either never remove anything or break someone.

**Example.** A working process:
1. Announce with a date, in the changelog and directly to known consumers.
2. Send `Deprecation: true` and `Sunset: <date>` headers on the old endpoint.
3. Track usage **per client** (by API key), and contact the remaining users individually.
4. **Brownout**: make the old version fail for short windows (one minute per hour, growing) so
   remaining clients notice while there is still time.
5. Remove.

**Advanced.** Brownouts are the technique people have not usually heard of and they work
remarkably well: a client that ignores emails and headers will not ignore intermittent failures, and
the scheduled brownout means they discover it during working hours rather than at the final
shutdown. Mention it and explain why it is kinder than a surprise removal.

#### 4. Clients you cannot upgrade

**Theory.** Mobile applications live in users' pockets for years. Some users never update. Partner
integrations may be maintained by nobody.

**Example.** Therefore: mobile APIs need indefinite backward compatibility, or a **forced upgrade**
mechanism — the app checks a minimum-supported-version endpoint at launch and blocks with an
"update required" screen. That mechanism must exist in version one of the app, because you cannot add
it to already-deployed clients.

**Advanced.** This is a design constraint on your API, not just an operational one: because you can
never fully retire a mobile version, prefer server-driven behaviour (the server decides what to show
and what is allowed) over client-side logic that you would later need to change. Every rule baked
into a mobile client is a rule you cannot change for years.

### Interview questions

- "How do you deprecate a field consumed by an unknown number of mobile clients?"
- "URL, header or query-parameter versioning? Defend your choice."
- "Is adding a field to a response a breaking change?"
- "What is a brownout and why use one?"

---

## A07 · Error contracts

`Intermediate` · Requires: `A03`, `F07` · Unlocks: `A22`

### Preface

Errors are part of your API's contract and deserve as much design attention as the success cases. A
client needs to know: what went wrong, whether it was their fault, whether retrying could help, and
what to show the user.

Consistency matters more than the specific format. One shape, every endpoint.

### Details

#### 1. RFC 7807 problem details

**Theory.** The standard error format, `Content-Type: application/problem+json`, with fields:
`type` (a URI identifying the error kind), `title` (a short human summary), `status` (the HTTP code),
`detail` (a human explanation of this occurrence), `instance` (a URI for this occurrence), plus any
extensions you add.

**Example.**

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 409,
  "detail": "Account balance is 1200 but 2500 was requested.",
  "instance": "/transfers/abc123",
  "code": "INSUFFICIENT_FUNDS",
  "traceId": "4bf92f3577b34da6",
  "balanceMinor": 1200
}
```

**Advanced.** Include a short, stable `code` alongside `type`. Clients switch on the code; the URI is
documentation. Never make clients parse `detail` — it is prose, it will be reworded, and it may be
localised. Changing a code is a breaking change (`M24`); changing a message is not.

#### 2. Field-level validation errors

**Theory.** A validation failure usually concerns several fields, and the client wants to highlight
each one.

**Example.**

```json
{
  "type": ".../validation-error", "status": 422, "code": "VALIDATION_ERROR",
  "errors": [
    { "field": "email", "code": "invalid_format", "message": "must be a valid email" },
    { "field": "lines[0].quantity", "code": "min", "message": "must be at least 1", "min": 1 }
  ]
}
```

Field paths should match the request structure exactly, including array indices, so the client can
map them to inputs without guessing.

**Advanced.** Return **all** validation failures, not the first one. An API that reports one error at
a time makes the user fix and resubmit repeatedly, and it makes automated clients slow to converge.
Most validation libraries collect all errors by default — make sure your configuration does not stop
at the first (`F06`).

#### 3. Retryable versus terminal

**Theory.** Machine clients need to know whether to retry. Communicate it through the status code
and, where helpful, explicitly.

**Example.** 429 and 503 with `Retry-After` say "retry after this long". 500 and 502 and 504 are
worth one or two retries with backoff. 400, 401, 403, 404 and 422 will never succeed on retry.
Returning 500 for a validation failure causes clients to retry pointlessly, amplifying load during an
incident (`M09`).

**Advanced.** `Retry-After` accepts either seconds or an HTTP date, and clients handle both
inconsistently — prefer seconds. And distinguish "retry this exact request" from "retry after
fixing something": a 409 version conflict is not retryable as-is, but is retryable after re-reading
the resource, which is worth saying in the error body.

#### 4. Not leaking internals

**Theory.** Error responses must not contain stack traces, SQL, internal hostnames, library versions
or file paths (`F07`).

**Example.** The pattern: log everything server-side with a correlation id; return the correlation id
and nothing else. Support then asks the user for the reference and finds the exact request. Also be
careful that error *codes* do not leak — distinguishing "no such user" from "wrong password"
confirms which accounts exist (`S13`).

**Advanced.** Also consider what a 403 tells an attacker. For resources where existence is sensitive,
return 404 for both "does not exist" and "exists but you may not see it" (`S09`). It is slightly less
helpful to legitimate users and materially safer, and the fact that it is a deliberate trade — not an
oversight — is what you want to convey.

### Interview questions

- "Design an error response a mobile client can act on programmatically."
- "Why a separate `code` field when you already have an HTTP status?"
- "Do you return one validation error or all of them?"
- "Why might you return 404 instead of 403?"

---

## A08 · Rate limiting and quotas

`Advanced` · Requires: `A01`, `M06` · Unlocks: `S13`, `SD11`

### Preface

Rate limiting protects your service from being overwhelmed — by abuse, by a buggy client, or by a
legitimate customer who suddenly grew.

Two distinct purposes get conflated: **protection** (keep the service up — low limits, applied at the
edge, fail closed) and **fairness or monetisation** (each plan gets its allowance — higher limits,
per customer, with clear headers). Design them separately.

### Details

#### 1. The algorithms

**Theory.**
- **Fixed window** — count per calendar minute. Simplest; allows a double burst at the boundary
  (100 at 10:00:59 and 100 at 10:01:00 = 200 in a second).
- **Sliding window log** — store a timestamp per request and count those in the last N seconds.
  Exact, and memory grows with traffic.
- **Sliding window counter** — weight the previous window by how far into the current one you are.
  A good approximation at low cost, and the usual production choice.
- **Token bucket** — tokens refill at a steady rate up to a capacity; each request takes one. Allows
  controlled bursts, which is what real clients need. The most common answer.
- **Leaky bucket** — requests queue and drain at a fixed rate; smooths output completely.

**Example.** Token bucket for an API: capacity 100, refill 10 per second. A client idle for a while
can burst 100 immediately, then settles to 10 per second. That matches real usage — clients batch —
far better than a hard per-second cap.

**Advanced.** Implementing token bucket in Redis needs atomicity, because read-compute-write races
under concurrency. A Lua script does the whole operation atomically in one round trip (`Q08`): read
the token count and last refill time, compute the refill, decide, write back. That is the concrete
answer to "implement a distributed rate limiter".

#### 2. What to limit by

**Theory.** The key determines who is affected. Options: API key or user id (fairest — requires
authentication), IP address (works for unauthenticated traffic; breaks for users behind shared NAT
and is trivially evaded with many addresses), tenant, or endpoint cost.

**Example.** Layer them: a global limit per IP at the CDN to absorb crude floods; a per-API-key limit
for fairness; and a stricter limit on expensive or sensitive endpoints (login, search, export,
password reset). Different limits for different endpoint costs is the refinement people forget — a
search that costs 50x a simple read should not share a budget with it.

**Advanced.** IPv6 makes per-IP limiting harder: a single user may have a /64 with billions of
addresses, so limit on the /64 prefix rather than the individual address. And for authenticated APIs
prefer limiting by the account, not the key, or a customer can simply create more keys. These
specifics distinguish someone who has operated a limiter from someone who has read about one.

#### 3. What the client sees

**Theory.** Return **429 Too Many Requests** with `Retry-After`, and expose the client's current
state so well-behaved clients can pace themselves rather than probing.

**Example.** The standard headers (now formalised as `RateLimit-*`):

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 1000
RateLimit-Remaining: 0
RateLimit-Reset: 30
```

Publish the limits in your documentation, and return them on **successful** responses too, so clients
can slow down before hitting the wall.

**Advanced.** Watch the interaction with retries: a client that retries a 429 immediately makes
things worse. That is why `Retry-After` matters and why clients should apply jitter (`M09`). It is
also why a rate limiter should be **cheap** to evaluate — if rejecting costs as much as serving, the
limiter does not protect you under a real flood (`M30`).

#### 4. Distributed accuracy

**Theory.** With many instances, a shared counter means a network round trip per request; local
counters are fast and inaccurate.

**Example.** The trade: centralised Redis counters are accurate and add latency plus a dependency on
Redis for every request (and a decision about what to do when Redis is down — fail open, staying
available, or fail closed, staying protected). Local counters with periodic synchronisation are fast
and can overshoot by up to the number of instances.

**Advanced.** For protection limits, approximate is fine — overshooting by 10% does not matter, and
failing open when Redis is unavailable is usually right for a customer-facing API. For **billing**
quotas, accuracy matters and a centralised counter with a durable store is justified. Distinguishing
those two cases is exactly the judgement the question is testing.

### Interview questions

- "Implement a distributed rate limiter. Now make it exact. Now make it cheap."
- "Rate limiting by IP breaks a corporate customer behind one NAT. Now what?"
- "Your Redis is down. Do you fail open or closed?"
- "Which algorithm and why?"

---

## A09 · Idempotency keys

`Advanced` · Requires: `A02`, `M09` · Unlocks: `SD10`

### Preface

`POST` is not idempotent, and networks fail in ways that make the client unsure whether its request
arrived. An idempotency key resolves that: the client generates a unique key per logical operation,
and the server guarantees the operation happens at most once per key.

This is the mechanism that makes retries safe for payments, orders and any other create operation
that must not happen twice. Be able to write the full algorithm.

### Details

#### 1. The protocol

**Theory.** The client sends `Idempotency-Key: <uuid>` with the request. The server: atomically
claims the key; if it was already claimed and completed, returns the stored response; if claimed and
still in progress, returns 409 or waits; otherwise executes the operation and stores the response
against the key.

**Example.** The algorithm in detail:

```
1. read key from header; if absent → 400 (for endpoints requiring it)
2. INSERT INTO idempotency_keys (key, user_id, endpoint, request_hash, state)
   VALUES ($key, $user, $endpoint, $hash, 'in_progress')
   -- unique index on (user_id, endpoint, key)
3. on unique violation:
     load the existing row
     if request_hash differs           → 422 (same key, different payload)
     if state = 'completed'            → return stored status + body
     if state = 'in_progress'          → 409 Conflict, ask the client to retry shortly
4. otherwise: execute the operation
5. UPDATE the row: state='completed', response_status, response_body
6. return the response
```

**Advanced.** The claim in step 2 must be **atomic** — a unique index and catching the violation, not
a `SELECT` followed by an `INSERT` (`DB05`). Two concurrent retries arrive at the same moment
routinely, and a check-then-insert lets both through, which defeats the entire mechanism.

#### 2. Where the key lives

**Theory.** The key record and the operation's effect should commit **together**, so you cannot have
one without the other.

**Example.** If the operation is a database write, put the key row in the same transaction as the
effect — then a crash rolls back both, and a retry executes cleanly. If the operation calls an
external provider, you cannot do that; the safest sequence is: record `in_progress` and commit; call
the provider with the same key; record the result. A crash in the middle leaves `in_progress`, and
recovery means querying the provider for that key's status rather than retrying blindly (`M08`).

**Advanced.** This is why the provider's own idempotency support matters. Stripe, for example,
accepts an `Idempotency-Key` header and returns the original response for a repeat — so passing your
key through means the double-charge cannot happen even if your own record is lost. Chaining the same
key end to end is the design that actually works.

#### 3. Scope, expiry and payload binding

**Theory.** A key must be scoped to a user and an endpoint, so one client cannot interfere with
another's operations or replay a key across different actions. It should expire, or the table grows
forever.

**Example.** Typical policy: keys unique per `(user, endpoint)`, retained 24 hours to a few days,
pruned by a scheduled job (`DB24` partitioning by day makes this a partition drop). Store a hash of
the request body and reject a reused key with a different payload (422) — that indicates a client
bug, and silently returning the old response would hide it.

**Advanced.** The retention window must exceed the longest plausible retry interval, including a
client that retries after a long outage. Too short and a late retry re-executes; too long and the
table is large. A few days is the usual compromise, and the pruning job must be monitored — a
silently-failed cleanup job turns into a table nobody can delete from quickly.

#### 4. Who generates the key

**Theory.** The **client** generates it, once per logical operation, and reuses it for every retry of
that operation. If the server generated it, a retry would get a new one and the mechanism would do
nothing.

**Example.** In a browser, generate the key when the user opens the checkout form (not when they
click), so double-clicking Submit reuses the same key. In a service-to-service call, derive it
deterministically from the message id if you are processing a queue message (`M16`) — then redelivery
of the same message naturally produces the same key, and the two mechanisms reinforce each other.

**Advanced.** Deriving the key from stable business data — `hash(orderId + 'capture')` — is even
better than a random UUID, because it is reproducible after a total loss of client state. The rule is
that the key must identify the **intent**, not the attempt. That sentence is a good one to have ready;
it captures the whole idea.

### Interview questions

- "Write the full server-side algorithm for `Idempotency-Key` on `POST /charges`, including the case
  where the first request crashed halfway."
- "Who generates the key, and when?"
- "Same key, different body. What do you do?"
- "How long do you keep keys, and why?"
