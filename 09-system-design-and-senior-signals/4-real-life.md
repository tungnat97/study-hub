[← back to the field index](README.md)

# System Design & Senior Signals · Part 4 — Real-life production problems

Parts 1–3 teach the design round as it is usually taught: scope, estimate, draw, deep-dive. This part
is the other half — the questions where an interviewer describes something that actually happened to
them ("we built X, at scale Y happened, what now?") and listens for whether you have been there.

The good answers here are rarely the textbook ones. They are the counter-intuitive fix, the operational
trick, the trade-off you only learn after being paged, and the judgement call about what *not* to do.

**How to use this file.** Read the pre-knowledge first; it is written to cover every question below.
Then, for each question, answer out loud for two to three minutes as if in the room — clarify, name
the likely cause, propose the fix, name what it costs — *before* reading the direction line. The
direction is a pointer, not a model answer: if yours reached the same insight by a different route,
that is fine. If it did not, reread the section it cites.

Levels run from the questions you will almost certainly be asked (Level 1) to the rare, compound ones
that come up in staff-leaning loops (Level 10).

---

## Pre-knowledge

### 1. Numbers that matter in practice

The textbook latency table (L1 0.5 ns, RAM 100 ns, SSD read 100 µs, cross-continent 150 ms) is
necessary but not what gets you through a production question. These are the numbers you reason with
when something is on fire (`SD02`, `C14`).

**Per-component rough ceilings (single node, sane hardware, 2024–26):**

| Component | Comfortable | Heroic | What breaks first |
|---|---|---|---|
| Postgres primary, simple OLTP writes | 5–10k TPS | 30–50k TPS | WAL fsync, lock contention on hot rows, autovacuum debt |
| Postgres point reads (cached) | 20–50k QPS | 100k+ | connection count (each backend ~5–10 MB; context switching past ~300–500 active) |
| One hot row update | 200–1,000/s | ~2–5k/s | row lock serialisation; every writer queues on one tuple |
| Redis single shard | 100k ops/s | 500k+ with pipelining | single-threaded CPU; one big key (`HGETALL` on 1M fields) stalls everyone |
| Kafka partition | 5–10 MB/s | 50 MB/s | consumer processing speed, not the broker |
| Kafka consumer group | partitions × per-consumer rate | | partition count caps parallelism (`Q14`) |
| Node.js process, JSON API | 2–5k RPS | 10k+ | one CPU; a 50 ms synchronous task blocks everything (`C04`) |
| Load balancer | effectively unlimited | | per-target connection limits, cold targets |
| Third-party payment API | 20–100 RPS per account by default | negotiated | their rate limit, not yours |
| Email/SMS/push provider | 10–100/s per account by default | | provider quotas and per-carrier throttling |

**Latency users feel.** 100 ms feels instant; 1 s breaks flow; 10 s loses the user. Mobile networks
add 50–300 ms RTT and packet loss, so a server-side p99 of 800 ms is 2–3 s on a train.

**Retry amplification.** If each of *n* layers retries *r* times, one failing leaf request becomes
(r+1)^n attempts. Three layers × 3 retries = 64 attempts per user request. This is the single most
important number in overload incidents (§8).

**Fan-out tail.** If a request fans out to *N* backends each with p99 = 100 ms, the probability that
at least one exceeds its p99 is 1 − 0.99^N. At N = 100 that is 63%: the *median* parent request sees a
backend p99. Hence hedged requests, partial results and fan-out caps (§17).

**Utilisation and queueing.** From M/M/1, waiting time scales as ρ/(1−ρ). At 50% utilisation queueing
adds 1× service time; at 80%, 4×; at 90%, 9×; at 95%, 19×. This is why "we hit 85% CPU and latency
tripled" is normal, and why headroom targets sit at 50–70% (§2).

**Availability multiplies.** Three synchronous dependencies at 99.9% each cap you at ~99.7% before
your own failures; 99.99% on top of them is arithmetic, not ambition. Four nines means ~4.3 minutes of
downtime a month — shorter than most people's time to acknowledge a page (`SD07`, `M34`).

**Cost anchors (order of magnitude, public cloud list prices):** a vCPU-month ~$20–40; 1 GB RAM-month
~$3–5; object storage ~$23/TB-month, infrequent-access ~$12, archive ~$1–4; cross-AZ transfer
~$0.01/GB each way; internet egress ~$0.05–0.09/GB; NAT gateway processing ~$0.045/GB; managed
Kafka/Redis often 2–4× self-run compute; a senior engineer-year ~$150–300k fully loaded. The last is
the one most often forgotten in build-vs-buy (§14).

**Useful conversions.** 1M requests/day ≈ 12 RPS average; peak is usually 3–10× average for consumer
traffic and 100×+ for launches and ticket drops. 1 KB × 1B rows = 1 TB. 100M daily users × 10 writes =
1B writes/day ≈ 12k writes/s average. A Postgres table past ~1–2 TB or ~1B rows starts making vacuum,
index builds and backups the real problem, before queries are.

### 2. Capacity planning, headroom and load testing

**Headroom is for the failure, not the peak.** With three AZs you must survive losing one: the other
two take 1.5× load each, so steady-state utilisation must be ≤ ~60% or an AZ loss becomes a capacity
outage. With N+1 on a three-node fleet the ceiling is ~66%.

**Autoscaling is slower than the spike.** Scale-out takes 1–5 minutes (metric window, decision, node
provisioning, image pull, JVM warm-up, readiness). A ticket drop peaks in seconds. Anything spiky needs
scheduled pre-scaling, a buffer (queue, waiting room), or permanent over-provisioning for that tier.
Scaling on CPU also lags when the bottleneck is downstream, and can make things worse: more pods = more
DB connections = the database falls over faster (`DB21`, `O06`).

**Scale the bottleneck, not the stateless tier.** Doubling app pods against a saturated Postgres
doubles the lock contention. Always find which resource is actually saturated — utilisation,
saturation, errors (USE) per resource (`O14`, `O17`).

**Load tests lie in predictable ways (`C15`).** Cache hit rates are unrealistically high (the same
100 test users); data volume is tiny so the planner picks different plans (`DB18`); no background
jobs, compaction or vacuum run; the load generator is closed-loop (waits for responses), so it slows
down when the system does and hides the collapse (coordinated omission). Use open-loop generators,
production-shaped data, and replayed production traffic.

**Capacity planning in practice.** Forecast from the business — launch calendar, marketing pushes,
seasonality (Black Friday, tax deadline, New Year's Eve) — not a linear trend. Keep a per-service
capacity sheet: current peak, tested ceiling, bottleneck resource, and *lead time to add capacity*
(a vertical DB resize is minutes with a failover; a new shard is weeks; a provider quota increase can
be days; hardware in a reserved-capacity region can be months).

**Warm-up.** New JVM pods serve slowly for 30–120 s (JIT); new cache nodes are empty; new replicas have
cold buffer caches. Recovery and scale-out therefore temporarily *reduce* per-node capacity. Use
load-balancer slow-start, warm-up traffic and pre-warmed caches.

**Limits nobody writes down.** File descriptors, ephemeral ports (≈28k by default per source IP →
destination pair; NAT gateways hit this), conntrack tables, Kubernetes pod-per-node limits, cloud API
rate limits (your autoscaler itself gets throttled during a big scale-out), load-balancer target
registration delay, DNS resolver QPS (Kubernetes' default `ndots:5` turns one lookup of an external
name into several), IP addresses in a subnet, Lambda concurrency per account, managed DB
`max_connections`, Postgres transaction-ID age and vacuum lag on high-churn tables. Production outages
at scale are more often one of these than a CPU limit. The fix for NAT port exhaustion is usually
connection reuse (keep-alive pools), not a bigger NAT.

### 3. Hot spots, celebrities and skew

Everything in `SD09`, plus the operational detail.

**Detection before mitigation.** You need per-key visibility: top-N keys by request rate (Redis
`--hotkeys` under an LFU policy, or client-side sampling), per-partition lag in Kafka, per-shard CPU in
the database. A count-min sketch or the space-saving algorithm gives approximate top-K in fixed memory
(`DB33`).

**Hot-key remedies, cheapest first:**
1. **Local in-process cache** with a short TTL (1–5 s) in front of Redis. A key read 100k/s across
   200 pods becomes 200 reads per TTL window. Nearly always the right first move (`Q09`).
2. **Request coalescing / single-flight**: one in-flight fetch per key per process (`Q04`).
3. **Key replication**: store `k#0..k#N`, read a random copy, write all. Spreads a hot read across
   Redis Cluster slots.
4. **Key splitting for writes**: sharded counters (§4), bucketed lists.
5. **Dedicated capacity**: move the whale to its own shard or pool.
6. **Shed or degrade the hot object specifically**: serve it stale, cap its fan-out.

**Celebrity fan-out.** Push (fan-out on write) for normal accounts; pull (fan-out on read) above a
follower threshold, merged at read time (`SD12`). The details the textbook skips:
- The threshold should be **dynamic**: an account going viral crosses it mid-day.
- For large-but-not-celebrity accounts, fan-out is **asynchronous and prioritised**: active followers
  (seen within 7 days) first; inactive followers' timelines are rebuilt lazily on their next login.
  Most follower lists are majority-inactive, so this cuts write volume 3–10×.
- Cap stored timeline length (e.g. 800 IDs); older history comes from pull.
- Deletes and privacy changes must also propagate; pull-based celebrities are *easier* to retract.

**Tenant skew.** One tenant 1,000× the median breaks sharding by tenant. Sub-partition the whale
(tenant + hash of entity), give it a dedicated cell (§11), and apply per-tenant quotas so its batch
job cannot starve others (`M29`, `Q21`).

**Temporal skew.** Everyone does the same thing at the same moment: midnight cron across every
customer, top-of-the-hour notifications, apps that all sync when a push lands, TTLs set at deploy time
that expire together, certificates issued in one batch that all expire on one day. The fix is
**jitter** everywhere: jittered cron start, jittered TTLs (base ± 10–20%), jittered client timers.

**Sequential keys.** Monotonic keys (timestamps, auto-increment) into a range-partitioned store
(Bigtable, HBase, DynamoDB sort-key-heavy designs, Spanner) send every write to the last partition.
Hash-prefix or bit-reverse the key, or use random UUIDs there — while in a B-tree Postgres index the
opposite applies: random UUIDv4 fragments the index and thrashes the buffer cache, so time-ordered
UUIDv7/ULIDs are better (`DB13`).

### 4. Counters, likes and views at scale

A "like" counter on a viral post is a hot row (`SD06`). The ladder:

1. **Sharded counters.** N sub-rows (`post_id, shard, count`); increment a random shard; read by
   summing. N = 10–100 turns 1k/s on one row into 10–100/s per row. Reads get N× dearer, so cache the
   sum.
2. **Buffer and flush.** Increment in Redis (`INCR`, ~100k/s per shard) or in-process, flush
   aggregates every few seconds. Loses up to one interval on a crash; for likes and views that is
   acceptable, and you must say so.
3. **Log then aggregate.** Events into Kafka, a stream job aggregates per window and writes totals
   (`Q17`). Durable and replayable; seconds of lag.
4. **Approximate counting.** HyperLogLog for unique viewers (~12 KB per counter, ~0.8% standard error
   in Redis); count-min sketch for frequencies (`DB33`). Display "1.2M" — nobody can check the last
   digit.

**The idempotency trap.** A "like" is set membership, not a counter: the truth is the `(user, post)`
edge with a unique constraint; the count is derived. Double taps and retries then cannot double count.
The count can be eventually consistent; the membership cannot.

**View counts and money.** Raw views are cheap to inflate; production view counts dedupe per viewer
per window, filter bots, and are sometimes deliberately *delayed or frozen* while verification catches
up. When money depends on the number (creator payouts, ad billing) a delayed correct number beats a
live wrong one — and the billing number comes from a separate, audited pipeline, not the display
counter.

**Reconcile.** Every derived counter drifts (lost flushes, bugs, partial failures). A periodic job
recomputes from the source of truth and corrects it. Build the correction in from day one.

### 5. Feeds, timelines and notifications

- **Timeline storage**: a per-user list of post IDs (Redis sorted set or wide-column row), hydrated at
  read time from a post cache. IDs not content, so edits and deletes are one write.
- **Ranking** is a second stage: candidate generation (a few hundred IDs), then scoring at read time.
- **Pagination** is cursor-based (`A05`); offset pagination on a live feed duplicates and skips.
- **Notifications** fan out like feeds but have *delivery* semantics: dedupe keys per (user, event),
  collapse bursts ("Anna and 41 others liked…"), quiet hours, per-user caps, and transactional traffic
  (password reset, 2FA) on separate queues *and separate provider accounts* from marketing — so a
  campaign cannot delay a login code.
- **The push-induced herd**: a push to 10M users makes a large fraction open the app within a minute
  or two. Stagger sends over 10–30 minutes, put the content in the push payload so the app need not
  fetch, and pre-warm the read path.
- **Unread counts** are per-user counters with a hot spot for heavy users; cap at "99+" so you never
  need the exact number beyond it.
- **Presence / "who's online"** is expensive and rarely valuable at scale: approximate it, scope it to
  friends, or delete it.

### 6. Booking, inventory and contention

From `SD10`, the patterns that survive real traffic.

**Reservation with TTL.** Claim → hold (5–15 min) → pay → confirm, or expire. The claim is an atomic
conditional update (`UPDATE seats SET held_by = $1, held_until = now() + interval '10 min' WHERE id =
$2 AND (held_by IS NULL OR held_until < now())`) or Redis `SET key val NX PX 600000`. Expiry is
enforced **in the claim condition**, not only by a sweeper — sweepers fall behind exactly when load
is highest.

**Inventory as tokens, not a counter.** For general admission (10,000 identical tickets) a single
`remaining` counter is a hot row. Split stock into buckets (e.g. 20 × 500) or pre-create one row per
unit, and claim with `SELECT … FOR UPDATE SKIP LOCKED LIMIT 1` (`DB09`). Contention drops by the bucket
count; the last few units need a sweep across buckets.

**Queue-based admission / virtual waiting room.** When demand is 100× supply, do not let everyone reach
the booking system. Queue users at the edge (a static CDN-served page plus a small token service),
admit at the rate the backend sustains (say 500 users/minute), and give each admitted user a signed,
short-lived token that the booking API checks. Properties:
- Absolute backend protection — throughput is set by you, not the crowd.
- Explicit fairness: random position assignment at opening time beats "whoever refreshed fastest",
  which rewards bots.
- The waiting page is static, so it survives millions of refreshes.
- It turns an outage into a *wait*: a much better experience and a much better press story.

**Oversell deliberately — or never.** Airlines oversell and compensate; concerts must not. Sometimes
the right answer is "allow a 1–2% oversell with a compensation flow", because strict correctness costs
more than the rare apology. Know which business you are in.

**Locks do not give exclusion; the database does.** A Redis lock with a TTL fails whenever the
holder pauses longer than the TTL (GC, VM stall, network): two holders act at once. Correctness must be
enforced at the write — a unique constraint on `(room_id, night)`, a conditional update, or a fencing
token the storage checks (`C11`, `DB05`). Treat the lock as an efficiency optimisation only. The same
applies to caps like "first 1,000 redemptions": pre-allocate 1,000 claimable rows, or use an atomic
conditional increment, never count-then-insert.

**Holds are an attack surface.** TTL holds let bots hoard inventory and release it at the last second.
Limit holds per account/device/card, require a queue token, shorten TTLs under attack (`S13`).

### 7. Payments correctness

From `SD10` and `DB38`, what production teaches.

**Exactly-once is an illusion; effectively-once is the goal.** Delivery is at-least-once everywhere;
correctness comes from idempotency at every hop (`M16`, `A09`). A payment carries an idempotency key
generated **by the client before the first attempt**, stored with the outcome — and passed to the
provider too (Stripe's `Idempotency-Key`, a merchant reference elsewhere), so a retry after a timeout
cannot charge twice.

**The unknown outcome.** A provider timeout means *unknown*, not *failed*. Never mark it failed and let
the user retry with a new key. Mark it pending, resolve by querying the provider by your reference, by
webhook, or by reconciliation. The user sees "processing".

**State machines.** Payments are explicit state machines (`created → authorised → captured →
refunded`, plus `failed`, `unknown`), with transitions enforced by conditional updates (`WHERE state =
'authorised'`). Webhooks arrive late, duplicated and out of order; store each raw webhook first (inbox,
`M15`) and apply it only if the transition is valid.

**Double-entry ledger.** Money moves as balanced entries (debits and credits summing to zero),
append-only, never updated (`DB38`). Balances are derived, with snapshots for speed. Corrections are
new reversing entries. Integer minor units, currency on every amount, no floats — and minor units
differ per currency (JPY has none, KWD has three), so the exponent comes from the currency, and FX
rates are recorded per transaction, never recomputed at display time.

**Kafka "exactly-once" has a boundary.** Transactions give exactly-once for read-process-write
*inside* Kafka. An email, an HTTP call or a DB write outside the transaction is still at-least-once;
the side-effecting consumer needs its own dedupe key (order ID + message type) (`Q15`).

**Batch money jobs** (subscription renewals, payouts) must be idempotent per (customer, period) with
a unique constraint, resumable from a checkpoint, and pass provider keys derived deterministically
from the period — so a crashed and restarted run cannot charge twice.

**Order before charge.** Create the order in a pending state first, charge with the order's key,
then confirm; a sweeper resolves orphans against the provider. 2PC across an external API is not
available, so the intent record is the substitute (`M13`).

**Reconciliation is the real correctness mechanism.** Daily (or hourly), match the ledger against the
provider's settlement report and the bank statement — three-way reconciliation. Every mismatch becomes
a case: missing on our side, missing on theirs, amount differs. Mature teams run the mismatch rate as
an SLO. It catches what idempotency cannot: provider bugs, manual refunds in the provider dashboard,
FX rounding, chargebacks, fees.

**Multiple providers.** Routing between providers (cost, authorisation rate, failover) means one
logical payment can have attempts at two providers. Each attempt has its own provider-side key and
record; the logical payment is captured at most once, enforced by a unique partial index on
`payment_id WHERE state = 'captured'`. **Failing over on timeout is dangerous**: the first provider may
have succeeded. Fail over only on a definitive decline or a pre-request error (connection refused, DNS
failure); on a timeout resolve first, or accept the rare double authorisation and void one.

**Authorise, capture, void.** Authorise at checkout, capture at fulfilment; uncaptured authorisations
expire (commonly ~7 days for cards), so late fulfilment needs re-authorisation. A double authorisation
is recoverable (void); a double capture is a refund, a support ticket and possibly a chargeback fee.

**Outbox for side effects.** "Charge succeeded → update order, send receipt, emit event" uses the
transactional outbox so the state change and the event are atomic (`M15`, `Q18`).

**Money in distributed flows.** Wallet top-ups, payouts and transfers are sagas (`M14`) with
compensations; a payout that has left the bank cannot be compensated, only reversed by a new
transaction — so the irreversible step goes last, after every check.

### 8. Retry storms, metastable failures and thundering herds

**Metastable failure.** A system is fine at load L, gets a trigger (a DB blip, a deploy, a cache
flush), and then *stays* broken at load L after the trigger has gone. The sustaining effect is work
amplification: timeouts cause retries, retries raise load, load causes timeouts. Classic loops:
- Client retries (the (r+1)^n amplification of §1).
- The server still processing requests whose clients have timed out — work nobody will read
  (goodput collapse).
- A cache flush: DB slows, fills take longer, more misses.
- Queue backlog processed FIFO: consumers work on stale messages whose users already gave up.
- GC death spiral: load → heap → longer pauses → timeouts → retries.
- Connection storms: timeouts close connections, reconnects cost TLS handshakes and auth, the
  handshakes themselves saturate CPU.
- Redelivery: a queue's visibility timeout (or ack deadline) shorter than processing time under load
  makes messages reappear and be processed twice, doubling load and slowing processing further. Extend
  visibility by heartbeat and cap in-flight work.
- Cache-hit-rate cliffs: a 95% → 80% hit rate is 5% → 20% misses, i.e. 4× backend load. A deploy that
  changes the cache key format is a full cache flush in disguise; roll it out gradually or read old
  keys as a fallback.

**Breaking the loop** means reducing load *below* the trigger point, not back to it: stop retries
(budgets, breakers), shed (§9), drop expired work (deadline propagation, `M09`), and switch backlogged
queues to **LIFO or drop-oldest** so fresh requests succeed while stale ones are discarded. Often the
fastest recovery is to block traffic at the edge entirely, let queues drain, then readmit gradually.

**Retry budgets.** Instead of "retry 3 times", cap retries at ~10% of requests per client (the approach
in Finagle and gRPC retry throttling). Normally retries still happen; under overload the budget runs
out and retries stop by themselves. Retry at one layer only — usually the outermost that can.

**Thundering herd on recovery.** When a dependency returns, everyone waiting hits it at once: breakers
across 500 pods close together; clients with fixed backoff retry on the same second; a cache warms
from empty; mobile clients reconnect. Fixes:
- **Jittered exponential backoff** — full jitter: `sleep = random(0, min(cap, base × 2^attempt))`.
- **Gradual readmission**: ramp traffic 1% → 10% → 50% → 100% via a flag or LB weights.
- **Pre-warm the cache** from a snapshot or hot-key list before readmitting traffic.
- **Half-open breakers with a probe fraction**, not an all-at-once close.
- For WebSockets: server-supplied reconnect delays and staggered reconnection windows.

**Deadline propagation.** Every request carries an absolute deadline; each hop checks it before doing
work and drops the request if it cannot finish in time. This kills goodput collapse at the root.

**The counter-intuitive rule.** During overload, *reducing* timeouts and retries often restores
service, and adding capacity alone often does not: the new capacity is eaten by the amplification, and
new cold nodes add connection storms.

### 9. Load shedding and graceful degradation

**Shed early and cheaply.** Rejecting must cost almost nothing: shed at the load balancer or the first
middleware, before auth, deserialisation or DB calls. Good signals: requests in flight (Little's Law,
`C14`) and queue wait time, better than CPU; adaptive concurrency limits (gradient/Vegas-style, as in
Netflix's concurrency-limits library) track the healthy limit automatically (`M10`, `M30`).

**Prioritised shedding.** Tag requests with a priority — critical (checkout, login), normal (browse),
sheddable (recommendations, analytics beacons, prefetch) — and drop the lowest tier first. Health
checks and control-plane traffic must never be shed, or the orchestrator kills healthy-but-busy pods
and makes it worse.

**Brownout / feature degradation.** Predefined degradation levels controlled by flags (`O19`): turn off
personalisation and serve popular items; disable search facets; stop real-time stock checks and show
"usually in stock"; freeze comment counts. Each is a product decision agreed *before* the incident: a
list with owners.

**Static fallbacks.** A pre-rendered, CDN-hosted copy of critical pages (home, product, status),
regenerated periodically. If the origin fails, the CDN serves stale (`stale-if-error`,
`stale-while-revalidate`, `A04`, `Q09`). Retailers survive peak outages because the catalogue is
static and only the cart is dynamic.

**Fail open vs fail closed**, per dependency, written down. Rate limiter or flag service down → fail
open with safe defaults. Authorisation down → fail closed. Fraud scoring down → degrade: allow
small-value payments, queue large ones for review.

**Bulkheads.** Separate thread and connection pools per dependency so a slow recommendations service
cannot take the connections checkout needs.

**Fallbacks must be cheaper than the primary.** A timeout that falls back to a broader, more expensive
query (or a second service) amplifies overload. Fallbacks come from cache, static data or a smaller
computation, and share the same deadline budget.

**Logging and telemetry are dependencies too.** Synchronous logging to a full pipeline blocks
request threads; agents filling node disks get pods evicted. Use bounded, non-blocking buffers that
drop (and count the drops), rate-limit log volume per service, and sample (`F08`, `O13`).

**Slow is worse than dead.** A node that is slow but passing health checks keeps receiving traffic,
and with least-connections balancing it can look "less loaded" once its queue fills and clients give
up. Use outlier detection that ejects on latency and error rate, not only on health checks (`M07`).

**Liveness probes that check dependencies are a trap.** If liveness fails when the DB is slow,
Kubernetes restarts every pod at once during a DB blip, turning a slowdown into a full outage and a
cold-start herd. Liveness checks the process; readiness may check dependencies, carefully (`O06`).

### 10. Rate limiting and fairness, globally

From `SD11` and `A08`:

- **Local vs global.** An exact global counter in one Redis is a hot key and a single point of failure.
  Production systems mostly run **local limits with periodic sync**: each node enforces roughly
  `limit / node_count`, or leases tokens from a global budget in chunks (take 100 at a time). Accept a
  few percent over-admission for a lot of resilience.
- **Fail open** when the limiter store is down, with a coarse local fallback.
- **Layered limits**: edge (per IP/device, anti-abuse, `S13`), gateway (per API key/tenant, the
  commercial quota), service (per dependency, protective). Different questions, different layers.
- **Cost-based limits**: a search is not a GET by ID. Charge in cost units (GraphQL query cost, `A15`).
- **Fairness**: per-tenant concurrency caps and weighted fair queuing for background work (`Q21`) —
  round-robin across tenants rather than FIFO across all jobs, so one tenant's million-job import does
  not delay everyone else's single job.
- **Outbound limits** (a provider allowing 100 RPS) need a *global* token bucket or one queue with a
  fixed-rate consumer, plus priorities so marketing does not consume the transactional budget.
- **Return 429 with `Retry-After`** and make clients honour it; your own mobile app is often the worst
  offender, and its release cycle means a bad retry policy lives for months. Keep a server-controlled
  kill switch for client behaviour (remote config for polling intervals and retry caps), minimum-
  version gates, and the ability to shed by app version at the edge.
- **Window shape matters.** Fixed per-minute windows allow 2× bursts at the boundary and per-minute
  dashboards hide per-second bursts; token buckets or sliding windows match what clients experience.
  Show customers usage at the granularity you enforce.
- **Polling is a rate problem**: conditional requests (`ETag`/`If-None-Match` → 304) make most polls
  nearly free, and a server-driven interval lets you slow everyone down during an incident (`A04`).
- **Per-destination isolation for outbound retries**: webhook delivery keeps a queue or concurrency
  cap and a breaker *per endpoint*, parks persistently failing ones, so one customer's dead endpoint
  does not delay everyone's deliveries (`A16`, `Q16`).
- **Quota per key, not per account**: scoped API keys with their own sub-quotas and anomaly alerts,
  so a leaked key exhausts only itself and can be rotated without locking the customer out (`A18`).

### 11. Cells, shuffle sharding and blast radius

**Cell-based architecture** (`M31`): the whole stack (app, DB, cache, queues) is replicated into
independent cells, each serving a subset of customers; a thin routing layer maps customer → cell. A bad
deploy, poison message or runaway tenant takes out one cell (say 5% of customers), not all. Deploy cell
by cell, canary cell first. Cells have a fixed maximum size, so scaling means adding cells, and the
biggest cell is always a size you have tested.

Costs: the router is now critical (keep it dumb, cached, statically stable); cross-cell queries
(global search, admin reports) need a separate aggregation path; moving a customer between cells is a
migration (§12); per-cell fixed overhead raises cost.

**Shuffle sharding.** Give each customer a random subset of k workers out of N. With N = 100, k = 5
there are ~75M combinations, so a poison customer that kills its five workers fully affects only
customers sharing all five — almost nobody — and partially affects a few whose other workers still
serve. Used by AWS (Route 53 among others) to isolate noisy or malicious tenants.

**Static stability.** The data plane keeps working in its current state when the control plane is
down: cached config, no per-request dependency on the control plane. Classic big outages come from data
planes that call a control plane per request (metadata, discovery, config, auth), which then falls over
at recovery time when everything restarts and asks at once.

**Zonal isolation.** Keep traffic within an AZ (zone-aware routing): cuts cross-AZ cost and makes AZ
failure clean — you evacuate one zone rather than every request touching every zone.

**The shared dependency nobody drew.** Cells fail together when they share a DNS provider, an identity
provider, a secrets manager, a CI/CD pipeline that deploys all cells at once, a global config push, or
one TLS certificate. Blast-radius reviews list these explicitly.

### 12. Migrations under live traffic

From `SD14`, `DB22`, `M28` — the patterns and their real traps.

**Expand/contract** (parallel change): add the new schema (expand), write both, backfill, switch reads,
stop writing the old, remove the old (contract). Each step independently deployable and reversible.
The trap: steps that look reversible but are not — once you drop the old column or stop dual writing,
rollback needs a reverse backfill.

**Dual writes vs CDC.** Application dual writes are simple but not atomic: the second write fails, or
two concurrent writes land in different orders, and the stores diverge silently. **CDC** (read the old
store's WAL/binlog via Debezium, apply to the new, `Q18`) gives one ordered source of truth. Prefer CDC
for data migrations; if you must dual write, make one store the source of truth, write the other
asynchronously, and run a verifier.

**Shadow reads.** Read from both, serve the old result, compare asynchronously, log mismatches with
enough context to classify them. Burn the mismatch rate to zero or to known, explained classes before
switching. Classes you always find: timestamp precision, collation and sort order, NULL vs empty
string, float rounding, time zones, and replication lag (re-read on mismatch after a delay before
counting it).

**Dark launch.** Run the new path in production on real traffic with its result discarded. Validates
load and correctness at once. Beware side effects: a dark-launched path must not send emails or charge
cards — stub the effectful calls.

**Backfill with verification.** Batches by primary-key range (never OFFSET), throttled by replica lag
and DB CPU, resumable from a checkpoint, idempotent (upserts), and aware of live writes — the backfill
must not overwrite newer data (compare a version or `updated_at`, or start CDC from the snapshot
position). Verify with row counts plus per-range checksums (hash of sorted rows per 10k-ID range), so a
mismatch can be localised and re-copied.

**Strangler fig.** Route by endpoint or feature through a facade; move one slice at a time; the old
system shrinks until deleted. The trap: the last 10% (reports, admin tools, a cron nobody owns, a
partner integration on an undocumented endpoint) takes as long as the first 90%. Inventory every
consumer — database access logs and network flow logs are the truth, not the architecture diagram.

**Cutover.** Flag-controlled, per tenant or per percentage, reversible. When a moment of consistency
is needed: a seconds-long write freeze per tenant rather than a global outage; or an ordered handover
(drain CDC lag to zero, flip the flag, confirm). Keep the old path warm with reverse replication for a
rollback window of days to weeks.

**Big-bang is sometimes right.** Small data, a maintenance window measured in minutes, a rollback that
is "restore the snapshot": a planned 30-minute downtime can be far less risky than three months of dual
running. Saying so is a senior signal.

**Migration recipes worth having ready:**
- **Splitting a shared database**: give each table one owner first, replace cross-boundary joins with
  API calls or replicated read models, drop cross-boundary foreign keys, *then* split physically. The
  logical split precedes the physical one (`M22`, `M23`).
- **State off the stateless tier** (uploads on local disk, in-memory sessions): move to object storage
  with direct signed uploads (`F17`); migrate lazily (read-through: on miss, copy from the old
  location) plus a background backfill for the long tail.
- **Re-keying or re-encrypting live data**: store a key or format version per record, read all
  versions, write the new one, backfill in throttled checkpointed batches, retire the old version when
  counts per version reach zero (`S10`, `S11`).
- **Password hashes to a new identity provider**: lazy migration on login (verify the old hash,
  create the new credential), then force resets only for the inactive remainder (`S02`).
- **Changing a Kafka partition key**: a new topic with the new key, bridged by a stream job; move
  consumers one at a time at a recorded offset boundary; retire the old topic (`Q14`, `Q19`).
- **Search index rebuilds**: build into a new index and swap an alias atomically; keep the full
  rebuild as the recovery path even after moving to CDC-driven incremental updates, and version
  documents so late updates cannot overwrite newer ones (`DB31`).
- **Flag bucketing**: bucket by the entity that must see a consistent result (customer, cart, tenant),
  never per request — per-request bucketing gives one customer two prices (`O19`).
- **Finding unknown consumers**: DB audit logs and `pg_stat_statements` by role, network flow logs,
  credential usage; then revoke in stages (read-only, then timed brownouts) so unknown consumers
  surface before the final cut (`DB39`).

**Rewrites.** A from-scratch rewrite of a stable, revenue-critical system discards years of
encoded edge cases and runs two systems for longer than planned. Prefer strangling the painful parts;
measure the real maintenance cost before arguing from "unmaintainable" (`F27`).

**Hidden dependencies on the old behaviour** (Hyrum's law): ordering of results, default sort,
implicit timezone, error message strings parsed by a client, an ID format assumed to be numeric. Shadow
comparison finds some; contract tests (`A22`) and consumer inventories find the rest.

### 13. Multi-region, active-active and data residency

From `SD13`, `M33`, `O20`.

**Establish the driver**: latency (users far away), availability (surviving a region), or regulation
(residency). Each leads to a different design; most companies need one and should not pay for all
three.

**Home-region routing.** Each user or tenant has a home region where their data is authoritative;
requests route there (a global directory, or the region encoded in the ID or token). Local writes for
most users, no multi-master conflicts. This is the most common "active-active" in practice: active-
active *across users*, single-writer *per user*.

**True multi-master** needs conflict resolution: last-writer-wins (loses data silently; clock skew
chooses the winner, `M21`), application merge, or **CRDTs** — data types designed to merge
deterministically (G-counters, PN-counters, OR-sets, LWW registers). CRDTs suit counters, carts,
presence and collaborative editing; they cannot enforce invariants like "balance ≥ 0" or "one booking
per seat", which need a single authority (route those operations to one region, or use consensus-based
distributed SQL, `DB35`, and pay cross-region latency on them only). Naive merging of a replicated
set resurrects removed items (the "deleted item comes back in the cart" bug); an OR-set tags each add
so removes win against the adds they observed. For inventory across regions, pre-allocate stock
quotas per region and rebalance, rather than merging a counter.

**Single-region assumptions that break at failover**: database sequences (two regions issue the
same ID), cron jobs that assume one instance (run twice or not at all), distributed locks held in the
failed region, and caches warm only in one region. Fix with region-safe IDs, leader-elected schedulers
with fencing (`M20`), and database-enforced idempotency for anything that moves money.

**Cross-region workflows** (an EU buyer purchasing from a US seller) are sagas (`M14`): the
reservation happens in the authoritative region for the inventory, the buyer's personal data stays in
theirs, and only the minimum (order ID, amount, pseudonymous IDs) crosses.

**Right to erasure across systems**: a data map of every copy (search, caches, analytics, backups,
partner feeds), a deletion workflow that fans out and verifies per system, and crypto-shredding —
per-customer encryption keys deleted to make immutable copies such as backups unreadable (`DB37`).

**Replication lag at failover.** Async replication means failover loses the last seconds of writes
(RPO > 0). Failback is harder than failover: the old primary holds writes the new one lacks. Plan the
reconciliation, not just the switch.

**Failover is a decision, not a reflex.** Automated regional failover on health checks causes more
outages than it prevents (flapping, split-brain, failing into a region without capacity). Mature teams
automate the mechanics, keep a human decision on the trigger, and rehearse with game days. Also check
capacity: the surviving region must have headroom (and quota) for 2× traffic.

**Data residency** means where data is stored *and processed* — including backups, logs, traces,
analytics copies, support tooling and sub-processors. Practical design: region-pinned cells, per-region
encryption keys, a global service holding only non-personal routing data, pseudonymised IDs for global
analytics. The usual leak is logs and traces shipped to a central observability vendor, and support
staff in another country querying production (`S12`).

**Global uniqueness** (usernames, emails) across regions needs one authority or a reservation
protocol; eventual consistency does not work for it.

### 14. Cost-aware design, build vs buy, reversibility

**Cost is a design constraint.** Estimate the bill the way you estimate QPS. The hidden costs that
dominate real bills: cross-AZ traffic (chatty services across zones), NAT gateway processing, log
ingestion (every request body at 1M RPS), metrics cardinality (a `user_id` label), idle
over-provisioned databases, internet egress, and per-request pricing of managed services at high
volume (`O18`).

**Unconventional cost moves:** sample logs and keep full logs only for errors and a trace sample; tier
storage; move analytics off OLTP replicas to columnar storage (`DB32`); batch small writes; compress on
the wire; spot capacity for stateless and batch work; keep traffic in-zone; delete data (retention is
the cheapest optimisation); and put a price on a feature so product can decide if it is worth it.

**Build vs buy.** Buy when it is not your differentiator and the vendor is mature; build when it is
core, when vendor limits are a hard ceiling at your scale, or when the per-unit price at your volume
exceeds a team's cost. Compare total cost: licence + integration + migration-out vs engineers × years
+ on-call + opportunity cost. Always ask "what is our exit if the vendor doubles the price or
disappears?" and keep an abstraction at the seam you might move.

**One-way and two-way doors.** Most decisions are reversible (a library, a cache, an internal API):
decide fast. Irreversible ones — public API shape, the data model of a core entity, ID format, the
primary database, customer contracts, deleting data — deserve the design doc, the review and the
prototype. Seniority is spending effort asymmetrically, and turning one-way doors into two-way doors
(flags, versioned APIs, reversible migrations, keeping the old path alive).

**Boring technology.** Every new technology spends from a limited innovation budget: new failure modes,
no runbooks, nobody on-call who knows it. Postgres + Redis + a queue covers a startling range of scale.
Choose the novel tool only where it solves a problem the boring one cannot.

### 15. Incident command and communication

**Roles.** Incident commander (coordinates and decides; does *not* debug), technical lead(s) (debug and
act), communications lead (status page, stakeholders, support), scribe (timeline). On a small team one
person holds two roles, but the IC must not be heads-down in logs.

**Priorities, in order:** stop the bleeding, restore, then understand. If anything changed recently,
roll back first — "roll back, then investigate" beats "investigate, then roll back". Fast, safe
mitigations (roll back, fail over, shed, disable a feature, scale up) come before root cause.

**Cadence.** Declare early (declaring and downgrading is cheap; declaring late is expensive). Update on
a fixed cadence (every 15–30 minutes) even with no news: what we know, what we are doing, when the next
update is. Customer messages describe impact in customer terms, not service names. Never speculate on
root cause externally.

**Severity** is defined by customer and business impact, not by how interesting the bug is.

**Postmortems.** Blameless; a timeline; contributing factors (plural — there is rarely one root
cause); what went well; action items with owners and dates, weighted towards *detection* and
*mitigation* (so next time is shorter), not only prevention. Track completion: unfinished actions are
how the same incident happens twice.

**Error budgets as a negotiation tool** (`M34`): when the budget is spent, reliability work takes
precedence over features — agreed with product in advance, so it is policy, not an argument.

**Hard calls you should be able to reason about:** data loss vs downtime (restore with an hour lost,
or stay down six hours to recover everything?); partial correctness (serve stale balances, or show
nothing?); who to wake; when to escalate to leadership; when to tell customers; when to notify a
regulator (GDPR breach notification within 72 hours, `S12`, `S16`).

**Incident anti-patterns:** thirty people in one call; five people making changes at once without
announcing them; a fix deployed without anyone recording it; the most senior person debugging while
nobody coordinates; closing the incident when the graph recovers rather than when the backlog has been
reconciled.

### 16. Design docs, RFCs, disagreement and influence

**A design doc that gets read** is short at the top: context, problem, goals and *non-goals*, the
recommendation, alternatives considered (and why not), risks, rollout and rollback, open questions.
Reviewers check the non-goals and rejected alternatives first.

**Agree before the meeting.** Walk key stakeholders through it individually; the review ratifies.
Surprise objections in a large meeting kill good proposals.

**Disagreement.** Separate facts from preferences; settle facts with a cheap experiment or a time-boxed
prototype; make decision criteria explicit ("this year we optimise for time to market over cost");
disagree and commit once decided, and record it (an ADR) so it is not relitigated. Escalation is a
tool, not a failure — escalate *together*, with a shared framing.

**Pushing back on a deadline or design.** Do not say "no"; present options with costs: "we can ship on
the date with X cut, ship everything two weeks later, or ship on the date with risk Y, which I would
not accept for payments." Put the risk in business terms (money, customers, legal), then let the
accountable person decide — and write the decision down.

**Trade-off communication.** State it in one sentence ("we accept a counter up to 5 seconds stale to
avoid a hot row"), quantify it, name who is affected, and name the trigger that would make you revisit.

**Saying "we should not build this".** The highest-leverage senior move is often deletion: the feature
nobody uses, the microservice that should be a module, the real-time pipeline that could be a nightly
batch. Back it with data (usage, cost, incident history).

**Influence without authority.** Offer to do the unglamorous part (the migration tooling, the
runbook); make the right thing the easy thing (a library, a template, a lint rule); measure and
publish (a dashboard of who is on the old version). Mandates without paved roads stall.

### 17. Unconventional techniques catalogue

Techniques that show up in real answers and rarely in textbooks:

- **Request hedging**: send a second request to another replica if the first has not returned by the
  p95; take the first reply. Cuts tail latency sharply for ~5% extra load. Idempotent reads only
  (Dean and Barroso, *The Tail at Scale*) — and always with a budget (e.g. ≤ 5% of requests), because
  unbudgeted hedging doubles load exactly when the backend is slowest.
- **Probabilistic early refresh** (XFetch): refresh a key before expiry with a probability that rises
  as expiry nears, so there is never a synchronised miss (`Q04`).
- **Serve stale on error** and **stale-while-revalidate** at every cache layer.
- **Negative caching** of "not found" with a short TTL, and a Bloom filter in front for keys known
  absent (`DB33`), so a missing-key storm cannot reach the DB.
- **Admission control by token** (waiting room) instead of scaling for peak.
- **LIFO under overload** for queues whose users time out.
- **Drop by deadline**: discard requests whose deadline has passed before doing any work.
- **One-box and canary-by-cell/tenant**: a single instance with the new build before any percentage
  rollout; internal tenants first.
- **Constant work**: design the system to do the same work regardless of load (push the full config
  every cycle instead of deltas), so there is no mode switch under stress. A system that always does
  the full job cannot be surprised by a spike.
- **Poison-pill quarantine**: a message that repeatedly crashes consumers is parked automatically
  after N attempts, so one bad record cannot halt a partition (`Q16`).
- **Write-ahead intent records**: before calling an external system, persist "about to do X with key
  K", so a crash leaves a record to reconcile rather than a mystery.
- **Soft deletes with delayed purge**, so deletes are recoverable for N days.
- **Kill switches** for every expensive feature, exercised regularly so they work when needed.
- **Synthetic probes** performing real journeys (log in, add to cart, pay a test amount) every minute,
  so you detect breakage before customers do (`O16`).
- **Game days and chaos**: rehearse failover, cache loss, region loss; an untested runbook does not
  work.
- **Per-request cost attribution**: DB time and CPU per tenant or endpoint, so you can see who is
  expensive.
- **Heat-adaptive strategy**: precompute at write time for read-heavy objects, compute at read time for
  write-heavy ones, and switch per object as its heat changes (the celebrity hybrid, generalised).
- **Tolerate and compensate** instead of preventing (oversell and apologise; a rare duplicate email)
  when prevention costs more than the rare fix.
- **Shrink the scope of coordination**: per-tenant, per-cell, per-shard locks and sequences instead of
  global ones.
- **Delete the feature**: the cheapest scaling fix is removing the "total users online" counter.

---

## Questions

### Level 1 — Everyday scaling pain

The questions nearly every senior loop asks: something got slow or fell over under growth, what now.

**1.** Our API was fine at 2k RPS; after a marketing campaign we hit 6k and p99 went from 120 ms to
4 s, while app CPU sat at 45%. The team's first instinct was to double the pods. What do you check
before agreeing, and what do you expect to find?

> **Direction:** Find the saturated resource with USE per component — usually DB connections or a pool, not app CPU — and note that more pods means more connections and makes it worse; §2, §1, `O14`, `DB21`.

**2.** We run Postgres with a read replica. Users complain that after editing their profile the page
shows the old value "about half the time", and it got worse since traffic grew. What is happening and
how would you fix it without routing all reads to the primary?

> **Direction:** Replica lag breaks read-your-writes; route that user's reads to the primary for a short window after a write (or by LSN/session token), not globally; §2, `SD08`, `DB23`.

**3.** Our product page cache is Redis with a 60 s TTL. Every minute, on the minute, DB CPU spikes to
100% for about five seconds and p99 jumps. Nobody changed anything. Explain the graph and fix it.

> **Direction:** Synchronised expiry of hot keys causing a stampede; jitter TTLs, single-flight the refill, or refresh early probabilistically; §3, §17, `Q04`.

**4.** A Redis node in our cache cluster failed over and came back empty. The database, which normally
serves 5% of reads, went down within 90 seconds and stayed down after Redis recovered. What would you
have designed differently, and what do you do right now?

> **Direction:** A cache that carries load is a capacity dependency; now: shed and readmit gradually while warming; design: local cache tier, coalescing, DB sized or protected for a cold cache; §8, §9, `SD05`, `Q04`.

**5.** Our nightly report job runs against the production primary. It used to take 20 minutes; now
it takes three hours and during that window checkout latency doubles. The business wants the report
by 7am. What do you do?

> **Direction:** Move analytics off the OLTP primary (replica, or better a columnar copy fed by CDC) and throttle; the report is a different workload, not a tuning problem; §14, §12, `DB32`, `Q18`.

**6.** We autoscale on CPU. During a flash sale traffic went 10× in 40 seconds and we had a 6-minute
outage before autoscaling caught up — by which time the sale was over. How do you design for this
next time?

> **Direction:** Autoscaling lags spikes by minutes; pre-scale on the known schedule, add admission control or a waiting room, and warm up capacity before the event; §2, §6, `C15`.

**7.** Our Node.js service's p99 is fine most of the day but every few minutes jumps to 2 s across
every endpoint at once, including health checks. CPU is modest. What is your hypothesis and how do you
prove it?

> **Direction:** A synchronous CPU-bound task (big JSON parse, sync crypto, regex) blocking the event loop; measure event-loop lag and move the work off the loop; §1, `C04`, `C13`.

**8.** We added an index to speed up a slow query and it helped. A month later the same query is slow
again, and EXPLAIN in staging shows the index being used, but production is doing a sequential scan.
What is different about production?

> **Direction:** Data volume and distribution change plans; stale statistics or a skewed parameter make the planner choose differently — staging data is not production-shaped; §2, `DB18`, `DB19`.

**9.** A load test showed our service handling 20k RPS comfortably. In production it collapsed at
8k. List the ways the load test lied to us.

> **Direction:** Unrealistic cache hit rates, small data, no background jobs, and a closed-loop generator hiding collapse (coordinated omission); §2, `C15`.

**10.** We have 400 app pods, each with a connection pool of 20, pointed at one Postgres. After a deploy
everything timed out on DB connections even though the DB was mostly idle. What happened and what is
the long-term fix?

> **Direction:** 8,000 potential connections exceed `max_connections` and cost memory per backend; put a pooler (PgBouncer, transaction mode) in front and size pools from Little's Law, not per pod; §1, §2, `DB21`, `C14`.

**11.** Our p50 latency is 40 ms and flat, but p99 crept from 300 ms to 1.8 s over three months while
traffic grew 30%. Management only looks at the average. How do you make the case, and where do you
look?

> **Direction:** Tail latency grows non-linearly with utilisation and fan-out; explain with ρ/(1−ρ) and 1−0.99^N, then look at queueing, fan-out and GC; §1, §2, `C14`, `M34`.

**12.** Our homepage calls 14 backend services and waits for all of them. When any one is slow, the
whole page is slow. We were asked to "make every service faster". What would you propose instead?

> **Direction:** Fan-out multiplies tail probability; set per-call deadlines, render partial results, hedge idempotent reads, and precompute the page; §1, §17, §9, `M10`.

**13.** A single bad deploy of a config value took down every region at once last week. The config was
valid JSON; it just set a timeout to 0. What changes to the delivery design do you make?

> **Direction:** Treat config as code: validate semantics, roll out progressively by cell/region with bake time, and keep a fast rollback; blast radius beats better review; §11, §17, `O19`, `O09`.

**14.** Our search endpoint gets a burst of queries for product IDs that do not exist — a scraper
enumerating IDs — and each one misses the cache and hits the database. How do you stop it hurting us
without blocking legitimate traffic?

> **Direction:** Negative-cache misses with a short TTL, a Bloom filter for known-absent keys, and per-client limits at the edge; §17, §10, `DB33`, `S13`.

**15.** Our Kafka consumer lag grows every evening from 0 to 2 hours and recovers overnight. We have 12
partitions and 30 consumer instances. The team wants to add more consumers. Will it help?

> **Direction:** Parallelism is capped by partition count — 18 consumers are idle; increase partitions (with key-ordering caveats) or make per-message processing cheaper or batched; §1, `Q14`, `Q15`.

**16.** Our mobile app polls `/notifications` every 30 seconds. We now have 5M daily users and that
endpoint is 70% of our traffic. What do you do?

> **Direction:** Push or long-lived connections for the few who need immediacy, server-controlled polling interval with jitter and backoff via remote config, and cheap conditional responses (ETag/304); §10, §5, `A04`, `A16`.

**17.** Our service's memory grows steadily and pods are OOM-killed about every 18 hours; restarting
fixes it. The team added a cron to restart pods every 12 hours. Is that acceptable?

> **Direction:** As a time-boxed mitigation yes, but restarts cause cold starts and hide the leak; stagger restarts, add heap profiling, and put an owner and date on the root-cause fix; §2, §15, `C12`.

**18.** We store user uploads on local disk of the app servers behind a load balancer with sticky
sessions. We need to scale from 3 to 30 servers. What breaks and what is the migration?

> **Direction:** State on the stateless tier breaks scaling and failover; move to object storage with direct signed uploads and migrate with a lazy read-through copy plus a backfill; §12, `F17`, `O11`.

**19.** Our database is 3 TB and growing 150 GB a month. Queries are fine, but backups take 9 hours,
restoring a replica takes a day and vacuum never finishes on the biggest table. Nobody is complaining
yet. What do you propose?

> **Direction:** The operational limits arrive before query limits; partition by time and archive or drop old partitions, set retention, and measure restore time as a real RTO; §1, §14, `DB24`, `DB36`.

**20.** Our cloud bill doubled in six months while traffic grew 20%. Finance asks engineering to
"optimise". Where do you look first, and in what order?

> **Direction:** Attribute cost first; the usual culprits are log ingestion, metric cardinality, cross-AZ and NAT traffic, and idle over-provisioned databases — not compute; §14, §1, `O18`.

### Level 2 — Hot spots, counters and feeds

Skew: the key, user or tenant that is a thousand times bigger than the rest.

**1.** A celebrity with 40M followers posted and our timeline fan-out queue backed up for 90 minutes;
ordinary users' posts were delayed too. How do you redesign the fan-out?

> **Direction:** Hybrid push/pull with a dynamic follower threshold, celebrity posts merged at read time, and fan-out on a separate priority queue so one post cannot delay everyone; §3, §5, `SD09`, `SD12`.

**2.** Our "likes" count is a column on the posts table. A viral post causes lock waits on that row and
checkout (same database) slows. Fix it without changing the product.

> **Direction:** Hot row; make the like an edge with a unique constraint and derive the count via sharded counters or buffered increments, cached; §4, `SD06`, `DB09`.

**3.** A single product in our catalogue is being viewed 200k times per second during a launch. Redis
Cluster shows one node at 100% CPU and the others at 10%. What do you do in the next ten minutes and
what do you build afterwards?

> **Direction:** Now: in-process cache with a short TTL in front of Redis; afterwards: hot-key detection and replicated keys across slots; §3, `Q04`, `Q09`.

**4.** We show a "views" count on videos that creators get paid from. A creator found they could
inflate it with a script. The product team wants the count to stay live. How do you design this?

> **Direction:** Separate the display counter (approximate, fast) from the billing count (deduped, bot-filtered, delayed, audited pipeline); a delayed correct number beats a live wrong one when money depends on it; §4, `S13`, `Q17`.

**5.** Our unique-visitors metric per page uses a Redis set of user IDs. Memory usage hit 180 GB and
keeps growing. The number is only shown on a dashboard. What do you change?

> **Direction:** HyperLogLog gives ~0.8% error in ~12 KB per counter; the product does not need exact uniques; §4, `DB33`, `Q08`.

**6.** One tenant in our B2B SaaS is 40% of all data and its nightly import makes every other tenant's
API slow. We shard by tenant ID. What are the options, in the order you would try them?

> **Direction:** Per-tenant quotas and fair scheduling first, then sub-partition the whale by tenant plus entity hash, then a dedicated cell or shard as a product feature; §3, §10, §11, `M29`, `Q21`.

**7.** We send a daily digest email to all 12M users at 9am local time. At 9:00 in each time zone our
API gets a spike that is 20× baseline as people click through. What do you change?

> **Direction:** Temporal skew — spread sends across a window with jitter and prefetch/cache the landing content; smoothing the cause beats scaling for the peak; §3, §5, §2.

**8.** We send a breaking-news push to 25M devices at once. Our API dies within two minutes every time.
Walk me through the fix end to end.

> **Direction:** Push-induced herd; stagger sends over 10–30 minutes, carry the content in the payload, pre-warm and statically serve the article, and shed non-critical calls from the app on open; §5, §9, §8.

**9.** Our leaderboard is a Redis sorted set of 50M players. Rank queries are fast, but the nightly
reset (delete and rebuild) blocks Redis for 40 seconds. How do you fix it?

> **Direction:** Big-key operations block single-threaded Redis; build into a new key and atomically rename (or `UNLINK` asynchronously), or key the board by period so reset is a new key; §3, §1, `Q05`, `Q08`.

**10.** We use auto-increment order IDs as the partition key for a DynamoDB-style table. Write
throttling happens even though we are under provisioned capacity overall. Why?

> **Direction:** Monotonic keys concentrate writes on one partition; hash-prefix or randomise the key — and note the opposite trade-off in a B-tree RDBMS; §3, `DB28`, `M32`.

**11.** Our timelines are rebuilt when users log in after being inactive, which takes 3–8 seconds for
users following thousands of accounts. Returning users bounce. What do you do?

> **Direction:** Serve a cheap approximate timeline immediately (popular or recent from pull), rebuild asynchronously and stream in, and prioritise rebuilds for returning users; §3, §5, `SD12`.

**12.** Our unread-notification count is computed with `COUNT(*)` on each page load. For most users it
is fast; for 2,000 power users with 300k notifications each it takes seconds and they load the most
pages. What is the fix?

> **Direction:** Maintain a counter incrementally and cap it ("99+"), so you never count past the display limit; the product constraint removes the scaling problem; §5, §4.

**13.** A single chat room with 400k members in our messaging app causes a broadcast storm whenever
someone types; typing indicators alone are 90% of our WebSocket traffic. What do you change?

> **Direction:** Degrade features by room size: disable or sample typing and presence above a threshold, batch broadcasts, and fan out via a tiered pub/sub rather than per-connection sends; §5, §9, §17, `F16`.

**14.** Our "trending hashtags" feature counts every tag in every post in Postgres and runs a GROUP BY
every minute. It now takes 70 seconds. Redesign it.

> **Direction:** Streaming approximate top-K (count-min sketch plus heap, or windowed aggregation in a stream job) instead of exact counts on the OLTP store; §4, §3, `Q17`, `DB33`.

**15.** We cache the rendered home feed per user for 5 minutes. A celebrity deleted an offensive post
and it stayed visible in millions of cached feeds. Legal is unhappy. How do you design deletes?

> **Direction:** Cache IDs not content and hydrate at read time, so a delete is one write to the post cache; pulled celebrity content is easier to retract; add a denylist check on render; §5, §3, `Q03`.

**16.** Our Kafka topic is keyed by customer ID for ordering. One customer generates 30% of events and
its partition is permanently lagging while the others are idle. The customer requires ordering only
per order, not per customer. What do you change?

> **Direction:** Key by the finest entity that needs ordering (order ID), not the tenant; the ordering requirement defines the key; §3, `Q14`, `M32`.

**17.** Our "only 3 left!" badge uses a live DB count on every product view. During a sale it is the
most expensive query we run. Product insists the badge is important. What do you propose?

> **Direction:** The badge tolerates staleness; precompute it on stock change or cache for seconds, and degrade it under load; exact truth is enforced only at checkout; §4, §9, `SD08`.

**18.** Every customer in our SaaS has a scheduled export at "midnight", and most chose the default.
At 00:00 UTC our workers saturate for two hours. What is the cheap fix and what is the proper one?

> **Direction:** Cheap: jitter start times deterministically per tenant; proper: a fair scheduler with per-tenant concurrency caps and a promise of "by 06:00" rather than "at 00:00"; §3, §10, `Q21`, `F14`.

**19.** We store a comment thread as one JSON document per post. Viral posts have 200k comments, the
document hit the store's size limit, and every new comment rewrites megabytes. What is the redesign?

> **Direction:** Unbounded growth inside one record is a hot spot; store comments as separate rows keyed by post plus time/bucket and paginate by cursor; §3, §5, `SD04`, `DB27`.

**20.** We run a "follow suggestions" job that computes friends-of-friends for every user nightly. For
users with 5k friends it explodes and the job no longer finishes by morning. What do you do?

> **Direction:** Cap and sample high-degree nodes, handle celebrities separately, and compute incrementally or on demand for active users only; §3, §14, `SD09`.

### Level 3 — Overload, retries and graceful degradation

What the system does when demand exceeds capacity, and why the obvious reactions make it worse.

**1.** A downstream inventory service had a 30-second blip. Our checkout service stayed at 100% error
rate for 25 minutes after inventory recovered. Traffic was normal. What kept it broken?

> **Direction:** A metastable failure sustained by retry amplification and queued stale work; break it by cutting load below the trigger (disable retries, shed, drop expired work) then readmit gradually; §8, `M09`, `M10`.

**2.** Our gateway retries 3 times, the BFF retries 3 times, and each service retries its database
calls 3 times. Someone says "retries make us resilient". Quantify why they are wrong and propose a
policy.

> **Direction:** (r+1)^n gives 64 attempts per request at the leaf; retry at one layer, with jittered backoff and a retry budget of ~10%; §1, §8, `M09`.

**3.** During an overload incident the team added 50% more pods and the error rate went *up*. Explain
how that is possible.

> **Direction:** New cold pods bring connection storms and warm-up cost, more connections to a saturated DB, and new capacity gets eaten by retry amplification; §8, §2, `DB21`.

**4.** Our Kubernetes liveness probe calls `/health`, which checks the database. The DB had a 20-second
slowdown and every pod in the cluster restarted, causing a 15-minute outage. What is the fix?

> **Direction:** Liveness must check only the process; dependency checks belong (carefully) in readiness; a dependency blip must not restart the fleet; §9, `O06`.

**5.** When our payment provider is slow, our checkout threads all block waiting on it, and the product
catalogue — same service — stops responding too. What design change isolates this?

> **Direction:** Bulkheads: separate pools per dependency with tight timeouts, plus a breaker on the provider, so one slow dependency cannot consume shared threads; §9, `M10`, `C16`.

**6.** We use a work queue for image processing. After a two-hour outage we have 4M jobs backlogged,
and users are uploading new images that will not be processed for hours. What order should the queue
drain in, and why?

> **Direction:** Process newest first (LIFO or a separate fresh lane) so current users succeed, and backfill or drop the stale jobs whose users are gone; §8, §17, `Q21`.

**7.** The product team asks what the site should do when the recommendation service is down. Today
the whole product page returns 500. Design the degraded behaviour.

> **Direction:** Predefined brownout: tight timeout, fall back to static popular items, and a flag to disable the call entirely; agree the degradation list with product before incidents; §9, `O19`.

**8.** Our origin went down on Black Friday for 12 minutes; the site was completely unavailable. The CDN
was in front the whole time. What would have kept us partly up?

> **Direction:** Static fallbacks and `stale-if-error`: pre-rendered catalogue and product pages served from the CDN while only the cart needs the origin; §9, `A04`, `Q09`.

**9.** Our server keeps processing requests for 30 seconds, but the mobile client times out after 10
and retries. Under load, the server's CPU is mostly spent on responses nobody receives. Fix it.

> **Direction:** Deadline propagation: pass the client deadline down and drop work whose deadline has passed at each hop, killing goodput collapse; §8, §17, `M09`.

**10.** We shed load by returning 503 when CPU is over 90%. It barely helps: by the time we shed,
latency is already terrible, and shedding itself costs a full auth and DB lookup. What would a better
shedder look like?

> **Direction:** Shed on in-flight concurrency or queue wait with adaptive limits, at the first middleware before any expensive work, by request priority; §9, §1, `C14`, `M30`.

**11.** After a 40-minute outage of our auth service, all 600 pods' circuit breakers closed within the
same second, auth fell over again, and this repeated four times. How do you design recovery?

> **Direction:** Thundering herd on recovery; half-open breakers with probe fractions and jitter, gradual readmission via weights or flags, and warmed capacity; §8, `M10`.

**12.** Our clients reconnect WebSockets with a fixed 5-second retry. After a deploy restarted our
realtime tier, 2M clients reconnected at once and the TLS handshakes alone saturated the new pods.
What do you change?

> **Direction:** Jittered exponential backoff on the client, server-supplied reconnect delays, staggered drain on deploy, and connection admission limits per pod; §8, §2, `F16`, `A11`.

**13.** Our health check endpoint is itself behind the same rate limiter and middleware as user
traffic. During a surge the load balancer marked healthy pods as unhealthy. What went wrong?

> **Direction:** Control-plane and health traffic must never be shed or queued behind user traffic; give it its own path and priority; §9, `O06`.

**14.** Our fraud-scoring service has 99.5% availability, and checkout calls it synchronously and fails
closed. That caps checkout availability below what the business wants. What options do you offer?

> **Direction:** Degrade instead of failing closed: allow low-value orders with post-hoc review, queue high-value ones, cache recent scores; fail-open vs fail-closed is decided per dependency with the business; §9, §16, `SD07`.

**15.** Every Monday at 9am our B2B customers' integrations sync at once and our API returns 429 for
40 minutes. The customers complain the rate limit is broken. What do you do?

> **Direction:** Temporal skew plus clients ignoring `Retry-After`; publish jittered sync guidance, return `Retry-After`, offer webhooks or bulk endpoints, and use fair per-tenant limits; §3, §10, `A08`, `A21`.

**16.** A bug caused our service to log every request body at ERROR level. The log pipeline fell
behind, the logging agents filled node disks, and the pods were evicted. How do you make logging safe
for the service?

> **Direction:** Logging is a dependency that can take you down; non-blocking bounded log buffers that drop, sampling and rate limits on log volume, and disk quotas; §9, §14, `F08`, `O13`.

**17.** Our search service times out under load, and the frontend's response to a timeout is to retry
the search with the filters relaxed, which is a more expensive query. What is wrong with this?

> **Direction:** A fallback that is more expensive than the original amplifies overload; fallbacks must be cheaper (cached results, fewer facets) and under the same budget; §8, §9.

**18.** After a regional network blip, our Java services' GC pauses went from 50 ms to 3 s and stayed
there even when traffic dropped back to normal. Why would it stay?

> **Direction:** A GC death spiral as a sustaining loop: backlog grows the heap, pauses cause timeouts and retries that grow it further; cut load below trigger or restart behind shed traffic; §8, `C12`.

**19.** We have a kill switch for our expensive personalisation feature. During last week's incident we
flipped it and nothing happened — the flag service was part of the outage. What design lessons do
you take?

> **Direction:** Kill switches must be statically stable (cached, with defaults) and exercised regularly; the control path must not share the failure; §11, §17, `O19`.

**20.** Our p99 is dominated by one slow replica out of eight at any moment (GC, noisy neighbour). The
query is an idempotent read. What cheap technique would you try?

> **Direction:** Request hedging: send a second request after the p95 and take the first reply, costing a few percent extra load; §17, §1.

### Level 4 — Correctness in money and inventory

Where "mostly right" is wrong: bookings, payments, ledgers, exactly-once illusions.

**1.** A customer was charged twice for one order. Logs show our service timed out calling the payment
provider, returned an error, and the user clicked "Pay" again. How do you make this impossible?

> **Direction:** A timeout is an unknown outcome: client-generated idempotency key per order passed to the provider, a pending state resolved by lookup or webhook, never a new key on retry; §7, `A09`, `SD10`.

**2.** We sold 10,000 tickets for a 9,800-seat venue. Stock is a `remaining` column decremented after a
`SELECT` check. What went wrong and what is the fix at high concurrency?

> **Direction:** Check-then-act race; an atomic conditional update or per-unit/bucketed inventory with `SKIP LOCKED` — plus deciding whether any oversell is acceptable; §6, `DB09`, `DB10`.

**3.** Seat holds expire after 10 minutes via a sweeper job. During the big on-sale the sweeper fell
behind and 30% of seats appeared taken but were actually expired. Fix it.

> **Direction:** Enforce expiry in the claim condition (`held_until < now()`), so the sweeper is housekeeping, not correctness; §6, `SD10`.

**4.** Webhooks from our payment provider sometimes arrive before our own API call returns, sometimes
twice, and occasionally "refunded" before "captured". Our order state is getting corrupted. Design
the handling.

> **Direction:** Store raw webhooks first (inbox), dedupe by event ID, apply as state-machine transitions with conditional updates, and fetch current state from the provider when out of order; §7, `M15`, `M16`.

**5.** Finance says our revenue total is off from the payment provider's settlement by £14,000 this
month and nobody can explain it. We have idempotency everywhere. What is missing?

> **Direction:** Reconciliation: three-way matching of ledger, provider settlement and bank, with mismatch cases and a mismatch-rate SLO; it catches manual refunds, fees, FX and provider bugs idempotency cannot; §7, `DB38`.

**6.** Our wallet balance is a column updated with `balance = balance - x`. After an incident, some
balances are wrong and we cannot tell what happened. How would you redesign it?

> **Direction:** Append-only double-entry ledger with derived balances and snapshots; corrections are reversing entries, giving an audit trail; §7, `DB38`.

**7.** We added a second payment provider for failover. When provider A times out, we immediately try
provider B. Some customers are now charged by both. What is the correct failover policy?

> **Direction:** Fail over only on definitive declines or pre-request errors; on timeout resolve A first or void one of the double authorisations; enforce one capture per logical payment with a unique constraint; §7.

**8.** Our "exactly-once" Kafka pipeline sends a confirmation email per order. Customers occasionally
get two. The team says Kafka has exactly-once semantics, so it must be a Kafka bug. Is it?

> **Direction:** Kafka EOS covers read-process-write within Kafka, not external side effects; make the email sender idempotent by a dedupe key (order ID + template); §7, `M16`, `Q15`.

**9.** A flash sale of 1,000 units draws 300k users in the first minute. Our Postgres-backed checkout
melted. The business wants fairness and no oversell. Design it.

> **Direction:** Virtual waiting room admitting at backend rate with signed tokens, inventory split into buckets or pre-allocated tokens, and random queue positions at opening for fairness; §6, `SD10`.

**10.** Bots are holding all the seats in our booking system (10-minute holds) and releasing them just
before expiry, so real users see "sold out". What do you do?

> **Direction:** Holds are an attack surface: limit holds per account, device and card, require a queue token, shorten TTLs under attack, and add abuse detection; §6, `S13`.

**11.** We capture card payments when orders ship. A supplier delay meant many orders shipped after 10
days and the captures failed. How should the design handle long fulfilment?

> **Direction:** Authorisations expire (~7 days); track expiry, re-authorise before it lapses or capture partially earlier, and surface the failure path as a state; §7.

**12.** Our marketplace pays sellers out via a saga: debit platform ledger, call the bank's payout API,
mark paid. A crash after the bank call left payouts in "pending" and a retry paid some sellers twice.
Fix the saga.

> **Direction:** Write an intent record with an idempotency key before the irreversible external call, pass the key to the bank, and resolve pending by lookup; the irreversible step goes last; §7, §17, `M14`.

**13.** Our booking system double-booked a hotel room when two requests hit different app servers. We
use a Redis lock with a 5-second TTL. One of the holders had a 7-second GC pause. What is the real fix?

> **Direction:** Locks with TTL cannot guarantee exclusion; enforce it in the database with a unique constraint or fencing token on the write; §6, `C11`, `DB05`.

**14.** Customers in Japan see prices ending in odd decimals and some refunds fail with "amount
exceeds original". We store money as floats in pounds converted at display time. What is the fix and
what is the migration?

> **Direction:** Integer minor units with a currency on every amount (JPY has no minor unit), FX recorded per transaction; migrate with expand/contract and a verified backfill; §7, §12, `DB04`, `DB38`.

**15.** Our airline client says overselling by 3% is industry standard and they want it. Our engineering
team insists on strict inventory. How do you design and argue this?

> **Direction:** Oversell deliberately with a bounded allowance and a compensation workflow when prevention costs more than the rare fix; make the business trade-off explicit; §6, §17, §16.

**16.** A refund was issued manually in the payment provider's dashboard by support. Our system still
shows the order as paid and later tried to refund it again via the API. How do you prevent drift like
this?

> **Direction:** Treat the provider as a second writer: consume its events and reconcile against settlement reports, and route support actions through your system; §7.

**17.** Our subscription billing runs as a nightly job. It crashed halfway through last night and when
restarted, charged some customers twice. What properties does the job need?

> **Direction:** Idempotent per (customer, billing period) with a unique constraint, resumable from a checkpoint, and provider idempotency keys derived deterministically from the period; §7, `F13`, `Q16`.

**18.** Promo codes limited to "first 1,000 uses" were used 1,340 times during a campaign. Redemptions
check a count and then insert. How do you enforce the cap under concurrency without a global lock?

> **Direction:** Pre-allocate 1,000 redemption tokens (or bucketed counters) claimed atomically, or an atomic conditional increment; a small bounded overshoot may be acceptable if stated; §6, §4, `DB10`.

**19.** We charge the card, then create the order in our DB. When the DB insert fails, customers are
charged with no order. The team wants 2PC. What do you propose instead?

> **Direction:** Create the order first in a pending state (intent), charge with an idempotency key, then confirm; reconcile orphans with a sweeper against the provider; 2PC across an external API is impossible; §7, §17, `M13`, `M14`.

**20.** Our loyalty-points balance is shown on every page and updated by 15 event types from different
services, with a "no negative balance" rule. Balances drift and occasionally go negative. How do you
restructure it?

> **Direction:** A single owner with an append-only points ledger, idempotent consumption by event ID, and invariants enforced at the owner with conditional writes, plus periodic reconciliation; §7, §4, `M16`, `M22`.

### Level 5 — Migrations under live traffic

Changing the engine while the plane is flying: schemas, stores, services and the cutover.

**1.** We need to move our orders table from MySQL to Postgres with no downtime. The first plan is to
dual write from the application. What goes wrong with that plan and what would you do instead?

> **Direction:** Dual writes are not atomic and race, so stores diverge silently; use CDC from the source's binlog, shadow reads to compare, then a flagged cutover with reverse replication for rollback; §12, `Q18`.

**2.** We are renaming a column that 14 services read. The last attempt to rename it caused an outage
because one service deployed late. Describe the sequence that cannot cause an outage.

> **Direction:** Expand/contract: add the new column, write both, backfill, move readers one by one, stop writing the old, drop it last — each step independently deployable; §12, `DB22`, `M24`.

**3.** Our backfill of 800M rows into a new column locked the table and replication lag hit 40
minutes, breaking read-after-write for users. How should the backfill have been built?

> **Direction:** Small primary-key-range batches, throttled on replica lag and DB CPU, resumable and idempotent, never one big UPDATE or OFFSET; §12, `DB22`, `DB23`.

**4.** We ran shadow reads comparing the old and new user services; the mismatch rate is 0.3% and the
team wants to cut over because "it's low". What do you want to know first?

> **Direction:** Classify the mismatches — precision, collation, NULL vs empty, time zones, replication lag — and only cut over when every class is explained; 0.3% of unexplained differences could be all one critical path; §12.

**5.** We are strangling a monolith. After 18 months, 90% of traffic is on new services but the
monolith still cannot be turned off, and the last 10% has no owner. How do you finish?

> **Direction:** Inventory every remaining consumer from DB access and network logs (not diagrams), assign owners, and publish a burn-down; the last 10% is where strangler projects die; §12, §16, `M28`.

**6.** We want to move from a single Postgres to a sharded setup. The CTO wants it "done by Q3". What
do you do before starting the sharding project?

> **Direction:** Check whether vertical scaling, read replicas, partitioning, archiving or moving hot workloads buys years instead; sharding is a one-way door that should be deferred if possible; §14, §1, `DB25`.

**7.** During a migration to a new database, a backfill finished and verification showed row counts
match. Two weeks after cutover we found 40k rows with stale data. How did counts miss it and how
should verification work?

> **Direction:** Counts miss content drift — the backfill overwrote newer live writes; use per-range checksums and version/`updated_at` guards or CDC from the snapshot position; §12.

**8.** We dark-launched a new notifications service reading real traffic in parallel with the old one.
Customers got every email twice. What went wrong and what is the rule?

> **Direction:** A dark launch must stub every side effect (email, charges, webhooks); only the computation is exercised, results compared and discarded; §12, §17.

**9.** We are moving to a new ID format (UUIDv4 instead of bigint). Some clients store IDs as integers,
and our Postgres writes got slower after a pilot. What trade-offs are you weighing?

> **Direction:** IDs are a one-way door leaking into clients; random UUIDs fragment B-tree indexes so time-ordered UUIDv7/ULID is better, and migrate with expand/contract keeping both IDs; §3, §14, §12, `DB13`.

**10.** We cut over to a new payments service on Friday, found a bug on Saturday, and discovered the
rollback was impossible because the old service had not received 30 hours of writes. What should the
cutover plan have included?

> **Direction:** Reverse replication keeping the old path warm for a rollback window, a documented rollback procedure tested before cutover, and not cutting over money on a Friday; §12, §15, `M27`.

**11.** Our migration to Kubernetes is 70% done but half the team wants to pause it because incidents
went up. How do you decide whether to continue, pause or roll back?

> **Direction:** Look at incident causes (migration-caused vs coincident), cost of running two platforms, and reversibility; running both indefinitely is usually the worst option; decide with data and communicate; §12, §14, §16.

**12.** A migration requires rewriting a 2 TB table. A colleague proposes a 45-minute maintenance window
on Sunday at 3am instead of a three-month online migration. Everyone else says "zero downtime is
mandatory". What is your position?

> **Direction:** Big-bang is sometimes right: weigh the real cost of 45 minutes of planned downtime against months of dual running and its risk; ask whether the business actually needs zero downtime; §12, §14, §16.

**13.** We are splitting a shared database between two services. Both teams have foreign keys and joins
across the boundary. What is the sequence?

> **Direction:** Establish one owner per table, replace cross-boundary joins with API calls or replicated read models, drop foreign keys, then split physically; the logical split precedes the physical one; §12, `M22`, `M23`.

**14.** We migrated search from Elasticsearch to OpenSearch. Results are "the same" in testing but
conversion dropped 4% in production. How do you find out why and how should the migration have been
run?

> **Direction:** Ranking differences show up only on real traffic; shadow-compare result sets on production queries, then A/B the cutover by business metric, not just correctness; §12, `DB31`, `M27`.

**15.** We need to re-encrypt 500M records with a new key while the system is live. How would you
structure it?

> **Direction:** Key version stored per record, read both versions, write with the new key, background re-encryption batches with throttling and checkpoints, verify by key version counts before retiring the old key; §12, `S10`, `S11`.

**16.** A mobile client relies on the API returning results sorted by creation date, though the docs
never promised it. Our new backend returns them in a different order and the app broke. What does
this teach you about migrations?

> **Direction:** Hyrum's law: every observable behaviour is depended upon; shadow comparison should include ordering, and contract tests plus consumer inventories catch implicit dependencies; §12, `A22`.

**17.** We moved a table from one database to another and the new one is faster, but the old database's
nightly cron jobs, BI tool and an internal admin page still read the old table. Nobody knew. How do
you find all consumers before a migration?

> **Direction:** Database audit logs, `pg_stat_statements` by user, network flow logs and credential usage; revoke access in steps (read-only, then brownout) to flush out unknown consumers; §12, `DB39`.

**18.** Our feature-flagged rollout of a new pricing engine reached 50% when finance noticed the two
engines disagree on tax for 0.1% of orders. The flag is per request. What problem does per-request
bucketing cause here?

> **Direction:** Per-request bucketing gives one customer different prices across requests; bucket by customer or cart, and shadow-compute the new engine before exposing it; §12, `O19`.

**19.** We want to change the partition key of a Kafka topic that 12 consumers read, some of which rely
on per-key ordering. How do you do it without breaking ordering?

> **Direction:** Create a new topic with the new key, dual publish (or bridge via a stream job), migrate consumers one by one at a known offset boundary, then retire the old topic; §12, `Q14`, `Q19`.

**20.** A migration from an in-house auth system to a managed identity provider requires moving 8M
password hashes in a format the provider does not support. How do you migrate without forcing a
reset for everyone?

> **Direction:** Lazy migration on login: verify against the old hash, then create the credential in the new provider; after a window, force resets only for the inactive remainder; §12, `S02`.

### Level 6 — Rate limiting, tenancy and fairness

Protecting shared systems from their most enthusiastic users.

**1.** Our global rate limiter is a single Redis counter per API key. The Redis node went down and we
returned 500 on every request. How would you redesign it?

> **Direction:** Fail open with a coarse local limit, and prefer local limits with periodic sync or leased token chunks over a per-request global counter; §10, `A08`, `SD11`.

**2.** Our largest customer's integration sends 5k RPS in bursts and every other tenant's latency
rises. They pay the most. The sales team says we cannot rate-limit them. What do you propose?

> **Direction:** Per-tenant isolation instead of throttling: dedicated capacity or cell as a paid tier, fair queuing, and contractual quotas; make the trade-off a commercial conversation; §10, §11, §16, `M29`.

**3.** We rate-limit by requests per minute. A customer found that one GraphQL query can fetch 50k
objects and costs us 1,000× a normal request. How do you limit fairly?

> **Direction:** Cost-based limiting: compute or estimate query cost units and budget those, with depth and complexity caps; §10, `A15`.

**4.** Our SMS provider allows 100 messages per second on our account. A marketing campaign used all
of it and 2FA codes were delayed by 20 minutes. Design the outbound side.

> **Direction:** A global outbound token bucket with priority queues, and separate provider accounts or sender pools for transactional and marketing traffic; §10, §5.

**5.** Our job system is FIFO. One tenant enqueued 2M jobs for a data import and all other tenants'
jobs waited 6 hours. What scheduling would you use?

> **Direction:** Weighted fair queuing across tenants with per-tenant concurrency caps, round-robin rather than global FIFO; §10, `Q21`.

**6.** We have 40 gateway instances each enforcing `limit / 40` locally. After autoscaling to 120, clients
are being throttled to a third of their quota. What went wrong and how should local limits work?

> **Direction:** Per-node share must track fleet size (from discovery) or use leased tokens from a shared budget; accept small over-admission; §10.

**7.** An abusive client rotates through thousands of IPs to bypass our per-IP rate limit on login.
What layers do you add?

> **Direction:** Layered limits keyed on account, device fingerprint and credential as well as IP, plus bot detection and progressive challenges at the edge; §10, `S13`.

**8.** Our own mobile app, after a bad release, retries failed requests in a tight loop without
backoff. It is 60% of our traffic and the release takes a week to roll out. What do you do right now?

> **Direction:** Server-side: remote config kill switch for client retry behaviour, 429s with `Retry-After`, and targeted shedding by app version; long term: client backoff with jitter and a remote retry cap; §10, §8.

**9.** A single tenant's query pattern causes Postgres to use a terrible plan that affects all tenants on
that shard. How do you find out which tenant is expensive and contain them?

> **Direction:** Per-request cost attribution by tenant (DB time, CPU), then quotas, statement timeouts per tenant role, or moving them to a dedicated shard; §17, §3, `M29`, `DB39`.

**10.** A noisy tenant's malformed payload crashes the worker process that handles it. Because all
tenants share the worker pool, every crash takes down processing for everyone. How do you isolate
it?

> **Direction:** Poison-pill quarantine after N attempts, and shuffle sharding so a bad tenant only affects its own subset of workers; §11, §17, `Q16`.

**11.** Our public API has per-key quotas. A customer's key leaked and was used to exhaust the quota,
locking the customer out of their own service. How would you design quotas to limit this damage?

> **Direction:** Scoped keys with per-key sub-quotas and anomaly alerts, separate limits per key rather than per account, and fast rotation; §10, `S10`, `A18`.

**12.** Our webhook delivery system retries failed deliveries. One customer's endpoint is down and the
retries for it are clogging delivery for everyone. Redesign delivery.

> **Direction:** Per-destination queues or concurrency caps with circuit breaking per endpoint, exponential backoff and eventual parking, so one dead endpoint only affects itself; §10, §11, `A16`, `Q16`.

**13.** We want to offer "guaranteed throughput" to enterprise customers. How do you build this without
dedicating a whole stack to each one?

> **Direction:** Reserved capacity in a shared pool via weighted fair queuing and admission control, with dedicated cells only for the largest; §10, §11, `M29`.

**14.** Our free tier is being used for crypto-mining style abuse of our compute feature and costs us
£40k a month. Rate limits have not helped because accounts are created en masse. What do you do?

> **Direction:** Limit what free accounts can consume in cost units, add friction at sign-up (verification, payment method for compute), and detect clusters; price the feature; §10, §14, `S13`.

**15.** A downstream partner API allows 50 RPS total across all our services, and three of our teams
call it independently. We keep getting banned. What is the design?

> **Direction:** One owner of the outbound integration with a single global token bucket or rate-paced queue, priority lanes per caller, and caching of responses; §10.

**16.** Our API returns 429 when a tenant is over quota, but the tenants' SDKs treat 429 like a 500
and retry immediately, tripling the load. What do you change on both sides?

> **Direction:** Send `Retry-After`, update SDKs to honour it with jittered backoff, and make rejection cheap at the edge so retries cost little; §10, §8, §9.

**17.** Our search cluster is shared by a public site and an internal analytics team. The analytics
team's queries occasionally saturate it. They say they need real-time data. What do you propose?

> **Direction:** Workload isolation: a replica or separate cluster fed from the same source, or a columnar store for analytics; separate "real-time" from "on the same cluster"; §10, §14, `DB31`, `DB32`.

**18.** We run a multi-tenant Kafka cluster. One team's topic with 3,000 partitions made controller
failover take minutes and affected all teams. How do you govern shared infrastructure?

> **Direction:** Quotas and limits per tenant on shared infra (partitions, throughput), plus capacity reviews; shared infrastructure needs explicit fairness like any multi-tenant service; §10, §11, `Q12`, `M29`.

**19.** A customer asks why they are being rate-limited when our dashboard shows them at 60% of quota.
Our limiter uses a fixed window per minute. Explain and fix.

> **Direction:** Fixed windows allow bursts at the boundary and per-minute averages hide per-second bursts; use a sliding window or token bucket and show the metric at the enforcement granularity; §10, `A08`.

**20.** We need per-user rate limits for 50M users across three regions. A global Redis would add 80 ms
of cross-region latency. What design gives acceptable accuracy?

> **Direction:** Enforce locally per region with the user's home region owning the budget, or split the budget per region with periodic rebalancing; accept small over-admission; §10, §13.

### Level 7 — Multi-region, residency and cells

Geography and blast radius: the designs that look simple on a whiteboard and cost a year in practice.

**1.** The CEO wants "active-active multi-region" after a competitor's outage made the news. We are a
single-region Postgres shop with 99.9% SLA. What questions do you ask and what do you probably
recommend?

> **Direction:** Establish the driver (latency, availability or regulation) and cost; often multi-AZ hardening plus a tested warm-standby region with a human failover decision is the right answer, not active-active; §13, §14, `SD13`.

**2.** We run active-active in two regions with last-writer-wins replication. Users occasionally report
that settings they changed "reverted". What is happening and what would you change?

> **Direction:** LWW silently discards concurrent writes and clock skew picks the winner; use home-region routing for single-writer-per-user, or per-field merges/CRDTs where merging is valid; §13, `M21`.

**3.** We expanded to the EU and legal says EU customer data must stay in the EU. Our database is
regional, but what else is likely leaking data out of the region?

> **Direction:** Logs, traces, analytics copies, backups, support tooling and third-party processors; residency covers processing, not just the primary store; §13, `S12`.

**4.** We failed over from us-east to us-west during an outage. It worked, but failing back took three
days and we found 12 minutes of orders that existed only in the old primary. What should the plan
have been?

> **Direction:** Async replication means RPO > 0; plan failback as a reconciliation of divergent writes, not a reverse switch, and rehearse it; §13, `O20`, `M33`.

**5.** Our automated regional failover triggered on a health-check blip, moved all traffic to the
secondary region, which lacked capacity, and caused a real outage. What would you change?

> **Direction:** Automate mechanics but keep a human decision on regional failover, ensure the target has headroom and quota for 2×, and use hysteresis; §13, §2.

**6.** We want usernames to be globally unique across our three regional deployments, each with its own
database. How do you do it?

> **Direction:** Global uniqueness needs a single authority or reservation protocol — a global directory service for usernames only; eventual consistency cannot guarantee it; §13, `SD13`.

**7.** A bad deploy took down 100% of our customers because every service deploys to all customers at
once. The team proposes more testing. What architectural change reduces the blast radius?

> **Direction:** Cell-based architecture with deploys cell by cell, a canary cell first and bake time, so a bad deploy hits one cell's customers; §11, `M31`, `O09`.

**8.** We moved to cells. Last week all cells went down together anyway. What kinds of shared
dependencies should you look for?

> **Direction:** Shared DNS, identity provider, secrets manager, global config push, CI/CD deploying all cells at once, certificates, and the cell router itself; §11.

**9.** Our cell router looks up tenant → cell in a central database on every request. That database
had an outage and every cell became unreachable although all cells were healthy. Redesign it.

> **Direction:** Static stability: the router caches the mapping (or the mapping is embedded in tokens/DNS) and keeps serving if the control plane is down; §11.

**10.** A tenant outgrew its cell. How do you move a live tenant from one cell to another?

> **Direction:** A per-tenant migration: CDC or replication into the new cell, verification, a brief per-tenant write freeze, router flip, and reverse replication for rollback; §11, §12.

**11.** Our Australian users complain about 900 ms page loads; everything runs in the EU. The CFO will
not pay for a second full region. What cheaper options improve their experience?

> **Direction:** CDN edge caching and static content, regional read replicas or caches for read-heavy paths, connection reuse and TLS at the edge; move writes only if needed; §13, §14, `Q09`, `A10`.

**12.** Our shopping cart is replicated active-active across two regions. Users sometimes find removed
items reappearing. Which data type suits a cart and why?

> **Direction:** An OR-set CRDT (add wins, removes tracked by tag) merges deterministically; naive set merge resurrects removed items; §13.

**13.** Our inventory service is active-active in two regions and we oversold during a network partition
between regions. Could CRDTs have saved us?

> **Direction:** No — invariants like "stock ≥ 0" need a single authority; route inventory writes to one region or pre-allocate stock quotas per region; §13, §6.

**14.** A noisy customer's traffic pattern took down a shared worker pool twice. We have 100 workers.
Shuffle sharding was suggested. Explain how it limits damage and what it costs.

> **Direction:** Each tenant gets a random k-of-N subset so a poison tenant only fully affects tenants sharing all k; costs include routing complexity and lower per-tenant peak capacity; §11.

**15.** Cross-AZ data transfer is now our third-largest cloud cost line. Our services call each other
freely across three zones. What do you change?

> **Direction:** Zone-aware routing to keep traffic in-zone, which also makes AZ failure cleaner; accept imbalance handling; §11, §14, `O18`.

**16.** We need to serve Chinese users and must host in-country. The rest of the product runs globally.
What is the architecture and what is the organisational cost?

> **Direction:** A separate region-pinned deployment (effectively its own cell) with only non-personal global routing, separate keys and operators; it becomes a second product to operate; §13, §11, §14.

**17.** Our DR plan says "restore from backup in another region, RTO 4 hours". It has never been
tested. How do you make it real?

> **Direction:** Game days that actually restore and serve traffic, measuring real RTO/RPO, including quota and capacity in the target region; an untested runbook does not work; §13, §17, `O20`, `DB36`.

**18.** We have a global metadata service that all regions call on startup. During a large recovery,
every pod in every region restarted, and the metadata service fell over under the load, preventing
recovery. What design principle was violated?

> **Direction:** Static stability and constant work: data planes must not depend on control planes to start or serve, and recovery must not be the largest load the control plane ever sees; §11, §17, §8.

**19.** We want to run the EU and US deployments as fully independent stacks for residency, but the
product team wants a single global admin dashboard with search across all customers. How do you
square this?

> **Direction:** A global aggregation layer holding only non-personal or pseudonymised data, with drill-down performed in-region by federated queries; §13, §11.

**20.** Our DynamoDB global tables replicate between two regions and a bug caused a write loop that
doubled our bill in a week before anyone noticed. What guardrails do you want for multi-region data?

> **Direction:** Cost and write-rate anomaly alerts per table, per-region write attribution, and replication-origin tagging to prevent loops; cost is a production signal; §14, §13, `O18`.

### Level 8 — Metastable and emergent failures

The incidents that only appear at scale, from interactions nobody designed.

**1.** After a 10-minute database failover, our system never recovered on its own; we had to block all
traffic at the edge for 5 minutes and let it back in slowly. Explain why the block worked when
scaling up did not.

> **Direction:** Metastable failure: the sustaining loop (retries, backlog, cold caches) keeps load above capacity; blocking drops load below the trigger so queues drain, then gradual readmission avoids re-triggering; §8.

**2.** Our Redis cluster runs at 40% CPU normally. During an unrelated incident, client timeouts were
set to 50 ms with 3 retries, and Redis went to 100% and stayed there. Why?

> **Direction:** Aggressive timeouts plus retries multiply load on a slightly slowed system and each retry makes it slower; widen timeouts, cap retries with a budget; §8, §1.

**3.** A certificate for an internal service expired at 00:00 UTC on a Sunday and took down 40
services. Every one of those certificates had been issued in the same batch. What systemic fixes do
you propose?

> **Direction:** Temporal skew in expiry — jitter or stagger issuance, automate renewal well before expiry, and alert on days-to-expiry; a hidden shared dependency across cells; §3, §11, `S10`.

**4.** Our autoscaler scaled from 200 to 1,200 pods during a spike, then the cloud API throttled our
autoscaler, the load balancer target registration lagged, and new pods sat unused while old ones
died. What limits were we not planning for?

> **Direction:** Control-plane limits: cloud API rate limits, target registration latency, IP exhaustion in subnets, image pull bandwidth; capacity planning must include them; §2.

**5.** During a large traffic increase, our NAT gateway started dropping connections to a third-party
API even though bandwidth was fine. What is the likely limit?

> **Direction:** Ephemeral port exhaustion per destination through the NAT; reuse connections with keep-alive pooling, add NAT IPs, or use private connectivity; §2, `A12`, `O02`.

**6.** Our cache hit rate dropped from 95% to 80% after a deploy that changed nothing but the cache key
format for a new field. The DB load quadrupled. Explain the maths and what you would do next time.

> **Direction:** Misses go from 5% to 20% — 4× DB load; a key-format change is a cold cache — roll out gradually, dual read old keys, or pre-warm; §8, §2, `Q03`.

**7.** We have a Kafka consumer that commits offsets after processing. A single malformed message makes
it crash; Kubernetes restarts it; it reads the same message and crashes again. The partition is
stuck for 6 hours. Design around this class of failure.

> **Direction:** Poison-pill quarantine: count attempts per message, park to a DLQ after N, alert and replay later; one record must not halt a partition; §17, `Q16`.

**8.** Our service mesh retry policy and our application's retry policy both retry on 503. Nobody knew
about the mesh one. During an incident the effective retry count was 16. How do you prevent hidden
retry layers?

> **Direction:** One layer owns retries, with retry budgets instead of counts, and a documented inventory of every layer's policy (SDK, mesh, gateway, app); §8, §1, `M26`.

**9.** We noticed that every day at 14:00 our p99 doubles for 10 minutes. No cron runs then. Traffic
is flat. How do you investigate an emergent periodic problem?

> **Direction:** Look for synchronised background work — TTL expiry cohorts from a deploy time, JVM/GC or compaction schedules, a partner's cron, log rotation, backups, vacuum; periodic problems are almost always temporal skew; §3, `O17`.

**10.** Our system was stable at 70% utilisation for a year. After a 10% traffic increase it began
failing daily. Nothing else changed. How can 10% cause that?

> **Direction:** Queueing non-linearity: from ρ 0.7 to 0.77 wait time grows and tail latency crosses timeouts, triggering retries — a metastable threshold; §1, §8, `C14`.

**11.** A DNS resolver in our cluster started timing out under load, and every service call slowed by 5
seconds (the resolver timeout). Why did a DNS issue look like a whole-platform slowdown?

> **Direction:** DNS is a hidden per-request dependency; resolver QPS limits and `ndots` search domains multiply lookups; cache DNS locally and reduce lookups; §2, §11, `A13`.

**12.** We enabled request hedging to improve p99. Two weeks later, during a minor slowdown, database
load doubled and the slowdown turned into an outage. What went wrong?

> **Direction:** Hedging without a budget doubles load exactly when the backend is slow; cap hedges to a small percentage and disable under overload; §17, §8.

**13.** Our clients cache a feature-flag configuration and refresh every 60 seconds. We pushed a flag
change and 3M clients fetched the config within one second because they all started at deploy time.
What is the fix?

> **Direction:** Jitter refresh timers, serve the config from a CDN, and push deltas or use constant-work full pushes to a cache; §3, §17, `O19`.

**14.** Our database's autovacuum could not keep up with a high-churn table; transaction ID wraparound
warnings appeared and Postgres threatened to shut down writes. Nobody saw it coming. What operational
design should have caught it?

> **Direction:** Operational limits arrive before query limits; monitor vacuum lag and XID age as capacity metrics, partition high-churn tables, and treat it as a capacity sheet item; §1, §2, `DB08`, `DB39`.

**15.** A slow memory leak in one of 300 pods made it slow but not dead; the load balancer kept sending
it traffic via least-connections, and because it held connections longer, it received *more*
traffic. Why does this happen and what fixes it?

> **Direction:** Least-connections and slow nodes interact badly (a slow node looks busy or, with some algorithms, gets more); use outlier detection/ejection on latency and errors; §9, §17, `M07`.

**16.** We deploy by rolling restart. On our largest service, each rollout causes a 3-minute error
spike because new JVMs are slow and the old pods drain too quickly. What do you change?

> **Direction:** Warm-up: slow-start on the load balancer, warm-up traffic before readiness, slower rollout pace, and graceful drain; recovery and rollout temporarily reduce capacity; §2, `F28`, `O09`.

**17.** Our message queue has a visibility timeout of 30 seconds. Under load, processing takes 40
seconds, messages reappear, other workers pick them up, and load doubles, making processing slower
still. Explain and fix.

> **Direction:** A metastable loop via redelivery; extend visibility with heartbeats, make processing idempotent, and cap in-flight work per worker; §8, `Q10`, `M16`.

**18.** One slow disk on one Elasticsearch data node made the whole cluster's search latency terrible,
because every query touches every shard. What design choices limit the damage of a single slow node?

> **Direction:** Fan-out tail amplification; adaptive replica selection, hedging to another replica, per-shard timeouts with partial results, and ejecting slow nodes; §1, §17, `DB31`.

**19.** We have perfect dashboards but during our last big incident nothing alerted for 25 minutes
because the metrics pipeline was part of the outage. How do you design monitoring that survives the
incident it is meant to detect?

> **Direction:** Out-of-band synthetic probes from outside the platform, a minimal independent alerting path, and alerting on missing data; §17, §11, `O16`.

**20.** Our service works perfectly until the monthly batch billing run, which pushes the DB to 90% and
causes checkout errors. The billing team says their job is "only running for an hour". How do you
frame and fix this?

> **Direction:** Two workloads sharing one resource at ρ 0.9 creates queueing collapse for the critical path; throttle the batch by DB latency, run on a replica or separate store, or prioritise with resource groups; §1, §9, §10, `Q21`.

### Level 9 — Senior judgement: incidents, pushback and trade-offs

The questions where the technical answer is only half of it: leading, deciding and saying no.

**1.** You are on call and join an incident call with 25 people, three of whom are making changes in
production at the same time. Nobody is sure what has been changed. What do you do in the first five
minutes?

> **Direction:** Establish incident command: one IC who does not debug, freeze uncoordinated changes, assign a scribe and comms lead, and focus on mitigation over root cause; §15.

**2.** An incident started 20 minutes after a deploy. The engineer who wrote the change is certain it is
unrelated and wants to keep debugging. What is your call and how do you handle the conversation?

> **Direction:** Roll back first, investigate after — rollback is cheap and reversible; frame it as removing a variable, not blame; §15.

**3.** Our database was corrupted by a bad migration. We can restore from a backup and lose 90 minutes
of orders, or keep the site down for about six hours while we repair in place and lose nothing. You
are the incident commander. How do you decide?

> **Direction:** A business decision with engineering input: quantify both (revenue, customers, recoverability of the lost 90 minutes from provider records and logs), involve the accountable leader, and often restore plus reconstruct from secondary sources; §15, §7, `DB36`.

**4.** Product wants a new real-time feature shipped in four weeks. Your estimate is ten weeks for a
safe version. The VP says "just make it work". How do you respond?

> **Direction:** Offer options with costs — cut scope, move the date, or accept named risks — in business terms, and let the accountable person choose, in writing; §16.

**5.** A senior colleague proposes moving our whole platform to an event-sourced architecture with
Kafka. You think it is the wrong call. How do you handle the disagreement?

> **Direction:** Separate facts from preferences, agree decision criteria, propose a time-boxed prototype on one bounded context, then disagree and commit with an ADR; also check innovation budget; §16, §14, `M18`.

**6.** Your team is asked to build an internal feature-flag platform. Several SaaS products exist. How do
you make the build-vs-buy recommendation?

> **Direction:** Total cost including engineer-years and on-call, differentiation, exit cost and vendor limits; flags are rarely core, so buy with an abstraction at the seam unless scale or compliance forbid it; §14.

**7.** After a major outage, leadership wants a list of every action that would have prevented it. The
draft postmortem has 34 action items. What is wrong with that, and what should it look like?

> **Direction:** Prioritise a few items with owners and dates, weighted to detection and mitigation as well as prevention, and track completion; long lists never get done; §15.

**8.** A customer-facing incident is ongoing. Sales wants to tell a large customer "it's a cloud
provider problem" because that is the rumour on the call. What do you advise?

> **Direction:** Never speculate on cause externally; communicate impact, what is being done and the next update time in customer terms; §15.

**9.** Your team spent its error budget in the first week of the quarter. Product has a big launch
planned in two weeks. How do you handle it?

> **Direction:** Apply the pre-agreed error-budget policy — reliability work first — and negotiate the launch with risk reduction (feature flag, cell canary, limited rollout); if no policy exists, that is the first thing to fix; §15, §16, `M34`.

**10.** You believe a service your team owns should be deleted: 2% of users use it, and it caused 30% of
last year's incidents. The original author is now your director. How do you make the case?

> **Direction:** Lead with data (usage, cost, incidents), propose a deprecation path with a migration for the 2%, and pre-socialise with the director one-to-one before any wider meeting; §16, §14.

**11.** You are writing a design doc for a new payments ledger. Reviewers keep arguing about the
queueing technology instead of the ledger model. How do you structure the doc and the review?

> **Direction:** State goals and non-goals, spend effort on one-way doors (ledger data model, IDs, invariants) and treat the queue as a two-way door; pre-review with key stakeholders; §16, §14, §7.

**12.** A junior engineer ran a script that deleted 40,000 customer records in production. The CTO asks
you who is to blame. What do you say and what do you change?

> **Direction:** Blameless: the system allowed an unreviewed destructive script against production; fix access, add soft deletes with delayed purge, two-person review for destructive operations, and tested restores; §15, §17.

**13.** At 3am you find what looks like a data breach — an S3 bucket with customer data was public for
unknown time. You are alone on call. What do you do, in order?

> **Direction:** Contain (make it private, preserve logs), escalate immediately to security and leadership, start a timeline, and do not delete evidence; regulatory clocks (72 hours under GDPR) may be running; §15, `S12`, `S16`.

**14.** Two teams both want to own the customer-profile data. Each has built partial copies and they
drift. You are asked to resolve it. How?

> **Direction:** Establish a single owner by domain responsibility, make others consumers via events or read models, and escalate together with a shared framing if the teams cannot agree; §16, `M22`, `M19`.

**15.** You are asked in an interview: "Tell me about a technical decision you made that turned out to
be wrong." What makes a senior answer?

> **Direction:** A real decision with stakes, why it seemed right, the signal that showed it was wrong, how you reversed or contained it, and what you now do differently — reversibility thinking; §14, §16, `SD17`.

**16.** Your team is asked to guarantee 99.99% availability for a new service that depends on three
99.9% services synchronously. Leadership has already announced it. What do you say?

> **Direction:** Availability multiplies — the ceiling is ~99.7%; offer options: decouple asynchronously, cache and degrade, or change the target, stated in numbers; §16, §9, `SD07`, `M34`.

**17.** An engineer on your team wants to rewrite a legacy service from scratch in a new language
because it is "unmaintainable". It is stable and earns most of the revenue. How do you decide?

> **Direction:** Rewrites are high-risk one-way-ish doors; prefer incremental strangling of the painful parts, measure the actual cost of maintenance, and spend innovation budget carefully; §12, §14, `F27`.

**18.** During an incident you realise the fix that restored service also disabled fraud checks for all
payments. The incident is "resolved" and everyone is leaving the call. What do you do?

> **Direction:** The incident is not over until side effects are reversed and reconciled; reopen, re-enable the checks, review the payments processed in the window, and record it in the timeline; §15, §7, §9.

**19.** You inherited a system with no runbooks, no dashboards and one engineer who knows it, who is
leaving in a month. What do you prioritise?

> **Direction:** Knowledge capture on failure modes and recovery first (runbooks from pairing through real operations), then synthetic probes and a minimal dashboard, then a game day while the expert is still there; §15, §17, `O16`.

**20.** An executive asks "Why did our recent outage last four hours when it took five minutes to fix?"
How do you explain it and what do you propose?

> **Direction:** Time went to detection and diagnosis, not the fix; propose investment in detection (synthetic probes, alerting), mitigation levers (kill switches, rollback) and incident process; §15, §17.

### Level 10 — Compound redesigns and rare failure modes

Staff-leaning questions: several constraints at once, where every textbook answer breaks something.

**1.** We run a ticketing platform. A stadium tour goes on sale in six weeks with 2M expected users for
80k seats across 10 cities, and last year our site went down for an hour. You have a small team.
What do you build, and what do you deliberately not build?

> **Direction:** A static virtual waiting room with signed admission tokens, bucketed inventory with TTL holds enforced in the claim, pre-scaling and a load test with production-shaped data, bot limits on holds — and not a rewrite or multi-region; §6, §2, §9, §14, `SD10`.

**2.** Our global payment platform uses three providers and processes £2B a year. We see a 0.02%
reconciliation mismatch rate, mostly unexplained, and the auditors are unhappy. Design the correctness
programme.

> **Direction:** Per-attempt records with provider references, write-ahead intents, state machines with inbox-stored webhooks, and automated three-way reconciliation with categorised mismatch cases and an SLO; §7, §17, `DB38`.

**3.** We want to migrate a 40 TB, 15-year-old Oracle database behind a revenue-critical monolith to
Postgres, service by service, without downtime. Lay out the programme and the risks.

> **Direction:** Strangler per bounded context, CDC from Oracle as the single source of order, shadow reads with mismatch classification, per-tenant flagged cutovers with reverse replication, and a consumer inventory from DB audit logs; §12, `M28`, `Q18`.

**4.** Our SaaS serves 3,000 tenants in one stack. A single tenant's bug-triggered traffic took the
whole platform down twice this year. The board asks for "no single customer can take us down". How do
you get there over a year?

> **Direction:** Cells with a statically stable router and cell-by-cell deploys, shuffle sharding for shared pools, per-tenant quotas and fair scheduling, and per-tenant cost attribution; sequence by blast-radius reduction per effort; §11, §10, §17, `M31`.

**5.** Our social app has a celebrity whose posts get 5M likes and 400k comments in an hour, and the
same account is also used for paid promotions billed per engagement. Design the post, the counters and
the billing path.

> **Direction:** Pull-based fan-out for celebrity posts, sharded or buffered counters for display, like edges as the truth, and a separate deduped, bot-filtered, delayed billing pipeline reconciled against the edges; §3, §4, §7.

**6.** We need active-active across the EU and US for availability, with EU data residency for EU
customers, a global marketplace where EU buyers purchase from US sellers, and inventory that must never
oversell. Design the data placement.

> **Direction:** Home-region per user and seller, inventory authoritative in the seller's region, cross-region purchases as a saga with a reservation in the seller's region, pseudonymised global routing data only, and residency-aware logs; §13, §6, §7, `M14`.

**7.** After a large cloud provider's regional outage, our recovery took 11 hours even though our data
was intact, because every service restarted at once and our config, auth and discovery services fell
over under the herd. Redesign the platform's recovery behaviour.

> **Direction:** Static stability (data planes serve from cached config and credentials), constant-work control planes, staged and jittered restart and readmission, and game days that rehearse cold start of the whole platform; §8, §11, §17.

**8.** Our mobile app has 30M installs across 400 app versions, some years old, which retry
aggressively and ignore `Retry-After`. Any backend blip becomes a self-inflicted DDoS. We cannot force
upgrades. What do you do?

> **Direction:** Server-side controls — per-version shedding and priority at the edge, cheap rejection, remote config for retry and polling where supported — plus minimum-version gates for the worst offenders; treat old clients as an adversarial workload; §10, §8, §9.

**9.** We are asked to reduce infrastructure cost by 40% in six months without reducing reliability, on a
platform of 200 microservices. What is your plan?

> **Direction:** Attribute cost per service and tenant, attack the hidden lines (logs, metrics cardinality, cross-AZ/NAT, idle databases, retention), consolidate services that should be modules, and use spot for stateless work; §14, §11, §16, `O18`.

**10.** A metastable failure occurs roughly monthly: a brief dependency blip leads to a 30–60 minute
outage. Each postmortem blames a different trigger. How do you address the class rather than the
triggers?

> **Direction:** Identify the sustaining loop (retries, backlog, cold caches) and remove it: retry budgets, deadline propagation, LIFO or drop-oldest under backlog, adaptive shedding, and load tests that inject the blip; §8, §9, §2.

**11.** We want to add a second region for disaster recovery, but our system uses Postgres sequences,
cron jobs that assume a single instance, and a Redis-based distributed lock for payouts. What breaks
in failover and how do you prepare?

> **Direction:** Sequences, singletons and locks assume one authority; switch to region-safe IDs, leader-elected schedulers with fencing, and database-enforced idempotency for payouts, then rehearse failover; §13, §7, `C11`, `M20`.

**12.** Our search index is rebuilt nightly from Postgres, taking 7 hours. Product wants near-real-time
search, and the rebuild is also our only recovery path if the index corrupts. Redesign it.

> **Direction:** CDC-driven incremental indexing with an alias-swapped full rebuild kept as the recovery path, versioned documents to avoid out-of-order overwrites, and shadow comparison of the two paths; §12, `Q18`, `DB31`.

**13.** Our booking system for a national vaccination programme must handle 10M people eligible at
08:00 on one day, with fairness scrutinised by the press and every slot a real clinic capacity.
Design it.

> **Direction:** Waiting room with randomised queue position at opening, static information pages, admission at the backend's measured rate, holds with short TTLs and per-person limits, and pre-agreed degradation and communication plans; §6, §9, §15.

**14.** We process 5B events a day for usage-based billing. Customers dispute invoices, and we cannot
prove any individual number. Design the metering pipeline for auditable correctness.

> **Direction:** Idempotent event ingestion with dedupe keys, an immutable raw log, deterministic replayable aggregation, reconciliation between raw counts and invoiced totals, and per-customer drill-down; §4, §7, `Q17`, `M16`.

**15.** Our platform must support a customer that requires its data in a dedicated, single-tenant
deployment in its own cloud account, while we keep one codebase and deploy weekly. How do you
architect and operate this?

> **Direction:** Treat it as a cell with the same automation, configuration by cell, deploy the same artefacts through the cell pipeline, and remote-operable observability that respects the boundary; price the operational cost; §11, §13, §14.

**16.** A regulator requires that we can prove a deleted customer's data is gone from all systems within
30 days, including backups, analytics and search. We have 60 services. How do you design this?

> **Direction:** A data map, a deletion workflow with fan-out and verification per system, crypto-shredding with per-customer keys for immutable copies such as backups, and residency-aware processing; §13, §17, `S12`, `DB37`.

**17.** Our ride-hailing marketplace matches drivers and riders per city. During New Year's Eve, three
cities overload the matching service and degrade the other 400 cities too. Redesign for isolation
and graceful degradation.

> **Direction:** Cell per city or city group so hot cities are isolated, prioritised shedding (matching over ETAs and analytics), pre-scaling on the known date, and degradation such as coarser matching radius; §11, §9, §2.

**18.** Our company acquired a competitor with a different stack, overlapping customers and a separate
billing system. You are asked to plan the platform consolidation over two years. What is your
approach?

> **Direction:** Decide the target per domain, not wholesale; identity and billing first behind facades, strangler per domain with CDC and reconciliation, and keep both running where consolidation has no business value; §12, §14, §16.

**19.** Our distributed cache, feature flags and service discovery all use the same etcd/ZooKeeper-style
cluster. An upgrade of that cluster caused a partial outage across everything. How do you restructure
dependencies on coordination services?

> **Direction:** Separate critical coordination from convenience uses, make data planes statically stable with cached values, and upgrade progressively by cell; a shared coordination service is a hidden global dependency; §11, §17, `M20`.

**20.** You join as a senior engineer. In your first month you find the platform has no load shedding,
no cells, untested DR, retries at every layer and a 99.95% SLA promised to customers. You cannot fix
everything. What do you do first, and how do you get buy-in?

> **Direction:** Rank by risk × likelihood per effort — retry policy and shedding are cheap and prevent metastable outages, DR testing reveals unknowns — write a short doc with options and costs, and pre-socialise with leadership; §8, §9, §16, §15.

