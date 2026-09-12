[← back to the index](../README.md)

# Field 2 — Databases

Deepest-scoring field for a senior backend interview. Postgres/MySQL are assumed; NoSQL is
comparative. Legend: `B` beginner · `I` intermediate · `A` advanced · `X` expert.

---

## Nodes

#### DB01 · Relational model, keys, normalisation
`B` · Requires: — · Unlocks: DB02, DB05, DB11, DB26
- Key: relation/tuple/domain; candidate, primary, natural vs surrogate keys; 1NF → 2NF → 3NF → BCNF
  in one sentence each ("every non-key attribute depends on the key, the whole key, and nothing but
  the key"); deliberate denormalisation for read paths and what it costs in write consistency.
- UUIDv4 vs bigint vs UUIDv7/ULID as PK: index locality, page splits, size in every secondary index.
- Q: "Normalise this schema; now tell me where you would denormalise it and why."
- Q: "UUID or auto-increment primary key? Defend it." (v4 kills B-tree locality and WAL/replication
  efficiency → UUIDv7/ULID or bigint; UUID helps with shard-local generation and enumeration safety).

#### DB02 · SQL fundamentals
`B` · Requires: DB01 · Unlocks: DB03, DB04, DB07, DB19
- Key: logical query processing order (FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT →
  ORDER BY → LIMIT) — this explains why you cannot use a SELECT alias in WHERE; inner/left/right/
  full/cross joins; anti-joins (`NOT EXISTS` vs `NOT IN` and the NULL trap); aggregates; `HAVING` vs
  `WHERE`; set operations.
- Q: "`NOT IN (subquery)` returns nothing. Why?" (a single NULL in the subquery makes it UNKNOWN).
- Q: "Write a query for the top 3 orders per customer." (window function or lateral join).

#### DB03 · Advanced SQL
`I` · Requires: DB02 · Unlocks: DB20, DB32
- Key: window functions (`ROW_NUMBER/RANK/DENSE_RANK`, `LAG/LEAD`, running totals, frame clauses
  ROWS vs RANGE); CTEs and recursive CTEs (trees, graphs, gap-filling); LATERAL / `CROSS APPLY`;
  `GROUPING SETS`, `ROLLUP`; `FILTER`; upsert (`INSERT ... ON CONFLICT` / `MERGE`); `RETURNING`;
  `DISTINCT ON`; JSONB operators and indexing.
- Q: "Deduplicate a table keeping the newest row per key."
- Q: "Compute a 7-day rolling average per user in one query."
- Q: "Was a CTE an optimisation fence?" (Postgres < 12 always materialised; 12+ inlines unless
  `MATERIALIZED` — a real senior-level gotcha).

#### DB04 · Types, NULL semantics, time, text
`I` · Requires: DB02 · Unlocks: DB13, DB38
- Key: three-valued logic and NULL propagation; `numeric/decimal` vs float for money; `timestamptz`
  vs `timestamp` (Postgres stores UTC, converts on I/O — always use timestamptz); storing user-local
  time needs the zone id, not the offset; DST and "wall clock" arithmetic; collation and why
  changing glibc collation silently corrupts indexes; `text` vs `varchar(n)`; enums vs lookup tables;
  arrays and JSONB (when they are a schema smell).
- Q: "Where do you store the timezone for a recurring 9am meeting?" (the IANA zone, not an offset).
- Q: "Why is `float` wrong for money and what do you use instead?"

#### DB05 · Constraints & data integrity
`B` · Requires: DB01 · Unlocks: DB09, DB22
- Key: PK/unique/FK/check/not-null/exclusion constraints; ON DELETE CASCADE vs RESTRICT and the
  hidden lock/write amplification of cascades; deferrable constraints; partial unique indexes for
  soft delete; why FKs are often dropped at extreme scale and what you lose; enforcing invariants in
  the DB vs the app (the DB is the only place that holds under concurrency).
- Q: "Your app checks uniqueness with a SELECT before INSERT. What is the bug?" (race — you need a
  unique index; handle the 23505 error).
- Q: "Should the database enforce FKs in a microservice world?" (yes within a service, impossible
  across services → eventual consistency + reconciliation).

#### DB06 · Transactions & ACID
`I` · Requires: DB02 · Unlocks: DB07, DB09, DB12, M13, M15, F10
- Key: atomicity, consistency (invariants, application-defined), isolation, durability; what a
  transaction actually costs; long transactions as the root of most Postgres pain (bloat, lock
  queues, replication lag); savepoints; transaction scope should not wrap network calls.
- Q: "What breaks if you call a third-party API inside a DB transaction?"
- Q: "What does the C in ACID actually mean?" (the least interesting letter — app invariants).

#### DB07 · Isolation levels & anomalies
`A` · Requires: DB02, DB06 · Unlocks: DB08, DB09, DB10, SD08
- Key: anomalies — dirty read, non-repeatable read, phantom, **lost update**, **write skew**, read
  skew. Levels: READ UNCOMMITTED / READ COMMITTED (Postgres default) / REPEATABLE READ (MySQL
  default; in Postgres it is snapshot isolation, no phantoms) / SERIALIZABLE (Postgres SSI with
  serialization failures you must retry). MySQL RR ≠ Postgres RR.
- Write skew canonical example: two doctors both go off-call because each sees the other on call.
- Q: "Explain write skew and why REPEATABLE READ doesn't prevent it."
- Q: "Your app runs SERIALIZABLE and randomly throws 40001. What must the app do?" (retry loop with
  backoff; keep transactions short).
- Q: "Two concurrent `balance = balance - 10` — walk through it at each isolation level."

#### DB08 · MVCC internals
`A` · Requires: DB07 · Unlocks: DB11, DB12, DB18, DB39
- Key: Postgres: every row version has xmin/xmax, snapshots decide visibility, readers never block
  writers; dead tuples → **bloat** → VACUUM (autovacuum tuning, `vacuum freeze`, transaction-ID
  wraparound); HOT updates and why an indexed column update is more expensive; MySQL InnoDB keeps
  old versions in undo logs (history list length) instead.
- Q: "A table is 4x its data size and queries got slow. Cause?" (bloat from long-running
  transactions/idle-in-transaction blocking vacuum).
- Q: "Why does `UPDATE` in Postgres write a whole new row version, and what does that do to indexes?"

#### DB09 · Locking
`A` · Requires: DB05, DB06, DB07 · Unlocks: DB10, DB22, C11
- Key: row vs table locks; shared/exclusive; `SELECT ... FOR UPDATE [SKIP LOCKED | NOWAIT]`;
  InnoDB gap/next-key locks (and why they cause deadlocks under RR); lock queues — a weak lock
  request behind a strong one blocks everyone; deadlock detection and consistent lock ordering;
  advisory locks; `lock_timeout` and `statement_timeout` as mandatory safety.
- Q: "Implement a job queue in Postgres." (`FOR UPDATE SKIP LOCKED` + status + visibility timeout).
- Q: "You get random deadlocks on a two-row update. Fix?" (order updates by a deterministic key).
- Q: "Why did `ALTER TABLE ADD COLUMN` take the site down?" (ACCESS EXCLUSIVE waits behind a long
  query and every new query queues behind it — use `lock_timeout` + retry).

#### DB10 · Optimistic vs pessimistic concurrency
`I` · Requires: DB07, DB09 · Unlocks: DB38, SD10
- Key: optimistic = version column / `WHERE version = ?` and 0 rows updated means conflict; good for
  low contention and stateless APIs (also the HTTP ETag/If-Match pattern). Pessimistic = row lock,
  good for high contention and inventory. Compare-and-set as the primitive.
- Q: "Two users edit the same record in a UI form. Design the conflict handling."

#### DB11 · Physical storage
`A` · Requires: DB01, DB08 · Unlocks: DB13, DB29, DB32
- Key: pages (8KB), tuples, row vs column layout, fill factor, heap vs **clustered index** (InnoDB
  stores rows in the PK B-tree; secondary indexes store the PK → wide PK bloats every index and adds
  a second lookup); Postgres heap + ctid + TOAST for large values; alignment/column order padding.
- Q: "Why is a random UUID PK worse in InnoDB than in a Postgres heap table?" (page splits in a
  clustered index; still hurts both via index locality and WAL volume).

#### DB12 · WAL, durability, crash recovery
`A` · Requires: DB06, DB08 · Unlocks: DB23, DB36, Q13
- Key: write-ahead logging (log before data pages), fsync, group commit, checkpoints and their I/O
  spikes, full-page writes, `synchronous_commit` trade-off, fsync lies at the disk/cloud layer, redo
  vs undo, crash recovery, PITR from base backup + WAL.
- Q: "How does the database guarantee durability without fsyncing every data page?"
- Q: "What do you lose by setting `synchronous_commit = off`?" (up to a few hundred ms of committed
  transactions on crash — but no corruption; different from `fsync = off`, which risks corruption).

#### DB13 · B+Tree indexes
`I` · Requires: DB04, DB11 · Unlocks: DB14, DB15, DB16, DB17, DB18, DB29
- Key: structure (sorted, balanced, O(log n), leaf-linked for range scans), why the height is ~3-4
  for millions of rows, what an index costs on write, why indexes support `=`, ranges, prefix
  `LIKE 'abc%'`, `ORDER BY` and `MIN/MAX`; index bloat and rebuilds (`REINDEX CONCURRENTLY`).
- Q: "How many index pages are read for a point lookup on a 100M-row table?"
- Q: "Why does `LIKE '%abc'` not use a B-tree, and what would?" (trigram / reversed index).

#### DB14 · Other index types
`A` · Requires: DB13 · Unlocks: DB31, DB34
- Key: hash (equality only), GIN (JSONB, arrays, full-text — fast reads, expensive writes, pending
  list), GiST (geometry, ranges, KNN), BRIN (huge append-only tables with physical correlation —
  tiny index), bitmap index scans and bitmap heap scans, covering vs filtering, MySQL adaptive hash
  index, vector indexes (HNSW/IVFFlat) for embeddings.
- Q: "1TB append-only events table, queries by time range. Which index?" (BRIN on the timestamp,
  plus partitioning).

#### DB15 · Composite indexes, selectivity, cardinality
`A` · Requires: DB13 · Unlocks: DB16, DB19, DB20
- Key: **leftmost prefix rule**; column order = equality columns first, then range, then sort
  (E-R-S); an index on (a,b) serves `a` and `a,b` but not `b` alone; selectivity/cardinality drives
  whether the index is used at all; index-and-then-filter vs index-only; too many indexes slow
  writes and confuse the planner.
- Q: "Given `WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY created_at DESC`, design
  the index." (`(tenant_id, status, created_at DESC)`).
- Q: "Why is the planner ignoring your index?" (low selectivity, type mismatch/casting, function on
  the column, stale stats, small table, `LIMIT` interplay).

#### DB16 · Covering indexes & index-only scans
`A` · Requires: DB15 · Unlocks: DB20
- Key: all needed columns in the index → no heap access; Postgres `INCLUDE` columns; visibility map
  requirement for index-only scans in Postgres (vacuum matters!); InnoDB secondary index always
  covers the PK.
- Q: "Your index-only scan still hits the heap. Why?" (visibility map not set — table needs vacuum).

#### DB17 · Partial & expression indexes
`A` · Requires: DB13 · Unlocks: DB20
- Key: `WHERE deleted_at IS NULL`, `WHERE status = 'pending'` for a hot small subset; expression
  indexes (`lower(email)`, `(payload->>'id')`) must match the query expression exactly; unique
  partial indexes for "one active row per user".
- Q: "Enforce 'only one active subscription per user' in the database." (partial unique index).

#### DB18 · The query planner
`X` · Requires: DB08, DB13 · Unlocks: DB19, DB20
- Key: rule-based vs cost-based; statistics (n_distinct, histograms, MCV, correlation), `ANALYZE`,
  extended statistics for correlated columns; cardinality estimation errors compound through joins;
  cost constants (`random_page_cost` for SSDs), `work_mem` and spill to disk; join order search /
  genetic optimiser; plan caching and generic vs custom plans (the prepared-statement cliff).
- Q: "The planner estimates 1 row but gets 2 million. Consequences and fixes?" (nested loop chosen →
  disaster; fix with ANALYZE, extended stats, rewriting, or materialising).
- Q: "A query is fast in psql and slow from the app." (generic plan from a prepared statement,
  different `search_path`/settings, parameter sniffing in SQL Server/MySQL terms).

#### DB19 · Reading EXPLAIN / ANALYZE
`A` · Requires: DB02, DB15, DB18 · Unlocks: DB20, DB39
- Key: `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`; estimated vs actual rows and loops (actual time is
  per-loop!); scan types (seq, index, index-only, bitmap); join algorithms — **nested loop** (small
  outer + indexed inner), **hash join** (big unsorted, needs work_mem), **merge join** (both sorted);
  sort/hash spill to disk; rows removed by filter; the first expensive node from the inside out.
- Q: "Walk me through this plan and tell me the fix." (be ready to do it live)
- Q: "When is a sequential scan the right plan?" (low selectivity, small table, no useful index).

#### DB20 · Query tuning patterns
`A` · Requires: DB03, DB16, DB17, DB19 · Unlocks: DB21, DB39, SD05
- Key: **N+1** (ORM lazy loading → join/prefetch/dataloader); sargability (no functions on indexed
  columns, no implicit casts, no `OR` across columns → `UNION ALL`); **keyset/seek pagination**
  instead of `OFFSET` (which scans and discards); `COUNT(*)` on huge tables (estimate from stats or
  maintain a counter); bulk insert (COPY, multi-row VALUES, batching); avoiding `SELECT *`;
  materialised views and refresh strategy; `LIMIT` + `ORDER BY` index alignment.
- Q: "Page 5000 of a listing takes 8 seconds. Fix it." (keyset pagination on an indexed sort key).
- Q: "Give five reasons a query that was fast last month is slow now."

#### DB21 · Connections & pooling
`A` · Requires: DB20 · Unlocks: DB39, F24, C14
- Key: a Postgres connection is a process (~5-10MB) — hundreds of connections thrash; pool sizing
  ≈ cores x 2 + effective spindles, not "one per request"; PgBouncer modes (session/transaction/
  statement) and what transaction pooling forbids (session state, prepared statements pre-PG14,
  advisory locks, `SET`); pool exhaustion under slow queries; serverless/lambda connection storms;
  timeouts (`statement_timeout`, `idle_in_transaction_session_timeout`) as non-negotiable settings.
- Q: "200 Nest pods x 10 pool size against one Postgres. What happens and what do you do?"
- Q: "Why did enabling PgBouncer transaction pooling break your ORM?"

#### DB22 · Zero-downtime schema migration
`A` · Requires: DB05, DB09, M27 · Unlocks: DB36, SD14, O09
- Key: **expand → migrate → contract**; add nullable column (cheap in modern PG/MySQL), backfill in
  batches with throttling, add constraint `NOT VALID` then `VALIDATE`, `CREATE INDEX CONCURRENTLY`,
  rename via new column + dual write, never a blocking `ALTER` on a hot table; `lock_timeout` +
  retry; MySQL online DDL / gh-ost / pt-online-schema-change; migrations must be compatible with the
  currently-running app version (both directions during a rolling deploy).
- Q: "Rename `user.name` to `full_name` with zero downtime and 200M rows. Give the exact steps."
- Q: "Add a NOT NULL column with a default to a 500M row table."

#### DB23 · Replication & read scaling
`A` · Requires: DB12, M12, M33 · Unlocks: DB24, DB35, DB36, SD05
- Key: physical/streaming vs logical replication (and what logical enables: selective, cross-version,
  CDC); sync vs async, `synchronous_standby_names`, quorum commit; **replica lag** and its causes;
  routing reads to replicas breaks read-your-writes → route post-write reads to the primary or wait
  on LSN; failover tooling (Patroni), promoting, re-basing the old primary.
- Q: "You moved reads to a replica and users report missing data. Explain and fix."
- Q: "When does a read replica *not* help?" (write-bound workloads; the replica applies the same
  write volume single-threadedly).

#### DB24 · Partitioning
`A` · Requires: DB23 · Unlocks: DB25, DB32
- Key: declarative range/list/hash partitioning; partition pruning (needs the key in the predicate);
  benefits are cheap `DROP PARTITION` retention, smaller indexes, better vacuum; costs are planning
  overhead, no global unique index without the key, cross-partition queries, and the fact that
  partitioning is **not** sharding.
- Q: "Events table grows 100M rows/month. Design retention." (monthly range partitions + drop).

#### DB25 · Sharding
`X` · Requires: DB24, M22, M32 · Unlocks: DB35, M29, SD06
- Key: shard key choice is irreversible in practice — pick for even distribution *and* query locality
  (tenant_id is usually right); hot shards/celebrity problem; cross-shard queries, joins and
  transactions (scatter-gather, fan-out cost = slowest shard); resharding (double-write, directory/
  lookup service, virtual buckets); globally unique IDs (Snowflake); Vitess/Citus.
- Q: "When do you shard, and what do you do before sharding?" (index/query tuning, caching, read
  replicas, vertical scaling, archiving, functional split — sharding is last).
- Q: "You sharded by user_id; now you must query by email. Options?"

#### DB26 · NoSQL taxonomy & when to use it
`I` · Requires: DB01 · Unlocks: DB27, DB28, DB31, DB32
- Key: key-value, document, wide-column, graph, time-series, vector; NoSQL trades ad-hoc query
  flexibility and joins for horizontal scale and flexible schema; "schemaless" means the schema
  lives in your application code; polyglot persistence and its operational cost.
- Q: "Choose a store for a social graph / a shopping cart / an audit log / a product catalogue, and
  justify each."
- Q: "Default answer for a new service?" (Postgres, until you can prove otherwise).

#### DB27 · Document databases (MongoDB)
`A` · Requires: DB26 · Unlocks: DB29
- Key: embed vs reference (embed for "always read together" and bounded size; 16MB doc limit,
  unbounded array anti-pattern); indexes incl. compound and the ESR rule; write concern
  (`w:1/majority`, `j:true`) and read concern/read preference; replica sets and elections; sharded
  clusters and chunk balancing; multi-document transactions exist but are a smell if you need them
  often; aggregation pipeline.
- Q: "Model comments on a post in Mongo." (embed recent, reference the rest — bounded arrays).
- Q: "What does `w:1` actually risk?" (acknowledged by primary only — a failover can lose it).

#### DB28 · Wide-column & key-value (Cassandra, DynamoDB)
`X` · Requires: DB26, DB32 · Unlocks: DB29, DB35
- Key: **query-first modelling** — one table per access pattern, duplicate data freely; partition key
  (distribution) + sort/clustering key (in-partition order); no joins, no ad-hoc filters; DynamoDB
  single-table design, GSIs/LSIs, hot partitions and adaptive capacity, item size limits, conditional
  writes for optimistic concurrency; Cassandra tunable consistency (R + W > N), lightweight
  transactions (Paxos, slow), tombstones and the delete problem.
- Q: "Model an activity feed in DynamoDB and give the exact keys."
- Q: "Why is `scan` a red flag?" / "How do you paginate and how do you avoid a hot partition?"

#### DB29 · LSM trees vs B-Trees
`X` · Requires: DB11, DB13, DB27, DB28 · Unlocks: DB32
- Key: LSM = memtable → immutable SSTables → compaction; great write throughput and compression,
  costs read amplification (bloom filters and block caches mitigate), space amplification, and
  compaction stalls; levelled vs size-tiered compaction; B-tree = in-place writes, stable read
  latency, write amplification via WAL + full-page writes. RocksDB is the common engine.
- Q: "Why is Cassandra faster at writes than Postgres, and what do you pay for it?"
- Q: "Explain read/write/space amplification and which one each design optimises."

#### DB30 · Redis as a data store
`I` · Requires: DB26 · Unlocks: Q05
- Key: in-memory, single-threaded command execution, data structures as the API, persistence
  (RDB/AOF) making it *durable-ish* — never the system of record for money. Full treatment in the
  Caching field (Q05-Q08).
- Q: "When is Redis an acceptable source of truth?" (rarely — ephemeral state, rate limits, locks
  with fencing, sessions with acceptable loss).

#### DB31 · Search engines (Elasticsearch / OpenSearch)
`A` · Requires: DB14, DB26 · Unlocks: SD12
- Key: inverted index, analyzers (tokenise, lowercase, stem, synonyms), term vs match queries,
  relevance (BM25), near-real-time refresh interval, shards/replicas and why you cannot change the
  primary shard count, deep pagination (`search_after` not `from/size`), ES as a *derived* store fed
  by CDC — never the source of truth, reindex/alias switch for mapping changes.
- Q: "Product search with filters and typo tolerance. Postgres FTS or Elasticsearch? Where's the
  line?"
- Q: "How do you keep ES in sync with Postgres?" (outbox/CDC + versioned docs + periodic reconcile).

#### DB32 · Analytics, columnar & OLAP
`A` · Requires: DB03, DB11, DB24, DB26, DB29 · Unlocks: SD06
- Key: OLTP vs OLAP; columnar storage + compression + vectorised execution (ClickHouse, BigQuery,
  Snowflake, DuckDB, Parquet); star/snowflake schema, facts and dimensions; why you never run
  analytics on the OLTP primary; ETL/ELT, CDC into the warehouse, materialised aggregates;
  approximate aggregates.
- Q: "The data team is killing your production DB with dashboards. Options in order of preference?"

#### DB33 · Probabilistic & specialised structures
`A` · Requires: DB13 · Unlocks: —
- Key: Bloom filter (no false negatives, tunable false positives — used in LSM reads, cache
  admission), HyperLogLog (unique counts in KB), count-min sketch (heavy hitters), skip lists
  (Redis sorted sets), tries, inverted index, merkle trees (anti-entropy, replicas).
- Q: "Count unique daily visitors across 200M events with a fixed memory budget."
- Q: "Where would a Bloom filter save you a database round-trip?"

#### DB34 · Full-text search in Postgres
`I` · Requires: DB14 · Unlocks: DB31
- Key: `tsvector`/`tsquery`, GIN index, stored generated column vs expression index, `ts_rank`,
  `pg_trgm` for fuzzy/`ILIKE %x%`, unaccent, language configs and their limits.
- Q: "When is Postgres FTS enough?" (single language, modest corpus, no complex relevance tuning,
  one less system to operate).

#### DB35 · Distributed SQL / NewSQL
`X` · Requires: DB23, DB25, DB28, M20 · Unlocks: —
- Key: Spanner (TrueTime + commit wait → external consistency), CockroachDB (Raft per range, range
  splits, clock skew limits), YugabyteDB, TiDB, Vitess (sharded MySQL), Aurora (storage-compute
  separation, log-as-the-database, 6-way quorum); serverless Postgres (Neon) with
  storage/compute separation and branching.
- Q: "Why is a distributed SQL DB slower per transaction than a single Postgres, and when is the
  trade worth it?" (consensus round trips per write vs horizontal scale and multi-region survival).

#### DB36 · Backup, PITR, disaster recovery
`I` · Requires: DB12, DB22, DB23 · Unlocks: O20
- Key: logical (`pg_dump`) vs physical base backups + WAL archiving; PITR to a timestamp/LSN;
  **an untested backup is not a backup** — restore drills and measured restore time; RPO vs RTO;
  replicas are not backups (a `DELETE` replicates instantly); delayed replicas; backup encryption and
  retention; restoring a single table from a 2TB cluster.
- Q: "A bad migration deleted 3M rows 40 minutes ago. Walk me through recovery."

#### DB37 · Database security & privacy
`I` · Requires: DB05, S07 · Unlocks: S12
- Key: least-privilege roles (the app should not own its schema), separate migration user, row-level
  security for multi-tenant, encryption at rest (TDE/volume) vs column-level/application-level
  encryption, key management, PII classification, GDPR erasure vs immutable backups (crypto-shred),
  audit logging, redaction of queries in logs.
- Q: "Your app connects as the table owner. Why is that bad and what is the alternative?"

#### DB38 · Money, ledgers and correctness
`A` · Requires: DB04, DB10 · Unlocks: SD10
- Key: integer minor units or `numeric` — never float; double-entry ledger (append-only entries,
  balance = sum or a maintained snapshot); idempotency keys on every mutation; no `UPDATE balance`
  without a lock or CAS; currency and rounding rules; reconciliation jobs against the provider;
  auditability over convenience.
- Q: "Design the schema for a wallet with transfers. Prove no money can be created or destroyed."
- Q: "Balance is now a slow `SUM()` over 50M rows. What do you do?" (periodic balance snapshots +
  delta, with an invariant check job).

#### DB39 · Database observability & production debugging
`I` · Requires: DB08, DB19, DB20, DB21 · Unlocks: O14, O17
- Key: `pg_stat_statements` (total time, not just slow-query-log outliers), `pg_stat_activity` +
  wait events, `pg_locks` blocking trees, autovacuum and bloat monitoring, replication lag, cache hit
  ratio (and why it is a weak metric), connection saturation, slow query log in MySQL, `performance_
  schema`. The four questions: what is running, what is blocked, what is bloated, what is lagging.
- Q: "CPU is 100% on the primary at 2am. Walk through your first five minutes."

#### DB40 · ORM internals & escape hatches
`A` · Requires: DB20, DB21, F09 · Unlocks: F09, F21, F22
- Key: identity map, unit of work, dirty checking, lazy vs eager loading, the N+1 it creates;
  TypeORM/Prisma (Prisma's query engine, relation loading and its `IN` batching, no real lazy
  loading), Django ORM (queryset laziness, `select_related` join vs `prefetch_related` second query,
  `only`/`defer`, `bulk_create`, `iterator()`), Hibernate (session/persistence context, flush order,
  `LazyInitializationException`, `@Transactional` proxy self-invocation, first/second-level cache).
  Know when to drop to raw SQL and how to keep it safe (parameterised, typed).
- Q: "Show me an N+1 your ORM generated and three ways to fix it."
- Q: "What does the ORM do at flush time and why does ordering matter for FK constraints?"

---

## Topological order (study waves)

```
Wave 0  DB01
Wave 1  DB02  DB05  DB26
Wave 2  DB03  DB04  DB06  DB27  DB30  DB34
Wave 3  DB07  DB31(after DB14)  DB33
Wave 4  DB08  DB09  DB12
Wave 5  DB10  DB11  DB13  DB39(partial)
Wave 6  DB14  DB15  DB17  DB18  DB22  DB23  DB38
Wave 7  DB16  DB19  DB24  DB28  DB29  DB36  DB37
Wave 8  DB20  DB25  DB32
Wave 9  DB21  DB35
Wave 10 DB39  DB40
```

Cross-field parents: `M12` consistency, `M20` consensus, `M22` data ownership, `M27` deploys,
`M32` partitioning, `M33` replication, `S07` injection, `F09` persistence layer.

**Highest-yield 12 for a senior interview:** DB07, DB08, DB09, DB13, DB15, DB19, DB20, DB21, DB22,
DB23, DB25, DB40. Be able to draw a B+tree, read an EXPLAIN out loud, and describe write skew from
memory.
