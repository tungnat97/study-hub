[← back to the index](../README.md)

# Field 1 — Microservices & Distributed Systems

35 nodes. Each node has a **preface** (the general idea), then **details** broken into points, each
with **theory**, an **example** from production, and **advanced** knowledge. Interview questions
close every node.

Levels: `Beginner` · `Intermediate` · `Advanced` · `Expert`.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Foundations & communication](1-foundations.md) | `M01`–`M07` | Why services, decomposition, sync vs async, protocols, discovery, gateway, load balancing |
| [2 — Failure & consistency](2-failure-and-consistency.md) | `M08`–`M16` | Failure modes, retries, circuit breakers, CAP, consistency models, 2PC, sagas, outbox, idempotency |
| [3 — Events, data & coordination](3-events-data-and-coordination.md) | `M17`–`M23` | Event-driven architecture, event sourcing, CQRS, consensus, clocks, data ownership, cross-service queries |
| [4 — Evolution & delivery](4-evolution-and-delivery.md) | `M24`–`M28` | Versioning, tracing, service mesh, deployment strategies, strangler fig |
| [5 — Scale & reliability](5-scale-and-reliability.md) | `M29`–`M35` | Multi-tenancy, backpressure, blast radius, partitioning, replication, SLOs, testing |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `M01` | Why services at all | Beginner | — | M02, M03, M22, M28 |
| `M02` | Decomposition: bounded contexts, aggregates | Intermediate | M01 | M17, M22, M24, M29 |
| `M03` | Synchronous vs asynchronous | Beginner | M01 | M04, M06, M08, M17, Q10 |
| `M04` | Protocols: REST, gRPC, messaging | Intermediate | M03, A01 | M24, M26 |
| `M05` | Service discovery | Intermediate | M03 | M07, M26 |
| `M06` | API gateway and BFF | Intermediate | M03 | M26, A08, S13 |
| `M07` | Load balancing | Intermediate | M05 | M10, M31, C14 |
| `M08` | Failure modes and fallacies | Intermediate | M03 | M09, M11, M20, M21, M30 |
| `M09` | Timeouts, retries, backoff, idempotency | Intermediate | M08 | M10, M16, SD10, Q16 |
| `M10` | Circuit breaker, bulkhead, load shedding | Advanced | M07, M09 | M26, M30, M31, SD07 |
| `M11` | CAP and PACELC | Intermediate | M08 | M12, M20, M33 |
| `M12` | Consistency models | Advanced | M11 | M14, M19, M23, SD08, DB23 |
| `M13` | Distributed transactions and 2PC | Advanced | M12, DB06 | M14, M20 |
| `M14` | Sagas and compensation | Advanced | M12, M13 | M15, M16, SD10 |
| `M15` | Outbox, inbox and CDC | Advanced | M14, DB06, Q10 | M16, M18, Q18 |
| `M16` | Idempotent consumers, exactly-once myth | Advanced | M09, M14, M15 | SD10, Q15, Q16 |
| `M17` | Event-driven architecture | Intermediate | M02, M03 | M18, M19, M24, Q10, Q19 |
| `M18` | Event sourcing | Expert | M15, M17 | M19 |
| `M19` | CQRS and read models | Advanced | M12, M17, M18 | M23, SD05, SD12 |
| `M20` | Consensus, quorums, leader election | Expert | M08, M11, M13 | M31, M33, DB35 |
| `M21` | Time, clocks and ordering | Advanced | M08 | M33, DB35, Q14 |
| `M22` | Data ownership: database per service | Intermediate | M01, M02 | M23, M25, SD04, DB25 |
| `M23` | Querying across services | Advanced | M12, M19, M22 | SD12 |
| `M24` | Contracts and backward compatibility | Advanced | M02, M04, M17 | M27, A22, Q19 |
| `M25` | Distributed tracing and correlation | Intermediate | M22, O13 | M26, O15 |
| `M26` | Service mesh and sidecars | Advanced | M04, M05, M06, M10, M25 | M27, S15 |
| `M27` | Deployment and release strategies | Intermediate | M24, M26 | M28, O09, SD14 |
| `M28` | Strangler fig migration | Advanced | M01, M22, M27 | SD14 |
| `M29` | Multi-tenancy | Advanced | M02, DB25 | SD13, S05 |
| `M30` | Backpressure and flow control | Advanced | M08, M10 | C16, Q15 |
| `M31` | Blast radius: cells, shuffle sharding | Expert | M07, M10, M20 | SD07, SD13 |
| `M32` | Partitioning and consistent hashing | Advanced | M07 | M33, DB25, Q14 |
| `M33` | Replication, failover, split-brain | Advanced | M11, M20, M21, M32 | DB23, SD07 |
| `M34` | SLIs, SLOs, error budgets | Intermediate | M08, O13 | O16, SD15 |
| `M35` | Testing distributed systems | Advanced | M24, M26, F12 | SD07 |

## Topological order (study waves)

Everything in a wave depends only on earlier waves, so a wave can be studied in any order.

```
Wave 0  M01
Wave 1  M02  M03
Wave 2  M04  M05  M06  M08  M17  M22
Wave 3  M07  M09  M11  M21  M24  M25  M32  M34
Wave 4  M10  M12  M29  M30
Wave 5  M13  M20  M23  M26  M33
Wave 6  M14  M27  M31  M35
Wave 7  M15  M28
Wave 8  M16  M18
Wave 9  M19
```

Cross-field parents referenced above: `A01` HTTP, `DB06` transactions, `DB25` sharding,
`Q10` queue basics, `O13` observability, `F12` testing.

**If you only have three days for this field:** M01 → M02 → M03 → M08 → M09 → M10 → M11 → M12 →
M14 → M15 → M16 → M24 → M33.
