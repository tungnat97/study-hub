[← back to the field index](README.md)

# Databases · Part 5 — NoSQL & Specialised Stores

Nodes `DB26`–`DB35`.

---

## DB26 · NoSQL taxonomy and when to use it

`Intermediate` · Requires: `DB01` · Unlocks: `DB27`, `DB28`, `DB31`, `DB32`

### Preface

"NoSQL" covers several unrelated families of database. What they share is giving up something a
relational database offers — usually joins, ad-hoc queries, or strict consistency — in exchange for
horizontal scale, a flexible schema, or a data shape that relational tables handle awkwardly.

The honest default for a new service is Postgres, until you can name the specific thing it cannot do
for you. That answer scores well because it is what experienced engineers actually do.

### Details

#### 1. The families

**Theory.**
- **Key-value** (Redis, DynamoDB, Memcached) — get and put by key. Fastest, least queryable.
- **Document** (MongoDB, Couchbase, Firestore) — nested JSON documents, queryable by fields inside.
- **Wide-column** (Cassandra, ScyllaDB, HBase) — rows partitioned by key with sorted columns;
  built for enormous write volume.
- **Graph** (Neo4j, Neptune) — nodes and edges, for queries that traverse many relationships.
- **Time-series** (InfluxDB, TimescaleDB, Prometheus) — timestamped measurements, heavy compression,
  automatic downsampling.
- **Search** (Elasticsearch, OpenSearch) — inverted index for text relevance.
- **Vector** (pgvector, Pinecone, Qdrant) — nearest-neighbour search over embeddings.

**Example.** Choosing for an e-commerce system: product catalogue → Postgres (relational, moderate
size, needs transactions); session store → Redis; product search → Elasticsearch, fed from Postgres;
activity feed → Cassandra or DynamoDB at scale; metrics → Prometheus. Several stores, each chosen
for a specific access pattern.

**Advanced.** Every additional store has an operational price: backups, monitoring, upgrades,
security patching, on-call expertise, and a data-synchronisation path with its own failure modes.
"Polyglot persistence" is correct in principle and frequently over-applied. A good rule: a new store
must be justified by an access pattern your existing one genuinely cannot serve — and Postgres serves
JSON, full text, geospatial, time-series (with TimescaleDB) and vectors passably well.

#### 2. Schemaless is a relocation, not a removal

**Theory.** A document store does not enforce a schema, so different documents can have different
fields. The schema still exists — it just lives in your application code, in every function that
reads the data, and it is now unenforced.

**Example.** Three years into a MongoDB collection, `user.address` might be a string in old
documents, an object in newer ones, and absent in some. Every read path needs defensive handling
forever, and there is no migration that makes the old data correct, because there is no constraint
to validate against. This is the most common regret reported by teams who chose a document store for
flexibility.

**Advanced.** The mitigation is to apply a schema deliberately: MongoDB's JSON Schema validation, or
a validation layer in the application (Mongoose, Zod, pydantic) applied to both writes **and** reads,
plus versioned documents with a `schemaVersion` field and migration-on-read. If you are going to do
all that, ask whether a relational database with a `jsonb` column for the genuinely variable part
would be simpler — usually it would.

#### 3. When NoSQL is genuinely right

**Theory.** The cases where the trade clearly pays: a write volume beyond one machine with a natural
partition key; documents that are always read as a whole and never joined; data whose shape varies
legitimately per record; access patterns that are purely key-based; and specialised workloads (text
search, time-series, graph traversal) where the specialised engine is an order of magnitude better.

**Example.** A device telemetry system ingesting a million points per second with a natural partition
key (device id), queried only by device and time range, with no joins and no transactions. Cassandra
or a time-series database is clearly right, and forcing it into Postgres would be the mistake.

**Advanced.** The corresponding warning sign: choosing a store for scale you do not have. A team
adopting Cassandra for 100 requests per second pays the full cost of query-first modelling,
eventual consistency and operational complexity, for scale they will not need for years. Match the
tool to the current problem plus a reasonable horizon, not to the aspiration.

#### 4. What Postgres already does

**Theory.** Modern Postgres covers several of these use cases well enough that a second system is
often unnecessary.

**Example.** `jsonb` with GIN indexes for documents; full-text search with `tsvector` (`DB34`);
PostGIS for geospatial, which is best in class; TimescaleDB or native partitioning for time-series;
`pgvector` for embeddings; `LISTEN`/`NOTIFY` and `SKIP LOCKED` for queues (`Q20`); and logical
replication for change data capture.

**Advanced.** Know where it genuinely runs out: Elasticsearch is materially better for relevance
tuning, analyzers and very large text corpora (`DB31`); ClickHouse is orders of magnitude faster for
analytical scans (`DB32`); Redis is far faster for sub-millisecond key-value operations and data
structures (`Q05`); and none of these scales writes horizontally without sharding. Being specific
about the boundary is what makes "just use Postgres" a senior answer rather than a lazy one.

### Interview questions

- "Choose a store for a social graph, a shopping cart, an audit log and a product catalogue.
  Justify each."
- "What is your default answer for a new service?"
- "'Schemaless' — what does that actually mean for your codebase in three years?"
- "Where does Postgres genuinely run out?"

---

## DB27 · Document databases (MongoDB)

`Advanced` · Requires: `DB26` · Unlocks: `DB29`

### Preface

A document database stores nested JSON-like documents and lets you query fields inside them. The
central design decision is **embed or reference**: put related data inside the parent document, or
store it separately and link by id.

The rule of thumb: embed what is always read together and is bounded in size; reference what is
large, unbounded, or shared.

### Details

#### 1. Embed versus reference

**Theory.** Embedding gives you the whole object in one read, with atomic updates for the whole
document. Referencing keeps documents small and lets data be shared, at the cost of a second query
(there are no real joins; `$lookup` exists but is limited and slow at scale).

**Example.** A blog post with comments. Embed the most recent 20 comments for instant page loads,
and store all comments in a separate collection referenced by post id. Reading a post is one query;
"show all comments" is a second query that most visitors never trigger. This hybrid is the standard
pattern.

**Advanced.** The hard limit is 16MB per document, and the failure is not gradual — writes start
failing once you approach it. The **unbounded array anti-pattern** is the usual cause: embedding a
list that grows forever (comments, events, log entries). It also degrades long before the limit,
because every update rewrites the whole document and every read transfers it all. Rule: never embed
an array that has no natural bound.

#### 2. Indexing and the ESR rule

**Theory.** MongoDB indexes are B-trees, like a relational database, and compound index order follows
the same logic: **Equality, Sort, Range** — note the order differs from the relational E-R-S because
MongoDB can use an index for sorting only if the sort fields come before any range field.

**Example.** For `find({tenant: X, status: Y, created: {$gt: Z}}).sort({created: -1})`, the index is
`{tenant: 1, status: 1, created: -1}`. Use `explain("executionStats")` and check
`totalDocsExamined` against `nReturned` — if the ratio is far from 1, the index is not doing its
job.

**Advanced.** MongoDB will pick a plan by racing candidate plans and caching the winner, which means
plan choices can change and can be wrong for atypical parameters — the same plan-caching hazard as
`DB18`. Also note that a query with a covered index (all fields in the index, `_id` excluded) avoids
fetching the document entirely, exactly like an index-only scan.

#### 3. Write concern and read concern

**Theory.** **Write concern** says how many replicas must acknowledge a write: `w: 1` (the primary
only), `w: "majority"`, and `j: true` (flushed to the journal). **Read concern** says what you are
allowed to see: `local`, `majority` (only data that cannot be rolled back), `linearizable`.
**Read preference** says which node to read from: `primary`, `secondary`, `nearest`.

**Example.** `w: 1` is fast and risks losing the write: if the primary fails before replicating, the
write is rolled back when it rejoins — acknowledged to your user and then gone. For anything
important, use `w: "majority", j: true`. For reading your own writes from a secondary, you need
`readConcern: majority` plus causal consistency (a session with `afterClusterTime`), which the
official drivers support.

**Advanced.** This is MongoDB's version of the durability/latency trade in `DB12` and `M33`, exposed
per operation rather than as a server setting — which is a genuine advantage: you can use
`w: "majority"` for orders and `w: 1` for analytics events in the same application. Being able to
map MongoDB's knobs onto the general concepts shows you understand both.

#### 4. Replica sets, sharding and transactions

**Theory.** A **replica set** is one primary and several secondaries with automatic election on
failure. **Sharding** adds a routing layer (`mongos`), a config server holding the metadata, and
chunk balancing across shards. Multi-document **transactions** exist since 4.0 (4.2 across shards).

**Example.** Choosing a shard key has the same constraints as `DB25`: it must appear in your queries
or every query broadcasts to all shards, and it must not be monotonically increasing (a timestamp
sends all writes to one chunk). Hashed shard keys spread writes but disable range queries.

**Advanced.** Transactions in MongoDB work, and needing them frequently is a signal that your
document boundaries are wrong — the document is supposed to *be* the transaction boundary, in the
same way an aggregate is in DDD (`M02`). They also have real costs: a 60-second default limit,
increased contention, and cross-shard transactions requiring two-phase commit internally. Use them
for occasional cross-document invariants, not as a substitute for modelling.

### Interview questions

- "Model comments on a post in MongoDB."
- "What does `w: 1` actually risk?"
- "When would you use a multi-document transaction, and what does it suggest about your model?"
- "What is the unbounded array anti-pattern?"

---

## DB28 · Wide-column and key-value at scale (Cassandra, DynamoDB)

`Expert` · Requires: `DB26`, `DB32` · Unlocks: `DB29`, `DB35`

### Preface

These databases scale writes almost linearly by adding machines. To get that, they give up almost
everything else: no joins, no ad-hoc queries, no cross-partition transactions.

The consequence is **query-first modelling**: you write down your access patterns, then design a
table for each one, duplicating data freely. This is genuinely the opposite of relational design and
is where most people go wrong.

### Details

#### 1. Partition key and sort key

**Theory.** The primary key has two parts. The **partition key** decides which machine holds the
data — all rows with the same partition key live together. The **sort key** (clustering key in
Cassandra) orders rows within a partition, enabling efficient range queries inside it.

**Example.** A chat application in DynamoDB:

```
PK = "CONVERSATION#123"      SK = "MSG#2026-09-13T10:00:00Z#msg_a"
```

"The last 50 messages in conversation 123" is a single query against one partition, returning items
in sort order. Fast, predictable, and it scales because conversations spread across partitions.

**Advanced.** You can only query efficiently **by partition key**, optionally with a condition on the
sort key. There is no "find all messages by user X across conversations" unless you built a table or
index for it. This is the fundamental constraint, and the reason you must know your access patterns
before you design anything. If they change, you add a new table or index and backfill.

#### 2. Single-table design

**Theory.** DynamoDB's advanced pattern: store several entity types in one table, using generic
`PK`/`SK` attributes with prefixes, so related entities share a partition and can be retrieved in one
query.

**Example.**

```
PK              SK                  data
USER#1          PROFILE             {name, email}
USER#1          ORDER#2026-09-01    {total, status}
USER#1          ORDER#2026-09-05    {total, status}
```

One query on `PK = USER#1` returns the profile and all orders together — the equivalent of a join,
done at write time by choosing keys.

**Advanced.** Single-table design is powerful and widely over-applied. It makes the data very hard to
read, makes new access patterns expensive to add, and demands that everyone on the team understands
the encoding. Many teams get better results with several simple tables. Be able to argue both sides —
interviewers often use this to see whether you follow advice or evaluate it.

#### 3. Secondary indexes, capacity and hot partitions

**Theory.** A **Global Secondary Index (GSI)** has its own partition key and is maintained
asynchronously (so it is eventually consistent). A **Local Secondary Index (LSI)** shares the
partition key with a different sort key and is consistent, but must be created with the table.
Capacity is per partition, so an uneven key distribution wastes it.

**Example.** The hot-partition problem: partitioning by `date` sends every write for today to one
partition, which is throttled while the rest of the table is idle. Fix it by adding a random or
derived suffix to the key (`2026-09-13#7`, with 10 suffixes) and querying all suffixes in parallel —
the salting technique from `M32`.

**Advanced.** DynamoDB's adaptive capacity mitigates moderate skew automatically by shifting capacity
toward hot partitions, and it does not rescue a genuinely single-key hotspot. Also know the limits
that shape designs: 400KB per item, 1MB per query response (so you must paginate), and a
`Scan` reading the whole table — seeing `Scan` in production code is a design error nearly every
time.

#### 4. Cassandra specifics: tunable consistency and tombstones

**Theory.** Cassandra is leaderless: any node accepts a write, and consistency is chosen per query
via the number of replicas that must respond (`ONE`, `QUORUM`, `ALL`). `R + W > N` gives you
read-your-writes.

**Example.** With replication factor 3: writing at `QUORUM` (2) and reading at `QUORUM` (2)
guarantees an overlap, so you read the latest write. Writing at `ONE` and reading at `ONE` is fastest
and may return stale data. Background processes — read repair, hinted handoff and anti-entropy
repair — converge the replicas over time.

**Advanced.** **Tombstones** are the classic Cassandra trap. A delete writes a marker rather than
removing data, because a leaderless system must be able to tell "deleted" from "not yet replicated".
Tombstones live until `gc_grace_seconds` (10 days by default) and are read on every query touching
that range. A queue-like workload with many deletes accumulates tombstones until reads time out —
which is why "do not use Cassandra as a queue" is standard advice. Lightweight transactions
(`IF NOT EXISTS`) exist, use Paxos, and are roughly an order of magnitude slower, so they are for
rare operations only.

### Interview questions

- "Model an activity feed in DynamoDB and give the exact keys."
- "Why is `Scan` a red flag?"
- "How do you avoid a hot partition?"
- "What are tombstones and why do they cause outages?"

---

## DB29 · LSM trees versus B-trees

`Expert` · Requires: `DB11`, `DB13`, `DB27`, `DB28` · Unlocks: `DB32`

### Preface

Two storage designs underlie almost every database. **B-trees** update data in place — the classic
relational approach. **LSM trees** (log-structured merge trees) only ever append, then merge files in
the background — used by Cassandra, RocksDB, LevelDB, ScyllaDB and many others.

The difference explains why some databases are much faster at writes and why they behave
unpredictably at certain moments.

### Details

#### 1. How an LSM tree works

**Theory.** Writes go to an in-memory sorted structure (the **memtable**) plus a write-ahead log for
durability. When the memtable fills, it is flushed to disk as an immutable sorted file (an
**SSTable**). Reads check the memtable, then SSTables from newest to oldest. **Compaction** merges
SSTables in the background, discarding superseded and deleted entries.

**Example.** Writes are sequential appends, so throughput is very high and does not depend on random
I/O. Reads may have to check several files — mitigated by **Bloom filters** (`DB33`) per SSTable,
which cheaply answer "this file definitely does not contain your key", so most files are skipped
without being read.

**Advanced.** Compaction strategy is the main tuning decision. **Levelled** compaction keeps each
level's files non-overlapping, giving good read performance and higher write amplification (data is
rewritten more often). **Size-tiered** merges similarly-sized files, giving better write throughput
and worse reads and space use. RocksDB defaults to levelled; Cassandra offers both plus a
time-window strategy designed for time-series data with TTLs.

#### 2. The three amplifications

**Theory.** A useful framework for comparing any storage engine:
- **Write amplification** — bytes written to disk per byte of logical data.
- **Read amplification** — disk reads per logical read.
- **Space amplification** — disk space used per byte of logical data.

You cannot minimise all three; each design picks a trade.

**Example.** LSM: low write amplification at ingest (append only) but significant total write
amplification from compaction rewriting data repeatedly; higher read amplification (several files
per read); variable space amplification (obsolete data persists until compacted). B-tree: writes go
through the WAL plus the page, and **full-page writes** after a checkpoint add more (`DB12`); reads
are a predictable handful of pages; space is stable but pages are typically only ~70% full.

**Advanced.** The summary to offer in an interview: LSM trees are optimised for **write-heavy**
workloads and compress well because data is written in sorted immutable runs; B-trees give
**predictable read latency** and are better for read-heavy, update-in-place workloads. Neither is
universally better, and "which do you prefer" is a question about workload.

#### 3. Compaction stalls

**Theory.** Compaction is background work competing for disk and CPU with foreground traffic. If
writes arrive faster than compaction can keep up, files accumulate, reads get slower, and eventually
the database throttles or stops accepting writes.

**Example.** The classic Cassandra or RocksDB incident: a bulk import outpaces compaction; pending
compactions climb; read latency rises because each read checks many more SSTables; then writes are
throttled to let compaction catch up. Throughput collapses, and the graph shows a cliff rather than
a gradual decline. The fix is to rate-limit ingest, provision more I/O, or tune compaction
concurrency.

**Advanced.** This is the operational personality of LSM systems: they are extremely fast right up
until they are not, and recovery requires reducing load rather than adding it. Monitor pending
compaction bytes and SSTable count per table as leading indicators. B-tree systems have the
analogous problem in checkpoints and vacuum (`DB08`, `DB12`) — every storage engine has background
work that must keep up, and every one has an incident mode where it does not.

#### 4. Where you meet each

**Theory.** B-tree: Postgres, MySQL InnoDB, SQL Server, Oracle, MongoDB's WiredTiger (which is a
hybrid but behaves B-tree-like). LSM: Cassandra, ScyllaDB, RocksDB (and everything built on it —
CockroachDB, TiKV, Kafka Streams state stores, many others), LevelDB, HBase, ClickHouse's
MergeTree (a variation).

**Example.** Knowing that CockroachDB and TiDB store data in RocksDB/Pebble explains their
performance characteristics: excellent write throughput, compaction to manage, and careful attention
to read amplification.

**Advanced.** The industry has largely converged on RocksDB as the embedded storage engine, which
means LSM behaviour — compaction tuning, write stalls, bloom filter sizing — is broadly useful
knowledge rather than Cassandra trivia. It also shows up in Kafka Streams and Flink state stores,
where a stateful stream processor's mysterious latency is often compaction.

### Interview questions

- "Why is Cassandra faster at writes than Postgres, and what do you pay for it?"
- "Explain read, write and space amplification, and which one each design optimises."
- "What is a compaction stall?"
- "Where does a Bloom filter fit in an LSM read path?"

---

## DB30 · Redis as a data store

`Intermediate` · Requires: `DB26` · Unlocks: `Q05`

### Preface

Redis keeps everything in memory and executes commands one at a time on a single thread. That makes
it extremely fast and makes every command atomic, with no locking needed.

It can persist to disk, but it is best understood as a **fast, mostly-durable** store — excellent for
caches, counters, rate limits, sessions and queues, and a poor choice as the system of record for
anything you cannot afford to lose.

Full treatment is in the caching field (`Q05`–`Q08`); this node covers when it may hold data at all.

### Details

#### 1. Why it is fast

**Theory.** Everything is in RAM, so there is no disk I/O on the read path. Commands execute on one
thread, so there are no locks and no contention, and each command is atomic by construction. The
data structures are purpose-built and the protocol is simple.

**Example.** A single Redis instance handles on the order of 100,000 operations per second on
ordinary hardware, with sub-millisecond latency. Redis 6 added threaded I/O (reading and writing
sockets in parallel) but **command execution remains single-threaded** — a detail interviewers like
to probe.

**Advanced.** The single thread is also the main hazard: one slow command blocks everything. `KEYS *`
on a large keyspace, `SMEMBERS` on a huge set, a large `DEL`, or an expensive Lua script all stall
every other client. Use `SCAN` instead of `KEYS`, `UNLINK` instead of `DEL` for large values, and
keep scripts short. "Which Redis commands would you ban in production" is a good question to have an
answer for.

#### 2. Durability, honestly

**Theory.** RDB takes periodic point-in-time snapshots; AOF appends every write command to a file,
with `appendfsync everysec` by default. Both can lose data — RDB loses everything since the last
snapshot, AOF loses up to a second.

**Example.** So the failure mode is: Redis acknowledges your write, the process dies, and the write
is gone. For a cache that is irrelevant. For a session it means users are logged out. For a
financial balance it is unacceptable. Match the data to the guarantee.

**Advanced.** Even `appendfsync always` does not make Redis a relational database: replication is
asynchronous, so a failover can lose acknowledged writes (`M33`), and there is no multi-statement
rollback — `MULTI`/`EXEC` batches commands atomically but a logical error in the middle does not undo
the earlier ones. AWS MemoryDB is the durable variant (a distributed transaction log), which is worth
naming if someone insists on Redis as a source of truth.

#### 3. What it is legitimately good for

**Theory.** Data that is derived, ephemeral, or cheap to rebuild; and data structures that are
awkward or slow in a relational database.

**Example.** Good uses: caching (`Q02`), rate limiting counters (`A08`), session storage where
re-login is acceptable, leaderboards via sorted sets, real-time presence, deduplication sets with
TTLs, job queues via streams or lists (`Q08`), and pub/sub for fan-out where loss is tolerable.

**Advanced.** The boundary case is distributed locks: acceptable for **efficiency** ("avoid two
workers doing the same expensive job"), not for **correctness** ("never charge twice"), because a
lock can be lost to a failover or an eviction. For correctness you need a fencing token checked at
the point of effect, or a conditional write in the real database (`C11`). Getting this distinction
right is a frequent interview discriminator.

#### 4. Memory management

**Theory.** Redis holds everything in RAM, so `maxmemory` and the eviction policy define its
behaviour when full. `noeviction` returns errors on write; `allkeys-lru` and `allkeys-lfu` evict;
`volatile-*` variants only evict keys that have a TTL.

**Example.** The mistake to avoid: using one Redis instance for both a cache and for locks or
sessions, with `allkeys-lru`. Under memory pressure Redis may evict a lock or a session, and the
symptoms are baffling. Separate the instances, or use `volatile-lru` and set TTLs only on cache
entries so the non-cache keys are never evicted.

**Advanced.** `allkeys-lfu` (least frequently used) is usually better than LRU for caches, because it
resists a single large scan wiping the genuinely hot keys. Also watch memory fragmentation — the
`mem_fragmentation_ratio` in `INFO` — where the allocator holds more memory than the dataset
requires; `activedefrag` helps. See `Q06`.

### Interview questions

- "When is Redis an acceptable source of truth?"
- "Redis is single-threaded — how does it do 100,000 operations per second?"
- "Which Redis commands would you ban in production?"
- "You used the same Redis for cache and locks. What can go wrong?"

---

## DB31 · Search engines (Elasticsearch / OpenSearch)

`Advanced` · Requires: `DB14`, `DB26` · Unlocks: `SD12`

### Preface

A search engine is built around an **inverted index**: a map from each word to the documents
containing it. That is what makes "find documents containing these words, ranked by relevance" fast,
which a relational database cannot do well at scale.

The key architectural rule: Elasticsearch is a **derived** store. Your database remains the source
of truth, and search is kept in sync from it.

### Details

#### 1. The inverted index and analysis

**Theory.** At index time, text passes through an **analyzer**: a tokenizer splits it into terms,
then filters lowercase them, remove stop words, apply stemming, expand synonyms. The resulting terms
are stored in the inverted index. Query text passes through an analyzer too, and the terms are
matched.

**Example.** "Running Shoes" indexed with an English analyzer becomes `["run", "shoe"]`. A search
for "runs shoe" becomes `["run", "shoe"]` and matches. This is why a `LIKE '%running%'` in SQL
cannot compete — it has no concept of word stems, and it cannot use an index.

**Advanced.** The most common Elasticsearch bug is a mismatch between the index-time and query-time
analyzers, so terms never match and results are silently empty. The second most common is using a
`term` query (exact, unanalysed) on an analysed field — it compares your raw string against the
processed terms and matches nothing. Knowing the `text` (analysed) versus `keyword` (exact) field
distinction is basic Elasticsearch literacy.

#### 2. Relevance

**Theory.** Documents are scored and returned in order. The default algorithm is **BM25**, which
favours documents where the query terms are frequent (term frequency), where the terms are rare
across the corpus (inverse document frequency), and which are shorter (length normalisation).

**Example.** Real relevance tuning combines text scoring with business signals: boost exact title
matches, boost products in stock, decay by age, boost by popularity. That blending — and being able
to explain and debug it with the `explain` API — is where the work actually is, and it is what a
relational database gives you no way to express.

**Advanced.** The scoring subtlety worth knowing: IDF is calculated **per shard** by default, so with
few documents and several shards the same query can rank inconsistently. It converges as data grows;
for small indexes use `search_type=dfs_query_then_fetch` or fewer shards. This explains the classic
"my test index gives weird rankings" confusion.

#### 3. Operational realities

**Theory.** An index is divided into **primary shards** (fixed at creation — you cannot change the
number without reindexing) and **replicas** (changeable). New documents become searchable only after
a **refresh**, every second by default — Elasticsearch is near-real-time, not real-time.

**Example.** Two consequences that surface immediately in production. First, deep pagination:
`from=10000&size=10` requires every shard to return 10,010 results to be merged, which is why there
is a 10,000 limit and why you use `search_after` with a sort key (the keyset pagination of `DB20`).
Second, mapping changes: you cannot change a field's type in place. The standard procedure is to
create a new index, reindex into it, and switch an **alias** atomically — so always query through an
alias, never the index name directly.

**Advanced.** Shard sizing matters: aim for tens of gigabytes per shard. Too many small shards waste
memory and slow queries (each shard is a separate Lucene index with its own overhead); too few large
ones limit parallelism and make recovery slow. For time-based data, use time-based indices with an
alias and a lifecycle policy, so retention is an index deletion rather than a mass delete —
the same logic as partitioning (`DB24`).

#### 4. Keeping it in sync

**Theory.** Since the database is the source of truth, changes must flow into Elasticsearch. Options:
dual write from the application (unreliable — the dual-write problem, `M15`), change data capture
from the database log (reliable), or a periodic full reindex (simple, slow, and fine for small
datasets).

**Example.** The production-grade design: outbox or CDC → a queue → an indexer that writes to
Elasticsearch, with a **version number** on each document so out-of-order updates do not overwrite
newer data (`external_gte` versioning), plus a nightly reconciliation job comparing counts and
checksums, and a documented full-reindex procedure for when it drifts.

**Advanced.** Accept that it *will* drift — a failed indexer, a dropped message, a bug — so the
question is how quickly you detect and repair it, not how to prevent it. Design the reindex to be
routine and safe (build into a new index, verify, swap the alias) rather than an emergency
operation nobody has practised.

### Interview questions

- "Product search with filters and typo tolerance. Postgres full-text or Elasticsearch? Where is the
  line?"
- "How do you keep Elasticsearch in sync with Postgres?"
- "Why can't you change the number of primary shards?"
- "Why is deep pagination expensive in a distributed search engine?"

---

## DB32 · Analytics, columnar storage and OLAP

`Advanced` · Requires: `DB03`, `DB11`, `DB24`, `DB26`, `DB29` · Unlocks: `SD06`

### Preface

Operational databases (OLTP) are built for many small transactions touching a few rows. Analytical
databases (OLAP) are built for few queries scanning billions of rows and aggregating a handful of
columns.

The structural difference is **row storage versus column storage**, and it produces performance
differences of one or two orders of magnitude. It is why running analytics on your production
database is both slow and dangerous.

### Details

#### 1. Columnar storage

**Theory.** A row store keeps all of a row's columns together, so reading one row is one page read.
A column store keeps each column's values together, so reading one column across a billion rows
touches only that column's data.

**Example.** `SELECT avg(amount) FROM orders` over a billion rows with 40 columns. A row store reads
all 40 columns' worth of pages — perhaps 400GB. A column store reads only `amount` — perhaps 4GB, and
because adjacent values are similar it compresses extremely well (often 10:1), so maybe 400MB
actually leaves the disk. That is the entire difference.

**Advanced.** Columnar formats add more: **run-length** and **dictionary encoding** (a column with
five distinct values stores tiny integers), **min/max statistics per block** so irrelevant blocks are
skipped entirely (the same idea as BRIN, `DB14`), and **vectorised execution** — processing values in
batches through CPU SIMD instructions instead of row by row. Together these give the order-of-
magnitude gap, not storage layout alone.

#### 2. OLTP versus OLAP, side by side

**Theory.**
| | OLTP | OLAP |
|---|---|---|
| Query shape | few rows, by key | billions of rows, few columns |
| Writes | many small, concurrent | bulk loads, append-only |
| Storage | row | column |
| Indexes | many B-trees | few; block statistics instead |
| Normalisation | normalised | star schema, denormalised |
| Latency target | milliseconds | seconds to minutes |

**Example.** The same data lives in both: Postgres serves "show this customer's orders" in 2ms;
ClickHouse or BigQuery serves "revenue by country by month for two years" in 2 seconds over the same
data copied across by CDC. Neither could do the other's job well.

**Advanced.** The hybrid category (HTAP) — TiDB, SingleStore, Postgres with a columnar extension —
tries to serve both from one system, and usually involves maintaining a column-oriented copy
internally. It is attractive for reducing operational surface and is rarely as fast as a dedicated
analytical engine. DuckDB is worth naming as the modern lightweight option: an embedded columnar
engine that queries Parquet files directly and is remarkably capable for mid-sized analytics.

#### 3. Star schema

**Theory.** Analytical models are deliberately denormalised into **fact** tables (the events:
one row per order line, with measures and foreign keys) surrounded by **dimension** tables (the
descriptive context: customer, product, date, store).

**Example.** `fact_order_lines` with `date_key`, `customer_key`, `product_key`, `quantity`, `amount`;
dimensions holding the attributes. Queries join facts to a few dimensions and aggregate. Joins are
few and predictable, which is what analytical engines optimise for.

**Advanced.** **Slowly changing dimensions** are the subtle part: when a customer moves country, do
historical orders show the old country or the new one? Type 1 overwrites (history changes), Type 2
adds a new dimension row with validity dates (history is preserved, and the fact points at the
version current at the time). Type 2 is usually correct for analytics and is a good detail to raise —
it is the same "value at the time versus current value" question as `DB01`.

#### 4. Getting data there

**Theory.** ETL transforms before loading; **ELT** loads raw data and transforms inside the warehouse
(the modern default, because warehouse compute is cheap and it keeps the raw data available).
Change data capture streams row changes continuously (`Q18`).

**Example.** A common modern pipeline: Debezium reads the Postgres WAL → Kafka → a loader writes to
BigQuery/Snowflake/ClickHouse → dbt builds transformed models on a schedule → BI tools query those.
The production database is never touched by an analyst.

**Advanced.** The answer to "the data team is killing our production database", in order of
preference: (1) give them a warehouse fed by CDC; (2) failing that, a dedicated read replica they
cannot affect anyone with; (3) at minimum, a separate database role with a low `statement_timeout`
and restricted permissions. What you must not do is let ad-hoc analytical queries run on the primary
— they hold long transactions, which blocks vacuum and causes bloat (`DB08`), so the damage outlives
the query.

### Interview questions

- "The data team is killing your production database with dashboards. Options in order of
  preference?"
- "Why is a column store so much faster for aggregates?"
- "What is a star schema and why denormalise for analytics?"
- "What is a slowly changing dimension?"

---

## DB33 · Probabilistic and specialised data structures

`Advanced` · Requires: `DB13` · Unlocks: —

### Preface

Some questions do not need an exact answer. "Roughly how many unique visitors?" or "is this
definitely not in the set?" can be answered with a tiny fraction of the memory an exact answer needs,
by accepting a small, quantifiable error.

Knowing three or four of these and when to reach for them is a pleasant way to stand out in an
interview.

### Details

#### 1. Bloom filters

**Theory.** A bit array plus k hash functions. To add an item, set the k bits it hashes to. To test,
check those bits: if any is 0 the item is **definitely not** present; if all are 1 it is **probably**
present. No false negatives, tunable false positives, and items cannot be removed.

**Example.** The canonical use is avoiding pointless lookups. An LSM tree keeps a Bloom filter per
SSTable so a read skips files that certainly do not contain the key (`DB29`). In an application:
before querying the database for "has this user seen this article", check a Bloom filter — a negative
answer is certain and free, and a positive answer costs one query to confirm.

**Advanced.** Sizing is a formula worth knowing exists: for n items and a false positive rate p, the
bit array needs about `-n·ln(p)/(ln2)²` bits — roughly 10 bits per item for a 1% error rate. So a
Bloom filter for 10 million items at 1% error is about 12MB, versus hundreds of megabytes for the
actual keys. Variants: **counting** Bloom filters allow deletion; **cuckoo filters** allow deletion
and are more space-efficient at low error rates.

#### 2. HyperLogLog

**Theory.** Counts distinct items using a fixed, tiny amount of memory, by observing the longest run
of leading zeros in the hashes seen — a statistical property that correlates with cardinality.

**Example.** Redis implements it directly: `PFADD visitors:2026-09-13 user123` then
`PFCOUNT visitors:2026-09-13`. About 12KB per counter for roughly 0.81% error, regardless of whether
you counted a thousand or a billion items. Exact counting would require storing every distinct id.

**Advanced.** The property that makes it genuinely powerful is that HyperLogLogs are **mergeable**:
`PFMERGE` combines daily counters into a weekly unique count — which is not the sum of daily counts,
because of overlap, and would otherwise require the raw sets. This is why analytics systems store
HLL sketches per dimension per time bucket and combine them at query time.

#### 3. Count-min sketch

**Theory.** Estimates the frequency of items using a small 2D array of counters and several hash
functions. It can overestimate (through collisions) but never underestimates. Used for finding heavy
hitters.

**Example.** Identifying hot keys in a cache or hot partitions in a database (`M32`) without storing
per-key counters for millions of keys. Combined with a small heap of the current top-N, you get "the
20 busiest keys right now" in a few kilobytes — exactly the instrumentation you wish you had during
a hot-key incident.

**Advanced.** The same structure powers per-tenant rate limiting at scale when exact counting is too
expensive, and network telemetry for detecting traffic spikes by source. The bias is one-directional,
which matters: for abuse detection, overestimating is safe; for billing, it is not.

#### 4. Skip lists, tries and Merkle trees

**Theory.** A **skip list** is a sorted structure with probabilistic express lanes, giving B-tree-like
O(log n) operations with far simpler concurrent implementation — which is why Redis uses it for
sorted sets. A **trie** stores strings by shared prefix, giving fast prefix search and
autocomplete. A **Merkle tree** hashes data in a tree so two parties can find their differences by
comparing a few hashes.

**Example.** Merkle trees are how Cassandra and DynamoDB perform anti-entropy repair: two replicas
compare tree hashes, descend only into subtrees that differ, and exchange just the divergent data
instead of the entire dataset. Git uses the same idea for commits and trees.

**Advanced.** The general pattern across all of these: **trade a bounded, well-understood error or a
probabilistic guarantee for an enormous reduction in memory or network traffic.** Being able to state
that principle, and to say where the error is acceptable and where it is not, is worth more than
memorising the structures.

### Interview questions

- "Count unique daily visitors across 200 million events with a fixed memory budget."
- "Where would a Bloom filter save you a database round trip?"
- "How would you find the top 20 hottest keys without per-key counters?"
- "How do two replicas efficiently find out how they differ?"

---

## DB34 · Full-text search in Postgres

`Intermediate` · Requires: `DB14` · Unlocks: `DB31`

### Preface

Postgres has built-in full-text search that is good enough for a large fraction of applications, and
choosing it means one fewer system to run, back up, secure and keep in sync.

Knowing exactly where it stops being good enough — and being able to say so — is the valuable part.

### Details

#### 1. tsvector and tsquery

**Theory.** `to_tsvector(config, text)` produces a sorted list of normalised lexemes with positions.
`to_tsquery` / `plainto_tsquery` / `websearch_to_tsquery` produce a query. The `@@` operator matches
them. A GIN index on the `tsvector` makes it fast.

**Example.** The recommended shape uses a generated column so the vector is maintained
automatically:

```sql
ALTER TABLE articles ADD COLUMN search tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(body,'')),  'B')
  ) STORED;

CREATE INDEX ON articles USING gin (search);

SELECT id, ts_rank(search, q) AS rank
FROM articles, websearch_to_tsquery('english', $1) q
WHERE search @@ q ORDER BY rank DESC LIMIT 20;
```

`setweight` makes title matches outrank body matches. `websearch_to_tsquery` accepts human syntax
(quoted phrases, `or`, `-excluded`) without throwing on malformed input, which the older
`to_tsquery` does.

**Advanced.** The `english` configuration handles stemming and stop words for English only. For
multilingual content you need a configuration per language and a column recording which to use —
which is one of the points where Elasticsearch's richer analyzer support starts to pay. Also add the
`unaccent` extension so "café" matches "cafe"; it must be part of the indexed expression, not applied
at query time only.

#### 2. Trigram search for fuzzy matching

**Theory.** The `pg_trgm` extension breaks strings into three-character sequences and indexes them,
which supports similarity matching, typo tolerance, and — importantly — **substring search with a
leading wildcard**.

**Example.**

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX ON users USING gin (name gin_trgm_ops);

SELECT * FROM users WHERE name ILIKE '%john%';           -- now index-assisted
SELECT * FROM users ORDER BY name <-> 'jhon' LIMIT 10;   -- fuzzy, by distance
```

This solves the `LIKE '%x%'` problem from `DB13` and provides "did you mean" behaviour without
another system.

**Advanced.** Trigram indexes are large (each string produces many trigrams) and their performance
degrades with very short search terms, which match enormous numbers of rows. Combine approaches:
full-text search for word matching, trigram for fuzzy name lookup and autocomplete, and a length
check to reject one-character queries.

#### 3. When Postgres search is enough

**Theory.** It is enough for: a single language, a corpus up to a few million documents, simple
relevance needs, and search that is not your product's core differentiator.

**Example.** Searching your own application's data — orders, customers, documents, support tickets —
is nearly always fine in Postgres, and it has a decisive advantage: the search index is **always
consistent with the data**, in the same transaction, with no synchronisation pipeline, no drift and
no reconciliation job.

**Advanced.** Switch to a dedicated engine when you need: serious relevance tuning and
experimentation, multiple languages with different analyzers, faceted search with counts across
many dimensions, "more like this" and vector-hybrid search, aggregations over search results,
per-field boosting that you tune frequently, or scale beyond what one machine indexes comfortably.
Framing it as "consistency and simplicity versus relevance features and scale" is the clean way to
answer.

### Interview questions

- "When is Postgres full-text search enough?"
- "How do you support `ILIKE '%term%'` efficiently?"
- "How do you make title matches rank above body matches?"
- "What is the biggest advantage of keeping search inside your main database?"

---

## DB35 · Distributed SQL and NewSQL

`Expert` · Requires: `DB23`, `DB25`, `DB28`, `M20` · Unlocks: —

### Preface

These systems try to give you both things: the SQL, transactions and consistency of a relational
database, and the horizontal scale and survivability of a distributed one.

They largely succeed, and the cost is latency — a distributed transaction needs consensus round
trips that a single machine does not.

### Details

#### 1. How they work

**Theory.** Data is split into ranges or shards; each is replicated to (typically) three nodes and
kept consistent with a consensus protocol, usually Raft (`M20`). Transactions spanning ranges use
two-phase commit, with the coordinator itself made fault-tolerant by consensus — which removes 2PC's
blocking problem (`M13`).

**Example.** CockroachDB splits data into ~512MB ranges, each a Raft group. Ranges split and
rebalance automatically as data grows or access skews. A transaction touching two ranges runs 2PC
across two Raft groups. The application sees a Postgres-compatible SQL interface and mostly does not
need to know.

**Advanced.** The cost to be able to quantify: every write needs a majority of replicas to
acknowledge, so a single-region write is a few milliseconds and a cross-region write is tens to
hundreds of milliseconds. If you spread replicas across continents for survivability, every write
pays the distance. This is why these systems offer per-table locality settings — pin a table's
replicas to one region when global survivability is not needed for that data.

#### 2. Spanner and TrueTime

**Theory.** Google Spanner provides **external consistency** — transactions appear in an order
consistent with real time globally. It achieves this with **TrueTime**: GPS receivers and atomic
clocks in every data centre give an API returning a time *interval* with a bounded error, and a
committing transaction **waits out** that uncertainty (a few milliseconds) before making itself
visible.

**Example.** The trade is explicit and elegant: pay a small, bounded wait on every commit, and in
return get globally consistent snapshots and consistent reads from any replica without coordination.
It is the clearest example in the industry of buying a distributed-systems guarantee with hardware.

**Advanced.** Systems without atomic clocks approximate this. CockroachDB uses hybrid logical clocks
(`M21`) with a configured maximum clock offset (500ms by default); if a node's clock drifts beyond
it, the node **removes itself from the cluster** rather than risk violating consistency. That is a
striking illustration of how deeply clock assumptions are baked into distributed correctness.

#### 3. The sharded-compatibility approach

**Theory.** A different strategy: keep the existing single-node database and put sharding around it.
Vitess shards MySQL; Citus shards Postgres. You get a familiar engine and ecosystem, with sharding
handled by a routing layer.

**Example.** Vitess powers YouTube, Slack and GitHub. Citus (now part of Azure Cosmos DB for
Postgres) distributes tables by a key across Postgres nodes and is particularly effective for
multi-tenant applications where `tenant_id` is a natural distribution column — most queries then run
entirely on one node.

**Advanced.** The limitation is the same as manual sharding (`DB25`): cross-shard joins and
transactions are limited or slow, and the schema must be designed around the distribution key. The
benefit is that everything else — the SQL dialect, extensions, tooling, your team's knowledge —
continues to work. For a team already on Postgres with a clean tenant boundary, Citus is often a much
lower-risk answer than migrating to a new distributed database.

#### 4. Aurora and storage-compute separation

**Theory.** A different axis again: keep one writer but replace the storage layer with a distributed,
replicated service. Aurora ships **only the write-ahead log** to a storage layer that applies it
across six copies in three availability zones, with quorum reads and writes.

**Example.** The slogan is "the log is the database". Because pages are reconstructed in the storage
layer, the database does not write full pages (`DB12`), which cuts write amplification dramatically.
Replicas read from the same shared storage, so adding a read replica does not add write work and
replica lag is minimal. Failover is fast because the data does not move.

**Advanced.** Neon applies the same idea to open-source Postgres and adds **branching** — a
copy-on-write clone of your database, so a test environment or a preview deployment gets a full copy
of production data in seconds. That is a genuinely useful developer-experience feature and a good
example of an architectural choice enabling a product one. The general trade of storage-compute
separation: excellent elasticity and durability, some added read latency, and vendor lock-in.

### Interview questions

- "Why is a distributed SQL database slower per transaction than a single Postgres, and when is the
  trade worth it?"
- "What is TrueTime and what problem does it solve?"
- "Vitess or CockroachDB for an existing MySQL application? What decides it?"
- "What does 'the log is the database' mean in Aurora?"
