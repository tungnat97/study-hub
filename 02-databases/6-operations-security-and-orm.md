[← back to the field index](README.md)

# Databases · Part 6 — Operations, Security, Money & ORMs

Nodes `DB36`–`DB40`.

---

## DB36 · Backup, point-in-time recovery and disaster recovery

`Intermediate` · Requires: `DB12`, `DB22`, `DB23` · Unlocks: `O20`

### Preface

Backups exist for the day something destroys your data: a bad migration, a mistaken `DELETE`, a
ransomware event, a corrupted disk.

The sentence that matters: **an untested backup is not a backup.** Plenty of teams discover during
an incident that their backups have been failing for months, or that a restore takes nine hours when
the business expected one.

### Details

#### 1. Logical versus physical backups

**Theory.** A **logical** backup (`pg_dump`, `mysqldump`) exports SQL statements or a portable
archive. It is selective, human-readable, works across versions and platforms, and is slow to
restore on large databases. A **physical** backup copies the data files themselves
(`pg_basebackup`, filesystem or volume snapshots) — fast to take and restore, and tied to the same
major version and architecture.

**Example.** A sensible combination for a 2TB database: nightly physical base backups plus
continuous WAL archiving for fast full recovery and point-in-time restore; and a weekly logical dump
for the ability to extract a single table, and as protection against physical corruption being
faithfully copied into every physical backup.

**Advanced.** Restoring one table from a physical backup is genuinely awkward — you must restore the
whole cluster somewhere else and then export the table, which for 2TB means hours and a spare
machine. This is the specific case where a logical dump saves an incident, and it is a good reason
to keep both. Plan the *restore* paths, not the backup schedule.

#### 2. Point-in-time recovery

**Theory.** A base backup plus the WAL archived since it was taken lets you restore to any moment in
between: restore the base, then replay WAL up to a chosen timestamp or transaction id.

**Example.** The incident: a migration deleted three million rows 40 minutes ago. The recovery:
restore the base backup to a **new** instance, replay WAL to the moment just before the bad
statement, verify the data is correct, then extract the affected table and merge it back into
production — because the last 40 minutes contain legitimate writes you must not discard. Never
restore in place over a live database without a deliberate decision.

**Advanced.** The reason you can identify the precise moment is that you need `recovery_target_time`
or, better, `recovery_target_lsn` — which means your logging must let you find when the statement
ran. This is an argument for logging DDL and bulk operations with timestamps. Also note that
extracting and merging is usually application-specific work: budget hours, not minutes, and write the
runbook before you need it.

#### 3. RPO, RTO and what replicas do not give you

**Theory.** **RPO** (recovery point objective) is how much data you can afford to lose; **RTO**
(recovery time objective) is how long you can afford to be down. They are business decisions that
determine your architecture and your budget.

**Example.** RPO of 5 minutes means WAL archived at least every 5 minutes, or synchronous
replication. RTO of 15 minutes means you cannot restore 2TB from cold storage — you need a warm
standby ready to promote. Stating both as numbers, and deriving the architecture from them, is how
this question should be answered.

**Advanced.** **Replicas are not backups.** A `DELETE FROM orders` replicates to every replica in
milliseconds. Replicas protect against hardware failure, not against mistakes or malice. The
mitigations: a **delayed replica** (`recovery_min_apply_delay = '1h'`) that lags deliberately and
gives you an hour to notice; backups stored in a separate account with write-once retention, so
compromised credentials cannot delete them; and periodic restore drills. The delayed replica is a
particularly good answer because few candidates mention it.

#### 4. Testing and securing backups

**Theory.** A backup is only proven by a restore. Automate a periodic restore into a scratch
environment and verify the result — row counts, checksums, a smoke test of the application against
the restored data.

**Example.** A monthly automated drill: restore last night's backup to a temporary instance, run
`SELECT count(*)` on the ten largest tables and compare with recorded values, run the application's
migration check, record the wall-clock restore time, then destroy the instance. The recorded time is
your real RTO — and it usually surprises people the first time.

**Advanced.** Security matters as much as correctness: backups contain all your personal data, so
they must be encrypted at rest, access-controlled separately from production, and covered by your
retention policy. They also collide with the right to erasure (`S12`) — you cannot practically delete
one user from a backup, so the accepted position is that backups have a bounded retention window,
deletion is applied when a backup is restored, and this is documented in your privacy policy.

### Interview questions

- "A bad migration deleted three million rows 40 minutes ago. Walk me through recovery."
- "Why are replicas not backups?"
- "What is your real RTO and how would you find out?"
- "How do you satisfy a deletion request when the data is in six months of backups?"

---

## DB37 · Database security and privacy

`Intermediate` · Requires: `DB05`, `S07` · Unlocks: `S12`

### Preface

The database holds everything an attacker wants, so it deserves defences of its own rather than
relying on the application being perfect.

Two ideas do most of the work: **least privilege** (the application's account can do only what the
application needs) and **defence in depth** (a bug in one layer should not be enough on its own).

### Details

#### 1. Least privilege roles

**Theory.** Applications commonly connect as the owner of every table, which means a SQL injection or
a bug can drop tables, read every row, or alter the schema. Separate the roles: a migration role that
owns the schema, an application role with only `SELECT`, `INSERT`, `UPDATE`, `DELETE` on the tables
it needs, and read-only roles for analysts and support tools.

**Example.**

```sql
CREATE ROLE app_rw LOGIN PASSWORD '...';
GRANT USAGE ON SCHEMA public TO app_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_rw;
-- migrations run as a different role that owns the objects
```

The `ALTER DEFAULT PRIVILEGES` line is the one people forget — without it, tables created by future
migrations are not accessible to the application role and you get a permission error after the next
deploy.

**Advanced.** Set per-role limits too: `ALTER ROLE app_rw SET statement_timeout = '30s'` and a longer
one for the reporting role. This puts the safety limits in the database rather than relying on every
client configuring them. Combined with `CONNECTION LIMIT` per role, it prevents one misbehaving
consumer from exhausting the server.

#### 2. Row-level security

**Theory.** Postgres RLS attaches a policy to a table so that every query is automatically filtered,
regardless of what the application wrote. It is the strongest available defence against cross-tenant
data leaks (`M29`).

**Example.**

```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

The application sets `SET LOCAL app.tenant_id = '...'` at the start of each transaction. A query that
forgets its `WHERE tenant_id` now returns nothing instead of everyone's data.

**Advanced.** Three caveats to name. RLS is bypassed by the table owner and by superusers unless you
use `FORCE ROW LEVEL SECURITY` — so the application must not connect as the owner. It interacts badly
with transaction-mode connection pooling if you use `SET` rather than `SET LOCAL`, because session
state can leak to the next transaction on that backend (`DB21`). And policies add planning and
execution overhead, which is usually small but should be measured on hot paths.

#### 3. Encryption

**Theory.** **At rest**: full-disk or tablespace encryption protects against stolen disks and
mishandled backups. **In transit**: TLS between application and database, which is not on by default
in every setup. **Column-level or application-level**: individual fields encrypted before storage, so
even a database compromise does not reveal them.

**Example.** Be precise about what each protects against. Disk encryption protects against someone
taking the physical disk or a backup file — it does **not** protect against SQL injection, a
compromised application, or a curious database administrator, because to the database the data is
plaintext. Only application-level encryption protects against those, and it costs you the ability to
index, search or sort the encrypted column.

**Advanced.** Where you need both searchability and protection, the usual designs are: a
deterministic encryption or a keyed hash (HMAC) of the value stored in a separate indexed column for
exact-match lookup, accepting that equal values are visible as equal; or **tokenisation**, where the
sensitive value is kept in a small separate vault and the main database holds only a token. For card
data this is also the standard way to reduce PCI scope. Envelope encryption with a KMS-held key
encrypting per-record data keys is the usual key-management shape (`S11`).

#### 4. Auditing and safe logging

**Theory.** You need a record of who read or changed sensitive data, and your logs must not become a
copy of that data.

**Example.** Query logs are a real leak: `log_statement = 'all'` writes parameter values into a text
file that is shipped to a log aggregator with much weaker access controls than the database. Log
statements without parameters, or redact them, and treat database logs as sensitive. For auditing,
`pgaudit` gives structured, controllable audit logging; application-level audit tables give business
meaning ("user X exported the customer list").

**Advanced.** The most valuable audit records are the ones covering bulk access: a support tool that
reads one customer is normal, and one that reads fifty thousand is an incident. Alerting on
**volume** of access rather than merely recording access is what detects insider misuse and stolen
credentials (`S16`).

### Interview questions

- "Your app connects as the table owner. Why is that bad and what is the alternative?"
- "Encryption at rest protects you from what, exactly?"
- "How do you make cross-tenant leaks impossible at the database level?"
- "How do you search an encrypted column?"

---

## DB38 · Money, ledgers and correctness

`Advanced` · Requires: `DB04`, `DB10` · Unlocks: `SD10`

### Preface

Money has properties ordinary data does not: it must never be created or destroyed by a bug, every
change must be explainable months later, and the same instruction arriving twice must not move it
twice.

The pattern that satisfies all three is the **double-entry ledger**, which accountants settled on
centuries ago and which maps remarkably well onto an append-only table.

### Details

#### 1. Never mutate a balance

**Theory.** `UPDATE accounts SET balance = balance - 100` destroys information: you cannot tell why
the balance is what it is, you cannot audit it, and a duplicated request silently doubles the
deduction. Instead, **append an immutable entry** and derive the balance.

**Example.** A double-entry ledger:

```sql
CREATE TABLE ledger_entries (
  id            bigserial PRIMARY KEY,
  transfer_id   uuid    NOT NULL,
  account_id    uuid    NOT NULL,
  amount_minor  bigint  NOT NULL,          -- signed; negative = debit
  currency      char(3) NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now()
);
```

Every transfer writes **two** rows in one transaction: −100 on one account, +100 on the other. The
invariant is that the sum of `amount_minor` for any `transfer_id` is exactly zero — which you can
assert in a test and check continuously in production.

**Advanced.** That invariant is the whole point: it makes "money was created or destroyed"
detectable by a single query over the entire system. Add a database constraint where you can (a
deferred constraint trigger checking the sum per transfer), and run a continuous job asserting that
`SELECT sum(amount_minor) FROM ledger_entries GROUP BY currency` equals the expected total across all
accounts. An alert on that query is the single most valuable monitor a payments system has.

#### 2. Exact arithmetic

**Theory.** Use integer minor units (cents, pence) or `numeric` — never floating point (`DB04`).
Store the currency alongside every amount; an amount without a currency is meaningless and invites
mixing.

**Example.** Rounding must have a defined, tested policy: splitting £10 three ways gives 3.33, 3.33,
3.34 — the remainder must go somewhere explicit, not be lost. Tax calculations often have a legally
specified rounding mode. Centralise this in one module with tests, rather than letting each call site
decide.

**Advanced.** Multi-currency adds that an exchange rate is a **fact at a point in time** and must be
recorded with the transaction, not looked up later. A ledger entry for a converted amount should
record the original amount, the rate used and the converted amount, so the calculation can be
reproduced. Storing only the result makes disputes unresolvable.

#### 3. Idempotency and duplicate protection

**Theory.** Payment instructions arrive twice — retries, webhook redelivery, a user double-clicking.
The ledger must be structurally unable to apply the same instruction twice.

**Example.** Put a unique constraint on the natural key of the operation:

```sql
ALTER TABLE ledger_entries ADD CONSTRAINT uniq_transfer_account
  UNIQUE (transfer_id, account_id);
```

A repeated attempt fails with a unique violation, which you catch and treat as success (the work was
already done). Combined with an idempotency key from the client (`A09`), the same request can be
retried safely any number of times.

**Advanced.** The external side is harder: you must also not charge the card twice. That requires the
provider's idempotency key, and — crucially — a defined procedure for the unknown outcome: when a
charge request times out, **query the provider for the status of that key** rather than retrying
blindly (`M08`). Then a reconciliation job compares your ledger with the provider's daily settlement
file and raises anything that differs. Mentioning reconciliation unprompted is a strong signal.

#### 4. Making balances fast

**Theory.** Summing millions of entries per balance read eventually becomes too slow. The fix is a
periodic snapshot, not abandoning the ledger.

**Example.** Keep an `account_balances` table with `(account_id, balance, as_of_entry_id)`. The
current balance is that snapshot plus the sum of entries after `as_of_entry_id` — a small, indexed
range. A background job advances the snapshot. A separate job recomputes from zero periodically and
alerts if it disagrees with the snapshot, which catches any bug in the incremental path.

**Advanced.** Keep the snapshot **derived and rebuildable**: the entries remain the source of truth,
and the snapshot is a cache that can always be reconstructed. This is the same relationship as a
CQRS projection to its event log (`M19`). The temptation to make the snapshot authoritative "because
it is faster" is exactly the mistake that reintroduces unexplainable balances.

### Interview questions

- "Design the schema for a wallet with transfers. Prove no money can be created or destroyed."
- "Balance is now a slow `SUM()` over 50 million rows. What do you do?"
- "The charge request timed out. What do you do next?"
- "How do you guarantee the same payment instruction is not applied twice?"

---

## DB39 · Database observability and production debugging

`Intermediate` · Requires: `DB08`, `DB19`, `DB20`, `DB21` · Unlocks: `O14`, `O17`

### Preface

When the database is the problem, four questions answer nearly every incident: **what is running,
what is blocked, what is bloated, what is lagging.**

Have the queries ready. Being able to name them in an interview signals that you have been the person
paged at 2am.

### Details

#### 1. What is running

**Theory.** `pg_stat_activity` shows every connection, its state, its current query, how long it has
been running, and what it is waiting on.

**Example.**

```sql
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS duration, left(query, 80)
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

Look for: queries running far longer than expected; many connections in `idle in transaction`
(`DB06`); a high count of the same query (a retry storm or an N+1); and `wait_event_type = 'Lock'`,
which points you at `pg_locks`. `pg_cancel_backend(pid)` cancels a query;
`pg_terminate_backend(pid)` kills the connection.

**Advanced.** `wait_event` is the most underused diagnostic in Postgres. It tells you *why* a query
is slow rather than that it is: `IO/DataFileRead` means it is reading from disk (cache too small, or
a bad plan reading too much), `Lock/transactionid` means it is waiting for another transaction,
`LWLock/WALWriteLock` points at WAL write contention. Sampling `pg_stat_activity` every second and
aggregating wait events gives you a rough profile of where database time goes — this is what managed
"performance insights" products do.

#### 2. What is slow overall

**Theory.** The slow query log shows individual slow queries. `pg_stat_statements` shows **aggregate**
time per normalised query, which is more useful: a query taking 20ms but running a million times an
hour costs far more than one taking 5 seconds twice a day, and the slow log never shows it.

**Example.**

```sql
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time) AS mean_ms,
       rows, left(query, 80)
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
```

Sort by `total_exec_time` for where time goes overall, by `mean_exec_time` for the worst individual
offenders. Enable `pg_stat_statements` on every production database; it costs very little.

**Advanced.** `shared_blks_hit` versus `shared_blks_read` per statement tells you which queries are
actually going to disk. And a query whose `rows / calls` is very large is returning far more data
than a user could need — usually a missing `LIMIT` or an application-side filter that should be in
SQL. Those two ratios find a surprising number of problems quickly.

#### 3. What is blocked

**Theory.** Lock waits are usually visible as a chain: one query holds a lock, several wait behind it
(`DB09`).

**Example.** The blocking tree, using the built-in helper:

```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query, 60)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Follow the chain to the root — it is often a single long transaction or a stalled DDL — and decide
whether to cancel it. Enabling `log_lock_waits` records waits exceeding `deadlock_timeout`, so you
find them after the fact too.

**Advanced.** During an incident, the fastest reliable action is usually to terminate the *root*
blocker rather than the visible victims, then fix the cause. Have the query ready, and know that you
can filter by `application_name` (set it from your application!) to distinguish the reporting job
from the API. Setting `application_name` per service is a two-line change that makes every incident
faster.

#### 4. What is bloated or lagging

**Theory.** Bloat (`DB08`) and replication lag (`DB23`) both degrade silently — nothing errors, things
just get slower or return stale data.

**Example.**

```sql
-- dead tuples and vacuum activity
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- replication lag in bytes
SELECT client_addr, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

**Advanced.** The dashboard worth building for any production Postgres, and a good answer to "what do
you monitor": connections used versus `max_connections`; transactions per second; the slowest
statements by total time; cache hit ratio (treat it as weak — a high ratio with a bad plan is still
slow); dead tuple counts on the largest tables; replication lag in bytes and seconds; oldest
transaction age; replication slot lag; and disk space with a projection of when it runs out. That
list, offered unprompted, reads as operational experience.

### Interview questions

- "CPU is 100% on the primary at 2am. Walk through your first five minutes."
- "Slow query log or `pg_stat_statements`? Why?"
- "How do you find what is blocking a query?"
- "What would you put on a database dashboard?"

---

## DB40 · ORM internals and escape hatches

`Advanced` · Requires: `DB20`, `DB21`, `F09` · Unlocks: `F09`, `F21`, `F22`

### Preface

An ORM maps rows to objects so you do not write SQL for every operation. It saves a great deal of
time and hides the thing you most need to see: **which queries actually run**.

Senior expectation: use the ORM for the 90% that is routine, know exactly what SQL it generates,
and drop to raw SQL without guilt for the 10% that matters.

### Details

#### 1. The core patterns

**Theory.** Three ideas underlie most ORMs. **Identity map** — one object per row per session, so the
same row loaded twice gives the same object. **Unit of work** — changes are tracked and flushed
together at the end. **Lazy loading** — related data is fetched on first access.

**Example.** Hibernate implements all three fully: the persistence context is the identity map,
`flush()` writes pending changes in dependency order, and an unfetched association is a proxy that
triggers a query on access. Django's ORM has lazy querysets and no identity map by default. Prisma
deliberately has **no** lazy loading — you declare what you want with `include`/`select`, which makes
N+1 harder to create by accident and is a genuine design advantage.

**Advanced.** Flush ordering matters more than people expect. Hibernate orders statements by entity
type, not by the order you wrote them, which can violate a foreign key or a unique constraint in a
way that looks inexplicable — the classic case is deleting a row and inserting a replacement with
the same unique value in the same transaction, where the insert is flushed first. The fix is an
explicit `flush()` between the operations, and knowing to look for it is the valuable part.

#### 2. N+1, the defining ORM problem

**Theory.** Lazy loading plus a loop equals one query per iteration (`DB20`). It is invisible in the
code and obvious in the query log.

**Example.** How each stack expresses the fix:
- **TypeORM**: `relations: ['customer']`, or `qb.leftJoinAndSelect('order.customer', 'customer')`.
- **Prisma**: `include: { customer: true }` — issues a second query with an `IN` list, not a join.
- **Django**: `select_related('customer')` (join, for foreign keys) versus
  `prefetch_related('items')` (second query, for reverse and many-to-many).
- **Hibernate**: `JOIN FETCH` in JPQL, or `@EntityGraph`; `@BatchSize` to load associations in
  batches of N rather than one at a time.

**Advanced.** The **cartesian product** trap when fetching two collections at once: joining orders to
both `items` and `payments` multiplies rows (10 items x 3 payments = 30 rows per order), and
Hibernate's `MultipleBagFetchException` exists to stop exactly this. The correct approach is one
join fetch plus a batched second query. Knowing why two collection joins are wrong — and that the
fix is not a bigger join — is a strong signal.

#### 3. Where ORMs mislead you

**Theory.** Common gaps: a `count()` that loads everything into memory first; `save()` on a large
collection issuing one statement per row; updates that rewrite every column rather than the changed
ones; pagination implemented with offset (`DB20`); and migrations generated by diffing the schema,
which produce *correct* SQL that is not *safe* SQL.

**Example.** Django's `makemigrations` will happily generate `ALTER TABLE ... ALTER COLUMN TYPE`,
which rewrites a 200-million-row table under an exclusive lock (`DB22`). The generated migration is
right and the deployment is an outage. **Always read generated migrations before running them in
production** — that single habit prevents most migration incidents.

**Advanced.** Also watch transaction scope: `@Transactional` in Spring is proxy-based, so calling a
method from within the same class bypasses it entirely (`F10`, `F22`); Django's `ATOMIC_REQUESTS`
wraps an entire request in a transaction, which is convenient and holds a connection for the whole
request including slow external calls (`DB06`). Knowing where your framework opens and closes
transactions is not optional.

#### 4. Dropping to SQL safely

**Theory.** Complex reporting, window functions, recursive queries, bulk operations and anything
performance-critical are usually clearer and faster as hand-written SQL. The requirements are that
it must be parameterised and it should be typed.

**Example.**

```ts
// safe: parameterised
await dataSource.query('SELECT * FROM orders WHERE tenant_id = $1 AND total > $2', [tenantId, min]);

// unsafe: string interpolation — SQL injection (S07)
await dataSource.query(`SELECT * FROM orders WHERE tenant_id = '${tenantId}'`);
```

Keep raw SQL in named repository methods rather than scattered through services, so it is reviewable
and testable in one place. Query builders such as Kysely or jOOQ give you SQL with compile-time type
checking, which is often the best of both.

**Advanced.** The senior position to articulate: the ORM is for **writes and object graphs**, where
identity, change tracking and transaction management genuinely help; hand-written SQL is for
**reads**, especially reporting and list endpoints, where you want exact control over joins,
columns and indexes. That split maps neatly onto CQRS (`M19`) and resolves most "ORM versus SQL"
debates without dogma.

### Interview questions

- "Show me an N+1 your ORM generated and three ways to fix it."
- "What does the ORM do at flush time and why does ordering matter?"
- "Your ORM generated a migration. Would you run it? What do you check?"
- "When do you drop to raw SQL, and how do you keep it safe?"
