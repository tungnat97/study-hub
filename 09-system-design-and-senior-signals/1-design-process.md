[← back to the field index](README.md)

# System Design · Part 1 — The Design Process

Nodes `SD01`–`SD08`. Every node here consumes the other eight fields.

---

## SD01 · Requirements, scoping and the first five minutes

`Intermediate` · Requires: — · Unlocks: `SD02`, `SD03`, `SD15`

### Preface

The most common way to fail a design interview is to start drawing boxes. The interviewer's question
is deliberately vague, and the first thing being assessed is whether you narrow it.

Spend the first five minutes establishing what you are building and for whom, write the requirements
down, and get agreement. Everything after that is judged against them.

### Details

#### 1. Functional scope

**Theory.** Identify the two or three use cases that matter, and explicitly **cut** the rest. A design
that tries to cover everything covers nothing in useful depth.

**Example.** "Design Twitter" → ask: are we building posting and reading a timeline, or also search,
direct messages, notifications, advertising, moderation and analytics? Propose the core —
"I will focus on posting a tweet and reading a home timeline, and treat search and notifications as
out of scope unless you want them" — and let the interviewer redirect. That single sentence
demonstrates scoping ability before you have designed anything.

**Advanced.** Cutting scope explicitly is also how you control the clock: a 45-minute discussion
covers two use cases properly or six badly. Say what you are excluding and why, and offer to come back
to it. Interviewers frequently *want* you to cut, and candidates who try to cover everything run out
of time before reaching the interesting parts.

#### 2. Non-functional requirements — the numbers

**Theory.** These determine the architecture far more than the features do. Establish: scale (users,
requests per second, data volume, growth), latency targets, consistency requirements, availability
target, retention, and compliance constraints.

**Example.** The questions to ask, quickly:
- How many daily active users, and what is the read-to-write ratio?
- What is the latency target, and for which operations?
- How consistent must this be — is a few seconds of staleness acceptable, and where is it not?
- What availability do we need, and what happens to the business when it is down?
- How long do we keep the data? Any residency or regulatory constraints?

**Advanced.** The consistency question is the highest-value one, because the answer changes the whole
design (`SD08`). "Can a user see their own post immediately, and is it acceptable for followers to see
it a few seconds later?" — almost always yes, and it licenses an entire class of asynchronous,
scalable designs. Asking it early marks you out.

#### 3. Write it down and restate

**Theory.** Put the agreed requirements somewhere visible and refer back to them. It keeps the
discussion anchored and gives you a defence when the design is challenged.

**Example.** A visible list: `10M DAU · 100:1 read:write · p99 < 200ms reads · timeline may lag 5s ·
posts must never be lost · 99.95% · 2 years retention`. When the interviewer later asks "why not
strongly consistent?", you point at the third line. The requirement was agreed, so the trade-off is
justified rather than assumed.

**Advanced.** Restating is also how you catch a misunderstanding cheaply: "so to confirm — reads
dominate, the timeline may be a few seconds behind, and losing a post is unacceptable. Is that right?"
Thirty seconds that prevents designing the wrong system for twenty minutes.

#### 4. Constraints beyond the technical

**Theory.** Team size, existing stack, build-versus-buy, timeline and budget all shape a real design
and are legitimate to raise.

**Example.** "Is this a greenfield system or an addition to an existing one? What are we already
running — do we have Kafka, or would this introduce it? How big is the team?" A three-person team
should not be handed a design with nine services and a stream-processing cluster (`M01`).

**Advanced.** Proposing the **simplest thing that meets the stated requirements**, and naming what
would make you add complexity, is the strongest possible position. "I would start with Postgres and a
single service; the signal that would make me introduce a queue is X; the signal for sharding is Y."
That is what real senior engineers do, and it is the opposite of the buzzword-stacking that
interviewers explicitly downgrade (`SD15`).

### Interview questions

- "Design Twitter." → your first move is six to eight clarifying questions and a written requirement
  list.
- "What would you cut from this scope?"
- "How consistent does this need to be?"
- "What would make you add a message queue here?"

---

## SD02 · Estimation

`Intermediate` · Requires: `SD01`, `C14`, `C15` · Unlocks: `SD03`, `SD04`, `SD06`

### Preface

Numbers turn a design discussion from opinion into engineering. "We will need sharding" is an
assertion; "at 50,000 writes per second and 2KB per row, that is 8TB a month, so one database will
not do" is an argument.

Round aggressively, say that you are rounding, and show the working.

### Details

#### 1. From users to requests per second

**Theory.** Daily active users → actions per user per day → requests per day → average requests per
second → peak.

**Example.** The arithmetic, with the shortcut worth memorising — **a day is about 100,000 seconds**
(86,400, rounded):

```
10M DAU × 2 posts/day          = 20M writes/day   ÷ 100k = 200 writes/sec average
10M DAU × 200 reads/day        = 2B reads/day     ÷ 100k = 20,000 reads/sec average
peak = 2–10× average           → ~2,000 writes/sec and ~100,000 reads/sec at peak
```

The 100:1 read-to-write ratio is the number that then drives the entire design: caching, read
replicas, and precomputed read models (`SD05`).

**Advanced.** Choose the peak multiplier deliberately and say why: a global consumer service peaks at
perhaps 2-3x average; a regional one with a daily rhythm peaks at 5x; an event-driven one (a ticket
sale, a product launch) can peak at 100x and needs a queue or a waiting room rather than capacity
(`SD10`).

#### 2. Storage and bandwidth

**Theory.** Rows per day × bytes per row × retention × replication factor. Bandwidth is requests per
second × payload size.

**Example.**

```
20M posts/day × 1KB          = 20GB/day → 600GB/month → ~7TB/year
× 3 (replication)            = ~21TB/year
+ indexes (~30%)             → ~27TB/year
```

That number decides whether this fits on one machine (`DB25`). And bandwidth:
`100,000 reads/sec × 2KB = 200MB/s` — enough to matter for network capacity and, in a cloud, for the
transfer bill (`O18`).

**Advanced.** Distinguish **hot** from **cold** data, because it usually changes the answer entirely:
27TB total with only the last week (roughly 150GB) actively read means the working set fits in memory
and the rest can live in cheaper storage or a partition that is rarely touched (`DB24`). "How much of
this is hot?" is one of the highest-leverage estimation questions.

#### 3. Latency and capacity anchors

**Theory.** A few memorised numbers let you sanity-check any design.

**Example.** The set worth knowing:

| Operation | Order of magnitude |
|---|---|
| Memory read | 100 ns |
| SSD random read | 100 µs |
| Same-datacentre round trip | 0.5 ms |
| Postgres simple indexed query | 1 ms |
| Cross-continent round trip | 100-150 ms |
| One server, simple JSON API | 10,000-50,000 rps |
| One Postgres primary, simple writes | low thousands/sec |
| One Redis instance | ~100,000 ops/sec |
| One Kafka partition | ~10 MB/s |

**Advanced.** The useful application is **bounding**: if your design needs 100,000 writes per second
and one Postgres does a few thousand, you need roughly 30+ shards or a different store — and you know
that within seconds rather than after ten minutes of drawing. Equally, if the design needs 500
requests per second, one server does it and any distributed design needs justification (`M01`).

#### 4. Sizing from Little's Law

**Theory.** `concurrency = throughput × latency` (`C14`) converts request rates into resource counts.

**Example.** 20,000 reads per second at 50ms each means 1,000 concurrent requests in flight. At 200
concurrent requests per instance, that is five instances plus headroom — say eight for peak and zone
failure (`C15`). Same for connection pools: 2,000 writes per second at 5ms is 10 concurrent database
operations, so a pool of 20 is ample (`DB21`).

**Advanced.** This is also how you find the bottleneck before building anything: compute the required
concurrency for each component and compare it with what one instance can do. The component where the
number is uncomfortable is where the design work is needed — and naming it explicitly ("the write path
to Postgres is the constraint here") structures the rest of the discussion.

### Interview questions

- "1M DAU posting twice a day with a 100:1 read ratio. Size the system." (do this out loud)
- "How much storage after two years?"
- "How many application instances do you need?"
- "How much of that data is actually hot?"

---

## SD03 · High-level design

`Intermediate` · Requires: `SD01`, `SD02`, `M01`, `M03` · Unlocks: `SD04`, `SD05`, `SD06`, `SD07`

### Preface

Now you draw. The goal is a diagram simple enough that you can walk a request through it end to end,
with every arrow labelled and every box's responsibility stated in one sentence.

Start simple — deliberately, visibly simple — and add complexity only where a stated requirement
forces it. That sequencing is itself a signal.

### Details

#### 1. The standard skeleton

**Theory.** Most systems share a shape: clients → CDN/edge → load balancer → API gateway → services →
data stores, with an asynchronous pipeline alongside for work that does not block the response.

**Example.** Draw it and name each box's job: the CDN serves static assets and cacheable responses
(`Q09`); the gateway does TLS, authentication and rate limiting (`M06`); the service holds business
logic; Postgres is the source of truth; Redis caches reads; Kafka carries events to consumers that do
not block the user (`M03`). Label each arrow with its protocol and whether it is synchronous.

**Advanced.** Resist adding boxes that no requirement justifies. A design with nine services, a mesh,
a stream processor and a search cluster for a system with 500 requests per second is a **negative**
signal, and it is one interviewers name explicitly. Draw three boxes, say "this handles the stated
load; here is what would make me split it", and you look more senior, not less.

#### 2. Walk the primary flow

**Theory.** Narrate the main use case through the diagram, step by step, saying what happens at each
hop. This is the single most effective thing you can do in a design interview.

**Example.** "A user posts. The request hits the gateway, which verifies the token and applies a rate
limit. The service validates, writes the post to Postgres and an outbox row in the **same
transaction** (`M15`), and returns the created post so the client can display it immediately without
re-reading (`M12`). A relay publishes `PostCreated` to Kafka, keyed by author id (`Q14`). The fan-out
consumer writes it into followers' timelines. The search indexer updates Elasticsearch."

**Advanced.** Then walk the **read** path separately — it is usually the more interesting one at a
100:1 ratio, and it is where caching, precomputation and consistency decisions live. Explicitly saying
"let me walk the write path, then the read path" structures the whole discussion for both of you and
is a small, effective piece of interview technique.

#### 3. Choose the stack and justify it

**Theory.** Name the specific technologies and why, in terms of the requirements rather than
familiarity.

**Example.** "Postgres as the source of truth — the data is relational, we need transactions for
ordering, and one primary comfortably handles 2,000 writes per second. Redis for the timeline cache
because we need sub-millisecond reads of a sorted list, which is a sorted set (`Q05`). Kafka for the
fan-out because several independent consumers need the same events with replay (`Q10`)."

**Advanced.** Have a **default** and deviate deliberately: Postgres unless proven otherwise (`DB26`);
a message queue only when a consumer must not block the user; a cache only when a measured read is
expensive and repeated. Stating the default and the reason for deviating is exactly the judgement
being assessed, and it also protects you — you are never defending a choice you cannot justify.

#### 4. Identify the bottleneck before optimising

**Theory.** After the simple design, say where it breaks first. That directs the rest of the
conversation to where the engineering actually is.

**Example.** "At 100,000 reads per second, the bottleneck is the database read path — one primary
cannot serve that. So: cache the timeline in Redis, and precompute it on write rather than assembling
it on read (`SD05`, `SD09`)." Now you are solving a specific problem instead of adding generic
scalability.

**Advanced.** Naming the bottleneck **before** the interviewer does is one of the clearest senior
signals available. It shows you understand your own design's limits, and it means the follow-up
questions land on ground you have prepared rather than catching you out.

### Interview questions

- "Walk the write path, then the read path." (say this yourself)
- "Where does this design break first?"
- "Why Postgres here and not something else?"
- "What would you build first if you had two weeks?"

---

## SD04 · Data model and storage choice

`Advanced` · Requires: `SD03`, `DB01`, `DB26`, `M22` · Unlocks: `SD05`, `SD06`, `SD08`

### Preface

The data model is where a design becomes concrete, and where most of the eventual pain is decided. It
is also the part candidates most often skip.

The method: write down the **access patterns first**, then choose the store and the schema that serve
them. Not the other way round.

### Details

#### 1. Access patterns first

**Theory.** List the queries the system must answer, with their frequency and latency requirement.
The schema and the store follow from that list.

**Example.** For a timeline system:
- Get a user's home timeline, 50 items, newest first — 20,000/sec, under 100ms. **Dominant.**
- Get one user's posts — 1,000/sec.
- Get a single post by id — 5,000/sec.
- Create a post — 200/sec.
- Get a user's follower list — internal, for fan-out.

Now the design is directed: the first pattern dominates everything, and it is a per-user ordered list
— which is a Redis sorted set or a denormalised table, not a join across posts and follows.

**Advanced.** This is the discipline NoSQL forces (`DB28`) and that relational modelling lets you
postpone. Doing it even when you choose Postgres produces a better schema and the right indexes
(`DB15`), and it immediately tells you which queries need a precomputed read model (`M19`).

#### 2. The schema

**Theory.** Sketch the tables or documents with keys and the indexes that serve the listed patterns.
It need not be complete; it must show the important relationships and the access paths.

**Example.**

```
users(id PK, handle UNIQUE, name, created_at)
posts(id PK, author_id FK, body, created_at)
   INDEX (author_id, created_at DESC)          -- a user's own posts
follows(follower_id, followee_id, created_at)
   PK (follower_id, followee_id)
   INDEX (followee_id)                         -- who follows X, for fan-out
timeline(user_id, post_id, created_at)         -- precomputed read model (M19)
   PK (user_id, created_at DESC, post_id)
```

Naming the composite index and its column order (`DB15`) is a strong, concrete detail most candidates
skip.

**Advanced.** Distinguish the **source of truth** from **derived** stores, and say how the derived
ones stay in sync: `posts` is authoritative; `timeline`, the Redis cache and the search index are
derived, fed by events from the outbox (`M15`), rebuildable from the source, and allowed to lag
(`M19`). That sentence answers half the follow-up questions in advance.

#### 3. Choosing the store per pattern

**Theory.** Polyglot persistence is justified when access patterns genuinely differ — and every
additional store has an operational cost (`DB26`).

**Example.** A defensible split: Postgres for posts, users and follows (relational, transactional);
Redis for the hot timeline (sorted set, sub-millisecond); Elasticsearch for search (inverted index,
`DB31`); object storage for media; a columnar warehouse for analytics (`DB32`). Each has a specific
justification, and each is fed from Postgres rather than written directly.

**Advanced.** Be ready for "could you do it all in Postgres?" — and the honest answer is often yes at
moderate scale: `jsonb`, full-text search (`DB34`), and a timeline table with the right index. The
argument for the extra stores is scale and access-pattern fit, and if the numbers do not demand it,
saying "one store until the numbers say otherwise" is the stronger answer.

#### 4. Retention, archival and growth

**Theory.** Say what happens to data over time. A design that only describes writing data is
incomplete.

**Example.** "Posts are kept indefinitely, partitioned by month (`DB24`); the timeline read model
keeps only the most recent 1,000 entries per user, trimmed on write, because nobody scrolls further
and unbounded growth would be the real cost; analytics data goes to the warehouse via CDC (`Q18`) and
is deleted from the operational store after 90 days."

**Advanced.** Unbounded growth is the most common omission in system design answers, and it is what
actually breaks systems two years in — a table nobody trims, a Redis key with no TTL, an S3 bucket
with no lifecycle policy (`O18`). Stating the trimming or retention rule for every store you draw is a
cheap, high-signal habit.

### Interview questions

- "Why Postgres here and DynamoDB there?"
- "Give me the schema and the indexes for the dominant query."
- "What is the source of truth and what is derived?"
- "What happens to this data in two years?"

---

## SD05 · Scaling reads

`Advanced` · Requires: `SD04`, `DB23`, `Q02`, `Q04`, `Q09`, `A04`, `M19` · Unlocks: `SD09`, `SD12`

### Preface

Most systems are read-dominated, so this is usually where the design work is. The good news is that
reads scale in a well-understood ladder, each step cheaper and less consistent than the last.

Go up the ladder in order, and say what each step costs in staleness. Jumping straight to "add a
cache" without checking the query is the classic mistake.

### Details

#### 1. The ladder, in order

**Theory.**
1. **Fix the query** — indexes, N+1, pagination (`DB20`). Frequently a 10x win for a day's work.
2. **Connection pooling** — often the actual constraint (`DB21`).
3. **Read replicas** — linear read scaling, at the cost of replication lag (`DB23`).
4. **Application cache** — Redis, cache-aside with stampede protection (`Q02`, `Q04`).
5. **HTTP and CDN caching** — requests that never reach you at all (`A04`, `Q09`).
6. **Precomputed read models** — compute at write time, so reads are a single key lookup (`M19`).
7. **Denormalise into the store the read needs** — the timeline table, the search index.

**Example.** Applied to a slow product page: is the query indexed? Is it an N+1? Then a Redis cache
with a 60-second TTL. Then `Cache-Control` so the CDN serves most of it. Then, if it is still the
bottleneck, precompute the page's data on write. Each step is cheaper than the one after it.

**Advanced.** Steps 1 and 2 are where the largest and cheapest wins usually are, and they are the ones
candidates skip in interviews because they are less impressive. Saying "first I would check the query
plan" before proposing a caching layer is a genuine senior signal, and it is also what you would
actually do.

#### 2. What each step costs in consistency

**Theory.** Every step up the ladder adds staleness, and you must say how much and what the user sees.

**Example.** Replicas: milliseconds to seconds of lag, and it breaks read-your-writes (`M12`). Cache:
staleness up to the TTL, plus the invalidation race (`Q02`). CDN: staleness up to `max-age`, and purge
is not instant (`Q09`). Precomputed model: projection lag, potentially seconds (`M19`). For each, the
mitigation is the same family: read-your-own-writes from the primary, return the written entity, or
carry a version token.

**Advanced.** The answer users actually notice is **their own** data being stale. So the practical
rule is: serve other people's data from the cheapest layer, and serve **your own** data from the
freshest one. A timeline may lag five seconds for other people's posts and must show your own
immediately — which is achievable by merging your own recent posts client-side or at the API, and is a
very satisfying design detail to offer.

#### 3. Precomputation

**Theory.** Move the work from read time to write time. With a 100:1 read ratio, doing work once per
write instead of once per read is a hundredfold reduction.

**Example.** The timeline is the canonical case: instead of "select posts from everyone I follow,
ordered by time" on every read (an expensive join over a large table), write each new post into each
follower's timeline list at publish time. Reads become a single range query on one key. The cost is
write amplification — one post becomes N writes (`SD09`).

**Advanced.** Precomputation trades **write cost and storage** for **read latency**, and it only pays
when the read ratio justifies it. It also introduces the rebuild problem: when the computation changes,
you must recompute everything, so the job that rebuilds a read model from source must exist from day
one and be routinely exercised (`M19`).

#### 4. The cache-failure question

**Theory.** Whatever you add, you must be able to answer what happens when it is gone.

**Example.** "If Redis fails, cache reads fail open and we fall through to Postgres. At a 95% hit rate
that is twenty times the database load, which it cannot take — so we also shed load at the gateway and
serve a degraded response for non-essential parts (`M10`, `Q04`). Recovery means a cold cache, so we
warm the hottest keys before marking pods ready (`O06`)."

**Advanced.** That answer — fail open, size the miss path, degrade deliberately, plan the warm-up — is
a complete one, and it is asked in almost every design interview involving a cache. Rehearse it.

### Interview questions

- "Reads are 100x writes and the database is at 90% CPU. Give me your ordered plan."
- "What does each caching layer cost you in staleness?"
- "The cache dies. What happens?"
- "How does a user see their own post immediately if the timeline lags?"

---

## SD06 · Scaling writes

`Advanced` · Requires: `SD02`, `SD04`, `DB25`, `Q14`, `Q20`, `DB32` · Unlocks: `SD09`, `SD10`

### Preface

Writes are harder than reads, because you cannot cache them and you cannot serve them from a replica.

The techniques: accept the write quickly and process it asynchronously; batch; partition so writes
spread across machines; and avoid hot rows. Sharding is the last resort, not the first (`DB25`).

### Details

#### 1. Accept fast, process later

**Theory.** The user needs to know their write was **durably accepted**, not that every downstream
effect has completed. Do the minimum synchronously and the rest asynchronously (`M03`).

**Example.** A post: write the row and an outbox event in one transaction, return 201 — that is the
whole synchronous path, a few milliseconds. Fan-out to followers, search indexing, notifications and
analytics all happen asynchronously (`M15`). The write path is now bounded by one database insert.

**Advanced.** The limit of this technique is what the user expects to be true immediately afterwards:
if they refresh and their post is missing from their own profile, the asynchronous boundary was drawn
in the wrong place. The rule is that anything the user will immediately observe must be synchronous
(`SD05`), and everything else can be deferred.

#### 2. Batching and buffering

**Theory.** Many small writes are far more expensive than one batched write, because of per-operation
overhead: round trips, transaction commits, index maintenance and WAL flushes (`DB12`).

**Example.** For high-volume ingest (telemetry, analytics, view counts), buffer in memory for a short
window or a batch size, then write once. A thousand individual inserts become one `COPY` — often a
hundredfold improvement (`DB20`). The trade is a small window of possible loss if the process dies,
which for counters and telemetry is acceptable and for orders is not.

**Advanced.** For counters specifically, the better pattern is to **not write them synchronously at
all**: emit an event, aggregate in a stream processor or a periodic job, and store the result
(`Q17`). That removes the hot row entirely rather than making writing to it faster — which is the
difference between an optimisation and a design change.

#### 3. Partitioning the write path

**Theory.** To exceed one machine's write capacity you must spread writes across machines, which
means choosing a partition key that distributes evenly and keeps related data together (`DB25`,
`M32`).

**Example.** For telemetry at a million writes per second: a load balancer in front of stateless
collectors → Kafka partitioned by device id (`Q14`) → a stream processor aggregating per device and
time window → a columnar store for queries (`DB32`). Each layer scales horizontally, and the partition
key is consistent throughout so ordering per device holds.

**Advanced.** The key must be chosen for **both** even distribution and query locality, and those can
conflict (`DB25`). And it is effectively irreversible — changing it means moving all the data and
re-establishing ordering. Saying "this is the decision I would spend the most time on, because it is
the hardest to change" is exactly right.

#### 4. Hot rows and write conflicts

**Theory.** A single row updated by many writers serialises regardless of how many machines you have
(`DB09`).

**Example.** A viral post's like counter: every like updates the same row, so throughput is bounded by
lock hold time. The fixes, in order: **sharded counters** — N rows per post, each write picks one at
random, reads sum them (`M32`); **stream aggregation** — emit events and update the counter
periodically; or **approximate counting** for display, with an exact count computed on demand.

**Advanced.** This generalises: whenever a design has a single row, key or partition that every
request touches, that is the scaling limit, and the answer is always to **split it and aggregate**.
Recognising the shape — "there is a single hot object here" — is more valuable than any specific
technique, because it applies to counters, inventory, leaderboards and rate limiters alike (`SD09`).

### Interview questions

- "1M writes/sec of telemetry. Design the ingest path."
- "A single 'likes' counter on a viral post. Fix the hot row."
- "What must be synchronous in a write path and what can be deferred?"
- "How do you choose the partition key, and why is it hard to change?"

---

## SD07 · Availability and failure design

`Advanced` · Requires: `SD03`, `M10`, `M31`, `M33`, `M35`, `O20` · Unlocks: `SD13`, `SD15`

### Preface

A design is not finished until you have said what happens when each part of it fails. Interviewers ask
this deliberately, because it separates people who have operated systems from people who have drawn
them.

The structure: for each component, what breaks, what the user sees, and what limits the damage.

### Details

#### 1. Walk the failures

**Theory.** Go component by component and state the failure mode, the user impact and the mitigation.

**Example.** For the timeline design:
| Fails | User impact | Mitigation |
|---|---|---|
| A service instance | none | several replicas, health checks, load balancer removes it (`O06`) |
| Redis cache | reads hit Postgres; 20x load | fail open, shed load, degrade, warm on recovery (`Q04`) |
| Postgres primary | writes fail | automatic failover to a standby; reads continue from replicas (`DB23`) |
| Kafka | fan-out stops; posts still accepted | the outbox retains events; consumers catch up (`M15`) |
| A whole zone | reduced capacity | multi-AZ with capacity provisioned to survive it (`M31`) |
| The payment provider | checkout fails | queue the intent, retry, notify; or fail clearly (`M10`) |

**Advanced.** Note that a well-designed system degrades in **layers**: the cache failing is invisible
to users; Kafka failing delays a non-critical feature; only the primary database failing affects
writes. Being able to show that ordering — which failures are invisible, which are degraded, which are
user-visible — is what a resilient design looks like when described out loud.

#### 2. Availability maths

**Theory.** Synchronous dependencies multiply (`M03`). Redundant components at the same layer improve
availability; chained dependencies reduce it.

**Example.** Four synchronous services at 99.9% each gives 99.6% — about 35 hours a year. To do better
you must either reduce the number of synchronous dependencies (make some asynchronous), add
redundancy, or add fallbacks so a dependency failing does not fail the request. Saying this with the
arithmetic is far more convincing than asserting that the design is "highly available".

**Advanced.** Your SLO cannot exceed the combined availability of everything on the critical path
(`M34`), including managed services and third parties. If the payment provider offers 99.9%, checkout
cannot promise 99.99% unless it can complete without them — which is exactly the argument for
accepting the order and settling payment asynchronously (`SD10`).

#### 3. Containment

**Theory.** Beyond preventing failure, limit how far it spreads: timeouts, circuit breakers,
bulkheads, and partitioning by customer (`M10`, `M31`).

**Example.** The concrete controls in this design: a timeout on every outbound call, shorter than the
caller's deadline (`M09`); a circuit breaker per dependency so a failing one fails fast; a bulkhead so
the recommendation service cannot exhaust the pool used by checkout; bounded queues with load shedding
(`M30`); and per-tenant rate limits so one customer cannot consume the whole fleet (`Q21`).

**Advanced.** At scale, **cells** (`M31`) are the strongest containment: independent stacks each
serving a subset of customers, so a bad deploy, a poisoned cache or a corrupted dataset affects one
cell. It also makes deployment safer, since you roll cell by cell. Mention it as the next step when
blast radius becomes the dominant concern, rather than as something to build on day one.

#### 4. Recovery

**Theory.** Say how the system gets back to normal, not just how it survives — recovery is frequently
harder than the failure.

**Example.** The recovery problems in this design: a cold cache after a Redis restart (warm before
serving, `Q04`); a consumer backlog after a Kafka outage (it drains at the consumption rate, so how
long, and does the ordering still hold?); a retry storm when everything reconnects at once (jitter,
`M09`); and a replica that must catch up before it can take reads (`DB23`).

**Advanced.** The **thundering herd on recovery** is the pattern to name: everything restarts
simultaneously, all reconnect, all retry their backlog, and the system falls over again. Recovery
therefore often requires **admitting load gradually** — rate-limit at the edge, let the system
stabilise, then ramp up. Knowing that recovery needs its own plan is a distinctly operational insight
(`M08`).

### Interview questions

- "Walk through what happens to this design when the cache dies / a region dies / the payment
  provider is down for two hours."
- "What is your availability, given these dependencies?"
- "Which failures are invisible to users and which are not?"
- "How does the system recover after the outage ends?"

---

## SD08 · Consistency decisions

`Advanced` · Requires: `SD04`, `M12`, `DB07` · Unlocks: `SD10`, `SD13`

### Preface

Consistency is decided **per feature**, not per system. The skill is identifying the small number of
places that genuinely need strong guarantees, and being comfortable with eventual consistency
everywhere else.

Getting this right is what lets most of a system be fast, available and simple.

### Details

#### 1. Classify each operation

**Theory.** For each operation, ask: if this were a few seconds stale, what would go wrong? The
answers fall into three groups — no consequence, a confusing user experience, or an incorrect
business outcome.

**Example.** For an e-commerce system:
| Operation | Requirement | Why |
|---|---|---|
| Browse products | eventual | stale by a minute is fine |
| Search results | eventual | an index is inherently behind |
| Add to cart | read-your-writes | the user must see their own action |
| Stock decrement at checkout | **strong** | overselling is a business failure |
| Payment | **strong + idempotent** | money (`SD10`) |
| Order history | read-your-writes | the user expects their new order |
| Recommendations | eventual | nobody notices |
| Analytics | eventual, minutes | nobody is waiting |

**Advanced.** Notice how few rows need strong consistency — typically the ones involving money or a
finite resource. That is the normal distribution, and it is what licenses a design that is mostly
asynchronous with a small strongly-consistent core. Presenting the table is a compact way to
demonstrate the whole skill.

#### 2. Read-your-writes is the one users notice

**Theory.** Users tolerate other people's data being slightly stale and do not tolerate their own
changes vanishing (`M12`).

**Example.** The three fixes, best first: **return the written entity** from the write endpoint so the
UI updates without re-reading (free and exact); **route that user's reads to the primary** for a few
seconds after a write; or **merge locally** — show the user's own recent items from a separate,
fresh source combined with the cached list.

**Advanced.** The merge approach is what social platforms actually do: your own posts come from a
fresh query and everyone else's from the cached timeline, combined at read time. It gives the
appearance of immediate consistency over an eventually-consistent system, and describing it shows you
think about the user's perception rather than only about the data.

#### 3. Where strong consistency is required

**Theory.** For the few operations that need it, the mechanisms are: a single-node transaction
(easiest — keep the data together), a conditional write or version check (`DB10`), or explicit
locking (`DB09`).

**Example.** Stock decrement: keep stock in one row in Postgres and use a single atomic statement —
`UPDATE items SET stock = stock - 1 WHERE id = $1 AND stock >= 1` — checking the affected row count.
No distributed transaction, no lock held across a network call, correct under any concurrency. The
design consequence is that **stock must live in one place**, which is a constraint worth stating
explicitly.

**Advanced.** The generally correct architecture is a **strong core with an eventual edge**: a small,
strongly-consistent store for the invariants (stock, balances, uniqueness), with caches, search
indexes, read models and analytics derived from it asynchronously. Naming that shape is a strong,
reusable answer to almost any consistency question.

#### 4. Handling the gap

**Theory.** Where you accept eventual consistency, decide what happens during the window and how you
detect divergence.

**Example.** The reservation pattern for inventory: reserve stock with a TTL at checkout, confirm on
payment, release automatically on expiry (`SD10`). During the window the item shows as unavailable to
others — briefly, and correctly. And a reconciliation job compares derived stores with the source and
reports drift (`DB31`), because they **will** diverge and you want an alert rather than a customer
report.

**Advanced.** Every derived store needs three things: a **rebuild** path, **lag monitoring**, and a
**reconciliation** check. Teams that build the pipeline without those discover the drift months later
with no way to fix it except a manual migration. Stating all three when you draw a derived store is a
compact demonstration of operational maturity (`M19`).

### Interview questions

- "Which parts of your design are eventually consistent and how would a user notice?"
- "Where do you need strong consistency, and what does that constrain?"
- "How does a user see their own change immediately in an eventually-consistent system?"
- "How do you know your search index has drifted from the database?"
