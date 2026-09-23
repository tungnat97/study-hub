[← back to the field index](README.md)

# API & Protocols · Part 4 — Real-life production problems

Nodes `A01`–`A22`, seen from the pager rather than the textbook.

Parts 1–3 teach how HTTP, TCP, TLS, DNS, gRPC, GraphQL and friends are supposed to work. This part
is about how they actually fail: the 502 that only happens once a minute, the TLS handshake that
hangs only over the VPN, the rate limiter that doubles its limit every time the service autoscales.
Interviewers use these scenarios because the good answer is rarely the first one in the book — it
comes from having been paged.

How to use it:

- Read the **Pre-knowledge** once, properly. Every question below is solvable from it.
- For each question, answer out loud first: what you would check, in what order, what you think it
  is, and what you would change. Only then read the **Direction** line.
- The direction is a pointer, not a full answer. It names the non-obvious insight and the
  pre-knowledge section (`§n`) and node IDs to revisit if you missed it.
- Levels run from the most commonly asked (Level 1) to the rarest and most tangled (Level 10). A
  senior candidate should be fluent through Level 6 and able to reason their way through 7–10.

---

## Pre-knowledge

### 1. Connection reuse and the idle-timeout race

`A01`, `A12`. The single most common source of "random" 502s in HTTP systems.

- **Keep-alive is two independent timers.** Each side of a persistent connection decides on its own
  when an idle connection is dead. There is no negotiation: the `Keep-Alive: timeout=5` header is
  advisory and most clients ignore it.
- **The race.** If the *server* closes an idle connection at the same moment the *client* reuses it,
  the client's request lands on a socket the server has already closed. The server's kernel answers
  with a RST; the client sees `ECONNRESET` / "socket hang up" / "connection reset by peer", or the
  load balancer reports a 502. The window is small, so the symptom is a low, steady error rate
  (0.01–0.5%) that correlates with traffic *lulls*, not peaks.
- **The rule.** The upstream (server) idle timeout must be **longer** than the downstream (client or
  load balancer) idle timeout, so the client always closes first. Behind an AWS ALB (default idle
  timeout 60 s) the application's keep-alive timeout must exceed 60 s — e.g. 65 s.
- **Real defaults worth knowing.**
  - AWS ALB idle timeout: 60 s (configurable to 4000 s). Client keep-alive duration: 3600 s.
  - Node.js `http.Server.keepAliveTimeout`: 5 s. `headersTimeout` must be greater than
    `keepAliveTimeout` or you get a second race. Node 18+ `requestTimeout` 300 s.
  - Gunicorn `--keep-alive`: 2 s. Uvicorn `--timeout-keep-alive`: 5 s.
  - nginx `keepalive_timeout` (client side): 75 s. Upstream `keepalive_timeout`: 60 s.
  - Tomcat `keepAliveTimeout`: defaults to `connectionTimeout`, 20 s in Spring Boot's embedded
    config.
  - Go `http.Server.IdleTimeout`: falls back to `ReadTimeout`, and if both are zero, no limit.
  - Azure Load Balancer idle timeout: 4 min. GCP HTTP(S) LB backend keep-alive: 600 s, so backends
    must exceed 620 s.
- **Client-side pools have the same race outbound.** Apache HttpClient, OkHttp, Node's `agent`,
  Python `requests`/`urllib3` all keep idle connections. If the remote closes after 5 s and your pool
  holds connections for 30 s, reuse after 5–30 s of idleness fails. Fixes: set the pool's idle
  eviction shorter than the server's timeout (`validateAfterInactivity`, `freeSocketTimeout`,
  `keepAliveMsecs`), and retry *idempotent* requests once on a reset of a *reused* connection (this
  is exactly what browsers and Go's transport do).
- **Why retries at the right layer matter.** A request that failed on a reused connection before any
  byte of response arrived is safe to retry if idempotent (`A02`). POSTs are not, which is why nginx
  and many clients refuse to retry them — and why you add an idempotency key (`A09`, §14) rather
  than retrying blind.

### 2. Proxies, load balancers and deploys

`A01`, `M06`, `M07`. Knowing *who* generated an error is half the diagnosis.

- **Read the status code together with its author.** A 502 from the ALB (HTML body, `Server: awselb/2.0`)
  means the target closed or reset the connection or sent something malformed. A 504 from the ALB
  means the target did not answer within the idle timeout. A 502/504 from nginx is logged in
  `error.log` with the reason (`upstream prematurely closed connection`, `upstream timed out
  (110)`, `no live upstreams`). A 503 from Envoy with `upstream_reset_before_response_started` or
  response flag `UO`/`UF`/`URX` tells you overflow, connect failure or retry exhausted.
- **ALB-specific facts.** ALB access logs carry `request_processing_time`, `target_processing_time`
  and `response_processing_time`; a `-1` target time means the target never responded. `elb_status_code`
  vs `target_status_code` shows who produced the code. `460` means the client closed before the ALB
  answered; `561` is an IdP error on authentication.
- **nginx defaults that bite.** `proxy_read_timeout 60s` → 504 for anything slower; `client_max_body_size 1m`
  → 413; `proxy_buffering on` buffers streaming responses (§12); `underscores_in_headers off`
  **silently drops** request headers containing underscores (e.g. `X_API_KEY`);
  `large_client_header_buffers 4 8k` → 400/494 for large cookies or JWTs; `proxy_next_upstream error timeout`
  retries on another upstream, but since 1.9.13 not for non-idempotent methods unless
  `non_idempotent` is set. nginx resolves upstream hostnames **once at start-up** unless you use a
  variable in `proxy_pass` with a `resolver` directive (§7).
- **Rolling deploys and 502s (Kubernetes).** On pod termination, the kubelet sends SIGTERM *at the
  same time* as the endpoint is removed from Services, kube-proxy/iptables and the ALB target group.
  Those removals propagate over seconds; the ALB deregistration takes its `deregistration_delay`
  (default 300 s). If the app exits on SIGTERM immediately, in-flight and newly-routed requests hit a
  dead pod → 502. Fix: a `preStop` hook that sleeps 5–15 s, then graceful shutdown (stop accepting,
  finish in-flight, send `Connection: close` on keep-alive responses so clients move away), and a
  `terminationGracePeriodSeconds` longer than both. ALB target groups also need
  `deregistration_delay` tuned down (e.g. 30 s) or deploys crawl.
- **Health checks lie in two directions.** A `/health` that touches the database takes the whole
  fleet out when the DB blips (all targets unhealthy → ALB "fails open" and routes to all anyway).
  A `/health` that touches nothing stays green while the app is wedged. Separate liveness (process is
  alive) from readiness (can serve), and keep dependency checks out of liveness. Recovery is slow too: a target needs *healthy threshold ×
  interval* consecutive passes before it receives traffic again.
- **Headers proxies rewrite.** TLS terminates at the LB, so the app sees `http`; frameworks build
  redirect URLs with `http://` unless they trust `X-Forwarded-Proto`. `X-Forwarded-For` is a list:
  the leftmost entry is client-supplied and spoofable; the trustworthy client IP is the entry added
  by your first trusted proxy, counted from the right. Rate-limiting or geo-blocking by the leftmost
  entry is a vulnerability.
- **Redirects change methods.** 301/302 historically let clients switch POST to GET and drop the
  body; 307/308 preserve method and body. An HTTP→HTTPS or trailing-slash redirect in front of a POST
  endpoint silently turns writes into reads.
- **Request smuggling.** When front and back proxies disagree on message length (`Content-Length`
  vs `Transfer-Encoding: chunked`, or an HTTP/2 → HTTP/1.1 downgrade), one request can be smuggled
  inside another. Symptom in production: users occasionally receive someone else's response, or
  random 400s on the following request on the same connection.
- **`Expect: 100-continue`.** curl and some .NET/Java clients send it for bodies over ~1 KB and wait
  (curl: 1 s) for a `100 Continue`. Servers or proxies that ignore it add a flat 1 s to every large
  POST.

### 3. TCP in production

`A12`, `C02`, `O02`.

- **TIME_WAIT.** The side that closes first holds the 4-tuple in TIME_WAIT for 2×MSL — fixed at 60 s
  on Linux. Harmless on servers (it is keyed per remote tuple), dangerous on *clients* making many
  short connections to one destination: every closed connection pins a local port for 60 s.
- **Ephemeral port exhaustion.** Linux `ip_local_port_range` defaults to 32768–60999, about 28 k
  ports. 28 k / 60 s ≈ 470 new connections per second to a single `ip:port` before `connect()` fails
  with `EADDRNOTAVAIL` ("Cannot assign requested address"). Classic cause: an HTTP client created per
  request (new pool each time), or `Connection: close` everywhere. Fixes, in order: reuse
  connections; widen the port range; `net.ipv4.tcp_tw_reuse=1` (safe for outbound, needs
  timestamps); spread across more destination IPs. Never `tcp_tw_recycle` — it broke clients behind
  NAT and was removed in Linux 4.12.
- **NAT and SNAT exhaustion.** An AWS NAT Gateway supports about 55 000 simultaneous connections to
  each unique destination (IP, port, protocol); beyond that, `ErrorPortAllocation` in CloudWatch and
  failed connects. Azure's default SNAT allocation can be as low as 1024 ports per instance. Talking
  heavily to one SaaS endpoint through NAT is the typical trigger. Fixes: connection reuse, more NAT
  IPs, VPC endpoints / PrivateLink so traffic does not traverse NAT.
- **NAT and firewall idle timeouts.** AWS NAT Gateway drops idle flows after 350 s; AWS NLB after
  350 s (TCP, configurable); GCP Cloud NAT established-TCP idle 1200 s; many corporate firewalls
  far lower. The flow disappears silently: the next packet on it gets a RST or is black-holed.
  Database connections and gRPC channels idle over lunch die this way. TCP keepalive defaults
  (`tcp_keepalive_time` 7200 s) are far too long to help — set application or socket keepalives
  below the smallest idle timeout on the path.
- **conntrack.** Linux netfilter tracks every flow through NAT/iptables (Kubernetes nodes do this
  for Services). When `nf_conntrack_max` is reached the kernel logs `nf_conntrack: table full,
  dropping packet` and new connections time out with SYNs silently dropped. `conntrack -S` shows
  `insert_failed` and `drop` counters.
- **Accept backlog.** `listen()` backlog is capped by `net.core.somaxconn` (128 before kernel 5.4,
  4096 since). A full accept queue under a burst drops SYNs; clients retransmit after 1 s, then 3 s,
  giving tell-tale latency spikes at exactly +1 s and +3 s. `nstat -az TcpExtListenOverflows` counts it.
- **Nagle plus delayed ACK.** Nagle holds small writes until the previous segment is acked; delayed
  ACK holds the ack up to 40 ms on Linux (200 ms on Windows). An app that writes headers and body in
  two small `write()`s gets a 40 ms or 200 ms stall per request. Fix: `TCP_NODELAY` (most HTTP
  libraries set it; custom protocols and some older clients do not) or write the request in one
  buffer.
- **RST vs FIN.** A FIN is an orderly close; a RST is abortion. A server that closes a socket with
  unread data in its receive buffer sends RST, which can destroy the response it just sent before the
  client reads it — the reason graceful servers drain the request body before closing.
- **Retransmits and tail latency.** Minimum RTO on Linux is 200 ms; a single lost SYN costs 1 s. A
  0.1% packet loss is invisible on averages and dominant at p99.9.
- **Bandwidth-delay product.** Throughput per connection ≤ window / RTT. A 64 KB window across a
  100 ms link caps at about 5 Mbit/s regardless of pipe size. Window scaling usually fixes TCP; HTTP/2
  and gRPC flow-control windows reintroduce the cap (§5).
- **CLOSE_WAIT and FIN timeouts.** `CLOSE_WAIT` means the peer sent FIN and *our* process has not
  closed its socket — always an application leak (unconsumed response bodies, connections never
  returned to the pool), ending in file-descriptor exhaustion. `tcp_fin_timeout` governs FIN_WAIT_2,
  not TIME_WAIT, so lowering it does nothing for port exhaustion.
- **Lingering close.** To make sure an error response (413, 400) reaches a client that is still
  uploading, servers half-close and read-and-discard for a short time before closing (nginx
  `lingering_close`), avoiding the RST described above.
- **Connection counts multiply.** Pools are per process: total connections = processes per pod ×
  pods × pool size, and a sidecar proxy adds a second pooled hop with its own idle timeouts, to which
  the ordering rule in §1 applies again. Source-IP hashing (`balance source`) collapses customers
  behind NAT onto a few backends; balance per connection (least-connections) or per request instead.

### 4. MTU and Path MTU Discovery black holes

`A12`, `O02`.

- Standard Ethernet MTU is 1500 bytes; inside an AWS VPC, instances default to 9001 (jumbo frames),
  but traffic leaving via an internet gateway, VPN or peering across regions is 1500 or less. IPsec,
  GRE, WireGuard and VXLAN overlays (Kubernetes CNIs) consume 50–80 bytes of headroom.
- **PMTUD** works by sending packets with Don't Fragment set and relying on routers to return ICMP
  "Fragmentation Needed" (type 3, code 4; ICMPv6 "Packet Too Big"). Security groups or firewalls that
  block *all* ICMP break it: large packets vanish, small packets pass.
- **Signature symptom.** TCP handshake succeeds (small packets); small requests work; anything
  large hangs until timeout. TLS is the textbook case: ClientHello goes out, the ServerHello plus
  certificate chain (several KB) never arrives, so `curl -v` stops after "Client hello". Also: `git
  clone` hangs over VPN, POSTs above ~1.4 KB hang, SSH connects but `ls` of a large directory freezes.
- **Diagnosis.** `ping -M do -s 1472 host` (1472 + 28 header bytes = 1500) and step down until it
  passes; `tracepath` reports the discovered PMTU; `tcpdump` shows repeated retransmits of
  full-size segments with no ICMP back.
- **Fixes.** Allow ICMP type 3 code 4 (and ICMPv6 type 2) through firewalls; clamp MSS at the tunnel
  (`iptables ... --clamp-mss-to-pmtu`); lower the interface MTU; enable
  `net.ipv4.tcp_mtu_probing=1` so the kernel probes when black-holing is detected.

### 5. HTTP/2 and HTTP/3 in production

`A10`.

- **One connection, many streams.** An HTTP/2 client normally opens a single connection per origin
  and multiplexes everything on it. Consequences: all traffic from one client lands on one backend
  behind an L4 balancer; a stall of that one TCP connection stalls every stream.
- **Head-of-line blocking moved, not removed.** HTTP/2 removed HTTP-level queueing but a single lost
  TCP segment blocks every stream until retransmitted. On lossy mobile networks HTTP/2 can be slower
  than six HTTP/1.1 connections. HTTP/3 (QUIC over UDP) fixes this per stream — but UDP 443 is
  blocked on many corporate networks, so clients fall back after a race; and UDP gets less NIC
  offload, so QUIC costs more CPU per byte on servers.
- **`SETTINGS_MAX_CONCURRENT_STREAMS`.** Typically 100 (nginx `http2_max_concurrent_streams 128`,
  Go default 250, many CDNs 100). A client that fires 500 concurrent requests on one connection
  queues 400 of them *client-side*, and it looks like server latency. Some clients open extra
  connections when the limit is reached; many (older gRPC-Java, some Node versions) do not.
- **Flow control windows.** The HTTP/2 default initial window is 65 535 bytes per stream and per
  connection. Over a high-RTT link a large download on a server that never raises the window runs at
  window/RTT — e.g. 64 KB / 150 ms ≈ 3.5 Mbit/s — even though plain HTTPS on the same link is fast.
  Fix: raise `SETTINGS_INITIAL_WINDOW_SIZE` and connection window, or use BDP-based dynamic windows
  (gRPC does this).
- **GOAWAY.** How a server tells a client to stop opening new streams on a connection (graceful
  shutdown, max connection age, too many pings). Clients that ignore it or retry non-idempotent
  in-flight streams mis-handle deploys.
- **Header rules.** Header names must be lowercase; connection-specific headers (`Connection`,
  `Keep-Alive`, `Transfer-Encoding`, `Upgrade`) are forbidden. A backend that emits
  `Connection: keep-alive` in an HTTP/2 response makes strict clients reject it with
  `PROTOCOL_ERROR`. HPACK makes repeated large headers cheap on one connection, but the first request
  and every new connection pay full price.
- **Downgrade at the edge.** Most LBs and CDNs speak HTTP/2 to clients and HTTP/1.1 to origins, so
  your origin never sees multiplexing — and the translation is where smuggling bugs live (§2).
- **Rapid Reset (CVE-2023-44487).** Clients open and immediately cancel streams, so
  `MAX_CONCURRENT_STREAMS` never bounds the work. Mitigation is to cap the rate of resets per
  connection and close abusive connections, not to lower the stream limit.

### 6. gRPC in production

`A14`, `M07`.

- **L4 load balancing does not balance gRPC.** gRPC keeps one long-lived HTTP/2 connection per
  backend and multiplexes calls on it. A Kubernetes `ClusterIP` Service (kube-proxy, L4) picks a pod
  once per *connection*, so each client pins to one pod forever. Scale from 3 to 10 pods and the new
  7 get nothing; restart one pod and its clients pile onto the survivors and never come back.
- **Three fixes, with trade-offs.**
  1. **Client-side balancing**: a headless Service (DNS returns all pod IPs), `dns:///` target and
     `round_robin` policy. Cheap, but depends on DNS re-resolution, which gRPC only does on
     connection failure or GOAWAY.
  2. **L7 proxy** (Envoy, Linkerd, Istio, gRPC-aware ALB target groups) balancing per request.
  3. **`MAX_CONNECTION_AGE` on the server** (e.g. 5–30 min with `MAX_CONNECTION_AGE_GRACE`): the
     server sends GOAWAY periodically, clients reconnect and re-resolve, and load re-spreads. The
     unconventional but most portable fix; add jitter (gRPC does ±10%) so reconnects do not align.
- **Keepalive pings and `too_many_pings`.** Client keepalive (`GRPC_ARG_KEEPALIVE_TIME_MS`) keeps
  connections alive through NAT/LB idle timeouts (§3). But servers enforce a minimum ping interval
  (default 5 min, `permit_keepalive_time`) and reply GOAWAY `ENHANCE_YOUR_CALM` with debug data
  `too_many_pings` if the client pings more often. Both ends must be configured together.
- **Deadlines.** gRPC has no default deadline; a call without one can wait forever. Deadlines
  propagate across hops (the remaining budget is passed in `grpc-timeout`), so set them at the edge
  and let them shrink. `DEADLINE_EXCEEDED` on the server side of a chain often means the *caller's*
  budget was spent upstream.
- **Status codes.** `UNAVAILABLE` is the retryable one (connection failure, GOAWAY). `UNKNOWN` is
  usually an unhandled server exception. `RESOURCE_EXHAUSTED` covers both rate limits and
  message-size limits. `INTERNAL` with "RST_STREAM" often means a proxy killed the stream.
- **Message size.** Default max *receive* size is 4 MB. Response grows past 4 MB → `RESOURCE_EXHAUSTED`
  on the client only for the tenants with large data. Stream it, paginate it, or raise the limit
  knowingly.
- **Unary messages are buffered whole.** A unary call's request and response are held fully in
  memory on both sides; large blobs belong in client/server streaming in chunks, or in object storage
  with a reference passed over gRPC.
- **Retries and hedging.** Configured via service config (`retryPolicy`, `hedgingPolicy`) with
  `retryThrottling` (a token-bucket retry budget). Hedging cuts tail latency but multiplies load;
  only for idempotent methods.
- **gRPC through things that are not gRPC-aware.** Proxies that do not support HTTP/2 trailers break
  gRPC (status is in trailers). Browsers cannot speak native gRPC — gRPC-Web or Connect needs a
  translating proxy. An LB that does HTTP/2 to the client but HTTP/1.1 to the target returns errors
  such as `upstream connect error` or 502s for every call.

### 7. DNS in production

`A13`.

- **TTL is a request, not a guarantee.** Resolvers, OSes, runtimes and libraries each cache on their
  own terms. Lowering a TTL only helps once the *old* TTL has expired everywhere — so you lower it
  days before a migration, not during.
- **Runtime caching.**
  - **JVM**: `networkaddress.cache.ttl` defaults to 30 s without a security manager, and to *forever*
    with one (older app servers). Negative results cached 10 s (`networkaddress.cache.negative.ttl`).
    A JVM that "never notices the failover" is often caching forever.
  - **Node.js**: `dns.lookup` (used by `http`) calls `getaddrinfo` on the libuv threadpool (default 4
    threads, `UV_THREADPOOL_SIZE`) and does **no caching**. Slow DNS saturates the pool and also stalls
    `fs` and `crypto` work sharing it. High request rates hammer the resolver. Fix: a caching lookup
    (e.g. `cacheable-lookup`), keep-alive agents, larger threadpool.
  - **Go**: pure-Go resolver by default on Linux, no cache; cgo resolver when `nsswitch` demands it.
  - **Connection pools** cache DNS implicitly: an established connection keeps talking to the old IP
    until it is closed. Blue/green by DNS flip fails for clients with long-lived pools — you also
    need a maximum connection lifetime.
  - **nginx** resolves `proxy_pass` hostnames once at start-up (§2). An upstream whose IPs change
    (ELB, RDS failover) gets traffic sent to dead IPs until reload.
- **Kubernetes `ndots:5`.** Pod resolv.conf has `ndots:5` and several search domains, so
  `api.stripe.com` (2 dots) is first tried as `api.stripe.com.default.svc.cluster.local`,
  `...svc.cluster.local`, `...cluster.local`, and any VPC domain — each for A and AAAA — before the
  real name: up to 10 queries per lookup. Fix: trailing dot (`api.stripe.com.`) or lower `ndots` in
  `dnsConfig`.
- **The 5-second DNS delay.** glibc sends A and AAAA queries in parallel from the same UDP socket; a
  conntrack race on the node can drop one, and glibc waits the default 5 s timeout before retrying.
  Symptom: a small fraction of requests take exactly 5 s (or 2.5 s, 10 s) longer. Fixes:
  `options single-request-reopen` / `use-vc`, NodeLocal DNSCache, or disabling AAAA where not needed.
- **Resolver rate limits.** The AWS VPC resolver (`.2` address) allows 1024 packets per second per
  ENI; exceed it and queries are dropped silently. Uncached per-request lookups from a busy node hit
  it. CoreDNS pods can also saturate; watch their CPU and `SERVFAIL` counts.
- **Negative caching.** An NXDOMAIN is cached for the SOA minimum / negative TTL. Querying a record
  *before* creating it (a health check racing a deploy) caches its absence for minutes to hours.
- **IPv6 surprises.** A new AAAA record, or a client preferring IPv6 with a broken v6 path, gives
  connect timeouts to some clients only. Happy Eyeballs (RFC 8305) masks it in browsers and curl
  but not in every runtime.
- **Diagnosis.** `dig +trace` to follow delegation; `dig @8.8.8.8` vs `dig @<internal>` for split
  horizon; `dig +short` and TTL countdown on repeated queries; `getent hosts` to see what libc (and
  thus most apps) actually returns, which may differ from `dig` because of `/etc/hosts`, nsswitch
  and search domains.

### 8. TLS and certificates

`A11`, `S10`, `S15`.

- **Handshake cost.** TLS 1.2 adds 2 round trips, TLS 1.3 adds 1, plus TCP's 1. At 150 ms RTT a
  cold HTTPS request costs 300–450 ms before the first byte. RSA-2048 private-key operations cost
  roughly an order of magnitude more server CPU than ECDSA P-256. A fleet whose clients do not reuse
  connections spends its CPU on handshakes; "CPU doubled after we moved from HTTP to HTTPS internally"
  is almost always missing keep-alive, not encryption cost.
- **Resumption.** Session tickets or IDs skip the key exchange on reconnect. Behind a load balancer
  with many nodes, tickets only work if the ticket keys are shared (or the LB terminates TLS).
  TLS 1.3 **0-RTT** early data is replayable — only safe for idempotent requests (`A02`); servers
  should reject or mark early data on writes (`Early-Data: 1`, status 425 Too Early).
- **Chain problems.** The server must send the leaf plus intermediates. Browsers paper over a missing
  intermediate (they cache intermediates and fetch via AIA), but curl, Java, Go, Python and mobile
  apps fail with "unable to get local issuer certificate" / `PKIX path building failed`. Symptom:
  "works in Chrome, fails from our backend". Check with `openssl s_client -showcerts`.
- **Root expiry and trust stores.** When Let's Encrypt's old root (DST Root CA X3) expired on
  30 September 2021, clients with old trust stores or OpenSSL 1.0.2's path-building bug failed even
  though the leaf was valid. Containers with a stale `ca-certificates` package, pinned JDK trust
  stores and IoT devices are where this lives.
- **Certificate lifetimes are shrinking.** The CA/Browser Forum schedule reduces maximum public
  lifetime from 398 days to 200 days (March 2026), 100 days (2027) and 47 days (2029). Anything
  renewed by a human, or pinned by a partner, will break more often. Automate (ACME) and monitor
  expiry from *outside* — including every intermediate and every SNI name.
- **Revocation.** OCSP is soft-fail in browsers (an unreachable responder is ignored), adds a lookup,
  and leaks browsing. OCSP stapling fixes latency; "must-staple" turns a stapling outage into a hard
  outage. Let's Encrypt retired OCSP in 2025 in favour of CRLs, so tooling that required an OCSP URL
  broke.
- **SNI.** The client sends the hostname in the ClientHello so one IP can serve many certificates.
  Clients that connect by IP address, or very old clients, send no SNI and receive the *default*
  certificate — a hostname mismatch. Some LBs pick the certificate by SNI but route by `Host`, so a
  mismatch between the two can be abused or mis-routed (domain fronting).
- **Clock skew.** Certificates carry `notBefore`/`notAfter`. A device or container with a wrong clock
  (no RTC, bad NTP, a VM restored from snapshot) sees valid certs as "not yet valid" or "expired".
  A freshly issued cert served immediately fails for clients whose clocks are a few minutes behind —
  CAs backdate `notBefore` by an hour for this reason.
- **Pinning.** Pinning a leaf or intermediate in a mobile app or partner integration turns every
  routine rotation into an outage. Pin the public key of a CA or backup key, if at all.
  Certificate Transparency monitoring (alerts on certificates issued for your domains) gives much of
  the protection without the outage risk.
- **Rotation is a client-matrix problem.** Clients validate differently: old Android trust stores,
  containers with stale `ca-certificates`, JDK trust stores, health checkers that pin an
  intermediate. Test a new chain against the real client matrix before rotating.
- **mTLS.** Client certs expire too, often on devices you cannot update. LBs that terminate mTLS pass
  the verified identity to the app in a header — which must be stripped from inbound requests or it is
  forgeable.

### 9. HTTP caching and CDNs

`A04`, `Q09`, `SD05`.

- **The cache key is everything.** By default a CDN keys on method + host + path + (often) query
  string. Anything else that changes the response must be in the key, or users see each other's
  content; anything in the key that does not change the response destroys the hit ratio.
- **`Vary`.** Tells caches which request headers are part of the key. `Vary: Accept-Encoding` is
  normal. `Vary: User-Agent` or `Vary: Cookie` makes practically every request unique — hit ratio
  collapses to near zero. Missing `Vary: Origin` on responses with a per-origin
  `Access-Control-Allow-Origin` lets a CDN cache the header for the first origin and serve it to all
  others (§10). Missing `Vary: Accept` on content negotiation serves JSON to browsers wanting HTML.
- **Cookies and `Set-Cookie`.** Many CDNs (and Varnish by default) will not cache a response carrying
  `Set-Cookie`, or a request carrying cookies. A framework that sets a session or tracking cookie on
  every response quietly disables the CDN. Worse, a CDN configured to ignore cookies *and* cache a
  `Set-Cookie` response hands one user's session to everyone.
- **`Cache-Control` precision.** `private` = browsers only; `no-cache` = store but revalidate;
  `no-store` = do not store; `s-maxage` overrides `max-age` for shared caches; `stale-while-revalidate`
  serves stale while refreshing in the background; `stale-if-error` keeps serving through an origin
  outage. Responses to requests with `Authorization` are not cached by shared caches unless marked
  `public` or `s-maxage`. CloudFront applies a default TTL of 24 h to responses without caching
  headers — including your error pages unless error caching is configured.
- **Negative caching.** CDNs cache 404s and sometimes 5xx for a short TTL. A 404 cached during a
  deploy race survives the fix. Explicit short TTLs for errors, and purge-on-deploy, avoid it.
- **ETags across a fleet.** Apache-style ETags built from inode + mtime differ per server, so
  revalidation on a different node returns 200 instead of 304. nginx weakens ETags (`W/`) when it
  gzips. A strong ETag computed before compression and then served compressed breaks range requests.
- **Stampedes.** A hot key expiring at once sends every edge's misses to origin together. Request
  collapsing/coalescing (CDN "origin shield", nginx `proxy_cache_lock`), `stale-while-revalidate`,
  and jittered TTLs smooth it.
- **Cache poisoning.** Unkeyed inputs that influence the response (`X-Forwarded-Host` used to build
  absolute URLs, `X-Original-URL`, unkeyed query parameters) let an attacker store a malicious
  response for everyone. **Web cache deception**: `/account/settings/x.css` served by a framework that
  ignores the suffix, cached by a CDN that caches by extension.
- **Purging.** Purge by URL does not scale; tag responses with surrogate keys (`Surrogate-Key`,
  `Cache-Tag`) and purge by tag. Purges are eventually consistent across POPs — seconds to minutes.
- **Query strings.** `?a=1&b=2` and `?b=2&a=1` are different keys unless normalised; tracking
  parameters (`utm_*`, `fbclid`) should be stripped from the key.
- **Normalise varying headers.** If a header must vary the response (`Accept-Language`, device
  class), map it at the edge to a handful of canonical values before it enters the key; a raw
  `User-Agent` carrying an app version is unique per release.

### 10. CORS in production

`A19`, `S08`.

- **When a preflight happens.** Any non-simple request: methods other than GET/HEAD/POST, any custom
  header (`Authorization`, `X-Request-Id`), or `Content-Type` other than
  `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`. So `application/json`
  POSTs always preflight.
- **The cost.** A preflight is a full extra round trip (plus TLS if the connection is new) before the
  real request. Cached per origin + URL + method/headers, for `Access-Control-Max-Age` — but browsers
  cap it (Chrome 2 h, Firefox 24 h), and without the header Chrome caches for only 5 s. Because the
  cache is **per URL**, REST APIs with IDs in the path (`/orders/123`) preflight on every distinct
  resource. Mitigations: long `Max-Age`, serve the API same-origin via a path on the main domain
  (no CORS at all), or avoid triggers for hot read paths.
- **Preflights and auth.** OPTIONS requests carry no credentials. An auth middleware that runs
  before the CORS handler returns 401 to the preflight and every call fails with a CORS error.
- **CORS errors hide real errors.** If a 500 or 502 response lacks `Access-Control-Allow-Origin`
  (because it came from the proxy or an exception handler that skipped the CORS middleware), the
  browser reports a CORS failure and JavaScript cannot read the status. Many "CORS bugs" are server
  errors in disguise; look at the network tab status, not the console message.
- **Credentials.** With `credentials: 'include'`, `Access-Control-Allow-Origin: *` is rejected; you
  must echo a specific origin and send `Access-Control-Allow-Credentials: true` — and then `Vary:
  Origin` is mandatory (§9). Reflecting any origin with credentials is a CSRF-class vulnerability.
- **Exposed headers.** JavaScript can read only safelisted response headers unless listed in
  `Access-Control-Expose-Headers` — the reason a frontend "cannot see" `X-Total-Count`, `ETag` or
  `Retry-After`.
- **Private Network Access.** Chrome's rules for public sites calling private IPs (localhost,
  10.x) add a preflight with `Access-Control-Request-Private-Network` and are rolling out in stages;
  internal tools and local dev agents break when they land.

### 11. Webhooks

`A16`.

- **At-least-once, unordered, delayed.** Every serious provider retries on non-2xx or timeout
  (Stripe for up to 3 days with exponential backoff; GitHub does not auto-retry at all; Shopify
  about 48 h). Duplicates and reordering are normal. Consumers must dedupe on the event ID and must
  not assume `created` arrives before `updated`.
- **Thin events.** The robust pattern: treat the webhook as a notification, then fetch current state
  from the provider's API (or compare a version/sequence number) before acting. Replaying an old
  `updated` event then cannot overwrite newer state.
- **Acknowledge fast, process async.** Providers time out quickly (Stripe ~10–20 s, GitHub 10 s).
  Persist the raw event, return 2xx, process from a queue. Slow synchronous handlers cause timeouts,
  which cause retries, which cause duplicates, which cause more load.
- **Signatures.** HMAC-SHA256 over the **raw request bytes**. Frameworks that parse JSON and
  re-serialise before verification change whitespace, key order or Unicode escaping, so signatures
  fail intermittently (only on payloads with non-ASCII or floats). Verify first, parse after. Use
  constant-time comparison.
- **Replay protection.** Sign a timestamp together with the body (Stripe: `t=...,v1=...`, signing
  `t.body`) and reject outside a tolerance (Stripe default 300 s). This makes clock skew on the
  receiver a failure mode (§20).
- **Secret rotation.** Support multiple valid signatures during rotation (Stripe sends several `v1`
  values; Standard Webhooks allows a space-separated list), otherwise rotation is an outage.
- **Sender side.** If you *send* webhooks: queue per destination so one slow customer does not
  delay everyone (head-of-line blocking); cap concurrency and apply circuit breakers per endpoint;
  auto-disable endpoints after sustained failure and notify; sign and timestamp; include an event ID
  and sequence; let customers replay from a dashboard. Expose an events API that lists events after a
  cursor, so consumers can detect gaps by sequence and reconcile after outages instead of trusting
  delivery. Customer-supplied URLs are an **SSRF** vector:
  resolve the hostname, block private/link-local ranges (169.254.169.254), pin the resolved IP for
  the connection to defeat DNS rebinding, and do not follow redirects blindly.
- **Receiver infrastructure.** WAFs, bot protection and CDN challenges block webhook POSTs (no
  browser, no JavaScript). Provider IP allowlists change; prefer signature verification over IP
  filtering.

### 12. Realtime: SSE, WebSockets and long polling

`A16`, `F16`.

- **Buffering proxies.** nginx buffers upstream responses by default; SSE events arrive in bursts or
  only when the buffer fills. Fix: `proxy_buffering off` for that location, or the response header
  `X-Accel-Buffering: no`. Compression middleware also buffers (gzip needs data to compress); disable
  it for `text/event-stream` or flush explicitly. Some CDNs buffer entire responses.
- **Idle timeouts.** Every hop (ALB 60 s, nginx `proxy_read_timeout` 60 s, corporate proxies)
  closes a quiet stream. Send heartbeats (SSE comment line `:\n`, WebSocket ping) more often than the
  smallest timeout on the path.
- **Connection limits.** Browsers allow six HTTP/1.1 connections per origin; each SSE stream holds
  one, so the seventh tab hangs. HTTP/2 removes this. WebSockets each hold a full connection and file
  descriptor; tune `ulimit -n` and LB connection quotas.
- **Reconnect storms.** A deploy or LB failover drops 200 000 sockets at once; every client
  reconnects immediately and re-fetches state, producing a thundering herd that looks like a DDoS.
  Fixes: randomised reconnect delay with backoff, drain servers gradually (close connections in
  batches over minutes), SSE `retry:` field, and `Last-Event-ID` resumption so reconnects fetch only
  the gap.
- **Sticky state.** WebSocket servers hold per-connection state; scaling out requires a pub/sub
  fan-out (Redis, NATS) so a message published on node A reaches a client on node B. Autoscaling on
  CPU does not rebalance existing sockets — new nodes stay empty (same shape as §6).
  Route new connections by least-connections and actively shed a fraction of connections from hot
  nodes (clients reconnect with jitter) to rebalance.
- **Share streams across tabs.** A `SharedWorker` or `BroadcastChannel` lets one tab hold the stream
  and relay to the others, sidestepping the per-origin connection limit on HTTP/1.1.
- **Long polling.** Holding requests open for 55 s fails behind corporate proxies with 30 s limits,
  and thread-per-request servers burn a worker per waiting client. Keep polls below common proxy
  limits (20–25 s) and host them on an async server, or move to SSE.
- **Isolate realtime at the edge.** Reconnect storms consume LB capacity, connection-rate limits and
  TLS handshakes; put realtime on its own hostname and load balancer so REST traffic is not
  collateral damage.
- **Upgrade through proxies.** WebSockets need `Upgrade`/`Connection` headers forwarded (nginx
  requires explicit `proxy_set_header Upgrade $http_upgrade`); HTTP/2 WebSockets (RFC 8441) are not
  universally supported.

### 13. Pagination at scale

`A05`, `DB20`.

- **Offset is O(offset).** `OFFSET 500000 LIMIT 50` reads and discards half a million rows; deep
  pages get linearly slower, and crawlers or export scripts that walk to page 10 000 take the
  database down. Offset pages also skip or repeat rows when data is inserted or deleted between
  requests.
- **Keyset (cursor) pagination.** `WHERE (created_at, id) < (:c, :i) ORDER BY created_at DESC, id
  DESC LIMIT 50` with a composite index. Constant cost per page and stable under inserts. Needs a
  unique tiebreaker: sorting on a non-unique column alone loses or duplicates rows at page
  boundaries when many rows share a value (e.g. bulk-imported rows with the same timestamp).
- **Cursor design.** Opaque (base64 of the sort key values), optionally signed or encrypted so
  clients cannot craft them, versioned so you can change the sort without breaking clients. Encode
  the filter and sort in the cursor, or reject a cursor used with different filters.
- **Counts.** `COUNT(*)` on a large filtered set is often more expensive than the page itself.
  Return `has_more` (fetch `limit + 1`), estimated counts, or counts capped at "10 000+".
- **Sorting by mutable fields.** Paginating by `updated_at` while rows are updated moves them across
  pages: a sync job reading "everything changed since X" misses rows updated mid-scan, or loops. For
  sync, use an append-only change sequence (a monotonic ID or log position), and beware that
  transaction commit order is not ID order — a row with a lower ID can commit after you passed it.
  The usual trick is to only read up to "now minus a safety lag" or use a snapshot.
- **Page size limits.** Enforce a maximum; clients will ask for `limit=100000`.
  Likewise allowlist sortable fields that have supporting indexes — `sort=description` on 50 million
  rows is a denial-of-service.
- **Cursors are untrusted input.** Bind them to tenant and filters, sign them, and still authorise
  every row returned.
- **No snapshot, no consistent copy.** Paging over live data never yields a point-in-time copy. For
  full syncs, offer a snapshot token (a consistent read point), or a full export plus a change feed
  starting from a known position.

### 14. Idempotency keys: the edge cases

`A09`, `A02`, `M09`.

- **Scope.** The key is scoped to (tenant/API key, endpoint) — not global, or two customers' UUID
  collisions or a replay against a different endpoint returns someone else's result.
- **Fingerprint the request.** Store a hash of the request body with the key. Same key, different
  body → reject (`422`/`409`) rather than silently returning the first result for different input.
- **Concurrent duplicates.** Two requests with the same key arriving together (client timeout and
  retry while the first is still running). Insert the key row first with a unique constraint and
  state `in_progress`; the loser gets `409 Conflict` (retry later), not a second execution.
- **Atomicity with the effect.** Record the key and the business write in the **same database
  transaction**. Keys in Redis and orders in Postgres can disagree after a crash: the key says "done"
  and there is no order, or vice versa.
- **External side effects.** If the operation calls a payment provider, pass an idempotency key
  *downstream* too (derived deterministically from yours), and design as recovery points: a crash
  after the charge but before your commit must be resumable, not re-charged.
- **What to store on failure.** Cache 2xx and deterministic 4xx responses. Do **not** cache 5xx or
  timeouts as final — the client must be able to retry into a fresh attempt, but only after the
  in-progress lock is released or expired.
- **Retention.** Stripe keeps keys 24 h. A client retrying a day later, or a queue redelivering after
  a long outage, falls outside the window and duplicates. Business-level uniqueness (one payment per
  invoice ID) is the backstop.
- **Client bugs.** The classic: generating the key inside the retry loop instead of once per logical
  operation, so each retry has a new key. Another: a mobile app regenerating the key after the app is
  killed and restored.
- **Gateways and caches.** A proxy that retries POSTs on timeout (§2) without the key being honoured
  end-to-end creates duplicates the client never sent.

### 15. Rate limiting and load shedding

`A08`, `M06`, `SD11`.

- **Algorithms.** *Fixed window*: cheap, but allows 2× the limit across a window boundary.
  *Sliding log*: exact, memory-heavy. *Sliding window counter*: weighted blend of current and
  previous window — accurate enough, two counters per key. *Token bucket*: rate plus burst, the usual
  public-API choice. *Leaky bucket*: smooths output rate (queue). *GCRA*: token bucket implemented as
  one timestamp per key ("theoretical arrival time"), ideal for Redis.
- **Distributed counters.** A Redis `INCR` + `EXPIRE` pair is not atomic (a crash between them leaves
  a key without TTL that blocks the user forever); use a Lua script or `SET ... NX EX` plus `INCR`.
  One Redis key per (user, window) makes a hot shard for a heavy tenant. Latency: a network call per
  request; many large systems use local token buckets with periodic sync to a global store,
  accepting some overshoot.
- **Per-instance limits multiply.** A limit of 100 req/s implemented in each pod's memory becomes
  100 × N; autoscaling from 4 to 20 pods raises the effective limit five-fold, and a scale-in lowers
  it. Divide by replica count, or centralise.
- **What to key on.** IP is wrong behind NAT/CGNAT (a whole mobile carrier or office shares one IP)
  and spoofable via `X-Forwarded-For` if parsed carelessly (§2). API key or user is better; cost
  (weighted by endpoint expense) is better still for mixed workloads.
- **Responses.** `429` with `Retry-After`; the IETF `RateLimit` / `RateLimit-Policy` headers tell
  clients their budget. Without `Retry-After`, clients retry immediately and a 429 storm amplifies
  load. Synchronised clients (all cron jobs at :00) hit the limit together; jitter matters on both
  sides.
- **Rate limits vs concurrency limits.** Rate limiting protects fairness; it does not protect the
  server from slow requests. Little's law: concurrency = throughput × latency. When latency doubles,
  the same request rate needs double the workers. Adaptive concurrency limits (Netflix
  `concurrency-limits`, Envoy adaptive concurrency) shed load based on observed latency, which is what
  actually stops collapse.
- **Load shedding order.** Shed at the edge, cheaply, before doing work; prefer rejecting new work
  to timing out accepted work; prioritise (health checks, paying customers, idempotent reads) and
  drop the lowest class first. LIFO queueing under overload serves fresh requests whose clients are
  still waiting instead of stale ones whose clients already gave up.

### 16. Timeouts, retries and deadlines at the API edge

`M09`, `A07`.

- **Timeouts must shrink inward.** Edge 30 s, service A 25 s, service B 20 s. If an inner timeout is
  longer than an outer one, the inner work continues after the client has gone — wasted capacity, and
  for writes an unknown outcome.
- **Retry amplification.** Three layers each retrying three times turns one failure into 27 calls
  at the bottom. Retry at one layer (usually the outermost that knows idempotency), use a retry
  budget (e.g. retries ≤ 10% of requests), exponential backoff with **full jitter**.
- **Bulkheads.** Separate connection and thread pools per downstream so one slow dependency cannot
  consume the workers every other endpoint needs.
- **Proxy retry semantics differ.** Envoy's `retry_on` conditions (`reset`, `5xx`,
  `connect-failure`, per-try timeouts) are broader than nginx's defaults; a reset *after* the request
  was sent is not a connect failure and the request may have executed. Retries and hedges also count
  against rate limits unless excluded (e.g. by idempotency key).
- **Client disconnects.** When the client gives up, most servers keep working. Propagate
  cancellation (Go `context`, gRPC deadlines, `AbortSignal` in Node, checking `request.is_disconnected()`)
  so abandoned work stops — otherwise a slow dependency causes work to pile up for requests nobody
  is waiting for.
- **Hedged requests.** Send a second copy after the p95 latency elapses and take the first answer:
  cuts tail latency at a few percent extra load. Only for idempotent reads, and cap it.
- **Classify errors.** Retry: connection reset before response, 502/503/504, 429 (after
  `Retry-After`), gRPC `UNAVAILABLE`. Do not retry: 400, 401, 403, 404, 409, 422. Retrying a 500 on a
  non-idempotent call without an idempotency key is gambling.
- **Error contracts.** An API returning `200 {"error": ...}` defeats every retry policy, LB health
  signal and SLO dashboard built on status codes. Use proper status plus `application/problem+json`
  (`A07`).

### 17. GraphQL in production

`A15`, `DB20`.

- **N+1.** Resolvers run per field per object: a list of 100 orders each resolving `customer` issues
  100 queries. **DataLoader** batches loads within one tick of one request into a single `IN (...)`
  query and caches per request. The cache must be per request — a global DataLoader leaks data between
  users.
- **Query cost attacks.** A single query can nest (`friends { friends { friends ... } }`), use
  aliases to call an expensive field 1000 times in one request, or send arrays of operations in one
  HTTP request (batching) to multiply work and bypass per-request rate limits. Defences: depth limit,
  query cost analysis (static estimate of fields × list sizes, reject over budget), alias and batch
  limits, timeouts per resolver, and rate limiting by *cost*, not requests.
- **Persisted queries.** Allow only pre-registered query hashes in production (trusted documents).
  Kills arbitrary-query attacks and makes GET + CDN caching possible (APQ sends the hash in a GET).
- **Monitoring blind spot.** GraphQL returns 200 with an `errors` array even when resolvers fail, and
  every request is `POST /graphql`. Status-code dashboards show 100% success during an outage.
  Instrument by operation name and count `errors[]` entries.
- **Null propagation.** A non-null field that errors nulls its parent, up to the nearest nullable
  ancestor — one failing field can blank an entire page. Making fields nullable by default is a
  resilience decision, not laziness.
- **Introspection and suggestions.** Disabling introspection is not enough: "Did you mean ...?" field
  suggestions leak the schema. Turn both off in production if the schema is private.
- **Caching.** No HTTP caching by URL for POST; caching moves to normalised client caches and
  per-field server caches, or APQ over GET with `Cache-Control` computed from the least cacheable
  field.
  Any server-side resolver cache must include the authorisation context (user, role, tenant) in its
  key, or cache only public data.
- **Federation.** A gateway over several subgraphs resolves entities with `_entities` calls; a naive
  plan is an N+1 at the network layer. Inspect query plans, batch entity fetches, and compute cost
  from the plan.

### 18. Serialisation and schema evolution

`A17`, `A06`, `A14`.

- **int64 in JavaScript.** JS numbers are IEEE-754 doubles, exact only up to 2^53 − 1
  (9 007 199 254 740 991). A 64-bit ID such as a Snowflake or Twitter ID above that is silently
  rounded by `JSON.parse` — two different IDs become equal, lookups return the wrong row, and nothing
  errors. Twitter's fix was `id_str`. The protobuf canonical JSON mapping encodes `int64`/`uint64` as
  **strings** for this reason. Emit large IDs as strings in JSON APIs.
- **Money and floats.** JSON has one number type; many parsers use doubles. Represent money as
  integer minor units or a decimal string. `0.1 + 0.2` in a price is a customer complaint.
- **Protobuf compatibility.** Wire compatibility depends on field *numbers*: never reuse a number
  after deleting a field — mark it `reserved`. Changing a field's type between incompatible wire types
  corrupts data silently for old readers. proto3 scalars have no presence: `0`, `""` and `false` are
  indistinguishable from "unset" unless the field is `optional` or a wrapper type — a PATCH that sets
  a discount to 0 is read as "not provided".
- **Enums.** Adding an enum value breaks old clients that switch exhaustively or reject unknown
  values (Jackson `READ_UNKNOWN_ENUM_VALUES_AS_NULL` off by default; Swift/Kotlin generated code with
  exhaustive `when`). proto3 enums must reserve `0` as `UNSPECIFIED`; unknown values are preserved
  numerically, but JSON mapping of unknown enums fails in some libraries.
- **Unknown fields.** Jackson's `FAIL_ON_UNKNOWN_PROPERTIES` defaults to **true** (Spring Boot sets it
  false) — adding a response field breaks strict Java clients. Tolerant reader is the rule for
  consumers; additive-only for producers.
- **Nullable vs absent.** JSON distinguishes `"x": null` from a missing key; many languages do not
  (Go `omitempty`, Java nulls). PATCH semantics need a representation of "set to null" vs "leave
  alone" (JSON Merge Patch, field masks in protobuf).
- **Dates.** ISO-8601 with explicit offset, always. A JSON serialiser change that drops the `Z`, or
  a server moving time zone, shifts every timestamp for clients that parse local time.
- **Hyrum's law.** With enough consumers, every observable behaviour — field order, error message
  text, default sort order, the precision of a float — is depended upon by somebody. Changes that are
  "compatible by the spec" break real clients.

### 19. Compression, large payloads and streaming

`A20`, `C19`.

- **When compression hurts.** Below ~1 KB gzip adds overhead and CPU for no gain. Brotli at level 11
  is for static assets; on dynamic responses use level 4–6 or gzip 5–6 — level 11 can cost more CPU
  than the request itself. Already-compressed content (images, zip, video) should not be recompressed.
- **BREACH.** If a response is compressed over TLS *and* reflects attacker-controlled input *and*
  contains a secret (CSRF token, API key), an attacker who can make the victim send many requests
  can infer the secret from compressed sizes. Mitigations: don't compress responses carrying secrets,
  mask tokens per request (XOR with random), separate secrets from reflected input, add length
  randomisation. **CRIME** did the same to TLS-level compression, which is why TLS compression is
  gone.
- **Decompression bombs.** Accepting `Content-Encoding: gzip` on requests means a 1 MB body can
  inflate to gigabytes. Limit the *decompressed* size, not just the wire size.
- **Body size limits everywhere.** nginx `client_max_body_size` 1 MB (413); API Gateway 10 MB;
  Lambda payload 6 MB synchronous; ALB-to-Lambda 1 MB; Cloudflare 100 MB on lower plans; gRPC 4 MB
  receive (§6); header limits around 8–16 KB (Node 16 KB total, nginx 8 KB per line, 431 Request
  Header Fields Too Large). Each layer has its own and they are discovered one at a time in production.
- **Large uploads.** Don't proxy multi-GB uploads through the API tier. Use presigned URLs directly
  to object storage, multipart/resumable uploads (S3 multipart, tus) with per-part retry, and notify
  the API on completion.
- **Large responses.** Building a 500 MB JSON array in memory OOMs the pod; streaming it (NDJSON,
  chunked transfer, cursor-driven writes) keeps memory flat — but once the first byte is sent the
  status code is fixed, so mid-stream errors need an in-band error record or a trailer. Respect
  backpressure: write only when the socket drains, or a slow client makes the server buffer
  everything. In Node, `write()` returning `false` means stop until `'drain'`;
  `stream.pipeline()` handles it for you.
- **Content-Length vs chunked.** Some clients and proxies need `Content-Length` (progress bars,
  resumption, some Lambda integrations). Compression in a proxy removes `Content-Length`.
- **Range requests.** `Range`/`206` enable resumable downloads and parallel chunked fetches; they
  require stable strong ETags (§9) and uncompressed-at-rest representations.

### 20. Authentication, signatures and clock skew

`A18`, `S03`, `S04`.

- **Clock skew breaks signed things.** JWT `exp`/`nbf`/`iat`, AWS SigV4 (requests rejected if more
  than 5 minutes off, `RequestTimeTooSkewed` / `SignatureDoesNotMatch`), webhook timestamp tolerances
  (§11), TOTP, and certificates (§8). Validators should allow a small leeway (30–120 s). A container
  host with a drifting clock produces "invalid token" errors from one node only.
- **Canonicalisation.** HMAC request-signing schemes sign a canonical form (sorted query parameters,
  specific percent-encoding, lower-cased header names). A proxy that re-encodes the URL (`%20` vs
  `+`, `%2F` decoding), reorders query parameters, or adds headers covered by the signature breaks
  verification for some requests only. Corporate proxies may also rechunk, compress or add
  `Expect: 100-continue` to bodies; signing a digest of the raw body plus a minimal header set is more
  robust than signing everything.
- **JWKS rotation.** Verifiers cache the issuer's public keys. When the IdP rotates, tokens signed
  with the new `kid` fail until caches refresh. Fetch the JWKS on unknown `kid` (with a rate limit so
  forged `kid`s cannot DoS the IdP), and publish the new key before signing with it.
- **Token size.** JWTs stuffed with roles and groups grow past header limits (8 KB) — 400/431 from
  nginx or the LB for your most privileged users only. Cookies accumulate the same way across
  subdomains.
- **Refresh stampede.** Many parallel requests notice an expired token at once and all refresh;
  refresh-token rotation then invalidates all but one, logging the user out. Single-flight the
  refresh per client and refresh ahead of expiry.
- **Revocation.** Stateless JWTs cannot be revoked before `exp`; keep them short (5–15 min) and use a
  denylist for emergencies.
- **Header stripping and trust.** Identity headers set by an auth proxy (`X-User-Id`) must be stripped
  from external input; otherwise anyone can send them. Same for mTLS identity headers (§8).

### 21. Errors, versioning, long-running work and governance

`A06`, `A07`, `A21`, `A22`.

- **Long-running operations.** Anything that can exceed the LB timeout (60 s) must be asynchronous:
  `202 Accepted` + `Location` of an operation resource + `Retry-After`, or a webhook on completion.
  Raising the LB timeout to 15 minutes turns every stuck request into a held connection and
  worker.
- **Bulk operations.** Report per-item results (`207 Multi-Status` or a results array); make the
  whole batch idempotent; cap batch size; process items independently so one poison item does not
  fail 9 999 others.
- **Deprecation in practice.** Measure before removing: log API version, endpoint, and a
  client-identifying header (`User-Agent`, SDK version, API key) per request, and find who still
  calls the old path. Announce with `Deprecation` and `Sunset` headers. **Brownouts** — deliberately
  failing the deprecated endpoint for short scheduled windows before the final removal (GitHub,
  Heroku and others do this) — find the consumers that ignored every email.
- **Versioning by date.** Stripe pins each account to the API version current at signup and
  transforms responses through a chain of version changes. Lets you evolve without breaking
  everyone; costs a permanent compatibility layer.
- **Contract testing.** Consumer-driven contracts (Pact) and schema diffing (`buf breaking`,
  OpenAPI diff in CI) catch breaking changes before deploy. They do not catch behavioural changes
  (ordering, defaults, timing) — Hyrum's law (§18) — which is why shadow traffic and canaries exist.
- **Shadow traffic.** Mirror a copy of production requests to the new implementation (nginx
  `mirror`, Envoy `request_mirror_policies`), discard its responses, diff them offline. Only safe for
  side-effect-free requests, or against a sandboxed backend.
- **Canary cohorts.** Header- or cookie-based canaries only sample clients that send the header
  (usually browsers) and miss server-side SDK consumers entirely. Canary by tenant or API-key cohort
  so the largest integrations are exercised before 100%.

### 22. Diagnostic toolkit

- **curl timing breakdown.**
  ```
  curl -o /dev/null -s -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} \
  ttfb=%{time_starttransfer} total=%{time_total} code=%{http_code} ver=%{http_version}\n' URL
  ```
  Each value is cumulative from the start. Large `dns` → §7; `tcp − dns` ≈ RTT; `tls − tcp` =
  handshake (§8); `ttfb − tls` = server time; `total − ttfb` = transfer/bandwidth.
- **curl targeting.** `--resolve host:443:10.0.0.5` to hit one backend with the correct SNI and Host;
  `--connect-to` to redirect a host:port; `--http1.1` / `--http2` / `--http3` to compare protocols;
  `-v` for headers and TLS; `--limit-rate` to simulate slow clients; `-H 'Expect:'` to disable
  100-continue.
- **openssl.** `openssl s_client -connect host:443 -servername host -showcerts` (chain, SNI, protocol);
  `| openssl x509 -noout -dates -subject -issuer` for expiry; `-status` for OCSP stapling;
  `-tls1_2`/`-tls1_3` to test versions.
- **DNS.** `dig +trace`, `dig @resolver name`, `dig +short`, `getent hosts` (§7).
- **Sockets.** `ss -s` (summary), `ss -tan state time-wait | wc -l`, `ss -tnp` (owner process),
  `ss -ti` (per-connection RTT, cwnd, retransmits); `nstat -az` for `ListenOverflows`,
  `TCPRetransSegs`; `conntrack -S`; `sysctl net.ipv4.ip_local_port_range`.
- **Packets.** `tcpdump -i any -nn 'host X and port 443' -w cap.pcap`, then Wireshark: look for RST,
  retransmits, zero windows, ICMP. Decrypt TLS by exporting `SSLKEYLOGFILE` from the client (curl,
  browsers, Node `--tls-keylog`), not by breaking TLS. `mtr` for per-hop loss; `ping -M do -s` and
  `tracepath` for MTU (§4).
- **Runtime debug switches.** `NODE_DEBUG=http,net,tls`; `GODEBUG=http2debug=2`; Java
  `-Djavax.net.debug=ssl:handshake`; gRPC `GRPC_VERBOSITY=debug GRPC_TRACE=http,api`; `grpcurl` for
  calling gRPC services by hand.
- **Logs to join.** LB access logs (timings, `elb_status_code` vs `target_status_code`), proxy error
  logs, app logs by request ID. A request ID generated at the edge and propagated everywhere is what
  lets you join them (`O05`).

### 23. Catalogue of unconventional techniques

The moves that distinguish a war-story answer:

- **Invert the timeout ordering** instead of "increasing timeouts" — make the server outlast the
  client's idle timeout (§1).
- **`preStop` sleep** before graceful shutdown, to outwait endpoint propagation (§2).
- **Send `Connection: close` / GOAWAY on purpose** to move clients off a node before shutdown, or
  periodically to rebalance (`MAX_CONNECTION_AGE`) (§5, §6).
- **Lower DNS TTLs days before** a migration, and cap connection lifetime so pools re-resolve (§7).
- **Retry exactly once on a reused-connection reset** for idempotent calls; never blanket retries (§1, §16).
- **Retry budgets and full jitter** over fixed retry counts (§16).
- **Hedged requests** for tail latency on idempotent reads (§16).
- **Adaptive concurrency limits and LIFO under overload** instead of static rate limits (§15).
- **Thin webhooks plus fetch-latest** instead of trusting event order (§11).
- **Verify signatures on raw bytes before parsing** (§11).
- **Brownouts** to flush out consumers of deprecated APIs (§21).
- **Shadow traffic and response diffing** before a protocol or implementation switch (§21).
- **Surrogate-key purges, `stale-if-error`, request collapsing** to survive origin trouble (§9).
- **MSS clamping / allowing ICMP type 3 code 4** for "only large requests hang" (§4).
- **String-encode 64-bit IDs** at the API boundary (§18).
- **Persisted queries** as the real GraphQL security control (§17).
- **Presigned direct-to-storage uploads** to take bulk bytes off the API tier (§19).
- **`SSLKEYLOGFILE` + tcpdump** to see inside TLS in your own environment (§22).
- **Hit one backend deterministically** with `curl --resolve` to separate "one bad node" from "all
  nodes" (§22).
- **Monitor from outside**: synthetic checks for certificate expiry on every SNI name, DNS
  resolution from multiple resolvers, and CORS headers on error responses (§8, §10).

---

## Questions

### Level 1 — Everyday HTTP failures: 502s, 504s, timeouts and retries

The questions almost every backend interview asks in some form. Expect them.

1. Our Node.js API sits behind an AWS ALB. We see a steady 0.05% of 502s, all with a target
   processing time of `-1`, and they are worse at 3 a.m. than at peak. The app logs show no errors
   at all. What is happening?

   > **Direction:** The idle-timeout race: Node's 5 s `keepAliveTimeout` is shorter than the ALB's 60 s,
   > so the ALB reuses connections Node is closing; set Node's keep-alive (and `headersTimeout`) above
   > 60 s. Lulls make it worse because more connections sit idle. §1, §2, `A01`, `A12`.

2. Every deploy of our Kubernetes service produces a burst of 502s lasting about ten seconds, even
   though we have readiness probes and a graceful shutdown handler. Why, and what would you change?

   > **Direction:** SIGTERM arrives in parallel with endpoint removal, so traffic is still routed to the
   > dying pod; add a `preStop` sleep, drain with `Connection: close`, and tune
   > `terminationGracePeriodSeconds` and `deregistration_delay`. §2, `A01`.

3. A report endpoint takes 70–90 seconds for big customers. Users get a 504 at exactly 60 seconds,
   but our logs show the request completing successfully a little later. The team proposes raising
   every timeout to 5 minutes. What do you think?

   > **Direction:** The 504 comes from the proxy's 60 s idle/read timeout, and the work completes for no
   > one; move to an async `202` + operation resource (or streaming), not bigger timeouts that hold
   > workers. §2, §21, `A21`.

4. Our Python service calls a partner API with `requests`. A few times an hour we get
   `ConnectionResetError` on the *first* call after a quiet period; retrying immediately always
   succeeds. The partner says they see nothing. Explain it and fix it properly.

   > **Direction:** The outbound pool reuses a connection the partner's server (short keep-alive) has
   > already closed; evict idle connections before the partner's timeout and retry idempotent calls
   > once on a reset of a reused socket. §1, `A12`.

5. After putting nginx in front of a Spring Boot app, some API clients report that a header called
   `X_TENANT_ID` "stopped arriving". Nothing else changed. What happened?

   > **Direction:** nginx drops request headers containing underscores by default
   > (`underscores_in_headers off`); enable it or rename to hyphens. §2, `A01`.

6. We moved our HTTP API to HTTPS behind a TLS-terminating load balancer. Now the login flow
   redirects users to `http://` URLs and some POSTs to `/api/orders` arrive as GETs with no body.
   What two problems are in play?

   > **Direction:** The app does not trust `X-Forwarded-Proto` so it builds `http://` redirects, and a
   > 301/302 redirect (HTTP→HTTPS or trailing slash) lets clients turn POST into GET; trust the
   > proxy header and use 307/308. §2, `A01`, `A02`.

7. A mobile client uploads a 3 MB profile photo. Our production API returns `413` to everyone, while
   staging accepts the same upload. The app code is identical. Where do you look, and what is the
   longer-term design?

   > **Direction:** A body-size limit in a layer only production has — nginx `client_max_body_size` 1 MB
   > default, a gateway or Lambda limit; each layer has its own, and uploads should go direct to
   > object storage with presigned URLs. §19, §2, `A20`.

8. Our monitoring shows 100% HTTP 200 for a payments service during an incident where customers could
   not pay. The service returns `{"success": false, "error": "..."}` on failures. What is the lesson
   and what would you change?

   > **Direction:** 200-with-error-body blinds status-code SLOs, LB health and retry policies; return real
   > status codes with `problem+json` and classify retryable ones. §16, `A07`.

9. A .NET client posting 5 KB JSON bodies to our Go API sees every request take just over one second.
   `curl` from the same box also takes one second, but `curl -H 'Expect:'` takes 20 ms. What is going
   on?

   > **Direction:** `Expect: 100-continue` — the client waits up to 1 s for a `100 Continue` that a proxy
   > in the path never sends; disable the header client-side or make the proxy answer it. §2, `A01`.

10. Our service calls three downstreams with a 30 s timeout on each. During a partial outage of one
    downstream, our pods ran out of threads and the whole API died, not just the endpoint using that
    downstream. Walk me through why and what you would change.

    > **Direction:** Long timeouts turn a slow dependency into thread exhaustion (Little's law);
    > shrink timeouts inward within a deadline budget, isolate pools per dependency, and propagate
    > cancellation. §15, §16, `M09`.

11. Customers behind a large corporate proxy complain that our API returns intermittent 400 errors for
    users with many roles. Our own tests always pass. What is the likely cause?

    > **Direction:** Large JWTs or accumulated cookies exceed a header-size limit (8 KB nginx line, 16 KB
    > Node, proxy limits) only for privileged users; shrink tokens (reference tokens, fewer claims)
    > rather than raising limits everywhere. §20, §19, `A18`.

12. A client library retries every failed request three times, our API gateway retries twice, and our
    service retries its database call three times. During a DB slowdown, DB load went up by a factor
    of about 18. Explain and redesign.

    > **Direction:** Retries multiply per layer (3×2×3); retry at one layer, use retry budgets and full
    > jitter, and don't retry non-retryable classes. §16, `M09`.

13. Users occasionally see a 502 from nginx with `upstream prematurely closed connection while reading
    response header` in the error log, always on requests after an idle period, never under load
    tests. The upstream is Gunicorn. What do you check first?

    > **Direction:** Gunicorn's 2 s keep-alive is shorter than nginx's upstream `keepalive_timeout`;
    > invert it (upstream longer than downstream) or disable upstream keep-alive. §1, §2, `A12`.

14. Our health check endpoint queries the database. During a 30-second DB failover, the ALB marked
    every target unhealthy and customers saw errors for several minutes after the DB was back. Why
    minutes, and how would you design health checks instead?

    > **Direction:** Dependency checks in health endpoints take the whole fleet out together and recovery
    > waits for healthy-threshold × interval; split liveness from readiness and keep shared
    > dependencies out of LB health. §2, `M07`.

15. We enabled POST retries in nginx with `proxy_next_upstream ... non_idempotent` to hide 502s during
    deploys. A week later finance reports a handful of duplicated orders. What happened, and what is
    the right approach?

    > **Direction:** Retrying a non-idempotent request can re-execute one whose first attempt did commit
    > before the connection died; require idempotency keys end-to-end instead of proxy retries. §2,
    > §14, `A09`, `A02`.

16. A long file-export endpoint returns 200 and streams CSV. Some customers get truncated files with no
    error; our logs say the request succeeded. What is happening, and how would you make failure
    visible?

    > **Direction:** Once streaming starts the status is fixed at 200, so mid-stream failures or proxy
    > timeouts truncate silently; add an in-band terminator/row count or trailer, a `Content-Length` or
    > checksum, or produce the file async to object storage. §19, §21, `A20`.

17. Our rate limiter keys on client IP from `X-Forwarded-For`. A security review says anyone can bypass
    it, and at the same time an enterprise customer complains they are being rate-limited as one
    user. How can both be true?

    > **Direction:** Taking the leftmost XFF entry is client-controlled (bypass), while a whole office
    > behind NAT shares one real IP (over-limiting); take the entry added by your trusted proxy and key
    > on API key or user instead. §2, §15, `A08`.

18. An internal service-to-service call has p50 of 3 ms but a p99 of almost exactly 1003 ms, and
    nothing in the application traces explains the extra second. What would you suspect?

    > **Direction:** A dropped SYN retransmitted after the 1 s initial RTO — often an overflowing accept
    > backlog or conntrack drops; check `ListenOverflows`, `conntrack -S`, and connection reuse. §3,
    > `A12`.

19. A frontend team says "our API calls are slow" but server-side latency is 20 ms. From their region,
    browser timings show 600 ms. How do you break down where the time goes, with one command?

    > **Direction:** `curl -w` with `time_namelookup/connect/appconnect/starttransfer` separates DNS, TCP,
    > TLS, server and transfer; the usual culprits are cold TLS handshakes and preflights over high
    > RTT. §22, §8, §10.

20. During a load test, our API returns 503s from Envoy with the response flag `UO`, while the pods are
    at 40% CPU. Engineers want to add more pods. Is that the right call?

    > **Direction:** `UO` is upstream overflow — Envoy's circuit-breaker limits (max connections/pending
    > requests) are the bottleneck, not pod capacity; read the author and flag of the error before
    > scaling. §2, §15, `M06`.

### Level 2 — Caching, CDNs and CORS surprises

Very common, especially for public APIs and web backends; the answers hinge on headers.

1. We put CloudFront in front of our API. After a deploy, user A sometimes sees user B's account page.
   We only cache `GET` requests. What do you look for?

   > **Direction:** The cache key omits what personalises the response (cookie/Authorization), or a
   > `Set-Cookie` response was cached; mark personalised responses `private`/`no-store` and audit the
   > key. §9, `A04`.

2. The CDN hit ratio for our product catalogue dropped from 92% to 3% overnight. Traffic and URLs are
   unchanged; the only deploy was a framework upgrade. What is your first hypothesis?

   > **Direction:** The upgrade started emitting `Set-Cookie` or `Vary: Cookie`/`User-Agent` on every
   > response, making responses uncacheable or unique; diff response headers before and after. §9,
   > `A04`.

3. Our SPA on `app.example.com` calls `api.example.com`. Users in one region report CORS errors for
   one endpoint, but only intermittently; the endpoint works when called from Postman. Where do you
   start?

   > **Direction:** A CORS error often masks a 5xx/502 whose response lacks CORS headers (proxy or
   > error handler skipping middleware); check the actual status in the network tab and add CORS
   > headers on error paths. §10, `A19`.

4. We serve two customer domains from the same API with credentials, reflecting an allowlisted
   `Origin` into `Access-Control-Allow-Origin`. After enabling the CDN, one customer's site gets CORS
   failures while the other works. Why?

   > **Direction:** Missing `Vary: Origin`, so the CDN caches the ACAO header for the first origin and
   > serves it to the other; add `Vary: Origin`. §9, §10, `A19`, `A04`.

5. Our web app's page load makes 40 API calls to `/api/items/{id}`, and the browser shows 40 OPTIONS
   requests preceding them on every page view, even with `Access-Control-Max-Age: 86400`. Why, and
   what would you change?

   > **Direction:** The preflight cache is per URL, so every distinct ID preflights, and browsers cap
   > `Max-Age` (Chrome 2 h); remove the trigger on hot reads, batch, or serve the API same-origin
   > under a path. §10, `A19`.

6. After we added JWT auth middleware, every browser request from our frontend fails with a CORS error,
   though the same requests with the token work in `curl`. The CORS config looks right.

   > **Direction:** The preflight OPTIONS carries no Authorization, so auth middleware ordered before
   > CORS returns 401 to it; let OPTIONS through to the CORS handler first. §10, `A19`, `A18`.

7. We fixed a pricing bug and deployed, but for about 24 hours some customers still saw wrong prices,
   even though our API sends no `Cache-Control` headers. How is that possible?

   > **Direction:** No caching headers doesn't mean not cached — CloudFront applies a default TTL (24 h)
   > and browsers apply heuristic freshness; send explicit `Cache-Control` and purge on deploy. §9,
   > `A04`.

8. A product page that briefly returned 404 during a deploy race kept returning 404 for 10 minutes
   after the product existed. The origin returns 200 when hit directly. Explain.

   > **Direction:** Negative caching of the 404 at the CDN; set explicit short TTLs for error responses
   > and purge affected keys after deploys. §9, `A04`.

9. We run six origin servers. Browser revalidation with `If-None-Match` almost never gets a `304`, so
   every page reload downloads the full payload. The content does not change. What is wrong?

   > **Direction:** ETags differ per server (inode/mtime-based) or are rewritten by gzip, so revalidation
   > on another node mismatches; generate content-hash ETags consistently across the fleet. §9, `A04`.

10. Our frontend reads `X-Total-Count` from paginated responses. It works locally (same origin) but in
    production the header is always `null` in JavaScript, although it is visible in DevTools.

    > **Direction:** Non-safelisted response headers are hidden cross-origin unless listed in
    > `Access-Control-Expose-Headers`. §10, `A19`, `A05`.

11. A popular item's cache entry expires every 60 seconds, and each time the origin database spikes to
    100% CPU for a few seconds. How would you smooth this without making data meaningfully staler?

    > **Direction:** A stampede on expiry; use request collapsing/origin shield, `stale-while-revalidate`
    > and TTL jitter so one request refreshes while others get slightly stale data. §9, `Q09`.

12. Our origin had a 20-minute outage. Static pages stayed up via the CDN, but API GETs that were
    cacheable for 30 seconds all failed. The team asks whether the CDN could have protected us.

    > **Direction:** `stale-if-error` (or the CDN's serve-stale-on-origin-error setting) lets the edge
    > keep serving expired entries through origin failure. §9, §23, `A04`.

13. A security researcher shows that sending a request with `X-Forwarded-Host: evil.com` to our
    homepage made the CDN serve pages with script tags pointing to `evil.com` to everyone for an
    hour. How did that happen and how do you prevent it?

    > **Direction:** Cache poisoning through an unkeyed header used to build absolute URLs; strip or
    > ignore forwarding headers from untrusted clients at the edge, or include them in the cache key.
    > §9, §2, `A04`, `S08`.

14. Someone discovered that `https://example.com/account/settings/logo.png` returns the logged-in
    user's settings page, and the CDN caches it. Why is this dangerous, and whose bug is it?

    > **Direction:** Web cache deception: the app ignores the suffix, the CDN caches by extension, so an
    > attacker lures a victim and reads their page from cache; fix both — strict routing and caching
    > only on explicit `Cache-Control`. §9, `A04`.

15. Our CDN cache hit ratio is 40% on URLs that should all be identical, and logs show the same path
    with dozens of different query strings. What would you do?

    > **Direction:** Tracking parameters and unordered query strings fragment the cache key; normalise
    > (sort, strip `utm_*`/`fbclid`) at the edge. §9, `A04`.

16. We purge product pages by URL on every update. With 2 million products and category pages that
    embed products, purges take hours and categories show stale prices. What is the better model?

    > **Direction:** Tag responses with surrogate keys (product IDs) and purge by tag, accepting that
    > purges are eventually consistent across POPs. §9, `A04`, `SD05`.

17. An authenticated API response has `Cache-Control: max-age=300`. We expected the CDN to cache it,
    but it never does; the browser does. Why?

    > **Direction:** Shared caches won't store responses to requests with `Authorization` unless marked
    > `public` or `s-maxage` — and you should think hard before doing that. §9, `A04`, `A18`.

18. Our API supports JSON and CSV via `Accept`. After putting a cache in front, browsers that requested
    CSV sometimes got JSON. What's missing?

    > **Direction:** `Vary: Accept` (or separate URLs per format), so content-negotiated variants have
    > distinct cache entries. §9, `A04`.

19. Chrome started sending new CORS preflights with an `Access-Control-Request-Private-Network` header
    when our public SaaS page calls a small agent running on the user's `localhost`. Nothing changed in
    our code. What is going on?

    > **Direction:** Chrome's Private Network Access rules add preflights for public-to-private requests;
    > the local agent must answer them explicitly. §10, `A19`.

20. After enabling compression on our CDN, a partner integration that downloads 2 GB files with
    resumable range requests started failing and restarting from zero.

    > **Direction:** On-the-fly compression drops `Content-Length` and changes the representation, breaking
    > strong-ETag-based `Range`/`206`; don't compress already-compressed or range-served files. §9,
    > §19, `A20`, `A04`.

### Level 3 — Contracts, payloads and schema evolution breakages

Common in API-platform and product-team interviews: "we changed nothing breaking, and it broke".

1. Our JavaScript frontend started showing the wrong order details for some customers after we
   switched order IDs to Snowflake-style 64-bit integers. The backend returns the correct JSON, and
   the bug never reproduces in our Java integration tests.

   > **Direction:** IDs above 2^53 − 1 are rounded by `JSON.parse`, so distinct IDs collide in JS; emit
   > 64-bit IDs as strings (as protobuf's JSON mapping does). §18, `A17`.

2. We added a new optional field `loyaltyTier` to our customer response. Within an hour, a partner's
   Java integration failed on every call with a deserialisation exception. Adding a field is
   backwards compatible — so whose fault is it and what do we do now?

   > **Direction:** Jackson's `FAIL_ON_UNKNOWN_PROPERTIES` defaults to true outside Spring Boot, so strict
   > clients break on additive changes; roll back or gate the field per client, then push tolerant
   > readers and publish the additive-change policy. §18, §21, `A06`, `A17`.

3. We added a new value `REFUNDED_PARTIAL` to the `status` enum. The Android app crashes on orders
   with that status; iOS shows "unknown". Nothing else changed. How should enums evolve in a public
   API?

   > **Direction:** Enum additions break exhaustive clients; document enums as open, require an
   > unknown/default branch, and gate new values by client version or API version. §18, `A06`, `A17`.

4. A PATCH to set a product discount to `0` via our gRPC-backed API is silently ignored, while setting
   it to `5` works. The code "looks right".

   > **Direction:** proto3 scalars have no presence, so `0` is indistinguishable from unset; use
   > `optional`/wrapper types or a field mask for PATCH semantics. §18, `A14`, `A17`.

5. Someone deleted a deprecated protobuf field `legacy_code = 7` and a month later added
   `region_id = 7` as an `int32`. Old services now report nonsense region IDs for some records.
   Explain the mechanism and prevention.

   > **Direction:** Wire compatibility is by field number; reusing 7 makes old writers' data decode as
   > the new field; mark deleted numbers `reserved` and run `buf breaking` in CI. §18, §21, `A14`, `A22`.

6. Our API returns prices as JSON numbers like `19.99`. A finance partner reports totals off by a cent
   on roughly one invoice in a thousand. Our database uses `NUMERIC`. Where is the cent lost?

   > **Direction:** JSON numbers are parsed as doubles by many clients and rounding accumulates; send
   > money as integer minor units or decimal strings. §18, `A17`.

7. After upgrading a JSON library, all timestamps in our API responses lost their trailing `Z`. A
   week later a customer in Sydney says every appointment shows 10 hours off. What happened?

   > **Direction:** Timestamps without an offset are parsed as local time by clients; always emit
   > ISO-8601 with explicit offset, and contract-test the format. §18, §21, `A17`, `A22`.

8. We changed the default sort of `GET /transactions` from `created_at` to `id` because they are
   "basically the same". A partner's reconciliation job now reports missing transactions daily. The
   OpenAPI spec did not change.

   > **Direction:** Hyrum's law — observable behaviour like default order is part of the contract even
   > when unspecified; schema diffs miss it, so use shadow traffic/response diffing and version
   > behavioural changes. §18, §21, `A06`, `A22`.

9. We want to remove API v1, which the dashboard says gets "under 1% of traffic". Last time we removed
   an old endpoint, our largest customer's integration broke and we had to roll back. How would you
   run this removal?

   > **Direction:** Identify consumers by API key/SDK/User-Agent, announce with `Deprecation`/`Sunset`
   > headers, and run scheduled brownouts before removal to flush out silent consumers. §21, §23,
   > `A06`, `A22`.

10. Our `PUT /users/{id}` accepts partial JSON. A mobile app built in Go sends `"nickname": ""` when
    the user clears their nickname, but a web client that omits the field also clears it. Users
    complain their nicknames vanish. What is the design error?

    > **Direction:** Absent vs null vs empty are conflated (Go `omitempty`, PUT used as PATCH); use
    > JSON Merge Patch or field masks and define explicit semantics for null. §18, `A03`, `A17`.

11. We return errors as `{"message": "Card declined: insufficient funds"}`. A product manager
    reworded the message, and a large customer's checkout started treating declines as system
    errors. How do you design error contracts to avoid this?

    > **Direction:** Clients parse whatever is observable; give errors stable machine codes
    > (`problem+json` `type`, a `code` field) and treat message text as human-only. §16, §18, `A07`.

12. Our public SDK auto-retries on any 5xx. We started returning `500` for a validation bug in one
    endpoint, and customer SDKs hammered us with retries of requests that can never succeed. What
    should the error-code taxonomy look like?

    > **Direction:** Retryability must be encoded in status classes (4xx never, 503/429 with
    > `Retry-After` yes), and a validation failure should be a 4xx; SDKs retry only retryable classes
    > with budgets. §16, `A07`, `M09`.

13. A client sends gzip-compressed request bodies, which our server decompresses in memory. One day a
    single 2 MB request took a pod down with an OOM. What happened and what is the guard?

    > **Direction:** A decompression bomb — the wire size was checked, not the inflated size; enforce a
    > limit on decompressed bytes while streaming. §19, `A20`, `S08`.

14. An export endpoint builds a JSON array of up to 2 million rows in memory. Large tenants cause pods
    to OOM, and raising memory only moves the threshold. What would you change?

    > **Direction:** Stream (NDJSON or chunked JSON from a DB cursor) with backpressure, or produce the
    > file asynchronously to object storage and return a link. §19, §21, `A20`, `A21`.

15. Our API signs requests with HMAC over the canonical query string. After we put a new API gateway
    in front, about 3% of signed requests fail verification — always ones with spaces or slashes in
    parameters.

    > **Direction:** The gateway re-encodes URLs (`%20`↔`+`, decoding `%2F`) or reorders parameters, so
    > the canonical form differs; verify on the raw form or make the proxy pass the URL untouched. §20,
    > §2, `A18`.

16. A contract test suite (Pact) is green for every consumer, yet a deploy broke the mobile app because
    it depended on an `X-Request-Id` response header we stopped sending. Why didn't contracts catch it,
    and what would?

    > **Direction:** Contracts only cover what consumers declared; undeclared dependencies (headers,
    > order, timing) need traffic-based checks — shadow traffic, diffing, canaries by client version.
    > §21, §18, `A22`.

17. We started gzip-compressing all JSON responses to save bandwidth. p50 latency improved for mobile
    clients, but CPU on the API fleet rose 35% and p99 got worse for small internal calls. What would
    you tune?

    > **Direction:** Skip compression below ~1 KB and on internal hops, and use moderate levels (not
    > brotli 11) for dynamic responses. §19, `A20`, `C19`.

18. Our app returns a CSRF token in HTML that also reflects a search query, and everything is gzipped
    over HTTPS. A pentest report flags a compression side channel. Explain the attack and what you
    would change.

    > **Direction:** BREACH: compressed length leaks secrets when attacker-controlled input sits near
    > them; mask tokens per response, separate secrets from reflected input, or disable compression on
    > those responses. §19, `A20`, `S08`.

19. A bulk endpoint accepts 10 000 items. When one item is invalid we return 400 for the whole batch,
    so customers split batches manually and retry, sometimes creating duplicates. How would you
    redesign it?

    > **Direction:** Per-item results (207 or a results array), independent item processing, per-item
    > or batch idempotency keys, and a capped batch size. §21, §14, `A21`, `A09`.

20. Our date-versioned API (like Stripe's) has 40 version transforms. A bug fix in the newest version
    must not change behaviour for accounts pinned to older versions, but one fix leaked into all
    versions and broke integrations. How do you structure versioned behaviour?

    > **Direction:** Keep a single current implementation and express each version as a
    > request/response transform chain, with tests per pinned version; fixes that change observable
    > output need their own version gate. §21, `A06`, `A22`.

### Level 4 — Rate limiting, idempotency and retries under load

The "design it so it survives the thundering herd" family, asked at most senior interviews.

1. Our rate limiter allows 100 requests per minute per API key using a fixed window in Redis. One
   customer complains they were throttled in the first seconds of a minute, and an abuser pushed 200
   requests in two seconds without being throttled. How can both happen?

   > **Direction:** Fixed windows allow 2× across a boundary and treat bursts arbitrarily depending on
   > where they fall; use a token bucket/GCRA or a sliding window counter. §15, `A08`.

2. We implemented rate limiting in each pod's memory: 50 req/s per customer. After a traffic spike,
   autoscaling took us from 4 to 24 pods and a customer's scraper got 1200 req/s through. What went
   wrong, and what are the options?

   > **Direction:** Per-instance limits multiply by replica count; centralise (Redis/GCRA), divide by
   > replicas, or combine local buckets with periodic global sync. §15, `A08`.

3. Some users are permanently blocked by our rate limiter, with Redis showing their counter keys with
   no TTL. It started after a Redis failover.

   > **Direction:** Non-atomic `INCR` then `EXPIRE` — a failure between them leaves a key without TTL;
   > do both atomically (Lua script, `SET NX EX` then `INCR`). §15, `A08`.

4. A client retries a `POST /payments` after a timeout. Both requests carry the same idempotency key,
   and they arrive 50 ms apart on different pods. The customer was charged twice. Our code checks
   "does the key exist?" before processing. What is the bug?

   > **Direction:** Check-then-act race; insert the key row first under a unique constraint with an
   > `in_progress` state and return 409 to the concurrent duplicate. §14, `A09`.

5. Our idempotency keys live in Redis and orders in Postgres. After a pod crash, a customer retried
   and got "already processed" but there is no order. How did that happen and what is the fix?

   > **Direction:** Key and effect were written to different stores non-atomically; record the key and
   > the business write in the same DB transaction. §14, `A09`.

6. A mobile app sometimes creates duplicate orders even though it sends an `Idempotency-Key`. The
   server logic is correct. What would you look at in the client?

   > **Direction:** The key is generated per attempt (inside the retry loop) or regenerated after app
   > restart; generate once per logical operation and persist it with the pending action. §14, `A09`.

7. A customer reused an idempotency key by accident for two different payments of different amounts.
   Our API returned the first payment's result for the second request, and their books are wrong.
   What should the server have done?

   > **Direction:** Fingerprint the request with the key and reject a mismatch (422/409) instead of
   > replaying a result for different input. §14, `A09`.

8. The first attempt of an idempotent request failed with a 500 because the database was down. The
   client retried with the same key after recovery and got the cached 500 forever. What is the rule?

   > **Direction:** Cache 2xx and deterministic 4xx, never transient 5xx/timeouts; release the
   > in-progress lock so retries become a fresh attempt. §14, `A09`.

9. Our API calls a payment provider inside the request. We have idempotency on our side, but after a
   crash between the provider's charge and our commit, a retry charged the customer again. How do
   you make this safe?

   > **Direction:** Pass a deterministic idempotency key downstream to the provider and structure the
   > operation as resumable recovery points, so a retry resumes rather than re-charges. §14, `A09`,
   > `M09`.

10. Every night at 00:00 UTC our API gets 20× normal traffic for about 90 seconds from customers' cron
    jobs, and we return a wave of 429s, which makes it worse. How would you handle this from both
    sides?

    > **Direction:** Synchronised clients plus 429 without `Retry-After` produce immediate retry storms;
    > send `Retry-After`/`RateLimit` headers, jitter in SDKs, and offer bursts via token buckets or
    > async bulk endpoints. §15, §16, `A08`.

11. We rate limit at 1000 req/s per tenant, but during a slowdown of our database the service still
    collapsed at 300 req/s. Why didn't the rate limiter protect us, and what would?

    > **Direction:** Rate limits bound arrivals, not in-flight work; when latency rises, concurrency
    > rises (Little's law). Adaptive concurrency limits and early load shedding protect the server.
    > §15, `A08`, `SD11`.

12. Our heaviest tenant has a single rate-limit key in Redis receiving 40 000 ops per second, and that
    Redis shard is at 100% CPU while the others idle. How would you redesign?

    > **Direction:** Hot key; use local token buckets with periodic sync, shard the counter across
    > sub-keys and sum, or GCRA with a single timestamp to reduce operations. §15, `A08`.

13. Our endpoints vary wildly in cost: a search can cost 1000× a simple GET. Customers stay under the
    request rate limit but a few of them saturate the search cluster. What would you change?

    > **Direction:** Cost-based (weighted) rate limiting, or separate buckets per endpoint class, rather
    > than counting requests equally. §15, `A08`.

14. During an overload, our queue of pending requests grew to 30 seconds deep. Every request we served
    was already abandoned by its client, so goodput dropped to zero even though we were at full CPU.
    What is the counter-intuitive fix?

    > **Direction:** Bound queues, drop work whose deadline has passed, serve LIFO under overload so
    > fresh requests succeed, and propagate client cancellation. §15, §16, `SD11`.

15. We added hedged requests to our search client to cut p99 latency. p99 improved by 40%, but a week
    later, during an incident, the search cluster's load doubled and it fell over. What went wrong?

    > **Direction:** Unbounded hedging multiplies load exactly when the backend is slow; hedge only after
    > p95, cap hedges with a budget, and disable them under backend distress. §16, §6, `M09`.

16. Our retry policy uses exponential backoff with a 100 ms base and no jitter. After a 10-second
    dependency blip, we see synchronised spikes at 100 ms, 200 ms, 400 ms and so on, hammering the
    recovering service. Why, and what fixes it?

    > **Direction:** Without jitter all clients retry in lockstep waves; use full jitter (random 0..cap)
    > and a retry budget. §16, `M09`.

17. A partner's users reach our API through a mobile carrier's network and share a small pool of IP
    addresses. Our per-IP DDoS protection keeps blocking them. How would you design limits that work
    for them and still stop abuse?

    > **Direction:** CGNAT means many users per IP; key limits on authenticated identity (API key, user,
    > device) and keep IP limits only as a coarse, generous layer. §15, §2, `A08`.

18. Our idempotency keys expire after 24 hours. After a two-day outage of a customer's queue, it
    redelivered all its messages and we created duplicate shipments. The key logic worked as
    designed. What is the missing backstop?

    > **Direction:** Idempotency windows are finite; add business-level uniqueness (one shipment per
    > order line) as a database constraint. §14, `A09`.

19. Our API gateway returns 429 with no body or headers. The customer's SDK treats unknown 4xx as fatal,
    so jobs fail permanently during short spikes. What should the contract look like?

    > **Direction:** 429 must carry `Retry-After` and machine-readable limit headers (`RateLimit`,
    > `RateLimit-Policy`), and SDKs must treat 429 as retryable after the delay. §15, §16, `A07`, `A08`.

20. We run the rate limiter in a central Redis in one region. When cross-region latency spiked to
    200 ms, every API call in the other region got 200 ms slower. When Redis went down briefly, we
    failed closed and rejected all traffic. What design would you propose?

    > **Direction:** Keep the hot path local (in-process buckets with async sync), fail open to local
    > limits when the global store is unavailable, and accept bounded overshoot. §15, `A08`, `SD11`.

### Level 5 — Connections and TCP in the wild

Asked when the interviewer wants to know whether you have ever looked below HTTP.

1. A batch job that calls an internal HTTP API 2000 times per second starts failing after about a
   minute with `connect EADDRNOTAVAIL` / "Cannot assign requested address". The API is healthy. What
   is going on?

   > **Direction:** Ephemeral port exhaustion from new connections per request, each leaving a local
   > port in TIME_WAIT for 60 s (~28 k ports ⇒ ~470 conn/s per destination); reuse connections first,
   > then widen the port range or enable `tcp_tw_reuse`. §3, `A12`.

2. `ss -s` on our API server shows 40 000 sockets in TIME_WAIT and an engineer wants to enable
   `tcp_tw_recycle` and lower the FIN timeout. What do you say?

   > **Direction:** TIME_WAIT on a server is mostly harmless; `tcp_tw_recycle` broke NAT'd clients and
   > no longer exists; `tcp_fin_timeout` does not control TIME_WAIT. Find who closes first and fix reuse
   > instead. §3, `A12`.

3. Our services in a private subnet call a single SaaS API heavily. At peak, a fraction of outbound
   connections fail to establish, and the NAT Gateway metrics show `ErrorPortAllocation`. What is the
   constraint and what are the fixes?

   > **Direction:** NAT Gateway allows about 55 000 simultaneous connections per unique destination;
   > reuse connections, add NAT IPs/gateways, or use PrivateLink/VPC endpoints to avoid NAT. §3, `A12`,
   > `O02`.

4. Our reporting service holds a pool of database connections through a NAT. Every morning the first
   queries after the overnight lull hang for 15 minutes and then fail; after that everything is fine.

   > **Direction:** The NAT/firewall idle timeout (350 s on AWS NAT) silently dropped idle flows, and
   > TCP keepalive (7200 s) is too long to notice; set keepalives below the idle timeout and a max
   > idle/lifetime on the pool. §3, `A12`.

5. A custom binary protocol client sends a 20-byte header and then the body in two `write()` calls.
   Each request takes exactly 40 ms more than expected on Linux, and 200 ms on Windows servers.

   > **Direction:** Nagle's algorithm interacting with delayed ACK; set `TCP_NODELAY` or write the
   > message in a single buffer. §3, `A12`.

6. Under a traffic burst our service's p99 shows spikes at almost exactly +1 s and +3 s. Latency
   inside the app is flat. What do you check on the host?

   > **Direction:** A full accept queue dropping SYNs, which clients retransmit at 1 s then 3 s; check
   > `ListenOverflows` and raise `somaxconn`/backlog or add capacity. §3, `A12`.

7. Kubernetes nodes running a busy ingress start timing out new connections under load, and `dmesg`
   says `nf_conntrack: table full, dropping packet`. What happened and how do you fix it without
   waiting for the next incident?

   > **Direction:** The conntrack table is exhausted by many short-lived flows; raise `nf_conntrack_max`,
   > reduce flow churn with keep-alive, shorten timeouts for closed states, and alert on conntrack
   > usage. §3, `A12`, `O02`.

8. A client says our API "sometimes returns an empty response" — they receive a connection reset right
   after we send an error for an oversized upload. We definitely write the 413 before closing. Why do
   they not see it?

   > **Direction:** Closing a socket with unread request data in the receive buffer sends RST, which can
   > discard the 413 before the client reads it; drain (or partially drain) the body, or use lingering
   > close. §3, §19, `A12`.

9. Cross-region replication of large files over HTTPS between two of our data centres (100 ms RTT, 10
   Gbit/s links) never exceeds about 50 Mbit/s per connection. The network team says the pipe is
   fine.

   > **Direction:** Throughput is capped at window/RTT (bandwidth-delay product); check window scaling
   > and buffer sizes (and HTTP/2 flow-control windows if used), or use parallel connections. §3, §5,
   > `A12`, `A10`.

10. After we moved a service to a new VPN-connected data centre, health checks and small API calls
    work, but requests with larger JSON bodies hang until the client times out. Small payloads never
    fail.

    > **Direction:** A PMTUD black hole — the tunnel lowers MTU and ICMP "fragmentation needed" is
    > blocked, so full-size packets vanish; allow ICMP type 3 code 4, clamp MSS, or lower MTU. §4,
    > `A12`.

11. We see TLS handshakes to a partner hang right after the ClientHello, but only from our new
    Kubernetes cluster that uses a VXLAN overlay. From the old VMs it works.

    > **Direction:** The overlay reduces MTU; the ServerHello + certificate chain is the first large
    > packet and is black-holed; confirm with `ping -M do -s` and fix MTU/MSS clamping. §4, §8, `A12`,
    > `A11`.

12. Our Node.js service creates a new `https.Agent` per request "to avoid shared state". CPU is high,
    latency to a partner is 250 ms, and the partner has asked us to stop opening so many connections.

    > **Direction:** No connection reuse means a TCP + TLS handshake per request (CPU and RTTs) plus
    > TIME_WAIT churn; share one keep-alive agent per destination with sensible pool limits. §1, §3,
    > §8, `A12`, `A11`.

13. We raised the connection pool size in our HTTP client from 50 to 500 to fix latency to a
    downstream. Latency got worse and the downstream started returning 503s. Why?

    > **Direction:** A bigger pool only allows more concurrency against a saturated backend (Little's
    > law) and more connection overhead; find the bottleneck and bound concurrency rather than raise it.
    > §15, §3, `A12`.

14. A service behind an AWS NLB (TCP listener) sees clients' long-running idle connections die silently
    after about six minutes; the next request on them hangs and gets a RST.

    > **Direction:** The NLB's TCP idle timeout (350 s by default) drops idle flows; use TCP keepalive or
    > application pings under that interval, or raise the NLB idle timeout. §3, `A12`.

15. `tcpdump` on a slow API shows many retransmitted segments and duplicate ACKs, but only for one
    availability zone. p50 latency is fine, p99.9 is terrible. What does it tell you and what do you
    do next?

    > **Direction:** Packet loss on one path — invisible on averages, dominant at the tail through RTO
    > waits; confirm with `mtr`/`ss -ti` retransmit counters per AZ and route around or escalate. §3,
    > §22, `A12`.

16. A graceful-shutdown handler closes the listening socket, waits for in-flight requests, then exits.
    Clients with keep-alive connections still get resets during deploys. What is missing?

    > **Direction:** Idle keep-alive connections are not "in-flight"; the server must signal close
    > (`Connection: close` on responses, HTTP/2 GOAWAY) and close idle sockets after the LB has drained.
    > §1, §2, §5, `A12`.

17. We run a service mesh sidecar (Envoy) next to every app. After enabling it, the app's own
    connection-pool metrics look perfect but p99 to downstream services got worse and we see resets
    on long idle periods. Where are the hidden connections?

    > **Direction:** There are now two hops of pooling (app→sidecar, sidecar→remote) with their own idle
    > timeouts; the keep-alive ordering rule must hold on every hop. §1, §2, `A12`, `M07`.

18. Our Java service shows thousands of connections in `CLOSE_WAIT` and eventually runs out of file
    descriptors. What does `CLOSE_WAIT` tell you about whose bug it is?

    > **Direction:** `CLOSE_WAIT` means the peer closed but our process never closed its side — a leak
    > in our code (unconsumed response bodies, unreleased pooled connections). §3, §22, `A12`.

19. Traffic from one large enterprise customer arrives from 4 egress IPs. Our ALB balances well, but
    our self-hosted HAProxy with `balance source` sends almost all of it to two backends.

    > **Direction:** Source-IP hashing collapses NAT'd customers onto few backends; balance by
    > connection or request (leastconn/round-robin) and keep stickiness at the application layer if
    > needed. §3, §2, `A12`, `M07`.

20. A downstream team says our client "opens too many connections", and we say we reuse them. `ss
    -tnp` shows each of our 8 worker processes has its own pool of 100. How do you reconcile and size
    it?

    > **Direction:** Pools are per process (and per pod), so total connections = workers × pods × pool;
    > size from required concurrency via Little's law divided across instances. §15, §3, `A12`.

### Level 6 — DNS, TLS and certificates

Frequently asked, rarely answered well; the book answers here are often wrong in practice.

1. We failed over our database by updating a DNS CNAME with a 60-second TTL. Half our Java services
   followed within a minute; the other half kept hitting the old primary for hours until restarted.

   > **Direction:** JVM DNS caching (forever with a security manager, or custom `networkaddress.cache.ttl`)
   > and pooled connections that never re-resolve; set the JVM TTL, cap connection lifetime, and
   > restart or drain on failover. §7, `A13`.

2. Our Node.js API's latency spikes whenever the internal DNS server is slow — including endpoints that
   only read files from disk and never touch the network. How can DNS affect file reads?

   > **Direction:** `dns.lookup` uses the libuv threadpool (4 threads by default) shared with `fs` and
   > `crypto`, and Node doesn't cache DNS; add a caching lookup, keep-alive agents, and a larger
   > threadpool. §7, `A13`, `C04`.

3. In Kubernetes, calls from our pods to `api.partner.com` sometimes take exactly 5 seconds longer
   than usual. It happens to a small fraction of requests on busy nodes only.

   > **Direction:** The conntrack race on parallel A/AAAA UDP queries drops one and glibc waits its 5 s
   > timeout; use `single-request-reopen`, NodeLocal DNSCache, or TCP for DNS. §7, `A13`.

4. CoreDNS in our cluster uses a lot of CPU, and query logs show `api.partner.com.default.svc.cluster.local`,
   `api.partner.com.svc.cluster.local` and similar names failing constantly. What is going on and
   what are the fixes?

   > **Direction:** `ndots:5` plus search domains turns each external lookup into up to ten queries; use
   > fully qualified names with a trailing dot or lower `ndots` in `dnsConfig`. §7, `A13`.

5. We migrated a public API to a new provider. We changed the DNS record and set the TTL to 60 s at
   the same time. Some users hit the old provider for two days.

   > **Direction:** Caches honour the old TTL (e.g. 48 h) until it expires, so lower the TTL days before
   > the migration; also some resolvers and clients cap or ignore TTLs. §7, §23, `A13`.

6. Our nginx proxies to an AWS ELB by hostname. After AWS rotated the ELB's IPs, nginx sent traffic to
   dead IPs and returned 502 until we reloaded it.

   > **Direction:** nginx resolves `proxy_pass` hostnames once at startup; use a `resolver` directive
   > with a variable in `proxy_pass` (or an upstream with dynamic resolution). §7, §2, `A13`.

7. A new microservice was deployed, but for 30 minutes other services got `NXDOMAIN` for its name even
   though the record existed after the first minute. A health checker had been polling the name before
   the deploy.

   > **Direction:** Negative caching — the early NXDOMAIN was cached for the zone's negative TTL; avoid
   > querying names before they exist and keep the SOA negative TTL low. §7, `A13`.

8. On busy EC2 hosts, a small percentage of DNS lookups time out, and the VPC resolver shows no errors.
   The service does thousands of uncached lookups a second.

   > **Direction:** The VPC resolver drops above 1024 packets/s per ENI; cache locally (dnsmasq,
   > NodeLocal DNSCache, runtime caching) and reuse connections. §7, `A13`.

9. After we added an AAAA record, a subset of customers — all behind one ISP — report timeouts to our
   API, while everyone else is fine.

   > **Direction:** Clients prefer IPv6 and that ISP's v6 path is broken; runtimes without Happy Eyeballs
   > wait for the v6 connect timeout. Verify v6 reachability end to end before publishing AAAA. §7,
   > `A13`.

10. Our partner can call our API from `curl` on their laptop with no issues, but their Java service
    fails with `PKIX path building failed`. Chrome shows a valid padlock.

    > **Direction:** The server omits the intermediate certificate; browsers fetch or cache it via AIA,
    > Java/curl don't. Serve the full chain and check with `openssl s_client -showcerts`. §8, `A11`.

11. On 30 September 2021, some of our older Android clients and a couple of backend services running
    old OpenSSL suddenly failed TLS to our API, though our certificate was valid for another two
    months. What happened?

    > **Direction:** The Let's Encrypt DST Root CA X3 expiry and chain path-building issues in old
    > trust stores and OpenSSL 1.0.2; update trust stores or serve an alternate chain. §8, `A11`.

12. We renewed a certificate and deployed it immediately. For about ten minutes, some IoT devices
    rejected it as "not yet valid". A few devices keep rejecting it permanently.

    > **Direction:** Client clock skew versus `notBefore`; devices without good NTP/RTC are far off.
    > Backdated issuance helps; for devices, fix time sync or accept a longer overlap with the old
    > cert. §8, §20, `A11`.

13. Our certificate expired on a Saturday and took down the API, even though we had an expiry alert.
    The alert checked `api.example.com`, and the certificate that expired was on
    `api-eu.example.com`, served by the same load balancer.

    > **Direction:** Monitor every SNI name and every endpoint from outside, including intermediates,
    > and automate renewal — shrinking lifetimes (200 days from 2026) make this more frequent. §8,
    > §23, `A11`.

14. We migrated internal service calls from HTTP to HTTPS and CPU on the API fleet doubled. Security
    says "that's the cost of encryption". Is it?

    > **Direction:** Symmetric encryption is cheap; the cost is handshakes because clients don't reuse
    > connections or resume sessions. Enable keep-alive, session resumption and ECDSA certs. §8, §1,
    > `A11`.

15. A partner's legacy client connects to our API by IP address and gets a certificate for a totally
    different domain. Other clients are fine.

    > **Direction:** No SNI (connecting by IP or an old client) returns the default certificate; set a
    > sensible default cert or give the partner a dedicated endpoint. §8, `A11`.

16. Our mobile app pinned the leaf certificate's public key. The certificate was rotated with a new key
    and the app stopped working for every user who hadn't updated. How should pinning be done, if at
    all?

    > **Direction:** Pin a CA or backup public keys (multiple pins), never only the leaf, or avoid pinning
    > and rely on CT monitoring. §8, `A11`, `S15`.

17. We enabled OCSP must-staple on our certificate for security. A few weeks later, an outage at the
    CA's OCSP responder made our site fail for Firefox users.

    > **Direction:** Must-staple turns a stapling/OCSP outage into a hard failure; revocation is
    > soft-fail by default for good reason, and OCSP is being retired in favour of CRLs. §8, `A11`.

18. We turned on TLS 1.3 0-RTT on our CDN to speed up mobile API calls. A security review flags a risk
    of duplicate payments. Explain.

    > **Direction:** 0-RTT early data can be replayed by a network attacker; only allow it for
    > idempotent requests and reject writes with 425 Too Early. §8, `A11`, `A02`.

19. Behind our load balancer, `nginx` on each node terminates TLS itself. Mobile clients reconnecting
    after a few seconds of backgrounding pay a full handshake every time, even though session tickets
    are enabled.

    > **Direction:** Each node has its own ticket keys, so resumption only works if the client returns
    > to the same node; share and rotate ticket keys across the fleet, or terminate TLS at the LB. §8,
    > `A11`.

20. Our LB terminates mTLS and forwards the client certificate's subject to the app in
    `X-Client-Cert-Subject`. A pentest shows they can impersonate any partner through a second, public
    listener.

    > **Direction:** Identity headers must be stripped from inbound requests on every listener and only
    > set by the terminating hop; the app must trust them only from that hop. §8, §20, `A11`, `A18`.

### Level 7 — HTTP/2, gRPC and long-lived connections

Asked at companies running service meshes and gRPC; separates people who have operated it from people
who have read about it.

1. We scaled a gRPC service from 3 to 12 pods in Kubernetes, but the 9 new pods get almost no traffic
   and the original 3 are still at 90% CPU. The Service is a normal `ClusterIP`.

   > **Direction:** kube-proxy balances per connection and gRPC multiplexes on long-lived connections,
   > so clients stay pinned; use client-side round-robin via a headless Service, an L7 proxy, or
   > server `MAX_CONNECTION_AGE` to force periodic rebalancing. §6, `A14`, `M07`.

2. We set up client-side `round_robin` with a headless Service. After a rolling restart, all clients
   ended up connected to only the first two pods that came up. Why doesn't it fix itself?

   > **Direction:** gRPC only re-resolves DNS on connection failure or GOAWAY; with no failures the
   > address list stays stale. `MAX_CONNECTION_AGE` (with jitter) triggers re-resolution. §6, §7,
   > `A14`, `A13`.

3. Our gRPC clients log `GOAWAY received ... ENHANCE_YOUR_CALM, debug data: too_many_pings` and
   connections drop every few minutes. We recently added keepalive to survive NAT timeouts.

   > **Direction:** The client's keepalive interval is below the server's permitted minimum (default
   > 5 min); configure `permit_keepalive_time` on the server and the client interval together. §6,
   > §3, `A14`.

4. A gRPC call chain A → B → C returns `DEADLINE_EXCEEDED` from C, but C's own logs show it answering
   in 20 ms. Where did the time go?

   > **Direction:** Deadlines propagate; the budget was consumed upstream (in A or B, queueing or
   > retries) and C received an almost-expired deadline. Trace remaining budget per hop. §6, §16,
   > `A14`, `M09`.

5. A new tenant with a large catalogue gets `RESOURCE_EXHAUSTED: Received message larger than max`
   from our gRPC `ListProducts`, while all other tenants are fine. What is the fix — and what isn't?

   > **Direction:** The 4 MB default receive limit; raising it hides an unbounded response — paginate or
   > use server streaming, and raise the limit only deliberately. §6, §13, `A14`, `A05`.

6. We put gRPC services behind an HTTP proxy that "supports HTTP/2", but every call fails with an
   `INTERNAL` or `UNKNOWN` error even though the backend logs success.

   > **Direction:** The proxy doesn't forward HTTP/2 trailers (or downgrades to HTTP/1.1), and gRPC's
   > status lives in trailers; use a gRPC-aware proxy/target group. §6, §5, `A14`, `A10`.

7. After migrating our mobile API from HTTP/1.1 to HTTP/2, p50 improved but p99 on poor mobile
   networks got noticeably worse. Why can one connection be slower than six?

   > **Direction:** TCP head-of-line blocking: a single lost packet stalls every multiplexed stream on
   > the one connection; HTTP/3 fixes it (with UDP-blocking and CPU caveats). §5, `A10`.

8. A batch client fires 1000 concurrent requests at our HTTP/2 API through one connection. The server
   is idle, but client-measured latency is huge. What is limiting it?

   > **Direction:** `SETTINGS_MAX_CONCURRENT_STREAMS` (commonly 100–128) — the rest queue client-side;
   > the client must open more connections or bound its own concurrency. §5, `A10`.

9. Large file downloads over our HTTP/2 edge to Asia run at about 4 Mbit/s, while the same files over
   HTTP/1.1 run at 80 Mbit/s. Same servers, same network.

   > **Direction:** HTTP/2 flow-control windows (65 535 bytes default) cap throughput at window/RTT on
   > high-latency links; raise stream and connection windows or use dynamic BDP windows. §5, §3, `A10`.

10. A Go service behind our edge started failing for some clients with `PROTOCOL_ERROR` after we
    enabled HTTP/2 end to end. HTTP/1.1 works. The only change was the protocol.

    > **Direction:** The backend sends connection-specific headers (`Connection`, `Keep-Alive`,
    > `Transfer-Encoding`) or uppercase header names, which are illegal in HTTP/2; strip them. §5, `A10`.

11. During a gRPC server deploy, clients see a burst of `UNAVAILABLE` errors even though pods shut down
    gracefully. The server calls `GracefulStop()`.

    > **Direction:** GOAWAY plus in-flight streams interacting with endpoint propagation; add a preStop
    > delay so no new connections arrive, then graceful stop, and enable client retries for `UNAVAILABLE`
    > on idempotent methods. §6, §2, §5, `A14`.

12. Our gRPC client has a retry policy for `UNAVAILABLE`. During a backend outage, the backend's
    recovery was delayed by retry traffic from 2000 clients. What gRPC feature should have prevented
    that?

    > **Direction:** `retryThrottling` in the service config (token-bucket retry budget per client),
    > plus backoff with jitter. §6, §16, `A14`, `M09`.

13. We want browsers to call our gRPC services directly. The frontend team tried and "it doesn't work
    at all". What are the options and trade-offs?

    > **Direction:** Browsers can't control HTTP/2 frames or read trailers; use gRPC-Web or Connect via
    > a translating proxy (Envoy), or expose a REST/JSON gateway. §6, `A14`.

14. A gRPC channel to a downstream in another cloud goes idle overnight. The first calls each morning
    hang for the full deadline and then fail with `UNAVAILABLE`; then everything recovers.

    > **Direction:** A NAT/LB idle timeout killed the flow silently and the client doesn't know until its
    > writes time out; enable gRPC keepalive pings under the idle timeout (within the server's allowed
    > rate). §6, §3, `A14`, `A12`.

15. We use an AWS ALB with a gRPC target group. It balances well, but when we deploy, long-running
    server-streaming RPCs are all killed at once and clients reconnect in a spike.

    > **Direction:** Streams are pinned to a target; drain gradually (GOAWAY with jittered connection
    > age, staggered deploys), and make clients resume from a checkpoint with jittered reconnects. §6,
    > §12, `A14`, `A16`.

16. Our HTTP/2 edge was hit by a flood of streams opened and immediately cancelled. CPU went to 100%
    even though concurrent streams never exceeded the limit of 100.

    > **Direction:** Rapid Reset (CVE-2023-44487): cancelled streams don't count against the concurrency
    > limit; cap the reset rate per connection and close abusive connections, and patch the server.
    > §5, `A10`, `S13`.

17. After moving a chatty internal API from REST to gRPC, one caller that makes thousands of tiny calls
    got faster, but a caller that uploads 200 MB blobs got slower and memory usage spiked.

    > **Direction:** gRPC unary messages are fully buffered in memory and bounded by size limits; large
    > payloads need client streaming in chunks or direct object-storage transfer. §6, §19, `A14`, `A20`.

18. We turned on gRPC hedging for a read method to cut tail latency. Latency improved, but the
    downstream's metrics show some writes duplicated. How is that possible for a read?

    > **Direction:** The "read" method had side effects (audit record, cache warm, counter); hedging and
    > retries are only safe for truly idempotent methods — mark methods explicitly. §6, §16, `A14`, `A02`.

19. Our HTTP/3 rollout to mobile clients showed no improvement for enterprise users and higher server
    CPU overall. Leadership wants to know if HTTP/3 "failed".

    > **Direction:** Corporate networks often block UDP 443 so clients fall back to HTTP/2; QUIC costs
    > more CPU per byte (less offload). Measure by network type, not overall. §5, `A10`.

20. Behind an L4 NLB, our WebSocket servers stay unbalanced for days after scale-out: the new nodes
    have 5% of connections. CPU autoscaling keeps adding nodes that stay empty.

    > **Direction:** Long-lived connections don't rebalance on scale-out; use least-connections for new
    > connections and actively shed connections from hot nodes (with jittered client reconnect). §12,
    > §6, `A16`, `M07`.

### Level 8 — Webhooks, realtime and asynchronous APIs

Common where the product integrates with third parties or pushes events to clients.

1. We receive Stripe webhooks. Occasionally a subscription ends up in the wrong state: we process
   `customer.subscription.updated` before `customer.subscription.created`, or an old update overwrites
   a newer one. How do you make the consumer correct?

   > **Direction:** Webhooks are unordered and at-least-once; dedupe by event ID and treat events as
   > thin notifications — fetch current state from the API (or compare versions) before applying. §11,
   > `A16`.

2. Our webhook signature verification fails for about 1% of events, always ones containing non-ASCII
   characters or decimal numbers. The secret is correct.

   > **Direction:** The framework parses and re-serialises JSON before verification; compute the HMAC
   > over the raw request bytes, then parse. §11, `A16`, `A18`.

3. Our webhook handler does all the work synchronously and takes about 25 seconds for large events.
   We now see the same events processed three or four times.

   > **Direction:** The provider times out (~10–20 s) and retries; acknowledge fast after persisting the
   > raw event, process from a queue, and dedupe. §11, `A16`, `A09`.

4. Stripe webhooks to our endpoint started failing signature checks with "timestamp outside the
   tolerance zone" from just one of our pods.

   > **Direction:** Clock skew on that node versus the 300 s tolerance on signed timestamps; fix NTP on
   > the host and alert on clock drift. §11, §20, `A16`.

5. We send webhooks to customers. One customer's endpoint started taking 30 seconds to respond, and
   within an hour all customers' webhooks were delayed by hours. What is the design flaw?

   > **Direction:** A shared delivery queue has head-of-line blocking; queue and limit concurrency per
   > destination, with per-endpoint circuit breakers and auto-disable. §11, `A16`.

6. Customers configure a webhook URL in our dashboard. A pentester set it to
   `http://169.254.169.254/latest/meta-data/` and received our cloud credentials in the delivery logs.

   > **Direction:** SSRF: resolve and block private/link-local ranges, pin the resolved IP to defeat DNS
   > rebinding, don't follow redirects, and don't echo responses into logs visible to customers. §11,
   > `A16`, `S08`.

7. We rotated our webhook signing secret and all customers' verifications failed until they updated
   their config. How should rotation work?

   > **Direction:** Sign with both old and new secrets during an overlap window (multiple signatures in
   > the header) so receivers can switch at their own pace. §11, `A16`.

8. Our provider's webhooks stopped arriving after our security team enabled a WAF with bot protection.
   The provider's dashboard shows 403s with an HTML challenge page.

   > **Direction:** Bot protection challenges non-browser POSTs; exempt the webhook path and rely on
   > signature verification rather than IP allowlists or browser checks. §11, `A16`.

9. SSE notifications in our app arrive in bursts every 30 seconds instead of immediately, but only in
   production, which has nginx and gzip in front.

   > **Direction:** Proxy buffering and compression buffering; disable `proxy_buffering` (or send
   > `X-Accel-Buffering: no`) and don't compress `text/event-stream`. §12, `A16`, `A20`.

10. Users with more than six tabs of our dashboard open report that the seventh tab never loads data.
    Each tab opens an SSE stream.

    > **Direction:** The browser's six-connections-per-origin HTTP/1.1 limit is consumed by SSE streams;
    > serve over HTTP/2, share one stream across tabs (BroadcastChannel/SharedWorker), or use a
    > separate host. §12, §5, `A16`, `A10`.

11. WebSocket connections through our ALB drop every 60 seconds when a user is idle, and our chat UI
    shows "reconnecting" constantly.

    > **Direction:** The ALB idle timeout (60 s) closes quiet connections; send ping/heartbeat frames
    > more often than the smallest idle timeout on the path. §12, §1, `A16`.

12. After a deploy of our realtime tier, 300 000 WebSocket clients reconnected simultaneously and took
    down the auth service and the new pods. How do you deploy realtime servers safely?

    > **Direction:** Drain connections gradually in batches, randomised exponential reconnect on
    > clients, resume from `Last-Event-ID`/offset to avoid full refetch, and admission limits per pod.
    > §12, §23, `A16`.

13. We scaled our WebSocket chat servers to three nodes. Users now say messages "sometimes don't
    arrive": the sender and receiver are on different nodes.

    > **Direction:** Connection state is per node; add a pub/sub backplane (Redis, NATS) so messages fan
    > out to every node holding a recipient. §12, `A16`, `Q01`.

14. A report generation API returns `202 Accepted` with a job URL. Clients poll it every 100 ms and
    polling traffic is now 90% of our load.

    > **Direction:** Return `Retry-After` on the operation resource, enforce a minimum polling interval,
    > offer a completion webhook, and make status polls cheap (cached, conditional GETs). §21, §11,
    > `A21`, `A04`.

15. Our GitHub webhook integration missed events during a 20-minute outage and never received them
    later. The Stripe integration recovered fine from the same outage. Why the difference?

    > **Direction:** Providers differ: GitHub doesn't retry failed deliveries automatically while Stripe
    > retries for days; design reconciliation (replay via API or delivery log) rather than trusting
    > delivery. §11, `A16`.

16. We process webhooks from a payment provider and credit accounts. An attacker captured a legitimate,
    signed webhook and replayed it a day later; we credited the account twice.

    > **Direction:** Signature alone doesn't prevent replay; verify the signed timestamp within tolerance
    > and dedupe on event ID persistently. §11, §14, `A16`, `A09`.

17. Our long-poll endpoint holds requests for up to 55 seconds. It works in dev but in production a
    corporate customer's clients get errors after 30 seconds, and our servers run out of workers at
    moderate load.

    > **Direction:** Intermediary proxies have shorter timeouts than yours, and thread-per-request
    > servers can't hold idle waits cheaply; shorten the poll below common proxy limits and move to an
    > async server or SSE. §12, §2, `A16`, `C04`.

18. We send webhooks with retries for 72 hours. A customer's endpoint was misconfigured for a week and
    returned 500s; we built up 40 million pending retries, and delivery to healthy customers slowed.

    > **Direction:** Auto-disable endpoints after sustained failure with notification, cap retry backlog
    > per destination, and offer replay from a dashboard instead of infinite retries. §11, `A16`.

19. Our mobile app uses SSE through a CDN. Events work in some regions but not others, where the CDN
    seems to hold the response until it finishes.

    > **Direction:** Some CDN configurations buffer or cache whole responses; bypass caching/buffering
    > for `text/event-stream` paths, or connect realtime directly to origin. §12, §9, `A16`.

20. We publish events to customers via webhooks and they ask "how do we know we haven't missed any?"
    What would you add to the protocol?

    > **Direction:** Per-endpoint monotonic sequence numbers or a cursor, and an events API to list/replay
    > since a cursor, so consumers detect gaps and reconcile. §11, §13, `A16`, `A05`.

### Level 9 — GraphQL, pagination and data-heavy APIs at scale

Rarer, usually from companies with big public or partner APIs; answers need both API and database
depth.

1. Our GraphQL `orders { customer { name } }` query for a page of 100 orders issues 101 SQL queries.
   We added DataLoader and it got better, but then a bug report showed user A seeing user B's
   customer data in rare cases.

   > **Direction:** DataLoader fixes N+1 by batching, but its cache must be per request; a shared
   > (global or per-process) loader leaks data across users. §17, `A15`, `DB20`.

2. A single GraphQL request with 500 aliases of an expensive `search` field took our search cluster
   down. Our rate limiter counted it as one request.

   > **Direction:** Alias amplification bypasses request-count limits; enforce query cost analysis
   > (fields × list sizes), alias/batch limits and rate-limit by cost. §17, §15, `A15`, `A08`.

3. During an outage of our inventory service, our GraphQL API dashboards showed 100% success and p50
   latency improved. Customers saw empty product pages.

   > **Direction:** GraphQL returns 200 with `errors[]`, and failing fast looks "faster"; instrument by
   > operation name and error count, not HTTP status. §17, §16, `A15`, `A07`.

4. One nested field in our GraphQL product page failed, and the entire `product` object came back
   `null`, so the page showed "product not found".

   > **Direction:** Non-null error propagation nulls the nearest nullable parent; make risky fields
   > nullable so failures stay local. §17, `A15`.

5. We disabled introspection in production for our private GraphQL API, but a researcher reconstructed
   most of our schema anyway.

   > **Direction:** Field suggestions ("Did you mean ...?") leak names; disable suggestions too, and use
   > persisted queries so arbitrary operations are rejected. §17, `A15`.

6. We want CDN caching for our public GraphQL API's most popular queries, but every request is a POST
   to `/graphql`. What would you do?

   > **Direction:** Persisted/automatic persisted queries sent as GET with the hash, and compute
   > `Cache-Control` from the least cacheable field. §17, §9, `A15`, `A04`.

7. A partner's integration pages through `GET /events?offset=N&limit=100`. Page 5000 takes 8 seconds,
   and during their nightly sync our database CPU is at 90%.

   > **Direction:** Offset cost grows with depth; switch to keyset pagination with a composite index and
   > opaque cursors. §13, `A05`, `DB20`.

8. After switching to cursor pagination on `created_at`, a customer reports that an export of 1
   million rows is missing about 200 records. They were bulk imported with the same timestamp.

   > **Direction:** Keyset on a non-unique column drops/duplicates ties at page boundaries; add a unique
   > tiebreaker (`created_at, id`) to both ORDER BY and the cursor. §13, `A05`.

9. Our "changes since" sync endpoint pages by `updated_at > :cursor`. Partners occasionally never
   receive some updates, even though they poll every minute.

   > **Direction:** Commit order isn't timestamp order — a transaction with an earlier `updated_at` can
   > commit after the cursor passed it; read only up to now minus a safety lag or use a monotonic change
   > log/sequence. §13, `A05`, `DB20`.

10. Our list endpoint returns `total_count`. For large tenants the `COUNT(*)` takes 12 seconds while the
    page query takes 40 ms. Product says the count is "required".

    > **Direction:** Return `has_more` (fetch limit + 1), estimated or capped counts ("10 000+"), or a
    > separately cached count. §13, `A05`, `DB20`.

11. Customers can pass `?sort=` any field and `?limit=` any number. One customer's `limit=500000&sort=description`
    request took the primary database down.

    > **Direction:** Enforce a max page size and allowlist sortable fields backed by indexes; unbounded
    > sort/limit is a denial-of-service. §13, `A05`, `DB20`.

12. We changed the internal sort key of a paginated endpoint. Clients mid-way through pagination with
    old cursors started getting duplicate and missing items.

    > **Direction:** Cursors must be opaque and versioned (encode sort and filters), so old cursors are
    > either decoded correctly or rejected explicitly. §13, `A05`, `A06`.

13. A mobile client builds its own cursor by base64-encoding `{"id": 123}` and now scrapes other
    tenants' records by guessing IDs. Our endpoint trusted the cursor's contents.

    > **Direction:** Cursors are untrusted input; sign or encrypt them, bind them to the tenant and
    > filters, and still apply authorisation to every row. §13, `A05`, `A18`.

14. Our GraphQL gateway federates five subgraphs. A query that looks cheap fans out into 400 subgraph
    requests because of how entities are resolved. How would you find and fix that?

    > **Direction:** Entity resolution across subgraphs is an N+1 at the network layer; inspect query
    > plans, batch `_entities` calls, and apply cost limits based on the plan, not the query text. §17,
    > `A15`, `M04`.

15. An attacker sends an array of 1000 GraphQL operations in one HTTP request to brute-force login
    codes, and our per-request login rate limit never triggers.

    > **Direction:** Query batching multiplies operations per HTTP request; limit batch size, count
    > operations (or cost) in rate limiting, and rate-limit sensitive mutations per identity. §17, §15,
    > `A15`, `A08`.

16. A customer downloads a 50 GB dataset from our API via a single streamed response. Transfers fail
    at random points after an hour and must restart from the beginning.

    > **Direction:** Support resumable downloads with `Range`/`206` and strong ETags, or split into
    > signed object-storage parts; a single hour-long stream has too many failure points. §19, §9,
    > `A20`, `A21`.

17. Our bulk import API accepts a 2 GB CSV upload through our API pods. Pods OOM, uploads fail midway
    over slow links, and the ALB times out.

    > **Direction:** Upload directly to object storage with presigned multipart/resumable uploads, then
    > process asynchronously and report status via an operation resource. §19, §21, `A20`, `A21`.

18. A streamed JSON response to a slow mobile client makes our Node.js server's memory climb to
    several GB for one request.

    > **Direction:** Writes ignored backpressure, so the server buffered the whole response in memory;
    > respect `write()` returning false and wait for `drain` (or use `pipeline`). §19, `A20`, `C04`.

19. Our GraphQL server caches resolver results in Redis keyed by field name and arguments. After
    enabling it, some users saw fields they were not authorised to see.

    > **Direction:** Field-level caches must include the authorisation context (user/role/tenant) in the
    > key, or cache only public data; auth is part of the result. §17, §9, `A15`, `A18`.

20. A client syncs a large dataset by paging with `limit=1000` and the provider changes data during
    the scan; the client's final copy never matches the source. What guarantees can the API offer?

    > **Direction:** Pagination over live data isn't a snapshot; offer a snapshot token/consistent read
    > point, or a full export plus a change feed from a known position. §13, §21, `A05`, `A21`.

### Level 10 — Cross-layer, multi-cause incidents

The rarest and hardest: several of the above interacting. Interviewers use these to see how you reason
when the first two hypotheses are wrong.

1. After migrating from EC2 to Kubernetes behind the same ALB, we see 0.2% 502s. We fixed Node's
   keep-alive timeout to 65 s, which halved them, and fixed deploy draining, which halved them again.
   The remaining 0.05% appear on pods with an Envoy sidecar only. What is left?

   > **Direction:** A second hop of keep-alive — ALB→Envoy→app (and Envoy's own idle timeouts) — each
   > hop needs upstream idle timeout > downstream; check Envoy's `idle_timeout` and upstream pool
   > settings, and read Envoy's response flags. §1, §2, `A12`, `M07`.

2. A partner integration fails only for requests over roughly 1400 bytes, only from their EU office,
   only over IPv6, and only since we moved to a new CDN. Where do you start and why?

   > **Direction:** A PMTUD black hole on the v6 path (ICMPv6 Packet Too Big filtered) exposed by a new
   > edge with different MTU/MSS; test with `ping -6 -M do` and `tracepath`, and have the CDN or firewall
   > permit PTB or clamp MSS. §4, §7, `A12`, `A13`.

3. Every day at about 09:00, API latency jumps for 10 minutes. It coincides with a burst of
   certificate-related errors in client logs and a spike in DNS queries. The certificates are valid.

   > **Direction:** A morning connection storm (laptops waking, pools re-established) causes full TLS
   > handshakes and uncached DNS lookups en masse; look at handshake counts, resumption rates, OCSP
   > fetches and resolver limits, and smooth with longer-lived connections, session resumption and
   > local DNS caching. §8, §7, §1, `A11`, `A13`.

4. A Java client calling our gRPC service over an AWS NLB gets `UNAVAILABLE` roughly every 350
   seconds of idleness. We enabled client keepalive every 60 s, and now the server sends GOAWAY with
   `too_many_pings`. Then we disabled keepalive and the original issue returned. What is the full fix?

   > **Direction:** Two constraints: the NLB's 350 s idle timeout and the server's minimum ping
   > interval; set client keepalive between them (e.g. 240 s) and raise the server's
   > `permit_keepalive_time` to match, or increase the NLB idle timeout. §6, §3, `A14`, `A12`.

5. After a DNS-based region failover, traffic moved in minutes for browsers, but mobile apps kept
   hitting the dead region for an hour, and our backend-to-backend traffic never moved until we
   restarted pods.

   > **Direction:** Three different caches: resolver TTL (browsers), OS/runtime caching and long-lived
   > HTTP/2 connections in apps, and JVM/pooled connections in backends; failover needs connection
   > lifetime caps, `GOAWAY`/`Connection: close` from the dead region, and runtime TTL settings. §7,
   > §5, §6, `A13`, `A10`.

6. Our API's p99 doubled after a "harmless" change: the frontend added an `X-Request-Id` header to all
   calls. Server-side latency is unchanged.

   > **Direction:** The custom header turned simple requests into preflighted ones, and the per-URL
   > preflight cache means an extra round trip (and sometimes TLS) for most calls; drop the header on
   > cross-origin reads, raise `Max-Age`, or serve same-origin. §10, §22, `A19`.

7. An internal service's error rate spikes whenever we scale in. Autoscaling removes pods, clients get
   resets, retries hit the rate limiter — which is per pod — and legitimate traffic gets 429s even
   though total traffic went down.

   > **Direction:** Three interacting faults: no drain on scale-in, retries without budget, and
   > per-instance rate limits whose effective total falls as pods disappear; fix draining, add retry
   > budgets, and centralise or re-divide limits dynamically. §2, §15, §16, `A08`, `M09`.

8. A webhook provider's retries, our autoscaler and our idempotency store interacted during an outage:
   deliveries timed out, we scaled up, new pods had a cold connection pool to the idempotency DB,
   lock waits piled up, and duplicate processing appeared. Walk me through breaking the loop.

   > **Direction:** Acknowledge webhooks fast and persist raw events, so provider retries stop; dedupe
   > with a unique-constraint insert (not locks held across processing); bound concurrency so scale-up
   > doesn't overwhelm the DB. §11, §14, §15, `A16`, `A09`.

9. A mobile app release increased our CDN egress by 40% and origin load by 3×. The only API change was
   adding the app version to the `User-Agent`, and a new `Accept-Language` negotiation for one
   endpoint.

   > **Direction:** Headers in `Vary` or the cache key (`User-Agent`, `Accept-Language`) fragmented the
   > cache; normalise them at the edge into a small set of values, or drop them from the key. §9,
   > `A04`.

10. Some customers see stale data only through our API gateway, only for authenticated requests, and
    only after they change roles. Purges don't help.

    > **Direction:** Stale identity at several layers: JWT claims cached until expiry, JWKS/role caches
    > at the gateway, and a response cache keyed without the role; shorten token life, key caches on
    > authorisation context, or revalidate roles on sensitive paths. §20, §9, `A18`, `A04`.

11. We switched internal JSON APIs to protobuf over gRPC for performance. A week later, a data
    pipeline in JavaScript that reads the same events via the gRPC JSON gateway started producing
    wrong joins, and a Java service began rejecting some messages.

    > **Direction:** protobuf's JSON mapping emits `int64` as strings (JS code comparing numbers to
    > strings) and enums by name; unknown enum values fail strict JSON parsers. Pin the JSON mapping
    > options and contract-test both encodings. §18, §6, `A17`, `A14`.

12. Our API is fronted by a CDN and an ALB. During a traffic spike, we see 504s from the CDN, 460s in
    ALB logs and successful 200s in app logs — all for the same requests.

    > **Direction:** Timeouts are out of order: the CDN's origin timeout fired first, the CDN closed the
    > connection (ALB logs 460 client-closed), and the app finished anyway. Align timeouts to shrink
    > inward and propagate cancellation. §16, §2, `A01`, `M09`.

13. A load balancer migration from nginx to Envoy increased duplicate POSTs to our payment API by an
    order of magnitude. Both configs "retry on connect failure only".

    > **Direction:** Envoy's retry conditions (e.g. `reset`, `5xx`, per-try timeouts) and its HTTP/2
    > upstream semantics differ; a reset after the request was sent is not a connect failure. Restrict
    > retries to idempotent methods, and require idempotency keys. §2, §14, §16, `A09`, `M07`.

14. We added HTTP/2 between our ALB and targets. The number of backend connections fell from 2000 to
    30, and suddenly two pods out of twenty are hot while the rest idle, even with round-robin.

    > **Direction:** With multiplexing, per-connection balancing concentrates load on the few connections'
    > targets; the LB's per-request balancing and connection count per target need checking, or cap
    > streams per connection and connection age. §5, §6, `A10`, `M07`.

15. A TLS certificate rotation went fine for browsers, but broke a subset of Android clients, a
    Python service, and our own health checker — each with a different error.

    > **Direction:** Different trust stores and validation paths: the new chain uses a root missing on
    > old Android, the Python container has a stale `ca-certificates`, and the checker pins the old
    > intermediate. Test rotations against the real client matrix with `openssl s_client`. §8, §22,
    > `A11`.

16. Our realtime tier and REST API share a hostname through the same ALB. After a surge of WebSocket
    reconnects, REST latency spiked and ALB 503s appeared, although REST pods were idle.

    > **Direction:** Shared edge capacity — connection-rate limits, LB capacity units and TLS handshakes
    > — is consumed by the reconnect storm; isolate realtime on its own LB/hostname and add jittered
    > reconnects. §12, §8, §2, `A16`, `A11`.

17. A partner sees intermittent signature failures on our signed requests: only through their
    corporate proxy, only for POSTs, and only when the body is larger than a few kilobytes.

    > **Direction:** The proxy is altering the request — adding `Expect: 100-continue`, rechunking or
    > compressing the body, or normalising headers covered by the signature; sign a body digest and
    > minimal headers and verify raw bytes. §20, §2, §19, `A18`.

18. Our public API has a sliding-window rate limit in Redis. After enabling HTTP/2 at the edge and a
    new SDK with connection reuse and hedging, customers hit limits at half their previous request
    rates.

    > **Direction:** Hedging and SDK retries count as extra requests, and HTTP/2 lets clients burst far
    > more concurrently; exclude hedges/retries from limits (e.g. by idempotency key) or set explicit
    > burst capacity with a token bucket. §15, §16, §5, `A08`, `A10`.

19. A gradual rollout of a new API version via a header-based canary looks healthy on metrics, but
    after 100% rollout several large customers break. Their traffic never went through the canary.

    > **Direction:** Header- or cookie-based canaries skip clients that never send the header (server
    > SDKs, cached CDN responses); canary by tenant/API key cohort and mirror shadow traffic from real
    > large consumers before full rollout. §21, §23, `A06`, `A22`.

20. You join a company where the API has "random" issues: occasional 502s, periodic 5-second latency
    spikes, sporadic CORS errors and duplicate webhooks. Nobody knows where to start. How do you
    triage across all of them in the first two weeks?

    > **Direction:** Attribute each error to its author and join edge, proxy and app logs by request ID;
    > then check the known patterns: keep-alive ordering (502), DNS/SYN timing signatures (5 s / 1 s
    > spikes), CORS masking server errors, and webhook ack latency. §22, §1, §7, §10, §11.
