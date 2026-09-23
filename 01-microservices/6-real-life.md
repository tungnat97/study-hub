[← back to the field index](README.md)

# Microservices · Part 6 — Real-life production problems

The first five parts teach the nodes `M01`–`M35` as a textbook would. This part is different. In a
senior interview the question often arrives as a story: "we had this incident, here are the
symptoms, what would you do?" The textbook answer ("add a circuit breaker") is rarely what the
interviewer is listening for. They want the answer of someone who has been paged: the default
timeout nobody knew about, the retry that multiplied load by 27, the health check that lied, the
fix that looks wrong until you understand the mechanism.

How to use this file:

1. Read the **pre-knowledge** first. It holds every mechanism, default and technique the questions
   rely on. It is deliberately dense; treat it as the reference you come back to.
2. For each question, answer **out loud** before reading the direction line. Say what you would
   look at first, what you suspect, how you would confirm it, and what you would change — short
   term and long term.
3. Then read the **direction**. It names the non-obvious insight and the section (`§n`) and nodes
   it relies on. If your answer missed it, reread that section.

Levels run from the questions asked in almost every senior loop (Level 1) to the ones only people
who have run large systems recognise (Level 10). Defaults quoted below are correct for recent
versions at the time of writing; always check the version you run.

---

## Pre-knowledge

### 1. Timeouts and deadlines — the defaults that bite

**Every client library has a default, and many defaults are "forever".** Knowing them is half of
incident response (`M08`, `M09`).

| Component | Default | Consequence |
|---|---|---|
| Go `http.Client{}` | no timeout at all | a black-holed peer hangs the goroutine forever |
| Python `requests` | no timeout unless `timeout=` passed | same; one of the most common Python outage causes |
| Java `HttpURLConnection` | connect and read timeouts 0 (infinite) | threads pile up in `socketRead0` |
| Java 11 `HttpClient` | no request timeout unless set | same |
| OkHttp | connect/read/write 10s each, call timeout 0 | a slow-drip response can run forever, because the read timeout resets on every byte |
| Node.js `http` server `keepAliveTimeout` | 5s | behind an AWS ALB (idle timeout 60s) the server closes idle connections the ALB still thinks are open → sporadic 502s |
| AWS ALB idle timeout | 60s | long-poll or slow report endpoints return 504 at exactly 60s |
| AWS NLB TCP idle timeout | 350s (configurable since 2024) | idle pooled connections silently dropped; next write gets RST or hangs |
| AWS NAT gateway idle timeout | 350s, then drops state *without* RST | client sees a hang until TCP retransmission gives up |
| Envoy route timeout | 15s | a 20s report endpoint behind a mesh fails at 15.0s with `504 UT` |
| Envoy `stream_idle_timeout` | 5 min | long-lived streaming RPCs with no traffic are reset |
| Envoy upstream HTTP `idle_timeout` | 1 hour | longer than most LB/NAT idle timeouts → stale connections |
| Nginx `proxy_read_timeout` | 60s | |
| gRPC | no deadline unless the client sets one | server work continues long after the caller gave up |
| PostgreSQL `statement_timeout` | 0 (off) | a runaway query holds a connection and its locks indefinitely |
| HikariCP `connectionTimeout` | 30s | a pool-starved service waits 30s *per request* before failing — the real error hides behind latency |

**The keep-alive rule.** For any hop, the *server's* idle timeout must be **longer** than the
*client's* (or LB's) idle timeout. Otherwise the server closes a connection at the instant the
client reuses it: the request is written into a closing socket, the client gets `ECONNRESET` or the
LB returns 502. Fix: Node `server.keepAliveTimeout = 65_000` and `headersTimeout = 66_000` behind a
60s ALB; or make the client pool evict idle connections sooner than the server does. The symptom
signature: low-rate 502s, worse at *low* traffic (more idle connections), vanishing at peak.

**TCP-level timeouts matter when a peer vanishes without closing.**

- `net.ipv4.tcp_syn_retries = 6` → a connect to a black-holed IP takes about **127s** to fail
  (1+2+4+8+16+32+64). Always set an explicit connect timeout (100ms–1s inside a datacentre).
- `net.ipv4.tcp_retries2 = 15` → on an established connection whose peer disappeared, unacked data
  is retransmitted with exponential backoff for roughly **15 minutes** before the kernel errors the
  socket. A client with no read timeout writing to a dead replica waits this long.
- TCP keepalive default: first probe after **7200s** idle (`tcp_keepalive_time`), then 9 probes
  75s apart. Useless against a 350s NAT idle timeout unless lowered (e.g. 60s) per socket.
- `TCP_USER_TIMEOUT` caps how long written data may stay unacknowledged; gRPC and libpq
  (`tcp_user_timeout`) expose it. It is the cure for "the replica was killed and the pool took 15
  minutes to notice".

**Deadlines, not timeouts.** A timeout is per hop; a deadline is an absolute time that travels with
the request. With per-hop timeouts of 5s at every layer, a four-hop chain can keep working for 20s
after the user left. gRPC propagates the deadline (`grpc-timeout` header) and cancels downstream
work when it expires; with HTTP you propagate a header (e.g. `X-Request-Deadline`) and check it
before expensive work. Rules:

- Each layer's timeout must be *shorter* than its caller's, or the caller times out first and the
  callee's work is wasted (and the caller's retry doubles load).
- Subtract elapsed time and a margin at each hop; if the remaining budget is below the callee's
  known minimum latency, fail fast without calling.
- Server side: check cancellation (`ctx.Err()`) before expensive steps, and drop queued requests
  whose deadline has passed ("dead on arrival") — under overload that is most of the queue.

**Choosing a value.** Use the callee's measured p99.9 under normal load plus headroom, not a round
number. Too tight: false timeouts and retries during a GC pause. Too loose: resources held for the
whole outage. A call with p99 40ms and a 30s timeout effectively has no timeout.

**The slow-drip problem.** Read timeouts are usually *inactivity* timeouts: they reset whenever a
byte arrives. A peer trickling one byte every 9s passes a 10s read timeout forever. Only a *total*
call timeout or a deadline protects you.

**Server-side timeouts for vanished clients.** The server has the mirror-image problem: a client
that disappears mid-transaction leaves its database session holding locks until TCP notices.
Postgres: `idle_in_transaction_session_timeout` (off by default) kills sessions idle inside a
transaction; `tcp_keepalives_idle` and `tcp_user_timeout` on the server detect dead peers;
`lock_timeout` stops a statement waiting forever for a lock. A `SELECT ... FOR UPDATE` used as a
cross-service lock is only as safe as these settings.

**Timeouts are a fleet-wide contract.** Most services inherit timeouts from a shared client library
or platform default. Changing one default (1s → 30s) silently changes the blast radius of every
dependency in every service. Make critical timeouts explicit per call, and make the effective
values visible (config dump endpoint, startup log).

### 2. Retries — amplification, budgets, hedging

**Amplification.** If each of N layers retries R times, worst-case load on the bottom layer is
(R+1)^N. Three layers each making 3 attempts → 27× load on the database, exactly when it is already
struggling. Rule: **retry at one layer only** (usually the one closest to the user request, or the
mesh — not both) and let inner layers fail fast.

**Hidden retries you did not write:**

- Istio's default HTTP retry policy: **2 retries** on `connect-failure,refused-stream,unavailable,
  cancelled,retriable-status-codes` — on top of your application's retries.
- AWS SDKs: 3 attempts in standard mode with a token-bucket retry quota. Kafka producer `retries`
  defaults to `Integer.MAX_VALUE`, bounded by `delivery.timeout.ms` = 120s.
- Nginx `proxy_next_upstream error timeout` (default) tries the next upstream; since 1.9.13 it
  skips non-idempotent methods unless `non_idempotent` is set, but older configs and other proxies
  may retry POSTs.
- Message brokers redeliver: SQS after the visibility timeout (default 30s), RabbitMQ on channel
  close, Kafka on rebalance.
- HTTP client libraries silently retry on a stale pooled connection (Apache HttpClient's
  `DefaultHttpRequestRetryHandler`, Go's transport for idempotent requests, OkHttp
  `retryOnConnectionFailure=true`).

**Retry budgets.** Instead of "retry up to 3 times", cap retries as a *fraction* of normal traffic
(10–20% over a sliding window). In a full outage extra load is bounded at +20% instead of +200%.
Envoy: `retry_budget` in cluster circuit breakers (`budget_percent` default 20%,
`min_retry_concurrency` 3). Finagle: 20% plus a small per-second allowance. AWS SDK standard mode:
a bucket of 500 tokens, each retry costs 5 (10 for a timeout), each success refunds 1 — after about
100 failed retries, retrying stops until successes refill it.

**Client-side adaptive throttling** (Google SRE book). Each client tracks `requests` and `accepts`
over two minutes and locally rejects new requests with probability
`max(0, (requests − K·accepts) / (requests + 1))`, K=2. When the backend rejects most traffic,
clients stop sending most of it, without coordination and without the backend paying for the
rejections.

**Backoff and jitter.** Exponential backoff without jitter synchronises clients into waves. Full
jitter: `sleep = random(0, min(cap, base · 2^attempt))`. After a fleet-wide event (a deploy, a
network blip) unjittered retries arrive as a second outage. Honour `Retry-After` on 429/503.

**What not to retry:** non-idempotent operations without a key (`M16`); errors that will not change
(4xx, validation, auth); requests whose deadline is nearly spent; anything while the breaker is
open. Retry *connect* failures freely (the request never arrived); retry *read timeouts* only with
idempotency.

**Per-try timeout vs overall timeout.** With retries, set a per-try timeout (e.g. 300ms) and an
overall budget (1s). Without a per-try timeout, the first try consumes the whole budget and the
retry never runs.

**Hedged requests** (Dean & Barroso, "The Tail at Scale"). Send to one replica; if there is no reply
by the p95 latency, send a second copy to another replica, take the first answer and cancel the
other. Cost ~5% extra load; p99.9 drops sharply when tail latency comes from per-replica hiccups
(GC, compaction, noisy neighbour). Idempotent reads only. **Tied requests**: send to two replicas at
once, each tagged with the other's id; the first to *start* executing tells the other to drop it.
Hedging hurts when the tail comes from overload, so cap it with a budget. gRPC: `hedgingPolicy` in
service config; Envoy: `hedge_on_per_try_timeout`.

### 3. Connections, pools and the network layer

**Pool sizing with Little's law.** In-flight = throughput × latency. 2,000 rps at 50ms = 100
connections. If latency jumps to 500ms you need 1,000; a pool of 100 now queues 900 requests. Pool
exhaustion *looks* like the callee being slow, but the time is spent waiting for a connection
locally — measure pool wait time separately from call time.

**Database connections multiply.** 200 pods × pool 20 = 4,000 Postgres connections; each backend is
a process using several MB, and `max_connections` defaults to 100. Autoscaling pods can exhaust the
database. Fixes: PgBouncer in transaction mode (session features — `SET`, session advisory locks,
`LISTEN`, and prepared statements before PgBouncer 1.21 — break), RDS Proxy, smaller per-pod pools.
More active connections than database cores mostly adds contention.

**Ephemeral port and SNAT exhaustion.** A client opening a connection per request leaves sockets in
`TIME_WAIT` for 60s. The default ephemeral range 32768–60999 gives ~28,000 ports per destination
IP:port, so above ~470 new connections per second to one destination you get `EADDRNOTAVAIL`
("cannot assign requested address"). Behind NAT it is worse: AWS NAT gateway allows 55,000
simultaneous connections per unique destination; Azure's default SNAT allocation can be as low as
1,024 ports per instance. Fix: reuse connections. `tcp_tw_reuse=1` is safe for outgoing
connections; `tcp_tw_recycle` was removed in Linux 4.12 because it broke clients behind NAT.
Diagnose: `ss -s`, `ss -tan state time-wait | wc -l`, NAT gateway `ErrorPortAllocation`.

**conntrack.** Kubernetes nodes track every flow in `nf_conntrack`. When the table is full, new
packets are dropped silently and `dmesg` shows `nf_conntrack: table full, dropping packet`.
Symptom: random connect timeouts on one node. A related Linux conntrack race on parallel UDP
packets (A and AAAA queries from the same socket) caused Kubernetes DNS lookups to take exactly
**5 seconds** — the resolver's retry timeout. Fixes: `single-request-reopen`, NodeLocal DNSCache,
kernel patches.

**DNS in Kubernetes.** `ndots:5` means a lookup of `api.example.com` (2 dots) first tries every
search domain (`<ns>.svc.cluster.local`, `svc.cluster.local`, `cluster.local`, cloud domains) for
both A and AAAA before the real name — up to 10 queries per lookup. CoreDNS becomes a bottleneck.
Fixes: FQDN with trailing dot, lower `ndots`, NodeLocal DNSCache, caching in the client.

**DNS caching in clients.** The JVM caches successful lookups for 30s by default
(`networkaddress.cache.ttl`), or forever with a security manager. Most pools resolve only when
creating a connection, so a DNS-based failover (RDS Multi-AZ flips the CNAME) is invisible to
existing pooled connections until they break. Set a maximum connection lifetime (HikariCP
`maxLifetime` default 30 min) shorter than any infrastructure limit.

**HTTP/2 and gRPC load balancing.** gRPC multiplexes all requests on one long-lived HTTP/2
connection. A Kubernetes `ClusterIP` Service balances *connections* (kube-proxy picks a backend at
connect time), so every request on a connection goes to the same pod. Pods added by autoscaling get
no traffic; one pod runs hot while others idle. Fixes: client-side balancing over a headless
Service (`round_robin` policy), an L7 proxy/mesh that balances per request, or server-side
`MAX_CONNECTION_AGE` (e.g. 5 min plus grace), which sends GOAWAY and forces reconnection. gRPC
servers add ±10% jitter to the age so connections do not all expire together; clients must also
reconnect with jittered backoff, or a server rollout (every connection GOAWAY'd at once) becomes a
synchronised reconnect storm against the new pods.

**HTTP/2 specifics.** `SETTINGS_MAX_CONCURRENT_STREAMS` (commonly 100; Go's server 250) caps
in-flight requests per connection; beyond it, depending on the library, the client queues silently
or opens a new connection. One TCP connection also means TCP head-of-line blocking: a lost packet
stalls every multiplexed stream.

**Half-open connections.** When a node dies without FIN/RST (power loss, security group change, NAT
state loss), clients keep "healthy" pooled connections to nowhere. Writes succeed into the kernel
buffer; reads hang until the read timeout or `tcp_retries2`. Protections: read timeouts,
`TCP_USER_TIMEOUT`, application pings (gRPC keepalive, HTTP/2 PING), validating idle connections.

**MTU black holes.** Tunnels (VPN, VXLAN overlays, IPsec, some CNIs) shrink the MTU. If ICMP
"fragmentation needed" is filtered, small requests work and large responses hang: the health check
passes, the TLS handshake may or may not complete, and the big JSON body never arrives. Fix: MSS
clamping, allow ICMP type 3 code 4, lower the interface MTU.

### 4. Load balancing and Kubernetes lifecycle pathologies

**Round-robin sends equal work to unequal servers.** A slow instance still gets its share and
builds a queue. Least-outstanding-requests or **power of two choices** (pick two random backends,
send to the one with fewer in-flight) adapts; P2C avoids the herd effect of pure least-connections,
where every client picks the same "emptiest" host at once.

**The fast-failing server wins the load balancer.** A least-requests balancer favours the host that
finishes fastest — including a broken host returning 500 in 1ms. It sucks traffic into a black
hole. Guards: outlier detection and weighting by success, not only latency.

**Envoy outlier detection defaults.** `consecutive_5xx: 5`, `interval: 10s`, `base_ejection_time:
30s` (multiplied by times ejected), `max_ejection_percent: 10`. With 10%, a fleet of 5 hosts can
eject at most one (Envoy always allows ejecting at least one host). It is passive — based on real
traffic — so a host receiving little traffic is ejected slowly. Each Envoy decides independently.

**Panic threshold.** Envoy's `healthy_panic_threshold` defaults to **50%**: when fewer than half the
hosts are healthy, Envoy ignores health and balances across *all* hosts, including unhealthy ones.
Deliberate — better than crushing the healthy half — but surprising during an incident. AWS ALB
fails open similarly when all targets are unhealthy.

**Slow start and warm-up.** A fresh JVM runs interpreted code until the JIT compiles hot paths;
caches and pools are cold. A full traffic share on start gives a latency spike every deploy. Fixes:
slow-start mode (ALB `slow_start.duration_seconds`, Envoy `slow_start_config`), synthetic warm-up
before readiness passes. Slow start does nothing when *all* instances are new at once.

**Zone-aware routing and cross-zone cost.** AWS charges $0.01/GB in each direction for cross-AZ
traffic; with random balancing in 3 AZs, ~2/3 of calls cross a zone. Kubernetes topology-aware
routing and Envoy zone-aware routing keep traffic local, but with **uneven zone capacity** they
overload the smaller zone. ALB cross-zone balancing is on by default; NLB is off by default — an NLB
with 2 targets in AZ-a and 8 in AZ-b sends 50% of traffic to each zone, so AZ-a targets carry 4×
the load of AZ-b targets.

**Connection stickiness after deploys.** Long-lived connections (gRPC, WebSockets, DB pools) stay
where they were opened. After a rolling restart the first-restarted instance has the most
connections, the last-restarted the fewest. Fix: periodic recycling (max connection age).

**The termination race.** On pod deletion, *in parallel*: the kubelet runs preStop and sends
SIGTERM, and the endpoint controller removes the pod from EndpointSlices, which then propagates to
kube-proxy on every node, ingress controllers and sidecars (can take seconds). An app that exits
immediately on SIGTERM receives requests after it closed its listener → connection refused / 502
on every deploy. Fix: `preStop` sleep 5–15s (native `sleep` action since Kubernetes 1.29), then
graceful shutdown: stop accepting, drain in-flight, close keep-alive connections (`Connection:
close`, GOAWAY), exit. `terminationGracePeriodSeconds` (default 30s) includes the preStop time;
after it comes SIGKILL (exit code 137).

**Liveness vs readiness.** Liveness restarts the container; readiness removes it from endpoints. A
liveness probe that checks the database turns a database blip into a restart storm — restarted pods
warm up, reconnect all at once and make it worse. Liveness should answer only "is this process
wedged?". Readiness may consider dependencies, but if every pod fails readiness together the
Service has zero endpoints and a partial outage becomes total. Probe timeouts default to 1s: a pod
under GC or CPU throttling fails probes while healthy.

**Health checks lie both ways.** A constant `/health` 200 misses a full disk, exhausted pool or
deadlocked worker (grey failure, `M08`). A deep check that calls dependencies creates a correlated
fleet-wide failure. Compromise: health reflects *local* ability to serve (pool not saturated, event
loop lag, recent success rate); dependency failure is handled by breakers.

**Pod disruption and eviction.** A `PodDisruptionBudget` protects against voluntary disruption
(node drains) but not against OOM kills or node failure. With `maxUnavailable: 0` or
`minAvailable` equal to replicas, node drains hang forever and cluster upgrades stall. Memory limit
exceeded → `OOMKilled` regardless of JVM heap settings if off-heap (direct buffers, metaspace,
thread stacks) is not counted; set `-XX:MaxRAMPercentage` around 70–75%, not 100%.

### 5. Overload, load shedding and metastable failure

**Queueing near saturation.** For an M/M/1 queue, time in system = service time / (1 − ρ). At 50%
utilisation latency is 2× service time; at 90%, 10×; at 95%, 20×. Latency is flat until a knee and
then explodes.

**Goodput vs throughput.** Under overload a server may complete as many requests as ever while
every response arrives after the client has timed out — throughput high, goodput zero. Fixes, in
roughly the order they help:

1. **Bounded queues** — reject when full rather than wait (`M30`).
2. **Deadline-aware dropping** — discard requests whose propagated deadline has passed (§1).
3. **Adaptive LIFO + CoDel** (Facebook). Under congestion serve newest first — their clients are
   still waiting. CoDel: if the *minimum* queueing delay over an interval exceeds a target (e.g.
   5ms), switch to a short queue timeout (e.g. 10ms) instead of the normal one (e.g. 100ms).
4. **Concurrency limits rather than rate limits.** A requests-per-second limit is wrong as soon as
   latency changes; a limit on in-flight requests adapts via Little's law. **Adaptive concurrency**
   (Netflix `concurrency-limits`, Vegas/gradient algorithms; Envoy `adaptive_concurrency`) learns
   minimum latency and shrinks the limit when latency rises; excess gets an immediate 503/429.
5. **Priority shedding.** Tag requests by criticality (Google: `CRITICAL_PLUS`, `CRITICAL`,
   `SHEDDABLE_PLUS`, `SHEDDABLE`) and shed from the bottom — batch, prefetch, analytics, retries —
   before checkout. Criticality propagates with the request like a deadline.
6. **Cheap rejection.** The rejection path must cost far less than serving. If rejecting still
   parses the body, authenticates via a remote call or logs a stack trace, a server at 10× load dies
   anyway. Shed at the earliest point: edge, accept queue, first middleware.

**Circuit breakers in practice (`M10`).** Resilience4j defaults: failure-rate threshold 50% over a
count window of 100 calls, `minimumNumberOfCalls` 100, open for 60s, 10 permitted calls in
half-open. Pitfalls: low-traffic endpoints never reach the minimum and never trip; a breaker per
*service* opens for all hosts because one host is bad; clients going half-open together stampede
the recovering service; a breaker that counts 4xx as failures opens on a client bug. Envoy
"circuit breakers" are **concurrency limits** per cluster: `max_connections`,
`max_pending_requests`, `max_requests` default **1024**, `max_retries` **3**. Hitting them returns
503 with flag `UO` *without contacting the upstream* — often misread as the upstream failing.

**Bulkheads.** Separate pools per dependency. The classic story: one 200-thread pool, five
dependencies, one dependency slows to 10s, within seconds all 200 threads wait on it and the four
healthy features die too. In async runtimes, use per-dependency semaphores.

**Metastable failure** (Bronson et al., 2021). A system has a good stable state and a bad stable
state. A *trigger* (latency spike, cache flush, deploy) pushes it into the bad state; a *sustaining
effect* keeps it there after the trigger is gone:

- **Retry storms** — timeouts cause retries, which add load, which cause timeouts.
- **Cache loss** — a cold cache sends load to the database, which is too slow to refill the cache.
- **Slowness amplification** — slow requests hold locks and connections longer, reducing capacity.
- **GC spirals** — queued requests grow the heap, GC slows everything, more requests queue.

The signature: *the trigger is removed and the system does not recover*; rolling back changes
nothing. The fix is to remove the sustaining effect: shed hard at the edge (let through e.g. 20%),
disable retries, let queues drain, then ramp traffic back in steps. Design against it with retry
budgets, bounded queues, LIFO under overload, static stability and cache warm-up.

**Autoscaling does not save you from spikes.** HPA syncs every 15s; metrics lag 30–60s; then pod
scheduling, image pull, start and warm-up — 2–5 minutes to real capacity, plus 1–3 minutes if a node
is needed. Scale-down stabilisation defaults to 300s. CPU-based scaling misleads: a service blocked
on a slow dependency has low CPU and high latency so it does not scale (and scaling would only add
callers to the slow dependency). Scale on concurrency or queue depth, keep headroom, pre-scale
for known events.

**CPU limits and CFS throttling.** A limit of 1 CPU means 100ms of CPU time per 100ms CFS period
across all threads. A JVM with 20 busy threads burns 100ms in 5ms of wall time and is then
**frozen for 95ms**. Average CPU shows 30% while p99 has ~100ms steps. Diagnose via
`container_cpu_cfs_throttled_periods_total`. Fixes: no CPU limits (keep requests) for latency-
sensitive services, thread pools sized to the limit, `GOMAXPROCS` matching the limit (Go reads host
cores before 1.25; `automaxprocs`), JVM container awareness (`-XX:ActiveProcessorCount`).

**Cache stampede.** A hot key expires and thousands of requests miss together. Fixes: **request
coalescing** (Go `singleflight` — one fetch per key per instance), **probabilistic early refresh**
(XFetch), **stale-while-revalidate**, a lease on miss (only one client may fill), jittered TTLs so
keys do not expire together. After a full cache flush or a fresh cluster, warm the cache or ramp
traffic.

**Static stability (`M31`).** A system is statically stable if it keeps working during a dependency
failure *without making changes*: capacity pre-provisioned in every zone for zone loss rather than
autoscaling during the event (the control plane you need may be impaired); data planes keep the
last-known-good config when the config service is down. Static stability usually has a **time
limit** — cached certificates, tokens, DNS entries and leases expire. Know the time-to-failure of
every cached credential and alert long before it.

**Backlogs and recovery.** After an outage, drain time = backlog ÷ (consumer concurrency ÷ per-item
latency). 900,000 items at 5s each with 100 workers is 12.5 hours. During recovery: expire work
that is no longer useful, serve new user-facing work first (LIFO for interactive, FIFO for
ordered pipelines), add consumer capacity only up to what the downstream can absorb, and rate-limit
catch-up so a lagging producer (an outbox relay, a CDC connector) does not emit hours of events in
minutes. Bulk work (backfills, replays) belongs on separate topics/queues with separate consumer
capacity, so "low priority" is enforced by the system rather than stated in a ticket.

**Recovery oscillation.** Returning capacity (a zone coming back, a restarted fleet) is cold. If
routing snaps all its share back at once, it slows, fails readiness, is removed, and the survivors
overload again — traffic flaps. Ramp returning capacity gradually (slow start, weight ramps) and
damp routing decisions (hysteresis: require sustained health before re-adding).

**Autoscaling can move the bottleneck.** Scaling consumers on lag or API pods on CPU multiplies
connections and concurrency against shared downstreams (a database, a partner API). Cap
autoscaler maximums by downstream capacity, and bulkhead shared databases per client team.

### 6. Service mesh, sidecars and proxies

**Envoy response flags** — read these first in a mesh access log:

| Flag | Meaning |
|---|---|
| `UH` | no healthy upstream hosts |
| `UF` | upstream connection failure |
| `UO` | upstream overflow (circuit breaker / concurrency limit) |
| `URX` | rejected after retry limit or max connect attempts |
| `UC` | upstream connection terminated (often the keep-alive race) |
| `UT` | upstream request timeout |
| `NR` | no route configured |
| `DC` | downstream (client) connection terminated |
| `LR` | connection local reset |

Low-rate `503 UC` with "upstream connect error or disconnect/reset before headers. reset reason:
connection termination" is the keep-alive race of §1: the app closed an idle pooled connection
Envoy reused. Fix: app idle timeout longer than Envoy's upstream `idle_timeout` (set via
`DestinationRule` `connectionPool.http.idleTimeout`), or retry on `reset` for idempotent calls.

**Sidecar lifecycle problems.**

- *Startup race*: the app starts before Envoy is ready; its first outbound calls fail. Istio
  `holdApplicationUntilProxyStarts: true`.
- *Shutdown race*: on SIGTERM both containers stop; if Envoy exits first, the app's draining and
  in-flight outbound calls fail. Istio `EXIT_ON_ZERO_ACTIVE_CONNECTIONS`,
  `terminationDrainDuration`.
- *Jobs never finish*: a Job's pod completes only when all containers exit; a classic sidecar never
  exits. Native sidecars (init containers with `restartPolicy: Always`, on by default from
  Kubernetes 1.29) fix start and stop ordering.
- *Init containers* run before the sidecar and cannot reach the network once `istio-init` has
  installed iptables redirection.

**Control-plane scale.** By default every sidecar gets config for every service in the mesh. With
thousands of services each Envoy holds hundreds of MB, every change pushes to every proxy (push
storms, istiod CPU spikes, sidecars OOMKilled). The `Sidecar` resource restricting egress hosts per
namespace cuts config by orders of magnitude. If istiod is down the data plane keeps running on
last-known config (statically stable), but new pods cannot get certificates or config.

**mTLS and certificates.** Istio workload certificates default to **24h** and rotate
automatically. Root or intermediate CA expiry is the classic mesh-wide outage — nobody remembers
the date. `PERMISSIVE` mode accepts plaintext and mTLS; needed during migration, a hole if left on.
A `DestinationRule` with `ISTIO_MUTUAL` towards a workload without a sidecar breaks it.
Switching a namespace to `STRICT` breaks every non-mesh caller at once; migrate by watching which
callers still arrive in plaintext (Istio telemetry `connection_security_policy`) before flipping.

**Certificate rotation outside the mesh.** Renewing a server certificate can change the issuing
intermediate or root. Clients with an old truststore (a JVM `cacerts` baked into an image years
ago), pinned certificates, or a missing intermediate in the served chain fail while others work.
Treat certificate changes as deploys: test against every client runtime, serve the full chain,
and roll out gradually.

**Per-hop remote auth.** Calling an auth service on every internal hop multiplies its latency and
load by the depth of the call chain; a 35ms slowdown at 12 hops is 420ms plus held connections.
Validate signed tokens locally (JWT with cached JWKS keys) and pass identity downstream.

**Protocol detection.** Istio infers protocol from the port name (`http-`, `grpc-`, `tcp-`) or
`appProtocol`. A misnamed port makes HTTP look like opaque TCP: no retries, no per-request
balancing, no L7 metrics — a gRPC service silently reverts to connection-level balancing.
Server-first protocols (MySQL, SMTP) break with sniffing because the server speaks first.

**Mesh cost.** Each sidecar hop adds roughly 1–3ms at p50 and more at p99; a chain of 6 services is
12 proxy traversals. Sidecar CPU and memory multiply by pod count. Ambient mode (per-node ztunnel for
L4, waypoint proxies for L7) trades some of this.

**Double policies.** The mesh adds timeouts, retries and breakers on top of the application's. A
mesh `perTryTimeout` 1s with 2 retries under an app timeout of 2s means the app gives up while the
mesh's third attempt is still running. Choose one owner per policy.

**Diagnostics.** `istioctl proxy-config clusters|endpoints|routes|listeners <pod>`, `istioctl
proxy-status` (SYNCED/STALE), `istioctl analyze`, Envoy admin `localhost:15000/stats`
(`upstream_rq_pending_overflow`, `upstream_cx_destroy_remote_with_active_rq`, `upstream_rq_retry`,
`outlier_detection.ejections_active`), `/clusters` (per-host health), `/config_dump`.

### 7. Messaging, duplicates and ordering

**Kafka consumer group mechanics that cause incidents.**

- `max.poll.interval.ms` default **300,000** (5 min). If processing one `poll()` batch
  (`max.poll.records` default **500**) takes longer — 500 records × 1s slow downstream — the
  consumer is expelled, its partitions reassigned, uncommitted records reprocessed by another member
  that is equally slow: a **rebalance loop** with no progress and growing duplicates. Fix: fewer
  records per poll, pause partitions and process asynchronously, or raise the interval.
- `session.timeout.ms` default **45s** (Kafka 3.0+) with heartbeats from a background thread: it
  detects dead processes, not stuck ones.
- Eager rebalancing stops the whole group on any membership change; a rolling deploy of 50
  consumers means dozens of stop-the-world rebalances. Use `CooperativeStickyAssignor` and **static
  membership** (`group.instance.id`) so a restart within the session timeout does not rebalance.
  Kafka 4.0's consumer protocol (KIP-848) moves assignment to the broker.
- Auto-commit (every 5s) commits offsets of records *polled*, not *processed*. With async
  processing after poll, a crash loses messages.
- Alert on **lag in time** (age of the oldest unprocessed message), not offsets.

**Ordering is per partition and easily broken.** With retries and more than one in-flight request,
a failed batch retried after a later batch succeeded reorders — unless `enable.idempotence=true`
(default since Kafka 3.0), which preserves order with up to 5 in flight. Consumers processing a
partition on a thread pool reorder by design; hash keys to per-key queues inside the consumer for
parallelism with per-key order. Increasing the partition count remaps keys (`hash(key) %
partitions`): new events for a key land on a different partition from its old ones, and consumers
can process them out of order during the transition.

**Poison messages.** One message that always fails blocks its partition (a log has no per-message
ack). Retry a bounded number of times, then dead-letter and move on — but if per-key order
matters, skipping one event and processing later ones for the same key breaks invariants; park *the
key* (route its later events to the retry path too until the head is resolved). SQS:
`maxReceiveCount` on the redrive policy; FIFO queues block the message group, not the queue.

**SQS visibility timeout.** Default 30s. If processing takes longer, the message reappears and a
second worker processes it concurrently. Extend via `ChangeMessageVisibility` heartbeats and be
idempotent anyway. FIFO deduplication (`MessageDeduplicationId`) covers a **5-minute** window only.

**RabbitMQ.** Unbounded prefetch lets one consumer grab 100,000 messages into memory; when it dies
all are redelivered; a slow consumer with a large prefetch hoards work others could do. Memory or
disk alarms **block all publishers** on the node — a slow consumer on one queue stops publishing to
unrelated queues. Quorum queues have a delivery limit (default 20 since 4.0) against poison
messages.

**Exactly-once is a scope (`M16`).** Kafka transactions give exactly-once for consume-from-Kafka,
produce-to-Kafka within one transaction, read with `isolation.level=read_committed`. Any side
effect outside Kafka is at-least-once unless the consumer is idempotent. `read_committed` consumers
cannot read past the **last stable offset**: one producer with a hung open transaction stalls every
such consumer of that partition until `transaction.timeout.ms` (default 60s; broker max 15 min).

**Idempotent consumers.**

- A processed-message table keyed by message id, inserted **in the same database transaction** as
  the business write (`INSERT ... ON CONFLICT DO NOTHING`, check the row count).
- Natural idempotency: `SET status = 'PAID' WHERE status = 'PENDING'`, versioned updates, not
  `balance = balance + 10`.
- Keep dedup records longer than the maximum redelivery/replay window.
- Dedupe by *business key* when the same fact can arrive with different message ids (a relay that
  re-publishes with fresh ids, two producers emitting the same fact).

**API idempotency keys.** Store key → (request hash, state, response). States: *in progress* (a
concurrent duplicate gets 409 or waits), *completed* (replay the stored response), *failed and
retryable*. Reject a reused key with a different body. Scope keys by client or tenant. Record the key
**before** calling external providers and pass a derived key downstream (Stripe's
`Idempotency-Key`, kept at least 24h). A key in Redis with a short TTL while the ledger is in
Postgres is not idempotency: a Redis failover or eviction forgets it.

**Replays.** Resetting a consumer group to fix a bug re-sends days of events to side effects —
emails, webhooks, payments. Replays must pass the same idempotency layer, or side-effecting
consumers need a replay mode that suppresses external actions.

### 8. Cross-service consistency: dual writes, outbox, CDC, sagas

**The dual-write bug.** `db.save(order); kafka.send(OrderCreated)` fails two ways: DB commits and
the send fails (event lost), or the send succeeds and the transaction rolls back (a ghost event for
something that does not exist). Sending inside the transaction does not help. Only outbox, CDC or
event sourcing make state and event atomic (`M15`).

**Outbox relay — the sequence-gap bug.** A relay polling `WHERE id > :last ORDER BY id` misses rows.
Ids are assigned at insert, but transactions **commit in a different order**: tx A takes id 100, tx
B takes 101 and commits first; the relay reads 101, advances to 101, then A commits and 100 is never
published. Fixes: read in commit order (WAL-based CDC does this); `published` flag with
`SELECT ... FOR UPDATE SKIP LOCKED`; lag the high-water mark behind by a safety window and dedupe;
in Postgres, only read rows whose `xmin` is below `pg_snapshot_xmin(pg_current_snapshot())`.
Multiple relay instances without locking publish every row several times; a single relay without
per-aggregate ordering may publish out of order when it parallelises.

**CDC (Debezium) footguns.**

- A Postgres logical replication slot retains WAL until the consumer confirms it. If the connector
  stops (failed, or deleted without dropping the slot), **WAL grows until the disk fills and the
  primary stops accepting writes**. Guard with `max_slot_wal_keep_size` (PG 13+; the slot is
  invalidated instead) and alert on slot lag in `pg_replication_slots`.
- If captured tables are quiet but the database is busy elsewhere, the slot's confirmed position
  does not advance and WAL accumulates; Debezium `heartbeat.interval.ms` with a heartbeat-table
  write (`heartbeat.action.query`) fixes it.
- Logical slots are not carried over to a promoted standby before PG 17 (failover slots). After a
  failover Debezium must re-snapshot or has a gap.
- The initial snapshot of a huge table holds a long transaction (vacuum blocked, bloat); use
  incremental snapshots via the signal table.
- CDC leaks the internal schema: a column rename becomes a breaking change for every consumer. The
  outbox publishes an explicit contract instead.
- Debezium is at-least-once: after a crash, events since the last committed offset are re-emitted.

**Sagas in production (`M14`).** Compensation is a new forward action (refund, release, apology
email), not an undo. What only shows in production:

- *Compensations fail too* — they need retries and, eventually, a human queue.
- *The pivot* — order steps so the irreversible one (charge, ship) comes after the steps likely to
  fail (reserve stock, validate address).
- *Semantic locks* — mark records `PENDING` so other flows do not act on half-finished state.
- *Timeouts are "unknown", not "failed"* — resolve by querying the participant; the payment may
  have succeeded.
- *Durable orchestration state* (Temporal, Step Functions, a saga table) — an in-memory
  orchestrator loses every in-flight saga on deploy.
- *Versioning in-flight workflows* — a saga started on code version N may resume on N+1; workflow
  engines need explicit versioning (Temporal `GetVersion`/patching) or deterministic replay breaks.

**Late replies.** A participant may answer after the saga has timed out and compensated (the
fraud check approves an order already cancelled). Every participant action and every reply must be
checked against the saga's current state: late replies are ignored or trigger their own
compensation; participants check "is this order still active?" before irreversible actions.

**Read models, replicas and read-your-writes (`M19`, `M23`).** A projection fed by events lags the
write model. For read-your-writes without making everything synchronous: return the write's
version (or log position) to the client; the read side waits briefly until the projection reaches
that version, or falls back to the write model for that user's next few reads. Projections must be
**rebuildable** (replay retained events, or snapshot plus CDC, into a new index, then swap
atomically). Caches populated from lagging replicas pin stale data *beyond* the replication lag —
fill caches from the primary or check versions before overwriting. When a service needs another
service's data on its hot path (credit limit, prices), keeping a local replica fed by events
decouples availability at the cost of bounded staleness; record the replica's age and refuse or
degrade if it is too old. A read model partitioned by a different key (orders by seller when the
write side shards by customer) replaces scatter-gather queries.

**Reconciliation is the real safety net.** Any system with money or inventory across services needs
a job comparing both sides (our orders vs the provider's settlement report; reservations vs
warehouse counts) that routes mismatches to automatic repair or humans. Real-time logic aims to be
right; reconciliation proves it and catches its bugs. Mentioning it unprompted is a strong senior
signal.

**Dual-run migrations (`M28`).** When replacing a system, keep the old one as source of truth,
mirror writes to the new one, compare reads in the background (**shadow/dark reads**), and track a
mismatch dashboard by category. Cut reads over when mismatches are near zero and explained.
Twitter's Diffy compared old vs new *and* old vs a second old instance to filter nondeterministic
noise (timestamps, random ids, ordering of sets).

### 9. Time, leases, leader election and split-brain

**Clocks are not reliable (`M21`).** NTP usually keeps hosts within milliseconds, but VMs pause,
NTP can *step* the clock backwards, and a misconfigured host can be minutes off. Consequences:
last-write-wins by wall-clock silently drops newer writes from a host with a slow clock; tokens
rejected as "not yet valid" from a host a few seconds ahead; TTLs computed from wall time expire
early. Use monotonic clocks for durations, allow 30–60s leeway on token validation, and order by
logical versions. Leap seconds: Google and AWS **smear** over 24h; mixing smeared and unsmeared
sources gives up to 0.5s disagreement.

**Leases and the pause problem.** A lease lock (Redis `SET NX PX`, ZooKeeper ephemeral node, etcd
lease) holds only while belief matches reality. Client A acquires a 10s lease, pauses 15s (GC, VM
migration, CFS throttling), the lease expires, B acquires and writes, A wakes still believing it
holds the lock and writes too. No client-side check prevents this: the pause can fall between the
check and the write. **Fencing tokens**: the lock service returns a monotonically increasing number
with every grant (ZooKeeper `zxid`, etcd revision, a DB sequence); the protected resource rejects
writes carrying a lower token than the highest seen. If the resource cannot check tokens, the lock
is an efficiency optimisation, not a correctness guarantee. This is the core of Kleppmann's
critique of Redlock.

**Kubernetes leader election** (client-go): `leaseDuration` 15s, `renewDeadline` 10s,
`retryPeriod` 2s. A leader that cannot renew must stop acting — only if the code checks. A slow API
server (etcd latency) makes every controller lose leadership at once.

**ZooKeeper and etcd.** A ZooKeeper session timeout must exceed the worst GC pause, or a paused
healthy leader loses its ephemeral nodes and triggers failover. etcd is sensitive to fsync latency
(`etcd_disk_wal_fsync_duration_seconds` p99 should be under ~10ms); on slow shared disks leader
elections flap and the Kubernetes API becomes unavailable.

**Split-brain (`M33`).** Two nodes both act as primary. Causes: failover without quorum; a paused
primary that resumes; a promotion script that does not **fence** the old primary (STONITH: cut its
storage or network, or have it self-demote when it loses contact with a quorum). Two-node clusters
cannot have a majority; add a witness. Recovery means reconciling divergent writes, often by hand:
diff both histories by key, apply business precedence rules, escalate true conflicts to people, and
state the data loss honestly. Tools like Patroni avoid this by holding leadership as a lease in a
consensus store (etcd, Consul, ZooKeeper): a primary that cannot renew the lease demotes itself.

**Multi-region writes.** Active-active with asynchronous replication and last-write-wins by wall
clock loses concurrent updates silently (an address change reverted after a partition). Options:
a **home region per record** (single writer per key; other regions forward writes), version
vectors with explicit conflict resolution, or CRDTs where the data type allows merging (counters,
sets). Order causally related events by a per-aggregate sequence or **hybrid logical clocks**, not
by `created_at` from different hosts. Rate limiters and time windows computed on each node's clock
disagree at boundaries; use one clock source (e.g. the Redis server's `TIME`) or sliding windows.

**Ids after failover.** Sequences that are not durably replicated can be reissued by a promoted
replica. Prefer ids that do not depend on one primary's sequence — UUIDv7/ULID (time-ordered,
index-friendly) or ranges allocated in blocks — and bump sequences past the old high-water mark
after any failover.

**Failover is where data is lost.** Async promotion loses writes not yet replicated (RPO > 0);
clients connected to the old primary keep writing until fenced; caches keep serving the old
primary's values; id sequences on the new primary may reissue ids already handed out and stored by
other services. Kafka: `unclean.leader.election.enable=false` (default) picks unavailability over
losing committed data; `acks=all` with `min.insync.replicas=2` and RF=3 is the durable baseline —
with `min.insync.replicas=1`, `acks=all` means "the leader alone".

**Schedulers and time boundaries.** "02:30 local" does not exist on spring-forward and happens
twice on fall-back — schedule in UTC. A scheduler in every replica runs a job N times. Kubernetes
`CronJob` `concurrencyPolicy: Allow` (default) overlaps long runs; if more than 100 schedules are
missed (controller outage) and `startingDeadlineSeconds` is unset, the CronJob stops scheduling.
Everyone scheduling at `0 * * * *` creates top-of-the-hour spikes on shared dependencies; add
jitter.

### 10. Contracts, schema evolution and deployment

**Tolerant readers vs strict deserialisers.** Adding a field is backward compatible only if every
consumer ignores unknown fields. Jackson's `FAIL_ON_UNKNOWN_PROPERTIES` defaults to **true**
(Spring Boot's auto-configured `ObjectMapper` sets it false; a hand-built `new ObjectMapper()` does
not). New enum values are worse: consumers mapping to a closed enum throw, `switch` statements fall
through, TypeScript exhaustiveness lies at runtime. Protobuf 3 keeps unknown enum values as the
integer, surfaced as `UNRECOGNIZED` in Java, which business logic often mishandles.

**Protobuf rules.** Never reuse or renumber a field (mark removed ones `reserved`); int32→int64 is
wire-compatible, int32→string is not; a field the producer stops sending reads as its default (0,
"", false) — which may be a valid business value ("price 0", "is_active false"). Without `optional`
or wrapper types, default and unset are indistinguishable.

**Expand–contract.** For any contract or schema change across independent deploys: expand (add the
new alongside the old, write both), migrate readers, contract (remove the old), each step separately
deployable and reversible. The previous version must be able to read everything the new version
writes ("N−1 compatibility"), or rollback is impossible once the new version has written data.

**Schema registry modes.** Confluent default `BACKWARD`: the new schema can read data written with
the previous one — upgrade consumers first. `FORWARD`: old readers can read new data — upgrade
producers first. `FULL`: both. `*_TRANSITIVE` checks against every version, needed when consumers
replay old data from a long-retention log.

**Deploys and config.** Canary with automated comparison against a baseline running the old
version *at the same time*; bake long enough to cross the daily peak and batch jobs; roll out by
zone or cell. **Config and feature-flag changes are deploys**: many of the largest public outages
came from a global config push (a WAF regex, a malformed file propagated everywhere in seconds).
Flags need percentage and cell rollout too, and a kill switch that does not depend on the failing
system.

**Rollback may not be safe.** If the new version ran a destructive migration, emitted events in a
new format consumers stored, or changed cached data, rolling back the code does not roll back the
world. Roll forward, or keep every release rollback-safe via expand–contract.

**Shared caches across versions.** During a rolling deploy old and new versions share Redis. New
writes a serialised object with a new shape; old reads fail, or silently drop the field and write
back without it. Version the key (`user:v2:{id}`) when the shape changes.

**Shadow traffic.** Mirror production requests to the new version (Envoy `request_mirror_policies`,
Istio `mirror` + `mirrorPercentage`) with responses discarded; compare offline. Shadows must not
cause side effects — stub writes or mirror only reads — and they add load to shared downstreams.
Envoy appends `-shadow` to the Host header of mirrored requests.

### 11. Partitioning, hot keys, multi-tenancy and cells

**Hot keys (`M32`).** Hash partitioning spreads *keys*, not *load*: a celebrity, a whale tenant or
a viral product lands all its traffic on one shard. Kafka: one key → one partition → one consumer
thread; a tenant producing 40% of events caps the pipeline at one consumer's speed. DynamoDB: a
partition handles about 3,000 RCU and 1,000 WCU; adaptive capacity helps but one item's throughput
remains bounded. Fixes: **key splitting** (`key#0..N-1`, merge on read; loses per-key order and
costs fan-out reads); **replicate hot reads** (local in-process L1 cache with a short TTL, or copies
on several cache nodes); **detect** (Redis `--hotkeys` with an LFU policy, client-side sampling,
count-min sketch); **isolate** the top tenants on dedicated partitions.

**Consistent hashing in practice.** Without virtual nodes load varies widely; with 100–200 vnodes
per node it is within a few percent. Resharding moves data that competes with live traffic.
**Consistent hashing with bounded loads** caps each node at (1+ε) × average and spills to the next
— protects against hot keys at some cost to cache locality. Adding one cache node to a
modulo-hashed cluster (`hash % N`) remaps almost every key: effectively a full cache flush.

**Noisy neighbours (`M29`).** Shared pools let one tenant's burst consume connections, threads,
queue positions and I/O. Controls: per-tenant rate *and* concurrency limits; **weighted fair
queuing** (round-robin across tenants, not FIFO across jobs — one tenant's million-job import must
not delay everyone else's single job); per-tenant cost attribution; placement of big tenants in
their own cells or shards.

**Shuffle sharding (`M31`).** With 8 workers and each tenant on a random 2, there are C(8,2) = 28
pairs; a poison tenant that kills its 2 workers fully takes down only tenants sharing *both* — 1/28
of tenants. With 100 nodes and shards of 5, C(100,5) ≈ 75 million combinations. Requires clients
that retry across the members of their shard.

**Tenancy models at scale.** Schema-per-tenant is attractive for isolation but at thousands of
tenants the catalogue bloats, migrations run for hours (one per schema), and connection pools
cannot be shared across schemas cheaply. The usual end state is hybrid: small tenants in shared
tables with a `tenant_id` (enforced by row-level security), large tenants in their own database or
cell. Rate limiters are hot keys too: a single Redis counter for a tenant with 20,000 users behind
one API key serialises on one shard — use local token buckets with periodic sync, sharded
counters, or hierarchical limits (tenant then user) evaluated in-process.

**Rebalancing costs.** Kafka partition reassignment and cache/shard rebalancing copy data over the
same disks and network as live traffic; throttle it (`kafka-reassign-partitions --throttle`, which
sets `leader/follower.replication.throttled.rate`), move a few partitions at a time, and ramp new
nodes' share while they warm.

**N+1 fan-out and polling.** Rendering a list by calling another service once per item multiplies
load and tail probability; batch the lookup, cache it, or denormalise into a read model. Thousands
of clients polling a hot value every second should hit a micro-TTL (~1s) cache with request
coalescing, or be switched to push (SSE/WebSocket) — the source of truth should see a handful of
reads per second.

**Cells.** A cell is an independent copy of the stack serving a subset of customers, behind a thin
router mapping customer → cell. A bad deploy or poison request hits one cell. Hard parts: the router
is global (keep it trivial, cache the map, statically stable); cross-cell operations (moving a
customer, global views); slower, cell-by-cell releases; fragmented capacity.

**Fan-out and tail latency.** With N parallel leaf calls each 1% likely to exceed p99, the chance
that a request is slow is 1 − 0.99^N: 10% at N=10, 63% at N=100. Mitigations: hedging (§2),
partial results at a deadline ("answer with 97 of 100 shards"), fewer and larger shards on
latency-critical paths, reducing variance in leaves.

### 12. Observability and debugging

**Percentiles lie in specific ways.** You cannot average p99s across hosts or windows — merge
histograms. A server-side p99 misses client-side time (pool wait, DNS, TLS, retries). **Coordinated
omission**: a load tester that waits for each response before sending the next stops sending during
a stall, so the stall counts as one slow sample instead of hundreds; use constant-arrival-rate
tools (wrk2, k6 arrival-rate executors). Averages hide bimodality: 95% hits at 1ms plus 5% misses
at 200ms averages 11ms, and no request takes 11ms.

**Tracing gaps (`M25`).** Context propagation breaks at thread-pool hand-offs, async callbacks,
message queues (trace context must go into message headers) and uninstrumented libraries. Head
sampling at 1% drops 99% of the rare errors you most need; **tail-based sampling** (OpenTelemetry
Collector `tail_sampling`) keeps errors and slow traces but needs all spans of a trace on the same
collector (route by trace id). Cross-host clock skew makes children appear to start before parents.

**Metric cardinality.** Labels with user ids, raw URLs or high-churn pod names create millions of
series; Prometheus memory explodes and queries time out — the monitoring fails during the incident.
Use route templates and bounded label sets.

**Logging can cause the outage.** Synchronous logging to a slow disk or shipper blocks request
threads; an error logged with a stack trace per request at 10k rps saturates CPU and fills disks.
Use async appenders with bounded queues that *drop* under pressure, and rate-limit repeated errors.

**SLO burn-rate alerting (`M34`).** Alert on how fast the error budget burns. Google's multi-window
recommendation: page when burn rate exceeds 14.4 over both 1h and 5m (2% of a 30-day budget in an
hour); page at 6 over 6h and 30m; ticket at 1 over 3 days and 6h. The short window makes the alert
clear quickly after recovery. **Monitoring must not share fate** with what it monitors: dashboards,
alerting and chat-ops in the same cluster as production go dark in a cluster-wide failure. Keep an
external synthetic probe and a dead-man's switch (an always-firing heartbeat alert whose
*absence* pages, e.g. Alertmanager `Watchdog` to an external service). Measure SLIs at the edge — a server-side SLI never sees requests that
failed to arrive. Low-traffic services need synthetic traffic or longer windows, or one error pages.

**Differential diagnosis.** One host or all (grey failure, noisy neighbour)? One zone? One endpoint?
One tenant? One client version? Did it start at a deploy, config change, traffic change or a time
boundary (midnight UTC, top of the hour, month end)? Does it correlate with GC, cron, backups,
compaction, certificate renewal? Is the error from our code or a proxy (check the `server` header
and response flags)?

**Commands.** `ss -tanp` (states, Send-Q/Recv-Q), `ss -s`, `ss -ltn` (on a listening socket Recv-Q
is the current accept backlog), `nstat -az | grep -i -E "retrans|overflow|drop"`
(`TcpExtListenOverflows` = accept-queue drops), `conntrack -S`, `tcpdump -i any host X and port Y`,
`dig +trace`, `curl -w` timing (`time_namelookup`, `time_connect`, `time_appconnect`,
`time_starttransfer`), `kubectl get endpointslices`, `kubectl describe pod` (OOMKilled, exit 137 =
SIGKILL, 143 = SIGTERM), `jstack`/async-profiler, `py-spy dump`, Go `pprof` goroutine dump
(thousands of goroutines blocked on the same line = found it).

**Latency in exact 1s/3s steps.** The listen backlog (`somaxconn`, 4096 since Linux 5.4, 128
before) overflows when the app accepts too slowly (blocked event loop, saturated acceptor); SYNs are
dropped and clients retransmit after 1s, then 3s total. Latency clusters at ~1s and ~3s are a strong
signal of SYN retransmission. Clusters at exactly 5s point to DNS retries (§3); at exactly 200ms
or 40ms, to Nagle's algorithm interacting with delayed ACK (set `TCP_NODELAY`).

### 13. Testing and verification

**Fault injection.** Mesh-level faults (Istio `fault.delay`, `fault.abort`) test timeouts and
breakers without code changes. Chaos experiments need a steady-state hypothesis, a bounded blast
radius and an abort switch. Game days rehearse humans too — the runbook, the access, the dashboard
nobody opened in a year.

**Jepsen-style testing.** Run concurrent operations under partitions and clock skew, record the
history, and check it against a consistency model (Knossos, Elle). Many databases have failed their
documented guarantees this way.

**Deterministic simulation** (FoundationDB, TigerBeetle, Antithesis): run the system in one process
with simulated network, disk and clock driven by a seed, so every failure replays exactly.

**Contract tests (`M24`, `M35`).** Consumer-driven contracts (Pact) catch "the provider removed a
field a consumer reads" in CI; schema registry checks do the same for events. They protect only
what consumers declare: publishing pacts must be part of consumer CI, and providers should verify
against the versions actually deployed (Pact Broker `can-i-deploy`) before releasing.

**Game days decay.** A passing zone-failure exercise is a snapshot: placement of partition leaders,
DNS and connection lifetimes, dashboards and runbooks drift afterwards. Run exercises regularly
with fresh scenarios, automate checks for the drifting properties (leader placement per zone,
max connection lifetimes, runbook links), and give runbooks owners like code.

**Load tests that mean something.** Production-shaped traffic (key skew, payload sizes, cache hit
rate), constant arrival rate, with a dependency and a zone failed, pushed until it breaks — *how*
it breaks (graceful shedding or collapse) matters more than the number.

---

## Questions

### Level 1 — Timeouts, retries and everyday failures

The questions asked in almost every senior loop: something is slow or failing, and the answer turns on a default nobody set.

1. "Our Go service calls a partner API. Last night the partner had an outage and our service stopped serving *all* traffic, even endpoints that never call the partner. Memory climbed until the pods were OOMKilled. What happened?"
   > **Direction:** A zero-value `http.Client` has no timeout, so goroutines waiting on the black-holed partner piled up without limit; set connect and total timeouts and a per-dependency concurrency bulkhead (§1, §5; `M09`, `M10`).

2. "We see 0.1% 502s from our AWS ALB in front of a Node.js API. They are worse at 3am than at peak, and no application log shows an error for those requests. What is going on?"
   > **Direction:** The keep-alive race: Node's 5s `keepAliveTimeout` is shorter than the ALB's 60s idle timeout, so the ALB reuses a connection Node is closing; set `keepAliveTimeout` above 60s (§1; `M08`).

3. "A checkout call to the payment provider timed out after 10 seconds and our code retried. Some customers were charged twice. The team proposes lowering the timeout to 3 seconds. What do you say?"
   > **Direction:** A timeout is an unknown outcome, not a failure; the fix is an idempotency key recorded before the call and passed to the provider, plus a status lookup and reconciliation — not a different timeout (§7, §8; `M08`, `M16`).

4. "A report endpoint that takes about 20 seconds worked fine until we moved the service into the Istio mesh. Now it fails every time at exactly 15 seconds with a 504. Nothing in our code changed."
   > **Direction:** Envoy's default route timeout is 15s (`504` with flag `UT`); set a route timeout for that path in the VirtualService, and question whether a 20s synchronous call should be async (§1, §6; `M26`).

5. "Our database had a 30-second failover. The database recovered but our API stayed down for 20 more minutes; restarting the API pods fixed it instantly. Why?"
   > **Direction:** Pooled connections to the old primary were half-open and hung until TCP's `tcp_retries2` (~15 min) gave up, and/or the client cached the old address; use `TCP_USER_TIMEOUT`, read timeouts, connection max lifetime and validation (§1, §3; `M33`).

6. "We have a chain: gateway → orders → inventory → Postgres. Each layer has 'retry 3 times on failure'. During a slow-database episode Postgres went from 60% to 100% CPU and stayed there. Explain the numbers."
   > **Direction:** Retries multiply per layer — 4 × 4 × 4 = up to 64 attempts per user request on Postgres; retry at one layer only, with a retry budget and jitter (§2; `M09`).

7. "A Python worker occasionally hangs for hours; the process is alive, CPU is zero, and the last log line is 'calling enrichment service'. How do you find and fix this class of bug across the codebase?"
   > **Direction:** `requests` has no default timeout; confirm with `py-spy dump` showing threads in socket reads, then enforce timeouts centrally (a shared session wrapper, a lint rule) rather than per call site (§1, §12; `M09`).

8. "Our mobile app times out after 10 seconds. The API gateway times out after 30, the backend after 60. During an incident, backend CPU stayed pinned long after users gave up. What is wrong with the design?"
   > **Direction:** Timeouts are inverted: inner layers need shorter budgets than outer ones, and work whose caller has gone should be cancelled via a propagated deadline (§1, §5; `M09`).

9. "We added exponential backoff to our retries, but every time a dependency blips we see a second, larger spike a few seconds later, then a third. Why didn't backoff help?"
   > **Direction:** Backoff without jitter keeps clients synchronised, so retries arrive in waves; use full jitter and cap total retries with a budget (§2; `M09`).

10. "One endpoint makes a single call to a downstream with p99 of 40ms. We set a 30-second timeout 'to be safe'. During a downstream incident our thread pool emptied in seconds. What should the timeout be and why?"
    > **Direction:** A timeout far above p99.9 is effectively none; size it from measured p99.9 plus headroom — by Little's law a 30s timeout holds threads 750× longer than normal (§1, §3; `M09`).

11. "Our retry policy is 'retry up to 3 times, total timeout 1 second'. Traces show almost no retries ever happen — the first attempt fails at 1 second and that is it."
    > **Direction:** Without a per-try timeout the first attempt consumes the whole budget; set a per-try timeout (e.g. 300ms) inside the overall deadline (§2; `M09`).

12. "A downstream team says our service hammers them when they return 429. We do retry on 429 with backoff. What are we probably getting wrong?"
    > **Direction:** Honour `Retry-After`, count 429 retries against a retry budget, and use client-side adaptive throttling so clients stop sending when most requests are rejected (§2; `M09`, `M30`).

13. "Some downloads from our file service hang for many minutes even though we configured a 10-second read timeout in OkHttp. The server is known to be flaky. How is that possible?"
    > **Direction:** Read timeouts are inactivity timeouts that reset on every byte, so a slow-drip response never trips them; set a total call timeout (`callTimeout`) or a deadline (§1; `M09`).

14. "After every deploy, 'connection refused' errors spike for about 10 seconds on each pod rollout, then disappear. Readiness probes look fine. What is the mechanism and the fix?"
    > **Direction:** The termination race — SIGTERM and endpoint removal happen in parallel, so traffic still arrives after the app exits; add a `preStop` sleep and a graceful drain (§4; `M27`).

15. "Our Postgres-backed API returns 500s whenever a heavy analytics query runs. All 20 pool connections are in use. Requests don't fail fast — they take 30 seconds, then fail. Why 30 seconds, and what would you change?"
    > **Direction:** HikariCP's 30s `connectionTimeout` disguises pool starvation as latency; add `statement_timeout`, a separate pool (bulkhead) or replica for analytics, and a short acquisition timeout (§1, §3, §5; `M10`).

16. "A gRPC client calls a server with no deadline set. The user closes the app, yet server metrics show the expensive query still running minutes later. How do you fix this properly?"
    > **Direction:** Set a deadline on every call so gRPC propagates `grpc-timeout` and cancels downstream, and have the server check context cancellation before and during expensive work (§1; `M04`, `M09`).

17. "Health checks pass on every pod, but 5% of user requests fail with 'disk full'. The dashboard is green. Who is lying, and how do you make the health signal honest without making it dangerous?"
    > **Direction:** A constant `/health` misses grey failure; health should reflect local ability to serve (disk writable, pool not saturated) but must not check shared dependencies, or the whole fleet fails together (§4; `M08`).

18. "We call a slow partner API from the request path. Its p99 is 4 seconds and product insists the feature stays. How do you stop that partner dominating our tail latency and threads?"
    > **Direction:** Bulkhead it with a short timeout and a degraded or cached fallback, or move the call off the request path (async enrichment); treat 'too slow' as failed (§1, §5; `M03`, `M10`).

19. "Our service publishes to Kafka on the request path. During a broker rolling restart, `send()` blocked request threads for up to two minutes. Why two minutes, and what would you change?"
    > **Direction:** The producer retries internally until `delivery.timeout.ms` (120s) and `send()` blocks up to `max.block.ms` when its buffer is full; bound both and do not wait synchronously — or write to an outbox instead (§2, §7, §8; `M15`).

20. "Calls to one downstream take either about 1 second or about 3 seconds when slow, never in between, and the downstream's own latency metrics look normal. What does that pattern tell you?"
    > **Direction:** Latency in 1s/3s steps is SYN retransmission — the accept backlog is overflowing or packets are dropped before the app sees them; check `TcpExtListenOverflows` and `somaxconn` (§12; `M08`).

### Level 2 — Load balancing, connections and deployments

Common but less textbook: traffic goes to the wrong place, connections outlive their welcome, and every deploy leaves a mark.

1. "We autoscaled a gRPC service from 4 to 12 pods during a spike. CPU on the original 4 stayed at 95%, the 8 new pods sat at 5%, and latency did not improve. Kubernetes Service, no mesh. Why?"
   > **Direction:** gRPC uses long-lived HTTP/2 connections and a ClusterIP Service balances connections, not requests; use client-side round-robin over a headless Service, an L7 proxy, or `MAX_CONNECTION_AGE` (§3; `M07`).

2. "Every deploy of our JVM service causes a p99 latency spike from 80ms to 2 seconds for about three minutes, even though the rollout is gradual and readiness passes. What is happening and how do you remove it?"
   > **Direction:** New pods are cold (JIT, caches, pools) but get a full traffic share on readiness; warm up before readiness passes and use slow-start weighting (§4; `M07`, `M27`).

3. "Our load balancer uses least-connections. One pod started returning 500 in 2ms after a bad config load, and within a minute it was receiving 60% of all traffic. Explain and fix."
   > **Direction:** A fast-failing host looks like the least loaded, so least-connections feeds it; add outlier detection and weight by success, not only latency (§4; `M07`, `M10`).

4. "One pod out of 30 has a noisy neighbour on its node and is three times slower. Round-robin still sends it only a thirtieth of traffic, yet overall p99 is terrible. Why does one slow pod hurt p99 so much, and what balancing algorithm helps?"
   > **Direction:** Round-robin sends equal work to unequal servers, so the slow pod queues and dominates the tail; power-of-two-choices on in-flight requests routes around it automatically (§4; `M07`).

5. "We moved from an ALB to an NLB for a TCP service. We have 2 targets in eu-west-1a and 8 in eu-west-1b. The two targets in 1a are overloaded; the others idle. Nothing about the targets differs."
   > **Direction:** NLB cross-zone balancing is off by default, so each zone gets 50% of traffic regardless of target count; enable cross-zone or balance targets per zone (§4; `M07`).

6. "Our cloud bill shows cross-AZ data transfer as the third-largest line item, larger than the database. We have about 40 chatty microservices across three zones. What is driving it and what would you do?"
   > **Direction:** Random balancing sends ~2/3 of calls cross-zone at $0.01/GB each way; use topology-aware or zone-aware routing, while watching for overload when zone capacity is uneven (§4; `M07`, `M31`).

7. "After a rolling restart of our API, connection counts on the database are wildly uneven across pods: the first pod restarted holds 3× the connections of the last. Why, and does it matter?"
   > **Direction:** Long-lived connections stay where they were opened during the rollout; it matters for load skew and failover — recycle connections with a max lifetime (§3, §4; `M07`).

8. "A service calls `api.partner.com` about 5,000 times a second from Kubernetes. CoreDNS CPU is pegged and lookups are slow, though the partner's name barely changes. What is multiplying DNS load?"
   > **Direction:** `ndots:5` makes every lookup try all search domains for A and AAAA first — up to 10 queries per call; use an FQDN with a trailing dot, lower `ndots`, and NodeLocal DNSCache (§3; `M05`).

9. "Random requests from one Kubernetes node take exactly 5 seconds longer than normal. Other nodes are fine. The extra time appears before the TCP connect. What is your first suspect?"
   > **Direction:** The conntrack race on parallel A/AAAA UDP queries drops a DNS packet and the resolver retries after 5s; fix with `single-request-reopen` or NodeLocal DNSCache, and check conntrack stats (§3; `M05`).

10. "A batch job making HTTP calls to one internal service starts failing after a few minutes with 'cannot assign requested address'. It makes about 800 requests per second. What is exhausted?"
    > **Direction:** Ephemeral ports: a new connection per request leaves each socket in TIME_WAIT for 60s and ~28k ports run out at ~470/s per destination; reuse connections with a keep-alive pool (§3; `M08`).

11. "Our services in private subnets call a SaaS API through a NAT gateway. At peak, we see connection errors to that API only, and CloudWatch shows `ErrorPortAllocation` rising. Adding pods made it worse."
    > **Direction:** NAT gateways allow ~55,000 simultaneous connections per destination; pooling and keep-alive, more NAT gateways/IPs, or a private endpoint remove the SNAT limit (§3; `M08`).

12. "We set a liveness probe that checks the database. The database had a 20-second hiccup, and within a minute all 60 API pods restarted; the outage lasted 15 minutes. Explain the amplification."
    > **Direction:** Liveness checking a dependency turns its blip into a fleet restart; restarted pods warm up and reconnect all at once, prolonging the outage — liveness should only detect a wedged process (§4; `M08`).

13. "During a partial database outage all our pods failed readiness together, the Service had zero endpoints, and the gateway returned 503 for every endpoint — including ones that never touch the database."
    > **Direction:** Readiness on a shared dependency makes the whole fleet unready at once; keep readiness local and handle dependency failure with breakers and degraded responses per endpoint (§4; `M10`).

14. "In our mesh, 3 of 5 pods of a service are failing. We expected Envoy to send everything to the healthy 2, but traffic is still going to the failing ones. Is Envoy broken?"
    > **Direction:** With fewer than 50% hosts healthy Envoy enters panic mode and balances across all hosts; and outlier detection's 10% `max_ejection_percent` limits ejections — both are deliberate (§4; `M10`, `M26`).

15. "Our WebSocket service is scaled by HPA on CPU. After a scale-out, new pods stay idle while old ones hit their connection limit. After scale-in, thousands of clients reconnect at the same second and crash the service."
    > **Direction:** Long-lived connections do not rebalance on scale-out; on scale-in they reconnect as a herd — add reconnect jitter, connection-count-based balancing, and gradual draining (§2, §4; `M07`, `M30`).

16. "A Kubernetes Job with an Istio sidecar has been 'Running' for three days although the main container finished in 10 minutes. Separately, its first outbound call sometimes fails at startup."
    > **Direction:** The classic sidecar never exits and may start after the app; use native sidecars (init containers with `restartPolicy: Always`) or `holdApplicationUntilProxyStarts` (§6; `M26`).

17. "After a DNS-based failover of our RDS database, half our pods reconnected to the new primary and half kept failing with 'read-only transaction' errors for 30 minutes."
    > **Direction:** Existing pooled connections still point at the old primary (now a replica) and some JVMs cache DNS; bound connection lifetime, handle read-only errors by evicting connections, and check JVM DNS TTL (§3; `M33`).

18. "We added a pod to a 4-node memcached cluster using client-side `hash(key) % N`. The database immediately got 5× its normal read load and some pages timed out. What did adding capacity do?"
    > **Direction:** Modulo hashing remaps almost every key when N changes — effectively a full cache flush; use consistent hashing with virtual nodes and warm or ramp new nodes (§5, §11; `M32`).

19. "Large API responses from one service time out, but only for clients connecting over the new site-to-site VPN. Small requests and health checks work perfectly. What do you check?"
    > **Direction:** An MTU black hole: the tunnel lowers MTU and ICMP 'fragmentation needed' is filtered, so large packets vanish; fix with MSS clamping or allowing ICMP type 3 code 4 (§3; `M08`).

20. "We are behind an NLB and use connection pooling to a backend. Every morning the first few requests after the quiet overnight period hang for 15 minutes, then everything is fine."
    > **Direction:** Idle pooled connections exceeded the NLB/NAT 350s idle timeout and were silently dropped without RST, so the first write hangs on TCP retransmission; set pool idle eviction below 350s, TCP keepalive, and `TCP_USER_TIMEOUT` (§1, §3; `M08`).

### Level 3 — Overload, backpressure and cascading failure

Frequently asked at senior level: the system is up but drowning, and the obvious fixes make it worse.

1. "Our API handled 5,000 rps fine. At 6,000 rps, latency did not rise gradually — it went from 50ms to 20 seconds in under a minute, and throughput actually dropped. Explain the shape."
   > **Direction:** Queueing latency is 1/(1−ρ), flat until the knee then explosive; beyond saturation work completes after clients time out, so goodput collapses — bound queues and shed (§5; `M30`).

2. "During a traffic spike our service's request queue grew to 50,000 entries. We scaled up, traffic dropped, but for 10 minutes the service processed requests nobody was waiting for. How should the queue have behaved?"
   > **Direction:** Unbounded FIFO queues serve dead requests; bound the queue, drop requests past their deadline, and under congestion switch to adaptive LIFO with CoDel-style short queue timeouts (§5; `M30`).

3. "We have a rate limit of 1,000 rps per instance. Yesterday a downstream slowed from 20ms to 400ms and our instances fell over while still under 1,000 rps. Why didn't the rate limit protect us?"
   > **Direction:** A rate limit ignores latency; in-flight work = rate × latency grew 20×, so limit concurrency instead, ideally adaptively (§5; `M10`, `M30`).

4. "When we are overloaded we return 503 — but the 503 path still authenticates the user via the identity service and writes an audit log. At 5× load the service dies anyway. What principle is being violated?"
   > **Direction:** Rejection must be much cheaper than service; shed at the earliest point (edge, first middleware) before remote auth, body parsing and heavy logging (§5; `M10`).

5. "A 30-second network blip between our services and Redis ended an hour ago. The network is fine now, but the database is still at 100% CPU and the API is still timing out. Rolling back last week's deploy changed nothing. What kind of failure is this?"
   > **Direction:** A metastable failure: the cold cache and retries sustain the overload after the trigger is gone; remove the sustaining effect — shed most traffic at the edge, disable retries, warm the cache, ramp back up (§5; `M08`, `M30`).

6. "We have a single thread pool of 200 threads serving all endpoints. The recommendations service slowed to 10 seconds and within 30 seconds checkout, login and search all failed too. What is the fix and how do you size it?"
   > **Direction:** No bulkheads: one slow dependency consumed every thread; give each dependency its own bounded pool or semaphore sized by Little's law, with fast rejection when full (§3, §5; `M10`).

7. "Our HPA scales on CPU. During last Black Friday, latency went through the roof but pods stayed at 30% CPU and HPA never scaled. What happened, and would scaling have helped?"
   > **Direction:** A service blocked on a slow dependency has low CPU and high latency; scaling on CPU misses it, and adding callers would only load the slow dependency more — scale on concurrency or queue depth and pre-scale for known events (§5; `M30`).

8. "A latency-sensitive Java service has average CPU at 35% of its limit, yet p99 shows regular 90ms steps. Removing the CPU limit made the spikes disappear. Explain."
   > **Direction:** CFS throttling: many threads burn the 100ms-per-period quota early and the container is frozen for the rest of the period; check `cfs_throttled_periods`, drop limits or size threads to them (§5; `M30`).

9. "A product page's cache key expires every 60 seconds. Every minute, database load spikes to 40× for a second. The page gets 20,000 rps. Fix it without adding more database capacity."
   > **Direction:** A cache stampede; use request coalescing (singleflight), stale-while-revalidate or probabilistic early refresh, and jittered TTLs (§5; `M30`).

10. "Our Resilience4j circuit breaker on a low-traffic admin endpoint never opens, even though the dependency is 100% failing. On the high-traffic path it works fine. Why?"
    > **Direction:** The breaker needs `minimumNumberOfCalls` (default 100) in its window before evaluating; tune thresholds per traffic level or use time-based windows (§5; `M10`).

11. "We put a circuit breaker per downstream service. One of the 20 instances of that service is broken and returns errors; the breaker opened and blocked calls to all 20. How would you design it differently?"
    > **Direction:** A per-service breaker punishes all hosts for one; use per-host outlier ejection at the load-balancer layer and keep the service-level breaker for whole-service failure (§4, §5; `M10`).

12. "After a downstream recovered, our breakers across 300 client pods all went half-open within the same second and immediately knocked it over again. This repeated four times."
    > **Direction:** Synchronised half-open probes stampede the recovering service; jitter the open duration, limit half-open permits, and ramp traffic back gradually (§2, §5; `M10`).

13. "Our Envoy sidecars return 503 with flag `UO` to a healthy upstream during peak. The upstream's own metrics show it is at 40% capacity and never saw those requests."
    > **Direction:** Envoy cluster circuit breakers are concurrency limits (`max_requests`/`max_pending_requests` default 1024); raise them for this cluster or reduce in-flight work — the upstream is not failing (§5, §6; `M26`).

14. "Our batch reprocessing job and the user-facing API share the same order service. Every night the batch saturates it and customers see errors. We cannot move the batch. What do you do?"
    > **Direction:** Priority shedding: tag requests by criticality and shed sheddable batch traffic first under load, propagating criticality end to end; or a separate pool per class (§5; `M10`, `M31`).

15. "A consumer service reads from a queue and writes to a database. When the database slows down, the queue grows by millions and memory on the consumer explodes. How should backpressure flow here?"
    > **Direction:** Bound in-memory buffers (prefetch, batch size) so a slow sink slows consumption rather than accumulating; the durable queue is the buffer, and alert on lag in time (§5, §7; `M30`).

16. "A service logs every failed request with a full stack trace. During a downstream outage at 8,000 rps, the service itself became unresponsive, even for requests that don't touch that downstream."
    > **Direction:** Synchronous logging became the bottleneck (CPU, disk, blocked appender); use async bounded appenders that drop under pressure and rate-limit repeated errors (§12; `M08`).

17. "We rely on autoscaling for zone failure: if a zone goes down, the other two scale up. During a real zone outage, scaling took 20 minutes and the remaining zones collapsed in the meantime."
    > **Direction:** Static stability: pre-provision enough capacity per zone to absorb a zone loss, because the control plane you need to scale may itself be impaired (§5; `M31`).

18. "We saw a 5% error rate spike, turned on retries at the edge 'to smooth it out', and the error rate went to 60% within two minutes. Why did retries convert a small problem into a big one?"
    > **Direction:** Retries add load to an already saturated service, pushing it past the knee — a retry storm; use a retry budget (≤10–20% extra) so retries cannot multiply load in an outage (§2, §5; `M09`).

19. "We added hedged requests to our search backend to cut p99. It worked for a week, then during a traffic spike p99 got much worse than before hedging. Why?"
    > **Direction:** Hedging fixes per-replica hiccups but doubles load when the tail comes from overload; cap hedges with a budget and disable them when the backend is saturated (§2; `M10`).

20. "Our API gateway has a global timeout of 30s. A downstream with 5s p99 has a queue in front of it. Under load, the queue fills with requests that will obviously miss the gateway deadline. How do you make the downstream stop doing useless work?"
    > **Direction:** Propagate the deadline and have the downstream drop queued requests whose remaining budget is below its minimum service time — dead-on-arrival shedding (§1, §5; `M09`, `M30`).

### Level 4 — Messaging, duplicates and ordering

Asynchronous systems in production: messages arrive twice, out of order, or not at all, and the consumer group stops making progress.

1. "Our Kafka consumer group keeps rebalancing every few minutes and lag grows without bound. Each consumer calls a downstream API per record; that API recently slowed from 50ms to 800ms. No consumer is crashing."
   > **Direction:** 500 records per poll × 800ms exceeds `max.poll.interval.ms` (5 min), so consumers are expelled mid-batch and the reassigned consumer hits the same wall — a rebalance loop; cut `max.poll.records` or process asynchronously with pause/resume (§7; `M30`).

2. "Customers receive the same 'order shipped' email two or three times, always around our deploys. The consumer commits offsets after sending the email. Where do the duplicates come from, and how do you stop them?"
   > **Direction:** Deploys trigger rebalances and records processed but not yet committed are redelivered; make the consumer idempotent with a dedup record stored atomically with the side-effect state, since at-least-once is the real guarantee (§7; `M16`).

3. "We use `enable.auto.commit=true` and process records on a separate thread pool after `poll()`. After a pod crash, a few hundred orders were never processed. Kafka shows no gaps. What happened?"
   > **Direction:** Auto-commit commits offsets of polled records, not processed ones, so async processing plus a crash loses messages; commit manually after processing (§7; `M16`).

4. "A single malformed event has stopped one partition of our Kafka topic for 6 hours; the consumer retries it forever and lag on that partition is 2 million. How do you unblock it safely, and what should the design be?"
   > **Direction:** A poison message blocks a log partition; bounded retries then a dead-letter topic, but if per-key order matters park the key rather than skipping one event (§7; `M16`, `M17`).

5. "Our SQS workers sometimes process the same message concurrently on two machines; we see duplicate invoice rows created within seconds of each other. Processing takes about 45 seconds."
   > **Direction:** The 30s default visibility timeout expired mid-processing, so the message became visible again; extend visibility with heartbeats and enforce idempotency with a unique constraint (§7; `M16`).

6. "Events for the same account arrive at the consumer out of order — 'account closed' before 'deposit'. The producer uses the account id as the key. What could reorder them?"
   > **Direction:** Producer retries with multiple in-flight batches (without idempotence), a consumer processing a partition on a thread pool, or a recent partition-count change remapping keys; check each (§7; `M17`, `M21`).

7. "We increased a topic from 12 to 48 partitions to scale consumers. The next day, a downstream projection had account balances that were wrong for a small number of accounts."
   > **Direction:** Changing partition count remaps `hash(key) % partitions`, so a key's new events land on a different partition and can be processed before its old ones; drain or migrate per key, or create a new topic (§7; `M32`).

8. "A rolling deploy of our 40-instance consumer takes 20 minutes, and during that time the whole group barely makes progress. How do you make deploys invisible to throughput?"
   > **Direction:** Eager rebalances stop the whole group on each restart; use `CooperativeStickyAssignor` plus static membership (`group.instance.id`) so restarts within the session timeout do not rebalance (§7; `M27`).

9. "Our lag alert is 'more than 100,000 messages behind'. It fires constantly on one topic that is fine, and never fired on another topic that was 6 hours behind. What alert would you use instead?"
   > **Direction:** Offset lag means different things per topic; alert on lag in time — the age of the oldest unprocessed message — against the business freshness need (§7, §12; `M34`).

10. "A RabbitMQ consumer with prefetch unlimited crashed; afterwards the broker redelivered 180,000 messages and the other consumers fell hours behind. Also, while the queue was huge, publishers to *unrelated* queues on the same broker were blocked."
    > **Direction:** Unbounded prefetch hoards messages; large in-memory queues trigger memory alarms that block all publishers on the node — set a bounded prefetch and watch memory/disk alarms (§7; `M30`).

11. "Our FIFO SQS queue uses `MessageDeduplicationId`, so we assumed no duplicates. A producer retried a batch after an outage 10 minutes later and we got duplicates anyway."
    > **Direction:** FIFO deduplication only covers a 5-minute window; end-to-end idempotency still has to live in the consumer (§7; `M16`).

12. "We process payments from Kafka with transactions and `read_committed`. One morning every consumer of one partition stopped for exactly 15 minutes, then resumed. The producer app had been restarted."
    > **Direction:** A hung open transaction pins the last stable offset, stalling `read_committed` consumers until the transaction times out (up to `transaction.max.timeout.ms`, 15 min); lower `transaction.timeout.ms` and alert on LSO lag (§7; `M16`).

13. "An engineer reset a consumer group offset by three days to fix a projection bug. The projection was fixed, but 40,000 customers received old push notifications. How should replays have worked?"
    > **Direction:** Replays re-trigger every side effect; side-effecting consumers need the same idempotency layer with dedup retention longer than the replay window, or a replay mode that suppresses external actions (§7; `M16`, `M18`).

14. "Our dedup table stores processed message ids with a 24-hour TTL. A partner resent a week-old batch and we double-credited accounts. What design assumption broke?"
    > **Direction:** Dedup retention must exceed the maximum redelivery or replay window, and dedup by business key when the same fact can arrive with different ids (§7; `M16`).

15. "We parallelised a Kafka consumer by handing each record to a 32-thread pool. Throughput went up 10×, then we found balance updates applied out of order and offsets committed for records still in flight."
    > **Direction:** A thread pool breaks per-key order and offset semantics; hash keys to per-key queues inside the consumer and commit only the contiguous processed offset (§7; `M16`, `M30`).

16. "A tenant that produces 40% of our events is keyed by tenant id. Adding consumers doesn't help — one consumer is always at 100% and the rest idle. Keep per-tenant ordering where it matters."
    > **Direction:** A hot key maps to one partition and thus one consumer; key by a finer entity (e.g. order id) where only entity-level order matters, or split the hot tenant with sub-keys (§7, §11; `M32`).

17. "We use `acks=all` and thought we were durable. After a broker failure we lost a few seconds of acknowledged messages. The topic has replication factor 3."
    > **Direction:** With `min.insync.replicas=1`, `acks=all` means the leader alone; set `min.insync.replicas=2` and keep unclean leader election disabled (§9; `M33`).

18. "A consumer processes an event by calling an external API and then writing to our DB. It crashes between the two. On restart it calls the API again. The API is not idempotent. How do you make this safe?"
    > **Direction:** Record intent first (an outbox/state row with an idempotency key) and pass a key the external API can dedupe, or query its state before retrying; otherwise reconcile (§7, §8; `M15`, `M16`).

19. "Two instances of our outbox relay were accidentally deployed, and every event was published twice. Downstream counters doubled. Then someone fixed it by adding a Redis lock around the relay. What is still wrong?"
    > **Direction:** Consumers must be idempotent regardless; a Redis lease lock without fencing still allows two relays after a pause — prefer `FOR UPDATE SKIP LOCKED` row claiming in the database itself (§8, §9; `M15`, `M16`).

20. "Our event consumers use `new ObjectMapper()` to parse JSON. A producer added one optional field and every consumer in three teams started sending messages to the DLQ."
    > **Direction:** Jackson's `FAIL_ON_UNKNOWN_PROPERTIES` defaults to true outside Spring's configured mapper; consumers must be tolerant readers, and schema compatibility should be checked in CI (§10; `M24`).

### Level 5 — Consistency across services

Data that should agree across services does not. The textbook says "use a saga"; production asks what happens at each unknown step.

1. "Our order service saves an order and then publishes `OrderCreated` to Kafka. Roughly once a week, finance finds an order with no matching invoice, and occasionally an invoice for an order that doesn't exist. Explain both."
   > **Direction:** The dual write: commit-then-send loses events on send failure, and send-then-rollback creates ghost events; use a transactional outbox or CDC so state and event are atomic (§8; `M15`).

2. "Our outbox relay polls `SELECT * FROM outbox WHERE id > :last ORDER BY id LIMIT 500`. Once in a while an event is simply never published, and the row sits in the table. There is no error anywhere."
   > **Direction:** The sequence-gap bug: ids are assigned at insert but transactions commit out of order, so the high-water mark skips a late commit; use `SKIP LOCKED` claiming, a lagged window with dedupe, or read below the oldest in-flight transaction (§8; `M15`).

3. "Our Postgres primary's disk filled over a weekend and the database stopped accepting writes. The biggest thing on disk was WAL. Nobody had changed anything except deleting an old Debezium connector on Friday."
   > **Direction:** The connector's logical replication slot was left behind and retained all WAL; drop orphaned slots, set `max_slot_wal_keep_size`, and alert on slot lag (§8; `M15`).

4. "Debezium on a Postgres database where the captured tables change only a few times a day. WAL on the primary grows by 50GB a day even though Debezium is healthy and 'caught up'."
   > **Direction:** The slot's confirmed position only advances when captured changes flow; enable Debezium heartbeats with a heartbeat-table write so the slot advances on a busy but otherwise uncaptured database (§8; `M15`).

5. "The payment step of our checkout saga timed out. The saga compensated by releasing stock and cancelling the order. An hour later the payment provider's webhook said the payment succeeded. Now what, and how should the saga have been designed?"
   > **Direction:** A timeout is 'unknown', not 'failed'; the saga should wait and query the participant before compensating, put the irreversible step last, and rely on reconciliation for the remainder (§8; `M14`).

6. "Our saga orchestrator ran in memory inside the order service. During a deploy, 3,000 orders were left with stock reserved and payment taken but never shipped. How do you recover them, and what changes?"
   > **Direction:** Orchestration state must be durable (saga table, Temporal, Step Functions); recover by reconciling the participants' states for each in-flight order (§8; `M14`).

7. "Our refund compensation in a saga failed because the payment provider was down for two hours. The saga engine gave up after 5 retries and marked the saga 'failed'. What is wrong with that?"
   > **Direction:** Compensations must eventually succeed — retry with backoff for a long time, then route to a human queue; a failed compensation is an inconsistency, not an end state (§8; `M14`).

8. "Users update their profile and immediately see the old value on the next page, which reads from a CQRS read model fed by events. Product calls it a bug. How would you fix it without making everything synchronous?"
   > **Direction:** Read-your-writes via a version token: return the write's version, and have the read side wait briefly or fall back to the write model until the projection reaches it (§8; `M12`, `M19`).

9. "Our finance and order services disagree on about 0.02% of orders every month. Each team is sure its own real-time logic is correct. How do you run this problem?"
   > **Direction:** Build reconciliation: a periodic job comparing both sides (and the provider's settlement file) that categorises mismatches and routes them to repair or humans; the categories point at the bugs (§8; `M14`, `M22`).

10. "Two services both update a customer's loyalty points by consuming events and doing `points = points + n`. After an incident with redeliveries, some customers have more points than they earned."
    > **Direction:** Increments are not idempotent under at-least-once delivery; store processed ids atomically with the update, or apply absolute versioned state (§7; `M16`).

11. "Our inventory service uses last-write-wins with `updated_at` timestamps from each service's clock. Some stock adjustments disappear, always from one particular host."
    > **Direction:** That host's clock is behind, so its newer writes lose; order by logical versions (per-row version or sequence), not wall-clock time (§9; `M21`).

12. "We migrated from a monolith by having the new service write to its own database and the monolith's database in the same request. It has been 'mostly consistent' for months. What would you do?"
    > **Direction:** An application-level dual write can never be atomic; pick one source of truth, propagate via CDC or outbox, and run reconciliation during the migration (§8; `M15`, `M28`).

13. "We use CDC topics of the orders table as the integration contract for five other teams. A DBA renamed a column for clarity and three consumers broke in production."
    > **Direction:** CDC exposes the internal schema as a public contract; publish explicit events through an outbox (or a transformation layer) so internals can change (§8, §10; `M15`, `M24`).

14. "Our product search is fed by events from the catalogue service. After a bad deploy of the projector, the search index is corrupt for 10% of products. How do you repair it, and what did the design lack?"
    > **Direction:** Rebuild the projection by replaying from retained events or a snapshot plus CDC, into a new index, then swap; projections must be rebuildable, and replays must be safe (§7, §8; `M18`, `M19`).

15. "Our 'place order' saga reserves stock, charges the card, then books a courier. Courier booking fails 3% of the time, which triggers refunds and angry customers who see a charge then a refund."
    > **Direction:** Reorder around the pivot: validate and book (or tentatively reserve) the likely-to-fail steps before the irreversible charge, or use an authorisation that is captured only at the end (§8; `M14`).

16. "During a saga, a second request on the same order modified it between steps 2 and 3, and the compensation then restored the wrong state. How do you prevent concurrent sagas interfering?"
    > **Direction:** Semantic locks: mark the order `PENDING`/in-saga so other flows reject or queue, and use versioned compensations that check the state they expect (§8; `M14`).

17. "A long-running Temporal workflow (weeks for subscription renewals) started failing with non-determinism errors right after we deployed a small change to the workflow code."
    > **Direction:** In-flight workflows replay their history on the new code; changes must be versioned (patching / `GetVersion`) so old executions follow the old path (§8; `M14`, `M24`).

18. "The order service needs a customer's credit limit to accept an order. The customer service is often slow. The team wants to query it synchronously on every order. What alternatives do you offer, and what are the risks?"
    > **Direction:** Keep a local replica of the needed data via events (accepting staleness, with a bounded-staleness check) or reserve credit asynchronously; the synchronous query couples availability (§5, §8; `M22`, `M23`).

19. "We have an idempotency-key table in Redis for our payments API, with a 1-hour TTL. After a Redis failover, some payments were processed twice. The ledger is in Postgres."
    > **Direction:** Idempotency records must live with the state they protect — store the key in Postgres in the same transaction as the ledger entry; Redis loses keys on failover or eviction (§7; `M16`).

20. "Two concurrent requests with the same idempotency key arrive 5ms apart; both check the table, both find nothing, and both charge the card. How should the idempotency layer handle concurrency?"
    > **Direction:** Insert the key atomically in an 'in progress' state first (unique constraint), so the second request gets 409 or waits; only the winner calls the provider, passing the key downstream (§7; `M16`).

### Level 6 — Mesh, proxies and the platform layer

The infrastructure between services has its own failure modes, and the error message usually names the wrong component.

1. "In our Istio mesh we see a steady 0.05% of `503` responses with 'upstream connect error or disconnect/reset before headers. reset reason: connection termination'. Flag `UC`. The upstream app logs nothing."
   > **Direction:** The keep-alive race between Envoy's upstream pool and the app's shorter idle timeout; make the app's idle timeout longer than Envoy's (or set `idleTimeout` in the DestinationRule shorter) and allow retry on reset for idempotent calls (§1, §6; `M26`).

2. "After enabling the mesh, the application's retry logic and the mesh's retry logic combined gave a downstream 9× its normal load during an incident. Nobody configured mesh retries."
   > **Direction:** Istio retries twice by default on connect failures and 503; with app retries on top, attempts multiply — choose one owner for retries and set the other to zero (§2, §6; `M09`, `M26`).

3. "A gRPC service in the mesh suddenly lost per-request load balancing and retries after a Helm chart refactor. Traffic still flows, but one pod takes most of the load."
   > **Direction:** The port was renamed without the `grpc-`/`http2-` prefix or `appProtocol`, so Istio treats it as opaque TCP with connection-level balancing; fix the port name (§3, §6; `M26`).

4. "Our mesh has 4,000 services. Istiod CPU spikes on every deploy, sidecars use 400MB each, and some sidecars are OOMKilled. What would you change first?"
   > **Direction:** Every sidecar receives config for every service by default; use `Sidecar` resources to restrict egress hosts per namespace, cutting config size and push fan-out (§6; `M26`).

5. "At 02:00 on a Sunday, every service-to-service call in the mesh failed with TLS errors. Nothing was deployed. The mesh was installed exactly one year earlier with a plug-in intermediate CA."
   > **Direction:** The intermediate CA certificate expired; track CA expiries as first-class alerts and rehearse root/intermediate rotation — workload certs rotate automatically, CAs do not (§6; `M26`, `S15`).

6. "Istiod was down for 40 minutes during a control-plane upgrade. Existing traffic was fine, but every pod created during that window failed. Why the asymmetry, and is it a good design?"
   > **Direction:** The data plane keeps last-known config (static stability) but new pods need certificates and config from istiod; it is good design, but scale-out during a control-plane outage fails — keep istiod highly available and capacity pre-provisioned (§5, §6; `M26`, `M31`).

7. "Our app's timeout to a downstream is 2 seconds. The mesh has `perTryTimeout: 1s` and 2 retries. Traces show the downstream still receiving a third attempt after our app returned an error to the user."
   > **Direction:** Double policies: mesh retries continue after the app's deadline has passed; align budgets so the mesh's total is inside the app's timeout, or let one layer own retries (§2, §6; `M26`).

8. "Access logs show `503 UH` for a service during a deploy, although pods are Running and passing readiness. It lasts about 20 seconds per deploy."
   > **Direction:** `UH` means Envoy had no healthy endpoints — EDS propagation lag or all pods ejected by outlier detection; check `istioctl proxy-config endpoints` and deploy surge settings (§4, §6; `M26`, `M27`).

9. "Our pods in the mesh take 30 seconds to shut down on every deploy, and sometimes in-flight outbound calls from the app fail during shutdown with connection refused on localhost:15001."
   > **Direction:** The sidecar stopped before the app finished draining; delay Envoy exit (`EXIT_ON_ZERO_ACTIVE_CONNECTIONS`, drain duration) or use native sidecars for correct ordering (§4, §6; `M26`).

10. "Migrating to mTLS, we set the namespace to `STRICT`. A legacy service without a sidecar that calls into it broke immediately. Separately, a security review found that `PERMISSIVE` had been on for 18 months elsewhere."
    > **Direction:** STRICT rejects plaintext from non-mesh clients; migrate with PERMISSIVE plus telemetry showing plaintext callers, then flip — and treat leftover PERMISSIVE as a security finding (§6; `M26`, `S15`).

11. "An init container in our pod that runs database migrations fails with connection timeouts, but only in the mesh namespace. The same image runs fine elsewhere."
    > **Direction:** Init containers run before the sidecar starts but after iptables redirection is installed, so their traffic goes to a proxy that does not exist yet; exclude the port/range or use native sidecars (§6; `M26`).

12. "Adding the mesh increased our checkout p99 by 40ms. The checkout call chain has 7 services. Management asks whether the mesh is worth it. How do you answer with numbers?"
    > **Direction:** Each hop crosses two proxies (~1–3ms p50, more at p99), so 7 services is ~14 traversals; weigh that against retries, mTLS, telemetry you would otherwise build, and consider ambient mode or trimming hops (§6; `M26`).

13. "Our MySQL clients in the mesh hang on connect, while HTTP services work. The Service port for MySQL is named `db`."
    > **Direction:** MySQL is server-first; protocol sniffing waits for the client to speak first and the connection deadlocks — name the port `tcp-mysql` or `mysql` (or set `appProtocol`) so sniffing is skipped (§6; `M26`).

14. "Envoy outlier detection is enabled, but a pod that returned 100% errors for three minutes was never ejected. The service has 6 pods and each Envoy sends it about 1 rps."
    > **Direction:** Outlier detection is passive and per-Envoy: each proxy saw too few requests to hit `consecutive_5xx`, and `max_ejection_percent` caps ejections; add active health checks or fleet-level signals (§4, §6; `M10`).

15. "We turned on Istio request mirroring to test a rewritten pricing service. The next day, the analytics team reported doubled 'price quote viewed' events."
    > **Direction:** Shadow traffic must not have side effects; the mirror emitted real events — stub or suppress side effects in the shadow (it can detect the `-shadow` Host suffix) and mirror only reads (§10; `M27`, `M28`).

16. "Our API gateway (Envoy-based) returns `504` at exactly 60 seconds for a streaming download endpoint, but only when the client is slow. Fast clients succeed even for large files."
    > **Direction:** Some timeout on the path is a total, not an inactivity, timeout (route timeout or LB idle/response timeout); streaming routes need the route timeout disabled and idle timeouts tuned per hop (§1, §6; `M06`).

17. "We use an AWS ALB → Nginx ingress → Envoy sidecar → app. We get intermittent 502s. Each layer has its own idle timeout. How do you reason about which layer is at fault?"
    > **Direction:** Apply the keep-alive rule at every hop — each server's idle timeout must exceed its client's; read the error source from response flags and `server` headers to locate the hop (§1, §6, §12; `M06`).

18. "After a Kubernetes upgrade, cluster-wide leader election for our controllers kept flapping and the API server was intermittently slow. Our own services were fine but deploys failed."
    > **Direction:** etcd is sensitive to fsync latency; slow disks cause leader elections and API slowness, and client-go controllers lose leases together — check `etcd_disk_wal_fsync_duration_seconds` (§9; `M20`).

19. "Our pods get OOMKilled with exit code 137 even though the JVM heap is set to 1.5GB and the container limit is 2GB. GC logs show the heap is never full."
    > **Direction:** The container limit counts all process memory — metaspace, thread stacks, direct buffers, code cache — not just heap; size the heap with `MaxRAMPercentage` ~70% and measure native memory (§4; `M08`).

20. "Cluster upgrades have been stuck for a week because node drains never complete. The platform team blames a service team's PodDisruptionBudget."
    > **Direction:** A PDB with `minAvailable` equal to replicas (or `maxUnavailable: 0`) blocks every voluntary eviction; set it so at least one pod can be disrupted, and have enough replicas to honour it (§4; `M27`).

### Level 7 — Observability, releases and debugging in the dark

Less common and more senior: the dashboards disagree with users, the release looked safe, and the evidence is scattered across twenty services.

1. "Our p99 latency dashboard averages each pod's p99. It says 120ms. Users report multi-second page loads, and client-side telemetry shows p99 of 2.4 seconds for the same endpoint. Which is right?"
   > **Direction:** Averaging percentiles is meaningless, and server-side latency misses client pool wait, DNS, TLS and retries; merge histograms and measure at the edge/client (§12; `M34`).

2. "Our load test shows p99 of 80ms at 10,000 rps. In production at 7,000 rps we see p99 of 900ms. The load tester waits for each response before sending the next request."
   > **Direction:** Coordinated omission: a closed-loop tester stops sending during stalls and under-counts slow requests; test at a constant arrival rate with production-shaped data (§12, §13; `M35`).

3. "We sample 1% of traces. A payment bug affects 0.05% of requests and we cannot find a single trace of it. How do you change the tracing setup without 100× cost?"
   > **Direction:** Head sampling drops rare errors; use tail-based sampling that keeps all errored and slow traces, with spans routed to collectors by trace id (§12; `M25`).

4. "Traces for our order flow stop at the Kafka producer; the consumer side appears as unrelated root spans. Also some async handlers lose the trace id in logs. Why, and what is the fix?"
   > **Direction:** Context propagation breaks at thread hand-offs and message boundaries; inject trace context into message headers and propagate context through executors (§12; `M25`).

5. "A trace shows a child span starting 40ms before its parent. A junior engineer says our tracing library is broken. What is your explanation?"
   > **Direction:** Clock skew between hosts; span timestamps come from different clocks, so reason about relative durations within a host, not across (§9, §12; `M21`, `M25`).

6. "Prometheus ran out of memory during an incident and dashboards went blank exactly when we needed them. The week before, someone added a `path` label to the HTTP latency histogram."
   > **Direction:** Raw paths with ids explode series cardinality (× histogram buckets); label with route templates and bounded values (§12; `M25`, `M34`).

7. "Our availability SLO is 99.9%. The error-rate alert fires at more than 1% errors over 5 minutes; it pages constantly for blips and missed a slow 0.5% error leak that burned the whole month's budget in a week."
   > **Direction:** Use multi-window burn-rate alerts (14.4× over 1h and 5m, 6× over 6h and 30m) so fast burns page quickly and slow burns are caught too (§12; `M34`).

8. "Our server-side availability SLI says 99.99%, but the customer success team says clients experience outages every week. Both can be true. How?"
   > **Direction:** A server-side SLI never sees requests that failed before reaching the server (DNS, LB, TLS, mesh, client timeouts); measure at the edge and with synthetic or client telemetry (§12; `M34`).

9. "A canary of our new version showed no errors for 30 minutes and was promoted. Two hours later, at the nightly batch run, the new version failed. How should canary analysis have been set up?"
   > **Direction:** Bake time must cover the traffic patterns that matter (peak, batch), and compare canary against a concurrently running baseline, not yesterday (§10; `M27`).

10. "We added a new enum value `REFUNDED_PARTIAL` to an order status in our Protobuf contract. Nothing failed at deploy time, but a day later a consumer's reporting job silently counted those orders as 'unknown' and finance numbers were off."
    > **Direction:** Protobuf keeps unknown enum values as `UNRECOGNIZED`, which business logic mishandles; new enum values are a contract change needing consumer readiness first (§10; `M24`).

11. "A producer stopped sending the `discount` field in a Protobuf message because it was 'no longer used'. Downstream, invoices started showing a discount of zero instead of 'no discount data'."
    > **Direction:** A missing proto3 scalar reads as its default, indistinguishable from a real zero; use `optional`/wrappers for presence and never remove fields without expand–contract (§10; `M24`).

12. "We rolled back a release after an incident, and the rollback made things worse: the old version crashed on records the new version had written in the meantime."
    > **Direction:** Rollback requires N−1 compatibility; the new version wrote data the old cannot read — use expand–contract so every release is rollback-safe, or roll forward (§10; `M24`, `M27`).

13. "During a rolling deploy, users intermittently lost their saved preferences. Old and new versions share a Redis cache of the user profile, and the new version added a field."
    > **Direction:** Mixed versions share a cache format; old pods read and rewrite the object without the new field — version the cache key when the shape changes (§10; `M24`, `M27`).

14. "Our biggest outage last year was not a code deploy but a feature-flag change that enabled a new code path for 100% of traffic in every region at once. How would you prevent a repeat without banning flags?"
    > **Direction:** Treat config and flags as deploys: staged, percentage and cell-by-cell rollout with automated analysis and a kill switch independent of the failing system (§10; `M27`, `M31`).

15. "We are replacing a legacy pricing engine. The new one passes all tests, but we are nervous about switching. How do you gain confidence using production traffic without risk?"
    > **Direction:** Shadow or dual-run: send production reads to both, compare outputs with a diff dashboard (filtering nondeterministic noise, Diffy-style), cut over when mismatches are explained (§8, §10; `M28`).

16. "A service uses Confluent Schema Registry in `BACKWARD` mode. The team deployed the producer with a new schema first; the consumers, which had not been upgraded, failed to deserialise. The registry accepted the schema. Why?"
    > **Direction:** `BACKWARD` guarantees new readers can read old data — upgrade consumers first; producer-first rollouts need `FORWARD` or `FULL` (and transitive for replays) (§10; `M24`).

17. "Our consumer-driven contract tests with Pact were green, but a provider change still broke a consumer in production. The consumer had added a new field usage without updating its pact."
    > **Direction:** Contract tests only protect what the consumer declares; make pact publishing part of the consumer's CI and verify the provider against deployed versions (`can-i-deploy`) (§13; `M24`, `M35`).

18. "Latency on one endpoint shows two humps: many requests at 2ms and a group at exactly 40ms. There is no downstream call in that path. What would you check?"
    > **Direction:** A fixed 40ms (or 200ms) mode is Nagle's algorithm interacting with delayed ACK on small writes; set `TCP_NODELAY` (§12; `M08`).

19. "An intermittent bug corrupts one order in about a million. It only happens in production and we have never reproduced it. The system has 12 services and uses Kafka. How do you approach finding it?"
    > **Direction:** Instrument for correlation (trace/order id across logs and events), keep full traces for anomalies via tail sampling, reconcile to detect instances quickly, and consider deterministic or Jepsen-style testing for the concurrency hypothesis (§8, §12, §13; `M25`, `M35`).

20. "We want to run a chaos experiment in production: kill a zone's worth of pods for our checkout. Leadership is nervous. How do you design it so it is informative and safe?"
    > **Direction:** Steady-state hypothesis with explicit metrics, bounded blast radius (one cell or a percentage), automatic abort, a rehearsed rollback, and a game day that tests the humans and runbooks too (§13; `M35`).

### Level 8 — Scale: hot keys, tenants, fan-out and cells

Rarer questions, asked by teams operating at real scale: load is not uniform, one customer is not like the others, and averages hide everything.

1. "A celebrity with 80 million followers posted, and the Redis shard holding their profile hit 100% CPU while the other 63 shards were idle. The whole feed service degraded. How do you fix it immediately and permanently?"
   > **Direction:** A hot key; short-lived in-process L1 caching of hot items and replicating the key across shards (read a random copy) fix reads; detect hot keys continuously (§11; `M32`).

2. "Our biggest tenant runs a 2-million-row import every Monday. During it, every other tenant's jobs wait hours in the same queue. The queue is FIFO. What would you change?"
   > **Direction:** FIFO across jobs lets one tenant monopolise; use per-tenant queues with weighted fair scheduling plus per-tenant concurrency limits (§11; `M29`, `M30`).

3. "One tenant's malformed document crashes our parser workers. The retry logic spreads it to every worker in turn and the whole fleet is down for all tenants. How do you limit blast radius structurally?"
   > **Direction:** Shuffle sharding: assign each tenant a small random subset of workers so a poison tenant only takes down tenants sharing its whole shard; plus poison-message quarantine (§7, §11; `M31`).

4. "Our search request fans out to 100 shards and waits for all. Each shard's p99 is 50ms, but the overall p99 is 400ms. Explain the math and give three mitigations."
   > **Direction:** P(some shard exceeds its p99) = 1 − 0.99^100 ≈ 63%; hedge slow shard calls, return partial results at a deadline, and reduce leaf variance or shard count (§2, §11; `M32`).

5. "DynamoDB throttles writes on our `events` table even though consumed capacity is 20% of provisioned. The partition key is `date`."
   > **Direction:** All of a day's writes go to one partition, which is capped (~1,000 WCU); use a high-cardinality key or write-sharded suffixes merged on read (§11; `M32`).

6. "We want to move our largest customers into their own isolated cells. What breaks in the rest of the system when you do that, and how do you route?"
   > **Direction:** A thin, statically stable router maps customer → cell; the hard parts are cross-cell operations, migrating a customer's data between cells, and cell-by-cell deployment (§11; `M31`).

7. "We added 4 nodes to our 12-node consistent-hashing cluster. Hit rate dropped only by 25% as expected, but the new nodes were still overloaded for an hour and the old nodes had spare capacity. Why?"
   > **Direction:** Moving data competes with live traffic and new nodes start cold; ramp their weight, warm them, and use enough virtual nodes to spread the moved ranges (§11; `M32`).

8. "Our multi-tenant Postgres uses a schema per tenant. At 6,000 tenants, migrations take 9 hours, the catalogue is huge, and connection pooling is a nightmare. What are your options?"
   > **Direction:** Schema-per-tenant scales poorly in catalogue, migrations and pooling; move small tenants to a shared schema with a tenant id (and row-level security) and keep isolated databases only for the largest (§3, §11; `M29`).

9. "One tenant is using 70% of the capacity of a shared service but pays for 5%. Nobody noticed for months. How do you detect and control this class of problem?"
   > **Direction:** Per-tenant cost attribution and metrics (with bounded cardinality), plus per-tenant rate and concurrency quotas enforced at the edge (§11, §12; `M29`).

10. "Our consistent-hashing cache ring has 5 physical nodes and no virtual nodes. One node consistently gets 35% of the load. Adding a node made the imbalance worse."
    > **Direction:** Without virtual nodes, arc sizes vary widely; use 100–200 vnodes per node, or consistent hashing with bounded loads (§11; `M32`).

11. "A hot product sold out in a flash sale; 200,000 clients polled its stock endpoint every second, and the inventory database fell over. The stock value only changes a few times per second."
    > **Direction:** Coalesce and cache the hot read (singleflight, micro-TTL of ~1s at the edge or in-process), and push updates instead of polling; the database should see a handful of reads per second (§5, §11; `M30`).

12. "Our per-user rate limiter is a Redis counter keyed by user id. For a B2B customer with 20,000 employees behind one NAT IP and one API key, the limiter itself became the hot key."
    > **Direction:** A single counter is a hot key; use local token buckets with periodic sync, sharded counters, or hierarchical (tenant → user) limits evaluated locally (§11; `M29`, `M32`).

13. "We shard orders by customer id. A marketplace seller has 30% of all orders and queries 'all my orders' — that query hits every shard and is our slowest endpoint."
    > **Direction:** The access pattern doesn't match the shard key; maintain a secondary read model partitioned by seller (CQRS projection) rather than scatter-gather (§8, §11; `M19`, `M23`, `M32`).

14. "We rebalanced our Kafka partitions across brokers to fix hot brokers. During the reassignment, producer latency tripled and consumer lag grew across the cluster."
    > **Direction:** Reassignment replicates data and competes with live traffic; throttle replication (`leader.replication.throttled.rate`) and move a few partitions at a time (§11; `M32`, `M33`).

15. "Two large tenants share a cell. One's traffic spikes 10× during a marketing campaign and the other's latency triples. Both are paying enterprise customers. What short-term and long-term actions?"
    > **Direction:** Short term: per-tenant concurrency limits and priority shedding; long term: capacity headroom per cell and placement moving big tenants apart (or into dedicated cells) (§5, §11; `M29`, `M31`).

16. "Our API composition layer calls the user service once per item to render a list of 200 items. p50 is fine, but p99 is terrible and the user service is overloaded."
    > **Direction:** N+1 fan-out multiplies tail probability and load; batch the lookup, cache, or denormalise into a read model (§11; `M23`).

17. "Every hour at :00, our shared config service and the database get a 20× spike for 30 seconds. No single team's cron looks guilty."
    > **Direction:** Many independent jobs schedule on `0 * * * *` and synchronise; add jitter/splay to schedules and TTLs, and spread cache expiries (§5, §9; `M30`).

18. "We shuffle-shard 400 tenants across 16 workers, 2 workers each. A poison tenant still caused a noticeable outage for other tenants. The client does not retry to the other worker. Why did it hurt?"
    > **Direction:** Shuffle sharding only works if clients use their shard's other member; without retry across the pair every tenant sharing one worker is hurt — also check the combination count (C(16,2)=120) (§11; `M31`).

19. "A global leaderboard is a single Redis sorted set updated 50,000 times a second. We have one master and it is at 95% CPU."
    > **Direction:** Split into sharded sorted sets merged periodically (or per-region then global), accept slightly stale ranks, and read from replicas or a cached top-N (§11; `M32`).

20. "Our cell router maps customers to cells via a lookup in a central database. That database had a 10-minute outage and every cell went down, even though each cell was healthy."
    > **Direction:** The router must be statically stable: cache the full mapping locally (and serve last known good), so cells keep working when the mapping store is down (§5, §11; `M31`).

### Level 9 — Coordination, time, failover and split-brain

Rare and hard: correctness bugs that appear only when clocks, pauses and partitions line up. Few candidates have seen them; those who have answer very differently.

1. "We use a Redis lock with a 10-second TTL so only one worker processes each customer's billing run. Once a month a customer is billed twice. Workers never run longer than 3 seconds in our logs."
   > **Direction:** A GC or VM pause (or CFS throttling) outlasted the lease, so two holders overlapped; logs do not show the pause — use fencing tokens checked by the resource (e.g. a version column), or make billing idempotent per period (§9; `M20`, `M16`).

2. "An engineer proposes Redlock across 5 Redis nodes to make our lock 'safe'. The lock protects writes to a third-party API that knows nothing about us. What do you tell them?"
   > **Direction:** Redlock still has no fencing and assumes bounded pauses and clock drift; if the resource cannot check a token, the lock is only an efficiency hint — design the operation to be idempotent instead (§9; `M20`).

3. "Our Postgres HA pair uses a script: if the replica cannot reach the primary for 10 seconds, promote the replica. After a network blip we had two primaries accepting writes for 25 minutes."
   > **Direction:** Failover without quorum and without fencing causes split-brain; use a quorum-based manager (Patroni with etcd, a witness) and fence the old primary (STONITH, or self-demotion on losing the DCS lease) (§9; `M33`, `M20`).

4. "After recovering from that split-brain, both databases have accepted writes for the same order ids. How do you reconcile, and what do you tell the business?"
   > **Direction:** Diff the divergent histories by key and time, apply business rules (payments before anything else), escalate conflicts to humans, and communicate RPO honestly; prevention is fencing and quorum (§8, §9; `M33`).

5. "Our Kubernetes controller uses client-go leader election. During an API-server slowdown, two replicas of the controller both reconciled the same resources and created duplicate cloud load balancers."
   > **Direction:** The old leader continued its in-flight reconcile after its lease expired; check leadership before side effects, make reconciliation idempotent (tag cloud resources, look before create), and fence where possible (§9; `M20`).

6. "Our JWT validation started rejecting 2% of tokens as 'not yet valid' after we added a new pool of nodes. Tokens are issued by another service."
   > **Direction:** Clock skew between issuer and the new nodes (NTP misconfigured); allow leeway on `nbf`/`iat`, fix time sync, and alert on host clock offset (§9; `M21`).

7. "A distributed rate limiter uses wall-clock windows keyed by `floor(now / 60s)` on each API node. Customers occasionally get twice their quota in a minute or are throttled early."
   > **Direction:** Nodes disagree about window boundaries due to skew, and fixed windows allow bursts at the boundary; use a shared clock source (the Redis server's time) or sliding windows/token buckets (§9, §11; `M21`).

8. "At a leap second, one of our services consumed 100% CPU on every node and a scheduler fired a batch of timers early. Part of our fleet uses Google's time servers, the rest `pool.ntp.org`."
   > **Direction:** Leap second handling (step vs smear) and mixing smeared and unsmeared sources give up to 0.5s disagreement; standardise on one smeared source and use monotonic clocks for timers (§9; `M21`).

9. "A ZooKeeper-based leader for our scheduler keeps failing over every few hours, although the leader process never crashes. Failovers correlate with full GC pauses of 8–12 seconds."
   > **Direction:** The session timeout is shorter than GC pauses, so ephemeral nodes expire and a healthy leader is deposed; fix GC (heap sizing, pause-oriented collector) or raise the session timeout, and fence the old leader's actions (§9; `M20`).

10. "After a Postgres failover, our order service started throwing duplicate-key errors on new orders, and a few new orders reused ids that another service already had stored."
    > **Direction:** Async failover lost the latest sequence advances, so the new primary reissued ids; bump sequences after failover and prefer ids that do not depend on a single primary's sequence (UUIDv7/ULID or allocated ranges) (§9; `M33`).

11. "Our Kafka cluster lost an entire broker's disk. Some partitions had only that broker in the ISR. An engineer wants to enable `unclean.leader.election` to restore availability. What is the trade-off?"
    > **Direction:** Unclean election elects an out-of-sync replica and silently discards committed messages; decide per topic whether availability or data matters, and prevent it with RF=3 and `min.insync.replicas=2` (§9; `M33`).

12. "A daily settlement job runs at 02:30 Europe/London. One Sunday in March it did not run at all; one Sunday in October it paid merchants twice."
    > **Direction:** DST: 02:30 local does not exist on spring-forward and occurs twice on fall-back; schedule in UTC and make the job idempotent per business date (§9; `M21`, `M16`).

13. "Our 3-node etcd cluster runs on network-attached disks. During a storage slowdown, Kubernetes API calls started failing and every controller in the cluster lost leadership simultaneously."
    > **Direction:** etcd fsync latency drives leader elections; a slow disk makes etcd unavailable and every lease-based leader times out together — dedicated low-latency disks and monitoring fsync p99 (§9; `M20`).

14. "We run a scheduled job with a Kubernetes CronJob every 5 minutes. After a 12-hour control-plane incident, the CronJob stopped running entirely, and before that, two runs overlapped and processed the same batch."
    > **Direction:** More than 100 missed schedules without `startingDeadlineSeconds` stops the CronJob; `concurrencyPolicy: Allow` (default) overlaps runs — set `Forbid`, a deadline, and make the job idempotent (§9; `M16`).

15. "Two data centres run active-active with async replication and last-write-wins. After a 3-minute partition, a customer's address change was reverted and their order shipped to the old address."
    > **Direction:** LWW with wall clocks loses concurrent writes silently; use per-record home regions (single writer per key), version vectors with conflict handling, or CRDTs where semantics allow (§9; `M12`, `M21`, `M33`).

16. "A 2-node active/passive cluster with a heartbeat link lost the link. Both nodes became active. The vendor says the cluster is 'highly available'. What is missing?"
    > **Direction:** Two nodes cannot form a majority; add a third witness/quorum device and fencing so a node without quorum stops serving (§9; `M20`, `M33`).

17. "We use `SELECT ... FOR UPDATE` in Postgres as a distributed lock between services. A service pod was network-partitioned mid-transaction and the lock stayed held for 15 minutes, blocking all order processing."
    > **Direction:** The server does not notice a vanished client until TCP gives up; set `idle_in_transaction_session_timeout`, TCP keepalives or `tcp_user_timeout` on the server side, and keep lock-holding transactions short (§1, §9; `M20`).

18. "A cache is invalidated by an event after each database write. After a primary failover, users saw stale prices for hours although invalidation events were flowing."
    > **Direction:** The failover lost the last writes, but the cache kept the values from the old primary (or invalidations fired for writes that were lost); flush or version caches on failover and treat failover as a consistency event (§9; `M33`).

19. "Our job queue ensures single processing with a lease: a worker claims a job for 60 seconds and renews every 20. Under heavy load some jobs are executed twice, and the database shows two workers writing results."
    > **Direction:** Renewals were delayed (CPU throttling, overloaded event loop, slow DB) so leases lapsed; use a fencing token (claim version) in the result write so the stale worker's write is rejected (§5, §9; `M16`, `M20`).

20. "We store event ordering by `created_at` timestamp across services, and a downstream audit trail sometimes shows 'refund' before 'charge' for the same payment, 3–50ms apart."
    > **Direction:** Wall-clock timestamps from different hosts cannot order causally related events; carry causal order (a per-aggregate sequence, the parent event's version, or hybrid logical clocks) (§9; `M21`).

### Level 10 — Multi-system cascades and subtle failures at scale

The rarest questions: several mechanisms interacting across systems, where each team's component "worked as designed". Only people who have run large fleets through bad nights will recognise them.

1. "A 45-second Redis failover at peak caused a 3-hour outage. Timeline: cache misses, database CPU to 100%, API timeouts, clients retrying, HPA scaling the API from 80 to 300 pods, database connections exhausted. Redis was healthy after one minute. Walk me through why it didn't recover and how you would have ended it in 10 minutes."
   > **Direction:** A metastable failure sustained by retries, a cold cache and autoscaling multiplying DB connections; end it by shedding most traffic at the edge, disabling retries, capping pods and pool sizes, warming the cache, then ramping back (§3, §5; `M30`, `M10`).

2. "After a zone outage, our remaining two zones got its traffic. They coped. When the zone came back, the returning pods received traffic immediately, were slow, failed readiness, were removed, and the other zones re-overloaded. It oscillated for an hour."
   > **Direction:** Cold returning capacity plus zone-aware routing snapping traffic back causes oscillation; ramp traffic to the recovering zone with slow start and warm-up, and damp routing changes (§4, §5; `M31`, `M07`).

3. "A config service pushed a malformed routing file to every Envoy in the fleet within 30 seconds. Every service lost its routes. Rollback of the config needed the deploy system, which runs as a service behind the same mesh."
   > **Direction:** Global config pushes need staged, validated, cell-by-cell rollout with last-known-good fallback; recovery tooling must not depend on the system it recovers (§5, §6, §10; `M31`, `M26`).

4. "Our SLO dashboards, alerting and the incident chat bot run in the same Kubernetes cluster as production. During a cluster-wide DNS failure, we had no alerts and no dashboards for 40 minutes; customers told us."
   > **Direction:** Monitoring shares fate with what it monitors; run out-of-band monitoring and synthetic checks from outside the failure domain, including an external dead-man's switch (§3, §12; `M31`, `M34`).

5. "A small latency increase in the auth service (from 5ms to 40ms) caused a full outage of checkout 20 minutes later. Auth never errored. Checkout calls auth on each of its 12 internal hops."
   > **Direction:** Per-hop synchronous auth multiplies latency and connection holding across the chain until pools and threads exhaust; validate tokens locally (signed JWTs, cached keys) and cut repeated per-hop calls (§3, §5, §6; `M03`, `M10`).

6. "During a database brownout, our outbox relay fell behind by 40 minutes. When it caught up, it published 4 million events in 3 minutes, which overloaded three consumer services and one external webhook partner, who blocked our IP."
   > **Direction:** Catch-up after lag is a burst; rate-limit the relay and consumers, give external webhooks their own queues and per-partner limits, and make consumers apply backpressure rather than accept everything (§5, §7, §8; `M15`, `M30`).

7. "A new version of a shared client library changed its default timeout from 1s to 30s. Within a week, three unrelated services had outages, each blamed on a different downstream. What happened and how do you prevent this organisationally?"
   > **Direction:** A library default change silently removed bulkheading everywhere; treat client-library defaults as a platform contract with explicit per-call timeouts and fleet-wide config visibility (§1, §5; `M09`, `M10`).

8. "Our Kafka consumers for 20 services autoscale on lag. A producer bug emitted 50 million duplicate events. Consumers scaled to their max, the downstream database was overloaded by all of them, and three other teams' services that share that database went down."
   > **Direction:** Scaling on lag moves the bottleneck to shared downstreams; cap consumer concurrency to the downstream's capacity, bulkhead the shared database per team, and dedupe at the producer or consumer (§5, §7, §11; `M30`, `M31`).

9. "We moved all pods to a service mesh with automatic retries and outlier detection. In the next incident, one bad host was ejected, which pushed its load onto the remaining hosts, which slowed, which got two more ejected, and so on until the panic threshold kicked in."
   > **Direction:** Ejection shifts load onto survivors and can cascade when the fleet is near capacity; keep `max_ejection_percent` low, keep capacity headroom, and understand panic mode as the circuit breaker on ejection itself (§4, §5, §6; `M10`, `M26`).

10. "During a slow-database episode, customers created duplicate orders — not from double clicks. The order service has no retries. Logs show the same request id arriving at two different order pods 30 seconds apart; one pod had timed out towards Nginx but still committed."
    > **Direction:** A hidden retry: an old Nginx config with `proxy_next_upstream ... non_idempotent` (or an HTTP client silently retrying a stale connection) replayed the POST to the next upstream; enforce idempotency keys at the service and audit every layer's retry behaviour (§2, §7; `M09`, `M16`).

11. "A certificate rotation job renewed the TLS certificate for our internal API gateway. Half of our services continued to work, half failed with TLS errors. The failing half were all written in Java and had been running for more than a few months."
    > **Direction:** The new chain used a different intermediate or root not in the JVM truststore baked into old images (or pinned certs); certificate changes are deploys — test against every client runtime, stage the rollout and serve both chains during transition (§6, §10; `M26`, `S15`).

12. "Our checkout saga and our fraud service are both correct in isolation. Under load, fraud checks took 8 seconds, the saga timed out and compensated, but the fraud service then approved the order and triggered shipment. We shipped goods for cancelled orders."
    > **Direction:** A timeout is an unknown outcome; late replies must be matched against saga state and ignored or compensated, participants must check the saga's current state before acting, and reconciliation must catch the rest (§8; `M14`, `M16`).

13. "We ran a large data backfill through the same Kafka topics as live traffic. Live events were delayed by 6 hours behind the backfill, and several downstream SLOs were blown. The backfill was 'low priority' in the ticket."
    > **Direction:** Priority must exist in the system, not the ticket: separate topics/queues for bulk and live, per-class consumer capacity, and shedding or pausing bulk work when live lag grows (§5, §7; `M30`).

14. "After migrating a service to gRPC, one client pod out of 200 received a GOAWAY storm during a server rollout; its reconnection attempts had no jitter and synchronised with 199 others, flattening the new server pods on startup."
    > **Direction:** `MAX_CONNECTION_AGE` and rollouts force mass reconnects; add jitter to connection age (gRPC's age grace/jitter), reconnect backoff with jitter, and slow start on new pods (§2, §3, §4; `M07`).

15. "The payment provider had an outage and our retries were well-behaved: 1 retry, budgeted. Yet our outage lasted 30 minutes after theirs ended, because our order queue had 900,000 pending messages each with a 5-second payment call."
    > **Direction:** Backlog drain time = backlog / (concurrency ÷ latency); after recovery the queue must be drained with fresh requests prioritised (LIFO for user-facing, expire stale work) and extra consumer capacity bounded by the partner's limits (§5, §7; `M30`).

16. "A cross-region replication lag of 3 seconds, combined with our 'read from nearest region' policy and a cache populated on read, caused some customers to see their account balance jump back and forth for an hour after a large batch."
    > **Direction:** Reading stale replicas and caching the result pins stale data beyond the lag; populate caches only from the primary or with version checks, and use session consistency (version tokens) for read-your-writes (§8, §9; `M12`, `M33`).

17. "Our mesh's mTLS workload certificates are valid for 24 hours. Istiod was unhealthy for 26 hours over a long weekend due to a crash loop nobody noticed. On Monday morning, the whole mesh failed."
    > **Direction:** Static stability has a time limit: the data plane runs on cached certs until they expire; alert on control-plane health and on cert age, and know your time-to-failure for every cached credential (§5, §6; `M26`, `M31`).

18. "We cut over reads to a new service after 30 days of shadow comparison with a 0.01% mismatch rate. Two days later, a month-end job in the new service produced wrong invoices for 12% of customers."
    > **Direction:** The shadow period never covered month-end and batch paths; dual-run must include the full business cycle and write paths, and mismatches must be explained by category, not just rate-limited (§8, §10; `M28`).

19. "A single tenant's API integration sent a request that made our search service's query planner choose a pathological plan. It took 90 seconds, held a connection, and was retried by our gateway three times, by the mesh twice, and by the tenant's client five times, across a shared database. Everyone went down."
    > **Direction:** Retries across layers multiply a query of death (up to 3 × 2 × 5 in flight per original); bound query time with `statement_timeout`, isolate tenants, quarantine the request signature, and own retries at one layer with a budget (§1, §2, §11; `M09`, `M29`, `M31`).

20. "Our system passed a zone-failure game day in March. In a real zone failure in November, it failed: the database failed over, but three services had cached the old primary's IP, the queue broker's partition leaders were all in the lost zone, and our runbook referenced a dashboard deleted in June. What do you change beyond fixing those three things?"
    > **Direction:** Game days decay; run them continuously or regularly with fresh scenarios, automate checks for dependency placement and DNS/connection lifetimes, test humans and tooling, and treat runbooks as code with owners (§3, §9, §13; `M35`, `M33`).
