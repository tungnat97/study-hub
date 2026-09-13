[← back to the index](../README.md)

# Field 2 — Databases

40 nodes — the deepest-scoring field in a senior backend interview. Postgres and MySQL are assumed;
NoSQL is covered comparatively.

Each node has a **preface** (the general idea), then **details** broken into points, each with
**theory**, a production **example**, and **advanced** knowledge. Interview questions close every
node.

## Study files

| Part | Nodes | Covers |
|---|---|---|
| [1 — Relational model & SQL](1-relational-model-and-sql.md) | `DB01`–`DB05` | Keys, normalisation, SQL, window functions, CTEs, types, NULL, time, constraints |
| [2 — Transactions & concurrency](2-transactions-and-concurrency.md) | `DB06`–`DB12` | ACID, isolation levels, write skew, MVCC, bloat, locking, optimistic vs pessimistic, storage, WAL |
| [3 — Indexes & query planning](3-indexes-and-query-planning.md) | `DB13`–`DB21` | B+trees, index types, composite indexes, covering, partial, the planner, EXPLAIN, tuning, pooling |
| [4 — Migrations, replication & sharding](4-migrations-replication-and-sharding.md) | `DB22`–`DB25` | Zero-downtime DDL, backfills, replication, partitioning, sharding |
| [5 — NoSQL & specialised stores](5-nosql-and-specialised-stores.md) | `DB26`–`DB35` | NoSQL families, MongoDB, DynamoDB/Cassandra, LSM vs B-tree, Redis, Elasticsearch, OLAP, sketches, FTS, distributed SQL |
| [6 — Operations, security & ORMs](6-operations-security-and-orm.md) | `DB36`–`DB40` | Backups and PITR, database security, money and ledgers, observability, ORM internals |

## Node map

| ID | Node | Level | Requires | Unlocks |
|---|---|---|---|---|
| `DB01` | Relational model, keys, normalisation | Beginner | — | DB02, DB05, DB11, DB26 |
| `DB02` | SQL fundamentals | Beginner | DB01 | DB03, DB04, DB07, DB19 |
| `DB03` | Advanced SQL | Intermediate | DB02 | DB20, DB32 |
| `DB04` | Types, NULL, time, text | Intermediate | DB02 | DB13, DB38 |
| `DB05` | Constraints and integrity | Beginner | DB01 | DB09, DB22 |
| `DB06` | Transactions and ACID | Intermediate | DB02 | DB07, DB09, DB12, M13, M15, F10 |
| `DB07` | Isolation levels and anomalies | Advanced | DB02, DB06 | DB08, DB09, DB10, SD08 |
| `DB08` | MVCC internals | Advanced | DB07 | DB11, DB12, DB18, DB39 |
| `DB09` | Locking | Advanced | DB05, DB06, DB07 | DB10, DB22, C11 |
| `DB10` | Optimistic vs pessimistic concurrency | Intermediate | DB07, DB09 | DB38, SD10 |
| `DB11` | Physical storage | Advanced | DB01, DB08 | DB13, DB29, DB32 |
| `DB12` | WAL, durability, recovery | Advanced | DB06, DB08 | DB23, DB36, Q13 |
| `DB13` | B+tree indexes | Intermediate | DB04, DB11 | DB14–DB18, DB29 |
| `DB14` | Other index types | Advanced | DB13 | DB31, DB34 |
| `DB15` | Composite indexes, selectivity | Advanced | DB13 | DB16, DB19, DB20 |
| `DB16` | Covering indexes, index-only scans | Advanced | DB15 | DB20 |
| `DB17` | Partial and expression indexes | Advanced | DB13 | DB20 |
| `DB18` | The query planner | Expert | DB08, DB13 | DB19, DB20 |
| `DB19` | Reading EXPLAIN | Advanced | DB02, DB15, DB18 | DB20, DB39 |
| `DB20` | Query tuning patterns | Advanced | DB03, DB16, DB17, DB19 | DB21, DB39, SD05 |
| `DB21` | Connections and pooling | Advanced | DB20 | DB39, F24, C14 |
| `DB22` | Zero-downtime migrations | Advanced | DB05, DB09, M27 | DB36, SD14, O09 |
| `DB23` | Replication and read scaling | Advanced | DB12, M12, M33 | DB24, DB35, DB36, SD05 |
| `DB24` | Partitioning | Advanced | DB23 | DB25, DB32 |
| `DB25` | Sharding | Expert | DB24, M22, M32 | DB35, M29, SD06 |
| `DB26` | NoSQL taxonomy | Intermediate | DB01 | DB27, DB28, DB31, DB32 |
| `DB27` | Document databases (MongoDB) | Advanced | DB26 | DB29 |
| `DB28` | Wide-column and key-value at scale | Expert | DB26, DB32 | DB29, DB35 |
| `DB29` | LSM trees vs B-trees | Expert | DB11, DB13, DB27, DB28 | DB32 |
| `DB30` | Redis as a data store | Intermediate | DB26 | Q05 |
| `DB31` | Search engines | Advanced | DB14, DB26 | SD12 |
| `DB32` | Analytics, columnar, OLAP | Advanced | DB03, DB11, DB24, DB26, DB29 | SD06 |
| `DB33` | Probabilistic data structures | Advanced | DB13 | — |
| `DB34` | Full-text search in Postgres | Intermediate | DB14 | DB31 |
| `DB35` | Distributed SQL / NewSQL | Expert | DB23, DB25, DB28, M20 | — |
| `DB36` | Backup, PITR, disaster recovery | Intermediate | DB12, DB22, DB23 | O20 |
| `DB37` | Database security and privacy | Intermediate | DB05, S07 | S12 |
| `DB38` | Money, ledgers, correctness | Advanced | DB04, DB10 | SD10 |
| `DB39` | Observability and debugging | Intermediate | DB08, DB19, DB20, DB21 | O14, O17 |
| `DB40` | ORM internals | Advanced | DB20, DB21, F09 | F09, F21, F22 |

## Topological order (study waves)

```
Wave 0   DB01
Wave 1   DB02  DB05  DB26
Wave 2   DB03  DB04  DB06  DB27  DB30
Wave 3   DB07
Wave 4   DB08  DB09  DB12
Wave 5   DB10  DB11  DB13
Wave 6   DB14  DB15  DB17  DB18  DB22  DB23  DB33  DB34  DB37  DB38
Wave 7   DB16  DB19  DB24  DB28  DB31  DB36
Wave 8   DB20  DB25  DB29
Wave 9   DB21  DB32  DB35
Wave 10  DB39  DB40
```

Cross-field parents: `M12` consistency, `M20` consensus, `M22` data ownership, `M27` deployments,
`M32` partitioning, `M33` replication, `S07` injection, `F09` persistence layer.

**Highest-yield twelve:** `DB07`, `DB08`, `DB09`, `DB13`, `DB15`, `DB19`, `DB20`, `DB21`, `DB22`,
`DB23`, `DB25`, `DB40`. Be able to draw a B+tree, read an `EXPLAIN` out loud, and describe write
skew from memory.
