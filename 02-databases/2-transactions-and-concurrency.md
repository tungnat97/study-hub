[← back to the field index](README.md)

# Databases · Part 2 — Transactions, Isolation & Concurrency

Nodes `DB06`–`DB12`. The highest-yield part of the whole study pack for a senior interview.

---

## DB06 · Transactions and ACID

`Intermediate` · Requires: `DB02` · Unlocks: `DB07`, `DB09`, `DB12`, `M13`, `M15`, `F10`

### Preface

A transaction groups several statements so they either all take effect or none do. That is the
whole idea, and it is what lets you move money between two accounts without inventing or destroying
any.

ACID names four properties: **Atomicity** (all or nothing), **Consistency** (your rules stay true),
**Isolation** (concurrent transactions do not corrupt each other), **Durability** (once committed,
it survives a crash).

The practical senior skill is knowing where a transaction should begin and end — and knowing that
long transactions are the root of most database pain.

### Details

#### 1. What each letter actually means

**Theory.** **Atomicity** is implemented by undo information, so a failed transaction can be rolled
back. **Consistency** is the odd one out: it means your constraints and application invariants hold
before and after — the database only enforces what you declared. **Isolation** is about concurrency
and is the interesting letter (`DB07`). **Durability** means committed data survives a crash,
implemented by write-ahead logging (`DB12`).

**Example.** A transfer: debit A, credit B. Atomicity means you can never debit without crediting.
Consistency means a `CHECK (balance >= 0)` constraint holds. Isolation means another transaction
reading both balances never sees the money missing from both. Durability means that after `COMMIT`
returns, a power failure cannot undo it.

**Advanced.** The C is often described as the least meaningful letter, because it is mostly the
application's responsibility — the database will happily let you write business-nonsense that
violates no declared constraint. It is worth saying this in an interview; it shows you have thought
about the acronym rather than recited it.

#### 2. Where a transaction begins and ends

**Theory.** A transaction should cover exactly one business operation, start as late as possible,
and finish as early as possible. Everything inside holds locks and, in MVCC systems, holds back
cleanup of old row versions.

**Example.** The rule to state plainly: **never make a network call inside a transaction.**

```js
// bad: the transaction stays open for the duration of an external HTTP call
await dataSource.transaction(async (m) => {
  const order = await m.save(Order, dto);
  await stripe.charges.create(...);   // 2 seconds, or hangs
  await m.update(Order, order.id, { status: 'PAID' });
});
```

If Stripe takes 5 seconds, you hold row locks for 5 seconds. If it hangs, you hold them until the
timeout. Under load, every request does this and the database runs out of connections. The correct
shape: transaction 1 records the intent, the external call happens outside any transaction,
transaction 2 records the result — with an idempotency key so a retry is safe (`A09`).

**Advanced.** The same reasoning applies to any slow work inside a transaction: image processing,
large in-memory computation, waiting on a lock, or a user interaction. "Open a transaction when the
user opens the edit form and commit when they save" is the classic disaster; the correct pattern is
optimistic concurrency with a version column (`DB10`).

#### 3. Savepoints and nested transactions

**Theory.** SQL has no true nested transactions. What exists is **savepoints**: a marker you can
roll back to without aborting the whole transaction.

**Example.** Django's `transaction.atomic()` uses a real transaction at the outermost level and
savepoints for inner blocks. So an inner block can fail and be rolled back while the outer
transaction continues. Spring's `@Transactional(propagation = NESTED)` is the same mechanism, while
`REQUIRES_NEW` suspends the current transaction and opens a genuinely separate one — which needs a
second connection and can deadlock against its own parent if both touch the same rows.

**Advanced.** Savepoints are not free: in Postgres, many savepoints in one transaction cost memory
and can degrade performance noticeably. A loop that wraps each of 10,000 iterations in a savepoint
(a common ORM pattern for "try this row, skip failures") is a real performance problem. Prefer
validating before writing, or batching.

#### 4. Long transactions are the enemy

**Theory.** A long-running transaction holds locks, blocks schema changes, and in Postgres prevents
`VACUUM` from cleaning up dead row versions anywhere in the database — not just in tables it
touched.

**Example.** The most common production symptom: a connection sits in `idle in transaction` because
the application opened a transaction and then did something slow (or leaked it entirely). Effects:
table bloat grows, autovacuum cannot reclaim space, queries slow down, and an `ALTER TABLE` waiting
behind it blocks every subsequent query on that table (`DB09`). Set
`idle_in_transaction_session_timeout` so the database kills these automatically — it is one of the
highest-value settings in Postgres.

**Advanced.** Batch jobs are the usual culprit: "update 5 million rows" in one transaction produces
enormous undo/WAL volume, a long lock, and a rollback that takes as long as the work if it fails.
The correct pattern is chunking — process 1,000 rows per transaction with a resumable cursor,
committing between chunks, with a small pause to let replication and vacuum keep up (`DB22`).

### Interview questions

- "What breaks if you call a third-party API inside a DB transaction?"
- "What does the C in ACID actually mean?"
- "What is `idle in transaction` and why does it matter?"
- "How do you update 5 million rows safely?"

---

## DB07 · Isolation levels and anomalies

`Advanced` · Requires: `DB02`, `DB06` · Unlocks: `DB08`, `DB09`, `DB10`, `SD08`

### Preface

Isolation levels define how much concurrent transactions are allowed to interfere. Higher isolation
means fewer surprises and less concurrency.

Almost nobody runs at the strictest level, which means your application is exposed to specific,
nameable anomalies. Knowing which ones, and how to defend against them, is a core senior skill and a
very common interview question.

### Details

#### 1. The anomalies

**Theory.**
- **Dirty read** — you read data another transaction has written but not committed. Prevented at
  READ COMMITTED and above.
- **Non-repeatable read** — you read a row twice in one transaction and get different values,
  because someone committed in between.
- **Phantom read** — you run the same query twice and get different *rows*, because someone
  inserted.
- **Lost update** — two transactions read a value, both modify it, and the second overwrites the
  first's change.
- **Write skew** — two transactions each read a set of rows, each makes a decision based on what it
  read, and each writes to *different* rows; individually correct, together they violate a rule
  spanning both.

**Example.** Lost update, the most common in real applications:

```
T1: SELECT stock FROM items WHERE id=1;   -- 10
T2: SELECT stock FROM items WHERE id=1;   -- 10
T1: UPDATE items SET stock = 9  WHERE id=1;
T2: UPDATE items SET stock = 9  WHERE id=1;   -- should be 8
```

Two items sold, one deducted. The fix is not a higher isolation level but writing
`UPDATE items SET stock = stock - 1 WHERE id = 1 AND stock > 0` — a single atomic statement — or
locking the row with `SELECT ... FOR UPDATE`.

**Advanced.** **Write skew** is the anomaly that snapshot isolation does *not* prevent and that
interviewers use to separate levels of understanding. The canonical case: a hospital requires at
least one doctor on call. Two doctors, each in their own transaction, check "is anyone else on
call?" — both see the other, both go off call, and the invariant is violated even though neither
transaction wrote the row the other read. Because they wrote different rows, no write-write conflict
is detected. Real-world versions: two bookings for the same room, a balance shared across two
accounts, allocating the last seat.

#### 2. The levels

**Theory.**
| Level | Dirty read | Non-repeatable | Phantom | Lost update | Write skew |
|---|---|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible | possible | possible |
| READ COMMITTED | no | possible | possible | possible | possible |
| REPEATABLE READ (Postgres = snapshot) | no | no | no* | detected** | possible |
| SERIALIZABLE (Postgres SSI) | no | no | no | no | no |

\* Postgres's REPEATABLE READ is snapshot isolation and does prevent phantoms.
\*\* Postgres aborts the second writer with a serialization failure rather than losing the update.

**Example.** Defaults differ and it matters: **Postgres defaults to READ COMMITTED**, **MySQL InnoDB
defaults to REPEATABLE READ**, and they implement REPEATABLE READ differently — MySQL uses
next-key locking to prevent phantoms in many cases, while Postgres uses snapshots. Code that is
correct on MySQL can be subtly wrong on Postgres and vice versa.

**Advanced.** In Postgres READ COMMITTED, each *statement* sees a fresh snapshot, so two queries in
one transaction can see different data. A subtle consequence: an `UPDATE ... WHERE status='pending'`
that waits for a lock re-evaluates its `WHERE` clause against the *new* version of the row after the
lock is released — so a row that no longer matches is skipped. This "EvalPlanQual" behaviour is
usually what you want and occasionally very surprising.

#### 3. Serializable and retry loops

**Theory.** Postgres's SERIALIZABLE uses Serializable Snapshot Isolation: it lets transactions run
optimistically and aborts one when it detects a dangerous pattern, raising a serialization failure
(SQLSTATE `40001`). It is genuinely correct — no anomalies at all — at the cost that your
application **must** retry.

**Example.** Every transaction needs a retry wrapper:

```js
async function withRetry(fn, attempts = 3) {
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (e) {
      if (e.code === '40001' && i < attempts - 1) { await sleep(50 * 2 ** i * Math.random()); continue; }
      throw e;
    }
  }
}
```

Requirements: transactions must be short (longer means more conflicts), must be safe to re-execute
(so no side effects such as sending emails inside them), and the retry must have backoff and a
limit.

**Advanced.** SSI's cost rises with contention and with the number of concurrent transactions,
because it tracks read dependencies. Pragmatic guidance: use READ COMMITTED as the default with
explicit locking or atomic statements where needed; use SERIALIZABLE for the specific transactions
where write skew is a genuine risk and the logic is complex enough that reasoning about locks is
error-prone. Being able to justify that mixed approach is a strong answer.

#### 4. Defending without raising isolation

**Theory.** Most anomalies can be handled at READ COMMITTED with three techniques: atomic statements,
explicit locks, and version checks.

**Example.** Same problem, three solutions:
- **Atomic**: `UPDATE items SET stock = stock - 1 WHERE id = $1 AND stock > 0` — check the affected
  row count; zero means out of stock. Best when it fits.
- **Pessimistic**: `SELECT ... FOR UPDATE` to lock the row, then compute and write. Necessary when
  the decision needs application logic.
- **Optimistic**: `UPDATE ... SET v = v + 1 WHERE id = $1 AND v = $expected` — zero rows means
  someone else changed it; retry or report a conflict (`DB10`).

For write skew specifically, you must lock something that represents the *rule*: either lock all the
rows you read (`SELECT ... FOR UPDATE` on the doctors table), or materialise the conflict by writing
to a shared row, or use SERIALIZABLE.

**Advanced.** "Materialising the conflict" is the technique worth naming: if there is no row that
both transactions write, create one — a row per shift in a `shifts` table that both transactions
update — so the database can detect the write-write conflict. This turns write skew into a normal,
detectable conflict, and it is the standard trick when SERIALIZABLE is too expensive.

### Interview questions

- "Explain write skew and why REPEATABLE READ does not prevent it."
- "Your app runs SERIALIZABLE and randomly throws 40001. What must the app do?"
- "Two concurrent `balance = balance - 10` — walk through it at each isolation level."
- "What is Postgres's default isolation level? MySQL's? Do they mean the same thing?"

---

## DB08 · MVCC internals

`Advanced` · Requires: `DB07` · Unlocks: `DB11`, `DB12`, `DB18`, `DB39`

### Preface

Multi-Version Concurrency Control is how modern databases let readers and writers run at the same
time without blocking each other: instead of overwriting a row, a write creates a **new version**,
and each transaction sees the version that was current when it started.

The consequence you must understand operationally: old versions accumulate and must be cleaned up.
In Postgres this is `VACUUM`, and when it cannot keep up you get **bloat** — which is the single
most common cause of a Postgres database mysteriously getting slower.

### Details

#### 1. How Postgres MVCC works

**Theory.** Every row version carries `xmin` (the transaction that created it) and `xmax` (the
transaction that deleted or superseded it). A transaction takes a **snapshot** listing which
transactions were committed at its start, and a row version is visible if `xmin` is visible to the
snapshot and `xmax` is not. An `UPDATE` is therefore an insert of a new version plus marking the old
one dead — Postgres never modifies a row in place.

**Example.** Consequences that follow directly: readers never block writers and writers never block
readers; a rolled-back transaction leaves dead rows behind rather than undoing anything; and a table
updated a million times contains a million dead versions until vacuum removes them, even if it holds
only one live row.

**Advanced.** Because an update writes a whole new row version, **updating one column rewrites the
entire row**, including any large values, and normally requires updating every index. The
optimisation for this is **HOT** (heap-only tuple) updates: if no indexed column changed and there
is free space on the same page, the new version links from the old one within the page and no index
update is needed. This is why leaving free space (`fillfactor`, typically 80-90 for
update-heavy tables) and avoiding unnecessary indexes measurably improves update throughput.

#### 2. Bloat and VACUUM

**Theory.** `VACUUM` marks dead row versions as reusable space (it does not shrink the file);
`VACUUM FULL` rewrites the table compactly but takes an exclusive lock and needs double the disk.
Autovacuum runs it automatically based on thresholds. **Bloat** is the accumulated dead space, and
it slows everything because the same number of live rows occupies more pages, so more I/O and more
memory are needed for the same work.

**Example.** A 2GB table holding 500MB of live data. Sequential scans read four times more than
necessary, the cache holds mostly dead rows, and index scans touch more pages. The usual causes:
a long-running transaction or an `idle in transaction` connection holding back the visibility
horizon (vacuum cannot remove versions that *might* still be visible to it); autovacuum tuned too
conservatively for a high-write table; or a very high update rate on a small table.

**Advanced.** The tuning knobs worth knowing: `autovacuum_vacuum_scale_factor` (default 0.2 means a
table is only vacuumed after 20% of it is dead — far too lazy for a large hot table; set it much
lower per table), `autovacuum_vacuum_cost_limit` (how much work vacuum may do before pausing — the
default is often too low for modern hardware), and per-table overrides for the worst offenders. And
know the emergency: **transaction ID wraparound**. Postgres transaction ids are 32-bit; rows must be
"frozen" before 2 billion transactions pass or the database refuses writes to protect itself. If
vacuum cannot keep up, you get escalating warnings and eventually a forced shutdown — a genuine,
famous outage mode.

#### 3. MySQL InnoDB's approach

**Theory.** InnoDB keeps the current row in place and stores old versions in the **undo log**. Reads
that need an older version reconstruct it by walking the undo records backwards.

**Example.** The equivalent problem has a different name: the **history list length**. A long-running
transaction forces InnoDB to retain undo records, the history list grows, and reads that must
reconstruct old versions get progressively slower. Purge threads clean it up, and a long transaction
blocks purge — the same root cause as Postgres bloat, with different symptoms.

**Advanced.** The trade-off between the two designs: Postgres makes rollback nearly free (just mark
the transaction aborted) and pays with vacuum; InnoDB makes rollback expensive (apply undo records)
and reads of old versions expensive, but avoids table bloat. Being able to compare them accurately
is a strong senior signal, and it explains why "just switch databases" rarely removes a
concurrency problem — it moves it.

#### 4. Operational detection

**Theory.** You need to know your bloat level, your oldest transaction, and whether autovacuum is
keeping up.

**Example.** The queries to have ready: `pg_stat_user_tables` for `n_dead_tup`, `last_autovacuum`
and `n_live_tup`; `pg_stat_activity` filtered for `state = 'idle in transaction'` ordered by
`xact_start` to find the transaction holding back the horizon; and `pg_stat_progress_vacuum` to see
whether a vacuum is running and how far it has got.

**Advanced.** The three settings that prevent most of this trouble, worth naming in an interview:
`idle_in_transaction_session_timeout` (kill leaked transactions), `statement_timeout` (kill runaway
queries), and per-table `autovacuum_vacuum_scale_factor` on your largest, hottest tables. Setting
these proactively is the difference between a database that degrades gradually and one that stays
predictable.

### Interview questions

- "A table is four times its data size and queries got slow. Cause?"
- "Why does `UPDATE` in Postgres write a whole new row version, and what does that do to indexes?"
- "What is transaction ID wraparound?"
- "How does InnoDB's MVCC differ from Postgres's, and what does each cost?"

---

## DB09 · Locking

`Advanced` · Requires: `DB05`, `DB06`, `DB07` · Unlocks: `DB10`, `DB22`, `C11`

### Preface

Even with MVCC keeping readers and writers apart, two writers to the same row must be serialised —
that is what row locks do.

Most production incidents involving locks are not about two competing writes. They are about a lock
queue: one slow statement takes a strong lock, and every subsequent query piles up behind it, even
queries that would not have conflicted with each other.

### Details

#### 1. Lock modes and what conflicts

**Theory.** Locks come in modes; two locks conflict only if their modes are incompatible. Row-level:
`FOR UPDATE` (exclusive), `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`. Table-level, from
weakest to strongest: `ACCESS SHARE` (taken by `SELECT`), `ROW EXCLUSIVE` (taken by
`INSERT/UPDATE/DELETE`), `SHARE UPDATE EXCLUSIVE` (`VACUUM`, `CREATE INDEX CONCURRENTLY`),
`SHARE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` (most `ALTER TABLE`, `DROP`, `VACUUM FULL`).

**Example.** `ACCESS EXCLUSIVE` conflicts with everything, including plain `SELECT`. So an
`ALTER TABLE` blocks all reads for its duration — which is fine if it takes 2ms, and an outage if it
takes 4 minutes.

**Advanced.** Note that a foreign key reference takes a `FOR KEY SHARE` lock on the parent row. This
means inserting many child rows referencing one hot parent serialises on that parent, and it is a
frequent hidden cause of contention in schemas with a heavily-referenced lookup table.

#### 2. The lock queue — the outage pattern

**Theory.** Lock requests queue in order. A pending strong lock **blocks every later request**, even
weaker ones that would not have conflicted with the current holder. One long-running query plus one
`ALTER TABLE` equals a full stop on that table.

**Example.** The classic incident, step by step:
1. A reporting `SELECT` starts and runs for 10 minutes, holding `ACCESS SHARE`.
2. A deploy runs `ALTER TABLE orders ADD COLUMN ...`, which requests `ACCESS EXCLUSIVE` and waits.
3. Every new `SELECT` on `orders` now queues behind the pending `ALTER`.
4. Connections exhaust, the application stops, and the cause looks like "the deploy broke the site"
   when the real cause is the long query plus an unguarded DDL.

**Advanced.** The defence is to make DDL fail fast rather than wait:

```sql
SET lock_timeout = '2s';
ALTER TABLE orders ADD COLUMN note text;   -- fails quickly if it cannot get the lock
```

Then retry in a loop. This is mandatory practice for migrations on busy tables (`DB22`). Always set
`lock_timeout` for DDL, and never run migrations while a long reporting query is in progress.

#### 3. SELECT FOR UPDATE and SKIP LOCKED

**Theory.** `SELECT ... FOR UPDATE` locks the selected rows so no other transaction can modify them
until you commit. `NOWAIT` fails immediately instead of waiting. `SKIP LOCKED` silently ignores
rows already locked by someone else.

**Example.** `SKIP LOCKED` turns a table into a reliable work queue — one of the most useful
patterns in backend engineering:

```sql
BEGIN;
SELECT id, payload FROM jobs
WHERE status = 'pending' AND run_at <= now()
ORDER BY run_at
FOR UPDATE SKIP LOCKED
LIMIT 10;
-- process, then:
UPDATE jobs SET status = 'done' WHERE id = ANY($1);
COMMIT;
```

Many workers can run this concurrently: each grabs a different batch, no worker blocks another, and
a crash rolls back so the rows become available again. This gives transactional job handling with no
extra infrastructure (see `Q20`).

**Advanced.** Two refinements for production. Add a visibility timeout (`locked_until`) so a worker
that dies without releasing does not hold the job forever — or rely on the transaction ending on
disconnect, which is simpler and works if each job is processed within one transaction. And note
that `FOR UPDATE` in a transaction held open across a long processing step reintroduces the long
transaction problem (`DB06`) — for slow jobs, claim the row in a short transaction by setting
`status = 'running'`, then process outside it.

#### 4. Deadlocks

**Theory.** Two transactions each hold a lock the other wants. The database detects the cycle and
kills one with a deadlock error. It is normal and expected under concurrency; the application must
handle it by retrying.

**Example.** The cause is almost always inconsistent lock ordering:

```
T1: UPDATE accounts WHERE id=1;  then  UPDATE accounts WHERE id=2;
T2: UPDATE accounts WHERE id=2;  then  UPDATE accounts WHERE id=1;
```

The fix is to always acquire locks in a deterministic order — sort the ids before updating:
`ORDER BY id`. A bulk `UPDATE ... WHERE id IN (...)` can deadlock for the same reason if two
statements process the same rows in different orders, so sort there too.

**Advanced.** MySQL's InnoDB at REPEATABLE READ takes **gap locks** and **next-key locks** (locking
the gap between index entries to prevent phantoms), which produces deadlocks in situations that
would be conflict-free in Postgres — notably concurrent inserts into the same index range. If a team
reports frequent deadlocks on MySQL and not on Postgres, gap locking is usually why. Read the
deadlock log (`SHOW ENGINE INNODB STATUS`, or `log_lock_waits` in Postgres) rather than guessing.

#### 5. Advisory locks

**Theory.** Advisory locks are application-defined locks with no connection to any row. You choose a
number; the database guarantees only one session holds it.

**Example.** Ensuring only one instance runs a migration or a scheduled job:

```sql
SELECT pg_try_advisory_lock(12345);   -- returns true if acquired
```

This is a clean way to make a cron job safe across many pods (`F14`) without another dependency. Use
the session-level version for "one leader at a time" and the transaction-level version
(`pg_advisory_xact_lock`) when you want automatic release at commit.

**Advanced.** Advisory locks are tied to the **session**, so they do not survive a connection drop —
which is a feature (automatic release) and a hazard (silent loss of the lock while your work
continues). They also do not work through a transaction-mode connection pooler such as PgBouncer,
because you may get a different backend for the next statement (`DB21`). The transaction-scoped
variant is safe with pooling; the session-scoped one is not.

### Interview questions

- "Implement a job queue in Postgres."
- "You get random deadlocks on a two-row update. Fix it."
- "Why did `ALTER TABLE ADD COLUMN` take the site down?"
- "What is `SKIP LOCKED` and what problem does it solve?"

---

## DB10 · Optimistic versus pessimistic concurrency

`Intermediate` · Requires: `DB07`, `DB09` · Unlocks: `DB38`, `SD10`

### Preface

Two strategies for "two people might change this at the same time".

**Pessimistic**: lock it first, so nobody else can. **Optimistic**: do not lock; check at write time
whether anything changed, and fail if it did.

Optimistic wins when conflicts are rare, which is most of the time, and it is the only workable
option across a stateless HTTP request where you cannot hold a lock between page load and save.

### Details

#### 1. Optimistic concurrency

**Theory.** Each row carries a version (an integer, or a timestamp, or a row hash). Reads return the
version. Writes include the expected version in the `WHERE` clause. If zero rows are updated,
someone else changed it — you have a conflict.

**Example.**

```sql
UPDATE documents
SET content = $1, version = version + 1
WHERE id = $2 AND version = $3;
-- affected rows = 0  →  conflict
```

The application then decides: reject with a 409 and ask the user to reload, merge automatically if
the changes are to different fields, or retry the whole operation with fresh data.

**Advanced.** This is exactly the HTTP `ETag` / `If-Match` mechanism (`A04`), which is why it fits
REST APIs so naturally: return an `ETag` with the resource, require `If-Match` on the update, and
return `412 Precondition Failed` on mismatch. ORMs implement it too — Hibernate's `@Version`, and
TypeORM's `@VersionColumn`. Know that an ORM's optimistic locking only protects updates *through the
ORM*; raw SQL bypasses it unless the version check is in the statement.

#### 2. Pessimistic concurrency

**Theory.** Lock the row before reading it and hold the lock until you commit. Nobody else can
change it in the meantime. Correct, simple to reason about, and it serialises access, so throughput
on a hot row is limited by how long each transaction holds it.

**Example.** Decrementing inventory where the decision needs logic that cannot be expressed in one
statement:

```sql
BEGIN;
SELECT stock, reserved FROM items WHERE id = $1 FOR UPDATE;
-- application logic decides
UPDATE items SET stock = stock - $2 WHERE id = $1;
COMMIT;
```

**Advanced.** The failure mode is queueing on a hot row: if one transaction holds the lock for 50ms
and 200 requests per second want it, you have a queue that grows without bound. Mitigations: keep
the locked section as short as possible (do the logic before taking the lock where you can); use
`NOWAIT` or a `lock_timeout` so waiters fail fast instead of piling up; or remove the hot row
entirely by sharding the counter into N rows and summing them (`M32`).

#### 3. Choosing between them

**Theory.** Optimistic when conflicts are rare, transactions are long or span requests, or you
cannot hold a lock. Pessimistic when conflicts are likely, when a retry is expensive or
user-visible, or when you must guarantee progress rather than risk repeated failures.

**Example.** A user editing a document over five minutes: optimistic — you cannot hold a database
lock for five minutes. Decrementing the last unit of stock during a flash sale: pessimistic, or a
single atomic `UPDATE`, because under high contention optimistic retries would fail repeatedly and
livelock.

**Advanced.** Under heavy contention, optimistic concurrency degrades badly: every transaction does
the work, then fails, then repeats — wasted work grows with contention while throughput falls. That
is the crossover point where pessimistic locking (or a queue that serialises per key) becomes
faster. Being able to describe that crossover is the senior answer to "which is better".

#### 4. The atomic-statement alternative

**Theory.** Often neither is needed: expressing the change as a single statement makes the database
do the serialisation for you, with no application round trip.

**Example.** `UPDATE items SET stock = stock - 1 WHERE id = $1 AND stock >= 1` is atomic, correct
under any isolation level, needs no lock held across a round trip, and tells you via the affected row
count whether it succeeded. Prefer this whenever the logic fits into an expression.

**Advanced.** The limit is that you cannot return rich information about *why* it failed, and you
cannot make a decision that requires reading other tables. That is where the other two strategies
earn their place. A good design instinct is: atomic statement first, then optimistic, then
pessimistic — moving down only when the simpler option cannot express the rule.

### Interview questions

- "Two users edit the same record in a UI form. Design the conflict handling."
- "When does optimistic locking perform worse than pessimistic?"
- "How does `If-Match` relate to a version column?"
- "Implement 'decrement stock, never below zero' three ways."

---

## DB11 · Physical storage

`Advanced` · Requires: `DB01`, `DB08` · Unlocks: `DB13`, `DB29`, `DB32`

### Preface

Databases read and write fixed-size **pages** (8KB in Postgres, 16KB in InnoDB), never individual
rows. Almost every performance characteristic follows from that: how many rows fit in a page, how
many pages a query must touch, and how many of those are already in memory.

The other structural fact is whether rows are stored in a heap (unordered) or clustered inside the
primary key index — Postgres does the first, MySQL InnoDB the second, and it changes how you design
keys and indexes.

### Details

#### 1. Pages and rows

**Theory.** A page holds a header, a list of item pointers, and the row data. Reading one row costs
reading its whole page. Therefore: narrower rows mean more rows per page, fewer pages per query, and
better cache use.

**Example.** A table with 20 columns where your query needs 3. Reading a row still loads the whole
page containing all 20 columns' worth of data. `SELECT *` on a wide table when you need three
columns wastes memory, network and CPU — and it prevents index-only scans (`DB16`).

**Advanced.** Column order affects row size because of **alignment padding**: Postgres aligns values
to their natural boundaries, so a `boolean` followed by a `bigint` wastes 7 bytes of padding.
Ordering columns from widest to narrowest (8-byte types, then 4-byte, then 2-byte, then 1-byte, then
variable-length) can shrink a row by 10-20% on wide tables. It is a micro-optimisation, but it is a
real one and it is a good answer to "how would you make this table smaller without changing the
data".

#### 2. Heap versus clustered index

**Theory.** **Postgres**: rows live in a heap in no particular order; every index (including the
primary key) stores a pointer (`ctid`) to the physical location. **InnoDB**: rows are stored *inside*
the primary key's B-tree, in key order; every secondary index stores the **primary key value**, not
a physical pointer.

**Example.** The InnoDB consequences are significant. A secondary index lookup needs two traversals:
find the primary key in the secondary index, then find the row in the primary index. And because
every secondary index stores the primary key, a wide primary key (a UUID, or a composite of three
columns) inflates every secondary index on the table.

**Advanced.** This is the precise reason a random UUID primary key is worse in InnoDB than in
Postgres: inserts land at random positions in the clustered index, causing page splits and
fragmentation of the *table itself*, not only of an index. In Postgres, inserts always append to the
heap regardless of the key, so the damage is limited to index locality. Both benefit from a
time-ordered key, but InnoDB benefits far more.

#### 3. Large values and TOAST

**Theory.** A row cannot span pages, so Postgres moves large values out of line into a separate
TOAST table, compressed, leaving a pointer behind. This happens automatically above roughly 2KB.

**Example.** A table with a large `jsonb` column: queries that do not select that column never touch
the TOAST table and are fast. Queries that do select it pay extra reads and decompression. This is a
concrete reason to avoid `SELECT *` and, sometimes, to move a large column into its own table so the
main table stays dense.

**Advanced.** TOAST interacts with updates: changing any column rewrites the row version (`DB08`),
but if the large TOASTed value did not change, the pointer is reused and the big value is not
rewritten. However, a `jsonb` column that you update frequently *is* rewritten in full each time —
which is why storing a large, frequently-mutated document in one JSONB column produces surprising
write amplification. Splitting hot fields into real columns fixes it.

#### 4. Fill factor and free space

**Theory.** `fillfactor` controls how full a page is packed on write. 100 means completely full, and
any later update must go to another page. Leaving space allows in-page (HOT) updates.

**Example.** For a heavily-updated table in Postgres, setting `fillfactor = 85` leaves room for new
row versions on the same page, enabling HOT updates that avoid touching indexes. The cost is 15%
more space and slightly more I/O for scans. For an append-only table, keep it at 100.

**Advanced.** The same idea applies to indexes: a B-tree built with `fillfactor = 90` leaves room for
inserts and reduces page splits, which matters for indexes on random keys. And it explains why
`REINDEX` sometimes dramatically improves performance — it rebuilds a fragmented, half-empty index
densely.

### Interview questions

- "Why is a random UUID primary key worse in InnoDB than in a Postgres heap table?"
- "What is TOAST and when does it hurt?"
- "How does column order affect table size?"
- "Why does `SELECT *` cost more than selecting three columns, even from the same rows?"

---

## DB12 · Write-ahead logging, durability and recovery

`Advanced` · Requires: `DB06`, `DB08` · Unlocks: `DB23`, `DB36`, `Q13`

### Preface

When you commit, the database must guarantee the change survives a crash — but writing the actual
data pages to disk on every commit would be far too slow, because they are scattered.

The solution is the **write-ahead log**: append a compact description of the change to a sequential
file and flush *that*. Sequential writes are fast, and after a crash the log is replayed to
reconstruct the data pages.

This one mechanism underlies durability, crash recovery, replication and point-in-time restore.

### Details

#### 1. The write-ahead rule

**Theory.** The rule: the log record describing a change must reach durable storage **before** the
modified data page does. On commit, the log up to that point is flushed with `fsync`. Data pages are
written later, in the background.

**Example.** A commit therefore costs one sequential flush rather than several random writes. After
a crash, recovery reads the log from the last checkpoint and replays anything not yet reflected in
the data files (redo), then rolls back transactions that never committed (undo, in systems that need
it).

**Advanced.** **Group commit** is the optimisation that makes this scale: instead of one `fsync` per
commit, the database batches concurrent commits into a single flush. This is why commit throughput
does not fall linearly with concurrency, and why a system with many concurrent small transactions
can outperform one doing the same work sequentially.

#### 2. Checkpoints

**Theory.** A checkpoint writes all dirty pages to disk and records that the log before that point is
no longer needed for recovery. Frequent checkpoints mean fast recovery and more constant I/O; rare
checkpoints mean less I/O but longer recovery and large I/O spikes.

**Example.** A badly-tuned Postgres shows periodic latency spikes every few minutes as a checkpoint
flushes a large volume of dirty pages at once. Tuning `max_wal_size` upward, and
`checkpoint_completion_target` to spread the writes over most of the interval, smooths it out.
Seeing "latency spikes on a regular cycle" and thinking "checkpoint" is a useful diagnostic reflex.

**Advanced.** **Full page writes** are the related cost: after a checkpoint, the first modification
of each page writes the *entire page* to the log, not just the change, to protect against torn pages
(a partial page write during a power failure). This means WAL volume spikes right after every
checkpoint, which matters for replication bandwidth and for cloud storage costs. It is also why
frequent checkpoints can increase total write volume substantially.

#### 3. What durability settings actually trade

**Theory.** `synchronous_commit = on` flushes the log before returning from commit.
`synchronous_commit = off` returns immediately and flushes within a short window. `fsync = off`
disables the flush entirely.

**Example.** The distinction that matters and is often confused:
- `synchronous_commit = off` — a crash loses up to a few hundred milliseconds of **committed
  transactions**, but the database remains **consistent and uncorrupted**. For some workloads
  (analytics ingest, clickstream) that is an entirely reasonable trade for a large throughput gain.
- `fsync = off` — a crash can leave the database **corrupted** and unrecoverable. Never in
  production. Acceptable only for throwaway test databases.

**Advanced.** `synchronous_commit` can be set **per transaction**, which is the sophisticated answer:
keep it on for financial writes and off for high-volume, low-value logging inserts in the same
database. Also worth knowing that `fsync` is only as honest as the storage beneath it — some
consumer drives and some virtualised storage acknowledge before the data is durable, which is why
databases on unknown storage should be tested for it.

#### 4. WAL as the foundation of everything else

**Theory.** Once you have an ordered, durable log of every change, you get several capabilities for
free: physical replication (ship the log to another server and replay it), point-in-time recovery
(restore a base backup and replay the log to a chosen moment), and change data capture (decode the
log into logical row changes).

**Example.** Point-in-time recovery in practice: a nightly base backup plus continuous WAL
archiving means you can restore to any second in the retention window — for instance to the moment
before a bad migration deleted three million rows (`DB36`). Debezium reads the same log to produce
change events for Kafka (`Q18`, `M15`).

**Advanced.** The operational hazard is **replication slots**: a slot guarantees the server keeps WAL
until a consumer has read it. If a replica or a CDC connector stops and nobody notices, WAL
accumulates until the disk fills and the primary shuts down. This is a genuinely common outage.
Always monitor slot lag and set `max_slot_wal_keep_size` so the server sacrifices the slot rather
than itself.

### Interview questions

- "How does the database guarantee durability without fsyncing every data page?"
- "What do you lose by setting `synchronous_commit = off`? How is that different from `fsync = off`?"
- "Your database has latency spikes every five minutes. What would you look at?"
- "An inactive replication slot filled the disk. Explain the mechanism."
