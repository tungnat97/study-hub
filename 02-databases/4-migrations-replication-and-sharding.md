[← back to the field index](README.md)

# Databases · Part 4 — Migrations, Replication, Partitioning & Sharding

Nodes `DB22`–`DB25`.

---

## DB22 · Zero-downtime schema migration

`Advanced` · Requires: `DB05`, `DB09`, `M27` · Unlocks: `DB36`, `SD14`, `O09`

### Preface

Changing a schema on a live database is the most dangerous routine operation in backend work. It
combines two hazards: some statements take locks that stop all traffic, and during a rolling deploy
the old and new versions of your code both run against the new schema.

The safe method is always the same: **expand, migrate, contract** — add the new thing, move to it,
then remove the old thing, as separate deployments.

### Details

#### 1. Which operations are safe

**Theory.** In modern Postgres, some `ALTER TABLE` operations only change metadata and are instant;
others rewrite the whole table while holding an `ACCESS EXCLUSIVE` lock.

**Example.** Rough guide for Postgres 12+:

| Operation | Cost |
|---|---|
| `ADD COLUMN` nullable, no default | instant (metadata only) |
| `ADD COLUMN` with a constant default | instant since PG11 (stored in catalog) |
| `ADD COLUMN` with a **volatile** default (`now()`, `gen_random_uuid()`) | full table rewrite |
| `DROP COLUMN` | instant (marked dropped, space reclaimed by vacuum) |
| `ALTER COLUMN TYPE` | usually full rewrite; `varchar(50)`→`varchar(100)` is free |
| `SET NOT NULL` | full scan (unless a matching `CHECK ... NOT VALID` was validated first) |
| `ADD CONSTRAINT ... CHECK` | full scan, unless `NOT VALID` then `VALIDATE` |
| `CREATE INDEX` | blocks writes; `CREATE INDEX CONCURRENTLY` does not |

**Advanced.** Two refinements worth knowing. `SET NOT NULL` can be made cheap: add
`CHECK (col IS NOT NULL) NOT VALID`, run `VALIDATE CONSTRAINT` (which takes only a weak lock), then
`SET NOT NULL` — Postgres 12+ uses the validated constraint as proof and skips the scan. And
`CREATE INDEX CONCURRENTLY` cannot run inside a transaction, which means most migration frameworks
need an explicit escape (`atomic = False` in Django, `transaction: false` or raw SQL in TypeORM); it
also **can fail and leave an invalid index** behind, which you must drop and recreate.

#### 2. The lock timeout pattern

**Theory.** Even an instant operation must first acquire `ACCESS EXCLUSIVE`. If a long query holds
the table, your DDL waits — and every query behind it queues (`DB09`). The migration itself becomes
the outage.

**Example.** Always guard DDL:

```sql
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN note text;
```

If the lock is not available in 3 seconds, the statement fails and you retry — with a short sleep,
a few times. A failed migration attempt is a non-event; a 4-minute lock queue is an incident. Make
this the default in your migration tooling, not something each author remembers.

**Advanced.** For MySQL, the equivalents are `ALGORITHM=INPLACE, LOCK=NONE` where supported, and
external tools (`gh-ost`, `pt-online-schema-change`) that build a shadow table, copy rows in batches,
keep it in sync with triggers or the binlog, and swap names atomically. Those tools exist precisely
because MySQL historically rewrote tables under a lock; knowing why they exist is more useful than
memorising their flags.

#### 3. Backfilling

**Theory.** Adding a column is instant; **filling it** is not. A single `UPDATE` over 200 million
rows holds a long transaction, writes an enormous amount of WAL, blows up replication lag, and
cannot be resumed.

**Example.** A resumable, throttled backfill:

```sql
-- repeat until no rows are updated
WITH batch AS (
  SELECT id FROM orders WHERE new_col IS NULL ORDER BY id LIMIT 1000 FOR UPDATE SKIP LOCKED
)
UPDATE orders o SET new_col = compute(o) FROM batch b WHERE o.id = b.id;
```

Run it in a loop with a short pause, monitoring replication lag and pausing when it rises. Keep a
progress marker so it can resume after an interruption. Run it as a separate job, not inside the
deployment.

**Advanced.** Batching by primary key range is usually better than `LIMIT` with an offset, because
the latter scans further each iteration. And remember that each batch creates dead row versions
(`DB08`) — backfilling a large table can double its size until vacuum catches up, so check disk
headroom before starting. Pace the job against replication lag, not against wall-clock time.

#### 4. The full rename sequence

**Theory.** Renaming is the canonical example because it combines every concern.

**Example.** Renaming `users.name` to `users.full_name` on 200 million rows, with zero downtime:
1. **Migration A**: `ADD COLUMN full_name text` (instant).
2. **Deploy 1**: application writes **both** columns, reads `name`.
3. **Backfill**: copy `name` → `full_name` in batches; verify counts match.
4. **Deploy 2**: application reads `full_name`, still writes both.
5. **Migration B**: add a `NOT NULL` constraint via `NOT VALID` + `VALIDATE` if required.
6. **Deploy 3**: application stops writing `name`.
7. **Migration C**: `DROP COLUMN name` — days later, once you are certain.

**Advanced.** Steps 2 and 3 exist specifically because both versions of the application run at once
during a rolling deploy (`M24`). A database trigger keeping the two columns in sync is an
alternative to dual writes and is sometimes simpler, especially when several applications write the
table — but it is hidden behaviour, so document it and remove it in step 7. Never compress these
seven steps into fewer deployments to save time; that is exactly how the outage happens.

### Interview questions

- "Rename `user.name` to `full_name` with zero downtime and 200 million rows. Give the exact steps."
- "Add a `NOT NULL` column with a default to a 500-million-row table."
- "Why do you set `lock_timeout` before a migration?"
- "Why can't `CREATE INDEX CONCURRENTLY` run inside a transaction?"

---

## DB23 · Replication and read scaling

`Advanced` · Requires: `DB12`, `M12`, `M33` · Unlocks: `DB24`, `DB35`, `DB36`, `SD05`

### Preface

A replica is a second copy of the database kept up to date from the primary's write-ahead log. It
gives you a standby for failover, a target for backups, and capacity for read queries.

The essential caveats: a replica is always slightly behind, and it does not help write-heavy
workloads at all — it must apply every write the primary does.

### Details

#### 1. Physical versus logical replication

**Theory.** **Physical (streaming)** replication ships the raw WAL; the replica is a byte-identical
copy of the whole cluster, and it can only be read, not written. **Logical** replication decodes the
WAL into row-level changes (insert, update, delete) and applies them; it can replicate selected
tables, across different major versions, and into a database that also has its own writable tables.

**Example.** Use physical replication for high availability and read replicas — it is simpler,
lower overhead and lower lag. Use logical replication for a major-version upgrade with minimal
downtime (replicate old → new, then switch over), for sending a subset of tables to another system,
or as the basis of change data capture (`Q18`).

**Advanced.** Logical replication has real limitations to name: it does not replicate DDL (schema
changes must be applied separately on both sides, in the right order), sequences are not advanced on
the subscriber (so a failover needs them fixed manually), and tables need a replica identity
(a primary key, usually) for updates and deletes to be replicated. These are exactly the details
that bite during a version upgrade.

#### 2. Synchronous configuration

**Theory.** `synchronous_commit` plus `synchronous_standby_names` determine whether a commit waits
for a replica. The levels are `off`, `local` (local flush only), `remote_write` (the replica
received it), `on` (the replica flushed it to disk), and `remote_apply` (the replica has applied it
and it is visible to readers there).

**Example.** `synchronous_standby_names = 'ANY 1 (rep_a, rep_b)'` waits for whichever of two
replicas confirms first. You get "the write exists on two machines" without a single slow replica
stalling all writes — a much better configuration than naming one mandatory standby, which couples
your availability to it.

**Advanced.** `remote_apply` is the only setting that guarantees a read on the replica immediately
after a write sees that write — it solves read-your-writes at the cost of the slowest replica's
apply time on every commit. Most systems use `on` and solve read-your-writes in the application
instead (see below), because paying that latency on every write to fix a problem affecting a few
reads is a poor trade.

#### 3. Replica lag and read routing

**Theory.** Lag is the delay between a commit on the primary and its visibility on the replica.
Under normal load it is milliseconds; under bulk writes or with long queries on the replica it can
reach minutes. Routing reads to replicas therefore breaks read-your-writes (`M12`).

**Example.** Practical routing rules for an application:
- Writes and any read in the same transaction → primary.
- Reads within N seconds of that user's last write → primary (track it in the session or a cache).
- Everything else → replica.
- Monitor lag continuously and remove a replica from rotation when it exceeds a threshold.

**Advanced.** Postgres gives you a precise mechanism instead of a time heuristic: capture the write's
LSN with `pg_current_wal_lsn()`, pass it back to the client, and on the next read check
`pg_last_wal_replay_lsn()` on the replica — route to the primary if it has not caught up. This is
exact rather than approximate, and it is the answer that distinguishes someone who has actually
implemented replica routing.

**Causes of lag worth naming**: a single-threaded replay process unable to keep up with a highly
parallel write workload; long-running queries on the replica conflicting with replay (Postgres will
either delay replay or cancel the query, controlled by `max_standby_streaming_delay` and
`hot_standby_feedback`); bulk operations; and network saturation between regions.

#### 4. Failover

**Theory.** When the primary dies, a replica must be promoted. This involves choosing which replica
(the most caught-up one), fencing the old primary so it cannot accept writes, repointing the
application, and eventually rebuilding the old primary as a replica.

**Example.** Patroni with etcd is the standard self-managed setup: it holds a leader lease in etcd,
runs elections, promotes a standby, and updates a virtual IP or a proxy so the application follows.
Managed services (RDS Multi-AZ, Cloud SQL) do this for you with a DNS or endpoint switch, typically
in 30-120 seconds.

**Advanced.** Two things to be able to say. With **asynchronous** replication, a failover loses any
writes that had not replicated — acknowledged to the user, then gone. You must decide whether that
is acceptable, and if not, run at least one synchronous standby. And the old primary must be
**fenced** before promotion, or it may keep serving writes to clients that still reach it —
split-brain (`M33`). After the failover the old primary usually cannot simply rejoin, because it may
contain writes the new primary never saw; `pg_rewind` handles this, or you rebuild from a base
backup.

### Interview questions

- "You moved reads to a replica and users report missing data. Explain and fix."
- "When does a read replica *not* help?"
- "Physical or logical replication for a major version upgrade? Why?"
- "An async replica is promoted. What did you lose?"

---

## DB24 · Partitioning

`Advanced` · Requires: `DB23` · Unlocks: `DB25`, `DB32`

### Preface

Partitioning splits one logical table into several physical tables inside the same database. Queries
still address one table name; the database routes them to the relevant partitions.

The most valuable benefit is not query speed — it is **cheap deletion**: dropping last year's data
becomes an instant `DROP TABLE` instead of a `DELETE` of 500 million rows.

Partitioning is not sharding. Everything is still on one server.

### Details

#### 1. The three schemes

**Theory.** **Range** — by a continuous value, typically a date. **List** — by a discrete value, such
as region or tenant. **Hash** — by a hash of the key, to spread rows evenly with no natural
grouping.

**Example.** Range partitioning by month is the standard shape for events and logs:

```sql
CREATE TABLE events (id bigint, created_at timestamptz, ...) PARTITION BY RANGE (created_at);
CREATE TABLE events_2026_09 PARTITION OF events
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
```

Retention becomes `DROP TABLE events_2026_03` — instant, and it reclaims the space immediately,
unlike a `DELETE` which leaves dead rows for vacuum (`DB08`).

**Advanced.** Partitions must be created **before** the data arrives, so you need automation — a
scheduled job creating next month's partition, or an extension such as `pg_partman`. A missing
partition means inserts fail (or land in a default partition, which then grows unbounded and quietly
defeats the design). This operational requirement is the most common reason partitioning projects go
wrong.

#### 2. Partition pruning

**Theory.** The optimisation that makes partitioning fast: if the query's `WHERE` clause constrains
the partition key, the planner skips irrelevant partitions entirely. Without the key in the
predicate, every partition is scanned — usually slower than an unpartitioned table.

**Example.** `WHERE created_at >= '2026-09-01'` touches one partition. `WHERE user_id = 42` with no
date filter touches **all** of them, and does so through many smaller indexes rather than one, which
is worse than not partitioning. So partitioning helps only when your dominant query pattern includes
the partition key.

**Advanced.** Pruning happens at plan time when the values are constants, and at execution time for
parameters (`runtime partition pruning`, Postgres 11+) — visible in the plan as
"Subplans Removed: N". Older versions could not prune parameterised queries at all, which is the
origin of the advice to inline date literals. Check the plan rather than assuming.

#### 3. What partitioning costs

**Theory.** More planning overhead with many partitions; **no global unique index** unless the
partition key is part of it; foreign keys referencing a partitioned table have limitations; and
cross-partition queries do more work.

**Example.** The unique-index restriction is the one that constrains design most: on a table
partitioned by `created_at`, you cannot have a plain `UNIQUE (email)` — only
`UNIQUE (email, created_at)`, which does not enforce what you wanted. If you need global uniqueness,
either the partition key must be part of the key, or you enforce uniqueness elsewhere (a separate
small table with the constraint).

**Advanced.** Keep the partition count moderate — dozens to low hundreds, not thousands. Each
partition is a table with its own indexes, statistics and vacuum work, and planning time grows with
the count. Monthly partitions with a few years of retention is comfortable; hourly partitions for
years is not. If you need that many, you probably need a purpose-built time-series or columnar store
(`DB32`).

#### 4. When to partition

**Theory.** Partition when you have a clear retention policy to enforce, when the table is large
enough that maintenance (vacuum, index rebuilds, backups) is painful, and when the dominant query
pattern includes the partition key. Do not partition merely because a table is "big".

**Example.** Good: an events table with 90-day retention, queried by time. Bad: a users table of
50 million rows queried by id — a B-tree handles that perfectly well, and partitioning adds
complexity with no benefit.

**Advanced.** Partitioning an existing large table requires a migration, because you cannot convert
in place: create the partitioned table, copy data in batches, then swap names — the same shape as
any large data migration (`DB22`, `SD14`). Plan it before the table is huge; retrofitting at 2TB is
substantially harder than at 200GB.

### Interview questions

- "Events table grows 100 million rows a month. Design retention."
- "Why might partitioning make queries slower?"
- "Can you have a unique constraint on a partitioned table?"
- "What is the difference between partitioning and sharding?"

---

## DB25 · Sharding

`Expert` · Requires: `DB24`, `M22`, `M32` · Unlocks: `DB35`, `M29`, `SD06`

### Preface

Sharding splits data across **separate database servers**, each holding a subset. It is how you
scale writes beyond what one machine can do.

It is also the least reversible decision in backend engineering. Cross-shard queries, transactions
and joins become hard or impossible, and changing the shard key later means moving all your data.

The senior answer to "should we shard?" is almost always "not yet, and here is what I would do
first".

### Details

#### 1. Everything to do before sharding

**Theory.** Sharding is the last resort. The cheaper options usually buy years.

**Example.** The ladder, in order:
1. Fix the queries — indexes, N+1, pagination (`DB20`). Frequently a 10x win for a week's work.
2. Connection pooling (`DB21`).
3. Cache the hot reads (`Q02`).
4. Read replicas for read-heavy load (`DB23`).
5. Vertical scaling — modern machines take terabytes of RAM and hundreds of cores; this is cheap
   compared with an engineering year.
6. Archive or partition old data so the working set stays small (`DB24`).
7. Functional split — move a high-volume table to its own database (`M22`).
8. **Then** shard.

**Advanced.** Quantify it in an interview: a single well-tuned Postgres on good hardware handles
tens of thousands of transactions per second and terabytes of data. If you are not near that, the
problem is the schema or the queries, not the machine. Saying this confidently, with the ladder, is
a much stronger answer than jumping to a sharding design.

#### 2. Choosing the shard key

**Theory.** The shard key determines which server holds a row. It must spread data and load evenly,
and it must be present in almost every query — otherwise every query becomes a scatter-gather across
all shards.

**Example.** For a B2B SaaS, `tenant_id` is usually right: queries are almost always
tenant-scoped, so they hit one shard, and a tenant's data stays together so joins within a tenant
still work. For consumer social, `user_id` is typical. A poor choice: sharding by `created_at`,
which sends all new writes to one shard — the hot-shard problem in its purest form.

**Advanced.** The key must also avoid extreme skew. If one tenant is 40% of your data, hashing
`tenant_id` puts 40% of the load on one shard and nothing you do to the other shards helps. Common
handling: give the largest tenants dedicated shards (routing by an explicit lookup table rather than
pure hashing), which also lets you offer them isolation as a product feature (`M29`).

#### 3. Routing: hash, range or directory

**Theory.** **Hash** — `shard = hash(key) % N`, or consistent hashing to make resizing cheaper
(`M32`). Even distribution, no range queries. **Range** — shard 1 holds A-F. Range queries work,
skew is likely. **Directory** — a lookup table maps key → shard. Maximum flexibility, including
per-tenant placement and easy migration, at the cost of an extra lookup and a component that must be
highly available and cached.

**Example.** The directory approach is what most mature systems converge on, because it lets you
move a single tenant to a different shard without rehashing anything: update one row in the
directory. Keep the directory tiny, cache it in every application instance, and version it so
changes propagate safely.

**Advanced.** The standard refinement is **virtual buckets** (also called vnodes or slots): hash the
key into a fixed large number of buckets (say 1,024), and map buckets to physical shards in a small
table. Rebalancing means moving buckets, not rehashing keys, and the mapping table stays small. This
is how Redis Cluster (16,384 slots) and many custom systems work, and it gives you the flexibility of
a directory with the simplicity of hashing.

#### 4. What you lose

**Theory.** Cross-shard joins, cross-shard transactions, global unique constraints, global
auto-increment ids, and simple aggregate queries.

**Example.** The consequences and their workarounds:
- **Joins** — denormalise so related data shares a shard, or join in the application.
- **Transactions** — a saga across shards (`M14`), or design so a transaction never spans shards.
- **Unique ids** — Snowflake-style ids (timestamp + machine id + sequence) or UUIDv7, generated
  locally without coordination.
- **Aggregates** — query every shard and combine (scatter-gather), which costs the latency of the
  slowest shard; or maintain rollups in a separate store.
- **Pagination across shards** — genuinely hard; usually solved by restricting the query to one
  shard, or by a search index holding a global view.

**Advanced.** Resharding is the operation to plan for before you need it: dual-write to old and new
layouts, backfill, verify, switch reads, then switch writes — the `SD14` migration pattern at data
scale. Managed alternatives exist and are worth naming: Vitess (sharded MySQL, used by YouTube and
Slack), Citus (sharded Postgres), or a distributed SQL database that shards internally (`DB35`).
Choosing one of those over hand-rolled sharding is usually the correct engineering decision.

### Interview questions

- "When do you shard, and what do you do before sharding?"
- "You sharded by `user_id`; now you must query by email. Options?"
- "How do you generate unique ids across shards?"
- "One tenant is 40% of your data. How does that affect your shard key?"
