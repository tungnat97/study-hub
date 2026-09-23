[← back to the index](../README.md)

# Field 3 — Backend Frameworks (NestJS deep · Django · Spring Boot · others)

28 nodes. Every node gives the **concept** first, then how Nest, Django and Spring Boot each express
it — so one piece of understanding answers a question about any of the three.

Strategy for the interview: a candidate who says *"this is the DI container; Nest calls them
providers, Spring calls them beans, Django mostly does without one and here is what that costs"*
outranks a candidate who only knows one framework.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Request lifecycle & DI](1-request-lifecycle-and-di.md) | `F01`–`F08` | Lifecycle, routing, middleware pipeline, dependency injection, config, validation, errors, logging |
| [2 — Persistence, jobs & realtime](2-persistence-jobs-and-realtime.md) | `F09`–`F17` | ORMs, transactions, auth, testing, background jobs, scheduling, caching, WebSockets, file upload |
| [3 — Architecture & deep dives](3-architecture-and-deep-dives.md) | `F18`–`F23` | Internal architecture, NestJS deep dive and internals, Django, Spring Boot, other stacks |
| [4 — Performance, security & choosing](4-performance-security-and-choosing.md) | `F24`–`F28` | Framework performance, OpenAPI, web security checklist, stack choice, runtime and graceful shutdown |
| [5 — Real-life production problems](5-real-life.md) | Pre-knowledge + 200 questions | Keep-alive 502s, pool starvation and deadlock, transaction and ORM traps, DI scope and pipeline order, job double-runs, graceful shutdown, containers, leaks, context propagation, caching, proxies and upgrade defaults |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `F01` | Request lifecycle | Beginner | A01 | F02, F03, F28 |
| `F02` | Routing and binding | Beginner | F01 | F06, F07, F25, A03 |
| `F03` | Middleware / interceptor pipeline | Intermediate | F01 | F07, F08, F11, F15, F26 |
| `F04` | Dependency injection | Intermediate | F01 | F09, F12, F18, F19, F22 |
| `F05` | Configuration and secrets | Beginner | F01 | F18, O19, S10 |
| `F06` | Validation and serialisation | Intermediate | F02 | F07, F25, S09 |
| `F07` | Error handling | Intermediate | F02, F03, F06 | F08, A07 |
| `F08` | Logging | Intermediate | F03, F07 | O13, M25 |
| `F09` | Persistence layer | Intermediate | F04, DB02 | F10, F13, DB40 |
| `F10` | Transactions in the framework | Advanced | F09, DB06 | F13, M15 |
| `F11` | AuthN / AuthZ integration | Intermediate | F03, S02 | S05, F26 |
| `F12` | Testing | Intermediate | F04 | F18, M35, O08 |
| `F13` | Background jobs | Advanced | F09, F10, Q10 | F14, Q21, SD10 |
| `F14` | Scheduling and cron | Intermediate | F13 | Q21 |
| `F15` | Framework-level caching | Intermediate | F03, Q02 | SD05 |
| `F16` | WebSockets and SSE | Advanced | F03, A16 | SD12, C16 |
| `F17` | File upload and streaming | Intermediate | F02, C19 | S14 |
| `F18` | Application architecture | Advanced | F04, F05, F12 | F19, F21, F22, F27 |
| `F19` | NestJS deep dive | Advanced | F04, F18 | F20, F28 |
| `F20` | NestJS internals | Expert | F19 | — |
| `F21` | Django deep dive | Advanced | F18, DB40 | F23, F27 |
| `F22` | Spring Boot deep dive | Advanced | F18, DB40 | F23, F27 |
| `F23` | Other stacks | Intermediate | F21, F22 | F27 |
| `F24` | Framework performance | Advanced | F09, DB21, C03 | F28, C14 |
| `F25` | API docs and codegen | Intermediate | F02, F06 | A22 |
| `F26` | Framework security | Advanced | F03, F11, S06 | S06, S08 |
| `F27` | Choosing and migrating stacks | Advanced | F18, F21, F22, F23 | SD15 |
| `F28` | Runtime and graceful shutdown | Advanced | F01, F19, F24 | O03, O06, C03, C07 |

## Topological order (study waves)

```
Wave 0  F01
Wave 1  F02  F03  F04  F05
Wave 2  F06  F09  F11  F12  F15  F17
Wave 3  F07  F10  F16  F18  F25  F26
Wave 4  F08  F13  F19  F21  F22  F24
Wave 5  F14  F20  F23  F28
Wave 6  F27
```

Cross-field parents: `A01` HTTP, `A16` realtime protocols, `DB02` SQL, `DB06` transactions,
`DB21` pooling, `DB40` ORM internals, `Q02` caching, `Q10` queues, `C03`/`C07` runtimes,
`S02`/`S06` security.

## The five questions you will definitely be asked

1. Nest pipeline order — middleware → guards → interceptors → pipes → handler → interceptors →
   filters (`F03`).
2. DI scopes, and why a TypeScript interface needs an injection token (`F04`, `F19`).
3. N+1 and how your ORM causes it (`DB40`, `F21`).
4. Transaction boundaries, and the enqueue-before-commit bug (`F10`).
5. Graceful shutdown, end to end (`F28`).

## Cross-training exercise (two weeks of evenings)

Build the **same** small service three times — a `POST /orders` with validation, a transaction, an
outbox row, a background job and one integration test — in Nest, Django/DRF and Spring Boot. You
will then answer any comparative question from experience rather than from reading.
