[← back to the field index](README.md)

# API & Protocols · Part 3 — Auth, CORS, Payloads & Governance

Nodes `A18`–`A22`.

---

## A18 · API authentication and authorisation

`Advanced` · Requires: `A01`, `S03`, `S04` · Unlocks: `S05`

### Preface

Every API has to answer "who is calling?" before it can answer "may they?". The mechanism depends on
the caller: a browser, a mobile app, another service, or a third-party integration each want
something different.

The question you will be asked is almost always some form of "session cookies or JWT?", and the
answer that scores well acknowledges JWT's revocation problem rather than treating stateless as
automatically better.

### Details

#### 1. The options

**Theory.**
- **Session cookie** — an opaque id in an `HttpOnly` cookie; the server holds the session state.
  Revocable instantly, requires a session store, and needs CSRF protection (`A19`, `F26`).
- **JWT bearer token** — a signed token the server verifies without a lookup. Stateless and
  scalable, and it **cannot be revoked** before it expires.
- **API key** — a long-lived secret for service or partner access. Simple, needs scoping, rotation
  and per-key rate limiting.
- **mTLS** — the client presents a certificate; strongest for service-to-service (`A11`, `S15`).

**Example.** Typical assignment: browser application → session cookie or a short-lived JWT in an
`HttpOnly` cookie; mobile application → OAuth2 with PKCE producing access and refresh tokens;
partner integration → API key with scopes; internal service-to-service → mTLS or a short-lived
service token.

**Advanced.** The strongest general answer for a first-party web application is "cookies, unless
there is a specific reason otherwise". Cookies with `HttpOnly` cannot be read by injected JavaScript,
are revocable server-side, and are handled by the browser correctly. The usual argument for JWT —
"stateless scaling" — is solving a problem most systems do not have, since a session lookup in Redis
is sub-millisecond.

#### 2. JWT, precisely

**Theory.** Three base64url-encoded parts: header (algorithm), payload (claims), signature. **It is
signed, not encrypted** — anyone holding the token can read the payload. Verification means checking
the signature and the standard claims: `exp` (expiry), `nbf` (not before), `iss` (issuer), `aud`
(audience).

**Example.** The validation checklist that must all be present:
- Pin the expected **algorithm**. Accepting whatever the token's header says allows `alg: none`
  (accept an unsigned token) and RS256-to-HS256 confusion (verify an RSA token as HMAC using the
  public key, which the attacker also has).
- Check `exp` and `nbf`, with only a small clock skew allowance.
- Check `iss` and, crucially, `aud` — otherwise a token issued for service A can be replayed against
  service B.
- Fetch signing keys from JWKS, cache them, and handle key rotation by `kid`.

**Advanced.** The revocation problem is the crux: a stolen token is valid until it expires. The
mitigations are a **short access-token lifetime** (5-15 minutes) plus a **refresh token with
rotation and reuse detection** — if an old refresh token is presented, the whole family is revoked,
which detects theft. For immediate revocation you need server state anyway (a denylist of `jti`
values, or a per-user "tokens issued before X are invalid" timestamp), at which point you have
partially reinvented sessions. Saying that out loud is the senior answer.

#### 3. OAuth2 in one paragraph

**Theory.** OAuth2 is about **delegated authorisation**: letting an application act on a user's
behalf without receiving their password. OIDC layers identity on top, adding an `id_token`.

**Example.** Which grant to use (`S04`): **authorisation code with PKCE** for every interactive
client, including single-page and mobile applications; **client credentials** for machine-to-machine;
**device code** for input-constrained devices. The implicit grant and the
resource-owner-password grant are deprecated and should not appear in a new design.

**Advanced.** Two parameters do specific security work and are worth being able to name: `state`
prevents CSRF on the callback by binding the response to the session that started the flow; and
**PKCE** prevents an intercepted authorisation code from being redeemed, because the attacker does
not have the original verifier. Exact redirect-URI matching matters for the same reason — a loose
match is an account-takeover vector.

#### 4. Scopes, audiences and the token chain

**Theory.** A token should grant the minimum needed: scopes limit what may be done, and audience
limits which service will accept it.

**Example.** A token issued for the orders service with `audience: orders-api` must be rejected by
the payments service. Without audience checking, any service holding a valid user token can act as
that user everywhere — so a single compromised service becomes a full compromise.

**Advanced.** In a call chain, the two models are: **propagate the user's token** downstream, so each
service authorises as the user and can do no more than they could; or **exchange** it for a
downstream-specific token (RFC 8693 token exchange). Propagation is simple and means the token's
audience must include every service, weakening the audience check. Exchange is stricter and needs
infrastructure. Know both, and note that background work triggered by a user action outlives the
token, which is why a queued job usually runs with a service identity plus a recorded "on behalf of"
(`F11`).

### Interview questions

- "Cookie or JWT for your web app? Defend it."
- "How do you revoke a JWT immediately?"
- "Why PKCE, given the client secret already exists?"
- "What does the `aud` claim prevent?"

---

## A19 · CORS

`Intermediate` · Requires: `A01` · Unlocks: `F26`, `S08`

### Preface

The browser's same-origin policy stops a page on one site from reading responses from another. CORS
is the mechanism by which a server **opts in** to being read by a named origin.

The single most important thing to understand: **CORS protects your users' browsers, not your API.**
It is not an access-control system, and an attacker with curl ignores it entirely.

### Details

#### 1. How it works

**Theory.** For a "simple" request (GET/HEAD/POST with a small set of content types and no custom
headers), the browser sends the request and then decides whether the page may **read** the response,
based on `Access-Control-Allow-Origin`. Note that the request **was still sent** — which is why CSRF
exists as a separate problem.

For anything else, the browser first sends a **preflight** `OPTIONS` request carrying
`Access-Control-Request-Method` and `Access-Control-Request-Headers`, and only proceeds if the server
approves.

**Example.** A `PUT` with `Content-Type: application/json` and an `Authorization` header triggers a
preflight. The server must answer the `OPTIONS` with `Access-Control-Allow-Origin`,
`Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, and — if credentials are involved —
`Access-Control-Allow-Credentials: true`.

**Advanced.** Set `Access-Control-Max-Age` (say 86,400) so the browser caches the preflight result
and does not send an `OPTIONS` before every request. Without it, every API call becomes two round
trips, which doubles latency for cross-origin clients — a real and frequently missed performance
problem.

#### 2. The credentials rule

**Theory.** If the request includes credentials (cookies, or `credentials: 'include'`), then
`Access-Control-Allow-Origin` **may not be `*`** — it must name a specific origin — and
`Access-Control-Allow-Credentials: true` must be present.

**Example.** So a server that wants to support several origins must echo back the specific requesting
origin — and it must do so from an **allow-list**:

```ts
const allowed = new Set(['https://app.example.com', 'https://admin.example.com']);
if (allowed.has(req.headers.origin)) {
  res.setHeader('Access-Control-Allow-Origin', req.headers.origin);
  res.setHeader('Vary', 'Origin');            // or caches will serve the wrong value
}
```

**Advanced.** Reflecting the `Origin` header **without** an allow-list is the classic vulnerability:
it means every origin is allowed, so any website can make authenticated requests as your logged-in
user and read the responses. And the `Vary: Origin` header is mandatory whenever the response varies
by origin, or a shared cache will serve one origin's `Access-Control-Allow-Origin` to another
(`A04`).

#### 3. What CORS does not do

**Theory.** CORS does not authenticate, does not authorise, and does not prevent requests from being
made — only from being **read** by scripts in a browser.

**Example.** Points to be able to make: curl, Postman, a mobile app and a server-side script ignore
CORS entirely. A simple `POST` is still delivered and still executes even if the response is blocked
— which is exactly why CSRF protection is a separate mechanism (`F26`). And a permissive CORS policy
on an API with proper authentication is not itself a vulnerability; it becomes one when combined with
cookie-based authentication.

**Advanced.** The refinement: CORS is a **browser-enforced** policy, so its security value is
protecting *your users* from malicious sites reading their data. Your API still needs authentication
and authorisation for every request, because non-browser clients exist. A candidate who says "we have
CORS configured so only our frontend can call the API" has a fundamental misunderstanding worth
correcting gently.

#### 4. Debugging

**Theory.** CORS errors are reported in the browser console and the request often looks fine in
server logs, which makes them confusing.

**Example.** The diagnostic that settles it: `curl` the endpoint. If curl works and the browser does
not, it is CORS. Then check, in order: does the `OPTIONS` preflight get a 2xx (a global
authentication guard that rejects unauthenticated `OPTIONS` requests is a very common cause — the
preflight carries no credentials); are the allowed methods and headers complete; is the origin
matched exactly including scheme and port; and is `Allow-Credentials` set if credentials are used.

**Advanced.** The preflight-blocked-by-auth problem is worth remembering specifically: the browser
sends `OPTIONS` **without** the `Authorization` header, so an authentication filter that runs before
CORS handling rejects it with 401, and the browser reports a CORS failure. The fix is to handle CORS
before authentication in the filter chain — which is a concrete instance of pipeline ordering
mattering (`F03`).

### Interview questions

- "CORS error in the browser but curl works. Explain to a junior."
- "Does CORS protect your API?"
- "Why can't you use `*` with credentials?"
- "Your preflight returns 401. What is wrong?"

---

## A20 · Payloads: compression, streaming and large responses

`Intermediate` · Requires: `A04`, `A10` · Unlocks: `C19`, `F17`

### Preface

The cheapest way to make a response faster is to make it smaller. Compression is nearly free;
returning fewer fields and fewer rows is better still.

When a response genuinely must be large, stream it rather than building it in memory — and consider
whether an asynchronous job with a download link would serve the user better.

### Details

#### 1. Compression

**Theory.** The client advertises support with `Accept-Encoding`; the server compresses and responds
with `Content-Encoding`. **gzip** is universal; **brotli** compresses text 15-20% better at similar
speed; **zstd** is fast with good ratios and is gaining support.

**Example.** Where to compress: at the CDN or load balancer for static content (it can cache the
compressed form); in the application for dynamic responses, or offloaded to the proxy. Do not
compress already-compressed content — images, video, PDFs, zip files — because you spend CPU for no
gain. Do not compress very small responses; the overhead exceeds the saving below roughly 1KB.

**Advanced.** Compression plus secrets plus attacker-controlled input equals the **BREACH** attack: if
a response contains both a secret (a CSRF token) and attacker-supplied text, the compressed size
leaks information about the secret through repeated guesses. Mitigations: do not reflect user input
into responses containing secrets, mask tokens per response, or disable compression for those
responses. Worth knowing that compression has a security dimension at all.

#### 2. Streaming responses

**Theory.** With `Transfer-Encoding: chunked` (or HTTP/2 data frames), the server can begin sending
before it knows the total size. That means constant memory regardless of response size.

**Example.** A CSV export of ten million rows, done correctly: stream from a database cursor, through
a transform that formats each row, into the response (`C19`). Memory stays flat. Done incorrectly:
build an array of ten million objects, serialise it, and send — the pod runs out of memory.
**NDJSON** (one JSON object per line) is the usual format for streaming structured data, because a
consumer can process each line without waiting for the whole document.

**Advanced.** The drawback to acknowledge: once you have sent a 200 status and started the body, you
cannot change your mind. A failure halfway produces a truncated response that looks successful.
Mitigations: send a trailer or a final sentinel line that consumers check for, or avoid the problem by
generating into object storage and returning a link (`A21`). For anything a customer relies on, the
asynchronous approach is more robust.

#### 3. Range requests

**Theory.** `Range: bytes=1000-1999` asks for part of a resource; the server replies **206 Partial
Content** with `Content-Range`. This is what makes resumable downloads and video seeking work.

**Example.** Advertise support with `Accept-Ranges: bytes`. A client whose download is interrupted
resumes from where it stopped rather than starting again. Video players use it to seek without
downloading everything before the target point.

**Advanced.** Range requests require a **strong** validator (`A04`): if the resource changes between
two range requests, the client would assemble a corrupt file, so `If-Range` with a strong ETag guards
against it. In practice, serve large static files from object storage or a CDN, which implement all
of this correctly, rather than from your application.

#### 4. Keeping payloads small in the first place

**Theory.** Before optimising transport, reduce what you send. Most large responses are large by
accident.

**Example.** The checklist: return only requested or needed fields (`F06`); paginate everything
(`A05`); avoid deeply nested expansions by default and make them opt-in; do not return large text
blobs in list endpoints; and use ids plus a separate fetch rather than embedding full related objects
where the client may not need them.

**Advanced.** This is where sparse fieldsets (`?fields=id,name,total`) and GraphQL (`A15`) come in —
both let the client control the payload. Sparse fieldsets are the cheap version and worth offering on
list endpoints for mobile clients, without adopting an entire query language. Mentioning that
middle option shows you consider proportionate solutions.

### Interview questions

- "An export endpoint OOMs the pod. Fix it."
- "Where do you compress, and what do you not compress?"
- "You started streaming and the query failed halfway. What does the client see?"
- "How do you support resumable downloads?"

---

## A21 · Long-running operations and bulk APIs

`Advanced` · Requires: `A03`, `A05` · Unlocks: `SD12`

### Preface

Some operations cannot finish inside a request: generating a report, importing a file, running a
batch. Holding an HTTP connection open for four minutes is fragile — proxies, load balancers and
mobile networks all have their own timeouts.

The pattern is to accept the work, return immediately with a way to track it, and let the client poll
or be notified.

### Details

#### 1. The 202 pattern

**Theory.** Respond **202 Accepted** with a `Location` header pointing at a resource representing the
job. The client polls that resource until it reports completion, then follows a link to the result.

**Example.**

```
POST /reports
→ 202 Accepted
  Location: /jobs/abc123

GET /jobs/abc123
→ 200 { "status": "running", "progress": 0.4 }
→ 200 { "status": "succeeded", "result": "/reports/xyz", "expiresAt": "..." }
→ 200 { "status": "failed", "error": { "code": "SOURCE_UNAVAILABLE", ... } }
```

Include `Retry-After` on the job resource so clients poll at a sensible interval rather than every
100ms.

**Advanced.** Make the job resource itself useful: progress, an estimated completion time, the
parameters it was created with, and a stable id the user can quote to support. And decide the
retention policy — how long a completed job and its result stay available — and communicate it with
`expiresAt`. Jobs that accumulate forever become a storage problem nobody owns.

#### 2. Alternatives to polling

**Theory.** Polling is simple and wasteful. The alternatives: a webhook when the job completes
(`A16`), an SSE stream of progress events, or a WebSocket.

**Example.** For a server-to-server integration, a webhook is better: the caller does not poll, and
the notification is immediate. For a user watching a progress bar, SSE gives a live update without
the client managing a polling loop. Offer polling as the baseline regardless, because it always
works and requires nothing of the client.

**Advanced.** If you offer a webhook, you now own the delivery guarantees (`A16`): retries, signing,
and the case where the receiver is down. A common and sensible compromise is "webhook plus a
pollable job resource" — the webhook is the fast path, and the job resource is the source of truth
the client can always fall back to.

#### 3. Bulk endpoints and partial success

**Theory.** A bulk operation on 1,000 items where three fail raises a question HTTP does not answer
well: what status code describes "mostly worked"?

**Example.** The pragmatic design: return **200** with a per-item result array, and let the client
inspect it.

```json
{
  "results": [
    { "index": 0, "status": "created", "id": "a1" },
    { "index": 1, "status": "failed", "error": { "code": "DUPLICATE_SKU" } }
  ],
  "summary": { "succeeded": 998, "failed": 2 }
}
```

State clearly in the documentation whether the operation is **atomic** (all or nothing) or
**per-item**. Silently applying some and not others, with no per-item report, is the worst outcome.

**Advanced.** Bulk endpoints need their own limits: a maximum batch size (so one request cannot
consume unbounded resources), a per-request timeout, and idempotency (`A09`) — if the client retries
after a timeout, the successful items must not be applied twice. A per-item idempotency key, or a
single key for the whole batch with a stored response, both work; choose and document one.

#### 4. Cancellation and expiry

**Theory.** A long job should be cancellable, and its results should not live forever.

**Example.** `DELETE /jobs/abc123` requests cancellation. The worker must actually check for a
cancellation flag between chunks — cancellation is cooperative, and a job that never checks cannot be
stopped. Results expire; generated files are removed on a schedule; and links to them are time-limited
presigned URLs (`F17`).

**Advanced.** Cancellation is also a resource-protection mechanism: a user who starts an expensive
report, navigates away, and starts another should not leave the first running. Propagating
cancellation from a disconnected client (an `AbortSignal` in Node, context cancellation in Go) is the
same idea one layer up, and it is the sort of detail that separates a considered design from a
functional one (`M04`).

### Interview questions

- "Design `POST /reports` where generation takes four minutes."
- "Bulk create with 1,000 items, three of which fail validation. What is the response?"
- "How does a client cancel a long-running job?"
- "Polling or webhook for completion? Why not both?"

---

## A22 · Contract testing and API governance

`Advanced` · Requires: `A06`, `A07`, `A14`, `M24` · Unlocks: `M35`

### Preface

Once several teams depend on your API, "we tested it manually" stops being enough. Governance is the
set of automated checks and shared conventions that keep an API consistent and stop it breaking
consumers.

The goal is that a breaking change fails **your** build, before it fails someone else's production.

### Details

#### 1. Detecting breaking changes in CI

**Theory.** Generate the API description in CI, compare it with the published version, and fail the
build on an incompatible change unless it is explicitly approved.

**Example.** Tools: `oasdiff` for OpenAPI, `buf breaking` for protobuf, and schema-registry
compatibility checks for Avro (`Q19`). The pipeline step: build → generate spec → diff against the
version in the main branch → fail on breaking changes. An override flag exists for the deliberate
case, and using it requires a reviewer.

**Advanced.** This only works if the generated description is faithful (`F25`). Validate real
responses against the schema in your integration tests, so a drift between the annotation and the
behaviour is caught too. Otherwise you are diffing documentation rather than the API.

#### 2. Consumer-driven contract tests

**Theory.** Each consumer declares what it actually uses; the provider verifies all consumer
expectations in its own pipeline (`M24`, `M35`).

**Example.** With Pact: the consumer's tests run against a mock provider and produce a contract file
describing the requests it makes and the fields it needs. The provider's pipeline replays those
contracts against the real service. Removing a field a consumer uses fails the provider's build,
naming the consumer.

**Advanced.** The value beyond catching breakage is knowing what is **safe to change**: the contracts
tell you which fields are actually consumed, so you can remove the rest confidently. Without that
data, teams either never remove anything or remove something and break a consumer they did not know
about. The cost is organisational — consumers must maintain contracts and someone must run a broker —
so it suits a handful of important consumers rather than dozens of casual ones.

#### 3. Style and consistency

**Theory.** An API where each endpoint has its own conventions is hard to use, regardless of how good
each endpoint is on its own. Automate the conventions.

**Example.** A linter such as Spectral enforces: naming (plural collections, consistent casing), that
every endpoint documents its error responses, that every list endpoint is paginated, that security
schemes are declared, that examples are present, and that descriptions exist. Run it in CI so the
review conversation is about design rather than style.

**Advanced.** Write down the conventions that a linter cannot check: the error code catalogue, the
pagination style, the versioning and deprecation policy, the idempotency requirements for writes, the
rate-limit headers. A short API guideline document, plus a service template that already follows it,
gets far better compliance than review comments (`S17`).

#### 4. Publishing and change management

**Theory.** Consumers need to know what changed, when, and what they must do.

**Example.** What good looks like: a versioned specification published as an artefact; a changelog
entry for every change, categorised as added, changed or deprecated; `Deprecation` and `Sunset`
headers on the API itself (`A06`); per-consumer usage metrics so you know who is affected; and
generated client SDKs released alongside.

**Advanced.** The metric worth tracking is **time from deprecation announcement to zero usage**. If it
is measured in years, your deprecation process is not working and you will accumulate versions
forever. That number turns "we should deprecate this" into a measurable process, and quoting it is a
strong signal of having operated an API at scale rather than only built one.

### Interview questions

- "How do you guarantee you never break a consumer, without integration-testing against all of them?"
- "What does a contract test tell you that a schema diff does not?"
- "How would you enforce API consistency across ten teams?"
- "What metric tells you your deprecation process works?"
