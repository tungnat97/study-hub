[← back to the field index](README.md)

# Security · Part 1 — Threat Modelling, Identity and Access

Nodes `S01`–`S05`.

---

## S01 · Threat modelling and the security mindset

`Intermediate` · Requires: — · Unlocks: `S05`, `S06`, `S16`

### Preface

Security is not a feature you add; it is a way of looking at a design. The core habit is asking, for
each part of a system: who could interact with this, what could they want, and what stops them?

Threat modelling is that habit done deliberately and written down. A senior engineer does a rough
version in their head during every design discussion.

### Details

#### 1. The four questions

**Theory.** The simplest useful framework: **What are we building?** (a diagram with data flows).
**What can go wrong?** (threats). **What are we going to do about it?** (mitigations). **Did we do a
good job?** (review).

**Example.** For a file-upload feature: the diagram shows browser → API → object storage → worker.
Threats: a malicious file stored and served to other users; a huge file exhausting storage or memory;
a user reading another tenant's file; a crafted filename escaping the intended path; the worker being
exploited by a malformed file. Mitigations follow from each, and none of them are obvious without
enumerating the threats first (`F17`).

**Advanced.** Do this at **design** time, not after implementation. Retrofitting security means
changing architecture, which rarely happens; a threat identified on a whiteboard costs an hour, and
the same threat found in a penetration test costs a sprint. Being the person who asks these questions
during design review is a concrete, visible senior behaviour worth describing in an interview.

#### 2. STRIDE

**Theory.** A checklist of threat categories, useful for making sure you have not missed a class:
**S**poofing (pretending to be someone else), **T**ampering (modifying data), **R**epudiation
(denying an action, with no evidence to the contrary), **I**nformation disclosure, **D**enial of
service, **E**levation of privilege.

**Example.** Applied to a payments API: spoofing → authentication and mTLS between services;
tampering → TLS and signed webhooks; repudiation → an immutable audit log (`S16`); disclosure →
authorisation and encryption; denial of service → rate limits (`A08`); elevation → careful
authorisation and least privilege.

**Advanced.** Repudiation is the one most often skipped and the one that matters for money and
compliance: if a customer disputes a transaction and you have no tamper-evident record of who did
what and when, you cannot resolve it. That is why an append-only ledger (`DB38`) and an audit log are
security controls, not just features.

#### 3. Trust boundaries

**Theory.** A trust boundary is a line where data moves from a less-trusted context to a more-trusted
one. Every boundary crossing needs validation and authorisation.

**Example.** The boundaries in a typical system: internet → gateway; gateway → service; service →
service; service → database; your system → a third party; and — frequently forgotten — an admin tool
→ production data. Each is a place where "where did this come from and may it do this?" must be
answered.

**Advanced.** The boundary people forget is **internal**. "It is inside the VPC so it is trusted" was
the old model, and one compromised container disproves it. Zero-trust means authenticating and
authorising every call regardless of network position (`S15`). The practical version: services verify
tokens even when the gateway already did, and network policy restricts which services can reach
which (`M26`).

#### 4. Defence in depth and failing closed

**Theory.** No single control is reliable, so layer them so that a failure in one does not become a
breach. And when a control fails, it should deny rather than allow.

**Example.** Layers preventing cross-tenant data access: the application filters by tenant; the
repository enforces it structurally; row-level security enforces it in the database (`DB37`); tests
assert cross-tenant access fails; and audit logs would show it if it happened. Any one of these can
have a bug; all five failing simultaneously is unlikely.

**Advanced.** Fail-closed is not always the right default, and being able to say which is which shows
judgement: an authorisation service being unreachable should **deny** (fail closed); a rate limiter
being unreachable should usually **allow** (fail open), because refusing all traffic is a worse
outcome than briefly unlimited traffic. Decide per control, deliberately, and write it down (`A08`).

### Interview questions

- "Threat model the feature you built last quarter in three minutes."
- "Where are the trust boundaries in a Nest API behind a gateway calling a payment provider?"
- "What does STRIDE's R stand for and why does it matter?"
- "Name a control that should fail open and one that should fail closed."

---

## S02 · Authentication

`Intermediate` · Requires: `S01` · Unlocks: `S03`, `S04`, `S13`, `F11`

### Preface

Authentication is proving who someone is. Almost every system still does it with passwords, so the
mechanics of storing them safely are non-negotiable knowledge.

The rest of it — reset flows, multi-factor, lockout — is where real systems are actually broken,
because the password storage is usually fine and the recovery path is an afterthought.

### Details

#### 1. Password storage

**Theory.** Never store passwords, and never store a plain hash. Use a **deliberately slow**,
**salted** password-hashing function: **argon2id** (preferred), **scrypt**, or **bcrypt**. The work
factor is tuned so hashing takes a meaningful fraction of a second on your hardware.

**Example.** Why SHA-256 is wrong: it is designed to be fast, so an attacker with a GPU tries billions
of candidates per second against a stolen database. bcrypt at a sensible cost factor takes ~100ms per
attempt, reducing that to a handful per second per core. The **salt** (unique per user, stored
alongside the hash) prevents one precomputed table from attacking every user at once.

**Advanced.** Two refinements. **Pepper**: a secret value, held outside the database (in a key manager
or environment), mixed into the hash — so a stolen database alone is not enough to attack offline.
And the work factor must be **revisited**: a cost tuned in 2018 is weak now, so re-hash on next
successful login when you raise it. argon2id is preferred because it is memory-hard, which makes GPU
and ASIC attacks much more expensive than bcrypt does.

#### 2. Login flow hygiene

**Theory.** The login endpoint is the most attacked part of most applications. It needs rate
limiting, protection against credential stuffing, and care not to reveal which accounts exist.

**Example.** The controls: rate limit per account **and** per IP (`A08`); progressive delays or
temporary lockout after repeated failures — with the caveat that lockout by username is itself a
denial-of-service against a known user; check credentials against known-breached password lists;
and return an **identical** response for "no such user" and "wrong password".

**Advanced.** Response **timing** leaks too: if a missing user returns in 5ms and a wrong password in
100ms (because the hash was computed), an attacker can enumerate accounts by timing alone. The fix is
to hash a dummy password when the user does not exist, so both paths cost the same. That detail is a
reliable signal of genuine security awareness (`S13`).

#### 3. Password reset

**Theory.** The reset flow is a way to obtain an account without the password, so it must be at least
as strong as the login. This is where real breaches happen.

**Example.** The correct design: generate a cryptographically random token (`crypto.randomBytes`,
never `Math.random()`); **store a hash of it**, not the token, so a database leak does not grant
resets; make it single-use and short-lived (15-60 minutes); bind it to the user; invalidate all
existing sessions and refresh tokens when the password changes; and notify the user by email that
their password changed. Respond identically whether or not the address is registered.

**Advanced.** Two subtle ones: the reset link must not leak through the **Referer** header to
third-party resources on the landing page (use a redirect that strips it, or `Referrer-Policy`); and
host-header poisoning — if you build the reset URL from the request's `Host` header, an attacker can
cause your email to contain a link to their domain. Use a configured base URL, never the request
header.

#### 4. Multi-factor and passkeys

**Theory.** A second factor means a stolen password alone is insufficient. In increasing strength:
SMS (vulnerable to SIM swapping), TOTP authenticator apps, and **WebAuthn/passkeys** (hardware-backed,
phishing-resistant).

**Example.** Passkeys are phishing-resistant because the credential is **bound to the origin**: a
fake site cannot obtain a usable signature, no matter how convincing it is. TOTP codes can be
phished in real time by a proxy site; passkeys cannot. For a new system, passkeys with a password
fallback is the modern recommendation.

**Advanced.** Recovery is the weak point of every MFA scheme: if losing a phone means losing the
account, users will not enable it, and if recovery is easy, it becomes the attack path. Design it
explicitly — backup codes generated once and stored securely, or multiple registered devices — and
recognise that support-driven recovery ("prove who you are to an agent") is often the real
vulnerability, because social engineering targets people rather than code.

### Interview questions

- "Design password reset end to end and name three ways it is commonly broken."
- "Why bcrypt and not SHA-256?"
- "How would an attacker enumerate accounts, and how do you stop them?"
- "Why are passkeys phishing-resistant when TOTP is not?"

---

## S03 · Sessions versus tokens

`Advanced` · Requires: `S02` · Unlocks: `S04`, `A18`, `S13`

### Preface

Once a user has authenticated, subsequent requests need to carry proof. The two mechanisms are a
**server-side session** referenced by an opaque cookie, and a **self-contained signed token** (a JWT).

The trade is simple and is asked constantly: sessions are revocable and need a lookup; tokens need no
lookup and cannot be revoked before expiry. Everything else follows from that sentence.

### Details

#### 1. Sessions

**Theory.** The server stores session state and gives the client an opaque random id in a cookie. Each
request looks up the session. Revocation is deleting a row.

**Example.** The cookie flags are the whole security surface: `HttpOnly` (JavaScript cannot read it,
so an XSS cannot steal it), `Secure` (HTTPS only), `SameSite=Lax` or `Strict` (limits CSRF, `S08`),
a sensible `Path` and `Domain`, and a short expiry with sliding renewal. Store sessions in Redis or a
database so any instance can serve any request (`F28`).

**Advanced.** **Session fixation**: if the session id does not change when the user authenticates, an
attacker who planted a known id before login now holds an authenticated session. Always regenerate the
id on login, and on any privilege change. Also invalidate all of a user's sessions on password change
or on suspicious activity — which is trivially possible with sessions and genuinely hard with tokens.

#### 2. JWTs

**Theory.** Three base64url parts — header, payload, signature. The server verifies the signature and
trusts the claims without any lookup. **Signed, not encrypted**: anyone holding it can read the
payload.

**Example.** So never put anything sensitive in a JWT payload. And the validation must check, every
time: the **algorithm** (pinned, not read from the token's header), the signature, `exp` and `nbf`,
`iss`, and `aud` (`A18`).

**Advanced.** The two historical attacks are worth naming. `alg: none` — a token claiming no algorithm
that a naive library accepts as valid. And **algorithm confusion** — a token signed with HMAC using
the RSA **public key** as the secret, which verifies successfully if the library picks the algorithm
from the token. Both are fixed in modern libraries and both reappear in hand-rolled verification. Pin
the expected algorithm explicitly.

#### 3. The revocation problem

**Theory.** A JWT is valid until it expires. Logging out, disabling an account, or revoking a
permission has no effect on a token already issued.

**Example.** The standard mitigation is **short-lived access tokens plus refresh tokens**: an access
token of 5-15 minutes, and a long-lived refresh token used to obtain new ones. Revocation then takes
effect within the access token's lifetime, because the refresh will be refused.

**Advanced.** **Refresh token rotation with reuse detection** is the important refinement: each
refresh issues a new refresh token and invalidates the old one. If an old one is ever presented again,
either it was stolen or it was replayed — so invalidate the **entire family**, forcing re-authentication.
That converts a stolen refresh token from a permanent compromise into a detectable event. For
**immediate** revocation you need server state anyway — a denylist of token ids, or a per-user
"issued before" timestamp — at which point you have partially reinvented sessions, which is exactly
the honest conclusion to draw.

#### 4. Where to store the token in a browser

**Theory.** `localStorage` is readable by any JavaScript on the page, so an XSS steals the token. An
`HttpOnly` cookie is not readable by JavaScript, and is sent automatically, which reintroduces CSRF.

**Example.** The comparison that matters:
- **localStorage** — immune to CSRF, vulnerable to XSS, and the token can be exfiltrated and used
  anywhere until expiry.
- **`HttpOnly` cookie** — vulnerable to CSRF (mitigated by `SameSite` and tokens), immune to theft by
  XSS; an XSS can still *make requests* as the user but cannot steal a reusable credential.

The second is generally stronger, because a stolen token is worse than a session-bound attack.

**Advanced.** The pragmatic modern pattern: the access token in memory (a JavaScript variable, gone on
refresh) and the refresh token in an `HttpOnly`, `SameSite=Strict` cookie scoped to the refresh
endpoint. XSS cannot read the refresh token; the access token is short-lived and not persisted. It is
more work and it is the design to describe when asked how you would do it properly.

### Interview questions

- "Someone steals a JWT with an hour left. What can you do?"
- "Explain refresh token rotation and what reuse detection catches."
- "Cookie or localStorage for the token? Defend it."
- "What is algorithm confusion in JWT verification?"

---

## S04 · OAuth2 and OpenID Connect

`Advanced` · Requires: `S03` · Unlocks: `A18`, `S05`

### Preface

OAuth2 is about **delegated authorisation**: letting an application act on a user's behalf at another
service without seeing their password. OpenID Connect adds **identity** on top, so it can also tell
you who the user is.

Most teams consume it rather than implement it, so the valuable knowledge is which flow to use, and
what each parameter defends against.

### Details

#### 1. The roles and the main flow

**Theory.** Four roles: the **resource owner** (the user), the **client** (your application), the
**authorisation server** (issues tokens), and the **resource server** (the API accepting them).

**Example.** The authorisation code flow: your app redirects the user to the authorisation server →
they log in and consent → the server redirects back to your registered `redirect_uri` with a
short-lived **code** → your backend exchanges the code (plus a client secret or PKCE verifier) for
tokens. The code goes through the browser; the tokens do not — which is the point.

**Advanced.** The `state` parameter is a random value you generate, send, and verify on return. It
binds the callback to the session that started the flow, preventing **CSRF on the callback** — an
attacker completing a flow with their own account and having your user's session silently linked to
it. Omitting `state` is a real vulnerability, not a formality.

#### 2. Choosing a grant

**Theory.**
- **Authorisation code + PKCE** — every interactive client: web applications, single-page
  applications, mobile. PKCE is now recommended even for confidential clients.
- **Client credentials** — machine-to-machine, no user involved.
- **Device code** — input-constrained devices (a TV, a CLI).
- **Implicit** and **resource owner password** — deprecated; do not propose them.

**Example.** **PKCE** works by the client generating a random `code_verifier`, sending its hash as
`code_challenge` at the start, and presenting the verifier when exchanging the code. An attacker who
intercepts the authorisation code — from a mobile URL scheme hijack, a browser history entry, a
referrer leak — cannot exchange it without the verifier.

**Advanced.** The implicit flow returned tokens directly in the URL fragment, where they end up in
browser history, referrer headers and logs. It existed because browsers could not make cross-origin
token requests; CORS solved that, so the flow is obsolete. Being able to explain *why* it was
deprecated, rather than just that it was, is the senior version of this answer.

#### 3. OIDC and what the tokens mean

**Theory.** OAuth2 alone gives an **access token** — an opaque or structured credential for calling
an API. OIDC adds an **`id_token`**: a JWT about the *user*, for your application to consume, plus a
`userinfo` endpoint.

**Example.** The distinction that is routinely confused: the `id_token` is for **your client** to
learn who the user is — validate it, read the claims, then discard it. The `access_token` is for
**calling APIs** and should be passed to them unchanged. Sending an `id_token` to an API as a bearer
credential is a misuse, and an API that accepts one is misconfigured.

**Advanced.** Validation of an `id_token` requires checking `iss`, `aud` (must be your client id),
`exp`, the signature against the provider's JWKS, and the `nonce` you sent (which prevents replay of
a token obtained elsewhere). Fetch and cache JWKS, key by `kid`, and handle rotation — a provider
rotating keys while you cache indefinitely is a scheduled outage.

#### 4. Redirect URIs and scopes

**Theory.** The `redirect_uri` must be registered and matched **exactly**. Scopes limit what the token
permits.

**Example.** Loose redirect matching is an account-takeover vector: if the server allows any path
under your domain, and any part of your domain has an open redirect or hosts user content, an attacker
can have the authorisation code delivered to themselves. Exact matching, including scheme, host, port
and path, is the only safe configuration.

**Advanced.** Scopes are coarse and are not a substitute for authorisation: `scope: orders.read` says
this application may read orders, not *which* orders. The resource server must still enforce that
this user may read this order (`S05`, `S09`). Teams that treat scopes as authorisation end up with an
API where any application with a valid token can read anything of that type.

### Interview questions

- "Why PKCE, given the client secret already exists?"
- "You SSO with Google. Walk through the flow and name what each parameter defends against."
- "What is the difference between an `id_token` and an `access_token`?"
- "Are scopes authorisation?"

---

## S05 · Authorisation models

`Advanced` · Requires: `S01`, `S04`, `M29` · Unlocks: `S09`, `SD13`

### Preface

Authorisation is deciding whether this principal may perform this action on this resource. It is the
number one category of real-world API vulnerability, and it is mostly a modelling problem rather than
a cryptographic one.

The models scale in expressiveness: **roles** for simple systems, **attributes** for conditional
rules, and **relationships** for anything with sharing, hierarchies or teams.

### Details

#### 1. RBAC

**Theory.** Users have roles; roles have permissions; permissions gate actions. Simple, auditable,
and it grows badly when rules depend on the specific resource.

**Example.** `admin`, `editor`, `viewer` works for a small product. It breaks at the first requirement
like "an editor may edit only documents in their own team", because that depends on the resource, not
on the user. The usual response is to invent `editor_team_a`, `editor_team_b` — **role explosion** —
which becomes unmanageable and impossible to audit.

**Advanced.** RBAC is the right starting model and should be kept for **coarse** capabilities
("may access the admin area", "may issue refunds") while resource-level decisions are handled by a
different mechanism. Mixing the two deliberately — roles for capabilities, ownership or relationships
for objects — is what most real systems do and is a good answer to "which model would you use".

#### 2. ABAC

**Theory.** Decisions are computed from attributes of the subject, the resource, the action and the
environment: `allow if user.department == document.department and user.clearance >= document.level`.

**Example.** Expressive and general. The costs are that answering "who can access this document?"
requires evaluating the policy against every user, which makes auditing hard; and policies scattered
through application code become impossible to reason about. Policy engines (OPA with Rego, Cedar)
centralise the rules so they can be tested, versioned and reviewed.

**Advanced.** The architectural pattern is **PDP and PEP**: a policy decision point evaluates the
rules, and policy enforcement points in each service ask it. Calling a remote PDP per request adds
latency, so the usual deployment is the engine as a **sidecar or library** with policies distributed
to it — decisions stay local, and policy management is central (`M26`).

#### 3. ReBAC

**Theory.** Relationship-based access control: permissions derive from a graph of relationships.
"You may view this document because you are a member of a team that has editor access to its parent
folder." Google's Zanzibar is the canonical design; OpenFGA and SpiceDB are open implementations.

**Example.** This is the correct model for anything with sharing and hierarchy — documents, folders,
repositories, projects. Relationships are stored as tuples (`document:1#viewer@team:eng#member`), and
a check walks the graph. It answers both "can this user do this?" and "who can access this?" — the
latter being what RBAC and ABAC struggle with.

**Advanced.** Zanzibar's hard problem is consistency: permissions are stored separately from
application data, so a revocation must not be visible before the change that caused it ("the new
enemy problem" — someone removed from a document could otherwise still see it via a stale cache).
Zanzibar solves it with **zookies**, consistency tokens that ensure a check is evaluated against a
snapshot at least as new as the last relevant change. Naming that problem is a genuinely advanced
signal.

#### 4. Enforcing it where it cannot be forgotten

**Theory.** Whatever the model, the decision must be enforced somewhere a developer cannot skip. The
best place is the data access layer, so every query is scoped by construction.

**Example.** The layered approach: a route guard for coarse capability (`F11`); a service-layer check
for the business rule; and **the repository automatically scoping every query** by tenant and
ownership from the request context — so a query that forgets returns nothing rather than everything.
Add row-level security in the database as a final backstop (`DB37`), and tests that attempt
cross-tenant access and assert failure.

**Advanced.** Caching authorisation decisions is tempting and dangerous: a cached "yes" outlives a
revocation. If you cache, keep the TTL very short (seconds), include a version of the user's
permissions in the cache key so any change invalidates everything, and never cache a **deny** into a
later **allow**. This is the same invalidation problem as `Q03` with a much worse failure mode.

### Interview questions

- "Design permissions for a docs product with orgs, teams, folders, sharing links and guests."
- "Where do you enforce authorisation: controller, service or repository? Argue it."
- "What is role explosion and how do you avoid it?"
- "Would you cache authorisation decisions?"
