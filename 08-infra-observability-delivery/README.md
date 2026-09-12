[← back to the index](../README.md)

# Field 8 — Infrastructure, Observability & Delivery

You are not applying for SRE, but a senior backend engineer owns their service in production. The
bar: you can debug a live incident, explain what your deploy actually does, and read a dashboard.
Legend: `B` `I` `A` `X`.

---

## Nodes

#### O01 · Linux fundamentals for backend engineers
`I` · Requires: — · Unlocks: O02, O04, O17
- Key: processes and PIDs, signals (**SIGTERM vs SIGKILL** — the basis of graceful shutdown), exit
  codes, file descriptors and `ulimit -n` (a real production limit for connection-heavy services),
  memory (RSS vs virtual vs page cache), load average vs CPU utilisation, `/proc`, cgroup limits,
  the OOM killer, standard tools (`ps`, `top/htop`, `lsof`, `strace`, `df`, `dmesg`), stdout/stderr
  as the log destination in containers.
- Q: "`Too many open files` in production. What is happening and what do you check?"
- Q: "Load average is 12 on an 8-core box. Is that bad?" (depends — it counts runnable *and*
  uninterruptible-sleep tasks; check whether it's CPU or I/O wait).

#### O02 · Network debugging
`I` · Requires: O01, A12, A13 · Unlocks: O17
- Key: `curl -v`/`-w` timing breakdown, `dig`/`nslookup`, `ss -tanp` (states: ESTABLISHED,
  TIME_WAIT, CLOSE_WAIT — `CLOSE_WAIT` piling up means your app isn't closing sockets), `tcpdump`,
  `traceroute`/`mtr`, `telnet`/`nc` for reachability, MTU issues, proxy and DNS resolution inside a
  container/pod, `/etc/resolv.conf` and `ndots`.
- Q: "Service A can't reach service B. Give me your diagnostic order." (DNS → TCP connect → TLS →
  HTTP → auth → app; and check from inside the pod, not from your laptop).

#### O03 · Containers
`I` · Requires: F28 · Unlocks: O04, O05, O08, S14
- Key: image layers and caching (order your Dockerfile so dependencies cache), **multi-stage builds**,
  small/distroless base images, non-root user, `.dockerignore`, build args vs runtime env, signal
  handling (`exec` form so your process is PID 1 and receives SIGTERM; `tini` for zombie reaping),
  immutable images and tag-by-digest, one process per container, image size vs cold start.
- Q: "Your container ignores SIGTERM on deploy. Two likely causes." (shell form CMD wrapping the
  process; no signal handler in the app).
- Q: "Rebuilding takes 9 minutes because one line changed. Fix the Dockerfile."

#### O04 · Container runtime internals & resources
`A` · Requires: O01, O03 · Unlocks: O06, C12
- Key: namespaces (pid, net, mnt, uts, ipc, user) and **cgroups** (cpu shares/quota, memory limit);
  CPU throttling from a low CPU limit causing latency spikes at low average utilisation (a classic);
  memory limit → **OOMKilled (exit 137)** with no stack trace; runtimes reading cgroup limits
  (`UV_THREADPOOL`/`libuv`, JVM `MaxRAMPercentage`, `availableProcessors`, Node's default heap);
  ephemeral storage limits.
- Q: "Pods are being OOMKilled but the app's heap metric looks fine. Explain." (RSS includes
  non-heap: buffers, native, threads; and the heap max was above the container limit).
- Q: "p99 latency spikes every few seconds at 30% CPU. Cause?" (CFS quota throttling).

#### O05 · Kubernetes basics
`I` · Requires: O03 · Unlocks: O06, O07, O09, M05
- Key: pod (the unit, possibly multi-container/sidecar), ReplicaSet/Deployment, Service (ClusterIP/
  NodePort/LoadBalancer) and kube-proxy, Ingress/Gateway API, ConfigMap and Secret (base64 is not
  encryption), namespaces, labels/selectors, the declarative reconciliation loop, `kubectl` triage
  (`get`, `describe`, `logs -p`, `events`, `exec`, `top`).
- Q: "Pod is CrashLoopBackOff. Your first four commands?"
- Q: "What actually happens between `kubectl apply` and traffic reaching a new pod?"

#### O06 · Kubernetes for service owners
`A` · Requires: O04, O05, C12 · Unlocks: O09, O16, M05, F28
- Key: **requests vs limits** (requests drive scheduling, limits drive throttling/OOM; QoS classes;
  the "no CPU limit" argument), **probes** — liveness (restarts you; a bad liveness probe causes
  cascading restarts under load), readiness (removes from endpoints) and startup (protects slow
  boots) — and why liveness must never check dependencies; rolling update parameters
  (maxSurge/maxUnavailable), `terminationGracePeriodSeconds` + `preStop` + SIGTERM handling (the
  full zero-downtime recipe), HPA on CPU vs custom/queue metrics, PodDisruptionBudget, anti-affinity
  and topology spread, node pressure/eviction, resource quotas.
- Q: "Give me the complete list of things needed for zero-dropped-requests deploys." (readiness
  probe, preStop sleep ≥ endpoint propagation, SIGTERM → stop accepting → drain in-flight → close
  pools, grace period > drain time, maxUnavailable=0, keep-alive handling at the LB, and backward-
  compatible migrations).
- Q: "Your liveness probe hits `/health` which checks the database. Why is that dangerous?"

#### O07 · Stateful workloads & platform services
`I` · Requires: O05 · Unlocks: O20
- Key: StatefulSet (stable identity, ordered rollout), PersistentVolumes and storage classes,
  operators for databases, and the honest senior answer: **use the managed service** (RDS, MSK,
  ElastiCache) unless you have a platform team; what you give up (tuning, extensions, cost) and gain
  (backups, failover, patching, on-call).
- Q: "Would you run Postgres in Kubernetes? Defend your answer."

#### O08 · CI/CD pipelines
`I` · Requires: O03, F12, S14 · Unlocks: O09, S17
- Key: stages (lint → unit → build → integration with services/testcontainers → security scan →
  publish image → deploy per environment), fast feedback and pipeline duration as a DX metric,
  caching dependencies and layers, **build once, promote the same artifact** across environments
  (never rebuild per env), branch strategy (trunk-based + short-lived branches + feature flags vs
  gitflow), required checks, ephemeral preview environments, flaky test quarantine, secrets in CI,
  deployment automation and permissions.
- Q: "Your pipeline takes 40 minutes. How do you get it to 10?"
- Q: "Why must you deploy the identical artifact you tested?"

#### O09 · Deployment & release
`A` · Requires: O05, O06, O08, M27, DB22 · Unlocks: O16, SD14
- Key: rolling/blue-green/canary with automated analysis, feature flags to separate deploy from
  release (and flag debt), migrations run **before** the code that needs them and compatible with
  the old version (expand/contract), backfills as separate throttled jobs, rollback plan that
  accounts for data, deployment freeze vs continuous delivery, change failure rate and MTTR (DORA).
- Q: "Walk me through your last deploy from merge to production, including what you'd check."
- Q: "The new version is fine but the old one crashes on the new schema. What did you do wrong?"

#### O10 · Infrastructure as code
`I` · Requires: O05 · Unlocks: O11, O19
- Key: declarative infra (Terraform/Pulumi/CDK), state files and locking, plan/apply review, drift,
  modules and environments, immutable infrastructure, GitOps (ArgoCD/Flux) for cluster state,
  secrets excluded from state, blast radius of a `terraform apply`.
- Q: "Someone changed a security group in the console. What happens on the next apply, and how do
  you prevent this class of problem?"

#### O11 · Cloud primitives
`I` · Requires: A13, O10, S15 · Unlocks: O12, O18, O20, SD13
- Key: regions vs AZs, VPC/subnets (public vs private), security groups vs NACLs, NAT gateways
  (and their cost), ALB/NLB and target groups, autoscaling groups, managed DB (RDS/Aurora
  multi-AZ, read replicas, parameter groups), object storage semantics (S3 durability, consistency,
  presigned URLs, lifecycle policies), queues (SQS/SNS), IAM roles and policies, quotas/limits.
- Q: "Design the network layout for a 3-tier app and say what is reachable from the internet."
- Q: "Why is your data-transfer bill so high?" (cross-AZ traffic, NAT gateway, egress).

#### O12 · Serverless & alternative compute
`I` · Requires: O11 · Unlocks: O18
- Key: Lambda/Cloud Run/Fargate; cold starts and mitigation (provisioned concurrency, small bundles,
  snapstart); execution time and payload limits; **connection management to a relational DB** (a
  thousand concurrent lambdas = a thousand connections → RDS Proxy/pooler); statelessness; event
  sources; per-request cost model and where it beats or loses to containers; vendor lock-in.
- Q: "When is serverless the wrong choice for a backend service?" (steady high traffic, long-running
  work, heavy DB connection use, latency-sensitive cold paths, complex local dev).

#### O13 · Observability fundamentals
`I` · Requires: F08 · Unlocks: O14, O15, M25, M34, DB39
- Key: monitoring (known unknowns) vs observability (ask new questions of existing telemetry); the
  three signals — **logs** (events, high cardinality, expensive at volume), **metrics** (aggregates,
  cheap, low cardinality), **traces** (causality across services) — plus profiles and events;
  structured logging, log levels and sampling, cost control; correlation between all three
  (trace id in logs, exemplars in metrics); what you instrument by default in every service.
- Q: "You can only keep one of logs, metrics or traces. Which and why?"
- Q: "What do you instrument in a new service on day one?" (RED metrics per endpoint, error logs
  with trace id, dependency latency, saturation signals, business counters).

#### O14 · Metrics & dashboards
`A` · Requires: O13, C14, DB39 · Unlocks: O16, SD15
- Key: counters, gauges, **histograms** (and why histograms are the only way to aggregate
  percentiles across instances — you cannot average p99s); Prometheus model (pull, labels,
  **cardinality explosion** from user ids/urls as labels), PromQL basics (`rate()`, `histogram_
  quantile`, `sum by`), **RED** (rate, errors, duration) for services and **USE** (utilisation,
  saturation, errors) for resources; the four golden signals; dashboards that answer "is it broken,
  where, and is it getting worse".
- Q: "Why can't you average p99 across pods?"
- Q: "Your Prometheus fell over after a new label was added. Explain."

#### O15 · Distributed tracing in practice
`A` · Requires: O13, M25 · Unlocks: O17
- Key: OpenTelemetry as the standard (SDK, auto-instrumentation, collector, vendor-agnostic export);
  spans, attributes, events, context propagation across HTTP/gRPC/**message queues** and async
  boundaries (the common gap); head vs **tail sampling** (keep the slow and error traces); linking
  logs and traces; span naming and cardinality; cost.
- Q: "How do you propagate a trace through a Kafka topic and a BullMQ job?"

#### O16 · Alerting & on-call
`A` · Requires: O06, O09, O14, M34, S16 · Unlocks: O17
- Key: alert on **symptoms** (user-visible SLO burn) not causes (CPU 80%); multi-window
  multi-burn-rate SLO alerts; every page must be actionable and have a runbook; ticket vs page;
  alert fatigue as a real failure mode; escalation, handover, follow-the-sun; error budget policy;
  blameless postmortems with action items that actually get scheduled.
- Q: "Write the alerts for a checkout service. Which ones page at 3am?"
- Q: "Your team gets 40 pages a week. How do you fix that?"

#### O17 · Debugging production
`A` · Requires: O01, O02, O15, O16, C13, DB39 · Unlocks: SD15
- Key: a repeatable method — establish user impact and blast radius → check recent changes (deploy,
  flag, config, migration, traffic, dependency) → narrow by signal (which endpoint, which tenant,
  which pod, which dependency) → form and test a hypothesis → **mitigate before you root-cause**
  (rollback/flag off/scale/shed) → then investigate; the four saturation suspects (CPU, memory,
  connection pool, queue); reading a trace waterfall; correlating a latency graph with a deploy
  marker; communication during an incident.
- Q: "p99 tripled 10 minutes ago. Talk me through the next 15 minutes." (this is the single most
  common senior "ops" interview question — rehearse it as a script)
- Q: "Tell me about the hardest production bug you've debugged." (have two ready: one distributed,
  one database)

#### O18 · Cost & efficiency
`I` · Requires: O11, O12, C15 · Unlocks: SD15
- Key: right-sizing from real utilisation, autoscaling floors/ceilings, spot/preemptible for
  stateless workers, storage tiers and lifecycle policies, egress and cross-AZ traffic, log/metric
  retention as a top-3 bill item, N+1 and chattiness as a cost problem, cost per request as a metric,
  managed-service premium vs headcount.
- Q: "Cut this service's infrastructure cost 40% without hurting latency. Where do you look first?"

#### O19 · Runtime configuration & feature flags
`I` · Requires: F05, O10, S10 · Unlocks: O09
- Key: config precedence, hot reload vs restart, config as a deploy-free change (and its risk —
  a bad config push is a global outage: stage it, canary it, validate it), feature flag systems,
  kill switches for dependencies, flag lifecycle and cleanup, per-tenant overrides, secret rotation
  without restart.
- Q: "Config changes caused two of your last three outages. What do you change?"

#### O20 · Resilience of the platform: DR & capacity
`A` · Requires: O07, O11, DB36, M33 · Unlocks: SD13, SD07
- Key: multi-AZ as table stakes, multi-region (active-passive vs active-active) and its data
  consistency cost, RPO/RTO defined per system by the business, **tested** failover and restore
  drills, dependency inventory (what breaks if S3/Stripe/Auth0 is down), degraded modes, game days,
  static stability, capacity headroom and scaling limits (quotas, connection counts, partition
  counts).
- Q: "Your primary region goes down. What is your RTO honestly, and what would you need to halve it?"

---

## Topological order (study waves)

```
Wave 0  O01  O03(after F28)  O13
Wave 1  O02  O04  O05  O08  O10
Wave 2  O06  O07  O11  O14  O15  O19
Wave 3  O09  O12  O16  O20
Wave 4  O17  O18
```

Cross-field parents: `A12/A13` TCP & DNS, `C12-C15` memory/profiling/latency, `DB22` migrations,
`DB36` backups, `DB39` DB observability, `F05/F08/F12/F28` framework, `M25/M27/M33/M34`, `S14-S16`.

**Most-asked four:** zero-downtime deploy end to end (O06/O09), "p99 tripled, debug it live" (O17),
requests/limits and OOMKilled (O04/O06), what you'd instrument and alert on (O13/O14/O16).
