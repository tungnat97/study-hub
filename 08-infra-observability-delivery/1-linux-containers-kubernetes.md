[← back to the field index](README.md)

# Infrastructure & Delivery · Part 1 — Linux, Containers and Kubernetes

Nodes `O01`–`O07`.

---

## O01 · Linux fundamentals for backend engineers

`Intermediate` · Requires: — · Unlocks: `O02`, `O04`, `O17`

### Preface

Your code runs on Linux, in a container, as a process. A handful of operating-system concepts explain
a large share of production incidents: signals, file descriptors, memory accounting, and the OOM
killer.

You do not need to be a systems administrator. You need to recognise the symptoms.

### Details

#### 1. Processes and signals

**Theory.** A signal is an asynchronous notification to a process. The two that matter:
**`SIGTERM`** (15) politely asks it to stop, and the process chooses how to respond; **`SIGKILL`** (9)
terminates it immediately and cannot be caught.

**Example.** This is the whole basis of graceful shutdown (`F28`). Kubernetes sends `SIGTERM`, waits
`terminationGracePeriodSeconds`, then sends `SIGKILL`. An application without a `SIGTERM` handler dies
instantly, dropping in-flight requests and unacknowledged jobs on every single deploy.

**Advanced.** **PID 1** has special behaviour: it does not get default signal handlers, so a process
running as PID 1 with no explicit handler **ignores `SIGTERM` entirely** and is killed after the grace
period. And PID 1 is responsible for reaping orphaned child processes, which a normal application does
not do — leading to zombie accumulation. Both are why you use the exec form of `CMD` (so your process
is PID 1 and receives signals) plus an init like `tini` if you spawn children (`O03`).

#### 2. File descriptors

**Theory.** Every open file, socket and pipe consumes a file descriptor. The per-process limit
(`ulimit -n`) is often 1,024 by default, which is far below what a connection-heavy service needs.

**Example.** `EMFILE: too many open files` under load is this limit. Two causes: legitimately many
connections (raise the limit), or a **leak** — sockets or files not closed on an error path, which
grows steadily until the process fails. `lsof -p <pid> | wc -l` counts them; a monotonic increase over
hours is a leak.

**Advanced.** Connection leaks and descriptor leaks often show as `CLOSE_WAIT` sockets accumulating
(`O02`): the peer closed and your application never called `close()`. Common in error paths that
return early without cleaning up — which is exactly what `try/finally` and Node's `pipeline()` exist
to prevent (`C16`).

#### 3. Memory

**Theory.** **RSS** (resident set size) is physical memory currently used; **virtual** is address
space reserved, which can be far larger and is mostly meaningless. **Page cache** is file data the
kernel keeps in memory and will release under pressure.

**Example.** In a container, the limit applies to RSS plus page cache attributable to the cgroup
(`O04`). When it is exceeded, the kernel's **OOM killer** terminates the process — exit code 137,
no stack trace, nothing in the application log. "The pod restarted and there is nothing in the logs"
is almost always this.

**Advanced.** The gap between your runtime's heap metric and RSS is where the confusion lives:
buffers, native allocations, thread stacks, and the JIT's code cache are all outside the heap
(`C12`). A Node process reporting 200MB of heap can legitimately use 400MB of RSS, so
`--max-old-space-size` must leave headroom below the container limit.

#### 4. Load, CPU and the basic tools

**Theory.** **Load average** counts processes that are runnable **or** in uninterruptible sleep
(usually disk I/O) — so on Linux it is not purely a CPU metric. CPU utilisation is the fraction of
time the CPU was executing.

**Example.** A load average of 12 on an 8-core machine may mean CPU saturation, or it may mean
processes blocked on slow I/O. Check `%iowait` in `top` or `vmstat` to tell them apart — the
remediation is completely different (more CPU versus faster storage or fewer I/O operations).

**Advanced.** The tools worth being able to name: `ps`/`top`/`htop` for processes, `lsof` for open
files, `ss -tanp` for sockets (`O02`), `strace` for syscalls, `dmesg` for kernel messages including
OOM kills, `df`/`du` for disk, and `/proc/<pid>/` for everything about a process. In a distroless
container none of these exist, which is why an ephemeral debug container
(`kubectl debug`) is the modern way to investigate (`O05`).

### Interview questions

- "`Too many open files` in production. What is happening and what do you check?"
- "Load average is 12 on an 8-core box. Is that bad?"
- "A pod restarted with exit code 137 and no logs. What happened?"
- "Why might a process ignore `SIGTERM` entirely?"

---

## O02 · Network debugging

`Intermediate` · Requires: `O01`, `A12`, `A13` · Unlocks: `O17`

### Preface

"Service A cannot reach service B" is one of the most common production problems, and it has a
reliable diagnostic order: work up the stack, and always test **from inside the pod**, not from your
laptop.

DNS → TCP → TLS → HTTP → auth → application. Each step has a command.

### Details

#### 1. The diagnostic ladder

**Theory.** Each layer can fail independently, and the error messages often do not distinguish them.
Test them in order and stop at the first failure.

**Example.** The commands, in order:

```bash
nslookup orders                   # 1. DNS: does the name resolve?
nc -vz orders 8080                # 2. TCP: can we connect to the port?
openssl s_client -connect orders:443 -servername orders   # 3. TLS: handshake, cert, expiry
curl -v https://orders/health     # 4. HTTP: status, headers, redirects
curl -v -H "Authorization: ..."   # 5. auth
```

Run these from a shell in the pod (`kubectl exec`, or `kubectl debug` for a distroless image), because
network policy, DNS configuration and service accounts differ from your machine.

**Advanced.** `curl -w` gives a latency breakdown that localises the problem immediately:

```bash
curl -o /dev/null -s -w 'dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n' https://orders/health
```

A large `time_namelookup` points at DNS (`A13`); a large `time_connect` at the network or a backlog;
a large `time_appconnect` at TLS; a large gap between `starttransfer` and `connect` at the
application.

#### 2. Reading socket states

**Theory.** `ss -tanp` (or `netstat -tanp`) shows every socket and its TCP state. The states tell you
what is wrong.

**Example.**
- Many **`TIME_WAIT`** on the client side — connection churn; you are not reusing connections
  (`A12`). Fix with keep-alive, not kernel tuning.
- Many **`CLOSE_WAIT`** on your side — the peer closed and your application has not called `close()`.
  An application bug, usually a leak in an error path (`O01`).
- **`SYN_SENT`** piling up — you cannot reach the peer at all: a firewall, a security group, or a
  network policy dropping packets silently.
- A full **accept queue** (`ss -lnt` shows `Recv-Q` against `Send-Q` for listening sockets) — the
  application is not accepting fast enough (`A12`).

**Advanced.** Distinguishing a **dropped** packet from a **rejected** connection matters: a rejection
returns immediately with "connection refused" (nothing is listening), while a drop hangs until the
timeout (a firewall silently discarding). "It hangs for 30 seconds then fails" is almost always a
firewall or security group; "it fails instantly" is nothing listening on that port.

#### 3. Inside Kubernetes

**Theory.** Container networking adds layers: DNS is provided by CoreDNS, service IPs are virtual and
implemented by kube-proxy or eBPF, and NetworkPolicy can silently drop traffic.

**Example.** Common causes of "cannot connect" in a cluster: the Service selector does not match the
pod labels (so the endpoint list is empty — check `kubectl get endpoints`); the pod is not **ready**,
so it was removed from endpoints (`O06`); a NetworkPolicy denies the traffic; the container listens on
`127.0.0.1` rather than `0.0.0.0`, so it is unreachable from outside the pod; or the port name in the
Service does not match the container port.

**Advanced.** The `0.0.0.0` one catches people regularly: a process bound to localhost works in
testing on the developer's machine and is unreachable in a pod, with a connection-refused error that
suggests the process is not running. And an empty `Endpoints` list is the single most informative
thing to check first — it immediately distinguishes a routing problem from a pod problem.

#### 4. Packet capture, when necessary

**Theory.** When the layers above have not explained it, `tcpdump` shows what is actually on the wire.

**Example.** `tcpdump -i any -n port 5432 -w /tmp/db.pcap` captures database traffic for analysis. Use
it to answer questions like: are the packets leaving at all; is the peer responding; who sent the
`RST`; is the TLS handshake failing and at which step. It is a last resort and it is definitive.

**Advanced.** In Kubernetes, capture from a **sidecar or ephemeral container sharing the pod's network
namespace**, since the application container usually lacks the tooling and the capability. Note that
capturing on a busy service produces enormous files, so always filter by port and host, and be aware
that captures may contain sensitive data — treat the file as sensitive and delete it afterwards
(`S12`).

### Interview questions

- "Service A cannot reach service B. Give me your diagnostic order."
- "What does a pile of `CLOSE_WAIT` sockets indicate?"
- "The connection hangs for 30 seconds then fails. What does that suggest?"
- "`kubectl get endpoints` returns nothing. What does that tell you?"

---

## O03 · Containers

`Intermediate` · Requires: `F28` · Unlocks: `O04`, `O05`, `O08`, `S14`

### Preface

A container image is a set of filesystem layers plus metadata about how to run them. A container is a
process on the host, isolated with namespaces and constrained by cgroups (`O04`).

For a backend engineer, three things matter: building images that are small and cache well, handling
signals correctly, and not shipping secrets or root privileges.

### Details

#### 1. Layers and build caching

**Theory.** Each instruction creates a layer. Layers are cached, and a change invalidates that layer
**and every layer after it**. So ordering determines build time.

**Example.** The standard pattern — dependencies before source, because source changes far more often:

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci                 # cached unless package files change
COPY . .
RUN npm run build

FROM node:22-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/main.js"]
```

Copying everything first means `npm ci` re-runs on every source change, turning a 20-second build into
five minutes.

**Advanced.** **Multi-stage builds** keep build tools, source and development dependencies out of the
final image — smaller, faster to pull, and a smaller attack surface (`S14`). Also remember that a
secret used in an earlier stage or deleted in a later layer **still exists in the layer where it was
added** and can be extracted from the image; use BuildKit secret mounts instead.

#### 2. Signals and PID 1

**Theory.** The `CMD` form determines whether your process receives signals. The **exec form**
(`CMD ["node", "app.js"]`) runs your process directly as PID 1. The **shell form**
(`CMD node app.js`) runs `/bin/sh -c` as PID 1, and the shell does not forward `SIGTERM`.

**Example.** With the shell form, Kubernetes sends `SIGTERM`, the shell ignores it, nothing drains,
and after the grace period the container is `SIGKILL`ed — dropping requests on every deploy (`F28`).
Always use the exec form.

**Advanced.** If your process spawns children (a worker pool, a subprocess), PID 1 must reap them or
zombies accumulate. Use `tini` (`--init` in Docker, or an init container pattern), or handle
`SIGCHLD` yourself. Also make sure that a wrapper script, if you use one, `exec`s the real process so
it replaces the shell rather than remaining its parent.

#### 3. Image hygiene

**Theory.** The image is part of your attack surface and part of your deployment speed.

**Example.** The checklist: use a small base (`-slim`, alpine, or **distroless** for the smallest
surface); pin the base image by digest so builds are reproducible; `.dockerignore` to keep
`node_modules`, `.git` and local files out; run as a **non-root** user; do not install debugging tools
in the production image; rebuild regularly so base-image patches are picked up; and scan the image in
CI (`S14`).

**Advanced.** Distroless images contain no shell, which is good for security and means
`kubectl exec` gives you nothing. The modern answer is `kubectl debug` with an **ephemeral container**
that shares the pod's namespaces and brings its own tools — worth knowing, because "how do you debug a
distroless container?" is a natural follow-up.

#### 4. Configuration and state

**Theory.** The same image must run in every environment, configured only by the environment (`F05`).
The container filesystem is ephemeral: anything written inside is lost on restart.

**Example.** Consequences: no configuration files baked per environment; logs to **stdout** rather
than to files inside the container (the platform collects them); uploads to object storage rather than
local disk (`F17`); and anything that must persist goes to a volume or an external store.

**Advanced.** A read-only root filesystem (`readOnlyRootFilesystem: true`, `S15`) enforces this and
frequently reveals surprises — a library writing a cache to `/tmp`, a framework writing a build
artefact at startup. Mount an `emptyDir` at `/tmp` for those. It is a small hardening step that also
makes the ephemeral nature of the filesystem explicit.

### Interview questions

- "Your container ignores `SIGTERM` on deploy. Two likely causes."
- "Rebuilding takes nine minutes because one line changed. Fix the Dockerfile."
- "How do you debug a distroless container?"
- "A secret was used during build and deleted in a later layer. Is it gone?"

---

## O04 · Container runtime and resources

`Advanced` · Requires: `O01`, `O03` · Unlocks: `O06`, `C12`

### Preface

Containers are not virtual machines. They are processes on the host, isolated by **namespaces**
(what they can see) and limited by **cgroups** (what they can use).

Two consequences dominate production: a CPU limit throttles you in ways that are invisible in average
CPU metrics, and a memory limit kills you with no warning and no stack trace.

### Details

#### 1. Namespaces and cgroups

**Theory.** **Namespaces** isolate the view: PID (its own process tree), network (its own interfaces),
mount (its own filesystem view), UTS (hostname), IPC, and user. **Cgroups** limit consumption: CPU,
memory, block I/O, and process count.

**Example.** This is why a container sees itself as PID 1 with its own network stack, and why it is
much lighter than a virtual machine — there is no second kernel, just a constrained view of the host's.
It is also why container isolation is weaker than a VM's: a kernel vulnerability crosses the boundary
(`S15`).

**Advanced.** The classic consequence of namespace isolation being incomplete: older runtimes let a
process read `/proc/cpuinfo` from the **host**, so a runtime sizing its thread pool by "available
processors" saw 64 cores on a container limited to one. The JVM and Node are now cgroup-aware, and
libraries frequently are not — so check what your thread pools and worker counts actually size
themselves from (`F28`).

#### 2. CPU limits and throttling

**Theory.** A CPU **request** is used for scheduling — how much the node reserves for you. A CPU
**limit** is enforced by the CFS quota: within each 100ms period, the container gets at most its
quota, and once exhausted it is **throttled** until the next period.

**Example.** This produces a genuinely confusing symptom: p99 latency spikes every few hundred
milliseconds while average CPU utilisation shows 30%. The container is bursting to its quota, being
throttled for the remainder of the period, and requests in flight simply stop. The metric to look at
is `container_cpu_cfs_throttled_periods_total` — utilisation will not show it.

**Advanced.** This is why many practitioners recommend setting CPU **requests** and **no CPU limit**:
requests guarantee your share under contention, and the absence of a limit lets you use idle capacity
instead of being throttled while the node is idle. The counter-argument is predictability and
noisy-neighbour protection. Being able to argue both sides — and to note that memory limits are a
different case, because memory is not compressible and must be limited — is the senior answer.

#### 3. Memory limits and OOMKill

**Theory.** Memory cannot be throttled: when a cgroup exceeds its limit, the kernel kills a process in
it. The container exits with code **137** (128 + SIGKILL) and Kubernetes reports `OOMKilled`.

**Example.** There is no stack trace and no application log entry, because the process was killed
without warning. Diagnose from the pod's `lastState.terminated.reason`, from `dmesg`, or from the
container restart count. Then the question is whether the limit is too low or the application leaks
(`C12`).

**Advanced.** The runtime's own memory setting must sit **below** the container limit, with headroom
for non-heap memory: `--max-old-space-size` for Node at roughly 75-80% of the limit,
`-XX:MaxRAMPercentage=75` for the JVM (which is container-aware by default in modern versions). If the
runtime's cap is above the container's, you are OOMKilled before the garbage collector ever feels
pressure — which is why "the heap looks fine but we keep getting killed" is so common (`O01`).

#### 4. Requests, limits and quality of service

**Theory.** Kubernetes assigns a QoS class from the relationship between requests and limits:
**Guaranteed** (requests equal limits for all resources — evicted last), **Burstable** (requests below
limits), **BestEffort** (neither — evicted first).

**Example.** Under node memory pressure, the kubelet evicts BestEffort pods first, then Burstable ones
exceeding their requests, then Guaranteed. So a critical service should set memory requests equal to
limits, making it Guaranteed and the last to be evicted. This matters more than people realise during
a node incident.

**Advanced.** Requests also drive **scheduling** and therefore cost: a pod requesting 2 CPUs occupies
that allocation on a node whether or not it uses it. Systematically over-requesting is one of the
largest sources of waste in a cluster (`O18`) — right-size from observed usage (p95 of actual, plus
headroom) rather than from guesses, and revisit it periodically.

### Interview questions

- "Pods are being OOMKilled but the application's heap metric looks fine. Explain."
- "p99 latency spikes every few seconds at 30% CPU. Cause?"
- "Would you set a CPU limit? Argue both sides."
- "What determines which pod gets evicted under memory pressure?"

---

## O05 · Kubernetes basics

`Intermediate` · Requires: `O03` · Unlocks: `O06`, `O07`, `O09`, `M05`

### Preface

Kubernetes is a control loop: you declare the desired state, and controllers work continuously to
make reality match it.

For a backend engineer the useful knowledge is the handful of objects your service consists of, and
the four commands that diagnose most problems.

### Details

#### 1. The objects you actually use

**Theory.**
- **Pod** — one or more containers sharing a network namespace and volumes. The unit of scheduling.
- **Deployment** — manages a ReplicaSet, which keeps N identical pods running; handles rolling
  updates.
- **Service** — a stable virtual IP and DNS name in front of a set of pods selected by labels.
- **Ingress / Gateway** — routes external HTTP traffic to Services.
- **ConfigMap / Secret** — configuration and credentials, injected as environment variables or files.
- **Job / CronJob** — run-to-completion work.

**Example.** A typical service is a Deployment (3 replicas), a Service (ClusterIP), an Ingress rule, a
ConfigMap for configuration, and a Secret for credentials — five YAML objects.

**Advanced.** Kubernetes `Secret`s are **base64-encoded, not encrypted**, and are stored in etcd. They
are only meaningfully secret if encryption at rest is enabled on etcd, RBAC restricts who can read
them, and `automountServiceAccountToken` is off where unnecessary (`S15`). Many teams use an external
secret manager synced into Kubernetes, which is better, and the pod still ends up holding the value.

#### 2. Labels, selectors and the reconciliation loop

**Theory.** Objects are connected by **labels** and **selectors**, not by direct references. A Service
sends traffic to whichever pods match its selector, at any moment. Controllers continuously compare
desired state with actual state and act on the difference.

**Example.** This explains a common failure: a Service whose selector does not match the pods' labels
produces an empty `Endpoints` list and "connection refused" or a hang, with nothing obviously wrong in
either object (`O02`). Always check `kubectl get endpoints <service>` — an empty list localises the
problem instantly.

**Advanced.** The reconciliation model is why Kubernetes is resilient and why it is sometimes
surprising: deleting a pod does not remove it, because the ReplicaSet recreates it; and a manual
change to a managed object is reverted at the next reconciliation. Everything must be changed through
the declared desired state — which is the argument for GitOps (`O10`).

#### 3. Triage commands

**Theory.** Four commands answer most questions.

**Example.**

```bash
kubectl get pods                    # status, restarts, age
kubectl describe pod <name>         # events: scheduling, image pull, probe failures, OOMKilled
kubectl logs <name> --previous      # logs from the container BEFORE the last restart
kubectl get events --sort-by=.lastTimestamp
```

`describe` is the highest-value one: the Events section at the bottom explains most failures —
`ImagePullBackOff`, `FailedScheduling` (insufficient resources), `Unhealthy` (probe failing),
`OOMKilled`.

**Advanced.** `--previous` is essential for crash loops, because the current container has just
started and its logs are empty — the useful logs are from the instance that died. And the status tells
you the category: `CrashLoopBackOff` (the container starts and exits — an application error),
`ImagePullBackOff` (registry or credentials), `Pending` (cannot be scheduled — resources, node
selectors, or unbound volumes), `ContainerCreating` (volumes or secrets not available).

#### 4. What happens on `kubectl apply`

**Theory.** The API server validates and stores the object; controllers notice and act; the scheduler
assigns pods to nodes; the kubelet on each node pulls images and starts containers; probes determine
readiness; endpoints update; traffic flows.

**Example.** Being able to narrate that sequence localises problems: stuck at `Pending` is the
scheduler (resources, affinity, volumes); stuck at `ContainerCreating` is the kubelet (image, volumes,
secrets); `CrashLoopBackOff` is your application; running but not receiving traffic is readiness or
the Service selector (`O06`).

**Advanced.** Endpoint propagation is **eventually consistent** and takes time to reach every
kube-proxy and every client: this is precisely why a terminating pod must keep serving during a
`preStop` sleep (`M05`, `F28`). Understanding that the endpoint update and the pod's termination
happen **in parallel**, not in sequence, is the key insight behind zero-downtime deploys.

### Interview questions

- "A pod is in `CrashLoopBackOff`. Your first four commands?"
- "What actually happens between `kubectl apply` and traffic reaching a new pod?"
- "The Service exists and the pods are running, but nothing works. What do you check?"
- "Are Kubernetes Secrets encrypted?"

---

## O06 · Kubernetes for service owners

`Advanced` · Requires: `O04`, `O05`, `C12` · Unlocks: `O09`, `O16`, `M05`, `F28`

### Preface

This is the node that matters most for a backend engineer, because it contains the settings that
determine whether your service is stable and whether deploys drop requests.

Two things above all: the **probes** (and why liveness must not check dependencies), and the complete
**zero-downtime deploy** recipe.

### Details

#### 1. The three probes

**Theory.**
- **Liveness** — is the process wedged? Failing it **restarts** the container.
- **Readiness** — can it serve traffic now? Failing it **removes** the pod from Service endpoints,
  without restarting.
- **Startup** — has it finished starting? While it is running, liveness and readiness are suspended,
  so a slow boot is not mistaken for a hang.

**Example.** The dangerous mistake: a liveness probe that checks the database. The database has a
hiccup, every pod's liveness fails simultaneously, every pod restarts, they all reconnect at once, the
database is overwhelmed, and a brief degradation becomes a total outage with a restart loop. **Liveness
must check only this process's own health** — typically that the event loop is responsive.

**Advanced.** Readiness **may** check dependencies, carefully: it removes the pod from traffic rather
than killing it, so there is a recovery path. But if every pod's readiness depends on the database,
you still take the whole service out of rotation — which may be correct (it cannot serve anyway) or
may be worse than serving degraded responses. Decide deliberately, and consider a readiness check that
only fails when the **pod's own** connection pool is unusable, not when a query is slow.

#### 2. The complete zero-downtime deploy

**Theory.** Dropped requests during a deploy are caused by a pod being removed from service before it
stops receiving traffic, or being killed before it finishes in-flight work. Both are configuration
problems.

**Example.** The full recipe — be able to list all of it:
1. `maxUnavailable: 0` and `maxSurge: 1` so capacity never dips during the roll.
2. **Readiness probe** configured so a new pod receives traffic only when it can serve.
3. On `SIGTERM`: fail readiness, stop accepting new work, finish in-flight requests, close the
   database pool and broker connections, flush telemetry, exit (`F28`).
4. A **`preStop` hook** that sleeps 5-15 seconds, during which the pod **keeps serving** — because
   endpoint removal propagates asynchronously (`M05`).
5. `terminationGracePeriodSeconds` longer than the preStop sleep plus the maximum request duration.
6. Clients and load balancers handle connection closure — send `Connection: close` while draining, or
   set a maximum connection age.
7. **Database migrations** compatible with both versions, because both run simultaneously (`DB22`,
   `M24`).

**Advanced.** The counter-intuitive step is 4: the pod must keep serving **after** it has been told to
stop. Everyone's first instinct is to close the server on `SIGTERM` immediately, and that is precisely
what causes the dropped requests. Explaining *why* — parallel, eventually-consistent endpoint
propagation — is what makes this a senior answer rather than a memorised list.

#### 3. Scaling

**Theory.** The **HorizontalPodAutoscaler** adjusts replica count based on a metric. CPU is the
default and is frequently the wrong signal.

**Example.** For an I/O-bound service, CPU stays low while latency climbs — so scaling on CPU never
triggers. Better signals: requests per second per pod, p99 latency, queue depth or **oldest-message
age** for workers (`Q21`), or concurrent in-flight requests. These need a metrics adapter (KEDA is the
common choice, and it scales on queue length directly).

**Advanced.** Configure the scaling **behaviour**, not just the target: scale up quickly and scale
down slowly (a `stabilizationWindowSeconds` on scale-down), or you get flapping — scaling down after
a brief lull and immediately back up, with each cycle causing cold starts and connection churn. And
remember that scaling out multiplies your database connections (`DB21`), so the maximum replica count
must be consistent with the database's capacity.

#### 4. Disruption and placement

**Theory.** A **PodDisruptionBudget** limits how many pods may be voluntarily disrupted at once
(node drains, cluster upgrades). **Anti-affinity** and **topology spread constraints** keep replicas
on different nodes and zones.

**Example.** Without a PDB, a node drain during a cluster upgrade can evict every replica of your
service at once. With `minAvailable: 2`, the drain waits. Without anti-affinity, all three replicas
may be scheduled onto the same node, so one node failure is a total outage — which entirely defeats
running three replicas.

**Advanced.** A PDB that cannot be satisfied **blocks** node drains indefinitely — a
`minAvailable` equal to the replica count means the cluster can never be upgraded. Set it as
`maxUnavailable: 1` or a proportion, and make sure `replicas > minAvailable`. This is a real
operational trap that surfaces months later during an upgrade.

### Interview questions

- "Give me the complete list of things needed for zero-dropped-request deploys."
- "Your liveness probe hits `/health` which checks the database. Why is that dangerous?"
- "You scale on CPU but your I/O-bound service never scales. What do you use instead?"
- "What does a PodDisruptionBudget do, and how can it break a cluster upgrade?"

---

## O07 · Stateful workloads and platform services

`Intermediate` · Requires: `O05` · Unlocks: `O20`

### Preface

Stateless services are easy to run on Kubernetes: any pod is interchangeable. Databases are not —
they have identity, persistent storage, ordered startup, and failover semantics.

The honest senior answer to "would you run Postgres in Kubernetes?" is usually **use the managed
service**, with a clear account of what you give up and what you gain.

### Details

#### 1. StatefulSets and persistent volumes

**Theory.** A **StatefulSet** gives pods stable identities (`db-0`, `db-1`) and stable storage: each
pod keeps its own PersistentVolumeClaim across restarts and rescheduling. Startup and termination are
ordered.

**Example.** That is enough for a single-instance database with a volume, and it is **not** enough for
a clustered one: Kubernetes knows how to start pods in order, and it does not know how to promote a
replica, re-point clients, or rejoin a recovered primary. That logic lives in an **operator** — a
controller encoding the database's own failover procedure (CloudNativePG, Zalando's Postgres
operator, Strimzi for Kafka).

**Advanced.** Storage is the part that surprises people: a PersistentVolume is usually zone-bound
(an EBS volume exists in one availability zone), so a pod using it can only be scheduled in that zone.
A zone failure means that pod cannot be rescheduled anywhere until the zone returns. Replication
across zones therefore has to be handled by the **database**, not by the storage layer.

#### 2. The managed-service argument

**Theory.** Running a database well means backups that are tested, point-in-time recovery, failover
that works, version upgrades, patching, monitoring, and someone who knows what to do at 3am. A managed
service provides all of it.

**Example.** What you give up: extensions the provider does not support, superuser access,
fine-grained configuration, some performance at the margin, and portability. What you gain: automated
backups with PITR (`DB36`), multi-AZ failover, patching, monitoring, and — most importantly — not
being the person responsible when it fails.

**Advanced.** The honest case for self-hosting: cost at very large scale, a specific extension or
configuration you need, data-residency requirements the provider cannot meet, or an existing platform
team with the expertise. The case against: everything else. For a product team without a platform
team, running your own database is a commitment whose cost is invisible until the first incident.

#### 3. If you do run it yourself

**Theory.** Use an operator, and treat the database's operational requirements as first-class.

**Example.** The requirements: an operator that handles failover and has been tested under failure;
fast local storage (network-attached storage adds latency to every write, and databases are
latency-sensitive); resource **guarantees** rather than bursty allocations (requests equal to limits,
`O04`); anti-affinity so replicas are on different nodes and zones (`O06`); a PodDisruptionBudget so
a cluster upgrade does not take a quorum down; and **tested** backup and restore (`DB36`).

**Advanced.** The failure mode to prepare for is that Kubernetes will try to "help": a node becomes
unreachable, the pod is marked for deletion and rescheduled, and now two instances may believe they
are primary — split-brain (`M33`). The operator must fence the old instance before promoting. Verify
that your operator does this, and test it by killing a node, before relying on it.

#### 4. Other stateful platform services

**Theory.** The same reasoning applies to Kafka, Redis, Elasticsearch and anything else holding data:
managed unless you have a strong reason.

**Example.** Kafka is the clearest case — partition rebalancing, broker replacement, and upgrades are
genuinely difficult, and a mistake loses data. MSK or Confluent Cloud costs more and removes an
entire operational category. Redis as a **cache** is the most defensible to self-host, because losing
it is survivable (`Q04`).

**Advanced.** A useful framing when asked: ask what happens when it fails at 3am on a Sunday. If the
answer is "the provider handles it", that is what you are paying for. If the answer is "I get paged
and follow a runbook I have never used", the managed service is cheaper than it looks once you price
the incident, the on-call burden and the risk.

### Interview questions

- "Would you run Postgres in Kubernetes? Defend your answer."
- "What does a StatefulSet give you that a Deployment does not?"
- "Why can a pod with a persistent volume only be scheduled in one zone?"
- "What is the risk of Kubernetes rescheduling a database pod?"
