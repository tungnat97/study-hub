[← back to the field index](README.md)

# Caching, Queues & Streaming · Part 1 — Caching and Redis

Nodes `Q01`–`Q09`.

---

## Q01 · Why and where to cache

`Beginner` · Requires: — · Unlocks: `Q02`, `Q09`

### Preface

A cache stores the result of expensive work so the next request gets it cheaply. You trade
**freshness** for latency, throughput and cost.

That trade is the whole subject. Every question about caching reduces to: how stale is acceptable
here, and what happens when the cache is wrong, empty or gone?

### Details

#### 1. The layers

**Theory.** Caching happens at many levels, and each is cheaper than the one behind it: the client
(browser memory, local storage), the CDN or edge, the API gateway, the application process
(in-memory), a shared cache (Redis), the database's own buffer pool, and materialised views.

**Example.** A single product page might be served from: the browser's HTTP cache for 60 seconds; the
CDN for 5 minutes; a Redis entry for an hour; and, on a miss, a database query whose pages are already
in the buffer pool. Each layer absorbs the traffic the one in front did not.

**Advanced.** The most valuable cache is always the **outermost** one that can correctly serve the
request, because it eliminates everything behind it. A CDN hit costs you nothing at all; a Redis hit
still costs a network round trip and an application process. So before adding an application cache,
ask whether an HTTP cache header would do the job (`A04`).

#### 2. Hit rate economics

**Theory.** The benefit is proportional to the hit rate, and the **miss** path still has to be sized
for the traffic that reaches it. A 90% hit rate means the backend still sees 10% — and 100% if the
cache fails.

**Example.** The arithmetic that matters: 10,000 requests per second with a 95% hit rate means 500
reach the database. That is fine. If Redis becomes unavailable, it is 10,000 — twenty times the load,
instantly. Whether that is survivable is the question you must have asked in advance (`Q04`).

**Advanced.** Improving a hit rate from 95% to 99% removes 80% of the remaining backend load, which is
often more valuable than it appears. Conversely, a cache below about 50% hit rate may be costing more
than it saves — you pay the lookup on every request and get the benefit half the time. Measure hit
rate per cache, per key pattern, and treat a low one as a design problem rather than a fact of life.

#### 3. What to cache

**Theory.** Cache things that are expensive to produce, read far more often than written, and
tolerant of being slightly stale. Do not cache things that are cheap, rarely reused, or must be exact.

**Example.** Good candidates: reference data (countries, categories, feature flags), rendered
fragments, expensive aggregates, permission lookups, third-party API responses (which are also
rate-limited, so caching protects your quota). Poor candidates: a user's account balance, anything a
single user reads once, and anything whose staleness has legal or financial consequences.

**Advanced.** The subtle case is caching per-user data. It is usually a low hit rate (each user reads
their own data a few times) and a high risk (a key-scoping bug leaks between users, `Q03`). Cache the
**shared, expensive** parts and assemble the per-user view from them — for example cache the product
catalogue and compute the personalised ordering per request.

#### 4. What goes wrong

**Theory.** Three failure families: **staleness** (users see old data), **inconsistency** (different
users or pods see different data), and **cache failure** (the cache is gone and the backend cannot
cope).

**Example.** Each has a design response: staleness → TTLs and explicit invalidation (`Q03`);
inconsistency → a shared cache rather than per-pod memory (`Q09`); cache failure → confirm the
backend can survive the miss rate, add a circuit breaker, and degrade rather than cascade (`Q04`,
`M10`).

**Advanced.** The question to be able to answer immediately — because it is asked constantly — is
"what happens when Redis goes down?". The good answer has three parts: cache reads fail open (treat
an error as a miss, never propagate it); the backend has been load-tested at 0% hit rate, or you shed
load if it cannot cope; and you have measured how long a cold start takes to warm up, because that is
your recovery time.

### Interview questions

- "Where would you add a cache in this architecture and what breaks when it is stale?"
- "Your cache hit rate is 99% and the database still falls over on deploys. Why?"
- "What would you not cache?"
- "Redis goes down entirely. What happens to your service?"

---

## Q02 · Caching strategies

`Intermediate` · Requires: `Q01` · Unlocks: `Q03`, `Q04`, `F15`, `SD05`

### Preface

There are a handful of named patterns for how reads and writes interact with a cache. The one used
almost everywhere is **cache-aside**: the application checks the cache, and on a miss loads from the
database and populates it.

Knowing the others matters mainly so you can explain why cache-aside is usually right, and so you can
discuss the consistency race it contains.

### Details

#### 1. The patterns

**Theory.**
- **Cache-aside (lazy loading)** — the application manages the cache: check, on miss load and
  populate. Only requested data is cached; the cache can fail without breaking correctness; the first
  request after a miss is slow.
- **Read-through** — the cache library loads on a miss, so the application only talks to the cache.
  Cleaner code, and the cache becomes a hard dependency.
- **Write-through** — write to cache and database together. The cache is always fresh; writes are
  slower; and you cache data that may never be read.
- **Write-behind (write-back)** — write to the cache and flush to the database later. Very fast
  writes, and you can lose data if the cache dies.
- **Refresh-ahead** — proactively refresh entries before they expire, so users never hit a miss.

**Example.** Cache-aside for a product catalogue: on a miss, query the database, put the result in
Redis with a TTL, return it. It is resilient — if Redis errors, you fall through to the database and
serve correctly, just slower.

**Advanced.** Write-behind is the one to be cautious about: it means the cache holds data the database
does not, so a cache failure loses writes. It is appropriate for metrics, counters and analytics
where some loss is acceptable, and never for business records. Naming that boundary is the useful
part.

#### 2. The cache-aside race

**Theory.** Cache-aside has a well-known interleaving that leaves a stale value cached indefinitely:

```
Reader:  cache miss → reads DB (gets value v1)
Writer:  writes v2 to DB → deletes cache key
Reader:  writes v1 into the cache        ← stale value now cached with a full TTL
```

**Example.** The reader's database read happened before the write, but its cache population happened
after the invalidation. The cache now holds v1 until the TTL expires. This is a genuine production
bug and an excellent interview question, because it tests whether you can reason about interleavings
rather than recite patterns.

**Advanced.** The mitigations, in increasing strength:
- **Short TTL** — bounds the damage. Simple, usually sufficient, and does not fix the race.
- **Delete after write, not update** — deleting is safer than writing the new value, because two
  concurrent writers could otherwise cache the wrong one.
- **Delayed double delete** — delete, write, then delete again after a short delay, catching readers
  that populated in between. Widely used and inelegant.
- **Versioned keys** — the key includes a version that changes on write, so a stale write lands on a
  key nobody will read (`Q03`).
- **Write-through or CDC-driven population** — the cache is updated from the database's change
  stream, so ordering is determined by the database (`Q18`).

#### 3. Choosing per use case

**Theory.** Match the strategy to the read/write ratio and the consistency requirement.

**Example.** A product catalogue (read-heavy, rarely written, staleness fine) → cache-aside with a
long TTL. A user session (read on every request, written occasionally, must be current) →
write-through or the cache **as** the store (`Q05`). A view counter (write-heavy, exact value
unimportant) → write-behind, flushed periodically. A price (read-heavy, staleness has financial
consequences) → short TTL plus explicit invalidation on change, or do not cache.

**Advanced.** Note that "cache" and "store" blur: a session in Redis with no database behind it is not
a cache at all — a miss is not recoverable, it is data loss. Be explicit about which you are building,
because it changes the durability requirements entirely (`DB30`).

#### 4. Negative caching

**Theory.** Caching the *absence* of a result prevents repeated expensive lookups for things that do
not exist.

**Example.** Without it, a request for a non-existent product hits the database every time — and an
attacker requesting random ids can drive unlimited database load (**cache penetration**, `Q04`). With
it, the miss is cached for a short period and subsequent lookups are free.

**Advanced.** Use a **shorter TTL** for negative entries than positive ones: an item that does not
exist may be created at any moment, and you do not want a 10-minute delay before it appears. A few
seconds to a minute is typical. For very large key spaces, a Bloom filter in front is the
memory-efficient alternative (`DB33`).

### Interview questions

- "Cache-aside versus write-through — which and why for a product catalogue? For a session?"
- "In cache-aside, what is the race between a concurrent read and write, and how do you fix it?"
- "Why delete the key on write rather than update it?"
- "What is negative caching and what attack does it prevent?"

---

## Q03 · Invalidation and key design

`Advanced` · Requires: `Q02` · Unlocks: `Q04`, `SD05`

### Preface

Getting data into a cache is easy. Getting it out at the right moment is the hard part, and the
mistakes are expensive in two directions: leaving stale data, or leaking one user's data to another
through a badly-designed key.

Two rules cover most of it. **Always set a TTL**, even when you invalidate explicitly, because it
bounds the damage from any bug. And **the key must contain everything the value depends on**.

### Details

#### 1. Key design

**Theory.** The key must include every input that affects the value: the entity id, the tenant, the
user or role if the value is personalised, the locale, and a version of the serialisation format.

**Example.** Progressively safer keys:

```
product:123                                  — wrong in a multi-tenant system
tenant:acme:product:123                      — tenant-scoped
tenant:acme:product:123:locale:en-GB         — locale-scoped
tenant:acme:product:123:locale:en-GB:v3      — plus a schema version
```

The `v3` suffix means a deploy that changes the cached object's shape does not have to invalidate
anything — old entries are simply never read and expire on their own.

**Advanced.** The schema version is the detail people miss, and it prevents a genuinely nasty class of
incident: a deploy changes a DTO, old cached objects are deserialised into the new shape, and fields
are silently `undefined`. Bumping a version in the key prefix makes deploys safe by construction.
Use a consistent naming convention with colons, and document it — `SCAN`-based debugging depends on
predictable prefixes.

#### 2. Explicit invalidation

**Theory.** When data changes, remove the cached copies. The difficulty is knowing which copies
exist — one entity may appear in a dozen cached views.

**Example.** A product price changes. Affected: `product:123`, the category listing containing it, the
search results, the homepage promotion block, and any user's cart total. Deleting them all requires
knowing they exist, and a missed one serves the old price indefinitely.

**Advanced.** Two techniques scale better than enumeration. **Versioned namespaces**: store
`product:123:version = 7`, include it in every derived key, and bump it on change — every derived
entry becomes unreachable at once, with no deletion required, and the orphans expire naturally.
**Tag-based invalidation**: maintain a set of keys per tag (Redis set `tag:product:123`), and delete
all members on change — exact, at the cost of maintaining the sets. Versioned namespaces are simpler
and usually the better answer.

#### 3. TTLs

**Theory.** A TTL is the safety net: whatever bugs exist in your invalidation logic, the cache is
wrong for at most the TTL. Choose it by how stale the data may acceptably be.

**Example.** Reasonable defaults: reference data hours; product data minutes; personalised data
seconds; anything financial, seconds or not cached. And **add jitter**: identical TTLs set during a
cache warm-up all expire at the same moment, producing a synchronised stampede (`Q04`). A TTL of
`300 + random(0, 60)` seconds spreads the expiries out.

**Advanced.** An entry that must **never** be stale should not have a long TTL "because we invalidate
it anyway" — invalidation fails (a message is lost, a code path is missed, a second writer appears)
and the TTL is what limits the blast radius. Treat the TTL as the correctness bound and invalidation
as the optimisation.

#### 4. Never scan the keyspace

**Theory.** Redis is single-threaded (`Q05`), so `KEYS *` blocks the entire server while it walks
every key. On a large instance that is seconds of total unavailability.

**Example.** Use `SCAN` with a cursor, which returns keys incrementally without blocking — and accept
that it gives a best-effort snapshot rather than an exact one. Better still, do not design anything
that needs to enumerate keys: if you need "all keys for tenant X", maintain a set of them explicitly,
or use versioned namespaces so you never need to find them.

**Advanced.** Redis Cluster makes this worse: `SCAN` must be run against every node, and multi-key
operations cannot cross hash slots (`Q07`). Designing keys so related data shares a slot, using hash
tags, is something to decide early — retrofitting it means changing every key in production.

### Interview questions

- "One user's data appears in another user's response. What was wrong with the cache key?"
- "How do you invalidate everything derived from a product when its price changes?"
- "Why set a TTL if you invalidate explicitly?"
- "Why is `KEYS *` forbidden in production?"

---

## Q04 · Stampede, hot keys and cache failure

`Advanced` · Requires: `Q02`, `Q03` · Unlocks: `SD11`, `SD05`

### Preface

Caches fail in characteristic ways, and each has a name worth knowing: a **stampede** when a popular
entry expires and everyone recomputes it at once; a **hot key** that overwhelms one node;
**penetration** when lookups for non-existent data pass straight through; and an **avalanche** when
the cache is gone entirely.

These are among the most commonly asked caching questions, because they test whether you have
operated a cache rather than just used one.

### Details

#### 1. Cache stampede

**Theory.** A popular key expires. The next thousand concurrent requests all miss, all query the
database, and all write the same value back. The database receives a thousand identical expensive
queries at the same instant.

**Example.** The defences, roughly in order of preference:
- **Request coalescing / singleflight** (`C17`) — within a process, the first miss starts the load and
  the rest await the same promise. Free, and only protects per instance.
- **Distributed lock** — one instance wins the right to recompute; others wait briefly or serve stale.
  Effective, and adds a lock dependency on the hot path.
- **Probabilistic early expiration (XFetch)** — as an entry approaches expiry, each reader has a small
  and increasing chance of refreshing it early, so exactly one usually refreshes before it expires and
  nobody ever sees a miss. Elegant and stateless.
- **Never expire + background refresh** — a scheduled job keeps hot entries fresh; readers never miss.
  Best for a known small set of very hot keys.
- **Stale-while-revalidate** — serve the expired value immediately and refresh in the background. The
  user never waits, at the cost of one stale response.

**Advanced.** The probabilistic approach is worth being able to describe because it needs no
coordination: refresh when `now - delta * beta * ln(random()) >= expiry`, where `delta` is how long
the recomputation takes. Popular keys are read often, so one of the many readers refreshes early;
unpopular keys are not refreshed unnecessarily. It solves the problem without a lock.

#### 2. Hot keys

**Theory.** One key receives a disproportionate share of traffic. In a clustered cache it lives on one
node, so that node saturates while the others idle — and you cannot shard a single key.

**Example.** A celebrity's profile, a viral product, a global configuration object. Mitigations:
**local caching** in each application instance in front of the shared cache (a two-level cache, `Q09`)
— the most effective, because the hot key is then served from process memory; **key replication**,
storing `key:1` through `key:N` and reading a random one so load spreads across nodes; and
**client-side request coalescing** so each instance only fetches it once per interval.

**Advanced.** Detecting a hot key is the hard part, because you cannot afford per-key metrics at
scale. Redis has `--hotkeys` (which uses the LFU counters) and `MONITOR` (which is far too expensive
for production). The scalable approach is a count-min sketch in the client to track approximate top-N
keys (`DB33`). Having this instrumentation before an incident is the difference between diagnosing in
minutes and guessing for hours (`M32`).

#### 3. Penetration and avalanche

**Theory.** **Penetration**: requests for keys that do not exist bypass the cache entirely and hit the
database every time. **Avalanche**: a large number of keys expire simultaneously, or the cache itself
fails, and the full load lands on the database at once.

**Example.** Penetration is also an attack — an adversary requests random ids and every one becomes a
database query. Defences: negative caching (`Q02`) and a Bloom filter in front for very large key
spaces (`DB33`). Avalanche defences: TTL jitter (`Q03`), staggered warm-up, and a circuit breaker to
the database so a total cache loss degrades rather than cascades (`M10`).

**Advanced.** The **cold start** case is the one to have thought about: after a Redis restart or a
failover, the cache is empty and every request is a miss. If your database cannot serve 100% of
traffic, you must warm the cache before accepting traffic, or admit load gradually. That is a
readiness-probe decision (`O06`) — do not mark the pod ready until the cache is warm — and it is an
impressive thing to raise unprompted.

#### 4. Cache failures must not cascade

**Theory.** A cache is an optimisation, and an optimisation must never be a single point of failure. A
cache error should be treated as a miss, not as an error.

**Example.** The rules: wrap cache calls in a short timeout (a cache that takes 500ms is worse than no
cache); catch errors and fall through to the source; add a circuit breaker so you stop calling a dead
cache entirely rather than waiting for each timeout; and emit a metric so you know it is happening.
The failure mode to avoid is a service returning 500 because Redis is unavailable.

**Advanced.** The exception is when the cache is not a cache: rate-limit counters, distributed locks
and sessions in Redis are **state**, and failing open there may be wrong (allowing unlimited requests,
or logging everybody out). That is the strongest argument for separating cache instances from state
instances — different failure policies, different eviction policies (`Q06`), different capacity
planning.

### Interview questions

- "A celebrity's profile is 40% of your traffic and lives on one Redis shard. Fix it."
- "Redis goes down entirely. What happens to your service?"
- "Explain cache stampede and three defences."
- "Your Redis restarted and the cache is cold. Now what?"

---

## Q05 · Redis fundamentals

`Intermediate` · Requires: `DB30`, `Q01` · Unlocks: `Q06`, `Q07`, `Q08`, `A08`

### Preface

Redis is an in-memory data structure server. Not a key-value store with strings — a server offering
lists, sets, sorted sets, hashes, streams and more, each with operations that run atomically because
**commands execute on a single thread**.

Choosing the right structure is most of using Redis well, and the single-threaded model explains both
its speed and its most dangerous failure mode.

### Details

#### 1. The single-threaded execution model

**Theory.** Redis executes one command at a time. There are no locks, no contention, and every
individual command is atomic. Redis 6 added threaded I/O for reading and writing sockets, but command
execution remains single-threaded.

**Example.** This is why `INCR` is safe without any coordination, why `SET key val NX` is a reliable
lock primitive, and why Lua scripts are atomic (`Q08`). It is also why throughput is around 100,000
operations per second on ordinary hardware — no locking overhead, everything in memory.

**Advanced.** The corollary is the danger: **one slow command blocks everything**. The commands to
avoid or bound: `KEYS`, `FLUSHALL` on a large instance, `SMEMBERS` or `LRANGE 0 -1` on a huge
collection, `DEL` of a multi-gigabyte value (use `UNLINK`, which frees memory in a background thread),
and long Lua scripts. A "Redis is slow" incident is very often one client running one of these.

#### 2. The data structures

**Theory.** Pick the structure that makes your operation a single command.

**Example.**

| Need | Structure | Commands |
|---|---|---|
| Cache entry, counter | String | `GET`, `SET`, `INCR`, `SET NX PX` |
| Object with fields | Hash | `HGET`, `HSET`, `HINCRBY` |
| Queue, recent items | List | `LPUSH`, `BRPOP`, `LTRIM` |
| Unique membership | Set | `SADD`, `SISMEMBER`, `SINTER` |
| Leaderboard, delayed jobs | Sorted set | `ZADD`, `ZRANGE`, `ZPOPMIN` |
| Approximate unique count | HyperLogLog | `PFADD`, `PFCOUNT` |
| Durable stream with groups | Stream | `XADD`, `XREADGROUP`, `XACK` |
| Feature flags per user id | Bitmap | `SETBIT`, `BITCOUNT` |

**Advanced.** The sorted set is the most versatile and the most under-used: score by timestamp for a
delayed job queue (`ZRANGEBYSCORE` for due items), score by points for a leaderboard, score by
insertion time for a capped recent-items list with `ZREMRANGEBYRANK`. It is a skip list internally
(`DB33`), giving O(log n) insertion and O(log n + m) range queries.

#### 3. Expiry

**Theory.** Keys can carry a TTL. Redis expires them **lazily** (when accessed) and **actively** (a
background job samples random keys with TTLs and removes expired ones).

**Example.** The practical consequence: an expired key still occupies memory until one of those two
mechanisms removes it, so `used_memory` can exceed the size of live data. With many expiring keys, the
active cycle may not keep up and memory lags behind reality.

**Advanced.** Two details worth knowing. Writing a new value to a key with `SET` **clears its TTL**
unless you use `KEEPTTL` — a classic source of entries that unexpectedly become permanent. And in a
replica, keys are not expired independently: the primary sends an explicit `DEL` when a key expires,
so a replica may briefly return a logically-expired key if read directly.

#### 4. What Redis is for

**Theory.** Anything where sub-millisecond access to a shared, structured value is worth more than
durability.

**Example.** Caching (`Q02`), rate limiting (`A08`), sessions, leaderboards, presence, deduplication
sets with TTLs, distributed locks with the caveats in `C11`, delayed and priority queues, pub/sub for
fan-out, and streams as a lightweight durable queue (`Q08`).

**Advanced.** What it is **not** for: as the only copy of data you cannot lose (`DB30`); as a queue
requiring strong delivery guarantees (use a real broker, `Q11`); or as a database with complex
queries. When someone proposes Redis as a primary store, the question to ask is what happens when a
failover loses the last few seconds of writes — because with asynchronous replication, it will.

### Interview questions

- "Which Redis structure for a leaderboard, rate limiting, a session, a job queue, unique daily
  visitors?"
- "Redis is single-threaded — how does it do 100,000 operations per second?"
- "Which Redis commands would you ban in production?"
- "Why did my key stop expiring?"

---

## Q06 · Redis persistence, eviction and memory

`Advanced` · Requires: `Q05` · Unlocks: `Q07`

### Preface

Redis keeps everything in memory, so two operational questions dominate: what happens when the process
dies, and what happens when memory fills up.

Both have configurable answers, and the defaults are frequently wrong for what people are actually
storing.

### Details

#### 1. RDB and AOF

**Theory.** **RDB** takes point-in-time snapshots by forking and writing a compact binary file.
**AOF** appends every write command to a log, with `appendfsync` controlling how often it is flushed
(`always`, `everysec` — the default, or `no`). They can be used together, which is the usual
recommendation.

**Example.** The trade: RDB restarts fast and loses everything since the last snapshot (potentially
minutes). AOF loses at most a second with the default setting and restarts more slowly, replaying the
log. With both enabled, Redis uses the AOF for recovery, so you get the better durability and can
still use the RDB for backups.

**Advanced.** The **fork** is the operational hazard. Taking an RDB snapshot forks the process;
copy-on-write means the child shares memory until pages are modified, so a write-heavy instance can
approach **double** its memory usage during a snapshot. On a 30GB instance that is an out-of-memory
kill triggered by the backup. Keep `maxmemory` well below the machine's RAM, and enable
`vm.overcommit_memory=1` — which Redis warns about at startup for exactly this reason.

#### 2. Eviction policies

**Theory.** When `maxmemory` is reached, the policy decides what happens: `noeviction` (writes
error), `allkeys-lru`, `allkeys-lfu`, `allkeys-random`, `volatile-lru`, `volatile-lfu`,
`volatile-ttl`, `volatile-random` (the `volatile-*` variants only consider keys with a TTL).

**Example.** Choosing:
- A pure cache → `allkeys-lfu`. **LFU** (least frequently used) is generally better than LRU for
  caches because a single large scan does not evict genuinely hot keys.
- A mixed instance holding cache entries **and** state (locks, sessions) → `volatile-lru`, and set
  TTLs only on the cacheable entries so the state is never evicted. Better still: separate instances.
- A data store → `noeviction`, so you get an error rather than silent data loss.

**Advanced.** The failure that sounds impossible and is not: with `allkeys-lru`, Redis can evict a
**distributed lock** or a rate-limit counter under memory pressure, producing two workers processing
the same job or a bypassed rate limit, with nothing in the logs to explain it. Separating cache from
state is the fix, and mentioning this scenario demonstrates operational experience.

#### 3. Memory accounting

**Theory.** `INFO memory` distinguishes `used_memory` (what Redis thinks it holds) from
`used_memory_rss` (what the operating system has allocated). The ratio is fragmentation.

**Example.** A fragmentation ratio above about 1.5 means significant memory is held by the allocator
but unused, often after many keys have been deleted. `activedefrag yes` lets Redis compact
incrementally. A ratio **below** 1 means Redis is being swapped to disk, which is catastrophic for
latency — Redis should never swap.

**Advanced.** Memory-efficiency techniques worth knowing: small hashes, lists and sorted sets are
stored in a compact "listpack" encoding until they exceed configured thresholds, so **many small
hashes use far less memory than the same data as individual keys** (each key has around 50-100 bytes
of overhead). Restructuring a million top-level keys into ten thousand hashes of a hundred fields can
cut memory use several-fold — a genuinely useful optimisation.

#### 4. What durability actually means here

**Theory.** Even with AOF and `appendfsync always`, Redis is not a database: replication is
asynchronous, so a failover loses writes the replica had not received.

**Example.** So the honest statement is: Redis persistence protects against a **process restart**, not
against a **failover** and not against **data loss in general**. If the data must survive, it belongs
in a system designed for that — or in AWS MemoryDB, which uses a distributed transaction log and is
genuinely durable.

**Advanced.** This is why "we store sessions in Redis" needs a follow-up: is logging every user out
acceptable? For many products, yes, and that is a fine answer. For a banking application mid-transaction,
no. Matching the durability of the store to the consequence of losing the data is the general
principle, and it is the same judgement as `DB30`.

### Interview questions

- "You used the same Redis for the cache and for your distributed locks. What can go wrong?"
- "`noeviction` and you hit `maxmemory` — what does the client see?"
- "Why can taking an RDB snapshot double your memory usage?"
- "LRU or LFU for a cache, and why?"

---

## Q07 · Redis high availability and clustering

`Advanced` · Requires: `Q06` · Unlocks: `SD05`

### Preface

One Redis instance is a single point of failure and is limited by one machine's memory. The two
answers are **Sentinel** (automatic failover for a single primary) and **Cluster** (sharding across
many primaries).

Cluster is the one with application-visible consequences: multi-key operations are restricted, which
changes how you design keys.

### Details

#### 1. Replication and Sentinel

**Theory.** A primary replicates asynchronously to replicas. **Sentinel** is a set of processes that
monitor the primary, agree when it has failed, elect a replica, promote it, and inform clients.

**Example.** A typical setup: one primary, two replicas, three Sentinel processes (an odd number, for
quorum). Clients ask Sentinel for the current primary's address rather than hard-coding it. Failover
takes a few seconds, during which writes fail.

**Advanced.** Because replication is asynchronous, a failover **loses** any writes the promoted
replica had not yet received — acknowledged to your application and then gone (`M33`). `WAIT numreplicas
timeout` blocks until N replicas have acknowledged, which gives you a stronger guarantee at the cost
of latency, and it is still not a consensus protocol. For genuine durability guarantees, Redis is the
wrong tool.

#### 2. Cluster and hash slots

**Theory.** Redis Cluster divides the key space into **16,384 hash slots**, each owned by one primary.
The client computes `CRC16(key) mod 16384` to find the slot, and connects to the node owning it. Nodes
redirect with `MOVED` (permanent) or `ASK` (during a slot migration).

**Example.** Adding a node means moving slots to it, which happens online, key by key. Clients that
understand cluster mode cache the slot map and refresh it on redirection. This is consistent hashing
with a fixed bucket count (`M32`) — the buckets are the slots, and rebalancing moves buckets rather
than rehashing keys.

**Advanced.** The application-visible limitation: **a multi-key operation must involve keys in the
same slot**, or you get a `CROSSSLOT` error. That covers `MGET`, `MSET`, `SINTER`, transactions, and
Lua scripts touching several keys. The solution is **hash tags** — only the part inside braces is
hashed, so `{user:123}:profile` and `{user:123}:settings` land in the same slot deliberately.

#### 3. Designing for Cluster from the start

**Theory.** Retrofitting hash tags means changing every key in production, so decide the grouping
before you need Cluster.

**Example.** Group by the entity that operations span: `{tenant:acme}:...` if operations are
tenant-scoped, or `{user:123}:...` if they are user-scoped. Be careful about the granularity — hash
tagging everything by tenant puts a large tenant's entire dataset on one node, recreating the hot-node
problem you were sharding to avoid (`M32`).

**Advanced.** Lua scripts in Cluster must declare their keys in `KEYS[]` and all must be in one slot;
a script computing key names internally will work on a single instance and fail in Cluster. That is a
specific and commonly-hit migration trap worth naming (`Q08`).

#### 4. Sizing and managed options

**Theory.** Choose by dataset size and operational appetite: a single primary with replicas is far
simpler and sufficient up to the memory of one machine.

**Example.** A 20GB cache fits comfortably on one instance — use replication plus Sentinel, or a
managed equivalent. A 500GB dataset needs Cluster. Managed services (ElastiCache, MemoryDB, Redis
Cloud) handle failover, backups and patching, which is usually worth the premium given how much of
Redis operations is failover handling.

**Advanced.** Know the trade with managed services: you typically cannot run arbitrary commands
(`CONFIG SET` may be restricted), version upgrades are on their schedule, and some (ElastiCache
Redis) still have asynchronous replication with the same data-loss-on-failover property. MemoryDB is
the durable variant, at higher cost and latency. Naming that distinction shows you have evaluated
them rather than assumed managed means safe.

### Interview questions

- "Your `MGET` started failing with CROSSSLOT after moving to cluster. Fix?"
- "Sentinel or Cluster for a 20GB cache? For a 500GB one?"
- "What does a Redis failover cost you?"
- "How would you design keys today so a future move to Cluster is painless?"

---

## Q08 · Redis advanced patterns

`Advanced` · Requires: `Q05` · Unlocks: `A08`, `C11`, `Q21`

### Preface

Beyond get and set, Redis has four capabilities that solve real backend problems: **Lua scripts** for
atomic multi-step operations, **pipelining** to remove round trips, **streams** as a real queue, and
**pub/sub** for fan-out.

The most important is Lua, because it is how you build anything that must read, decide and write
atomically — rate limiters, locks, and conditional updates.

### Details

#### 1. Lua scripts and atomicity

**Theory.** A Lua script runs to completion on Redis's single thread with no other command
interleaved, so the whole script is atomic. This turns any read-modify-write sequence into a single
atomic operation.

**Example.** A token bucket rate limiter (`A08`), which cannot be done correctly with separate
commands:

```lua
-- KEYS[1] = bucket key; ARGV = capacity, refill_rate, now, cost
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or tonumber(ARGV[1])
local ts     = tonumber(b[2]) or tonumber(ARGV[3])
tokens = math.min(tonumber(ARGV[1]), tokens + (ARGV[3] - ts) * ARGV[2])
if tokens < tonumber(ARGV[4]) then return 0 end
redis.call('HMSET', KEYS[1], 'tokens', tokens - ARGV[4], 'ts', ARGV[3])
redis.call('EXPIRE', KEYS[1], 60)
return 1
```

One round trip, atomic, no race.

**Advanced.** Keep scripts short — they block the server (`Q05`). Always declare keys in `KEYS[]`
rather than constructing them inside the script, both for Cluster compatibility (`Q07`) and because
Redis needs to know which keys are touched. Use `EVALSHA` with the cached script hash to avoid sending
the source every time, and handle `NOSCRIPT` by re-loading. Functions (Redis 7) are the modern,
persistent replacement for script caching.

#### 2. Transactions versus scripts

**Theory.** `MULTI`/`EXEC` queues commands and executes them together without interleaving — but it
is not a transaction in the database sense: there is no rollback, and a command failing mid-block does
not undo the earlier ones. `WATCH` adds optimistic locking: if a watched key changes before `EXEC`,
the block aborts.

**Example.** `WATCH key; GET key; MULTI; SET key newval; EXEC` is compare-and-set. If another client
changed the key, `EXEC` returns nil and you retry. This is optimistic concurrency (`DB10`) and it
works, and a Lua script is usually simpler because it does not need a retry loop.

**Advanced.** The practical rule: use Lua for anything conditional. `MULTI`/`EXEC` is mainly useful
for batching independent commands atomically, and even then pipelining is often what you actually
wanted (`below`). Being clear that Redis transactions do not roll back is a good discriminator — many
people assume they behave like SQL transactions.

#### 3. Pipelining

**Theory.** Pipelining sends many commands without waiting for each reply, so N commands cost one
round trip instead of N. It is not atomic — other clients' commands can interleave — it is purely a
latency optimisation.

**Example.** Fetching 100 keys individually over a 1ms link costs 100ms. Pipelined, about 1ms. Most
client libraries expose this directly, and many do it automatically for concurrent commands. For
bulk loading, pipelining is the difference between minutes and seconds.

**Advanced.** Do not pipeline unboundedly: the replies accumulate in the client's memory and the
server's output buffer, and a very large pipeline can trigger the client output buffer limit and have
your connection closed. Batch in chunks of a few hundred to a few thousand. This is the same bounded-
buffer discipline as everywhere else (`C16`).

#### 4. Streams and pub/sub

**Theory.** **Pub/sub** is fire-and-forget: messages go to currently-connected subscribers and are
never stored. A subscriber that is disconnected misses them entirely. **Streams** are an append-only
log with consumer groups, acknowledgements and a pending-entries list — a genuine queue.

**Example.** Use pub/sub for cache invalidation broadcasts and presence, where loss is acceptable
(`Q09`). Use streams for work queues: `XADD` to append, `XREADGROUP` to consume as part of a group,
`XACK` to acknowledge, and `XAUTOCLAIM` to take over messages from a consumer that died holding them.
That last one is what makes streams safe — unacknowledged messages are recoverable, which pub/sub and
plain lists cannot do.

**Advanced.** Compared with Kafka (`Q12`), streams give you consumer groups, acknowledgements and
replay, with a much simpler operational story, and without Kafka's partition-level ordering guarantees
across many brokers or its retention scale. For moderate volumes where you already run Redis, streams
are an entirely reasonable choice — and you must cap the stream length (`XADD ... MAXLEN ~ 1000000`)
or it grows until memory is exhausted.

### Interview questions

- "Write an atomic token-bucket rate limiter in Redis. Why does it need Lua?"
- "Why is `DEL lock` on release a bug?"
- "Redis transactions — do they roll back?"
- "Pub/sub or streams for a work queue?"

---

## Q09 · HTTP, CDN and in-process caching

`Intermediate` · Requires: `Q01`, `A04` · Unlocks: `SD05`

### Preface

The cheapest cache hit is one your servers never see. HTTP caching and a CDN handle that, using
headers you already control (`A04`).

At the other end, an in-process cache is the fastest possible lookup — no network at all — and
introduces per-instance inconsistency, because each pod has its own copy.

### Details

#### 1. CDN caching

**Theory.** A CDN caches responses at edge locations near users. It serves cacheable content without
touching your origin, and reduces latency by geography.

**Example.** What it is for: static assets (with content-hashed filenames and a very long `max-age`);
public API responses that are the same for everyone; images and media. What it is not for: anything
personalised, unless you are extremely careful with `Vary` and `Cache-Control: private` (`A04`).

**Advanced.** Two features worth knowing. **Origin shielding** designates one edge location as the
only one that talks to your origin, so a cold cache across 200 edges produces one origin request
rather than 200 — which directly addresses the stampede problem at CDN scale. And
**stale-while-revalidate** plus **stale-if-error** let the edge serve slightly old content while
refreshing, and keep serving it if your origin is down — turning the CDN into an availability layer,
not just a performance one.

#### 2. Purging versus versioning

**Theory.** To change cached content you either **purge** (tell the CDN to drop a key, which takes
seconds to propagate and has rate limits) or **version** the URL so the new content has a new key.

**Example.** Versioning is strictly better where you control the URL: `app.a4f2c1.js` with a one-year
`max-age` is never stale, never needs purging, and makes rollbacks trivial because the old file still
exists. Purging is necessary for content at a stable URL — an API response, an article page — and
should be treated as best-effort rather than instant.

**Advanced.** **Surrogate keys** (tag-based purging) are the scalable version: tag each response with
the entities it contains, and purge by tag when an entity changes — one call invalidates every page
containing that product. This is the CDN equivalent of tag-based cache invalidation (`Q03`), and it is
how large content sites keep long TTLs with fresh content.

#### 3. In-process caching

**Theory.** A cache in the application's memory: nanosecond access, no network, and private to that
process.

**Example.** Right for: immutable or slow-changing reference data (currency codes, categories),
compiled artefacts (regexes, templates, parsed schemas), and feature flags with a short refresh.
Wrong for: anything a user will notice changing, because with N pods a user's successive requests hit
different copies and see values flip back and forth (`F15`).

**Advanced.** It must be **bounded** — an unbounded in-memory cache is the single most common Node
memory leak (`C12`). Use an LRU with a maximum size, and prefer bounding by **estimated bytes** rather
than entry count when entries vary in size, because 10,000 entries could be 10MB or 10GB.

#### 4. Two-level caching

**Theory.** A local cache in front of a shared cache: local hit (nanoseconds) → shared hit
(microseconds to a millisecond) → origin. It gives you both speed and consistency, at the cost of
invalidating the local copies.

**Example.** The invalidation mechanism is a broadcast: when a value changes, publish the key on Redis
pub/sub, and every instance drops its local copy (`Q08`). Local TTLs should be short (seconds) so a
missed broadcast self-corrects quickly. This design is what effectively solves hot keys (`Q04`).

**Advanced.** The failure mode: a lost pub/sub message means one pod serves stale data until its local
TTL expires — and pub/sub has no delivery guarantee, so this **will** happen. Keep local TTLs short
enough that the worst case is acceptable, and do not use two-level caching for data where a few
seconds of staleness on one instance is unacceptable. Naming both the mechanism and its unavoidable
weakness is the complete answer.

### Interview questions

- "You added a 60-second in-memory cache and users see data flip-flop between old and new. Why?"
- "Purge or version a changed asset? Why?"
- "What is a surrogate key at the CDN?"
- "Design a two-level cache and tell me how it fails."
