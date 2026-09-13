[← back to the field index](README.md)

# Infrastructure & Delivery · Part 3 — Observability and Operations

Nodes `O13`–`O20`.

---

## O13 · Observability fundamentals

`Intermediate` · Requires: `F08` · Unlocks: `O14`, `O15`, `M25`, `M34`, `DB39`

### Preface

Monitoring tells you whether the things you predicted would break have broken. Observability is being
able to ask questions you did not anticipate, from the telemetry you already have.

The three signals — **logs**, **metrics**, **traces** — answer different questions and are weak
individually. The value comes from correlating them.

### Details

#### 1. The three signals and what each is for

**Theory.**
- **Metrics** — numeric aggregates over time. Cheap, long retention, low cardinality. Answer "is it
  broken, how much, and is it getting worse?"
- **Logs** — discrete events with detail. Expensive at volume, high cardinality. Answer "what exactly
  happened to this request?"
- **Traces** — the path of one request across services with timings. Answer "where did the time go
  and which component failed?"

**Example.** The debugging path that requires all three: a metric alert fires on error rate → a trace
shows the failure is in the payment service's call to a provider → that service's logs, filtered by
the trace id, show the actual error message. Any one signal alone leaves you guessing.

**Advanced.** A fourth signal is increasingly standard: **continuous profiling** (`C13`), which
answers "which code is consuming the CPU" and lets you diff across deploys. And events — deployment
markers, configuration changes, feature-flag toggles — overlaid on your graphs answer "what changed?"
faster than any amount of analysis (`O17`).

#### 2. Correlation is the point

**Theory.** Each signal must carry identifiers that let you move between them: the trace id in every
log line, exemplars linking metrics to sample traces, and consistent resource attributes (service,
version, environment, pod).

**Example.** The concrete requirements: a request id and trace id in every log line, propagated across
services and through queues (`M25`, `O15`); the service version as a label, so you can compare
before and after a deploy; and the tenant id where multi-tenancy matters (`M29`) — as a log field and
a trace attribute, **not** as a metric label (`O14`).

**Advanced.** OpenTelemetry exists to make this consistent: one SDK producing all three signals with
shared context and shared resource attributes, exported to whatever backend you use. Adopting it means
instrumentation is vendor-neutral, which matters because observability vendors are expensive and
switching otherwise means re-instrumenting everything.

#### 3. What to instrument on day one

**Theory.** A default set that every service should emit before it sees production traffic.

**Example.** The list:
- **RED per endpoint** — request rate, error rate, duration histogram (`O14`).
- **Dependency metrics** — latency and error rate per downstream, per database, per cache.
- **Saturation** — connection pool utilisation and wait time, queue depth and oldest-message age,
  event loop delay (`C03`), thread pool utilisation, memory as a fraction of the limit.
- **Business counters** — orders placed, payments failed, signups. These catch problems no technical
  metric shows.
- **Structured error logs** with the trace id (`F08`).
- **Traces** with tail sampling (`O15`).

**Advanced.** Business metrics are the most valuable and most often missing. "Orders per minute" falling
to zero is unambiguous, whereas every technical metric can look normal while a bug silently rejects
every order. The classic example is a deploy that breaks a validation rule: 200 responses, normal
latency, zero orders. Alert on the business metric and you catch it in minutes (`O16`).

#### 4. Cost

**Theory.** Observability data is frequently among the largest infrastructure line items, occasionally
exceeding the cost of running the service.

**Example.** The drivers: log volume (especially debug logging left on, or logging every request body);
metric **cardinality** (`O14`); trace volume without sampling (`O15`); and long retention. Controls:
sample logs on high-volume successful paths, keep all errors, tier retention (7 days hot, 90 days in
cheap storage), drop metric labels you never query, and use tail sampling.

**Advanced.** The framing to use: **retention is a decision per signal, not one policy**. Metrics are
cheap and worth keeping for a year for capacity planning. Detailed logs are expensive and worth days.
Traces are worth days for errors and hours for the rest. Applying one retention policy to everything
either wastes money or loses the data you need (`O18`).

### Interview questions

- "You can only keep one of logs, metrics or traces. Which and why?"
- "What do you instrument in a new service on day one?"
- "How do you get from a metric alert to the actual error message?"
- "Your observability bill is larger than your compute bill. What do you cut?"

---

## O14 · Metrics and dashboards

`Advanced` · Requires: `O13`, `C14`, `DB39` · Unlocks: `O16`, `SD15`

### Preface

Metrics are the cheap, always-on signal. The two things to get right are **using histograms for
latency** (because percentiles cannot be averaged) and **keeping cardinality under control** (because
it is what destroys metric systems).

### Details

#### 1. The metric types

**Theory.**
- **Counter** — only increases (requests, errors). You query its **rate**, not its value.
- **Gauge** — goes up and down (memory in use, queue depth, active connections).
- **Histogram** — counts observations into buckets, so percentiles can be computed and, crucially,
  **aggregated across instances**.
- **Summary** — percentiles computed client-side; cheaper to query and **cannot be aggregated**.

**Example.** For latency, always a histogram. `histogram_quantile(0.99, sum(rate(http_duration_bucket[5m])) by (le))`
computes a true p99 across every instance, because the buckets are summed first and the percentile is
derived from the combined distribution.

**Advanced.** This is the practical reason **you cannot average percentiles** (`C14`): averaging each
instance's p99 gives a number that is not the p99 of anything. Same for time: the average of 60
one-minute p99s is not the hourly p99. Histograms are the only correct way, and bucket boundaries must
be chosen to bracket your SLO threshold — a histogram whose buckets are 0.1, 1, 10 seconds cannot tell
you the proportion under 300ms.

#### 2. Cardinality

**Theory.** Each unique combination of label values is a separate time series. Series count is what
determines memory, storage and query cost, and it multiplies across labels.

**Example.** The fatal mistake: putting a user id, an order id, a full URL with path parameters, or an
email address in a label. A million users means a million series per metric — and combined with three
other labels, hundreds of millions. Prometheus falls over. Use a **route template**
(`/orders/:id`, not `/orders/12345`) and keep labels to service, endpoint, method, status class.

**Advanced.** For high-cardinality questions ("is this tenant slow?"), use **traces and logs**, which
are designed for it, or **exemplars** attached to histogram buckets which link a metric to a sample
trace. And be careful with status code as a label — use the class (2xx, 4xx, 5xx) or you get a series
per distinct code, which is acceptable, while a label carrying an error **message** is not.

#### 3. RED and USE

**Theory.** **RED** for request-driven services: **R**ate, **E**rrors, **D**uration.
**USE** for resources: **U**tilisation, **S**aturation, **E**rrors. Together they cover both "is the
service healthy" and "which resource is the cause".

**Example.** For an API: RED per endpoint. For the database connection pool: utilisation (in use over
maximum), saturation (requests waiting, and wait time), errors (acquisition timeouts). For a worker
fleet: RED on job processing, plus saturation as queue age (`Q21`).

**Advanced.** **Saturation is the leading indicator** and the least instrumented (`C14`). Utilisation
tells you how busy something is now; saturation tells you how much work is queued, which predicts the
latency cliff before it arrives. Pool wait time, queue age, event loop delay and consumer lag are the
four that matter most for a typical backend service.

#### 4. Dashboards that work

**Theory.** A dashboard should answer, in order: is it broken, where, and is it getting worse. Not
"here is everything we can measure".

**Example.** A useful structure: a top row of SLO-level indicators (availability, latency against the
threshold, error budget remaining, `M34`); a second row of RED per key endpoint; a third of
dependencies (database, cache, downstream services); a fourth of saturation (pool, queue, memory,
CPU); with deployment markers overlaid on everything.

**Advanced.** Keep a separate **incident dashboard** with only the ten things you look at first — a
dashboard with ninety panels is unusable at 3am. And make graphs comparable: the same time range, the
same units, and a fixed y-axis where a spike matters. Small things, and they determine whether the
dashboard is used under pressure or abandoned for ad-hoc queries.

### Interview questions

- "Why can't you average p99 across pods?"
- "Your Prometheus fell over after a new label was added. Explain."
- "What is saturation and why does it matter more than utilisation?"
- "What is on your incident dashboard?"

---

## O15 · Distributed tracing in practice

`Advanced` · Requires: `O13`, `M25` · Unlocks: `O17`

### Preface

A trace shows one request's journey across every service, with timings, as a waterfall. It answers
"where did the time go?" and "which service actually failed?" in seconds.

The concepts are in `M25`; this node is about making it work in production — propagation through the
awkward boundaries, sampling that keeps what matters, and cost.

### Details

#### 1. OpenTelemetry

**Theory.** OpenTelemetry is the vendor-neutral standard: SDKs for every language, auto-instrumentation
for common libraries, a **collector** that receives, processes and exports, and a wire protocol.

**Example.** A realistic setup: auto-instrumentation for HTTP, the database driver, the broker client
and the cache client — which gets you most of the value with almost no code; plus manual spans around
business operations worth seeing separately. The collector runs as a sidecar or DaemonSet, handles
batching, sampling and export, and means changing backend requires no application change.

**Advanced.** Put processing in the **collector**, not the application: batching (to avoid a network
call per span), attribute scrubbing (removing anything sensitive before export, `S12`), tail sampling,
and export. The application should do as little as possible, so the observability pipeline can be
changed without redeploying every service.

#### 2. Propagation through the awkward places

**Theory.** The trace context travels in the W3C `traceparent` header. HTTP is handled automatically;
the gaps are everywhere else.

**Example.** The three places propagation breaks:
- **Message queues** — put `traceparent` in the message headers and restore it in the consumer.
- **Background jobs** — carry it in the job payload and restore it in the worker (`F13`).
- **Async boundaries within a process** — a detached promise, a `setTimeout`, a thread pool handoff
  (`C06`), a reactive pipeline. The context is stored in `AsyncLocalStorage` or a `ThreadLocal` and
  can be lost.

**Advanced.** For asynchronous work, prefer a **span link** to a parent-child relation: the consumer's
span is caused by the producer's but is not contained within it, since the parent has already ended.
Otherwise a message sitting in a queue for four hours produces a trace that appears to last four
hours, which distorts every latency view. Links keep the causal connection without the duration.

#### 3. Sampling

**Theory.** **Head sampling** decides at the start (keep 1%) — simple, and it discards most errors and
slow requests. **Tail sampling** buffers the complete trace and decides afterwards — keep all errors,
all slow traces, and a small percentage of the rest.

**Example.** A production policy in the collector: 100% of traces containing an error; 100% above a
latency threshold; 100% when a debug header is present; 1% of everything else. You keep what is useful
at a fraction of the cost.

**Advanced.** Tail sampling requires every span of a trace to reach the **same** collector instance,
which needs load balancing by trace id — an operational detail that surprises teams adopting it. And
the sampling decision must be **consistent** across services (it propagates in the `traceparent`
flags), or you get partial traces that are worse than none.

#### 4. Getting value from it

**Theory.** Tracing pays off when it is part of the debugging path, not a system people remember
exists.

**Example.** What makes it useful: exemplars so you can click from a latency spike straight to a slow
trace (`O14`); the trace id in every log line so you can pivot from trace to logs (`F08`); the trace id
surfaced in error responses so a support ticket carries it (`A07`); and span attributes carrying the
high-cardinality context — tenant, user, endpoint, cache hit or miss, retry count — that metrics
cannot hold.

**Advanced.** Trace data also answers questions metrics cannot: the **service dependency graph**
derived from real traffic (often revealing calls nobody knew about); which downstream contributes most
to p99; and how many services a single user request actually touches — which is usually more than the
architecture diagram suggests, and is the argument for reducing fan-out (`C17`).

### Interview questions

- "A request is slow 1% of the time. How do you find where?"
- "How do you propagate a trace through Kafka and through a background job?"
- "Head sampling or tail sampling? What does each cost?"
- "Why would a trace appear to last four hours?"

---

## O16 · Alerting and on-call

`Advanced` · Requires: `O06`, `O09`, `O14`, `M34`, `S16` · Unlocks: `O17`

### Preface

An alert should mean: a human must act now. Everything else is a dashboard or a ticket.

The two principles: alert on **symptoms** (users are affected) rather than **causes** (CPU is high),
and make every page **actionable** — if the responder cannot do anything, it should not have woken
them.

### Details

#### 1. Symptoms, not causes

**Theory.** Cause-based alerts (high CPU, low disk, a restarted pod) fire constantly without user
impact and miss impact that has no obvious cause. Symptom-based alerts (error rate, latency, a
business metric falling) fire when something that matters is wrong.

**Example.** "CPU above 80%" is not an alert: the service may be fine, and autoscaling may handle it.
"The checkout error rate exceeded 1% for five minutes" is — users cannot buy. Keep cause metrics on
dashboards to help diagnose, and page on the symptom.

**Advanced.** The exception is a **predictable** cause with a lead time: a disk filling at a rate that
will exhaust it in four hours, a certificate expiring in seven days (`S10`), a quota approaching its
limit (`O11`). These are genuinely actionable and genuinely urgent-ish — and should be tickets with a
deadline rather than pages, unless the lead time is short.

#### 2. SLO burn-rate alerts

**Theory.** Rather than a fixed threshold, alert on how fast you are consuming the error budget
(`M34`). This automatically scales with your traffic and expresses urgency in terms of user impact.

**Example.** The standard multi-window, multi-burn-rate configuration:
- **Page**: burning at 14x the sustainable rate over the last hour **and** the last five minutes —
  at that rate a month's budget is gone in about two days.
- **Ticket**: burning at 3x over the last six hours and the last thirty minutes — a slow leak.

Requiring both a long and a short window prevents a momentary spike from paging and prevents a slow
burn from going unnoticed.

**Advanced.** The long window controls **sensitivity** and the short window controls **reset time** —
without the short window, an alert stays firing long after the problem is fixed, which erodes trust.
This pairing is the single most useful alerting pattern to know, and being able to explain both halves
is what distinguishes understanding it from having copied it.

#### 3. Alert hygiene

**Theory.** Every page must be actionable, and a page that is routinely ignored or acknowledged
without action is worse than none, because it trains people to ignore the next one.

**Example.** The review questions for each alert: has it fired in the last quarter; did the responder
do anything; is there a runbook; could it have been automated away; is the threshold still right.
Delete or downgrade anything that fails. Track pages per week as a metric of on-call health and treat
a rising number as a problem to fix rather than a fact of life.

**Advanced.** Alert fatigue is a **reliability** risk, not a comfort issue: a team receiving forty
pages a week will miss the important one. The fix is usually a combination of raising thresholds,
switching from cause to symptom, grouping related alerts into one notification, and automating the
common remediations — a pod that restarts itself does not need a human.

#### 4. On-call and runbooks

**Theory.** On-call needs a rota with sustainable load, a clear escalation path, a handover, and a
**runbook per alert** telling the responder what it means, how to confirm impact, what to check, and
the common remediations.

**Example.** A good runbook answers: what does this alert mean in user terms; is the user impact
confirmed and how; the three most common causes and how to distinguish them; the safe mitigations
(roll back, flag off, scale, shed); who to escalate to; and what to record. Link it directly from the
alert so it is one click away at 3am.

**Advanced.** The most valuable practice after an incident is a **blameless post-incident review**
producing **scheduled, owned** actions — an action item with no owner and no date does not exist.
Track the completion rate of post-incident actions as a metric; a team that reviews diligently and
never implements the actions has a documentation exercise rather than a learning process (`S16`).

### Interview questions

- "Write the alerts for a checkout service. Which ones page at 3am?"
- "Your team gets 40 pages a week. How do you fix that?"
- "Why alert on burn rate instead of error rate?"
- "Why do burn-rate alerts use two windows?"

---

## O17 · Debugging production

`Advanced` · Requires: `O01`, `O02`, `O15`, `O16`, `C13`, `DB39` · Unlocks: `SD15`

### Preface

This is the most commonly asked "senior" operations question, in some form of: *"p99 latency tripled
ten minutes ago. Talk me through the next fifteen minutes."*

Rehearse it as a script. The structure matters more than the specific tools: establish impact, check
what changed, narrow, hypothesise, **mitigate before root-causing**, then investigate.

### Details

#### 1. Establish impact first

**Theory.** Before investigating, know who is affected and how badly. It determines urgency, whether
to declare an incident, and what mitigation is acceptable.

**Example.** The questions: which endpoints; what proportion of requests; all users or one tenant or
one region; is it errors or latency; is it getting worse; what is the business impact (orders failing,
or a report page loading slowly). "All checkout requests failing" and "the admin search is slow" get
very different responses.

**Advanced.** Communicate early and keep communicating: a status update every fifteen minutes even
when there is nothing new, because silence makes everyone ask, which interrupts the person fixing it.
Separate the roles when the incident is large — one person investigating, one communicating, one
coordinating. Saying this shows you have been in a real incident rather than only imagining one.

#### 2. What changed?

**Theory.** The overwhelming majority of incidents are caused by a change. Check changes before
theorising.

**Example.** The list, in order of likelihood: a **deploy** (yours or a dependency's); a **feature
flag** or configuration change (`O19`); a **migration** or backfill (`DB22`); a **traffic** change (a
marketing campaign, a new large customer, a bot); a **dependency** incident (check their status page);
an **infrastructure** change; a certificate or credential **expiry** (`S10`); and time-based events
(month end, a scheduled job).

**Advanced.** This is why deployment markers on dashboards are so valuable (`O14`): "latency tripled
at 14:32, the deploy was at 14:31" answers the question instantly and converts a long investigation
into a rollback. Make every change type visible on the timeline — deploys, flags, config, migrations —
and most incidents become ten-minute events.

#### 3. Narrow, then hypothesise

**Theory.** Bisect the problem space along available dimensions before forming a theory.

**Example.** Narrow by: endpoint (one or all); tenant (one customer's data shape); region or zone;
pod (one instance or the fleet — if one, it is that instance; if all, it is shared); dependency (which
downstream's latency moved first); and time (exactly when did it start, to the minute). Then the four
saturation suspects: **CPU, memory, connection pool, queue** (`C14`).

**Advanced.** Use the traces (`O15`) to find *where* the time is, then the metrics to see *how much*,
then the logs for *why*. And be aware of the common misattribution: your service's latency rose
because a **downstream** got slower, and the downstream's own dashboard looks fine because their
latency is measured after their queue. Per-dependency client-side latency metrics are what settle
this (`M08` grey failure).

#### 4. Mitigate before root-causing

**Theory.** Stopping the user impact is the priority; understanding why can happen afterwards with
the evidence preserved.

**Example.** The mitigation toolkit, fastest first: **roll back** the deploy; **turn off** the feature
flag; **scale up** if it is a capacity problem; **shed load** or tighten rate limits to protect the
core path (`M10`); **fail over** to another region or replica; **disable** the expensive feature
(brownout); **restart** the affected instance as a last resort — and take a heap dump or a profile
**first**, because restarting destroys the evidence (`C12`).

**Advanced.** The instinct to fully understand before acting is the most common mistake in an
incident, and it extends outages substantially. Roll back first, investigate from the logs and traces
afterwards — they are still there. The exception is when a rollback itself is risky (an irreversible
migration, `DB22`), which is precisely why expand/contract matters. Saying "I would roll back first and
diagnose after" is the answer interviewers are listening for.

### Interview questions

- "p99 tripled 10 minutes ago. Talk me through the next 15 minutes."
- "Tell me about the hardest production bug you have debugged." (have two ready: one distributed, one
  database)
- "Your service is slow but its own dashboards look fine. What now?"
- "When would you not roll back immediately?"

---

## O18 · Cost and efficiency

`Intermediate` · Requires: `O11`, `O12`, `C15` · Unlocks: `SD15`

### Preface

Cost is an engineering concern, and treating it as one is a distinctly senior behaviour. Most
infrastructure bills contain a large proportion of waste that nobody owns.

The useful metric is **cost per unit of business value** — cost per request, per order, per tenant —
because it makes the bill comparable over time and tells you whether growth is efficient.

### Details

#### 1. Where the money actually goes

**Theory.** The usual distribution surprises people: compute is rarely the whole story. Over-provisioned
instances, data transfer, storage that is never deleted, and observability data are the recurring
large items.

**Example.** The specific ones worth checking:
- **Over-provisioned requests** — pods requesting far more than they use, so the cluster needs more
  nodes (`O04`).
- **Cross-zone data transfer** — billed per gigabyte; a chatty service spread across three zones pays
  for every internal call that crosses one (`O11`).
- **NAT gateway** — per gigabyte processed; often a surprisingly large line item.
- **Observability** — log and metric ingestion and retention, sometimes exceeding compute (`O13`).
- **Storage** — old snapshots, unattached volumes, objects nobody deletes, and no lifecycle policy.
- **Idle non-production environments** running overnight and at weekends.

**Advanced.** Egress to the internet is the most expensive transfer category, which is why a CDN
sitting in front of your origin is a **cost** optimisation as well as a performance one (`Q09`) — the
CDN's egress pricing is far lower and most requests never reach you.

#### 2. Right-sizing

**Theory.** Set requests and limits from observed usage plus headroom, not from guesses, and revisit
them as the workload changes.

**Example.** A reasonable method: CPU request at roughly the p95 of observed usage; memory request at
the p99 plus a margin, with the limit equal to the request for critical services so they are
Guaranteed QoS (`O04`); then verify that autoscaling handles the peaks. Vertical Pod Autoscaler in
recommendation mode produces these numbers from real data.

**Advanced.** The trade to be explicit about: over-provisioning costs money, under-provisioning costs
availability. Right-sizing is not "make everything smaller" — it is matching allocation to measured
need with deliberate headroom, and **static stability** (`M31`) means keeping enough spare capacity to
survive a zone failure without scaling. That headroom is a cost you are choosing to pay for
reliability, and saying so frames it correctly.

#### 3. Architectural efficiency

**Theory.** The largest savings usually come from doing less work, not from cheaper resources.

**Example.** Efficiency improvements that are also performance improvements: caching to avoid repeated
computation (`Q02`); fixing N+1 queries (`DB20`); batching to reduce round trips (`C17`); paginating
so you transfer less; compressing responses (`A20`); and moving analytical queries off the operational
database onto cheaper columnar storage (`DB32`). Each reduces both latency and bill.

**Advanced.** The cheapest request is the one you do not serve: a well-cached, CDN-fronted endpoint
costs nothing. And the cheapest storage is data you deleted — retention policies enforced by scheduled
jobs, with partitioned tables so deletion is a partition drop rather than a mass `DELETE` (`DB24`).
Both are engineering decisions with direct financial consequences.

#### 4. Making cost visible

**Theory.** Cost that nobody sees is nobody's responsibility. Tag resources by team and service,
report cost per service, and put it somewhere teams look.

**Example.** What works: mandatory tags enforced at provisioning (`O10`); a monthly cost report per
service; an alert on unusual increases (a runaway job, a misconfigured autoscaler, an accidental
infinite retry loop); and cost per request as a tracked metric so growth can be judged.

**Advanced.** Beware optimising the wrong thing: engineering time is expensive, and a week spent
saving £200 a month is a poor trade unless it compounds. Prioritise by absolute size and by how much
it will grow — a 10% saving on the largest line is usually worth more than eliminating a small one.
That judgement, rather than enthusiasm for cutting, is the senior position.

### Interview questions

- "Cut this service's infrastructure cost 40% without hurting latency. Where do you look first?"
- "Why is your data transfer bill so high?"
- "How do you right-size a service?"
- "When is a cost optimisation not worth doing?"

---

## O19 · Runtime configuration and feature flags

`Intermediate` · Requires: `F05`, `O10`, `S10` · Unlocks: `O09`

### Preface

Configuration that can change without a deploy is powerful during an incident and dangerous the rest
of the time: a bad config push takes effect everywhere at once, with none of the safety of a rolling
deployment.

The rule: treat configuration changes as production changes — reviewed, validated, rolled out
progressively, and reversible.

### Details

#### 1. Static versus dynamic

**Theory.** **Static** configuration is read at startup and requires a restart: connection strings,
pool sizes, ports. **Dynamic** configuration can change at runtime: feature flags, rate limits, log
levels, timeouts.

**Example.** Be deliberate about which is which. Dynamic configuration is valuable precisely where
you need to act quickly — turning off an expensive feature, tightening a rate limit, raising the log
level for one tenant — and each dynamic value is a thing that can change unexpectedly, so keep the set
small and intentional.

**Advanced.** Anything dynamic needs a **safe fallback**: if the configuration service is unreachable,
the SDK serves the last known good value, and if there is none, a compile-time default. A service that
fails to start or fails requests because a flag service is down has made its availability depend on a
non-critical system — which is exactly backwards (`M10`).

#### 2. Config changes as outages

**Theory.** A configuration push reaches every instance at once, with no canary and often no review.
Some of the largest cloud outages on record were caused by configuration, not code.

**Example.** The controls that help: configuration in version control and reviewed (`O10`);
**validation** before acceptance (schema, ranges, and a dry run against the current state);
**progressive rollout** — one instance, one zone, one region, or a percentage (`M31`); an automatic
rollback on error-rate regression; and a complete audit trail of who changed what when.

**Advanced.** The insidious failures are values that are individually valid and jointly wrong: a
timeout shorter than the downstream's typical latency; a pool size that multiplied by the replica
count exceeds the database's limit (`DB21`); a rate limit below normal traffic. Validation should
check **relationships**, not just individual values — and the ones that cannot be validated need the
progressive rollout to catch them.

#### 3. Feature flags at runtime

**Theory.** Flags are dynamic configuration with targeting: enable for internal users, for a
percentage, for a specific tenant, or for a cohort.

**Example.** Operationally the most valuable use is the **kill switch** — one per non-essential
feature and per external dependency, so during an incident you can disable the expensive report, the
recommendations call, or the third-party integration in seconds rather than deploying (`O17`).

**Advanced.** Evaluation should be **local**: the SDK holds the rule set in memory and evaluates
without a network call, with the rules streamed or polled in the background. A remote evaluation call
per request adds latency and a hard dependency to your hot path. And the SDK's own failure behaviour
must be verified — test what happens when the flag service is unreachable, because you will find out
eventually either way.

#### 4. Secrets at runtime

**Theory.** Secrets are configuration with stricter handling: they must not be logged, must be
rotatable, and ideally should not require a restart to rotate (`S10`).

**Example.** Rotation without downtime requires the application to accept **both** the old and new
values during an overlap — two valid database passwords, two signing keys in the JWKS (`A18`). If the
application reads a secret once at startup and holds it forever, rotation means a rolling restart,
which is acceptable and must be planned.

**Advanced.** The stronger pattern is short-lived dynamic credentials issued per instance at startup
(`S10`), which makes rotation automatic and removes the long-lived secret entirely. Where that is not
available, re-reading a mounted secret file periodically gives you rotation without a restart —
mounted secrets are updated in place by the platform, which is a small and useful thing to know.

### Interview questions

- "Config changes caused two of your last three outages. What do you change?"
- "What happens to your service if the feature flag provider is unreachable?"
- "Which configuration should be dynamic and which static?"
- "How do you rotate a database password without downtime?"

---

## O20 · Disaster recovery and platform resilience

`Advanced` · Requires: `O07`, `O11`, `DB36`, `M33` · Unlocks: `SD13`, `SD07`

### Preface

Disaster recovery is the answer to "a whole zone, region or provider is gone" — and to the far more
common "someone deleted the production database".

The two numbers that drive every decision are **RPO** (how much data you can lose) and **RTO** (how
long you can be down). They are business decisions, and everything technical follows from them.

### Details

#### 1. RPO and RTO

**Theory.** **RPO** — recovery point objective: the maximum acceptable data loss, measured in time.
**RTO** — recovery time objective: the maximum acceptable downtime. They are set per system, by the
business, and they determine the architecture and the cost.

**Example.** RPO of five minutes means continuous WAL archiving or synchronous replication (`DB23`).
RPO of zero means synchronous replication with all its latency cost. RTO of fifteen minutes means a
warm standby ready to promote — you cannot restore 2TB from cold storage in fifteen minutes. RTO of
four hours allows a restore from backup.

**Advanced.** The honest question is "what is your **actual** RTO?", and most teams do not know because
they have never timed a restore (`DB36`). Measure it: restore a production-sized backup into a scratch
environment and record the wall-clock time. That number is usually several times the assumed one, and
producing it is a genuinely valuable piece of work.

#### 2. Levels of redundancy

**Theory.** In increasing order of cost and protection: **multi-AZ** (survive a data centre failure —
table stakes); **multi-region active-passive** (survive a region failure, with a failover procedure);
**multi-region active-active** (survive a region failure with no failover, and every write-consistency
problem in `M33`).

**Example.** Multi-AZ is usually automatic with managed services and is where most systems should stop.
Multi-region is a substantial commitment: data replication with lag, a decision procedure for
failover, DNS or anycast traffic steering, and the fact that your **dependencies** must also be
multi-region or you have simply moved the single point of failure.

**Advanced.** Active-active is a **consistency** decision before it is an infrastructure one (`SD13`):
two regions accepting writes to the same data means either conflict resolution or partitioning data by
region so each record has one home. Most "active-active" systems in practice are active-active for
reads and partitioned or single-homed for writes — and describing it that precisely is the senior
answer.

#### 3. The failures that actually happen

**Theory.** Regional outages are rare. Deletion, corruption, expiry and dependency failures are
common, and DR planning that only addresses the rare case misses the likely ones.

**Example.** What to plan for: an accidental `DELETE` or a bad migration (PITR, `DB36`); a corrupted
deployment (rollback, `O09`); an expired certificate or credential (`S10`); a dependency outage
(degrade, `M10`); a quota exhausted (`O11`); and a compromised credential (revoke and rotate, `S16`).
Each needs a runbook, and each is far more likely than losing a region.

**Advanced.** **Replicas are not backups** (`DB36`): a `DELETE` replicates in milliseconds. Protections
against mistakes are different from protections against hardware failure — backups in a separate
account with write-once retention so compromised credentials cannot delete them, a **delayed replica**
lagging deliberately by an hour, and `prevent_destroy` on critical resources (`O10`).

#### 4. Testing it

**Theory.** An untested DR plan is a document, not a capability. The things that fail in a real
failover are the ones nobody exercised.

**Example.** What to test, in increasing ambition: restore a backup and verify the data monthly; fail
over a database in staging; run a **game day** killing a zone in a controlled way and observe what
breaks; and practise the full runbook with the on-call team so they have done it before the night it
matters.

**Advanced.** What testing reliably reveals: DNS TTLs longer than anyone expected (`A13`); a hard-coded
endpoint in a configuration file; a dependency that is single-region; a runbook step referring to a
decommissioned system; nobody knowing who is authorised to declare a failover; and capacity in the
surviving region that was never provisioned for the full load — which is the **static stability**
point (`M31`) and the most common real failure of a multi-region plan.

### Interview questions

- "Your primary region goes down. What is your RTO honestly, and what would you need to halve it?"
- "Why are replicas not backups?"
- "What actually breaks during a real failover?"
- "What is the difference between active-passive and active-active, really?"
