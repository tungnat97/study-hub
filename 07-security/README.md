[← back to the index](../README.md)

# Field 7 — Security for Backend Engineers

Senior candidates are expected to design securely by default, not to be pentesters. Legend: `B` `I`
`A` `X`.

---

## Nodes

#### S01 · Threat modelling & security mindset
`I` · Requires: — · Unlocks: S05, S06, S16
- Key: assets, trust boundaries, entry points, adversaries; **STRIDE** (spoofing, tampering,
  repudiation, information disclosure, DoS, elevation of privilege); defence in depth; least
  privilege; fail secure/closed; secure by default; never trust the client; the server is the only
  place authorisation happens; blast radius thinking.
- Q: "Threat model the feature you built last quarter in 3 minutes."
- Q: "Where are the trust boundaries in a Nest API behind a gateway calling a third-party payment
  provider?"

#### S02 · Authentication
`I` · Requires: S01 · Unlocks: S03, S04, S13, F11
- Key: password storage — **bcrypt/scrypt/argon2id** with per-user salt and a tuned work factor,
  never MD5/SHA/plain; peppering; password policies per NIST (length over complexity, breach-list
  checks, no forced rotation); login rate limiting and lockout vs enumeration; MFA (TOTP, WebAuthn/
  passkeys as the strong option); email verification, password reset tokens (single use, short TTL,
  hashed at rest, invalidate sessions on reset); "remember me" tokens; account recovery as the
  weakest link.
- Q: "Design password reset end to end and name three ways it's commonly broken."
- Q: "Why bcrypt and not SHA-256?" (deliberately slow + salted; hashing speed is the attacker's
  friend).

#### S03 · Sessions vs tokens
`A` · Requires: S02 · Unlocks: S04, A18, S13
- Key: server-side sessions (opaque id in an `HttpOnly; Secure; SameSite=Lax` cookie, revocable,
  needs a session store) vs **JWT** (header.payload.signature, stateless, verifiable, **not
  encrypted** — never put secrets in the payload); `alg: none` and algorithm-confusion attacks
  (RS256 verified as HS256 with the public key as the secret) — always pin the expected algorithm;
  `exp`/`nbf`/`iss`/`aud` validation; the revocation problem → short-lived access token + refresh
  token with **rotation and reuse detection**; where to store tokens in a browser (cookie >
  localStorage because of XSS); session fixation and regeneration on privilege change.
- Q: "Someone steals a JWT with 1 hour left. What can you do?"
- Q: "Explain refresh token rotation and what reuse detection catches."

#### S04 · OAuth2 & OIDC
`A` · Requires: S03 · Unlocks: A18, S05
- Key: roles (resource owner, client, authorisation server, resource server); **authorisation code +
  PKCE** for all interactive clients now, **client credentials** for machine-to-machine, device code
  for TVs/CLIs; implicit and password grants are deprecated; `state` for CSRF on the callback;
  redirect URI exact matching (open redirect = account takeover); scopes vs permissions;
  OIDC adds `id_token` and userinfo; token exchange and on-behalf-of for service chains; audience
  validation so a token for service A cannot be replayed to service B.
- Q: "Why PKCE, given the client secret already exists?" (public clients can't keep a secret; PKCE
  stops interception of the code).
- Q: "You SSO with Google. Walk through the flow and name what each parameter defends against."

#### S05 · Authorisation models
`A` · Requires: S01, S04, M29 · Unlocks: S09, SD13
- Key: RBAC (roles → permissions; simple, explodes into role sprawl), ABAC (attributes/policy —
  flexible, harder to audit), **ReBAC** (relationship graph — Google Zanzibar/OpenFGA/SpiceDB; the
  right model for "can user X view document Y because they're in folder Z's team"); policy engines
  (OPA/Cedar) and centralised decision + local enforcement; **object-level authorisation** enforced
  in the data access layer (scope every query by tenant/owner — the single most effective defence
  against IDOR); permission caching and its invalidation risk; admin impersonation and its audit
  requirements.
- Q: "Design permissions for a docs product with orgs, teams, folders, sharing links and guests."
- Q: "Where do you enforce authorisation: controller, service or repository? Argue it." (all three
  in depth, with row scoping at the data layer as the backstop).

#### S06 · OWASP Top 10 & the attack catalogue
`I` · Requires: S01 · Unlocks: S07, S08, S09, F26
- Key: be able to name and give one concrete example of each — broken access control (#1),
  cryptographic failures, injection, insecure design, security misconfiguration, vulnerable
  components, identification/authentication failures, software & data integrity failures, logging &
  monitoring failures, SSRF. Plus the API Top 10 (BOLA/BFLA, unrestricted resource consumption).
- Q: "What's the most common serious vulnerability in real APIs?" (broken object-level
  authorisation — and it never shows up in a scanner).

#### S07 · Injection
`A` · Requires: S06 · Unlocks: DB37, S14
- Key: **parameterised queries / prepared statements** always; why escaping and blocklists fail;
  ORM safety and the escape hatches that break it (raw SQL with template strings, `extra()`,
  dynamic `ORDER BY`/identifiers which cannot be parameterised — use an allowlist); NoSQL injection
  (`{$gt: ""}` in a Mongo filter from unvalidated JSON); command injection (never shell out with
  user input; pass argv arrays); LDAP/XPath/template injection; SSTI; **prompt injection** as the
  modern addition if the service calls an LLM (untrusted content is data, never instructions).
- Q: "Show me a query in your codebase that would be injectable and fix it."
- Q: "How do you support user-specified sort columns safely?"

#### S08 · Browser-adjacent attacks
`A` · Requires: S06, A19 · Unlocks: F26
- Key: XSS (stored/reflected/DOM; output encoding by context, CSP, `HttpOnly` cookies limit the
  damage) — a backend concern because you serve the data and the headers; **CSRF** (only for
  cookie/implicit credentials; `SameSite=Lax/Strict`, anti-CSRF token, verify Origin — and why a
  Bearer header is immune); **SSRF** (user-supplied URLs → block private ranges/metadata endpoint
  169.254.169.254, resolve-then-validate to beat DNS rebinding, use an egress proxy/allowlist);
  XXE (disable external entities); insecure deserialisation (Java/Python pickle/Node
  prototype pollution); open redirects; clickjacking headers.
- Q: "Your service fetches a URL the user supplies for OG previews. Threat model it."
- Q: "Do you need CSRF tokens for a JSON API with Bearer auth? Prove it."

#### S09 · Broken access control in practice
`A` · Requires: S05, S06, F06 · Unlocks: S16
- Key: **IDOR/BOLA** (`GET /invoices/42` returns someone else's — always scope by owner, never trust
  the id alone; UUIDs are obfuscation, not authorisation); function-level authz (admin endpoints
  reachable by guessing); **mass assignment** (DTO whitelists — `role`, `isAdmin`, `balance`,
  `tenantId` must not be bindable); missing authz on nested resources; excessive data exposure in
  responses (serialise explicitly, don't return the whole entity); authz bypass via a second entry
  point (GraphQL, batch endpoint, internal gRPC, admin tool).
- Q: "Here's a controller. Find the access-control bug." (expect a live code-reading exercise)
- Q: "How do you make IDOR structurally impossible rather than fixing it case by case?"
  (repository-level tenant/owner scoping, request-scoped context, tests that assert cross-tenant
  access fails, RLS in the database).

#### S10 · Transport, secrets & key management
`A` · Requires: S01, A11 · Unlocks: S11, S15, O19
- Key: TLS everywhere including internal traffic; HSTS; certificate lifecycle/expiry monitoring;
  secrets never in code, images, env-var dumps, logs or CI logs; secret managers (Vault/AWS Secrets
  Manager/KMS), dynamic/short-lived credentials, rotation without downtime (dual-key overlap),
  envelope encryption (DEK wrapped by KEK), what to do after a leak (rotate first, investigate
  second, and assume git history is public forever).
- Q: "A DB password was committed to git 8 months ago. Walk me through the response."

#### S11 · Applied cryptography
`A` · Requires: S10 · Unlocks: S12
- Key: encoding ≠ hashing ≠ encryption (base64 is not security); symmetric (AES-GCM — **never reuse
  a nonce**) vs asymmetric (RSA/ECDSA/Ed25519) and the hybrid pattern; HMAC for integrity and
  webhook signatures; constant-time comparison to prevent timing attacks; secure randomness
  (`crypto.randomBytes`, never `Math.random()` for tokens); don't roll your own; password hashing vs
  data hashing; signed URLs; at-rest encryption levels (disk vs column vs client-side) and what each
  actually protects against.
- Q: "Sign and verify a webhook. Which primitive, and what stops replay?" (HMAC over
  timestamp+body, constant-time compare, reject old timestamps, store recent ids).
- Q: "Encryption at rest protects you from what, exactly?" (stolen disks/backups — not from your own
  compromised app or a SQL injection).

#### S12 · Privacy, PII & compliance
`I` · Requires: S11, DB37 · Unlocks: S16
- Key: data classification and minimisation; PII/PHI/PCI scope reduction (tokenise card data,
  never store PAN); GDPR rights — access, portability, **erasure** (and how it collides with
  immutable logs, backups and event streams → crypto-shredding, retention windows); data residency;
  consent and lawful basis; retention policies enforced by jobs, not by intention; anonymisation vs
  pseudonymisation; logs and analytics as the most common accidental PII store; DPA/subprocessors.
- Q: "A user invokes the right to erasure. Which systems must change and which can't?" (DB, caches,
  search index, event log, warehouse, backups, third parties — give the honest answer for each).

#### S13 · Abuse, rate limiting & anti-automation
`I` · Requires: S02, S03, A08, M06 · Unlocks: SD11
- Key: per-user/per-IP/per-key limits, progressive delays, CAPTCHA as a last resort, account
  enumeration (identical responses and timing on login/reset/signup), credential stuffing defence
  (breach lists, device fingerprint, anomaly detection), scraping, resource-exhaustion attacks
  (huge payloads, zip bombs, expensive GraphQL queries, regex catastrophic backtracking = **ReDoS**),
  business-logic abuse (coupon reuse, referral farming) — often the real-world risk.
- Q: "How do you stop credential stuffing without annoying real users?"
- Q: "A regex on user input pinned a CPU at 100%. What happened and how do you prevent the class?"

#### S14 · Supply chain security
`I` · Requires: S07 · Unlocks: S15, O08
- Key: lockfiles committed and respected in CI (`npm ci`), dependency review and pinning, transitive
  dependency risk, typosquatting and malicious postinstall scripts, `npm audit`/Snyk/Dependabot and
  triaging noise, SBOM, base image minimalism (distroless) and scanning, provenance/signing (sigstore),
  build reproducibility, protecting CI credentials from third-party actions (pin by SHA).
- Q: "A transitive dependency has a critical CVE with no patch. Your options?"

#### S15 · Infrastructure & identity security
`A` · Requires: S10, S14, M26 · Unlocks: O11, SD13
- Key: least-privilege IAM roles per workload (no wildcard, no long-lived keys — use workload
  identity/IRSA), network segmentation and security groups, private subnets for databases, no public
  S3 buckets, bastion/SSM instead of SSH keys, **mTLS/SPIFFE** for service identity instead of shared
  secrets, egress filtering (limits exfiltration and SSRF), container security (non-root, read-only
  fs, drop capabilities, no privileged), K8s RBAC and namespace isolation, secrets in etcd encrypted.
- Q: "Your pod is compromised. What does the attacker reach, and how do you shrink that?"

#### S16 · Detection, logging & incident response
`I` · Requires: S01, S09, S12 · Unlocks: O16, O17
- Key: security-relevant audit logs (authn/authz decisions, privilege changes, data exports, admin
  actions) that are tamper-evident and separate from app logs; never log secrets/tokens/full PII;
  alerting on anomalies; an incident runbook — contain, eradicate, recover, notify (72h GDPR breach
  clock), blameless postmortem; responsible disclosure channel; tabletop exercises.
- Q: "How would you know if an attacker had been reading your customers' data for a month?"

#### S17 · Security in the SDLC
`I` · Requires: S06, S14 · Unlocks: O08
- Key: threat modelling at design time, security review in PRs (what you personally look for), SAST/
  DAST/secret scanning in CI and keeping the signal:noise usable, dependency policy, pentest and bug
  bounty cadence, security champions, secure defaults in the framework/template so the easy path is
  the safe path.
- Q: "What do you look for when reviewing a teammate's PR for security?" (authz on every new
  endpoint, input validation, output serialisation, new raw SQL, new dependency, secrets, logging of
  sensitive fields, new egress).

---

## Topological order (study waves)

```
Wave 0  S01
Wave 1  S02  S06
Wave 2  S03  S07  S08  S10  S14
Wave 3  S04  S11  S15  S17
Wave 4  S05  S12  S13
Wave 5  S09
Wave 6  S16
```

Cross-field parents: `A08` rate limiting, `A11` TLS, `A18/A19` API auth & CORS, `DB37` DB security,
`F06/F11/F26` framework, `M06` gateway, `M26` mesh, `M29` multi-tenancy.

**Near-certain questions:** JWT vs session and revocation (S03), how you prevent IDOR structurally
(S05/S09), password storage (S02), SSRF on a URL-fetching feature (S08), and "what do you look for
in a security code review" (S17).
