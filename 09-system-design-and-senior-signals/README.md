[← back to the index](../README.md)

# Field 9 — System Design & Senior Signals (the capstone)

17 nodes. Every node here consumes the other eight fields. This is where the offer is decided: the
design round tests whether you can *use* the knowledge under ambiguity, and the behavioural round
tests whether you can be trusted with scope.

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — The design process](1-design-process.md) | `SD01`–`SD08` | Scoping, estimation, high-level design, data modelling, scaling reads and writes, failure design, consistency decisions |
| [2 — Scale patterns & migration](2-scale-patterns-and-migration.md) | `SD09`–`SD14` | Hot spots and fan-out, correctness in payments and bookings, rate limiting at scale, canonical designs, geography and tenancy, migration |
| [3 — Communication & behavioural](3-communication-and-behavioural.md) | `SD15`–`SD17` | How to communicate in the round, the practice problem set, senior behavioural signals |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `SD01` | Requirements and scoping | Intermediate | — | SD02, SD03, SD15 |
| `SD02` | Estimation | Intermediate | SD01, C14, C15 | SD03, SD04, SD06 |
| `SD03` | High-level design | Intermediate | SD01, SD02, M01, M03 | SD04–SD07 |
| `SD04` | Data model and storage choice | Advanced | SD03, DB01, DB26, M22 | SD05, SD06, SD08 |
| `SD05` | Scaling reads | Advanced | SD04, DB23, Q02, Q04, Q09, A04, M19 | SD09, SD12 |
| `SD06` | Scaling writes | Advanced | SD02, SD04, DB25, Q14, Q20, DB32 | SD09, SD10 |
| `SD07` | Availability and failure design | Advanced | SD03, M10, M31, M33, M35, O20 | SD13, SD15 |
| `SD08` | Consistency decisions | Advanced | SD04, M12, DB07 | SD10, SD13 |
| `SD09` | Skew, hot spots, fan-out | Advanced | SD05, SD06, Q14, M32 | SD12 |
| `SD10` | Correctness in user-facing flows | Expert | SD06, SD08, M14, M16, A09, DB38, Q21, C11 | SD15 |
| `SD11` | Rate limiting and fairness at scale | Advanced | A08, Q04, S13, M29 | SD13 |
| `SD12` | Canonical design patterns | Advanced | SD05, SD09, M23, A15, A16, A21, DB31, Q17 | SD16 |
| `SD13` | Geography, tenancy, regulation | Expert | SD07, SD08, SD11, M29, M31, O11, O20, S15 | SD15 |
| `SD14` | Migration, rollout, backfill | Advanced | SD03, DB22, M27, M28, O09, Q19 | SD15 |
| `SD15` | Communicating like a senior | Advanced | SD01, SD07, SD10, SD13, SD14, O14, O17, O18, M34, F27 | SD17 |
| `SD16` | Practice problem set | Advanced | SD12 | — |
| `SD17` | Senior behavioural signals | Advanced | SD15 | — |

## Topological order (study waves)

```
Wave 0  SD01
Wave 1  SD02
Wave 2  SD03
Wave 3  SD04
Wave 4  SD05  SD06  SD07  SD08  SD11
Wave 5  SD09  SD10  SD14
Wave 6  SD12  SD13
Wave 7  SD15
Wave 8  SD16  SD17
```

Every node here also requires nodes from Fields 1-8; the cross-field master order is in the
[root index](../README.md).

## The three rounds to rehearse

**Round A — deep technical (60 min).** "Explain what happens when a request hits our API"
(`A01`, `A12`, `F01`) · "Your endpoint got slow this week, walk me through it" (`O17`, `DB20`, `C14`) ·
"Explain isolation levels; what is write skew" (`DB07`) · "Design idempotency for a payment endpoint"
(`A09`, `M16`) · "Order these Node outputs" (`C03`).

**Round B — architecture (60 min).** "Design ticket booking" (`SD10`), with follow-ups on the seat
hold, the payment timeout, the 500,000-user thundering herd, the multi-region question, and "what
would you build first if you had two weeks".

**Round C — ownership and behaviour (45 min).** An incident story · a technical disagreement · a
migration you sequenced · your biggest mistake · what you would change about your current
architecture · your questions for them (`SD17`).
