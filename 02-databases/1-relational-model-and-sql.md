[← back to the field index](README.md)

# Databases · Part 1 — The Relational Model & SQL

Nodes `DB01`–`DB05`.

---

## DB01 · The relational model, keys and normalisation

`Beginner` · Requires: — · Unlocks: `DB02`, `DB05`, `DB11`, `DB26`

### Preface

A relational database stores data in tables of rows and columns, where each row is uniquely
identified by a key and relationships between tables are expressed by storing one table's key
inside another.

Normalisation is the discipline of storing each fact exactly once. The reason is not tidiness: if a
fact is stored in three places, the three copies will eventually disagree, and no code review
catches that.

Denormalisation is deliberately breaking that rule to make reads faster — a trade you make with
your eyes open, never by accident.

### Details

#### 1. Keys: natural, surrogate, composite

**Theory.** A **candidate key** is any column set that uniquely identifies a row. The **primary
key** is the one you choose. A **natural key** comes from the data itself (an ISBN, an email
address). A **surrogate key** is meaningless and generated (an auto-increment integer, a UUID). A
**foreign key** is a column holding another table's primary key.

**Example.** Using email as the primary key for users looks natural and causes years of pain: people
change email addresses, so you must cascade the change through every referencing table; emails are
large, so every index holding them is large; and case sensitivity and normalisation rules become
correctness problems. Use a surrogate primary key and put a **unique index** on the email — you get
uniqueness enforcement without the key problems.

**Advanced.** Composite primary keys are right in specific cases: a junction table
(`order_id, product_id`), or a multi-tenant table where `(tenant_id, id)` gives you tenant locality
in a clustered index and makes cross-tenant leaks structurally harder. The cost is that every
foreign key referencing it must carry both columns, which propagates width through your schema.

#### 2. Choosing the primary key type

**Theory.** The choice between `bigint`, UUIDv4 and UUIDv7/ULID affects index size, insert
performance, replication volume and how easy your data is to enumerate from outside.

**Example.** Concrete differences:
- `bigint` (8 bytes, sequential): smallest indexes, perfect insert locality, but ids are guessable
  and reveal volume ("they only have 4,012 customers"), and they must be generated centrally.
- **UUIDv4** (16 bytes, random): generated anywhere, unguessable, and **random insert position**,
  which is the problem — every insert lands on a different index page, so the cache holds the whole
  index rather than the hot end, and pages split constantly.
- **UUIDv7 / ULID** (16 bytes, time-ordered prefix): unguessable enough, generated anywhere, and
  sequential insert locality. This is the modern default when you want UUIDs.

**Advanced.** The cost of UUIDv4 compounds in unexpected places: in InnoDB, the primary key is the
physical row order, so random keys cause page splits and fragmentation on every insert; and because
**every secondary index stores the primary key**, a 16-byte key inflates all of them. In Postgres
the effect is smaller (the heap is not clustered by the key) but the index locality and WAL volume
costs remain. If you have a table with a UUIDv4 primary key and heavy inserts, that is a real and
measurable problem, and switching to v7 is a genuine optimisation.

#### 3. Normal forms, in plain language

**Theory.** The memorable summary: *every non-key attribute depends on the key, the whole key, and
nothing but the key.*
- **1NF** — each column holds a single value, no repeating groups (no `tags` as a comma-separated
  string).
- **2NF** — no column depends on only part of a composite key.
- **3NF** — no column depends on another non-key column (no `city` and `country` derived from
  `postcode` sitting in the same table as everything else).
- **BCNF** — a stricter version of 3NF for tables with several overlapping candidate keys.

**Example.** An order table containing `customer_name` violates 3NF: the name depends on the
customer, not the order. When the customer renames, the old orders show the old name. Sometimes
that is a bug; sometimes it is exactly what you want (an invoice must show the name as it was).
Deciding which is the interesting part, and it is a domain question, not a normalisation question.

**Advanced.** 3NF is enough for almost all operational databases; BCNF and higher rarely change a
real design. The more useful distinction in practice is between *derived* data (a total you can
always recompute) and *recorded* data (the total as agreed at the time). Recording the agreed value
is not denormalisation — it is storing a different fact.

#### 4. Deliberate denormalisation

**Theory.** Storing a value in more than one place to avoid a join or an aggregate at read time. It
costs you the obligation to keep copies in sync, which needs a defined mechanism.

**Example.** A `comment_count` column on `posts` instead of `COUNT(*)` on comments. Reads become
instant; writes must now maintain the counter. The mechanisms, in increasing robustness: application
code (breaks the moment anything else writes comments), a database trigger (always runs, adds write
contention and hidden behaviour), or a periodic recount that repairs drift. In practice you use a
trigger or careful application code **plus** a periodic reconciliation, because the counter will
drift.

**Advanced.** Denormalise only after measuring, and write down how the copy is maintained and how
drift is detected. The most common failure is not the initial implementation — it is the second
write path added a year later by someone who did not know the counter existed. A reconciliation job
that logs discrepancies turns a silent corruption into a visible alert.

### Interview questions

- "Normalise this schema; now tell me where you would denormalise it and why."
- "UUID or auto-increment primary key? Defend it."
- "Why is a random UUID a problem for an index?"
- "Your order shows the customer's current name instead of the name at the time of purchase. Is
  that a normalisation bug?"

---

## DB02 · SQL fundamentals

`Beginner` · Requires: `DB01` · Unlocks: `DB03`, `DB04`, `DB07`, `DB19`

### Preface

SQL describes *what* you want, and the database decides how to get it. That gap is where both its
power and its surprises live.

The single most useful thing to know is the **logical order of evaluation**, which is not the order
you write the clauses in. It explains most beginner confusion and a fair amount of senior debugging.

### Details

#### 1. Logical query processing order

**Theory.** The database evaluates clauses in this order: `FROM` → `JOIN` → `WHERE` → `GROUP BY` →
`HAVING` → `SELECT` → `DISTINCT` → `ORDER BY` → `LIMIT`.

**Example.** This explains several everyday rules. You cannot use a `SELECT` alias in `WHERE`,
because `WHERE` runs first — but you *can* use it in `ORDER BY`, which runs after. `WHERE` filters
rows before grouping; `HAVING` filters groups after. And `LIMIT` runs last, which is why
`LIMIT 10` on a query with an expensive sort still has to sort everything first.

**Advanced.** This is the *logical* order, defining the meaning. The optimiser is free to execute
differently as long as the result is identical — pushing a filter below a join, evaluating `LIMIT`
early when an index provides the order, or rewriting a subquery as a join. Knowing both — meaning
from logical order, performance from the actual plan — is what lets you read an `EXPLAIN` (`DB19`).

#### 2. Joins and anti-joins

**Theory.** `INNER` keeps matching rows only. `LEFT` keeps all rows from the left with NULLs where
there is no match. `FULL` keeps both sides. `CROSS` is every combination. An **anti-join** finds
rows with *no* match.

**Example.** The three ways to write "customers with no orders", and their differences:

```sql
-- NOT EXISTS: correct, handles NULLs, usually the best plan
SELECT * FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- LEFT JOIN ... IS NULL: correct, sometimes a better plan for large sets
SELECT c.* FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;

-- NOT IN: a trap
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

**Advanced.** The `NOT IN` trap is worth memorising: if the subquery returns even one NULL, the
comparison evaluates to UNKNOWN for every row and the query returns **nothing** — silently, with no
error. This follows from three-valued logic (`DB04`): `id NOT IN (1, 2, NULL)` means
`id <> 1 AND id <> 2 AND id <> NULL`, and the last is UNKNOWN. Default to `NOT EXISTS`, which has no
such hazard.

#### 3. Aggregation

**Theory.** `GROUP BY` collapses rows into groups; aggregate functions summarise each group.
`COUNT(*)` counts rows; `COUNT(column)` skips NULLs — a difference that silently changes results.
`HAVING` filters groups after aggregation.

**Example.** `SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id HAVING COUNT(*) > 5`.
Note that filters belong in `WHERE` when they apply to rows (`WHERE status = 'paid'`) and in
`HAVING` only when they apply to the aggregate. Putting a row filter in `HAVING` works but makes the
database aggregate rows it will then discard.

**Advanced.** Postgres allows `SELECT` columns that are functionally dependent on the grouped
primary key (`GROUP BY o.id` lets you select `o.total`), while other databases require every
selected column to be grouped or aggregated. MySQL historically allowed *any* column and returned
an arbitrary value — a source of silently wrong results before `ONLY_FULL_GROUP_BY` became the
default. `FILTER (WHERE ...)` on an aggregate is the clean way to compute several conditional
counts in one pass instead of several subqueries.

#### 4. Subqueries

**Theory.** A **scalar** subquery returns one value. An **IN/EXISTS** subquery tests membership. A
**correlated** subquery references the outer query and is conceptually evaluated per outer row. A
**derived table** is a subquery in `FROM`.

**Example.** A correlated subquery in the `SELECT` list — "each customer's most recent order date" —
reads naturally and can be slow, because it may execute once per row. A window function or a
`LATERAL` join usually expresses the same thing in one pass (`DB03`).

**Advanced.** Modern optimisers often "decorrelate" a correlated subquery into a join automatically,
so the naive version is not always slow — check the plan rather than assuming. The reliable rule is
that readable SQL is worth writing first, and rewriting is a response to a measured plan, not a
reflex.

### Interview questions

- "`NOT IN (subquery)` returns nothing. Why?"
- "Write a query for the top 3 orders per customer."
- "Why can you use a `SELECT` alias in `ORDER BY` but not in `WHERE`?"
- "`COUNT(*)` versus `COUNT(column)` — when does it matter?"

---

## DB03 · Advanced SQL

`Intermediate` · Requires: `DB02` · Unlocks: `DB20`, `DB32`

### Preface

The features here replace loops in application code with a single query. Two are worth real
investment: **window functions** (calculations across related rows without collapsing them) and
**CTEs** (naming a subquery so a complex query stays readable).

If you can write a running total and a deduplication query from memory, you are ahead of most
candidates.

### Details

#### 1. Window functions

**Theory.** A window function computes a value over a set of rows related to the current row, while
keeping every row. `OVER (PARTITION BY ... ORDER BY ...)` defines the window.

**Example.** Deduplication — keeping the newest row per key — is the canonical use and a common
interview task:

```sql
DELETE FROM events e USING (
  SELECT id, ROW_NUMBER() OVER (PARTITION BY external_id ORDER BY created_at DESC) AS rn
  FROM events
) d
WHERE e.id = d.id AND d.rn > 1;
```

Others: `RANK()` (ties share a rank, then it skips), `DENSE_RANK()` (ties share, no skip),
`LAG`/`LEAD` for comparing to the previous or next row, and
`SUM(x) OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)` for a 7-row moving average.

**Advanced.** `ROWS` and `RANGE` frames differ in a way that bites: `ROWS` counts physical rows,
`RANGE` includes all rows with the same `ORDER BY` value (**peers**). A "7-day moving average" using
`ROWS 6 PRECEDING` is wrong if a day has multiple rows or if days are missing — you need
`RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW`, or a generated date series joined to
the data first. Getting this right in an interview is a strong signal.

#### 2. CTEs and recursive CTEs

**Theory.** `WITH name AS (...)` names a subquery so you can compose a complex query in readable
steps. A `WITH RECURSIVE` CTE references itself, which lets you walk trees and graphs.

**Example.** Walking an organisation hierarchy:

```sql
WITH RECURSIVE tree AS (
  SELECT id, manager_id, name, 1 AS depth FROM employees WHERE id = $1
  UNION ALL
  SELECT e.id, e.manager_id, e.name, t.depth + 1
  FROM employees e JOIN tree t ON e.manager_id = t.id
)
SELECT * FROM tree;
```

Also used for gap-filling (generating a date series and left-joining data so missing days appear as
zero) and for graph traversal.

**Advanced.** Two things to know. **Cycle protection**: a recursive CTE over data with a cycle runs
forever — guard it with a depth limit or an array of visited ids (Postgres 14+ has a `CYCLE`
clause). And the **optimisation fence**: in Postgres before version 12, a CTE was *always*
materialised, which was sometimes a useful optimisation hint and often a performance trap because
filters could not be pushed into it. From 12 onwards, CTEs are inlined when it is safe, and you can
force either behaviour with `MATERIALIZED` or `NOT MATERIALIZED`. Knowing this version boundary is a
genuine senior-level detail.

#### 3. LATERAL joins

**Theory.** A `LATERAL` subquery can reference columns from earlier items in the `FROM` clause —
effectively a for-each loop that stays inside SQL. It is the natural way to express "the top N per
group".

**Example.**

```sql
SELECT c.id, o.*
FROM customers c
LEFT JOIN LATERAL (
  SELECT * FROM orders o WHERE o.customer_id = c.id ORDER BY created_at DESC LIMIT 3
) o ON true;
```

Compared with the window-function version, `LATERAL` can be much faster when there is an index on
`(customer_id, created_at)`, because it fetches only 3 rows per customer instead of ranking all of
them.

**Advanced.** This is the "N+1 done properly" pattern — one query, an index-driven loop inside the
database, rather than N round trips from the application. When someone asks how to avoid N+1 without
loading everything, `LATERAL` (or `CROSS APPLY` in SQL Server) is the precise answer.

#### 4. Upserts and RETURNING

**Theory.** `INSERT ... ON CONFLICT` (Postgres) or `MERGE` (standard, MySQL's
`ON DUPLICATE KEY UPDATE`) inserts or updates atomically. `RETURNING` gives you back the affected
rows without a second query.

**Example.**

```sql
INSERT INTO counters (key, value) VALUES ($1, 1)
ON CONFLICT (key) DO UPDATE SET value = counters.value + 1
RETURNING value;
```

One statement, atomic, no race between checking and inserting, and you get the new value back. This
is the correct way to implement "create if missing" — the read-then-write version has a race window
that *will* be hit under concurrency.

**Advanced.** `ON CONFLICT` needs a unique index or constraint to work against. Two subtleties:
`DO NOTHING` returns no rows, so `RETURNING` gives you nothing when the row already existed, which
surprises people implementing idempotent inserts — add `ON CONFLICT (key) DO UPDATE SET key =
EXCLUDED.key` to force a returned row. And a failed upsert still consumes a sequence value, so
auto-increment ids develop gaps under contention. That is normal and not worth fixing.

### Interview questions

- "Deduplicate a table keeping the newest row per key."
- "Compute a 7-day rolling average per user in one query."
- "Was a CTE an optimisation fence in Postgres?"
- "Top 3 orders per customer — window function or LATERAL? Which is faster and why?"

---

## DB04 · Types, NULL, time and text

`Intermediate` · Requires: `DB02` · Unlocks: `DB13`, `DB38`

### Preface

Most data corruption bugs come from three places: NULL behaving unlike anything in your programming
language, floating-point numbers used for money, and timestamps stored without a timezone.

Each has a simple rule. Learn the rule and you avoid an entire class of production incidents.

### Details

#### 1. NULL and three-valued logic

**Theory.** SQL has three truth values: TRUE, FALSE and **UNKNOWN**. Any comparison with NULL yields
UNKNOWN, and `WHERE` only keeps rows where the condition is TRUE. So `NULL = NULL` is not true, and
neither is `NULL <> 'x'`.

**Example.** `SELECT * FROM users WHERE status <> 'active'` does **not** return users whose status
is NULL, which is almost never what the author intended. You need
`WHERE status IS DISTINCT FROM 'active'` (Postgres) or `WHERE status <> 'active' OR status IS NULL`.
Similarly, `UNIQUE` constraints allow multiple NULLs in standard SQL, because two NULLs are not
"equal" — so a nullable unique column does not prevent many rows with no value.

**Advanced.** NULL means "unknown", not "empty" and not "zero". Three practical consequences:
aggregates skip NULLs (`AVG` of `[1, NULL, 3]` is 2, not 1.33), `COUNT(col)` differs from
`COUNT(*)`, and NULL sorts either first or last depending on the database and on `NULLS FIRST/LAST`.
The engineering advice is to prefer `NOT NULL` with a sensible default wherever the domain allows,
and reserve NULL for genuinely unknown values — it removes whole categories of bug.

#### 2. Numbers and money

**Theory.** `float`/`double` are binary floating point: they cannot represent 0.1 exactly, so sums
drift. `numeric`/`decimal` is exact decimal arithmetic, slower but correct. Integers of minor units
(cents) are exact and fast.

**Example.** In any language with IEEE floats, `0.1 + 0.2 = 0.30000000000000004`. Summing a million
float prices produces an answer that disagrees with the accountants, and no test catches it because
each individual value looks right. Use `numeric(12,2)` in Postgres, `DECIMAL` in MySQL, or store
integer cents.

**Advanced.** Storing integer minor units is fast and exact but needs care for currencies with
different subunits (Japanese yen has none, Kuwaiti dinar has three), for division (splitting £10
three ways requires an explicit rounding policy so the pennies are not lost), and for tax
calculations where the rounding rule is specified by law. Whichever you pick, define the rounding
rule once, centrally, and test it. See `DB38`.

#### 3. Time

**Theory.** `timestamptz` in Postgres stores an absolute point in time (internally UTC) and converts
to the session timezone on input and output. `timestamp` (without time zone) stores a wall-clock
reading with no timezone information — it is ambiguous. Use `timestamptz` for events; the exception
is genuinely local times such as "shop opens at 09:00".

**Example.** A recurring meeting at 9am every Monday in London. If you store the absolute UTC
instant, the meeting shifts to 10am when daylight saving changes. You must store the local time
**plus the IANA timezone name** (`Europe/London`) and compute the instant for each occurrence. And
you must store the zone name, not the offset: `+01:00` is a fact about one moment, while
`Europe/London` is a rule that survives DST changes and legislative changes.

**Advanced.** Other traps worth naming: daylight saving means some local times do not exist (the
clock jumps 01:00 → 02:00) and some occur twice — so "every day at 01:30" needs a defined policy;
date arithmetic must distinguish "24 hours later" from "tomorrow at the same local time";
and clients send timestamps you should validate rather than trust (see `M21`). Store UTC, compute in
UTC, convert for display only.

#### 4. Text, collation and encoding

**Theory.** Collation determines sorting and comparison rules for text, including case and accent
handling. It is set per database, per column or per query. In Postgres, `text` and `varchar(n)` have
identical performance — the length limit is a constraint, not an optimisation.

**Example.** The dangerous one: an operating system upgrade changes the C library's collation rules
(glibc 2.28 changed them), which silently makes existing text indexes **invalid** — the index is
sorted by the old rules and the database now compares with the new ones. Symptoms are queries
missing rows that exist. The fix is reindexing every text index after such an upgrade. Using
`collation = "C"` or ICU collations avoids surprises, at the cost of less natural sorting.

**Advanced.** Case-insensitive comparison is best done with an explicit approach — a `citext` column,
an expression index on `lower(email)`, or an ICU collation with the right rules — rather than
scattering `LOWER()` through queries, because a function on a column prevents index use unless there
is a matching expression index (`DB17`). Also note MySQL's `utf8` is a three-byte subset that cannot
store emoji; the real encoding is `utf8mb4`. That trips people up in production.

### Interview questions

- "Where do you store the timezone for a recurring 9am meeting?"
- "Why is `float` wrong for money and what do you use instead?"
- "`WHERE status <> 'active'` misses rows. Why?"
- "An OS upgrade broke your text index. Explain."

---

## DB05 · Constraints and data integrity

`Beginner` · Requires: `DB01` · Unlocks: `DB09`, `DB22`

### Preface

A constraint is a rule the database enforces on every write, no matter which application, script or
person makes it.

This matters because application-level checks cannot be correct under concurrency. "Check whether
the email exists, then insert" has a gap between the two steps, and two simultaneous requests both
pass the check. Only the database can close that gap.

### Details

#### 1. The constraint types

**Theory.** `PRIMARY KEY` (unique and not null), `UNIQUE`, `FOREIGN KEY`, `NOT NULL`, `CHECK`
(an arbitrary boolean expression), `DEFAULT`, and in Postgres `EXCLUDE` (no two rows may have
overlapping values — the natural way to prevent double-booking a room).

**Example.** An `EXCLUDE` constraint preventing overlapping bookings:

```sql
ALTER TABLE bookings ADD CONSTRAINT no_overlap
EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

`during` is a `tstzrange`. Two bookings for the same room with overlapping time ranges are now
impossible — enforced by the database, under any concurrency, without application locking. This is
one of the strongest answers to a booking-system question.

**Advanced.** `CHECK` constraints must be immutable — they cannot reference other rows or use
functions like `now()`, because the database must be able to re-validate them at any time.
Cross-row rules ("at most 3 active subscriptions per user") need either a trigger, a partial unique
index where the shape allows, or serialisable isolation (`DB07`).

#### 2. Check-then-insert is always a race

**Theory.** Any "does it exist?" check followed by a write has a window between the two in which
another transaction can act. At READ COMMITTED, both transactions see no row and both insert.

**Example.** The wrong and right versions:

```js
// wrong: two concurrent requests both pass the check
const existing = await repo.findOne({ email });
if (existing) throw new ConflictException();
await repo.save({ email });

// right: let the database decide, handle the violation
try {
  await repo.save({ email });
} catch (e) {
  if (e.code === '23505') throw new ConflictException();  // unique_violation
  throw e;
}
```

The second version needs a unique index on `email` and knowledge of the error code — `23505` in
Postgres, `ER_DUP_ENTRY` (1062) in MySQL.

**Advanced.** This generalises to a principle: **let the database enforce the invariant and handle
the failure**, rather than checking first. It applies to unique constraints, foreign keys and
exclusion constraints alike. The pattern is faster (one round trip), correct under concurrency, and
the error path is the same code you needed anyway for a race you cannot otherwise prevent.

#### 3. Foreign keys: keep or drop?

**Theory.** Foreign keys guarantee that references point at rows that exist. They cost a lookup on
every insert and update of the referencing column, and they take locks on the referenced row, which
can cause contention and deadlocks on hot parent rows.

**Example.** `ON DELETE CASCADE` is convenient and hides cost: deleting one customer can delete
millions of rows in one transaction, holding locks and generating enormous WAL. `ON DELETE
RESTRICT` (the safe default) forces you to handle it explicitly, usually with a soft delete or a
batched cleanup job. Prefer explicit deletion in batches for large hierarchies.

**Advanced.** At very high write volumes, and across services, foreign keys are often dropped — the
first for contention, the second because you cannot have a foreign key into another service's
database (`M22`). What you lose is certainty; what you need instead is validation in the owning
service plus a reconciliation job that finds and reports orphans. Say clearly that dropping a
foreign key is a deliberate trade with a compensating control, not a simplification.

#### 4. Partial unique indexes and soft delete

**Theory.** A partial index applies to a subset of rows. This solves the common conflict between
"emails must be unique" and "we soft-delete users".

**Example.**

```sql
CREATE UNIQUE INDEX users_email_active ON users (email) WHERE deleted_at IS NULL;
```

An address can be reused after the previous account is deleted, and remains unique among active
users. The same shape enforces "one active subscription per user":
`UNIQUE (user_id) WHERE status = 'active'`.

**Advanced.** Soft delete has costs beyond uniqueness that are worth raising: every query must
remember `WHERE deleted_at IS NULL` (a missed filter is a data leak), indexes carry dead rows
forever, and it does not satisfy a legal deletion request. A frequently better pattern is to move
deleted rows to an archive table — queries stay simple, the main table stays small, and real
deletion is possible. Choose deliberately rather than reaching for a `deleted_at` column by habit.

#### 5. Deferrable constraints

**Theory.** A deferrable constraint is checked at the end of the transaction rather than on each
statement, which allows temporarily inconsistent intermediate states.

**Example.** Swapping two rows' unique `position` values. With an immediate constraint, the first
update collides. With `DEFERRABLE INITIALLY DEFERRED`, both updates happen and the check runs at
commit, when the state is valid again. Circular foreign keys between two tables need the same.

**Advanced.** Deferring means errors surface at `COMMIT` rather than at the offending statement,
which makes them harder to attribute — your application must handle failures at commit time, which
many ORMs do poorly. Use it where genuinely needed rather than as a default.

### Interview questions

- "Your app checks uniqueness with a SELECT before INSERT. What is the bug?"
- "Should the database enforce foreign keys in a microservice world?"
- "Enforce 'only one active subscription per user' in the database."
- "Prevent double-booking a meeting room without application-level locking."
