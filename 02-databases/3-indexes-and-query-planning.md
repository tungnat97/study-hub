[← back to the field index](README.md)

# Databases · Part 3 — Indexes, the Planner & Query Tuning

Nodes `DB13`–`DB21`. If you can design a composite index and read an `EXPLAIN` out loud, you will
pass most database interviews.

---

## DB13 · B+tree indexes

`Intermediate` · Requires: `DB04`, `DB11` · Unlocks: `DB14`, `DB15`, `DB16`, `DB17`, `DB18`, `DB29`

### Preface

An index is a sorted copy of some columns, with pointers back to the rows. Sorted data can be
searched by halving the search space repeatedly, so finding one row among a hundred million takes a
handful of page reads instead of scanning everything.

Essentially every general-purpose index is a **B+tree**. Knowing its shape explains exactly which
queries an index can help and which it cannot.

### Details

#### 1. The structure

**Theory.** A B+tree is a balanced tree of pages. Internal pages hold keys that direct the search;
**leaf pages hold every key in sorted order, with pointers to rows, and are linked to their
neighbours**. Because it is balanced, every lookup costs the same. Because leaves are sorted and
linked, range scans are cheap: find the start, then walk sideways.

**Example.** A table of 100 million rows. With roughly 8KB pages and a small key, each page holds
hundreds of entries, so the tree is about 4 levels deep. A point lookup reads about 4 pages — and
the top levels are almost always cached, so in practice it is one or two actual disk reads. That is
the whole reason indexes work.

**Advanced.** This is why index depth grows so slowly: the branching factor is in the hundreds, so
depth grows with the logarithm base ~200 of the row count. Going from 1 million to 1 billion rows
adds about one level. It also explains why a **wide key hurts**: fewer entries fit per page, the
branching factor drops, the tree gets deeper, and every lookup costs more. Narrow keys are fast
keys.

#### 2. What a B+tree can and cannot serve

**Theory.** It can serve: equality (`=`), ranges (`<`, `>`, `BETWEEN`), prefix matching
(`LIKE 'abc%'`), sorting in the index's order, `MIN`/`MAX` (read the first or last leaf), and
`IN` lists (several lookups). It cannot serve: suffix or substring matching (`LIKE '%abc'`),
conditions on a *function* of the column, or a column that is not the leftmost part of the key
(`DB15`).

**Example.** `WHERE email LIKE 'john%'` uses an index, because the sorted order puts all such values
together. `WHERE email LIKE '%@gmail.com'` cannot, because matching values are scattered throughout
the sorted order — a full scan is required. The fixes: a trigram index (`DB14`), or storing a
reversed copy of the column and indexing that so the suffix becomes a prefix.

**Advanced.** Sort order matters for multi-column `ORDER BY`. An index on `(a ASC, b ASC)` can serve
`ORDER BY a, b` and also `ORDER BY a DESC, b DESC` (by scanning backwards), but **not**
`ORDER BY a ASC, b DESC` — for that you need an index declared as `(a ASC, b DESC)`. This catches
people out on paginated queries with mixed sort directions.

#### 3. What an index costs

**Theory.** Indexes make reads faster and writes slower. Every insert, delete, and update of an
indexed column must also update the index. Each index also consumes disk and, more importantly,
memory in the page cache — competing with your data.

**Example.** A table with 12 indexes: each insert performs 13 writes. Under heavy write load, that
is usually the bottleneck. Worse, unused indexes cost exactly as much as useful ones. Find them:

```sql
SELECT relname, indexrelname, idx_scan
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;
```

Anything with zero scans since the last statistics reset is a candidate for removal (check it is not
enforcing a constraint, and that it is not needed by a rare but critical query).

**Advanced.** In Postgres, an update of a **non-indexed** column can use a HOT update and skip index
maintenance entirely (`DB08`). So adding an index on a frequently-updated column has a cost beyond
the index itself: it disables HOT updates for that table's updates to that column, increasing bloat
and write amplification. That is a genuinely advanced point and a good reason to be selective about
indexing hot columns.

#### 4. Index bloat and rebuilding

**Theory.** Indexes fragment over time as entries are deleted and pages become half-empty. A bloated
index is larger, so it needs more reads and more cache.

**Example.** `REINDEX INDEX CONCURRENTLY idx_name` rebuilds without an exclusive lock (Postgres
12+). Before that, the common trick was `CREATE INDEX CONCURRENTLY` with a new name, then drop the
old one and rename. Check bloat with the `pgstattuple` extension or one of the standard bloat
estimate queries.

**Advanced.** Regular `REINDEX` is rarely needed in modern Postgres for ordinary workloads;
persistent index bloat usually indicates a deeper problem — long transactions preventing cleanup
(`DB08`) or a workload that deletes large ranges. Fix the cause rather than scheduling a rebuild.

### Interview questions

- "How many index pages are read for a point lookup on a 100-million-row table?"
- "Why does `LIKE '%abc'` not use a B-tree, and what would?"
- "How do you find indexes you can safely drop?"
- "Why can an index on `(a ASC, b ASC)` not serve `ORDER BY a ASC, b DESC`?"

---

## DB14 · Other index types

`Advanced` · Requires: `DB13` · Unlocks: `DB31`, `DB34`

### Preface

B+trees handle ordered, scalar values. Other data shapes need other structures: documents and
arrays, geometry and ranges, huge append-only tables, full text, and vectors.

Knowing which index type fits which shape is a practical, high-signal piece of knowledge.

### Details

#### 1. GIN — for values containing many items

**Theory.** A **G**eneralized **IN**verted index maps each contained element to the rows that hold
it. Built for columns where one value has many searchable parts: arrays, `jsonb`, full-text
documents.

**Example.** `CREATE INDEX ON events USING gin (payload jsonb_path_ops)` makes
`WHERE payload @> '{"type":"signup"}'` fast. `jsonb_path_ops` is smaller and faster than the default
for containment queries, at the cost of supporting fewer operators — a good trade when containment
is all you need.

**Advanced.** GIN writes are expensive because one row update may touch many index entries. Postgres
softens this with a **pending list** (`fastupdate`), which buffers insertions and merges them later
— so inserts are fast but a query may have to scan the unmerged pending list, producing occasional
unexpectedly slow reads. On write-heavy tables, either tune `gin_pending_list_limit` or disable
`fastupdate` to trade insert speed for predictable reads.

#### 2. GiST and SP-GiST — for geometry, ranges and nearest-neighbour

**Theory.** A **G**eneralized **S**earch **T**ree is a framework for indexing data where "contains",
"overlaps" or "is near" matter rather than "less than".

**Example.** Geographic search with PostGIS ("all shops within 5km") uses GiST. Range types use it
for overlap queries, which is what powers the `EXCLUDE` constraint that prevents double bookings
(`DB05`). GiST also supports ordering by distance, so "the 10 nearest shops" is an index scan rather
than a scan-and-sort.

**Advanced.** GiST is lossy: it returns a superset of candidates which must then be rechecked
against the actual rows. That recheck cost is why a GiST index can look less effective than expected
in `EXPLAIN` output — you will see "Rows Removed by Index Recheck". SP-GiST suits data that
partitions unevenly, such as IP address ranges or points with heavy clustering.

#### 3. BRIN — tiny indexes for huge ordered tables

**Theory.** A **B**lock **R**ange **IN**dex stores only the minimum and maximum value per block
range (128 pages by default). To answer a query it skips ranges that cannot contain matches. It is
extraordinarily small and only works when the physical row order correlates with the column's
values.

**Example.** A 2TB append-only events table with a `created_at` column. A B-tree on it might be
40GB; a BRIN index is a few megabytes. `WHERE created_at BETWEEN ... AND ...` skips almost all
blocks. Because rows were inserted in time order, physical order matches value order, and the
correlation holds.

**Advanced.** BRIN degrades to useless if correlation breaks — after heavy updates and reuse of free
space, or if rows arrive out of order. Check `correlation` in `pg_stats` for the column: near 1 (or
-1) means BRIN will work, near 0 means it will not. Combining BRIN with table partitioning by time
is the standard pattern for very large time-series tables in Postgres (`DB24`).

#### 4. Hash, bitmap scans, and vector indexes

**Theory.** **Hash** indexes support only equality and are rarely worth it — a B-tree does equality
nearly as fast and does much more. **Bitmap scans** are not an index type but a *scan strategy*:
Postgres builds an in-memory bitmap of matching rows from one or more indexes, combines them with
AND/OR, and then reads the heap in physical order. **Vector** indexes (HNSW, IVFFlat via `pgvector`)
support approximate nearest-neighbour search over embeddings.

**Example.** A bitmap scan is how Postgres uses two separate single-column indexes for
`WHERE a = 1 AND b = 2` — it scans both and intersects the bitmaps. It is decent, and usually still
slower than one composite index on `(a, b)`, because that requires a single traversal and returns
exactly the matching rows.

**Advanced.** Seeing "Bitmap Heap Scan" plus "Recheck Cond" and a high "Rows Removed by Filter" in a
plan is a strong hint that a better composite index exists. Also note bitmap scans can become "lossy"
when the bitmap exceeds `work_mem` — it degrades to tracking whole pages rather than individual
rows, forcing a recheck of every row on those pages, which is a sudden and confusing performance
cliff.

### Interview questions

- "1TB append-only events table, queries by time range. Which index?"
- "What is a GIN index for, and what does it cost on writes?"
- "You see 'Bitmap Heap Scan' in a plan. What does that suggest?"
- "How would you index a `jsonb` column for containment queries?"

---

## DB15 · Composite indexes, selectivity and cardinality

`Advanced` · Requires: `DB13` · Unlocks: `DB16`, `DB19`, `DB20`

### Preface

Most real queries filter on several columns, so most real indexes are multi-column. Getting the
**column order** right is the single highest-leverage indexing skill.

The rule to memorise: **equality columns first, then the range column, then the sort column.**
Remember it as E-R-S.

### Details

#### 1. The leftmost prefix rule

**Theory.** A composite index is sorted by the first column, then the second within equal values of
the first, and so on. So it can serve queries that use a **prefix** of its columns: an index on
`(a, b, c)` serves `a`, `(a, b)` and `(a, b, c)` — but not `b` alone, and not `(b, c)`.

**Example.** With an index on `(tenant_id, status, created_at)`:
- `WHERE tenant_id = 1` — uses it.
- `WHERE tenant_id = 1 AND status = 'open'` — uses it.
- `WHERE status = 'open'` — cannot use it (no leading column).
- `WHERE tenant_id = 1 AND created_at > $1` — uses the `tenant_id` part, then filters; it cannot
  jump straight to the `created_at` range because `status` is unconstrained in between.

**Advanced.** Postgres can sometimes do a **skip scan**-like trick via a bitmap index scan on a
non-leading column, and MySQL 8 has index skip scan when the leading column has very few distinct
values — but neither should be relied on. Design the order for your actual queries. A practical
consequence: you rarely need both `(a, b)` and `(a)` — the first serves both, so the second is
redundant and should be dropped.

#### 2. Column order: equality, range, sort

**Theory.** Put columns tested with `=` first (they narrow the search to one contiguous block), then
the column used in a range, then columns used only for ordering. After a range column, no later
column can be used for further seeking — only for filtering — which is why the range goes late.

**Example.** For
`WHERE tenant_id = $1 AND status = $2 AND created_at > $3 ORDER BY created_at DESC LIMIT 20`,
the index is:

```sql
CREATE INDEX ON orders (tenant_id, status, created_at DESC);
```

Two equality columns pin the search to one region; `created_at` then provides both the range and the
ordering, so the database can read 20 entries and stop — no sort step at all.

**Advanced.** This "no sort step" property is what makes paginated list endpoints fast, and it is
worth stating explicitly in an interview: the index supplies the order, so `LIMIT` genuinely limits
the work. Without it, the database must find every matching row, sort them all, and discard all but
20 — which is why an apparently identical query is a thousand times slower.

#### 3. Selectivity and cardinality

**Theory.** **Cardinality** is the number of distinct values in a column. **Selectivity** is the
fraction of rows a condition keeps. An index is only useful when the condition is selective enough
that reading index entries plus fetching those rows beats just scanning the table.

**Example.** A `status` column with values `active` and `deleted`, 95% active. An index on `status`
is useless for `WHERE status = 'active'` (a scan is cheaper than 95% random row fetches) but
valuable for `WHERE status = 'deleted'`. The right answer for such a column is often a **partial
index** on the rare value (`DB17`).

**Advanced.** The threshold is usually somewhere around 5-20% selectivity, and it depends on
**correlation**: if matching rows are physically clustered together, the index stays useful at much
lower selectivity because the row fetches are sequential rather than random. That is why
`random_page_cost` matters on SSDs (`DB18`) — the default assumes random reads are four times more
expensive than sequential, which was true for spinning disks and is far too pessimistic for SSDs,
causing the planner to avoid indexes it should use.

#### 4. Too many indexes

**Theory.** Each index slows writes, uses memory, and gives the planner more options to get wrong.
Redundant indexes are pure cost.

**Example.** Audit checklist for a table's indexes: drop any index that is a **prefix** of another
(`(a)` when `(a, b)` exists); drop indexes with zero scans; consolidate near-duplicates (`(a, b)`
and `(a, c)` might both be served by `(a, b, c)` depending on the queries — verify, do not assume);
and make sure every foreign key has an index if you delete or update parents (otherwise the
referential check scans the child table).

**Advanced.** The unindexed-foreign-key problem is a classic hidden cause of slow deletes and of
lock contention: deleting a parent row requires checking every child table for references, and
without an index that is a sequential scan per table, holding locks the whole time. Most ORMs index
foreign keys by default, but hand-written migrations frequently miss them — worth checking as a
routine audit.

### Interview questions

- "Given `WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY created_at DESC`, design
  the index."
- "Why is the planner ignoring your index?"
- "You have indexes on `(a)` and `(a, b)`. Which would you drop?"
- "Why does a low-cardinality column make a bad index, and what would you use instead?"

---

## DB16 · Covering indexes and index-only scans

`Advanced` · Requires: `DB15` · Unlocks: `DB20`

### Preface

Normally an index lookup has two steps: find the entry in the index, then fetch the row from the
table for the other columns. If the index already contains every column the query needs, the second
step can be skipped entirely — an **index-only scan**.

This can be the difference between 200ms and 2ms, because the random row fetches usually dominate.

### Details

#### 1. Covering an index

**Theory.** An index "covers" a query when it contains all the columns the query references — in the
`SELECT` list, the `WHERE` clause and the `ORDER BY`. Postgres supports `INCLUDE` columns, stored in
the leaf pages only: they are available for the query but not used for searching or ordering, so
they do not widen the tree's internal pages.

**Example.**

```sql
-- query: SELECT id, total FROM orders WHERE tenant_id = $1 AND status = $2;
CREATE INDEX ON orders (tenant_id, status) INCLUDE (id, total);
```

The index now answers the query without touching the table. Use `INCLUDE` rather than adding the
columns as key columns: key columns make the index bigger at every level and impose ordering you do
not need.

**Advanced.** InnoDB gets partial covering for free, because every secondary index already contains
the primary key — so `SELECT id FROM t WHERE indexed_col = ?` is always index-only in MySQL. MySQL
has no `INCLUDE`, so covering extra columns means adding them as key columns, with the associated
size cost.

#### 2. The Postgres visibility catch

**Theory.** Postgres cannot decide from the index alone whether a row is visible to your transaction
— that information lives in the row itself (`DB08`). So an index-only scan must consult the
**visibility map**, a compact structure marking pages where all rows are visible to everyone. If the
page is not marked, Postgres must fetch the row anyway, and the "index-only" scan is not index-only.

**Example.** In `EXPLAIN (ANALYZE, BUFFERS)` you see `Index Only Scan` with `Heap Fetches: 48213` —
a large number means the visibility map is stale and you are paying for heap access regardless. The
fix is `VACUUM`, which updates the visibility map. On an append-only table where autovacuum rarely
triggers (few dead rows, so the threshold is never reached), this is a common and confusing
situation; tune autovacuum or run a manual vacuum.

**Advanced.** This is a good example of the internals mattering in practice: the same query plan
performs completely differently depending on vacuum state, and the only visible difference is
`Heap Fetches`. Being able to explain that connection — index-only scan → visibility map → vacuum —
is a strong Postgres signal.

#### 3. When not to cover

**Theory.** Covering indexes get wide. A wide index holds fewer entries per page, consumes more
cache, and slows writes. Covering is a targeted optimisation for a specific hot query, not a
default.

**Example.** Adding `INCLUDE (description, metadata)` to cover a query that returns large text
fields roughly duplicates that data inside the index. You have doubled storage and halved cache
efficiency to avoid one heap fetch. Not worth it. Cover narrow, frequently-read columns only.

**Advanced.** A useful way to decide: covering pays when the query returns **many** rows (so you
avoid many random fetches) and the extra columns are **small**. For a query returning one row, the
single heap fetch is negligible and covering is pointless. Quantify it — that framing turns a
judgement call into a calculation.

### Interview questions

- "Your index-only scan still hits the heap. Why?"
- "When is a covering index the wrong choice?"
- "What is `INCLUDE` and why not just add the column to the key?"
- "Why does MySQL get some covering for free?"

---

## DB17 · Partial and expression indexes

`Advanced` · Requires: `DB13` · Unlocks: `DB20`

### Preface

A **partial index** indexes only some rows. An **expression index** indexes the result of a
computation rather than the raw column.

Both exist to make an index small and precisely targeted, which makes it fast and cheap. They are
among the most underused features in Postgres and are excellent things to bring up in an interview.

### Details

#### 1. Partial indexes

**Theory.** `CREATE INDEX ... WHERE condition` indexes only rows matching the condition. The index
is smaller, faster to scan, and cheaper to maintain (rows outside the condition never touch it).

**Example.** A jobs table with 50 million rows where only a few thousand are pending at any moment:

```sql
CREATE INDEX ON jobs (run_at) WHERE status = 'pending';
```

The index holds thousands of entries rather than 50 million. The worker's query
`WHERE status = 'pending' AND run_at <= now()` uses it; as soon as a job is completed, its entry is
removed automatically. Similarly, `WHERE deleted_at IS NULL` for soft-deleted tables keeps the index
proportional to live data.

**Advanced.** For the planner to use a partial index, it must **prove** the query's condition implies
the index's condition. Simple equality works; parameterised or complex conditions may not. If the
index is not used, check that the query's predicate literally matches — `WHERE status = 'pending'`
matches, but `WHERE status = $1` cannot be proven at plan time for a generic plan, so the index may
be skipped. That subtlety catches people out.

#### 2. Partial unique indexes

**Theory.** Uniqueness enforced only over a subset of rows. This resolves several common modelling
tensions cleanly, without triggers.

**Example.** Two canonical uses:

```sql
-- one active subscription per user, any number of cancelled ones
CREATE UNIQUE INDEX ON subscriptions (user_id) WHERE status = 'active';

-- email unique among non-deleted users, reusable after deletion
CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL;
```

Enforced by the database under any concurrency, with no application logic.

**Advanced.** This pattern is the correct answer to a surprising number of "how do you enforce X"
interview questions. It is also worth noting the limitation: it enforces "at most one", not "exactly
one" — nothing forces a user to have an active subscription. Rules of the form "at least one" need a
different mechanism (a deferred constraint trigger, or accepting eventual repair).

#### 3. Expression indexes

**Theory.** Index the result of an expression. The index is only usable when the query contains the
**same expression**, character for character in effect.

**Example.**

```sql
CREATE INDEX ON users (lower(email));
-- used by:   WHERE lower(email) = lower($1)
-- NOT used by: WHERE email = $1
```

Also useful for JSON: `CREATE INDEX ON events ((payload->>'user_id'))` makes
`WHERE payload->>'user_id' = $1` an index lookup instead of a scan.

**Advanced.** The expression must be `IMMUTABLE` — it must always return the same output for the same
input — because the index is computed once at write time. `lower()` is immutable; `now()` is not;
and a timezone conversion such as `created_at AT TIME ZONE 'UTC'` is only immutable for
`timestamptz`, not for `timestamp`. Marking a volatile function as immutable to force an index is a
way to silently corrupt query results — the classic version is indexing a date conversion that
depends on the session timezone.

#### 4. Combining the two

**Theory.** Partial and expression indexes compose, producing very small, very targeted indexes.

**Example.**

```sql
CREATE UNIQUE INDEX ON users (lower(email)) WHERE deleted_at IS NULL;
```

Case-insensitive uniqueness among active users only, in one small index. That single line replaces
a trigger, a check, and a piece of application logic.

**Advanced.** The maintenance consideration is that these indexes are invisible to most
schema-comparison tools and easy to lose in a naive migration or a database rebuild. Keep them in
version-controlled migrations with a comment explaining the rule they enforce — otherwise someone
"cleaning up unused indexes" removes the thing preventing duplicate accounts.

### Interview questions

- "Enforce 'only one active subscription per user' in the database."
- "Your expression index is not being used. Why?"
- "How do you keep an index small on a table where only 0.1% of rows are ever queried?"
- "Why must an indexed expression be immutable?"

---

## DB18 · The query planner

`Expert` · Requires: `DB08`, `DB13` · Unlocks: `DB19`, `DB20`

### Preface

You write what you want; the planner decides how to get it. It considers many possible plans,
estimates the cost of each using statistics about your data, and picks the cheapest.

Nearly all mysterious performance problems come from the planner **estimating wrongly** — usually
expecting a few rows and getting millions, which makes it choose a plan that is catastrophic at the
real size.

### Details

#### 1. Statistics

**Theory.** The planner needs to guess how many rows a condition will produce. It uses statistics
gathered by `ANALYZE`: the number of distinct values (`n_distinct`), a list of the most common
values and their frequencies (MCV), a histogram of the value distribution, the fraction of NULLs,
and the physical/logical correlation of the column.

**Example.** `WHERE status = 'pending'` on a table where `pending` is 0.1% of rows: the MCV list
tells the planner this directly, so it estimates a small number and chooses an index scan. Without
statistics — right after a bulk load, before autoanalyze runs — it guesses, often badly, and can
choose a sequential scan over 50 million rows. **Always `ANALYZE` after a large data load.**

**Advanced.** `default_statistics_target` (default 100) controls how many MCV entries and histogram
buckets are collected. Raising it per column
(`ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000`) helps on columns with skewed distributions,
at the cost of slower `ANALYZE` and slightly slower planning. This is the standard fix when a
specific column's estimates are consistently wrong.

#### 2. Correlated columns and extended statistics

**Theory.** By default the planner assumes columns are **independent**: it estimates
`P(a AND b) = P(a) x P(b)`. When columns are correlated, this underestimates dramatically.

**Example.** A table with `city` and `country`. `WHERE city = 'Paris' AND country = 'France'`: the
planner multiplies the two selectivities as if a Paris row could be in any country, so it might
estimate 50 rows where the truth is 50,000. It then picks a nested loop join suited to 50 rows and
the query takes minutes. The fix:

```sql
CREATE STATISTICS city_country (dependencies, ndistinct) ON city, country FROM addresses;
ANALYZE addresses;
```

**Advanced.** Underestimation is far more dangerous than overestimation. Underestimating leads to a
**nested loop** — repeat the inner lookup once per outer row — which is superb for 10 rows and
disastrous for a million. Overestimating leads to a hash join, which is merely somewhat slower than
optimal. That asymmetry is why "the estimate said 1 row, the reality was 2 million" is the single
most common root cause of a query that suddenly takes minutes.

#### 3. Cost constants

**Theory.** The planner converts estimated work into an abstract cost using constants:
`seq_page_cost` (1.0), `random_page_cost` (4.0), `cpu_tuple_cost`, and so on. `work_mem` decides how
much memory a sort or hash may use before spilling to disk.

**Example.** `random_page_cost = 4` encodes the assumption that a random read is four times more
expensive than a sequential one — true for spinning disks, false for SSDs and cloud block storage
where the ratio is closer to 1.1-2. Leaving the default on SSD-backed hardware makes the planner
systematically prefer sequential scans over index scans. Setting `random_page_cost = 1.1` is one of
the most common and highest-impact Postgres tuning changes.

**Advanced.** `work_mem` is **per operation, per connection**, not per query — a query with three
sorts and 100 connections can use 300x `work_mem`. Setting it high globally is how people run
databases out of memory. The right approach is a modest global value plus `SET LOCAL work_mem` for
specific heavy queries. In a plan, look for "external merge Disk: 240MB" — that is a sort that
spilled, and raising `work_mem` for that query will help a lot.

#### 4. Plan caching and parameter sensitivity

**Theory.** Prepared statements can be planned once and reused. A **generic plan** is made without
knowing the parameter values; a **custom plan** is made for the specific values. Postgres uses
custom plans for the first five executions, then switches to a generic plan if it does not look
worse on average.

**Example.** The classic symptom: a query is fast in `psql` and slow from the application. Causes to
check, in order: the application uses a prepared statement and got a generic plan that is bad for
skewed data; different session settings (`work_mem`, `search_path`); different parameter types
causing an implicit cast that disables an index; or the application's data volume differs from your
test. Force the behaviour with `plan_cache_mode = force_custom_plan` to confirm the diagnosis.

**Advanced.** This is the same family of problem as SQL Server's "parameter sniffing", where the
plan is built for the first parameter seen and is terrible for the rest. It matters most for
queries on skewed columns — a `tenant_id` filter where one tenant holds 90% of rows needs different
plans for different tenants, and any single cached plan will be wrong for someone.

### Interview questions

- "The planner estimates 1 row but gets 2 million. Consequences and fixes?"
- "A query is fast in psql and slow from the app. Why?"
- "What is `random_page_cost` and why is the default wrong on SSDs?"
- "Two correlated columns give bad estimates. What do you do?"

---

## DB19 · Reading EXPLAIN and EXPLAIN ANALYZE

`Advanced` · Requires: `DB02`, `DB15`, `DB18` · Unlocks: `DB20`, `DB39`

### Preface

`EXPLAIN` shows the plan the database intends to use. `EXPLAIN ANALYZE` actually runs the query and
shows what happened. The gap between estimated and actual rows is where the answer usually is.

Expect to be handed a plan in an interview and asked to talk through it. Practise on real queries
from your own application until it is comfortable.

### Details

#### 1. How to read a plan

**Theory.** Read it **inside out and bottom up**: the most indented nodes run first and feed their
parents. Each node shows estimated cost, estimated rows, and with `ANALYZE`, actual time, actual
rows, and loop count.

**Example.** The command to use every time:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) SELECT ...;
```

`BUFFERS` is essential — it shows how many pages were read from cache (`shared hit`) versus from
disk (`shared read`), which tells you whether you have an I/O problem or a CPU problem. Without it
you are guessing.

**Advanced.** The trap that catches almost everyone: **`actual time` and `rows` are per loop, not
totals.** A node showing `actual time=0.8..0.9 rows=1 loops=50000` did not take 0.9ms — it took
about 45 seconds. Always multiply by `loops`. This single detail is the most common reason people
misread a plan, and getting it right in an interview is a clear signal.

#### 2. Scan types

**Theory.**
- **Seq Scan** — read the whole table. Correct for small tables or unselective conditions.
- **Index Scan** — walk the index, fetch each matching row from the table. Good for few rows.
- **Index Only Scan** — answer entirely from the index (`DB16`).
- **Bitmap Heap Scan** — build a bitmap of matching rows from one or more indexes, then read the
  table in physical order. Good for a medium number of rows, or for combining indexes.

**Example.** Seeing a `Seq Scan` is not automatically bad. On a 500-row lookup table it is the right
plan. It is a problem when the table is large and the filter is selective — then look for a missing
index, a non-sargable predicate (`DB20`), or bad statistics.

**Advanced.** The progression as row count rises is: Index Scan (few rows) → Bitmap Heap Scan (more
rows, avoids repeated random access by sorting first) → Seq Scan (many rows). If you see a Bitmap
Heap Scan with a high `Rows Removed by Filter`, a better composite index usually exists that would
make it a plain Index Scan with no filtering.

#### 3. Join algorithms

**Theory.**
- **Nested Loop** — for each row of the outer input, look up matches in the inner. Excellent when
  the outer is small and the inner has an index. Catastrophic when the outer is large.
- **Hash Join** — build a hash table from the smaller input, probe with the larger. Best for large
  unsorted inputs. Needs `work_mem`; spills to disk if it does not fit.
- **Merge Join** — both inputs sorted, walk them together. Best when both are already sorted, for
  example from index scans.

**Example.** The disaster pattern: the planner estimates the outer input at 10 rows, chooses a
nested loop, and the real count is 500,000 — so the inner lookup runs half a million times. In the
plan you see `loops=500000` and a total time in minutes. The fix is to correct the estimate
(`ANALYZE`, extended statistics) rather than to fight the join type.

**Advanced.** You can disable a join type temporarily for diagnosis (`SET enable_nestloop = off`) to
see what the alternative would cost. This is a diagnostic technique, not a fix to ship — if
disabling nested loops makes the query fast, the real problem is the row estimate, and that is what
you should correct.

#### 4. Finding the actual problem

**Theory.** A practical checklist, in order: find the node where estimated and actual rows diverge
most (the root cause is usually its *input*, not the node itself); check for high `loops`; check for
`Rows Removed by Filter` (rows read and thrown away — an index opportunity); check for sorts
spilling to disk; check the buffers split between hit and read.

**Example.** A worked reading: `Sort (actual rows=1200000) ... Sort Method: external merge
Disk: 340MB` means the query sorted 1.2 million rows on disk. Options: an index providing the order
so the sort disappears; a `LIMIT` applied earlier; more `work_mem` for this query; or reducing the
rows before sorting.

**Advanced.** Use a visualiser for complex plans — `explain.dalibo.com` or `pev2` — which highlight
the slowest node and the worst estimate automatically. In an interview, saying "I usually paste it
into a plan visualiser, but let me walk you through it manually" is perfectly good, provided you can
then actually do it.

### Interview questions

- "Walk me through this plan and tell me the fix." (be ready to do this live)
- "When is a sequential scan the right plan?"
- "A node says `actual time=0.9 rows=1 loops=50000`. How long did it take?"
- "What does `BUFFERS` tell you that the plan alone does not?"

---

## DB20 · Query tuning patterns

`Advanced` · Requires: `DB03`, `DB16`, `DB17`, `DB19` · Unlocks: `DB21`, `DB39`, `SD05`

### Preface

A handful of patterns cause most slow queries in real applications. Knowing them by name lets you
diagnose in seconds what would otherwise take an afternoon.

The big four: **N+1 queries**, **non-sargable predicates**, **offset pagination**, and **counting
everything**.

### Details

#### 1. N+1 queries

**Theory.** Fetch a list of N items, then issue one query per item for related data. Total: N+1
queries. Each is fast; together they are slow, because you pay the round-trip latency N times.

**Example.** In TypeORM, iterating orders and accessing `order.customer` on a lazy relation issues a
query per order. The fixes: eager loading with a join (`relations: ['customer']` or an explicit
`leftJoinAndSelect`); a second batched query (`WHERE customer_id IN (...)`), which is what Prisma and
Django's `prefetch_related` do; or a DataLoader that batches within one request tick (`A15`).

**Advanced.** Joining is not always better than a second query. A join multiplies rows: fetching 100
orders each with 50 line items returns 5,000 rows, and the parent columns are repeated 50 times each
— significant network and parsing cost. Two queries (100 orders, then 5,000 items in one query) can
be faster and use less memory. Django makes the trade explicit with `select_related` (join) versus
`prefetch_related` (second query); knowing when to use each is a genuine skill (`F21`).

#### 2. Sargability

**Theory.** "Sargable" means the predicate can use an index. A predicate becomes non-sargable when
you apply a function to the column, force an implicit type cast, or use a leading wildcard.

**Example.** Non-sargable on the left, the fix on the right:

| Slow | Fast |
|---|---|
| `WHERE YEAR(created_at) = 2024` | `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` |
| `WHERE lower(email) = $1` | index on `lower(email)`, or a `citext` column |
| `WHERE id::text = $1` | `WHERE id = $1::int` — cast the parameter, not the column |
| `WHERE name LIKE '%foo%'` | trigram index, or full-text search |
| `WHERE a = $1 OR b = $2` | `... WHERE a = $1 UNION ALL ... WHERE b = $2` (with care over duplicates) |

**Advanced.** Implicit casts are the sneakiest: a `bigint` column compared with a string parameter
(common when an ORM or a driver sends everything as text) makes the database cast the **column**, so
the index is unusable. It shows up as a sequential scan on a query that obviously has an index. Check
the plan for a cast around the column name — and fix the parameter type in the application rather
than adding an expression index.

#### 3. Pagination

**Theory.** `OFFSET n LIMIT m` must generate and discard the first `n` rows. Cost grows linearly
with the page number. **Keyset (or seek) pagination** uses the last row's sort key as the starting
point, so every page costs the same.

**Example.**

```sql
-- offset: page 5000 reads and throws away 100,000 rows
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 100000;

-- keyset: constant time, uses the index directly
SELECT * FROM orders
WHERE (created_at, id) < ($last_created_at, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

The tuple comparison `(created_at, id) < (...)` handles ties correctly — and requires an index on
`(created_at DESC, id DESC)`.

**Advanced.** Keyset pagination is also **stable**: with offset pagination, a row inserted while the
user is browsing shifts everything, so they see a duplicate on the next page or miss a row entirely.
Keyset has no such problem. The trade-off is that you cannot jump to page 57 — which is nearly
always acceptable, since almost nobody does, and infinite scroll does not offer it anyway (`A05`).

#### 4. Counting

**Theory.** `SELECT COUNT(*)` on a large table must inspect every row (Postgres cannot use an index
alone because of visibility, `DB16`). On a 100-million-row table that is seconds, every page load.

**Example.** The options in order of preference: (1) do not show a total — "showing 1-20" and a
Next button; (2) show an **estimate** from the planner's statistics:

```sql
SELECT reltuples::bigint FROM pg_class WHERE relname = 'orders';
```

or for a filtered count, parse the planner's row estimate from `EXPLAIN`; (3) maintain a counter
table updated by triggers, accepting the write contention; (4) cap it —
`SELECT count(*) FROM (SELECT 1 FROM t WHERE ... LIMIT 1000) x` and display "1000+".

**Advanced.** MyISAM kept an exact row count and MVCC engines cannot, because different transactions
see different numbers of rows — the count is genuinely a per-snapshot value. Explaining *why* exact
counts are expensive in an MVCC database, rather than just offering workarounds, is the senior
answer.

#### 5. Bulk operations

**Theory.** Inserting rows one at a time pays a round trip and a transaction commit each time.
Batching is one or two orders of magnitude faster.

**Example.** From slowest to fastest: 10,000 individual inserts (~10,000 round trips); multi-row
`INSERT ... VALUES (...), (...), ...` in batches of 500-1,000; `COPY` from a stream, which is by far
the fastest path into Postgres. For updates, join against a `VALUES` list or a temporary table rather
than issuing N statements.

**Advanced.** Do not batch without limit: a single enormous transaction holds locks, generates huge
WAL, blocks vacuum (`DB08`) and cannot be resumed if it fails. The sweet spot is usually 1,000-10,000
rows per transaction with a short pause between batches so replication and vacuum can keep up. For
very large loads, dropping indexes, loading, then rebuilding them is often much faster than
maintaining them during the load.

### Interview questions

- "Page 5000 of a listing takes 8 seconds. Fix it."
- "Give five reasons a query that was fast last month is slow now." (data growth, stale statistics,
  a dropped or bloated index, a plan flip from parameter change, a new concurrent workload)
- "Show me an N+1 your ORM generated and three ways to fix it."
- "Why is `COUNT(*)` slow, and what do you show the user instead?"

---

## DB21 · Connections and pooling

`Advanced` · Requires: `DB20` · Unlocks: `DB39`, `F24`, `C14`

### Preface

A database connection is expensive. In Postgres each one is a separate operating-system process
using several megabytes, so hundreds of connections consume gigabytes and thrash the scheduler.

The counter-intuitive rule: **fewer connections usually give more throughput.** A pool of 20 doing
work steadily beats 200 fighting each other.

### Details

#### 1. Sizing the pool

**Theory.** Beyond the point where all CPUs are busy, extra concurrent queries add context switching
and lock contention rather than throughput. A common starting formula is
`connections ≈ (core_count x 2) + effective_spindle_count` — so for an 8-core server with SSDs,
roughly 20, not 200.

**Example.** The arithmetic people forget: the pool size is **per process**. 20 Kubernetes pods with
a pool of 10 each is 200 connections to one database. Add a worker deployment with 10 pods and you
have 300. Postgres's default `max_connections` is 100. The calculation is
`pods x pool_size + workers x pool_size + migrations + admin < max_connections`, and it must be
redone every time you scale out.

**Advanced.** Little's Law explains why a small pool works (`C14`): if a query takes 5ms, one
connection serves 200 queries per second, so 20 connections serve 4,000 per second. If you need more
than that, the answer is faster queries, not more connections. When the pool is exhausted, requests
**queue** — and a queue in front of the pool is usually better than queueing inside the database,
because you control the timeout and can shed load.

#### 2. Connection poolers

**Theory.** PgBouncer (or the built-in pooling in RDS Proxy, Supavisor, pgcat) sits between your
application and Postgres, multiplexing many client connections onto few server connections. Three
modes:
- **session** — a server connection is held for the whole client session. Safe, little benefit.
- **transaction** — a server connection is held only for the duration of a transaction. The
  standard choice, and a huge multiplier.
- **statement** — per statement. Multi-statement transactions are impossible.

**Example.** With transaction pooling, 2,000 application connections can share 25 server connections,
because most of the time an application connection is idle between transactions.

**Advanced.** Transaction pooling **breaks anything that relies on session state**, and this is a
classic production surprise: `SET` statements, session-level advisory locks (`DB09`), `LISTEN`/
`NOTIFY`, temporary tables, and prepared statements (before PgBouncer 1.21 and Postgres protocol
improvements) may land on a different backend than expected. If your ORM uses prepared statements by
default, you must either disable them or use a pooler that supports them. Knowing this specific
incompatibility list is a strong, practical signal.

#### 3. Timeouts

**Theory.** Without timeouts, one bad query holds a connection indefinitely and the pool drains. The
essential settings: `statement_timeout` (kill long queries),
`idle_in_transaction_session_timeout` (kill leaked transactions), `lock_timeout` (do not wait
forever for a lock), plus the pool's own acquisition timeout and the client's query timeout.

**Example.** Sensible defaults for a web application: `statement_timeout = 30s` globally, with a
lower value (say 5s) for the interactive API role and a much higher one for a reporting role;
`idle_in_transaction_session_timeout = 60s`; `lock_timeout = 3s` for migrations. Set them per role
with `ALTER ROLE ... SET`, so they apply without application changes.

**Advanced.** Timeouts must be layered consistently: the client timeout should be slightly longer
than the server `statement_timeout`, so the server cancels the query rather than the client
abandoning a query that keeps running. A client that times out without cancelling leaves the
database still working — the same wasted-work problem as deadline propagation in `M04`.

#### 4. Serverless and connection storms

**Theory.** Serverless functions scale to many concurrent instances, each wanting its own connection,
and they cannot share a pool because each lives in its own process. A traffic spike becomes a
connection spike that can take the database down.

**Example.** 1,000 concurrent Lambdas = up to 1,000 connection attempts. Mitigations: a proxy such as
RDS Proxy or PgBouncer in front; a data API over HTTP; limiting function concurrency; or using a
database designed for it (Neon, PlanetScale's HTTP driver). This is a genuine architectural
constraint of serverless plus relational databases, and naming it is a good sign of practical
experience.

**Advanced.** Connection *establishment* is itself expensive — TCP handshake, TLS handshake, Postgres
authentication, then backend process creation. A storm of new connections can saturate the database
before a single query runs. This is why a pooler helps even when the total connection count would be
acceptable: it keeps warm connections and absorbs the churn.

### Interview questions

- "200 Nest pods with a pool size of 10 against one Postgres. What happens and what do you do?"
- "Why did enabling PgBouncer transaction pooling break your ORM?"
- "How do you size a connection pool?"
- "Which timeouts do you set on a production database, and why each?"
