[← back to the index](../README.md)

# Field 4 — API Design, HTTP & Network Protocols

22 nodes. The field that separates "writes endpoints" from "designs interfaces other teams depend on
for years".

Each node has a **preface**, then **details** with **theory** / **example** / **advanced**, and
interview questions.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — HTTP & REST](1-http-and-rest.md) | `A01`–`A09` | HTTP semantics, idempotency, REST modelling, caching, pagination, versioning, errors, rate limiting, idempotency keys |
| [2 — Transport & protocols](2-transport-and-protocols.md) | `A10`–`A17` | HTTP/2 and HTTP/3, TLS, TCP, DNS, gRPC, GraphQL, realtime and webhooks, serialisation |
| [3 — Auth, payloads & governance](3-auth-payloads-and-governance.md) | `A18`–`A22` | API auth, CORS, compression and streaming, long-running and bulk operations, contract testing |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `A01` | HTTP/1.1 semantics | Beginner | — | A02, A03, A04, A10, A19, F01, M04 |
| `A02` | Safety, idempotency, cacheability | Beginner | A01 | A03, A09, M09 |
| `A03` | REST and resource modelling | Intermediate | A02, F02 | A05, A06, A07, A21 |
| `A04` | HTTP caching | Intermediate | A01 | A20, Q09, SD05 |
| `A05` | Pagination, filtering, sorting | Intermediate | A03, DB20 | A21, SD05 |
| `A06` | Versioning and evolution | Advanced | A03, M24 | A22 |
| `A07` | Error contracts | Intermediate | A03, F07 | A22 |
| `A08` | Rate limiting and quotas | Advanced | A01, M06 | S13, SD11 |
| `A09` | Idempotency keys | Advanced | A02, M09 | SD10 |
| `A10` | HTTP/2 and HTTP/3 | Advanced | A01 | A11, A12, A20 |
| `A11` | TLS | Advanced | A10, A12 | S10, S15, M26 |
| `A12` | TCP and transport | Advanced | — | A10, A11, A13, C02, O02 |
| `A13` | DNS | Intermediate | A12 | O02, O11 |
| `A14` | gRPC and protobuf | Advanced | A10, A17, M04 | A22 |
| `A15` | GraphQL | Advanced | A03, DB20 | SD12 |
| `A16` | Realtime protocols and webhooks | Advanced | A01, A10 | F16, SD12 |
| `A17` | Serialisation and schema evolution | Advanced | A01 | A14, Q19 |
| `A18` | API authentication | Advanced | A01, S03, S04 | S05 |
| `A19` | CORS | Intermediate | A01 | F26, S08 |
| `A20` | Compression and streaming | Intermediate | A04, A10 | C19, F17 |
| `A21` | Long-running and bulk APIs | Advanced | A03, A05 | SD12 |
| `A22` | Contract testing and governance | Advanced | A06, A07, A14, M24 | M35 |

## Topological order (study waves)

```
Wave 0  A01  A12
Wave 1  A02  A04  A10  A13  A17  A19
Wave 2  A03  A11  A16  A20
Wave 3  A05  A06  A07  A08  A09  A14  A15  A18  A21
Wave 4  A22
```

Cross-field parents: `F02`/`F07` framework routing and errors, `DB20` query tuning, `M06` gateway,
`M09` retries, `M24` contracts, `S03`/`S04` tokens and OAuth.

**Most-asked five:** idempotency keys end to end (`A09`), rate limiter design (`A08`), pagination for
a live feed (`A05`), JWT versus sessions and revocation (`A18`), CORS explained correctly (`A19`).
