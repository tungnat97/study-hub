[← back to the index](../README.md)

# Field 5 — Concurrency, Runtimes & Performance

19 nodes. Where "senior" is actually tested for a Node engineer: they will ask about the event loop,
and then whether you understand the *other* models well enough to move stacks.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Concurrency models](1-concurrency-models.md) | `C01`–`C08` | Processes and threads, blocking I/O, the Node event loop, CPU-bound work, promises, JVM threads and virtual threads, Python GIL, memory model |
| [2 — Coordination & memory](2-coordination-and-memory.md) | `C09`–`C13` | Locks, lock-free and single-writer, distributed locks and fencing, garbage collection and leaks, profiling |
| [3 — Latency & throughput](3-latency-and-throughput.md) | `C14`–`C19` | Percentiles and Little's Law, load testing, backpressure, network efficiency, data structures, streaming |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `C01` | Process, thread, coroutine | Beginner | — | C02, C06, C07, C09 |
| `C02` | Blocking vs non-blocking I/O | Intermediate | C01, A12 | C03, C06, C07, C19 |
| `C03` | The Node.js event loop | Advanced | C02 | C04, C05, C12, F24, F28 |
| `C04` | CPU-bound work in Node | Advanced | C03 | C13, F24 |
| `C05` | Promises and async/await | Intermediate | C03 | C16, C17 |
| `C06` | JVM concurrency and virtual threads | Advanced | C01, C02 | C08, C09, C12, F22 |
| `C07` | Python GIL, asyncio, WSGI/ASGI | Advanced | C01, C02 | C12, F21 |
| `C08` | Memory model, visibility, atomics | Expert | C06 | C09, C10 |
| `C09` | Locks and coordination | Advanced | C01, C06, C08, DB09 | C10, C11 |
| `C10` | Lock-free, immutability, actors | Expert | C08, C09 | C11 |
| `C11` | Distributed locks and fencing | Advanced | C09, C10, Q08, M21 | SD10 |
| `C12` | Memory management and GC | Advanced | C03, C06, C07 | C13, O06 |
| `C13` | Profiling and benchmarking | Advanced | C04, C12 | C14, O17 |
| `C14` | Percentiles, Little's Law, queueing | Advanced | C13, DB21, M07 | C15, C16, O14, SD02 |
| `C15` | Load testing and capacity planning | Intermediate | C14 | SD02, O18 |
| `C16` | Backpressure end to end | Advanced | C05, C14, M30 | F16, Q15 |
| `C17` | Network efficiency patterns | Intermediate | C05 | SD05 |
| `C18` | Data structures and algorithmic cost | Intermediate | — | C13, DB33 |
| `C19` | Streaming and zero-copy | Advanced | C02, A20 | F17 |

## Topological order (study waves)

```
Wave 0  C01  C18
Wave 1  C02
Wave 2  C03  C06  C07  C19
Wave 3  C04  C05  C08  C12
Wave 4  C09  C13
Wave 5  C10  C14  C17
Wave 6  C11  C15  C16
```

Cross-field parents: `A12` TCP, `A20` payloads, `DB09` locking, `DB21` pooling, `M07` load
balancing, `M21` clocks, `M30` backpressure, `Q08` Redis atomics.

**The four you must nail:** the event-loop output-ordering question (`C03`); Little's Law and
percentile reasoning (`C14`); distributed locks and fencing (`C11`); and "how would you find this in
production" for a leak or a latency spike (`C12`, `C13`).
