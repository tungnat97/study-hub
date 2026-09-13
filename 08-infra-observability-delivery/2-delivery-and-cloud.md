[← back to the field index](README.md)

# Infrastructure & Delivery · Part 2 — CI/CD, Infrastructure as Code and Cloud

Nodes `O08`–`O12`.

---

## O08 · CI/CD pipelines

`Intermediate` · Requires: `O03`, `F12`, `S14` · Unlocks: `O09`, `S17`

### Preface

A pipeline turns a commit into a deployable artefact and then into a running deployment. Its job is
to give fast, trustworthy feedback and to make releasing boring.

Two principles carry most of the value: **build once, promote the same artefact** through every
environment; and keep the pipeline fast enough that people run it on every change rather than
batching.

### Details

#### 1. The stages

**Theory.** A typical pipeline: lint and type-check → unit tests → build → integration tests against
real dependencies → security scans → publish the image → deploy to staging → smoke tests → deploy to
production.

**Example.** Order them by **speed and likelihood of failure**: the checks most likely to fail and
fastest to run go first, so feedback arrives in seconds rather than after a twenty-minute suite.
Lint and type-check in under a minute; unit tests in two or three; integration tests with
Testcontainers in five to ten (`F12`).

**Advanced.** Parallelise independent stages, and split slow test suites across runners by timing
rather than alphabetically. But be careful about parallelising things that share state — two
integration jobs against the same database will interfere. Each parallel job should get its own
container-based dependencies, which is another argument for Testcontainers over a shared test
database.

#### 2. Build once, promote

**Theory.** The artefact tested in staging must be **byte-identical** to the one deployed to
production. Rebuilding per environment means production runs something never tested.

**Example.** So the image is built once, tagged with the commit SHA, and the **same digest** is
deployed to each environment; only configuration differs (`F05`). A rebuild could pick up a different
base-image patch, a different transitive dependency, or a different build-tool version — and the
difference is invisible until it breaks.

**Advanced.** Reference images by **digest** (`@sha256:...`) rather than by tag in production
manifests, because a tag can be moved to point at different content. Combined with signing and
admission control (`S14`), that gives you a verifiable chain from source to running container. It also
makes rollback exact: you redeploy a specific digest, not "whatever `v1.2.3` points to now".

#### 3. Speed

**Theory.** Pipeline duration is a developer-experience metric with a direct effect on quality: a
40-minute pipeline means people batch changes, context-switch away, and review less carefully.

**Example.** The usual wins: cache dependencies between runs (keyed on the lockfile hash); use Docker
layer caching (`O03`); run tests in parallel; skip stages that cannot be affected by the change (path
filters in a monorepo); use `npm ci` rather than `npm install`; and keep the integration suite small
and focused rather than letting it accumulate everything.

**Advanced.** Flaky tests are the hidden tax: a suite that fails 5% of the time for no reason trains
people to re-run rather than investigate, which means a genuine failure is also re-run. **Quarantine**
flaky tests automatically (detect, mark, run separately, and fix or delete them on a deadline) rather
than tolerating them. A green build must mean something, or the pipeline is theatre.

#### 4. Branching and environments

**Theory.** **Trunk-based development** — short-lived branches merged to main daily, with incomplete
work behind feature flags (`M27`) — produces small changes, easy reviews and simple rollbacks.
Long-lived branches produce large, risky merges.

**Example.** Ephemeral **preview environments** — one per pull request, torn down on merge — are a
significant quality improvement where the architecture allows: reviewers can use the change, and
integration problems appear before merge. The cost is the infrastructure to create and destroy them
cheaply.

**Advanced.** Pipeline **security** is often overlooked and is a serious attack surface (`S14`): CI
has credentials to deploy to production, so a malicious dependency or a compromised third-party action
can exfiltrate them or push a malicious artefact. Controls: pin actions to a commit SHA; restrict
secrets by environment so a pull-request build cannot access production credentials; require approval
for deployments; and never run untrusted pull-request code with access to secrets.

### Interview questions

- "Your pipeline takes 40 minutes. How do you get it to 10?"
- "Why must you deploy the identical artefact you tested?"
- "How do you handle flaky tests?"
- "What could a malicious dependency do in your CI pipeline?"

---

## O09 · Deployment and release

`Advanced` · Requires: `O05`, `O06`, `O08`, `M27`, `DB22` · Unlocks: `O16`, `SD14`

### Preface

Deploying is putting new code on servers. Releasing is exposing new behaviour to users. Separating
them — with feature flags — is what makes frequent deployment safe.

The part that cannot be rolled back by redeploying is the **database migration**, which is why it is
handled separately and why it must be compatible with both versions (`DB22`).

### Details

#### 1. The rollout

**Theory.** A rolling update replaces pods gradually. The parameters and probes determine whether it
is seamless (`O06`). The full zero-downtime recipe is in `O06`; the release-level concerns are
progressive exposure and the ability to stop.

**Example.** What a good rollout has: a canary stage with automated analysis (`M27`); a defined
"abort" condition and an automatic rollback; and a deployment marker emitted to your monitoring so
every graph shows when it happened — which turns "did the deploy cause this?" from a debate into a
glance (`O17`).

**Advanced.** **Rollback must be tested**, not assumed. The common discoveries during a real rollback:
the previous image has been garbage-collected from the registry; the previous version cannot read data
written by the new one; a migration is not reversible; or a feature flag default changed. Practise a
rollback in staging, and treat "can we roll back?" as a release-readiness question.

#### 2. Migrations in the pipeline

**Theory.** Schema changes run as a separate, ordered step — before the code that needs them — and
must be compatible with the version currently running (`DB22`, `M24`).

**Example.** Why **not** at application startup: several pods race (you need an advisory lock,
`DB09`); a long migration delays readiness until Kubernetes kills the pod; a failure produces a crash
loop rather than a clear error; and rollback of the deployment does not undo the schema change. Run it
as a pipeline stage or a Kubernetes Job that must succeed before the rollout begins.

**Advanced.** **Backfills** are separate again: a throttled, resumable job, not part of the deployment
(`DB22`). And the expand/contract sequence means a single logical change ("rename this column")
becomes three or more releases over days — which must be planned as such, not compressed because it
feels slow. Compressing it is how the outage happens.

#### 3. Feature flags and progressive delivery

**Theory.** A flag lets you deploy code disabled, enable it for internal users, then a percentage,
then everyone — and disable it instantly without a deploy (`M27`).

**Example.** The operational value is the **kill switch**: at 3am, turning off a feature is a config
change taking seconds, while a rollback is a deployment taking minutes and may be blocked by a
migration. Every risky feature and every new external dependency should have one.

**Advanced.** Flags are a production dependency: if the flag service is unreachable, the SDK must fall
back to cached values and then to safe defaults, never fail (`O19`). And flags accumulate — each one
doubles the paths through the code and the combinations become untestable. Require an owner and an
expiry date per flag, and sweep regularly.

#### 4. Measuring delivery

**Theory.** The **DORA** metrics describe delivery performance: **deployment frequency**, **lead time
for changes**, **change failure rate**, and **time to restore service**.

**Example.** The counter-intuitive finding from the research is that frequency and stability move
**together**, not against each other: teams deploying many times a day have *lower* change failure
rates, because each change is small, well understood, and easy to roll back. "We deploy monthly to be
safe" produces large, risky releases.

**Advanced.** The useful way to use these is as a diagnostic rather than a target: a high change
failure rate points at testing and review; a long time to restore points at rollback capability and
observability; a long lead time points at the pipeline and the review process. Quoting them as goals
invites gaming; using them to find the bottleneck is the point (`SD15`).

### Interview questions

- "Walk me through your last deploy from merge to production, including what you would check."
- "How do you roll back a deploy that already ran a destructive migration?"
- "Where do database migrations run in your pipeline, and why not at startup?"
- "Why do teams that deploy more often have fewer failures?"

---

## O10 · Infrastructure as code

`Intermediate` · Requires: `O05` · Unlocks: `O11`, `O19`

### Preface

Infrastructure defined in code is reviewable, versioned, reproducible and auditable. Infrastructure
created by clicking in a console is none of those, and nobody remembers why it is the way it is.

The core concepts are **state** (what the tool believes exists), **plan** (what it intends to change),
and **drift** (reality diverging from the declaration).

### Details

#### 1. Declarative infrastructure

**Theory.** You declare the desired state; the tool computes the difference from current state and
applies it. Terraform, Pulumi, CloudFormation and CDK all work this way.

**Example.** The workflow: change the code → open a pull request → CI runs `terraform plan` and posts
the diff → a human reviews what will actually change → merge → apply. The review step is the value:
a plan showing "will destroy: aws_db_instance.main" is caught before it happens rather than after.

**Advanced.** The **state file** is critical and sensitive: it records the mapping from your code to
real resources, and it contains values that may be secret. It must be in remote storage (S3 with
versioning, or Terraform Cloud) with **locking** so two simultaneous applies cannot corrupt it, and it
must be treated as a secret (`S10`). Losing the state file means Terraform no longer knows it owns
anything, and re-importing a large estate by hand is a very bad week.

#### 2. Drift

**Theory.** Drift is when reality no longer matches the declaration — someone changed a security group
in the console, or a process modified a resource outside the tool.

**Example.** The next `apply` reverts the manual change, which is usually what you want and is
occasionally a surprise outage ("who reverted my firewall fix?"). The fix is process, not tooling:
restrict console write access to break-glass only, run `plan` on a schedule and alert on unexpected
drift, and make the code path the fast path so nobody is tempted.

**Advanced.** Some drift is legitimate and should be excluded: autoscaling changes the desired count,
a managed service updates a minor version. Use `ignore_changes` for those specific attributes rather
than fighting them. Distinguishing "drift that indicates a problem" from "drift that is normal" is
what makes drift detection useful rather than noisy.

#### 3. Modules, environments and blast radius

**Theory.** Structure matters: reusable modules for repeated patterns, separate state per environment
so a staging change cannot touch production, and small enough units that one apply cannot destroy
everything.

**Example.** A common structure: a `modules/` directory with a service module, and per-environment
directories each with their own state, instantiating the modules with different parameters. Keeping
production state separate means an apply in staging physically cannot affect production — which is a
stronger guarantee than a careful engineer.

**Advanced.** **Blast radius** is the design criterion. One enormous state file means every apply
touches everything and a mistake is catastrophic; too many tiny ones means cross-references become
painful. Split by lifecycle and ownership: networking (changes rarely, high risk) separate from
application resources (changes often, low risk). And use `prevent_destroy` on databases and anything
else whose accidental deletion is unrecoverable.

#### 4. GitOps

**Theory.** For Kubernetes, GitOps extends the idea: a controller in the cluster (ArgoCD, Flux)
continuously reconciles the cluster against a Git repository. Git is the desired state, and nobody
runs `kubectl apply` against production.

**Example.** The benefits: the repository is an exact record of what is deployed; drift is corrected
automatically; rollback is a revert; and access to the cluster is not required to deploy, which
removes a large class of credential distribution. Deployment becomes a pull request.

**Advanced.** The trade: an extra component, and an indirection that makes "why is this version
running?" a repository question rather than a `kubectl` one. Also decide how secrets reach the cluster
— they must not be in the repository, so you need sealed secrets, an external secrets operator, or a
secret manager integration (`S10`). That decision is the main design work in adopting GitOps.

### Interview questions

- "Someone changed a security group in the console. What happens on the next apply?"
- "Where does Terraform state live and why does it matter?"
- "How do you limit the blast radius of an apply?"
- "What is GitOps and what does it change?"

---

## O11 · Cloud primitives

`Intermediate` · Requires: `A13`, `O10`, `S15` · Unlocks: `O12`, `O18`, `O20`, `SD13`

### Preface

You do not need to be a cloud architect, and you do need to understand the building blocks your
service runs on: how the network is arranged, what is reachable from where, what the managed services
guarantee, and where the cost comes from.

The organising question is the same as in security: what can reach what?

### Details

#### 1. Regions, zones and the network

**Theory.** A **region** is a geographic location; an **availability zone** is an isolated data centre
within it, with independent power and networking. A **VPC** is your private network, divided into
subnets, each in one zone.

**Example.** The standard three-tier layout: **public** subnets containing only the load balancer and
NAT gateways; **private** subnets for application instances; and **isolated** subnets for databases
with no route to the internet at all. Security groups then allow only the specific flows needed —
the load balancer to the application on one port, the application to the database on another.

**Advanced.** Multi-AZ is table stakes for availability: resources spread across at least two zones so
a zone failure is survivable (`O20`). It is also where a large share of cost quietly comes from:
**cross-zone data transfer is billed** in most clouds, so a chatty service whose replicas are spread
across three zones pays for every internal call that crosses one (`O18`).

#### 2. Load balancers and DNS

**Theory.** A layer-7 load balancer (ALB, Cloud Load Balancing) terminates TLS, routes by host and
path, and health-checks targets. A layer-4 one (NLB) forwards TCP with lower latency and no HTTP
awareness (`M07`).

**Example.** The usual arrangement: DNS (Route 53) → load balancer → target group of instances or pods
→ your service. Certificates are managed by the cloud provider and renewed automatically, which
removes the most common TLS outage (`S10`).

**Advanced.** Health check configuration is where availability is won or lost: too aggressive and a
brief slowdown removes healthy targets, leaving the rest overloaded and cascading; too lenient and
traffic keeps going to a broken instance. Also note the interaction with deregistration delay — the
load balancer must stop sending traffic and let in-flight requests finish before the target goes away,
which is the cloud equivalent of the `preStop` sleep (`O06`).

#### 3. Managed services

**Theory.** The cloud provides managed databases, caches, queues and object storage. You trade
control and cost for operations.

**Example.** What to know about the common ones: **RDS/Aurora** — multi-AZ failover, automated
backups with PITR, read replicas, and a maintenance window you do not fully control (`DB36`). **S3** —
extremely durable, cheap, with lifecycle policies to move old data to colder tiers, presigned URLs
(`F17`), and versioning to protect against accidental deletion. **SQS/SNS** — effectively unlimited
scale, at-least-once, minimal operations (`Q20`). **Secrets Manager / KMS** for secrets and keys
(`S10`).

**Advanced.** Every managed service has **quotas and limits**, and hitting one during an incident is a
special kind of misery: API rate limits, connection limits, partition or shard limits, maximum message
size, maximum concurrency. Know the ones on your critical path, monitor your headroom against them,
and request increases **before** you need them — approval is not instant.

#### 4. Identity and access

**Theory.** Cloud IAM controls what each identity may do to which resources. Workloads should have
their own identities with scoped permissions, not shared long-lived keys (`S15`).

**Example.** The correct pattern: a role per workload, assumed via the platform's workload identity
(IRSA on EKS, workload identity on GKE), granting specific actions on specific resources. No access
keys in environment variables, nothing with `*` on `*`.

**Advanced.** IAM policy evaluation is worth understanding at a basic level, because it explains
confusing denials: an explicit **Deny** always wins; a permission must be granted by both the identity
policy and any resource policy (an S3 bucket policy, a KMS key policy); and service control policies
at the organisation level can deny things your account-level policy allows. "Access denied" with a
policy that looks correct is usually one of those three.

### Interview questions

- "Design the network layout for a three-tier app and say what is reachable from the internet."
- "Why is your data-transfer bill so high?"
- "What limits and quotas are on your critical path?"
- "Your IAM policy allows it and you still get access denied. Why?"

---

## O12 · Serverless and alternative compute

`Intermediate` · Requires: `O11` · Unlocks: `O18`

### Preface

Serverless means you provide a function or a container and the platform handles scaling, including
scaling to zero. You pay per request rather than per instance-hour.

It is excellent for spiky, event-driven and low-volume workloads, and it has genuine constraints that
make it a poor fit for others — particularly around database connections and cold starts.

### Details

#### 1. The options

**Theory.** **Functions** (Lambda, Cloud Functions) — a function per event, maximum granularity, most
constraints. **Container-based serverless** (Cloud Run, Fargate, App Runner) — your container, scaled
by the platform, fewer constraints. **Traditional** — you manage instances or pods.

**Example.** Cloud Run and similar are the pragmatic middle ground: you build a normal container
(`O03`), it scales to zero when idle and up under load, and you keep your framework, your local
development workflow and your portability. For many teams it is a better answer than functions.

**Advanced.** The programming-model difference matters: with functions, each invocation handles one
request, so per-request concurrency is 1 and connection pooling within a process is useless. Container
serverless handles **many concurrent requests per instance**, so pooling works normally — which alone
removes the biggest practical problem with serverless and relational databases.

#### 2. Cold starts

**Theory.** When there is no warm instance, the platform must start one: fetch the image, start the
runtime, initialise the application, and only then handle the request. That latency is visible to the
user.

**Example.** Typical magnitudes: a small Node function, 100-500ms; a large one with heavy
initialisation, seconds; a JVM without optimisation, several seconds. Mitigations: keep the deployment
small; move initialisation out of the request path; use provisioned concurrency (which costs money and
removes the scale-to-zero benefit); and for the JVM, SnapStart, AOT or GraalVM native image.

**Advanced.** Cold starts are worst exactly when they hurt most: during a traffic **spike**, many new
instances start at once and a large share of requests pay the penalty. So a workload with sharp spikes
and tight latency requirements is the worst fit, which is the opposite of the intuition that
"serverless handles spikes". It handles them in throughput, not in latency.

#### 3. Connections to a relational database

**Theory.** A relational database has a hard connection limit, and each concurrent function instance
wants its own connection (`DB21`).

**Example.** A thousand concurrent invocations means up to a thousand connection attempts against a
database whose `max_connections` is 100 — and connection **establishment** is itself expensive
(handshake, TLS, authentication, process creation), so the storm can take the database down before any
query runs. Mitigations: a proxy or pooler (RDS Proxy, PgBouncer, Supavisor); an HTTP data API;
capping the function's reserved concurrency; or a database designed for it (Neon, PlanetScale).

**Advanced.** This is the single most common reason a serverless plus relational architecture fails in
production, and it usually appears only under load. It is also the strongest argument for
container-based serverless, where a pool is shared across many concurrent requests in one instance.
Being able to explain the mechanism — not just "use RDS Proxy" — is the senior answer.

#### 4. When not to use it

**Theory.** The constraints: execution time limits (15 minutes for Lambda), payload size limits, no
persistent local state, limited control over the runtime, harder local development and debugging, and
vendor lock-in at the event-source and IAM level.

**Example.** Poor fits: steady high traffic (a reserved instance is much cheaper per request); long
running work (it will be cut off); latency-critical paths with spiky traffic (cold starts); WebSocket
servers (long-lived connections, `F16`); and anything needing a large in-process cache, since each
instance has its own.

**Advanced.** The cost crossover is real and worth quantifying: serverless is dramatically cheaper at
low and spiky volumes and **more** expensive at steady high volume. A service handling a constant
thousand requests per second is almost always cheaper on reserved capacity. Framing the decision as
"where is the crossover for this workload?" rather than as an architectural preference is exactly the
judgement being tested (`O18`).

### Interview questions

- "When is serverless the wrong choice for a backend service?"
- "A thousand concurrent Lambdas against one Postgres. What happens?"
- "Why are cold starts worst during a traffic spike?"
- "At what point does serverless stop being cheaper?"
