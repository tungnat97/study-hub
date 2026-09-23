[← back to the field index](README.md)

# Databases · Part 7 — Real-life production problems

The first six parts are the textbook. This part is the pager. Every question below is a scenario an
interviewer narrates — a real symptom, real numbers, a real stack — and the answer they are hoping
for is rarely the first one in the book. "Add an index" and "scale up the instance" are the answers
everyone gives; the senior answer names the mechanism, the counter-intuitive fix, and the thing you
would never do again once you had been burnt by it.

How to use it:

1. Read the **Pre-knowledge** once, properly. It is dense on purpose: internals, real defaults, the
   diagnostic queries and the tricks. Every question's direction line points back to a section here.
2. For each question, **answer out loud first** — symptoms, hypothesis, what you would look at, what
   you would change right now, what you would change permanently. Two minutes, as if in the room.
3. Only then read the `Direction` line. If your answer missed the insight, reread the section it
   cites and the node it names.

Levels run from the questions you will almost certainly be asked (Level 1) to the ones only a panel
with a staff engineer who has run a large fleet will ask (Level 10). Postgres is the default engine
unless the question says otherwise.

---

## Pre-knowledge

### 1. The incident mindset and the four-question triage

- Every database incident reduces to four questions (`DB39`): **what is running, what is blocked,
  what is bloated, what is lagging.** Have one query ready for each; do not improvise at 3am.
- **Stop the bleeding before diagnosing the root cause.** Cancel the runaway query, shed the batch
  job, flip the feature flag, fail reads over to the primary. Restoring service and understanding
  it are different jobs; say explicitly which one you are doing.
- **Change one thing at a time** and write down the time you changed it. Latency graphs are
  meaningless if three people restarted things in the same minute.
- **Suspect the most recent change first**: a deploy, a migration, a new index, a new cron, a
  version upgrade, an OS patch, a traffic shape change (a marketing email at 09:00). Most "the
  database suddenly got slow" incidents are a change that nobody connected to the database.
- **The database is usually the victim, not the cause.** A retry storm, a cache that expired, an
  N+1 in a new endpoint or a pool misconfiguration upstream shows up as database CPU. Look at
  `calls` in `pg_stat_statements` before `mean_exec_time`: a query that got 50× more frequent is
  a different incident from a query that got 50× slower.
- **Percentiles, not averages.** A p50 of 2 ms with a p99 of 4 s is a lock, a checkpoint, a
  flush of a GIN pending list, an autovacuum truncation, a GC pause, or a cold cache — not "slow
  queries" in general.
- Safety valves you should know by heart:

```sql
SELECT pg_cancel_backend(pid);      -- cancel the current query, keep the connection
SELECT pg_terminate_backend(pid);   -- kill the backend (rolls back its transaction)
ALTER ROLE app SET statement_timeout = '5s';          -- per-role guard rails
ALTER ROLE app SET idle_in_transaction_session_timeout = '30s';
ALTER ROLE app SET lock_timeout = '3s';
ALTER DATABASE app SET default_transaction_read_only = on;  -- emergency read-only mode
```

- Per-role settings are the underused lever: give the web role a 5 s `statement_timeout`, the
  reporting role 10 minutes, the migration role a 2 s `lock_timeout`. They apply on new
  connections only — existing pooled connections keep the old value until recycled. PG17 added
  `transaction_timeout` for a cap on the whole transaction.

### 2. The diagnostic toolbox

**Postgres.**

```sql
-- what is running and what it waits on
SELECT pid, usename, state, wait_event_type, wait_event, backend_xmin,
       now() - xact_start AS xact_age, now() - query_start AS query_age, left(query, 100)
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY xact_start;

-- who blocks whom
SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query, 80)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- where database time goes overall (needs the pg_stat_statements extension)
SELECT queryid, calls, total_exec_time, mean_exec_time, rows,
       shared_blks_hit, shared_blks_read, temp_blks_written, left(query, 80)
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;

-- bloat and vacuum health per table
SELECT relname, n_live_tup, n_dead_tup, n_tup_upd, n_tup_hot_upd,
       last_autovacuum, last_autoanalyze, autovacuum_count
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- replication
SELECT client_addr, state, write_lag, flush_lag, replay_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_behind
FROM pg_stat_replication;
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```

- `wait_event` tells you *why*: `Lock/relation` (DDL or a table lock), `Lock/transactionid` (row
  lock — waiting for another transaction to finish), `LWLock/BufferMapping` or `BufferContent`
  (hot pages), `LWLock/WALWrite` (WAL fsync contention), `IO/DataFileRead` (cache misses),
  `Client/ClientRead` while `idle in transaction` (the app is holding a transaction open and
  doing something else), `LWLock/MultiXact*` or `SubtransSLRU` waits (§3).
- Progress views: `pg_stat_progress_vacuum`, `pg_stat_progress_create_index`,
  `pg_stat_progress_analyze`, `pg_stat_progress_copy`, `pg_stat_progress_basebackup`.
  `pg_stat_io` (PG16+) shows who is doing I/O — backends writing dirty buffers themselves means
  the background writer and checkpointer cannot keep up.
- Logging that should be on in every production cluster: `log_min_duration_statement` (e.g.
  500 ms), `log_lock_waits = on` (logs waits longer than `deadlock_timeout`, 1 s),
  `log_autovacuum_min_duration` (e.g. 1 s), `log_temp_files = 0`, `log_checkpoints = on`
  (default on since PG15), and `auto_explain` with `log_min_duration` so the plan of the slow
  execution is captured *at the moment it was slow* — the plan you get later from `EXPLAIN` may
  be different (§6).
- **Statistics are per node.** `pg_stat_user_indexes.idx_scan = 0` on the primary does not mean
  an index is unused: the reporting replica may use it all day. Check every node before dropping.
  Stats also reset on crash recovery and on `pg_stat_reset()`; check `stats_reset`.
- `pg_stat_statements` normalises literals, so one `queryid` can hide both a 1 ms tenant and a
  20 s tenant. Averages lie; use `auto_explain` samples or log sampling to find the skew.

**MySQL.** `SHOW ENGINE INNODB STATUS` (latest deadlock, history list length, semaphore waits),
`performance_schema.data_locks` and `data_lock_waits` (8.0), `sys.innodb_lock_waits`,
`sys.schema_table_lock_waits` and `performance_schema.metadata_locks` (MDL queues), the slow log
plus `pt-query-digest`, `SHOW REPLICA STATUS`.

**MongoDB.** `db.currentOp()`, the profiler (`db.setProfilingLevel(1, { slowms: 100 })`),
`explain("executionStats")` (compare `totalKeysExamined` and `totalDocsExamined` to `nReturned`),
`serverStatus().wiredTiger.cache`, `rs.printSecondaryReplicationInfo()`.

**Cassandra.** `nodetool tpstats` (dropped messages, pending), `tablestats` (tombstones per slice,
max partition size), `tablehistograms`, `compactionstats`, `proxyhistograms`.

### 3. MVCC, vacuum, bloat and transaction ID wraparound

- Postgres never updates in place: an `UPDATE` writes a new tuple and marks the old one dead via
  `xmax` (`DB08`). A `DELETE` only sets `xmax`. Dead tuples stay in the heap and in every index
  until `VACUUM` removes them, and it can only remove tuples older than the **oldest snapshot still
  running anywhere** — the "xmin horizon".
- **What holds the horizon back** (and therefore stops vacuum cleaning *every table in the
  cluster*, not just the one being queried): a long transaction or `idle in transaction` session;
  a long query on a replica with `hot_standby_feedback = on`; an **inactive replication slot**
  (`pg_replication_slots.xmin` / `catalog_xmin`); a forgotten **prepared transaction**
  (`pg_prepared_xacts`, from two-phase commit); a long `pg_dump` (it is one repeatable-read
  snapshot). Diagnose with `backend_xmin` in `pg_stat_activity`, the slot view and
  `pg_prepared_xacts`, ordered by `age(xmin)`. `VACUUM VERBOSE` prints "N dead row versions cannot
  be removed yet, oldest xmin: X" — the tell. Prepared transactions are the sneaky one: they have no
  session, survive restarts, and hold locks and the horizon until someone runs `COMMIT PREPARED` or
  `ROLLBACK PREPARED 'gid'` (check with the coordinator that created them first). Keep
  `max_prepared_transactions = 0` unless you really use two-phase commit.
- **Autovacuum triggers** when dead tuples exceed `autovacuum_vacuum_threshold` (50) +
  `autovacuum_vacuum_scale_factor` (0.2) × rows. On a 500-million-row table that is 100 million
  dead rows before it even starts — far too late. Per-table overrides are the fix:
  `ALTER TABLE events SET (autovacuum_vacuum_scale_factor = 0, autovacuum_vacuum_threshold = 100000)`.
  Analyze has its own pair (threshold 50, scale 0.1). Insert-only tables only got insert-triggered
  vacuum in PG13 (`autovacuum_vacuum_insert_threshold` 1000, scale 0.2); before that an append-only
  table was not vacuumed until wraparound forced it, so its visibility map was never set and
  index-only scans degraded into heap fetches (`DB16`).
- **Autovacuum throttling**: `autovacuum_vacuum_cost_limit` (defaults to `vacuum_cost_limit` 200)
  shared across `autovacuum_max_workers` (3), sleeping `autovacuum_vacuum_cost_delay` (2 ms since
  PG12, 20 ms before). On modern SSDs the defaults are conservative; raising the cost limit to
  1000–2000 is common. More workers without a higher cost limit makes each worker *slower*, because
  the budget is shared. `maintenance_work_mem` (64 MB default) bounds how many dead tuple IDs one
  pass can hold; too small and vacuum rescans every index several times (PG17 removed the 1 GB cap
  and made the storage far more compact).
- Autovacuum **yields** to lock conflicts: if your DDL needs a lock, a normal autovacuum cancels
  itself. It then restarts later, and a table that is DDL'd or locked frequently may never finish
  vacuuming. The exception is below.
- **Transaction ID wraparound.** Xids are 32-bit; comparison is circular, so only ~2.1 billion
  transactions can be "in the past". Old tuples must be **frozen** (marked visible to everyone)
  before the counter comes round. `autovacuum_freeze_max_age` (200 million) forces an
  **anti-wraparound autovacuum** on any table whose `relfrozenxid` is older than that — even with
  autovacuum disabled on it — and that vacuum **does not yield to lock requests**. The classic
  surprise: a routine `ALTER TABLE` hangs behind `autovacuum: VACUUM public.orders (to prevent
  wraparound)`, and everything queues behind the `ALTER` (§4). Do not kill it repeatedly; it just
  comes back and you lose ground. Run your DDL later, or run a manual `VACUUM (FREEZE)` in a quiet
  window so the forced one never triggers during the day.
- The endgame: warnings at 40 million xids left ("database must be vacuumed within N
  transactions"); at around 3 million left (1 million on older versions) Postgres refuses to assign
  new xids — writes stop, the database is effectively read-only. Older versions required
  single-user mode to recover; modern versions accept a plain `VACUUM` by a superuser. PG14 added
  `vacuum_failsafe_age` (1.6 billion): vacuum drops its cost delay and skips index cleanup to freeze
  as fast as possible. Before vacuuming, **remove whatever holds the horizon** or the vacuum cannot
  freeze anything.
- Monitor it: `SELECT datname, age(datfrozenxid) FROM pg_database;` and per table
  `age(relfrozenxid)` in `pg_class`. Alert at 500 million–1 billion, not at the warning. Burn rate
  matters: a system doing 20k write transactions per second consumes 200 million xids in under
  three hours. Batching writes into fewer transactions slows the burn; plain `SELECT`s do not
  consume xids, but `SELECT ... FOR UPDATE` does, and every subtransaction that writes gets its own.
- **Multixact wraparound** is the sibling nobody watches: when several transactions share a row
  lock (`FOR SHARE`, or `FOR KEY SHARE` taken implicitly by every foreign-key insert on the parent
  row), Postgres allocates a multixact ID. `autovacuum_multixact_freeze_max_age` (400 million)
  governs it; monitor `mxid_age(datminmxid)`. Heavy FK traffic onto a few parent rows (every order
  referencing the same `merchant`) creates multixact storms and `MultiXact*` SLRU waits; PG17 made
  the SLRU caches configurable.
- **Subtransaction overflow**: each backend caches 64 subtransaction xids; beyond that, visibility
  checks on *every* backend must consult the `pg_subtrans` SLRU. A code path that opens more than
  64 writing savepoints in one transaction (an ORM wrapping each row of a loop in a nested
  `atomic()`/`SAVEPOINT`, or PL/pgSQL `EXCEPTION` blocks in a loop — each is a subtransaction) plus
  one long transaction elsewhere can collapse throughput cluster-wide with `SubtransSLRU` waits,
  on replicas too. The fix is removing the savepoints, not tuning.
- **Bloat does not shrink by itself.** `VACUUM` makes space reusable inside the table; it only
  returns space to the OS by truncating empty pages at the *end* of the file. Truncation takes an
  `ACCESS EXCLUSIVE` lock briefly; on a replica the replay of that lock cancels long queries and on
  the primary it can pile up waiters. `ALTER TABLE ... SET (vacuum_truncate = false)` (PG12) turns
  it off for tables where that bites.
- Reclaiming space: `VACUUM FULL` and `CLUSTER` rewrite the table under `ACCESS EXCLUSIVE` for the
  whole duration — an outage on a big table. **`pg_repack`** (or `pg_squeeze`) rebuilds online with
  a trigger-fed copy and a brief lock at the swap; it needs free disk roughly equal to the table
  plus indexes, and a primary key or unique not-null index. For indexes, `REINDEX CONCURRENTLY`
  (PG12+). Estimating bloat: the `pgstattuple` extension (exact, reads the table) or the
  well-known catalog-based estimation queries (cheap, approximate).
- **Queue tables are a bloat special case.** A table used as a job queue with `SELECT ... FOR
  UPDATE SKIP LOCKED` (`DB09`) is constantly inserted and deleted; dead tuples accumulate at the
  low end of the index the dequeue query scans, so each dequeue walks thousands of dead index
  entries and latency climbs from 1 ms to 500 ms while the live row count stays tiny. Fixes: very
  aggressive per-table autovacuum, keep the table small, partition by time and `DROP`/`TRUNCATE` old
  partitions instead of deleting, and kill whatever holds the horizon — one long analytics query
  anywhere on the cluster makes the queue rot.
- **`DELETE` of a large fraction of a table is the wrong tool.** It generates a dead tuple and WAL
  per row, bloats every index, and fights vacuum. Options: copy the rows you keep into a new table
  and swap names in one short transaction; partition and drop partitions; delete in small batches
  keyed on the primary key with pauses, and let vacuum keep up between batches.

### 4. Locks, the lock queue, deadlocks and contention

- Postgres table-lock levels that matter (`DB09`): `ACCESS SHARE` (every `SELECT`), `ROW
  EXCLUSIVE` (DML), `SHARE UPDATE EXCLUSIVE` (`VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`,
  `VALIDATE CONSTRAINT`), `SHARE` (plain `CREATE INDEX`, and plain `REINDEX` on the table plus
  `ACCESS EXCLUSIVE` on the index — both block writes), `SHARE ROW EXCLUSIVE` (adding a foreign key,
  on both tables; `CREATE TRIGGER`), `ACCESS EXCLUSIVE` (most `ALTER TABLE`, `DROP`, `TRUNCATE`,
  `VACUUM FULL`, `CLUSTER`, `LOCK TABLE`). `ACCESS EXCLUSIVE` conflicts with everything including
  `SELECT`.
- **The lock queue.** Lock requests queue in order; a new request that conflicts with a *waiting*
  request waits behind it. An `ALTER TABLE` that needs `ACCESS EXCLUSIVE` waits behind a 40-minute
  analytics `SELECT` — and every new `SELECT` on the table now waits behind the `ALTER`. The `ALTER`
  itself would have taken 2 ms; the table is down for 40 minutes. Symptom: sudden spike of
  connections all in `Lock/relation`, pool exhausted, and the culprit is an innocuous-looking long
  query (or an `idle in transaction` session) at the head.
- **The fix is `lock_timeout` plus retry**, never "run it at night and hope":

```sql
SET lock_timeout = '2s';           -- give up quickly rather than block the world
SET statement_timeout = '30s';
ALTER TABLE orders ADD COLUMN note text;   -- retried by the migration runner with backoff and jitter
```

  Framework migration tools usually do not do this; senior teams wrap them (a migrations linter
  such as `strong_migrations`/`squawk`, and a runner that sets `lock_timeout` and retries N times).
- **MySQL has the same trap as metadata locks (MDL).** Any open transaction that has *touched* the
  table — even a single `SELECT` in an uncommitted transaction — holds a shared MDL. `ALTER TABLE`
  waits for the exclusive MDL, and everything behind it shows `Waiting for table metadata lock`.
  Look in `performance_schema.metadata_locks` and kill the idle transaction. `lock_wait_timeout`
  defaults to one year; set it to a few seconds for DDL sessions.
- **Row locks**: in Postgres, row locks live in the tuple header (`xmax`), not in shared memory, so
  locking millions of rows is cheap in memory, but a waiter blocks on the *transaction ID* of the
  holder (`Lock/transactionid`) and writing the lock dirties the page (WAL, and a `SELECT FOR
  UPDATE` over many rows is a write). There is no lock escalation in Postgres; SQL Server escalates
  row locks to a table lock at about 5,000 locks per statement, which is why a big batch update
  there suddenly blocks unrelated readers.
- **Foreign keys take locks you did not ask for.** Inserting a child row takes `FOR KEY SHARE` on
  the parent row; updating the parent's non-key columns takes `FOR NO KEY UPDATE`, which does not
  conflict with it. But an ORM that issues `SELECT ... FOR UPDATE` on the parent (the strong lock)
  *does* conflict with every child insert — a hot parent row (a "wallet" or "tenant" row) then
  serialises all child writes. Use `FOR NO KEY UPDATE` when you are not changing the key.
- **Missing index on the referencing column**: `DELETE FROM customers WHERE id = 7` must check
  `orders.customer_id` for references; without an index on `orders(customer_id)` that is a full scan
  of `orders` per deleted parent row, holding locks the whole time. `ON DELETE CASCADE` over a large
  fan-out does the same work inside one transaction. Postgres does not index foreign-key columns
  automatically; MySQL InnoDB does.
- **Deadlocks** are detected after `deadlock_timeout` (1 s) and one victim is aborted (`40P01`).
  Real-world causes: two code paths updating the same rows in different orders; batch updates whose
  `WHERE id IN (...)` rows are locked in scan order rather than sorted; triggers updating a parent;
  FK checks. Fixes: lock in a consistent order (`SELECT ... ORDER BY id FOR UPDATE` first), smaller
  transactions, retry on `40P01` and `40001`.
- **Hot rows**: a single counter row (`UPDATE stats SET views = views + 1 WHERE id = 1`) serialises
  every writer and creates a dead tuple per increment. Unconventional fixes: shard the counter into
  N rows (`slot = random() * 16`) and sum on read; append to an insert-only table and aggregate
  asynchronously; accumulate in Redis and flush every second; for balances, make the check part of
  the write (`UPDATE ... SET balance = balance - $1 WHERE id = $2 AND balance >= $1`, check rows
  affected) so the lock is held for one statement instead of a read-modify-write round trip.
- **Advisory locks** (`pg_advisory_lock(key)`, `pg_advisory_xact_lock(key)`) are cheap
  application-level mutexes — good for "only one worker runs this cron". Session-level ones leak
  across requests through a pooler in transaction mode (§7) and are held until the *server*
  connection closes; prefer the `_xact_` variant, or `pg_try_advisory_xact_lock` to skip rather than
  wait.
- **Queue pattern**: `SELECT id FROM jobs WHERE status = 'ready' ORDER BY id LIMIT 10 FOR UPDATE
  SKIP LOCKED` lets N workers dequeue without blocking each other. Without `SKIP LOCKED` they
  convoy on the first row. Combine with §3's bloat advice.

### 5. InnoDB specifics (MySQL, Aurora MySQL)

- Default isolation is **REPEATABLE READ**, implemented with **next-key locks** (row lock plus the
  gap before it) on locking reads and on `UPDATE`/`DELETE` scans. Consequences:
  - `SELECT ... FOR UPDATE WHERE email = ?` on a *non-existent* row locks the gap. Two sessions
    doing "check, then insert if missing" both get the gap lock (gap locks are compatible with each
    other), then both insert; each insert-intention lock waits on the other's gap lock → deadlock.
    The most common InnoDB deadlock in the wild. Fix: just `INSERT` and handle the duplicate-key
    error, or `INSERT ... ON DUPLICATE KEY UPDATE`.
  - An `UPDATE` whose `WHERE` is not served by an index locks **every row it scans** (and the gaps)
    — effectively a table lock. The first question for an InnoDB lock storm is "is the `WHERE`
    indexed?".
  - Switching to `READ COMMITTED` removes most gap locking (requires `binlog_format = ROW`) and is
    what many large MySQL shops run.
- **Purge lag / history list length**: InnoDB keeps old versions in undo logs; a long-running
  transaction (or a forgotten consistent-snapshot export) stops purge. `History list length` in
  `SHOW ENGINE INNODB STATUS` climbing into the millions means reads walk longer version chains and
  queries on hot rows get slower and slower. Before 8.0 undo lived in `ibdata1`, which never
  shrinks — the disk is gone until a rebuild.
- **AUTO_INCREMENT**: `innodb_autoinc_lock_mode = 2` (8.0 default) interleaves values and is only
  safe with row-based replication. Before 8.0 the counter was reset to `MAX(id)+1` on restart, so
  deleting the top rows and restarting reused IDs — a real source of "an old order's ID came back"
  and of collisions with IDs held in caches, logs and other systems.
- **`utf8` is 3-byte `utf8mb3`**: emoji and some CJK characters fail or get truncated. Converting to
  `utf8mb4` enlarges index keys (old 767-byte index prefix limit → `VARCHAR(255)` no longer fits; the
  3072-byte limit with the `DYNAMIC` row format solves it).
- **Implicit conversions kill indexes**: `WHERE phone = 447700900123` on a `VARCHAR` column casts
  every row to a number — full scan, and wrong matches (`'0447...'` compares equal to `447...`).
  Collation or charset mismatch between joined columns does the same to joins.
- **Online DDL**: `ALGORITHM=INSTANT` (8.0.12+ for adding columns, extended in 8.0.29), `INPLACE`,
  or `COPY`. Even `INPLACE` needs a brief exclusive MDL at start and end (§4), and replicas apply the
  DDL single-threaded *after* the primary finished it — a 3-hour `ALTER` becomes 3 hours of replica
  lag. Hence **gh-ost** (triggerless, tails the binlog, throttles on replica lag, controllable
  postponed cut-over) and **pt-online-schema-change** (trigger-based; the triggers add write load
  and lock contention). Both need roughly 2× the table's disk. gh-ost's cut-over briefly locks the
  original table, applies the remaining binlog events to the ghost table and swaps them with an
  atomic `RENAME`; with `--postpone-cut-over-flag-file` you choose the moment. Writes blocked during
  that window wait on the lock — apps with short timeouts and no retry see errors, and a long
  transaction on the table delays the lock and stretches the window.
- Finding the transaction that stops purge: `information_schema.innodb_trx` (`trx_started`,
  `trx_mysql_thread_id`) lists every open InnoDB transaction, including idle sessions that started
  one and never committed; join to `performance_schema.threads` for the host. Since 8.0 undo lives in
  separate undo tablespaces that can be truncated.
- The buffer pool (`innodb_buffer_pool_size`, typically 60–75% of RAM) and
  `innodb_flush_log_at_trx_commit` (1 = durable; 2 = write per commit, fsync once a second, can
  lose ~1 s on OS crash) are the knobs everybody asks about; `sync_binlog = 1` is the
  replication-safety twin.
- **Aurora (MySQL and Postgres)**: storage is a shared distributed volume (six copies across three
  AZs), replicas read the same storage, so replica lag is usually tens of milliseconds and failover
  is ~30 s with a DNS flip of the cluster endpoint. The traps: clients caching DNS (the JVM can
  cache forever under a security manager — set `networkaddress.cache.ttl`), connection pools that
  keep stale sockets to the old writer (now a reader — writes fail with "read-only" errors until the
  pool recycles), and replica reboots when a replica falls too far behind.

### 6. The planner in production: statistics, estimates and plan flips

- The planner is a cost model fed by statistics in `pg_statistic` (readable via `pg_stats`):
  null fraction, `n_distinct`, most-common values (MCV) and a histogram, per column, from a sample
  of 300 × `default_statistics_target` (100) rows = 30,000 rows regardless of table size (`DB18`).
  On a 2-billion-row table with a skewed column that sample is thin: rare values are missing from
  the MCV list and get a generic estimate. `ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000` (up to
  10,000) and re-`ANALYZE` widens it; `ALTER COLUMN c SET (n_distinct = -0.05)` overrides a bad
  distinct estimate (negative = fraction of rows).
- **Correlated columns**: the planner multiplies selectivities as if columns were independent.
  `WHERE country = 'DE' AND city = 'Berlin'` is estimated at `P(DE) × P(Berlin)` — far too low —
  and a 100× underestimate turns a hash join into a nested loop executed 2 million times.
  `CREATE STATISTICS s (dependencies, ndistinct, mcv) ON country, city FROM addresses;` then
  `ANALYZE` fixes it (PG10+, MCV lists PG12+). The tell in `EXPLAIN ANALYZE` (`DB19`): `rows=12`
  estimated versus `actual rows=240000` on the node feeding a `Nested Loop`.
- **Stale statistics after bulk changes**: autoanalyze fires at 10% changed rows; a nightly job
  that loads 5% new rows with *new* dates means queries for "today" are estimated from a histogram
  that ends yesterday — the planner thinks almost no rows match and picks a nested loop. Always
  `ANALYZE` explicitly after bulk loads, inside the job. **Temporary tables are never
  autoanalyzed**; analyse them yourself before joining on them.
- **`pg_upgrade` does not carry planner statistics** (until PG18). Right after a major-version
  upgrade every table has no stats and plans are terrible until `vacuumdb --all --analyze-in-stages`
  finishes. This is the most common "the upgrade went fine and then the site fell over" story.
- **Prepared statements and generic plans**: a prepared statement (and every parameterised query
  through JDBC, `node-postgres` named statements, or ORMs with server-side prepare) is planned with
  the actual parameters for the first five executions ("custom plans"); after that Postgres switches
  to a **generic plan** if its estimated cost is not much worse than the average custom plan. The
  generic plan is costed for an *average* parameter. For a skewed column — one tenant with 40% of
  rows, thousands with 10 rows each — the generic plan chosen on the sixth call can be catastrophic
  for the big tenant, and it sticks for the life of the connection. Symptom: fast in `psql`, slow
  from the app, and "fixed by restarting the app" for a while. Reproduce with `PREPARE` and `EXPLAIN
  EXECUTE` six times. Fix: `plan_cache_mode = force_custom_plan` (PG12+, settable per role or
  session) for the affected queries, or inline the literal for the skewed column.
- **Partial indexes and generic plans**: a partial index `WHERE status = 'pending'` cannot be used
  by a generic plan for `WHERE status = $1`, because the planner cannot prove the predicate. Write the
  literal into the SQL (`WHERE status = 'pending'`) rather than binding it.
- **The `ORDER BY ... LIMIT` trap**: `SELECT * FROM events WHERE user_id = 42 ORDER BY created_at
  DESC LIMIT 10`. With separate indexes on `created_at` and `user_id`, the planner may walk
  `created_at` backwards and filter, expecting to find 10 matches quickly because it assumes matching
  rows are spread uniformly. If user 42's last event was two years ago it walks hundreds of millions
  of index entries. Fixes: a composite index `(user_id, created_at)` that serves both; or, as an
  emergency, make the sort expression non-indexable (`ORDER BY created_at + interval '0'`, or
  `ORDER BY id + 0`) to force the filter-first plan.
- **Optimisation fences**: before PG12, a CTE was always materialised — people used `WITH` to force
  join order. PG12 inlines non-recursive, side-effect-free CTEs referenced once, so upgrades changed
  plans for such queries in both directions. `WITH x AS MATERIALIZED (...)` restores the fence;
  `OFFSET 0` in a subquery is the old-school fence.
- **Postgres has no plan hints or plan pinning in core.** Levers, in order: fix statistics (targets,
  extended stats, `ANALYZE`); rewrite the query; adjust costs per session or role (`random_page_cost
  = 1.1` on SSDs instead of the 4.0 default is legitimate and often overdue; `SET LOCAL
  enable_nestloop = off` inside one transaction as a scalpel); the `pg_hint_plan` extension (whose hint table
  attaches hints to a normalised query text — the escape hatch when you cannot change vendor SQL); Aurora's
  Query Plan Management. Never change `enable_*` globally.
- **JIT** (on by default since PG12) kicks in above `jit_above_cost` (100,000). An OLTP query with a
  bad row estimate crosses that threshold and spends 50–500 ms compiling for a query that runs in
  5 ms; `EXPLAIN ANALYZE` shows a `JIT:` section with its timing. Most OLTP deployments set
  `jit = off`.
- **`work_mem` spills**: a sort or hash that exceeds `work_mem` (4 MB default) goes to temp files
  (`Sort Method: external merge Disk: 812MB`, `Batches: 64` on a hash). `SET LOCAL work_mem = '256MB'`
  for the one report, never globally (§12). `hash_mem_multiplier` (2.0 since PG15) gives hashes more.
- **Index not used for type or collation reasons**: comparing a `bigint` column to a `numeric`
  parameter, or `lower(email)` without an expression index (`DB17`), prevents index use. `LIKE 'abc%'`
  can use a B-tree only with the `C` collation or a `text_pattern_ops` index. Drivers are a hidden
  source: the SQL Server JDBC driver sends strings as `NVARCHAR` by default
  (`sendStringParametersAsUnicode`), turning every lookup on a `VARCHAR` key into a scan; MySQL's
  equivalent is §5's implicit conversion.
- **`NOT IN (subquery)`** with a nullable column returns no rows if any value is NULL and cannot be
  planned as an anti-join; `NOT EXISTS` is correct and fast. **`OR` across two columns** often defeats
  single-column indexes; rewrite as `UNION ALL` of two indexed queries or rely on a `BitmapOr`. Huge
  `IN (...)` lists: prefer `= ANY($1::bigint[])` — one parameter, one plan shape.
- **`count(*)` on a big table is always a scan** (MVCC — no stored row count, `DB08`). For a UI
  counter use `reltuples` from `pg_class` or an `EXPLAIN` estimate, or a maintained counter.
- **Many-table joins**: the planner searches join orders exhaustively only up to
  `join_collapse_limit`/`from_collapse_limit` (8 each); beyond that it keeps the written join order
  for the rest, and from `geqo_threshold` (12) tables the genetic optimiser (GEQO) takes over, a
  heuristic search that can settle on a far worse order, and small changes in statistics can swing
  its result — a 14-table query whose plan quality varies from day to day. Raise the limits
  for that session, or write the joins in a good order explicitly.
- **Offset pagination** (`OFFSET 500000 LIMIT 50`) reads and discards 500,000 rows; bots crawling
  page 20,000 of a listing are a classic CPU incident. Keyset pagination (`WHERE (created_at, id) <
  ($1, $2) ORDER BY created_at DESC, id DESC LIMIT 50`) is constant cost (`DB20`); cap the maximum
  offset for the rest.

### 7. Connections, pooling and the network in between

- Postgres forks one **process per connection** (`DB21`); each costs several MB of private memory
  and more as its catalogue and plan caches grow — long-lived connections touching thousands of
  partitions can reach hundreds of MB each. `max_connections` defaults to 100. Before PG14, snapshot
  acquisition cost scaled with the number of connections, so 3,000 mostly idle connections made every
  query slower; PG14 fixed most of that, but high connection counts still cost memory and
  lock-manager contention.
- **Pool sizing**: throughput peaks when active connections ≈ a small multiple of cores; beyond that
  you add context switching and contention. A pool of 10–30 per service instance class, fronted by a
  queue, beats 500. Total demand = pods × pool size: 80 pods × 20 = 1,600 connections — autoscaling
  the app is how you DoS the database.
- **PgBouncer modes**: `session` (a server connection per client for its lifetime — little gain),
  `transaction` (server connection assigned per transaction — the useful one), `statement`.
  **Transaction mode breaks session state**: `SET` without `LOCAL` (leaks to the next client),
  session advisory locks, `LISTEN`/`NOTIFY`, temporary tables, `WITH HOLD` cursors, and
  **protocol-level prepared statements** (prepared on one server connection, executed on another →
  `prepared statement "S_1" does not exist`). PgBouncer 1.21+ tracks prepared statements in
  transaction mode (`max_prepared_statements`); otherwise disable server-side prepares in the driver
  (`prepareThreshold=0` in pgJDBC, `statement_cache_size=0` in asyncpg). `default_pool_size` is 20 per
  user/database pair.
- **RDS Proxy pinning**: the proxy pins a client to a server connection when it sees session state
  (`SET`, some prepared-statement use, temp tables, advisory locks). Pinned connections no longer
  multiplex, so the proxy quietly behaves like no proxy. The `DatabaseConnectionsCurrentlySessionPinned`
  metric is the tell.
- **Serverless / Lambda**: each concurrent invocation is a process with its own pool; a burst of 1,000
  invocations is 1,000 connections, and frozen containers keep sockets open. Use a proxy or pooler,
  pool size 1 per function, and reserved concurrency as a hard cap.
- **`idle in transaction`**: the app began a transaction and is doing something else (an HTTP call, a
  slow serialisation, waiting on a queue). It holds locks and the xmin horizon (§3) and occupies a
  pool slot. `idle_in_transaction_session_timeout` kills them; the real fix is transaction scope in
  code (`DB06`). Frameworks that open a transaction per request (`ATOMIC_REQUESTS`,
  open-session-in-view) hold it across template rendering and outbound calls.
- **Pool deadlock**: a request holds connection A (in a transaction) and, inside it, code asks the pool
  for connection B (Spring `REQUIRES_NEW`, an audit write on a separate connection, a second
  repository using a different `DataSource` wrapper). With pool size N and N concurrent requests, every
  request holds one connection and waits for a second: the service freezes while the database is
  idle. Fix: remove the nested acquisition, or size pool ≥ threads × (connections per request − 1) + 1.
- **Connection storms**: a database restart or failover drops every connection; all app instances
  reconnect at once, each new Postgres backend forks and authenticates (SCRAM is deliberately
  CPU-expensive), `max_connections` is hit, clients retry immediately, and the storm sustains itself.
  Fixes: jittered reconnect backoff, a pooler in front, minimum-idle settings below the maximum, and
  staggered pod restarts.
- **Middleboxes kill idle connections silently**: AWS NLB and NAT gateways drop idle TCP flows after
  350 s; many firewalls after an hour. The pool thinks the connection is alive; the next query hangs
  until TCP retransmission gives up (~15 minutes on Linux defaults). Fix: pool `maxLifetime` and idle
  timeouts below the middlebox timeout (HikariCP `maxLifetime` defaults to 30 minutes; set
  `keepaliveTime`), TCP keepalives (`tcp_keepalives_idle` on the server, `keepalives_idle` in libpq),
  and a client-side socket timeout.
- **Timeout layering**: HTTP request timeout ≥ client socket timeout > `statement_timeout` > the query
  you expect. When the HTTP layer times out first, the query keeps running on the server, the client
  retries, and you have two copies running — cancellation must propagate (driver cancel on context
  cancellation), and `statement_timeout` must sit below the caller's timeout.
- **DNS and failover**: managed failover flips a DNS name. JVM DNS caching, long-lived pools and
  `/etc/hosts` pins keep writing to the old primary (now a replica, returning `cannot execute INSERT
  in a read-only transaction`). Libpq multi-host connection strings with
  `target_session_attrs=read-write`, and pools that validate connections and evict on that error,
  handle it.
- **Pool hygiene details that cause incidents**: connections opened together and retired by the same
  `maxLifetime` all reconnect in the same second (HikariCP subtracts a small random variance for
  exactly this reason; home-grown pools often do not); pool leak detection
  (`leakDetectionThreshold`) finds code paths that check out a connection and never return it; and
  "pool timeout while the database is idle" always means connections are checked out but not
  executing — held around non-database work or leaked.
- **Health checks that touch the database** turn a 30-second database blip into a fleet-wide restart:
  liveness fails, the orchestrator kills every pod, and they all reconnect at once. Liveness should
  check the process only; readiness may check the database but should degrade (serve cached or
  read-only responses) rather than flap.

### 8. Indexes in production

- **Every index is a write tax**: each `INSERT` updates every index; each non-HOT `UPDATE` (§9)
  inserts an entry into every index. Tables with 15 indexes on a write-heavy path are common and are
  often the real cause of "the database cannot keep up with writes". Find unused ones via
  `pg_stat_user_indexes.idx_scan` on *every* node (§2), duplicates (an index that is a prefix of
  another), and drop them with `DROP INDEX CONCURRENTLY`.
- **`CREATE INDEX CONCURRENTLY`** (`DB22`) does two table scans and waits for every transaction that
  could see the table to finish — including unrelated long transactions, so it "hangs" behind them. It
  cannot run inside a transaction block (migration frameworks wrap migrations in one by default —
  disable it per migration). If it fails (deadlock, unique violation, cancel) it leaves an **INVALID
  index** that is still maintained on every write but never used; find them with `SELECT
  indexrelid::regclass FROM pg_index WHERE NOT indisvalid;` and drop before retrying.
- **Unique constraint online**: `CREATE UNIQUE INDEX CONCURRENTLY`, then `ALTER TABLE t ADD CONSTRAINT
  u UNIQUE USING INDEX idx` — the `ALTER` is metadata-only.
- **Index bloat**: B-trees do not merge half-empty pages back; after mass deletes or with random-key
  churn, indexes can be several times their ideal size. PG13's deduplication and PG14's bottom-up
  deletion help a lot for duplicate-heavy and update-heavy cases. `REINDEX CONCURRENTLY` rebuilds.
- **Random keys vs sequential keys**: a UUIDv4 primary key inserts at random places in the B-tree, so
  every insert touches a random leaf page — the index's working set is the *whole* index, the cache hit
  ratio falls as the table grows, and each checkpoint cycle triggers a full-page image for every newly
  dirtied page (§12), inflating WAL several times. Time-ordered IDs (UUIDv7, ULID, Snowflake, or a
  `bigint` identity) keep inserts on the right-most pages. InnoDB is worse, because the primary key
  *is* the clustered table and every secondary index stores the PK.
- **Right-most page contention** is the opposite problem at extreme insert rates: every insert fights
  for the same last leaf page (`LWLock/BufferContent`). Rare below tens of thousands of inserts per
  second per table; the answers are hash partitioning or a key with a small random prefix.
- **GIN** (`DB14`, `DB34`): with `fastupdate = on` (default) new entries go to a pending list, flushed
  when it exceeds `gin_pending_list_limit` (4 MB) or by vacuum. The unlucky insert that triggers the
  flush takes seconds — a periodic p99 spike on a JSONB or full-text table. Fix: `ALTER INDEX ... SET
  (fastupdate = off)` or a smaller pending list, plus frequent vacuum.
- **Substring search**: `ILIKE '%term%'` cannot use a B-tree at all; a trigram index (`CREATE
  EXTENSION pg_trgm; CREATE INDEX ... USING gin (email gin_trgm_ops)`) serves `LIKE`/`ILIKE` with
  leading wildcards and similarity search, at the cost of a large index and GIN write overhead.
  Often the better answer is product-level: prefix search, or a search engine (§20).
- **BRIN** is tiny and great for append-only, time-ordered data, and useless once rows are updated or
  loaded out of order (block ranges' min/max overlap). **Hash** indexes are crash-safe only since PG10.
- **Covering indexes and the visibility map**: an index-only scan still visits the heap for any page
  not marked all-visible. `Heap Fetches: 1843221` in `EXPLAIN ANALYZE` means vacuum is behind (§3) —
  the fix is vacuum, not a different index.
- **Expression indexes must match exactly**: an index on `lower(email)` is not used by `ILIKE`, or when
  a cast inside the expression differs.
- **Collation changes corrupt indexes**: text B-trees are ordered by the OS's libc collation. glibc
  2.28 (Debian 10, RHEL 8, Ubuntu 18.10+) changed the sort order of many strings; upgrading the OS
  under an existing data directory (or running a physical replica on a newer OS) leaves indexes whose
  order no longer matches the comparator. Symptoms: duplicate values in unique indexes, rows missed
  by an index lookup that a seq scan finds, `ORDER BY` results in a different order on the replica.
  Detect with `amcheck` (`bt_index_check`), fix with `REINDEX`. Prevent with versioned ICU
  collations or the `C` collation for identifiers, and never mix OS versions in a physical
  replication set.

### 9. Row storage: HOT, fillfactor, TOAST, identifiers

- **HOT (heap-only tuple) updates**: if an update changes no indexed column *and* the new version fits
  on the same page, Postgres chains it within the page and writes **no index entries**; later accesses
  prune the chain without vacuum. HOT ratio = `n_tup_hot_upd / n_tup_upd` in `pg_stat_user_tables`.
  Two ways teams destroy it without noticing: adding an index on a frequently updated column
  (`updated_at`, `status`, `last_seen_at`), and pages with no free space. PG16 lets BRIN indexes not
  block HOT.
- **`fillfactor`**: tables default to 100 (pages packed full). For update-heavy tables `ALTER TABLE t
  SET (fillfactor = 80)` leaves room for HOT; it applies to newly written pages, so existing data needs
  a rewrite (`pg_repack`) to benefit. Trade-off: 20% more space and more pages to scan.
- **TOAST**: values that would make a row exceed ~2 KB are compressed (pglz, or lz4 since PG14) and
  moved out of line to the table's TOAST relation in ~2 KB chunks. Consequences: `SELECT *` pulls every
  TOASTed column — selecting only needed columns can be 10× faster; updating one key of a 1 MB `jsonb`
  document rewrites the whole 1 MB (and WAL-logs it), so "we store the user profile as one JSONB blob
  and update `last_login` in it" is a WAL and bloat generator; move hot small fields to real columns.
  Unchanged TOASTed values are not rewritten when *other* columns change.
- **TOAST OID exhaustion**: each out-of-line value needs a unique 32-bit OID within its TOAST table. A
  table with billions of large values approaches 4 billion; inserts then spend long periods searching
  for a free OID and slow to a crawl. Rare, but it is the answer when inserts into a huge, write-once
  blob table degrade with no other explanation. Fix: partition (each partition has its own TOAST table).
- **`int4` exhaustion**: `serial`/`int` IDs top out at 2,147,483,647. Nobody notices until inserts fail
  with `integer out of range` or `nextval: reached maximum value` (MySQL: `Duplicate entry
  '2147483647' for key 'PRIMARY'`). Referencing FK columns are often `int` too. `ALTER COLUMN TYPE
  bigint` is a full rewrite under `ACCESS EXCLUSIVE`. The playbook: add a `bigint` column, keep it in
  sync with a trigger, backfill in batches, build the index concurrently, then swap in a short
  transaction. **The emergency trick**: restart the sequence at `-2147483648` counting upward
  (`ALTER SEQUENCE s MINVALUE -2147483648 RESTART WITH -2147483648`) — buys another two billion IDs in
  minutes, provided nothing in the app assumes IDs are positive or monotonic. Monitor `last_value`
  against the type maximum for every sequence.
- **Sequences are not transactional**: rolled-back inserts burn values, and a crash can skip up to 32
  pre-logged values. Never assume gap-free (invoice numbers need a separate, locked counter — `DB38`).
  With `CACHE n` > 1 each session reserves blocks, so IDs are not in commit order across sessions —
  which matters for §14.

### 10. Migrations and backfills without downtime

- **Expand / migrate / contract** (`DB22`): add the new shape alongside the old, dual-write, backfill,
  switch reads, stop writing the old, drop it — each step a separate deploy, because during a rolling
  deploy old and new code run at the same time. Renaming a column in one step breaks every old pod;
  ORMs that cache column lists break even on "harmless" drops (the old code `SELECT`s the dropped
  column) — first tell the ORM to ignore the column (`ignored_columns` in Rails, remove it from the
  entity), deploy, then drop.
- **What rewrites the table** (full copy under `ACCESS EXCLUSIVE`): changing a column type except
  binary-compatible cases (`varchar(n)` → larger `varchar` or `text` is free; `int` → `bigint` is not;
  `timestamp` → `timestamptz` rewrites unless the session time zone is UTC on PG12+); adding a column
  with a *volatile* default (`clock_timestamp()`, `random()`, `gen_random_uuid()`); `SET LOGGED`.
  Adding a column with a constant or stable default is metadata-only since PG11.
- **What scans under a strong lock**: `SET NOT NULL` scans the whole table under `ACCESS EXCLUSIVE`.
  Trick (PG12+): `ADD CONSTRAINT c CHECK (col IS NOT NULL) NOT VALID` (instant), `VALIDATE CONSTRAINT
  c` (scans under `SHARE UPDATE EXCLUSIVE` — writes continue), then `SET NOT NULL` uses the valid
  check and skips the scan, then drop the check. The same `NOT VALID` → `VALIDATE` split works for
  foreign keys.
- **Backfills**: never one giant `UPDATE` — it is one transaction holding the horizon, doubling the
  table through dead tuples, generating WAL that lags replicas and can fill the disk. Batch by
  primary-key range (`WHERE id >= $1 AND id < $1 + 10000`), not `OFFSET`; commit each batch; sleep
  between batches; throttle on replica lag and WAL rate; make it idempotent and resumable (store the
  high-water mark); run it as a job, not in the migration. Rows touched by live traffic need the
  dual-write in place first, or the backfill overwrites fresh data with stale.
- **Default-transaction migrations**: many tools run each migration inside one transaction, so a
  migration that alters a table and then backfills holds the `ACCESS EXCLUSIVE` lock until the
  backfill ends. Split DDL from data changes.
- **Sync triggers during migrations** add write latency to every statement on the table and can
  deadlock with the backfill; keep them trivial and test the cut-over.
- **Table swap by rename**: build `orders_new`, sync it, then in one transaction with `lock_timeout`
  lock both, apply the final delta, `ALTER TABLE orders RENAME TO orders_old; ALTER TABLE orders_new
  RENAME TO orders;`. Views, functions and FKs referencing the old table follow the OID, not the name —
  they still point at `orders_old`. Check `pg_depend` first.
- **Major-version upgrades**: `pg_upgrade --link` is minutes of downtime but needs the post-upgrade
  `ANALYZE` (§6) and a rollback plan (with `--link`, the old cluster is unusable once the new one has
  started). **Logical replication to a new-version cluster** (or managed blue/green deployments) gives
  near-zero downtime; its caveats: sequences are not replicated (set them past the max before
  cut-over), DDL is not replicated (freeze schema changes during the sync), tables need a primary key
  or `REPLICA IDENTITY`, large objects are skipped, and the initial table copy of a multi-TB database
  runs for hours holding a snapshot and a slot on the source (bloat and WAL retention there). The
  cut-over itself: stop writes, wait for the subscriber to catch up to the source LSN, bump sequences,
  verify counts, repoint the application, `ANALYZE`.
- **Schema-per-tenant at scale**: tens of thousands of schemas means hundreds of thousands of
  relations. Catalogue tables bloat, each backend's relation and catalogue caches grow with every
  table it touches (memory per connection explodes), `pg_dump` and every migration walk all of them,
  and autovacuum spends its time on tiny tables. Beyond a few thousand tenants, shared tables keyed by
  `tenant_id` (with row-level security, `DB37`) or tenant-per-cluster sharding are the escape routes.

### 11. Replication in practice

- **Physical streaming replication** (`DB23`) ships WAL; replicas replay it. Lag has three parts in
  `pg_stat_replication`: `write_lag`, `flush_lag`, `replay_lag`. Replay is a single process in
  Postgres — a replica with fast disks can still lag if the primary's write rate exceeds what one
  process can replay (big index builds, mass updates, vacuum of a huge table).
- **Lag measured in seconds lies when idle**: `now() - pg_last_xact_replay_timestamp()` grows during
  quiet periods with no writes, paging someone at 4am for a healthy replica. Measure bytes
  (`pg_wal_lsn_diff`), or write a heartbeat row every second on the primary and measure its age on the
  replica (the `pt-heartbeat` approach). MySQL's `Seconds_Behind_Source` is similarly misleading — it
  reads 0 when the applier is idle because the IO thread is stalled, and jumps during a big
  transaction.
- **Replica query conflicts**: when the primary vacuums away rows that a replica query still needs,
  replay must either wait or cancel the query — `ERROR: canceling statement due to conflict with
  recovery`. `max_standby_streaming_delay` (30 s) is how long replay waits before cancelling (so lag can
  reach 30 s from that alone). Options: `hot_standby_feedback = on` (the replica reports its xmin — no
  cancellations, but long replica queries now bloat the primary, §3); a dedicated analytics replica with
  a large or `-1` delay, accepting lag; or move analytics to a warehouse. Lock replay (`ACCESS
  EXCLUSIVE` from DDL or vacuum truncation) also cancels replica queries.
- **Replication slots** guarantee WAL is kept until the consumer confirms it. An abandoned slot — a
  decommissioned replica, a paused Debezium connector, a failed CDC task — makes the primary retain WAL
  until **the disk fills and the primary stops**. Logical slots also hold `catalog_xmin`, bloating the
  system catalogues. Set `max_slot_wal_keep_size` (PG13+) so the slot is invalidated instead of the
  primary dying, and alert on retained bytes. Logical slots did not survive failover before PG17 —
  after a failover, CDC had to resnapshot.
- **Synchronous replication**: with `synchronous_standby_names` set and the only sync standby down,
  **every commit hangs** (committed locally, the client waits). Use `ANY 1 (a, b)` with two standbys;
  `SET synchronous_commit = local` in one session bypasses it. Sync replication protects durability,
  not visibility: reads on the standby can still be behind unless `synchronous_commit = remote_apply`.
- **Read-your-writes on replicas**: a user saves a profile and the next page (read from a replica)
  shows the old value. Options: route the user's reads to the primary for N seconds after a write
  (sticky flag in the session); capture `pg_current_wal_lsn()` after the write and have the replica
  read wait until `pg_last_wal_replay_lsn() >= lsn` (or fall back to the primary); read from the
  primary for the entity just written. MySQL: GTID waits (`WAIT_FOR_EXECUTED_GTID_SET`). Balancers that
  spread reads across replicas make "the value flickers between old and new", because each replica is
  at a different point.
- **Failover loses data under async replication**: the last commits acknowledged by the old primary may
  not exist on the new one. If the old primary comes back and accepts writes (split brain), you have two
  divergent histories; `pg_rewind` rewinds it to follow the new primary, discarding its divergent
  writes — extract them first. Fencing (STONITH, revoking the old primary's network) is part of
  failover, not an afterthought. After failover, sequence values on the new primary can be *behind*
  IDs already handed to clients.
- **MySQL replication**: the applier was single-threaded historically; `replica_parallel_workers` with
  `replica_parallel_type = LOGICAL_CLOCK` and WRITESET dependency tracking (8.0) parallelise it. One
  20-million-row `DELETE` is one transaction in the binlog and stalls every replica for as long as it
  took on the primary — batch it. Statement-based replication of non-deterministic statements
  (`UPDATE ... LIMIT` without `ORDER BY`, `UUID()`) silently diverges replicas; use row-based.
  `pt-table-checksum` detects divergence.
- **Cross-region replicas** add network latency to lag; a replica in another region is a DR asset, not
  a read-scaling one for latency-sensitive paths.

### 12. WAL, checkpoints, I/O, memory and disk

- **Checkpoints** flush dirty buffers to data files; after each checkpoint, the first modification of
  any page writes a **full-page image** into WAL (`full_page_writes`, torn-page protection). With the
  defaults `checkpoint_timeout = 5min` and `max_wal_size = 1GB`, a busy system checkpoints every minute
  or two ("checkpoints are occurring too frequently" in the log), and full-page images can be 50–90% of
  WAL volume. Raising `max_wal_size` to tens of GB and `checkpoint_timeout` to 15–30 minutes cuts WAL
  volume dramatically (and replica lag, and archive size), at the cost of longer crash recovery.
  `checkpoint_completion_target = 0.9` spreads the writes; `wal_compression` compresses the images.
- **I/O cliffs on cloud disks**: AWS gp2 volumes give 3 IOPS per GB with bursts to 3,000 from a credit
  bucket; a 200 GB volume has a 600 IOPS baseline. The database is fine for hours, then after the
  credits run out latency jumps 10× with no change in traffic — watch `BurstBalance`. gp3 has a flat
  3,000 IOPS / 125 MB/s baseline, provisioned separately from size. Instance types also have their own
  EBS bandwidth cap, often reached before the volume's.
- **Disk full** on the WAL volume makes Postgres PANIC and stop. **Never delete files from `pg_wal` by
  hand** — you corrupt the cluster. Find why WAL is retained (a slot, a failing `archive_command`,
  `wal_keep_size`), fix that, and let Postgres recycle. A failing `archive_command` retains WAL
  indefinitely: check `pg_stat_archiver.failed_count`. If WAL is already gone, `pg_resetwal` will let the server start, but
  it discards the record of what was in flight and can leave data inconsistent — use it only to dump
  the data out into a fresh cluster, after trying replicas and backups first. Keep a ballast file (a few GB of pre-allocated
  junk) on the volume to delete in an emergency.
- **Temp files** from spilled sorts and hashes (§6) can fill the data volume; `temp_file_limit` caps
  per-session temp usage.
- **Memory arithmetic**: `shared_buffers` ~25% of RAM; the OS page cache does the rest. `work_mem` is
  per sort/hash node, per backend, and parallel workers each get their own: 200 connections × 3 hash
  nodes × 64 MB = 38 GB. Raising `work_mem` globally to fix one report is how instances get OOM-killed.
  **The Linux OOM killer** killing any backend makes the postmaster terminate every backend and run
  crash recovery — a full outage. Use `vm.overcommit_memory = 2` on dedicated hosts, protect the
  postmaster's OOM score, use huge pages for `shared_buffers`, and disable transparent huge pages
  (latency stalls on Postgres, Redis and MongoDB).
- **`synchronous_commit = off`** per transaction or per role: commits return before the WAL flush; a
  crash can lose the last ~600 ms (3 × `wal_writer_delay`) of *those* transactions, but never corrupts.
  Legitimate for analytics events, sessions, audit fire-hoses — a big throughput win for low-value
  writes, and the honest alternative to `fsync = off` (which risks corruption and must never be used
  in production).
- **Group commit and batching**: many tiny transactions each waiting on an fsync bottleneck on WAL
  flush latency; multi-row `INSERT`, `COPY` and fewer, larger transactions raise throughput by orders
  of magnitude. `COPY` is 5–10× faster than row-by-row inserts.
- **Unlogged tables** skip WAL — fast for staging and scratch data; truncated after a crash and not
  replicated.
- **Cold cache after restart or restore**: after a failover or reboot `shared_buffers` is empty and
  the first minutes are served from disk; `pg_prewarm` (with `autoprewarm`) reloads the previous
  working set. **Restored cloud snapshots load lazily**: an RDS or EBS volume restored from a snapshot
  fetches each block from S3 on first access, so a freshly restored database can be 10–100× slower
  until hydrated — read every block (`pg_prewarm`, a full scan per table, `fio`/`dd` on the device)
  before sending traffic, and include this in the RTO.

### 13. Partitioning and sharding under real load

- **Partition pruning** (`DB24`) needs the partition key in the `WHERE` clause in a form the planner
  can compare: `WHERE created_at >= $1` prunes; `WHERE date(created_at) = $1` or a join to a
  dimension table often does not. Runtime pruning (PG11+) handles parameters and nested-loop inner
  sides; `EXPLAIN` shows `Subplans Removed: N`. A query without the key touches every partition.
- **Too many partitions**: planning time and memory grow with the number of partitions the planner
  must consider; lock acquisition does too (each partition and each of its indexes is a lock — beyond
  the 16 fast-path slots per backend, locks spill to the shared lock manager and `LWLock/LockManager`
  contention appears). Thousands of partitions with queries that do not prune is a latency disaster;
  daily partitions for 10 years = 3,650. Choose coarser ranges and prune reliably.
- **No global unique indexes**: a unique constraint on a partitioned table must include the partition
  key. "Unique email across all partitions" needs a separate lookup table or a different key.
- **Retention by `DROP`/`DETACH`** is the main operational win: dropping a month is instant and
  generates no dead tuples. `DETACH PARTITION ... CONCURRENTLY` (PG14) avoids the `ACCESS EXCLUSIVE`
  on the parent. Creating a new partition while a **default partition** exists scans the default
  partition to check no rows belong to the new range — under a lock; keep the default empty or do
  not have one. Pre-create future partitions (`pg_partman`) — the classic outage is inserts failing at
  midnight on the first of the month because nobody created next month's partition.
- **Hot shards / hot tenants** (`DB25`): hashing by tenant spreads tenants, not load; one enterprise
  tenant with 30% of traffic makes one shard the bottleneck. Options: isolate whale tenants onto
  dedicated shards (a directory/lookup-based placement instead of pure hash), split a tenant by a
  secondary key, or cache its hot reads. Directory-based placement also lets you move a tenant
  without rehashing everyone.
- **Resharding online**: copy the tenant's data to the new shard (logical replication, CDC or
  dual-write), verify with checksums/counts, briefly block writes for that tenant only, flip the
  directory entry, drain. Vitess (`MoveTables`/`Reshard` with VReplication) and Citus
  (`citus_move_shard_placement`) productise this.
- **Cross-shard queries and IDs**: scatter-gather queries are as slow as the slowest shard, and their
  failure probability is the union of all shards'. IDs must be globally unique: Snowflake-style
  (timestamp + worker + sequence) or UUIDv7; avoid per-shard auto-increment collisions. Clock
  skew on Snowflake generators produces duplicates if the clock goes backwards — generators must
  refuse to issue while the clock is behind the last timestamp.

### 14. Time, ordering, CDC and correctness

- **`now()` is the transaction start time** (`DB04`) — a long transaction stamps every row with the
  time it began; `clock_timestamp()` is wall-clock. A batch job that runs for 40 minutes writes rows
  "created" 40 minutes in the past.
- **Commit order is not timestamp order and not ID order.** Transaction A takes `id = 100` and
  `created_at = 12:00:00.000`, then does slow work; B takes `id = 101` at `12:00:00.050` and commits
  first. A poller running `WHERE id > last_seen` (or `updated_at > last_seen`) at 12:00:00.100 sees
  101, advances its watermark, and **never sees 100**. This is how "incremental sync" jobs, ETL
  extracts and home-grown outbox pollers silently lose rows. Fixes: re-read an overlap window
  (`updated_at > last_seen - interval '5 minutes'`, idempotent consumer), only read rows older than
  the longest possible transaction, use `txid`/`pg_snapshot_xmin(pg_current_snapshot())` as a safe
  watermark, or — properly — log-based CDC from the WAL/binlog (Debezium), which is in commit order.
- **Outbox pattern** (`M15`): write the event row in the same transaction as the state change; relay it
  with CDC or a poller that respects the ordering caveat above.
- **Time zones**: store `timestamptz` (UTC instant). `timestamp without time zone` plus servers in
  different zones, or a DST change, produces duplicated or missing hours in reports, jobs that run
  twice or never at 01:30 on the changeover night, and "tomorrow's" orders. Aggregating by day must
  use the user's zone (`date_trunc('day', ts AT TIME ZONE 'Europe/London')`), and an index on
  `created_at` is not used by that expression — precompute the range bounds in UTC instead.
- **Money** (`DB38`): never floats; `numeric` or integer minor units; rounding rules explicit; currency
  stored with the amount. Balances derived from an append-only ledger with double entry; the stored
  balance is a cache that reconciliation checks. **Idempotency keys enforced by a unique constraint**
  — not by a "check then insert" in code — are what stop double charges when a client retries after a
  timeout whose request actually succeeded.
- **Lost updates** (`DB07`, `DB10`): read-modify-write in application code under `READ COMMITTED`
  loses concurrent updates. ORMs that `UPDATE` every column (not just dirty ones) turn two unrelated
  edits to the same row into a lost update. Fix: atomic `SET x = x + 1`, a version column, or
  `SELECT ... FOR UPDATE`. **Write skew** (two doctors both going off call) needs `SERIALIZABLE` or
  materialising the conflict into a row you lock; `SERIALIZABLE` in Postgres means retrying `40001`.
- **Floating correctness bugs that look like database bugs**: duplicate rows because a unique
  constraint was "enforced in code"; phantom duplicates from an at-least-once consumer without a
  dedupe key; `ON CONFLICT DO NOTHING` hiding a real bug; case-sensitive uniqueness on emails
  (`citext` or a unique index on `lower(email)`).

### 15. Backups, recovery, corruption and upgrades

- **Logical vs physical** (`DB36`): `pg_dump` is a consistent snapshot of one database — but the
  snapshot is held for the whole dump (hours, holding the xmin horizon, §3), and restore rebuilds every
  index (a 2 TB database can take a day to restore). Physical base backups plus continuous WAL
  archiving (pgBackRest, WAL-G, Barman, or the managed equivalent) give **point-in-time recovery**.
- **An untested backup is a hope.** Scheduled automated restores into a scratch instance, with a row
  count or checksum check, are the only proof. Restore time is part of the RTO — and includes
  hydration of lazily loaded snapshots (§12) and replaying days of WAL.
- **Recovering from a bad `DELETE` or `UPDATE` without rolling back the world**: restore a PITR copy to
  a *side* instance at a timestamp just before the mistake (`recovery_target_time`, or better
  `recovery_target_xid`/`recovery_target_lsn` found from the logs or `pg_waldump`), then copy the
  affected rows back into production with `INSERT ... ON CONFLICT` or targeted `UPDATE`s — production
  keeps all the good writes made since. Rolling the whole database back is almost never right.
- **A delayed replica** (`recovery_min_apply_delay = '4h'`) is a fast "undo" for fat-finger mistakes:
  pause replay (`pg_wal_replay_pause()`), promote or query it, copy the rows. It is not a backup —
  the same corruption arrives four hours later.
- **Replicas are not backups**: a `DROP TABLE` replicates in milliseconds.
- **Corruption**: enable **data checksums** (`initdb --data-checksums`; `pg_checksums` can enable them
  offline; default on for new clusters from PG18); errors look like `invalid page in block N` or
  `could not read block`. `amcheck` verifies B-tree and heap invariants. `zero_damaged_pages` is a last
  resort that destroys data to let a dump proceed. The first step is always to take a file-level copy
  before trying anything. Common causes: storage/firmware faults, `fsync` lies, OS collation changes
  (§8), someone copying a data directory from a running server.
- **The `fsync` surprise**: before PG12, if the kernel reported an fsync error once and then cleared it,
  Postgres could retry, get success and lose data silently ("fsyncgate", 2018). Modern Postgres PANICs
  on fsync failure and relies on WAL replay — a crash is the correct behaviour.

### 16. ORM behaviour that causes incidents

- **N+1** (`DB40`): a list endpoint issues one query per row; with 50 rows it is fine in dev, with a
  tenant who has 5,000 it is 5,001 queries. Shows up as a huge `calls` count in `pg_stat_statements`
  for a trivial `SELECT ... WHERE id = $1`. Fix with eager loading (`select_related`/`prefetch_related`,
  `JOIN FETCH`, `include`), and add a query-count assertion in tests.
- **Lazy loading outside the transaction** (Hibernate `LazyInitializationException`) tempts people into
  open-session-in-view, which holds a connection (and often a transaction) through view rendering.
- **Parameter limits**: the Postgres wire protocol allows at most **65,535 bind parameters** per
  statement; `WHERE id IN (:ids)` with 70,000 IDs fails, and a bulk insert of 10,000 rows × 8 columns
  fails. Hibernate generates a distinct SQL string per IN-list length, flooding its plan cache and the
  database's statement cache (`hibernate.query.in_clause_parameter_padding`). Use `= ANY(array)`.
- **Batching that is not batching**: `saveAll()` issuing one `INSERT` per entity because identity
  generation (`IDENTITY`) disables JDBC batching in Hibernate; `reWriteBatchedInserts=true` in pgJDBC.
- **Transactions implicitly everywhere or nowhere**: Django autocommit per query vs `ATOMIC_REQUESTS`;
  Spring `@Transactional` ignored on self-invocation (proxy bypass) or on private methods; a
  `readOnly = true` transaction still opening a transaction per call. Each is a real "why is this not
  atomic" or "why is this holding a connection" incident.
- **Migration generators**: an auto-generated migration that renames by drop-and-add (data loss), adds
  a `NOT NULL` column with a default that rewrites on an old engine, or builds indexes non-concurrently.
  Read every generated migration; lint for dangerous operations (§10).
- **Retries at every layer**: the ORM retries, the HTTP client retries, the load balancer retries;
  one slow query becomes 27 identical queries. Retry at one layer, with budgets and jitter.

### 17. MongoDB in production

- **WiredTiger cache** defaults to the larger of 50% of (RAM − 1 GB) or 256 MB. When dirty data in cache
  exceeds its targets (dirty 5% target / 20% trigger, total 80% target / 95% trigger), application
  threads are drafted into eviction — latency spikes across *all* operations. Causes: working set
  larger than cache, huge documents rewritten often, long-running transactions or snapshot reads
  pinning old versions in cache.
- **Documents are rewritten whole** on update in WiredTiger (no in-place update); a 5 MB document with a
  growing array that gets `$push`ed to on every event is a write-amplification machine and eventually
  hits the **16 MB document limit**. Unbounded arrays are the most common Mongo modelling failure
  (`DB27`): use the bucket pattern (one document per N events or per hour) or a child collection.
- **Write concern / read concern**: `w: "majority"` is the default since 5.0. With `w: 1`, writes
  acknowledged by a primary that fails before replicating are **rolled back** when it rejoins — written
  to `rollback/` files on disk, not lost silently but not in the database either. `readPreference:
  secondary` reads are stale and may go backwards across secondaries; causal consistency sessions fix
  read-your-writes.
- **Oplog window**: the oplog is a capped collection; if a secondary (or a change-stream consumer) falls
  further behind than the oplog covers, it cannot catch up and needs a full initial sync (or the
  change stream's resume token becomes invalid — `ChangeStreamHistoryLost`). Size the oplog in hours
  of peak writes, not in GB; `replSetResizeOplog` changes it online.
- **Sharding**: a monotonically increasing shard key (`ObjectId`, timestamp) sends every insert to the
  last chunk — one hot shard while the rest idle; use a hashed key or a compound key with a
  high-cardinality prefix. Queries without the shard key scatter to every shard. Low-cardinality keys
  create **jumbo chunks** the balancer cannot split or move. Chunk migrations add load; restrict the
  balancer window. Since 5.0 shard keys can be refined/resharded online, at significant cost.
- **Indexes**: `COLLSCAN` in `explain` = no index; ESR rule for compound indexes (Equality, Sort,
  Range). An index that does not fit in RAM with random access patterns thrashes. Regex without an
  anchored prefix and `$ne`/`$nin` scan broadly. `$lookup` against an unindexed foreign field is a
  nested collection scan.
- **Transactions**: multi-document transactions default to a 60-second limit
  (`transactionLifetimeLimitSeconds`), hold cache pressure while open, and abort with
  `TransientTransactionError` that must be retried.
- **Connection count**: each driver instance has its own pool (`maxPoolSize` 100 by default) per mongos
  or replica-set member; 200 pods × 100 is 20,000 connections, each a thread on `mongod`.

### 18. DynamoDB and Cassandra

- **DynamoDB partitions** (`DB28`) each serve at most **3,000 RCU and 1,000 WCU** and hold ~10 GB.
  A table with 40,000 WCU provisioned still throttles if one partition key takes more than 1,000
  writes/s. **Adaptive capacity** shifts unused throughput to hot partitions (and isolates hot items
  by splitting) but cannot exceed the per-partition cap for a *single* key. Hot-key fixes: **write
  sharding** (append a suffix `0..N-1` to the key, scatter reads across N and merge), caching hot reads
  (DAX), or a different key design.
- **GSI back-pressure**: a global secondary index has its own capacity; if a GSI is throttled, writes to
  the **base table** are throttled too. A GSI on a low-cardinality attribute (`status`) is a hot
  partition by construction. GSIs are eventually consistent — read-your-writes through a GSI fails
  intermittently.
- **On-demand is not infinitely elastic**: a new on-demand table (or one that has never seen high
  traffic) can instantly handle up to double its previous peak; a launch-day spike far beyond that
  throttles until it scales. Pre-warm by switching to provisioned with a high capacity briefly (or
  the newer warm-throughput setting) before the event.
- **Limits and costs**: items ≤ 400 KB; `Query`/`Scan` pages ≤ 1 MB; `Scan` is billed on data read,
  not returned; filters apply after the read. Transactions cover up to 100 items and cost double.
  LSIs impose a 10 GB limit per partition key value (item collection). **Global tables** replicate
  between regions asynchronously with last-writer-wins conflict resolution: two regions updating the
  same item concurrently means one update silently disappears. Conditional writes are checked only against
  the local region's copy, so they do not prevent this. Home each item's writes in one region (route
  by user), or use the multi-region strong consistency mode where the latency cost is acceptable. TTL deletes are background and
  can lag by days — never rely on TTL for correctness; filter expired items on read.
- **Retries on throttling** without jittered backoff amplify the hot partition; the SDKs back off, but
  custom retry loops often do not.
- **Cassandra tombstones**: deletes, TTL expiry, collection overwrites and **inserting `null` values**
  (a prepared statement binding `null` for an unset column writes a tombstone — use "unset" in the
  driver) all write tombstones. A read must scan tombstones in its slice; past
  `tombstone_warn_threshold` (1,000) it warns, past `tombstone_failure_threshold` (100,000) the query
  fails with `TombstoneOverwhelmingException`. Tombstones are purged only by compaction *after*
  `gc_grace_seconds` (10 days). Using Cassandra as a queue (insert, read, delete) is the textbook way
  to create this.
- **Zombie data**: if a node misses a delete and **repair does not run within `gc_grace_seconds`**, the
  tombstone is purged elsewhere and the missed node's old value resurrects on the next repair or read
  repair. Repairs (Reaper, incremental repair) on a schedule shorter than `gc_grace_seconds` are
  mandatory; lowering `gc_grace_seconds` to "fix" tombstones without faster repair causes resurrection.
- **Large partitions** (hundreds of MB, e.g. a time series without a time bucket in the partition
  key) cause GC pressure, slow reads, and uneven nodes; bucket the partition key (`sensor_id, day`).
- **Compaction strategies**: STCS (default; can need ~50% free disk during a major compaction), LCS
  (read-optimised, more write I/O), TWCS for TTL'd time series (whole SSTables expire and drop — but
  out-of-order writes or read repair mixing old data into new windows defeats it).
- **Last-write-wins by timestamp**: client or node clock skew makes a "later" write lose to an earlier
  one with a skewed-forward timestamp. **LWTs** (`IF NOT EXISTS`) use Paxos — four round trips, and
  mixing LWT and non-LWT writes on the same data breaks their guarantees.
- **Consistency levels**: `QUORUM` read + `QUORUM` write overlap; `LOCAL_QUORUM` per DC. `ONE` for writes
  plus hinted handoff (default hint window 3 hours) means a node down longer than that needs repair.

### 19. Redis as a store and a dependency

- **Single-threaded command execution** (`DB30`; I/O threads exist since 6.0 but commands still run on
  one thread): any O(N) command on a large key blocks every client. `KEYS *` on 50 million keys
  blocks for seconds — use `SCAN`. `DEL` on a hash with 10 million fields blocks — use `UNLINK` (frees
  in a background thread) and `lazyfree-lazy-*` settings. `HGETALL`/`SMEMBERS`/`LRANGE 0 -1` on big
  keys, and long Lua scripts, block the same way. Find big keys with `redis-cli --bigkeys` /
  `--memkeys` and `MEMORY USAGE`; `SLOWLOG GET` shows the offenders.
- **Fork and copy-on-write**: `BGSAVE` and AOF rewrite fork the process; pages modified during the
  save are copied. A write-heavy instance using 12 GB can need up to 24 GB during a save — the OOM
  killer then takes Redis down. Leave headroom (`maxmemory` well below RAM), disable transparent huge
  pages (a 2 MB page copied per small write), and check `latest_fork_usec` in `INFO` — fork itself
  stalls the main thread for tens to hundreds of ms on large heaps. A common arrangement is to
  disable persistence on the primary and take RDB snapshots on a replica instead.
- **Eviction**: `maxmemory-policy` defaults to `noeviction` — at the limit, writes fail with `OOM command
  not allowed`. `volatile-*` policies evict only keys with a TTL; if the app sets no TTLs they behave
  like `noeviction`. A cache should be `allkeys-lru`/`allkeys-lfu`; a store should not be evicting at
  all and needs alerting on memory instead.
- **Expiry**: keys expire lazily on access plus an active sampling cycle. Millions of keys written with
  the same TTL all expire in the same second — CPU spikes in the expiry cycle and a cache stampede on
  the backing database. Add jitter to TTLs.
- **Replication buffers**: a replica that falls behind exceeds `client-output-buffer-limit replica
  256mb 64mb 60`, is disconnected, requests a full resync (another fork + RDB transfer), falls
  behind again — a **full-sync loop**. Raise the limit and `repl-backlog-size` so partial resync works.
- **Durability**: replication is asynchronous; a failover loses acknowledged writes. `WAIT n timeout`
  narrows but does not close the window. AOF `appendfsync everysec` loses up to ~1 s. Redis as the only
  copy of important data needs that trade-off stated out loud.
- **Cluster mode**: 16,384 hash slots; multi-key commands and Lua scripts must touch keys in one slot
  — use hash tags (`{user:42}:cart`). A single hot key lives on one shard no matter how many shards you
  add: split the key (for a hot read, store N copies `product:42#0..N-1` in different slots and read a
  random one; for a hot counter, N sub-counters summed on read), add client-side caching (RESP3
  tracking) or a short-lived in-process cache, or read from replicas.

### 20. Elasticsearch and analytical stores

- **Mapping explosion** (`DB31`): dynamic mapping creates a field for every new JSON key. Indexing
  user-supplied maps (`attributes.{anything}`, log labels, HTTP headers) grows the mapping into tens of
  thousands of fields; the mapping lives in the **cluster state**, which the master must publish to
  every node on each change — master CPU pegs, indexing stalls, the cluster goes yellow/red.
  `index.mapping.total_fields.limit` (1,000 default) is the guard that fails the writes. Fix:
  `dynamic: strict` or `false` at the root, the `flattened` field type for arbitrary maps, or a
  key/value nested array (`[{k, v}]`).
- **Mappings cannot change type in place**: reindex into a new index and **swap an alias** atomically;
  applications should always talk to aliases. For a big reindex: `refresh_interval: -1`,
  `number_of_replicas: 0` on the target, sliced `_reindex`, then restore settings.
- **Shard sizing**: aim for roughly 10–50 GB per shard; thousands of tiny shards (a daily index per
  tenant) inflate cluster state and per-shard overhead — oversharding is the most common ES capacity
  problem. Keep JVM heap ≤ ~31 GB (compressed object pointers) and ≤ 50% of RAM; the rest is file
  system cache for Lucene.
- **Disk watermarks**: low 85% (no new shards allocated to the node), high 90% (shards relocated away),
  flood stage 95% — **every index with a shard on that node becomes read-only**
  (`index.blocks.read_only_allow_delete`), and writes fail with `cluster_block_exception`. Since 7.4 the
  block lifts automatically when disk drops below the high watermark; before that you had to clear it
  by hand after freeing space.
- **Near-real-time**: documents become searchable after a refresh (`refresh_interval` 1 s). "Write
  then immediately search" tests fail intermittently; `?refresh=wait_for` on the write, not
  `refresh=true` in a loop (which creates tiny segments and kills indexing throughput).
- **Deep pagination**: `from + size` ≤ `index.max_result_window` (10,000); each page makes every shard
  collect `from + size` hits. Use `search_after` with a point-in-time (PIT) for deep or export paging.
- **Memory**: `fielddata` on `text` fields (for sorting/aggregating) loads the whole field into heap —
  use `keyword` with doc values. High-cardinality `terms` aggregations and huge `size` trip circuit
  breakers. Leading wildcards (`*foo`) scan the term dictionary.
- **Search is not the source of truth**: rebuildable from the primary database via reindex; dual
  writes from the app drift — prefer CDC → indexer, with versioning (`version_type: external`) so
  out-of-order updates do not overwrite newer ones.
- **ClickHouse and columnar stores** (`DB32`): each `INSERT` creates a part; thousands of tiny inserts
  per second produce "Too many parts" errors (the per-partition threshold is in the hundreds to low
  thousands depending on version) because merges cannot keep up. Batch inserts (≥ 10k rows or ≥ 1 s
  worth), or use `async_insert`, or a Kafka engine / buffer. `ALTER TABLE ... DELETE/UPDATE` are
  *mutations* that rewrite whole parts — never use them as OLTP updates; model with
  `ReplacingMergeTree`/`CollapsingMergeTree` and read with `FINAL` or `argMax`. Partitioning by a
  high-cardinality key creates too many parts as well.
- **Running analytics on the OLTP primary** is the root of a whole family of incidents above: it
  holds the xmin horizon, evicts the OLTP working set from cache, and saturates I/O. The fix is
  architectural: a replica with accepted lag, CDC into a warehouse, or a columnar store.

---

## Questions

### Level 1 — Everyday slowness

The questions nearly every backend interview opens with: a page got slow, a query is hot, the database CPU is up.

1. Our orders list endpoint went from 80 ms to 3 s over two months with no code change. The table has grown from 2 million to 40 million rows and the query is `WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 20`. What do you look at first and what do you change?

> **Direction:** Read `EXPLAIN (ANALYZE, BUFFERS)` and expect either a sort of all the customer's rows or a backwards walk of a `created_at` index filtering by customer; one composite index `(customer_id, created_at)` serves filter and order together (§6, §8; `DB15`, `DB19`).

2. Database CPU jumped from 30% to 90% right after a Friday deploy, but `mean_exec_time` for every query in `pg_stat_statements` looks unchanged. Where is the load coming from?

> **Direction:** Sort by `calls` and `total_exec_time`, not by mean — a new N+1 or a retry loop multiplies call counts of a cheap query; the database is the victim of an application change (§1, §2, §16; `DB39`, `DB40`).

3. A product listing with `OFFSET`-based pagination is fine for users, but every night the database pegs at 100% and the slow log is full of `OFFSET 480000 LIMIT 50`. What is happening and how do you fix it without breaking existing links?

> **Direction:** Crawlers walking deep pages make every request read and discard hundreds of thousands of rows; switch to keyset pagination for next/previous and cap the maximum offset for legacy URLs (§6; `DB20`).

4. The dashboard shows "Total customers: 38,412,907" at the top of every admin page and that page takes 9 seconds. The PM says the number only needs to be roughly right. What do you do?

> **Direction:** `count(*)` is a full scan under MVCC; read `reltuples` from `pg_class` or an `EXPLAIN` estimate, or maintain a counter asynchronously (§6; `DB08`).

5. A support tool searches customers with `WHERE email ILIKE '%' || $1 || '%'` and takes 12 s on 20 million rows. Adding a B-tree on `email` did nothing. What would you do?

> **Direction:** A B-tree cannot serve a leading wildcard or `ILIKE`; use a trigram GIN index (`pg_trgm`), or change the product to a prefix search on `lower(email)` with an expression index (§8; `DB14`, `DB17`).

6. Our login query `WHERE lower(email) = lower($1)` is a sequential scan even though there is a unique index on `email`. The developer insists the index "should work". Explain and fix.

> **Direction:** The index is on `email`, not on the expression; create an expression index on `lower(email)` (or use `citext`) and make the query match it exactly (§8, §14; `DB17`).

7. A MySQL table has a `VARCHAR(20)` column `phone` with an index. Lookups from the new service take 800 ms and sometimes return the wrong customer; the old service's lookups take 1 ms. The only difference is the new service passes `phone` as an integer. Why?

> **Direction:** Comparing a string column to a number casts every row, disabling the index and matching `'0447...'` to `447...`; bind the parameter as a string (§5; `DB04`).

8. A report joins `orders` to `order_items` and aggregates a month of data; `EXPLAIN ANALYZE` shows `Sort Method: external merge Disk: 1.2GB`. The DBA suggests doubling `work_mem` globally. Your view?

> **Direction:** `work_mem` is per sort/hash node per backend and multiplies across connections — raise it with `SET LOCAL` for that report's transaction only, or pre-aggregate (§6, §12; `DB20`).

9. An API endpoint that loads a user with their 30 most recent orders issues 31 queries. It is 40 ms in staging but 2 s in production for enterprise accounts. How do you find and fix this class of problem across the codebase?

> **Direction:** N+1 from lazy loading; fix with eager loading or a single join/`ANY(array)` query, and add a query-count assertion in tests so it cannot regress (§16; `DB40`).

10. A background job does `SELECT * FROM documents WHERE status = 'new'` and processes rows; each row has a 400 KB `body` column. The job is slow and the database's network egress is enormous. What is going on?

> **Direction:** `SELECT *` detoasts every large value; select only the columns needed and fetch bodies separately when required (§9; `DB11`).

11. A search page filters `WHERE status = $1 AND region = $2 AND created_at > $3`. There are three separate single-column indexes and the plan is a `BitmapAnd` of all three, still slow. What index would you build and in what column order?

> **Direction:** One composite index with the equality columns first and the range column last (`status, region, created_at`) beats combining three bitmaps; order equality before range (§8; `DB15`).

12. The same slow query shows up in `pg_stat_statements` with a mean of 40 ms, but users complain about 8-second page loads on that endpoint. How can both be true?

> **Direction:** One `queryid` normalises literals and hides skew — a few large tenants run the same query for seconds; use `auto_explain` samples or log sampling by parameter to find the outliers (§2; `DB39`).

13. A Django admin page with filters on eight columns has become slow and someone proposes adding an index on each column. The table takes 3,000 writes per second. What do you say?

> **Direction:** Every index taxes every write and can kill HOT updates; index only the combinations actually used (check `pg_stat_statements`), consider partial indexes, and move admin search to a replica (§8, §9; `DB13`, `DB17`).

14. A query `WHERE deleted_at IS NULL AND tenant_id = $1` is the hottest in the system. 97% of rows have `deleted_at` set. What index shape is best?

> **Direction:** A partial index `ON (tenant_id) WHERE deleted_at IS NULL` is tiny and hot in cache; the predicate must be written literally in the query for the planner to match it (§6, §8; `DB17`).

15. An index-only scan was the whole point of adding `INCLUDE (amount)` to an index, but `EXPLAIN ANALYZE` shows `Heap Fetches: 2,400,000` and it is barely faster. Why?

> **Direction:** Index-only scans skip the heap only for all-visible pages; vacuum is not keeping up (or it is an insert-only table on old Postgres), so fix vacuum, not the index (§3, §8; `DB16`).

16. A `SELECT ... WHERE id NOT IN (SELECT user_id FROM banned)` suddenly returns zero rows in production after someone inserted a row into `banned` with a null `user_id`. It is also slow. Explain both.

> **Direction:** `NOT IN` with any NULL yields unknown for every row, and it cannot be planned as an anti-join; rewrite as `NOT EXISTS` (§6; `DB04`).

17. A new filter `WHERE a = $1 OR b = $2` on a 50-million-row table is a sequential scan despite indexes on both `a` and `b`. How do you make it fast?

> **Direction:** Rewrite as a `UNION` (or `UNION ALL` with de-duplication) of two indexed queries, or check the planner can use a `BitmapOr`; `OR` across columns often defeats single index paths (§6; `DB20`).

18. A batch importer inserts 2 million rows one `INSERT` per row and takes 90 minutes. The database is barely busy on CPU. What is the bottleneck and how do you get it under five minutes?

> **Direction:** Each commit waits on a WAL fsync — latency bound, not CPU bound; batch with multi-row `INSERT` or `COPY` in larger transactions (and consider `synchronous_commit = off` for the import) (§12; `DB12`).

19. A developer sends 70,000 IDs in `WHERE id IN (...)` and the request fails with an error about too many parameters; with 30,000 IDs it works but Hibernate memory keeps growing. What is going on?

> **Direction:** The Postgres protocol caps bind parameters at 65,535, and each distinct IN-list length is a new SQL string in the ORM's plan cache; pass one array with `= ANY($1)` or join to a temp table (§6, §16; `DB40`).

20. A query is fast when you run it in `psql` with literal values and slow when the application runs it. Nothing else differs. Name your first three hypotheses.

> **Direction:** Generic plan from a prepared statement after the fifth execution, a parameter type mismatch from the driver disabling the index, or different session settings (`search_path`, `work_mem`, role-level settings); reproduce with `PREPARE`/`EXPLAIN EXECUTE` (§6, §7; `DB18`, `DB21`).

### Level 2 — Connections, pools and timeouts

The second most common family: the database is fine but the application cannot reach it, or holds on to it too long.

1. After scaling the API from 20 to 80 pods for a sale, the database started refusing connections with `FATAL: sorry, too many clients already`, even though database CPU is 35%. What happened and what is the durable fix?

> **Direction:** Connections scale with pods × pool size, not with load; put a transaction-mode pooler in front and size per-pod pools against the total at maximum scale (§7; `DB21`).

2. We put PgBouncer in transaction mode in front of Postgres and immediately saw intermittent `prepared statement "S_3" does not exist` errors from our Java services. Why, and what are the two ways out?

> **Direction:** Server-side prepared statements are per server connection, and transaction mode hands you a different one each time; upgrade to PgBouncer 1.21+ with `max_prepared_statements`, or disable server prepares in the driver (`prepareThreshold=0`) (§7; `DB21`).

3. Since introducing PgBouncer, a report sometimes runs with a 5-second statement timeout meant for the web tier, and sometimes the web tier gets the report's 10-minute one. Explain.

> **Direction:** Plain `SET` is session state that leaks to whichever client gets that server connection next in transaction mode; use `SET LOCAL` inside the transaction or per-role settings (§1, §7; `DB21`).

4. Our service freezes completely under moderate load: all threads blocked waiting for a connection, database shows almost no active queries. Pool size is 10. Restarting fixes it for a while. What pattern do you look for in the code?

> **Direction:** A pool deadlock — a request holding one connection inside a transaction asks for a second (`REQUIRES_NEW`, a separate audit write); with N requests each holding one, none can get the second (§7; `DB21`, `DB06`).

5. Every morning at around 08:05 the first requests to our reporting service hang for exactly 15 minutes and then succeed. Overnight there is no traffic. The database is in a different VPC behind a NAT gateway. What is going on?

> **Direction:** The NAT/NLB dropped idle TCP flows after 350 s without telling either side; the pool hands out dead sockets and the kernel retransmits for ~15 minutes — set pool `maxLifetime`/idle timeout below the middlebox limit and enable TCP keepalives (§7; `DB21`).

6. `pg_stat_activity` shows 180 connections in state `idle in transaction`, some for 20 minutes, and autovacuum is falling behind. The app is a Django service. Where are these coming from and what do you set today?

> **Direction:** Per-request transactions held across outbound HTTP calls or slow rendering; set `idle_in_transaction_session_timeout` now, then fix transaction scope so no network call happens inside a transaction (§3, §7; `DB06`).

7. After a planned primary failover on RDS, some pods kept failing writes with `cannot execute INSERT in a read-only transaction` for 40 minutes, while others recovered in 30 seconds. Why the difference?

> **Direction:** Those pods kept pooled connections (or cached DNS) pointing at the old primary, now a replica; evict connections on that error, honour DNS TTL (JVM `networkaddress.cache.ttl`), or use `target_session_attrs=read-write` (§5, §7; `DB23`).

8. A Postgres restart during a maintenance window took 20 seconds, but the service was down for 12 minutes afterwards with the database at 100% CPU mostly in authentication. What happened?

> **Direction:** A reconnect storm — every client reconnecting at once, forking backends and doing SCRAM, hitting `max_connections`, retrying immediately; jittered backoff, a pooler, and lower minimum-idle settings stop the self-sustaining loop (§7; `DB21`).

9. We moved an API to AWS Lambda. At peak we see 1,500 connections to Postgres and memory pressure on the instance, though each function does one query. What do you change?

> **Direction:** Each concurrent invocation holds its own connection and frozen containers keep them; use RDS Proxy or PgBouncer, pool size 1 per function, and cap reserved concurrency (§7; `DB21`).

10. We added RDS Proxy to fix connection counts but the database connection count barely dropped. The metric `DatabaseConnectionsCurrentlySessionPinned` is high. What is pinning, and how do you find the culprit code?

> **Direction:** Session state (`SET`, temp tables, advisory locks, some prepared statements) makes the proxy pin a client to one server connection; find the statements that set session state and move them to `SET LOCAL` or per-role config (§7; `DB21`).

11. Our HTTP gateway times out at 30 s and retries once. During an incident we saw the same slow report query running four times concurrently on the database. Explain and fix.

> **Direction:** The query outlives the request that started it; cancellation does not propagate and retries duplicate work — set `statement_timeout` below the caller's timeout and cancel the query when the request context is cancelled (§7; `DB21`).

12. A team raised their pool size from 20 to 200 per instance "to handle more load" and p99 latency got worse, not better, while throughput stayed flat. Why?

> **Direction:** Beyond a small multiple of cores, more active connections add context switching, lock and buffer contention; a small pool with a queue in front delivers higher throughput (§7; `DB21`).

13. A cron worker uses `pg_advisory_lock(42)` to ensure only one instance runs. After moving behind PgBouncer, sometimes two run at once and sometimes none runs for hours. What is wrong?

> **Direction:** Session-level advisory locks are held by the server connection, not the client, and leak or unlock unpredictably in transaction mode; use `pg_try_advisory_xact_lock` inside a transaction (§4, §7; `DB09`).

14. A service using `LISTEN`/`NOTIFY` for cache invalidation stopped receiving notifications after we introduced a pooler. What happened and what are the options?

> **Direction:** `LISTEN` is session state that transaction pooling cannot preserve; give listeners a dedicated direct (or session-mode) connection, or move invalidation to a real message bus (§7; `DB21`).

15. Our Postgres instance has 64 GB RAM and 2,500 mostly idle connections. It keeps swapping, even though `shared_buffers` is only 16 GB. Where is the memory going?

> **Direction:** Each backend's private memory — catalogue and plan caches grow with the number of tables/partitions touched — times thousands of connections; reduce connections with a pooler and recycle long-lived ones (§7, §12; `DB21`).

16. A health check endpoint runs `SELECT 1` every second from 300 pods. During a database incident, the health checks started failing, Kubernetes restarted all pods, and the outage got longer. What would you have designed differently?

> **Direction:** Health checks that depend on the database turn a database blip into a restart storm and reconnect storm; liveness should not touch the database, readiness should degrade gracefully (§1, §7; `DB21`).

17. We set `statement_timeout = 5s` for the application role, but a nightly migration that adds an index now fails every night. What is the right configuration?

> **Direction:** Use per-role settings — a migration role with a long statement timeout and a short `lock_timeout` — rather than one global value; note role settings apply only to new connections (§1, §4, §10; `DB22`).

18. HikariCP logs `Connection is not available, request timed out after 30000ms` but the database shows only 5 active queries out of a pool of 20. Where are the other 15 connections?

> **Direction:** Checked out but not executing — held in transactions around non-database work, or leaked by code paths that never close them; look for `idle in transaction` and enable Hikari's leak detection (§7; `DB06`, `DB21`).

19. We run two replicas and a primary; the app's connection string lists all three hosts. After a failover the app kept sending writes to a host that had become a replica. What libpq feature would have prevented this?

> **Direction:** A multi-host connection string with `target_session_attrs=read-write` makes the client pick the current primary on connect; combine with pool validation that evicts on read-only errors (§7, §11; `DB23`).

20. Latency on a service jumps every 30 minutes on the dot for about 10 seconds. No cron, no checkpoint at that interval. The pool is HikariCP with defaults. What is your hypothesis?

> **Direction:** `maxLifetime` defaults to 30 minutes, so connections created together are retired together and many requests pay reconnect cost at once; stagger lifetimes (Hikari adds variance — check the version) and keep minimum idle warm (§7; `DB21`).

### Level 3 — Locks, deadlocks and hot rows

Contention incidents: everything is waiting on something, and the thing at the front of the queue looks innocent.

1. A developer ran `ALTER TABLE orders ADD COLUMN notes text` at 14:00 — a metadata-only change — and the whole site went down for 25 minutes. The `ALTER` itself eventually finished in milliseconds. Reconstruct what happened.

> **Direction:** The `ALTER` queued for `ACCESS EXCLUSIVE` behind a long-running query or `idle in transaction` session, and every later `SELECT` queued behind the `ALTER`; always run DDL with a short `lock_timeout` and retries (§4; `DB09`, `DB22`).

2. The same thing happens on MySQL 8: a quick `ALTER TABLE` and all queries on the table show `Waiting for table metadata lock`. `SHOW PROCESSLIST` shows nothing long-running. Where is the blocker?

> **Direction:** An idle connection with an open transaction that once touched the table still holds a shared MDL — it does not appear as running; find it in `performance_schema.metadata_locks` and set a low `lock_wait_timeout` for DDL (§4, §5; `DB09`).

3. We get a few hundred `deadlock detected` errors per day on an endpoint that updates inventory for all items in a basket in one transaction. How do you eliminate them rather than just retrying?

> **Direction:** Two baskets sharing items lock rows in different orders; sort the item IDs and lock them in a consistent order (`SELECT ... ORDER BY id FOR UPDATE`), keep the transaction short, and retry on `40P01` as a safety net (§4; `DB09`).

4. On MySQL, a sign-up flow does `SELECT ... FOR UPDATE WHERE email = ?` and inserts if not found. Under load it deadlocks constantly, even though two users never have the same email. Why?

> **Direction:** Locking a non-existent row takes a gap lock; two sessions share the gap lock and then deadlock on each other's insert-intention locks — just `INSERT` against a unique index and handle the duplicate, or switch to `READ COMMITTED` (§5; `DB09`).

5. A page-view counter `UPDATE articles SET views = views + 1 WHERE id = $1` works fine until a viral article: then p99 for every write on the site climbs to seconds. What is happening and what are the non-obvious fixes?

> **Direction:** One hot row serialises all writers and churns dead tuples; shard the counter across N rows, append-only events aggregated asynchronously, or accumulate in Redis and flush periodically (§4, §19; `DB09`, `DB30`).

6. Inserting into `order_items` got dramatically slower after a teammate added `SELECT ... FOR UPDATE` on the parent `orders` row in the checkout path. They are not even updating `order_items` concurrently. Why?

> **Direction:** Each child insert takes `FOR KEY SHARE` on the parent for the FK check, which conflicts with `FOR UPDATE`; use `FOR NO KEY UPDATE` when the key does not change (§4; `DB09`, `DB05`).

7. Deleting a single customer takes 40 seconds and blocks other activity. `customers` has 2 million rows, `orders` has 300 million and references it with `ON DELETE CASCADE`. What is missing?

> **Direction:** Postgres does not index FK columns automatically; without an index on `orders(customer_id)` each referential check is a scan of `orders` — add the index concurrently, and for large fan-outs delete children in batches first (§4; `DB05`, `DB13`).

8. We use a Postgres table as a job queue. With five workers, throughput is the same as with one, and `pg_stat_activity` shows four workers waiting on `transactionid`. What is the one-keyword fix and what problem will you hit next?

> **Direction:** Without `SKIP LOCKED` every worker convoys on the same first row; with it, the next problem is dead-tuple bloat at the head of the queue index, so tune autovacuum aggressively or partition and truncate (§3, §4; `DB09`).

9. A wallet service reads the balance, checks it in code, then writes the new balance. Under concurrency we saw two withdrawals both succeed and the balance go negative. Without moving to `SERIALIZABLE`, how do you fix it with one statement?

> **Direction:** Make the check part of the write — `UPDATE ... SET balance = balance - $1 WHERE id = $2 AND balance >= $1` and check rows affected — plus a `CHECK (balance >= 0)` constraint as the backstop (§4, §14; `DB10`, `DB38`).

10. A nightly batch job updates 5 million rows in one transaction and during that hour unrelated user requests time out updating their own profiles. The batch does not touch profiles. What could the link be?

> **Direction:** The batch touches rows that user flows also update (e.g. a shared `users.updated_at` or a counter), holds its row locks for the whole hour, and bloats everything behind the horizon; commit in small keyed batches (§3, §4, §10; `DB06`).

11. An on-call engineer tried to fix a lock pile-up by running `pg_terminate_backend` on the oldest blocked query, and nothing improved. What did they get wrong about the lock graph?

> **Direction:** The oldest *waiter* is not the blocker; use `pg_blocking_pids()` to walk to the head of the chain — often an `idle in transaction` session or a DDL waiting for `ACCESS EXCLUSIVE` — and cancel that (§2, §4; `DB39`).

12. On SQL Server, a monthly job that updates 20,000 rows at once suddenly blocks all reads on the table for minutes, while updating 3,000 rows at a time never did. What changed at that threshold?

> **Direction:** Lock escalation from row to table lock at around 5,000 locks per statement; batch below the threshold (Postgres has no escalation, but the same batching is healthy there too) (§4; `DB09`).

13. A reporting query runs `SELECT ... FOR UPDATE` over 2 million rows "to get a consistent view". The WAL volume doubles every time it runs and replicas lag. Why would a read produce WAL?

> **Direction:** Row locks in Postgres are written into tuple headers, dirtying every page and generating WAL; a repeatable-read snapshot gives consistency without locking anything (§4; `DB07`, `DB09`).

14. A cron job acquires `pg_advisory_lock` to be a singleton. After a pod was OOM-killed mid-run, no instance could run the job for six hours. Why, and how do you make it self-healing?

> **Direction:** With a pooler the lock belongs to the server connection, which stayed alive after the pod died; use the transaction-scoped `pg_try_advisory_xact_lock` so the lock dies with the transaction (§4, §7; `DB09`).

15. Our logs show `still waiting for ShareLock on transaction 123456789 after 1000.123 ms`. What does that message tell you precisely, and what setting produced it?

> **Direction:** A row-lock wait on another transaction's xid logged by `log_lock_waits` once it exceeds `deadlock_timeout` (1 s); look up the holder's pid from the log context or `pg_locks` (§2, §4; `DB09`, `DB39`).

16. Checkout uses optimistic locking with a `version` column. During a flash sale on a single product, 95% of checkouts fail with version conflicts and retries make it worse. What would you change for that hot item?

> **Direction:** Optimistic concurrency collapses under high contention on one row; switch the hot path to an atomic conditional decrement, or pre-split the stock into N buckets/reservation tokens (§4; `DB10`).

17. Two microservices each hold a transaction on their own database and call each other over HTTP while holding it. Occasionally both hang for exactly the HTTP timeout. What is the shape of this bug?

> **Direction:** A distributed deadlock invisible to either database's detector — each waits on the other's locks through the network call; never make network calls inside transactions (§4, §7; `DB06`, `M13`).

18. `REINDEX TABLE users` during business hours blocked all writes to the table for 12 minutes, even though reads worked. Which lock did it take, and what should have been used?

> **Direction:** Plain `REINDEX` takes a lock that blocks writes (and `ACCESS EXCLUSIVE` on the index); `REINDEX CONCURRENTLY` (PG12+) rebuilds without blocking (§3, §4, §8; `DB09`).

19. A `TRUNCATE` on a staging table used by an ETL job caused queries on unrelated tables to hang. The ETL runs all its steps in one transaction. Explain.

> **Direction:** `TRUNCATE` takes `ACCESS EXCLUSIVE` held until the transaction ends; everything that touches the truncated table (views, joins, FK checks from other tables) queues behind it — commit early or use `DELETE`/a swap (§4; `DB09`).

20. Deadlock errors started right after we added a trigger that maintains an `order_count` on `customers` when orders are inserted. Why would a trigger introduce deadlocks?

> **Direction:** Each insert now also locks the parent customer row, in an order that conflicts with other code paths updating customers then orders; denormalised counters on parents create hot rows — maintain them asynchronously or with consistent lock ordering (§4; `DB09`).

### Level 4 — Schema changes and backfills

Every senior has broken production with a migration at least once. These questions check you learnt from it.

1. We need to add `NOT NULL` to a `status` column on a 600-million-row table without downtime. The naive `ALTER TABLE ... SET NOT NULL` locks it for 40 minutes in staging. What sequence would you use?

> **Direction:** Add `CHECK (status IS NOT NULL) NOT VALID`, `VALIDATE` it under a weak lock, then `SET NOT NULL` (PG12+ uses the validated check and skips the scan), then drop the check (§10; `DB22`).

2. A migration renamed `users.fullname` to `users.display_name`. During the rolling deploy, half the pods threw errors for eight minutes. What process would have avoided it?

> **Direction:** Expand/contract: add the new column, dual-write, backfill, switch reads, stop writing old, drop later — each a separate deploy compatible with the previous code version (§10; `DB22`, `M27`).

3. We dropped a column the new code no longer uses and the old pods, still running during the deploy, started failing with `column "legacy_flag" does not exist` on a plain `save()`. Why would old code reference it if it never reads it?

> **Direction:** ORMs cache the entity's column list and `INSERT`/`SELECT` every mapped column; first remove the column from the model or mark it ignored, deploy, and drop only in a later release (§10, §16; `DB40`).

4. A backfill `UPDATE users SET tier = compute(...)` over 80 million rows in one statement ran for three hours, filled the disk with WAL, and replicas fell two hours behind. Design the backfill properly.

> **Direction:** Batch by primary-key range, commit each batch, throttle on replica lag and WAL rate, make it resumable and idempotent, and run it as a job outside the migration (§10, §11, §12; `DB22`).

5. We need to change `orders.id` from `integer` to `bigint` on a 1.9-billion-row table; `ALTER COLUMN TYPE` would lock it for hours. What is the plan?

> **Direction:** New `bigint` column kept in sync by trigger, batched backfill, concurrent unique index, then a short swap transaction (rename columns, move the sequence and constraints, update FKs) (§9, §10; `DB22`).

6. `CREATE INDEX CONCURRENTLY` on a busy table has been running for four hours and progress shows it stuck at "waiting for old snapshots". The table only takes 20 minutes to scan. What is it waiting for?

> **Direction:** Each phase waits for every transaction that could see the table — including unrelated long transactions anywhere; find and end them via `backend_xmin` in `pg_stat_activity` (§3, §8; `DB22`).

7. A `CREATE UNIQUE INDEX CONCURRENTLY` failed halfway on a duplicate value. Two weeks later writes to that table are noticeably slower than before. Why?

> **Direction:** A failed concurrent build leaves an `INVALID` index that is still maintained on every write but never used; find it with `pg_index.indisvalid = false` and drop it (§8; `DB22`).

8. Our framework runs each migration in a transaction. A migration that adds a column and then updates 5 million rows took the table offline for 20 minutes. What happened, even though adding a column is instant?

> **Direction:** The `ACCESS EXCLUSIVE` lock from the `ALTER` is held until the transaction commits — through the whole update; split DDL and data changes and batch the data change (§4, §10; `DB22`).

9. We're adding a foreign key from `payments.order_id` to `orders.id`. Both tables are large and hot. How do you add it without blocking writes for the duration of the validation scan?

> **Direction:** `ADD CONSTRAINT ... NOT VALID` (brief lock, enforces new rows), then `VALIDATE CONSTRAINT` in a separate step under a weaker lock, with `lock_timeout` on both (§4, §10; `DB05`, `DB22`).

10. A teammate proposes `ALTER TABLE events ADD COLUMN id uuid DEFAULT gen_random_uuid()` on a 500-million-row table, arguing "Postgres 11+ makes defaults instant". Is that right?

> **Direction:** Only non-volatile defaults are metadata-only; a volatile default like `gen_random_uuid()` forces a full table rewrite under `ACCESS EXCLUSIVE` — add the column nullable and backfill in batches (§10; `DB22`).

11. We changed a column from `timestamp` to `timestamptz` "because it's the right type" and the migration locked the table for an hour. Which type changes are free, and which rewrite?

> **Direction:** Binary-compatible changes (`varchar(n)` to larger or to `text`) are metadata-only; most others, including `int`→`bigint` and usually `timestamp`→`timestamptz`, rewrite the table — use the add-column-and-backfill pattern (§10; `DB04`, `DB22`).

12. On MySQL 5.7 a 3-hour `ALTER TABLE` completed on the primary with online DDL, but then every replica lagged by three hours. Why, and what tool would you use next time?

> **Direction:** Replicas replay the DDL serially after it finished on the primary; use gh-ost (binlog-based, throttles on replica lag, controllable cut-over) or `ALGORITHM=INSTANT` where available (§5, §11; `DB22`, `DB23`).

13. A dual-write migration copied `address` into a new `addresses` table via a backfill. Afterwards, 0.3% of users had outdated addresses in the new table. The dual-write code was deployed before the backfill started. What race did we miss?

> **Direction:** The backfill read a row, a live update dual-wrote the new value, then the backfill wrote its stale copy over it; backfills must not overwrite newer data (upsert with a version/`updated_at` guard or `ON CONFLICT DO NOTHING`), and verify with a reconciliation pass (§10; `DB22`, `DB10`).

14. We swapped `orders` with `orders_new` via renames in one transaction. Afterwards a reporting view and two functions still read from the old data. Why?

> **Direction:** Views, functions and FKs reference tables by OID, not by name, so they follow the renamed old table; check `pg_depend` and recreate dependents in the swap (§10; `DB22`).

15. A new partition for next month failed to be created by a cron that had been silently broken; at 00:00 on the 1st all inserts into `events` started failing. How do you prevent and how do you recover in the moment?

> **Direction:** Create the partition immediately (and backfill any rows parked elsewhere), then pre-create partitions months ahead with `pg_partman` and alert on the furthest existing partition; avoid relying on a default partition, which makes later attaches scan it under lock (§13; `DB24`).

16. Adding a column to a hot table on Postgres is fast, but our migration runner occasionally times out after 2 seconds of `lock_timeout` and the deploy fails. Management wants the timeout removed. What do you propose?

> **Direction:** The timeout is protecting production from the lock queue; keep it and add automatic retries with backoff and jitter, and schedule around long-running jobs (§4, §10; `DB22`).

17. A migration generated by the ORM for a "rename" drops the old column and adds a new one. It passed code review. What checks should exist in the pipeline to catch this and other dangerous migrations?

> **Direction:** Lint generated migrations for destructive and locking operations (drop, rename, type change, non-concurrent index, `SET NOT NULL`, volatile default) with a tool like `squawk` or `strong_migrations`, and require humans to read the SQL (§10, §16; `DB22`, `DB40`).

18. We need a unique constraint on `(tenant_id, external_ref)` on a live 200 GB table. How do you add it so that the lock is only held for milliseconds?

> **Direction:** `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... UNIQUE USING INDEX` which is metadata-only; clean duplicates first or the build fails and leaves an invalid index (§8, §10; `DB05`, `DB22`).

19. A data migration needs to delete 70% of a 1 TB `audit_log` table and keep the rest. Batched deletes are projected to take nine days and bloat the table. What is the faster, cleaner alternative?

> **Direction:** Copy the 30% you keep into a new table (or partition going forward), swap names in a short transaction, and drop the old one — no dead tuples, no vacuum debt (§3, §10; `DB24`).

20. After a major upgrade from Postgres 13 to 16 using `pg_upgrade --link`, the maintenance finished in 6 minutes and then the site was unusably slow for an hour. What step was skipped?

> **Direction:** `pg_upgrade` (before PG18) does not carry planner statistics; run `vacuumdb --all --analyze-in-stages` immediately as part of the runbook, before opening traffic (§6, §10; `DB18`).

### Level 5 — Replicas, lag and read consistency

Read scaling works until the product notices the data is stale. Asked in almost every interview that mentions replicas.

1. Users save their profile, get redirected to the profile page and see the old value; a refresh fixes it. Reads go to a replica with ~200 ms typical lag. The product team wants it fixed without sending all reads to the primary. Options?

> **Direction:** Read-your-writes by routing that user's reads to the primary for a short window after a write, or waiting until the replica's replay LSN passes the LSN captured after the write (§11; `DB23`, `M12`).

2. With three read replicas behind a load balancer, a list page sometimes shows an order, then after refresh does not, then does again. How do you explain this to the PM and fix it?

> **Direction:** Each replica is at a different replay point, so successive reads go backwards in time; use session stickiness to one replica, or LSN-based monotonic reads, or read that view from the primary (§11; `DB23`, `M12`).

3. Our replica-lag alert pages every night at 03:00 saying lag is 40 minutes, but when the on-call checks, data is current. The metric is `now() - pg_last_xact_replay_timestamp()`. What is wrong?

> **Direction:** With no writes on the primary the last replayed timestamp stops moving, so "lag" grows while the replica is fully caught up; measure bytes behind or use a heartbeat row written every second (§11; `DB23`, `DB39`).

4. Analysts running long queries on the read replica keep getting `canceling statement due to conflict with recovery`. The quick fix someone applied was `hot_standby_feedback = on`. A month later the primary is 3× bigger. Connect the dots.

> **Direction:** Feedback makes the primary keep rows the replica's long queries might need, so vacuum cannot clean anything — bloat on the primary; give analytics a dedicated replica with a large replay delay tolerance, or move it to a warehouse (§3, §11; `DB23`, `DB32`).

5. The primary's disk went from 40% to 100% in six hours and Postgres stopped. Data size barely grew; `pg_wal` is 900 GB. Nobody changed WAL settings. What do you check first, and what must you never do?

> **Direction:** An inactive replication slot (a dead replica or a paused CDC connector) retaining WAL, or a failing `archive_command`; never delete files in `pg_wal` by hand — drop the slot or fix archiving, and set `max_slot_wal_keep_size` (§11, §12; `DB12`, `DB23`).

6. We made replication synchronous for durability. Last night one standby host died and the entire application stopped being able to write, although the primary was healthy. Why, and how should it have been configured?

> **Direction:** With a single synchronous standby, commits wait forever for its acknowledgement; use `ANY 1 (s1, s2)` with two standbys, and have a runbook to drop to async (§11; `DB23`, `DB12`).

7. A replica usually has sub-second lag, but every Sunday it falls 45 minutes behind during a maintenance job on the primary, even though the replica's CPU and disks are mostly idle. Why is it not keeping up?

> **Direction:** WAL replay is a single process; a large bulk operation (index rebuild, mass update, big vacuum) produces WAL faster than one process can apply, whatever the replica's hardware — throttle or batch the job (§11, §12; `DB23`).

8. On MySQL, `Seconds_Behind_Source` shows 0, yet the replica is clearly missing the last 10 minutes of orders. How can the metric be 0?

> **Direction:** It measures the applier's position relative to what the IO thread has received; if the IO thread is stalled the applier is "caught up" with nothing — use a heartbeat table (`pt-heartbeat`) for real lag (§11; `DB23`).

9. A single `DELETE FROM sessions WHERE expires_at < now()` removed 20 million rows on MySQL, taking 6 minutes on the primary. Every replica then lagged by at least 6 minutes. Why and what should the job do instead?

> **Direction:** The whole transaction is applied as one unit on replicas after it commits on the primary; delete in small batches (or drop partitions) so replicas interleave and parallel apply can help (§11, §13; `DB23`, `DB24`).

10. After an automatic failover, customers reported that orders they had been shown confirmations for no longer existed. Replication was asynchronous. Explain and describe how to quantify and recover what was lost.

> **Direction:** Commits acknowledged by the old primary but not yet shipped are missing on the new one; extract the divergent transactions from the old primary's WAL (or before `pg_rewind`) and replay or reconcile them, then decide whether critical writes need synchronous replication (§11, §15; `DB23`, `DB36`).

11. After failover, inserts started failing with duplicate key violations on `orders_pkey` for a few minutes, then stopped. Why would the new primary hand out IDs that already exist?

> **Direction:** Sequence state on the new primary can lag the values the old primary already issued (lost async commits, or logical replication not carrying sequences); advance sequences past the maximum ID in the failover runbook (§9, §10, §11; `DB23`).

12. We use Debezium for CDC. The connector was paused for a week during a migration. Now the primary's catalog tables are bloated and disk is growing. What is the mechanism?

> **Direction:** A logical slot holds both WAL and `catalog_xmin`, so catalogues cannot be vacuumed and WAL piles up while the connector is paused; monitor slot lag, cap it with `max_slot_wal_keep_size`, and drop slots of abandoned consumers (§3, §11; `DB23`).

13. A cross-region read replica is used by EU users to reduce latency. After a network blip the replica was 20 minutes behind and EU users saw stale prices at checkout. What design rule was broken?

> **Direction:** Correctness-critical reads (price, stock, balance at checkout) must go to the primary or validate against it; cross-region replicas are for tolerant reads and DR, with a lag threshold that ejects them from rotation (§11; `DB23`, `M12`).

14. A split-brain occurred: the old primary came back after a network partition and accepted writes for 4 minutes before being fenced. How do you reconcile the two histories?

> **Direction:** Before rejoining it with `pg_rewind` (which discards its divergent writes), extract those writes via logical decoding or `pg_waldump`/row diffs, then reconcile at the business level; improve fencing so it cannot accept writes after failover (§11, §15; `DB23`, `DB36`).

15. Our Aurora cluster's writer failed over; the app came back after 35 seconds except for one Java service that kept connecting to the old writer's IP for ten minutes. Why?

> **Direction:** The JVM cached the DNS answer for the cluster endpoint; set `networkaddress.cache.ttl` low and have the pool evict connections on read-only/connection errors (§5, §7; `DB23`).

16. We added `max_standby_streaming_delay = -1` on the reporting replica to stop query cancellations. Now reports run fine but the lag regularly reaches hours. Is that acceptable and what else could you do?

> **Direction:** It trades cancellations for unbounded lag, acceptable only if that replica is explicitly a lagging analytics copy excluded from app reads; alternatives are feedback (bloat on primary) or a logical/warehouse copy (§11; `DB23`, `DB32`).

17. A `VACUUM` on the primary truncated the end of a large table, and at the same moment several long reports on the replica were cancelled even though `hot_standby_feedback` is on. Why didn't feedback protect them?

> **Direction:** Feedback prevents cleanup conflicts, not lock conflicts; the truncation takes `ACCESS EXCLUSIVE`, whose replay cancels conflicting replica queries — disable `vacuum_truncate` for that table (§3, §11; `DB23`).

18. On MySQL, replicas occasionally diverge from the primary — a few rows have different values — and nobody knows why. The binlog format is `STATEMENT`. What is your hypothesis and your check?

> **Direction:** Non-deterministic statements (`UPDATE ... LIMIT` without `ORDER BY`, `UUID()`) replay differently; switch to row-based replication and verify with `pt-table-checksum` (§11; `DB23`).

19. A replica reads from a table whose index was built on the primary before an OS upgrade on the replica host. Queries by email on the replica sometimes return nothing for users who exist. The primary finds them. What happened?

> **Direction:** The replica's newer glibc sorts strings differently, so the physically replicated B-tree no longer matches its comparator; never mix OS/collation versions in a physical replication set — rebuild the replica on the same version, check with `amcheck` (§8, §11; `DB13`, `DB23`).

20. You're asked to route "read-only" API endpoints to replicas automatically based on HTTP method. What goes wrong with that heuristic in practice?

> **Direction:** GET handlers that write (sessions, counters, lazy creation) fail on replicas, and reads immediately after a write elsewhere see stale data; route by explicit intent and consistency need, with read-your-writes handling (§11; `DB23`, `M12`).

### Level 6 — Vacuum, bloat, disk and WAL

The Postgres-specific questions that separate people who have run it from people who have used it.

1. A 20 GB table has grown to 300 GB over six months while the row count stayed constant. `n_dead_tup` is huge and `last_autovacuum` is weeks ago. What do you look for first?

> **Direction:** Something holding the xmin horizon — a long transaction, `idle in transaction`, a stale replication slot, a prepared transaction, or feedback from a replica — so vacuum cannot remove anything; fix that, then tune per-table autovacuum and reclaim with `pg_repack` (§3; `DB08`).

2. Autovacuum is running constantly on our biggest table but never seems to finish, and the table keeps growing. `pg_stat_progress_vacuum` shows it scanning indexes over and over. What settings would you change?

> **Direction:** Throttled by the shared cost limit and a small `maintenance_work_mem` forcing repeated index passes; raise `autovacuum_vacuum_cost_limit` and `maintenance_work_mem`, and lower the table's scale factor so vacuums are smaller and more frequent (§3; `DB08`).

3. The logs show `WARNING: database "app" must be vacuumed within 38000000 transactions`. The team has never heard of this. How serious is it, what exactly happens next, and what do you do in the next hour?

> **Direction:** Xid wraparound protection — at around 3 million remaining Postgres stops accepting writes; remove whatever holds the horizon, then run manual `VACUUM (FREEZE)` on the oldest tables by `age(relfrozenxid)`, and monitor far earlier (§3; `DB08`).

4. A 2 ms `ALTER TABLE` migration hung for an hour behind `autovacuum: VACUUM public.events (to prevent wraparound)`. The engineer killed the autovacuum three times and it kept coming back. What should they have done?

> **Direction:** Anti-wraparound autovacuum does not yield and must complete; let it finish (raise its cost limit if needed), reschedule the DDL, and in future run manual freezes in quiet windows so it never triggers at a bad time (§3, §4; `DB08`).

5. A queue table has never more than 5,000 live rows, yet dequeue latency climbed from 1 ms to 400 ms over a day. `VACUUM VERBOSE` says "90000 dead row versions cannot be removed yet". What is going on?

> **Direction:** Dead tuples at the head of the dequeue index cannot be removed because a long-running transaction elsewhere holds the horizon; find and kill it, then tune the queue table's autovacuum aggressively or partition and truncate (§3, §4; `DB08`, `DB09`).

6. We deleted 80% of a 1 TB table last week. Disk usage has not changed at all and the finance team wants the storage cost back. What are the options, and what are the risks of each?

> **Direction:** Vacuum only makes space reusable; returning it needs a rewrite — `VACUUM FULL` (long `ACCESS EXCLUSIVE` outage) or `pg_repack` (online, needs free disk ≈ table + indexes, and a PK) (§3; `DB11`).

7. An update-heavy `sessions` table has a HOT ratio of 2%. Six months ago it was 95%. Nothing changed in the query that updates it. What probably changed, and how do you restore it?

> **Direction:** Someone added an index on an updated column (often `updated_at`/`last_seen_at`), making every update non-HOT; drop or rethink that index, and set `fillfactor` below 100 so pages have room (§9; `DB11`, `DB13`).

8. We store a user's settings as a single 800 KB JSONB document and update one key on every page view. WAL volume is 2 TB a day and replicas lag. What is the root cause and the fix?

> **Direction:** Updating any part of a TOASTed value rewrites and WAL-logs the whole value; move frequently changing fields into their own columns or table (§9, §12; `DB11`).

9. Write latency has a sawtooth pattern: fine for two minutes, then spikes, repeatedly. WAL volume looks much larger than the data changes justify. `log_checkpoints` shows checkpoints every 90 seconds. What do you change?

> **Direction:** `max_wal_size` is too small, forcing frequent checkpoints and a full-page image per page after each; raise `max_wal_size` and `checkpoint_timeout`, keep `checkpoint_completion_target` at 0.9, enable `wal_compression` (§12; `DB12`).

10. After a sustained import our RDS instance's write latency jumped 10× and stayed there, though IOPS were far below the provisioned number we thought we had. The volume is gp2, 300 GB. What happened?

> **Direction:** gp2's baseline is 3 IOPS/GB (900 here); the import drained the burst credit bucket — watch `BurstBalance`, move to gp3 with provisioned IOPS, or check the instance's EBS bandwidth cap (§12; `DB11`).

11. An append-only `events` table on Postgres 12 has index-only scans that keep visiting the heap, and it has never been autovacuumed despite billions of inserts. Why, and what changed in later versions?

> **Direction:** Before PG13 autovacuum triggered only on dead tuples, so insert-only tables were never vacuumed and the visibility map stayed empty; schedule manual vacuums or upgrade to insert-triggered autovacuum (§3, §8; `DB16`).

12. The data disk filled up at 3am because of temp files from a single ad-hoc query. What guard rails should have been in place?

> **Direction:** `temp_file_limit` per session, `statement_timeout` for ad-hoc roles, `log_temp_files` to see it coming, and analytics routed away from the primary (§1, §6, §12; `DB39`).

13. The Postgres process was killed by the Linux OOM killer and the whole database restarted, although only one report used lots of memory. Why did one backend dying take everything down, and how do you size memory to avoid it?

> **Direction:** Any backend killed abnormally forces the postmaster to reset all backends and run crash recovery; budget `work_mem` × nodes × connections, use `vm.overcommit_memory = 2` and protect the postmaster (§12; `DB12`).

14. A log table receives 50,000 inserts per second and we lose nothing if the last second is lost on crash. The team wants to set `fsync = off` to go faster. What is the correct lever instead?

> **Direction:** `synchronous_commit = off` for that role or those transactions loses at most ~600 ms on a crash without risking corruption; `fsync = off` risks the entire cluster (§12; `DB12`).

15. After moving from `bigserial` keys to UUIDv4 on a high-insert table, WAL volume tripled and the buffer cache hit ratio dropped from 99% to 85%. Explain the mechanism.

> **Direction:** Random keys dirty a random leaf page per insert, so the whole index is the working set and each checkpoint triggers many new full-page images; use time-ordered IDs such as UUIDv7 (§8, §12; `DB13`).

16. We are running out of disk and the `pg_wal` directory is large. The archive bucket has not received a file for two days. What happened and what is the quick check?

> **Direction:** A failing `archive_command` makes Postgres retain every WAL segment; check `pg_stat_archiver` for `failed_count` and `last_failed_wal`, fix credentials/permissions, and alert on archive age (§12, §15; `DB36`).

17. Our biggest table shows `n_dead_tup` of 0 and autovacuum runs fine, but `pgstattuple` reports 60% free space. Queries scan far more pages than the row count suggests. Is this bloat, and what would you do?

> **Direction:** Yes — space made reusable by vacuum but never returned or refilled; if the table will not grow into it, rewrite online with `pg_repack` during a quiet period, possibly with a lower `fillfactor` if it is update-heavy (§3, §9; `DB11`).

18. A JSONB table with a GIN index has periodic 3-second insert latency spikes, roughly every few minutes, while the average is 5 ms. What is the likely cause?

> **Direction:** The GIN pending list fills up and the insert that crosses `gin_pending_list_limit` flushes it synchronously; set `fastupdate = off` or a smaller limit and vacuum more often (§8; `DB14`).

19. A multi-tenant SaaS runs `pg_dump` of the whole production database every night for "backups". The dump now takes 9 hours and the daytime performance is getting worse each week. Why would a backup affect daytime performance?

> **Direction:** `pg_dump` holds one snapshot for its full duration, holding back the xmin horizon cluster-wide so bloat builds up; use physical backups with WAL archiving, and dump only what needs a logical copy (§3, §15; `DB36`).

20. Autovacuum is keeping up on every table, yet one table's multixact age keeps rising and SLRU waits appear in `pg_stat_activity`. The table is `merchants`, referenced by the very hot `payments` table. What is the mechanism?

> **Direction:** Every FK insert takes `FOR KEY SHARE` on the parent row; concurrent inserts referencing the same merchant create multixacts, so monitor `mxid_age` and multixact freezing, and consider reducing concurrent share-locking on hot parents (§3, §4; `DB08`, `DB09`).

### Level 7 — Planner surprises and plan flips

The query did not change; the plan did. Asked by teams that have been bitten, and the answers are rarely "add an index".

1. A query that ran in 20 ms for a year suddenly takes 90 s after last night's autoanalyze. No code or data-shape change we know of. How do you get production back now, and how do you stop it happening again?

> **Direction:** Capture the bad plan (`auto_explain`), compare estimates with actuals to find the misestimated node; stabilise now with a targeted rewrite or `SET LOCAL` planner setting for that query, then fix the estimate with higher statistics targets or extended statistics (§2, §6; `DB18`, `DB19`).

2. A tenant-filtered query is fast for most tenants but takes 40 s for our largest tenant, and only after the application has been running for a while — restarting the app "fixes" it for a few minutes. What is the mechanism?

> **Direction:** After five executions the prepared statement switches to a generic plan costed for an average tenant, which is terrible for the whale; set `plan_cache_mode = force_custom_plan` for that role or query, or inline the tenant literal (§6; `DB18`, `DB21`).

3. `EXPLAIN ANALYZE` shows `Nested Loop (rows=8) (actual rows=1,900,000)` on a join filtered by `country = 'DE' AND city = 'Berlin'`. Statistics are fresh. What is the planner getting wrong and how do you fix it without hints?

> **Direction:** Correlated columns: the planner multiplies independent selectivities; `CREATE STATISTICS ... (dependencies, mcv) ON country, city` and re-`ANALYZE` gives it the joint distribution (§6; `DB18`, `DB19`).

4. `SELECT * FROM events WHERE user_id = $1 ORDER BY id DESC LIMIT 20` is 2 ms for active users and 3 minutes for users who have not been active for years. Why is the same query so different, and what is the emergency fix?

> **Direction:** The planner walks the `id` index backwards hoping to find 20 matching rows early; for inactive users it scans almost the whole index — force the filter-first plan with `ORDER BY id + 0`, then add `(user_id, id)` (§6; `DB15`, `DB18`).

5. After upgrading from Postgres 11 to 13, a complex reporting query got 20× slower. It uses several `WITH` clauses that the author wrote "to control the join order". What changed?

> **Direction:** From PG12 single-reference CTEs are inlined, removing the optimisation fence the author relied on; add `AS MATERIALIZED` where the fence was intentional (§6; `DB03`, `DB18`).

6. A simple indexed lookup query sometimes takes 300 ms instead of 3 ms, and `EXPLAIN ANALYZE` shows a `JIT:` section with 290 ms of timing. What is happening and what setting would you change?

> **Direction:** A bad cost estimate pushed the query over `jit_above_cost`, so it compiled code for a tiny query; turn JIT off for OLTP (`jit = off`) or raise the thresholds, and fix the estimate (§6; `DB18`).

7. A nightly job loads the day's data, then immediately runs aggregation queries filtered by today's date, which are always terribly slow — yet the same queries are fast when rerun manually two hours later. Why?

> **Direction:** Statistics still describe yesterday (histogram ends before today), so the planner estimates almost no rows for today; run `ANALYZE` on the table inside the job right after loading (§6; `DB18`).

8. A job creates a temporary table with 3 million rows and joins it to `orders`. The join always uses a nested loop and takes an hour. Autovacuum is on. Why is the plan so bad?

> **Direction:** Autovacuum never analyses temporary tables, so the planner has no statistics; run `ANALYZE` on the temp table after filling it (§6; `DB18`).

9. We have a partial index `WHERE status = 'pending'` that makes the dashboard fast in `psql`, but from the application, which binds `status` as a parameter, the index is never used. Why?

> **Direction:** A generic plan for `status = $1` cannot prove the partial predicate; put the literal in the SQL or force custom plans for that statement (§6, §8; `DB17`).

10. On SQL Server, lookups on a `VARCHAR(32)` primary key from a Java service are index scans taking 400 ms, while the same query from SSMS is an index seek. What is the driver doing?

> **Direction:** The JDBC driver sends strings as `NVARCHAR` by default, forcing an implicit conversion of the column; set `sendStringParametersAsUnicode=false` (§6; `DB04`).

11. A query plan uses a sequential scan on a 100 GB table even though an index lookup for the requested range would touch 2% of rows. The database runs on NVMe SSDs with default configuration. What cost setting is likely to blame?

> **Direction:** `random_page_cost = 4.0` assumes spinning disks, making index access look far too expensive; lower it (around 1.1) for SSD storage, tested on representative queries (§6; `DB18`).

12. A column `status` has 12 distinct values, one of which covers 0.01% of 800 million rows. Queries for that rare value sometimes use a sequential scan. Statistics target is the default. What is happening and what do you tune?

> **Direction:** The 30,000-row sample may not include the rare value in the MCV list, so it gets a generic estimate; raise that column's statistics target, or use a partial index for the rare value (§6, §8; `DB18`).

13. The query planner picked a merge join that sorts 40 million rows, and the sort spilled to disk. `EXPLAIN` estimated 40,000 rows. How do you find which estimate went wrong first, and why does "first" matter?

> **Direction:** Read `EXPLAIN (ANALYZE, BUFFERS)` bottom-up and find the lowest node where estimated and actual rows diverge; everything above it inherits the error, so fixing that node fixes the rest (§6; `DB19`).

14. Someone added `SET enable_seqscan = off` to the application's connection initialisation "because seq scans are bad". What are the consequences you would expect?

> **Direction:** Global planner switches force terrible plans for queries where a scan is correct and hide real estimate problems; use them only `SET LOCAL` inside a single transaction as a diagnostic or scalpel (§6; `DB18`).

15. A query joining 14 tables is planned very differently each time and sometimes takes 50× longer. Statistics look fine. What planner limit are you hitting?

> **Direction:** Beyond `join_collapse_limit`/`from_collapse_limit` (8) the planner stops searching all join orders and beyond `geqo_threshold` (12) uses the genetic optimiser, a heuristic that can land on much worse orders; raise the limits for that query or restructure joins explicitly (§6; `DB18`).

16. After a partitioned table grew to 4,000 partitions, simple primary-key lookups that include the partition key went from 1 ms to 40 ms. The execution time in `EXPLAIN ANALYZE` is still 0.2 ms. Where did the time go?

> **Direction:** Planning time and lock acquisition across thousands of partitions (fast-path lock slots exhausted, lock manager contention); check `Planning Time`, reduce partition count, and make pruning happen at plan time (§6, §13; `DB24`).

17. A query with `WHERE created_at::date = $1` on a partitioned table scans every partition, while the same query with a range scans one. Why?

> **Direction:** Pruning (and index use) needs the partition key compared directly; a cast or function on the column defeats it — rewrite as a half-open range on the raw column (§6, §13; `DB17`, `DB24`).

18. `SELECT ... WHERE id = ANY($1)` with 5,000 IDs is fast, but the ORM generates `WHERE id IN ($1, ..., $5000)` and different list lengths create thousands of distinct plans in `pg_stat_statements` and the ORM's cache. What would you change?

> **Direction:** Pass one array parameter with `= ANY`, or enable IN-list padding in the ORM, so there is one statement shape to plan and cache (§6, §16; `DB40`).

19. A planner regression happens every Monday morning, after a weekend of batch deletes. By Tuesday it is gone without intervention. Explain the timeline.

> **Direction:** Mass deletes shift distributions and the table's `reltuples`/stats lag until autoanalyze triggers at the 10% threshold; analyse explicitly at the end of the batch job (§3, §6; `DB18`).

20. You cannot change the application's SQL for two weeks (vendor code), yet one query's plan is causing an outage. List what you can do from the database side alone.

> **Direction:** Fix estimates (statistics targets, extended statistics, `n_distinct` overrides), add or drop an index to change the available paths, per-role settings (`plan_cache_mode`, `random_page_cost`, `work_mem`), and `pg_hint_plan` hints registered by query text (§1, §6; `DB18`, `DB20`).

### Level 8 — NoSQL and specialised stores under load

The same kinds of incidents in the other engines on a typical stack: MongoDB, DynamoDB, Cassandra, Redis, Elasticsearch, ClickHouse.

1. A DynamoDB table is provisioned at 40,000 WCU and consumed capacity is 8,000, yet writes are being throttled. The key is `tenant_id`. What is happening, and what do you change in the key design?

> **Direction:** A single partition key is capped at about 1,000 WCU regardless of table capacity; a hot tenant needs write sharding (suffix `0..N-1` and scatter-gather reads) or a different key (§18; `DB28`).

2. Writes to our DynamoDB orders table are throttled even though the table itself has plenty of spare capacity. The only change was adding a GSI on `status`. Why would an index throttle the base table?

> **Direction:** A throttled GSI back-pressures base-table writes, and a GSI keyed on a low-cardinality attribute is a hot partition by construction; redesign the index key (e.g. shard the status value) (§18; `DB28`).

3. We launched on a new on-demand DynamoDB table expecting on-demand to "just scale". At launch traffic went from zero to 30,000 writes per second and we were throttled for 20 minutes. What should we have done?

> **Direction:** On-demand scales instantly only to about twice the previous peak; pre-warm by switching to high provisioned capacity (or warm throughput) before the event (§18; `DB28`).

4. A Cassandra table used to track pending jobs (insert, read, delete) now returns `TombstoneOverwhelmingException` on reads. How did we get here and why is lowering `gc_grace_seconds` a dangerous fix?

> **Direction:** Queue-like delete patterns fill partitions with tombstones that reads must scan; lowering `gc_grace_seconds` without repairs completing in that window resurrects deleted data — redesign (time-bucketed partitions, TWCS, or a real queue) (§18; `DB28`, `DB29`).

5. Our Cassandra tombstone warnings are huge on a table where we never delete anything. The Java service writes every column on every insert, sometimes with nulls. Why are there tombstones?

> **Direction:** Binding `null` in a prepared statement writes a tombstone; leave unset values unset in the driver instead of binding null (§18; `DB28`).

6. Data that was deleted from Cassandra two weeks ago reappeared after a node that had been down for 12 days rejoined. Explain precisely.

> **Direction:** The deleted data's tombstones were compacted away after `gc_grace_seconds` (10 days) on other nodes; the returning node still had the old values and no tombstone to shadow them — repair must run within `gc_grace_seconds`, and nodes down longer must be rebuilt, not rejoined (§18; `DB28`).

7. A MongoDB collection stores one document per user with an `events` array that gets `$push` on every action. Write latency has crept up and some writes now fail. What is the modelling problem and fix?

> **Direction:** Unbounded arrays make each update rewrite a growing document and eventually hit the 16 MB limit; use the bucket pattern or a separate events collection (§17; `DB27`).

8. A MongoDB sharded cluster with the shard key `_id` (ObjectId) has one shard at 100% write load and the others idle. Why, and what are the options now that the data exists?

> **Direction:** ObjectIds increase monotonically, so all inserts go to the last chunk on one shard; use a hashed or compound shard key — refine or reshard online (5.0+), accepting the cost (§17; `DB27`, `DB25`).

9. MongoDB latency spikes across every operation, and `serverStatus` shows application threads doing eviction. The working set is supposed to fit in RAM. What is happening?

> **Direction:** WiredTiger dirty-cache or total-cache thresholds were crossed (large document rewrites, long transactions pinning versions), drafting application threads into eviction; reduce rewritten bytes, shorten transactions, and size cache for the working set (§17; `DB27`).

10. After a MongoDB primary stepped down during a network blip, some orders acknowledged to users disappeared. The driver used `w: 1`. Where did they go?

> **Direction:** Writes acknowledged by the old primary but not replicated were rolled back when it rejoined and saved to `rollback/` files; use `w: "majority"` for important writes and recover from the rollback files (§17; `DB27`).

11. A Redis instance used as a cache went unresponsive for 4 seconds every time someone ran a cleanup script. The script uses `KEYS session:*` then `DEL`. What's wrong and what's the fix?

> **Direction:** `KEYS` is O(N) and blocks the single command thread, and `DEL` on big keys blocks too; use `SCAN` with batches and `UNLINK`, or better, TTLs so no cleanup is needed (§19; `DB30`).

12. Our Redis node with 12 GB of data on a 16 GB host was killed by the OOM killer during a nightly `BGSAVE`. Memory use is otherwise stable. Why?

> **Direction:** The fork's copy-on-write duplicates every page modified during the save, up to double memory on a write-heavy instance (worse with transparent huge pages); leave headroom, disable THP, or persist from a replica (§19; `DB30`).

13. Redis `maxmemory` is reached and all writes fail with `OOM command not allowed`, though the policy is set to `volatile-lru`. Why is nothing being evicted?

> **Direction:** `volatile-*` policies only evict keys with a TTL; if the application sets none, it behaves like `noeviction` — use `allkeys-lru`/`allkeys-lfu` for a cache, and alert on memory for a store (§19; `DB30`).

14. A Redis replica keeps disconnecting and re-syncing every few minutes and the primary's CPU and memory spike each time. Nothing is wrong with the network. What loop is it in?

> **Direction:** During a full sync the replica's output buffer exceeds `client-output-buffer-limit replica`, it is disconnected and starts another full sync; raise the buffer limit and `repl-backlog-size` so partial resync works (§19; `DB30`).

15. Elasticsearch indexing stopped entirely and the master node's CPU is pegged. The team recently started indexing customers' custom attributes as `attributes.<name>`. What happened?

> **Direction:** Dynamic mapping explosion — every new key adds a field to the cluster state the master must publish; use `flattened`, a key/value nested shape, or `dynamic: false`, and keep `total_fields.limit` as a guard (§20; `DB31`).

16. All writes to our Elasticsearch cluster fail with `cluster_block_exception ... read_only_allow_delete` after a log spike. Disk on one node hit 96%. What are the watermarks and how do you recover?

> **Direction:** The 95% flood-stage watermark made every index with a shard on that node read-only; free or add disk (delete old indices, add nodes), and on older versions clear the block manually (§20; `DB31`).

17. An export feature pages through Elasticsearch with `from`/`size` and fails at page 201 with a result-window error; raising `max_result_window` made the cluster unstable. What's the right approach?

> **Direction:** Deep `from` makes every shard collect `from + size` hits; use `search_after` with a point-in-time for exports (§20; `DB31`).

18. A ClickHouse cluster started rejecting inserts with "Too many parts". The ingestion service inserts every event individually as it arrives, about 2,000 per second. Fix it.

> **Direction:** Each insert creates a part and merges cannot keep up; batch inserts (thousands of rows or once per second), use `async_insert`, or a Kafka/buffer engine (§20; `DB32`).

19. A MongoDB change-stream consumer was down for a weekend and on restart failed with `ChangeStreamHistoryLost`. What happened and how do you design to survive this?

> **Direction:** The resume token fell off the oplog, which covers only a window of time; size the oplog for the longest tolerated outage in hours, alert on consumer lag, and have a resnapshot path (§17; `DB27`).

20. A viral product page puts 200,000 reads per second on a single Redis key in a 12-shard cluster. One shard is saturated, the rest idle. What are your options?

> **Direction:** A key lives in one slot on one shard; add a short-lived in-process cache or client-side caching, replicate the key under several names and read a random one, or read from replicas (§19; `DB30`).

### Level 9 — Disaster, recovery and data integrity

Rarer, higher-stakes questions: data is wrong or gone, and every choice has a cost.

1. At 14:32 an engineer ran `UPDATE accounts SET plan = 'free'` without a `WHERE` clause on production. It committed. Writes have continued for 20 minutes since. How do you recover without losing those 20 minutes of other writes?

> **Direction:** PITR-restore a side instance to just before the bad transaction (target the xid/LSN found in logs or `pg_waldump`), then copy the affected column back into production with a targeted `UPDATE ... FROM` — never roll the whole database back (§15; `DB36`).

2. Your backups have run successfully every night for three years. The CTO asks how long a full restore of the 4 TB production database would take. Nobody knows. What do you do, and what surprises do you expect?

> **Direction:** Run automated restore tests; expect restore time to be dominated by WAL replay and, for cloud snapshots, lazy block hydration from object storage, making the restored instance slow until prewarmed — measure it into the RTO (§12, §15; `DB36`).

3. We restored an RDS snapshot to investigate an incident and pointed a copy of the app at it. It was unusably slow for hours, though it is the same instance class as production. Why?

> **Direction:** Snapshot restores load blocks lazily from S3 on first access; hydrate by reading every block (`pg_prewarm`, full table scans, `fio`/`dd`) before putting load on it (§12, §15; `DB36`).

4. A developer dropped a table in production. We have a streaming replica and daily logical dumps. The replica already has the drop. What is the fastest safe recovery, and what would have made it trivial?

> **Direction:** Replicas replicate mistakes; restore the table from PITR on a side instance (or the last dump plus nothing newer), copy it back; a delayed replica (`recovery_min_apply_delay`) would have allowed pausing replay and copying the table out in minutes (§15; `DB36`, `DB23`).

5. The logs show `ERROR: invalid page in block 48213 of relation base/16384/24576` on a busy table. What is your first action, and what is your sequence after that?

> **Direction:** First take a file-level copy of the data directory before touching anything; then identify the relation, check `amcheck`/checksums, fail over to a healthy replica or restore the affected table from backup, and use `zero_damaged_pages` only as a last resort to salvage the rest (§15; `DB36`, `DB12`).

6. After upgrading the OS on our database servers from Debian 9 to 10 (in-place, same data directory), a unique index on `users.username` now contains duplicates and some lookups miss existing rows. What happened?

> **Direction:** glibc 2.28 changed collation order, so existing text indexes are sorted by the old rules; run `amcheck` to find affected indexes and `REINDEX` them, de-duplicating conflicting rows first (§8, §15; `DB13`).

7. Payments occasionally double-charge a customer when the mobile client retries after a timeout. The payment service checks for an existing charge before creating one. Why does the check not work, and what is the correct fix?

> **Direction:** Check-then-insert races between concurrent requests; enforce the idempotency key with a unique constraint and return the stored result on conflict (§14; `DB38`, `DB05`).

8. An ETL job reads new rows with `WHERE id > :last_max_id` every minute. Finance found that about 1 in 50,000 orders never reached the warehouse. The IDs come from a sequence. How can rows be skipped?

> **Direction:** IDs are allocated at insert, not at commit, so a slow transaction commits a lower ID after the watermark has moved past it; re-read an overlap window idempotently, use a snapshot-based watermark, or switch to log-based CDC (§14; `DB23`, `M15`).

9. The same class of bug but with `updated_at > :last_sync` and `updated_at DEFAULT now()`. Rows are updated inside long-running batch transactions. Why are some updates missed?

> **Direction:** `now()` is the transaction start time, so a long transaction stamps rows in the past and commits them after the poller has advanced; use `clock_timestamp()` or, better, CDC/an outbox with an overlap window (§14; `DB04`, `M15`).

10. Our `orders.id` is an `integer` and a monitoring alert says the sequence is at 2,120,000,000. We insert 15 million orders a day. The proper migration to `bigint` will take weeks. What do you do tonight?

> **Direction:** Buy time by restarting the sequence at −2,147,483,648 counting up (if nothing assumes positive or ordered IDs) — about another 2 billion — while running the add-column/backfill/swap migration to `bigint`; audit FK columns and clients (e.g. 32-bit ints in mobile apps) too (§9; `DB22`).

11. A MySQL 5.7 table had its most recent rows deleted by a cleanup job; after a restart, new orders reused IDs that exist in our invoicing system and caches. How is that possible?

> **Direction:** Before 8.0 InnoDB recomputed `AUTO_INCREMENT` as `MAX(id)+1` on restart, so deleted top IDs were reused; never delete the tail, upgrade to 8.0, or use IDs not derived from the table (§5, §9; `DB01`).

12. Customer names with emoji are stored as `????` in our MySQL database and some inserts fail. The column is `utf8`. What is going on and how do you migrate safely?

> **Direction:** MySQL's `utf8` is 3-byte `utf8mb3`; convert to `utf8mb4` (connection charset too), watching index key-length limits and doing the table conversion online (gh-ost) (§5; `DB04`).

13. A daily revenue report double-counts about an hour of orders on one Sunday in October and misses an hour in March. The column is `timestamp without time zone` storing local time. Explain and fix.

> **Direction:** DST transitions make local wall-clock hours repeat or vanish; store `timestamptz` (UTC instants) and group by `date_trunc` in the business time zone, computing range bounds in UTC so indexes still work (§14; `DB04`).

14. A replica promoted during failover turned out to have been silently diverged for weeks: some rows differ from the old primary's. How could a physical replica diverge, and how would you detect it earlier?

> **Direction:** Physical replicas replay bytes, so divergence means storage/hardware corruption or collation differences between hosts rather than logic; enable data checksums, run periodic `amcheck` and table checksums against the primary (§8, §15; `DB12`, `DB23`).

15. We use Redis as the primary store for shopping carts with AOF `everysec` and one replica. After a failover, some users' carts lost items added in the last seconds. The product says "never lose a cart item". What honest options do you offer?

> **Direction:** Async replication and `everysec` lose recent writes on failover; `WAIT` narrows the window but does not guarantee, so either accept and state the bounded loss, or keep carts in a durable database with Redis as a cache (§19; `DB30`).

16. A ledger's account balances no longer match the sum of the ledger entries for 0.01% of accounts. How do you find the cause, and how do you design so this is detectable and fixable?

> **Direction:** Treat the stored balance as a cache of an append-only double-entry ledger; reconcile continuously, and look for lost updates (read-modify-write, ORMs writing full rows) or non-atomic writes of entry and balance (§14; `DB38`, `DB10`).

17. Two support agents edit the same customer record in an admin UI; one agent's change to the address silently undoes another's change to the phone number, although they edited different fields. Why?

> **Direction:** The ORM writes every column from its stale in-memory copy, turning disjoint edits into a lost update; update only dirty fields and add a version column for optimistic concurrency (§14, §16; `DB10`, `DB40`).

18. The on-call accidentally deleted all files in `pg_wal` to free disk space during an incident. Postgres will not start. What are the options, from best to worst?

> **Direction:** Best is failing over to a replica or restoring from base backup plus archived WAL; `pg_resetwal` makes the server start but may leave the cluster inconsistent — only to dump data out, never to keep running on (§12, §15; `DB12`, `DB36`).

19. Our Cassandra cluster uses client-supplied timestamps. After an NTP misconfiguration on one app host, updates from other hosts appear to be ignored for certain rows. Explain.

> **Direction:** Last-write-wins by timestamp: writes stamped in the future by the skewed host win over later real writes until real time catches up; fix clock sync, prefer server-side timestamps, and repair affected rows (§18; `DB28`).

20. A disaster-recovery test in another region succeeded technically, but it took 11 hours instead of the promised one. The backups were fine. Where does time usually go in a real DR, and how do you shrink it?

> **Direction:** WAL replay volume, cold-cache and lazy-snapshot hydration, DNS/config/credential changes, application dependencies nobody listed, and human decision time; keep a warm cross-region replica, rehearse regularly, and automate the runbook (§11, §12, §15; `DB36`).

### Level 10 — Deep internals and cascading failures

The rarest questions: multi-system chains, engine internals, and trade-offs that only appear at scale.

1. Throughput across the entire cluster, including replicas, dropped by 80% after a release. `pg_stat_activity` shows many sessions waiting on `SubtransSLRU` / `SubtransBuffer`. The release added a loop that wraps each row write in a nested `transaction.atomic()`. Explain the mechanism.

> **Direction:** Each savepoint that writes is a subtransaction; beyond 64 per backend the subxid cache overflows and every snapshot visibility check must consult the `pg_subtrans` SLRU, which combined with a long transaction thrashes cluster-wide — remove the per-row savepoints (§3; `DB08`, `DB40`).

2. The xmin horizon is stuck, no long transaction exists on the primary, no replica has feedback on, and there are no replication slots. Autovacuum cannot clean anything. What else can hold the horizon?

> **Direction:** A forgotten prepared transaction from two-phase commit (`pg_prepared_xacts`) — it survives restarts; roll it back with `ROLLBACK PREPARED` after checking with its coordinator (§3; `DB08`, `M13`).

3. A write-once table storing documents as large text values has had steadily slowing inserts for months, now 200 ms each, with no locks, bloat or I/O saturation. It holds 3.5 billion out-of-line values. What limit are you approaching?

> **Direction:** TOAST OID exhaustion — each out-of-line value needs a unique 32-bit OID in its TOAST table, and as the space fills inserts loop searching for a free one; partition the table so each partition gets its own TOAST relation (§9; `DB11`).

4. Wraparound: the database has stopped accepting writes ("database is not accepting commands to avoid wraparound data loss"). It is a 6 TB database. Walk through recovery and how to make it as fast as possible.

> **Direction:** Remove anything holding the horizon (prepared transactions, slots, old sessions), then `VACUUM (FREEZE)` the tables with the oldest `relfrozenxid` first, unthrottled and skipping index cleanup where the version allows (failsafe), rather than vacuuming the whole database blindly (§3; `DB08`).

5. A payments service does thousands of inserts per second into `payments` that reference a small number of `merchants`. Latency spikes coincide with `LWLock:MultiXactOffsetSLRU` waits and multixact freezing on `merchants`. What are the unconventional mitigations?

> **Direction:** FK checks take `FOR KEY SHARE` on hot parent rows, creating multixacts; mitigate by enlarging the multixact SLRU caches (PG17+), aggressive multixact freeze on the parent, and, as a trade-off, dropping the FK on that hot path and enforcing the reference differently (§3, §4; `DB05`, `DB08`).

6. Postgres at 20,000 inserts per second into one table with a `bigserial` key shows `LWLock:BufferContent` waits on the index. Moving to UUIDv4 fixed the waits but tripled WAL. What is the balanced design?

> **Direction:** Sequential keys contend on the right-most leaf page, random keys scatter writes and full-page images; use hash partitioning on the key, or a key with a small shard prefix plus time ordering, to spread inserts over a few hot pages instead of one or all (§8, §12; `DB13`, `DB24`).

7. A chain: a CDC connector stalled → its logical slot retained WAL → the primary's disk filled → Postgres PANICked → failover promoted a replica → the connector could not resume and the new primary lacked the slot → downstream search index missed hours of updates. Where would you break this chain, at each link?

> **Direction:** Alert on slot lag, cap retention with `max_slot_wal_keep_size`, keep a disk ballast, use failover-synchronised logical slots (PG17+) or a resnapshot runbook, and make the indexer rebuildable and version-aware (§11, §12, §20; `DB23`, `DB31`).

8. After failing over a Postgres cluster whose physical standby was on newer hardware and a newer OS image, queries on text columns returned inconsistent results and a unique index accepted duplicates. What went wrong in the platform process?

> **Direction:** Different glibc/ICU versions between primary and standby mean the standby's B-trees are ordered under different collation rules; pin OS/collation versions in a replication set and treat an OS upgrade as a logical-replication migration or a reindex (§8, §11; `DB13`, `DB23`).

9. Our 18-hour online migration used gh-ost on a 3 TB MySQL table. At cut-over the application saw errors for 90 seconds and some writes were lost. What are the cut-over mechanics and where can writes be lost?

> **Direction:** gh-ost briefly locks the original table, drains remaining binlog events, and atomically renames; writes lost usually come from apps holding connections past the lock with retries failing, or from long transactions at cut-over — use postponed cut-over, a low `lock_wait_timeout`, and verify row counts/checksums (§4, §5; `DB22`).

10. A Kubernetes rollout of 200 pods coincided with a primary failover. The result was a 40-minute outage although each alone recovers in a minute. Describe the feedback loop and the design changes.

> **Direction:** Connection storms from new pods plus reconnects after failover saturate `max_connections` and CPU on authentication, health checks fail, pods restart and reconnect again; break it with a pooler, jittered backoff, liveness that does not depend on the database, and rate-limited rollouts (§7; `DB21`).

11. A Postgres primary with 8,000 partitions and 1,500 connections shows high CPU in `LWLock:LockManager` and planning times of hundreds of milliseconds on simple queries after partition pruning was lost by a code change. How do these interact?

> **Direction:** Queries that do not prune lock every partition and its indexes, overflowing per-backend fast-path lock slots into the shared lock manager, and planning scales with partitions considered; restore plan-time pruning, reduce partition count and connections (§6, §7, §13; `DB24`).

12. You need to move a 10 TB Postgres database to a new major version and a new cloud provider with under five minutes of write downtime. Outline the plan and the traps.

> **Direction:** Logical replication into the new cluster, verify, then a controlled cut-over; traps are sequences not replicated, DDL freeze during the sync, tables without replica identity, large objects, initial-copy xmin hold on the source, and post-cut-over `ANALYZE` (§10, §11, §6; `DB22`, `DB23`).

13. A multi-tenant Postgres uses one schema per tenant, 30,000 schemas. Migrations take 14 hours, `pg_dump` is unusable, and backend memory per connection is huge. What happened architecturally and how do you escape?

> **Direction:** Hundreds of thousands of relations bloat catalogues and per-backend relcache/catcache memory, and every migration and dump walks them; move to shared tables with a `tenant_id` (plus RLS) or shard tenants across clusters, migrating tenants gradually (§7, §10; `DB25`, `DB37`).

14. A DynamoDB global table serves two regions. Users occasionally see their updates disappear shortly after making them. There is no bug in the app's write logic. What is the conflict model and how would you design around it?

> **Direction:** Global tables replicate asynchronously with last-writer-wins by timestamp, so concurrent updates to the same item in two regions silently overwrite each other; home each user's writes to one region (conditional writes only check the local copy) or pay for multi-region strong consistency (§14, §18; `DB28`, `M12`).

15. Elasticsearch search results show an older version of a product than the database after a burst of rapid updates, even though every update was indexed. The indexer consumes a partitioned queue with parallel workers. Why?

> **Direction:** Updates for the same document were processed out of order by different workers, so an older version was indexed last; key the queue by document ID and use external versioning so stale writes are rejected (§14, §20; `DB31`, `M15`).

16. A Postgres replica used for heavy analytics with `hot_standby_feedback = on` caused an anti-wraparound emergency on the primary. Nobody touched the primary. Trace the causal chain.

> **Direction:** A multi-hour replica query's xmin was fed back to the primary, holding the freeze horizon so vacuum could neither clean nor freeze, and xid age crossed the thresholds; bound replica query duration, or isolate analytics on a non-feedback replica or warehouse (§3, §11; `DB08`, `DB23`).

17. Your MySQL primary's history list length is 40 million, read latency on hot rows is rising daily, and `ibdata1` has grown by 300 GB. There is no obvious long query. What do you hunt for, and what is the long-term clean-up?

> **Direction:** An old open transaction or consistent snapshot (a stuck backup, an idle session with a started transaction) stopping purge; find it in `information_schema.innodb_trx`, kill it, and — pre-8.0 — reclaim the system tablespace only by rebuilding, so move to separate undo tablespaces (§5; `DB08`).

18. Your Postgres cluster crashed and crash recovery has been running for 55 minutes. Management asks why a "restart" takes an hour and how to make it seconds next time without losing durability.

> **Direction:** Recovery replays WAL since the last checkpoint, so large `max_wal_size`/`checkpoint_timeout` trade recovery time for less WAL; the real fix for availability is failover to a hot standby rather than waiting for recovery, with checkpoint settings tuned to an agreed recovery time (§11, §12; `DB12`).

19. A Cassandra cluster using STCS ran out of disk during compaction at 70% usage and nodes started failing. Why at 70%, and what would you change in the operating model?

> **Direction:** Size-tiered compaction can temporarily need as much free space as the SSTables being merged (up to ~50% of the data); keep headroom accordingly, or move to LCS/TWCS matched to the workload, and add nodes before, not during, the crisis (§18; `DB28`, `DB29`).

20. You join a company where the core Postgres primary is at 96% CPU on the largest available instance, with 4,000 queries per second and no obvious single bad query. Nobody wants to shard. What is your 90-day plan?

> **Direction:** Profile by total time and wait events, remove N+1 and chatty patterns, cut index and write amplification (unused indexes, HOT, WAL), move analytics off the primary, add a pooler and read replicas with read-your-writes, cache hot reads, then split by workload or tenant before true sharding (§1, §2, §7, §8, §11; `DB20`, `DB25`, `DB39`).
