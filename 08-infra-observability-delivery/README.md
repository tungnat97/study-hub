[← back to the index](../README.md)

# Field 8 — Infrastructure, Observability & Delivery

20 nodes. You are not applying for an SRE role, and a senior backend engineer owns their service in
production. The bar: you can debug a live incident, explain what your deploy actually does, and read
a dashboard.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Linux, containers & Kubernetes](1-linux-containers-kubernetes.md) | `O01`–`O07` | Signals and file descriptors, network debugging, container images, cgroups and OOMKill, Kubernetes objects, probes and zero-downtime deploys, stateful workloads |
| [2 — CI/CD, IaC & cloud](2-delivery-and-cloud.md) | `O08`–`O12` | Pipelines, deployment and release, infrastructure as code, cloud primitives, serverless |
| [3 — Observability & operations](3-observability-and-operations.md) | `O13`–`O20` | The three signals, metrics and dashboards, tracing, alerting and on-call, debugging production, cost, runtime config, disaster recovery |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `O01` | Linux fundamentals | Intermediate | — | O02, O04, O17 |
| `O02` | Network debugging | Intermediate | O01, A12, A13 | O17 |
| `O03` | Containers | Intermediate | F28 | O04, O05, O08, S14 |
| `O04` | Container runtime and resources | Advanced | O01, O03 | O06, C12 |
| `O05` | Kubernetes basics | Intermediate | O03 | O06, O07, O09, M05 |
| `O06` | Kubernetes for service owners | Advanced | O04, O05, C12 | O09, O16, M05, F28 |
| `O07` | Stateful workloads | Intermediate | O05 | O20 |
| `O08` | CI/CD pipelines | Intermediate | O03, F12, S14 | O09, S17 |
| `O09` | Deployment and release | Advanced | O05, O06, O08, M27, DB22 | O16, SD14 |
| `O10` | Infrastructure as code | Intermediate | O05 | O11, O19 |
| `O11` | Cloud primitives | Intermediate | A13, O10, S15 | O12, O18, O20, SD13 |
| `O12` | Serverless | Intermediate | O11 | O18 |
| `O13` | Observability fundamentals | Intermediate | F08 | O14, O15, M25, M34, DB39 |
| `O14` | Metrics and dashboards | Advanced | O13, C14, DB39 | O16, SD15 |
| `O15` | Distributed tracing in practice | Advanced | O13, M25 | O17 |
| `O16` | Alerting and on-call | Advanced | O06, O09, O14, M34, S16 | O17 |
| `O17` | Debugging production | Advanced | O01, O02, O15, O16, C13, DB39 | SD15 |
| `O18` | Cost and efficiency | Intermediate | O11, O12, C15 | SD15 |
| `O19` | Runtime config and feature flags | Intermediate | F05, O10, S10 | O09 |
| `O20` | Disaster recovery | Advanced | O07, O11, DB36, M33 | SD13, SD07 |

## Topological order (study waves)

```
Wave 0  O01  O03  O13
Wave 1  O02  O04  O05  O08  O10
Wave 2  O06  O07  O11  O14  O15  O19
Wave 3  O09  O12  O16  O20
Wave 4  O17  O18
```

Cross-field parents: `A12`/`A13` TCP and DNS, `C12`–`C15` memory, profiling and latency, `DB22`
migrations, `DB36` backups, `DB39` database observability, `F05`/`F08`/`F12`/`F28` framework, and
`M25`, `M27`, `M33`, `M34`, `S10`, `S14`–`S16`.

**Most-asked four:** zero-downtime deploy end to end (`O06`, `O09`); "p99 tripled, debug it live"
(`O17`); requests, limits and OOMKilled (`O04`, `O06`); and what you would instrument and alert on
(`O13`, `O14`, `O16`).
