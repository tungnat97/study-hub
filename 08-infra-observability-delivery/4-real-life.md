[← back to the field index](README.md)

# Infrastructure, Observability & Delivery · Part 4 — Real-life production problems

Parts 1–3 teach the nodes `O01`–`O20` the way a textbook would. This part is the way an interviewer
who has been paged at 3am tests them: they describe a real production problem, with real symptoms,
and wait to see whether your first instinct is the textbook answer or the one that actually fixes it.

How to use it:

- Read the **Pre-knowledge** first. It is dense on purpose: every question below can be solved with
  what is in it, and most of it is the kind of detail that is only learned by being paged.
- For each question, **answer out loud before reading the direction line**. Say what you would look
  at first, what you expect to find, and what you would change. Then compare.
- The **direction** line is not a full answer. It names the non-obvious insight and points at the
  pre-knowledge section (`§n`) and the nodes it relies on.
- Levels run from the most commonly asked (Level 1) to the rarest and most compound (Level 10). A
  senior candidate should be comfortable to Level 6 and able to reason through the rest.

---

## Pre-knowledge

### 1. Processes, signals and PID 1

- **`SIGTERM` (15)** asks a process to stop; **`SIGKILL` (9)** cannot be caught. Exit codes from a
  signal are `128 + n`: **137** = killed by `SIGKILL` (OOM killer or grace period expiry), **143** =
  exited on `SIGTERM`, **139** = `SIGSEGV`, **134** = `SIGABRT` (often a native assertion or a JVM
  crash). Exit code **0 on `SIGTERM`** is what a clean shutdown handler produces (`O01`).
- **PID 1 is special.** The kernel does not apply default signal dispositions to PID 1 in a namespace,
  so a PID 1 with no handler for `SIGTERM` simply ignores it. The pod then sits for the full
  `terminationGracePeriodSeconds` (default **30s**) and is `SIGKILL`ed. Symptom: every deploy takes
  exactly 30 seconds per pod and in-flight work is lost.
- **Shell form `CMD npm start`** runs `/bin/sh -c "npm start"`. `sh` becomes PID 1, does not forward
  signals, and your app never sees `SIGTERM`. `npm` and `yarn` as wrappers historically also swallowed
  or mangled signals. Use the **exec form** (`CMD ["node", "server.js"]`) or `exec` in an entrypoint
  script (`exec "$@"`) so the real process replaces the shell (`O03`).
- **Zombie reaping.** An orphaned child is re-parented to PID 1, which must `wait()` on it. An app that
  shells out (`ImageMagick`, `git`, headless Chrome, a health-check script) as PID 1 accumulates
  `<defunct>` processes. They hold a PID each, and the cgroup `pids.max` limit (or the node's
  `kernel.pid_max`) eventually makes `fork()` fail with `EAGAIN` / "Resource temporarily unavailable".
- **`tini` / `dumb-init`** are tiny inits: they run as PID 1, forward signals to the child, and reap
  zombies. `docker run --init` injects tini. `tini -g` sends the signal to the whole process group;
  `dumb-init` can rewrite signals (`--rewrite 15:3`) for apps that expect a different stop signal
  (nginx graceful quit is `SIGQUIT`). The `STOPSIGNAL` Dockerfile instruction also changes it.
- **`shareProcessNamespace: true`** in a pod makes the `pause` container PID 1, which reaps zombies —
  and also means your app is no longer PID 1, so default signal behaviour returns.
- **Graceful shutdown order**: stop accepting (close listener / fail readiness), drain in-flight,
  stop consumers and commit offsets, flush telemetry, close pools, exit. Many frameworks close the
  listener immediately on `SIGTERM`, which is wrong on Kubernetes (see §7).

### 2. File descriptors, sockets and connection tables

- **FD limits.** `ulimit -n` soft/hard, per process; `/proc/<pid>/limits` shows the effective value;
  `ls /proc/<pid>/fd | wc -l` counts usage. `EMFILE` = per-process limit, `ENFILE` = system-wide
  (`fs.file-max`). A monotonic climb over hours is a leak, usually on an error path (`O01`).
- **The opposite trap.** Some container runtimes shipped `LimitNOFILE=infinity`, giving containers a
  soft limit around **1,073,741,816**. Programs that loop over every possible FD on start-up to close
  them (older daemons, some `fork`/`exec` helpers, Python `subprocess` with `close_fds` on old versions,
  `yum`/`rpm`) then take minutes to start or spin at 100% CPU. The fix is to set a sane limit
  (e.g. 1,048,576 or 65,536), not to raise it further.
- **Deleted but open.** A log file deleted while a process still holds it open keeps consuming disk:
  `df` says full, `du` says empty. `lsof +L1` lists them; restarting or truncating via
  `/proc/<pid>/fd/<n>` frees the space.
- **`inotify` limits.** `fs.inotify.max_user_instances` defaults to **128** and
  `max_user_watches` to 8,192 on many distributions (newer kernels scale it by RAM). They are per user
  and shared across all containers on a node running as the same UID. Symptom: "too many open files"
  from `kubectl logs -f`, file watchers, or config hot-reloaders, while the FD count is low.
- **TCP states that matter.** `CLOSE_WAIT` accumulating = your side never called `close()` after the
  peer closed (a leak in your code). `TIME_WAIT` accumulating = your side closed first; on Linux it
  lasts a fixed **60s** and is harmless unless you run out of ports. `SYN_SENT` stuck = the far side
  or a firewall is silently dropping. `ss -tan state close-wait | wc -l`, `ss -s` for a summary.
- **Ephemeral ports.** Outbound connections to one `(dst IP, dst port)` from one source IP can use
  `net.ipv4.ip_local_port_range`, default **32768–60999** (~28k ports). At ~28k / 60s TIME_WAIT you
  cap out near **470 new connections per second** to a single destination. Symptom:
  `EADDRNOTAVAIL` / "Cannot assign requested address". Fixes in order: **reuse connections**
  (keep-alive, pooling), widen the range, `net.ipv4.tcp_tw_reuse=1` (safe for outbound).
  `tcp_tw_recycle` was broken behind NAT and **removed in kernel 4.12**; anyone recommending it is
  quoting an old blog.
- **conntrack.** Netfilter tracks every flow for NAT and stateful rules (kube-proxy, Docker, security
  groups on the host). When `nf_conntrack_max` is reached the kernel logs
  **`nf_conntrack: table full, dropping packet`** in `dmesg` and silently drops new flows — looks like
  random timeouts. kube-proxy sets the max to `max(131072, 32768 × cores)` by default. Established TCP
  entries live **5 days** (`nf_conntrack_tcp_timeout_established=432000`), so leaked idle connections
  fill it. Check `conntrack -S` (`insert_failed`, `drop`) and `/proc/sys/net/netfilter/nf_conntrack_count`.
- **Listen backlog.** `net.core.somaxconn` (4096 since kernel 5.4, 128 before) caps the accept queue;
  the app's `listen(backlog)` also caps it. Overflows show in `nstat -az TcpExtListenOverflows
  TcpExtListenDrops` and as `ss -lnt` Recv-Q equal to Send-Q. Clients see connect timeouts, not errors,
  and retry after 1s, 3s — the tell-tale "some requests take exactly 1s or 3s more".
- **Keep-alive idle timeouts must be ordered.** The **upstream (server) idle timeout must be longer
  than the downstream (client/LB) one.** AWS ALB idle timeout defaults to **60s**; Node's
  `server.keepAliveTimeout` defaults to **5s**. The server closes an idle connection, the ALB reuses it
  in the race window, and you get sporadic **502s** with no application error. Same shape between an
  HTTP client pool and a server, and between an app pool and a NAT gateway (§14) (`O02`, `A12`).
  Idle timeouts count traffic in **both** directions, so a WebSocket whose client only listens is cut
  at 60s unless the server sends ping frames.
- **MTU black holes.** Overlay networks (VXLAN adds 50 bytes), VPNs and IPsec reduce the effective MTU.
  If ICMP "fragmentation needed" is blocked, **small requests work and large responses hang** —
  TLS handshakes with big certificate chains, large JSON responses, database result sets. Test with
  `ping -M do -s 1472 <host>` and lower sizes; fix with correct MTU on the CNI or MSS clamping.

### 3. Linux memory, page cache and the OOM killer

- **RSS** is resident anonymous + file-backed pages; **VSZ** is reserved address space and mostly
  irrelevant (Go and the JVM reserve huge ranges). **Page cache** is file data kept in RAM and
  reclaimable — but not all of it is instantly reclaimable (dirty pages must be written first; active
  pages are reclaimed after inactive ones) (`O01`, `C12`).
- **Off-heap memory.** Heap is only part of RSS: thread stacks (default 1MB reserved per JVM thread,
  8MB virtual per glibc thread), metaspace and code cache, direct `ByteBuffer`s, Netty pools, native
  libraries (`sharp`, `librdkafka`, gRPC core), and **glibc malloc arenas** — up to `8 × cores` arenas,
  each fragmenting separately. `MALLOC_ARENA_MAX=2` or switching to `jemalloc` routinely cuts RSS by
  30–50% on multi-threaded native-heavy processes. RSS that grows with no heap growth is usually this.
- **JVM in containers.** Modern JVMs (10+, backported to 8u191) are container-aware; default max heap is
  **25%** of the container limit, which is usually too small — set `-XX:MaxRAMPercentage=60..75`, never
  100. Node: `--max-old-space-size` must leave headroom for buffers. Go: `GOMEMLIMIT` (1.19+) gives the
  GC a soft target below the limit; without it, the heap can double before GC (default `GOGC=100`).
- **Transparent huge pages** (`/sys/kernel/mm/transparent_hugepage/enabled = always`) can cause RSS
  bloat and latency spikes from `khugepaged` compaction; Redis, MongoDB and many databases recommend
  `madvise` or `never`.
- **OOM killer.** When a cgroup (or the node) cannot reclaim enough memory, the kernel picks the process
  with the highest `oom_score` (roughly its memory share plus `oom_score_adj`) and sends `SIGKILL`.
  The evidence is in the kernel log, not the application log:
  `dmesg -T | grep -i -E "killed process|oom"` shows "Memory cgroup out of memory: Killed process ...".
  In Kubernetes `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.
- **Container OOM vs node OOM vs eviction** are three different things (§6). A container OOMKill is
  the cgroup limit; a node-level OOM kills whichever process scores highest on the node; an eviction is
  the kubelet deciding to terminate a pod gracefully before the kernel has to.
- **`oom_score_adj` by QoS class**: Guaranteed **−997**, BestEffort **1000**, Burstable computed as
  `min(max(2, 1000 − 1000 × memoryRequest / nodeCapacity), 999)` — so a Burstable pod with a tiny
  request is almost as killable as BestEffort.
- **Swap** was historically disabled on Kubernetes nodes; swap support (`NodeSwap`) exists in recent
  versions but is opt-in and limited to Burstable pods.

### 4. cgroups: memory accounting and CPU throttling

- **cgroup v2** is the default on modern distributions and Kubernetes ≥ 1.25 supports it fully. Files:
  `memory.max` (hard limit), `memory.high` (throttle-and-reclaim threshold), `memory.current`,
  `memory.stat`, `memory.events` (`oom`, `oom_kill`, `high`), `memory.pressure` (PSI);
  `cpu.max` (`quota period`), `cpu.stat` (`nr_throttled`, `throttled_usec`), `pids.max`.
  v1 used `memory.limit_in_bytes`, `cpu.cfs_quota_us` / `cpu.cfs_period_us`.
- **Page cache is charged to the cgroup** that first touched the page. A container reading or writing
  large files (log shipping, a file upload buffered to disk, SQLite, a batch export) shows memory
  climbing to the limit. That is usually fine — it is reclaimable — except that:
  - **`container_memory_working_set_bytes`** = usage − inactive_file. It includes **active** page cache,
    so dashboards and the kubelet can see "memory near limit" that is mostly cache.
  - `container_memory_usage_bytes` includes all cache and is almost useless for alerting. **Alert on
    working set and RSS**, and look at `memory.stat` (`anon`, `file`, `active_file`, `shmem`) to
    see what is really there.
  - **tmpfs counts as memory.** An `emptyDir` with `medium: Memory`, `/dev/shm`, or a large `/tmp` on
    tmpfs is charged to the container and cannot be reclaimed — writing a 2GB temporary file to it
    OOMKills a 2GB pod.
- **CFS CPU quota.** A CPU **limit** becomes a quota per **100ms period**. `limits.cpu: 1` = 100ms of
  CPU time per 100ms, **across all threads**. A process with 8 busy threads on an 8-core node burns the
  quota in 12.5ms and is **throttled for 87.5ms** — p99 latency jumps by tens of milliseconds while
  average CPU usage looks like 40%. Metrics:
  `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])`;
  inside the container `cat /sys/fs/cgroup/cpu.stat`. Old kernels (< 5.4) also had a quota-expiry bug
  that throttled apps well below their limit (`O04`).
- **The unconventional fix**: for latency-sensitive services, set CPU **requests** accurately and
  **remove CPU limits** (or set them well above request), keeping memory limit = request. CPU is
  compressible; the scheduler uses requests for placement and the CFS share weights still apportion
  CPU under contention. Many large operators run this way. Where limits are mandatory, match thread
  pools to the limit, not the node.
- **Runtimes guess cores from the node.** Old JVMs, Node's libuv thread pool (`UV_THREADPOOL_SIZE`,
  default 4), .NET, and Go before **1.25** (which made `GOMAXPROCS` cgroup-aware; before that use
  `uber-go/automaxprocs`) size GC threads and schedulers from the host's cores. Node's libuv pool
  runs `fs` calls, `dns.lookup` (which is what `http.request` uses by default), `crypto.pbkdf2`/`scrypt`,
  and `zlib` — a slow DNS server or heavy hashing starves file I/O while event-loop lag looks fine. A Go service with
  `GOMAXPROCS=64` under a 2-CPU limit throttles constantly. `-XX:ActiveProcessorCount` pins the JVM.
- **PSI** (pressure stall information: `/proc/pressure/{cpu,memory,io}` and per cgroup) reports the
  share of time tasks were stalled on a resource — the best single saturation signal on a node.

### 5. Container images and debugging containers

- **Layers** are cached by instruction and content. `COPY . .` before `RUN npm ci` invalidates the
  dependency layer on every source change; copy the lockfile first. Deleting a file in a later layer
  does not shrink the image (and does not remove a leaked secret from history). Multi-stage builds keep
  build tools out of the runtime image (`O03`).
- **Tags are mutable.** `:latest` or `:1.4` can point to a different image tomorrow; two pods of the
  same Deployment can run different code if the tag moved between pulls. Deploy by **digest**
  (`image@sha256:...`) and pin base images by digest with automated bumps.
- **Architecture.** An image built on an arm64 laptop and pushed under a shared tag fails on amd64
  nodes with `exec format error` (and vice versa). Build multi-arch manifests in CI (`docker buildx
  --platform linux/amd64,linux/arm64`), never push release tags from laptops.
- **`imagePullPolicy`**: `Always` for `:latest` (implicit), `IfNotPresent` otherwise. With
  `Always`, a registry outage or rate limit stops pods from starting even though the image is cached.
  Anonymous Docker Hub pulls are rate-limited **per source IP**, and a whole cluster behind one NAT
  gateway shares one IP. Mirror base images into your own registry.
- **Distroless / scratch** images have no shell, so `kubectl exec -- sh` fails. Use an
  **ephemeral container**: `kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>`
  shares the target's process namespace; the target's filesystem is at `/proc/<pid>/root`. For node
  issues, `kubectl debug node/<node> -it --image=ubuntu` gives a pod with the host filesystem at
  `/host`.
- **`strace`/`perf` in containers** need `SYS_PTRACE` / `perfmon` capabilities, or run them from the
  node against the host PID (`crictl inspect` → `.info.pid`, or `nsenter -t <pid> -n -p`). `nsenter
  -t <pid> -n ss -tanp` runs the node's `ss` inside the pod's network namespace — works on distroless.
- **Alpine / musl** is not a free size win: musl's `malloc` is markedly slower for multi-threaded
  allocation-heavy workloads, its resolver ignores `single-request-reopen`, and versions before 1.2.4
  did not retry over TCP when a UDP DNS answer was truncated (large record sets fail to resolve).
  Debian-slim, distroless or Chainguard-style glibc images avoid all three.
- **Image pull time** dominates cold scale-up: a 2GB image on a fresh node can take a minute. Pre-pull
  with a DaemonSet, use lazy-loading snapshotters, or simply make the image small.
- **Writable layer.** Writes inside the container filesystem go to the overlay upper dir on the node
  and count against **ephemeral-storage**; an app writing logs to a file inside the container fills
  the node disk and triggers eviction (§6).

### 6. Kubernetes resources, QoS and eviction

- **Requests** are what the scheduler reserves; **limits** are what the kernel enforces. Memory limit
  exceeded = OOMKill; CPU limit reached = throttling (§4) (`O04`, `O06`).
- **QoS classes**: **Guaranteed** (every container has requests = limits for CPU and memory),
  **Burstable** (some requests set), **BestEffort** (none). Under node pressure the kubelet evicts
  BestEffort first, then Burstable pods using the most memory **above their request**. Memory
  requests = limits is the standard advice for anything important.
- **Node allocatable** = capacity − `kube-reserved` − `system-reserved` − eviction threshold. If
  reservations are too small, the kubelet, containerd or journald get OOMKilled and the node goes
  `NotReady` — and every pod on it is rescheduled at once.
- **Eviction thresholds** (defaults): hard `memory.available<100Mi`, `nodefs.available<10%`,
  `imagefs.available<15%`, `nodefs.inodesFree<5%`. The kubelet samples on an interval (~10s), so a fast
  allocation spike beats it and the **kernel OOM killer acts first** — the pod shows OOMKilled, not
  Evicted. Evicted pods remain as `Failed` objects until garbage-collected.
- **Ephemeral storage**: container logs, writable layer and `emptyDir` (disk). Set
  `resources.limits.ephemeral-storage` or one noisy pod evicts its neighbours via `DiskPressure`.
  Inode exhaustion (millions of small cache files) triggers pressure with free space showing.
- **Kubelet log rotation**: `containerLogMaxSize` 10Mi, `containerLogMaxFiles` 5 by default. A chatty
  pod's logs rotate faster than the log shipper reads them, and lines are lost silently.
- **Overcommit.** Sum of limits can exceed node capacity; sum of requests cannot. Nodes full of pods
  with low requests and high limits are fine until they are all busy at once — then the node OOMs.
- **ResourceQuota / LimitRange** can inject default requests and limits you did not write; check
  `kubectl describe limitrange` before assuming your manifest is what runs.
- **Secrets and ConfigMaps**: mounted as volumes they update in place (kubelet sync, roughly a minute),
  **but not when mounted with `subPath`**, and **never when consumed as environment variables**. A
  rotated database password therefore reaches half your pods, or none, until they restart.

### 7. Probes, termination and zero-downtime rollouts

- **Liveness** = "restart me"; **readiness** = "stop sending me traffic"; **startup** = "do not run
  the other probes until I have started". Defaults: `periodSeconds 10`, `timeoutSeconds 1`,
  `failureThreshold 3` (`O06`).
- **The classic misuse**: a liveness probe that checks the database or a downstream. When the
  dependency is slow, every pod fails liveness together, all restart together, the cold restarts
  stampede the dependency, and a brownout becomes a full outage. **Liveness should check only the
  process itself** (is the event loop / thread pool responsive). Readiness may check critical local
  state, but checking a shared dependency from readiness can also remove every pod at once — the
  surviving answer is often "serve degraded rather than go unready".
- **`timeoutSeconds: 1`** is tight: a GC pause, CPU throttling (§4) or a slow cold JIT fails probes
  under load, causing restarts precisely when capacity is needed. **`exec` probes** fork a process
  every period in every pod — hundreds of short-lived shells per second on a busy node, visible only
  to `execsnoop`, not `top`. Prefer HTTP, TCP or gRPC probes. Probes also compete for the same
  thread pool as real traffic — a saturated server fails its own health check.
- **Termination sequence.** On delete, two things happen **in parallel**: the kubelet runs `preStop`
  then sends `SIGTERM`; and the endpoint controller removes the pod from EndpointSlices, which
  kube-proxy on every node, ingress controllers and service meshes then apply **asynchronously** —
  often 1–5s, longer on large clusters. A pod that exits on `SIGTERM` immediately receives traffic
  it can no longer serve: **connection refused / 502 on every deploy**.
- **The fix**: a `preStop` hook that **sleeps** (5–15s) before `SIGTERM` is delivered, so routing
  converges while the app still serves. Kubernetes 1.29+ has a native `sleep` preStop action for
  distroless images with no `sleep` binary. After `SIGTERM`, keep serving in-flight requests, stop
  accepting new keep-alive requests (send `Connection: close`), then exit.
  **`terminationGracePeriodSeconds` counts the preStop time too** — a 30s grace with a 25s sleep
  leaves 5s to drain.
- **Rollout parameters**: Deployment defaults `maxSurge 25%`, `maxUnavailable 25%`,
  `progressDeadlineSeconds 600`, `minReadySeconds 0`, `revisionHistoryLimit 10`. `maxUnavailable: 0`
  plus `maxSurge: 1` is safest but slowest and needs spare capacity. `minReadySeconds` protects against
  pods that pass readiness and crash 20 seconds later. A rollout that exceeds the progress deadline is
  marked failed but **not rolled back automatically**.
- **Sidecars.** An app container that starts before its mesh proxy gets connection refused on its
  first outbound calls; on shutdown the proxy may exit before the app finishes draining. **Native
  sidecars** (init containers with `restartPolicy: Always`, beta and on by default since 1.29) start
  before and stop after the main containers, and stop Jobs hanging forever on a sidecar that never
  exits.
- **Warm-up.** JIT-compiled runtimes are slow for the first seconds; a pod marked ready immediately
  takes a full share of traffic and times out. Warm up before reporting ready, or ramp traffic
  (`slow_start` on some load balancers and meshes).
- **Admission webhooks** with `failurePolicy: Fail` whose backing service is down block **every**
  matching create — including the pods that would fix the webhook. Scope them with
  `namespaceSelector` and exclude `kube-system` and the webhook's own namespace.

### 8. Scheduling, disruption and autoscaling

- **PodDisruptionBudgets** protect against **voluntary** disruptions (drains, upgrades, autoscaler
  scale-down) only. A PDB with `minAvailable` equal to replicas or `maxUnavailable: 0` makes the
  eviction API return **429** forever: node drains hang, cluster upgrades stall, the cluster autoscaler
  cannot remove nodes and cost grows. A single-replica Deployment with any PDB has the same effect.
  PDBs do nothing for involuntary disruption (node crash, OOM).
- **Topology spread.** Without `topologySpreadConstraints` on `topology.kubernetes.io/zone`, the
  scheduler can place most replicas in one AZ, and an AZ loss takes most of your capacity.
  `whenUnsatisfiable: DoNotSchedule` guarantees spread but can leave pods Pending when a zone is out of
  capacity; `ScheduleAnyway` is a preference. Pod anti-affinity per hostname protects against
  node loss.
- **Zonal volumes.** EBS / persistent disks are zonal. A StatefulSet pod whose PVC lives in the lost AZ
  **cannot** be rescheduled elsewhere — it stays Pending by design. `WaitForFirstConsumer` volume
  binding avoids creating a volume in a zone the pod cannot schedule to.
- **HPA mechanics**: sync every **15s**, tolerance **10%**, scale-down stabilisation window **300s**
  by default, target computed as `ceil(current × currentMetric / target)`. Utilisation is measured
  **against requests**, so changing a CPU request changes scaling behaviour. Metrics from
  metrics-server lag by tens of seconds; with pod start-up and image pull, reaction to a spike is
  often **1–3 minutes** — HPA is for trends, not bursts. The `behavior` field sets per-direction
  stabilisation windows and rate policies (e.g. scale up at most 100% per minute, never scale down more
  than 10% per minute); for known events, pre-scale on a schedule (raise `minReplicas`).
- **HPA pitfalls**: JVM start-up CPU spikes cause scale-up that causes more start-ups (flapping); pods
  not yet ready are handled specially but still skew averages; scaling on CPU for an I/O-bound service
  never triggers; scaling on queue depth without considering per-pod throughput overshoots; HPA and
  VPA on the same resource fight. Scaling on **saturation** (in-flight requests, queue age, pool wait)
  via custom or external metrics (KEDA) tracks load far better.
- **Cluster autoscaler** adds nodes only for **Pending** pods and removes underused nodes if their pods
  can move. It will not scale down nodes with pods using local storage, PDB blocks, or the
  `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"` annotation. New nodes take minutes —
  **overprovisioning** with low-priority placeholder pods that get pre-empted buys instant capacity.
- **Priority and pre-emption**: a high-`PriorityClass` pod can evict lower ones to schedule — which
  can also mean a bad batch job with high priority evicts your API.
- **CronJobs**: `concurrencyPolicy` (Allow by default — overlapping runs), `startingDeadlineSeconds`,
  and the controller refuses to start a CronJob that has missed **more than 100** schedules without a
  deadline set. Jobs retry per `backoffLimit` (default 6), so a non-idempotent job may run seven times.

### 9. Kubernetes networking and DNS

- **`ndots:5`.** Pod `resolv.conf` has `search <ns>.svc.cluster.local svc.cluster.local cluster.local
  <node search domains>` and `options ndots:5`. Any name with **fewer than 5 dots** is tried against
  every search domain first. `api.stripe.com` (2 dots) becomes up to 4–5 failed lookups before the real
  one, **each for A and AAAA** — ~10 DNS queries per resolution. At scale CoreDNS CPU spikes and
  resolution latency adds tens of milliseconds to cold connections (`A13`).
- **Fixes**: use a **trailing dot** (`api.stripe.com.`) for external FQDNs; set
  `dnsConfig.options: [{name: ndots, value: "2"}]`; run **NodeLocal DNSCache**; reuse connections so you
  resolve rarely; scale CoreDNS (and its `cache` plugin) with the cluster.
- **The 5-second DNS timeout.** glibc sends A and AAAA queries in parallel from the same socket; a
  conntrack race on UDP insert drops one, and the resolver waits its default **5s** timeout. Symptom:
  a small fraction of requests take **exactly 5s longer**. Fixes: `options single-request-reopen`
  (glibc; **Alpine/musl ignores it**), NodeLocal DNSCache (TCP to upstream, avoids the race).
- **Runtime DNS caching.** The JVM caches successful lookups for 30s by default (forever with a
  security manager in old versions — `networkaddress.cache.ttl`); Node does not cache at all (each
  `http.request` resolves unless the agent keeps the socket); Go's resolver does not cache. Long-lived
  pooled connections ignore DNS changes entirely — a failover that updates DNS does nothing to a
  pool that never reconnects. Set a max connection lifetime.
- **kube-proxy modes.** `iptables` mode evaluates Service rules sequentially and rewrites the whole
  table on changes; with **tens of thousands** of Services/endpoints, rule sync takes seconds to
  minutes and endpoint updates lag (making the termination race in §7 worse). **IPVS** and
  **nftables** (GA in 1.33) scale better; eBPF data planes (Cilium) replace kube-proxy entirely.
- **Services load-balance connections, not requests.** HTTP/2 and gRPC multiplex on one long-lived
  connection, so a ClusterIP Service pins each client to one backend; new pods from a scale-up get no
  traffic. Use client-side load balancing (headless Service + round robin), a mesh, or connection
  max-age.
- **`externalTrafficPolicy: Cluster`** (default) SNATs and may hop nodes (losing client IP, adding
  latency); `Local` preserves client IP but drops traffic to nodes without a local pod unless the LB
  health-checks correctly.
- **Topology-aware routing** keeps traffic in-zone (saving cross-AZ cost and latency) but can
  overload a zone with fewer pods; it disables itself when endpoints are unbalanced.
- **CNI IP exhaustion.** AWS VPC CNI assigns real VPC IPs to pods; a node's max pods is bounded by ENIs
  × IPs per ENI, and a small subnet runs out of IPs — pods stay `ContainerCreating` with "failed to
  assign an IP address". Prefix delegation or larger / secondary CIDRs fix it.

### 10. Stateful workloads and platform services

- **StatefulSets** give stable identity and per-pod PVCs; rolling updates go one pod at a time in
  reverse ordinal order, and a stuck pod blocks the rest. `podManagementPolicy: Parallel` speeds
  start-up but loses ordering guarantees (`O07`).
- **PVC deletion**: deleting a StatefulSet does **not** delete its PVCs by default (good), and
  `reclaimPolicy: Delete` on the StorageClass means deleting a PVC **deletes the cloud disk** (bad, if
  someone "cleans up").
- **Volume attach/detach** on node failure: a volume attached to a dead node may need a force detach
  (the "multi-attach error"), which can take ~6 minutes by default before Kubernetes gives up waiting.
- **Operators** run databases well in the happy path and turn a failure into a two-system debugging
  problem. A managed database is the default answer for a backend team.
- **Leader election** via Lease objects: a pod paused by GC or throttling longer than the lease
  duration loses leadership while still believing it is leader — use fencing tokens for side effects
  (`M33`).
- **etcd limits**: objects up to ~**1.5MiB**, ConfigMaps/Secrets **1MiB**. Huge ConfigMaps, many
  Helm releases (each revision is a Secret; cap with `--history-max`) and chatty controllers slow the
  API server for everyone.

### 11. CI/CD pipelines

- **Build once, promote the artifact.** Rebuilding per environment means production runs a binary that
  was never tested — a floating base tag or dependency range can differ between the staging build and
  the production build. Promote the **same image digest** through environments (`O08`).
- **Lockfiles and reproducibility.** `npm install` may update the lockfile; `npm ci` fails if it
  disagrees. A transitive dependency published at 2am breaks a pipeline that passed at midnight with
  no code change. Pin, vendor or proxy dependencies through an internal registry.
- **Cache poisoning.** CI caches keyed too loosely (branch name only, or no lockfile hash) restore
  stale or wrong dependencies: green builds that fail on a clean runner. Caches writable from pull
  requests of forks, or shared between trusted and untrusted jobs, are a **supply-chain** path — an
  attacker writes a malicious dependency into the cache that the main-branch build then restores
  (`S14`). Key on the lockfile hash, make untrusted jobs read-only, and periodically build without
  cache.
- **Flaky tests.** Causes, roughly in order of frequency: shared state between tests (database rows,
  ports, global singletons), **test ordering** (passes alone, fails in suite), time (`now()` near
  midnight, DST, time zone of the runner), concurrency and sleeps instead of waiting on a condition,
  external network calls, resource starvation on shared runners. Re-running until green hides real
  race conditions that will also happen in production. Quarantine with an owner and a deadline, track
  flake rate per test, and seed/record randomness.
- **Runner exhaustion.** Self-hosted runners fill up with Docker layers and build caches; the pipeline
  fails with "no space left on device" in an unrelated step. Ephemeral runners avoid state leaking
  between jobs (including secrets and credentials).
- **Secrets in CI**: prefer **OIDC federation** (the CI job assumes a cloud role for minutes) over
  long-lived keys stored as variables. Logs mask exact secret values, not base64 or URL-encoded forms.
- **Pipeline as a production system.** When the deploy pipeline is broken during an incident, you
  cannot ship the fix. Keep a documented, tested break-glass path (manual `kubectl set image` or a
  rollback button that does not depend on the broken system).
- **Deploy freezes** reduce change during high-risk periods but concentrate risk: the first deploy
  after a two-week freeze carries two weeks of changes. Keep shipping small changes to non-critical
  paths, or release the backlog in small batches with extra canary time.
- **Monorepo CI**: path filters that skip tests for "unaffected" packages miss changes to shared config,
  lockfiles and code generators.

### 12. Deployment, release and rollback

- **Deploy is not release.** Deploy puts code on servers; release exposes behaviour (flags, §21).
  Canary on a small traffic share with automated analysis of error rate and latency **against the
  baseline pods**, not against a fixed threshold (`O09`).
- **Rollbacks that do not roll back.**
  - **Schema.** Rolling back code after a migration that dropped or renamed a column leaves old code
    against a new schema. The only safe pattern is **expand / contract** (`DB22`): add, dual-write,
    backfill, switch reads, and only drop in a later release once rollback past that point is no
    longer possible. Every migration must be compatible with **N−1** code.
  - **Data.** New code writes a new enum value or a new message format; old code, after rollback,
    crashes reading it from the database or from a queue. Readers must tolerate unknown values
    **before** writers produce them (ship readers first) (`M27`).
  - **Config and infrastructure.** A config change shipped alongside the code (a new env var, a
    changed queue name, an IAM permission removed) is not reverted by redeploying the old image.
  - **Caches.** A new version populates a shared cache with a new serialisation; rolled-back pods fail
    to deserialise. Version cache keys.
  - **Client state.** Mobile apps and browser bundles already downloaded do not roll back.
- **Canary blind spots**: a canary on 1% of traffic will not reveal a bug in a rare tenant path, a
  memory leak that takes 6 hours, a nightly batch job, or a problem that only appears at full load
  (connection count to the database scales with pod count). Bake time matters as much as percentage.
  A canary that shares a queue with the baseline also processes the baseline's messages, so errors
  are attributed to the wrong version.
- **Connection storms on deploy.** Each new pod opens a full connection pool. `maxSurge 25%` on 40
  pods × 20 connections is 200 extra database connections at once; managed Postgres `max_connections`
  is often a few hundred. Use a pooler (PgBouncer, RDS Proxy) and size pools against the **total**
  at maximum surge.
- **Blue/green** doubles capacity and gives an instant switch back — as long as both colours share
  a compatible schema and neither holds long-lived connections that a DNS/LB switch does not move.
- **Rollback speed** is a design goal: keep previous ReplicaSets (`revisionHistoryLimit`), keep the
  previous image in the node cache, make rollback one command, and practise it.
- **Same tag, no rollout.** Re-pushing `:v1.4` and applying the same manifest changes nothing in the
  pod template, so Kubernetes does nothing. `kubectl rollout restart` (it patches an annotation)
  forces new pods; deploying by digest avoids the ambiguity.

### 13. Infrastructure as code (Terraform)

- **State** maps configuration to real resource IDs. Remote state with **locking** (S3 + DynamoDB,
  or S3 native locking via `use_lockfile` in Terraform 1.10+, or Terraform Cloud). A crashed apply
  leaves a stale lock: `terraform force-unlock <id>` only after confirming nothing is running (`O10`).
- **Drift**: someone changes a resource in the console. `terraform plan` shows it will revert; a
  `-refresh-only` plan shows drift without proposing changes. Decide per case whether to codify the
  manual fix or revert it. `lifecycle { ignore_changes = [...] }` is for fields legitimately managed
  elsewhere (e.g. `desired_count` managed by an autoscaler).
- **Blast radius.** One state file for the whole company means every plan touches everything, takes
  minutes, and one bad change can destroy unrelated resources. Split state by environment and
  component — the smallest unit you would want to apply independently.
- **Read the plan for replacements.** `-/+` or "forces replacement" on a database, a load balancer or
  a subnet is an outage. Common triggers: changing an immutable attribute (name, AZ, engine), a
  provider upgrade that changes defaults, and **`count` index shifts** — removing item 0 from a list
  used with `count` renumbers every later resource, destroying and recreating them. Use `for_each`
  with stable keys.
- **Refactoring safely**: `moved` blocks (1.1+) rename in state without destroying; `import` blocks
  (1.5+) bring existing resources under management with a plan preview; `removed` blocks (1.7+) stop
  managing without destroying. `terraform state rm` / `mv` are the manual equivalents.
- **Guard rails**: `prevent_destroy` on stateful resources, `create_before_destroy` for things that
  must not have a gap, deletion protection on the cloud resource itself (it survives even if Terraform
  is wrong), policy-as-code on plans (OPA/Sentinel), and plan output reviewed in the pull request.
  Apply the **saved plan file** that was reviewed, not a fresh plan computed later.
- **Secrets in state.** State stores resource attributes in plain text, including generated passwords.
  Encrypt the backend and restrict who can read it.
- **Provider and module pinning.** An unpinned provider upgraded under you changes behaviour on the
  next plan. Pin versions and commit `.terraform.lock.hcl`.
- **Two owners of one resource.** A resource managed by both Terraform and a Kubernetes controller (or
  by two state files) flip-flops on every apply. Each field needs exactly one owner.

### 14. Cloud primitives and their limits

- **Availability zones.** A region has several AZs with independent power and networking. Real AZ
  events are often **partial** (a subset of racks, elevated latency, one service's control plane) —
  health checks pass while requests fail. **Zonal shift / evacuation** (moving traffic away from an AZ
  on purpose) is a better tool than waiting for health checks. Capacity: after losing one of three
  AZs you need 150% of normal per-AZ capacity in each remaining AZ, and everybody else in the region is
  launching instances in the same two AZs at the same moment (`O11`).
- **Cross-AZ traffic** costs roughly **$0.01/GB in each direction** on AWS. Chatty services, Kafka
  replication and a database in another AZ are typical surprise bills. Kafka consumers read from the
  partition leader in any zone unless **rack-aware follower fetching** (`client.rack` plus
  `replica.selector.class`) lets them read from an in-zone replica.
- **One NAT gateway per AZ.** A single NAT gateway lives in one AZ; if every private subnet routes
  through it, losing that AZ cuts egress for all zones. Use one per AZ with per-AZ route tables.
- **NAT gateways.** Charged per hour **and per GB processed** (about $0.045/GB in `us-east-1`),
  so pulling images or talking to S3 through NAT is expensive — use **VPC gateway endpoints** for S3
  and DynamoDB (free) and interface endpoints for other services. Each NAT gateway IP supports about
  **55,000 simultaneous connections to a single destination** (IP, port, protocol);
  `ErrorPortAllocation` in CloudWatch means you hit it (add secondary IPs or spread destinations).
  Idle timeout is **350s**: connections idle longer are silently dropped and the next write gets a
  RST or hangs — set TCP keep-alive or pool max-idle below 350s.
- **Instance network allowances.** EC2 instances have per-instance limits on bandwidth, **packets per
  second**, connection tracking entries and link-local traffic (DNS, IMDS, NTP). Exceeding them drops
  packets silently; the only evidence is `ethtool -S eth0` counters `bw_in_allowance_exceeded`,
  `pps_allowance_exceeded`, `conntrack_allowance_exceeded`, `linklocal_allowance_exceeded`. DNS to the
  VPC resolver is capped at **1,024 packets per second per ENI** — a node doing uncached DNS for many
  pods hits it (see §9).
- **Burstable instances and volumes.** T-family instances earn CPU credits; when credits run out the
  instance drops to its **baseline** (a fraction of a vCPU) — a service that was fine for weeks
  suddenly gets slow at a random time. `unlimited` mode avoids that for a cost. **gp2** EBS volumes
  burst to 3,000 IOPS on credits with a baseline of 3 IOPS/GB (a 100GB volume = 300 IOPS once credits
  are gone); **gp3** gives a flat 3,000 IOPS / 125 MB/s baseline independent of size. CloudWatch
  `CPUCreditBalance` and `BurstBalance` show it.
- **Noisy neighbours.** On shared hosts, **CPU steal** (`%st` in `top`/`vmstat`) is time your vCPU
  wanted but the hypervisor gave elsewhere. On a Kubernetes node, the neighbours are other pods:
  one pod saturating disk I/O, memory bandwidth or the node's conntrack table hurts every pod.
- **S3.** Strongly consistent for read-after-write, overwrite and list **since December 2020** — old
  "eventual consistency" workarounds are unnecessary. Throughput scales to at least **3,500
  PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second per partitioned prefix**; bursts above
  that on a fresh prefix get **503 SlowDown** while S3 repartitions. Spread keys across prefixes,
  retry with backoff. Per-request pricing makes millions of tiny objects costly; with versioning on,
  "deleted" objects keep their storage bill until a lifecycle rule expires non-current versions.
- **API throttling.** Cloud control-plane APIs are rate-limited per account and region (token bucket).
  A controller, Terraform run or Lambda fleet calling `Describe*` in a loop throttles **every** caller
  in the account, including the autoscaler. STS `AssumeRole` and KMS `Decrypt` also have quotas;
  decrypting a secret per request rather than caching it hits KMS limits at scale (decrypt once and
  cache, or envelope encryption with a cached data key). Always use the SDK's
  retry with exponential backoff and **jitter**.
- **Instance metadata (IMDS).** IMDSv2 requires a session token with a **hop limit** (default 1 on
  many AMIs). A container behind a bridge network adds a hop, so the SDK's token request times out and
  credential lookup falls back through the provider chain — slow start-up (seconds per attempt) or
  "no credentials". Set hop limit 2 or, better, use IRSA / EKS Pod Identity so pods do not use the node
  role at all.
- **Quotas** (vCPUs per instance family, ENIs, EIPs, load balancers, Lambda concurrency) are silent
  until you hit them, usually during a scale-up or a failover. Track them as capacity.
- **Load balancers** scale internally; a sudden 10× spike to a cold load balancer may see errors while
  it scales. Deregistration delay (default **300s** on ALB target groups) makes deploys slow if left at
  default, and must be at least as long as your longest request.

### 15. Serverless

- **Cold starts**: runtime init + your module init; worse with large bundles, JVM without SnapStart,
  and heavy SDK clients created per invocation. Provisioned concurrency trades cost for
  predictability (`O12`).
- **Concurrency** is per region with a default account quota of **1,000**; one function scaling
  without **reserved concurrency** starves every other function in the account. Each concurrent
  execution opens its **own** database connections — 1,000 Lambdas × 1 connection exhausts
  `max_connections`. Use RDS Proxy or a data API.
- **Freeze and thaw.** After the handler returns, the execution environment is **frozen**: background
  timers, unflushed log/metric batches and half-sent requests pause and resume on the next invocation
  (possibly minutes later, possibly never). Connections held across invocations may be dead on thaw
  (the NAT/LB idle timeout passed while frozen). Flush telemetry before returning.
- **Timeouts chain**: API Gateway's integration timeout (29s by default) < Lambda timeout (up to 15
  min). **SQS visibility timeout must exceed the function timeout** (AWS recommends 6×), or messages
  are redelivered while still being processed. Batch failures redeliver the whole batch unless you
  report partial failures. Work longer than the gateway timeout must become **asynchronous**: accept,
  return a job ID with 202, and let the client poll or receive a callback.
- **Recursive invocation**: a function triggered by S3 writes that writes back to the same bucket
  loops and scales to your concurrency limit. AWS now detects some loops, not all.

### 16. Metrics: types, cardinality and the ways they lie

- **Counters** are queried with `rate()`/`increase()`, which handle counter resets from restarts.
  `increase()` extrapolates to the window edges, so it returns non-integers ("1.33 errors") — expected,
  not a bug. **The `rate()` window should be at least 4× the scrape interval**; `rate(x[30s])` with a
  15s scrape often has fewer than two samples and returns nothing, so graphs have holes (`O14`).
- **Histograms vs summaries.** Histogram buckets can be summed across pods and then turned into a
  quantile: `histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`.
  **Summaries cannot be aggregated** — averaging per-pod p99s is not a p99. Histogram quantiles are
  **linear interpolations inside a bucket**: with buckets at 0.5s and 1s, a "p99 of 0.98s" only means
  "somewhere between 0.5s and 1s". The default client buckets (5ms…10s) rarely bracket your SLO; put a
  boundary **at** the SLO threshold. If most requests exceed the highest finite bucket, the quantile
  is clamped to that bucket's bound — "p99 is exactly 10s" means "off the scale". Native (exponential)
  histograms fix much of this.
- **Averages hide.** A mean latency of 50ms can be 99% at 10ms and 1% at 4s. Look at percentiles and,
  better, the **fraction of requests over the threshold** (`le` bucket ratio) — that is what an SLO
  counts.
- **Cardinality explosion.** Series = product of distinct label values. A label with user IDs, full
  URL paths, pod names from constantly restarting pods, or error messages multiplies series. Symptoms:
  Prometheus memory and WAL replay time balloon (it OOMs, restarts, and replays for minutes), queries
  time out, vendor bills jump (custom-metric pricing is per series). Find it with
  `topk(10, count by (__name__) ({__name__=~".+"}))` or the TSDB status page; drop with
  `metric_relabel_configs` at scrape time or a per-target `sample_limit`; move per-entity questions to
  logs and traces (`O13`).
- **Churn cardinality.** Every pod restart or deployment creates new series (the `pod` label changes).
  A crash-looping Deployment or a CronJob every minute churns millions of short-lived series.
- **Staleness and gaps.** Prometheus marks a series stale ~**5 minutes** after it disappears. A
  crashed pod's last value lingers on dashboards; a pod that stops being scraped silently vanishes —
  `sum()` just gets smaller, no alert fires. Monitor `up == 0` and scrape counts.
- **Batch jobs** that finish before a scrape are invisible to pull-based metrics. The Pushgateway
  keeps the last pushed value **forever** — a job that stopped running still looks healthy. Push a
  `last_success_timestamp` and alert on its age.
- **Survivorship bias in metrics.** Latency measured in the application excludes requests that never
  reached it (rejected at the LB, queued in the accept backlog, timed out in the client). Measure at
  the load balancer and the client too. A request that times out after 30s may be recorded by the
  server as a 200 in 31s — or not at all if the pod was killed.
- **Coordinated omission.** Load tools and some clients that wait for a response before sending the
  next request under-report latency during stalls, because they stop sending while the server is
  stuck. Constant-rate generators (wrk2, k6 arrival-rate executors) avoid it (`C14`).
- **The metrics endpoint is code on the request path.** A `/metrics` handler that computes values on
  scrape (querying the database, walking a large cache) produces latency spikes at exactly the scrape
  interval; scrapes that exceed `scrape_timeout` also show the target as down.
- **Load tests lie in other ways too**: a few hot keys served from cache, reused connections with no
  TLS or DNS cost, and one client IP flatter the result; model the real key distribution and
  connection churn (`C15`).
- **Exemplars** attach a trace ID to a histogram bucket sample, linking "p99 spiked" directly to a slow
  trace.

### 17. Logs and traces in production

- **Head-based sampling** decides at the root span (e.g. keep 1%). It is cheap and consistent across
  services, but **rare errors and slow outliers are mostly dropped** — exactly the traces you want.
  **Tail-based sampling** decides after the trace completes (keep all errors, all over 1s, 1% of the
  rest) in the OpenTelemetry Collector; it requires **all spans of a trace to reach the same collector
  instance** (a load-balancing exporter keyed on trace ID) and buffers traces in memory for a decision
  wait (e.g. 10–30s), so long async traces get split and a collector restart drops what it buffered
  (`O15`).
- **Sampling bias** also affects metrics derived from traces: if you only keep errors, span-derived
  error rates are wrong. Generate RED metrics **before** sampling (span-metrics connector) or from the
  application.
- **Broken propagation** shows as many short root traces instead of one long one: message queues,
  thread pools, async boundaries, and proxies that strip unknown headers drop the `traceparent`. A mix
  of propagation formats (B3 vs W3C) splits traces at the boundary.
- **Clock skew.** Span and log timestamps come from each host's clock. A few hundred milliseconds of
  skew makes a child span start before its parent and reorders logs across services during exactly the
  incident you are reconstructing. Use NTP/chrony (`chronyc tracking`), trust durations measured on one
  host, and order cross-service events by causal IDs (trace/span IDs, sequence numbers) rather than
  wall time. Skew also breaks JWT `nbf`/`exp` checks, TLS validity and signed URLs. A **clock step**
  (NTP correcting a large offset at once) makes wall-clock-based durations negative or huge — measure
  durations with a monotonic clock.
- **Logs lost at the worst moment.** An OOMKilled or `SIGKILL`ed process loses whatever its logger
  buffered in memory — the last lines, which explain the crash, are never written. Async loggers with
  bounded queues **drop** under load (or block the request thread if configured to block). Write to
  stdout unbuffered or flush on fatal signals, and alert on dropped-log counters.
- **stdout can block.** If the node's log pipeline stalls (disk full, a slow journald), writes to stdout
  block and a synchronous logger freezes the application — a logging outage becomes an application
  outage.
- **Multi-line logs** (stack traces) split into one event per line by the shipper, so searching for the
  exception finds the first line without the cause. Log structured JSON with the stack trace in one
  field.
- **Log levels as a runtime switch**: being able to raise one pod or one tenant to debug for ten
  minutes without a deploy is worth more than always-on verbosity.
- **Structured logs** with a trace ID and request ID in every line are what make "from alert to the
  exact error" a two-click path. High-cardinality fields belong here, not in metric labels.

### 18. Alerting and on-call

- **Symptom-based alerting**: page on what users feel (error rate, latency against SLO, freshness of a
  pipeline, queue **age**), not on causes (CPU 90%, one pod restarted, disk 80% on a node that
  autoscaling replaces). Cause-based signals are for dashboards and tickets (`O16`, `M34`).
- **Burn-rate alerts** (SRE workbook) for a 99.9% / 30-day SLO: page when **2% of the budget burns in
  1 hour** (burn rate **14.4**, with a 5-minute short window) or **5% in 6 hours** (burn rate **6**,
  30-minute short window); open a ticket for **10% in 3 days** (burn rate **1**, 6-hour short window).
  Both windows must exceed the threshold: the long window stops a 2-minute blip paging, the short
  window makes the alert **reset quickly** after recovery. Burn rate = observed error ratio / (1 − SLO).
- **Low traffic breaks ratios.** At 10 requests per minute, one failure is a 10% error rate. Use a
  minimum request count condition, longer windows, or synthetic traffic to keep the denominator
  meaningful.
- **Missing data is not "OK".** An alert on `error_rate > 0.05` does not fire when the metric stops
  existing (the service is down, the exporter is broken, a label was renamed in a deploy). A ratio
  with zero requests is `NaN`, which never satisfies `>`. Pair with `absent()` / `absent_over_time()`
  or alert on `up == 0`, and run a **dead-man's switch** (an always-firing Watchdog alert sent to an
  external service that pages if it **stops** arriving) to catch a broken alerting pipeline.
- **`for:` duration** avoids flapping but delays detection; an alert with `for: 10m` on a condition
  that clears for one evaluation every few minutes **never fires**, because the pending timer resets.
  Alert on a windowed value (`max_over_time`, `avg_over_time`) or on the age of the oldest item.
- **Predictive alerts** for things that fill: `predict_linear(node_filesystem_avail_bytes[6h], 4*3600)
  < 0` pages when a stateful disk will be full in four hours, not at an arbitrary 80%.
- **Synthetic checks** must exercise a real user journey through the same edge users hit; a
  synthetic that calls `/health` proves only that the process is up.
- **Error-budget policy**: an agreed rule that when the budget is spent, feature releases slow and
  reliability work takes priority. Without it, an SLO changes no decisions.
- **Alert fatigue**: every page must be actionable, urgent and user-impacting. Measure pages per shift
  and the share that needed action; delete or demote the rest. Group and **inhibit** (a region-down
  alert suppresses the 300 per-service alerts under it) in Alertmanager.
- **Runbooks** linked from each alert: what it means, how to confirm, how to mitigate (roll back,
  shift traffic, scale, disable a flag), who to escalate to.
- **Incident roles**: incident commander (coordinates, does not debug), operations lead, communications.
  **Mitigate first, root-cause later** — rolling back the last change is correct even before you know
  it was the cause. Blameless post-incident reviews focus on why the system allowed the failure.
- **Alerts that depend on the thing they monitor**: alerting hosted in the same cluster, region or
  account as the service goes down with it. External synthetic checks and the dead-man's switch cover
  that.

### 19. Debugging production

- **Start with change.** Most incidents follow a change: deploy, config, flag, dependency upgrade,
  certificate expiry, traffic shape, a cron that started today. Overlay deploy and flag-change markers
  on dashboards; check `kubectl rollout history`, the flag audit log and the infra change log (`O17`).
- **Narrow by dimension.** Is it all pods or one? One AZ, one node, one tenant, one endpoint, one
  client version? A problem on **one node** is infrastructure (noisy neighbour, conntrack, disk); on
  **one version** is code; on **one tenant** is data. Compare a good pod with a bad pod.
- **USE for resources** (utilisation, saturation, errors) and **RED for services** (rate, errors,
  duration). Walk the request path and apply USE to each resource on it — including the invisible ones:
  connection pools, thread pools, file descriptors, conntrack, ephemeral ports, API quotas.
- **The first 60 seconds on a host** (Brendan Gregg): `uptime`, `dmesg -T | tail`, `vmstat 1`,
  `mpstat -P ALL 1`, `pidstat 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `sar -n TCP,ETCP 1`, `top`.
- **eBPF tools** (bcc / bpftrace), usable on the node without restarting anything: `execsnoop`
  (short-lived processes — a health check forking a shell every second), `opensnoop` (files opened —
  which config did it really read), `tcpconnect` / `tcpretrans` / `tcplife` (connections and
  retransmits per process), `biolatency` (disk latency distribution), `runqlat` (CPU scheduler queue
  latency — proof of CPU saturation when utilisation looks fine), `offcputime` (where threads block),
  `oomkill`, `profile` (CPU flame graphs across all processes).
- **`perf` and JITs**: `perf` sees JIT frames as hex unless the runtime writes a symbol map — Node
  `--perf-basic-prof`, JVM via `async-profiler` (preferred, also avoids safepoint bias). Sampling
  profilers at ~99Hz cost around 1% CPU and are safe in production.
- **`strace -f -p <pid> -T -e trace=network,file`** shows system calls and their durations; it slows
  the process heavily (ptrace), so use briefly or prefer eBPF equivalents.
- **Thread and heap dumps**: `jcmd <pid> Thread.print` three times, 5 seconds apart, to see what
  threads are stuck on; heap dumps pause the JVM and can be as large as the heap — take them from a pod
  removed from traffic, and make sure the dump fits on the pod's disk. Node: `--heapsnapshot-signal`,
  `--cpu-prof`.
- **Take one pod out of rotation** (change a label so it no longer matches the Service selector) to
  keep a misbehaving instance alive for debugging while the ReplicaSet creates a replacement.
- **Retry amplification**: when a dependency slows, every layer's retries multiply load (3 retries at
  3 layers = up to 64× attempts). Retry budgets and circuit breakers (`M09`, `M10`).
- **Metastable failures**: a system that stays broken after the trigger is gone because its recovery
  work (retries, cache refill, reconnect storms, queue backlog) keeps it overloaded. Recovery needs
  **load shedding**, not waiting: cut traffic, let it recover, then ramp.

### 20. Cost and efficiency

- **Where the money usually goes**: over-requested pods (requests set to peak and never revised — the
  gap between requests and actual usage is the waste), idle non-production environments, NAT gateway
  and cross-AZ transfer (§14), log ingestion, metric cardinality (§16), unattached volumes and old
  snapshots, over-provisioned databases (`O18`).
- **Kubernetes bin-packing**: the cluster pays for nodes, not usage. Cost per service = its share of
  requested CPU/memory. Right-size requests from p95 usage over weeks (VPA in recommendation mode),
  use **spot** for stateless and batch with graceful handling of the **2-minute** interruption notice.
  DaemonSets are paid on every node — ten agents at 200m CPU each is two cores per node.
- **Log cost**: CloudWatch Logs ingestion is about **$0.50/GB** — one debug line per request at 5k
  RPS is terabytes per month. Sample successful-path logs, keep all errors, drop health-check access
  logs, tier retention.
- **Data transfer**: egress to the internet is the most expensive; cross-region replication and
  cross-AZ chatter next. Moving a chatty service next to its database can save more than any compute
  optimisation.
- **Unit economics**: cost per request, per tenant, per order — so growth and waste can be told apart.
- **Commitments**: savings plans / reserved instances for the steady baseline, on-demand for bursts,
  spot for interruptible — and right-size before committing, or you lock in the waste.

### 21. Runtime configuration and feature flags

- **Config changes are deployments.** Many of the largest public outages were a global config push,
  not code. Roll config out progressively (one cell, one region, a percentage) with the same canary
  and rollback discipline as code, and validate it **before** it reaches the fleet — a config that
  parses but has a semantic error (an empty allow-list, a regex that matches everything, a zero
  timeout, a file twice the expected size) is the dangerous kind (`O19`).
- **Flag system as a dependency.** If the flag SDK fetches per request, a flag service outage becomes
  your outage. SDKs should evaluate **locally** from a cached ruleset streamed in the background, with
  sensible **coded defaults** when nothing has loaded. The default must be the **safe** value, which
  is not always "off" — a flag that gates an already-launched feature defaulting to off turns the
  feature off for everyone on SDK start-up failure.
- **Kill switches** for expensive or risky features (recommendations, a new payment provider,
  non-critical background jobs) are the fastest mitigation available — faster than rollback — but only
  if they are tested.
- **Flag interactions and debt.** Two flags on together produce a path nobody tested; old flags left
  at 100% hide dead code that a later change "reactivates". Give flags owners and expiry dates.
- **Percentage rollout pitfalls**: bucketing must be **sticky** per user (hash of user ID), or a user
  flips between versions per request; rolling a flag to 50% in a system with shared caches may
  populate caches with both variants.
- **Hot-reload risk**: a configuration watcher that reloads on file change will also reload a
  half-written file; write atomically (write a temp file, rename). Kubernetes mounted ConfigMaps update
  by an atomic symlink swap, so watchers must watch the directory, not the file inode.
- **Audit and diff**: every config and flag change needs who, what, when and a one-click revert; it is
  the first thing checked in an incident.

### 22. Disaster recovery and region failover

- **RPO** = how much data you can lose (time since last good copy); **RTO** = how long until service is
  back. Asynchronous cross-region replication gives an RPO equal to replication lag **at the moment of
  failure** — which is highest during the heavy-load incidents that cause failovers (`O20`, `DB36`).
- **An untested backup is not a backup.** Common failures on restore: backups encrypted with a KMS key
  that was deleted or lives in the failed region/account; logical dumps that take 10× longer to
  restore than expected (a 2TB restore plus index build is many hours); backups in the same account as
  production (a compromised admin or ransomware deletes both — use a separate account and object
  lock); only the database is backed up but not secrets, config, or object storage; snapshot restores
  of cloud volumes that are **lazily loaded** from object storage, so the restored database is very
  slow until every block has been read once. Schedule **restore tests** with a timed RTO.
- **Point-in-time recovery** needs the base backup **and** the continuous log (WAL/binlog) with no
  gaps; retention windows (often 7 days by default) mean corruption noticed on day 8 is unrecoverable.
  Replication faithfully replicates a bad `DELETE` — replicas are not backups.
- **Region failover gotchas**:
  - **Capacity and quotas** in the standby region are sized for idle, not for full traffic; everyone
    else fails over to the same region at the same time.
  - **Hidden dependencies on the failed region**: the CI/CD system, container registry, secrets
    manager, identity provider, DNS control plane, or monitoring all running in the region that is
    down. The global control plane of some cloud services lives in one region.
  - **DNS TTLs and client caching**: a 60s TTL does not help clients that cache longer (runtime
    settings, pooled connections that never re-resolve, §9).
  - **Split brain**: both regions accept writes after a partition unless one is fenced. Failback is
    harder than failover — data written in the secondary must be reconciled.
  - **Sequencing**: promote the database first, then point apps at it; apps that fail over first talk
    to a read-only replica and error.
- **Static stability**: design so that the data plane keeps working when the control plane is down
  (existing instances keep serving if the autoscaling API is unavailable; pre-provisioned capacity
  rather than launching during an event).
- **Game days** and **chaos experiments** (kill a pod, a node, an AZ, block a dependency) in
  production-like environments prove the runbook and reveal what monitoring misses. Start small and
  with a stop button.
- **Cells and blast radius**: partitioning customers into independent cells limits any failure (bad
  deploy, poison tenant, config) to one cell; progressive rollouts go cell by cell.
- **Certificates and secrets expire on a calendar**, not on a deploy: an expired intermediate
  certificate, a root CA removed from an old client trust store, or a rotated signing key breaks
  services that have not changed in months. Monitor expiry dates as alerts with weeks of notice.

---

## Questions

### Level 1 — Pod crashes, restarts and OOM

The questions nearly every interviewer asks: a pod dies or restarts and the logs are no help.

1. "Our Node API pods restart roughly every six hours. The application logs just stop mid-line — no
   error, no stack trace. `kubectl get pods` shows a restart count climbing. Where do you look?"
   > **Direction:** No log line means the kernel did it: check `kubectl describe pod` for
   > `OOMKilled`/exit 137 and `dmesg` on the node, then compare RSS with heap, since buffers and native
   > memory live outside the V8 heap (§3, §17; `O01`, `O04`).

2. "A Java service has `-Xmx2g` and a 2Gi memory limit. It gets OOMKilled under load even though GC
   logs show the heap never exceeds 1.6GB. The team wants to raise the limit to 4Gi. What do you say?"
   > **Direction:** Heap is only part of RSS — metaspace, thread stacks, direct buffers and code cache
   > sit on top — so size the heap as a percentage of the limit (`MaxRAMPercentage` around 70) rather
   > than doubling the limit (§3; `O04`, `C12`).

3. "A pod shows memory usage at 98% of its limit on the dashboard all day, yet it is never OOMKilled
   and performance is fine. Someone opens a P2 for a memory leak. Is it one?"
   > **Direction:** Probably page cache charged to the cgroup; check `memory.stat` (`anon` vs `file`)
   > and alert on working set and RSS, not `container_memory_usage_bytes` (§4; `O04`, `O14`).

4. "After we containerised a legacy service, every deploy takes exactly 30 seconds per pod and the
   in-flight jobs are lost. Locally, Ctrl-C shuts it down cleanly. What's going on?"
   > **Direction:** The process is PID 1 (or sits behind `sh -c` from a shell-form `CMD`) and never
   > receives or handles `SIGTERM`, so the kubelet waits the full grace period and sends `SIGKILL`;
   > use the exec form and tini (§1; `O01`, `O03`).

5. "A PDF-rendering service that shells out to headless Chrome runs fine for days, then every request
   fails with 'Resource temporarily unavailable' when spawning a process. Memory and CPU are fine.
   What would you check?"
   > **Direction:** Zombie processes piling up because the app is PID 1 and never reaps children,
   > until `pids.max` is hit; `ps` shows `<defunct>`, fixed by tini or `dumb-init` (§1; `O01`, `O03`).

6. "A pod is in `CrashLoopBackOff`. `kubectl logs` shows only the start-up banner of the current
   attempt, which then gets killed. How do you find out why it died?"
   > **Direction:** `kubectl logs --previous` and the `Last State` exit code (137, 143, 139, 1) tell
   > you whether the kernel, the kubelet or the app ended it — and a liveness probe killing a slow
   > starter is a common answer, fixed with a startup probe (§1, §7; `O05`, `O06`).

7. "Several unrelated pods on the same node were all terminated within a minute with status `Evicted`.
   None of them hit their own limits. What happened, and how do you stop it recurring?"
   > **Direction:** Node pressure eviction — usually `DiskPressure` from one pod's logs or writable
   > layer, or memory from pods far above their requests; set ephemeral-storage limits and realistic
   > requests (§5, §6; `O04`, `O06`).

8. "A service processing uploads buffers each file into `/tmp` before streaming it to S3. Pods with
   a 1Gi limit get OOMKilled on large uploads, though RSS stays around 300MB. Why?"
   > **Direction:** `/tmp` is on a memory-backed `emptyDir` or tmpfs, which is charged to the
   > container and cannot be reclaimed; move it to a disk-backed volume or stream without buffering
   > (§4, §6; `O04`).

9. "A Go service's memory climbs to the 512Mi limit and it gets OOMKilled every few hours, but pprof
   heap profiles show only 150MB live. What's your theory and fix?"
   > **Direction:** The Go GC lets the heap grow to twice the live size before collecting and knows
   > nothing about the cgroup limit; set `GOMEMLIMIT` below the limit (§3; `O04`, `C12`).

10. "A Python worker gets OOMKilled on a 2-CPU, 4Gi pod. Profiling shows the Python heap is stable,
    but RSS grows steadily with thread count. Nothing in the code looks leaky. Where does the memory go?"
    > **Direction:** glibc malloc arenas fragmenting per thread; `MALLOC_ARENA_MAX=2` or jemalloc
    > often cuts RSS by a third or more with no code change (§3; `O01`, `C12`).

11. "A Burstable pod with a 128Mi memory request and 2Gi limit keeps getting killed on a busy node
    even though it never reaches 2Gi. The Guaranteed pods next to it survive. Why?"
    > **Direction:** Under node pressure, eviction ranks pods by usage above request, and the kernel's
    > `oom_score_adj` for a tiny request is near BestEffort's 1000; set request equal to limit for
    > anything important (§3, §6; `O04`, `O06`).

12. "The whole node went `NotReady` and all 40 pods on it were rescheduled at once, causing a
    thundering-herd restart elsewhere. The node's own logs show the kubelet was OOMKilled. How do you
    prevent that?"
    > **Direction:** Pods overcommitted the node because `kube-reserved`/`system-reserved` were too
    > small, so the kernel killed the kubelet; reserve memory for system daemons and keep the sum of
    > limits near allocatable (§6; `O04`, `O06`).

13. "A pod in `ContainerCreating` for ten minutes with events saying 'failed to assign an IP address
    to container'. The node has plenty of CPU and memory. What's the constraint?"
    > **Direction:** The VPC CNI has run out of IPs for the node's ENIs or the subnet itself is
    > exhausted; fix with prefix delegation or bigger / secondary subnets, not more nodes (§9; `O05`,
    > `O11`).

14. "An image works on the developer's machine and in staging, but in production pods fail with
    `exec format error`. The Dockerfile hasn't changed. What changed?"
    > **Direction:** An architecture mismatch — the image was built for arm64 on a laptop and pushed
    > under the same tag to amd64 nodes (or vice versa); build multi-arch in CI and deploy by digest
    > (§5, §11; `O03`, `O08`).

15. "Pods stay in `ImagePullBackOff` during a scale-up, even for an image that ran on those nodes an
    hour ago. The registry is Docker Hub. Other clusters in the company are fine. What happened?"
    > **Direction:** Anonymous pull rate limiting per source IP — the whole cluster pulls through one
    > NAT IP — made worse by `imagePullPolicy: Always`; mirror images and use `IfNotPresent` with
    > digests (§5, §14; `O03`).

16. "A service that has been stable for a year starts OOMing only on the first of each month. There is
    no deploy in that window. What do you suspect and how do you prove it?"
    > **Direction:** A calendar-triggered workload — a monthly report or invoice batch loading a whole
    > dataset into memory — so correlate with CronJob schedules and request patterns, and capture a heap
    > profile from a pod taken out of rotation (§6, §19; `O06`, `O17`).

17. "Your distroless container is misbehaving in production. There is no shell, no `curl`, no `ps`.
    `kubectl exec` fails. How do you investigate without rebuilding the image?"
    > **Direction:** `kubectl debug` with an ephemeral container targeting the app container shares its
    > process namespace, and `/proc/<pid>/root` exposes its filesystem; or `nsenter` from the node
    > (§5; `O03`, `O17`).

18. "A node's disk is 100% full and pods are being evicted, but `du` on the node accounts for only 40%
    of the disk. What's using the rest?"
    > **Direction:** Files deleted while still held open (a rotated log a process kept writing to);
    > `lsof +L1` finds them, and restarting the process or truncating through `/proc/<pid>/fd` frees
    > the space (§2; `O01`).

19. "A Node service's memory graph climbs in a sawtooth for a week, then it OOMs. Restarting 'fixes'
    it, so the team added a nightly restart CronJob. What's wrong with that, and what would you do
    instead?"
    > **Direction:** It hides a real leak (often a growing cache, listeners, or unclosed handles) and
    > turns it into a scheduled outage risk; take heap snapshots hours apart from one pod and diff them
    > (§3, §19; `O17`, `C12`).

20. "A sidecar-injected Job runs to completion — the app container exits 0 — but the Job never shows
    as complete and the pod stays Running forever. Why?"
    > **Direction:** The mesh sidecar never exits, so the pod never completes; native sidecars (init
    > containers with `restartPolicy: Always`) or telling the proxy to quit on app exit fixes it (§7;
    > `O06`).

### Level 2 — Deploys, rollouts and graceful shutdown

The second most common topic: every deploy causes a blip, and the candidate must know why.

1. "Every deploy produces a burst of 502s at the ALB for about three seconds per pod. The app has a
   proper `SIGTERM` handler that stops the server gracefully. What's missing?"
   > **Direction:** Endpoint removal propagates asynchronously while `SIGTERM` is delivered at once, so
   > the pod stops listening before routing stops sending; a `preStop` sleep of a few seconds closes
   > the race (§7; `O06`, `O09`).

2. "Sporadic 502s at the ALB — about 0.05% of requests, not correlated with deploys — with no errors
   in the Node application logs. Where would you look?"
   > **Direction:** Node's 5s `keepAliveTimeout` is shorter than the ALB's 60s idle timeout, so the
   > app closes a connection the LB is about to reuse; make the server's idle timeout longer than the
   > LB's (§2; `O02`, `A12`).

3. "You added a 20-second `preStop` sleep to fix deploy errors. Now some long requests are cut off
   during deploys even though the app drains gracefully. What did you break?"
   > **Direction:** `terminationGracePeriodSeconds` includes the preStop time, so 30s minus 20s leaves
   > only 10s to drain; raise the grace period accordingly (§7; `O06`).

4. "A liveness probe hits `/health`, which checks Postgres. During a 90-second database failover, all
   60 API pods restarted, and the outage lasted 12 minutes instead of 90 seconds. Explain and fix."
   > **Direction:** Liveness tied to a shared dependency restarts everything at once and the cold
   > restarts stampede the recovering database; liveness should only check the process itself (§7,
   > §19; `O06`, `M10`).

5. "A rollout of a JVM service shows every new pod becoming Ready, taking traffic, and then p99 latency
   spiking to 4 seconds for about a minute per pod. Old pods were fine. What's happening?"
   > **Direction:** Cold JIT and empty caches: the pod is Ready before it is warm and takes a full
   > share at once; warm up before reporting ready, or ramp traffic with slow start (§7; `O06`, `O09`).

6. "You rolled back a bad release in 30 seconds, but the errors continued for another hour. The
   release included a migration renaming a column. What happened, and how should it have been done?"
   > **Direction:** Code rolled back, schema did not; the old code queries a column that no longer
   > exists — renames need expand/contract across several releases so every migration works with N−1
   > code (§12; `O09`, `DB22`).

7. "A new release added a `status` value `PARTIALLY_REFUNDED`. It had a bug and was rolled back. Now the
   old version crashes on some orders and some Kafka messages. Why, and what would you change in the
   process?"
   > **Direction:** Data written by the new version survives the rollback, and old readers cannot parse
   > it; readers must tolerate unknown values before any writer produces them (§12; `O09`, `M27`).

8. "Your team re-pushed a fixed image to the same tag `v2.3.1` and ran `kubectl apply`. Nothing
   changed; the bug is still live. Then half the pods got the fix after a node replacement. What's
   going on?"
   > **Direction:** An unchanged pod template triggers no rollout, and the mutable tag means pods run
   > whatever the node pulled; deploy by digest and use `rollout restart` if you must (§5, §12; `O03`,
   > `O09`).

9. "During a rollout of a 40-pod service, the database starts rejecting connections with 'too many
   clients'. Nothing about query volume changed. Why during the deploy?"
   > **Direction:** Surge pods each open a full pool while old pods still hold theirs, so connection
   > count peaks well above steady state; size pools for maximum surge or put a pooler in front (§12;
   > `O09`, `DB39`).

10. "A deployment reports `ProgressDeadlineExceeded`. The new ReplicaSet has 3 of 10 pods ready, the
    old one has 8. The team assumes Kubernetes rolled it back. Did it?"
    > **Direction:** No — the Deployment is only marked failed and sits half-rolled with mixed
    > versions serving traffic; rollback must be triggered by you or a controller like Argo Rollouts
    > (§7; `O06`, `O09`).

11. "New pods pass readiness and receive traffic, but about 20 seconds later some crash on a lazily
    initialised dependency. The rollout completed before anyone noticed. How do you make the rollout
    itself catch this?"
    > **Direction:** `minReadySeconds` makes a pod count as available only after staying ready for a
    > while, which slows the rollout enough to stop on flapping pods (§7; `O06`, `O09`).

12. "A gRPC service scaled from 4 to 12 pods during a deploy, but the new pods get almost no traffic
    while the old ones stay hot. The Kubernetes Service looks correct. Why?"
    > **Direction:** A ClusterIP Service balances connections, and gRPC keeps one long-lived HTTP/2
    > connection per client; use client-side balancing over a headless Service, a mesh, or a max
    > connection age (§9; `O06`, `A14`).

13. "Kafka consumers in your service reprocess thousands of messages on every deploy, and some side
    effects happen twice. Shutdown handling exists. What's likely wrong?"
    > **Direction:** The shutdown order commits offsets too late, or `SIGKILL` arrives before the
    > consumer leaves the group; stop polling, finish in-flight, commit, then close within the grace
    > period — and make handlers idempotent anyway (§1, §7; `O06`, `M09`).

14. "The canary at 5% showed no errors for 30 minutes and was promoted. Six hours later, every pod OOMs
    within ten minutes of each other. What did the canary process miss?"
    > **Direction:** Slow leaks and time-based issues need bake time, not traffic share; compare memory
    > trends between canary and baseline, not only error rates (§12; `O09`, `O14`).

15. "A blue/green switch moved the load balancer to green, yet ten minutes later blue still serves a
    third of the traffic. What's keeping it alive?"
    > **Direction:** Long-lived connections (HTTP keep-alive, gRPC, WebSockets, DB clients) and client
    > DNS caches stay on blue; close idle connections on blue and set connection max age (§9, §12;
    > `O09`).

16. "After enabling a service mesh, some pods fail their first few outbound calls at start-up with
    'connection refused', and on shutdown in-flight requests fail. Without the mesh both were fine.
    Explain."
    > **Direction:** Container ordering — the app starts before the proxy is ready and the proxy exits
    > before the app drains; native sidecars or the mesh's hold-until-proxy-starts and drain settings
    > fix it (§7; `O06`).

17. "A deploy was blocked for 40 minutes because pods from the new ReplicaSet stayed Pending. The
    cluster was at 85% requested CPU. `maxSurge` was 50%. What's the fix beyond 'add nodes'?"
    > **Direction:** Surge needs spare capacity; lower `maxSurge` with some `maxUnavailable`, keep
    > overprovisioning placeholder pods, or request less — the rollout strategy is a capacity decision
    > (§7, §8; `O06`, `O09`).

18. "Nobody can deploy anything to the cluster — every pod creation fails with a webhook timeout,
    including the fix for the webhook. How did you get here and how do you get out?"
    > **Direction:** A validating/mutating webhook with `failurePolicy: Fail` whose service is down;
    > temporarily delete or patch the webhook configuration, then scope it to exclude its own namespace
    > and system namespaces (§7; `O05`, `O06`).

19. "A database password was rotated in the Secret. Half the pods connected fine; the other half
    started failing authentication after their connections recycled. No deploy happened. Why only
    half?"
    > **Direction:** Secrets consumed as env vars (or via `subPath`) never update in running pods, while
    > volume mounts do; either restart on rotation or support dual credentials during the overlap (§6;
    > `O06`, `S10`).

20. "A worker Deployment is scaled from 10 to 30 replicas by a batch spike, then a rollout starts
    while HPA is scaling it back down. The rollout takes two hours. Why so slow, and what would you
    do?"
    > **Direction:** Surge and unavailability are percentages of a moving replica count while HPA
    > fights the rollout, and long drains multiply; pause the HPA or roll out between spikes, and use
    > absolute surge values (§7, §8; `O06`, `O09`).

### Level 3 — Latency and CPU mysteries

Latency is bad and the obvious resource graphs look fine; the candidate must find the hidden queue.

1. "An API's p99 went from 80ms to 400ms after we moved it to Kubernetes. Average CPU is 40% of the
   limit, so the team says it isn't CPU. You have 10 minutes. What do you check first?"
   > **Direction:** CFS throttling — a multi-threaded process burns its 100ms quota early in each
   > period and waits out the rest; check the throttled-periods ratio and consider removing CPU limits
   > with accurate requests (§4; `O04`, `C14`).

2. "A Go service with a 2-CPU limit runs on 64-core nodes. It is heavily throttled and GC pauses are
   long, even at low traffic. The code is fine on a 2-core VM. Why?"
   > **Direction:** Before Go 1.25 the runtime sized `GOMAXPROCS` from the host's 64 cores, not the
   > cgroup quota, so 64 threads fight for 2 CPUs of quota; upgrade or use `automaxprocs` (§4; `O04`).

3. "A small internal service has been fine for three weeks. Since this morning, it is slow all the time,
   with no deploy and no traffic change. It runs on a `t3.medium`. What do you suspect?"
   > **Direction:** CPU credits exhausted, dropping the instance to baseline; check
   > `CPUCreditBalance`, and move to unlimited mode or a non-burstable type (§14; `O11`).

4. "A self-managed database on EC2 handles the nightly import fine for the first two hours, then write
   latency jumps tenfold for the rest of the run. Its volume is a 200GB gp2. Explain."
   > **Direction:** gp2 burst credits run out and IOPS fall to the 3-per-GB baseline (600 IOPS);
   > `BurstBalance` proves it, and gp3 gives a flat 3,000 IOPS baseline regardless of size (§14;
   > `O11`, `O07`).

5. "About 1% of outbound HTTP calls from our pods take almost exactly 5 seconds longer than the rest.
   The downstream's own metrics show fast responses. Where is the time going?"
   > **Direction:** The conntrack race on parallel A/AAAA UDP DNS queries drops one, and the resolver
   > waits its 5s timeout; `single-request-reopen` (glibc only) or NodeLocal DNSCache fixes it (§9;
   > `O02`, `A13`).

6. "Under bursts, a fraction of requests take exactly 1 second or 3 seconds longer, yet the server's
   own request timing shows everything under 50ms and CPU is idle. What's happening?"
   > **Direction:** The accept queue overflows and SYNs are dropped, so clients retransmit after 1s and
   > again after 2s more; check `TcpExtListenOverflows` and raise the backlog / `somaxconn` or add
   > acceptors (§2; `O02`, `A12`).

7. "Latency is bad only for pods on three specific nodes, and those nodes show normal CPU utilisation.
   Moving a pod off them fixes it. How do you prove what's wrong with the nodes?"
   > **Direction:** A noisy neighbour — check CPU steal, disk I/O and PSI on the node, and `runqlat`
   > for scheduler queueing that utilisation hides; then cordon or isolate (§14, §19; `O17`, `O04`).

8. "p99 latency spikes every 15 seconds like clockwork on every pod, even at night. There is no cron
   at that interval. What would you suspect?"
   > **Direction:** Something periodic in the platform — the Prometheus scrape hitting an expensive
   > `/metrics` endpoint that computes on demand, or a health probe doing real work; time the metrics
   > handler and make it cheap (§7, §16; `O14`, `O17`).

9. "As traffic rises past about 70% of capacity, pods start restarting, and each restart makes it
   worse. The restarts are liveness failures, but the app never crashes. Explain the loop."
   > **Direction:** Probes share the saturated thread pool and hit the 1s default timeout, so the
   > kubelet kills pods exactly when capacity is needed; give probes a cheap path, a longer timeout
   > and more failures before restart (§7; `O06`).

10. "A Node service reads small files, resolves hostnames and hashes passwords. Under load, file reads
    that normally take 1ms take 800ms, while the event loop lag looks fine. What's the shared
    bottleneck?"
    > **Direction:** The libuv thread pool (default 4 threads) runs `fs`, `dns.lookup` and crypto work,
    > so slow DNS or bcrypt starves file I/O; raise `UV_THREADPOOL_SIZE` and avoid `dns.lookup` on the
    > hot path (§4, §9; `C14`, `O04`).

11. "A JVM service takes 3 minutes to start on Kubernetes versus 25 seconds on a laptop, and
    sometimes never becomes ready because liveness kills it. It has a 500m CPU limit. What's the fix
    you'd propose?"
    > **Direction:** JIT and class loading are throttled by the tiny quota during start-up; add a
    > startup probe and give start-up CPU room (higher limit or no limit), since steady state needs far
    > less than boot (§4, §7; `O04`, `O06`).

12. "A platform team 'optimised' every service to Guaranteed QoS by setting CPU limits equal to
    requests. Nothing crashed, but tail latency across the company rose noticeably. Why?"
    > **Direction:** Services lost the ability to burst into idle node CPU and are now throttled on
    > every short burst; the memory half of the change is good, the CPU half is not (§4, §6; `O04`,
    > `O06`).

13. "The node's load average is 45 on 8 cores, but CPU utilisation is 15%. The API pods on it are
    slow. What does the load average actually mean here?"
    > **Direction:** Load counts tasks in uninterruptible sleep, usually blocked on I/O — often a
    > network filesystem (NFS/EFS) or a saturated disk; look at `%iowait`, `D`-state processes and
    > `biolatency` (§19; `O01`, `O17`).

14. "A pod close to its memory limit becomes very slow for minutes before it finally gets OOMKilled —
    or sometimes just stays slow. There is no GC problem. What's happening in the kernel?"
    > **Direction:** The cgroup is reclaiming and thrashing page cache (including the process's own code
    > pages) or is held at `memory.high`; memory PSI shows the stall, and more headroom fixes it (§3,
    > §4; `O04`).

15. "After a kernel upgrade on the nodes, a service shows occasional 200ms stalls and RSS 30% higher
    than before, with no code change. What kernel setting do you check?"
    > **Direction:** Transparent huge pages set to `always`, causing compaction stalls and RSS bloat;
    > set THP to `madvise` or `never` for that workload (§3; `O01`).

16. "A service moved from a Debian base image to Alpine to save 200MB. Throughput dropped by 30% on
    the same hardware and DNS errors appeared. Why would the base image matter?"
    > **Direction:** musl's allocator is much slower for multi-threaded workloads and its resolver
    > behaves differently (no `single-request-reopen`, different search handling); use a glibc slim or
    > distroless image instead (§5, §9; `O03`).

17. "Latency for requests routed through pods in `eu-west-1a` is 3ms higher than the others and our
    cross-AZ data bill is large. Pods are evenly spread. What's the likely layout problem?"
    > **Direction:** Traffic crosses zones on every hop — the database primary or a cache lives in one
    > AZ and Services pick endpoints anywhere; topology-aware routing and zone-local replicas cut both
    > latency and cost (§9, §14; `O11`, `O18`).

18. "CPU utilisation on the nodes averages 50%, the app shows no throttling, yet request latency
    rises with load as if CPU-bound. How do you prove or disprove CPU contention?"
    > **Direction:** Utilisation averages hide queueing; measure run-queue latency with `runqlat` or
    > CPU PSI, and compare per-core `mpstat` for a few hot cores (§4, §19; `O17`, `C14`).

19. "p99 latency rose after the team added detailed request tracing and structured logging at every
    step. CPU rose only 5%. Where else could the cost be hiding?"
    > **Direction:** Synchronous logging to stdout blocking on a slow log pipeline, and exporters with
    > small buffers doing work on the request path; use async, bounded exporters and check write
    > latency to stdout (§17; `O13`, `O15`).

20. "A service's latency doubled when it was scaled from 10 to 30 pods, although per-pod traffic
    dropped. Its downstream is a managed Postgres. What's your theory?"
    > **Direction:** Three times the pods means three times the connections; the database spends its
    > effort on connection overhead and contention, so the fix is a pooler and fewer total
    > connections, not more pods (§12, §19; `O09`, `DB39`).

### Level 4 — Connections, DNS and networking

The network is not reliable, and neither are the tables the kernel keeps about it.

1. "A service that calls a partner API under load starts failing with `EADDRNOTAVAIL: cannot assign
   requested address`. CPU and memory are fine. The partner is healthy. What's happening?"
   > **Direction:** Ephemeral port exhaustion to one destination because each request opens a new
   > connection and ports sit in `TIME_WAIT` for 60s; reuse connections with keep-alive pooling
   > rather than tuning the kernel (§2; `O02`, `A12`).

2. "`ss` on a pod shows 20,000 sockets in `CLOSE_WAIT`, rising steadily, and eventually the process
   hits its FD limit. What does that state tell you, and where is the bug?"
   > **Direction:** The peer closed and your code never called `close()` — a leak in your own error
   > path, usually a response body not consumed or a client not released; not a network problem
   > (§2; `O01`, `O02`).

3. "Random connection timeouts across many different pods, all on busy nodes, with nothing in any
   application log. `dmesg` on one node shows a lot of lines. What do you expect to find?"
   > **Direction:** `nf_conntrack: table full, dropping packet` — the conntrack table is full, often
   > from long-lived idle entries with a 5-day timeout; raise the max, shorten timeouts and reduce
   > connection churn (§2; `O02`, `O17`).

4. "CoreDNS pods are at 100% CPU during traffic peaks and DNS latency adds 30ms to cold connections.
   Query logs show lots of `NXDOMAIN` for names like `api.stripe.com.payments.svc.cluster.local`. Fix it."
   > **Direction:** `ndots:5` makes every external name try each search domain for A and AAAA first;
   > trailing dots on FQDNs, a lower `ndots`, NodeLocal DNSCache and connection reuse (§9; `A13`,
   > `O05`).

5. "Calls from our pods to an external API work fine when busy, but after a quiet period of about ten
   minutes the first call hangs until our 30-second timeout. Then everything is fine again. Why?"
   > **Direction:** The NAT gateway drops flows idle for more than 350s without telling either side,
   > so the pooled connection is dead; set the pool's max idle time or TCP keep-alive below 350s
   > (§2, §14; `O11`, `O02`).

6. "During a campaign, calls from our 200 pods to one third-party endpoint start failing, and the NAT
   gateway's `ErrorPortAllocation` metric is non-zero. What limit did you hit and what are the options?"
   > **Direction:** About 55,000 simultaneous connections per NAT IP to a single destination; reuse
   > connections, add NAT IPs or gateways, or spread traffic across destination IPs (§14; `O11`).

7. "A new VPN link to a partner works for health checks and small API calls, but large responses and
   some TLS handshakes hang forever. Firewall rules look correct. What's going on?"
   > **Direction:** An MTU black hole — packets above the tunnel's MTU are dropped and ICMP
   > 'fragmentation needed' is blocked; test with don't-fragment pings and clamp MSS (§2; `O02`).

8. "We failed the database over to a new primary and updated the DNS record with a 30s TTL. Ten
   minutes later, the Java services are still sending writes to the old, now read-only node. Why?"
   > **Direction:** Pooled connections never re-resolve and the runtime may cache DNS longer than the
   > TTL; give pools a max connection lifetime and set the resolver cache TTL explicitly (§9, §22;
   > `A13`, `O20`).

9. "In a cluster with 15,000 Services, deploys cause a few seconds of errors even though you use a
   `preStop` sleep. The errors are longer on nodes with more rules. What's slow?"
   > **Direction:** kube-proxy in iptables mode rewrites large rule sets, so endpoint removals reach
   > nodes late; lengthen the preStop sleep as a stopgap and move to IPVS, nftables or eBPF (§7, §9;
   > `O05`, `O06`).

10. "After enabling topology-aware routing to cut cross-AZ cost, one zone's pods run at 90% CPU while
    the others idle, because that zone has fewer pods. How do you keep the savings without the hot
    spot?"
    > **Direction:** Zone-local routing assumes balanced endpoints per zone; enforce zone spread with
    > topology spread constraints and HPA headroom so each zone's capacity matches its traffic (§8, §9;
    > `O06`, `O18`).

11. "We switched a Service to `externalTrafficPolicy: Local` to keep client IPs. Now about a third of
    requests from the load balancer time out. What did we miss?"
    > **Direction:** Nodes without a local pod drop the traffic, and the LB only avoids them if it
    > health-checks the node port kube-proxy exposes for that purpose; configure the health check or
    > run pods on every targeted node (§9; `O05`).

12. "Our access logs show every request coming from a handful of internal 10.x IPs, so per-client rate
    limiting throttles everyone together. Why, and how do you get the real client IP?"
    > **Direction:** `externalTrafficPolicy: Cluster` and the LB both SNAT; read the client IP from
    > `X-Forwarded-For` or PROXY protocol, trusting only the hops you control (§9; `O05`, `M07`).

13. "`kubectl logs -f` and the app's config hot-reloader both fail with 'too many open files' on one
    node. `ls /proc/<pid>/fd` shows only 200 descriptors. What limit are you really hitting?"
    > **Direction:** The per-user `inotify` instance or watch limit, shared by every container running
    > as that UID on the node; raise `fs.inotify.max_user_instances` / `max_user_watches` (§2; `O01`).

14. "After a base-image change, pods take 20 seconds to start and sometimes log 'unable to load
    credentials'. On the old image it was instant. Running on EKS nodes with IMDSv2. What's up?"
    > **Direction:** The IMDSv2 token request exceeds a hop limit of 1 from inside the container and
    > the SDK falls through the credential chain with timeouts; set hop limit 2 or use IRSA/Pod Identity
    > (§14; `O11`, `S15`).

15. "On our largest nodes, DNS resolution fails intermittently at peak even though CoreDNS is
    healthy and CPU is low. `ethtool -S` on the node shows a non-zero counter. Which one, and why?"
    > **Direction:** `linklocal_allowance_exceeded` — the node sends more than about 1,024 packets per
    > second to the VPC resolver; add NodeLocal DNSCache and cut query volume (ndots) (§9, §14; `O11`).

16. "We renewed our TLS certificate. Browsers are fine, but a set of older Android devices and a
    partner's Java batch job now fail the handshake. What would you check?"
    > **Direction:** The chain served (missing intermediate, or a new chain to a root the old trust
    > stores do not have); test with `openssl s_client -showcerts` from an old client and serve a
    > compatible chain (§22; `O02`, `A11`).

17. "A Node service's HTTP client pool to an internal service throws `ECONNRESET` on about 1 in 5,000
    requests, always on a reused connection, never on new ones. What's the race?"
    > **Direction:** The server closes an idle keep-alive socket just as the client reuses it; keep
    > the client's idle timeout below the server's, and retry idempotent requests once on a reset of a
    > reused socket (§2; `A12`, `M09`).

18. "WebSocket connections through the ALB drop every 60 seconds for clients that are only
    listening, never sending. Active clients are fine. Fix it."
    > **Direction:** The ALB's 60s idle timeout counts traffic in both directions; send periodic
    > ping frames from the server (or raise the idle timeout) and handle reconnects anyway (§2, §14;
    > `O11`, `A16`).

19. "Only our Alpine-based services fail to resolve one partner's hostname; Debian-based pods resolve
    it fine. The record has many A records. What's different?"
    > **Direction:** Older musl resolvers did not fall back to TCP when a UDP DNS answer was truncated,
    > so large responses fail; use a newer musl, a glibc image, or a local cache (§9; `A13`, `O03`).

20. "A partner started rejecting our requests with 403 this morning. Nothing in our code changed, but
    a Terraform apply ran overnight touching the VPC module. What do you suspect?"
    > **Direction:** A NAT gateway or Elastic IP was replaced, changing the egress IP the partner
    > allow-listed; read the plan for replacements and protect those resources (§13, §14; `O10`, `O11`).

### Level 5 — Scaling, scheduling and capacity

Autoscalers, schedulers and disruption budgets doing exactly what they were told, which was wrong.

1. "A flash sale starts at 10:00. Traffic goes 5× in 30 seconds. The HPA eventually scales from 10 to
   50 pods, but only after four minutes of errors. The HPA target is 60% CPU. What would you change?"
   > **Direction:** HPA reacts to lagging metrics and then waits for nodes and image pulls — it is for
   > trends, not bursts; pre-scale on a schedule for known events and keep overprovisioning
   > placeholder pods for instant node capacity (§8; `O06`, `C15`).

2. "A JVM service's replica count oscillates between 6 and 20 every few minutes, even with steady
   traffic. Each new pod burns CPU at start-up. Explain the feedback loop and break it."
   > **Direction:** Start-up CPU spikes push average utilisation over target, which adds pods, whose
   > start-ups push it again; exclude warm-up from the metric, lengthen the scale-up stabilisation, or
   > scale on a saturation metric instead of CPU (§8; `O06`).

3. "An I/O-bound API that waits on a slow downstream is timing out under load, but the HPA never
   scales because CPU stays at 25%. What should it scale on?"
   > **Direction:** Saturation — in-flight requests per pod, pool wait time, or queue age — via
   > custom or external metrics; CPU is the wrong signal for a service that mostly waits (§8, §16;
   > `O06`, `C14`).

4. "Someone halved the CPU request of a service to save money. The next day it was running three
   times as many pods as before. Why did the saving backfire?"
   > **Direction:** HPA utilisation is measured against the request, so halving the request doubles the
   > reported utilisation and the HPA scales out; requests, HPA targets and limits must be changed
   > together (§8; `O06`, `O18`).

5. "A cluster upgrade has been stuck for three hours on one node that won't drain. `kubectl drain`
   keeps retrying. What's blocking it, and how do you fix it without breaking the service?"
   > **Direction:** A PDB with `maxUnavailable: 0` or `minAvailable` equal to replicas, often on a
   > single-replica Deployment, makes eviction return 429 forever; scale up first, then fix the PDB to
   > allow at least one disruption (§8; `O06`).

6. "Our cluster grew from 30 to 80 nodes over a quarter, but average utilisation is 20%. The cluster
   autoscaler logs say it can't remove nodes. What's usually stopping it?"
   > **Direction:** Pods that block scale-down — strict PDBs, local storage, `safe-to-evict: false`
   > annotations, kube-system pods without PDBs — plus inflated requests; find the blocking pod per
   > node from the autoscaler status (§8, §20; `O06`, `O18`).

7. "An AZ outage took out 70% of the API's capacity, although the cluster spans three zones. Why were
   most replicas in one zone, and how do you guarantee it doesn't happen again?"
   > **Direction:** The scheduler spreads by default only weakly; add `topologySpreadConstraints` on
   > the zone key and keep enough headroom that two zones can carry the load (§8, §14; `O06`, `O11`).

8. "During the same AZ outage, a StatefulSet pod stayed Pending for the entire incident while there
   was spare capacity in the other two zones. The team was surprised. Should they have been?"
   > **Direction:** No — its PersistentVolume is zonal, so the pod can only run in the lost AZ;
   > zone-level redundancy must come from replication at the application layer, not rescheduling
   > (§8, §10; `O07`, `O20`).

9. "After adding a strict zone spread constraint, a rollout left pods Pending for an hour because one
   zone had no instance capacity. How do you get both spread and progress?"
   > **Direction:** `DoNotSchedule` trades availability of the rollout for spread; use
   > `ScheduleAnyway` with a sensible `maxSkew`, diversify instance types, and keep spare capacity per
   > zone (§8; `O06`, `O11`).

10. "We moved stateless workers to spot instances. Every few days, a wave of interruptions kills a
    third of them at once and in-flight jobs are lost. How do you make spot safe?"
    > **Direction:** Diversify instance types and zones so interruptions are uncorrelated, handle the
    > two-minute notice by draining nodes, and make jobs checkpointed and idempotent (§8, §20; `O18`,
    > `Q21`).

11. "A data team's nightly batch job, running in the same cluster, evicted a third of the API's pods at
    02:00. Nobody deleted anything. How was that possible?"
    > **Direction:** The batch pods had a higher `PriorityClass` and pre-empted the API pods to schedule;
    > give production services the higher priority and put batch in its own node pool or quota (§8;
    > `O06`).

12. "After the cluster was scaled to zero over a long weekend to save cost, a CronJob that runs every
    minute never started again on Monday. There's no error in the pod list. What happened?"
    > **Direction:** The CronJob controller refuses to schedule after more than 100 missed runs without
    > a `startingDeadlineSeconds`; the event says so, and setting a deadline fixes it (§8; `O06`).

13. "A reconciliation CronJob scheduled every 10 minutes sometimes takes 15. Twice this month two runs
    overlapped and double-posted ledger entries. What in the manifest allowed that?"
    > **Direction:** `concurrencyPolicy` defaults to `Allow`; set `Forbid`, and still make the job
    > idempotent because Jobs retry and the controller can start a run twice (§8; `O06`, `M16`).

14. "A Kubernetes Job sending a monthly customer email crashed halfway. Customers got the email up to
    seven times. Explain the number."
    > **Direction:** `backoffLimit` defaults to 6, so the pod ran seven times, each from the start;
    > record progress per recipient and make sending idempotent (§8; `O06`, `M16`).

15. "We scale queue workers with KEDA on queue depth. When a backlog of a million messages built up,
    it scaled to 300 pods and the database went down. What's missing from the scaling design?"
    > **Direction:** Scaling consumers beyond what the downstream can absorb moves the queue into the
    > database; cap `maxReplicas` by downstream capacity and scale on queue age with rate limits (§8,
    > §19; `O06`, `Q21`).

16. "The autoscaler adds nodes quickly, but new pods take four minutes to become ready on fresh nodes,
    and only one minute on warm nodes. The image is 3GB. What do you do?"
    > **Direction:** Image pull dominates cold start; shrink the image, pre-pull with a DaemonSet or node
    > image, or use a lazy-loading snapshotter (§5, §8; `O03`, `O06`).

17. "After a traffic spike ends, the HPA scales from 40 pods back down to 10 over about five minutes,
    and during that period some requests get 502s. Why, and why five minutes?"
    > **Direction:** The 300s scale-down stabilisation window explains the timing; the 502s are the same
    > termination race as deploys, so scale-down needs the `preStop` sleep and draining too (§7, §8;
    > `O06`).

18. "A load test said each pod handles 5,000 RPS. In production, pods saturate at 1,500. The code is
    the same. What was the load test not measuring?"
    > **Direction:** Unrealistic load — a few hot keys served from cache, reused connections, no TLS
    > handshakes or DNS, one client IP, closed-loop clients that hide stalls; model real key
    > distribution and use arrival-rate load (§16; `C15`, `C14`).

19. "Both the VPA (in auto mode) and an HPA on CPU are enabled for the same Deployment. Replica counts
    and requests keep swinging. What's happening?"
    > **Direction:** They control the same signal — VPA changes the request that HPA divides by — so
    > they fight; use VPA in recommendation mode, or HPA on a non-resource metric (§8; `O06`).

20. "The cluster shows 40% actual CPU usage, but new pods stay Pending with 'Insufficient cpu'. The
    team wants more nodes. What would you check first?"
    > **Direction:** Scheduling uses requests, not usage; requests are inflated relative to real use,
    > so right-size requests from usage history before buying nodes (§6, §20; `O06`, `O18`).

### Level 6 — Metrics, logs and traces that lie

The dashboard says one thing and the users say another; the candidate must know how the signal is
produced.

1. "The dashboard shows p99 latency as the average of each pod's p99, and it says 180ms. Customers
   report multi-second waits. Which number do you trust, and how would you compute the right one?"
   > **Direction:** Neither the average nor per-pod summaries give a fleet p99; sum histogram buckets
   > across pods first, then take the quantile (§16; `O14`, `C14`).

2. "A panel shows p99 at exactly 10.0 seconds, flat, for the whole incident. The engineers conclude the
   system was stable at 10s. What is it really telling you?"
   > **Direction:** The quantile fell into the `+Inf` bucket and is clamped to the highest finite
   > boundary — the real value is unknown and larger; add buckets that cover the tail (§16; `O14`).

3. "Your SLO is 300ms, and the histogram's buckets are 0.1, 0.5 and 1 second. The report says 99% of
   requests meet the SLO. Can you believe it?"
   > **Direction:** No — everything between 100ms and 500ms is interpolated linearly, so you cannot
   > count requests under 300ms; put a bucket boundary exactly at the SLO threshold (§16; `O14`,
   > `M34`).

4. "Prometheus started OOMing and restarting every hour after this morning's deploy of an unrelated
   service. Each restart takes 15 minutes to replay. What happened and how do you stop the bleeding?"
   > **Direction:** A cardinality explosion from a new label (user ID, raw path, error text) on one
   > target; find it with `count by (__name__)`, drop it with `metric_relabel_configs` or a
   > `sample_limit`, then fix the code (§16; `O14`).

5. "Application metrics show a 0.1% error rate during the incident, but support is flooded and the CDN
   shows many failures. Where are the missing errors?"
   > **Direction:** Requests that never reached the app — rejected by the LB, dropped in the accept
   > queue, timed out in clients, or served by pods that died — are invisible to it; measure at the LB
   > and the client too (§16; `O13`, `O14`).

6. "You sample traces at 1% at the root. A bug that fails 0.05% of checkouts is impossible to find in
   the tracing tool. How do you keep costs down and still catch it?"
   > **Direction:** Head sampling drops rare errors; tail-based sampling keeps all errors and slow traces
   > plus a small share of the rest (§17; `O15`).

7. "You deployed tail sampling on a three-replica collector behind a Service. Now traces are often
   missing spans and sampling decisions look random. What's wrong with the deployment?"
   > **Direction:** Spans of one trace land on different collectors, each deciding on a partial trace;
   > put a load-balancing exporter keyed on trace ID in front of the sampling tier (§17; `O15`).

8. "In a trace, a child span in the payments service starts 300ms before its parent in the gateway.
   A junior asks whether the tracing library is broken. What do you tell them?"
   > **Direction:** Clock skew between hosts — timestamps come from each machine's clock; trust
   > durations within one host, check NTP/chrony, and order across services by causality (§17; `O15`,
   > `M21`).

9. "A service crashed and the last log lines before the crash — the ones that would explain it — are
   never in the log system. Earlier lines are. Why, reliably?"
   > **Direction:** An OOMKill or `SIGKILL` destroys the logger's in-memory buffer; flush on fatal
   > signals, write unbuffered for errors, and get the cause from kernel events instead (§3, §17;
   > `O13`, `O01`).

10. "A very chatty pod's logs have random gaps of a few seconds under load, and the log shipper
    reports no errors. Where are the lines going?"
    > **Direction:** Kubelet rotates container logs at 10Mi and keeps a few files, so the file rotates
    > away before the shipper reads it; cut volume, raise rotation size, and watch shipper lag (§6,
    > §17; `O13`).

11. "A nightly batch job reports its success via the Pushgateway. The dashboard showed 'last run:
    success' for four days while the job wasn't running at all. How do you monitor batch jobs
    properly?"
    > **Direction:** The Pushgateway keeps the last value forever; push a last-success timestamp and
    > alert on its age, which also catches a job that never starts (§16, §18; `O14`, `O16`).

12. "During an incident, the total request rate graph dropped by 30%. Everyone assumed traffic had
    fallen. It hadn't. What else makes a summed rate fall?"
    > **Direction:** Pods that stop being scraped simply vanish from `sum()` — no error, no alert; check
    > `up` and target counts before trusting aggregates (§16; `O14`).

13. "A panel shows `increase(errors_total[5m])` values of 1.33 and 2.67. A developer claims the counter
    is buggy. Explain."
    > **Direction:** `increase()` extrapolates to the window boundaries, so non-integers are expected;
    > it is a rate estimate, not an exact count (§16; `O14`).

14. "A graph using `rate(requests_total[30s])` has holes and jumps around, although the service is
    fine. Scrape interval is 15s. What's wrong?"
    > **Direction:** The window must contain at least two samples, ideally four; use a window of
    > at least 4× the scrape interval (e.g. `[1m]`), or `$__rate_interval` in Grafana (§16; `O14`).

15. "Your load test tool reports p99 of 50ms at 2,000 RPS. In production at the same RPS users see
    multi-second p99 during GC pauses. Which result is wrong?"
    > **Direction:** A closed-loop tool stops sending while the server stalls, so it never measures the
    > queued requests — coordinated omission; use a constant arrival-rate generator (§16; `C14`, `C15`).

16. "After adding a `customer_id` tag to one metric in the vendor's SDK, the monthly observability
    bill tripled. Nothing else changed. Explain, and where should that dimension live instead?"
    > **Direction:** Vendors bill per unique series, and customer IDs multiply series; keep per-tenant
    > detail in logs and traces, or aggregate to tenant tiers (§16, §20; `O14`, `O18`).

17. "Error rates computed from your tracing backend don't match the application's metrics — traces
    say 12% errors, metrics say 0.5%. Both are 'correct'. How?"
    > **Direction:** Tail sampling keeps all errors and only some successes, so trace-derived ratios are
    > biased; generate RED metrics before sampling (§17; `O15`).

18. "Traces for the order flow end at the API: every Kafka consumer shows up as a separate root trace
    with no parent. How do you get one end-to-end trace?"
    > **Direction:** Context is not propagated through the message; inject `traceparent` into message
    > headers and extract it in consumers, using links for batch consumption (§17; `O15`, `M25`).

19. "Searching logs for a `NullPointerException` finds thousands of events that contain only the first
    line of the stack trace, and none show the root cause. What's happening?"
    > **Direction:** Multi-line stack traces are split into one event per line by the shipper; log
    > structured JSON with the whole stack in one field (§17; `O13`).

20. "p99 spiked at 14:03. You want to go from that point on the graph to the exact slow request and its
    logs in two clicks. What has to be in place?"
    > **Direction:** Exemplars on the latency histogram carrying trace IDs, and the same trace ID in
    > every structured log line (§16, §17; `O14`, `O15`).

### Level 7 — Alerting, on-call and incident handling

What pages, what should, and what you do in the first ten minutes.

1. "Your team gets paged about 40 times a week, mostly for 'CPU above 80%' and 'pod restarted'. Most
   pages need no action. Engineers have started muting the channel. What do you change?"
   > **Direction:** Page only on user-facing symptoms against an SLO and move cause-based signals to
   > dashboards and tickets; measure pages per shift and the share that were actionable (§18; `O16`,
   > `M34`).

2. "The checkout service was completely down for 25 minutes and the 'error rate above 5%' alert never
   fired. The alert rule is correct. How is that possible?"
   > **Direction:** With no traffic reaching the app, the ratio is `NaN` or the series disappears, so
   > the condition is never true; pair with `absent()`/`up == 0` and a traffic-drop alert (§16, §18;
   > `O16`).

3. "Alertmanager crashed on Friday night after a config change, and nobody was paged for anything
   until Monday. What single mechanism would have caught it?"
   > **Direction:** A dead-man's switch — an always-firing Watchdog alert delivered to an external
   > service that pages when it stops arriving (§18; `O16`).

4. "An alert pages when the error rate exceeds 1% over 5 minutes. It pages for harmless 2-minute blips
   and misses a slow 0.5% error rate that burned the monthly budget in a week. Design better alerts."
   > **Direction:** Multi-window burn-rate alerts — for example 14.4× over 1h with a 5m short window,
   > 6× over 6h, and a ticket at 1× over 3 days — tie urgency to budget consumption (§18; `O16`,
   > `M34`).

5. "An internal admin API gets 20 requests a minute. Its error-rate alert pages whenever one request
   fails, because that's 5%. How do you alert meaningfully on a low-traffic service?"
   > **Direction:** Ratios are noise with tiny denominators; require a minimum request count, use longer
   > windows, or add synthetic traffic to create a steady denominator (§18; `O16`).

6. "A 'queue consumer lag high' alert with `for: 10m` never fired during a two-hour incident, even
   though lag was high most of the time. Why?"
   > **Direction:** The condition dipped below the threshold briefly every few minutes, resetting the
   > pending timer; alert on a smoothed or windowed signal (`max_over_time`, age of oldest message)
   > instead of a raw flapping value (§18; `O16`, `Q15`).

7. "A regional network event generated 400 separate pages across 60 services in ten minutes. The
   on-call couldn't see the real cause in the noise. What should the alerting setup do?"
   > **Direction:** Group by incident dimensions and use inhibition rules so a region- or dependency-level
   > alert suppresses the per-service alerts beneath it (§18; `O16`).

8. "The incident was fixed at 14:10, but the latency alert kept firing until 15:05, and the on-call
   kept investigating a problem that no longer existed. What's wrong with the alert?"
   > **Direction:** A long window alone keeps the alert active until the bad period ages out; the
   > short-window condition in a multi-window burn-rate alert makes it reset quickly (§18; `O16`).

9. "A 'queue depth above 10,000' alert fires every morning during a normal batch import and was
   muted. Last week the consumers stalled at a depth of 3,000 and nobody noticed for four hours. What
   should you alert on?"
   > **Direction:** The age of the oldest message (or consumer lag in time), which captures stalls at
   > any depth and ignores healthy backlogs (§18; `O16`, `Q21`).

10. "Errors spiked ten minutes after a deploy. The team wants to understand the cause before touching
    anything, because the diff 'looks harmless'. What do you do as incident lead?"
    > **Direction:** Mitigate first — roll back the correlated change now and investigate afterwards;
    > rolling back a change that turns out innocent costs little (§18, §19; `O16`, `O17`).

11. "Fifteen engineers are in the incident call, all running their own queries and suggesting fixes,
    and two of them just made conflicting changes. What structure is missing?"
    > **Direction:** An incident commander who coordinates rather than debugs, one operator making
    > changes, and a communications role; changes are announced before they are made (§18; `O16`,
    > `S16`).

12. "You get paged for 'node disk above 80%' on autoscaled worker nodes about twice a week. The nodes
    are replaced automatically. Should this page? What about the database's disk?"
    > **Direction:** Not on cattle nodes — the kubelet and autoscaler handle it; for stateful disks,
    > alert on predicted time to full (`predict_linear`) with enough lead time to act (§18; `O16`,
    > `O07`).

13. "The cluster's control plane had a bad upgrade and was unreachable for an hour. Prometheus and
    Alertmanager ran in the same cluster, and no one was paged. What's the design fix?"
    > **Direction:** Monitoring must not share the failure domain it watches; run external synthetic
    > checks and a dead-man's switch from outside the cluster and region (§18; `O16`, `O20`).

14. "Your synthetic check hitting `/health` every minute has been green through three customer-facing
    outages. What's wrong with the check?"
    > **Direction:** It tests the process, not the user journey; synthetics should exercise a real path
    > (login, read, write) through the same edge users use (§18; `O16`, `O13`).

15. "You inherit 300 alerts for a service. Nobody knows why most thresholds are what they are, and
    none have runbooks. Where do you start?"
    > **Direction:** Rank alerts by pages and action rate, delete or demote those never actioned, and
    > require a runbook and an SLO link for anything that still pages (§18; `O16`).

16. "The post-incident review concludes: 'root cause: engineer ran the wrong command in production'.
    As a senior, what do you push back on?"
    > **Direction:** Human error is where the investigation starts; ask why the system allowed a single
    > command to do that — missing guard rails, confusing tooling, no dry run — and fix those (§18;
    > `O16`, `S16`).

17. "The team pages whenever any single pod restarts. With 200 pods that's several pages a day. But
    they're afraid to remove it because a crash-loop once went unnoticed. What would you alert on
    instead?"
    > **Direction:** Aggregate symptoms — available replicas below desired for some minutes, or restart
    > rate across the Deployment — plus the service's SLO alerts (§18; `O16`, `O06`).

18. "An internal certificate expired on a Saturday and broke service-to-service calls. It was a
    one-year certificate created manually. What should exist so this can't recur?"
    > **Direction:** Alerts on certificate expiry weeks ahead from an inventory of every certificate, and
    > ideally automated issuance and rotation (§18, §22; `O16`, `A11`).

19. "The service has used its entire monthly error budget by the 12th. Product wants to keep shipping
    features on schedule. What do you argue, and what's the mechanism?"
    > **Direction:** An agreed error-budget policy — when the budget is spent, releases slow down and
    > reliability work takes priority until it recovers; the SLO only matters if it changes decisions
    > (§18; `M34`, `O16`).

20. "Your latency alert is on average latency above 500ms. During an incident, 5% of users got 10-second
    timeouts, the average stayed at 350ms, and nothing fired. What should the alert measure?"
    > **Direction:** The fraction of requests slower than the threshold, from histogram buckets,
    > compared with the SLO — averages hide tails (§16, §18; `O14`, `O16`).

### Level 8 — Pipelines, infrastructure as code and config changes

Changes that were not code, or code that changed without anyone touching it.

1. "The pipeline was green at midnight and red at 06:00 on a branch nobody touched. The failure is in
   a library's type definitions. What happened, and how do you prevent it?"
   > **Direction:** A transitive dependency was published overnight and resolved fresh because the
   > build does not honour the lockfile; use `npm ci` (or equivalent), commit lockfiles, and proxy
   > dependencies (§11; `O08`, `S14`).

2. "A test fails about once in thirty runs; the team re-runs until green. Three months later the same
   race causes duplicate payments in production. What should the process have been?"
   > **Direction:** Flakes are often real race conditions; quarantine with an owner and deadline, track
   > flake rate, and reproduce under load or randomised ordering instead of retrying (§11; `O08`,
   > `M35`).

3. "A test passes alone and in its own file, but fails when the full suite runs in CI. What are the
   usual causes and how do you find the culprit quickly?"
   > **Direction:** Shared state or ordering dependence — database rows, globals, ports; bisect the
   > suite order and run tests in random order with a recorded seed (§11; `O08`).

4. "Tests fail every night between 23:00 and 01:00 UTC and pass the rest of the day. The runners are
   in the US. What do you look for?"
   > **Direction:** Date and time-zone logic — 'today' computed in different zones by code and tests,
   > or date boundaries crossing; freeze the clock in tests and use explicit zones (§11; `O08`,
   > `M21`).

5. "A bug reproduced in production but not in staging, although 'the same commit' was deployed to both.
   The pipeline builds the image separately for each environment. Why does that matter?"
   > **Direction:** Each build resolves base images and dependencies again, so the environments ran
   > different binaries; build once and promote the same digest (§11; `O08`, `O09`).

6. "CI is green, but a new developer's clean build fails with a missing module. It turns out CI has
   been passing for weeks with a dependency that was removed from `package.json`. How?"
   > **Direction:** A loosely keyed cache restored the old `node_modules`; key caches on the lockfile
   > hash and run a regular cache-free build (§11; `O08`).

7. "A security review flags that pull requests from forks can run the build job and write to the same
   dependency cache the main-branch release build restores. Why is that serious?"
   > **Direction:** Cache poisoning — an attacker writes a malicious package into the cache that a
   > trusted build then ships; untrusted jobs must have read-only or separate caches (§11; `O08`,
   > `S14`).

8. "Builds on self-hosted runners fail randomly with 'no space left on device' in different steps, and
   occasionally a build picks up a file from someone else's job. What's the root problem?"
   > **Direction:** Long-lived runners accumulate layers, caches and state between jobs; use ephemeral
   > runners, which also stop secrets and artifacts leaking between jobs (§11; `O08`, `S14`).

9. "During an outage, the fix was ready in 10 minutes but took 50 to ship, because the CI system
   depended on the same failing internal DNS. What would you have in place beforehand?"
   > **Direction:** A tested break-glass deploy path that does not depend on the systems likely to be
   > broken, plus a one-command rollback (§11, §12; `O08`, `O09`).

10. "A Terraform plan to remove one of five SQS queues from a list shows four queues being destroyed and
    recreated. What went wrong in the code, and how do you fix it without an outage?"
    > **Direction:** `count` indexes shifted when an element was removed; switch to `for_each` with
    > stable keys and use `moved` blocks to map existing state to the new addresses (§13; `O10`).

11. "A CI job running `terraform apply` was cancelled mid-run. Now every plan fails with 'Error
    acquiring the state lock'. A teammate suggests `force-unlock`. What do you check first?"
    > **Direction:** That no apply is still running anywhere (the cancelled runner may still be
    > applying) and what partial changes were made; then unlock and run a plan to see the real state
    > (§13; `O10`).

12. "Someone fixed a production outage by editing a security group in the console. A week later a
    routine Terraform apply reverted it and caused the same outage again. How do you stop this?"
    > **Direction:** Drift detection with scheduled `-refresh-only` plans, and a rule that emergency
    > console changes are codified straight after the incident (§13; `O10`, `O16`).

13. "Every Terraform apply resets the ECS service's desired count to 2, although the autoscaler had
    scaled it to 12 for the evening peak. What's the fix?"
    > **Direction:** Two owners for one field; let the autoscaler own it and add
    > `lifecycle { ignore_changes = [desired_count] }` (§13; `O10`, `O11`).

14. "A routine provider version bump produced a plan that replaces the production RDS instance. The
    plan was auto-applied by CI. How should this have been prevented, at three levels?"
    > **Direction:** Pin providers and commit the lock file, require human review of plans containing
    > replacements, and set `prevent_destroy` plus cloud-side deletion protection on stateful
    > resources (§13; `O10`, `O08`).

15. "The whole company's infrastructure is one Terraform root module. Plans take 20 minutes, and a
    change to a DNS record once deleted an unrelated IAM role. What would you do?"
    > **Direction:** Split state by environment and component to limit blast radius and plan time,
    > migrating resources between states with `moved`, `import` and `removed` blocks (§13; `O10`).

16. "A developer renamed a Terraform module from `bucket` to `assets_bucket`. The plan destroyed and
    recreated the production S3 bucket, and the bucket's contents were lost. What would have made the
    rename safe?"
    > **Direction:** A `moved` block (or `terraform state mv`) so the address changes without
    > replacement, and `prevent_destroy` on data-bearing resources (§13; `O10`).

17. "A config change to the rate-limiter's rules was pushed to every region at once and took the whole
    platform down in two minutes. Code changes go through canaries. What do you change?"
    > **Direction:** Treat config as code — validate semantically, and roll out progressively by cell
    > or region with automatic rollback (§21; `O19`, `M31`).

18. "Your feature-flag vendor had a 40-minute outage. During it, every service fell back to 'flag off',
    which disabled checkout for a launched payment method. How should the SDK and flags be set up?"
    > **Direction:** Evaluate locally from a cached ruleset, and make each flag's coded default the safe
    > value for today — for a launched feature, that is 'on' (§21; `O19`).

19. "A new checkout UI was rolled out to 50% via a feature flag. Users complain the page switches
    between old and new design as they click around. What's wrong with the rollout?"
    > **Direction:** Bucketing is not sticky — evaluated per request or keyed on something unstable;
    > hash on a stable user ID (§21; `O19`).

20. "A service watches its mounted ConfigMap file for changes and reloads. After a ConfigMap update, it
    never reloads, although the file content inside the pod has changed. Why?"
    > **Direction:** Kubernetes updates mounted ConfigMaps by atomically swapping a symlink, so a watcher
    > on the original file inode sees nothing; watch the directory and re-resolve the link (§21, §6;
    > `O19`, `O06`).

### Level 9 — Cloud limits, cost and serverless surprises

The provider's quotas, pricing and hidden per-instance limits, learned from the bill or the outage.

1. "The monthly AWS bill shows NAT gateway charges larger than the EKS compute. The services mostly
   talk to S3, ECR and DynamoDB. What's going on and what's the quick win?"
   > **Direction:** NAT charges per GB processed, and image pulls and S3 traffic are going through it;
   > add free gateway endpoints for S3 and DynamoDB and interface endpoints for ECR (§14, §20; `O11`,
   > `O18`).

2. "Cross-AZ data transfer is now the third-largest line on the bill, mostly between application pods
   and a three-broker Kafka cluster. How do you cut it without losing zone redundancy?"
   > **Direction:** Clients read from leaders in any zone; rack-aware follower fetching lets consumers
   > read from a replica in their own zone, and zone-aware routing does the same for services (§9,
   > §14; `O18`, `Q12`).

3. "A new ingestion job writes 20,000 objects per second to a fresh S3 prefix and gets a storm of
   `503 SlowDown` errors for the first ten minutes, then it settles. Explain and fix."
   > **Direction:** Each partitioned prefix starts at around 3,500 writes per second and S3 splits
   > partitions as load grows; spread keys across many prefixes and retry with jittered backoff (§14;
   > `O11`).

4. "The team deleted 40TB of old data from an S3 bucket, but the storage bill didn't go down at all.
   Why?"
   > **Direction:** Versioning keeps the deleted objects as non-current versions; add a lifecycle rule
   > to expire non-current versions and delete markers (§14, §20; `O11`, `O18`).

5. "A service stores each event as its own 1KB object in S3 — about 300 million a month. Storage is
   cheap, but the S3 bill is large. Where's the cost, and what design would you use?"
   > **Direction:** Per-request pricing dominates tiny objects; batch events into larger files (for
   > example compressed hourly batches) before writing (§14, §20; `O18`).

6. "Suddenly Terraform plans, the cluster autoscaler and a deploy tool all fail with 'Rate exceeded'
   from EC2 APIs in the same account. Nobody changed them. What do you look for?"
   > **Direction:** Account-level API throttling — one misbehaving controller or script polling
   > `Describe*` in a loop consumes the shared token bucket; find it in CloudTrail by caller and add
   > caching and backoff with jitter (§14; `O11`).

7. "Under peak load, a service starts failing with `ThrottlingException` from KMS. It decrypts a
   configuration secret on every request. What's the fix?"
   > **Direction:** KMS has per-account request quotas; decrypt once and cache in memory, or use envelope
   > encryption with a cached data key (§14; `O11`, `S10`).

8. "A marketing Lambda scaled to 900 concurrent executions during a campaign, and at the same time the
   payment webhooks Lambda started getting throttled. They are unrelated. Why?"
   > **Direction:** Concurrency is an account-and-region pool of 1,000 by default; give critical
   > functions reserved concurrency and cap the others (§15; `O12`).

9. "A Lambda-based API works in testing, but at launch Postgres runs out of connections within
   minutes. Each function opens one connection. Explain the arithmetic and the fix."
   > **Direction:** Every concurrent execution is its own process with its own connection, so
   > concurrency equals connections; put RDS Proxy (or a data API) in front and cap concurrency (§15;
   > `O12`, `DB39`).

10. "A Lambda's custom metrics and traces are frequently missing or arrive minutes late, and some
    outbound calls fail with connection resets at the start of an invocation. What's the common cause?"
    > **Direction:** The execution environment freezes after the handler returns, pausing unflushed
    > telemetry and letting idle connections time out; flush before returning and tolerate dead
    > connections on thaw (§15; `O12`, `O13`).

11. "An SQS-triggered Lambda with a 5-minute timeout processes some messages twice, even though it
    never errors. The queue's visibility timeout is 60 seconds. What's happening?"
    > **Direction:** Messages become visible again while still being processed; set the visibility
    > timeout well above the function timeout (AWS suggests 6×) and make processing idempotent (§15;
    > `O12`, `Q16`).

12. "A Lambda that generates thumbnails when images land in an S3 bucket produced a $20,000 bill over a
    weekend. It writes the thumbnails to the same bucket. Explain and prevent."
    > **Direction:** A recursive trigger loop — each output triggers another invocation; write to a
    > separate bucket or filtered prefix, and set concurrency caps and budget alarms (§15, §20; `O12`,
    > `O18`).

13. "A report endpoint behind API Gateway returns 504 after 29 seconds, but the Lambda logs show it
    completed successfully at 45 seconds. Users retry and the report is generated three times. What's
    the design fix?"
    > **Direction:** The gateway's integration timeout ends the request while the function keeps
    > running; make long work asynchronous (accept, return a job ID, poll or notify) with an
    > idempotency key (§15; `O12`, `A21`).

14. "CloudWatch Logs costs more than the service's compute. Logs are mostly INFO lines for every
    successful request plus health checks. What do you cut and what do you keep?"
    > **Direction:** Ingestion is priced per GB; drop health-check logs, sample successful-path logs,
    > keep all errors, and tier retention (§17, §20; `O18`, `O13`).

15. "The cluster runs 120 small nodes of 2 vCPUs each. About 30% of each node goes to DaemonSets
    and system reservations. How would you reduce cost without touching the applications?"
    > **Direction:** Per-node overhead is fixed, so small nodes waste a larger share; move to fewer,
    > larger nodes and trim DaemonSet requests (§6, §20; `O18`, `O06`).

16. "Finance bought three-year reserved capacity matching last quarter's usage. Two months later, a
    right-sizing project cut usage by 40%, and now the commitment is partly wasted. What was the order
    of operations mistake?"
    > **Direction:** Commit only after right-sizing — reserve the steady baseline that remains, keep
    > bursts on-demand and interruptible work on spot (§20; `O18`).

17. "During a real AZ outage, your autoscaling group tried to replace lost capacity in the other two
    zones but kept failing with insufficient capacity errors. Service stayed degraded for two hours.
    What design would have avoided it?"
    > **Direction:** Static stability — pre-provision enough capacity in each zone to survive one zone's
    > loss rather than relying on launching instances during the event, when everyone else is too (§14,
    > §22; `O11`, `O20`).

18. "About 30% of requests served from one AZ fail with timeouts, but every health check in that AZ
    passes. The cloud status page says nothing. What do you do right now?"
    > **Direction:** Partial zonal failures fool health checks; evacuate the zone yourself with zonal
    > shift or by weighting traffic away, rather than waiting for the provider (§14; `O11`, `O20`).

19. "Deploys of a service behind an ALB take 40 minutes for 10 pods. Each old pod lingers in
    'draining' for five minutes. Requests are all under 2 seconds. What do you tune?"
    > **Direction:** The target group's deregistration delay defaults to 300s; set it just above the
    > longest request plus the preStop window (§7, §14; `O11`, `O09`).

20. "A Java Lambda behind an API has a p99 of 6 seconds on cold starts, and traffic is spiky enough that
    cold starts are frequent. What are the options, and their trade-offs?"
    > **Direction:** Smaller packages and lazy initialisation, SnapStart for the JVM, or provisioned
    > concurrency at a fixed cost — or move the latency-sensitive path to always-on containers (§15;
    > `O12`).

### Level 10 — Compound failures, kernel depths and disaster recovery

Rare, multi-cause, and the kind of story only someone who has been there can tell.

1. "The database had a 2-minute failover. The database recovered fine, but the platform stayed down
   for an hour afterwards, with every service at full CPU and the database overloaded. Nothing else
   was broken. Explain and recover it."
   > **Direction:** A metastable failure — retries at every layer, reconnect storms and cold caches
   > keep the system overloaded after the trigger has gone; shed load or cut traffic hard, let it
   > recover, then ramp, and add retry budgets (§19; `O17`, `M10`).

2. "Node CPUs show 30% system time and high context switching, with no busy process in `top`. The
   pods on it are slower than elsewhere. What would you run on the node, and what might you find?"
   > **Direction:** `execsnoop` finds short-lived processes `top` never shows — typically exec probes or
   > scripts forking shells every second across many pods; replace them with HTTP/gRPC probes (§19,
   > §7; `O17`, `O01`).

3. "A controller that elects a leader through a Kubernetes Lease processed the same payments twice
   during a period of heavy GC. Both pods believed they were leader. How is that possible and how do
   you make it safe?"
   > **Direction:** A paused leader keeps acting after its lease expired and another pod took over;
   > leases cannot prevent that, so side effects need fencing tokens checked by the resource (§10, §4;
   > `M33`, `C11`).

4. "After a node image upgrade, one legacy service takes eight minutes to start, pinned at 100% CPU
   before it logs anything. `strace` shows millions of `close()` calls on invalid descriptors. What
   changed?"
   > **Direction:** The runtime now grants an enormous `nofile` limit (around a billion), and the
   > program closes every possible descriptor at start-up; set a sane `ulimit -n` rather than raising
   > it (§2; `O01`, `O04`).

5. "Your first real restore test: the 2TB database backup takes 14 hours to restore and another four
   before queries perform normally. The stated RTO is one hour. What did you learn, and what changes?"
   > **Direction:** Logical restores rebuild every index and cloud snapshot restores load blocks lazily,
   > so restore time must be measured; use physical backups or a warm standby, and pre-warm restored
   > volumes (§22; `O20`, `DB36`).

6. "During a restore drill in the DR account, every snapshot fails to restore with an access error
   about the encryption key. The snapshots copied fine. Why?"
   > **Direction:** The backups are encrypted with a KMS key from the production account or region
   > that the DR side cannot use (or that was scheduled for deletion); re-encrypt copies with a key
   > owned by the DR account and test restores regularly (§22; `O20`, `S10`).

7. "An attacker with a compromised admin credential deleted the production database and then all its
   automated backups, which lived in the same account. What should the backup architecture have
   been?"
   > **Direction:** Copies in a separate account with separate credentials and immutable retention
   > (object lock or vault lock), so no single identity can delete both (§22; `O20`, `S16`).

8. "You triggered a region failover during a real outage. The standby region's deployments scaled up,
   then stalled at 40% capacity with vCPU quota errors. What did the DR plan miss?"
   > **Direction:** Quotas and capacity in the standby region were sized for idle, and other
   > customers were failing over too; raise quotas ahead of time and keep a warm baseline (§14, §22;
   > `O20`, `O11`).

9. "The failover itself took 3 hours, although the runbook said 20 minutes: the container registry,
   the CI system and the secrets manager all lived in the region that was down. How do you find these
   dependencies before the next time?"
   > **Direction:** Game days that actually cut off the primary region reveal hidden regional
   > dependencies; replicate registries and secrets and keep a deploy path in each region (§22; `O20`,
   > `O08`).

10. "After a network partition between regions, both regions accepted writes for 20 minutes. The link
    is back. What are your immediate priorities, and what should have prevented it?"
    > **Direction:** Stop writes on one side, then reconcile conflicting records with business rules;
    > prevention is fencing so only one side can be primary, and failback is its own planned
    > operation (§22; `M33`, `O20`).

11. "In the failover drill, the application tier switched regions in one minute but produced errors for
    ten, all 'cannot execute INSERT in a read-only transaction'. What was the sequencing mistake?"
    > **Direction:** Apps moved before the database replica was promoted; promote the database first,
    > confirm it accepts writes, then shift traffic (§22; `O20`, `M33`).

12. "A bug corrupted order totals silently. It was discovered nine days later. The database has PITR
    with seven days of retention and two replicas. What can you recover, and what would you change?"
    > **Direction:** Replicas copied the corruption and PITR is out of window; rebuild from
    > application logs or events, and extend retention with long-term snapshots, since replicas are
    > not backups (§22; `DB36`, `O20`).

13. "A node's clock was 40 seconds behind; chrony stepped it forward. For a minute, pods on that node
    rejected valid JWTs and a metrics histogram recorded negative durations. Explain both."
    > **Direction:** Wall-clock skew breaks `nbf`/`exp` checks, and durations computed from wall time go
    > wrong when the clock steps; allow a small leeway on token checks and measure durations with a
    > monotonic clock (§17; `M21`, `O17`).

14. "During an incident, an engineer attached `strace -f` to the busiest API process to see what it
    was doing, and that pod's latency went from 200ms to 8 seconds, cascading into timeouts upstream.
    What should they have used?"
    > **Direction:** ptrace stops the process on every syscall; use eBPF tools (`opensnoop`,
    > `tcpconnect`, `offcputime`, `profile`) with near-zero overhead, or trace a pod taken out of
    > rotation (§19; `O17`, `C13`).

15. "You run `perf` on a node to find why a JVM service burns CPU. The flame graph shows the hot frames
    as hex addresses. How do you get usable symbols, and what else is wrong with safepoint-based
    profilers?"
    > **Direction:** The JIT's frames need a symbol map; use async-profiler (or a perf map agent), which
    > also avoids the bias of sampling only at safepoints (§19; `C13`, `O17`).

16. "To investigate a leak, someone took a heap dump of an 8GB JVM pod. The dump filled the pod's
    ephemeral storage, the pod was evicted, and the evidence was lost — and the pause dropped
    requests. How should it have been done?"
    > **Direction:** Take the pod out of rotation by relabelling it, write the dump to a volume sized
    > for it, and copy it off before anything restarts (§19, §6; `O17`, `C12`).

17. "One enterprise tenant's malformed import triggers a crash in a parser. The job is retried across
    every worker pod in turn, and within minutes all workers are crash-looping for all tenants. What's
    the architectural lesson?"
    > **Direction:** A poison input with shared workers has no blast-radius limit; quarantine after a
    > retry cap (DLQ), and partition tenants into cells or shuffle shards so one tenant cannot take
    > down all capacity (§22, §19; `M31`, `Q16`).

18. "A chaos experiment took down one AZ in staging. Everything failed over — except that every pod in
    the healthy zones lost internet access. What single resource was the hidden dependency?"
    > **Direction:** A single NAT gateway in the failed AZ serving every private subnet; run one NAT
    > gateway per AZ with per-AZ route tables (§14, §22; `O11`, `O20`).

19. "After a two-week holiday deploy freeze, the first release carried 60 merged changes. It broke
    three things at once, and rollback took six hours because nobody could tell which change was at
    fault. How would you run the next freeze?"
    > **Direction:** A freeze concentrates risk into the first deploy; ship the backlog in small,
    > separately canaried batches, and keep low-risk changes flowing during the freeze (§11, §12;
    > `O08`, `O09`).

20. "Cluster-wide, `kubectl` and every deploy became slow, and some controllers started timing out.
    The API server's etcd is at high latency. The team has hundreds of Helm releases with long
    histories. How are those connected?"
    > **Direction:** Every Helm revision is a Secret stored in etcd, and large objects and many
    > revisions bloat etcd and list calls for everyone; cap history (`--history-max`) and clean up old
    > release Secrets (§10; `O05`, `O07`).
