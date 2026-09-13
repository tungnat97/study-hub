[← back to the field index](README.md)

# System Design · Part 2 — Scale Patterns, Correctness and Migration

Nodes `SD09`–`SD14`.

---

## SD09 · Skew, hot spots and fan-out

`Advanced` · Requires: `SD05`, `SD06`, `Q14`, `M32` · Unlocks: `SD12`

### Preface

Real workloads are never evenly distributed. A few users, products or tenants account for most of the
traffic, and the average tells you nothing about them.

The canonical version is the **celebrity problem**: a design that works for a user with 200 followers
fails for one with 200 million. Every large system has an answer for it, and interviewers ask because
the naive design breaks in a specific, explainable way.

### Details

#### 1. Power laws are the norm

**Theory.** Traffic, followers, tenant size and object popularity all follow power-law distributions:
a small fraction accounts for a large majority. Designing for the mean guarantees failure at the tail.

**Example.** The consequences across a system: one tenant is 40% of your data, so hashing by tenant
creates a hot shard (`DB25`); one product is 30% of requests, so its cache key is a hot key (`Q04`);
one user has millions of followers, so fan-out on write for them is millions of writes. Each is the
same phenomenon at a different layer.

**Advanced.** The design instinct to demonstrate: whenever you draw a partitioning or distribution
scheme, immediately ask "what happens when one key is a thousand times bigger than the others?" That
question finds the hot-spot problem before it is built, and asking it unprompted is a strong signal.

#### 2. Fan-out on write versus on read

**Theory.** Two ways to build a feed.
**Fan-out on write (push)** — when a post is created, write it into every follower's timeline. Reads
are a single key lookup; writes are amplified by the follower count.
**Fan-out on read (pull)** — store posts once; at read time, gather from everyone the user follows.
Writes are cheap; reads are an expensive scatter-gather.

**Example.** With a 100:1 read ratio, push wins overwhelmingly for normal users: one write becomes 200
writes, and 200 reads become one lookup each. For a celebrity with 100 million followers, a single post
becomes 100 million writes — minutes of work, and a write spike that disrupts everything else.

**Advanced.** The standard answer is the **hybrid**: push for normal accounts, and for accounts above
a follower threshold, do not fan out — instead, at read time, merge the small number of
celebrity accounts a user follows into their precomputed timeline. Reads become "one lookup plus a
handful of small queries", which is bounded because nobody follows thousands of celebrities. Being
able to describe both halves and the threshold is the complete answer.

#### 3. Mitigating hot keys and hot partitions

**Theory.** The general techniques, applicable at every layer: **cache it closer** (a local cache in
front of the shared one, `Q09`); **split it** (sharded counters, salted keys, `M32`); **dedicate
capacity** (a separate shard or fleet for the largest tenants, `M29`); or **shed** (rate-limit the
hot object, `A08`).

**Example.** For a viral product page: a local in-process cache makes most requests never leave the
pod — the most effective single fix, because a hot key is by definition requested constantly and
therefore cached constantly. For a hot write key (a counter), split it into N rows and sum on read
(`SD06`).

**Advanced.** Detection matters as much as mitigation, and is usually missing: you need per-key or
top-N metrics to identify the hot key during an incident rather than guessing. A count-min sketch
(`DB33`) gives approximate top-N cheaply. Having that instrumentation before the incident is what
turns a two-hour investigation into a two-minute one.

#### 4. Tenant skew

**Theory.** In a multi-tenant system, one customer being orders of magnitude larger affects
partitioning, performance, cost and fairness (`M29`).

**Example.** The handling: partition by a finer key than tenant where ordering allows (`Q14`);
give the largest tenants dedicated resources — their own shard, their own worker pool — which also
becomes a product feature ("dedicated infrastructure"); apply per-tenant quotas (`Q21`); and use
shuffle sharding so one tenant's spike affects only a few others (`M31`).

**Advanced.** Routing by an explicit **directory** rather than by pure hashing (`DB25`) is what makes
this manageable: a lookup table mapping tenant to shard lets you move one large tenant without
rehashing anything. Building that indirection early, even when it maps everything to one shard, is
cheap, and retrofitting it is not.

### Interview questions

- "Design a news feed for 300M users where some have 100M followers."
- "One key gets 40% of your traffic. Fix it."
- "How do you detect a hot key during an incident?"
- "One tenant is a thousand times larger than the rest. What changes?"

---

## SD10 · Correctness in user-facing flows

`Expert` · Requires: `SD06`, `SD08`, `M14`, `M16`, `A09`, `DB38`, `Q21`, `C11` · Unlocks: `SD15`

### Preface

Payments, bookings and inventory are where "eventually consistent and probably fine" stops being
acceptable. Money must not be created or destroyed; a seat must not be sold twice; a retry must not
charge twice.

This node combines idempotency (`A09`), sagas (`M14`), ledgers (`DB38`) and locking (`DB09`) into the
designs interviewers ask about most.

### Details

#### 1. The unknown outcome

**Theory.** A request to an external payment provider that times out has three possible realities: it
never arrived, it failed, or it **succeeded** and the response was lost (`M08`). You cannot tell them
apart, and retrying blindly risks a double charge.

**Example.** The correct handling:
1. Record the **intent** in your database and commit — before calling the provider.
2. Call the provider with an **idempotency key** derived from that intent (`A09`).
3. On a timeout: do **not** retry blindly. **Query** the provider for the status of that key.
4. Record the outcome in a second transaction.
5. A **reconciliation job** compares your records with the provider's settlement file daily and
   raises anything that differs.

**Advanced.** Step 5 is the one that separates people who have built payments from people who have
read about them. Whatever the real-time logic, records diverge — a webhook is lost, a status query
fails, a manual refund happens in the provider's dashboard — and reconciliation is how you find out.
Mentioning it unprompted is one of the strongest signals available in a design interview.

#### 2. Reservations with expiry

**Theory.** For finite resources — seats, stock, rooms — you must hold the resource while payment
completes, without holding it forever if the user abandons.

**Example.** The pattern: on checkout, create a **reservation** with a TTL (10 minutes) that
atomically decrements available stock; on payment success, convert it to a confirmed order; on
failure or expiry, release it. Expiry is handled by a background job or a TTL index — and the release
must be **idempotent**, because it may race with a late confirmation.

**Advanced.** The race to have an answer for: payment succeeds at the moment the reservation expires
and another customer takes the seat. Options: make the expiry job check for an in-flight payment;
extend the hold when payment begins; or accept it and define the business remedy (refund and apologise,
or upgrade). There is no purely technical answer — being able to say that, and to propose the business
remedy, is the mature response.

#### 3. Atomic claims

**Theory.** The moment of claiming a finite resource must be atomic and must fail cleanly when the
resource is gone (`DB07`, `DB09`).

**Example.** For seats, the cleanest mechanism is a **unique constraint**: a `seat_reservations` table
with `UNIQUE (event_id, seat_id)`. Two concurrent claims — one succeeds, one gets a unique violation
which you catch and return as "seat taken". No locking, no race, correct at any isolation level
(`DB05`). For a stock **count**, the equivalent is the conditional update
`UPDATE items SET stock = stock - 1 WHERE id = $1 AND stock >= 1`, checking the affected row count.

**Advanced.** Postgres also offers an **exclusion constraint** for overlapping ranges, which is the
exact tool for room or resource bookings: no two reservations for the same room with overlapping time
ranges, enforced by the database (`DB05`). Proposing that instead of application-level locking is a
notably strong answer to a booking-system question.

#### 4. The flash-sale shape

**Theory.** When 500,000 people want 50,000 seats in the same second, the problem is not the database
— it is that 90% of the requests cannot succeed and must be rejected cheaply.

**Example.** The design: a **waiting room** or queue at the edge that admits users at a controlled
rate, so the core system sees manageable load; a rate limiter per user to stop scripts; the atomic
claim above; a short reservation TTL because demand is high; idempotent payment; and a clear
communication of position and expected wait. Precompute what you can (seat maps in a cache) so the
browsing load does not touch the database.

**Advanced.** The key insight is **admission control at the edge** (`M10`, `M30`): it is far cheaper
to reject or queue at the gateway than to let every request reach the database and fail there. And the
queue must be fair and visible, because an invisible queue produces refresh storms — which is why
waiting rooms show a position. This is a case where the user-experience design is part of the
scalability solution.

### Interview questions

- "Design ticket booking for a 50,000-seat venue with 500,000 people hitting refresh."
- "Your charge request timed out. What do you do next?"
- "Prove that no seat can be sold twice."
- "The reservation expires at the same moment the payment succeeds. What happens?"

---

## SD11 · Rate limiting and multi-tenant fairness at scale

`Advanced` · Requires: `A08`, `Q04`, `S13`, `M29` · Unlocks: `SD13`

### Preface

"Design a rate limiter" is a standard system-design question because it touches distributed counters,
consistency trade-offs, storage choice and API design in one small problem.

The core tension: an accurate limit needs shared state and therefore a network call on every request;
a cheap limit is approximate. Which you choose depends on whether the limit exists for **protection**
or for **billing**.

### Details

#### 1. Requirements first

**Theory.** Establish: what is being limited (requests, cost units, bandwidth), by what key (user,
API key, IP, tenant), how accurate it must be, what happens on failure, and what the client sees
(`SD01`).

**Example.** "10,000 customers on three plans; limits per API key per minute; bursts allowed;
approximate is fine as long as it never significantly under-limits; the limiter must not add more than
a millisecond; and if the limiter is unavailable we allow rather than block." Those five sentences
determine the design.

**Advanced.** Distinguish the two purposes explicitly, because they have opposite failure modes:
**protection** limits exist to keep the service up, so they should be cheap, approximate and fail
open; **quota** limits are a billing feature, so they must be accurate and may fail closed. Many
systems need both, implemented differently (`A08`).

#### 2. The algorithm and the storage

**Theory.** **Token bucket** for the algorithm (bursts plus a steady rate, `A08`), and Redis for
shared state, with the whole read-compute-write in a **Lua script** so it is atomic (`Q08`).

**Example.** The per-request flow: compute the key (`rl:{apiKey}:{window}`), run the Lua script which
refills tokens based on elapsed time and decrements if available, and act on the result. One round
trip, atomic, roughly 0.5ms. Return `429` with `Retry-After` and the `RateLimit-*` headers.

**Advanced.** At very high request rates, a Redis call per request becomes the bottleneck and a
single point of failure. The scalable design is **local buckets with periodic synchronisation**: each
instance holds a share of the budget locally, decrements without any network call, and reconciles with
Redis every second. Accuracy drops (you may overshoot by roughly the instance count) and cost drops
enormously. Choosing that trade deliberately, and saying by how much it can overshoot, is the senior
answer.

#### 3. Where to enforce

**Theory.** Enforce as early and as cheaply as possible: the earlier the rejection, the less capacity
it consumes (`M30`).

**Example.** The layers: the **CDN or edge** for crude volumetric limits per IP; the **gateway** for
per-API-key limits, before authentication where possible (`M06`); the **service** for expensive
specific endpoints; and the **database or worker pool** for per-tenant concurrency (`Q21`). Each layer
protects what is behind it.

**Advanced.** Different endpoints should have different costs: a search costing fifty times a simple
read should consume fifty tokens, not one. **Cost-based limiting** is more accurate than
request-counting, and it is what makes a limit meaningful for an API with heterogeneous endpoints —
it is also how GraphQL complexity limits work (`A15`).

#### 4. Fairness between tenants

**Theory.** Rate limiting prevents abuse; fairness prevents one legitimate heavy user from consuming
the shared capacity everyone else needs.

**Example.** The techniques: per-tenant quotas; weighted fair queueing for background work so no
tenant monopolises the workers (`Q21`); per-tenant concurrency caps on shared resources such as the
database connection pool; and shuffle sharding so a tenant's burst affects only a subset (`M31`).

**Advanced.** The subtlety is that **the limit must be enforced on the scarce resource**, not just on
requests. A tenant within their request limit can still exhaust the connection pool with slow queries,
or fill the job queue with long jobs. So limits belong at each scarce resource — requests, connections,
worker slots, storage — which is the bulkhead pattern applied per tenant (`M10`).

### Interview questions

- "Design a rate limiter for a public API with 10,000 customers on different plans."
- "Your Redis is down. Do you fail open or closed?"
- "Make it cheap enough for 100,000 requests per second."
- "A tenant is within their rate limit and still degrading everyone else. How?"

---

## SD12 · Canonical design patterns to have pre-solved

`Advanced` · Requires: `SD05`, `SD09`, `M23`, `A15`, `A16`, `A21`, `DB31`, `Q17` · Unlocks: `SD16`

### Preface

A handful of designs recur constantly, and each has a well-understood shape. Having a two-minute
skeleton for each means you spend the interview on the interesting trade-offs rather than deriving
the basics.

These are sketches, not full designs — enough to start confidently and go deep wherever the
interviewer pushes.

### Details

#### 1. Feed, chat and notifications

**Theory.** Three variations on delivering events to many users.

**Example.**
- **Feed / timeline** — hybrid fan-out (`SD09`), a precomputed per-user list in Redis or a
  denormalised table, trimmed to the most recent N, ranked at read time if ranking is needed, cursor
  pagination (`A05`).
- **Chat** — a WebSocket gateway layer holding connections; a pub/sub backplane so any instance can
  reach any connection (`F16`); messages **persisted first** and delivered second, with per-conversation
  ordering by sequence number; an unread cursor per user; and offline delivery via push notification.
- **Notifications** — a single service consuming domain events (`M17`), with templating, per-user
  channel preferences, deduplication and digesting (do not send twenty emails for twenty events),
  per-provider fallback, and rate limits per user.

**Advanced.** The shared insight across all three: **durable storage first, real-time delivery as an
optimisation**. A client that reconnects fetches what it missed by cursor. Designs that treat the
WebSocket as the delivery mechanism lose messages on every disconnection, and that is the failure
interviewers probe.

#### 2. Search and file storage

**Theory.** Both are cases of a derived store fed from a source of truth.

**Example.**
- **Search** — the database remains authoritative; CDC or an outbox feeds an indexing pipeline
  (`Q18`); documents carry a version so out-of-order updates do not regress (`DB31`); queries use
  `search_after` rather than deep offsets; mapping changes are handled by reindexing into a new index
  and switching an alias; and a reconciliation job detects drift.
- **File storage** — presigned upload directly to object storage so bytes never touch your API
  (`F17`); a metadata service holding ownership, size, type and status; a storage event triggering a
  worker for validation, virus scanning and transcoding; content-hash deduplication; CDN delivery with
  presigned or signed-cookie access control.

**Advanced.** For file storage, the detail worth raising is **orphan handling**: an upload that is
never confirmed leaves an object with no metadata row, and a metadata row whose upload failed points
at nothing. Both need a periodic reconciliation, and mentioning it shows you have thought past the
happy path.

#### 3. URL shortener, leaderboard, scheduler

**Theory.** Small problems with specific, well-known answers.

**Example.**
- **URL shortener** — a key-value store; id generation by counter with base62 encoding, or a random
  key with a uniqueness check; cache-heavy reads (`Q02`); a `301` or `302` redirect (302 if you want to
  keep counting clicks); analytics via an asynchronous event rather than a synchronous write (`SD06`).
- **Leaderboard** — a Redis sorted set (`Q05`): `ZADD` to update, `ZREVRANGE` for the top N,
  `ZREVRANK` for a user's position. For millions of entries, shard by region or time bucket and merge.
- **Scheduler at scale** — a table with a `due_at` index, polled and claimed with
  `FOR UPDATE SKIP LOCKED` (`DB09`), or a Redis sorted set scored by due time; jitter on bulk due
  times (`M09`); idempotent execution; and a separate concern for the "exactly one runner" problem
  (`F14`).

**Advanced.** For the shortener, the interesting question is id generation without coordination:
Snowflake-style ids (timestamp + machine id + sequence) give ordered, unique, distributed ids without
a central counter (`DB25`). For leaderboards, the interesting question is that an exact global rank is
expensive at scale, so most systems show an approximate rank outside the top N — worth saying rather
than promising exactness.

#### 4. Metrics pipeline and geo matching

**Theory.** Two high-volume designs with distinctive shapes.

**Example.**
- **Metrics / analytics pipeline** — stateless collectors behind a load balancer → Kafka partitioned
  by source (`Q14`) → stream processing with tumbling windows on event time and watermarks for late
  data (`Q17`) → a columnar store for queries (`DB32`), with pre-aggregated rollups for common
  time ranges and raw data retained briefly.
- **Geo matching (ride-hailing)** — drivers publish location every few seconds into an in-memory
  geospatial index (Redis geo, or a quadtree/geohash structure, `DB33`); matching queries a radius and
  ranks candidates; the trip is a state machine with events; location history goes to cold storage
  asynchronously.

**Advanced.** For the metrics pipeline, the crucial decision is **event time versus processing time**
(`Q17`) — using processing time makes backfills and replays produce wrong numbers. For geo matching,
the crucial one is that the **write rate is enormous and the data is disposable**: current location
does not need durability, so it belongs in memory, while the trip record does need durability and
belongs in a database. Separating those two is the design insight.

### Interview questions

- Any of the above, cold. Have the two-minute skeleton ready.
- "Why store the message before delivering it over the WebSocket?"
- "How does your search index stay in sync, and how do you know when it has not?"
- "Design a scheduler that fires a million reminders at 09:00."

---

## SD13 · Geography, tenancy and regulation

`Expert` · Requires: `SD07`, `SD08`, `SD11`, `M29`, `M31`, `O11`, `O20`, `S15` · Unlocks: `SD15`

### Preface

Going multi-region is usually driven by one of three things: latency for distant users, surviving a
regional failure, or a legal requirement that data stays in a jurisdiction.

Each has a different answer, and conflating them produces an expensive design that serves none of
them well. And the hardest part is never the infrastructure — it is deciding what happens to writes.

### Details

#### 1. Establish the actual driver

**Theory.** Ask which of the three you are solving, because the designs differ substantially.

**Example.**
- **Latency** — often solved without multi-region writes: a CDN for static and cacheable content
  (`Q09`), edge termination to shorten the TLS handshake, and read replicas in the distant region.
  Writes still go to one region, and users tolerate write latency far better than read latency.
- **Availability** — active-passive with a tested failover, and honest RPO and RTO numbers (`O20`).
- **Data residency** — partition by region: EU customers' data lives and is processed in the EU. This
  is not replication; it is **sharding by jurisdiction** (`DB25`).

**Advanced.** The residency case is the one that most changes the architecture, and it is often the
simplest to reason about: each region is an independent deployment serving its own tenants, with a
routing layer directing users to their home region. That is a **cell architecture** with cells defined
by jurisdiction (`M31`), and it gives you availability isolation as a bonus.

#### 2. Active-passive versus active-active

**Theory.** **Active-passive**: one region serves everything; the other is warm and takes over on
failover. Simple, and failover is a procedure with a real RTO. **Active-active**: both serve traffic,
so there is no failover — and you must resolve concurrent writes (`M33`).

**Example.** Active-active for **reads** is straightforward — replicate and serve locally.
Active-active for **writes** to the same data is where the difficulty is: two regions accepting writes
to the same row means either conflict resolution (last-write-wins, which loses data, `M21`; or CRDTs,
which are limited to particular data types, `M12`) or a consensus-based database paying cross-region
latency on every write (`DB35`).

**Advanced.** The pragmatic and most common answer is **partitioned ownership**: each record has a
home region that owns its writes, and other regions read a replica and forward writes to the owner.
You get local reads everywhere, correct writes, and no conflict resolution — at the cost of higher
write latency for users away from home. Most systems described as active-active are actually this, and
saying so precisely is a strong signal.

#### 3. What multi-region actually costs

**Theory.** It is not just infrastructure: it is replication lag, cross-region data transfer costs,
deployment complexity, testing, and the fact that every dependency must also be multi-region.

**Example.** The hidden costs: your database, cache, queue, secret manager, identity provider and
third-party APIs must all be available in both regions, or you have moved the single point of failure
rather than removed it. Deployments must roll across regions safely. Schema migrations run twice with
replication in between. And **cross-region data transfer is billed** (`O18`).

**Advanced.** The most common failure of a multi-region design is **capacity**: the surviving region
was never provisioned to handle the full load, so failover moves the outage rather than preventing it.
Static stability (`M31`) means each region can carry the whole load without scaling — which means
running at under 50% utilisation, and that cost must be part of the decision.

#### 4. Global uniqueness and coordination

**Theory.** Some things must be globally unique or globally ordered — usernames, invoice numbers,
identifiers — and global coordination is what multi-region designs try to avoid.

**Example.** The techniques: generate ids that are unique without coordination (UUIDv7, Snowflake with
a region component, `DB25`); keep a small globally-consistent service for the few things that
genuinely need it (username registration), accepting its latency because it is rare; or make
uniqueness **regional** (invoice numbers prefixed by region), which is often acceptable and removes
the problem entirely.

**Advanced.** The design principle to state: **push coordination to the smallest possible surface**.
If only username registration needs global consensus, only that path pays for it, and everything else
runs locally. Designs that need global coordination on the request path do not scale across regions,
and identifying which operations genuinely need it is the core of the work (`M20`).

### Interview questions

- "EU customer data must stay in the EU. How does that change your design?"
- "Active-active — for reads or for writes? What is the difference in difficulty?"
- "What is your real capacity plan when one region fails?"
- "How do you keep usernames globally unique across three regions?"

---

## SD14 · Migration, rollout and backfill

`Advanced` · Requires: `SD03`, `DB22`, `M27`, `M28`, `O09`, `Q19` · Unlocks: `SD15`

### Preface

You are almost never designing on a blank page. The realistic question is how to get from the system
that exists to the one you have designed, while it keeps serving traffic.

Interviewers like this because it cannot be answered from a textbook diagram. The answer is always
some form of: run both, compare, switch gradually, keep the ability to go back.

### Details

#### 1. The general shape

**Theory.** Expand, migrate, contract (`DB22`, `M24`), applied at the system level:
1. Build the new path alongside the old.
2. **Dual-write** or replicate so both have the data.
3. Compare outputs on real traffic (shadow mode).
4. Switch **reads** to the new path, gradually, behind a flag.
5. Switch **writes** to the new path.
6. Remove the old path — later, once you are certain.

**Example.** Each step is independently deployable and reversible, and the risky moment (step 5) comes
after you have proved correctness in steps 3 and 4. The whole sequence typically takes weeks, and
compressing it is how migrations fail.

**Advanced.** **Dual-writing is dangerous** and worth flagging: writing to two stores is the dual-write
problem (`M15`) — one succeeds, the other fails, and they diverge silently. Prefer one authoritative
writer plus replication (CDC, `Q18`) to the other. If you must dual-write, you need continuous
comparison and a defined resolution procedure, because divergence is not a possibility but a
certainty.

#### 2. Shadow traffic and parity testing

**Theory.** Run the new implementation on real traffic without using its results, and compare with the
old one. Differences are bugs — usually in your understanding of the old system.

**Example.** For each request, call the old path (and serve its result) and the new path (discarding
its result), recording any mismatch with the inputs. Run for a week. You will find undocumented
behaviour nobody remembers: a rounding rule, a special case for one customer, a quirk a partner
depends on (`M28`).

**Advanced.** The new path must have a **dry-run mode** with no side effects — no emails, no charges,
no writes to shared tables — or shadow traffic causes real damage. Building that mode is worthwhile
anyway for testing. And budget for mismatches that turn out to be the **old** system being wrong,
which turns a migration into a product decision about what the rule should be.

#### 3. Backfills

**Theory.** Moving historical data is a separate job from the code change: throttled, resumable,
verifiable, and running for days.

**Example.** The requirements: process in batches with a checkpoint so it can resume; throttle against
replication lag rather than wall-clock time (`DB22`); make it idempotent so a re-run is safe; log
progress and estimated completion; and **verify** — row counts, checksums, and spot comparisons —
rather than assuming success. Run it as its own deployment, never as part of a release.

**Advanced.** For a large backfill, the ordering matters: backfill **oldest first** so the most
recently-written data (which is most likely to be read) converges last and is covered by dual-writing
in the meantime; or newest first if recent data is what matters most. Decide deliberately, because the
window where old data is present and new data is not has different consequences depending on the
access pattern.

#### 4. The rollout and the exit

**Theory.** Switch gradually with a flag, with a defined success measure and an abort condition
(`M27`, `O09`).

**Example.** The sequence: internal users → 1% of traffic → 10% → 50% → 100%, with error rate, latency
and a business metric compared at each step, and automatic rollback on regression. Per-tenant
targeting is often better than a percentage, because it lets you start with tolerant customers and
keeps each customer's experience consistent.

**Advanced.** Define the **exit criteria** up front — what must be true before you delete the old
path — and then actually delete it. Migrations that stall at 90% leave two systems running forever,
which is worse than either alone: double the maintenance, double the bugs, and nobody knows which is
authoritative. The most common failure of a migration is not technical; it is not being finished.

### Interview questions

- "Move this table from Postgres to DynamoDB with zero downtime."
- "Split the user service out of the monolith while shipping features weekly."
- "How do you know the new implementation behaves identically?"
- "How do you run a backfill over 500 million rows without hurting production?"
