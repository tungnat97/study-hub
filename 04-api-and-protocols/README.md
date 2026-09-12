[← back to the index](../README.md)

# Field 4 — API Design, HTTP & Network Protocols

The field interviewers use to separate "writes endpoints" from "designs interfaces other teams
depend on for years". Legend: `B` `I` `A` `X`.

---

## Nodes

#### A01 · HTTP/1.1 semantics
`B` · Requires: — · Unlocks: A02, A03, A04, A10, A19, F01, M04
- Key: request line, headers, body; methods and their semantics; status classes and the ones that
  matter (200/201/202/204, 301 vs 302 vs 307/308, 400/401/403/404/409/410/422/429, 500/502/503/504
  — know the difference between 502, 503 and 504 and who emits them); headers you must know
  (`Content-Type`, `Accept`, `Authorization`, `Location`, `Retry-After`, `X-Forwarded-For`);
  persistent connections; why HTTP is stateless and what that forces.
- Q: "401 vs 403? 400 vs 422? 502 vs 504?"
- Q: "Client got a 504. Which component produced it and what does it tell you?"

#### A02 · Safety, idempotency, cacheability of methods
`B` · Requires: A01 · Unlocks: A03, A09, M09
- Key: safe (GET, HEAD, OPTIONS — no side effects), idempotent (GET, HEAD, PUT, DELETE, OPTIONS —
  same effect N times as once), neither (POST, PATCH). Idempotent ≠ same response (DELETE returns
  404 the second time — still idempotent). This table drives retries, caching and proxy behaviour.
- Q: "Is PATCH idempotent?" (not necessarily — `{"$inc": 1}` is not; a full-field patch is).
- Q: "Why can a proxy retry a GET but not a POST?"

#### A03 · REST & resource modelling
`I` · Requires: A02, F02 · Unlocks: A05, A06, A07, A21
- Key: resources as nouns, hierarchy and nesting depth (stop at one level), collection vs item,
  representing actions that aren't CRUD (`POST /orders/1/cancel` vs a `status` PATCH vs a
  `cancellations` resource — be able to argue), Richardson maturity levels, HATEOAS (and honestly:
  rarely used, know why), consistency of naming/plurals/casing, `PUT` vs `PATCH` (replace vs merge;
  JSON Merge Patch vs JSON Patch), partial responses.
- Q: "Model 'user transfers money to another user' as REST."
- Q: "When is REST the wrong shape?" (chatty client needs → GraphQL/BFF; internal high-volume →
  gRPC; streaming → WS/SSE; complex workflows → RPC-style endpoints).

#### A04 · HTTP caching
`I` · Requires: A01 · Unlocks: A20, Q09, SD05
- Key: `Cache-Control` (`max-age`, `s-maxage`, `no-store` vs `no-cache`, `private` vs `public`,
  `stale-while-revalidate`), `ETag` + `If-None-Match` → 304, `Last-Modified` + `If-Modified-Since`,
  `Vary` (and the `Vary: Authorization` trap), strong vs weak validators, CDN vs browser vs proxy
  caching, cache busting via content-hashed URLs, `If-Match` for optimistic concurrency on writes.
- Q: "Explain `no-cache` vs `no-store`." (revalidate vs never store).
- Q: "Two users see each other's data through the CDN. What header was wrong?"

#### A05 · Pagination, filtering, sorting
`I` · Requires: A03, DB20 · Unlocks: A21, SD05
- Key: offset/limit (simple, breaks on deep pages and shifting data — duplicates and skips) vs
  **cursor/keyset** (stable, O(1), no jump-to-page); opaque encoded cursors; a mandatory default and
  max page size; total counts are expensive (make them optional/estimated); consistent sort with a
  tiebreaker column; filter syntax design and injection risk; sparse fieldsets.
- Q: "Design pagination for an infinite feed with items being inserted at the top."
- Q: "Why is your cursor opaque and base64-encoded?" (it's an implementation detail you must be free
  to change; prevents clients constructing them).

#### A06 · Versioning & evolution
`A` · Requires: A03, M24 · Unlocks: A22
- Key: URL (`/v1`) vs header/media-type vs query param; the real answer is **evolve without
  versioning** (additive-only, tolerant reader, never remove or repurpose); when you must break:
  parallel versions with a deprecation policy, `Deprecation`/`Sunset` headers, usage telemetry per
  client, migration guides, brownouts to force migration.
- Q: "How do you deprecate a field consumed by an unknown number of mobile clients you can't force
  to upgrade?"

#### A07 · Error contracts
`I` · Requires: A03, F07 · Unlocks: A22
- Key: RFC 7807 `application/problem+json` (`type`, `title`, `status`, `detail`, `instance` +
  extensions); a stable machine-readable error **code** separate from human text; field-level
  validation errors; correlation/trace id; never leak internals; distinguish retryable
  (`Retry-After`) from terminal; consistent across all endpoints.
- Q: "Design an error response a mobile client can act on programmatically."

#### A08 · Rate limiting & quotas
`A` · Requires: A01, M06 · Unlocks: S13, SD11
- Key: algorithms — fixed window (boundary burst), sliding window log (accurate, memory heavy),
  sliding window counter, **token bucket** (bursts + steady rate, the usual answer), leaky bucket;
  per-user vs per-IP vs per-API-key vs per-tenant vs global; distributed counters in Redis (atomic
  Lua/`INCR` + TTL) and the accuracy/performance trade; response 429 + `Retry-After` +
  `RateLimit-*` headers; separating abuse protection from fair-use quotas; cost-based limits.
- Q: "Implement a distributed rate limiter. Now make it exact. Now make it cheap."
- Q: "Rate limit by IP breaks a corporate customer behind one NAT. Now what?"

#### A09 · Idempotency keys
`A` · Requires: A02, M09 · Unlocks: SD10
- Key: client generates a key per logical operation, server stores key → (status, response) with a
  TTL, atomically claims the key (unique index / `SET NX`), returns the stored response on replay,
  handles the in-flight replay (409 or wait), scopes the key per endpoint + per user, and rejects
  the same key with a different payload.
- Q: "Write the full server-side algorithm for `Idempotency-Key` on `POST /charges`, including the
  case where the first request crashed halfway."

#### A10 · HTTP/2 and HTTP/3
`A` · Requires: A01 · Unlocks: A11, A12, A20
- Key: HTTP/2 — binary framing, multiplexed streams over one TCP connection (no head-of-line
  blocking at HTTP level, but **TCP-level HOL remains**), HPACK header compression, server push
  (deprecated), flow control per stream; the load-balancing consequence (see M07). HTTP/3 — QUIC over
  UDP, streams independent so no TCP HOL, 0-RTT resumption, connection migration, TLS 1.3 built in.
- Q: "Does HTTP/2 eliminate head-of-line blocking?" (at the HTTP layer yes, at TCP no — that's
  exactly why HTTP/3 exists).

#### A11 · TLS
`A` · Requires: A10, A12 · Unlocks: S10, S15, M26
- Key: handshake (1-RTT in TLS 1.3, 0-RTT and its replay risk), certificate chain and trust store,
  SNI, ALPN, cipher suites, forward secrecy, session resumption, **mTLS** for service identity,
  certificate rotation/expiry as a top outage cause, termination point (LB vs pod) and re-encryption,
  HSTS, pinning trade-offs.
- Q: "Walk me through a TLS 1.3 handshake."
- Q: "Where do you terminate TLS and what do you lose by terminating at the load balancer?"

#### A12 · TCP & the transport layer
`A` · Requires: — · Unlocks: A10, A11, A13, C02, O02
- Key: three-way handshake and its RTT cost (why connection reuse matters), slow start and
  congestion control, Nagle + delayed ACK interaction, TIME_WAIT and ephemeral port exhaustion,
  backlog/`SOMAXCONN` and accept queue overflow, keep-alive vs application-level heartbeats,
  MTU/MSS, bandwidth-delay product, half-open connections and why you still need app-level timeouts.
- Q: "Your service has thousands of sockets in TIME_WAIT. What does it mean and what do you change?"
  (client-side connection churn → keep-alive/pooling).
- Q: "A connection is 'established' but nothing arrives. Why won't TCP tell you?"

#### A13 · DNS
`I` · Requires: A12 · Unlocks: O02, O11
- Key: resolution path (stub → recursive → root → TLD → authoritative), record types (A/AAAA/CNAME/
  SRV/TXT), TTL and propagation, negative caching, DNS-based load balancing and failover limits
  (client caching ignores your TTL — JVM used to cache forever), split-horizon DNS, K8s CoreDNS and
  `ndots` search-domain latency, DNS as a common outage root cause.
- Q: "You changed a DNS record for failover. Traffic still goes to the dead host. Explain."

#### A14 · gRPC & Protocol Buffers
`A` · Requires: A10, A17, M04 · Unlocks: A22
- Key: IDL-first, generated stubs, HTTP/2 transport, 4 call types (unary, server/client/bidi
  streaming), deadlines propagated through the call chain, status codes, interceptors, metadata,
  channel and connection pooling, load-balancing caveat (M07), **wire compatibility rules**: never
  change or reuse a field number, never change a type, `reserved` for removed fields, all fields
  optional on the wire, no required. Browser needs grpc-web/Connect.
- Q: "What are the rules for evolving a proto message safely?"
- Q: "gRPC vs REST for a public partner API?" (REST — tooling, debuggability, browser, firewalls).

#### A15 · GraphQL
`A` · Requires: A03, DB20 · Unlocks: SD12
- Key: single endpoint, client-specified queries, schema + resolvers, the **N+1 resolver problem**
  and DataLoader batching; no HTTP caching by default (persisted queries + GET to get it back);
  security: depth/complexity limits, cost analysis, disable introspection in prod, authorisation must
  be per-field/per-object not per-endpoint; errors as 200 + `errors[]`; federation/schema stitching
  for multi-service graphs; when it beats REST (many clients with divergent data needs, deep
  aggregation) and when it doesn't (simple CRUD, public API, caching-heavy read path).
- Q: "A single GraphQL query brings your DB down. Three defences."
- Q: "How does authorisation differ between REST and GraphQL?"

#### A16 · Realtime protocols & webhooks
`A` · Requires: A01, A10 · Unlocks: F16, SD12
- Key: polling vs long polling vs SSE (one-way, HTTP, auto-reconnect with `Last-Event-ID`) vs
  WebSocket (bidirectional, stateful, own auth/heartbeat/reconnect design) vs WebTransport;
  **webhooks as an API you provide**: signed payloads (HMAC + timestamp to stop replay), at-least-
  once delivery so consumers must be idempotent, retries with backoff, delivery log and manual
  replay, subscriber-side verification, SSRF risk on subscriber URLs, ordering not guaranteed.
- Q: "Design an outbound webhook system. How do you prove to the receiver the event is from you?"
- Q: "The receiver is down for 6 hours. What happens?"

#### A17 · Serialisation & schema evolution
`A` · Requires: A01 · Unlocks: A14, Q19
- Key: JSON (ubiquitous, verbose, no schema, number precision/int64 problem), protobuf (compact,
  schema, fast, tag-based evolution), Avro (schema in registry, great for Kafka, reader/writer
  schema resolution), MessagePack/CBOR, Thrift; forward vs backward vs full compatibility; nulls vs
  absent vs default; enums and unknown values; dates as ISO-8601 strings in UTC.
- Q: "Your JS client mangles a 64-bit id. Why and what do you do?" (double precision → send as
  string).
- Q: "Backward vs forward compatibility — define both in terms of who is upgraded first."

#### A18 · API authentication & authorisation
`A` · Requires: A01, S03, S04 · Unlocks: S05
- Key: API keys (service-to-service, rotation, scoping) vs session cookies (browser-first,
  `HttpOnly`+`Secure`+`SameSite`, server-side revocation) vs JWT bearer (stateless, but revocation
  needs short TTL + refresh rotation + a denylist); OAuth2 grant types (authorization code + PKCE
  for apps, client credentials for M2M, device code; implicit and ROPC are dead); OIDC adds
  identity (`id_token`); token audience/issuer/scope validation; JWKS and key rotation; service
  identity via mTLS/SPIFFE instead of shared secrets.
- Q: "Cookie or JWT for your web app? Defend it." (cookies unless you have a real reason; the
  interviewer is testing whether you know JWT's revocation problem).
- Q: "How do you revoke a JWT immediately?"

#### A19 · CORS
`I` · Requires: A01 · Unlocks: F26, S08
- Key: same-origin policy, simple vs preflighted requests (`OPTIONS`, `Access-Control-Request-*`),
  `Access-Control-Allow-Origin` (cannot be `*` with `credentials: include`), `Allow-Headers/Methods`,
  `Expose-Headers`, `Max-Age`, why CORS is a **browser** protection and not server security,
  reflecting the Origin header as a vulnerability.
- Q: "CORS error in the browser but curl works. Explain to a junior."
- Q: "Does CORS protect your API?" (no — it protects users' browsers; your API still needs authz).

#### A20 · Payloads: compression, streaming, large responses
`I` · Requires: A04, A10 · Unlocks: C19, F17
- Key: gzip/brotli/zstd and where to do it (LB vs app), `Transfer-Encoding: chunked`, streaming
  JSON/NDJSON for large exports, `Range` requests and resumable downloads, response size limits and
  pagination as the real fix, BREACH risk when compressing secrets with attacker-controlled input.
- Q: "An export endpoint OOMs the pod. Fix it." (stream from a DB cursor, or offload to a job +
  presigned S3 link).

#### A21 · Long-running operations & bulk APIs
`A` · Requires: A03, A05 · Unlocks: SD12
- Key: 202 Accepted + `Location` of a job resource + polling (or webhook/SSE on completion); job
  status model (pending/running/succeeded/failed + progress + result link); bulk endpoints and
  **partial success** (207-style per-item results, never all-or-nothing silently); batch size limits;
  idempotency across a batch.
- Q: "Design `POST /reports` where generation takes 4 minutes."
- Q: "Bulk create with 1000 items, 3 of which fail validation. What is the response?"

#### A22 · Contract testing & API governance
`A` · Requires: A06, A07, A14, M24 · Unlocks: M35
- Key: OpenAPI/proto as the contract; consumer-driven contract tests (Pact) vs provider schema
  tests; CI diff for breaking changes (`oasdiff`, `buf breaking`); style guides and linting
  (Spectral); a deprecation policy with teeth; SDK generation; changelogs.
- Q: "How do you guarantee you never break a consumer, without integration-testing against all of
  them?"

---

## Topological order (study waves)

```
Wave 0  A01  A12
Wave 1  A02  A04  A10  A13  A17  A19
Wave 2  A03  A11  A16  A20
Wave 3  A05  A06  A07  A08  A09  A14  A15  A18  A21
Wave 4  A22
```

Cross-field parents: `F02/F07` framework routing & errors, `DB20` query tuning, `M06` gateway,
`M09` retries, `M24` contracts, `S03/S04` tokens & OAuth.

**Most-asked five:** idempotency keys end-to-end (A09), rate limiter design (A08), pagination for a
live feed (A05), JWT vs sessions and revocation (A18), CORS explained correctly (A19).
