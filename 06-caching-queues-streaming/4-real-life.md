[← back to the field index](README.md)

# Caching, Queues & Streaming · Part 4 — Real-life production problems

The first three parts teach the nodes. This part is what an interviewer does once they believe you
know the textbook: they describe a real incident — symptoms, numbers, stack — and wait to see whether
you have been paged for it before. The good answers are rarely "add a cache" or "add more consumers".
They are the counter-intuitive fix, the default nobody reads, the metric that actually points at the
cause.

How to use it: read the pre-knowledge once, properly. Then, for each question, answer out loud as if
in the interview — first what you would *look at*, then what you think is happening, then the fix and
its trade-off — and only then read the **Direction** line. If your answer did not reach the same
insight, go back to the section it names.

The levels run from the incidents asked in almost every senior loop (Level 1) to the rare multi-system
failures that only come up with staff-level interviewers or teams that run this infrastructure
themselves (Level 10).

---

## Pre-knowledge

### 1. Redis execution model and slow commands

**One thread runs every command.** Redis executes commands on a single main thread. Since 6.0,
`io-threads` can parallelise socket reads/writes and protocol parsing, but command execution is still
serial. So any command that is O(N) on a large N blocks *every* client for its whole duration. A 200 ms
command at 50,000 ops/s means 10,000 requests queue behind it, and their client-side timeouts fire —
which looks like "Redis is down" while Redis CPU shows one short spike (`Q05`).

**The usual culprits.**
- `KEYS pattern` — scans the whole keyspace. On 30M keys it takes seconds. Replace with `SCAN` with a
  cursor and `COUNT` (a hint, not a limit); it never blocks for long, but can return duplicates and
  must be idempotent on the caller side. Rename or disable dangerous commands with ACLs
  (`-@dangerous`) or `rename-command` in older setups.
- `DEL` on a big key — freeing a hash with 5M fields is O(N) in the number of elements, and blocks.
  `UNLINK` removes the key from the keyspace in O(1) and frees memory in a background thread. The same
  applies to `FLUSHALL`/`FLUSHDB` (use `ASYNC`).
- **Implicit deletes are also DELs.** A big key that *expires* or is *evicted*, or is overwritten by
  `SET`/`RENAME`, is freed synchronously unless `lazyfree-lazy-expire`, `lazyfree-lazy-eviction`,
  `lazyfree-lazy-server-del` and `lazyfree-lazy-user-del` are `yes`. They defaulted to `no` for most
  of Redis's history (Valkey 8 and recent Redis versions flip several to `yes` — check yours). The
  classic incident: a nightly job builds a 2 GB sorted set with a TTL; the TTL fires at 03:00 and Redis
  stalls for 1.5 s every night.
- `HGETALL`, `SMEMBERS`, `LRANGE 0 -1`, `ZRANGE 0 -1` on big collections — O(N) and they also produce a
  huge reply that sits in the client output buffer. Use `HSCAN`/`SSCAN`/`ZSCAN` or paginate.
- `SORT`, `SUNION`/`SINTER` on large sets, `ZUNIONSTORE`, `LREM`, `LINSERT`, `LINDEX` on long lists.
- **Lua scripts and functions** run atomically, so a slow script is a slow command. After
  `busy-reply-threshold` (formerly `lua-time-limit`, default 5 s) Redis answers other clients with
  `BUSY`; `SCRIPT KILL` works only if the script has not written yet — otherwise the only way out is
  `SHUTDOWN NOSAVE`. Never loop over an unbounded key set in Lua.
- `MONITOR` — streams every command to the monitoring client; it can halve throughput. Never leave it
  on in production; sample for seconds only.
- `EVALSHA` after a restart or failover returns `NOSCRIPT` because the script cache is not persisted
  or replicated as scripts; clients must fall back to `EVAL` (most libraries do), or use Redis 7
  `FUNCTION`s, which are persisted.

**Pipelining vs latency.** Round trips dominate, not Redis CPU. 1,000 sequential `GET`s over a 0.5 ms
network = 500 ms; one pipelined batch or one `MGET` = about 1 ms. N+1 cache calls are the most common
"Redis is slow" complaint and it is not Redis. The opposite trap: a pipeline or `MGET` of 100,000 keys
is itself one huge blocking unit plus a huge reply — batch in chunks of a few hundred.

**Measuring.** `SLOWLOG GET` records commands slower than `slowlog-log-slower-than` (default 10,000
µs = 10 ms) and keeps `slowlog-max-len` (default 128) entries — it measures *execution* only, not
queueing or network, so a client can see 2 s timeouts while the slowlog is empty (the cause is then
network, client-side pool starvation, fork, or swapping). `LATENCY DOCTOR` and
`latency-monitor-threshold` capture event classes (fork, expire-cycle, eviction, AOF fsync).
`INFO commandstats` gives calls and µs per command — the fastest way to find the command family that
eats the CPU. `redis-cli --latency` and `--latency-history` measure from the client side.

**Blocked clients and CPU.** `INFO stats` → `instantaneous_ops_per_sec`; `INFO cpu` →
`used_cpu_sys`/`used_cpu_user`; the main thread pinned at 100% of one core is saturation even if the
host shows 12% (one of eight cores). Monitor *per-core* or the Redis process's main thread.

### 2. Redis memory: big keys, hot keys, encodings, expiry and eviction

**Big keys.** A key whose value is large (a 10 MB string) or whose collection is huge (a hash with
millions of fields). Problems: blocking DEL/expiry (§1), blocking migration in cluster resharding
(§4), network saturation (a 1 MB value read 5,000 times per second is 5 GB/s — the NIC saturates
long before the CPU), replication spikes, and uneven memory across cluster shards. Find them with
`redis-cli --bigkeys` (largest per type, by element count), `redis-cli --memkeys` (by bytes, uses
`MEMORY USAGE`), or offline by parsing an RDB file (`rdb-tools`, `redis-rdb-cli`) — the offline
route touches nothing in production.

**Hot keys.** A key receiving a disproportionate share of requests. `redis-cli --hotkeys` works only
under an LFU eviction policy (it reads `OBJECT FREQ`). Otherwise instrument the client (sample key
names per request), or sample `MONITOR` for a few seconds on a replica. The fixes are in §5.

**Compact encodings.** Small hashes, sorted sets and lists are stored as `listpack` (formerly
`ziplist`), small integer sets as `intset` — dramatically smaller than the hashtable/skiplist forms.
Thresholds: `hash-max-listpack-entries 128`, `hash-max-listpack-value 64` bytes,
`zset-max-listpack-entries 128`, `set-max-intset-entries 512`. One field with a value over 64 bytes
converts the whole hash to a hashtable, and it never converts back. The famous trick (Instagram's
media-to-user mapping): instead of 300M string keys, bucket them into hashes of ~1,000 fields each
(`HSET media:1155 1155315 939`), cutting memory by around an order of magnitude because per-key
overhead (dictEntry, robj, expiry entry) disappears. The cost: TTLs are per key, not per field
(Redis 7.4 added `HEXPIRE` for per-field TTL).

**Expiry mechanics.** Keys expire lazily (checked on access) and actively: a background cycle samples
keys with TTLs (20 per round) and keeps going while too many sampled keys were expired. If millions of
keys share the same expiry second — because a batch job wrote them all with `EX 3600` at the same
moment — the active cycle spends a lot of CPU and latency spikes exactly one TTL later. Add jitter to
TTLs (§5). Replicas do not expire keys themselves; they wait for the primary's `DEL`, but logically
hide expired keys on reads (since 3.2).

**Eviction.** `maxmemory` plus `maxmemory-policy`:
- `noeviction` (the default) — writes fail with `OOM command not allowed when used memory >
  'maxmemory'` once full. Correct for queues, locks and sessions (BullMQ requires it); wrong for a
  pure cache.
- `allkeys-lru` / `allkeys-lfu` — the right choice for a pure cache. LFU resists a one-off scan
  flushing the hot set; LRU does not.
- `volatile-lru` / `volatile-lfu` / `volatile-ttl` / `volatile-random` — evict only keys *with a TTL*.
  If most keys have no TTL, there is nothing to evict and Redis behaves like `noeviction` — writes
  start failing while you believed you had an eviction policy. The classic surprise.
- LRU/LFU are approximated by sampling `maxmemory-samples` (default 5) keys; 10 is closer to true LRU
  for slightly more CPU.
- Mixing a cache and durable data (sessions, rate-limit counters, job queues) in one instance under
  `allkeys-*` means eviction silently deletes the durable data. Separate instances by eviction
  policy.

**What maxmemory does not see.** `used_memory` is what Redis allocated; `used_memory_rss` is what the
OS gave it. `mem_fragmentation_ratio` = rss / used. Above ~1.5 is fragmentation (after mass deletes
of variable-sized values; fix with `activedefrag yes` on jemalloc or a controlled restart/failover);
below 1.0 means Redis is being **swapped**, which is far worse — latency goes from microseconds to
milliseconds. Replication backlog, client output buffers and AOF buffers consume memory too. Client
output buffers are the silent one: a slow client doing `HGETALL` on big keys or a slow Pub/Sub
subscriber can accumulate hundreds of MB (`CLIENT LIST` shows `omem` per client). Limits:
`client-output-buffer-limit normal 0 0 0` (unlimited!), `replica 256mb 64mb 60`, `pubsub 32mb 8mb 60`.
Set `maxmemory` well below the machine/container limit — leave room for fork COW (§3), buffers and
fragmentation. On ElastiCache, `reserved-memory-percent` (25% default) exists for exactly this.

**Keys without TTL are a leak.** A cache whose memory grows forever usually has a code path that
writes without `EX`. Find by sampling: `SCAN` + `TTL` (returns -1 for no expiry) + `OBJECT IDLETIME`
per key prefix. `INFO keyspace` shows `keys` vs `expires` per database — a large gap is the tell.

### 3. Redis persistence, fork and replication

**Fork and copy-on-write.** `BGSAVE` (RDB) and `BGREWRITEAOF` call `fork()`. The child shares memory
pages with the parent copy-on-write, so the snapshot costs little memory *if writes are light*. Under
heavy writes every touched page is copied, so memory can approach **2×** during the save. With
Transparent Huge Pages enabled, a single-byte write copies a 2 MB page instead of 4 KB — COW
amplification and latency spikes; Redis warns at startup and the fix is
`echo never > /sys/kernel/mm/transparent_hugepage/enabled`. The fork itself also blocks: copying page
tables for a 50 GB process takes hundreds of milliseconds (`INFO persistence` → `latest_fork_usec`),
worse on some VM hypervisors. If `vm.overcommit_memory` is 0, the kernel may refuse the fork on a
large instance — `Can't save in background: fork: Cannot allocate memory` — and with
`stop-writes-on-bgsave-error yes` (default) Redis then **rejects all writes**. Set
`vm.overcommit_memory = 1`.

**Container OOM.** In Kubernetes, a Redis with `maxmemory 6gb` in an 8 GiB-limit pod is killed by the
OOM killer during BGSAVE under write load, then restarts empty or from an old RDB. Rule of thumb:
`maxmemory` ≤ ~50–60% of the limit if you fork under write load, or do not persist on the primary.

**AOF.** `appendfsync everysec` (default) loses up to ~1–2 s on crash; `always` is safe and slow.
If the disk stalls, the fsync thread blocks and Redis delays writes ("Asynchronous AOF fsync is taking
too long"). AOF rewrite forks, like BGSAVE. Redis 7 uses a multi-part AOF (base + incremental files).

**Persistence plus automatic restart is dangerous.** If a primary with persistence disabled restarts
quickly (faster than Sentinel/cluster notices), it comes back **empty**, and its replicas faithfully
replicate the empty dataset — wiping every copy. Either persist on the primary or disable automatic
restart so failover happens first.

**Replication.** Asynchronous. A replica connects, sends `PSYNC <replid> <offset>`; if the offset is
still inside the primary's `repl-backlog-size` (default **1 MB** — tiny), it gets a partial resync;
otherwise a **full resync**: the primary forks, produces an RDB (to disk or, with
`repl-diskless-sync yes`, streamed directly — the default since 7.0), and meanwhile buffers all new
writes in that replica's output buffer. If that buffer exceeds `client-output-buffer-limit replica
256mb 64mb 60` (hard 256 MB, or 64 MB sustained for 60 s), the primary drops the replica — which
reconnects and requests another full sync. **The full-resync loop**: a 20 GB dataset takes minutes
to transfer, writes at 5 MB/s overflow the buffer, the sync never completes, the primary forks
repeatedly (CPU, COW memory, network), and eventually the primary is OOM-killed. Fixes: raise the
replica output buffer limit to hold (sync duration × write rate), raise `repl-backlog-size` to hundreds
of MB so short network blips give partial resyncs, reduce dataset size per shard.

**Failover and PSYNC2.** Since 4.0, a promoted replica keeps the replication ID history so other
replicas can partially resync to it. Asynchronous replication means acknowledged writes can be lost
on failover; `WAIT numreplicas timeout` narrows but does not close the window (it is not a
consensus protocol). `min-replicas-to-write` / `min-replicas-max-lag` make an isolated primary stop
accepting writes, limiting split-brain loss.

### 4. Redis HA, Cluster and client behaviour

**Sentinel.** Monitors the primary; after `down-after-milliseconds` (typical 5–30 s) and a quorum
agreement, promotes a replica and tells clients via Sentinel queries/Pub/Sub. Clients must be
Sentinel-aware; clients configured with a fixed IP keep talking to the old primary.

**Cluster.** 16,384 hash slots; a key's slot is `CRC16(key) mod 16384`. Each primary owns a range.
- **Multi-key commands** (`MGET`, `MSET`, `SUNIONSTORE`, `RENAME`, Lua with several `KEYS`,
  `MULTI/EXEC` across keys) require all keys in the same slot, else `CROSSSLOT Keys in request don't
  hash to the same slot`. **Hash tags** force co-location: only the substring in `{...}` is hashed, so
  `{user:42}:cart` and `{user:42}:profile` share a slot. Overuse (`{tenant}` for a huge tenant) creates
  a hot or oversized slot you cannot split.
- **Redirections.** `MOVED` (slot lives elsewhere; client should refresh its slot map) and `ASK`
  (slot migrating; one-shot redirect). A client that does not cache the slot map or refreshes it on
  every `MOVED` during resharding can multiply load. Client libraries that fan out `MGET` across slots
  hide the cost: one `MGET` of 200 keys becomes up to N node round trips.
- **Resharding** moves keys with `MIGRATE`, which is blocking per key — a 1 GB key freezes both source
  and target. Find big keys before resharding.
- `cluster-node-timeout` (default 15 s) sets failure detection; `cluster-require-full-coverage yes`
  (default) makes the **whole cluster refuse queries** if any slot is uncovered — one shard with both
  primary and replica lost takes everything down. Setting it to `no` keeps the rest serving.
- **Pub/Sub in cluster** broadcasts every `PUBLISH` to every node (cluster bus traffic grows with
  node count). Redis 7 sharded Pub/Sub (`SPUBLISH`/`SSUBSCRIBE`) keeps a channel on its slot.
- A cluster has no cross-slot transactions and only database 0.
- **Planned failover.** `CLUSTER FAILOVER` run on a replica is coordinated: the primary stops
  accepting writes, the replica catches up to its replication offset, then takes over — no lost
  writes. Upgrade replicas first, fail over manually, then upgrade the old primaries. Killing primaries
  and letting automatic failover happen loses the async tail and costs `cluster-node-timeout` of
  unavailability per shard. A too-low `cluster-node-timeout` on a noisy network produces false
  failovers (and a lost write tail each time).

**Pub/Sub vs Streams.** Pub/Sub is fire-and-forget: a subscriber that is disconnected (failover,
deploy, output-buffer-limit kill) simply misses messages, and nothing tells it. Streams (`XADD`,
`XREADGROUP`) persist entries with consumer groups: unacked entries sit in the pending entries list
(`XPENDING`); a dead consumer's entries must be reclaimed with `XAUTOCLAIM` after an idle time or they
are stuck forever; the stream must be trimmed (`XADD ... MAXLEN ~ 1000000`) or it grows until
`maxmemory`.

**Managed services and DNS.** ElastiCache/MemoryDB/Azure Cache fail over by moving a DNS name to a new
primary. Clients (or the JVM, `networkaddress.cache.ttl`, or a long-lived connection pool) that keep
the old IP connect to the demoted node — now a replica — and get `READONLY You can't write against a
read only replica` for minutes. Fixes: short DNS caching, reconnect/refresh the topology on
`READONLY`/`MOVED`, use the cluster-mode or Sentinel-aware client.

**Connection storms.** Default `maxclients` is 10,000; the accept backlog is `tcp-backlog 511`
(capped by `net.core.somaxconn`). After a failover, deploy or network blip, 800 pods × pool size 50
reconnect simultaneously — 40,000 connection attempts, TLS handshakes (expensive: TLS on a single
thread can consume most of Redis's CPU), `AUTH`, `CLIENT SETNAME`, `SELECT`, cluster `CLUSTER SLOTS`
calls. Redis spends its time accepting connections, clients time out and retry, and the storm
sustains itself. Fixes: reconnect with exponential backoff and **jitter**; smaller pools (Redis is
single-threaded — 50 connections per pod rarely help); a proxy (Envoy, twemproxy, Redis Cluster proxy)
to multiplex connections; keep connections long-lived (no connect-per-request, a classic in PHP and
serverless — Lambda cold starts reconnecting at scale).

**Client timeouts and retries.** A 1 s command timeout with 3 retries on a saturated Redis quadruples
load. Retries on non-idempotent commands (`INCR`, `LPUSH`) after a timeout double-apply — the command
may have executed. A cache client should have a **short** timeout (tens of ms) and fall through to the
source, not retry.

**Distributed locks.** `SET key token NX PX 30000`; release with a Lua compare-and-delete on the token.
Locks expire while the holder is paused (GC, stalled I/O), so a second holder appears; Redlock does not
fix this. Correctness needs a **fencing token** (monotonic number checked by the resource being
protected). Use Redis locks for efficiency (avoid duplicate work), not for correctness (`Q08`).

### 5. Stampedes, hot keys and cache failure

**Stampede (dogpile).** A hot key expires; every concurrent request misses and recomputes. Tools, from
cheapest (`Q04`):
- **Jittered TTLs** — `ttl = base × (1 ± 0.1 × random)` prevents keys written together from expiring
  together (the synchronised-expiry avalanche).
- **Request coalescing / singleflight** — one in-flight load per key per process; others await it. With
  200 pods this still means 200 loads — combine with the next.
- **Distributed recompute lock** — `SET lock:key NX PX` for the recomputing process; losers serve stale
  or wait briefly and re-read. Lock TTL must exceed the recompute time, or two recomputes run.
- **Stale-while-revalidate** — store a *soft* expiry inside the value and a longer *hard* TTL on the key.
  After the soft expiry, one caller (guarded by the lock) refreshes while everyone else keeps getting
  the stale value. This also gives **stale-if-error**: if the source is down, keep serving the stale
  copy instead of failing.
- **Probabilistic early expiration (XFetch)** — recompute early if
  `now − delta × beta × ln(rand()) ≥ expiry`, with `delta` the measured recompute time, `beta ≈ 1`.
  Hot keys get refreshed by one reader slightly before expiry; cold keys are not refreshed. No
  coordination required.
- **Refresh-ahead / never expire** — a background job refreshes a known set of hot keys. In-process
  libraries implement this: Caffeine `refreshAfterWrite` (serves old value while reloading) vs
  `expireAfterWrite` (blocks the caller).

**Hot keys.** One key saturates one shard (or one NIC). Options:
- **Local L1 cache** in each process with a very short TTL (1–5 s) in front of Redis — turns 200,000
  req/s on one key into (pods × 1 per TTL). The best fix nearly always; the cost is bounded staleness.
  Redis 6 client-side caching (`CLIENT TRACKING`) adds server-pushed invalidation.
- **Key replication** — write `hot:{i}` for i in 0..N−1 on different slots, read a random one. Writes
  must update all copies.
- **Read from replicas** for read-heavy hot keys (accepting replica lag).
- **Split the value** if the key is a big collection (§2).
- **Hot write keys** (a like counter `INCR`ed 50,000 times a second) cannot be fixed with replicas or
  L1 caches. Split the counter into N sub-keys on different slots (`likes:{post}:{i}` with random i),
  sum on read (and cache the sum); or aggregate in-process and flush `INCRBY` every 100 ms
  (write-behind — accepting loss of the in-memory delta on crash).

**Penetration.** Requests for keys that do not exist miss the cache every time and hit the database —
by accident (a client polling a deleted ID) or deliberately (enumeration). **Negative caching** stores
"not found" with a short TTL; a **Bloom filter** of existing IDs rejects impossible lookups without
touching either.

**Cache failure (avalanche).** When Redis goes away, the database sees 100% of reads. The honest
questions: can the database survive the miss rate? (Usually not, if the hit rate was 98%.) Design:
short cache timeouts and a **circuit breaker** so a dead Redis costs microseconds per request rather
than a timeout; load shedding/degradation at the source; a local L1 that keeps serving hot items; and
not letting cache errors become user-facing errors. Also consider the reverse: a cache that is *slow*
rather than *down* is worse, because the breaker never trips unless it counts latency.
Redis used for *control* rather than caching — a rate limiter, a feature-flag store, a session check
— needs an explicit failure mode decided in advance: a rate limiter that fails closed turns a Redis
outage into a total outage; failing open to a coarse per-process limiter keeps protection without the
dependency.

**Hit-rate traps.** Hit ratio falls when the key includes something high-cardinality by mistake
(a timestamp, a request ID, an unsorted query-string or JSON with a different field order,
locale/currency variations). Normalise keys. A sudden hit-rate drop after a deploy is usually a key
format change (§7).

### 6. Consistency: invalidation races, leases, versions, negative caching, poisoning

**The cache-aside race.** Reader A misses, reads the DB (value v1). Writer B updates the DB to v2 and
deletes the key. Reader A — delayed by a GC pause or slow network — now `SET`s v1. The cache holds
stale data until the TTL, with no further writes to correct it. Deleting instead of setting on write
narrows the race but does not eliminate it. Remedies (`Q02`, `Q03`):
- **Leases (Facebook memcache paper, 2013).** On a miss, the cache hands the reader a lease token; the
  reader's `SET` is accepted only with a valid token, and a `DELETE` invalidates outstanding tokens —
  so A's stale fill is rejected. Leases also throttle stampedes: the cache issues at most one token
  per key every ~10 s and tells other missers to wait briefly and retry (or accept a slightly stale
  value).
- **Versioned writes.** Store a version (row version, `updated_at`, or log sequence number) with the
  value and use a Lua compare-and-set: only write if the new version is greater than the cached one.
- **Short TTL as the backstop** — every entry must eventually expire, so the worst case is bounded.
- **Delayed double delete** — delete, write DB, delete again after (replica lag + typical read time).
  A pragmatic hack, not a guarantee.
- **Invalidation from the change log (CDC).** Tail the binlog/WAL (Debezium, Facebook's mcsqueal) and
  delete keys after commit. Survives application crashes between DB write and cache delete — the dual
  write problem — and catches writes from scripts and other services that bypass the application.

**Replica lag refills stale data.** The writer updates the primary and deletes the key; a reader
misses and fills from a **read replica** that has not applied the write yet — the stale value is back,
now with a fresh TTL. Fixes: fill from the primary for a short time after a write to that entity
(Facebook's "remote marker"); invalidate from the replica's CDC stream rather than the primary's;
or delay the invalidation by the replica lag.

**Composition vs denormalised entries.** Caching "user with team name embedded" means a team rename
must invalidate every member's entry — a fan-out you cannot find without an index. Cache entities
separately (`user:42`, `team:7`) and compose at read time (an `MGET`), so each fact is invalidated in one
place. Denormalise only immutable or very-rarely-changing data.

**Multi-region.** Each region's cache is invalidated only if the invalidation reaches it. Application
deletes in the writing region do not reach the others; drive invalidation from each region's **local
database replica's** change stream (so invalidation arrives after the data), which is how Facebook
invalidates regional memcache pools from MySQL replication.

**Transactions and caching.** Invalidating inside a DB transaction that later rolls back (or before
commit) lets a reader refill the old value before commit. Invalidate **after commit** (after-commit
hooks, outbox, CDC).

**Negative caching pitfalls.** Caching "not found" for 10 minutes causes "I just created my account and
the API says it doesn't exist". Creating an entity must delete its negative entry; keep negative TTLs
short (seconds to a minute).

**Poisoning.** Something wrong gets cached and served to everyone:
- Caching an **error or partial result** — a timed-out downstream returned an empty list, and the
  empty list is cached for an hour. Only cache successful, complete results; cache errors, if at all,
  for seconds and separately.
- **Wrong key scope** — a per-user or per-tenant response cached under a shared key (missing user ID,
  missing auth context, missing locale). A data leak, not just a bug.
- **CDN/HTTP cache poisoning** — a response varies on a header that is not in the cache key (unkeyed
  `X-Forwarded-Host`, `Accept-Encoding`, `Authorization`), or a response with `Set-Cookie` is cached
  and serves one user's session to others. Rule: never cache responses with `Set-Cookie` or
  `Authorization`-dependent content at shared caches; use `Cache-Control: private`; control `Vary`.
- **Serialisation mismatch** — v2 of the code writes a format v1 cannot read; during a rolling deploy
  both run. Version the key prefix or the payload, and make readers tolerant (treat decode failure as a
  miss, not a 500).

**Recovery tools.** Namespace versioning (`v7:product:42`, bump to v8 to invalidate everything
logically without `SCAN`+`DEL`) — at the cost of a fully cold cache (§7). Tag or surrogate keys for
group purges (CDN surrogate keys, Redis sets of member keys). A kill switch that bypasses the cache per
key pattern.

### 7. Cache lifecycle: warm-up, deploys, local and CDN caches

**Cold start.** A new Redis cluster, a flush, a failover to a replica that was not fully synced, or a
key-prefix change on deploy all produce a hit rate near zero — and the database sees the full read
load. Mitigations:
- **Warm before switching traffic** — replay the top-N keys from access logs, or copy from the old
  cluster; Facebook's **cold cluster warm-up** lets a new cluster fill its misses from a warm cluster
  rather than the database (with a hold-off on deletes to avoid races).
- **Shift traffic gradually** (1% → 10% → 100%) so the cache fills at a rate the database survives.
- **Persist the cache** (RDB) so a restart is warm — trading fork cost (§3) for warm starts.
- **Do not change key formats and deploy everywhere at once.** A key-format change is a full flush in
  disguise; roll it out gradually, or dual-read (new key, fall back to old key) for a TTL period.

**Consistent hashing.** memcached-style client-side sharding with modulo hashing remaps almost every
key when a node is added or removed (N → N+1 moves ~N/(N+1) of keys) — a scale-up causes a
near-total miss storm. Consistent hashing (ketama) moves only ~1/N. Redis Cluster slots avoid this,
but moving slots still produces misses if the client is not slot-aware.

**Local (in-process) caches.** Fastest, and each pod has its own copy: 100 pods means 100 fills after
each expiry and 100 different staleness windows. Invalidation via Redis Pub/Sub is at-most-once — a
disconnected pod misses messages and stays stale forever unless TTLs bound it. Memory counts against
the heap (GC pressure in the JVM, heap limits in Node). Size by entry count *and* bytes.

**CDN specifics.** The cache key (path, query-string rules, headers in `Vary`), `s-maxage` vs
`max-age`, `stale-while-revalidate` and `stale-if-error` directives (RFC 5861), **request collapsing**
at the edge, and an **origin shield** tier so N edge PoPs make one origin request instead of N.
Deploy-order trap: new HTML referencing new hashed assets goes live before the assets exist on the
origin, the CDN caches the **404** for the asset (often with a default TTL), and the page stays
broken after the assets arrive. Deploy assets first, never cache 404s long, purge by surrogate key.

### 8. Messaging fundamentals as they break in production

**At-least-once is the default everywhere.** Every broker redelivers on consumer crash, timeout or
rebalance. Consumers must be **idempotent**: a processed-message table with a unique constraint on
the message ID (inserted in the same DB transaction as the side effect — the *inbox* pattern), natural
idempotency (`UPSERT`, "set status to X"), or idempotency keys passed to downstream APIs (payment
providers support them). "Exactly-once" in marketing means "effectively once with idempotence"
(`Q10`, `M16`).

**Depth is the wrong alert; age is the right one.** Queue depth depends on traffic; **age of the
oldest message** (or consumer lag expressed in *time*) is what users feel and what an SLO can be
written against. A queue with 1M messages that drains in 30 s is fine; 50 messages that are 2 hours
old is an incident.

**Little's law for sizing.** Concurrency needed = throughput × processing time. 2,000 msg/s at 50 ms
each needs 100 concurrent workers. If processing time doubles because a downstream slows down, you
need twice the concurrency — or the queue grows without bound. Autoscaling on depth amplifies this
into the downstream (more consumers → more load on the slow dependency → slower). Scale on age, cap
concurrency at what the dependency can take, and use backpressure.

**Queues hide failures.** A queue in front of a failing consumer converts an immediate error into a
growing backlog that pages nobody until it is hours deep — and when the fix lands, the backlog
**drains as a flood** into downstream systems (a "retry storm" or "thundering backlog"). Throttle the
drain; decide whether old messages are still worth processing (a 3-hour-old "your driver is 2 minutes
away" push notification should be dropped — set message TTLs or check age on consume).

**Ordering is per partition / per group, and retries break it.** Any retry mechanism that moves a
message aside (retry topic, requeue at the back, DLQ) lets later messages for the same key overtake it.

**Large payloads.** Brokers are not blob stores: Kafka's default `message.max.bytes` is ~1 MB, SQS
256 KB historically (raised to 1 MiB in 2025), RabbitMQ handles large messages poorly (memory, head of
line). Use the **claim-check** pattern: store the payload in S3/object storage and send a pointer —
with a lifecycle rule longer than the maximum retention/replay window.

### 9. Kafka producers, durability and the ISR

**Producer defaults (Kafka 3.0+).** `acks=all` and `enable.idempotence=true` are the defaults;
`retries` is effectively infinite and bounded by `delivery.timeout.ms` (default 120 s);
`max.in.flight.requests.per.connection` default 5. With idempotence on, the broker de-duplicates by
(producer ID, sequence number) and preserves order with up to 5 in-flight requests. **Without**
idempotence, `retries > 0` and `max.in.flight > 1` can reorder: batch 1 fails, batch 2 succeeds,
batch 1 is retried and lands after 2. Old clients and some wrappers still ship with idempotence off.
Idempotence only covers retries within one producer session — an application that crashes and resends
produces a true duplicate (`Q13`).

**Batching and blocking.** `batch.size` (16 KB) and `linger.ms` (0 historically; 5 ms default in
Kafka 4.0) govern batching; tiny batches waste broker CPU on request overhead. `buffer.memory` (32 MB)
is the producer's local buffer; when brokers are slow and it fills, `send()` **blocks** for up to
`max.block.ms` (60 s) — inside a web request thread, that turns a Kafka slowdown into an API outage.
Treat `send()` as potentially blocking, or fail fast with a small `max.block.ms`. Compression
(`lz4`/`zstd`) at the producer typically gives 3–5× on JSON and cuts broker disk and network.

**Partitioner.** Keyed records go to `murmur2(key) mod partitionCount`. **Adding partitions changes
the mapping** for existing keys — per-key ordering breaks at the moment of the change, and any
consumer relying on key → partition locality (state stores, local caches) is now wrong. Plan
partition counts up front; if you must grow, create a new topic and migrate. Null-keyed records use the
sticky partitioner (2.4+), which fills a batch per partition before switching — a slow broker can
receive *more* traffic under the old sticky logic (fixed by KIP-794's adaptive partitioning in 3.3).

**Durability.** A write with `acks=all` is acknowledged when all **in-sync replicas** have it. The ISR
is the set of replicas caught up within `replica.lag.time.max.ms` (30 s since 2.5). `min.insync.replicas`
(topic/broker setting, default 1!) is the minimum ISR size for `acks=all` writes to succeed; below it,
producers get `NotEnoughReplicasException`. The standard production triple: **RF=3, min.insync=2,
acks=all** — survives one broker loss with no data loss and no write outage. Traps:
- `acks=all` with `min.insync.replicas=1` gives no more durability than `acks=1` when the ISR has
  shrunk to the leader alone — acknowledged writes live on one disk.
- RF=3 with `min.insync=3` means any single broker restart (rolling upgrade!) blocks all writes.
- RF=2 with min.insync=2 has the same problem; RF=2 with min.insync=1 loses data on leader failure.
- Kafka acknowledges from the **page cache**, not after fsync; durability comes from replication.
  A correlated failure (all replicas in one AZ losing power) can lose acknowledged data. Use rack
  awareness (`broker.rack`) so replicas span AZs.

**Unclean leader election.** If all ISR members are down, `unclean.leader.election.enable=true` lets an
out-of-sync replica become leader — availability restored, **acknowledged messages silently lost** and
offsets reused (consumers may see different data at the same offset). Default `false` since 0.11.
Leaving it false means the partition is offline until an ISR member returns — the explicit choice
between durability and availability.

**Broker health signals.** `UnderReplicatedPartitions` > 0 (the single most important broker metric),
`UnderMinIsrPartitionCount`, `OfflinePartitionsCount`, ISR shrink/expand rate (flapping = a slow
broker, often disk or GC), request handler idle ratio, and produce/fetch request latency percentiles.
**Leader skew**: after a broker restart, leadership stays on the others until a preferred-leader
election runs (`auto.leader.rebalance.enable`, checked every 5 minutes, or run
`kafka-leader-election.sh` manually) — one broker then carries more than its share of traffic.

**Partition reassignment** copies whole partitions; without `--throttle` it saturates the network and
disks and starves live replication — ISR shrinks cluster-wide during a "routine" rebalance. Always
throttle and move in batches (Cruise Control automates this).

**Page cache and lagging consumers.** Tailing consumers read from the page cache (the data was just
written). A consumer replaying days of data reads from disk, evicts the page cache, and suddenly
*tailing* consumers and even producers slow down (disk contention, cache misses). A big replay is a
cluster-wide event: throttle it with client quotas, or run it against followers/a separate cluster.

**Cross-AZ cost.** Consumers fetching from leaders in other AZs pay inter-AZ transfer on every byte.
KIP-392 (`replica.selector.class=RackAwareReplicaSelector` plus `client.rack`) lets consumers fetch
from a same-AZ follower. Producers still write to the leader.

### 10. Kafka storage: retention, segments, compaction and offsets

**Retention is per segment.** A partition is a sequence of segment files; only the active one is
written. Deletion happens by whole segment, only for **closed** segments, when the segment's
**largest timestamp** is older than `retention.ms` (default 7 days) or when the partition exceeds
`retention.bytes` (default unlimited, **per partition**, not per topic). Consequences:
- A low-volume topic with `segment.bytes` 1 GB and `segment.ms` 7 days keeps data far longer than
  `retention.ms` — the "we set retention to 1 day for GDPR but data from 10 days ago is still there"
  finding. Lower `segment.ms` for topics with short retention needs.
- One record with a **future timestamp** (a producer with a broken clock, `CreateTime` default) makes
  its segment "young" until that date, and nothing in it — or after it in time order — is deleted.
  Disk fills slowly. Use `message.timestamp.type=LogAppendTime`, or on newer versions bound accepted
  timestamps (`log.message.timestamp.before.max.ms` / `after.max.ms`, KIP-937).
- `retention.bytes` × partitions × RF is the real disk budget.

**Compaction** (`cleanup.policy=compact`) keeps the latest value per key. Details that bite:
- The **active segment is never compacted**; `min.cleanable.dirty.ratio` (0.5) and
  `min.compaction.lag.ms` delay it further. Compacted topics still hold duplicates for a key — readers
  must handle them.
- **Tombstones** (null value) delete a key, but are retained for `delete.retention.ms` (24 h) so
  consumers can see them. A consumer that bootstraps from a compacted topic slower than 24 h can miss
  a tombstone and keep a deleted entity forever (**resurrection**).
- Records without a key are rejected on compacted topics.
- The **log cleaner thread can die** (an exception on one corrupt segment, or not enough
  `log.cleaner.dedupe.buffer.size`), silently: compacted topics — including `__consumer_offsets` —
  then grow without bound. Monitor `max-dirty-percent` and `time-since-last-run-ms`; check
  `log-cleaner.log`.
- `cleanup.policy=compact,delete` compacts and also deletes by retention — for "latest state, but
  nothing older than 30 days".

**Consumer offsets.** Stored in the compacted `__consumer_offsets` topic. Committed offsets for a group
with **no active members** expire after `offsets.retention.minutes` (7 days since 2.0). A consumer
group that is paused for a long weekend plus a few days comes back with no committed offsets and
applies `auto.offset.reset`: `latest` (the default) **silently skips** everything produced meanwhile;
`earliest` reprocesses the entire retained topic. Same outcome when the topic is deleted and recreated
or offsets fall off the log because retention deleted the segment they pointed to
(`OffsetOutOfRange` → reset). Always set `auto.offset.reset` deliberately and alert on resets.

**Partition count.** More partitions = more parallelism, and more open files, more replication
fetchers, longer leader elections and controller failover (much improved with KRaft), more end-to-end
latency at low throughput. A consumer group can have at most one active consumer per partition —
extra consumers idle. Partition count is effectively permanent for keyed topics (§9).

### 11. Kafka consumer groups, rebalances and lag

**The liveness model.** Two independent checks (`Q15`):
- **Heartbeats** from a background thread every `heartbeat.interval.ms` (3 s); miss them for
  `session.timeout.ms` (45 s since 3.0, previously 10 s) and the coordinator evicts the member. This
  detects dead processes.
- **`max.poll.interval.ms`** (default 300 s) — the maximum time between `poll()` calls. If processing
  one batch of `max.poll.records` (default 500) takes longer, the client itself leaves the group. This
  detects stuck processing.

**The rebalance storm.** Processing 500 records at 1 s each = 500 s > 300 s. The consumer leaves, a
rebalance revokes all partitions from everyone (eager protocol), the same partition is assigned to
another consumer that pulls the same 500 records, which also takes 500 s, and the commit never
happens. Lag grows, the same messages are processed repeatedly (duplicate side effects), and every
rebalance pauses the whole group. Fixes: lower `max.poll.records`; raise `max.poll.interval.ms` to
fit the real worst case; move slow work off the poll thread with `pause()`/`resume()` so `poll()`
continues while partitions are paused; commit per record or per small batch; bound per-record time
with timeouts.

**Other rebalance triggers.** Rolling deploys (each pod restart = two rebalances: leave and join),
autoscaling on lag (scaling changes membership → rebalance → lag grows → more scaling), long GC pauses,
liveness probes killing a pod that is blocked during a rebalance, and a consumer that subscribes by
regex to topics that appear/disappear.

**Mitigations.**
- **Cooperative incremental rebalancing** (`partition.assignment.strategy=CooperativeStickyAssignor`)
  — only moved partitions are revoked; others keep processing. Switching from eager requires a two-step
  rolling bounce (add cooperative alongside the old assignor, then remove the old).
- **Static membership** (`group.instance.id` per pod, e.g. the StatefulSet pod name) — a member that
  restarts within `session.timeout.ms` rejoins with its old assignment and **no rebalance**. Pair with
  a session timeout longer than a pod restart.
- **KIP-848 (new consumer protocol, GA in Kafka 4.0, `group.protocol=consumer`)** — the broker computes
  assignments and rebalances are incremental per member, removing the group-wide sync barrier.
- Don't scale consumers on raw lag; add hysteresis and scale no further than the partition count.

**Diagnosing lag.** `kafka-consumer-groups.sh --describe --group g` shows current offset, log end
offset and lag *per partition*, plus the owning consumer. Read the shape:
- **All partitions lagging evenly** → throughput problem: processing too slow, downstream slow, too
  few consumers (up to the partition count).
- **One partition lagging, others at zero** → a hot key (partition skew), a poison message the
  consumer keeps retrying, or a stuck consumer instance (check which member owns it; thread dump).
- **Lag sawtooth** → batch commits or periodic GC/rebalances.
- **Lag with no owner** → partitions unassigned (group rebalancing constantly, or more partitions than
  the assignor could hand out due to errors).
- **Lag growing while consumers are idle** on a `read_committed` consumer → a hanging transaction
  holds the last stable offset (§13).
Express lag in **time** (Burrow, kafka-lag-exporter) — 100,000 messages is seconds on one topic and
days on another.

**Hot partitions.** Key skew (one tenant, one big merchant, `null`/"unknown" keys defaulting to a
constant, a celebrity user). Diagnosis: per-partition bytes-in on the broker, per-partition lag.
Options: a better key (sub-entity instead of tenant), **salting** the hot key (`merchant42#0..7`)
when ordering is only needed per sub-key, splitting hot tenants to a dedicated topic, or accepting
the skew and making per-message processing faster. Ordering requirements decide which is allowed.

**Parallelism beyond partitions.** One consumer per partition caps parallelism; processing in parallel
*within* a partition by key (Confluent Parallel Consumer, or a keyed worker pool) keeps per-key order
while using many threads — but offsets must then be committed at the **lowest contiguous completed
offset**, not the highest, or a crash loses the gap.

### 12. Offsets, poison pills, retries, DLQs and replay

**Commit semantics.** The committed offset is the **next** offset to read (last processed + 1).
`enable.auto.commit=true` (default) commits the offsets returned by the *previous* `poll()`, every
`auto.commit.interval.ms` (5 s), during `poll()`. With synchronous processing inside the poll loop that
is at-least-once. Hand records to a thread pool and return to `poll()`, and auto-commit commits records
still being processed — a crash **loses** them (at-most-once). Commit manually after processing when
work is asynchronous. Committing after every message synchronously caps throughput at the commit
round trip; commit asynchronously per batch and synchronously on revoke/shutdown
(`onPartitionsRevoked`).

**Poison pills.** A message that always fails — undeserialisable bytes, a schema the consumer cannot
read, a payload that triggers a bug. A naive consumer retries it forever and the partition stops
(every other message behind it is blocked: one bad record = one partition at 100% lag). Deserialisation
errors are especially nasty because they happen inside `poll()`, before your handler (Spring Kafka's
`ErrorHandlingDeserializer` wraps them so they can be routed to a DLT). Emergency fix: seek past the
offset (`kafka-consumer-groups.sh --reset-offsets --shift-by 1 --topic t:partition`) after copying the
record out for analysis.

**Retry design.** Classify errors first: **transient** (timeout, 503, lock conflict) → retry with
backoff and jitter; **permanent** (validation, 4xx, deserialisation) → DLQ immediately; retrying them
wastes time and blocks partitions. Patterns:
- **Blocking in-place retry** — a few quick retries in the consumer. Keeps ordering, blocks the
  partition while retrying. Must fit inside `max.poll.interval.ms`.
- **Non-blocking retry topics** — `orders.retry.1m`, `orders.retry.10m`, `orders.dlt` (Uber's
  pattern, Spring `@RetryableTopic`). The partition keeps flowing, **ordering per key is lost**. The
  retry consumer must wait until the message's due time — pausing the partition, not sleeping in the
  poll loop.
- **Ordered retry with key parking** — once a key has a message in retry, subsequent messages for that
  key are also diverted behind it (tracked in a store) so a key's order is preserved while other keys
  flow. More complex, necessary for state-changing events like `OrderCreated` → `OrderCancelled`.
- **Circuit breaker on the consumer** — when *all* messages fail because a dependency is down, sending
  everything to the DLQ is wrong. Pause consumption (stop polling partitions via `pause()`) until the
  dependency recovers; retry topics are for individual failures, not outages.

**DLQ hygiene.** Include original topic, partition, offset, key, timestamp, exception class and
message, attempt count and consumer version in headers. Keep the original bytes (not a re-serialised
object). Alert on DLQ **arrival rate**, not just size. Build a **redrive** tool that republishes to the
original topic (or a replay topic) with idempotency guaranteed downstream. A DLQ nobody reads is a
data-loss mechanism with extra steps. Set DLQ retention longer than the source's (§15 for the SQS
trap).

**Framework handlers.** Kafka Streams' `default.deserialization.exception.handler` is
`LogAndFailExceptionHandler` — one bad record crash-loops the whole application (every restart hits it
again); `LogAndContinueExceptionHandler` or a custom handler that writes to a DLQ keeps it running.
Spring Kafka's `DefaultErrorHandler` retries 9 times with no backoff by default before recovering.

**Replay safely.** Reset a group's offsets (`--reset-offsets --to-datetime 2025-03-01T10:00:00.000
--execute`, with the group stopped) or start a new consumer group. Before replaying, ask:
- Are side effects idempotent? Replaying `PaymentCaptured` into a consumer that sends emails sends a
  week of emails. Run replays in a mode that suppresses external side effects, or into a separate
  consumer group whose sink is idempotent.
- Is the logic the same? Replaying old events through new code can produce different results than
  history did — sometimes the point, sometimes a disaster.
- Is the downstream sized for it? A replay at full speed is a load test on everything downstream
  (throttle with quotas or a rate limiter), and on the brokers' disks (§9).
- **Cross-cluster failover** (MirrorMaker 2): offsets differ between source and target clusters.
  MM2 emits checkpoints translating committed offsets (`sync.group.offsets.enabled`); translation is
  approximate and lags, so failing consumers over means some duplicates by design — idempotence is not
  optional. Replication is asynchronous; the tail is lost on failover like any async replica.
- Is the data still there? Retention may have removed it; the source of truth may need to be a DB
  export or the data lake instead.

### 13. Transactions, exactly-once, the outbox and CDC

**Kafka transactions.** A transactional producer (`transactional.id`) writes to several partitions and
commits consumer offsets (`sendOffsetsToTransaction`) atomically. Consumers with
`isolation.level=read_committed` see only committed data. This gives exactly-once for
**read-from-Kafka, write-to-Kafka** pipelines (Kafka Streams `processing.guarantee=exactly_once_v2`).
Limits that interviewers probe:
- Nothing outside Kafka is covered — a DB write or HTTP call inside the "transaction" can happen twice.
- **Zombie fencing**: a restarted producer with the same `transactional.id` bumps the epoch and fences
  the old one (`ProducerFencedException`). Two live instances configured with the same
  `transactional.id` fence each other in a loop.
- **Last stable offset (LSO).** A `read_committed` consumer cannot read past the first open
  transaction. A producer that begins a transaction and hangs (or a broker bug leaving a "hanging
  transaction") holds the LSO; consumers show lag growing while receiving nothing, until
  `transaction.timeout.ms` (60 s default, up to the broker's `transaction.max.timeout.ms` of 15 min)
  aborts it — or, for true hanging transactions, until an operator runs `kafka-transactions.sh
  --find-hanging` and `abort` (KIP-664).
- Transactions add latency (commit markers, coordinator round trips); commit intervals of 100 ms+ are
  common, so read-committed consumers see data in bursts.

**The dual-write problem and the outbox** (`Q18`, `M15`). Writing to the DB and publishing to Kafka as
two steps loses one or the other on a crash. The outbox writes the event to an `outbox` table in the
same DB transaction; a relay (poller or CDC) publishes it. Pitfalls: the relay publishes
at-least-once (consumers dedupe by event ID); a polling relay ordered by an auto-increment ID can skip
rows whose transactions committed out of ID order (ID 101 commits before 100; the poller advances past
100) — poll with a safety lag or use CDC; the outbox table bloats without cleanup (partition it and
drop partitions).

**CDC (Debezium) operational traps.**
- **Postgres replication slots retain WAL until consumed.** If the connector is down or stuck, WAL
  accumulates on the **primary** until its disk fills and the primary stops — a CDC outage takes down
  the source database. Monitor `pg_replication_slots` (`pg_wal_lsn_diff(pg_current_wal_lsn(),
  confirmed_flush_lsn)`), set `max_slot_wal_keep_size` (PG 13+) as a circuit breaker, and drop
  abandoned slots.
- **Quiet tables, busy database.** A connector watching a low-traffic table does not advance its slot
  while other tables churn WAL — WAL grows even though the connector is healthy. Debezium's
  `heartbeat.interval.ms` plus `heartbeat.action.query` (write to a heartbeat table) fixes it.
- **Snapshots** on first start (or after offset loss) re-emit the whole table — a flood of `r`
  (read) events that downstream consumers must treat as upserts.
- MySQL binlog retention shorter than a connector outage → the connector cannot resume and must
  re-snapshot.
- Schema changes (DDL) flow through CDC and can break downstream consumers (§18).
- Events reflect the physical table, not a domain contract — publishing raw CDC couples every consumer
  to your schema. Outbox + CDC (Debezium outbox event router) gives both reliability and a contract.

### 14. RabbitMQ in production

**Prefetch.** `basic.qos(prefetch_count)` limits unacked messages per consumer/channel. **The default
is unlimited** (0) with manual acks — the broker pushes the entire queue into the first consumer's
memory. One consumer gets 2M messages and OOMs; the others idle; after the crash all 2M are redelivered
— to the next consumer. Set prefetch deliberately: roughly (processing time / network RTT) × a small
factor; 10–300 is typical; 1 for long jobs so work is fairly distributed (`Q11`).

**Acks and redelivery loops.** `autoAck=true` is at-most-once (messages are gone when delivered).
`basic.nack(requeue=true)` or `basic.reject(requeue=true)` puts the message back at (near) the **head**
of the queue — a failing message is redelivered immediately, forever, at 100% CPU, and can block
others (the requeue hot loop). Classic queues have no delivery limit; **quorum queues** have
`delivery-limit` (default 20 since 4.0) after which the message is dead-lettered or dropped. Track the
`x-death` header or `redelivered` flag; nack with `requeue=false` to a DLX for a delayed retry.

**Delayed retries.** TTL + dead-letter exchange: publish to a "wait" queue with a TTL and no consumers;
on expiry it dead-letters back to the work exchange. Trap: **per-message TTL expires only at the head of
the queue** — a 30-minute message at the head blocks a 5-second message behind it. Use one wait queue
per delay tier (queue-level `x-message-ttl`), or the delayed-message exchange plugin (which has its own
limits: stored on one node, not replicated, poor at millions of delayed messages).

**Consumer timeout.** Since 3.8.15, `consumer_timeout` (default 30 min) closes a channel whose delivery
has not been acked in time, with `PRECONDITION_FAILED`, and requeues its messages. Long-running jobs
(and Celery ETA tasks, §16) hit it — a job runs for 35 minutes, the channel dies, the job is redelivered
and runs again, forever.

**Flow control and alarms.** When memory passes `vm_memory_high_watermark` (0.4 of RAM historically,
0.6 in recent releases) or free disk falls below `disk_free_limit` (50 MB default — far too low for
production), the broker **blocks all publishing connections** on that node (cluster-wide for memory
alarms). Consumers keep working. The trap: an application that **publishes and consumes on the same
connection** — the connection is blocked, so the consumer's acks cannot be sent, so the queue cannot
drain, so the alarm never clears: a deadlock. Use separate connections for publishing and consuming.
Handle `connection.blocked` notifications so publishers fail fast rather than hang.

**Throughput shape.** Each queue is a single Erlang process — roughly one core — so one hot queue caps
at tens of thousands of msg/s regardless of cluster size. Shard (consistent-hash exchange, sharding
plugin, multiple queues) to scale. Long queues cost memory and slow the broker; RabbitMQ is happiest
with short queues. `x-max-length` with `overflow=drop-head` (default — silently drops oldest!) or
`reject-publish` (publisher confirms get nacked) bounds them.

**Durability checklist.** Durable queue + persistent message (`delivery_mode=2`) + publisher confirms;
plus `mandatory=true` (or an alternate exchange) because a message published to an exchange with no
matching binding is **silently dropped** — the classic "we lost messages after renaming a routing key".
Classic mirrored queues are deprecated and removed in 4.0; use **quorum queues** (Raft, need an odd
number of nodes, keep messages in memory index, prefer short queues) or **streams** for replayable
logs. Network partitions: `cluster_partition_handling = pause_minority` for consistency; `autoheal`
chooses availability and discards the losing side's changes.

**Connection and channel churn.** Opening a connection per publish (common in PHP and serverless) costs
TCP + AMQP handshakes and can exhaust Erlang processes and file descriptors; use long-lived
connections, a channel per thread, and a pool.

### 15. SQS and cloud queues

**Visibility timeout.** On receive, a message becomes invisible for the visibility timeout (default
**30 s**, max 12 h). If the consumer does not delete it in time — because processing took longer —
it becomes visible again and **another consumer processes it concurrently**. Duplicates rise exactly
when the system is slow. Fixes: set the timeout above the p99.9 processing time, and for variable-length
work run a **heartbeat** that calls `ChangeMessageVisibility` to extend it while working. For Lambda,
AWS recommends a queue visibility timeout of at least **6× the function timeout** (retries and
throttling happen inside that window).

**Standard vs FIFO.** Standard: at-least-once, occasional duplicates even without failures,
best-effort ordering, near-unlimited throughput. FIFO: ordering per `MessageGroupId`, 5-minute
deduplication window (`MessageDeduplicationId` or content-based), throughput historically 300
API calls/s per action (3,000 messages/s with batches of 10), much higher in high-throughput mode
(scaling per group-ID partitions). **Within one message group only one batch is in flight** — so one
slow or failing message blocks its whole group until it succeeds or reaches `maxReceiveCount` and goes to
the DLQ. A single group ID for everything serialises the whole queue; too many groups loses the ordering
you wanted. Choose the group ID = the entity whose order matters.

**Redrive and DLQ traps.** `maxReceiveCount` in the redrive policy moves a message to the DLQ after N
receives. Counting is per *receive*, so a consumer that crashes (or has too short a visibility
timeout) burns receives without ever trying — a healthy message lands in the DLQ. For standard queues,
**a message's retention clock keeps its original enqueue timestamp** when moved to the DLQ: with a
source retention of 4 days and a DLQ retention of 4 days, a message that took 3 days to fail has one
day left in the DLQ. Set DLQ retention to the 14-day maximum. Use the redrive API (`StartMessageMoveTask`)
to move messages back.

**Costs and polling.** `WaitTimeSeconds=20` (long polling) cuts empty receives — short polling from
100 idle workers is thousands of billed empty calls per minute. Batch sends/receives/deletes (10 per
call). Retention default 4 days, max 14.

**Lambda + SQS.** Lambda polls with a batch; if the function throws, **the whole batch** returns to the
queue and is retried (successful items are reprocessed) unless you enable `ReportBatchItemFailures` and
return the failed IDs. Lambda scales pollers up rapidly — a backlog can produce hundreds of concurrent
invocations that exhaust a database's connections; cap with the event source's **maximum
concurrency** setting (not only reserved concurrency, which causes throttles that burn receive counts).

**SNS/EventBridge fan-out.** SNS → SQS subscriptions give durable fan-out; filter policies reduce
traffic; raw message delivery avoids the SNS envelope. SNS FIFO → SQS FIFO for ordered fan-out.
EventBridge delivers at-least-once with retries up to 24 h and its own DLQ per target.

### 16. Job queues: BullMQ, Celery, Sidekiq, scheduling and fairness

**BullMQ (Redis).** A worker holds a lock on an active job (`lockDuration` 30 s) renewed periodically
by a timer on the event loop. **CPU-bound synchronous work blocks the Node event loop**, the lock is not
renewed, the job is declared **stalled**, moved back to wait, and processed by another worker — twice;
after `maxStalledCount` (default 1) stalls it fails with "job stalled more than allowable limit". Fixes:
sandboxed processors (separate process) or worker threads for CPU work, break work into chunks that
yield. Redis must run `maxmemory-policy noeviction` — eviction silently deletes job keys and locks.
`removeOnComplete`/`removeOnFail` must be set or completed jobs accumulate in Redis forever. Rate
limiting and concurrency are per worker unless using queue-level limiters.

**Celery.** Defaults bite:
- `task_acks_late=False` — the task is acked **when received**, so a worker crash, OOM kill or deploy
  loses it. Set `acks_late=True` (plus `task_reject_on_worker_lost=True`) for at-least-once, and make
  tasks idempotent.
- `worker_prefetch_multiplier=4` — each worker process reserves 4 tasks; short tasks wait behind a
  long one on a busy process while other processes idle. Use `prefetch_multiplier=1` (and `-O fair`
  on older versions) for mixed task lengths.
- **Redis broker `visibility_timeout`** (default 1 h) — with `acks_late`, a task not acked within an
  hour is redelivered to another worker. Tasks scheduled with `eta`/`countdown` further than the
  visibility timeout are **redelivered repeatedly** and execute multiple times. Raise the timeout
  above the longest ETA and the longest task, or use a real scheduler for far-future work.
- ETA tasks are held in worker memory (and count against RabbitMQ's `consumer_timeout`, §14).
- `celery beat` must run exactly once; two beats (two replicas of the scheduler deployment) double every
  periodic task. Use a single-replica deployment with a lock, or a DB-backed scheduler with leader
  election.
- Memory growth in long-lived workers → `worker_max_tasks_per_child` / `worker_max_memory_per_child`.

**Sidekiq.** Basic fetch uses `BRPOP` — a job popped by a process that then dies is lost; reliable fetch
(Pro/Enterprise `super_fetch`) moves jobs to a per-process working list. Retries default to 25 with
exponential backoff over ~20 days — a buggy job keeps retrying for weeks.

**Fairness and multi-tenancy** (`Q21`). A single FIFO queue lets one tenant's 2M-job bulk import delay
every other tenant for hours. Options: per-tenant queues with weighted round-robin polling;
per-tenant concurrency limits (a semaphore in Redis); separate queues by priority/lane (interactive vs
bulk) with reserved workers per lane; **shuffle sharding** (each tenant maps to a random subset of
queues/workers so one bad tenant affects only tenants sharing its exact subset). Plain priority queues
starve the low-priority lane under sustained load — use weighted scheduling, not strict priority.

**Scheduling.** Cron in every replica runs every job N times; missed runs during downtime are skipped
silently or all run at once on recovery (catch-up storms) depending on the scheduler. Jobs scheduled "at
midnight" for every tenant create a load spike at 00:00 UTC — spread them (hash the tenant ID into a
start offset).

**Long jobs.** Break into idempotent chunks with checkpoints; a 4-hour job that dies at 3h50 and
restarts from zero is a design flaw. Make progress visible (heartbeat), and make timeouts at each layer
(job, lock, visibility, consumer) consistent — the shortest one wins, silently.

### 17. Stream processing: time, watermarks, state and late data

**Event time vs processing time.** Aggregations by processing time are wrong whenever there is lag or
replay: a consumer that is 2 hours behind attributes 2-hour-old events to "now". Use **event time**
from the payload (`Q17`).

**Watermarks.** A watermark says "no more events older than T are expected". Common strategy: bounded
out-of-orderness — watermark = max event time seen − allowed delay. Windows fire when the watermark
passes their end; events arriving after that are **late**: dropped, sent to a side output, or used to
update results within an *allowed lateness* / *grace period* (Kafka Streams `grace()`).
Failure modes:
- **Idle partitions hold the watermark back.** The operator's watermark is the *minimum* across inputs;
  one partition with no traffic (a quiet region at night) stops all windows from firing — output stops
  even though data flows. Flink `withIdleness(Duration)` marks idle sources.
- **A future timestamp advances the watermark.** One device with its clock set to 2031 pushes max
  event time forward; the watermark jumps and every subsequent correct event is "late" and dropped.
  Clamp timestamps (reject or cap events too far in the future relative to ingestion time), or
  compute watermarks per source.
- **Replay speed.** When replaying, event time races ahead of wall-clock; timers and windows fire in
  bursts, and processing-time timers (session timeouts) misbehave.

**Kafka Streams specifics.** State stores are backed by changelog topics; a rebalance that moves a
task to a new instance must **restore the state** from the changelog — minutes to hours for a large
store, during which that task is not processing. `num.standby.replicas` keeps warm copies elsewhere;
static membership and cooperative rebalancing reduce moves; KIP-441 warm-up tasks move state before
switching. Repartition topics appear when you re-key (`selectKey`/`groupBy`) — extra traffic and a
source of co-partitioning errors in joins (both sides must have the same partition count and
partitioner). RocksDB memory lives off-heap and is not bounded by `-Xmx` — containers get OOM-killed
without a `RocksDBConfigSetter` bounding block cache and memtables.

**Flink specifics.** Checkpoints with barrier **alignment**: under backpressure, barriers queue behind
data and checkpoints time out, so the job cannot checkpoint precisely when it most needs to — unaligned
checkpoints (1.11+) let barriers overtake data. Savepoints for upgrades require stable operator
`uid`s; changing the job graph without them discards state. Exactly-once sinks to Kafka use
transactions committed on checkpoint completion — the output is visible to `read_committed` readers
only at checkpoint intervals, and a checkpoint interval longer than `transaction.timeout.ms` loses
data on recovery.

**Joins and enrichment.** Stream-table joins read the table's latest state *at processing time*; a
replay joins old events with **today's** reference data (temporal join semantics matter). Enrichment
by calling a DB per event does not scale and is non-deterministic on replay — materialise the table
into the processor via CDC.

**Deduplication in streams.** Dedup needs state bounded by a time window (keep IDs for 24 h); unbounded
dedup state grows forever.

### 18. Schemas and evolution in production

**Compatibility modes** (Confluent Schema Registry, default `BACKWARD`) (`Q19`): BACKWARD = new schema
can read data written with the previous one → **upgrade consumers first**; FORWARD = old schema can
read new data → upgrade producers first; FULL = both. `*_TRANSITIVE` checks against all versions, not
just the last — needed when consumers can replay old data from a long-retention topic.

**Traps.**
- Producers with `auto.register.schemas=true` in production register whatever the code has — a
  developer's local change or a new producer can register an incompatible subject version or a new
  subject naming strategy. Register schemas through CI; disable auto-registration in production.
- Avro: adding an **enum symbol** breaks old readers unless the reader enum has a `default`. Removing
  a field without a default breaks BACKWARD. Changing a field type (int → long is a promotion; long →
  int is not).
- Protobuf: never reuse or renumber field tags; `reserved` removed ones. JSON without a schema: renames
  silently become "field missing" nulls.
- Poison by schema: a consumer on an old deserialiser that encounters a newer, incompatible schema is a
  poison pill at the deserialiser level (§12) — every partition it touches stops.
- Deleting (soft or hard) a schema version that old messages still reference makes those messages
  unreadable on replay.
- CDC schemas follow the table — a column rename in a migration is a breaking event-schema change.
  Gate DDL on downstream compatibility, or publish domain events via the outbox instead.

### 19. The diagnostic toolkit

**Redis.** `INFO` (sections: `memory`, `stats`, `persistence`, `replication`, `clients`, `keyspace`,
`commandstats`), `SLOWLOG GET 50`, `LATENCY DOCTOR`, `CLIENT LIST` (look at `omem`, `qbuf`, `idle`,
`cmd`), `MEMORY USAGE key`, `MEMORY DOCTOR`, `OBJECT ENCODING|FREQ|IDLETIME key`, `redis-cli
--bigkeys|--memkeys|--hotkeys|--latency|--stat`, `SCAN ... TYPE`, `CLUSTER INFO`, `CLUSTER NODES`,
`CLUSTER COUNTKEYSINSLOT`. Key metrics: `evicted_keys`, `expired_keys`, `keyspace_hits/misses`,
`rejected_connections`, `connected_clients`, `blocked_clients`, `latest_fork_usec`,
`mem_fragmentation_ratio`, `master_link_status`, `sync_full`, `sync_partial_err`,
`master_repl_offset` vs replica offsets.

**Kafka.** `kafka-consumer-groups.sh --describe --group g` (lag, members; `--members --verbose` for
assignments), `--reset-offsets` (with `--dry-run` first), `kafka-topics.sh --describe
--under-replicated-partitions` / `--under-min-isr-partitions`, `kafka-configs.sh --describe
--entity-type topics`, `kafka-log-dirs.sh` (partition sizes per broker), `kafka-dump-log.sh`
(inspect segment contents and timestamps), `kafka-transactions.sh`, `kafka-leader-election.sh`,
`kafka-reassign-partitions.sh --throttle`. Metrics: `UnderReplicatedPartitions`,
`UnderMinIsrPartitionCount`, `IsrShrinksPerSec`, `RequestHandlerAvgIdlePercent`, produce/fetch total
time percentiles, consumer `records-lag-max`, `commit-latency`, `rebalance-rate`,
`last-poll-seconds-ago`, producer `record-error-rate`, `buffer-available-bytes`, `request-latency`.

**RabbitMQ.** Management UI / `rabbitmq-diagnostics` (`status`, `memory_breakdown`, `alarms`),
`rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers`, per-queue
`redeliver` rate, `connection.blocked` counts.

**SQS.** CloudWatch `ApproximateAgeOfOldestMessage` (the alert metric), `ApproximateNumberOfMessagesVisible`,
`ApproximateNumberOfMessagesNotVisible` (in flight — high means slow or crashing consumers),
`NumberOfEmptyReceives`, DLQ `ApproximateNumberOfMessagesVisible`.

**General method.** For any "the queue is backing up / the cache is slow" question: (1) is it
everywhere or one partition/shard/key? (2) what changed — deploy, traffic, data shape, a dependency?
(3) is the bottleneck the broker, the consumer, or a downstream? (4) stop the bleeding (shed, pause,
skip, serve stale) before the root-cause fix, and (5) make the fix leave a metric or an alert behind.

---

## Questions

### Level 1 — Everyday cache incidents

The caching failures that come up in nearly every senior backend loop: outages, stale data and hit-rate collapses.

1. Our Redis primary went down for four minutes last Tuesday. Hit rate had been 97%, and within
   thirty seconds Postgres CPU was at 100% and the whole site returned 503s — well after Redis came back.
   What would you change so the next Redis outage is a non-event?
   > **Direction:** Do the miss-rate arithmetic (33× DB load) and design for it: short cache timeouts, a circuit breaker that fails fast to the source, a local L1 for the hottest keys, and load shedding at the DB rather than "make Redis more available" (§5, §1; `Q04`, `Q01`).

2. The product page p99 jumps from 40 ms to 900 ms for about ten seconds at the top of every hour, and
   the database shows a burst of identical queries at the same moment. Nothing is scheduled on the hour.
   What is going on?
   > **Direction:** A warm-up or batch wrote many keys at once with the same `EX 3600`, so they all expire together — synchronised expiry; add TTL jitter and coalesce misses (§5, §2; `Q04`).

3. Every deploy of the catalogue service triples database load for roughly forty minutes, then it
   settles. The deploy does not touch the database schema. Where do you look?
   > **Direction:** The deploy changed the cache key format or serialisation (a class rename, a new field in the key), which is a full flush in disguise; dual-read old/new keys or version the payload and roll out gradually (§7, §6; `Q03`).

4. Users occasionally report that after editing their profile they see the old version for up to ten
   minutes — the TTL. We delete the key on every update. It is rare and happens more under load. Explain it.
   > **Direction:** The cache-aside race: a reader that loaded the old row before the write sets it after the delete; fix with leases or version-checked sets, with the TTL as backstop (§6; `Q02`, `Q03`).

5. Right after sign-up, some users get "account not found" from our API for a few minutes, then it
   works. Signup writes to the primary and returns 201 immediately. What have we probably built?
   > **Direction:** A negative cache entry (created by an earlier lookup, e.g. an email-availability check or the client polling) that the create path never deletes; creation must invalidate negative entries and negative TTLs stay short (§6; `Q04`).

6. The "top deals" homepage widget is backed by a 4-second aggregation query cached for 5 minutes.
   Every time it expires, 300 pods run the query simultaneously and Postgres stalls. What is your fix,
   and why not just a longer TTL?
   > **Direction:** Stale-while-revalidate with a soft expiry and a single refresher guarded by a lock (or XFetch), plus in-process singleflight; a longer TTL only makes the stampede rarer, not smaller (§5; `Q04`).

7. Redis memory grows steadily for weeks until writes start failing with OOM errors. We set
   `maxmemory-policy volatile-lru` specifically so this would not happen. Why did it?
   > **Direction:** `volatile-*` evicts only keys with a TTL; a code path writing without `EX` fills memory with unevictable keys and Redis behaves like `noeviction` — compare `keys` vs `expires` in `INFO keyspace` (§2; `Q06`).

8. Our order-history endpoint has p99 of 350 ms and traces blame Redis. Redis CPU is at 15%, the slowlog
   is empty, and each request makes about 300 `GET`s. What is really slow?
   > **Direction:** Round trips, not Redis — 300 sequential calls × RTT; use `MGET`/pipelining in bounded chunks, and note the slowlog only measures execution time (§1; `Q05`).

9. Cache hit rate on the search API dropped from 94% to 38% after we shipped a new filter feature.
   Traffic did not change and no one touched the TTLs. What do you check first?
   > **Direction:** The cache key now contains something high-cardinality — unsorted query params, a serialised filter object with unstable field order, a timestamp; normalise the key (§5; `Q03`).

10. A customer on tenant A saw tenant B's dashboard numbers for a few seconds. It is a cached endpoint.
    Walk me through containment and the root cause.
    > **Direction:** Cache poisoning by wrong key scope (tenant/user missing from the key); contain with a namespace version bump or kill switch, then fix the key and treat it as a data-leak incident (§6; `Q03`).

11. The recommendations service timed out during a deploy, our API cached its response — an empty list
    — and the product carousel was empty for an hour. What rule was broken?
    > **Direction:** Only cache complete, successful results; errors and partial results should never enter the cache (or only for seconds), and stale-if-error should keep serving the previous good value (§6, §5; `Q04`).

12. We put Redis in front of a partner API. Median latency improved from 120 ms to 4 ms, but p99 got
    worse — from 400 ms to 1.3 s. How is that possible?
    > **Direction:** The miss path now costs cache timeout + source, and client pool starvation or long Redis timeouts/retries add to the tail; give the cache a tens-of-ms timeout with no retries and a breaker, so a slow cache degrades to "no cache" (§4, §5).

13. A crawler is requesting random, non-existent product IDs at 3,000 req/s. Each one misses Redis and
    hits the database. Rate limiting it is on the list — what do you do in the cache layer?
    > **Direction:** Cache penetration: short-TTL negative caching of "not found" plus a Bloom filter of valid IDs to reject impossible keys without touching Redis or the DB (§5; `Q04`).

14. During a flash sale, one node of our six-node Redis Cluster sits at 100% CPU and the others at 10%.
    Adding nodes did nothing. Why, and what actually helps?
    > **Direction:** A hot key lives on one slot and cannot be sharded; find it (client sampling, `--hotkeys` under LFU) and absorb it with a 1–5 s in-process L1 cache or replicated key copies (§5, §2; `Q04`).

15. Each API pod keeps an in-memory cache of prices for 30 minutes. After a price change, some pods show
    the new price and some the old one for up to half an hour, and support is confused. What would you do?
    > **Direction:** Per-pod local caches diverge; bound their TTL to what the business tolerates (seconds), and if you add invalidation over Pub/Sub treat it as lossy and keep the TTL as the guarantee (§7; `Q09`).

16. An admin "clear product cache" button runs `KEYS product:*` then deletes the results. Last time
    someone clicked it, Redis stopped responding for eight seconds. How do you redesign cache clearing?
    > **Direction:** Never enumerate keys in the request path; use a namespace version in the key prefix (bump to invalidate logically) or tag sets, and if enumeration is truly needed use `SCAN` + `UNLINK` in a throttled background job (§6, §1).

17. After a frontend release, some users got a blank page for an hour because the new JS bundle
    returned 404 — even though the file was on the origin within a minute of the deploy. What happened?
    > **Direction:** HTML went live before the hashed assets existed, the CDN cached the 404 with a long default TTL; deploy assets first, never cache 404s long, and purge by surrogate key (§7; `Q09`).

18. A logged-in user's account page was served to other users from the CDN. The endpoint had
    `Cache-Control: public, max-age=60` because someone wanted it fast. What went wrong beyond that header?
    > **Direction:** Shared caches must not store responses with `Set-Cookie` or auth-dependent content; use `private`/`no-store` for personalised pages and cache only the shared fragments (§6, §7; `Q09`).

19. To stop OOM errors we switched Redis to `allkeys-lru`. Now users are randomly logged out and some
    rate-limit counters reset. Why, and what is the right layout?
    > **Direction:** Sessions and counters share an instance with cache data, and `allkeys-lru` evicts them; separate instances (or at least eviction domains) — `noeviction` for durable data, `allkeys-lfu` for the cache (§2; `Q06`).

20. After a Redis maintenance restart, the app logged thousands of `NOSCRIPT` errors and our rate
    limiter failed open for several minutes. What happened?
    > **Direction:** The script cache is not persisted, so `EVALSHA` fails after restart/failover; clients must fall back to `EVAL` (or load scripts on connect), or use Redis 7 functions which persist (§1; `Q08`).

### Level 2 — Everyday queue and consumer incidents

The messaging problems every team with a broker has had: duplicates, lag, stuck consumers and backlogs.

1. Customers occasionally receive two identical order-confirmation emails, a few seconds apart. The
   consumer reads from Kafka and calls our email provider. There are no errors in the logs. Explain and fix.
   > **Direction:** At-least-once delivery (a rebalance or crash between side effect and commit); make the consumer idempotent with a processed-message record keyed by event ID and an idempotency key to the provider (§8, §12; `Q10`, `M16`).

2. Consumer lag on our `orders` topic is growing on partition 7 only; the other 23 partitions are at zero.
   The consumer group has 24 members. What are the candidate causes and how do you tell them apart?
   > **Direction:** Hot key skew, a poison message being retried, or a stuck member; check per-partition bytes-in, whether the committed offset moves at all, and the owning consumer's thread dump (§11, §12; `Q15`).

3. One consumer instance logs the same deserialisation error thousands of times a minute and its
   partition has not advanced in two hours. What is your immediate action and your lasting fix?
   > **Direction:** A poison pill failing inside `poll()`; copy the record out and seek past the offset now, then wrap the deserialiser (e.g. `ErrorHandlingDeserializer`) so bad bytes go to a DLT instead of blocking (§12, §18).

4. We scaled the consumer deployment from 12 to 30 pods during a backlog and throughput did not change.
   The topic has 12 partitions. What now?
   > **Direction:** A group has at most one active consumer per partition — 18 pods idle; process in parallel within partitions by key (committing contiguous offsets), or plan a new topic with more partitions (§11, §10; `Q14`).

5. Our payment provider was down for two hours. Our retry topics did their job, retried each message
   three times, and 180,000 payments are now in the DLQ. How should this have worked, and how do you
   recover now?
   > **Direction:** Retry topics handle individual failures, not outages — a consumer circuit breaker should have paused partitions; recover with a throttled redrive to the original topic relying on idempotency keys (§12; `Q16`).

6. Our alert on "queue depth > 10,000" fires every morning at peak and is ignored. Last week a real
   incident — 200 messages stuck for three hours — paged nobody. Redesign the alert.
   > **Direction:** Alert on age of the oldest message / lag in time, which maps to user impact, not on depth, which maps to traffic (§8, §19; `Q15`).

7. The inventory service was down for three hours. When it came back, the queue drained at full speed
   and knocked its database over again within minutes. How do you bring a system back from a big backlog?
   > **Direction:** A draining backlog is a load spike; throttle consumer concurrency to what the dependency survives, and drop or deprioritise messages whose age makes them worthless (message TTL / age check) (§8; `Q16`).

8. The consumer logs show `CommitFailedException: ... the group has already rebalanced` and we see the
   same batch processed again and again. Each record calls an API that takes about 800 ms. Diagnose it.
   > **Direction:** 500 records × 800 ms exceeds `max.poll.interval.ms` (300 s), so the consumer leaves the group and the batch is re-handed out; shrink `max.poll.records`, pause/resume, or process asynchronously (§11; `Q15`).

9. On SQS, a small percentage of video-transcode jobs are processed twice — sometimes by two workers at
   the same time. The jobs take between 10 seconds and 4 minutes. What is happening?
   > **Direction:** The visibility timeout (30 s default) expires on long jobs; heartbeat `ChangeMessageVisibility` while working and make the job idempotent (§15; `Q20`).

10. Our RabbitMQ consumer pod is OOM-killed whenever it starts with a large backlog, and then the next
    pod dies the same way. Small backlogs are fine. What is the configuration bug?
    > **Direction:** Prefetch is unlimited (the default with manual acks), so the broker pushes the whole backlog into one consumer's memory and redelivers it to the next; set a bounded `basic.qos` prefetch (§14; `Q11`).

11. We lose a handful of Celery tasks on every deploy — they are simply never executed. No errors.
    What default is responsible?
    > **Direction:** `task_acks_late=False` acks on receipt, so tasks reserved by a worker that is killed are gone; enable `acks_late` with `task_reject_on_worker_lost` and idempotent tasks (§16; `F13`).

12. To speed up a Kafka consumer we moved processing onto a thread pool and kept the defaults. After a
    pod crash, some orders were never processed at all. Why?
    > **Direction:** Auto-commit commits what `poll()` returned, not what finished, so handing records to a thread pool turns at-least-once into at-most-once; commit manually the lowest contiguous completed offset (§12, §11; `Q15`).

13. We autoscale consumers on lag. When a downstream DB slows down, the autoscaler adds pods and lag gets
    worse, not better. Explain the dynamics.
    > **Direction:** Each scale event rebalances the group (processing pauses) and more consumers push more load onto the already-slow dependency; Little's law says capacity is bound by the dependency — cap concurrency, scale on time lag with hysteresis (§8, §11).

14. After we introduced non-blocking retry topics, some orders show `Cancelled` then flip back to
    `Created` in our read model. Why, and what is the fix without dropping retries?
    > **Direction:** Retry topics break per-key ordering — a retried `OrderCreated` overwrote the newer `OrderCancelled`; park the key (divert later events behind the retrying one) or apply version checks in the consumer (§12, §8; `Q14`, `Q16`).

15. Every rolling deploy of our 40-pod consumer group causes processing to pause several times and lag
    to spike for about ten minutes. How do you make deploys invisible?
    > **Direction:** Each restart triggers eager rebalances; use static membership (`group.instance.id`) with a session timeout longer than a pod restart, and the cooperative-sticky assignor (§11; `Q15`).

16. We found 30,000 messages in an SQS DLQ nobody had looked at for three weeks, and when we went to
    redrive them, the oldest had already disappeared. What went wrong twice?
    > **Direction:** No alert on DLQ arrivals, and standard-queue messages keep their original enqueue timestamp so DLQ retention equal to the source's expires them early; alert on DLQ arrival rate and set DLQ retention to 14 days (§15, §12; `Q16`).

17. Publishing an order with 5,000 line items fails with `RecordTooLargeException`. The team wants to raise
    `message.max.bytes` to 20 MB on the brokers. What do you propose instead?
    > **Direction:** Claim-check: store the payload in object storage and publish a reference (with a lifecycle longer than retention/replay); raising limits cascades into fetch sizes, memory and replication latency (§8, §9).

18. We deployed a new analytics consumer group on an existing topic with 7 days of data, and it only sees
    events from the moment it started. The team expected history. Why?
    > **Direction:** A new group has no committed offsets and `auto.offset.reset` defaults to `latest`; set it to `earliest` deliberately (and be ready for the backfill load) (§10; `Q15`).

19. A Lambda function processes SQS batches of 10. When one record is malformed, the other nine are
    reprocessed on every retry and customers get duplicate notifications. What is the fix?
    > **Direction:** A thrown error returns the whole batch; enable `ReportBatchItemFailures` and return only failed message IDs, plus a redrive policy so the bad one reaches the DLQ (§15; `Q16`).

20. When our Kafka cluster degraded last month, our checkout API — which only publishes an event after
    saving the order — had requests taking 60 seconds. Why did a messaging problem become an API outage?
    > **Direction:** The producer's `buffer.memory` filled and `send()` blocked for `max.block.ms` (60 s) on request threads; fail fast with a small `max.block.ms`, or write to a transactional outbox and publish asynchronously (§9, §13; `M15`).

### Level 3 — Redis under load: slow commands, memory and persistence

Redis-specific incidents where the answer depends on knowing the single-threaded model and how memory really behaves.

1. Every night at about 03:00, Redis stops answering for roughly 1.5 seconds and a burst of client
   timeouts hits the API. Nobody runs anything at 03:00 — but a nightly job at 02:00 builds a large
   sorted set with a one-hour TTL. Connect the dots.
   > **Direction:** The big key's expiry is a synchronous free of millions of elements on the main thread; enable `lazyfree-lazy-expire` (and friends) or `UNLINK` it explicitly, and split the key (§1, §2; `Q05`).

2. A developer ran `KEYS session:*` in production to debug something and took Redis out for six
   seconds. Beyond "tell people not to", how do you make this structurally impossible?
   > **Direction:** ACLs that deny `@dangerous` (or `rename-command` on old versions) for application and human users, `SCAN` tooling for debugging, and offline RDB analysis for key-space questions (§1, §2, §19).

3. Clients report Redis timeouts of up to 2 seconds several times an hour, but `SLOWLOG` is empty and
   CPU is modest. What could be happening that the slowlog cannot see?
   > **Direction:** The slowlog measures only command execution; look at fork latency (`latest_fork_usec`), swapping (fragmentation ratio < 1), AOF fsync stalls, network, and client-side pool starvation via `LATENCY DOCTOR` and client metrics (§1, §3, §2).

4. After deleting a large set of stale keys, `used_memory` dropped from 11 GB to 6 GB, but the Redis
   process still uses 11 GB RSS and the host is close to its limit. Why, and how do you get it back?
   > **Direction:** Allocator fragmentation (`mem_fragmentation_ratio` ≈ 1.8); enable `activedefrag` on jemalloc, or fail over to a freshly synced replica, and keep `maxmemory` well below the host limit (§2; `Q06`).

5. Our Redis pod in Kubernetes has `maxmemory 6gb` and an 8 GiB memory limit. It is OOM-killed
   roughly every hour, always while a `BGSAVE` is running. Explain the arithmetic.
   > **Direction:** `fork()` plus copy-on-write under write load can nearly double memory; the child's copied pages count against the container limit — lower `maxmemory` to ~50–60% of the limit, disable THP, or persist only on a replica (§3; `Q06`).

6. Since we enabled RDB snapshots on a write-heavy 40 GB instance, p99 latency spikes to 300 ms every few
   minutes. Where does the latency come from?
   > **Direction:** The fork itself blocks while copying page tables for a large process, and Transparent Huge Pages turn small writes into 2 MB page copies; check `latest_fork_usec`, disable THP, shard smaller or snapshot on a replica (§3).

7. Suddenly every write to Redis fails with `MISCONF Redis is configured to save RDB snapshots, but it's
   currently unable to persist to disk`. Memory is fine and reads work. What happened and what are the
   options?
   > **Direction:** A background save failed (disk full, or `fork` refused because of `vm.overcommit_memory=0`) and `stop-writes-on-bgsave-error yes` blocks writes; fix overcommit/disk, and decide consciously whether that safety switch belongs on a cache (§3).

8. We store 200 million `user:<id> → shard` string mappings in Redis and it costs 40 GB. Product wants
   to double the user base. How would you cut the memory by an order of magnitude without changing the
   access pattern much?
   > **Direction:** Bucket keys into small hashes (`HSET map:<id/1000> <id> <shard>`) that stay in `listpack` encoding, eliminating per-key overhead — the Instagram trick; mind the 128-entry/64-byte thresholds (§2; `Q05`).

9. Adding one "description" field to our cached product hashes increased Redis memory threefold, even
   though the average description is only 90 bytes. Why so much?
   > **Direction:** A value over `hash-max-listpack-value` (64 bytes) converts the hash from compact `listpack` to a full hashtable — per-field overhead explodes and it never converts back; keep big values in separate keys or raise the threshold knowingly (§2).

10. During a reporting job, Redis memory rises by 4 GB above the dataset and then eviction starts
    deleting hot cache entries. `CLIENT LIST` shows one client with `omem` in the gigabytes. What is
    going on?
    > **Direction:** The reporter is pulling huge replies (`HGETALL`/`LRANGE 0 -1` on big keys) faster than it reads them; normal-client output buffers are unlimited by default and count as memory — set `client-output-buffer-limit normal` and paginate with `*SCAN` (§2, §1).

11. Our leaderboard is a single sorted set with 50 million members. Reads are `ZREVRANGE` for pages of
    top players, and occasionally an admin export does `ZRANGE 0 -1`. Latency is fine except when it is
    catastrophic. What do you change?
    > **Direction:** It is a big key: O(N) whole-range reads block everyone and resharding cannot split it; paginate, run exports from a replica or an RDB, and split by bucket/season so no single key is enormous (§1, §2; `Q08`).

12. Our Redis-based rate limiter is a Lua script. After a change, the whole application started getting
    `BUSY Redis is busy running a script` errors and `SCRIPT KILL` did not work. What happened and how do
    you get out?
    > **Direction:** A script is atomic, so an unbounded loop blocks everything past `busy-reply-threshold`; `SCRIPT KILL` fails once it has written, leaving `SHUTDOWN NOSAVE` (i.e. a failover) — scripts must be O(1)/bounded and never iterate the key space (§1; `Q08`).

13. A retired feature left about 20 GB of keys under `legacy:*` on a busy production Redis. How do you
    remove them without an incident?
    > **Direction:** Find them offline from an RDB, then run a throttled background `SCAN MATCH legacy:* COUNT 500` + `UNLINK` loop with pauses, watching latency — never `KEYS` or a single `DEL` of a big key (§1, §2).

14. Our Redis dashboards show 12% host CPU and everything green, but clients are timing out at peak.
    What are the dashboards hiding?
    > **Direction:** Command execution is single-threaded: one core at 100% on an 8-core host reads as 12%; look at per-thread CPU and `INFO commandstats` to find the command family eating the thread (§1, §19).

15. Every day at 04:00 latency degrades for about a minute. At 03:00 a job imports 5 million keys, each
    written with `EX 3600`. What mechanism links the two?
    > **Direction:** Millions of keys expire in the same second and the active expiry cycle keeps running because most sampled keys are expired, eating the main thread; jitter the TTLs (§2, §5).

16. We use Redis Pub/Sub to push "order updated" events to our websocket servers. During peak, some
    users never see updates and a server occasionally logs a disconnect from Redis. What is happening
    and what would you use instead?
    > **Direction:** Pub/Sub is at-most-once and a slow subscriber is disconnected at the `pubsub` output-buffer limit, silently losing messages; use Streams with consumer groups (`XREADGROUP`, `XAUTOCLAIM`, `MAXLEN` trimming) or treat Pub/Sub as a hint backed by a resync (§4, §2).

17. After a nightly analytics job that reads every product key once, the cache hit rate for real traffic
    drops from 95% to 60% for hours. Nothing was deleted. Why?
    > **Direction:** A one-off scan pollutes an LRU cache and evicts the hot set; switch to `allkeys-lfu`, or run the job against a replica or the source database instead of the cache (§2).

18. Redis sits at 40% CPU but the host's 10 Gbit NIC is saturated, and latency climbs. One product's
    800 KB cached JSON is read about 1,500 times a second. What is the fix?
    > **Direction:** A big-value hot key saturates the network long before the CPU; cache it in-process for a few seconds, compress it, or split it into smaller keys so clients fetch only what they need (§2, §5).

19. Our Redis replicas report more keys and more memory than the primary, and a script that counts keys
    on the replica gives wrong answers. Why do they differ?
    > **Direction:** Replicas do not expire keys themselves — they wait for the primary's `DEL` and only hide expired keys on read — so expired-but-not-yet-deleted keys remain; count on the primary or accept the difference (§2).

20. To find a hot key, an engineer left `MONITOR` running in production for twenty minutes and throughput
    halved. What should they have done instead?
    > **Direction:** `MONITOR` streams every command and can halve throughput; use `--hotkeys` under an LFU policy, client-side sampling of key names, or a few seconds of `MONITOR` on a replica (§1, §2, §19).

### Level 4 — Cache consistency and invalidation races

Staleness bugs that survive every code review because they only happen under concurrency, replication lag or crashes.

1. Our price cache is cache-aside with delete-on-write. Under load, a small number of products show a
   stale price until the one-hour TTL expires, even though every write deletes the key. Nobody believes
   it is possible. Show them how, and give a fix stronger than a shorter TTL.
   > **Direction:** A slow reader's fill of the old value lands after the writer's delete; Facebook-style leases (delete invalidates the fill token) or version-compared sets in Lua prevent the stale fill (§6; `Q03`).

2. Staleness after writes only happens in the regions where reads go to database replicas. The primary
   region is fine. Why?
   > **Direction:** The invalidation arrives before the replica applies the write, so a miss refills from the replica with old data and a fresh TTL; fill from the primary briefly after a write (a "remote marker") or drive invalidation from the replica's change stream (§6; `Q03`).

3. We delete the cache key inside the same function that writes to the database, before the transaction
   commits. Occasionally the cache holds the old value after a successful update. Why is the placement
   wrong?
   > **Direction:** A reader can refill from the pre-commit state between the delete and the commit; invalidate after commit (after-commit hook, outbox or CDC) (§6, §13).

4. Very rarely — maybe twice a month — an entity is permanently stale until someone flushes it manually.
   We traced one case to a pod being killed between the database commit and the cache delete. How do
   you close that gap for good?
   > **Direction:** It is the dual-write problem; derive invalidations from the database change log (CDC / binlog tailer) so every committed write produces a delete, and keep TTLs as a backstop (§6, §13; `Q18`).

5. Another team runs nightly SQL scripts directly against our database, and our cache never learns about
   those changes. Asking them to call our API has failed politically. What is the technical answer?
   > **Direction:** Invalidate from CDC on the tables, which catches writes from any source, not only your application (§6, §13).

6. On update, our code writes the new value into the cache instead of deleting it. With two concurrent
   updates, the cache sometimes ends with the older of the two values forever. Why does "set on write"
   do this and what would you do?
   > **Direction:** Two writers' DB commits and cache sets can interleave in opposite orders; delete (idempotent, order-insensitive) instead of set, or set only if the version is newer (§6; `Q02`).

7. During a rolling deploy, v1 pods start throwing 500s reading cached objects written by v2 pods. The
   cache is shared. What should the design have been?
   > **Direction:** Version the key prefix or the payload so versions do not read each other's formats, and treat decode failure as a cache miss, never an error (§6, §7).

8. Merchandising changes a category's discount and needs all 200,000 product entries in that category
   invalidated immediately. Enumerating keys is off the table. How do you design for this?
   > **Direction:** Include a per-category generation number in the key (or keep tag sets) so one `INCR` invalidates the whole group logically; old entries age out by TTL (§6).

9. We cache `user:<id>` objects that embed the user's team name. Renaming a team leaves thousands of
   user entries stale. How would you restructure the cache?
   > **Direction:** Cache entities separately and compose at read time with an `MGET`, so each fact is invalidated in one place; denormalise only immutable data (§6).

10. We are active-active in two regions, each with its own Redis. Writes in EU are sometimes stale in US
    for an hour. What is missing?
    > **Direction:** Invalidations are only applied locally; each region must invalidate from its own replicated change stream so the delete arrives after the data does (§6).

11. We count article views with `INCR` in Redis and flush to the database every minute. After a Redis
    failover, about 30 seconds of counts were lost. The product manager asks if we can guarantee this never
    happens. What do you say?
    > **Direction:** Replication is asynchronous; acknowledged writes can be lost on failover and `WAIT` only narrows the window — either accept approximate counts, or write durably (append to a log or DB) and use Redis as the aggregator (§3).

12. During an incident, flipping a feature-flag kill switch took over a minute to take effect because
    flags are cached in-process for 60 seconds. How do you keep the cache and make kill switches fast?
    > **Direction:** Push invalidations (streaming/Pub/Sub with the TTL as a fallback), and have the kill switch path bypass or force-refresh the local cache; staleness bounds must be chosen per data type (§7, §6).

13. Our shopping carts live in Redis with write-behind to the database every five minutes. After a memory
    spike, carts vanished. Eviction policy is `allkeys-lru`. What is the category error?
    > **Direction:** A write-behind store is primary data, not a cache — it needs `noeviction`, persistence and its own instance (or the DB as source of truth) (§2, §3).

14. Our stampede lock has a 5-second TTL, but the recomputation sometimes takes 8 seconds. We see two
    recomputes and occasionally an older result overwriting a newer one. Explain and fix.
    > **Direction:** The lock expires mid-work so a second holder starts; size the TTL above the p99 recompute time and write results with a version/fencing check so the older result cannot win (§5, §4).

15. Implement early refresh for a hot cache entry without any locks or coordination between pods. Sketch
    the logic.
    > **Direction:** XFetch — recompute when `now − delta × beta × ln(rand()) ≥ expiry`, with `delta` the measured recompute time; hot keys refresh early once, cold keys never (§5; `Q04`).

16. We implemented stale-while-revalidate, but when the database had a 20-minute outage the site still
    went down once entries hit their hard TTL. How should it have behaved?
    > **Direction:** Stale-if-error: on refresh failure, extend the hard TTL (or keep serving the stale copy) rather than deleting, and alert on staleness age instead of failing the request (§5).

17. We guard order fulfilment with a Redis lock. Twice this month, two workers fulfilled the same order;
    one of them had a 12-second GC pause. The lock TTL is 10 seconds. Would Redlock fix it?
    > **Direction:** No — any lease-based lock can expire during a pause; correctness needs a fencing token checked by the resource (or a DB-level unique constraint/state transition), with the Redis lock only as an optimisation (§4; `Q08`).

18. Payment idempotency keys are stored in Redis with a 24-hour TTL. After a Redis failover, some
    payments were charged twice. What is wrong with the design?
    > **Direction:** Async replication lost the recent idempotency keys, so retries were not recognised; idempotency records must live in the durable store in the same transaction as the side effect (§3, §8; `M16`).

19. The coupon service caches "coupon not found" for 10 minutes. Marketing creates a coupon and emails it;
    the first customers who click the link get "invalid coupon". Fix it without removing the negative cache.
    > **Direction:** Creation must delete the negative entry (via the create event/CDC), and negative TTLs should be seconds, not minutes (§6; `Q04`).

20. Our HTTP response cache keys by URL. Some Spanish-speaking users got English pages, and hit rate is
    worse than expected because clients send query parameters in different orders. Fix both.
    > **Direction:** Normalise the key (sorted params, dropped tracking params) and include every input the response varies on (locale via `Vary` or an explicit key component) — both are key-design errors (§5, §6; `Q03`, `Q09`).

### Level 5 — Kafka consumer groups, rebalances and lag

Consumer-side Kafka incidents: the group protocol, liveness timeouts, lag diagnosis and parallelism.

1. Our consumer calls a fraud-scoring API that usually takes 50 ms but sometimes 3 s. When it degrades,
   the group enters a loop: rebalance, reprocess, rebalance. `max.poll.records` is 500. Walk me through
   the loop and give three fixes in order of preference.
   > **Direction:** Worst-case batch time exceeds `max.poll.interval.ms`, so members leave, partitions move and the same uncommitted batch is replayed; bound per-call timeouts, shrink `max.poll.records`, and move slow work off the poll thread with `pause()`/`resume()` (§11; `Q15`).

2. In Kubernetes, our consumer pods' liveness probe calls an HTTP endpoint that is served by the same
   thread pool as processing. During a rebalance, probes time out, pods restart and more rebalances
   follow. How do you break the cycle?
   > **Direction:** Liveness must not depend on processing progress during rebalances — separate the probe thread, use generous probe thresholds, and measure progress with a different signal; combine with static membership so restarts do not rebalance (§11).

3. We want to move 60 consumer pods from the eager range assignor to cooperative-sticky without an
   outage. A colleague plans to change the config and do one rolling restart. What goes wrong?
   > **Direction:** Members must agree on a common protocol; do a two-step bounce — first list both `CooperativeStickyAssignor` and the old assignor, then remove the old one in a second rollout (§11).

4. After enabling static membership, some pods crash-loop with `FencedInstanceIdException`. The consumers
   run as a Kubernetes Deployment with `group.instance.id` taken from an environment variable. What is
   wrong?
   > **Direction:** Two live members share a `group.instance.id` (Deployment pods are not stably named, or surge pods duplicate the ID); use StatefulSet ordinal names or another stable unique identity per replica (§11).

5. A JVM consumer with a 6 GB heap occasionally has 12-second full-GC pauses, and each one kicks it out of
   the group. `session.timeout.ms` is set to 10 seconds from an old tutorial. Fix both problems.
   > **Direction:** Heartbeat misses beyond the session timeout evict the member; raise the session timeout (the modern default is 45 s) and fix the GC (smaller heap, G1/ZGC, fewer retained records) (§11).

6. A group with 200 consumers across 600 partitions takes about two minutes per rebalance, during which
   nothing is processed. Rebalances happen several times a day. What are your options?
   > **Direction:** Eager rebalancing is a group-wide stop-the-world barrier; cooperative incremental assignment, static membership and — on Kafka 4.0+ — the KIP-848 broker-side protocol remove the global pause (§11).

7. A `read_committed` consumer shows lag growing on a few partitions for over an hour, but it is receiving
   no records at all and is healthy. Producers are transactional. What is happening?
   > **Direction:** An open or hanging transaction pins the last stable offset, so `read_committed` readers cannot advance; find it with `kafka-transactions.sh --find-hanging` and abort, and fix the producer that leaves transactions open (§13, §11).

8. One partition's lag has grown steadily for three hours while its consumer instance reports as healthy
   and its heartbeat is fine. What is your investigation?
   > **Direction:** Heartbeats come from a background thread, so a member can be "alive" with a stuck processing thread (e.g. an HTTP call with no timeout); check the owner via `--describe`, take a thread dump, and add timeouts on every downstream call (§11, §19).

9. Consumer lag dashboards show near zero, but customers see events reflected hours late. How can both be
   true?
   > **Direction:** Lag measures committed offsets, not completed work — async processing, internal queues or auto-commit can commit before processing; measure end-to-end latency from event timestamps to completion (§11, §12).

10. We need five times more throughput, the topic has 12 partitions, and we must preserve ordering per
    customer. Repartitioning is a multi-month project. What can you do this week?
    > **Direction:** Process in parallel within each partition by key (a keyed worker pool or Confluent Parallel Consumer) and commit the lowest contiguous completed offset (§11, §12; `Q14`).

11. Our `payments` topic is keyed by merchant ID. One merchant does 40% of the volume, so one partition is
    always behind. Ordering matters, but only per payment. What do you do?
    > **Direction:** The key is coarser than the ordering requirement; re-key by payment ID (or salt the hot merchant key), which is only safe because per-merchant order is not required (§11, §9; `Q14`).

12. Every time the data team creates a new topic, our consumer group rebalances. We subscribe with a
    regex. Explain and fix.
    > **Direction:** Pattern subscriptions re-evaluate metadata and trigger rebalances when matching topics appear or disappear; subscribe to an explicit list, or narrow the pattern, and use cooperative assignment to limit impact (§11).

13. Our consumer commits synchronously after every message and tops out at about 200 messages per second
    per partition. How do you get more without losing at-least-once?
    > **Direction:** Each sync commit is a broker round trip; commit asynchronously per batch or interval, and synchronously only in `onPartitionsRevoked` and on shutdown (§12).

14. After every rebalance we reprocess a few hundred messages per partition, which is causing duplicate
    webhooks. What are we probably missing in the consumer code?
    > **Direction:** Offsets for completed work are not committed on revocation; commit in the rebalance listener's `onPartitionsRevoked`, and keep consumers idempotent because some duplication is unavoidable (§12, §8).

15. A consumer group was stopped for ten days while a feature was paused. When it restarted, it skipped
    everything produced in those ten days. No one reset anything. Why?
    > **Direction:** Committed offsets for an empty group expire after `offsets.retention.minutes` (7 days), and `auto.offset.reset=latest` skips to the end; keep groups alive, raise retention, or record offsets externally before pausing (§10).

16. Our lag-based autoscaler oscillates between 10 and 40 pods every few minutes. What is feeding the
    oscillation and how do you tune it?
    > **Direction:** Scaling causes rebalances, which pause consumption and raise lag, which causes more scaling; scale on time lag with hysteresis and cooldowns, never beyond the partition count (§11, §8).

17. We doubled a topic from 12 to 24 partitions to add consumers. Since then, our consumer's local
    per-customer cache and aggregation state are wrong. Why?
    > **Direction:** `murmur2(key) mod partitions` changed, so keys moved partitions mid-stream and per-key ordering/locality broke; migrate to a new topic with a controlled cut-over instead of adding partitions to keyed topics (§9, §10; `Q14`).

18. Our Kafka Streams application takes 40 minutes to become useful after each deploy because tasks are
    restoring state stores. How do you get deploys down to seconds?
    > **Direction:** Moving tasks forces changelog restoration; use static membership and cooperative rebalancing to keep assignments sticky, `num.standby.replicas` for warm copies, and warm-up tasks so state moves before ownership does (§17, §11).

19. Our consumers run in three AZs and the monthly network bill for Kafka fetches is larger than the
    brokers themselves. What can you change without moving anything?
    > **Direction:** Consumers read from leaders in other AZs; enable follower fetching (KIP-392, `RackAwareReplicaSelector` plus `client.rack`) so each consumer reads from a same-AZ replica (§9).

20. A partition stopped progressing for three hours. The consumer pod was alive, in the group, and
    polling. It turned out the app pauses a partition when its internal worker queue is full — and the
    worker thread had died. What does this teach you about consumer health checks?
    > **Direction:** Liveness is not progress; health must track per-partition progress (committed offset advancing, worker heartbeats, `last-poll-seconds-ago`) and fail the pod or resume partitions when progress stops (§11, §19).

### Level 6 — Kafka durability, producers and retention

Where data is lost, reordered or kept too long — producer settings, ISR maths, retention and compaction.

1. After a broker failure we lost about 2,000 acknowledged messages. Producers use `acks=all`. The team is
   baffled. What is the first config you check?
   > **Direction:** `min.insync.replicas` (default 1): with the ISR shrunk to the leader alone, `acks=all` acknowledged writes on one disk; use RF=3, `min.insync.replicas=2` (§9; `Q13`).

2. During a routine rolling upgrade, all producers to one topic failed with `NotEnoughReplicasException`
   the moment any broker restarted. Replication factor is 3. What is the config and why is it wrong?
   > **Direction:** `min.insync.replicas=3` (or RF=2 with min.insync=2) makes any single broker restart a write outage; RF=3 with min.insync=2 tolerates one failure (§9; `Q13`).

3. After an outage, a consumer saw different records at the same offsets than before the outage, and a
   reconciliation job found missing orders. What setting allowed this?
   > **Direction:** `unclean.leader.election.enable=true` let an out-of-sync replica become leader, truncating acknowledged data and reusing offsets; keep it false for data you cannot lose and accept partition unavailability instead (§9; `Q13`).

4. A legacy service using an old Kafka client occasionally produces events out of order within a key,
   only during broker hiccups. The key is correct. What is happening?
   > **Direction:** Retries with more than one in-flight request and idempotence disabled let a retried batch land after a later one; enable idempotence (or set `max.in.flight.requests.per.connection=1`) (§9).

5. We enabled the idempotent producer, yet we still see duplicate events downstream. Why does
   idempotence not cover it?
   > **Direction:** Idempotence only dedupes retries within one producer session; application-level resends after a timeout or a restart create new records — dedupe at the consumer by event ID or use the outbox with event IDs (§9, §13, §8).

6. For GDPR, we set `retention.ms` to one day on a low-volume topic, but an audit found records ten days
   old still on disk. Explain.
   > **Direction:** Retention deletes whole closed segments; a 1 GB/7-day segment on a low-volume topic never rolls, so nothing expires — lower `segment.ms` to below the retention target (§10).

7. Broker disks have been filling for weeks, and retention on the busiest topic does not seem to delete
   anything. `kafka-dump-log.sh` shows a record timestamped in 2031. What happened?
   > **Direction:** Retention uses the segment's largest timestamp, so a future `CreateTime` keeps the segment (and everything after it) alive; use `LogAppendTime` or timestamp bounds, and fix the producer clock (§10, §19).

8. A compacted topic that should hold a few GB of latest state has grown to 800 GB over several months.
   What do you check?
   > **Direction:** The log cleaner thread may have died or be starved (dedupe buffer, a corrupt segment); check `log-cleaner.log`, `max-dirty-percent` and `time-since-last-run-ms`, and restart/fix it — the same failure grows `__consumer_offsets` (§10).

9. Services that bootstrap state from a compacted `users` topic occasionally resurrect users who were
   deleted weeks ago. Deletes are published as tombstones. Why?
   > **Direction:** Tombstones are kept only for `delete.retention.ms` (24 h by default); a consumer that reads slower than that, or restores from an old snapshot, never sees the delete — raise it above the longest bootstrap and snapshot age (§10; `Q18`).

10. We increased partitions on the `orders` topic during a traffic spike. The next day, fulfilment
    processed some `OrderShipped` events before their `OrderPaid`. Connect the two.
    > **Direction:** Adding partitions changes key → partition mapping, so in-flight events for the same key landed on different partitions with independent ordering; plan partition counts, or migrate topics with a cut-over (§9; `Q14`).

11. `UnderReplicatedPartitions` on one broker oscillates between zero and a few hundred all day, and
    produce latency spikes with it. Where do you look?
    > **Direction:** ISR shrink/expand flapping points at one slow broker — disk latency, GC, network or an overloaded request handler; investigate that broker's metrics rather than tuning `replica.lag.time.max.ms` to hide it (§9, §19).

12. We added three brokers and ran a partition reassignment. Within minutes, producers were timing out
    and ISR shrank cluster-wide. What went wrong?
    > **Direction:** Unthrottled reassignment saturates network and disks and starves live replication; reassign in small batches with `--throttle` (or Cruise Control) (§9).

13. After a broker was restarted for patching, it hosts no partition leaders and the other brokers are
    at 80% CPU. Why hasn't it balanced back?
    > **Direction:** Leadership moved away and stays until a preferred-leader election runs; trigger `kafka-leader-election.sh` (or check `auto.leader.rebalance.enable`) (§9).

14. A team replayed three days of a busy topic for a backfill. During the replay, produce latency and
    tailing-consumer latency tripled for every team on the cluster. Why did one consumer affect everyone?
    > **Direction:** Old data is read from disk, evicting the page cache that tailing reads and writes depend on; throttle the replay with client quotas or run it from a separate cluster/follower (§9, §12).

15. A service produces 30,000 small events per second and broker CPU is high while throughput is
    disappointing. Monitoring shows about one record per produce request. What do you tune?
    > **Direction:** Batching: raise `linger.ms` (5–20 ms) and `batch.size`, enable `lz4`/`zstd` compression — fewer, larger requests cut broker CPU and network (§9).

16. We set `retention.bytes` to 100 GB on a topic expecting it to use at most 100 GB. It used 7 TB. What
    did we misunderstand?
    > **Direction:** `retention.bytes` is per partition and replicas multiply it — 100 GB × 24 partitions × RF 3; plan disk from partitions × RF (§10).

17. A consumer that fell far behind suddenly jumped back to the beginning of the topic and reprocessed
    seven days of events. Nobody reset anything. What happened and how do you prevent it?
    > **Direction:** Retention deleted the segment holding its committed offset, so `OffsetOutOfRange` applied `auto.offset.reset=earliest`; alert on lag approaching retention and consider `auto.offset.reset=none` to fail loudly (§10, §12).

18. An entire AZ lost power. Every topic had RF=3, yet we lost acknowledged messages. How?
    > **Direction:** Kafka acknowledges from the page cache, not fsync, so if all replicas were in one AZ they died together; spread replicas with `broker.rack` rack awareness (§9; `Q13`).

19. When one broker becomes slow, the producers of null-keyed events seem to send it *more* traffic, not
    less, making it worse. Why?
    > **Direction:** The old sticky partitioner keeps filling a batch for a partition until it is sent, and slow partitions accumulate larger batches; KIP-794's adaptive partitioning (3.3+) routes away from slow brokers (§9).

20. A topic was deleted and recreated with the same name to "clean it up". The consumer group resumed
    with odd behaviour — some consumers stuck, others skipped data. Why, and what is the safe procedure?
    > **Direction:** Committed offsets point into the old topic's offsets, so they are out of range or meaningless; stop consumers, reset the group's offsets explicitly (or use a new group), and prefer a new topic name (§10, §12).

### Level 7 — Broker traps: RabbitMQ, SQS and job queues

Incidents that depend on one broker's specific defaults and semantics — the details you only know after running it.

1. RabbitMQ hit its memory alarm during a spike. Publishers blocked as expected — but consumers also
   stopped acking, the queue never drained and the alarm never cleared. Our service publishes and consumes
   over a single connection. Explain the deadlock.
   > **Direction:** The alarm blocks the whole publishing connection, including the consumer's acks on it, so nothing drains; use separate connections for publishing and consuming and handle `connection.blocked` (§14; `Q11`).

2. One malformed message pins a RabbitMQ consumer at 100% CPU and the broker's redelivery rate is tens of
   thousands per second. The handler does `nack(requeue=true)` on any exception. What is the fix?
   > **Direction:** Requeue puts it straight back at the head — a hot loop; nack with `requeue=false` to a DLX, count attempts via `x-death`, or use quorum queues' `delivery-limit` (§14; `Q11`, `Q16`).

3. We implemented delayed retries in RabbitMQ with per-message TTLs on a single wait queue: 5 s, 1 min,
   30 min. Some 5-second retries take 30 minutes. Why?
   > **Direction:** Per-message TTL is only evaluated at the head of the queue, so a 30-minute message blocks everything behind it; use one wait queue per delay tier with a queue-level TTL (or the delayed-message plugin, knowing its limits) (§14).

4. After a routing-key rename in one service, messages silently stopped arriving at a consumer. No errors
   anywhere, publisher confirms all succeeded. Why, and what would have caught it?
   > **Direction:** Unroutable messages are dropped and still confirmed; publish with `mandatory=true` (handling returns) or configure an alternate exchange, and alert on unroutable counts (§14).

5. Our 40-minute report jobs on RabbitMQ are redelivered and started again just after 30 minutes, forever.
   The workers are healthy. What is doing it?
   > **Direction:** `consumer_timeout` (30 min default) closes channels with unacked deliveries and requeues them; raise it for that queue/policy, or ack early and track job state elsewhere with its own heartbeat (§14, §16).

6. A single RabbitMQ queue tops out around 25,000 messages per second, and adding nodes to the cluster
   does nothing. How do you scale it?
   > **Direction:** Each queue is a single Erlang process on one node (one core); shard across multiple queues with a consistent-hash exchange or the sharding plugin (§14).

7. We set `x-max-length` on a queue to protect the broker. Weeks later, we find orders were never
   processed and nothing logged an error. What happened?
   > **Direction:** The default overflow is `drop-head`, which silently discards the oldest messages; use `reject-publish` so publishers get nacks, or dead-letter the overflow (§14).

8. After a network partition between RabbitMQ nodes healed, some acknowledged messages had vanished. The
   cluster uses `autoheal`. Explain the trade-off you made without knowing.
   > **Direction:** `autoheal` picks a winning partition and discards the losers' state for availability; `pause_minority` (with quorum queues) prefers consistency (§14).

9. Our SQS FIFO queue uses the constant `"orders"` as the `MessageGroupId`. Throughput tops out far below
   what we need and consumers idle. What is the design error?
   > **Direction:** A single group allows one in-flight batch at a time — the whole queue is serialised; use the entity whose order matters (order or customer ID) as the group ID, and enable high-throughput mode (§15).

10. On an SQS FIFO queue grouped by tenant, one tenant's orders stopped processing for four hours while
    everyone else was fine. What happened, and what should the redrive policy be?
    > **Direction:** A failing message at the head of its group blocks that group until it succeeds or hits `maxReceiveCount`; set a low `maxReceiveCount` with a DLQ so a poison message does not stall the tenant (§15, §12).

11. After a deploy with a crash-looping consumer, thousands of perfectly valid messages ended up in the
    SQS DLQ. Why, and how do you stop it happening again?
    > **Direction:** `maxReceiveCount` counts receives, not processing attempts, so consumers that crash or time out burn the count; set visibility and receive count with headroom, fail deploys fast, and redrive with `StartMessageMoveTask` (§15).

12. A backlog on an SQS queue triggered Lambda to scale to 900 concurrent invocations, which exhausted
    Postgres connections and took the database down. What knob was missing?
    > **Direction:** The SQS event source's maximum concurrency caps pollers without the throttling that reserved concurrency causes (which also burns receive counts); pair it with a connection pooler/proxy (§15, §8).

13. Our SQS bill is dominated by `ReceiveMessage` calls, most returning nothing, from 200 idle workers.
    What is the cheap fix?
    > **Direction:** Enable long polling (`WaitTimeSeconds=20`) and batch receives/deletes of up to 10 (§15).

14. Image-resize jobs in BullMQ sometimes run twice and some fail with "job stalled more than allowable
    limit". The processor uses a synchronous image library. What is happening?
    > **Direction:** CPU-bound work blocks the Node event loop, so the lock renewal timer never fires and the job is declared stalled and re-run; use sandboxed processors or worker threads (§16).

15. The Redis behind our BullMQ queues grows by several gigabytes a week, although the queues are nearly
    always empty. Why?
    > **Direction:** Completed and failed jobs are kept forever unless `removeOnComplete`/`removeOnFail` are set; add bounded retention (count or age) (§16).

16. Celery tasks scheduled with `countdown=7200` are being executed three times each. Short tasks are
    fine. The broker is Redis and `acks_late` is on. What is going on?
    > **Direction:** The Redis broker's `visibility_timeout` (1 h) expires before the ETA, so the unacked task is redelivered to other workers repeatedly; raise it above the longest ETA/task or use a real scheduler for delayed work (§16).

17. Our Celery workers show idle processes while short, urgent tasks wait several seconds in the queue
    behind long ones. What default is at fault?
    > **Direction:** `worker_prefetch_multiplier=4` lets a busy process reserve tasks it cannot start; set it to 1 (with `acks_late`) and separate long and short tasks into different queues (§16).

18. The monthly invoice run happened twice this month. We had just scaled the scheduler deployment to two
    replicas for resilience. What is the lesson?
    > **Direction:** Every scheduler replica fires every schedule (two `celery beat`s double every task); run one scheduler with a lock/leader election, and make the job itself idempotent per period (§16).

19. One tenant started a 2-million-row import and every other tenant's jobs waited for two hours. How do
    you design the job system so that cannot happen?
    > **Direction:** Replace the single FIFO queue with fairness: per-tenant queues with weighted round-robin, per-tenant concurrency caps, and separate interactive/bulk lanes — or shuffle sharding to contain noisy tenants (§16; `Q21`).

20. Our database spikes to 100% at exactly 00:00 UTC every night, and after last week's two-hour outage
    the scheduler fired every missed run at once when it recovered. Fix both.
    > **Direction:** Spread per-tenant schedules with a hashed offset instead of midnight for all, and set explicit misfire/catch-up policy (skip or run once, not all missed runs) (§16).

### Level 8 — Redis high availability, clustering and failover

What happens when Redis nodes fail, move or multiply — and the client behaviour that turns a failover into an outage.

1. After an ElastiCache primary failover, our Java services logged `READONLY You can't write against a
   read only replica` for about ten minutes, until they were restarted. Why didn't they follow the
   failover?
   > **Direction:** Clients (JVM DNS cache or long-lived pooled connections) kept talking to the old primary, now a replica; short DNS TTL caching, and reconnect/refresh topology on `READONLY` (§4; `Q07`).

2. A 20-second network blip to our Redis cluster was followed by 15 minutes of Redis at 100% CPU with
   almost no commands served. 1,000 pods each have a pool of 50 TLS connections. What is going on?
   > **Direction:** A reconnection storm — tens of thousands of simultaneous TLS handshakes on a single-threaded server; reconnect with backoff and jitter, shrink pools, and consider a connection-multiplexing proxy (§4).

3. A new replica for our 25 GB Redis never finishes syncing. The primary's memory climbs, it forks
   repeatedly, and last night it was OOM-killed. Logs mention the replica being disconnected due to
   output buffer limits. Explain the loop.
   > **Direction:** During the full sync, writes buffered for the replica exceed `client-output-buffer-limit replica` (256 MB/64 MB for 60 s), the replica is dropped and restarts a full sync; raise the limit to write-rate × sync time and increase `repl-backlog-size` (§3; `Q07`).

4. Our Redis primary runs without persistence for speed. It crashed, systemd restarted it in two seconds,
   and all three replicas came back empty too. How?
   > **Direction:** The restarted primary came back empty before failover detection and replicas resynced to it; either persist on the primary or disable auto-restart so Sentinel/cluster promote a replica first (§3; `Q06`, `Q07`).

5. We are migrating from a single Redis to Redis Cluster. `MGET`, our Lua scripts and `MULTI` blocks now
   fail with `CROSSSLOT`. How do you migrate without rewriting everything?
   > **Direction:** Co-locate keys that must be operated on together with hash tags (`{user:42}:...`), and split the remaining multi-key operations per slot — choosing tags carefully to avoid creating hot slots (§4; `Q07`).

6. After moving to cluster mode we used `{tenant_id}` as a hash tag everywhere. One shard now holds 70%
   of the memory and is always hot. What do you do?
   > **Direction:** A too-coarse hash tag pins a big tenant to one slot that cannot be split; tag at the smallest unit that needs atomic multi-key ops (a user, a cart), not the tenant (§4, §2).

7. During a routine reshard, the cluster froze for several seconds at a time and clients timed out. What
   was being migrated?
   > **Direction:** `MIGRATE` is blocking per key, so big keys freeze both source and target; find big keys with `--bigkeys`/`--memkeys` and split them before resharding (§4, §2).

8. We lost one Redis Cluster shard — both primary and its replica were on the same host. The whole
   cluster stopped answering, even for keys on healthy shards. Why?
   > **Direction:** `cluster-require-full-coverage yes` (default) makes the cluster refuse queries when any slot is uncovered; set it to `no` for caches, and spread replicas across hosts/AZs (§4).

9. During a network partition, our old Redis primary kept accepting writes from clients on its side while
   Sentinel promoted a new one. After healing, those writes were gone. How do you limit this?
   > **Direction:** Split brain with async replication; `min-replicas-to-write`/`min-replicas-max-lag` make an isolated primary stop accepting writes, bounding the loss (§3; `Q07`).

10. We use Pub/Sub on a 30-node Redis Cluster for live notifications, and cluster bus traffic now exceeds
    real traffic. Why?
    > **Direction:** Classic `PUBLISH` in a cluster is broadcast to every node; use Redis 7 sharded Pub/Sub (`SPUBLISH`/`SSUBSCRIBE`) so a channel lives on its slot (§4).

11. Every short network blip between our Redis primary and replica — even 10 seconds — triggers a full
    resync of 15 GB instead of a partial one. Why?
    > **Direction:** The default `repl-backlog-size` (1 MB) is overrun within seconds at our write rate; size the backlog to write rate × tolerated disconnect (hundreds of MB) (§3).

12. Our credit-balance service does `INCRBY` in Redis with a client that retries on timeout. After a period
    of Redis latency, several customers' balances were double-credited. Why?
    > **Direction:** A timed-out command may already have executed, so retrying non-idempotent commands double-applies; don't auto-retry non-idempotent writes, or make them idempotent (Lua with an operation ID) (§4).

13. We keep hitting `ERR max number of clients reached` on Redis while its CPU is at 20%. We have 1,200
    pods with a pool size of 50. What would you change?
    > **Direction:** 60,000 connections exceed `maxclients` 10,000 and buy nothing on a single-threaded server; shrink pools drastically (pipelining instead), or put a proxy in front (§4, §1).

14. A single `MGET` of 200 keys on Redis Cluster is ten times slower than on our old standalone Redis.
    The client "just works". What is it doing?
    > **Direction:** Cluster-aware clients split multi-key reads by slot and fan out to many nodes, so latency is the slowest of N round trips; hash-tag related keys or accept per-node batching (§4).

15. Our distributed lock is in Redis with Sentinel. After a failover, two processes held the same lock.
    Why can't Redis replication prevent this, and what do you do?
    > **Direction:** The lock write was acknowledged by the primary but not replicated before failover; lease locks on async replication are best-effort — use fencing tokens or a consensus store for correctness (§4, §3; `Q08`).

16. After losing writes in a Redis failover, the team proposes adding `WAIT 1 100` after every write. What
    does it guarantee, what does it cost, and what doesn't it fix?
    > **Direction:** It blocks until one replica acknowledged or the timeout passes (so it may return 0), adding a round trip; it is not consensus — a failover can still promote a replica that lacked the write (§3).

17. We moved an API to AWS Lambda and now Redis shows thousands of new connections per second and high
    CPU, while ops/s barely changed. What is happening and how do you fix it?
    > **Direction:** Connect-per-invocation churn (TCP/TLS/AUTH for each cold start); reuse connections across invocations, cap concurrency, or use a proxy that multiplexes (§4).

18. To detect failures faster we cut `cluster-node-timeout` to 2 seconds. Now we get several failovers a
    week with no real outage, and each loses a few writes. Explain.
    > **Direction:** A short timeout turns GC pauses, forks and network jitter into false failovers, each losing the async tail; keep a realistic timeout and fix the latency sources (§4, §3).

19. To relieve a hot key we started reading it from replicas. Now users complain that settings "revert"
    right after they save them. What is the trade-off, and how do you keep the relief?
    > **Direction:** Replica reads lag, breaking read-your-writes; route a user's reads to the primary for a short window after they write (or version-check), and serve replicas only to others (§5, §6).

20. We need to patch every node of a 12-shard Redis Cluster during business hours without losing writes.
    Last time we just restarted nodes and lost data. What is the procedure?
    > **Direction:** Upgrade replicas first, then run a coordinated `CLUSTER FAILOVER` on each replica (it waits for replication to catch up), then upgrade the demoted primaries — never kill primaries and wait for automatic failover (§4, §3).

### Level 9 — Stream processing, CDC, schemas and replay

Less common but decisive in data-heavy teams: event time, state, change data capture and schema evolution.

1. On Monday morning our Postgres primary ran out of disk and stopped accepting writes. The Debezium
   connector had been down since Friday evening. Nothing else changed. Explain, and how do you make sure a
   CDC outage can never take down the source database again?
   > **Direction:** The replication slot retains WAL until the connector confirms it; monitor slot lag, set `max_slot_wal_keep_size` as a circuit breaker (accepting a re-snapshot if it trips) and alert on inactive slots (§13; `Q18`).

2. Our Debezium connector is healthy and caught up, yet WAL on the primary grows by 50 GB a day. The
   connector watches one low-traffic table in a very busy database. Why?
   > **Direction:** The slot only advances when the connector receives changes for its tables; configure `heartbeat.interval.ms` with a `heartbeat.action.query` so the slot's confirmed LSN keeps moving (§13).

3. Our outbox relay polls `SELECT ... WHERE id > :last ORDER BY id`. Occasionally an event is never
   published, with no error. Explain how rows are skipped.
   > **Direction:** Sequence values are allocated before commit, so a higher ID can commit first and the poller advances past a lower ID that commits later; poll with a safety lag window, track unpublished rows by status, or use CDC on the outbox (§13; `Q18`).

4. Our Flink job computes per-minute metrics per region and emits nothing between 01:00 and 05:00 every
   night, then catches up. One region has almost no traffic at night. Why?
   > **Direction:** The watermark is the minimum across input partitions, so an idle partition stops it advancing and windows never fire; declare source idleness (`withIdleness`) (§17; `Q17`).

5. One morning our streaming job started dropping almost every event as late. We found a single IoT
   device whose clock was set to 2031. How did one device break everyone, and what is the defence?
   > **Direction:** Bounded-out-of-orderness watermarks follow the max event time, so one future timestamp jumps the watermark and makes all correct events late; clamp or reject timestamps far ahead of ingestion time, or compute watermarks per source (§17).

6. After a consumer outage and catch-up, our hourly revenue dashboard showed a huge spike at 14:00 and
   nothing for the three preceding hours. The data is correct in the database. What went wrong?
   > **Direction:** Aggregations were keyed by processing time, so the backlog was attributed to when it was processed; window by event time from the payload (§17).

7. To fix a bug, we replayed a week of `OrderCompleted` events into the fixed consumer. Thousands of
   customers received "thanks for your order" emails again. How should the replay have been run?
   > **Direction:** Replays must suppress or dedupe external side effects — a replay mode, a separate consumer group whose sink is idempotent, and throttling — decided before resetting offsets (§12; `Q16`).

8. A producer added a new value to an Avro enum. Within minutes, consumers on every partition crashed and
   stayed down. The registry accepted the schema. Why, and what prevents it?
   > **Direction:** Old readers cannot decode an unknown enum symbol without a reader-side `default`, so every new record is a deserialiser-level poison pill; design enums with defaults, and test compatibility beyond the registry's rules (§18, §12; `Q19`).

9. A developer's local build, pointed at production by mistake, registered a new schema version for a
   core topic, and producers began using it. What should have stopped this?
   > **Direction:** Production producers should run with `auto.register.schemas=false`; schemas are registered through CI with compatibility checks, and credentials restrict who can register (§18; `Q19`).

10. We use `BACKWARD` compatibility. A team deployed the producer first with a new required field and the
    consumers broke. What does `BACKWARD` actually promise about deploy order?
    > **Direction:** BACKWARD means the new schema can read old data — so consumers upgrade first; producers-first requires FORWARD, and replays of long-retention topics need `*_TRANSITIVE` (§18; `Q19`).

11. Our Kafka Streams pods keep getting OOM-killed by Kubernetes, but the JVM heap graphs look fine,
    well below `-Xmx`. What is using the memory?
    > **Direction:** RocksDB state stores allocate off-heap block caches and memtables per store; bound them with a `RocksDBConfigSetter` and size the container limit for heap plus RocksDB (§17).

12. Under peak load our Flink job's checkpoints start timing out, and after a restart it has to replay
    hours of data. Why do checkpoints fail exactly when load is highest?
    > **Direction:** Aligned checkpoint barriers queue behind backpressured data; enable unaligned checkpoints and address the backpressure source (§17).

13. Downstream consumers of our Flink job's Kafka output see data only once a minute, in bursts; and after
    a long outage we lost some output. Checkpoints are every 20 minutes. Explain both.
    > **Direction:** The exactly-once Kafka sink commits transactions on checkpoint, so `read_committed` readers see data per checkpoint, and a checkpoint interval near `transaction.timeout.ms` lets the broker abort transactions before commit (§17, §13).

14. We replayed three months of orders through our enrichment job and the output differs from the
    original run — customer segments are "wrong". The code has not changed. Why?
    > **Direction:** A stream-table join reads the table's current state, so old events joined today's reference data; use temporal (versioned) joins if history must be reproduced (§17).

15. Our stream deduplication job's state has grown to 400 GB and restores take hours. It stores every
    event ID ever seen. How should it be designed?
    > **Direction:** Dedup state must be bounded by a time window matching the realistic duplicate horizon (e.g. 24 h with TTL on state); unbounded dedup is a memory leak (§17).

16. After the Debezium connector lost its offsets, it re-snapshotted the whole 400-million-row `orders`
    table and downstream consumers were flooded for a day. What should consumers and operators have been
    ready for?
    > **Direction:** Snapshots emit `r` events for every row; consumers must treat them as idempotent upserts and throttle, and operators should protect connector offsets and consider incremental snapshots (§13).

17. A migration renamed a column in the `customers` table. Three downstream teams' consumers broke,
    although no one changed any event. How do you prevent this class of incident?
    > **Direction:** Raw CDC exposes the physical schema as the event contract; publish domain events via an outbox (Debezium outbox router) and gate DDL on downstream compatibility (§13, §18).

18. Two instances of our transactional producer keep throwing `ProducerFencedException` at each other
    in a loop, and throughput has collapsed. What is wrong?
    > **Direction:** Both use the same `transactional.id`, so each new session bumps the epoch and fences the other; the ID must be unique per logical producer instance (e.g. per input partition or pod ordinal) (§13).

19. Our Kafka Streams app runs with `exactly_once_v2`, yet customers were charged twice during a
    rebalance. The charge is an HTTP call inside the processor. Why doesn't exactly-once apply?
    > **Direction:** Kafka EOS covers only Kafka reads, writes and offset commits; external side effects re-run on task retry — use an idempotency key with the payment provider or emit a command to a downstream idempotent consumer (§13, §8).

20. Someone deleted an old schema version from the registry during a clean-up. Weeks later, a replay of
    that topic failed on the first old record. What is the rule?
    > **Direction:** Records reference schema IDs, so a version is needed as long as any retained (or archived) data uses it; never delete schema versions within the replay horizon (§18).

### Level 10 — Rare, multi-system and staff-level incidents

Cascades across cache, broker and database, and edge cases that only teams operating this infrastructure at scale have seen.

1. Redis did not go down — it became slow, with p99 of 800 ms. The circuit breaker never opened because
   there were no errors. Thread pools filled, and the whole API went down. Redesign the failure handling.
   > **Direction:** A slow dependency is worse than a dead one; tight cache timeouts (tens of ms), breakers that trip on latency, bulkheaded pools for cache calls, and treating cache latency as a miss (§5, §4; `M10`).

2. A 30-second Redis blip turned into a 20-minute outage. The mobile client retries three times, the API
   gateway retries twice, the service retries Redis three times and the Kafka consumer retries its own
   work. Quantify and fix.
   > **Direction:** Retries multiply across layers (3 × 2 × 3 = 18× load on the struggling component); retry at one layer only, with backoff, jitter and retry budgets, and cache calls that fall through rather than retry (§4, §8, §12; `M09`).

3. We scaled our memcached pool from 10 to 11 nodes to add capacity, and the database fell over within a
   minute. Why did adding cache capacity cause an outage?
   > **Direction:** Modulo-based client sharding remapped ~90% of keys, a near-total miss storm; use consistent hashing (ketama) and add capacity gradually or pre-warm (§7).

4. We failed over to our DR region during an incident. Its Redis was empty, and its database — sized for
   normal DR reads — could not handle 100% misses. How do you design regional failover for caches?
   > **Direction:** A cold cache is a capacity event: pre-warm (replicated hot keys or cold-cluster warm-up from the other cluster), shift traffic gradually, and size the DR database for the miss rate you will actually see (§7, §5).

5. A downstream DB slowed down. Consumer latency rose, the lag autoscaler added pods, the group rebalanced
   repeatedly, duplicates were reprocessed, and the extra work made the DB slower still. Describe the
   death spiral and the controls that break it.
   > **Direction:** A positive feedback loop between scaling, rebalances and duplicate work; cap concurrency at the dependency's capacity, back off on downstream latency (pause partitions), use cooperative/static membership, and scale on time lag with hysteresis (§8, §11).

6. An old DLQ that nobody drained filled a RabbitMQ node's disk. The disk alarm blocked publishers for
   every service sharing the cluster, including ones unrelated to the DLQ. What design failures stacked up?
   > **Direction:** Unbounded DLQs without alerts, a tiny `disk_free_limit`, and a shared cluster where one tenant's queues can trigger cluster-wide flow control; bound and alert on DLQs, set realistic disk limits, and isolate critical traffic (per-vhost limits or separate clusters) (§14, §12).

7. After a broker upgrade, a handful of partitions stopped delivering to `read_committed` consumers.
   Producers were healthy and aborts via the client did nothing. What tool do you reach for?
   > **Direction:** A hanging transaction left by the broker holds the LSO beyond any client timeout; `kafka-transactions.sh --find-hanging` then `abort` (KIP-664), and upgrade to versions with the fix (§13).

8. To protect the primary, we set `max_slot_wal_keep_size`. During a long connector outage it tripped
   and the slot was invalidated. What now, and was it the right trade-off?
   > **Direction:** The connector cannot resume and must re-snapshot (with downstream impact), but the source database stayed up — the right trade-off; plan the re-snapshot path (incremental snapshots, idempotent consumers) in advance (§13).

9. Jobs in BullMQ started vanishing without trace — not failed, not completed. We had recently pointed
   BullMQ at the shared cache Redis to save money. Explain.
   > **Direction:** The cache Redis uses `allkeys-lru`, which evicts job, lock and queue keys under memory pressure; BullMQ needs a `noeviction` instance of its own (§16, §2).

10. We replayed two days of clickstream through a Flink sessionisation job. Sessions came out merged and
    split in nonsensical ways compared to live processing. Why is replay different?
    > **Direction:** During replay event time runs far faster than wall-clock, and any processing-time timers (session gaps, idle timeouts) behave differently; use event-time timers throughout and bound replay speed (§17).

11. Our cache invalidation is driven by CDC through Kafka. During an unrelated backlog, invalidation lag
    reached 20 minutes and users saw stale prices. How do you design invalidation that stays fast when
    the pipeline is behind?
    > **Direction:** Invalidation latency equals consumer lag; give invalidations their own topic/consumer group with priority capacity, alert on its time lag, and keep TTLs as the bounded-staleness backstop (§6, §11).

12. Pods keep local caches invalidated by Redis Pub/Sub. After a Redis failover, some pods silently kept
    stale data for days. Why, and how do you fix the protocol?
    > **Direction:** Pub/Sub is at-most-once; messages published while a pod was reconnecting were lost; flush the local cache on every resubscribe, keep short TTLs, or use a versioned stream the pod can resume from (§7, §4).

13. One Kafka broker's disk filled because partition sizes were uneven. Its partitions went offline,
    `acks=all` producers stalled, their buffers filled, and API request threads blocked. Trace the cascade
    and the controls at each step.
    > **Direction:** Monitor per-broker disk and rebalance partitions (Cruise Control), cap `retention.bytes`, and at the producer fail fast with a small `max.block.ms` or decouple via an outbox so broker trouble never blocks request threads (§9, §10, §13).

14. Our payment flow is SQS standard → Lambda → DynamoDB → payment provider. We have seen duplicate charges
    under load despite "idempotent" code. Where can duplicates come from, and how do you make it
    effectively-once end to end?
    > **Direction:** Duplicates arise from standard-queue redelivery, visibility timeouts during slow runs, and whole-batch retries; record an idempotency key with a conditional write before the side effect and pass it to the provider (§15, §8).

15. Our Redis-based rate limiter failed closed during a Redis outage and rejected 100% of traffic, turning
    a cache incident into a total outage. What should the design be?
    > **Direction:** Decide failure modes for control-plane uses of Redis: fail open to a coarse local per-process limiter, with a breaker around Redis and alerts on degraded mode (§5, §4).

16. We run Kafka in two regions with MirrorMaker 2. During a regional failover, consumers in the DR region
    reprocessed hours of data and a few skipped some. Why, and what is realistic?
    > **Direction:** Offsets differ between clusters; MM2's translated checkpoints are approximate and lagging, and the replication tail is lost — expect duplicates, require idempotent consumers, and use `sync.group.offsets.enabled` with alerting on checkpoint lag (§12, §9).

17. A single corrupt record crash-looped our entire Kafka Streams application across 100 partitions,
    because every instance that picked up the task died on it. How do you make the app resilient?
    > **Direction:** The default `LogAndFailExceptionHandler` fails the app; use a deserialisation handler that logs and continues or routes bytes to a DLQ, and add production exception handling (§12, §18).

18. During an incident, we reset a consumer group "to 10:00" using `--to-datetime`, but it reprocessed
    much more — or much less — than expected on some partitions. Why?
    > **Direction:** Time-based reset uses record timestamps (`CreateTime` from producer clocks) to find offsets, so skewed or future timestamps misplace the position; check with `kafka-dump-log.sh` and a `--dry-run` first (§10, §12, §19).

19. A celebrity's post generates 50,000 likes per second. The like counter is a single Redis key, and
    read replicas and local caches have not helped. Why not, and what does?
    > **Direction:** Reads can be replicated and cached, writes cannot — split the counter across N sub-keys on different slots and sum on read, or aggregate in-process and flush `INCRBY` periodically (§5).

20. After a four-hour outage of our notification pipeline, 30 million push notifications are queued —
    "driver arriving", "flash sale ending", password resets. How do you bring it back, and what should the
    system have done by design?
    > **Direction:** Age makes many messages harmful; drop or collapse expired ones (per-type TTLs, age check on consume), prioritise critical types in their own lane, and throttle the drain to what providers and users tolerate (§8, §16).
