[← back to the field index](README.md)

# Security · Part 2 — The Vulnerability Catalogue

Nodes `S06`–`S09`.

---

## S06 · OWASP Top 10 and the attack catalogue

`Intermediate` · Requires: `S01` · Unlocks: `S07`, `S08`, `S09`, `F26`

### Preface

The OWASP Top 10 is the industry's shared list of what actually goes wrong. You should be able to
name the categories and give one concrete example of each — not to recite a list, but because the
categories are a checklist you can run a design against.

The single most important fact in it: **broken access control is number one**, ahead of injection.
The most common serious vulnerability is not exotic — it is forgetting to check whether this user may
see this record.

### Details

#### 1. The categories

**Theory.** The current list, with a one-line example each:
1. **Broken access control** — `GET /invoices/42` returns another customer's invoice (`S09`).
2. **Cryptographic failures** — personal data transmitted or stored unencrypted; weak hashing (`S11`).
3. **Injection** — SQL, NoSQL, command, or template injection (`S07`).
4. **Insecure design** — a flaw in the design itself: no rate limit on a reset flow, a business rule
   enforceable only on the client.
5. **Security misconfiguration** — default credentials, `DEBUG=true`, an open S3 bucket, verbose
   errors.
6. **Vulnerable and outdated components** — a dependency with a known CVE (`S14`).
7. **Identification and authentication failures** — weak passwords, session fixation, no MFA (`S02`).
8. **Software and data integrity failures** — unsigned updates, unsafe deserialisation, a compromised
   build pipeline (`S14`).
9. **Logging and monitoring failures** — a breach nobody notices for months (`S16`).
10. **Server-side request forgery** — your server fetches a URL an attacker chose (`S08`).

**Example.** The list is most useful as a **review checklist**: for any new feature, walk it and ask
whether each category applies. It takes five minutes and catches the ordinary problems, which is what
most vulnerabilities are.

**Advanced.** The API-specific list (OWASP API Security Top 10) is more relevant for backend work, and
its top entries are **BOLA** (broken object-level authorisation — the API version of number one),
**broken authentication**, **broken object property-level authorisation** (mass assignment and
excessive data exposure, `S09`), and **unrestricted resource consumption** (no rate limits or size
limits, `A08`). Knowing that this list exists, and that it differs, is a good signal.

#### 2. Insecure design versus implementation bugs

**Theory.** Some vulnerabilities cannot be fixed by better code, because the design itself is unsafe.
No amount of careful implementation fixes a flow that trusts a client-supplied price.

**Example.** Design flaws: trusting a price or a discount sent by the client; a password reset with no
rate limit; an operation whose authorisation depends on a value the client controls; a "confirm by
clicking this link" flow with a guessable token. These require a design change, which is why threat
modelling at design time matters (`S01`).

**Advanced.** The recurring shape is **trusting the client**. Anything computed, validated or enforced
only in a browser or a mobile app is a suggestion, not a control — the client is fully under the
user's control, and an attacker uses curl. Re-derive prices, totals, permissions and state transitions
on the server, every time.

#### 3. Misconfiguration

**Theory.** The vulnerability that requires no skill to exploit and is extremely common: something
left at a default, or exposed by accident.

**Example.** The audit list: `DEBUG` disabled in production; stack traces not returned to clients
(`F07`); default credentials changed; admin interfaces not publicly reachable; object storage buckets
private by default; database not accepting connections from the internet; directory listing off;
unnecessary HTTP methods disabled; security headers present (`F26`); and cloud metadata endpoints
restricted (`S08`).

**Advanced.** The durable fix is to make the **template** secure rather than relying on a checklist:
a service scaffold with security headers, validation, structured logging and protected-by-default
authentication (`F11`), plus infrastructure-as-code with secure defaults (`O10`) and automated
scanning for public buckets and open ports. Checklists are applied inconsistently; defaults apply
themselves (`S17`).

#### 4. Thinking like an attacker

**Theory.** The useful habit is asking, for each input and each step: what if this were hostile?

**Example.** Applied to a single endpoint: what if the id belongs to someone else (`S09`)? What if the
number is negative, or enormous? What if the string is a megabyte? What if the array has a million
entries? What if this is called a thousand times a second (`A08`)? What if it is called twice
concurrently (`DB07`)? What if the JSON contains fields I did not expect (`S09`)? What if a field
contains SQL, HTML or a URL (`S07`, `S08`)?

**Advanced.** Add the business-logic questions, which scanners never find: can I apply the coupon
twice? Can I refund more than I paid? Can I change the price between adding to cart and checking out?
Can I refer myself? These are the ones that cost real money, they come from understanding the domain
rather than from a tool, and raising them is a distinctly senior contribution.

### Interview questions

- "What is the most common serious vulnerability in real APIs?"
- "Give me one concrete example of each OWASP category that applies to our API."
- "What is the difference between insecure design and an implementation bug?"
- "What business-logic attacks would you look for in a checkout flow?"

---

## S07 · Injection

`Advanced` · Requires: `S06` · Unlocks: `DB37`, `S14`

### Preface

Injection happens when data is interpreted as code. The classic case is SQL, and the same shape
appears in shell commands, NoSQL queries, templates, LDAP and — recently — prompts to language
models.

The fix is always the same idea: **keep data and code separate**, so the data is never parsed. In SQL
that means parameterised queries. Escaping and blocklists are not the fix.

### Details

#### 1. SQL injection and parameterisation

**Theory.** A parameterised query sends the SQL text and the values **separately**. The database plans
the statement before it ever sees the values, so a value cannot change the statement's structure. No
amount of quoting inside a value can escape it.

**Example.**

```ts
// vulnerable
await db.query(`SELECT * FROM users WHERE email = '${email}'`);

// safe — the driver sends text and values separately
await db.query('SELECT * FROM users WHERE email = $1', [email]);
```

An `email` of `' OR '1'='1` changes the query in the first and is simply a value that matches nothing
in the second.

**Advanced.** Escaping fails because the rules are context-dependent and subtle: character-set
mismatches historically allowed multi-byte sequences to slip a quote past an escaper, and a
value used inside a `LIKE` pattern has additional metacharacters. Parameterisation removes the class
of bug rather than handling its instances — which is the general principle behind every good security
fix.

#### 2. What cannot be parameterised

**Theory.** Parameters work for **values**, not for identifiers or SQL keywords. You cannot
parameterise a table name, a column name, `ASC`/`DESC`, or the structure of the query.

**Example.** So a user-controlled sort column is the one place you genuinely must build SQL from
input — and the only safe method is an **allow-list**:

```ts
const SORTABLE = { created: 'created_at', total: 'total_minor' } as const;
const col = SORTABLE[req.query.sort as keyof typeof SORTABLE];
if (!col) throw new BadRequestException('invalid sort');
const dir = req.query.dir === 'desc' ? 'DESC' : 'ASC';
// safe: col and dir come from your own code, never from input
```

**Advanced.** The same applies to dynamic `IN` lists (generate the right number of placeholders, do
not join the values), to dynamic `WHERE` clauses (build from a fixed set of known fragments with
parameters for the values), and to schema names in multi-tenant systems. If input ever becomes part
of SQL text, it must have been validated against a closed set defined in your code.

#### 3. ORM escape hatches

**Theory.** ORMs parameterise by default, and every ORM has raw-query methods that do not.

**Example.** The patterns to grep for in a codebase: `query(`...${}...`)` with a template literal in
TypeORM or Prisma's `$queryRawUnsafe`; Django's `.extra()` and `RawSQL`; Hibernate string
concatenation into HQL; and any place a query string is built with `+`. Prisma's `$queryRaw` with a
tagged template is safe (it parameterises); `$queryRawUnsafe` is not, and the naming is deliberate.

**Advanced.** A lint rule that forbids template literals inside raw query calls is cheap and catches
this permanently — much more reliable than review (`S17`). And note that **ORMs do not protect
against NoSQL injection** in the same way: passing a parsed JSON object straight into a MongoDB
filter allows `{"password": {"$gt": ""}}` to match any password. Validate that user-supplied filter
values are of the expected **type**, not just present (`F06`).

#### 4. The other injections

**Theory.** The same pattern in other interpreters.

**Example.**
- **Command injection** — never build a shell string from input. Use the argv form
  (`execFile('convert', [input])`, not `exec('convert ' + input)`), which passes arguments directly
  without a shell.
- **Template injection** — user input used as a *template* rather than as *data* can execute code in
  most template engines. Render user content as a value, never compile it as a template.
- **LDAP, XPath, header injection** (a newline in a value splitting an HTTP header), and
  **log injection** (a newline forging log entries).
- **Prompt injection** — if your service sends untrusted content to a language model, that content
  may contain instructions. Treat retrieved or user-supplied text strictly as data, never as
  instructions, and never give a model access to actions based on content it read.

**Advanced.** Prompt injection is the current unsolved one, and it is worth a sentence if the role
touches AI features: there is no reliable parameterisation for natural language, so the defence is
architectural — limit what the model can *do*, require confirmation for side effects, and never let
content fetched from the internet authorise an action. That framing (contain the blast radius,
because you cannot sanitise the input) is the correct current answer.

### Interview questions

- "Show me a query in your codebase that would be injectable and fix it."
- "How do you support user-specified sort columns safely?"
- "Why is escaping not the answer?"
- "What does NoSQL injection look like?"

---

## S08 · Browser-adjacent attacks: XSS, CSRF, SSRF

`Advanced` · Requires: `S06`, `A19` · Unlocks: `F26`

### Preface

These three are grouped because backend engineers own part of each, and because they are routinely
confused with one another.

**XSS** — your data ends up executing as script in someone's browser. **CSRF** — another site makes
a browser send an authenticated request to yours. **SSRF** — your *server* is tricked into making a
request on an attacker's behalf. SSRF is the one most likely to appear in a backend interview.

### Details

#### 1. Cross-site scripting

**Theory.** Untrusted data is rendered into a page and executed as script. **Stored** XSS is persisted
(a comment, a profile field); **reflected** XSS comes back in a response; **DOM-based** XSS never
touches the server.

**Example.** The backend's responsibilities: do not build HTML by concatenation; set a
**Content-Security-Policy** that forbids inline script and restricts sources; set `HttpOnly` on
session cookies so an XSS cannot steal them (`S03`); serve user-uploaded files from a **separate
domain** with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff` (`F17`); and
never return user content with a `text/html` content type unless you intend it to render.

**Advanced.** The separate-domain rule is the one most often missed: if `app.example.com` serves a
user-uploaded HTML file, its script runs in your origin with access to cookies and storage. A
dedicated `usercontent` domain contains it. This is a backend decision — where files are served from —
not a frontend one.

#### 2. Cross-site request forgery

**Theory.** A malicious page causes the browser to send a request to your site, and the browser
**automatically attaches cookies**. If you authenticate with cookies, the request is authenticated.
The attacker cannot read the response (that is what CORS prevents, `A19`) — but the side effect has
happened.

**Example.** Defences, in layers: `SameSite=Lax` on the session cookie (the browser does not send it
on cross-site POSTs — the default in modern browsers and most of the protection); an anti-CSRF token
that the attacker cannot read or guess; and verifying the `Origin` header on state-changing requests.

**Advanced.** The precise rule to be able to state: **CSRF applies when the browser attaches
credentials automatically** — cookies, HTTP basic auth, client certificates. A token sent in an
`Authorization` header is **not** attached automatically, so a cross-site request cannot carry it, and
CSRF protection is unnecessary. That is the correct and complete answer to the question you will be
asked, and it explains *why* rather than asserting a rule (`F26`).

#### 3. Server-side request forgery

**Theory.** Your server fetches a URL supplied by a user. The attacker supplies a URL pointing at
something only your server can reach — internal services, the cloud metadata endpoint, or localhost.

**Example.** The classic escalation: a link-preview or webhook-registration feature fetches
`http://169.254.169.254/latest/meta-data/iam/security-credentials/`, the AWS metadata endpoint, and
returns temporary IAM credentials for the instance role. That is how the Capital One breach worked,
and it turns a small feature into a cloud account compromise.

**Advanced.** The defences, layered because each alone is bypassable:
- **Allow-list** the destinations where possible (far better than blocking).
- **Resolve the hostname yourself, validate the resulting IP** against private ranges (10/8,
  172.16/12, 192.168/16, 127/8, 169.254/16, and the IPv6 equivalents), **then connect to that IP** —
  otherwise DNS rebinding changes the answer between your check and the connection.
- Disable redirects, or re-validate after each one, since a redirect can point anywhere.
- Route outbound calls through an **egress proxy** with its own allow-list (`S15`), which also limits
  exfiltration.
- Require IMDSv2 (session-token based), which blocks the simple metadata fetch.

#### 4. Deserialisation and XXE

**Theory.** Deserialising untrusted data into objects can execute code, because some formats encode
which classes to instantiate and which methods to call. **XXE** is the XML equivalent, where an
external entity reference makes the parser read local files or make network requests.

**Example.** Java deserialisation of untrusted `ObjectInputStream` data, Python `pickle`, PHP
`unserialize` — all are remote code execution if the input is attacker-controlled. **Never
deserialise untrusted data in these formats**; use JSON with an explicit schema. For XML, disable
external entity resolution explicitly — most parsers now default to safe, and older ones do not.

**Advanced.** The Node equivalent is **prototype pollution**: merging attacker-controlled JSON into an
object can set `__proto__` properties affecting every object in the process, changing behaviour
elsewhere (including, in some cases, enabling code execution). Defences: validate and whitelist keys
before merging (`F06`), use `Object.create(null)` for maps built from input, and avoid deep-merge
utilities on untrusted data.

### Interview questions

- "Your service fetches a user-supplied URL for link previews. Threat model it."
- "Do you need CSRF tokens for a JSON API with Bearer auth? Prove it."
- "Why serve user-uploaded files from a different domain?"
- "What is prototype pollution?"

---

## S09 · Broken access control in practice

`Advanced` · Requires: `S05`, `S06`, `F06` · Unlocks: `S16`

### Preface

This is the number one vulnerability category and the one automated scanners cannot find, because
every request looks valid — it is authenticated, well-formed, and the endpoint exists. Only the
**data** is wrong.

Three shapes recur: reading someone else's object (**IDOR**), setting a field you should not be able
to set (**mass assignment**), and returning more data than intended.

### Details

#### 1. IDOR / broken object-level authorisation

**Theory.** The endpoint checks that you are logged in, then acts on the id you supplied without
checking whether that object is yours.

**Example.**

```ts
// vulnerable: the guard proved authentication, nothing more
@Get(':id')
async find(@Param('id') id: string) {
  return this.orders.findById(id);            // any authenticated user reads any order
}

// safe: ownership is part of the query
@Get(':id')
async find(@Param('id') id: string, @CurrentUser() user: User) {
  const order = await this.orders.findByIdForTenant(id, user.tenantId);
  if (!order) throw new NotFoundException();  // 404, not 403 — do not confirm existence
}
```

**Advanced.** Using UUIDs instead of sequential ids is **obfuscation, not authorisation**: it makes
enumeration harder and does nothing once an id leaks — through a URL shared in a support ticket, a
referrer header, a log, or a different endpoint that returns ids. Use UUIDs to prevent enumeration
*and* scope every query. The two are complementary, and only one of them is a control.

#### 2. Making it structurally impossible

**Theory.** Fixing IDOR endpoint by endpoint fails, because the next endpoint someone writes will
forget. The fix must be structural.

**Example.** The layered approach, which is the answer to give:
1. The tenant and user live in a **request-scoped context** (`F03`), not passed as parameters that can
   be omitted.
2. The **repository automatically applies** the tenant filter to every query — so a query that
   forgets returns nothing, not everything.
3. **Row-level security** in the database as a backstop (`DB37`).
4. **Tests** that attempt cross-tenant access for every resource type and assert failure.
5. Where the model is complex (sharing, hierarchies), a **central authorisation service** rather than
   scattered checks (`S05`).

**Advanced.** The failure mode to watch for even with all this: a **second entry point** that bypasses
the layer — a GraphQL resolver reaching the ORM directly (`A15`), a batch or admin endpoint, an
internal gRPC service, a background job, or a data export. The audit question is "what are all the
ways to read this table?", and the answer is usually longer than expected.

#### 3. Mass assignment

**Theory.** Binding request data directly to an entity lets a client set fields you never intended to
expose — `role`, `isAdmin`, `tenantId`, `balance`, `verified`.

**Example.** The defences: bind into a **DTO** that lists only the permitted fields, and enable
whitelisting so unknown properties are stripped or rejected (`F06`); never pass `req.body` to
`save()`; and separate the DTOs for create and update, because the fields a client may set differ
(an id may be settable on create and never on update).

**Advanced.** The subtler version is **field-level authorisation**: `status` may be settable by an
admin and not by a customer; `price` by the merchant and not the buyer. A single DTO cannot express
that, so you need either role-specific DTOs or an explicit check per sensitive field. This is the
"broken object property-level authorisation" category in the API Top 10, and it is commonly missed.

#### 4. Excessive data exposure

**Theory.** The response contains more than the client needs — internal ids, other users' details,
soft-deleted records, fields added later by someone who did not consider this endpoint.

**Example.** Returning an entity directly is the cause (`F06`). A user object with `passwordHash`,
`resetToken`, `internalNotes` and `stripeCustomerId` is one `res.json(user)` away. Map explicitly to
a response DTO so adding a column is safe by default, and check what nested relations serialise —
including an order that embeds a full customer object.

**Advanced.** Frontends filtering data for display is not a control: the raw response is visible in
developer tools and to any script. The related trap is a **generic filter or search endpoint** that
accepts arbitrary query parameters — it lets a client ask questions you never designed for, such as
filtering by a field that leaks information through the count of results. Constrain filters to an
allow-list (`A05`).

### Interview questions

- "Here is a controller. Find the access-control bug."
- "How do you make IDOR structurally impossible rather than fixing it case by case?"
- "A user POSTs `{ "role": "admin" }` to your update endpoint. What stops them?"
- "Are UUIDs an authorisation control?"
