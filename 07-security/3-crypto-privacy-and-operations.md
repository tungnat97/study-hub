[← back to the field index](README.md)

# Security · Part 3 — Secrets, Crypto, Privacy and Operations

Nodes `S10`–`S17`.

---

## S10 · Transport security, secrets and key management

`Advanced` · Requires: `S01`, `A11` · Unlocks: `S11`, `S15`, `O19`

### Preface

A secret is anything that grants access: a database password, an API key, a signing key, a token.
Managing them badly is one of the most common ways organisations get breached, and the mistakes are
mundane — a key in a repository, a credential that never rotates, a secret printed in a log.

The two rules: secrets never live in code or images, and every secret must be rotatable without
downtime.

### Details

#### 1. Where secrets live

**Theory.** The progression, worst to best: hard-coded in source; in a config file in the repository;
in a `.env` file on the server; in environment variables injected by the platform; mounted as a file
from a secret manager; fetched at runtime from a secret manager with short-lived credentials.

**Example.** A realistic good setup: secrets stored in a manager (Vault, AWS Secrets Manager, Google
Secret Manager); the platform injects them into the pod as files or environment variables at start;
developers never see production secrets at all; and local development uses a separate set of
throwaway values.

**Advanced.** Environment variables are convenient and leak in more places than people expect: they
appear in `/proc/<pid>/environ`, in crash dumps, in many error-reporting integrations that capture
the environment, in child processes, and in `docker inspect`. A **mounted file** read once at startup
is meaningfully better for high-value secrets, and supports rotation without a restart if you re-read
it (`O19`).

#### 2. Rotation

**Theory.** Every secret should have a defined lifetime and a rotation process that does not require
downtime. If rotation is painful, it will not happen, and a credential from four years ago is still
valid.

**Example.** Zero-downtime rotation needs an **overlap**: create the new credential, deploy so the
application accepts both, switch to the new one, then revoke the old. For signing keys, publish both
in the JWKS with different `kid` values so tokens signed with either verify (`A18`). Design for two
valid keys at once from the beginning — retrofitting it means a flag day.

**Advanced.** The strongest version is **dynamic secrets**: the application asks the secret manager
for a database credential at startup and receives one valid for an hour, created on demand and
automatically revoked. Nothing long-lived exists to steal. Vault's database engine does this, and
cloud IAM database authentication is the managed equivalent. It is the direction to describe when
asked how you would do it properly.

#### 3. When a secret leaks

**Theory.** Assume anything committed to a repository is public forever. Rewriting history does not
help — it has been cloned, cached, forked, and scanned by bots within minutes.

**Example.** The response order matters: **rotate first**, investigate second. Revoke the credential,
issue a new one, then determine what it could access, review logs for use of it during the exposure
window, and check whether anything was accessed. Only then clean up the repository, and add automated
secret scanning (gitleaks, GitHub secret scanning) as a pre-commit hook and a CI check so it does not
recur.

**Advanced.** GitHub secret scanning with **push protection** blocks the commit before it lands, which
is the only intervention that actually prevents the problem. And note that secrets leak into more
places than repositories: CI logs (mask them), error trackers (scrub the environment), client-side
bundles (an API key in a frontend build is public by definition), and Docker image layers (a secret
used during build remains in the layer even if deleted in a later one — use build secrets).

#### 4. Transport

**Theory.** TLS everywhere, including between internal services (`A11`). "It is inside the VPC" is not
a security boundary — one compromised container is enough.

**Example.** The checklist: TLS 1.2 minimum (prefer 1.3); HSTS so browsers refuse plaintext;
certificate expiry monitored with weeks of notice; internal traffic encrypted via mTLS or a mesh
(`M26`); and no plaintext credentials in query strings, ever — URLs appear in access logs, browser
history and referrer headers.

**Advanced.** Certificate expiry is worth naming as an operational security issue: it is the single
most common TLS-related outage, it is entirely predictable, and it is still missed. Automate issuance
and renewal (cert-manager, ACME), alert 30 days ahead, and include **internal** certificates — which
are the ones nobody monitors (`M31`).

### Interview questions

- "Where does a secret live from developer laptop to production pod?"
- "A database password was committed to git eight months ago. Walk me through the response."
- "How do you rotate a signing key without downtime?"
- "Why are environment variables a weak place for high-value secrets?"

---

## S11 · Applied cryptography

`Advanced` · Requires: `S10` · Unlocks: `S12`

### Preface

You will not implement cryptography, and you do need to choose and combine primitives correctly — and
to avoid the handful of mistakes that turn a correct algorithm into no protection at all.

Start from a clear distinction that is constantly confused: **encoding** is reversible and provides no
security; **hashing** is one-way; **encryption** is reversible with a key. Base64 is encoding.

### Details

#### 1. The primitives and what they are for

**Theory.**
- **Symmetric encryption** (AES-GCM, ChaCha20-Poly1305) — one key encrypts and decrypts. Fast; for
  bulk data. Use an **authenticated** mode (GCM) so tampering is detected.
- **Asymmetric encryption / signatures** (RSA, ECDSA, Ed25519) — a key pair. Slow; used to establish
  keys and to sign.
- **Hashing** (SHA-256) — a fixed-size fingerprint; one-way; for integrity and identifiers.
- **Password hashing** (argon2id, bcrypt) — deliberately slow and salted; *only* for passwords
  (`S02`).
- **HMAC** — a keyed hash proving both integrity and authenticity; for webhook signatures and tokens.

**Example.** The hybrid pattern is everywhere: asymmetric cryptography agrees a symmetric key, and the
symmetric key encrypts the data. TLS does exactly this (`A11`), and so does encrypted messaging,
because asymmetric operations are orders of magnitude slower.

**Advanced.** The single most dangerous mistake with AES-GCM is **nonce reuse**: encrypting two
different messages with the same key and nonce leaks the XOR of the plaintexts and destroys the
authentication guarantee entirely. Nonces must be unique per key — use a counter or a random 96-bit
value with a key-rotation policy, never a constant. This is a classic interview question for anyone
claiming crypto familiarity.

#### 2. Randomness and comparison

**Theory.** Security tokens must come from a cryptographically secure random source. And comparing
secrets must take **constant time**, or the comparison itself leaks the answer.

**Example.**

```js
// wrong: predictable; Math.random is not cryptographically secure
const token = Math.random().toString(36);

// right
const token = crypto.randomBytes(32).toString('base64url');

// wrong: === returns early on the first differing byte — a timing oracle
if (signature === expected) { ... }

// right
if (crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) { ... }
```

**Advanced.** The timing attack is practical over a network for local services and has been
demonstrated across the internet with enough samples. Use constant-time comparison for **any**
secret: signatures, tokens, API keys, MACs. Note `timingSafeEqual` throws if the buffers differ in
length, so compare lengths separately — and be aware that revealing the length is usually acceptable.

#### 3. Signing webhooks

**Theory.** A receiver needs to know a webhook genuinely came from you and has not been replayed
(`A16`).

**Example.** The standard construction: `HMAC-SHA256(secret, timestamp + "." + raw_body)`, sent in a
header with the timestamp. The receiver recomputes it, compares in constant time, and rejects
anything whose timestamp is more than a few minutes old. Including the timestamp **inside** the signed
content is what prevents replay — signing only the body lets an attacker resend an old valid message
forever.

**Advanced.** Sign the **raw bytes**, not the parsed-and-re-serialised JSON: key order, whitespace and
unicode escaping change the bytes, so a receiver that parses before verifying will fail on valid
messages. This means the receiver's framework must expose the raw body, which in Nest and Express
requires configuring the body parser to retain it — a genuinely common integration bug.

#### 4. Encryption at rest, and what it protects

**Theory.** Three levels, protecting against different things. **Disk or volume encryption** protects
a stolen disk or a mishandled backup. **Column or field encryption** protects against database
access. **Client-side encryption** protects against your own server.

**Example.** Be precise, because this is frequently overstated: full-disk encryption does **not**
protect against SQL injection, a compromised application, a curious administrator with query access,
or an accidentally public backup that is decrypted on restore. To the running database, the data is
plaintext. It protects against physical theft and decommissioned hardware, which is a real and
narrow threat.

**Advanced.** **Envelope encryption** is the standard pattern for field-level encryption: a data key
encrypts the data, a key-encryption key in a KMS encrypts the data key, and the encrypted data key is
stored alongside the ciphertext. Rotating the master key re-encrypts only the data keys, not the data.
It also enables **crypto-shredding** — destroy a subject's data key and their data is permanently
unreadable, which is how you satisfy deletion in immutable stores (`S12`, `M18`).

### Interview questions

- "Sign and verify a webhook. Which primitive, and what stops replay?"
- "Encryption at rest protects you from what, exactly?"
- "Why does nonce reuse break AES-GCM?"
- "Why is `===` the wrong way to compare a signature?"

---

## S12 · Privacy, personal data and compliance

`Intermediate` · Requires: `S11`, `DB37` · Unlocks: `S16`

### Preface

Personal data brings obligations that are not negotiable by engineering preference: people can ask
what you hold, ask for it back, and ask you to delete it — and you must be able to do all three.

The engineering consequence is that you need to **know where personal data is**, which is harder than
it sounds once data has flowed into caches, search indexes, event logs, warehouses and backups.

### Details

#### 1. Minimisation and classification

**Theory.** The cheapest way to protect data is not to hold it. Collect only what you need, keep it
only as long as you need it, and classify what you have so that controls can be applied.

**Example.** Classification in practice: tag each field as public, internal, personal, or sensitive
(health, financial, biometric). Then the rules follow — sensitive fields are encrypted at the field
level (`S11`), personal fields are excluded from logs (`F08`) and from non-production environments,
and retention periods are enforced by scheduled jobs rather than good intentions.

**Advanced.** The highest-leverage decision is not storing something at all. Card numbers are the
canonical example: **tokenise** them with the payment provider and store only a token, and your PCI
scope shrinks from the whole system to nearly nothing. Ask "could we hold a reference instead of the
value?" for every sensitive field.

#### 2. The rights you must implement

**Theory.** Under GDPR and similar regimes: **access** (what do you hold about me), **portability**
(give it to me in a machine-readable form), **rectification**, **erasure** ("right to be forgotten"),
and **restriction of processing**.

**Example.** Each is an engineering feature with a deadline (typically 30 days), and each requires
knowing where the data is. Access and portability need a tool that gathers a subject's data from every
store. Erasure needs the same map plus a deletion path for each.

**Advanced.** Erasure collides with several things you have built deliberately: an append-only ledger
(`DB38`), an event log (`M18`), analytics aggregates, search indexes, caches, third-party processors,
and backups. The accepted answers: **crypto-shredding** for immutable stores (`S11`);
**anonymisation** rather than deletion where the record must survive for accounting (replace identity
with a non-reversible token, keeping the transaction); a bounded **backup retention window** with
deletion re-applied on restore, documented in your privacy policy; and a documented list of
processors you must notify.

#### 3. Where personal data leaks accidentally

**Theory.** Data escapes the database into places with much weaker controls and much longer retention.

**Example.** The usual suspects: application **logs** (a logged request body containing an email);
**error trackers** (Sentry capturing request data and local variables); **analytics** events;
**caches** (`Q03`); **search indexes** (`DB31`); **event streams** (`Q19`); the **data warehouse**
(`DB32`); and **non-production environments** seeded with a copy of production.

**Advanced.** Production data in staging is extremely common and hard to defend: staging usually has
weaker access controls, more people with access, and no audit. The fix is synthetic or anonymised
data, and it needs investment to be usable — which is why it is often skipped. Raising it as a risk
with a concrete mitigation is a mature contribution.

#### 4. Compliance frameworks, briefly

**Theory.** **SOC 2** is an audit of your controls over security, availability, confidentiality and
privacy. **PCI DSS** applies if you handle card data. **HIPAA** applies to health data in the US.
**GDPR** and equivalents apply to personal data by jurisdiction.

**Example.** What they mean day to day for a backend engineer: access reviews and least privilege
(`DB37`); audit logging of access to sensitive data (`S16`); change management (code review, approvals,
deployment records — which your pipeline produces automatically, `O08`); encryption in transit and at
rest; vendor and subprocessor management; and evidence that all of this actually happens.

**Advanced.** The reason to build these in early is that retrofitting is far more expensive: adding
audit logging to a mature system, or making access reviews possible when the application connects as
the table owner (`DB37`), is months of work. A system designed with least privilege, per-user
credentials and structured audit events passes an audit almost incidentally.

### Interview questions

- "A user invokes the right to erasure. Which systems must change and which cannot?"
- "How do you satisfy deletion when the data is in six months of backups?"
- "Where does personal data leak outside the database?"
- "How would you reduce PCI scope?"

---

## S13 · Abuse, rate limiting and anti-automation

`Intermediate` · Requires: `S02`, `S03`, `A08`, `M06` · Unlocks: `SD11`

### Preface

Not every attack is a vulnerability. Much of the damage comes from legitimate operations performed at
illegitimate scale: credential stuffing, scraping, coupon abuse, and resource exhaustion.

The defences are mostly about raising the cost for an attacker without raising it for real users —
which is a product decision as much as a technical one.

### Details

#### 1. Credential stuffing

**Theory.** Attackers take username and password pairs from other services' breaches and try them
against yours. Because people reuse passwords, a small percentage succeed — and every one is a valid
login, so nothing looks anomalous per request.

**Example.** Defences that work: check passwords against breach corpora (Have I Been Pwned's k-anonymity
API lets you do this without sending the password); rate limit per account **and** per IP **and** per
device fingerprint (`A08`); require additional verification for a login from a new device or an
unusual location; and offer MFA (`S02`). Blocking on failure count alone is weak — attackers spread
attempts across many accounts and many addresses, so each individual counter stays low.

**Advanced.** The most effective signal is usually **aggregate**: a sudden rise in the ratio of failed
to successful logins across the whole service, or a spike in logins from one autonomous system.
Per-account thresholds miss a distributed attack entirely. Monitoring that ratio, with an alert, is
cheap and catches what per-request limits do not (`S16`).

#### 2. Account enumeration

**Theory.** Any difference in behaviour between "this account exists" and "it does not" lets an
attacker build a list of valid users — which makes every subsequent attack cheaper.

**Example.** The leaks: different error messages on login; different messages on password reset;
signup rejecting a taken address; different response **times** (`S02`); and different status codes.
The fix is identical responses and identical timing, with the actual outcome communicated through the
email channel ("if this address is registered, we have sent a link").

**Advanced.** This conflicts with usability — "that email is already registered" is genuinely helpful —
and the honest answer acknowledges the trade. For a consumer service where membership is not
sensitive, a rate-limited signup check may be acceptable. For anything where membership itself is
sensitive (a health service, a dating service), it is not. Deciding deliberately is the point.

#### 3. Resource exhaustion

**Theory.** An attacker makes a request that is cheap to send and expensive to serve, which is a
denial of service without any volume.

**Example.** The catalogue: a huge request body; a deeply nested JSON document (a parser recursion
attack); a compressed payload that expands enormously (a **zip bomb**); a pagination parameter of a
million (`A05`); an unbounded GraphQL query (`A15`); an expensive search term; a file upload that is
never completed; and **ReDoS** — a regular expression with catastrophic backtracking pinned to 100%
CPU by a crafted input.

**Advanced.** ReDoS deserves specific attention because it is easy to introduce accidentally: a
pattern with nested quantifiers such as `(a+)+$` takes exponential time on a non-matching input.
Defences: avoid nested quantifiers and ambiguous alternation, never build regular expressions from
user input, use a linter (`eslint-plugin-security`, `safe-regex`) and consider a
non-backtracking engine (RE2) for user-supplied patterns. In Node this blocks the event loop
entirely (`C03`), so one request takes down the whole process.

#### 4. Business-logic abuse

**Theory.** The rules of your product, exploited at scale. No scanner finds these; they come from
understanding the domain.

**Example.** Coupon codes applied repeatedly; referral schemes farmed with fake accounts; free-tier
limits evaded by creating accounts; a refund flow exploited by returning a different item; price
arbitrage between two endpoints that compute totals differently; inventory held by adding to cart and
never checking out.

**Advanced.** The defences are domain-specific and mostly about **idempotency and limits**: coupon
redemptions enforced by a unique constraint rather than a check (`DB05`); per-account and per-payment-
method limits; velocity checks; and holds with expiry rather than indefinite reservations (`SD10`).
Raising these during a design review — "what stops someone applying this twice?" — is a distinctly
senior contribution and a very good thing to describe in an interview.

### Interview questions

- "How do you stop credential stuffing without annoying real users?"
- "A regex on user input pinned a CPU at 100%. What happened and how do you prevent the class?"
- "How would you detect a distributed login attack?"
- "What business-logic abuse would you look for in a referral scheme?"

---

## S14 · Supply chain security

`Intermediate` · Requires: `S07` · Unlocks: `S15`, `O08`

### Preface

Your application is mostly other people's code. A typical Node project has hundreds of direct
dependencies and thousands of transitive ones, all executing with your application's privileges.

The risks are known vulnerabilities in packages you use, and malicious code deliberately introduced
into a package you trust.

### Details

#### 1. Lockfiles and reproducible installs

**Theory.** A lockfile records the exact resolved version and integrity hash of every package. `npm
ci` installs exactly what the lockfile says and fails if `package.json` disagrees; `npm install` may
resolve new versions.

**Example.** Always commit the lockfile and always use `npm ci` in CI and in Docker builds. Without
it, two builds of the same commit can contain different code — which breaks reproducibility, makes
"it worked yesterday" unanswerable, and means a compromised package version can enter without any
change on your side.

**Advanced.** The integrity hash in the lockfile protects against a registry serving different bytes
for the same version. It does not protect against a **new** version being malicious, which is what
version ranges expose you to. That is why lockfiles plus deliberate, reviewed upgrades beat automatic
range resolution.

#### 2. Known vulnerabilities

**Theory.** `npm audit`, Dependabot, Snyk and similar compare your dependency tree against
vulnerability databases and report matches.

**Example.** The practical difficulty is signal to noise: most reports are in transitive development
dependencies on code paths you never execute, and a flood of un-triaged alerts is ignored, which is
worse than none. Triage by **reachability** — is the vulnerable function actually called on a path
reachable from untrusted input — and prioritise runtime dependencies over development ones.

**Advanced.** When a critical vulnerability has no patch, the options are: `npm overrides` to force a
patched transitive version (verify compatibility); a temporary fork; mitigating at another layer
(a WAF rule, input validation, disabling the affected feature); or accepting the risk with a
documented decision and a review date. Saying "we would have to decide, and here is how" is better
than implying every alert has a clean fix.

#### 3. Malicious packages

**Theory.** Attackers publish packages designed to be installed by mistake or to compromise a
legitimate one: **typosquatting** (`expres` for `express`), **dependency confusion** (publishing a
public package with the same name as your internal one, which some resolvers prefer), **account
takeover** of a maintainer, and **install scripts** that run arbitrary code at install time.

**Example.** Defences: scope internal packages (`@yourcompany/...`) and configure the registry so
internal names never resolve publicly; use `--ignore-scripts` where feasible and audit the packages
that genuinely need install scripts; pin GitHub Actions to a **commit SHA** rather than a tag, since a
tag can be moved; and review new dependencies before adding them — how many downloads, how recently
maintained, how many maintainers, how many transitive dependencies it drags in.

**Advanced.** The most valuable review question for a new dependency is "what would this do if it were
malicious?" — a package that runs in the build pipeline has access to your CI secrets; a package in
the request path has access to your data. Weigh the risk by where it runs, and prefer a small,
well-scoped package (or fifty lines of your own code) over a large one with a deep tree.

#### 4. Images, SBOMs and provenance

**Theory.** The same applies to container images: base images contain an operating system with its own
vulnerabilities. An **SBOM** (software bill of materials) lists everything in an artefact, so that
when a vulnerability is announced you can answer "are we affected?" quickly.

**Example.** Practices: use minimal base images (distroless, alpine) to reduce the attack surface;
scan images in CI (Trivy, Grype) and fail on critical findings; rebuild regularly so base image
patches are picked up — an image built six months ago has six months of unpatched CVEs; run as a
non-root user with a read-only filesystem (`S15`); and generate an SBOM per build.

**Advanced.** **Provenance and signing** (Sigstore, cosign, SLSA) let you verify that an artefact was
built by your pipeline from your source, and admission control can refuse unsigned images in the
cluster. That is the answer to "how do you know the image running in production is the one you built",
which is exactly the question the recent generation of supply-chain attacks raised.

### Interview questions

- "A transitive dependency has a critical CVE with no patch. Your options?"
- "What is dependency confusion and how do you prevent it?"
- "Why pin GitHub Actions to a SHA rather than a tag?"
- "How do you know the image in production is the one you built?"

---

## S15 · Infrastructure and workload identity

`Advanced` · Requires: `S10`, `S14`, `M26` · Unlocks: `O11`, `SD13`

### Preface

Application security assumes the infrastructure beneath it is sound. This node is the part a backend
engineer owns: what your service can reach, what credentials it holds, and how much damage a
compromise of one container can do.

The organising question is: **if this pod were compromised, what could the attacker reach?** A good
design makes the answer short.

### Details

#### 1. Least-privilege workload identity

**Theory.** Each workload gets its own identity with only the permissions it needs, and credentials
should be short-lived and issued automatically rather than long-lived and copied around.

**Example.** On AWS, IRSA (IAM roles for service accounts) gives a pod a role assumed via its
Kubernetes service account, producing temporary credentials with no stored secret. On GCP, workload
identity does the equivalent. The policy should name specific resources —
`s3:GetObject` on `arn:aws:s3:::uploads/tenant-*` — not `s3:*` on `*`.

**Advanced.** Long-lived access keys in environment variables are the pattern to eliminate: they do
not rotate, they leak, and they are valid from anywhere. If you find one, the remediation is to move
to workload identity rather than to rotate it more often. This is also what makes SSRF (`S08`) so
dangerous in cloud environments — the metadata endpoint hands out the instance's credentials to
anything that can make a local HTTP request, which is why IMDSv2 and egress controls matter.

#### 2. Network segmentation

**Theory.** Restrict what can talk to what, so a compromise in one place does not reach everything.

**Example.** The layers: databases in private subnets with no route from the internet; security
groups allowing only the ports needed, from the specific sources that need them; Kubernetes
NetworkPolicies restricting pod-to-pod traffic (default-deny, with explicit allows); and **egress**
filtering so a compromised container cannot reach arbitrary internet hosts — which limits both
exfiltration and SSRF.

**Advanced.** Egress filtering is the control most often missing and one of the most valuable: most
attacks need to call out — to fetch a second stage, to exfiltrate data, to reach a command-and-control
server. An allow-list of outbound destinations turns a successful exploit into a contained one. It
requires knowing your legitimate destinations, which is a useful exercise in itself.

#### 3. Container hardening

**Theory.** A container is not a security boundary by default; it is a packaging boundary with some
isolation. Harden it so an exploited process has as little as possible.

**Example.** The checklist: run as a **non-root** user (`USER` in the Dockerfile, `runAsNonRoot` in
the pod spec); a **read-only root filesystem** with writable volumes only where needed; drop all
Linux capabilities and add back only what is required; `allowPrivilegeEscalation: false`; no
privileged containers; no mounting the Docker socket; and resource limits so one container cannot
starve the node (`O04`).

**Advanced.** In Kubernetes, also restrict what the pod's **service account** can do against the API
server — by default a compromised pod with a mounted token may be able to list secrets in its
namespace. Set `automountServiceAccountToken: false` unless the pod genuinely needs API access, and
apply RBAC with specific verbs on specific resources. This is the control that most directly limits
lateral movement inside a cluster.

#### 4. Service identity and zero trust

**Theory.** Rather than trusting network location, each service authenticates every caller. Mutual TLS
with per-workload certificates gives cryptographic identity (`A11`).

**Example.** SPIFFE defines the identity format (`spiffe://cluster/ns/default/sa/orders`) and SPIRE
issues short-lived certificates automatically. A service mesh (`M26`) provides this without
application changes, and authorisation policy becomes "the payments service may call the ledger
service's Write method", enforced at the proxy.

**Advanced.** The payoff is that a compromised container cannot impersonate another service, and an
attacker who reaches the network cannot call internal APIs at all. The cost is real operational
complexity, which is why it is usually adopted by a platform team rather than a product team. Be able
to describe the value and to say when it is not yet worth it — for five services in one namespace, a
NetworkPolicy and per-service credentials get most of the benefit.

### Interview questions

- "Your pod is compromised. What does the attacker reach, and how do you shrink that?"
- "Why is egress filtering valuable?"
- "What does running as root inside a container actually risk?"
- "How does a service prove its identity to another service?"

---

## S16 · Detection, audit logging and incident response

`Intermediate` · Requires: `S01`, `S09`, `S12` · Unlocks: `O16`, `O17`

### Preface

Prevention fails eventually. What separates a contained incident from a catastrophe is whether you
**notice**, and how fast you can answer what happened, what was accessed, and whether it is over.

The average time to detect a breach is measured in months. That number is a detection problem, not a
prevention problem.

### Details

#### 1. Audit logs

**Theory.** A security audit log records **who did what to which resource when**, separately from
application logs, with stronger retention and access controls.

**Example.** What belongs in it: authentication events (success and failure); authorisation denials;
privilege and role changes; access to sensitive records; data exports; administrative actions;
impersonation; configuration and secret changes. Each entry needs actor, action, target, timestamp,
source address, and the request or trace id linking it to the technical logs (`M25`).

**Advanced.** Audit logs must be **tamper-evident**: append-only, written to a separate system with
its own credentials, so an attacker with application access cannot erase their traces. The strongest
form is hash-chaining each entry to the previous one, which makes any modification detectable — the
same idea as a Merkle chain (`DB33`) and the reason ledgers are append-only (`DB38`).

#### 2. Detecting the things that matter

**Theory.** Alert on patterns that indicate compromise, not on individual events — individual events
are too noisy and too common.

**Example.** Signals worth alerting on: a spike in authorisation denials from one account (probing);
a user accessing far more records than usual (`S09`); an access pattern at an unusual time or from an
unusual location; a change to a permission or a role outside the normal process; a new outbound
destination from a service (`S15`); a sudden rise in the failed-to-successful login ratio (`S13`); and
any message arriving in a dead-letter queue that indicates a security control failed.

**Advanced.** **Volume-based** detection is what catches insider misuse and stolen credentials: a
support tool reading one customer is normal, and reading fifty thousand is an incident, and both
produce identical individual log entries. Alerting on the rate of access to sensitive data — per
user, compared with their own baseline — is one of the highest-value detections available and is
rarely implemented.

#### 3. Incident response

**Theory.** A defined process, rehearsed before it is needed: **contain**, **eradicate**, **recover**,
**learn** — with communication running alongside all four.

**Example.** What the runbook must specify: who declares an incident and who leads; how to revoke
credentials and sessions quickly (which requires the capability to exist — see `S03`); how to
preserve evidence before rebuilding (snapshot the disk, export the logs — rebuilding first destroys
the evidence); who decides about customer and regulator notification; and the **72-hour** GDPR
breach-notification clock, which starts at awareness, not at resolution.

**Advanced.** The capability most often missing when it is needed: **mass session revocation**. If
you cannot invalidate every session and token for a user or for everyone, your response to a suspected
token compromise is to wait for expiry. Build and test that capability in advance — it is a small
feature that transforms your options during an incident (`S03`).

#### 4. Learning from it

**Theory.** A blameless post-incident review that identifies contributing causes and produces
scheduled, owned actions. Security incidents deserve the same treatment as availability incidents
(`O16`).

**Example.** The questions to answer: how did they get in; what allowed them to move further; why did
detection take as long as it did; what limited the damage, and what would have limited it more. The
detection question is usually the most productive, because detection improvements help with every
future incident regardless of the entry point.

**Advanced.** Rehearse with **tabletop exercises**: walk through a scenario ("an engineer's laptop is
compromised and their credentials are used at 3am on a Sunday") and identify where the process breaks
down. It reliably surfaces gaps — nobody knows who can revoke production access out of hours, the
runbook references a system that was decommissioned, the on-call rota has no security escalation. All
cheap to find in a meeting and expensive to find during an incident.

### Interview questions

- "How would you know if an attacker had been reading your customers' data for a month?"
- "What goes in an audit log, and how do you stop it being tampered with?"
- "Walk me through the first hour of a suspected credential compromise."
- "What capability do most teams lack when they need it?"

---

## S17 · Security in the development lifecycle

`Intermediate` · Requires: `S06`, `S14` · Unlocks: `O08`

### Preface

Security that depends on people remembering fails. Security that is the default path mostly holds.

The goal is to move checks as early and as automatically as possible — into the template, the linter,
the pipeline and the review — so that the easy way to build something is also the safe way.

### Details

#### 1. What to look for in a code review

**Theory.** A short, memorable list applied to every change, rather than an exhaustive audit
occasionally.

**Example.** The checklist to be able to recite:
- Does this endpoint have **authorisation**, and is it **object-level** (`S09`)?
- Is input **validated** against an allow-list, and is binding restricted to permitted fields
  (`F06`)?
- Is the response mapped **explicitly**, or is an entity returned directly?
- Is there any **raw SQL** or shell invocation built from input (`S07`)?
- Does it fetch a **URL** the user supplied (`S08`)?
- Does it **log** anything sensitive (`F08`)?
- Does it add a **dependency**, and what would that dependency do if malicious (`S14`)?
- Does it add **egress** to somewhere new (`S15`)?
- Are new **secrets** handled properly (`S10`)?

**Advanced.** The most valuable habit is asking about the **negative cases**: what happens if this id
belongs to someone else; if this is called twice concurrently (`DB07`); if this array has a million
entries; if this string is a megabyte. Those questions find the bugs that tests do not, and they cost
thirty seconds in a review.

#### 2. Automated checks in the pipeline

**Theory.** Put the mechanical checks in CI so humans can focus on design and logic.

**Example.** The set worth having: **secret scanning** with push protection (`S10`); **dependency
scanning** with a policy for what fails the build (`S14`); **SAST** (CodeQL, Semgrep) tuned to a
small set of high-confidence rules; **container image scanning**; and **infrastructure-as-code
scanning** (public buckets, open security groups, missing encryption). Keep the failure threshold
high enough that the build failing means something.

**Advanced.** The failure mode is alert fatigue: a scanner producing 400 findings is ignored entirely,
including the three that matter. It is better to run a narrow, high-confidence rule set that fails the
build than a broad one that produces a dashboard nobody reads. Tune ruthlessly, and track the
**false-positive rate** of your own tooling as a metric.

#### 3. Secure defaults in the template

**Theory.** The highest-leverage intervention is the service scaffold every new service starts from,
because it applies to everything built afterwards with no ongoing effort.

**Example.** What belongs in it: a global validation pipe with whitelisting (`F06`); authentication
required by default with an explicit opt-out (`F11`); security headers configured (`F26`); structured
logging with redaction (`F08`); error handling that never leaks internals (`F07`); a health endpoint
that does not check dependencies (`O06`); timeouts on every outbound client (`M09`); and a
non-root, read-only container (`S15`).

**Advanced.** This is the difference between security as a review gate and security as an engineering
property. A template that is secure by default means every new service starts correct, and the review
only has to catch deviations. It also makes the "we did not have time" argument disappear, because the
secure path required no extra time.

#### 4. Testing, pentests and disclosure

**Theory.** Automated scanning finds known patterns; humans find logic flaws. Both are needed, on
different cadences.

**Example.** A reasonable programme: automated checks on every commit; a **penetration test**
annually and before major launches, scoped to your actual threat model rather than a generic web
scan; a **vulnerability disclosure policy** with a published contact so researchers can report
findings instead of publishing them; and eventually a bug bounty if the attack surface justifies it.

**Advanced.** Write **regression tests for security fixes**, exactly as you would for functional bugs:
a test that asserts cross-tenant access fails, a test that asserts the extra field is stripped. A
security bug that reappears after a refactor is a common and avoidable embarrassment, and the test is
usually five lines. Suggesting this unprompted shows you treat security as ordinary engineering rather
than as a special ritual.

### Interview questions

- "What do you look for when reviewing a teammate's PR for security?"
- "Your scanner produces 400 findings. What do you do?"
- "What goes in a secure-by-default service template?"
- "How do you stop a fixed security bug from coming back?"
