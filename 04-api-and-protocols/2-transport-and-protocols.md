[← back to the field index](README.md)

# API & Protocols · Part 2 — Transport, TLS, gRPC, GraphQL & Serialisation

Nodes `A10`–`A17`.

---

## A10 · HTTP/2 and HTTP/3

`Advanced` · Requires: `A01` · Unlocks: `A11`, `A12`, `A20`

### Preface

HTTP/1.1 sends one request at a time per connection, so browsers open several connections and
requests still queue. HTTP/2 fixes that by multiplexing many requests over one connection. HTTP/3
fixes what HTTP/2 could not, by replacing TCP with QUIC.

For backend work the consequences that matter are: connection reuse becomes essential, load
balancing changes (`M07`), and header compression changes how you think about header size.

### Details

#### 1. What HTTP/2 changed

**Theory.** Binary framing instead of text. **Multiplexing**: many concurrent streams over one TCP
connection, so requests no longer queue behind each other at the HTTP layer. **HPACK** header
compression, which matters because headers repeat heavily across requests. Stream prioritisation and
per-stream flow control. Server push (added, barely used, now removed from browsers).

**Example.** Practical effect on a page with 50 assets: HTTP/1.1 opens six connections and processes
them in waves. HTTP/2 opens one and requests all 50 concurrently. The saving is largest on
high-latency connections, which is where it matters most.

**Advanced.** HPACK maintains a shared dynamic table of previously-seen headers, so a repeated
`Authorization` header costs a couple of bytes after the first request. That changes the calculus on
header size for repeated requests — though the first request still pays, and intermediaries may have
smaller table sizes than you assume.

#### 2. The head-of-line blocking question

**Theory.** HTTP/2 removes head-of-line blocking **at the HTTP layer**: one slow response no longer
blocks others. It does not remove it at the **TCP** layer: TCP delivers bytes in order, so a single
lost packet stalls every stream on that connection until it is retransmitted.

**Example.** On a lossy mobile network, HTTP/2 can perform *worse* than HTTP/1.1 with six
connections, because one lost packet blocks all streams, whereas with six connections it blocks one
sixth of the work. This is a genuinely counter-intuitive result and a good thing to be able to
explain.

**Advanced.** That limitation is precisely why HTTP/3 exists. This is the question interviewers use
to check whether you understand the layering: "does HTTP/2 eliminate head-of-line blocking?" — at the
HTTP layer yes, at TCP no, and that gap is what QUIC closes.

#### 3. HTTP/3 and QUIC

**Theory.** QUIC runs over UDP and implements its own streams, reliability, congestion control and
encryption. Streams are genuinely independent, so a lost packet affects only its own stream. TLS 1.3
is built in, so connection setup takes one round trip (or zero when resuming). **Connection
migration** means a connection survives a change of network — a phone moving from Wi-Fi to cellular
keeps the same connection.

**Example.** For a mobile-heavy consumer application the wins are real: faster connection setup,
resilience to packet loss, and no reconnection when the network changes. For internal
service-to-service traffic in a low-loss data centre, the benefit is small.

**Advanced.** 0-RTT resumption has a genuine security caveat: data sent in the first flight can be
**replayed** by an attacker, so it must only carry idempotent requests. This is a case where a
transport feature imposes an application-level constraint, and it connects neatly to `A02`.

#### 4. The load-balancing consequence

**Theory.** One long-lived connection carrying many requests interacts badly with layer-4 load
balancing, which picks a backend per connection.

**Example.** A gRPC client (HTTP/2) behind an L4 balancer sends every request to one backend for the
life of the connection. The symptom is wildly uneven CPU across pods (`M07`). The fixes: an L7 proxy
that balances per request, client-side load balancing, or a maximum connection age forcing periodic
re-balancing.

**Advanced.** This is one of the most practically important consequences of HTTP/2 for backend
engineers, and it is frequently discovered only after a production incident. If you adopt gRPC or
HTTP/2 between services, decide the balancing strategy at the same time.

### Interview questions

- "Does HTTP/2 eliminate head-of-line blocking?"
- "When could HTTP/2 be slower than HTTP/1.1?"
- "What does QUIC give you that TCP cannot?"
- "Why is 0-RTT restricted to idempotent requests?"

---

## A11 · TLS

`Advanced` · Requires: `A10`, `A12` · Unlocks: `S10`, `S15`, `M26`

### Preface

TLS gives you three things: **confidentiality** (nobody can read the traffic), **integrity** (nobody
can change it undetected), and **authentication** (you are talking to who you think).

For backend work, the operational facts matter more than the cryptography: certificates expire and
cause outages, termination points determine what you can inspect, and mutual TLS is how services
prove their identity to each other.

### Details

#### 1. The handshake

**Theory.** In TLS 1.3 (one round trip): the client sends a hello with supported ciphers and a
key share; the server responds with its choice, its certificate, a signature proving it holds the
private key, and its key share. Both derive the same session key. Application data then flows
encrypted with symmetric cryptography, which is far faster than asymmetric.

**Example.** TLS 1.2 needed two round trips; TLS 1.3 needs one, and zero when resuming a previous
session. On a 100ms link that is a 100-200ms saving on every new connection — which is the practical
argument for connection reuse and keep-alive (`F24`).

**Advanced.** The asymmetric cryptography is used only to agree the key and prove identity; the bulk
data uses a symmetric cipher (AES-GCM or ChaCha20-Poly1305). That hybrid is the standard pattern
(`S11`). TLS 1.3 also removed the older, weaker options entirely — no RSA key exchange, no CBC modes
— so forward secrecy is mandatory rather than optional.

#### 2. Certificates and trust

**Theory.** A certificate binds a public key to a name, signed by a certificate authority. The
client verifies the chain up to a root it trusts, checks the name matches, checks validity dates, and
checks revocation.

**Example.** The most common production failure by a wide margin is **expiry**. Automate renewal
(cert-manager in Kubernetes, ACME/Let's Encrypt), and alert on certificates expiring within 30 days —
including internal ones, which are the ones people forget. Expiry is a scheduled outage you know
about in advance and still miss (`M31`).

**Advanced.** SNI (Server Name Indication) sends the hostname in the clear during the handshake so
one IP can host many certificates — which means the hostname is visible to observers, addressed by
Encrypted Client Hello. And be careful with certificate **pinning**: it defends against a
compromised CA and it turns a certificate rotation into an outage for every client that has not
updated. For mobile applications especially, pin with a backup pin and a short expiry, or not at all.

#### 3. Where you terminate

**Theory.** TLS can terminate at the CDN, the load balancer, an ingress, a sidecar, or the
application. After termination, traffic is plaintext unless re-encrypted.

**Example.** Terminating at the load balancer is common and means: the load balancer can route by
path and inspect headers; and traffic from there to your pods is unencrypted unless you re-encrypt.
In a shared network that is a real exposure, and it is why service meshes offer automatic mutual TLS
between pods (`M26`).

**Advanced.** Also note what you lose by terminating early: the client certificate is not visible to
your application unless the load balancer forwards it in a header, and the original protocol and
scheme must be passed via `X-Forwarded-Proto` — a missing one causes redirect loops when the
application thinks the request was plain HTTP.

#### 4. Mutual TLS

**Theory.** Normally only the server presents a certificate. With mutual TLS, the client does too, so
both sides authenticate. This gives cryptographic **workload identity** rather than a shared secret.

**Example.** Between internal services, mTLS means service A can prove it is service A, and service B
can enforce a policy about who may call it — without API keys that can be copied from a compromised
container. SPIFFE/SPIRE standardises the identity format and automates short-lived certificate
issuance (`S15`).

**Advanced.** The reason this is a mesh feature (`M26`) is that doing it well requires automated
issuance, rotation every few hours, and distribution to every workload — which is substantial
infrastructure. Doing it badly (long-lived certificates copied into images) is worse than not doing
it, because it creates a false sense of security and a credential that never expires.

### Interview questions

- "Walk me through a TLS 1.3 handshake."
- "Where do you terminate TLS and what do you lose by terminating at the load balancer?"
- "What is mutual TLS and what problem does it solve?"
- "What is the most common TLS-related outage?"

---

## A12 · TCP and the transport layer

`Advanced` · Requires: — · Unlocks: `A10`, `A11`, `A13`, `C02`, `O02`

### Preface

TCP provides a reliable, ordered byte stream over an unreliable network. The costs of that guarantee
— a handshake before any data, retransmission delays, ordered delivery — explain several backend
behaviours that otherwise look mysterious.

The two facts that matter most: **opening a connection is expensive**, and **TCP will not tell you
when the other side has gone away**.

### Details

#### 1. The handshake and why reuse matters

**Theory.** Opening a TCP connection costs one round trip (SYN, SYN-ACK, ACK) before any data. Adding
TLS costs another (`A11`). On a 50ms link that is 100ms of latency before your request is even sent.

**Example.** So an HTTP client that opens a new connection per request pays that on every call. With
keep-alive, the first request pays and the rest are free. This is why connection pools exist for
databases, HTTP clients, Redis and gRPC — and why `keepAlive: true` on a Node agent is one of the
cheapest latency improvements available (`F24`).

**Advanced.** `TIME_WAIT` is the state a socket sits in for up to two minutes after the side that
closed it finishes, to absorb delayed packets. A service making many short-lived outbound connections
accumulates thousands of `TIME_WAIT` sockets and can exhaust its ephemeral port range (about 28,000
by default), after which new connections fail. The fix is connection reuse, not tuning kernel
parameters — though `net.ipv4.ip_local_port_range` and `tcp_tw_reuse` exist and are the usual first
(wrong) answer.

#### 2. Congestion control and slow start

**Theory.** TCP does not know the available bandwidth, so it starts cautiously and increases its
sending window until it detects loss, then backs off. **Slow start** means a new connection cannot
use full bandwidth immediately.

**Example.** This is another reason connection reuse matters: a warm connection has a large
congestion window and transfers at full speed; a new one ramps up over several round trips. For
large responses over long distances, the difference is significant.

**Advanced.** Modern congestion control (BBR rather than CUBIC) models the bottleneck bandwidth and
round-trip time rather than treating loss as the only signal, which performs markedly better on
lossy links. It is worth knowing the name and the idea — that loss-based control confuses "the
network is congested" with "a packet was corrupted" — but this is infrastructure tuning rather than
application work.

#### 3. Half-open connections and why you still need timeouts

**Theory.** If the other side vanishes — a machine is powered off, a network partition occurs, a
container is killed without closing sockets — TCP does not notice. Your socket stays `ESTABLISHED`
forever, waiting for data that will never come.

**Example.** This is the reason every remote call needs an application-level timeout (`M09`).
TCP keep-alive exists but defaults to two hours before the first probe, which is useless for request
handling. Application-level heartbeats (WebSocket ping/pong, gRPC keepalive) detect it in seconds.

**Advanced.** `CLOSE_WAIT` accumulating on your server means the **peer closed** and your application
has not called `close()` — an application bug, usually a connection leak in error handling. Many
sockets in `CLOSE_WAIT` is a specific, actionable diagnosis (`O02`), and recognising it distinguishes
someone who has debugged production networking.

#### 4. Buffers, backlog and limits

**Theory.** The kernel queues completed connections in an accept queue until the application accepts
them. If the application is too slow or the queue (`SOMAXCONN`, and the `listen()` backlog) is too
small, connections are dropped or refused.

**Example.** Under a sudden traffic spike, a slow application produces connection failures rather than
slow responses, because the accept queue overflows. The symptom is confusing — clients see connection
refused or reset, not timeouts. Monitor listen-queue overflows (`netstat -s | grep -i listen`).

**Advanced.** Also note the file descriptor limit (`ulimit -n`, `O01`): each connection is a
descriptor, and the default of 1,024 in some environments caps you well below what the machine could
handle. `Too many open files` under load is this, and it is also what a connection leak eventually
produces.

### Interview questions

- "Your service has thousands of sockets in `TIME_WAIT`. What does it mean and what do you change?"
- "A connection is 'established' but nothing arrives. Why won't TCP tell you?"
- "What does a pile of `CLOSE_WAIT` sockets indicate?"
- "Why does connection reuse matter beyond avoiding the handshake?"

---

## A13 · DNS

`Intermediate` · Requires: `A12` · Unlocks: `O02`, `O11`

### Preface

DNS turns names into addresses. It is simple in principle and is a recurring cause of outages,
because it is cached at many layers, each with its own idea of how long to keep an answer — including
layers that ignore your TTL entirely.

The key operational fact: **you do not control when clients stop using an old answer.**

### Details

#### 1. Resolution and caching

**Theory.** A client asks a resolver; the resolver may answer from cache or walk the hierarchy: root
→ top-level domain → authoritative server. Each answer carries a TTL, and caches are supposed to
honour it.

**Example.** The layers that cache: the application's process (a library or runtime cache), the
operating system's resolver cache, the local network's resolver, the ISP's resolver, and any proxy in
between. Lowering the TTL on your DNS record does not shorten caches that already hold the old value
— so a TTL reduction must happen **well before** a planned change, not at the same time.

**Advanced.** The infamous example is the JVM, which historically cached DNS resolutions forever
(`networkaddress.cache.ttl = -1` with a security manager). A database failover changes the record and
the application keeps connecting to the dead host until restarted. Node and Go generally respect
TTLs; connection pools are the other trap, since existing connections never re-resolve at all.

#### 2. Record types worth knowing

**Theory.** `A`/`AAAA` map a name to an IPv4/IPv6 address. `CNAME` aliases one name to another (and
cannot coexist with other records at the same name, which is why it cannot be used at a domain apex —
hence provider-specific ALIAS records). `SRV` gives host and port, used by some service discovery.
`TXT` holds arbitrary text, used for domain verification, SPF and DKIM. `MX` for mail.

**Example.** In Kubernetes, a Service's name resolves to a stable cluster IP (an `A` record served by
CoreDNS), and a headless Service returns the pod IPs directly — which is how StatefulSet members and
some client-side load balancers find individual instances (`M05`).

**Advanced.** DNS-based load balancing and failover is weak precisely because of caching: you cannot
move traffic quickly, and clients may ignore the ordering of multiple A records. Use it for coarse
geographic routing, and use a load balancer or a service mesh for anything needing fast, reliable
failover.

#### 3. DNS in containers

**Theory.** Containers get a `/etc/resolv.conf` pointing at the cluster's DNS service, with a
`search` list and `ndots`.

**Example.** Kubernetes sets `ndots:5`, meaning any name with fewer than five dots is first tried
against each search domain in turn. Resolving `redis` may therefore issue four or five queries before
succeeding — per lookup, unless cached. On a high-traffic service this is measurable latency and real
load on CoreDNS. The fixes: use fully-qualified names ending with a dot, reduce `ndots` in the pod
spec, or run NodeLocal DNSCache.

**Advanced.** DNS resolution in Node happens on the **libuv thread pool** (`dns.lookup`, which uses
`getaddrinfo`), whose default size is four (`C03`). A burst of new connections to many hostnames can
saturate the pool and delay other pool users, including file I/O and some crypto. Using
`dns.resolve` (which is asynchronous at the network level) or a caching resolver avoids it. This is a
genuinely obscure and genuinely real production issue.

#### 4. When DNS is the cause

**Theory.** DNS failures present as intermittent, partial and confusing: some pods fail, some
succeed, and the same request works when you retry.

**Example.** Symptoms worth recognising: `EAI_AGAIN` or `getaddrinfo ENOTFOUND` errors in logs;
latency spikes correlated with new connections rather than with query load; failures that resolve
themselves after a few seconds; and problems that appear only after scaling up (more pods, more DNS
queries, an overloaded CoreDNS).

**Advanced.** The generic debugging order for "service A cannot reach service B" starts with DNS and
works up the stack: resolve the name from inside the pod (`nslookup`), open a TCP connection
(`nc -vz`), complete the TLS handshake (`openssl s_client`), make the HTTP request (`curl -v`), then
check authentication and the application (`O02`). Always from inside the pod, never from your laptop.

### Interview questions

- "You changed a DNS record for failover. Traffic still goes to the dead host. Explain."
- "What is `ndots` and why might it slow you down?"
- "Service A cannot reach service B. Give me your diagnostic order."
- "Why is DNS-based failover unreliable?"

---

## A14 · gRPC and Protocol Buffers

`Advanced` · Requires: `A10`, `A17`, `M04` · Unlocks: `A22`

### Preface

gRPC is remote procedure calls over HTTP/2, with messages defined in a `.proto` file and client and
server code generated from it.

What you get over REST/JSON: a machine-checked contract, much smaller and faster messages, streaming
in both directions, and deadlines that propagate through the call chain. What you give up: human
readability, browser support without a proxy, and the ability to debug with curl.

### Details

#### 1. The IDL and code generation

**Theory.** You define messages and services in a `.proto` file; the compiler generates typed
clients and server interfaces for every language you need.

**Example.**

```proto
syntax = "proto3";

service Pricing {
  rpc GetPrice(GetPriceRequest) returns (Price);
  rpc WatchPrices(WatchRequest) returns (stream Price);   // server streaming
}

message GetPriceRequest {
  string sku = 1;
  string currency = 2;
}
```

The field **numbers** (1, 2) are what travel on the wire; the names are only for code generation.
That is the key to how compatibility works.

**Advanced.** Because numbers identify fields, changing a number is equivalent to deleting one field
and adding another — old data is misinterpreted, silently. Keep the `.proto` files in a shared,
versioned repository with a breaking-change checker (`buf breaking`) in CI, and treat them as the
most carefully reviewed files you own.

#### 2. Compatibility rules

**Theory.** The rules for evolving a message safely:
- **Never** change an existing field's number or type.
- **Never** reuse a number from a removed field — mark it `reserved`.
- Adding a new field with a new number is always safe; old code ignores it.
- All fields are optional on the wire in proto3 — there is no `required`.
- Renaming a field is safe on the wire and breaks generated code, so it is a source-level break.

**Example.**

```proto
message Price {
  reserved 3;                 // 'discount' used to live here
  reserved "discount";
  string sku = 1;
  int64 amount_minor = 2;
  string currency = 4;        // new field, new number
}
```

**Advanced.** Because every field is optional, a consumer cannot distinguish "not set" from "set to
the default" for scalars — `0`, `""` and `false` are indistinguishable from absence. That matters for
partial updates: use `optional` (which re-enables presence tracking in proto3) or a `FieldMask` to
say which fields the caller intends to change. This is a real modelling problem that catches people
moving from JSON.

#### 3. Streaming and deadlines

**Theory.** Four call types: unary, server streaming, client streaming, and bidirectional. Every call
carries a **deadline** that propagates to downstream calls, so the whole chain stops when the time is
gone (`M04`).

**Example.** Server streaming suits progress updates, tailing logs and large result sets. Bidirectional
streaming suits chat-like protocols. Deadlines are the standout feature: `ctx` with a deadline flows
through, and a service that is out of time stops work instead of computing an answer nobody will
receive.

**Advanced.** gRPC also has built-in **keepalive** pings, which matter because HTTP/2 connections are
long-lived and intermediate proxies drop idle connections without telling either side (`A12`). Tune
`keepalive_time` and `keepalive_timeout`, and be aware that servers can enforce a minimum to prevent
abusive clients — mismatched settings cause `ENHANCE_YOUR_CALM` errors that look inexplicable.

#### 4. When to use it

**Theory.** Internal, high-volume, polyglot, strongly-typed, streaming — gRPC. Public, partner-facing,
browser-consumed, human-debuggable — REST.

**Example.** Browsers cannot speak gRPC directly because they cannot control HTTP/2 framing;
**gRPC-Web** needs a proxy (Envoy) to translate. **Connect** is a newer protocol that speaks both
gRPC and a plain HTTP/JSON form from the same definition, which removes much of this friction and is
worth naming as the modern option.

**Advanced.** The operational cost of gRPC is real: load balancing needs attention (`M07`, `A10`),
debugging needs `grpcurl` rather than curl, logs and traces need explicit instrumentation, and the
code-generation step must be in every pipeline. For two services owned by the same team, JSON over
HTTP is often the better trade. Saying that rather than advocating gRPC universally reads as
experience.

### Interview questions

- "What are the rules for evolving a proto message safely?"
- "gRPC or REST for a public partner API?"
- "How do deadlines work and why do they beat per-hop timeouts?"
- "Why can't a browser call gRPC directly?"

---

## A15 · GraphQL

`Advanced` · Requires: `A03`, `DB20` · Unlocks: `SD12`

### Preface

GraphQL gives clients one endpoint and a query language: the client asks for exactly the fields it
wants, across related objects, in one request.

That solves over-fetching and under-fetching for clients with varied needs. It moves the cost onto
the server, which must now handle arbitrary query shapes — which means the N+1 problem, the cost of
unbounded queries, and per-field authorisation all become your problem.

### Details

#### 1. Schema and resolvers

**Theory.** You define a typed schema; each field has a **resolver** that produces its value. The
engine walks the query, calling resolvers as it goes.

**Example.**

```graphql
type Order { id: ID!, total: Int!, customer: Customer!, lines: [OrderLine!]! }
type Query { order(id: ID!): Order }
```

A query for `order { customer { name } }` calls the `order` resolver, then the `customer` resolver on
the result. That nesting is the feature and the danger.

**Advanced.** The schema is the contract, and evolving it follows the same additive rules as anywhere
else (`M24`): adding fields and types is safe; removing or changing them is breaking. GraphQL has a
built-in `@deprecated` directive, and — usefully — the server can see exactly which fields each
client actually requests, which makes measuring usage before removal far easier than with REST
(`A06`).

#### 2. The N+1 problem and DataLoader

**Theory.** Resolvers are called per object. Fetching 100 orders and then each order's customer calls
the customer resolver 100 times — 100 queries (`DB20`).

**Example.** **DataLoader** solves it by batching within a tick of the event loop: instead of
querying immediately, each call registers the key; at the end of the tick, one query fetches all keys
(`WHERE id IN (...)`), and the results are distributed back. It also caches per request, so the same
id requested twice costs one lookup.

**Advanced.** The loader must be created **per request**, not shared globally, or you leak data
between users through the cache — a genuine authorisation bug (`S09`). This is the most common
GraphQL security mistake and a very good detail to mention. Note also that DataLoader batches only
within a single event-loop tick, so `await`ing inside a resolver before calling the loader breaks
batching.

#### 3. Cost and complexity control

**Theory.** A client can write a query that is arbitrarily expensive — deeply nested, or requesting
thousands of items at every level. You must bound it.

**Example.** The standard defences:
- **Depth limiting** — reject queries nested beyond N levels.
- **Complexity analysis** — assign each field a cost, multiply by requested list sizes, reject above
  a budget.
- **Persisted queries** — clients register queries in advance by hash and may only send known hashes;
  arbitrary queries are rejected entirely. The strongest defence for a first-party API.
- **Pagination required** on every list field, with a maximum page size.
- **Introspection disabled** in production so the schema is not a public map.
- **Timeouts** at the query level.

**Advanced.** Persisted queries also restore HTTP caching: because the query is identified by a hash,
it can be sent as a `GET` and cached by a CDN (`A04`) — which is otherwise the big thing GraphQL
gives up relative to REST. That combination is the mature production setup.

#### 4. Authorisation and errors

**Theory.** In REST, authorisation is per endpoint. In GraphQL, a single query can reach any part of
the graph, so authorisation must be enforced **per field and per object**, in the resolvers or the
data layer.

**Example.** The failure mode: `order(id: 42) { customer { email } }` — the order resolver checks
access to the order, and the customer resolver returns the email without checking anything, because
"we already authorised the order". Enforce at the data-fetching layer (the loader scopes by tenant)
rather than in each resolver, so a new resolver is safe by default (`F11`).

**Advanced.** GraphQL's error model is unusual: transport errors are HTTP-level, but field errors
come back as HTTP **200** with an `errors` array and `null` for the failed field. Clients must check
`errors`, and your monitoring must too — a service returning 200 for every failed query looks
perfectly healthy on a status-code dashboard while everything is broken. Alert on the error array,
not the status code.

### Interview questions

- "A single GraphQL query brings your database down. Three defences."
- "How does authorisation differ between REST and GraphQL?"
- "Why must DataLoader be per request?"
- "Why does a GraphQL error return 200, and what does that break?"

---

## A16 · Realtime protocols and webhooks

`Advanced` · Requires: `A01`, `A10` · Unlocks: `F16`, `SD12`

### Preface

HTTP is request-response: the client asks, the server answers. When the **server** has something to
say, you need a different mechanism.

Four options, in increasing complexity: polling, long polling, server-sent events, and WebSockets.
And when the recipient is another server rather than a browser, the answer is usually a **webhook** —
which is an API you provide, with its own delivery guarantees to design.

### Details

#### 1. Choosing a push mechanism

**Theory.** **Polling** — ask every N seconds; simple, wasteful, latency up to N. **Long polling** —
the server holds the request open until it has something; near-real-time over ordinary HTTP.
**SSE** — one long-lived response streaming events; one-way, automatic reconnection, event ids for
resumption. **WebSocket** — a full-duplex connection after an HTTP upgrade.

**Example.** Decide by direction and frequency: server-to-client only and moderate frequency (a
dashboard, notifications, a progress bar) → SSE. Frequent bidirectional traffic (chat, collaborative
editing, games) → WebSocket. Occasional updates where a few seconds of delay is fine → polling, which
is genuinely the right answer more often than it is chosen.

**Advanced.** SSE's advantages are underrated: it is plain HTTP, so it works with existing proxies,
authentication, and compression; the browser reconnects automatically; and `Last-Event-ID` gives you
resumption for free. Its limits are the browser's per-domain connection cap under HTTP/1.1 (solved by
HTTP/2, `A10`) and being one-way — which for most "notify the client" use cases is exactly what you
need (`F16`).

#### 2. Webhooks as an API you provide

**Theory.** You call the customer's URL when something happens. It is an outbound API, and it needs
the same care as an inbound one: a versioned payload, retries, authentication, and a way to replay.

**Example.** The design checklist:
- **Signature** — sign the payload with a shared secret (HMAC-SHA256 over timestamp + body) so the
  receiver can verify it came from you.
- **Timestamp** in the signed content, and the receiver rejects anything older than a few minutes,
  to prevent replay.
- **At-least-once delivery** with retries and exponential backoff over hours.
- **Idempotency** — include a unique event id so receivers can deduplicate (`M16`).
- **No ordering guarantee** — say so explicitly, and include a sequence number or timestamp so
  receivers can detect out-of-order arrivals.
- **A delivery log** the customer can inspect, with manual replay.

**Advanced.** Signature verification must use a **constant-time comparison** (`S11`), or the
comparison leaks the correct signature through timing. And sign the raw body bytes, not the parsed
and re-serialised JSON, since key order and whitespace change the bytes — a classic integration bug
that produces "signature mismatch" for a payload that looks identical.

#### 3. Receiving webhooks

**Theory.** When you consume someone else's webhooks, you are exposing an endpoint to the internet
that performs actions based on a payload you did not create.

**Example.** The receiving checklist: verify the signature **before** parsing or acting; respond 200
**quickly** and do the work asynchronously (providers time out and retry, causing duplicates);
deduplicate by event id; tolerate out-of-order arrival (check the event's timestamp against your
state); and handle unknown event types gracefully rather than erroring.

**Advanced.** The "respond fast, process async" rule matters more than it appears: doing the work
inline means a slow database makes the provider time out and retry, so you get duplicate processing
*and* the provider may disable your endpoint for repeated failures. Accept, store, acknowledge, then
process from a queue (`Q10`).

#### 4. Scaling push

**Theory.** Long-lived connections are stateful and pinned to one process (`F16`), which conflicts
with horizontal scaling and with rolling deploys.

**Example.** The architecture that works: a connection layer that holds the sockets and does nothing
else; a pub/sub backplane so any instance can deliver to any connection; a durable per-user inbox for
messages that must not be lost; and clients that reconnect with a cursor and fetch what they missed.
Real-time delivery is an optimisation over a durable inbox, not a replacement for one.

**Advanced.** Plan for the reconnect storm: a deploy or a network blip disconnects everyone at once,
and they all reconnect together — a self-inflicted thundering herd (`M09`). Client reconnect logic
must use exponential backoff with jitter, and the server should be able to shed connection attempts
under load. Mentioning this unprompted signals you have operated something like it.

### Interview questions

- "Design an outbound webhook system. How do you prove to the receiver the event is from you?"
- "The receiver is down for six hours. What happens?"
- "SSE or WebSocket for a live dashboard?"
- "You receive webhooks from a payment provider. What does your endpoint do first?"

---

## A17 · Serialisation and schema evolution

`Advanced` · Requires: `A01` · Unlocks: `A14`, `Q19`

### Preface

Every message that leaves your process is serialised. The format determines size, speed, whether
there is a schema, and how gracefully the two sides can change independently.

JSON dominates because it is readable and universal. Binary formats with schemas — protobuf, Avro —
win where volume is high or where the contract must be machine-enforced.

### Details

#### 1. The formats

**Theory.**
- **JSON** — text, universal, self-describing, no schema, verbose, slow to parse at scale.
- **Protobuf** — binary, compact, schema-required, field numbers for evolution, very fast. Schema
  travels out of band (`A14`).
- **Avro** — binary, schema-required, and the schema is resolved between writer and reader, which
  makes it the natural fit for Kafka with a schema registry (`Q19`).
- **MessagePack / CBOR** — binary JSON; smaller and faster, still schemaless.

**Example.** Rough guidance: JSON at the edge and for anything humans debug; protobuf for
service-to-service RPC; Avro for event streams with a registry; MessagePack only when you want
smaller JSON without changing your model.

**Advanced.** JSON's cost is not only size — `JSON.parse` is synchronous and blocking in Node, so a
large payload stalls the event loop (`C03`, `F24`). Binary formats parse faster and, in the case of
protobuf, can be parsed lazily. That said, measure before switching: for most services the
difference is irrelevant compared with the database.

#### 2. Number and date traps in JSON

**Theory.** JSON numbers are IEEE doubles in JavaScript, which can represent integers exactly only up
to 2⁵³. JSON has no date type.

**Example.** A 64-bit id (a Snowflake id, a Postgres `bigint`) sent as a JSON number is **silently
corrupted** in a JavaScript client — the last digits change. The fix is to serialise 64-bit integers
as **strings**. Protobuf does this in its JSON mapping for `int64` for exactly this reason. For
dates, use ISO-8601 with an explicit offset (`2026-09-13T10:00:00Z`), never a local string, and
never a Unix timestamp without documenting the unit (`DB04`).

**Advanced.** The same class of bug appears with money as a JSON number: `0.1 + 0.2` problems
(`DB04`) and precision loss. Send money as an integer of minor units plus a currency code, or as a
string. These two rules — big integers as strings, money as minor units — prevent a whole family of
production bugs.

#### 3. Compatibility directions

**Theory.** **Backward compatible** — new code can read old data. **Forward compatible** — old code
can read new data. **Full** — both. Which one you need depends on who upgrades first.

**Example.** Reasoning it through for a message queue: if producers deploy before consumers, new
messages reach old consumers, so you need **forward** compatibility. If consumers deploy first, they
read old messages, so you need **backward**. In a rolling deploy both happen at once, so you need
**full** (`Q19`, `M24`).

**Advanced.** Achieving forward compatibility requires the reader to **preserve unknown fields**
rather than dropping them — otherwise a service that reads, modifies and rewrites a message silently
deletes fields it did not understand. Protobuf preserves unknown fields by default; naive JSON
mapping into a typed object does not. This is a subtle and genuinely damaging bug in pipelines where
messages are enriched by several services.

#### 4. Where the schema lives

**Theory.** Three options: no schema (JSON — the contract is documentation and hope); schema in the
code (protobuf files compiled into both sides); or schema in a registry (Avro with Confluent Schema
Registry, which validates compatibility at publish time).

**Example.** The registry approach is the strongest for event streams: producers register a schema,
the registry **rejects** an incompatible change according to the configured mode, and messages carry
a small schema id instead of field names — which also saves significant space. See `Q19`.

**Advanced.** The general principle worth stating: **the earlier a contract violation is caught, the
cheaper it is.** Compile time (protobuf) beats publish time (registry) beats consume time (JSON with
validation) beats "a customer noticed". Choosing a serialisation format is largely choosing where on
that scale you want to be.

### Interview questions

- "Your JavaScript client mangles a 64-bit id. Why and what do you do?"
- "Backward versus forward compatibility — define both in terms of who upgrades first."
- "Why does preserving unknown fields matter?"
- "When would you move from JSON to protobuf?"
