[← back to the field index](README.md)

# Security · Part 4 — Real-life production problems

Nodes `S01`–`S17`, applied.

Parts 1–3 are the textbook. This part is the pager. Every question below is a scenario an
interviewer narrates the way it actually happened: a symptom, some numbers, a stack, and a clock
running. The answers they are listening for are rarely "add validation" or "use a WAF"; they are
the things you only know after being on the other end of an incident — the parser that disagrees
with the proxy, the revocation list that is itself the outage, the forensic evidence you destroy by
"fixing" the box.

How to use it:

1. Read the **Pre-knowledge** once, properly. It is written so that every question can be answered
   from it, and it goes deeper than Parts 1–3 on purpose.
2. For each question, answer **out loud** for two to three minutes before reading the direction
   line. Structure: what is actually happening, how you would confirm it, what you do in the next
   hour, what you change so it cannot recur.
3. Then read the `> **Direction:**` line. It names the non-obvious insight and the section it comes
   from. If you missed it, reread that section, not just the line.

Levels run from the questions almost every senior loop contains (Level 1) to the rare, layered
problems that only come up with a staff-level or security-adjacent panel (Level 10).

Everything here is defensive: mechanisms and mitigations, not exploit recipes.

---

## Pre-knowledge

### 1. JWTs in production: what actually goes wrong (`S03`, `S04`)

**The verification contract.** A JWT is only as safe as the verifier. A correct verifier pins the
**algorithm** from its own configuration (never from the token header), selects the key by `kid`
from a **trusted, pre-loaded** key set, checks the signature, then checks `exp`, `nbf`, `iss`,
`aud`, and a token-type claim (`typ`, or a custom `token_use`). Skipping any of these is a known
incident class.

- **`alg: none` and algorithm confusion.** Old libraries honoured the header's `alg`. With `none`,
  no signature was checked. With **RS256→HS256 confusion**, a verifier that was handed an RSA
  *public* key and told "verify with whatever `alg` says" would HMAC-verify using the public key
  bytes as the secret — and the public key is public. Defence: an explicit allow-list
  (`algorithms=["RS256"]`) and typed key objects rather than byte strings. Modern libraries force
  the allow-list; the bug returns when someone writes a "generic" verify helper that reads
  `header.alg` to be flexible.
- **`kid` injection.** `kid` is attacker-controlled input. Seen in the wild: `kid` used in a file
  path (path traversal to a predictable file), in a SQL lookup (injection), or as a URL. `jku`/`x5u`
  headers telling the verifier where to fetch keys are the same bug with SSRF on top. Defence: `kid`
  is only ever a lookup key into an in-memory map built from a JWKS URL in *your* config; unknown
  `kid` → reject, after at most one rate-limited refresh (below).
- **Audience confusion.** Two services trust the same IdP. A token minted for service A (low
  privilege, perhaps a public client) is replayed against service B, which checks signature and
  expiry but not `aud`. Everything verifies. This is the most common real JWT bug in microservice
  estates — far more common than `alg: none`. Also: ID tokens accepted as access tokens (same signing
  key; check `typ`/`token_use`).
- **Clock skew.** `exp` checks with zero leeway fail intermittently when node clocks drift; teams
  "fix" it with 5–10 minutes of leeway, silently extending every token's life. Keep leeway ≤ 60 s and
  fix NTP. A burst of "`iat`/`nbf` in the future" rejections from one AZ is a clock problem, not an
  attack.
- **Size.** Stuffing permissions into claims makes tokens exceed header limits (nginx default
  `large_client_header_buffers 4 8k`; many load balancers cap total headers at 16–64 KB; cookies at
  ~4 KB each). Symptom: 431/400 errors only for power users with many roles or groups. Fix: coarse
  claims in the token, fine-grained permissions looked up server-side (`S05`).

**Key rotation with JWKS caching.** Verifiers cache the JWKS (commonly 5 min to 24 h, often honouring
`Cache-Control`). The safe sequence is **publish, wait, sign, wait, retire**:

1. Add the new public key to the JWKS while still signing with the old one.
2. Wait at least the **maximum verifier cache TTL** (plus margin) so every verifier has the new key.
3. Start signing with the new key.
4. Keep the old public key published for at least the **maximum token lifetime**.
5. Remove the old key.

Skipping step 2 is the classic outage: tokens signed with a `kid` half the fleet has never seen →
401 storms for exactly one cache TTL. Verifier-side defence: **on unknown `kid`, refresh the JWKS
once**, single-flighted and rate-limited (at most once per 30–60 s per process), otherwise a flood of
garbage `kid`s turns every verifier into a DDoS amplifier against the IdP. For **emergency** rotation
(private key leaked) you cannot wait: remove the old key immediately, accept that every live token
dies, and force verifiers to refetch (config push or rolling restart) rather than waiting for TTLs.
Signing keys belong in a KMS/HSM so "leaked" is rare; the signer calls KMS `Sign` rather than holding
the private key.

**Revocation strategies**, from cheapest to strongest:

| Strategy | Time to revoke | Cost |
|---|---|---|
| Short access-token TTL (5–15 min) + refresh | ≤ TTL | Refresh traffic; revocation takes effect at refresh |
| Per-user "tokens issued before T are invalid" (`iat` watermark) | Seconds | One tiny, cacheable lookup per request; kills *all* of a user's tokens |
| `jti` deny-list with entry TTL = remaining token life | Seconds | Bounded size because entries expire; needs distribution (Redis, pub/sub) |
| Opaque tokens + introspection | Immediate | A network call per request (cache for seconds) |

Senior points: the deny-list only holds entries until the token would have expired anyway, so it is
bounded; the watermark is the right tool for "log out everywhere" and "password changed"; and **the
revocation store must fail in a chosen direction** — if Redis is down, do you accept every token
(fail open) or reject every token (self-inflicted outage)? Most choose fail open for low-risk reads
with an alert, fail closed for money movement and admin paths, and write that down. A local
in-process copy of the deny-list, fed by pub/sub, removes the per-request network hop entirely.

**Token propagation.** Forwarding the end user's token deep into the call graph means every
downstream service sees a token with a broad audience, and every sidecar that logs headers holds a
golden ticket. The better pattern is **token exchange** (RFC 8693) or a gateway-minted internal
token with a narrow `aud` per hop and a short TTL.

### 2. Sessions, cookies and refresh tokens (`S03`, `A19`)

**Cookie attributes that matter.** `HttpOnly` (no JS access), `Secure`, `SameSite=Lax` (Chromium's
default since 2020 — blocks cross-site subresource and POST sends but allows top-level GET
navigation; note Chromium's two-minute "Lax+POST" exception for freshly set cookies), `Strict`, and
the **`__Host-` prefix**, which forces `Secure`, `Path=/` and **no `Domain`** — so a sibling subdomain
(`blog.example.com`, perhaps a marketing CMS you do not control) cannot set or overwrite it. Cookie
tossing from a user-content or compromised subdomain is a real session-fixation vector that
`__Host-` shuts down. "Same-site" means same registrable domain, so `SameSite` does nothing against
a hostile sibling subdomain.

**Session fixation.** If the session ID is the same before and after login, anyone who can plant a
session ID in the victim's browser owns the session after the victim logs in. Rule: **rotate the
session identifier at every privilege change** — login, MFA step-up, role switch, impersonation
start and end. Frameworks do it on the standard login path; custom SSO callbacks, "remember me"
restores and magic-link logins are where it gets skipped.

**Refresh-token rotation and reuse detection.** Each refresh returns a new refresh token and
invalidates the old. All refresh tokens descending from one login form a **family**. If an
already-used refresh token is presented again, either an attacker or the legitimate client holds a
stale copy — you cannot tell which — so you **revoke the whole family** and force re-login. This is
the only way a stolen refresh token is ever noticed.

The production wrinkle: legitimate reuse happens. Mobile apps fire two parallel requests that both
see a 401 and both refresh; a flaky network loses the response after the server rotated; a browser
with two tabs races. Naive reuse detection then logs out real users (support tickets spike after a
mobile release). Fixes: a **grace window** (the previous token stays valid for 10–60 s and returns
the *same* successor, idempotently), and a client-side single-flight refresh lock. The grace window
is a deliberate, bounded weakening; name it as such. Store refresh tokens **hashed** server-side, so a
database read does not yield usable tokens.

**Logout that actually logs out.** With stateless JWTs, "logout" clears the client. Server-side
effect needs one of the revocation strategies in §1. For SSO, see §3 on back-channel logout.

**Where browser tokens live.** `localStorage` tokens are readable by any XSS. `HttpOnly` cookies are
not readable but are sent automatically (CSRF, mitigated by `SameSite` plus a CSRF token or a
required custom header). The pragmatic modern SPA answer is the **BFF (backend-for-frontend)**: the
browser holds only an `HttpOnly` session cookie to your own backend; OAuth tokens never reach
JavaScript. Note that XSS still lets an attacker *act* as the user from the victim's browser; the BFF
only stops token exfiltration for later offline use.

**Session binding.** Binding a session to the client IP breaks mobile users (carrier NAT, Wi-Fi to
LTE). Binding to a device key (DPoP, RFC 9449; or Chrome's Device Bound Session Credentials) makes a
stolen cookie or token useless elsewhere — the answer to infostealer malware harvesting session
cookies, which bypasses MFA entirely because the session is already authenticated.

### 3. OAuth2, OIDC and SSO edge cases (`S04`)

**`redirect_uri`.** Must be matched **exactly** against a registered list. Real failures: prefix
matching (`https://app.example.com` also matching `https://app.example.com.evil.net`), wildcard
subdomains where one subdomain hosts user content or a dangling CNAME, and open redirects on an
allowed host (the code lands on an allowed URL that then bounces it elsewhere, in the URL or
`Referer`). Authorisation codes in URLs leak through `Referer`, browser history and analytics
scripts on the landing page; exchange and strip them immediately and set `Referrer-Policy`.

**`state`** binds the callback to the browser that started the flow; without it, an attacker can
complete a login with *their* account in the victim's browser (login CSRF), so the victim then saves
their card into the attacker's account. **`nonce`** (OIDC) binds the ID token to the request and
stops replay. **PKCE** (`code_verifier`/`code_challenge`, method `S256`) makes a stolen code useless
without the verifier; mandatory for public clients and recommended for all in OAuth 2.1. Common
mistakes: accepting `plain`, not enforcing PKCE when the client simply omits it (downgrade), and
storing `state` in one shared slot so two concurrent logins in two tabs clobber each other (users see
random "invalid state" errors — a reliability bug, not an attack; key `state` storage by the value).

**Mix-up and IdP confusion.** With multiple IdPs, a client that does not track *which* IdP it sent
the user to can be tricked into sending one IdP's code to another's token endpoint. Defence: per-IdP
`state` and checking the `iss` parameter in the authorisation response (RFC 9207).

**Implicit and ROPC flows** are deprecated: implicit puts access tokens in URL fragments; password
grant trains users to type passwords into third parties and cannot do MFA.

**SSO edge cases that page people.**
- **Email as identity key.** Linking accounts by email across IdPs is an account-takeover path: an
  IdP that lets users set an unverified email, or a social provider accepting any email, hands over
  the existing account. Key accounts on `(iss, sub)`; treat email as an attribute; link only on an
  explicit, authenticated user action; check `email_verified`; for enterprise tenants, trust emails
  only in domains verified for that tenant. Also: `sub` is only unique **per issuer** (and for some
  providers per client — pairwise identifiers), so never key on `sub` alone.
- **Deprovisioning lag.** An employee removed from the corporate IdP keeps their app session for
  the refresh-token lifetime (often 30–90 days). Fixes: SCIM deprovisioning, OIDC back-channel
  logout, shorter maximum session age for enterprise tenants, periodic re-validation against the IdP.
- **SAML.** XML signature wrapping (the signed element is not the element the app reads), accepting
  unsigned assertions when the signature element is simply absent, and canonicalisation differences
  between XML libraries. Use a maintained library; require signed assertions **and** check that the
  signed element's ID is the one consumed; validate `Audience`, `Recipient`, `NotOnOrAfter`; keep an
  assertion-ID replay cache for the assertion's lifetime. IdP-initiated SAML has no `state`
  equivalent, so it is inherently more replayable and login-CSRF-prone; prefer SP-initiated.
- **IdP outage.** If the IdP is down, nobody can log in; if JWKS fetches fail and caches are short,
  nobody can even *use* existing sessions. Cache JWKS with a long stale-if-error window; keep a
  break-glass admin path that does not depend on the IdP (hardware-key protected, heavily audited,
  alarmed on use).
- **Tenant confusion in multi-tenant IdPs** (e.g. Entra ID's `common` endpoint): the token verifies,
  but the issuer/`tid` is a tenant you never onboarded. Validate the tenant claim against your
  mapping, not just the signature.

### 4. Authorisation at scale: IDOR, drift and mass assignment (`S05`, `S09`, `M29`)

**IDOR is a structural problem.** Every endpoint that fetches by ID and forgets the ownership check
is a bug; with 400 endpoints, some will. Structural fixes, in increasing strength:
- Repository methods that **require** a tenant/owner scope (`findForTenant(tenantId, id)`); no
  unscoped `findById` reachable from application code (enforced by lint or by not exposing it).
- A request-scoped tenant context set once by middleware from the verified token and applied
  automatically (ORM default scopes / global query filters), with explicit, audited escape hatches.
- **Row-level security** in the database (Postgres RLS with a per-transaction `app.tenant_id`).
  Beware connection pools: the setting must be `SET LOCAL` inside the transaction or reset on
  checkout, otherwise the next request on that connection inherits the previous tenant. Also: RLS is
  bypassed by table owners and superusers — the application role must not own the tables (`DB37`).
- Random IDs (UUIDv4/ULID) reduce *discovery*; they are not authorisation. Sequential IDs make
  scraping trivial and leak business volume.

**Finding IDOR you already have.** Two high-yield techniques: (1) **differential testing** in CI —
for each endpoint, call it as user A with user B's resource ID and assert 403/404; generate the
cases from the OpenAPI spec so new endpoints are covered automatically. (2) **Production detection**
— log `(actor_tenant, resource_tenant)` on every data access and alert when they differ outside
known cross-tenant paths. The second catches bugs the first never thought to test.

**404 versus 403.** Return 404 for resources the caller may not know exist, to avoid confirming
existence; be consistent; log the real reason internally.

**Authorisation drift.** Roles accumulate permissions, services add ad-hoc checks, and the real
policy lives in 30 places. Symptom: nobody can answer "who can refund over £1,000?". Remedies:
centralise decisions in a policy engine or shared library (OPA, Cedar, or one module) with
**decision logging**; run periodic access reviews from that log (who used what in 90 days; remove
the rest); treat permission changes as code with review. Beware **caching authorisation decisions**:
a permission cached for 15 minutes means a revoked admin stays admin for 15 minutes — tie cache
invalidation to role-change events, or keep sensitive checks uncached.

**Mass assignment.** Binding request JSON directly onto a model lets a client set fields it should
not (`role`, `tenant_id`, `email_verified`, `price`). Defence: explicit input DTOs / allow-lists per
endpoint, never "update all fields present". Subtle variants: a `PATCH` accepting nested objects
that re-parents a child into another tenant; GraphQL input types reused between user and admin
mutations; and an internal field added to the model later becomes writable through every endpoint
that uses a deny-list.

**Business-logic authorisation.** Checks at step 1 not re-checked at step 3 (price validated on the
cart page and trusted on confirm); race conditions (a single-use voucher redeemed by 20 parallel
requests — fix with a conditional update or unique constraint, not "check then write"); batch
endpoints that check only the first item; GraphQL resolvers that authorise the top-level object but
not nested edges (`order → customer → allOrders`).

**Indirect access paths.** Exports, search indexes, analytics replicas, webhooks, email
notifications, generated PDFs and file URLs are read paths that often bypass the main authorisation
layer. A search index built without tenant filtering, or a pre-signed object URL with a seven-day
expiry pasted into a support ticket, is an IDOR with extra steps.

**Admin and support tooling** is the least-reviewed, most-privileged surface. Impersonation must be
logged as "support agent X acting as user Y", time-boxed, reason-coded, and ideally approved; it
must not be usable to change credentials or payout details.

### 5. SSRF, cloud metadata and DNS rebinding (`S08`, `S15`)

**Why SSRF is serious in the cloud.** The instance metadata service (`169.254.169.254`; also
`fd00:ec2::254` on AWS IPv6) returns temporary IAM credentials for the instance role. An SSRF in any
"fetch this URL" feature (webhooks, link previews, PDF/HTML renderers, image import, SVG processing,
XML external entities) becomes cloud credential theft. The Capital One breach (2019) had this shape.

**IMDSv2** requires a `PUT` to obtain a session token plus a header on every request, and the token
response has a default IP hop limit of 1, so traffic from an extra network hop (a bridged container)
cannot use it unless the limit is raised. Most SSRF primitives cannot issue a `PUT` with custom
headers, so IMDSv2 defeats the common case. Enforce `HttpTokens=required` fleet-wide, but first watch
the `MetadataNoToken` metric to find old SDKs that still use v1 — enforcing blind breaks them. For
containers, prefer pod-level identity (IRSA/EKS Pod Identity, GKE Workload Identity) and block pod
access to node metadata with network policy. GCP and Azure metadata require a
`Metadata-Flavor: Google` / `Metadata: true` header — useful, but a header-controlled SSRF still
reaches them.

**Why URL validation fails.** Blocklists of "internal" hosts are defeated by alternative IP
encodings (decimal, octal, IPv6-mapped IPv4, `0.0.0.0`), redirects (validate the first URL, follow a
302 to an internal one), **DNS rebinding** (the hostname resolves to a public IP at validation time
and to an internal one at connect time, TTL 0), and parser differentials between the validator's URL
parser and the HTTP client's.

**The defence that works**: resolve **once**, validate the **resolved IP** against deny ranges
(RFC 1918, loopback, link-local `169.254.0.0/16`, CGNAT `100.64.0.0/10`, IPv6 ULA `fc00::/7` and
link-local, the metadata addresses, and your own VPC CIDRs), then **connect to that exact IP**
(keeping the original `Host`/SNI), with redirects disabled or re-validated per hop. Better: move all
outbound user-directed fetching to an **egress proxy** (Smokescreen-style) in an isolated segment
with no route to internal services or metadata, so application code cannot get it wrong. Cap
response size and time, restrict schemes (`http`/`https` only — no `file:`, `gopher:`, `dict:`),
and never return raw fetched content or detailed errors (blind SSRF still maps your network through
timing and error differences).

**DNS rebinding against internal tools.** Admin UIs and debug ports on `localhost` or private IPs
can be reached from an employee's browser via a rebinding domain. Defence: validate the `Host` header
against an allow-list on internal services, and require authentication even on "local-only" ports.

### 6. HTTP at proxy boundaries: smuggling, desync, cache deception (`S06`, `A01`, `A04`)

**Request smuggling / desync.** When a front proxy and a back-end server disagree about where one
request ends — typically `Content-Length` versus `Transfer-Encoding: chunked`, or obfuscated or
duplicated headers — bytes of one client's request become the start of the next request on a
**reused back-end connection**. Consequences: another user's request gets a prefix injected, cached
responses get poisoned, or responses are delivered to the wrong client (response queue poisoning).
HTTP/2-to-HTTP/1.1 downgrade at the edge adds variants (H2.CL, H2.TE) because HTTP/2 frames carry
their own lengths.

The operational signature is weird: **intermittent** 400s or wrong responses, users occasionally
seeing another user's data, correlated with connection reuse and often with a recent proxy or
server upgrade. Defences: normalise at the edge and **reject ambiguous requests** (both CL and TE,
duplicate CL, malformed chunking) rather than "fixing" them; HTTP/2 end-to-end where possible;
current proxy and server versions; and, as an emergency mitigation, **disable back-end connection
reuse** — it costs latency and connection churn, but a desync needs a shared connection to hurt
anyone else.

**Web cache deception.** A request for `/account/settings/x.css` is served by the app as the
dynamic account page (the framework ignores the suffix), but the CDN caches it because the extension
looks static. Anyone fetching that URL gets the victim's page. Variants use delimiters that the CDN
and origin parse differently (`;`, `%2F`, `%3F`, encoded dots, path normalisation). Defences: the
origin sends `Cache-Control: no-store, private` on every authenticated response and the CDN
**honours origin headers** instead of overriding by extension; cache rules keyed on explicit static
path prefixes and content type; strict routing that 404s unknown suffixes.

**Web cache poisoning** is the inverse: unkeyed inputs (`X-Forwarded-Host`, `X-Forwarded-Proto`,
unusual headers, query parameters excluded from the key) change the response, which is then cached
for everyone. Rule: anything that affects the response is in the cache key, or is stripped at the
edge. Caching error responses (a 500 or a 403 cached for 10 minutes) is a self-inflicted variant.

**Trusted-header spoofing.** Apps behind a proxy trust `X-Forwarded-For`, `X-Real-IP` or
`X-Forwarded-User`. If the app is ever reachable directly (a debug port, a second ingress, a
security group opened "temporarily"), clients set those headers themselves; IP rate limits and
"internal-only" checks break. Fix: the edge **overwrites**, never appends, identity headers; the app
reads the client IP at a fixed hop count from the right, from known proxy IPs only; and the app's
network accepts traffic only from the proxy. A common bug: rate limiting on the **leftmost**
`X-Forwarded-For` entry, which the client controls.

**Host header.** Password-reset links built from the request `Host` let an attacker trigger a reset
email pointing at their domain (the victim clicks, the token leaks). Build absolute URLs from
configuration, never from `Host`.

### 7. Injection and parser differentials (`S07`)

Parameterised queries solve classic SQL injection; production incidents come from the leftovers:
- **Dynamic identifiers** (`ORDER BY ${column}`, table names) cannot be parameterised — allow-list.
- **ORM escape hatches** (`raw()`, `extra()`, string-built `whereRaw`) and JSON-path operators with
  input-derived paths.
- **NoSQL operator injection**: a JSON body `{"password": {"$ne": null}}` passed straight into a
  Mongo query. Fix: validate *types* (a password is a string), not just presence.
- **Second-order injection**: data stored safely, later concatenated into a query or shell command
  by a batch job that "only processes trusted internal data".
- **Log injection and log interpretation**: CR/LF in input forging log lines; and (Log4Shell, 2021)
  log messages being *interpreted* by the logging library. Structured JSON logging neutralises
  forging; the Log4Shell lesson is that any component that interprets data is an injection sink.
- **Template injection** (user-provided templates rendered server-side) and **CSV/formula
  injection** in exports opened in spreadsheets (cells starting `=`, `+`, `-`, `@`): prefix a quote.
- **Parser differentials**: two components parse the same input differently — JSON parsers
  disagreeing on duplicate keys (`{"role":"user","role":"admin"}`, first-wins versus last-wins), a
  WAF and an app decoding URLs differently, Unicode normalisation (NFKC turns fullwidth characters
  into ASCII *after* validation), case-folding in usernames (`admin` versus `ADMİN`). The rule:
  validate the **exact** representation that will be used, after normalisation, in the component
  that uses it — and reject duplicates and ambiguity rather than resolving them.

**Deserialisation.** Native object deserialisation of untrusted data (Java serialisation, Python
`pickle`, YAML object tags, .NET `BinaryFormatter`) is remote code execution by design. Use data
formats (JSON) with explicit schemas; if a legacy path must stay, sign the blob and verify **before**
deserialising. Watch for it in unexpected places: session stores, cache values, message queues and
"remember me" cookies.

**File uploads.** Validate by content, not extension or `Content-Type`; re-encode images (strips
metadata, including GPS in EXIF, and polyglot payloads); serve user content from a **separate
registrable domain** (`usercontent-example.net`) with `Content-Disposition: attachment` and
`X-Content-Type-Options: nosniff`; process in a sandbox (image and document libraries are a rich
source of memory-safety bugs); and bound decompressed size (zip bombs, decompression bombs in PNG).

### 8. Passwords, credential stuffing and bots (`S02`, `S13`, `A08`)

**Hash cost versus login DoS.** Argon2id (OWASP minimum: m=19 MiB, t=2, p=1), bcrypt (cost 10–12;
cost 12 is roughly 250 ms on a typical server core) or scrypt. The cost is the point, and it is an
attack surface: 1,000 attempts/s at 250 ms of CPU each needs ~250 cores. Credential stuffing
therefore becomes a **CPU-exhaustion** incident before it becomes an account-takeover incident.
Defences: rate limit and bot-screen **before** hashing; cap concurrent hash operations with a
bounded pool so logins queue instead of starving the API; isolate auth in its own autoscaling pool;
and never lower the cost as "the fix". Memory-hard Argon2 has the mirror problem: 19–64 MiB per
concurrent hash can OOM a pod under a burst, so the concurrency cap is mandatory. Bcrypt truncates
input at 72 bytes; pre-hashing to lift that must encode the digest (base64) to avoid NUL bytes.

**Upgrading hashes** without a mass reset: rehash on successful login (`needsRehash`); for users who
never log in, **wrap** the old hash (`argon2(legacy_md5)`) so the weak form never sits at rest, and
migrate to a pure hash on their next login.

**User enumeration** leaks through different messages, status codes and **timing**: an unknown user
returns in 5 ms, a known one in 250 ms because only it runs the hash. Fix: hash against a dummy value
for unknown users; same message; for sign-up and reset, respond "if the account exists, we have sent
an email", and do the slow work asynchronously so timing is equal.

**Credential stuffing** is low-and-slow from residential proxies: hundreds of thousands of IPs, one
or two attempts each, realistic browsers. Per-IP limits barely help. Effective signals and controls:
- **Success-rate anomaly**: normal login success is roughly 60–90 %; a stuffing wave drops the
  global rate to 1–5 %. Alert on the ratio and on failures-per-distinct-username, not raw volume.
- Limits per **username** (protects the target), per **IP / subnet / ASN**, per **device/TLS
  fingerprint**, and a global failure budget that turns on stricter controls automatically.
- **Breached-password checks** at login and sign-up (k-anonymity range queries sending only a
  5-hex-character hash prefix); on login with a known-breached password, force a reset.
- Step-up (CAPTCHA, proof-of-work, MFA) only when risk is elevated, to protect conversion.
- **Do not hard-lock accounts** in consumer products: lockout is itself a DoS tool. Prefer
  progressive delays per username plus step-up.
- Post-login friction: new-device notifications, and step-up for sensitive actions (change email or
  phone, add payee, export data) — the attacker who guessed right still has hoops.

**Bot mitigation** is economics: raise the cost per attempt above the value of success. Tools:
device and TLS fingerprinting (JA3/JA4), behavioural signals, proof-of-work, and **silent shadow
responses** — confirmed bots receive plausible failures rather than 429s, so they cannot tune
against you. Attackers adapt within hours to any visible signal; the less feedback, the longer a
control lasts. Every control needs a kill switch, and false positives are measured through
conversion metrics, not complaints.

**Other abuse patterns.** SMS pumping / toll fraud (OTP sends to premium-rate number ranges — cap
per country prefix, per number, and a global daily spend with an alarm); sign-up abuse for free
credits; password-reset flooding; enumeration via "username available" checks; gift-card and
coupon brute force (make codes long and rate limit by account, not just IP); and scraping via
legitimate accounts (per-account quotas and behavioural baselines).

**MFA edges.** TOTP codes need replay protection (reject a code already used in its window) and
rate limits — six digits at unlimited guesses falls in minutes. Push fatigue (spamming prompts until
the user accepts) → number matching and prompt rate limits. SMS is SIM-swappable. Recovery flows
(support-desk resets, backup codes, "lost my phone") are usually the weakest link and where
social-engineering attacks land. WebAuthn/passkeys are phishing-resistant because the origin is bound
into the signature.

### 9. Timing and side channels (`S11`)

Comparing a secret (HMAC signature, API key, reset token) with `==` returns at the first differing
byte. Over a network the difference is nanoseconds, but repeated measurement and statistics can
recover it, particularly from within the same cloud region. Use constant-time comparison
(`hmac.compare_digest`, `crypto.timingSafeEqual`, `MessageDigest.isEqual`) — and compare fixed-length
digests, since some constant-time functions throw or leak on length mismatch.

The structural fix for looked-up secrets (API keys): split the key into a **public ID prefix** and a
secret part; look up by the ID, then constant-time compare a hash of the secret. The database index
never operates on the secret, keys are stored hashed (a database leak does not leak keys), and the
prefix makes keys greppable for secret scanning (`sk_live_…`-style formats).

Other side channels in backends: response **size** (compression of secrets alongside attacker input
— CRIME/BREACH), cache-hit timing revealing whether another tenant accessed a resource,
early-return authorisation paths that are measurably faster for non-existent resources, and error
message differences.

### 10. Secrets: leakage paths and zero-downtime rotation (`S10`)

**Where secrets actually leak**, roughly by real-world frequency:
- **Logs**: request/response body logging in debug mode, logged `Authorization`/`Cookie` headers,
  connection strings in exception messages, ORM query logs with bound parameters, and tokens in URL
  query strings (which also land in proxies, `Referer` and browser history).
- **Crash dumps and core files**: a heap dump contains every secret the process held. Heap dumps
  attached to a ticket or copied to a shared bucket for debugging are a secret leak.
- **Error trackers and APM**: local variables and request data captured automatically. Configure
  scrubbing at the SDK and at the server.
- **CI**: secrets echoed by `set -x`, printed by a failing command, exposed to pull-request builds
  from forks, or baked into images — `docker history` reveals `ARG`/`ENV` values, and a file deleted
  in a later layer still exists in the earlier one. Use build secrets (`--mount=type=secret`) and
  short-lived OIDC federation from CI to the cloud instead of stored keys. CI log masking is
  best-effort: a base64-encoded or split secret is not masked.
- **Git history**: deleting a secret in a new commit does not remove it; rewriting history does not
  remove it from forks, clones, caches, or platform scanning partners. **The only fix is rotation**;
  history rewriting is hygiene afterwards. Public GitHub leaks of cloud keys are typically exploited
  within minutes by automated scanners.
- **Environment variables**: readable in `/proc/<pid>/environ` by same-user processes, dumped by
  debug endpoints (Spring Boot Actuator `/env` and `/heapdump`, `phpinfo`), inherited by every child
  process, shown in `kubectl describe` and orchestration UIs. Mounted files or a secrets-manager fetch
  are better.
- **Client bundles and mobile apps**: anything shipped to a client is public, including "obfuscated"
  keys.

**Detection**: pre-commit and server-side secret scanning with push protection; **canary tokens**
(fake cloud keys planted in repos, wikis and config that alert when used); scanning log pipelines for
known key formats and high-entropy strings.

**Zero-downtime rotation: the dual-key pattern.** Every shared secret must be able to have **two
valid values at once**:
1. Create the new credential while the old stays valid (database: alternate between two users A/B,
   or providers that allow two passwords; API keys: two active keys; HMAC webhook secrets: sign with
   new, verifiers accept both — include a key ID in the signature header).
2. Roll it out to every consumer.
3. **Verify** that nothing still uses the old credential (last-used timestamps on cloud keys,
   connection logs on databases, provider usage data).
4. Revoke the old one.

Rotations fail because step 3 is skipped: a forgotten cron job on a pet server uses the old key and
breaks at 02:00. Also: long-lived connection pools keep sessions authenticated with the old password
until they reconnect — harmless for rotation, dangerous for revocation (killing a leaked database
password also means killing its existing sessions). **Rotate on a schedule** even without an
incident, because a rotation procedure that has never run will not work during one.

**Dynamic secrets** (Vault database engine, IAM auth to managed databases) issue per-workload
credentials with a lease of hours; leaks age out and are attributable to one workload. The trade-off
is a hard dependency on the secrets service at start-up and lease renewal — cache, and design the
failure mode.

### 11. Envelope encryption, KMS limits and crypto-shredding (`S11`, `S12`)

**Envelope encryption.** KMS holds a key-encryption key (KEK) that never leaves it. Per object or
record, generate a **data key** (DEK), encrypt the data locally with the DEK (AES-256-GCM), encrypt
the DEK under the KEK via KMS, and store the wrapped DEK beside the ciphertext. Decryption needs a
KMS call to unwrap the DEK. Use the **encryption context** (AAD) — e.g. tenant ID and record ID — so a
ciphertext moved to another row or tenant fails to decrypt, and so CloudTrail shows *which* record
each decrypt was for.

**KMS limits are an availability problem.** AWS KMS has per-account, per-region quotas on
cryptographic requests (thousands to tens of thousands per second depending on region and key type)
shared by everything in the account — S3 SSE-KMS, EBS, Secrets Manager and your application.
Symptoms: `ThrottlingException`, sudden S3 read latency, failures in unrelated services sharing the
account. Fixes: **cache unwrapped DEKs** in memory with a bounded lifetime and use count (the AWS
Encryption SDK's caching materials manager does this); one DEK per batch or per tenant per time
window rather than per row; **S3 Bucket Keys** (cut SSE-KMS calls by up to 99 %); separate accounts
for heavy workloads; request quota increases with evidence. The trade-off is explicit: cached DEKs
in memory increase blast radius if the process is compromised, and a KMS outage becomes survivable
only for the cache window.

**AES-GCM nonce limits.** With random 96-bit nonces, keep a key to roughly 2^32 encryptions; a nonce
reused under GCM leaks the XOR of plaintexts and enables forgery. High-volume encryption with one key
→ rotate DEKs, or use AES-GCM-SIV (nonce-misuse resistant) or XChaCha20-Poly1305 with 192-bit
nonces. Never build your own nonce counter shared across processes without a guarantee of
uniqueness — restarts and forks reset counters.

**Rotation of encrypted data.** Rotating a KMS KEK (automatic annual rotation) does not re-encrypt
data; old key material is retained for decryption. Re-wrapping DEKs under a new KEK is cheap.
Re-encrypting data is expensive and only needed if a DEK is suspected compromised. Store a **key
ID/version** with every ciphertext so either operation is possible, and so a lazy re-encrypt-on-read
migration can run.

**Crypto-shredding for GDPR.** Deleting a person from every backup, replica, lake and event log is
impractical. Instead, encrypt each subject's personal data with a **per-subject key**; erasure =
destroy that key; every copy everywhere becomes unreadable, including immutable backups and Kafka
topics you cannot rewrite. Caveats interviewers probe: the key store's own backups must not
resurrect deleted keys (keep deletion tombstones and replay them on restore); derived data (search
indexes, analytics aggregates, ML features, logs) must not hold plaintext; millions of subjects mean
a table of wrapped keys under one KEK, not millions of KMS keys (per-key cost and quotas); and legal
holds and retention duties can override erasure for specific fields.

**KMS deletion is a one-way door.** AWS enforces a 7–30 day waiting period on key deletion precisely
because deleting a KEK is instant data loss. Alarm on `ScheduleKeyDeletion` and `DisableKey`, deny
them by SCP outside a break-glass role, and treat an attacker who can delete your keys as a
ransomware scenario ("encrypt-and-delete-the-key" is a real cloud ransomware pattern).

### 12. Privacy and personal data in practice (`S12`)

- **PII in the wrong places**: logs, analytics events, error trackers, URLs, search indexes, cache
  keys, and test fixtures copied from production. A **data map** (where each PII field flows) is the
  only way to answer an erasure or breach question in hours rather than weeks.
- **Pseudonymisation versus anonymisation**: an unsalted hash of an email is pseudonymisation and
  still personal data (hash candidate emails to re-identify). A keyed HMAC with a protected key is
  stronger; aggregation with minimum group sizes approaches anonymisation. Low-cardinality fields
  (date of birth + postcode) re-identify people even without names.
- **Breach notification**: GDPR requires notifying the supervisory authority within **72 hours** of
  becoming aware of a personal-data breach (unless it is unlikely to result in risk), and data
  subjects without undue delay when risk is high. Awareness starts the clock; scoping fast is what
  makes it survivable. Document decisions even when you decide not to notify.
- **Erasure versus retention**: financial and tax records must be kept for years (UK: 6); anti-money-
  laundering rules similar. The answer is minimisation and separation, not ignoring either duty.
- **Production data in lower environments**: mask or synthesise; a staging copy of production is a
  breach waiting for staging's weaker controls.
- **Data residency**: backups, logs, support tooling and third-party processors (error trackers,
  LLM APIs) count as processing locations.

### 13. Supply chain (`S14`, `O08`)

- **Dependency confusion**: an internal package name (`acme-utils`) absent from the public registry;
  an attacker publishes it publicly with a higher version; a resolver configured with both registries
  picks the higher one. Defences: **scoped names** (`@acme/utils`) bound to the private registry; a
  single proxying registry that never falls through to public for internal names; registering
  internal names publicly as placeholders.
- **Typosquatting** and **starjacking** (a malicious package linking a popular repository to look
  legitimate). Defence: new-dependency review, allow-lists, and a registry proxy that quarantines
  versions younger than N days.
- **Lockfile drift**: a lockfile CI does not enforce (`npm install` instead of `npm ci`; `pip
  install` without `--require-hashes`) means production resolves versions nobody reviewed. **Lockfile
  injection**: a pull request that changes a `resolved` URL or integrity hash inside a huge lockfile
  diff nobody reads. Enforce frozen installs, hash checking, and automated checks on resolved hosts.
- **Compromised maintainers and install scripts**: `postinstall` scripts run arbitrary code on laptops
  and CI with their credentials (event-stream 2018, ua-parser-js 2021, the xz backdoor 2024, recurring
  npm worm campaigns stealing tokens to republish further packages). Defences: disable install scripts
  by default (`--ignore-scripts`, allow-list exceptions); builds with no ambient cloud credentials;
  egress-restricted CI runners; a cooling-off period before adopting new versions (many compromises
  are caught within 24–72 hours); short-lived, scoped publish tokens with trusted publishing (OIDC)
  for your own packages.
- **CI/CD as the crown jewels**: third-party CI actions pinned by mutable tag (`@v3`) instead of
  commit SHA (tj-actions/changed-files, 2025, dumped secrets into thousands of repositories' build
  logs); `pull_request_target` workflows that check out untrusted code with secrets available;
  self-hosted runners shared across trust levels and persisting between jobs; cache poisoning of
  shared build caches.
- **SBOM and provenance**: an SBOM (CycloneDX/SPDX) generated at build time per artefact answers
  "which services run the vulnerable version?" in minutes during the next Log4Shell — grepping
  repositories misses transitive dependencies, vendored code and shaded jars. Signed provenance (SLSA
  levels, Sigstore/cosign) plus cluster admission control ensures only artefacts built by your
  pipeline run. Pin images by **digest**, not tag.
- **Vulnerability triage**: scanners produce thousands of findings. Prioritise by reachability (is the
  vulnerable function called?), exposure (internet-facing?), exploitation evidence (CISA KEV, EPSS
  scores) and compensating controls. "Fix every critical in 7 days" without reachability produces
  alert fatigue and ignored dashboards. Base-image CVEs are best cut by **minimal/distroless images**
  — fewer packages, fewer findings, smaller attack surface.

### 14. Containers, least privilege and workload identity (`S15`, `M26`, `O01`)

**Container escape basics.** A container is a process with namespaces and cgroups sharing the host
kernel. Escapes come from `--privileged` or dangerous capabilities (`CAP_SYS_ADMIN`), mounting the
container runtime socket or host paths, `hostPID`/`hostNetwork`, kernel vulnerabilities, and runtime
bugs (runc CVE-2019-5736, CVE-2024-21626 "Leaky Vessels"). Baseline: non-root user, read-only root
filesystem, drop all capabilities, `allowPrivilegeEscalation: false`, `seccomp: RuntimeDefault`, no
host mounts, Pod Security Admission at `restricted`. For untrusted code (customer scripts, CI for
forks, user-supplied plugins), use a sandbox with its own kernel (gVisor, Firecracker/Kata), not just
a container.

**Least-privilege IAM in practice.** Policies start broad because narrowing is hard. What works:
generate policies from observed usage (CloudTrail-based access analysis) after a burn-in period;
**permission boundaries** and **service control policies** as guardrails no role can exceed; separate
accounts/projects per environment and blast radius; explicit denies on destructive or anti-forensic
actions (`kms:ScheduleKeyDeletion`, `s3:DeleteBucket`, `cloudtrail:StopLogging`). `iam:PassRole` and
`iam:CreatePolicyVersion` are privilege-escalation primitives — a role that can pass a more
privileged role to a Lambda or EC2 instance *is* that role.

**Confused deputy.** A privileged service performs an action on behalf of a caller without checking
that the caller is entitled to that specific resource. Cloud version: a SaaS vendor assumes a role in
your account; without an **`ExternalId`** condition, another customer of that vendor could get it to
assume *your* role. Service version: an internal "file service" with broad bucket access that takes
a bucket and key from the request. Defences: propagate the original caller's identity and authorise
against it, not against the deputy's own permissions; `aws:SourceArn`/`aws:SourceAccount` conditions
for AWS service principals; `ExternalId` for cross-account vendor roles.

**Workload identity** replaces static keys: pods get short-lived cloud credentials via projected
service-account tokens and OIDC federation (IRSA/EKS Pod Identity, GKE Workload Identity, Azure
Workload Identity). Service-to-service: mTLS with SPIFFE IDs issued by the mesh, and authorisation
policy on those identities (`M26`). Watch trust policies: a role trusting "any service account in the
cluster" rather than a specific namespace/name is a lateral-movement path. CI-to-cloud OIDC must pin
the `sub` claim to repository **and** branch or environment — pinning only the organisation lets any
repository, or a pull-request workflow, assume the production role.

**Kubernetes specifics.** Service-account tokens auto-mounted into every pod (set
`automountServiceAccountToken: false` where unused); RBAC `list`/`watch` on secrets is effectively
read-all-secrets; `create pods` in a namespace lets you mount any service account in that namespace;
Kubernetes Secrets are only base64 in etcd unless encryption at rest with a KMS provider is enabled;
node-level credentials (kubelet, node IAM role) are shared by every pod on the node, which is why
node metadata must be blocked.

### 15. Detection and audit logging (`S16`, `O16`)

**Audit log properties.** Who (actor, including the impersonating support agent), what (action,
resource), when (UTC, synchronised), from where (IP, user agent, session, request ID), outcome, and
**before/after** for changes. Stored **append-only** and outside the reach of what it audits: a
separate logging account, object lock / WORM storage (S3 Object Lock in compliance mode), or a hash
chain in which each entry commits to the previous one, with the chain head periodically anchored
elsewhere, so tampering is evident. An attacker with admin in the application account must not be
able to delete the trail; an organisation trail delivered to a separate logging account achieves
that.

**Audit versus application logs.** Audit events are a product feature with a schema, written
transactionally with the change (outbox pattern) so a crash cannot record the change without the
audit or vice versa; application logs are best-effort and sampled. Never sample audit or security
logs.

**What to detect** (high signal, low volume): logins from new countries or devices followed quickly
by sensitive actions; impossible travel (noisy with VPNs and mobile carriers — use as a score, not a
page); one session touching many tenants; exports and bulk reads above an actor's baseline; IAM
changes, new access keys, logging disabled (`StopLogging`, `DeleteTrail`), KMS keys disabled; secrets
read by a principal that has never read them; cloud API calls from an IP outside your egress ranges
using a role that should only be used inside them; canary-token triggers; and **honeytokens** — fake
records (a customer that does not exist, a fake admin credential, a decoy bucket) whose access is by
definition malicious, giving near-zero false positives.

**Detection engineering discipline.** Every rule has an owner, a runbook, and a measured
false-positive rate; test rules by replaying known-bad activity; prefer a few rules that page
reliably over hundreds nobody reads; and measure time-to-detect in exercises.

**Pitfalls**: local timezones or skewed clocks (cross-system correlation becomes impossible);
request IDs not propagated across services and queues; logs containing secrets or PII so the log
store becomes the most sensitive system; and retention shorter than typical dwell time — many
breaches are discovered weeks to months after entry, so 90 days hot and a year or more cold is a
common baseline, and data-access logs (S3 data events, database audit) must be switched on *before*
you need them.

### 16. Incident response, forensics and disclosure (`S16`)

**Phases**: detect → triage and declare → contain → eradicate → recover → post-incident review. A
declared incident has one **incident commander**, a separate communications lead, a written timeline
started immediately, and a dedicated channel — and for suspected account or insider compromise, an
**out-of-band** channel: if the attacker holds your chat admin or email, your incident channel is
their intelligence feed.

**Containment without destroying evidence.** The instinct to "terminate the instance and redeploy"
destroys memory, processes, network connections and local files you need to answer "what did they
take?". Instead:
- **Isolate, do not kill**: swap to a security group allowing only forensic access, detach from the
  load balancer, remove from the auto-scaling group (and protect from scale-in, or it will be
  terminated automatically), revoke its role's sessions. For Kubernetes, cordon the node, label and
  isolate the pod with network policy, and stop the deployment from replacing or reaping it.
- **Capture volatile state first** (order of volatility): memory, processes, network connections,
  then disk. Snapshot volumes; for containers, capture the image digest, the container filesystem
  diff, and preserve the node. Record hashes and a chain of custody for anything that may go to law
  enforcement, insurers or court.
- **Revoke in the right order**: revoking the leaked key before understanding persistence can leave
  the attacker's newly created keys, users, roles, OAuth apps or cron jobs in place and tell them
  they are discovered. Enumerate everything the credential did (CloudTrail by access key ID,
  including `CreateAccessKey`, `CreateUser`, `UpdateAssumeRolePolicy`), then remove persistence and
  revoke **together** in one coordinated step. For AWS role sessions, deleting a key does not kill
  already-issued session tokens — attach a deny policy conditioned on `aws:TokenIssueTime` ("revoke
  active sessions").
- **Preserve logs** past normal retention for the incident window, and export them somewhere the
  compromised principal cannot reach.
- **Do not tip off** in insider cases: coordinate with HR and legal before visible actions.

**Scoping** drives everything: access vector; blast radius (what the compromised principal *could*
reach, from IAM and network — not what you hope it touched); what the evidence shows *was* touched;
and whether the attacker is still present. Assume anything readable was read unless logs prove
otherwise.

**Recovery**: rebuild from known-good sources rather than cleaning hosts; rotate every secret the
compromised system could reach (including secrets in its environment and its CI); watch for re-entry
with the same technique; and verify backups were not tampered with before restoring from them.

**Communication and disclosure**: legal and privacy decide notification (GDPR 72 h; sector regulators;
enterprise contracts often 24–72 h). Statements must be accurate and specific, never speculative, and
do not publish details that help an attacker still inside. **Coordinated disclosure** for inbound
reports: a `security.txt` (RFC 9116), a triage SLA, a safe-harbour statement for good-faith research,
fix then disclose on an agreed timeline (90 days is the common norm). A report arriving with a
payment demand ("pay or we publish") is an extortion incident, not a bug bounty — engage legal, and
treat the described access as a live breach.

**Blameless post-incident review** asks why the system made the mistake easy and which detection
should have fired earlier; the output is a small number of owned, dated actions.

### 17. Security in the SDLC and programme trade-offs (`S17`, `S01`)

- **Paved roads beat policing**: a shared auth middleware, an SSRF-safe HTTP client by default, a
  secrets SDK, a logging library that redacts by default, a tenant-scoped repository base class.
  Engineers use the safe thing when it is also the easy thing.
- **Security review focus**: trust boundaries, authorisation on each new endpoint, new PII flows, new
  outbound calls, new dependencies, and any change to crypto, auth, CORS, cookies or proxy config.
  Diff size is irrelevant: a one-line CORS or cache-rule change can matter more than a 2,000-line
  feature.
- **Scanners** (SAST/DAST/SCA) need triage owners and suppressions with expiry dates, or they are
  ignored.
- **Shadow mode before enforcement**: every new blocking control (WAF rule, bot defence, stricter
  validation, IMDSv2 enforcement, a new authorisation policy) runs log-only first, measuring what it
  *would* block; then enforce behind a flag with a kill switch.
- **CORS** is not authorisation: reflecting the `Origin` header with
  `Access-Control-Allow-Credentials: true` lets any site read authenticated responses; the `null`
  origin must never be allow-listed (sandboxed iframes and `file:` send it); regex allow-lists fail
  on unanchored patterns.
- **Security headers that matter for APIs and apps**: `Strict-Transport-Security` (with care before
  `includeSubDomains; preload` — preload is effectively irreversible for months), a
  `Content-Security-Policy` rolled out via `Content-Security-Policy-Report-Only` first,
  `X-Content-Type-Options: nosniff`, `frame-ancestors` for clickjacking.
- **Risk acceptance** is a legitimate output: a documented, time-boxed decision by an owner with
  authority, revisited on a date. Seniors are expected to make and record these trade-offs rather
  than block delivery or ignore risk.

### 18. Numbers worth knowing

| Thing | Typical value |
|---|---|
| Access-token TTL | 5–15 min |
| Refresh-token lifetime | 7–90 days, sliding, with an absolute maximum |
| Refresh reuse grace window | 10–60 s |
| JWKS cache | 5 min – 24 h; refresh on unknown `kid` at most once per 30–60 s |
| JWT clock leeway | ≤ 60 s |
| bcrypt cost 12 | ~250 ms per hash per core |
| Argon2id OWASP minimum | m=19 MiB, t=2, p=1 |
| Normal login success rate | ~60–90 %; stuffing drops it to ~1–5 % |
| AES-GCM with random 96-bit nonces | ≤ ~2^32 messages per key |
| KMS key deletion waiting period | 7–30 days |
| GDPR breach notification | 72 h to the supervisory authority |
| Coordinated disclosure norm | 90 days |
| Cookie size limit | ~4 KB each |
| nginx default large header buffers | 4 × 8 KB |
| IMDSv2 default hop limit | 1 |
| Security log retention baseline | 90 days hot / 1 year+ cold |

---

## Questions

### Level 1 — Everyday access-control and auth bugs

The problems that turn up in almost every senior loop: IDOR, token checks, and the authorisation mistakes that ship weekly.

**1.** A customer emails support saying that by changing `/invoices/10482` to `/invoices/10483` in the address bar they can see another company's invoice. You have 400 endpoints across 12 services, and the team wants to "fix this endpoint and move on". What do you do today, and what do you do this quarter?

> **Direction:** Fix the endpoint, but treat it as evidence of a structural gap: tenant-scoped repositories or RLS so unscoped lookups cannot run, plus OpenAPI-generated cross-tenant differential tests in CI and production `(actor_tenant, resource_tenant)` mismatch alerts to find the siblings (§4; `S05`, `S09`).

**2.** A penetration tester reports that your `PATCH /users/me` endpoint lets them set `"role": "admin"` in the JSON body. The handler does `user.update(request.json)`. The team proposes adding `role` to a deny-list. Why is that the wrong fix?

> **Direction:** Deny-lists fail open for every field added later (`email_verified`, `tenant_id`); use per-endpoint input DTOs or allow-lists and hunt every "bind the whole body" pattern, including nested objects and shared GraphQL input types (§4; `S09`).

**3.** Two internal services, billing and notifications, both trust your IdP's JWTs. A security review finds that a token issued to the public mobile app can call billing's admin API directly. Signature, expiry and issuer all check out. What is missing, and how would you find every other service with the same gap?

> **Direction:** Audience and token-type validation: billing must require its own `aud`; audit every verifier config, and centralise verification in one shared library that cannot disable those checks (§1; `S03`, `S04`).

**4.** Your SPA stores the access token in `localStorage`. Marketing adds a third-party chat widget, and the security team panics. The product manager asks whether moving the token to a cookie "fixes it". Give a precise answer.

> **Direction:** A cookie stops token exfiltration but not script acting in the user's browser; the real answer is a BFF with `HttpOnly` `__Host-` cookies plus CSRF defence, a CSP rolled out report-only first, and treating the widget as code with full page access (§2, §17; `S03`, `S08`).

**5.** A user deletes their account; support later finds that a PDF invoice link from an old email still downloads the file. The link is a pre-signed object-storage URL. How did this happen, and what is the general lesson?

> **Direction:** Pre-signed URLs are bearer tokens that bypass the app's authorisation layer; generate short-expiry links on demand behind an authorised endpoint, and inventory the other indirect read paths (exports, search indexes, email links) (§4; `S09`).

**6.** An endpoint returns 403 for invoices that exist but belong to someone else, and 404 for IDs that do not exist. A researcher uses this, plus sequential IDs, to estimate your customers' invoice volume. Is it a vulnerability, and what do you change?

> **Direction:** Yes, it is an existence oracle compounded by enumerable IDs; return a consistent 404, log the real reason internally, and use non-enumerable IDs for discovery resistance while remembering they are not authorisation (§4; `S09`).

**7.** Your admin console has an "impersonate user" button for support. An audit shows 3,000 impersonation sessions last month, some lasting 8 hours, and the audit log records only the customer as the actor. What is wrong and what do you design instead?

> **Direction:** Impersonation must be logged as "agent X as user Y", time-boxed, reason-coded, possibly approved, with session rotation at start and end, and barred from credential and payout changes — admin tooling is the most privileged, least reviewed surface (§2, §4, §15; `S05`, `S16`).

**8.** A £50 voucher code is single-use. Finance notices one code was redeemed 17 times within 200 ms. The code checks `if not voucher.used:` and then sets `used = True`. What happened, and how do you fix it without adding a distributed lock?

> **Direction:** A check-then-write race; make redemption one conditional update (`... WHERE id = ? AND used = false`, check rows affected) or a unique constraint on redemptions, so the database arbitrates atomically (§4; `S09`).

**9.** Users with many directory group memberships get a blank page with HTTP 431 after logging in via SSO. Ordinary users are fine. What is going on?

> **Direction:** Groups stuffed into the token or cookie exceed header limits (nginx 4 × 8 KB buffers, ~4 KB per cookie, load-balancer caps); keep coarse claims in the token and resolve fine-grained permissions server-side instead of raising buffers indefinitely (§1, §18; `S03`, `S05`).

**10.** A developer writes a "verify any JWT" helper that reads `alg` from the token header so one function supports both HS256 service tokens and RS256 user tokens. Code review approves it. Why should the review have failed?

> **Direction:** Header-driven `alg` enables `none` and RS256→HS256 confusion, where the public key becomes an HMAC secret; pin allowed algorithms per key with typed key objects, and keep separate verifiers per token type (§1; `S03`).

**11.** Your GraphQL API checks that the caller owns `order(id)`. A researcher shows that following `order → customer → orders` returns every order for that customer, including ones on accounts the caller must not see in your B2B model. Where is the flaw?

> **Direction:** Authorisation only at the top-level resolver, not on nested edges; enforce it in the data-access layer or per-type loaders so every path to an object is checked, and cap query depth and complexity (§4; `S05`, `S09`).

**12.** A bulk endpoint `POST /documents/archive` takes an array of 500 IDs. Tests confirm a caller cannot archive someone else's single document, but a bug bounty report shows they can if it is the second item in the array. How do you fix and prevent this class?

> **Direction:** The batch handler authorised only the first item; authorise each item within one scoped query (`WHERE id IN (...) AND tenant_id = ?`, then compare counts), and include batch variants in the cross-tenant differential tests (§4; `S09`).

**13.** Password-reset emails for some users contain links to an attacker's domain with a valid token. No mail system was compromised. What is the likely bug?

> **Direction:** Reset URLs built from the request `Host` or `X-Forwarded-Host`; build absolute URLs from configuration and invalidate outstanding reset tokens after the fix (§6; `S06`).

**14.** A "delete account" feature returns 200 immediately, but three months later the user complains their profile still appears in partner search results. The primary database row is gone. What was forgotten, and how should deletion be designed?

> **Direction:** Derived copies (search index, caches, analytics, partner feeds) are separate data paths; you need a data map and a fan-out deletion workflow with verification, or crypto-shredding for copies you cannot edit (§4, §11, §12; `S12`).

**15.** Your login page says "No account with that email" versus "Wrong password". You unify the message, but a tester still enumerates accounts in minutes. How?

> **Direction:** Timing: unknown users skip the ~250 ms password hash; hash against a dummy value for unknown users, and make sign-up and reset respond identically with slow work done asynchronously (§8, §9; `S02`).

**16.** To silence browser errors, a developer made the API reflect the request `Origin` header and send `Access-Control-Allow-Credentials: true`. The API uses cookie sessions. What is the impact?

> **Direction:** Any website can make credentialed requests and read the responses — a cross-origin data leak for every logged-in user; use an exact origin allow-list, never reflect or allow `null`, and remember CORS relaxes browser protection rather than adding authorisation (§17; `S08`, `A19`).

**17.** Your multi-tenant Postgres uses row-level security with `SET app.tenant_id = ?` at the start of each request. Once or twice a week a customer sees another tenant's dashboard for one page load. No deploy correlates. What is the most likely cause?

> **Direction:** A session-level `SET` leaking across pooled connections when a request path skips the reset; use `SET LOCAL` inside the transaction or reset on checkout, and ensure the app role does not own the tables, since owners bypass RLS (§4; `DB37`, `M29`).

**18.** A user reports being logged into someone else's account after clicking a link on a forum. The URL contained `;jsessionid=...`. The app is a legacy Java service behind your new gateway. Explain the mechanism and the fix.

> **Direction:** Session fixation through a URL-embedded session ID; disable URL session tracking, use cookie-only sessions, and rotate the session ID at login and every privilege change (§2; `S03`).

**19.** A new internal "report service" is only reachable inside the VPC, so the team skipped authentication. A container in an unrelated service was compromised last week. What is your concern, and what is the minimum that service should have had?

> **Direction:** Network position is not identity; any compromised workload can call it, so require workload identity (mTLS/SPIFFE or narrow-`aud` service tokens) plus network policy restricting callers (§1, §14; `S01`, `S15`, `M26`).

**20.** After a role change, a demoted admin could still approve refunds for 15 minutes. The permission service sits behind a cache "for performance". The team suggests a 1-minute TTL. What is the better answer?

> **Direction:** Invalidate cached decisions on role-change events (or carry a permission version checked against a cheap watermark) and keep high-risk actions uncached, rather than trading correctness for a TTL (§1, §4; `S05`).

### Level 2 — Sessions, tokens and revocation in production

Common operational problems with JWTs, refresh tokens, key rotation and logout.

**1.** Security asks for "log out everywhere" after a password change. You use 1-hour stateless JWTs verified locally by 60 services, and you cannot add a network call per request. What do you build?

> **Direction:** A per-user `iat` watermark ("tokens issued before T are invalid") pushed via pub/sub into an in-process cache on each verifier, plus shorter access-token TTLs; bounded, cheap, and no per-token tracking (§1; `S03`).

**2.** You rotated the JWT signing key at 14:00. From 14:00 to 14:10 about half of API requests returned 401, then everything recovered by itself. What went wrong, and what does the correct runbook look like?

> **Direction:** Signing started before verifiers' 10-minute JWKS caches held the new key; publish, wait the maximum cache TTL, sign, keep the old key for the maximum token lifetime, then retire — and add rate-limited refresh-on-unknown-`kid` (§1; `S03`, `S10`).

**3.** After adding "refresh JWKS on unknown `kid`" to every verifier, the IdP's JWKS endpoint receives 40,000 requests per second during an unrelated incident and falls over. Why?

> **Direction:** Junk `kid`s caused a refresh per request across the fleet; single-flight and rate-limit refreshes per process (once per 30–60 s), serve stale on error, and reject unknown `kid`s thereafter (§1; `S03`).

**4.** You implemented refresh-token rotation with reuse detection. After the latest mobile release, "randomly logged out" tickets rise 400 %, with no sign of attack. What is happening, and how do you fix it without disabling reuse detection?

> **Direction:** Parallel refreshes or lost responses present a just-rotated token and trip family revocation; add a short idempotent grace window that returns the same successor, plus a client-side single-flight refresh lock (§2; `S03`).

**5.** Your revocation deny-list lives in Redis. Redis fails over and is unavailable for 90 seconds. Half the team says reject all tokens during that time; the other half says accept all. What do you decide, and how do you make it a non-decision next time?

> **Direction:** Choose fail direction per risk class — open for low-risk reads with an alert, closed for money movement and admin — and document it; then keep an in-process replica fed by pub/sub so Redis is off the request path (§1; `S03`, `S16`).

**6.** An auditor asks how large your `jti` deny-list can grow; engineers fear unbounded growth. Answer precisely.

> **Direction:** Entries need to live only until the token's own `exp`, so with per-entry TTL equal to remaining life the size is revocation rate × maximum access-token TTL — tiny with 15-minute tokens (§1; `S03`).

**7.** About 0.3 % of requests from one availability zone fail with "token used before issued". The auth team suspects replay. What do you check first?

> **Direction:** Clock skew on that AZ's nodes making `iat`/`nbf` look like the future; fix time sync and keep leeway at 60 s or less rather than widening it (§1; `S03`).

**8.** Someone widened JWT leeway to 10 minutes last year to "fix flaky auth". What is the security cost, and how do you unwind it safely?

> **Direction:** Every token effectively lives 10 minutes longer, weakening expiry-based revocation; fix NTP, measure would-be rejections in shadow mode, then reduce leeway gradually (§1, §17; `S03`).

**9.** A laptop holding a developer's personal API token was stolen. Your API tokens are opaque, never expire, and there is no record of which tokens exist per user. What do you do now, and what do you change?

> **Direction:** Revoke by user and review usage for that credential now; then redesign tokens with a public ID prefix, hashed secret, expiry, scopes, last-used tracking and self-service revocation (§9, §10; `S03`, `S10`).

**10.** Your API keys are stored in plaintext and looked up with `WHERE key = ?`. A reviewer flags a timing side channel; another says an index lookup is unexploitable. Who is right, and what design ends the argument?

> **Direction:** Store only a hash, look up by a public key-ID prefix, then constant-time compare the secret's hash — it removes the timing question, protects keys if the database leaks, and makes keys greppable for scanners (§9; `S11`, `S10`).

**11.** Your gateway forwards the user's access token to every downstream service. A logging sidecar in a low-value service wrote full request headers to a shared log index for two months. What is the blast radius, and how should propagation work?

> **Direction:** Each logged token was a broad-audience bearer credential valid everywhere until expiry; use token exchange or gateway-minted per-hop tokens with narrow `aud` and short TTL, and redact auth headers by default (§1, §10; `S03`, `S04`).

**12.** Users who log in via SSO in two tabs at once see "invalid state" in one tab. Attack or bug?

> **Direction:** A bug: one `state`/PKCE-verifier slot in the session is overwritten by the second flow; store per-flow state keyed by the `state` value with a short TTL (§3; `S04`).

**13.** An infostealer campaign is harvesting your customers' session cookies, and attackers are using them without ever hitting MFA. What can you do server-side?

> **Direction:** MFA protects login, not an existing session; bind sessions to device keys (DPoP or device-bound session credentials), score session reuse from new fingerprints or ASNs, shorten sensitive sessions, and step up for sensitive actions (§2, §8; `S03`, `S13`).

**14.** A security review asks you to bind session cookies to the client IP. Your users are mostly mobile. What do you say?

> **Direction:** IP binding breaks carrier-NAT and network-switching users for little gain; use risk scoring on IP/ASN changes, device binding, and step-up for sensitive actions instead (§2, §8; `S03`, `S13`).

**15.** A marketing subdomain running a third-party CMS is compromised, and afterwards users of `app.example.com` have sessions fixed to attacker-known values. `SameSite=Lax` is set. Why did that not help, and what does?

> **Direction:** Sibling subdomains are same-site and can set parent-domain cookies (cookie tossing); use the `__Host-` prefix, rotate the session at login, and move untrusted content to a separate registrable domain (§2; `S03`).

**16.** Your mobile app keeps 90-day sliding refresh tokens. A customer's employee who left six weeks ago still has access through the app, although the corporate IdP disabled them on day one. Why?

> **Direction:** Your refresh path never re-checks upstream; add SCIM deprovisioning or back-channel logout, an absolute session maximum for enterprise tenants, and upstream re-validation at refresh (§2, §3; `S04`).

**17.** A database backup containing your `refresh_tokens` table was exposed. How bad is it, and what would have made it a non-event?

> **Direction:** Plaintext tokens mean every session is hijackable — revoke all families now; storing only hashes of refresh tokens would have made the leaked table useless (§2; `S03`).

**18.** Your mobile team asks to embed the OAuth client secret in the app so it can use the confidential-client flow. What do you tell them?

> **Direction:** Anything shipped to clients is public; mobile apps are public clients and must use authorisation code with PKCE (`S256`, enforced, no downgrade), with app attestation only as a risk signal (§3, §10; `S04`).

**19.** After an XSS was found and fixed in your BFF-based SPA, a manager asks whether any tokens could have been stolen, since "tokens never reach JavaScript". What is the honest answer?

> **Direction:** OAuth tokens could not be exfiltrated, but the script could act as the user through the session cookie while pages were open; scope with request logs from affected sessions and rotate them if needed (§2; `S03`, `S08`).

**20.** Your services cache the IdP's JWKS for 5 minutes. The IdP had a 40-minute outage and the whole platform returned 401, even for users with valid tokens. How do you decouple your availability from the IdP?

> **Direction:** Public keys rarely change: cache JWKS with a long stale-if-error window and a last-known-good copy, so an IdP outage blocks new logins but not existing sessions (§1, §3; `S03`, `S04`).

### Level 3 — Login under attack: stuffing, bots and abuse

Account takeover, anti-automation and the economics of abuse — very common for consumer-facing roles.

**1.** At 03:00 your login service's CPU goes to 100 % and p99 latency on the whole API climbs to 12 s. Traffic is 20× normal, spread across 300,000 IPs with one or two attempts each. The on-call engineer proposes lowering the bcrypt cost from 12 to 8 "until it passes". What do you do instead?

> **Direction:** It is credential stuffing turning into CPU exhaustion; screen and rate-limit before hashing, cap concurrent hash operations with a bounded pool so auth queues without starving the API, isolate auth capacity — and never lower the cost (§8; `S02`, `S13`).

**2.** Your per-IP rate limit is 10 login attempts per minute, and it never triggers, yet you believe a stuffing attack is under way. What metric would prove it, and which limits actually work?

> **Direction:** Watch the global login success ratio (normally 60–90 %, falling to 1–5 % under stuffing) and failures per distinct username; limit per username, subnet/ASN and device/TLS fingerprint, with a global failure budget that switches on step-up (§8; `S13`, `A08`).

**3.** After an ATO wave, product asks you to lock accounts for 30 minutes after 5 failed attempts. Why might this create a bigger incident, and what do you propose?

> **Direction:** Lockout is itself a DoS tool — anyone with a list of usernames locks out your customers; use progressive per-username delays, risk-based step-up, and breached-password checks instead (§8; `S13`, `S02`).

**4.** You deployed a CAPTCHA on login. Within 6 hours the attack traffic resumed at the same success rate, now solving CAPTCHAs. Conversion dropped 4 %. What did you learn, and what is the better posture?

> **Direction:** Visible challenges are solved cheaply by farms and teach attackers what you detect; use risk-based step-up only when elevated, silent shadow responses for confirmed bots, and measure false positives through conversion (§8; `S13`).

**5.** A finance alert shows your SMS OTP bill jumped from £2,000 to £68,000 in a weekend, mostly to numbers in a handful of countries you have almost no customers in. There was no account takeover. What is this, and how do you stop it?

> **Direction:** SMS pumping / toll fraud via premium-rate ranges; cap sends per country prefix, per number, per session and globally with a spend alarm, block unneeded destinations, and prefer TOTP or passkeys (§8; `S13`).

**6.** Attackers who successfully log in with stuffed credentials immediately change the account email and phone number, then use "forgot password" to lock out the owner. What controls stop the damage even after a correct password guess?

> **Direction:** Step-up for sensitive changes, notification to the old email/phone with a revert link and delay window, new-device alerts, and forcing a reset when the password appears in breach corpora (§8; `S02`, `S13`).

**7.** A partner reports that your "check if username is available" endpoint is being called 5 million times a day from a single cloud provider's ranges. It returns a boolean. Why do you care?

> **Direction:** It is an enumeration oracle feeding targeted stuffing lists; rate-limit per client and session, require a sign-up session context, and add friction only when volume is anomalous (§8; `S13`).

**8.** Your gift-card redemption endpoint uses 10-character alphanumeric codes. Rate limiting is per IP. A fraud team sees steady low-rate redemption of cards that were never sold online. What is the attack and fix?

> **Direction:** Distributed brute force across many IPs; rate-limit per account and globally on failures, lengthen codes, and alert on failure ratios — IP is not the right key for a low-and-slow guessing attack (§8; `S13`).

**9.** A competitor scrapes your product catalogue and pricing hourly using thousands of real logged-in accounts created with disposable emails. Per-IP limits are useless. How do you approach it?

> **Direction:** Treat it as economics: per-account quotas, behavioural baselines, sign-up friction scaled by risk, and silent degradation (stale or shadow data) for confirmed scrapers rather than visible blocks they can tune against (§8; `S13`).

**10.** Security wants to require MFA for everyone; growth says it will cut sign-ups by 15 %. As the senior engineer, what design do you propose?

> **Direction:** Risk-based step-up — MFA on new devices, sensitive actions and anomalous logins — plus passkeys as the low-friction option and breached-password checks; record the residual risk as an explicit acceptance (§8, §17; `S02`, `S13`).

**11.** An employee's account was taken over via MFA push fatigue: 40 prompts at 01:00, then one accept. What changes do you make?

> **Direction:** Number matching, rate limiting and alerting on repeated prompts, showing location and context in prompts, and phishing-resistant WebAuthn/passkeys for staff (§8; `S02`).

**12.** Your TOTP verification endpoint has no rate limit because "codes expire in 30 seconds". Why is that wrong, and what else is often missing?

> **Direction:** A 6-digit space falls to unthrottled guessing quickly even across windows; rate-limit per account, lock the factor after N failures with recovery, and reject replay of a code already used in its window (§8; `S02`).

**13.** Your helpdesk resets MFA for users who can state their date of birth and last four card digits. A high-profile customer lost their account this way. What is the real weakness and fix?

> **Direction:** The recovery flow is the weakest link, and those facts are public or breached; require strong verification for resets, a waiting period with notification to existing factors, and audited, approval-based helpdesk tools (§8, §4; `S02`, `S16`).

**14.** Argon2id is configured with 64 MiB memory. During a login burst, auth pods start OOM-killing and restarting, which makes the backlog worse. Why, and what is the fix that keeps the security property?

> **Direction:** Memory-hard hashing multiplies memory by concurrency; cap concurrent hash operations with a bounded pool sized to pod memory (queue or shed the rest), and scale out rather than lowering parameters below the OWASP minimum (§8; `S02`).

**15.** You are migrating 30 million users from unsalted MD5 password hashes left by an acquisition. Only 20 % log in each year. The CEO refuses a forced reset. What do you do?

> **Direction:** Wrap every legacy hash immediately (`argon2(md5)`) so no weak hash sits at rest, and migrate each user to a pure Argon2 hash on their next successful login (§8; `S02`).

**16.** Users with very long passphrases report they can log in with only the first part of their password. The stack uses bcrypt. What is the cause, and what is the subtle trap in the obvious fix?

> **Direction:** bcrypt truncates at 72 bytes; pre-hashing fixes it only if the digest is encoded (e.g. base64) to avoid NUL-byte truncation — or move to Argon2id (§8; `S02`).

**17.** Your free tier gives £20 of compute credits per account. Abuse is burning £40,000 a month through automated sign-ups. Email verification is already required. Where do you add cost for the attacker without hurting real users?

> **Direction:** Raise attacker cost at the value point: risk-scored verification (card or phone checks only when signals are poor), device and fingerprint linkage across accounts, delayed credit release, and quotas that make farming unprofitable (§8; `S13`).

**18.** Your bot-detection vendor blocks traffic with a 403 and a branded page. The attacker's retool time after each new rule is under two hours. How would you change the response strategy?

> **Direction:** Stop giving feedback: shadow responses (plausible failures, delayed errors) for confirmed bots, randomise enforcement, and keep kill switches for false positives (§8; `S13`).

**19.** The "forgot password" endpoint is being flooded, sending 200,000 reset emails an hour to real customers and damaging your email sender reputation. How do you mitigate without enabling enumeration?

> **Direction:** Rate-limit per target account and per source, keep the "if an account exists" response identical, and coalesce duplicate resets within a window so one email is sent while responses stay uniform (§8; `S13`).

**20.** A new WAF rule to block stuffing is ready. The last WAF change blocked checkout for a large corporate customer behind a shared proxy for three hours. How do you roll this one out?

> **Direction:** Run the rule log-only to measure what it would block, check shared-egress customers, enforce behind a flag with a kill switch, and watch conversion metrics during rollout (§8, §17; `S13`, `S17`).

### Level 4 — Secrets, credentials and rotation

Leaked keys, rotation without downtime, and the places secrets actually end up.

**1.** A developer pushed an AWS access key to a public GitHub repository 11 minutes ago and has already deleted the commit. They ask whether a force-push rewriting history is enough. What do you do, in order?

> **Direction:** Public keys are harvested within minutes, so history rewriting is irrelevant: deactivate and rotate the key now, review CloudTrail by access key ID for what it did, including new keys, users or roles created, and remove any persistence (§10, §16; `S10`, `S16`).

**2.** Your production database password must be rotated quarterly. Last time, rotation caused a 20-minute outage because some services had not picked up the new value. How do you make rotation zero-downtime?

> **Direction:** Dual credentials: create the new one while the old stays valid (alternating users A/B), roll it out, verify from connection logs that nothing uses the old one, then revoke (§10; `S10`).

**3.** You revoked a leaked database password, but monitoring shows the attacker's connection is still issuing queries 30 minutes later. How?

> **Direction:** Revoking a password does not terminate existing authenticated sessions; kill that role's active connections explicitly, and remember that pooled app connections likewise survive rotation until they reconnect (§10; `S10`, `DB37`).

**4.** A heap dump from a production JVM was uploaded to a Jira ticket for a performance investigation. Why is this a security incident, and what policy do you set?

> **Direction:** A heap dump contains every secret and much of the PII the process held; treat it as a credential leak (rotate what was in memory), restrict ticket access, and store dumps only in an access-controlled, expiring location (§10; `S10`, `S12`).

**5.** Your error tracker captures local variables on exception. A search there for `password` returns 14,000 events. What happened and how do you fix it at the right layer?

> **Direction:** Automatic capture of locals and request bodies; configure scrubbing in the SDK before sending and server-side as a backstop, purge the historical events, and rotate any credentials found (§10, §12; `S10`).

**6.** A Docker image published to your internal registry was built with `ARG NPM_TOKEN` and a later `RUN rm .npmrc`. A reviewer says the token is gone. Is it?

> **Direction:** No — build args and earlier layers are recoverable via image history and layer extraction; rotate the token, use BuildKit secret mounts, and scan images for secrets (§10; `S10`, `S14`).

**7.** Your CI masks secrets in logs. A failing test printed a base64-encoded version of a service credential into public build logs of an open-source repository. Why did masking not catch it, and what is the structural fix?

> **Direction:** Masking matches exact strings only; the fix is not having long-lived secrets in CI at all — use OIDC federation for short-lived, narrowly scoped cloud credentials and rotate the exposed one (§10, §14; `S10`, `S14`).

**8.** Spring Boot Actuator was exposed on a public load balancer with `/env` and `/heapdump` enabled "for debugging". What is the exposure, and what defaults should your platform enforce?

> **Direction:** Environment variables and memory disclose secrets; rotate everything in that process's environment, expose management endpoints only on an internal port, and prefer mounted or fetched secrets over env vars (§10; `S10`, `F26`).

**9.** Your webhook signing secret for 3,000 customers must be rotated after a leak, but customers verify signatures in their own code, and you cannot coordinate a flag day. How do you rotate?

> **Direction:** Support two active secrets per customer with a key ID in the signature header, sign with both (or with new plus kid) during a migration window, and let customers switch before retiring the old secret (§10; `S10`).

**10.** A scheduled rotation of a third-party API key broke a nightly reconciliation job at 02:00 that nobody knew used that key. How do you avoid "unknown consumer" breakages?

> **Direction:** Step 3 of dual-key rotation: verify via last-used data and access logs that nothing uses the old credential before revocation, and give each consumer its own credential so usage is attributable (§10; `S10`).

**11.** Your team keeps secrets in environment variables set in the deployment manifest. An auditor objects. The team says "env vars are how twelve-factor apps work". Who is right?

> **Direction:** Env vars leak via `/proc/<pid>/environ`, child-process inheritance, debug endpoints, crash reports and orchestration UIs; mounted files or runtime fetches from a secrets manager narrow exposure, ideally with short-lived dynamic credentials (§10; `S10`).

**12.** You planted canary AWS keys in your internal wiki. One fires from an IP in a residential ISP at 04:00. What does that tell you, and what do you do next?

> **Direction:** Someone with wiki access (or a scraper of it) is harvesting credentials — treat it as a compromise of the wiki or an account with access: identify recent accessors and sessions, and rotate real secrets stored nearby (§10, §15, §16; `S16`).

**13.** A mobile app contains a "hidden" API key for a mapping provider. The provider bill increases 10×. The team wants to obfuscate it better. What is the correct answer?

> **Direction:** Anything in a client is public; restrict the key by app signature, referrer or bundle ID at the provider, set quotas and spend caps, or proxy through your backend with per-user limits (§10; `S10`, `S13`).

**14.** You move to Vault dynamic database credentials with 1-hour leases. During a Vault outage, all new pods fail to start and autoscaling stops working. What trade-off did you miss?

> **Direction:** Dynamic secrets make the secrets service a hard start-up dependency; design the failure mode — leases longer than the outage you want to survive, cached credentials, and a highly available Vault — accepting the attribution and leak-ageing benefits (§10; `S10`).

**15.** Your request logs include full URLs. A partner integration sends its API key as a query parameter. Where has that key now propagated?

> **Direction:** Access logs, proxy and CDN logs, analytics, browser history and `Referer` headers; rotate it, require keys in headers, and add log-pipeline redaction for known key formats (§10; `S10`).

**16.** A secret scanner in CI blocks 30 merges a week, 90 % of them false positives on test fixtures. Engineers now routinely bypass it. How do you fix the process?

> **Direction:** Tune for signal: provider-specific formats and verified-live detection, prefixed internal key formats that are easy to detect, and scoped suppressions with expiry — then turn on push protection for high-confidence matches only (§9, §10, §17; `S17`).

**17.** A contractor's laptop that had a production kubeconfig with a long-lived token was stolen. How do you scope and contain?

> **Direction:** Revoke that credential and its service-account tokens, review API server audit logs for its use, check for persistence (new service accounts, role bindings, workloads), and move humans to short-lived SSO-issued credentials (§10, §14, §16; `S15`, `S16`).

**18.** Someone proposes encrypting all secrets in the Git repository with a single shared key "so they're safe in Git". What are the operational problems?

> **Direction:** Every holder of the shared key can decrypt everything forever, rotation means re-encrypting and redistributing, and history keeps old values; prefer a secrets manager with per-workload access, or KMS-backed per-environment encryption with auditable decrypts (§10, §11; `S10`).

**19.** Your ORM logs SQL with bound parameters at DEBUG level. Someone set production to DEBUG for an hour during an incident. What is the aftermath checklist?

> **Direction:** Bound parameters include passwords, tokens and PII; purge or restrict those log segments, rotate credentials that passed through, assess personal-data exposure for notification, and make debug logging redact by default (§10, §12; `S10`, `S12`).

**20.** Nobody has rotated the root credentials of your oldest system in four years because "it's risky". An auditor gives you 30 days. How do you de-risk it?

> **Direction:** Treat rotation as a capability: inventory consumers via usage logs, introduce a second credential and migrate consumers one by one, rehearse in staging, then schedule regular rotations so the procedure stays exercised (§10; `S10`, `S17`).

### Level 5 — Vulnerable features: SSRF, injection and uploads

The feature-shaped vulnerabilities that come up when a product needs to fetch URLs, render files or build queries.

**1.** Your product lets customers register webhook URLs. A researcher shows that registering `http://169.254.169.254/latest/meta-data/iam/security-credentials/` returns the instance role name in the delivery log UI. What is the immediate containment, and what is the long-term design?

> **Direction:** Rotate the role's credentials and review CloudTrail for use from outside your egress IPs, enforce IMDSv2 and stop echoing response bodies; long term, send all user-directed fetches through an isolated egress proxy with no route to metadata or internal ranges (§5, §16; `S08`, `S15`).

**2.** After the SSRF fix, your validator resolves the hostname and blocks private IPs, then passes the URL to the HTTP client. A researcher still reaches `127.0.0.1`. Name two ways, and the design that closes both.

> **Direction:** DNS rebinding (the client resolves again and gets a different answer) and redirects to internal addresses; resolve once, validate the IP, connect to that exact IP, and disable or re-validate redirects per hop — or use the egress proxy (§5; `S08`).

**3.** You enforce IMDSv2 across the fleet on Monday. By Tuesday, three legacy services fail to get credentials and a batch pipeline stops. What should the rollout have looked like?

> **Direction:** Shadow mode first: watch the `MetadataNoToken` metric to find v1 callers (old SDKs), upgrade them, then enforce per account; also check hop limits for containerised workloads, whose token responses can die at a hop limit of 1 (§5, §17; `S15`).

**4.** Your link-preview feature fetches a URL and renders a thumbnail. Traffic analysis shows it is being used to port-scan your VPC: responses take 3 ms for closed ports and 2 s for filtered ones, although no content is returned. Why does blind SSRF still matter, and what do you change?

> **Direction:** Timing and error differences map internal networks even without content; block private ranges at the network layer via an isolated egress proxy, return uniform errors, and cap time and size (§5; `S08`).

**5.** A PDF-rendering service converts user-supplied HTML into invoices using a headless browser. A customer's template includes an iframe pointing at an internal admin URL, and the rendered PDF shows the admin page. What is the class, and where do you enforce?

> **Direction:** SSRF through the renderer; run it in a sandbox with egress restricted to an allow-list (or none), disable JavaScript and local file access, and treat every HTML-to-PDF or image converter as a URL fetcher (§5, §7; `S08`).

**6.** An `ORDER BY` column comes from a query parameter, and the ORM cannot parameterise identifiers, so someone concatenated it. A code scanner flagged it; the author says "it's only sorting". What is the risk and the fix?

> **Direction:** Identifier injection is full SQL injection; map allowed sort keys to fixed column names in an allow-list and reject anything else (§7; `S07`).

**7.** Your Node login endpoint queries Mongo with `{ email: body.email, password: body.password }` (legacy, pre-hashing). A tester logs in as anyone. No string concatenation exists. How?

> **Direction:** Operator injection: sending an object like `{"$ne": null}` where a string was expected; validate types with a schema (strings only) before building queries (§7; `S07`).

**8.** A nightly batch job builds a shell command from customer "company name" fields to generate reports. A customer changed their name last month, and the batch server now runs a crypto miner. Why did everyone think this was safe?

> **Direction:** Second-order injection: data stored safely is later concatenated into a sink by "internal" code; pass arguments as arrays without a shell, and treat stored user data as untrusted everywhere (§7; `S07`).

**9.** Finance opens your CSV exports in Excel. A customer's "notes" field contains a value beginning with `=`. A finance analyst's machine behaves strangely after opening the file. What is this, and what do you change in the export code?

> **Direction:** CSV/formula injection; prefix cells beginning with `=`, `+`, `-`, `@` (and tab/CR) with a quote, or export in a format that does not evaluate formulas (§7; `S07`).

**10.** Your API gateway validates `role` in a JSON body and rejects `admin`. A researcher sends `{"role":"user","role":"admin"}` and becomes an admin. What is happening?

> **Direction:** A parser differential: the gateway takes the first duplicate key, the service takes the last; reject duplicate keys and validate in the component that uses the value (§7; `S07`).

**11.** Usernames are validated to block `admin`. Someone registers with fullwidth characters that normalise to `admin`, and the account ends up sharing a mailbox and permissions with the real admin in a downstream system. What went wrong?

> **Direction:** Validation ran before Unicode normalisation; normalise (NFKC and case-fold) first, validate and enforce uniqueness on the normalised form, and use the same form everywhere (§7; `S07`).

**12.** An image-upload feature checks the file extension and `Content-Type`. A user uploads an HTML file named `avatar.png`; it is served from your main domain and runs script in other users' sessions. What layered fix do you apply?

> **Direction:** Validate by content and re-encode images, serve user content from a separate registrable domain with `nosniff` and `Content-Disposition: attachment`, and never serve uploads from the app origin (§7, §2; `S08`).

**13.** Your upload worker processes ZIP archives. A 42 KB file took down the worker fleet overnight. What happened, and what limits do you set?

> **Direction:** A decompression bomb; bound total decompressed size, file count and nesting depth while streaming, with time limits, and run parsers in a sandboxed worker (§7; `S06`).

**14.** A Python service stores user preferences in a cookie as a base64-encoded `pickle`. A developer argues it's fine because the cookie is "just preferences". What is the actual risk, and the migration?

> **Direction:** Unpickling attacker-controlled bytes is remote code execution; switch to JSON with a schema, and during migration sign legacy blobs and verify the signature before any deserialisation (§7; `S07`).

**15.** Log4Shell-style: a new CVE shows your logging library interprets `${...}` expressions inside log messages. Your team asks which of 200 services are affected. Grepping repositories finds 12. Why is that number wrong?

> **Direction:** Grep misses transitive dependencies, shaded or vendored jars and base images; query build-time SBOMs per deployed artefact, and mitigate by configuration or WAF while patching (§7, §13; `S14`).

**16.** An XML import feature for enterprise customers suddenly shows `/etc/passwd` in a validation error message. What is the class, and what configuration change fixes it across all parsers?

> **Direction:** XML external entities; disable DTDs and external entity resolution in every XML parser (a shared hardened factory), and stop echoing parser error details to users (§5, §7; `S07`, `S08`).

**17.** Your email-template feature lets customers write templates with a server-side template engine. A customer template reads environment variables. What is the fix that still lets customers customise emails?

> **Direction:** Server-side template injection; use a logic-less or sandboxed template language exposing only an explicit data context, never a full-featured engine on user-authored templates (§7; `S07`).

**18.** Your structured-logging migration is being deprioritised. Someone asks what concrete security risk plain-text logs carry. Give one incident-shaped answer.

> **Direction:** Log forging: CR/LF in user input creates fake log lines that mislead investigators or hide activity; JSON logging escapes fields, and interpreting logging libraries are an injection sink (§7, §15; `S07`, `S16`).

**19.** A photo-sharing app strips nothing from uploads. A journalist shows that users' home locations are recoverable from shared photos. Is that a security bug, and what changes?

> **Direction:** It is a privacy exposure (EXIF GPS is personal data); re-encode images server-side stripping metadata by default, and review other derived data that leaks location or identity (§7, §12; `S12`).

**20.** Your WAF blocks common SQL-injection patterns, and product asks whether you can skip fixing a raw-SQL endpoint the WAF "covers". What do you tell them?

> **Direction:** A WAF decodes and parses differently from your app (encoding and parser differentials), so it is a temporary compensating control; fix the query with parameters or allow-lists, and record the WAF rule as time-boxed risk acceptance (§7, §17; `S07`, `S17`).

### Level 6 — OAuth, OIDC and SSO in the real world

Federation bugs that show up once you have enterprise customers, social logins and multiple IdPs.

**1.** You added "Sign in with Provider X" and link accounts automatically when the email matches an existing user. A researcher takes over an existing account using a provider account with an unverified email. What is the design error?

> **Direction:** Email is an attribute, not an identity key; key identities on `(iss, sub)`, check `email_verified`, and link only through an explicit, authenticated user action (§3; `S04`).

**2.** Your OAuth client allows redirect URIs matching `https://app.example.com*`. A bug bounty report shows codes being delivered to a different host. How, and what is the correct rule?

> **Direction:** Prefix matching accepts hosts like `app.example.com.attacker.net`; require exact string matching against registered URIs, with no wildcards (§3; `S04`).

**3.** Redirect URIs are exact-matched, but an open redirect on `app.example.com/go?to=...` still leaks authorisation codes. Explain the chain and the fixes.

> **Direction:** The code lands on an allowed page that bounces it onward (in the URL or `Referer`); fix the open redirect, exchange and strip codes immediately, set `Referrer-Policy`, and enforce PKCE so a leaked code is useless (§3; `S04`).

**4.** Your login flow does not use the `state` parameter because "PKCE covers it". A support ticket shows a user's saved card ended up in someone else's account. How is that possible?

> **Direction:** Login CSRF: the attacker completes a flow with their own account in the victim's browser; bind the callback to the initiating session with `state` (PKCE does not cover the user's own account being swapped) (§3; `S04`).

**5.** An enterprise customer's IdP is your multi-tenant Entra ID app using the `common` endpoint. A security review finds that any Microsoft tenant's user can log in and land in a default tenant. What check is missing?

> **Direction:** Validate the issuer/tenant claim against your onboarded tenant mapping, not just the signature; reject tenants you have not onboarded (§3; `S04`, `M29`).

**6.** Your SAML integration passes all tests. A researcher sends an assertion with the signature element removed, and your library accepts it. How?

> **Direction:** The configuration required signatures only "if present"; require signed assertions, ensure the signed element is exactly the one consumed, and validate `Audience`, `Recipient` and `NotOnOrAfter` with a replay cache (§3; `S04`).

**7.** A customer asks for IdP-initiated SAML so users can click your tile in their portal. What risks do you raise, and how do you support it safely?

> **Direction:** No `state` equivalent makes it prone to login CSRF and replay; support it with strict assertion lifetimes and replay caches, then bounce into an SP-initiated flow where possible (§3; `S04`).

**8.** Your IdP went down for an hour; nobody could log in, including your on-call engineers to production admin tools. What should have existed?

> **Direction:** A break-glass path independent of the IdP — hardware-key-protected accounts, heavily audited and alarmed on use — plus long JWKS stale-if-error caching so existing sessions survive (§3, §15; `S04`, `S16`).

**9.** You accept ID tokens as API bearer tokens because "they are signed by the same IdP". What goes wrong?

> **Direction:** ID tokens are for the client, with the client's `aud` and no API scopes; APIs must require access tokens with their own audience and token type (§1, §3; `S04`).

**10.** A customer's enterprise users see "invalid PKCE verifier" randomly after the mobile team upgraded their auth library. What might have changed?

> **Direction:** The library now uses `S256` or regenerates the verifier on retry or app backgrounding, so a stale verifier is sent; store the verifier per flow, and do not fall back to `plain` to "fix" it (§3; `S04`).

**11.** Your platform supports two IdPs per tenant. A tester gets a code from IdP A delivered to your IdP B token endpoint handling. What is this attack, and the defence?

> **Direction:** An OAuth mix-up; track which IdP each flow targeted via per-IdP `state`, and verify the `iss` parameter in the authorisation response (RFC 9207) (§3; `S04`).

**12.** A third-party integration asks for your users' passwords to call your API on their behalf ("password grant"). The partner is large and important. What do you offer instead?

> **Direction:** ROPC is deprecated, trains phishing and defeats MFA; offer authorisation code with PKCE and scoped, revocable delegated tokens, with a consent screen (§3; `S04`).

**13.** Your OAuth authorisation server issues refresh tokens that never expire for third-party apps. A popular third-party app was breached. How do you respond, and what do you change?

> **Direction:** Revoke all tokens for that client (you need per-client token families), notify users, and move to rotating refresh tokens with reuse detection and maximum lifetimes, with a per-client kill switch (§2, §3; `S04`).

**14.** An enterprise customer offboards an employee in their IdP. Their SCIM integration with you has been silently failing for months. Who finds out, and how do you make deprovisioning trustworthy?

> **Direction:** Monitor SCIM health per tenant with alerts, re-validate sessions upstream at refresh, cap session age for enterprise tenants, and support back-channel logout (§3; `S04`, `S16`).

**15.** You key user accounts on `sub` from your social login provider. After enabling a second app ID with the same provider, every user appears as a new account. Why?

> **Direction:** `sub` is unique per issuer and, for pairwise-identifier providers, per client; key on `(iss, sub)` and understand the provider's identifier scope before adding clients (§3; `S04`).

**16.** A product manager wants "magic link" login. What security properties must the implementation have that teams commonly miss?

> **Direction:** Single-use, short-lived, hashed-at-rest tokens bound to the initiating browser where possible; session rotation on use; resistance to email scanners pre-fetching links (require a click-through POST); and no `Host`-derived URLs (§2, §3, §6; `S02`).

**17.** A customer's security team demands you support their IdP's "force re-authentication for sensitive actions". Your app has a 12-hour session. How do you implement step-up with OIDC?

> **Direction:** Request `prompt=login` or `max_age` with `acr_values`, verify `auth_time` and `acr` in the returned ID token, rotate the session, and store the step-up time for a bounded window (§2, §3; `S04`).

**18.** Your OAuth consent screen is used in a phishing campaign: users grant a malicious app full mailbox scope. What platform-side controls reduce the damage?

> **Direction:** Consent phishing; verify publishers, restrict high-risk scopes to reviewed apps, show clear consent details, allow tenant admins to block user consent, and give users and admins revoke-and-audit tools (§3, §15; `S04`, `S16`).

**19.** You receive a report that authorisation codes appear in your analytics provider's page-view logs. Where did they come from, and what is the fix?

> **Direction:** The callback URL with the code was loaded by a page running analytics scripts, and `Referer` leaked it; handle callbacks on a script-free endpoint, exchange and redirect immediately, and set `Referrer-Policy` (§3; `S04`).

**20.** You are asked to let customers bring their own OIDC IdP with a discovery URL they type in. What new risks does that introduce to your backend?

> **Direction:** Your server now fetches arbitrary URLs (SSRF via discovery and JWKS fetches) and trusts arbitrary issuers; fetch through the egress proxy, pin issuer per tenant, and never let one tenant's IdP assert identities for another (§3, §5; `S04`, `S08`, `M29`).

### Level 7 — Proxies, CDNs and the HTTP edge

Less common but memorable: bugs that live between components that parse HTTP differently.

**1.** After upgrading your edge proxy, a handful of users report seeing other users' account pages for a single request, maybe twice a day. Back-end logs show some requests with a strange prefix glued to the start of the method. What is happening, and what is the emergency mitigation?

> **Direction:** Request smuggling / desync on reused back-end connections; reject ambiguous requests (both `Content-Length` and `Transfer-Encoding`, duplicate CL) at the edge, and as an emergency measure disable back-end keep-alive so a desync cannot cross users (§6; `S06`, `A01`).

**2.** Your CDN caches by file extension. A researcher sends a victim a link to `/account/profile/avatar.css`; the app serves the dynamic profile page for it and the CDN caches it publicly. What is this, and what fixes it on both sides?

> **Direction:** Web cache deception; the origin sends `Cache-Control: no-store, private` on authenticated responses and the CDN honours origin headers, cache rules target explicit static prefixes, and routing 404s unknown suffixes (§6; `S06`, `A04`).

**3.** Your marketing pages are cached at the CDN for 10 minutes. A tester sends one request with a crafted `X-Forwarded-Host`, and for the next 10 minutes every visitor loads JavaScript from the tester's domain. How?

> **Direction:** Web cache poisoning via an unkeyed input that changes the response; strip or overwrite forwarding headers at the edge, and include anything that affects output in the cache key (§6; `S06`).

**4.** Your per-client rate limiter keys on the first IP in `X-Forwarded-For`. Attackers bypass it trivially. Why, and which IP should you use?

> **Direction:** The leftmost entry is client-controlled; take the IP at a fixed hop count from the right, added by your own trusted proxies, and have the edge overwrite rather than append (§6; `A08`, `S13`).

**5.** A service trusts an `X-User-Id` header injected by the gateway. A security-group change made the service's load balancer internet-reachable for two days. What might have happened, and what should the service have done?

> **Direction:** Anyone could have impersonated any user by setting the header; services should verify a gateway-signed token (or mTLS from the proxy) rather than trust plain headers, and review access logs for direct traffic in that window (§6, §14, §16; `S05`, `S15`).

**6.** After enabling HTTP/2 at your edge (which downgrades to HTTP/1.1 towards origins), a pen test finds a new desync that did not exist before. Why would a protocol upgrade introduce smuggling?

> **Direction:** HTTP/2 frames carry their own length, so injected `Content-Length` or `Transfer-Encoding` headers can disagree with the frame after downgrade (H2.CL/H2.TE); the edge must validate and strip these, or use HTTP/2 end-to-end (§6; `A10`).

**7.** Your CDN cached a 403 error page for an API path for 10 minutes after a brief auth-service hiccup, locking out all users. What was the configuration mistake?

> **Direction:** Caching error responses (a self-inflicted cache poisoning); do not cache 4xx/5xx on authenticated or API paths, or only for seconds, and make authenticated responses `private` or `no-store` (§6; `A04`).

**8.** A WAF blocks `../` in paths, but a researcher reads files outside the web root using `%2e%2e%2f`. Where is the gap?

> **Direction:** A decoding differential between WAF and app or framework; normalise and decode once at the edge, reject ambiguous encodings, and do path confinement in the application after normalisation (§6, §7; `S07`).

**9.** Your internal admin UI listens on `localhost:8080` on developer laptops and on a private IP in staging, with no authentication. Security says a public website could reach it from an employee's browser. How?

> **Direction:** DNS rebinding makes a hostile domain resolve to the internal IP after load, and the browser treats it as same-origin; validate `Host` against an allow-list and require authentication even on "local" ports (§5; `S08`).

**10.** Your API sets `Cache-Control: private` on user data, but a corporate proxy used by a big customer still serves one employee's data to another. What else should you have sent, and why?

> **Direction:** Some intermediaries mishandle `private`; use `no-store` for sensitive responses and `Vary` correctly, and avoid putting personalised data behind shared-cacheable URLs (§6; `A04`).

**11.** A CDN rule normalises paths (`/a/../b` → `/b`) before routing, but the origin framework does not normalise before its authorisation middleware. What class of bug can this create?

> **Direction:** Path-normalisation differentials let a request bypass path-based authorisation or cache rules; make one component canonicalise, reject non-canonical paths elsewhere, and authorise on resources rather than URL prefixes (§6, §7; `S09`).

**12.** You enabled `Strict-Transport-Security` with `includeSubDomains; preload` and submitted it to the browser preload list. A week later, a legacy HTTP-only internal subdomain stops working for everyone. Can you roll back quickly?

> **Direction:** Not really — preload removal takes browser release cycles; roll out HSTS with short max-age first, inventory subdomains, and only then add `includeSubDomains` and preload (§17; `A11`).

**13.** Your team rolls out a strict Content-Security-Policy and the checkout page breaks for 8 % of users because of a payment provider's script. How should the rollout have gone?

> **Direction:** Deploy `Content-Security-Policy-Report-Only` first, collect violation reports for a week, fix or allow-list legitimate sources, then enforce behind a flag (§17; `S08`).

**14.** A load balancer terminates TLS and forwards plain HTTP to your pods. An auditor asks whether a compromised pod on the same node could read other services' traffic. What do you answer and propose?

> **Direction:** Yes, if it can sniff or reach node networking; re-encrypt to the back end or use mesh mTLS with workload identity, so traffic is encrypted and authenticated per service (§14; `S10`, `M26`).

**15.** The CDN strips `Authorization` headers on cached routes. A developer moved an authenticated endpoint under a cached prefix and users started getting `401`s, then someone "fixed" it by adding `Authorization` to the cache key. What is still wrong?

> **Direction:** Personalised responses are now in a shared cache keyed on a bearer token (the cache holds user data and tokens as keys); move authenticated endpoints off cached routes and send `no-store` (§6; `A04`).

**16.** A partner reports that your webhook signatures sometimes fail verification only when their reverse proxy is in the path. Signatures cover the raw body. What is likely happening?

> **Direction:** The proxy re-serialises, re-encodes or decompresses the body; sign and verify the exact raw bytes, include a timestamp against replay, and document that intermediaries must not modify bodies (§6, §9; `S10`).

**17.** Your API returns a compressed JSON response containing a CSRF token alongside user-controlled search text, over HTTPS. A researcher claims they can recover the token. How, and what do you change?

> **Direction:** A compression side channel (BREACH-style): response size reveals matches between input and the secret; do not compress responses mixing secrets and reflected input, mask tokens per response, or move secrets out of such responses (§9; `S11`).

**18.** Security wants all internal services to sit behind a single new auth proxy. Six months later, a team adds a second ingress for a partner integration, bypassing the proxy. What structural control prevents this regression?

> **Direction:** Services verify identity themselves (signed tokens or mTLS), and network policy only admits traffic from approved proxies; the proxy is defence in depth, not the only check (§6, §14; `S01`, `M26`).

**19.** A response header `Server: nginx/1.14.0` and verbose stack traces on 500s were flagged in a pen test as "low". A manager wants to skip them. How do you prioritise honestly?

> **Direction:** They are information disclosure that speeds targeting; cheap to fix and worth doing, but the real priority is the outdated version itself, since version banners matter mostly when you are behind on patches (§13, §17; `S06`, `S17`).

**20.** After moving to a new CDN, a customer's security scanner reports that your origin IP is directly reachable, bypassing the CDN's WAF and rate limits. How do you lock this down?

> **Direction:** Restrict origin ingress to CDN ranges or an authenticated origin pull (mTLS or a secret header rotated like any credential), and make the app refuse requests without it (§6, §10; `S06`).

### Level 8 — Cloud identity, containers and the supply chain

IAM, workload identity, Kubernetes and dependency compromises — increasingly common in platform-heavy interviews.

**1.** Your CI pipeline assumes an AWS production role via GitHub OIDC. The trust policy checks only that the token comes from your GitHub organisation. What is the risk, and what should it check?

> **Direction:** Any repository in the organisation, including pull-request workflows, could assume the production role; pin `sub` to the specific repository plus a protected branch or environment (§14; `S15`).

**2.** An internal package `acme-auth-utils` is published only to your private registry. One morning, CI builds pull version 99.0.0 from the public npm registry, and it runs a `postinstall` script. What happened, and what are the fixes?

> **Direction:** Dependency confusion; use scoped names bound to the private registry, a single proxying registry that never falls back to public for internal names, placeholder public registrations, and `--ignore-scripts` by default (§13; `S14`).

**3.** A popular third-party GitHub Action you use at `@v3` was compromised and printed secrets into public build logs. Your repositories are public. What do you do in the first hour, and what policy do you adopt?

> **Direction:** Rotate every secret available to affected workflows and review logs and cloud usage; then pin actions by commit SHA, minimise secrets in CI with OIDC federation, and restrict which actions are allowed (§13, §10; `S14`, `S10`).

**4.** A PR from an external contributor modified `package-lock.json` in a 4,000-line diff. Reviewers approved the code changes. Why should the lockfile have been reviewed, and how do you automate it?

> **Direction:** Lockfile injection can change `resolved` URLs or integrity hashes to a malicious source; lint lockfiles for allowed registry hosts and unexpected changes, and enforce frozen installs in CI (§13; `S14`).

**5.** Production builds use `npm install`, and a transitive dependency got a new minor version with a backdoor overnight. Staging, built the day before, is clean. What did you learn?

> **Direction:** Lockfile drift: CI must use `npm ci` or equivalent frozen installs with hash checks, plus a cooling-off period before adopting newly published versions (§13; `S14`).

**6.** A CVE is announced in a widely used compression library at 16:00 on a Friday. Leadership asks which of your 300 services are exposed. How do you answer in under an hour?

> **Direction:** Query build-time SBOMs per deployed artefact (including base images and transitive dependencies), then prioritise by exposure and reachability, not by repository grep (§13; `S14`, `O08`).

**7.** Your vulnerability dashboard has 11,000 open findings, and teams ignore it. The CISO wants a "zero criticals" target. What do you propose instead?

> **Direction:** Prioritise by reachability, internet exposure and exploitation evidence (KEV, EPSS), shrink the base-image surface with distroless or minimal images, and set SLAs by real risk (§13; `S14`, `S17`).

**8.** A pod in your cluster was compromised through an RCE. It runs with the node's IAM role because the team never set up pod identity. What is the blast radius, and what should the architecture be?

> **Direction:** Every permission of the node role, shared by all pods on the node; use IRSA/EKS Pod Identity (or GKE/Azure equivalents), block pod access to node metadata, and scope roles per service account (§14, §5; `S15`).

**9.** A developer grants a service account a role with `list` on secrets "to read one config value". Why is that a bigger grant than it sounds?

> **Direction:** `list`/`watch` returns secret contents, so it is effectively read-all-secrets in that namespace; grant `get` on a named secret, or mount the one secret directly (§14; `S15`).

**10.** A SaaS vendor asks you to create an IAM role they can assume in your account. They give you their account ID. What else must the trust policy contain, and why?

> **Direction:** An `ExternalId` condition unique to you, to prevent the confused-deputy scenario where another customer of the vendor gets it to assume your role (§14; `S15`).

**11.** An internal file service with broad S3 access takes `bucket` and `key` parameters from callers. Every caller is authenticated. What is the design flaw?

> **Direction:** A confused deputy: the service uses its own permissions for caller-chosen resources; authorise the caller's identity against the specific resource, or have callers present scoped credentials (§14, §4; `S15`, `S05`).

**12.** A role can call `iam:PassRole` on `*` and create Lambda functions. The team says it has "no admin permissions". Explain why it effectively does.

> **Direction:** It can create a function that runs as any passable role, including admin roles; scope `PassRole` to specific role ARNs and treat it as a privilege-escalation primitive (§14; `S15`).

**13.** You run customer-supplied scripts in Docker containers with the default runtime. A security review says containers are not a sufficient boundary. Explain why, and what you would use.

> **Direction:** Containers share the host kernel, so a kernel or runtime bug escapes; for untrusted code use a separate-kernel sandbox (gVisor, Firecracker/Kata) with no credentials and restricted egress (§14; `S15`, `O01`).

**14.** A CI job needs Docker, so the team mounts `/var/run/docker.sock` into the build container. What have they granted?

> **Direction:** Root on the host: the socket can start privileged containers mounting the host filesystem; use rootless or daemonless builders (BuildKit rootless, Kaniko) or isolated ephemeral VMs (§14; `S15`, `S14`).

**15.** Your self-hosted CI runners are shared by all repositories and persist between jobs. A low-trust repository's build was compromised. What is now at risk?

> **Direction:** Anything later jobs place on that runner (credentials, caches, checked-out code) and any tampering with tools on disk; use ephemeral runners per job and separate runner pools by trust level (§13; `S14`).

**16.** Kubernetes Secrets are "encrypted" according to a platform doc. An auditor asks for proof. What do you actually check?

> **Direction:** By default, secrets are only base64 in etcd; verify that encryption at rest with a KMS provider is configured, and control etcd backup and snapshot access (§14; `S15`, `S10`).

**17.** A least-privilege project wants to tighten 400 IAM roles that all have wildcards. Teams fear breaking production. How do you do it safely?

> **Direction:** Generate policies from observed CloudTrail usage after a burn-in period, apply guardrail SCPs and permission boundaries first, roll out per role in shadow or monitor mode, and keep a fast rollback (§14, §17; `S15`).

**18.** An attacker with a compromised developer role tried to call `cloudtrail:StopLogging` and `kms:ScheduleKeyDeletion`. Both failed. What control made that happen, and what else belongs in the same place?

> **Direction:** Organisation SCP explicit denies outside a break-glass role; also deny deleting log buckets, disabling GuardDuty/Config and leaving the organisation, with alerts on any attempt (§14, §15; `S15`, `S16`).

**19.** Your container images are pulled by tag `:latest`. After a registry credential leak, you cannot tell whether production is running the image you built. What changes?

> **Direction:** Pin by digest, sign images with provenance (Sigstore/cosign, SLSA), and enforce signature verification with cluster admission control (§13; `S14`).

**20.** A maintainer of a small but deep transitive dependency hands the project to a helpful new contributor who, a year later, ships a backdoor in a release tarball that differs from the Git source. What supply-chain controls would have helped?

> **Direction:** The xz pattern: reproducible builds and provenance that tie artefacts to source, cooling-off periods, monitoring of maintainer and release-process changes, and an SBOM to find exposure quickly (§13; `S14`).

### Level 9 — Encryption, key management and privacy engineering

Rarer and deeper: KMS as a dependency, crypto at volume, and GDPR obligations that meet real architectures.

**1.** Your service encrypts every database row with a fresh data key from KMS. At peak, calls start failing with `ThrottlingException`, and at the same time unrelated S3 reads in the same account slow down. Explain the connection and fix it.

> **Direction:** KMS request quotas are per account and region, shared with S3 SSE-KMS and others; cache unwrapped DEKs with bounded lifetime and use count, use a DEK per batch or tenant window, enable S3 Bucket Keys, and isolate heavy workloads in their own accounts (§11; `S11`, `S10`).

**2.** Legal says a customer's right-to-erasure request must cover backups, the data lake and three years of Kafka topics. Deleting from immutable backups is impossible. What architecture makes this tractable?

> **Direction:** Crypto-shredding: encrypt each subject's personal data with a per-subject key, and erase by destroying the key so every copy becomes unreadable (§11, §12; `S12`, `S11`).

**3.** You implemented crypto-shredding. Six months later, you restore the key database from a backup after a corruption incident. What have you just done, and how do you prevent it?

> **Direction:** You resurrected deleted subjects' keys, undoing erasures; keep deletion tombstones outside the key store and replay them after any restore (§11; `S12`).

**4.** Product wants per-subject keys for 80 million users "in KMS". Finance flags the bill. What is the right shape?

> **Direction:** Do not create millions of KMS keys; store per-subject DEKs wrapped under a small number of KEKs in your own key table, with caching and encryption context (§11; `S11`).

**5.** A high-throughput event pipeline encrypts 50,000 messages per second with AES-GCM under one long-lived key and random 96-bit nonces. A reviewer raises a concern. What is it, and how do you address it?

> **Direction:** At ~2^32 messages the random-nonce collision risk becomes unacceptable (reached in about a day here), and GCM nonce reuse breaks confidentiality and integrity; rotate DEKs frequently or use XChaCha20-Poly1305 or AES-GCM-SIV (§11; `S11`).

**6.** A cleanup script with too broad a filter scheduled deletion of 40 KMS keys, including the one protecting your main customer database. It was noticed on day 5. What saved you, and what prevents a repeat?

> **Direction:** The 7–30 day waiting period; cancel deletion, then alarm on `ScheduleKeyDeletion`/`DisableKey`, deny them by SCP outside break-glass, and tag keys so tooling cannot target critical ones (§11, §14; `S11`, `S15`).

**7.** An attacker with admin rights in one of your AWS accounts re-encrypts S3 objects under a KMS key in their own account and then deletes the originals. What is this, and what controls would have stopped or limited it?

> **Direction:** Cloud ransomware using external keys; SCPs restricting use of keys to your organisation, versioning with Object Lock for critical buckets, backups in a separate account, and alerts on unusual cross-account KMS use (§11, §14, §15; `S15`, `S16`).

**8.** Your team encrypts a column with envelope encryption, but a bug moved ciphertext from one tenant's row to another's, and it decrypted fine for the wrong tenant. What was missing?

> **Direction:** Encryption context (AAD) binding the ciphertext to tenant and record IDs; with it, a moved ciphertext fails to decrypt and CloudTrail shows which record each decrypt was for (§11; `S11`, `M29`).

**9.** Your annual KMS key rotation ran, and a manager asks whether all historic data is now encrypted under the new key. What is the true answer, and when does it matter?

> **Direction:** No — automatic rotation keeps old key material for decryption and new material only encrypts new data; re-wrap DEKs if the KEK is suspect, and re-encrypt data only if DEKs are compromised, using stored key IDs (§11; `S11`).

**10.** Analytics receives "anonymised" events where the email is replaced by SHA-256(email). The DPO asks whether this is still personal data. What do you say, and what do you change?

> **Direction:** It is pseudonymised and re-identifiable by hashing candidate emails; use a keyed HMAC with a protected key (still personal data) and aggregate with minimum group sizes where anonymity is claimed (§12; `S12`).

**11.** A breach of a staging environment exposed a copy of production from last quarter. The team insists "it's only staging". Walk through what happens next.

> **Direction:** It is a personal-data breach: start the GDPR 72-hour clock, scope what fields were present, and assess notification; then mask or synthesise lower-environment data so staging never holds production PII (§12, §16; `S12`, `S16`).

**12.** A customer invokes erasure, but finance must retain their invoices for six years. The product team says the two requirements conflict. How do you reconcile them?

> **Direction:** Minimise and separate: keep only fields required for the legal obligation, in a restricted store with a retention clock, and erase or crypto-shred everything else (§12, §11; `S12`).

**13.** You are asked to encrypt a searchable email column so admins can still look up users by email. Encryption with random nonces breaks lookups. What do you do?

> **Direction:** Store a keyed HMAC "blind index" of the normalised email for equality lookups alongside the randomised ciphertext, accepting that it reveals equality; never use deterministic encryption without understanding that leak (§11, §12; `S11`).

**14.** Your service verifies HMAC webhook signatures with a plain string equality check. A researcher claims a timing attack. A colleague says network jitter makes it impossible. Who is right, and what is the cheap fix?

> **Direction:** Repeated measurements can average out jitter, especially from within the same cloud region; use a constant-time compare on fixed-length digests — it costs nothing (§9; `S11`).

**15.** You discover that a mobile app has been sending precise location with every analytics event to a third-party SDK for 18 months. Nobody listed that processor. What is your response?

> **Direction:** Treat it as a data-protection incident: stop the flow with a remote config or kill switch, update the data map and processor list, assess the lawful basis with the DPO, and request deletion from the processor (§12; `S12`).

**16.** A regulator asks you to demonstrate that support staff cannot read customer medical notes, although the notes are stored in the same database the support tool queries. What design gives you something to show?

> **Direction:** Application-level field encryption with keys only the clinical service can use (KMS key policies scoped to that role), audited decrypts via encryption context, and a support tool with no decrypt permission (§11, §15; `S11`, `S12`).

**17.** A team uses one shared DEK, cached in memory for the life of the process, to "avoid KMS costs". The service has 200 pods running for weeks. What risk did they trade, and what is a sensible middle ground?

> **Direction:** One memory disclosure exposes every record ever encrypted, with no rotation; bound DEK caching by time and use count, and scope DEKs per tenant or time window (§11; `S11`).

**18.** You run in the EU for residency reasons, but your error tracker, log vendor and LLM-based support summariser are US-hosted. An enterprise customer's DPA audit is next month. What do you check?

> **Direction:** Processors and log sinks are processing locations; map which personal data reaches each, scrub it at source, and use EU regions or transfer mechanisms for each processor (§12, §10; `S12`).

**19.** Your password-reset token is a random 128-bit value stored in plaintext with a 24-hour expiry. A database read replica was exposed briefly. What is the impact, and what design reduces it?

> **Direction:** Every outstanding reset token was usable for account takeover; store reset tokens hashed, expire them in minutes, make them single-use, and invalidate all outstanding ones now (§2, §9; `S02`).

**20.** You need to prove to an auditor that audit logs were not modified by an insider with database admin rights. The logs are in Postgres. What would convince them?

> **Direction:** Tamper evidence outside the admin's reach: a hash chain whose head is periodically anchored elsewhere, plus a copy streamed to WORM storage (Object Lock compliance mode) in a separate account (§15; `S16`).

### Level 10 — Live incidents, forensics and multi-layer failures

The rarest, most complex scenarios: active attackers, evidence handling, disclosure and trade-offs under pressure.

**1.** GuardDuty alerts that credentials from your production EC2 role are being used from an IP outside AWS. The on-call engineer wants to terminate the instance immediately. What do you do instead, step by step?

> **Direction:** Isolate, do not destroy: swap to a forensic security group, detach from the load balancer and protect from scale-in, capture memory and snapshot disks, revoke active role sessions with an `aws:TokenIssueTime` deny, then enumerate what the credentials did in CloudTrail (§16, §5; `S16`, `S15`).

**2.** You revoked a leaked IAM user access key within 10 minutes. Two days later, the attacker is back. What did you miss?

> **Direction:** Persistence: the key was used to create other keys, users, roles, trust-policy changes or functions; enumerate everything the key did in CloudTrail and remove persistence and revoke in one coordinated step (§16; `S16`, `S15`).

**3.** An attacker had admin access to your company's chat workspace for at least a week. The incident team starts coordinating in a new channel there. What is wrong?

> **Direction:** The attacker can read your response; move coordination out of band to separately authenticated systems, and assume email or chat admins are compromised until proven otherwise (§16; `S16`).

**4.** An email arrives: "I have your customer database; pay 20 BTC or I publish it in 7 days", with a sample of 50 real records. Your bug bounty programme is also running. How do you handle it?

> **Direction:** Treat it as extortion and a live breach, not a bounty report: engage legal and IR, verify the sample to identify the source and time window, scope and contain the access path, and start the notification assessment clock (§16, §12; `S16`, `S12`).

**5.** Forensics shows an attacker accessed a service with read access to a bucket holding 12 million customer records. Logs show they listed the bucket but data-access logging was off. Legal asks "how many records were taken?". What is your honest answer, and what changes?

> **Direction:** Without data events you must assume everything readable was read; scope the notification to the full blast radius, then enable S3 data events and database auditing before the next incident (§15, §16; `S16`, `S12`).

**6.** During an incident, a senior engineer rebuilds the compromised Kubernetes node pool "to be safe" before anyone captured anything. Leadership now asks what the attacker did. What was lost, and what should the runbook say?

> **Direction:** Memory, running processes, container filesystem diffs and network state — the evidence; the runbook should cordon and isolate nodes and pods with network policy, capture volatile state and image digests, and only then rebuild from known-good sources (§16, §14; `S16`, `S15`).

**7.** An insider on the data team is suspected of exfiltrating customer data to a personal cloud account. How does your response differ from an external breach?

> **Direction:** Avoid tipping off: coordinate with HR and legal first, collect evidence quietly with chain of custody, restrict access in ways that look routine, and use detection on exports and bulk reads to scope (§16, §15; `S16`).

**8.** A security researcher reports a critical auth bypass in your API via `security.txt` and asks for a 90-day disclosure timeline. A product VP wants to ask them to keep it quiet indefinitely. What do you do?

> **Direction:** Honour coordinated disclosure with safe harbour: acknowledge, fix quickly, check logs for prior exploitation, agree the timeline, and credit the researcher; suppression invites public disclosure and erodes trust (§16; `S16`, `S17`).

**9.** Your honeytoken customer record was queried last night through your normal customer-support tool, by a real support agent's account, from their usual IP. What are the possibilities, and how do you investigate?

> **Direction:** A honeytoken hit is malicious by definition — the agent's session or device is compromised, or the agent is acting badly; preserve evidence, review that account's full activity, check for session theft (infostealer) and escalate as an insider or ATO case (§15, §16, §2; `S16`).

**10.** Your auth service's JWT private signing key may have been exposed in a compromised build container. There are 4 million active sessions. What do you do, and what does it cost?

> **Direction:** Emergency rotation: remove the old key from the JWKS immediately and force verifiers to refetch, accepting that every token dies; investigate forged tokens in logs, and move signing into KMS/HSM so the private key is never in a container (§1, §16; `S03`, `S10`).

**11.** An npm worm compromised a developer's token, published malicious versions of two of your public packages, and those versions were installed in your CI for 3 hours. What is the full scope of your response?

> **Direction:** Unpublish or deprecate the versions and notify users; rotate every secret exposed to CI and developer machines in that window, review cloud activity from those credentials, check for self-propagation to other packages, and move publishing to trusted publishing with no long-lived tokens (§13, §10, §16; `S14`).

**12.** Your GDPR 72-hour clock started when a SOC analyst flagged suspicious exports on Friday evening. On Monday morning, scoping is still incomplete. What do you do?

> **Direction:** Notify the supervisory authority within 72 hours with what is known, marked preliminary, and supplement in phases; record decisions and reasoning, and keep customer-facing statements accurate and non-speculative (§12, §16; `S12`, `S16`).

**13.** A breach investigation needs 14 months of logs, but your hot retention is 30 days and cold archives were never tested. What happens, and what baseline do you set?

> **Direction:** You probably cannot establish dwell time or scope, so notifications must assume worst case; set 90 days hot and a year or more cold for security logs, test restores, and never sample audit logs (§15; `S16`).

**14.** An attacker with admin access to your production account tried to delete CloudTrail and the log bucket but failed, and you still have a full record. Which architecture decisions made that possible?

> **Direction:** An organisation trail delivered to a separate logging account, SCP denies on logging changes, and Object Lock on the log bucket, so the audited account cannot touch its own audit trail (§15, §14; `S16`, `S15`).

**15.** During an incident, the attacker's session is still active in your admin panel. Should you kill it now or watch it to learn more? Argue both sides and decide.

> **Direction:** Watching gains intelligence on persistence and intent but risks more damage; decide by what they can reach (money movement or PII exfiltration argues for immediate containment), and if you act, remove all known persistence in one coordinated step (§16; `S16`).

**16.** A request-smuggling vulnerability at your edge was exploited for two weeks before detection, possibly delivering some users' responses to others. How do you scope which users were affected?

> **Direction:** Correlate proxy and back-end logs on connection IDs and request timing to find mismatched request/response pairs, identify sessions whose responses may have crossed, rotate those sessions, and treat it as a potential personal-data breach (§6, §15, §16; `S16`, `S12`).

**17.** After a supply-chain compromise, your team must decide whether to rebuild every production image and rotate every secret in the company — about two weeks of work — or target the affected services. How do you make that call?

> **Direction:** Decide from blast radius, not hope: what the compromised component could reach (credentials in CI, runtime identity, network), backed by SBOM and provenance data; if you cannot bound it, rebuild and rotate broadly, and record the decision (§13, §16, §17; `S14`, `S16`).

**18.** A post-incident review is turning into "who approved the risky PR". As the senior engineer in the room, how do you redirect it, and what outputs do you push for?

> **Direction:** Blameless framing: why the system made the mistake easy and which detection should have fired earlier; the outputs are a few owned, dated actions — paved-road fixes, detections and tests — not training for one person (§16, §17; `S16`, `S17`).

**19.** Your restore-from-backup plan after a ransomware-style cloud incident uses backups in the same account the attacker controlled. Why is that dangerous, and what should the design be?

> **Direction:** Backups may be deleted, encrypted or tampered with (including planted persistence); keep immutable backups in a separate account with separate credentials, verify integrity before restore, and rotate secrets as part of recovery (§16, §11, §14; `S16`, `S15`).

**20.** An enterprise customer contractually requires notification within 24 hours of any suspected incident. You have a suspected but unconfirmed cross-tenant data exposure from a caching bug. Legal wants to wait for confirmation. What do you advise?

> **Direction:** Contract clocks often run from suspicion; notify on time with a factual preliminary statement, contain first (purge caches, kill switch), and scope with cross-tenant access logs and the data-access map, updating the customer as facts firm up (§16, §4, §6; `S16`, `S09`).
