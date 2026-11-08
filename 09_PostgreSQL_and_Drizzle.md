# PostgreSQL Engineering and Drizzle ORM for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to running PostgreSQL in production and talking to it from TypeScript with Drizzle ORM. It teaches how the database behaves internally, so you can explain a slow query, a bloated table, a lock pile-up or a failed migration, not just recall syntax.

**Audience:** backend engineers who can already write SQL and want production-level PostgreSQL skill.
**Prerequisites:** the SQL chapter (basic `SELECT`, `JOIN`, aggregation, transactions). This guide does not re-teach them. Comfort with TypeScript and Node.js for the Drizzle chapter.
**Scope:** PostgreSQL as a self-managed or managed service, from roughly version 13 onward, plus Drizzle ORM and drizzle-kit for PostgreSQL. Other databases and other ORMs are out of scope.
**Note on accuracy:** PostgreSQL behaviour described here is long-stable, but defaults and features move between major versions, so confirm against the documentation for your version (the version number is in the URL of the official docs). Drizzle's API evolves quickly. Treat every Drizzle snippet as a shape to confirm in the current Drizzle documentation, not as copy-paste truth.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 16 lists hands-on labs. Chapter 17 collects interview questions with model answers. Chapter 18 lists primary references. Chapter 19 is a glossary of hard terms. Each chapter starts with an "In plain words" summary.

**What "good enough" means for a backend role:** you can read an `EXPLAIN (ANALYZE, BUFFERS)` plan and say why it is slow, ship a schema change to a busy table without an outage, explain why connections and long transactions are dangerous, pick the right isolation level and retry strategy, and use an ORM without being trapped by it. Chapters 2, 3, 5, 6, 7, 11 and 13 matter most.

## Contents

1. Architecture: processes, memory and the write path
2. MVCC: tuple versions and visibility
3. VACUUM, autovacuum, wraparound, HOT updates
4. Indexes
5. The planner, statistics and reading EXPLAIN
6. Locking
7. Transactions and isolation levels
8. Data modelling features: constraints, generated columns, JSONB, full-text search, partitioning
9. Roles, privileges and row-level security
10. Replication, backup and point-in-time recovery
11. Connection management and PgBouncer
12. Managed and serverless PostgreSQL
13. Zero-downtime migrations
14. Monitoring and diagnosis
15. Drizzle ORM and drizzle-kit
16. Practice labs
17. Interview questions
18. References
19. Glossary

---

## 1. Architecture: processes, memory and the write path

**In plain words.** PostgreSQL starts one new process for each client that connects. Before it changes a data file, it first writes a short note of the change to a log. After a crash, it reads that log to repair the data.

### Understand

**Process-per-connection model.** A main process, the **postmaster**, listens on the port. For every client connection it forks a dedicated **backend** process. That backend parses, plans and executes the client's queries until the client disconnects. This is why a connection is expensive (chapter 11): it is an operating-system process with its own memory, not a lightweight thread.

**Helper processes** started by the postmaster include:
- **checkpointer**: periodically flushes dirty pages to data files (a checkpoint).
- **background writer**: trickles dirty pages out between checkpoints so backends rarely have to write them.
- **WAL writer**: flushes write-ahead log buffers to disk.
- **autovacuum launcher and workers**: run vacuum and analyze (chapter 3).
- **walsender / walreceiver**: ship and receive WAL for replication (chapter 10).
- **archiver**: copies completed WAL segments to an archive when archiving is enabled.

**Memory.**
- `shared_buffers`: a shared page cache owned by PostgreSQL. A common starting point is around a quarter of RAM on a dedicated host, but this is a tuning decision, not a law. PostgreSQL also relies on the operating system page cache, so data is often cached twice.
- `work_mem`: memory a single sort or hash operation may use before spilling to disk. It is **per operation, per query, per connection**, so a complex query with several sorts and hashes on many connections multiplies it. Raising it globally is a classic way to cause an out-of-memory kill.
- `maintenance_work_mem`: used by `VACUUM`, `CREATE INDEX` and similar maintenance work.
- `effective_cache_size`: a hint to the planner about how much cache (shared buffers plus OS cache) is likely available. It allocates nothing.

**The write path and WAL.** Changing a data page in place and hoping to survive a crash is unsafe. Instead PostgreSQL uses **write-ahead logging**: before a change to a data page may reach disk, a record describing the change must be written to the **WAL**. On commit, the WAL up to that point is flushed (`fsync`), and the transaction is durable even though the data page itself is still only in memory. The sequence for a commit:

1. The backend modifies the page in `shared_buffers` and appends a WAL record.
2. On `COMMIT`, WAL is flushed to disk. The client is told the commit succeeded.
3. Later, the checkpointer or background writer writes the dirty data page to its file.
4. After a crash, PostgreSQL replays WAL from the last checkpoint and reaches a consistent state.

**Checkpoints.** A checkpoint writes all dirty pages and records a point from which crash recovery can start. They are triggered by time (`checkpoint_timeout`) or by WAL volume (`max_wal_size`). Checkpoints spread their writes over a fraction of the interval (`checkpoint_completion_target`) to avoid I/O spikes. Too-frequent checkpoints (small `max_wal_size` under heavy writes) cause I/O storms and extra full-page writes in WAL. The first change to a page after a checkpoint writes the whole page to WAL (`full_page_writes`) to protect against torn pages.

**Durability knobs.** `synchronous_commit = off` lets a commit return before WAL is flushed. A crash can lose the last fraction of a second of acknowledged commits but cannot corrupt the database. `fsync = off` can corrupt the database and is not a tuning option for real data.

**WAL is also the replication and recovery stream.** Streaming replication, point-in-time recovery and logical decoding all read WAL. `wal_level` (`minimal`, `replica`, `logical`) controls how much information it carries.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Server settings | `SHOW ALL`, `SHOW shared_buffers`, `pg_settings` view, `postgresql.conf`, `ALTER SYSTEM` | Find a setting's value, its source and whether it needs a restart |
| Interactive client | `psql`: `\l \dt \d+ table \di+ \x \timing \watch \copy`, `\?` | Inspect any object and time any query without a GUI |
| Processes | `pg_stat_activity`, `ps` on the host | Map each connection to a backend process and its state |
| Checkpoint behaviour | `log_checkpoints`, the checkpoint statistics views for your version | Tell time-triggered from WAL-triggered checkpoints |
| WAL position | `pg_current_wal_lsn()`, `pg_wal_lsn_diff()` | Measure how much WAL a workload generates |

### Common mistakes

- Raising `work_mem` globally to fix one slow report, then running out of memory under concurrency. Raise it per session or per role for that job.
- Setting `shared_buffers` to most of RAM and starving the OS cache and per-backend memory.
- Disabling `fsync` for speed on data you care about.
- Assuming a commit means the data file is updated. Durability comes from WAL.

### Self-check

- Why does PostgreSQL fork a process per connection, and what does that cost?
- On `COMMIT`, what exactly is flushed to disk and what is not?
- What does a checkpoint do, and what makes it happen?
- Why is `work_mem` dangerous to set high globally?

---

## 2. MVCC: tuple versions and visibility

**In plain words.** When you change a row, PostgreSQL keeps the old copy and writes a new copy. Each transaction sees only the copy that was right when it started. This lets reading and writing happen together, but old copies pile up until they are cleaned.

### Understand

**The problem MVCC solves.** Readers should not block writers and writers should not block readers. PostgreSQL achieves this with **multi-version concurrency control**: instead of overwriting a row, it keeps multiple physical versions and lets each transaction see the version that was current for its snapshot (a fixed view of what was committed at one moment).

**Tuples and system columns.** A physical row version is a **tuple** stored in an 8 kB page of a table's heap file. Every tuple carries hidden system columns you can select explicitly:
- `xmin`: the ID of the transaction that created this tuple version.
- `xmax`: the ID of the transaction that deleted or locked it (zero if none).
- `ctid`: the physical location (page, offset) of the tuple.

You can see them with `SELECT xmin, xmax, ctid, * FROM t;`. Watching them change across an update is the fastest way to internalise MVCC.

**Why UPDATE creates a new tuple.** An `UPDATE` does not modify the old tuple's data. It marks the old tuple's `xmax` with the updating transaction's ID and inserts a new tuple with that ID as `xmin`. A `DELETE` only sets `xmax`. The old versions stay on the page until no snapshot can need them, then `VACUUM` reclaims them. Consequences:
- An update-heavy table accumulates **dead tuples**. Without cleanup this is **bloat**: the table and its indexes grow larger than their live data, making scans slower and caches less effective.
- Every update touches the heap and, unless it is a HOT update (chapter 3), every index too. Write amplification is real.
- `SELECT count(*)` has no cheap shortcut, because visibility is per tuple and per snapshot.

**Visibility.** A **snapshot** records which transactions were in progress when it was taken. A tuple is visible to a snapshot if its creating transaction committed before the snapshot and its deleting transaction (if any) did not commit before the snapshot. Commit status lives in a separate structure (commit log) and is cached as **hint bits** on tuples, so the first read after a bulk load can be surprisingly slow and can dirty pages.

**What defines the oldest thing vacuum must keep.** The oldest snapshot still in use (the "xmin horizon") bounds which dead tuples can be removed. A single **long-running transaction**, an idle-in-transaction session, a forgotten replication slot or a standby with `hot_standby_feedback` can hold that horizon back and stop cleanup for the entire database. This is among the most common causes of production bloat.

**Transaction IDs.** Transaction IDs are 32-bit and compared in a circular way (chapter 3 covers the consequence). Only transactions that write get an ID. Rolled-back transactions leave dead tuples that are invisible immediately, because their `xmin` never committed.

**Sequences are not transactional.** A rolled-back insert still consumed a sequence value, so gaps in ids are normal. Never rely on gap-free numbering from a sequence.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Seeing versions | `SELECT xmin, xmax, ctid, *`, `pageinspect` extension | Show an update creating a new tuple |
| Table and index size | `pg_relation_size`, `pg_total_relation_size`, `pg_size_pretty`, `\dt+` | Quantify bloat versus live data |
| Dead tuple counts | `pg_stat_user_tables.n_live_tup`, `n_dead_tup` | Spot tables vacuum is not keeping up with |
| Bloat estimation | `pgstattuple` extension | Measure real dead space on a suspect table |
| Oldest snapshot holders | `pg_stat_activity.backend_xmin`, `xact_start`, `pg_replication_slots` | Find what pins the horizon |

### Common mistakes

- Leaving transactions open while the application does slow external work (HTTP calls, file uploads). Keep transactions short and never hold one across a network call to another service.
- Treating `idle in transaction` sessions as harmless. They hold snapshots and often locks.
- Using a table as a high-churn queue with no vacuum tuning (chapter 3).
- Expecting sequence values to be contiguous.

### Self-check

- What do `xmin` and `xmax` mean, and what happens to them on `UPDATE` and `DELETE`?
- Why can a readers-only workload still cause a table to bloat?
- Why does `count(*)` on a large table cost a full scan?
- Name three things that can prevent vacuum from removing dead tuples.

---

## 3. VACUUM, autovacuum, wraparound, HOT updates

**In plain words.** VACUUM cleans old row copies so the space can be used again. Autovacuum runs it for you in the background. If cleaning falls far behind, tables grow too large, and in the worst case the database stops accepting writes.

### Understand

**What VACUUM does.**
- Removes dead tuples that no snapshot needs and marks their space reusable **within the table** (it normally does not shrink the file).
- Updates the **free space map** and the **visibility map** (which pages contain only tuples visible to everyone). The visibility map enables index-only scans (chapter 4) and lets vacuum skip pages.
- **Freezes** old tuples (below).
- `VACUUM ANALYZE` also refreshes planner statistics.
- `VACUUM FULL` rewrites the table to a new file and returns space to the operating system, but takes an `ACCESS EXCLUSIVE` lock for the duration. It is an outage on a busy table. Extensions such as pg_repack rebuild tables with much shorter locks.

**Autovacuum** is a background system that runs vacuum and analyze automatically. A table becomes eligible when dead tuples exceed `autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * reltuples` (defaults: 50 and 0.2, so 20% of the table). Analyze has its own threshold and scale factor (defaults 50 and 0.1). On a huge table, 20% is an enormous number of dead rows, so autovacuum runs rarely and then runs for a long time. The standard fix is to lower the scale factor **per table**:

```sql
ALTER TABLE jobs SET (autovacuum_vacuum_scale_factor = 0.01, autovacuum_vacuum_threshold = 1000);
```

Autovacuum throttles itself through cost-based delay settings so it does not overwhelm I/O. If it cannot keep up, tuning its cost limits and worker count (confirm names in the documentation for your version) is the next lever. Disabling autovacuum is almost never right.

**Transaction ID wraparound.** Transaction IDs are 32-bit, so only about two billion transactions can be "in the past" relative to any given one. To keep old tuples visible forever, vacuum **freezes** tuples old enough, marking them as visible to all future transactions regardless of ID. If freezing falls too far behind, PostgreSQL first forces aggressive anti-wraparound autovacuums (triggered when a table's age passes `autovacuum_freeze_max_age`, default 200 million), and in the last resort refuses new write transactions to prevent data loss. Modern versions add a failsafe that makes vacuum skip nonessential work when age becomes critical. Causes of real incidents: a long-running transaction or stuck replication slot blocking freezing, autovacuum disabled or starved, and a huge table that never finishes vacuuming. Monitor `age(datfrozenxid)` per database and `age(relfrozenxid)` per table.

**HOT updates and fillfactor.**
- A **heap-only tuple (HOT) update** happens when (a) no column used by any index is changed, and (b) the new tuple version fits on the **same page** as the old one. PostgreSQL then creates the new version without touching any index, and later page pruning can reclaim the old version cheaply without a full vacuum. HOT updates dramatically cut write amplification and bloat.
- **fillfactor** sets how full pages are packed on insert (default 100 for tables). Setting a lower value, for example 80 to 90, reserves room on each page so updates can stay on the page and be HOT. It is useful on update-heavy tables, at the cost of a larger table.
- Corollary: indexing a frequently updated column (such as `updated_at` or `status`) disables HOT for those updates. Index only what queries need.

**Queue-like and hot-row tables.** A table where rows are inserted, updated a few times and deleted quickly generates dead tuples at a very high rate. It needs aggressive per-table autovacuum settings, a lower fillfactor if updated in place, no long transactions, and often partitioning so old data is dropped by `DROP` or `DETACH` instead of `DELETE`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Is vacuum keeping up | `pg_stat_user_tables` (`n_dead_tup`, `last_autovacuum`, `last_autoanalyze`, `autovacuum_count`) | Rank tables by dead tuples and staleness |
| Live progress | `pg_stat_progress_vacuum`, vacuum activity in `pg_stat_activity` | Tell whether a vacuum is stuck or progressing |
| Manual runs | `VACUUM (VERBOSE, ANALYZE) table`, `ANALYZE table` | Run and read verbose output |
| Per-table tuning | `ALTER TABLE ... SET (autovacuum_*, fillfactor)`, `pg_class.reloptions` | Tune a hot table and verify the setting applied |
| Wraparound watch | `SELECT datname, age(datfrozenxid) FROM pg_database`, `age(relfrozenxid)` in `pg_class` | Alert long before the limit |
| Logging | `log_autovacuum_min_duration` | See what autovacuum does and how long it takes |
| Reclaiming space | `VACUUM FULL`, pg_repack extension | Know the lock cost of each |
| HOT effectiveness | `n_tup_upd`, `n_tup_hot_upd` in `pg_stat_user_tables` | Compute the HOT ratio for a table |

### Common mistakes

- Running `VACUUM FULL` in production hours to "fix bloat" and blocking all access.
- Leaving default scale factors on a table with hundreds of millions of rows.
- Turning off autovacuum because it "uses I/O".
- Adding an index on a frequently updated column and losing HOT updates.
- Ignoring age metrics until the database refuses writes.
- Believing plain `VACUUM` shrinks files. It mostly makes space reusable.

### Self-check

- What are the two conditions for a HOT update, and why does it matter?
- Why is the default autovacuum scale factor a problem on a very large table?
- Explain transaction ID wraparound and the failure mode you are protecting against.
- A table is 10 times larger than its live data. List causes and how you would confirm each.

---

## 4. Indexes

**In plain words.** An index is a sorted lookup list that helps the database find rows without reading the whole table. It makes reads faster but makes writes slower and uses disk space. Add an index only when a real query needs it.

### Understand

An index is a separate on-disk structure that maps column values to tuple locations (`ctid`). It speeds reads and slows writes, consumes disk and memory, and every non-HOT update must maintain it. Add an index because a measured query needs it, and drop it if usage statistics show it is not used.

**Index types and when each fits.**

| Type | Good for | Notes |
|---|---|---|
| **B-tree** (default) | equality, ranges, `ORDER BY`, prefix `LIKE 'abc%'` (with suitable collation or operator class), uniqueness | The right answer most of the time. Supports unique constraints |
| **Hash** | equality only | WAL-logged and crash-safe in modern versions, but rarely better than B-tree |
| **GIN** | many keys per row: `jsonb`, arrays, full-text `tsvector`, trigram search (`pg_trgm`) | Slower to update than B-tree. Fast lookups of "contains" style queries |
| **GiST** | geometric data, ranges, exclusion constraints, nearest-neighbour, some full-text | A framework for extensible tree indexes |
| **BRIN** | very large tables whose physical order correlates with the column (append-only timestamps, ids) | Tiny. Stores min/max per block range. Useless if data is not physically correlated |
| **SP-GiST** | partitioned search structures (some text and spatial cases) | Specialised |

**Index features.**
- **Composite (multicolumn) index.** Column order matters. A B-tree on `(a, b)` serves filters on `a`, and on `a` and `b`, and `ORDER BY a, b`, but is poor for a filter on `b` alone. Put equality columns first, then the range or sort column. Selectivity alone is not the rule.
- **Partial index.** Index only rows matching a predicate. Small and fast, and a good fit for "active" rows. The query's `WHERE` must imply the index predicate.
  ```sql
  CREATE INDEX jobs_queued_idx ON jobs (created_at) WHERE status = 'queued';
  ```
- **Covering index with `INCLUDE`.** Extra non-key columns stored in the index so a query can be answered by an **index-only scan** without visiting the heap. Index-only scans still depend on the visibility map, so a table that is rarely vacuumed gets fewer of them.
  ```sql
  CREATE INDEX orders_cust_idx ON orders (customer_id) INCLUDE (total_cents);
  ```
- **Expression index.** Index a computed value. The query must use the same expression.
  ```sql
  CREATE INDEX users_email_lower_idx ON users (lower(email));
  -- used by: WHERE lower(email) = lower($1)
  ```
- **Unique indexes** enforce uniqueness. A unique constraint creates one. Nulls are distinct by default, so several `NULL`s are allowed unless you choose otherwise in your version.
- **Foreign keys do not create an index on the referencing column.** Without one, deleting or updating a parent row scans the child table, and joins on it are slow. Index referencing columns you join or cascade on.

**Building and maintaining.**
- `CREATE INDEX` blocks writes to the table for the duration. `CREATE INDEX CONCURRENTLY` does not, at the cost of taking longer, scanning twice and being unable to run inside a transaction block. If it fails or is cancelled it leaves an **invalid** index that still slows writes, and you must drop it and retry.
- Indexes bloat too. `REINDEX CONCURRENTLY` (available in modern versions) rebuilds without blocking writes.
- Find unused indexes with `pg_stat_user_indexes.idx_scan` over a representative period, not a single quiet day, and remember replicas keep their own statistics.

**Choosing.** Start from the query: which predicates are equality, which are ranges, what is the sort, how selective is each, how many rows come back. A query returning a large fraction of a table is faster as a sequential scan, and the planner knows it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Index design | `CREATE INDEX` variants, `\d+ table`, `pg_indexes` | Pick type, column order, partial or covering form for a given query |
| Does it get used | `EXPLAIN`, `pg_stat_user_indexes` (`idx_scan`, `idx_tup_read`) | Prove an index is used or find dead ones |
| Safe builds | `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY`, `pg_stat_progress_create_index` | Build on a live table and recover from an invalid index |
| Invalid indexes | `pg_index.indisvalid` | Detect leftovers from failed concurrent builds |
| Text and JSON search | `pg_trgm` extension, GIN operator classes | Index `LIKE '%x%'` and containment queries |
| Size | `pg_relation_size('index')`, `\di+` | Weigh an index's cost against its benefit |

### Common mistakes

- Indexing every column "just in case", then suffering slow writes and no HOT updates.
- Wrong composite order, so the index cannot serve the query.
- Wrapping an indexed column in a function (`WHERE date(created_at) = ...`) so the index cannot be used. Rewrite as a range or create an expression index.
- Forgetting an index on foreign keys.
- Using plain `CREATE INDEX` on a live table and blocking writes.
- Leaving an invalid index behind after a cancelled concurrent build.
- Expecting an index on a low-selectivity column (a boolean) to help.

### Self-check

- When does the planner prefer a sequential scan even though an index exists?
- For `WHERE tenant_id = $1 AND created_at > $2 ORDER BY created_at`, what composite index would you build and why?
- What is an index-only scan and what does it need from vacuum?
- What state does a failed `CREATE INDEX CONCURRENTLY` leave behind?

---

## 5. The planner, statistics and reading EXPLAIN

**In plain words.** You tell PostgreSQL what result you want, and its planner chooses how to get it. The planner guesses how many rows each step returns, using saved statistics. When a query is slow, the guess is usually wrong, and EXPLAIN shows you where.

### Understand

**Declarative means the planner decides.** You describe the result. The **planner** enumerates ways to produce it (scan methods, join orders, join algorithms), estimates a **cost** for each from table statistics and cost constants, and runs the cheapest. Bad plans nearly always come from bad **estimates**, not bad arithmetic.

**Statistics.** `ANALYZE` samples each table and stores, per column, the fraction of nulls, number of distinct values, most common values with frequencies, a histogram, and correlation with physical order, in `pg_statistic` (viewable through `pg_stats`). The sample size is governed by `default_statistics_target` (default 100), adjustable per column. Statistics are per column, so the planner assumes columns are independent. Correlated columns (city and postcode) make it multiply selectivities and underestimate. **Extended statistics** (`CREATE STATISTICS`) can capture dependencies and multi-column distinct counts.

**Cost model.** Costs are in arbitrary units built from constants: `seq_page_cost`, `random_page_cost` (default 4.0, tuned for spinning disks, and commonly lowered on SSDs), `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`. The numbers are only meaningful in comparison with each other.

**Scan methods.**
- **Seq Scan**: read the whole table. Best when most rows are needed.
- **Index Scan**: walk the index, fetch each row from the heap. Good for few rows.
- **Index Only Scan**: answer from the index when the visibility map allows.
- **Bitmap Index Scan + Bitmap Heap Scan**: build a bitmap of matching pages from one or more indexes, then read pages in physical order. Good for medium selectivity and for combining indexes.

**Join algorithms.**
- **Nested Loop**: for each outer row, probe the inner side. Excellent when the outer side is small and the inner side has an index. Catastrophic when the outer estimate is wrong and is actually large.
- **Hash Join**: build a hash table of one side, probe with the other. Good for large, unsorted, equality joins. Spills to disk in batches when it exceeds `work_mem`.
- **Merge Join**: merge two sorted inputs. Good when inputs are already sorted (by an index) or very large.

**Reading EXPLAIN.**
- `EXPLAIN` shows the chosen plan and estimates without running the query.
- `EXPLAIN (ANALYZE, BUFFERS)` **executes the query** and adds actual rows, actual time and buffer counts. For `INSERT`, `UPDATE` or `DELETE`, wrap it in a transaction and roll back, because it really changes data:
  ```sql
  BEGIN;
  EXPLAIN (ANALYZE, BUFFERS) UPDATE orders SET status = 'x' WHERE id = 1;
  ROLLBACK;
  ```
- The plan is a tree. Read from the innermost, most indented node outward. Each node shows `cost=startup..total`, `rows=` (estimate) and, with `ANALYZE`, `actual time=first..last rows=N loops=L`. **Actual time and rows are per loop**, so multiply by `loops` to get the node total.
- **Compare estimated rows to actual rows at each node.** A gap of an order of magnitude or more is where the plan went wrong. Causes: stale statistics (run `ANALYZE`), correlated columns (extended statistics), expressions the planner cannot estimate (create an expression index or restructure), parameter values with skewed distributions, and function calls that hide selectivity.
- Other signals: `Rows Removed by Filter` (a large number means a scan reads far more than it returns, so an index is missing or unsuitable), `Sort Method: external merge` (spilled to disk, `work_mem` too small), `Buffers: shared read` versus `shared hit` (cache misses), `Heap Fetches` on an index-only scan (visibility map is stale), a Nested Loop whose inner side has a huge `loops` count.
- `Planning Time` and `Execution Time` are reported at the end. A fast execution with large planning time suggests very many partitions or complex joins.

**Prepared statements and plan choice.** A prepared statement may use a generic plan after several executions instead of planning per parameter value. Skewed data can make a generic plan bad for some values. `plan_cache_mode` (values `auto`, `force_custom_plan`, `force_generic_plan`; default `auto`; added in PostgreSQL 12) can force the behaviour.

**What not to do.** Do not "fix" plans by disabling planner settings (`enable_seqscan = off`) in production. Use them only as a diagnostic to see what the planner would otherwise do. Fix the statistics, the index or the query. Query hints do not exist in core PostgreSQL.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Reading plans | `EXPLAIN`, `EXPLAIN (ANALYZE, BUFFERS)`, `FORMAT JSON` for tools | Find the node where estimate and actual diverge, and say why |
| Statistics | `ANALYZE`, `pg_stats`, `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS`, `CREATE STATISTICS` | Inspect what the planner believes and improve it |
| Slow query capture | `log_min_duration_statement`, `auto_explain` extension, `pg_stat_statements` | Find slow queries and get their plans without guessing |
| Plan visualisers | Online and desktop EXPLAIN visualisers (treat as aids, not sources of truth) | Spot expensive nodes quickly |
| Planner constants | `random_page_cost`, `effective_cache_size`, `default_statistics_target` | Know what each changes and why |

### Common mistakes

- Running `EXPLAIN ANALYZE` on a destructive statement without a rollback.
- Reading only the total cost and ignoring estimate versus actual rows.
- Forgetting per-loop semantics and misjudging a nested loop node.
- Benchmarking on a tiny dev table. Plans depend on data volume and distribution.
- Disabling planner features in production instead of fixing the cause.
- Never running `ANALYZE` after a bulk load.

### Self-check

- A query was fast last week and is slow today with no code change. List what changed in the planner's world.
- The plan shows `rows=10` estimated and `rows=2,000,000` actual under a nested loop. What happened and what do you do?
- What does `loops=1000` on an inner node mean for reading `actual time`?
- When is a hash join preferred over a nested loop?

---

## 6. Locking

**In plain words.** A lock stops two transactions from changing the same thing at the same time. Some locks make other queries wait in a line. One slow lock request can block a whole table, so you set time limits.

### Understand

**Two layers.** PostgreSQL locks both tables and rows. MVCC means plain `SELECT` never blocks on row locks, but writes, DDL and explicit locking do.

**Row-level locks.** Taken implicitly by `UPDATE` and `DELETE`, or explicitly with `SELECT ... FOR`:
- `FOR UPDATE`: strongest. Blocks other updates, deletes and locks.
- `FOR NO KEY UPDATE`: like `FOR UPDATE` but allows foreign-key checks to proceed. This is what ordinary `UPDATE` takes when no key column changes.
- `FOR SHARE`, `FOR KEY SHARE`: weaker locks, the latter taken by foreign-key checks on the parent.
- Modifiers: `NOWAIT` (error immediately if locked) and `SKIP LOCKED` (silently skip locked rows).

Row locks are stored in the tuple (in `xmax`), not in a lock table, so locking many rows does not exhaust memory, but it does write to the heap.

**Table-level lock modes** (weakest to strongest): `ACCESS SHARE`, `ROW SHARE`, `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE`. Mental model:
- `SELECT` takes `ACCESS SHARE`. `INSERT/UPDATE/DELETE` take `ROW EXCLUSIVE`. Neither conflicts with the other.
- `CREATE INDEX` (non-concurrent) takes `SHARE`, which blocks writes. `CREATE INDEX CONCURRENTLY` and plain `VACUUM` take `SHARE UPDATE EXCLUSIVE`.
- Most `ALTER TABLE` forms, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL` take `ACCESS EXCLUSIVE`, which conflicts with everything including `SELECT`.

**The lock queue effect (the migration outage).** Lock requests queue. If a long-running transaction holds a lock, and an `ALTER TABLE` needing `ACCESS EXCLUSIVE` waits behind it, every later query on that table, even a plain `SELECT`, queues behind the `ALTER`. A migration that "only waits" can therefore freeze the whole table. The defence is `SET lock_timeout` so the migration gives up and retries (chapter 13).

**Deadlocks.** Transaction A holds a lock B wants, and B holds one A wants. PostgreSQL detects this after `deadlock_timeout` (default 1 second) and aborts one transaction with SQLSTATE `40P01`. Prevention: acquire locks in a consistent order (for example update rows sorted by id), keep transactions short, and make the application retry on `40P01`.

**Advisory locks.** Application-defined locks on arbitrary integer keys, not tied to any table. Use them for "only one worker runs this job" or cross-row mutual exclusion.
- Session level (`pg_advisory_lock`) lasts until released or the session ends. It is unsafe with transaction-mode pooling (chapter 11).
- Transaction level (`pg_advisory_xact_lock`) releases automatically at commit or rollback and is the safer default.
- `pg_try_advisory_xact_lock` returns immediately with a boolean instead of waiting.

**Queue and claim pattern with `FOR UPDATE SKIP LOCKED`.** Many workers must take distinct jobs without blocking each other or double-processing. `SKIP LOCKED` lets each worker lock the next unlocked row:

```sql
UPDATE jobs
SET status = 'running', started_at = now()
WHERE id = (
  SELECT id FROM jobs
  WHERE status = 'queued'
  ORDER BY id
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
RETURNING *;
```

Pair it with a partial index on queued rows, a visibility timeout or heartbeat so jobs held by crashed workers are reclaimed, and aggressive autovacuum on the table (chapter 3). It suits modest throughput. A dedicated queue is better when you need very high rates or complex scheduling.

**Timeouts as safety nets.**
- `lock_timeout`: give up waiting for a lock (SQLSTATE `55P03`).
- `statement_timeout`: cancel a statement that runs too long.
- `idle_in_transaction_session_timeout`: kill sessions that sit idle inside a transaction.
Set them per role or per session for application users, and tighter in migration sessions.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Who is waiting | `pg_stat_activity` (`wait_event_type`, `wait_event`, `state`), `pg_locks`, `pg_blocking_pids(pid)` | Find the blocker at the head of a lock queue |
| Lock modes | The lock conflict table in the official explicit-locking documentation | Predict what an `ALTER TABLE` will block |
| Cancelling | `pg_cancel_backend(pid)`, `pg_terminate_backend(pid)` | Cancel a statement before resorting to terminating a backend |
| Explicit locking | `SELECT ... FOR UPDATE [NOWAIT \| SKIP LOCKED]`, advisory lock functions | Implement a safe job-claim query |
| Logging | `log_lock_waits`, `deadlock_timeout` | Get lock waits and deadlock detail into the log |

A starting query for blockers (confirm columns for your version):

```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, state, wait_event_type, wait_event,
       now() - xact_start AS xact_age, left(query, 80) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Common mistakes

- Running `ALTER TABLE` on a busy table with no `lock_timeout`.
- Locking rows in inconsistent order across code paths, producing deadlocks.
- `SELECT ... FOR UPDATE` held across slow application work.
- Using session advisory locks through a transaction pooler.
- Treating a deadlock as a bug to eliminate rather than something to retry safely.
- `SKIP LOCKED` without a plan for stuck or crashed workers.

### Self-check

- Why can a single waiting `ALTER TABLE` stall all reads of a table?
- How does `SKIP LOCKED` let multiple workers claim jobs safely?
- How do you prevent deadlocks between two code paths that update the same set of rows?
- What is the difference between session and transaction advisory locks?

---

## 7. Transactions and isolation levels

**In plain words.** A transaction groups several steps so they all succeed or all fail. The isolation level decides how much one transaction can see of the changes made by others. Stronger levels are safer but can fail, so your code must try again.

### Understand

**What PostgreSQL actually offers.** The default is **Read Committed**. PostgreSQL implements three distinct behaviours: `READ COMMITTED`, `REPEATABLE READ` and `SERIALIZABLE`. `READ UNCOMMITTED` is accepted but behaves as Read Committed. Dirty reads never occur.

**Read Committed (default).** Each **statement** sees a snapshot taken at its start. Two statements in the same transaction can see different data if others committed in between (non-repeatable reads). Concurrent updates: if a statement tries to update a row another transaction has already modified and committed, it waits, then **re-evaluates its `WHERE` condition against the newest row version** and proceeds if it still matches. This makes `UPDATE ... SET n = n + 1` safe but lets read-modify-write cycles in application code lose updates:

```
-- unsafe in Read Committed: two requests read balance=100, both write 90
SELECT balance FROM accounts WHERE id = 1;   -- app computes 100 - 10
UPDATE accounts SET balance = 90 WHERE id = 1;
```

Fix by doing the arithmetic in SQL (`SET balance = balance - 10 WHERE balance >= 10`), by `SELECT ... FOR UPDATE`, by optimistic concurrency with a version column, or by a stronger isolation level.

**Repeatable Read.** The whole transaction uses **one snapshot** taken at its first query. In PostgreSQL this is snapshot isolation, and it also prevents phantom reads (stronger than the SQL standard requires). If the transaction tries to update or lock a row changed by a concurrent committed transaction since its snapshot, it fails with SQLSTATE `40001` (`serialization_failure`, "could not serialize access due to concurrent update"). It is excellent for consistent multi-query reports. It still permits **write skew**: two transactions each read overlapping data and write disjoint rows, producing a result no serial order would give (for example two on-call doctors both going off call).

**Serializable.** Uses **serializable snapshot isolation (SSI)**. PostgreSQL tracks read/write dependencies between concurrent transactions and aborts one when it detects a pattern that could not occur in any serial order. Results are as if transactions ran one at a time. The price: some transactions fail with `40001`, and it can raise false positives, so the application **must** retry. Predicate locks it tracks are visible in `pg_locks` as `SIReadLock`.

**Retry is part of the contract.** Under Repeatable Read and Serializable, a correct application wraps the whole transaction in a retry loop for `40001` (and for `40P01` deadlock). The retry must re-run the entire transaction logic including reads, must be bounded with backoff, and must not repeat non-idempotent side effects (emails, payments) from inside the transaction. Do side effects after commit, or use an outbox table (a table of pending messages, written in the same transaction and sent later).

**Choosing.**
- Read Committed with atomic SQL updates and explicit locks covers most OLTP code.
- Repeatable Read for read-only consistent snapshots and reports, with retry if it writes.
- Serializable when invariants span multiple rows and are hard to express as constraints or locks, accepting retries.
- Constraints (unique, exclusion, foreign key, check) are often the best concurrency tool, since the database enforces them under any isolation level.

**Other transaction facts.**
- DDL is transactional in PostgreSQL. A failed migration can roll back completely, except for statements that cannot run in a transaction (`CREATE INDEX CONCURRENTLY`).
- `SAVEPOINT` allows partial rollback. An error aborts the whole transaction until rolled back to a savepoint or fully rolled back ("current transaction is aborted, commands ignored").
- `INSERT ... ON CONFLICT DO UPDATE` (upsert) is the atomic way to "insert or update". `DO NOTHING` plus `RETURNING` is a common idempotency tool. Concurrent upserts rely on a unique index.
- `READ ONLY` and `DEFERRABLE` transactions exist for long Serializable reports (confirm in documentation).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Setting level | `BEGIN ISOLATION LEVEL ...`, `SET TRANSACTION`, `default_transaction_isolation` | Choose per transaction |
| Reproducing anomalies | Two `psql` sessions side by side | Demonstrate lost update, non-repeatable read and write skew |
| Error handling | SQLSTATE codes `40001`, `40P01`, `23505` (unique violation), `55P03` | Map codes to retry or fail decisions |
| Atomic writes | `UPDATE ... SET x = x + 1`, `ON CONFLICT`, `RETURNING`, optimistic version columns | Avoid read-modify-write races |
| Idempotency and outbox | Unique keys on request ids, outbox table pattern | Make retries safe |

### Common mistakes

- Assuming `BEGIN` alone gives repeatable reads. It does not under the default level.
- Using Serializable or Repeatable Read without a retry loop, then seeing sporadic 500s.
- Retrying only the failing statement instead of the whole transaction.
- Performing external side effects inside a transaction that may be retried.
- Holding transactions open across slow work.
- Treating gaps in sequences or retries as data corruption.

### Self-check

- A balance is read, decremented in application code and written back. Show how two concurrent requests lose an update under Read Committed, and give three fixes.
- What does PostgreSQL's Repeatable Read prevent that the SQL standard does not require, and what anomaly remains?
- What does a `40001` error mean and what must the application do?
- Why can DDL be rolled back in PostgreSQL, and what is an exception?

---

## 8. Data modelling features: constraints, generated columns, JSONB, full-text search, partitioning

**In plain words.** This chapter covers tools for storing data correctly. Constraints are rules the database enforces. JSONB stores flexible JSON data, full-text search finds words in text, and partitioning splits a big table into smaller parts.

### Understand

**Constraints are the last line of defence.** Application checks race. Database constraints do not.
- `NOT NULL`, `CHECK`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY` (with `ON DELETE` actions: `NO ACTION`, `RESTRICT`, `CASCADE`, `SET NULL`, `SET DEFAULT`).
- **Exclusion constraints** (with GiST) forbid overlapping ranges, for example double-booked rooms.
- Constraints can be `DEFERRABLE`, checked at commit instead of per statement.
- Prefer `bigint` identity columns (`GENERATED ALWAYS AS IDENTITY`) or UUIDs for keys. Choose time-ordered UUIDs where index locality matters, because fully random UUID primary keys scatter inserts across the index.
- Prefer `timestamptz` over `timestamp` for instants. Store money as integer minor units or `numeric`, never `float`. Use `text` rather than length-limited varchar unless the limit is a real rule.

**Generated columns.** `GENERATED ALWAYS AS (expression) STORED` computes a column from others at write time and stores it. It can be indexed. Typical use: a normalised search key or a `tsvector`. PostgreSQL 18 added `VIRTUAL` generated columns (computed when read) and made them the default kind, so write `STORED` explicitly. Versions 12 to 17 support only `STORED`. Indexing a virtual column directly is not supported as far as we know (unverified, so check `CREATE INDEX` for your version). Use `STORED` or an expression index.

**JSONB.** A binary, indexable JSON type. Use it for genuinely semi-structured or sparsely populated data (third-party payloads, flexible attributes). Do not use it to avoid schema design, because you lose type checking, foreign keys and planner statistics on the inner fields.
- Operators: `->` (json), `->>` (text), `#>` and `#>>` (path), `@>` (contains), `?` (key exists), plus SQL/JSON path functions such as `jsonb_path_query`.
- Index with **GIN** for containment and key-exists (`jsonb_ops` supports more operators, `jsonb_path_ops` is smaller and faster for `@>` only). For one frequently filtered field, a B-tree **expression index** on `(data->>'status')` is often better.
- Large values are moved out of line (**TOAST**, PostgreSQL's storage for big values) and compressed. Updating any part of a JSONB value rewrites the whole value, so very large, frequently updated documents are expensive.
- Cast carefully: `data->>'age'` is text. Compare with a cast and index the same expression.

**Full-text search basics.**
- `tsvector` is a normalised, stemmed list of lexemes. `tsquery` is a search expression. `@@` matches them.
- Build with `to_tsvector('english', text)`. Parse user input with `websearch_to_tsquery` (web-search style syntax, safe on arbitrary input) or `plainto_tsquery`. Rank with `ts_rank`.
- Index with GIN, usually on a stored generated `tsvector` column:
  ```sql
  ALTER TABLE articles ADD COLUMN search tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))) STORED;
  CREATE INDEX articles_search_idx ON articles USING GIN (search);
  SELECT id FROM articles WHERE search @@ websearch_to_tsquery('english', 'postgres vacuum');
  ```
- Good enough for many products. It lacks the relevance tuning, typo tolerance and faceting of dedicated search engines. For typo-tolerant or `LIKE '%x%'` matching, look at `pg_trgm`.

**Partitioning.** Declarative partitioning splits one logical table into physical partitions by `RANGE`, `LIST` or `HASH`.
- Benefits: **partition pruning** (queries filtering on the partition key touch only relevant partitions), cheap bulk deletion (`DROP` or `DETACH` an old partition instead of a huge `DELETE` that creates bloat), smaller indexes and per-partition maintenance.
- Constraints: a primary or unique key must include the partition key. There are no global unique indexes across partitions. Queries not filtering on the key scan all partitions. Thousands of partitions raise planning time.
- Partitions must be created ahead of time (by a job or an extension such as pg_partman). Detaching can be done concurrently in modern versions (confirm syntax for your version).
- Partition only when the table is large enough that pruning and lifecycle management matter. Typical case: time-series or event tables with retention.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Constraints | `ALTER TABLE ... ADD CONSTRAINT`, `\d+`, `pg_constraint` | Express invariants in the schema |
| JSONB | `->`, `->>`, `@>`, `jsonb_path_query`, GIN and expression indexes | Query and index semi-structured data correctly |
| Full-text | `to_tsvector`, `websearch_to_tsquery`, `ts_rank`, `pg_trgm` | Build a working search with a GIN index |
| Partitioning | `PARTITION BY RANGE`, `CREATE TABLE ... PARTITION OF`, `DETACH PARTITION`, `EXPLAIN` to verify pruning | Design a time-partitioned table with retention |
| Types | `timestamptz`, `numeric`, `uuid`, identity columns | Pick correct types by default |

### Common mistakes

- Missing constraints because "the application validates it".
- Storing everything in a JSONB column and then needing relational queries.
- Querying JSONB with operators the GIN index does not support.
- Partitioning a small table, or choosing a key queries never filter on.
- Using `timestamp` without time zone for instants.
- Using `float` for money.

### Self-check

- When is a unique constraint better than an application-level existence check?
- Which operators does a default GIN JSONB index serve, and when would you use an expression index instead?
- What does partition pruning need in the query to work?
- Why must the partition key be part of the primary key?

---

## 9. Roles, privileges and row-level security

**In plain words.** A role is a database account with a list of allowed actions. Give your application the smallest set of rights it needs. Row-level security adds a rule that hides rows, for example rows that belong to another customer.

### Understand

**Roles.** Users and groups are both **roles**. A role with `LOGIN` can connect. Roles can be members of other roles and inherit their privileges. Applications should connect as a dedicated, low-privilege role, not as a superuser or the object owner.

**Privilege model.** Privileges (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `USAGE`, `CREATE`, `EXECUTE` and others) are granted on databases, schemas, tables, sequences and functions with `GRANT` and removed with `REVOKE`.
- A common layout: an **owner** role runs migrations and owns objects, a **runtime** role has only the DML privileges the app needs, and a **read-only** role serves reporting and analysts.
- `ALTER DEFAULT PRIVILEGES` makes future objects created by the owner automatically grant the runtime role access, which avoids "permission denied for table" after each migration.
- Newer versions no longer let all users create objects in the `public` schema by default. Older databases may still allow it, so `REVOKE CREATE ON SCHEMA public FROM PUBLIC` is a common hardening step. Confirm for your version.
- Authentication and network access are controlled by `pg_hba.conf` (which hosts, users and databases may connect, with which method and whether TLS is required). Prefer SCRAM password authentication and TLS.

**Row-level security (RLS)** lets the database filter and check rows per role or per session context. It is a strong backstop for **multi-tenant** applications where every table has a `tenant_id`, because a forgotten `WHERE tenant_id = ...` in application code cannot leak data.

```sql
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE invoices FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON invoices
  USING (tenant_id = current_setting('app.tenant_id')::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

- `USING` filters which existing rows are visible or modifiable. `WITH CHECK` validates new or changed rows.
- **Who bypasses RLS:** superusers, roles with `BYPASSRLS`, and the **table owner** unless `FORCE ROW LEVEL SECURITY` is set. The application must therefore connect as a role that is neither owner nor superuser, or the policy does nothing. This is a frequent silent failure.
- With no policy and RLS enabled, access is denied by default for non-bypassing roles.
- **Setting the tenant per request.** The usual pattern sets a custom setting inside the transaction so it is scoped correctly under connection pooling:
  ```sql
  BEGIN;
  SELECT set_config('app.tenant_id', $1, true);  -- true = local to this transaction
  -- queries here are filtered by the policy
  COMMIT;
  ```
  `SET LOCAL` (or `set_config(..., true)`) reverts at transaction end, so it is safe with transaction-mode pooling. A plain session-level `SET` leaks to the next client that reuses the connection.
- Costs: the policy expression is added to every query, so indexes should lead with `tenant_id`. Policies calling expensive or non-leakproof functions can defeat index use. Test with `EXPLAIN`.
- RLS complements, not replaces, application authorisation. Decide whether you use it as defence in depth or as the primary tenant boundary, and test both the allow and the deny paths.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Roles and grants | `CREATE ROLE`, `GRANT`, `REVOKE`, `ALTER DEFAULT PRIVILEGES`, `\du`, `\dp`, `\ddp` | Build owner, runtime and read-only roles |
| Access control | `pg_hba.conf`, `SHOW hba_file`, `pg_hba_file_rules` view | Explain why a connection was accepted or rejected |
| RLS | `CREATE POLICY`, `ENABLE`/`FORCE ROW LEVEL SECURITY`, `\d+ table`, `pg_policies` | Implement tenant isolation and prove it with tests |
| Session context | `set_config(name, value, true)`, `current_setting(name, true)` | Pass tenant identity safely per transaction |

### Common mistakes

- Application connects as the table owner or a superuser, so RLS never applies.
- Session-level `SET` for tenant context under a pooler.
- Granting broad privileges to `PUBLIC` or using one all-powerful role everywhere.
- Forgetting sequence privileges (`USAGE` on sequences) for inserts.
- No test proving that tenant A cannot read tenant B.
- RLS policies that prevent index use, discovered only under load.

### Self-check

- Who bypasses row-level security, and how do you make the table owner subject to it?
- Why must tenant context be set with transaction-local scope under a pooler?
- What problem does `ALTER DEFAULT PRIVILEGES` solve?
- What is the difference between `USING` and `WITH CHECK`?

---

## 10. Replication, backup and point-in-time recovery

**In plain words.** Replication keeps a live copy of the database on another server, so you can switch if the first one fails. A replica is not a backup, because a mistake is copied too. A backup plus saved log files lets you restore the database to a chosen moment.

### Understand

**Why replicate.** For availability (a standby can be promoted) and to scale reads. Replication is not a backup: a `DROP TABLE` or bad `UPDATE` replicates immediately.

**Physical streaming replication.** A standby receives the primary's WAL stream and replays it, producing a byte-identical copy of the whole cluster. A standby that accepts read-only queries is a **hot standby**.
- **Asynchronous** (default): the primary does not wait for the standby. Low latency, but a failover can lose recently committed transactions (replication lag).
- **Synchronous** (`synchronous_standby_names`): commit waits for the standby, trading latency and availability for no data loss. Know the failure mode: if the only synchronous standby is down, writes block unless configured otherwise.
- **Replication slots** make the primary retain WAL until a consumer has received it, so a standby cannot fall too far behind. The danger: an **inactive or abandoned slot retains WAL forever and fills the disk**. Monitor slots and set limits where available.
- **Read replicas and lag.** Reads from a replica can be stale. Read-your-own-writes needs routing (read from the primary after a write, or wait for the replica to reach a given LSN). Long queries on a hot standby can conflict with replay (query cancellations), with `hot_standby_feedback` as a trade-off that can bloat the primary.
- **Failover** means promoting a standby and repointing clients. Tooling (managed platforms, Patroni and similar) automates it. The hard parts are avoiding **split brain** (two primaries) and deciding how much data loss is acceptable.

**Logical replication.** Publishes row changes (`INSERT/UPDATE/DELETE`) of selected tables through **publications**, consumed by **subscriptions**. Requires `wal_level = logical`. Uses: zero-downtime major version upgrades, replicating a subset of tables, feeding other systems, change data capture through logical decoding.
- It replicates data changes, **not** schema changes (DDL) and, by default, not sequences. Apply schema changes to the subscriber yourself and sync sequences at cutover.
- Tables need a replica identity (normally the primary key) for updates and deletes.
- Initial table copy plus streaming changes means a lagging subscriber again holds WAL via its slot.

**Backups.**
- **Logical backup:** `pg_dump` exports one database as SQL or an archive using a consistent snapshot without blocking writers. Custom (`-Fc`) and directory (`-Fd`, which allows parallel `-j`) formats restore selectively with `pg_restore`. Good for small to medium databases, migrations between versions and single-table recovery. Slow to restore at large scale. It does not dump roles (use `pg_dumpall --globals-only`).
- **Physical backup:** `pg_basebackup` or file-level snapshots copy the data directory. Combined with continuous **WAL archiving** (`archive_mode`, `archive_command`), this enables **point-in-time recovery (PITR)**: restore the base backup, then replay archived WAL up to a chosen time or transaction (`restore_command` and `recovery_target_time` settings plus a `recovery.signal` file in modern versions). PITR is how you undo a destructive mistake made at 14:03.
- Dedicated backup tools (pgBackRest, Barman, WAL-G) manage base backups, retention, compression and parallelism. Managed services provide automated backups and PITR windows. Know what yours guarantees.
- **A backup you have not restored is a hope, not a backup.** Schedule restore tests, time them (the real RTO), and record recovery point (RPO). Keep copies off the primary's failure domain and protect them from the credentials that could delete production.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Logical backup | `pg_dump -Fc`, `-Fd -j N`, `pg_dumpall --globals-only`, `pg_restore -d`, `--list`, `--clean` | Back up and selectively restore a database or table |
| Physical backup | `pg_basebackup`, WAL archiving, `restore_command`, `recovery_target_time` | Perform a PITR in a lab |
| Streaming health | `pg_stat_replication` (sent, write, flush, replay LSNs and lag columns), `pg_replication_slots`, `pg_last_wal_replay_lsn()` | Measure lag in bytes and time and find stuck slots |
| Logical replication | `CREATE PUBLICATION`, `CREATE SUBSCRIPTION`, `pg_stat_subscription` | Replicate a table between two instances |
| Backup tools | pgBackRest, Barman, WAL-G | Know what they add over raw tools |

### Common mistakes

- Treating a replica as a backup.
- Never testing a restore.
- Abandoned replication slots filling the primary's disk.
- Assuming logical replication copies schema changes and sequences.
- Reading from replicas where staleness breaks correctness (read-after-write).
- Backups stored where the same credentials can delete them.

### Self-check

- What is the difference between replication and backup, with an example failure each one handles?
- How does PITR work and what two things must you have?
- Why can an inactive replication slot take a database down?
- What does logical replication not replicate?

---

## 11. Connection management and PgBouncer

**In plain words.** Every connection costs the server memory and work, so you cannot open thousands. A pooler such as PgBouncer lets many clients share a few real connections. Pooling changes some behaviour, so a few session features stop working.

### Understand

**Why connections are expensive.** Each connection is a backend process (chapter 1) with its own memory, plus per-connection overhead in lock tables and snapshot computation. Hundreds of connections mostly idle still cost memory, and thousands of active ones thrash CPU and cache. `max_connections` (default 100) is a hard cap. Raising it to thousands is rarely the answer, because the database can only run roughly as many queries in parallel as it has cores and I/O capacity. Extra connections queue inside the server instead of in the pool.

**Application pooling.** Libraries (node-postgres `Pool`, postgres.js, Prisma and others) keep a pool per process. The total is `processes x pool size`. With many app instances, serverless functions or autoscaling, this exceeds what the database can handle. Size pools by `database capacity / number of instances`, and always set connect, idle and statement timeouts.

**External poolers.** A pooler such as **PgBouncer** sits between applications and PostgreSQL. Thousands of cheap client connections share a small number of real server connections. Key settings: `pool_mode`, `max_client_conn`, `default_pool_size`, and per-database or per-user pool limits (confirm current names in the PgBouncer documentation).

**Pool modes.**

| Mode | Server connection assigned | Result |
|---|---|---|
| **session** | for the whole client session | Full feature compatibility, little multiplexing |
| **transaction** | for the duration of one transaction | High multiplexing. Most common for web backends. Breaks session-level features |
| **statement** | for one statement | Multi-statement transactions are not allowed. Rarely used |

**What breaks in transaction pooling.** Anything stored in the server session may land on a different server connection next time:
- **Session state**: `SET` (non-local), `SET ROLE`, session settings, `search_path` changes, temporary tables and `DISCARD`/`RESET` assumptions. Use `SET LOCAL` inside a transaction instead.
- **Prepared statements**: classic protocol-level prepared statements are tied to one server connection, so the next transaction may reach a server that never saw `PREPARE`, giving errors such as "prepared statement does not exist". Remedies: disable prepared statements in the client driver (for example the `prepare: false` option of postgres.js, or avoid named statements in node-postgres), or use PgBouncer 1.21.0 or later, which tracks protocol-level named prepared statements when `max_prepared_statements` is non-zero (default 200 in current docs, and 0 disables it). SQL-level `PREPARE`, `EXECUTE` and `DEALLOCATE` commands are not tracked and are passed straight to the server.
- **`LISTEN`/`NOTIFY`**: a listener must stay on one connection to receive notifications. Use a separate direct (or session-mode) connection for listeners.
- **Session-level advisory locks**: acquired on one server connection and released on another or never. Use transaction-level advisory locks, or a direct connection.
- **`WITH HOLD` cursors, `pg_dump`, and migration tools** that rely on session features or advisory locks should use a direct connection.
- **Lost row-level-security context** if set with session-level `SET` (chapter 9).

**Practical architecture.**
- Runtime traffic through the pooled endpoint in transaction mode.
- Migrations, `LISTEN`, long admin sessions and backups on a **direct** connection.
- Cap pool size per instance, and monitor waiting clients (`SHOW POOLS` in the PgBouncer admin console).
- Keep transactions short. A pooled server connection held by an idle transaction starves everyone else.

**Failure modes to recognise:** `FATAL: sorry, too many clients already` or `remaining connection slots are reserved` (hit `max_connections`), client-side pool timeouts (pool exhausted by slow queries or leaked connections), and a thundering herd of reconnects after a failover or restart. Always release connections in a `finally` block.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Server connections | `pg_stat_activity` grouped by `state`, `application_name`, `usename`; `SHOW max_connections` | Explain who owns every connection |
| Pooler health | PgBouncer admin console `SHOW POOLS`, `SHOW STATS`, `SHOW CLIENTS` | Spot waiting clients and saturated pools |
| Client pools | node-postgres `Pool` options, postgres.js options | Size pools and set timeouts deliberately |
| Naming connections | `application_name` in connection settings | Attribute load to a service |
| Safe timeouts | `statement_timeout`, `idle_in_transaction_session_timeout` per role | Prevent a leaked session from holding resources |

### Common mistakes

- Each of many serverless or autoscaled instances opening a large pool.
- Raising `max_connections` instead of pooling.
- Transaction pooling plus session-level `SET`, named prepared statements, `LISTEN` or session advisory locks.
- Running migrations through the transaction pooler.
- Leaking connections by not releasing them on error paths.
- Holding a transaction open across external calls, which pins a pooled server connection.

### Self-check

- Why is a PostgreSQL connection costly, and what is the right fix for "too many connections"?
- Compare session, transaction and statement pooling.
- List four features that break under transaction pooling and the workaround for each.
- Why should migrations and `LISTEN` use a direct connection?

---

## 12. Managed and serverless PostgreSQL

**In plain words.** A managed service runs PostgreSQL for you, but you get fewer controls. Serverless services can pause when idle, so the first request after a pause is slow. Use the pooled address for the application and the direct address for migrations.

### Understand

**Managed PostgreSQL** (a cloud provider's hosted service) handles provisioning, backups, patching, replication and failover. You trade control for convenience, and you should know what you gave up:
- **Limited superuser.** No OS access, restricted settings, a curated list of allowed extensions and provider-specific roles. Check extension support (`pg_stat_statements`, `pg_trgm`, `pgcrypto` and others) before choosing a provider.
- **Parameter tuning** goes through the provider's parameter mechanism, and some settings need a restart.
- **Storage and IOPS limits** are provisioned or burstable. Hitting an I/O credit ceiling looks like sudden slowness (compare chapter 6 of the Linux guide, disk limits).
- **Failover and maintenance windows** cause brief connection drops. Clients need reconnect and retry logic with backoff.
- **Backups and PITR windows** have retention you must read, and restore usually creates a new instance, so test it.
- **Major version upgrades** vary by provider and may involve downtime or logical replication.
- **Connection limits** are often derived from instance size and are low on small tiers.

**Serverless PostgreSQL** platforms add elastic compute, branching and **scale-to-zero**. Considerations:
- **Pooled versus direct connection strings.** Providers typically expose two hostnames or ports: a **pooled** endpoint (a built-in PgBouncer-like layer, transaction mode) for application traffic and a **direct** endpoint to the database. Apply chapter 11 exactly: pooled for runtime queries, direct for migrations, `LISTEN`, session advisory locks and tools that need session state. Using the wrong one is a leading cause of confusing errors.
- **Scale-to-zero cold starts.** An idle compute may suspend. The first query after idle pays a resume delay, which shows up as a latency spike or a first-request timeout. Mitigations: keep a minimum compute size or disable suspend for production, set client connect timeouts above the cold-start time, and retry the first connection. Do not treat cold-start latency as an application bug.
- **Short-lived runtimes** (functions, edge workers) cannot hold TCP connection pools. Providers offer HTTP or WebSocket based drivers for this. They have their own limits (for example interactive transactions or session features may be restricted), so read the provider's documentation and the matching Drizzle driver notes before depending on them.
- **Branching** gives cheap copies of the database for previews and migration tests. Use it to rehearse migrations against realistic data, but never treat a branch's behaviour as proof of production lock behaviour under load.
- **Cost model.** You may pay by compute time, storage and egress. Chatty ORMs and polling loops cost money and keep a scaled-to-zero database awake.
- **Region and latency.** Put the database in the same region as the compute. A cross-region round trip per query multiplies across N+1 patterns.

**Provider-agnostic checklist before committing:** supported major version and upgrade path, extension list, connection limit and pooling story, PITR retention and restore time, failover behaviour and expected downtime, read replica support, observability (is `pg_stat_statements` available), network isolation and TLS, and an exit path (can you take a `pg_dump` or replicate out).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Choosing endpoints | Provider console and docs, connection string parameters | Say which URL each workload uses and why |
| Capability check | `SELECT * FROM pg_available_extensions`, `SHOW` for key settings | Verify what a managed instance supports |
| Resilience | Client retry with backoff, connect timeout, pool `error` handlers | Survive failover and cold start |
| Rehearsal | Branch or restore to a scratch instance, run migrations there | Test a migration before production |

### Common mistakes

- Running migrations through the pooled endpoint and hitting session-feature errors or timeouts.
- Using the direct endpoint from serverless functions and exhausting connections.
- Not planning for cold-start latency on a user-facing path.
- Assuming a provider branch or replica behaves like production under write load.
- No tested exit path from the provider.

### Self-check

- Why do providers offer both pooled and direct connection strings, and which do you use for migrations?
- Your first request after idle takes 3 seconds and later ones take 20 ms. What is the likely cause and what are your options?
- What extra questions do you ask about a managed service that you would not ask about a self-hosted one?

---

## 13. Zero-downtime migrations

**In plain words.** You can change a table while the application is running if you use small, safe steps. Add the new thing first, move the data, and remove the old thing last. Always set a short lock timeout so a waiting change does not block everyone.

### Understand

**The goal.** Change the schema while the old and new application versions are both running, without long locks or a rewrite of a large table. The two enemies are **lock queues** (chapter 6) and **table rewrites or full scans under lock**.

**Expand, migrate, contract.** Split every breaking change into backward-compatible steps, each deployable independently:

1. **Expand**: add the new structure (column, table, index) without breaking existing code. Old code ignores it.
2. **Migrate**: deploy code that writes to both old and new (or reads new with fallback), and **backfill** existing rows in small batches.
3. **Contract**: after all code uses only the new structure, remove the old one in a later release.

Example: renaming a column. Do not run `RENAME`. Add the new column, dual-write, backfill, switch reads, stop writing the old column, then drop it. Rollbacks are possible at every step because old code still works.

**Operation by operation.**

- **Add a column.** Without a default, or with a constant default in modern versions (the default is stored as metadata), it is quick, but still needs a brief `ACCESS EXCLUSIVE` lock, so it still needs a `lock_timeout`. A **volatile** default (for example `random()`) forces a table rewrite. Older versions rewrote for any default, so confirm for your version.
- **Add a NOT NULL constraint.** A plain `SET NOT NULL` scans the table under `ACCESS EXCLUSIVE`. Safer pattern: add a `CHECK (col IS NOT NULL) NOT VALID`, run `VALIDATE CONSTRAINT` (which takes a weaker lock and does not block writes), then `SET NOT NULL`, which in modern versions can use the validated check to skip the scan, then drop the check. Backfill existing nulls first in batches.
- **Create an index.** `CREATE INDEX CONCURRENTLY` (outside a transaction block). Check for an invalid index afterward. Do the same for unique indexes, then attach with `ADD CONSTRAINT ... USING INDEX` where needed.
- **Add a foreign key.** `ADD CONSTRAINT ... FOREIGN KEY ... NOT VALID`, then `VALIDATE CONSTRAINT` separately.
- **Change a column type.** Often rewrites the table. Prefer add new column, dual-write, backfill, swap. Some widening changes avoid a rewrite, but verify on a copy.
- **Drop a column.** Fast (metadata only), but only after no code references it, and Drizzle or another ORM must stop selecting it first.
- **Backfills.** Update in batches of a few thousand rows by primary key range, commit between batches, sleep or throttle, and monitor replication lag and vacuum. One giant `UPDATE` holds locks, generates huge WAL and creates a lot of dead tuples at once.
- **Lock timeouts.** Run DDL with a short `lock_timeout` and retry on failure, so a migration that cannot get its lock fails fast instead of queuing every other query behind it:

```sql
SET lock_timeout = '3s';
SET statement_timeout = '60s';

ALTER TABLE orders ADD COLUMN note text;

CREATE INDEX CONCURRENTLY orders_customer_idx ON orders (customer_id);

ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers (id) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_fk;

ALTER TABLE orders ADD CONSTRAINT orders_note_nn CHECK (note IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_note_nn;
ALTER TABLE orders ALTER COLUMN note SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT orders_note_nn;
```

(Run `CREATE INDEX CONCURRENTLY` as its own statement outside any wrapping transaction, and run each statement with its own timeout if your migration tool wraps files in transactions.)

**Process rules.**
- Check for long transactions before running DDL (`pg_stat_activity.xact_start`).
- Ship schema changes separately from the application changes that depend on them.
- Make migrations **idempotent** (safe to run twice) where possible (`IF NOT EXISTS`) and never edit a migration that has already run in any shared environment.
- Rehearse on a restored copy or branch with production-like data volume, and time it.
- Know your migration tool's transaction behaviour: a tool that wraps each migration file in a transaction cannot run `CREATE INDEX CONCURRENTLY` inside it.
- Plan the rollback: most expand steps are reversible, contract steps are not, so take a backup or ensure PITR coverage before them.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Safe DDL | `lock_timeout`, `CREATE INDEX CONCURRENTLY`, `NOT VALID` / `VALIDATE CONSTRAINT` | Apply any common change without blocking traffic |
| Observation | `pg_stat_activity`, `pg_locks`, `pg_stat_progress_create_index` | Watch a migration and find what it waits on |
| Backfill | Batched `UPDATE ... WHERE id BETWEEN`, a small script with sleeps | Backfill millions of rows without stalling |
| Migration frameworks | drizzle-kit generated SQL, hand-edited SQL migrations | Edit generated migrations to be safe |
| Linting | Static migration checkers (community tools exist; evaluate them yourself) | Catch unsafe DDL in review |

### Common mistakes

- Running `ALTER TABLE` with no `lock_timeout` behind a long transaction, freezing the table.
- `ADD COLUMN ... NOT NULL DEFAULT <volatile>` on a large table.
- Plain `CREATE INDEX` on a hot table.
- One monolithic backfill `UPDATE`.
- Renaming or dropping in a single release while old code still runs.
- Letting an ORM generate and apply a destructive migration automatically in production.

### Self-check

- Describe expand, migrate and contract for splitting `full_name` into `first_name` and `last_name`.
- How do you add a `NOT NULL` constraint to a 500-million-row table with minimal locking?
- Why can an `ALTER TABLE` that is "just waiting" cause an outage, and what prevents it?
- Why can `CREATE INDEX CONCURRENTLY` not run in a transaction block?

---

## 14. Monitoring and diagnosis

**In plain words.** PostgreSQL has built-in views that show what is running now and what has happened over time. Most counters only grow, so compare two readings to see the rate. When there is a problem, collect evidence first and then act.

### Understand

**Method.** Start from symptoms users feel (latency, errors), then locate the bottleneck: CPU, I/O, memory, locks, connections or a single bad query. The Linux guide's USE method applies to the host. PostgreSQL adds its own statistics views.

**Cumulative versus current.** Statistics views are mostly **cumulative counters since the last reset**. They are meaningful only as **deltas over time**. Sample and graph them.

**Key views and extensions.**
- **`pg_stat_activity`**: one row per backend: `state` (`active`, `idle`, `idle in transaction`), `query`, `xact_start`, `query_start`, `wait_event_type`/`wait_event`, `backend_xmin`, `application_name`. First stop in any incident.
- **`pg_stat_statements`** (extension, must be loaded via `shared_preload_libraries` and created with `CREATE EXTENSION`): aggregates statistics per normalised query: `calls`, `total_exec_time`, `mean_exec_time`, `rows`, and buffer counters. Normalised means constants are replaced by placeholders so similar queries group together. Sort by `total_exec_time` to find where the database spends time overall, by `mean_exec_time` for slow individual queries, and by `calls` for chatty N+1 patterns. Column names changed in version 13 (`total_time` became `total_exec_time`), so check your version.
- **`pg_locks`** and `pg_blocking_pids()`: lock waits (chapter 6).
- **`pg_stat_user_tables` / `pg_stat_user_indexes`**: sequential versus index scans, dead tuples, vacuum times, unused indexes, HOT update counts.
- **`pg_stat_database`**: commits, rollbacks, deadlocks, temp files and bytes (a nonzero, growing temp usage suggests `work_mem` spills), cache hit counters.
- **`pg_stat_replication`** and **`pg_replication_slots`**: replication lag and retained WAL.
- **Progress views** such as `pg_stat_progress_vacuum` and `pg_stat_progress_create_index`.

**Logging.** Useful settings: `log_min_duration_statement` (log slow queries), `log_lock_waits`, `log_checkpoints`, `log_autovacuum_min_duration`, `log_temp_files`, `log_connections` and `log_disconnections` when diagnosing connection churn, and `auto_explain` to log plans of slow queries. **pgBadger** parses PostgreSQL logs into a report of slowest queries, frequency, errors, locks and connection patterns, which is valuable after the fact or when you lack an APM.

**What to alert on.**
- Connection count versus `max_connections`, and pooler client wait.
- Replication lag (bytes and time) and slot retained WAL.
- Longest transaction age and `idle in transaction` duration.
- Transaction ID age (`age(datfrozenxid)`).
- Disk free space including WAL volume, and growth rate.
- Dead tuple ratio and time since last autovacuum for key tables.
- Deadlock and error rates, checkpoint frequency, temp file growth.
- p95 and p99 query latency from `pg_stat_statements` or application tracing.
- Cache hit ratio is a weak signal on its own. Look at actual I/O wait and read counts.

**Benchmarking.** `pgbench` ships with PostgreSQL. Initialise with `pgbench -i -s <scale>`, then run with `-c` (clients), `-j` (threads) and `-T` (duration seconds). `-S` runs a select-only workload and `-f` runs a custom script file. Use it to compare configuration changes and hardware under the same conditions. It is not a model of your workload unless you write a custom script that resembles it.

**An incident routine.**
1. `pg_stat_activity`: how many active, waiting, idle in transaction? Any very old `xact_start`?
2. Blockers: `pg_blocking_pids`. Resolve with `pg_cancel_backend` before `pg_terminate_backend`.
3. `pg_stat_statements`: which queries dominate time since the incident began (use deltas).
4. Host: CPU, I/O wait, memory, disk space.
5. Recent changes: deploy, migration, traffic, a new cron job, an autovacuum anti-wraparound run.
6. Capture evidence (queries, plans, lock state) before killing anything.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Live state | `pg_stat_activity`, `pg_locks`, `\watch 2` in `psql` | Triage a live incident in minutes |
| Query statistics | `pg_stat_statements` | Rank queries by total time, mean time and calls |
| Log analysis | `log_min_duration_statement`, `auto_explain`, **pgBadger** | Produce and read a slow-query report |
| Benchmarking | **pgbench** (`-i -s`, `-c -j -T`, `-S`, `-f`) | Run a repeatable load test and compare |
| Backup tooling | **pg_dump / pg_restore** | Dump and restore as part of diagnosis and rehearsal |
| Metrics export | A metrics exporter plus a dashboard system of your choice | Graph counters as rates |

### Common mistakes

- Reading cumulative counters as if they were current rates.
- Optimising a query that is slow once and ignoring one that runs a million times.
- Killing the symptom (terminating sessions) before capturing evidence.
- Enabling `log_min_duration_statement = 0` on a busy server and flooding the disk.
- Benchmarking with defaults on a data set that fits in memory and drawing conclusions about production.
- No alert on transaction ID age or replication slot lag.

### Self-check

- How do you find the query that consumes the most total database time?
- A deploy is followed by timeouts. What do you check in `pg_stat_activity` in the first minute?
- Why are `pg_stat_*` counters only useful as deltas?
- What does a growing `temp_bytes` value suggest?

---

## 15. Drizzle ORM and drizzle-kit

**In plain words.** Drizzle lets you write database queries in TypeScript that stay close to real SQL. Its tool drizzle-kit creates migration files from your schema. You still need to understand the database, because Drizzle does not hide it.

> **Confirm in the current Drizzle documentation.** Drizzle moves quickly: configuration keys, relation definitions, driver imports and some method names have changed between versions. The snippets below show well-established shapes. Check names and import paths against the version you install before relying on them.

### Understand

**What Drizzle is.** A TypeScript ORM that is deliberately close to SQL. You define tables in TypeScript, and queries read like SQL with full type inference. It does not hide the database behind an object graph, there is no query "magic" layer such as lazy loading, and it generates SQL you can predict. Its companion, **drizzle-kit**, handles schema migrations. Its position is "a typed SQL builder with migrations", which suits engineers who already understand the database (the earlier chapters), and it does not replace that understanding.

**Schema definition.** Tables are declared with `pgTable` and column helpers from the PostgreSQL module.

```ts
import { pgTable, bigint, text, timestamp, jsonb, index, uniqueIndex } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  email: text('email').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('users_email_uq').on(t.email),
]);

export const posts = pgTable('posts', {
  id: bigint('id', { mode: 'number' }).primaryKey().generatedAlwaysAsIdentity(),
  authorId: bigint('author_id', { mode: 'number' }).notNull().references(() => users.id),
  title: text('title').notNull(),
  meta: jsonb('meta'),
}, (t) => [
  index('posts_author_idx').on(t.authorId),
]);
```

The third-argument callback returns an **array** in the current docs (older versions returned an object, so check your installed version). Identity column helpers and the available column types (`serial`, `uuid`, `numeric`, `boolean`, enums via `pgEnum`) are documented per version. Remember chapter 4: a foreign key does not create an index, so declare it yourself as above. The docs show partial indexes with `.where(sql`...`)`, expression indexes with `.using('btree', sql`lower(${t.email})`)` and `.concurrently()`. `INCLUDE` syntax is not covered on that page (unverified), so fall back to hand-written SQL when the schema DSL cannot express what you need.

**Column typing gotchas that map to PostgreSQL facts.**
- `bigint` values exceed JavaScript's safe integer range. Drizzle offers modes (`number` or `bigint`) so you choose between losing precision and using `BigInt`. Pick deliberately.
- `numeric` is returned as a string by default to preserve precision. Do not silently convert money to floats.
- `timestamp` has options for time zone and for returning `Date` or string. Use `withTimezone: true` for instants (chapter 8).
- `jsonb` is typed as unknown unless you annotate it (for example with `.$type<Shape>()`). The annotation is a compile-time claim only. Nothing validates the stored data, so validate at the boundary (for example with a schema validator).

**Inferred types.** Derive row types from the table instead of hand-writing them:

```ts
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;
```

**Connecting.** You create a client for your driver (node-postgres, postgres.js or a serverless driver), then pass it to `drizzle(...)`. Import paths depend on the driver, so confirm them. The driver choice determines pooling and prepared statement behaviour: under a transaction pooler (chapter 11) disable named prepared statements in the driver, and confirm the recommended option in the Drizzle and driver documentation. Pass your `schema` object to `drizzle` if you want the relational query API.

**Query builder.** Operators come from `drizzle-orm`:

```ts
import { eq, and, gt, inArray, desc, sql } from 'drizzle-orm';

const rows = await db.select({ id: posts.id, title: posts.title })
  .from(posts)
  .where(and(eq(posts.authorId, userId), gt(posts.id, cursor)))
  .orderBy(desc(posts.id))
  .limit(20);

const [created] = await db.insert(users)
  .values({ email })
  .onConflictDoNothing()
  .returning();

await db.update(posts).set({ title: 'New' }).where(eq(posts.id, id));
await db.delete(posts).where(eq(posts.id, id));
```

- Values are sent as **parameters**, so builder calls are not injectable through values.
- `.returning()` maps to PostgreSQL `RETURNING` and avoids a second query.
- `onConflictDoNothing` and `onConflictDoUpdate` map to `ON CONFLICT` (chapter 7). They need a matching unique index or constraint.
- Select only needed columns. A bare `select()` returns all columns, which defeats index-only scans and moves needless data.
- **`update` or `delete` without `.where(...)` affects every row.** Review these in code review.
- Keyset (cursor) pagination, as above, scales better than large `OFFSET`.

**Relations and the relational query API.** You can describe relations with the `relations` helper and then query nested data:

```ts
import { relations } from 'drizzle-orm';

export const usersRelations = relations(users, ({ many }) => ({ posts: many(posts) }));
export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));

const result = await db.query.users.findMany({ with: { posts: true }, limit: 10 });
```

Relations are application-level metadata for querying. They are **separate from foreign keys**, which you still declare with `.references()`. The relational API builds a single SQL statement for nested results rather than issuing one query per row, which avoids classic N+1, but you should still log the generated SQL and run `EXPLAIN` on it for important paths. The snippet above is the older "v1" style. Newer Drizzle releases have a "relations v2" API: you call `defineRelations(schema, (r) => ({ ... }))` with `r.one.<table>({ from, to })` and `r.many.<table>(...)`, and pass `relations` to `drizzle({ client, relations })`. Queries still use `db.query.<table>.findMany({ with: ... })`. Whether v1 stays supported is not stated in the docs we read (unverified), so follow the docs for your installed version. Use plain joins in the query builder whenever you need exact control.

**Transactions.**

```ts
await db.transaction(async (tx) => {
  const [acct] = await tx.select().from(accounts).where(eq(accounts.id, id)).for('update');
  if (acct.balance < amount) tx.rollback();
  await tx.update(accounts).set({ balance: sql`${accounts.balance} - ${amount}` }).where(eq(accounts.id, id));
});
```

- The callback receives `tx`. **Use `tx`, not `db`, for every query inside**, or those queries run outside the transaction on another connection.
- Throwing an error rolls back and rethrows. `tx.rollback()` rolls back explicitly.
- The second argument to `db.transaction` accepts `isolationLevel` (`"read uncommitted"`, `"read committed"`, `"repeatable read"`, `"serializable"`), `accessMode` (`"read only"` or `"read write"`) and `deferrable` (boolean). Calling `tx.transaction(...)` inside creates a savepoint. For Serializable or Repeatable Read, wrap the call in your own retry loop for SQLSTATE `40001` (chapter 7). Drizzle does not retry for you. Inspect the driver's error object for the code (confirm the field name for your driver).
- Row-locking support on select (`.for('update')`, with options such as `skipLocked`) exists in recent versions, but the select docs page we checked does not describe it, so the exact signature is unverified. If in doubt use the raw SQL below.
- Under transaction-mode pooling, a transaction is safe because it stays on one server connection, but session state is not.

**The raw `sql` escape hatch.** The `sql` template tag embeds expressions and parameters. Interpolated values become **bound parameters**, not string concatenation:

```ts
const claimed = await db.execute(sql`
  UPDATE jobs SET status = 'running', started_at = now()
  WHERE id = (
    SELECT id FROM jobs WHERE status = 'queued'
    ORDER BY id LIMIT 1 FOR UPDATE SKIP LOCKED
  )
  RETURNING id, payload
`);
```

Use raw SQL for things the builder cannot express cleanly: `SKIP LOCKED` claims, window functions, `LATERAL`, CTEs, full-text and JSONB operators, advisory locks, `SET LOCAL` for row-level security. The return shape of `db.execute` differs per driver (for example a result object with `rows` for node-postgres, an array-like for postgres.js). The Drizzle docs do not state this, so it is unverified. Check with your driver. Never build SQL by concatenating user input into a string. If you must make an identifier dynamic, use the identifier helpers and a whitelist.

Setting a tenant for row-level security (chapter 9) inside a transaction:

```ts
await db.transaction(async (tx) => {
  await tx.execute(sql`SELECT set_config('app.tenant_id', ${tenantId}, true)`);
  // all queries on tx are now filtered by the policy
});
```

**drizzle-kit: generate, migrate, push.**

| Command | What it does | Use it for |
|---|---|---|
| `drizzle-kit generate` | Diffs your TypeScript schema against the previous snapshot and writes a **SQL migration file** plus metadata into the output folder | The normal, reviewable workflow for anything shared or production |
| `drizzle-kit migrate` | Applies pending generated migration files to the database and records them in a migrations table | Deploying migrations |
| `drizzle-kit push` | Compares your schema directly to the live database and applies the difference **without creating a migration file** | Local prototyping and throwaway dev databases only |
| `drizzle-kit pull` | Introspects an existing database and generates a schema | Adopting Drizzle on an existing database |
| `drizzle-kit studio` | Opens a browser data viewer | Local inspection |
| `drizzle-kit check`, `up`, `export` | Check migrations for collisions, upgrade old snapshots, print the schema as SQL | Housekeeping and CI |

A programmatic alternative to the CLI migrate command is a `migrate(db, { migrationsFolder })` function from the driver-specific migrator module (confirm the import path), which fits running migrations as a release step.

The configuration file (`drizzle.config.ts`) uses `defineConfig` with the dialect (`postgresql`), the schema path, the output folder and database credentials. Confirmed keys: `dialect`, `schema`, `out` (default `drizzle`), `dbCredentials` (`url` or `host`, `port`, `user` and so on), plus optional `migrations`, `tablesFilter`, `schemaFilter`, `verbose` and `breakpoints`. Older versions used other key names, so match your installed version.

**Why the difference matters.** `push` leaves no history, so it cannot be reviewed, replayed in another environment or rolled back, and a rename may be interpreted as drop plus create (data loss) unless you confirm the prompt correctly. Production should use `generate` then `migrate`, with generated SQL **read in code review**.

**Making generated migrations safe.** The generator knows the schema diff, not your table sizes or lock behaviour. Review every generated file against chapter 13:
- Edit the SQL to use `CREATE INDEX CONCURRENTLY`, and `NOT VALID` plus `VALIDATE CONSTRAINT`, as needed. Confirm how your migrator groups statements into transactions, since concurrent index builds cannot run in a transaction block.
- Add `SET lock_timeout` at the top.
- Replace destructive renames and type changes with expand, migrate, contract across releases.
- Check the generated down-path expectations yourself. Treat migrations as forward-only, and keep rollback plans separate.
- Run migrations over a **direct** connection (chapter 11, 12), from a release job rather than from every app instance at startup, so two instances do not race.
- Keep the migrations folder and snapshots committed. Never hand-edit an already-applied migration. Add a new one. Resolve merge conflicts in the snapshot metadata carefully, since the migration history is linear.

**Pitfalls.**
- Schema in TypeScript and database drift apart when someone changes the database by hand, and `generate` then produces surprising diffs. Detect drift with `pull` or a diff in CI.
- Using `push` against production.
- Forgetting `tx` inside a transaction callback.
- Large result sets loaded fully into memory. Paginate or stream with the driver's facilities.
- Trusting `.$type<>()` JSONB annotations as validation.
- Dates and time zones: be explicit about `withTimezone` and about the mapped JS type.
- Treating the ORM as a substitute for reading plans. Log SQL during development, and use `pg_stat_statements` in production to see what the ORM actually sends.
- Chatty code: loops that run one query per item. Batch with `inArray`, joins or the relational API.
- Pooling and prepared statements under a transaction pooler.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schema | `pgTable`, column helpers, indexes, constraints, enums, `$inferSelect` / `$inferInsert` | Model a real schema and derive types |
| Queries | `select`, `insert`, `update`, `delete`, `eq`/`and`/`or`/`inArray`, joins, `returning`, `onConflict*` | Write any CRUD and join query and predict its SQL |
| Relational API | `relations`, `db.query.<table>.findMany` with `with` | Fetch nested data and inspect the generated SQL |
| Transactions | `db.transaction`, `tx`, `rollback`, isolation options, retry wrapper | Implement a transfer or job claim correctly |
| Raw SQL | `sql` template, `db.execute`, `sql.identifier` helpers | Drop to SQL safely for advanced features |
| Migrations | **drizzle-kit** `generate`, `migrate`, `push`, `pull`, `studio`, `drizzle.config.ts` | Run a reviewable pipeline and know when `push` is acceptable |
| SQL logging | Drizzle logger option, `pg_stat_statements` | See and measure the SQL your code produces |

### Common mistakes

- Using `push` outside local development.
- Applying generated migrations unreviewed, including destructive drops and blocking index builds.
- Running migrations through the pooled connection or from every instance at startup.
- `update` or `delete` without a `where`.
- Using `db` instead of `tx` inside a transaction.
- Retrying nothing on `40001` under stronger isolation.
- Believing relations replace foreign keys or that types validate runtime data.
- Selecting every column everywhere.

### Self-check

- What is the difference between `generate`, `migrate` and `push`, and which belongs in production?
- Why does an `update` inside `db.transaction` still need to use `tx`?
- How does a Drizzle relation differ from a foreign key?
- A generated migration adds an index on a 200 million row table. What do you change before running it?
- Which Drizzle settings or options change under a transaction pooler and why?

---

## 16. Practice labs

**In plain words.** These exercises let you see each problem happen on a safe local database. Try each one before you read the answer. Never run them on a shared or production database.

Use a local PostgreSQL instance (a container is fine) so you can break things freely. Each lab states the goal and what "done" looks like. Attempt before looking anything up. Do not run these against shared or production databases.

1. **See MVCC.** Create a table, insert a row, and select `xmin, xmax, ctid`. Update it and select again. In a second session, open a transaction before the update and read the row after it. Explain every value you see.
2. **Make bloat, then fix it.** Load a few million rows, update every row several times with autovacuum disabled for that table, and measure `pg_relation_size` and `n_dead_tup`. Run `VACUUM`, then `VACUUM FULL`, and compare sizes and the locks each took (`pg_locks`).
3. **Long transaction versus vacuum.** Hold a transaction open in one session, churn rows in another, run `VACUUM VERBOSE`, and show dead tuples that could not be removed. Identify the holder from `backend_xmin`.
4. **HOT updates.** On a table with an index on a rarely changed column, measure `n_tup_hot_upd` versus `n_tup_upd`. Add an index on the updated column and re-measure. Then try a lower `fillfactor`.
5. **Index design.** For a query filtering on `tenant_id` and `created_at` with an `ORDER BY`, compare a seq scan, a single-column index and a composite index with `EXPLAIN (ANALYZE, BUFFERS)`. Add a partial and a covering variant and observe an index-only scan after `VACUUM`.
6. **Bad estimate.** Create two correlated columns, query on both, and capture an estimate-versus-actual mismatch. Fix it with `CREATE STATISTICS` and `ANALYZE`, and show the new plan.
7. **Join algorithms.** Force and observe nested loop, hash and merge joins on tables of different sizes by changing data volume and indexes (use the `enable_*` settings only as a diagnostic in your session).
8. **Lock queue.** In session A start a transaction and read a table. In session B run `ALTER TABLE ... ADD COLUMN`. In session C run `SELECT`, and observe it blocked. Diagnose with `pg_blocking_pids`. Repeat with `lock_timeout` set in B.
9. **Deadlock.** Produce a deadlock with two sessions updating two rows in opposite order. Read the server log. Fix it with consistent ordering.
10. **Job queue.** Build a `jobs` table and claim query with `FOR UPDATE SKIP LOCKED`. Run several concurrent workers in a script and prove no job is processed twice. Add a reclaim rule for crashed workers.
11. **Isolation anomalies.** Reproduce a lost update under Read Committed, a non-repeatable read, and write skew under Repeatable Read. Show Serializable aborting one transaction with `40001`, and write a retry loop in your language of choice.
12. **JSONB and search.** Store JSON documents, query with `@>`, add a GIN index and an expression index, and compare plans. Build a generated `tsvector` column with a GIN index and query it with `websearch_to_tsquery`.
13. **Partitioning.** Create a range-partitioned events table, load data, confirm pruning in `EXPLAIN`, then drop an old partition and compare with a large `DELETE` for time and bloat.
14. **Row-level security.** Build a two-tenant schema with policies and a non-owner application role. Prove tenant A cannot read or write tenant B, then show what happens if you connect as the owner. Use `set_config(..., true)` inside a transaction.
15. **Backup and PITR.** Take a `pg_dump -Fc` and restore one table with `pg_restore`. Then configure WAL archiving and a base backup, make a deliberate destructive change, and recover to a time just before it.
16. **Replication.** Set up a streaming standby and measure lag under load with `pg_stat_replication`. Create an unused replication slot and watch WAL accumulate. Set up a logical publication and subscription for one table.
17. **Pooler.** Put PgBouncer in front of your database in transaction mode. Demonstrate failures from session-level `SET`, named prepared statements, `LISTEN` and a session advisory lock, then apply each workaround.
18. **Zero-downtime migration.** On a table with several million rows, perform a column split using expand, migrate, contract while a script generates steady read and write traffic. Use a batched backfill, `CREATE INDEX CONCURRENTLY`, and `NOT VALID` plus `VALIDATE`. Record any errors seen by the traffic generator (the target is none).
19. **Monitoring.** Enable `pg_stat_statements` and `log_min_duration_statement`. Run `pgbench`, then produce a ranked list of top queries, and a pgBadger report from the log.
20. **Drizzle end to end.** Define a schema with relations, generate and review a migration, apply it with `migrate`, write a transactional money transfer using `tx` and a retry on `40001`, and a raw `sql` job-claim query. Log the SQL, and compare the relational API's SQL to an equivalent hand-written join.
21. **Drizzle migration safety.** Change a column in your schema in a way that generates a blocking statement. Edit the generated SQL to the safe pattern from chapter 13 and run it under concurrent traffic. Contrast with what `push` would have done.
22. **Incident write-up.** Cause a connection exhaustion and a lock pile-up on purpose, diagnose them using only the views in chapter 14, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 17. Interview questions

**In plain words.** This chapter lists common questions with short model answers. Each question has a level tag: Fresher, Mid or Senior. Practise saying the answers aloud in your own words. Model answers are study aids and do not promise any interview or job result.

Grouped by level. Every question has a concise model answer in the final subsection, numbered to match.

### Top 10 must-know

Learn these first. The model answer for each is in the numbered answers below.

1. [Fresher] Q1: What is MVCC and why does PostgreSQL use it?
2. [Fresher] Q4: What is the default isolation level and what does it guarantee?
3. [Fresher] Q6: What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?
4. [Mid] Q13: Why does an `UPDATE` create a new row version?
5. [Mid] Q17: How do you read an `EXPLAIN ANALYZE` plan?
6. [Mid] Q22: Why can an `ALTER TABLE` cause an outage, and how do you prevent it?
7. [Mid] Q24: What breaks under PgBouncer transaction pooling?
8. [Mid] Q27: Compare `drizzle-kit push`, `generate` and `migrate`.
9. [Senior] Q31: Explain transaction ID wraparound and its prevention.
10. [Senior] Q34: Design a backup and recovery strategy with RPO and RTO.

### Fresher

1. [Fresher] What is MVCC and why does PostgreSQL use it?
2. [Fresher] What do `xmin` and `xmax` represent?
3. [Fresher] What does `VACUUM` do, and what does autovacuum do?
4. [Fresher] What is the default isolation level and what does it guarantee?
5. [Fresher] When would you add an index, and when would it not help?
6. [Fresher] What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?
7. [Fresher] What is a deadlock and how does PostgreSQL handle it?
8. [Fresher] Why do we use connection pooling?
9. [Fresher] What is the difference between `JSON` and `JSONB`?
10. [Fresher] What are `pg_dump` and `pg_restore` for?
11. [Fresher] In Drizzle, how do you get the TypeScript type of a row from a table definition?
12. [Fresher] What does `drizzle-kit generate` produce?

### Mid-level

13. [Mid] Why does an `UPDATE` create a new row version and what are the consequences?
14. [Mid] Explain HOT updates and `fillfactor`.
15. [Mid] What column order do you choose for a composite index, and why?
16. [Mid] What is a partial index and a covering index with `INCLUDE`?
17. [Mid] How do you read an `EXPLAIN ANALYZE` plan, and what do you look at first?
18. [Mid] A plan estimates 10 rows but gets 2 million. What happened and what do you do?
19. [Mid] How do you implement a job queue in PostgreSQL?
20. [Mid] Describe Repeatable Read versus Serializable in PostgreSQL and the retry requirement.
21. [Mid] What is a lost update and how do you prevent it?
22. [Mid] Why can an `ALTER TABLE` cause an outage, and how do you protect against it?
23. [Mid] How do you add a `NOT NULL` column safely to a large table?
24. [Mid] What breaks under PgBouncer transaction pooling?
25. [Mid] How do you implement multi-tenant isolation with row-level security?
26. [Mid] How do you find the slowest queries in production?
27. [Mid] Compare `drizzle-kit push`, `generate` and `migrate`.
28. [Mid] How do Drizzle transactions work, and what is a common mistake with them?
29. [Mid] When do you reach for raw `sql` in Drizzle?
30. [Mid] How do you make read replicas safe for read-after-write flows?

### Senior

31. [Senior] Explain transaction ID wraparound and how you would prevent it.
32. [Senior] A table has become hugely bloated. Diagnose and fix without downtime.
33. [Senior] Walk through a zero-downtime column rename or type change.
34. [Senior] Design a backup and recovery strategy with defined RPO and RTO.
35. [Senior] How would you do a zero-downtime major version upgrade?
36. [Senior] Serverless functions are exhausting connections. Diagnose and redesign.
37. [Senior] When would you partition a table, and what are the costs?
38. [Senior] How would you design multi-tenancy: shared schema with RLS, schema per tenant, or database per tenant?
39. [Senior] Latency spiked after a deploy and you cannot reproduce it. How do you investigate on the database side?
40. [Senior] When would you choose PostgreSQL features (JSONB, full-text, queue via `SKIP LOCKED`) over a specialised system?
41. [Senior] How do you govern ORM-generated migrations and queries in a team?

### Model answers (condensed)

**1.** Multi-version concurrency control keeps multiple row versions so readers do not block writers and writers do not block readers. Each transaction sees a snapshot of committed data.

**2.** `xmin` is the ID of the transaction that created the tuple version. `xmax` is the ID of the transaction that deleted or locked it, or zero. Visibility is decided from these and the snapshot.

**3.** `VACUUM` removes dead tuples no snapshot needs, updates the free space and visibility maps and freezes old tuples. Autovacuum runs vacuum and analyze automatically based on table change thresholds.

**4.** Read Committed. Each statement sees the data committed before that statement began. No dirty reads, but non-repeatable reads and lost updates from read-modify-write code are possible.

**5.** Add one when a measured query filters, joins or sorts on a column and returns a small fraction of the table. It will not help when the query returns most rows, when a function wraps the column, on tiny tables, or on very low-selectivity columns. It also costs writes and disk.

**6.** `EXPLAIN` shows the estimated plan without running. `EXPLAIN ANALYZE` executes the query and adds actual rows and timing. For data-changing statements run it inside a transaction and roll back.

**7.** Two transactions each hold a lock the other needs. PostgreSQL detects it after `deadlock_timeout` and aborts one with SQLSTATE `40P01`. Prevent with consistent lock ordering and short transactions, and retry in the application.

**8.** Connections are costly processes. A pool reuses a small number of connections for many requests, protecting the database from overload and cutting connect latency.

**9.** `JSON` stores the text as given. `JSONB` stores a parsed binary form that is indexable (GIN) and supports containment operators, but does not preserve whitespace or key order and removes duplicate keys.

**10.** `pg_dump` makes a consistent logical backup of one database. `pg_restore` restores archive formats, selectively and in parallel. Roles need `pg_dumpall --globals-only`.

**11.** `typeof table.$inferSelect` for reads and `typeof table.$inferInsert` for inserts (confirm names in the current docs).

**12.** A SQL migration file reflecting the diff between your schema and the previous snapshot, plus snapshot metadata. It does not touch the database.

**13.** MVCC marks the old tuple's `xmax` and inserts a new tuple so concurrent snapshots can still see the old one. Consequences are dead tuples and bloat if vacuum lags, index maintenance for non-HOT updates, and higher write volume.

**14.** A HOT update occurs when no indexed column changes and the new tuple fits on the same page, so no index entries are written and cleanup is cheap. Lowering `fillfactor` leaves space on pages to make that more likely, at the cost of a larger table. Indexing frequently updated columns defeats it.

**15.** Equality columns first, then the range or sort column, so the index serves the filter and the ordering. The index serves a leftmost prefix of its columns. Order also depends on which queries are most common.

**16.** A partial index covers only rows matching a predicate, making it small and targeted, and the query must imply the predicate. An `INCLUDE` index stores extra payload columns so a query can use an index-only scan, subject to the visibility map.

**17.** Read from the innermost node outward. Compare estimated and actual rows per node, remember times are per loop, then look at rows removed by filter, sort spills, buffer reads, and loop counts. The first large estimate error usually explains the plan.

**18.** The planner's statistics are misleading: stale statistics, correlated columns, skew or an inestimable expression. Run `ANALYZE`, raise the statistics target or create extended statistics, add an expression index, or rewrite the query. Do not disable planner features in production.

**19.** A `jobs` table with a status, a partial index on queued rows, and a claim query using `FOR UPDATE SKIP LOCKED` inside an `UPDATE ... RETURNING`. Add a lease or heartbeat to reclaim crashed workers, retry limits, idempotent job handlers and tuned autovacuum.

**20.** Repeatable Read uses one snapshot for the transaction (it also prevents phantoms in PostgreSQL) but allows write skew. Serializable adds dependency tracking (SSI) and aborts transactions that would break serial order. Both can fail with `40001`, so the application must retry the whole transaction with backoff.

**21.** Two transactions read the same value and both write a result computed from it, so one write overwrites the other. Prevent with atomic SQL updates, `SELECT ... FOR UPDATE`, optimistic concurrency with a version column, or Serializable with retries.

**22.** DDL needing `ACCESS EXCLUSIVE` queues behind any transaction touching the table, and every later query queues behind it. Use a short `lock_timeout` with retries, check for long transactions first, and use non-blocking variants (`CONCURRENTLY`, `NOT VALID`).

**23.** Add the column as nullable (a constant default is cheap in modern versions), backfill in batches, add `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE CONSTRAINT`, then `SET NOT NULL`, then drop the check. Ship application changes in step.

**24.** Anything bound to a session: non-local `SET`, temporary tables, named prepared statements (unless the pooler tracks them), `LISTEN/NOTIFY`, session advisory locks and `WITH HOLD` cursors. Use `SET LOCAL`, transaction advisory locks, disable client prepared statements, and a direct connection for listeners and migrations.

**25.** Add `tenant_id` to every table, enable and force RLS with policies using a transaction-local setting, connect as a non-owner, non-superuser role, set the tenant with `set_config(..., true)` per transaction, lead indexes with `tenant_id` and test both allow and deny cases.

**26.** `pg_stat_statements` sorted by total and mean execution time and call count, with `log_min_duration_statement`, `auto_explain` and pgBadger for plans and history. Then `EXPLAIN (ANALYZE, BUFFERS)` the top offenders.

**27.** `generate` writes a reviewable SQL migration file. `migrate` applies generated files and records them. `push` syncs the live schema directly with no file or history. Use `generate` plus `migrate` for shared and production databases and `push` only for disposable local work.

**28.** `db.transaction(async (tx) => ...)` runs the callback in one transaction, commits on return and rolls back on a thrown error or `tx.rollback()`. The common mistake is using `db` inside the callback, which runs outside the transaction. Also missing retries on `40001`.

**29.** For SQL the builder cannot express cleanly: `SKIP LOCKED` claims, window functions, CTEs, `LATERAL`, JSONB and full-text operators, advisory locks, `SET LOCAL`. Interpolated values in the `sql` tag are bound as parameters.

**30.** Route reads that must see a user's own writes to the primary for a short time, or wait until the replica's replay position passes the write's LSN. Accept eventual consistency elsewhere and monitor lag.

**31.** Transaction IDs are 32-bit and compared circularly, so old tuples must be frozen before their age approaches about two billion. Prevent by keeping autovacuum healthy and tuned, avoiding long transactions and stale replication slots, monitoring `age(datfrozenxid)` and `age(relfrozenxid)`, and alerting well before limits. If critical, run vacuum freeze on the oldest tables.

**32.** Measure with `pgstattuple` and size versus live data. Find the cause: long transactions pinning the horizon, thresholds too lax, autovacuum starved, mass updates. Fix the cause first, then reclaim space online with pg_repack or a concurrent rebuild, or rebuild indexes with `REINDEX CONCURRENTLY`. Avoid `VACUUM FULL` on a live table.

**33.** Expand: add the new column or table and an index concurrently. Migrate: dual-write in code, backfill in batches, verify. Switch reads, stop writing the old column. Contract: drop the old column in a later release. Each step is backward compatible and uses lock timeouts.

**34.** Define RPO and RTO, use base backups plus continuous WAL archiving for PITR (or a managed equivalent), keep copies off the primary's failure domain, protect them from production credentials, add replicas for availability, and test restores on a schedule measuring real recovery time.

**35.** Logical replication to a new-version cluster: copy schema, set up publication and subscription, sync sequences, wait for lag to drain, pause writes briefly, cut over, and keep a rollback path. `pg_upgrade` is the in-place alternative with downtime. Rehearse either on a copy.

**36.** Each function instance opens connections, multiplying past `max_connections`. Use a pooled endpoint in transaction mode, small per-instance pools (often one), disable named prepared statements, use direct connections only for migrations, set timeouts and consider HTTP-based drivers. Verify with `pg_stat_activity`.

**37.** For very large tables with natural time or tenant keys, retention needs (drop instead of delete), and queries that filter on the key. Costs: keys must be part of unique constraints, no global uniqueness, partition management jobs, planning overhead with many partitions, and queries without the key scan everything.

**38.** Shared schema with `tenant_id` and RLS is cheapest to operate and scales to many small tenants, with noisy neighbour and migration-blast-radius risks. Schema per tenant gives stronger separation but many objects and slow migrations. Database per tenant gives the strongest isolation and per-tenant restore at high operational cost. Choose on tenant count, size variance, compliance and restore requirements.

**39.** Compare `pg_stat_statements` deltas before and after, check `pg_stat_activity` for waits and idle-in-transaction sessions, locks, connection counts, plan changes (new statistics, parameter skew, a generic plan), autovacuum or checkpoint activity, and new ORM query shapes (N+1). Use `auto_explain` to capture plans next time.

**40.** When data already lives in PostgreSQL, volume and relevance needs are moderate, and transactional consistency with your data is valuable, because it removes a system, its sync pipeline and its failure modes. Move to a specialised system when you hit relevance, scale, latency or throughput limits you have measured.

**41.** Review generated SQL in pull requests, forbid `push` outside local, run migrations from a release job over a direct connection, lint for unsafe DDL, rehearse on production-like data, enforce lock timeouts, log and sample ORM SQL, watch `pg_stat_statements` after deploys, and document expand, migrate, contract as the default process.

---

## 18. References

**In plain words.** These are the official sources to read when you need exact and current facts. Prefer them over blog posts. Pick the documentation version that matches the version you run.

Primary sources, preferred over blog posts. Versioned documentation is at the official site, so pick the major version you run.

- PostgreSQL documentation (all versions): https://www.postgresql.org/docs/
  - Concurrency control and isolation levels: the "Concurrency Control" chapter
  - Explicit locking and lock modes: "Explicit Locking" under Concurrency Control
  - Routine vacuuming and wraparound: "Routine Database Maintenance Tasks", "Routine Vacuuming"
  - Using EXPLAIN and planner statistics: "Performance Tips"
  - Indexes: "Indexes" part of the manual, including partial indexes and index-only scans
  - Write-ahead logging and reliability: "Reliability and the Write-Ahead Log"
  - Server configuration: "Server Configuration"
  - Monitoring statistics views: "Monitoring Database Activity" (the cumulative statistics views, `pg_stat_activity`, `pg_locks`)
  - Backup and restore, continuous archiving and PITR: "Backup and Restore"
  - High availability, replication, logical replication: "High Availability, Load Balancing, and Replication" and "Logical Replication"
  - Row security policies: "Data Definition" chapter, section on row security
  - JSON types, full-text search, table partitioning, generated columns: the corresponding chapters
  - Client applications `psql`, `pg_dump`, `pg_restore`, `pg_basebackup`, `pgbench`: the "PostgreSQL Client Applications" and "Additional Supplied Programs" references
  - `pg_stat_statements` and other contrib modules: the "Additional Supplied Modules" appendix
  - Internals (page layout, heap tuples, MVCC details): the "Internals" part, including "Database Physical Storage"
- PostgreSQL release notes (for per-version behaviour changes): https://www.postgresql.org/docs/release/
- PgBouncer documentation (pool modes, configuration, feature compatibility matrix): https://www.pgbouncer.org/
- pgBadger project page: https://github.com/darold/pgbadger
- Drizzle ORM documentation (schema, queries, relations, transactions, drizzle-kit): https://orm.drizzle.team/docs/overview
- Drizzle Kit documentation (generate, migrate, push, pull, config): https://orm.drizzle.team/docs/kit-overview (config file: https://orm.drizzle.team/docs/drizzle-config-file)
- node-postgres documentation (pooling, transactions): https://node-postgres.com/
- postgres.js repository and README (options including prepared statement behaviour): https://github.com/porsager/postgres
- Bruce Momjian, *Mastering PostgreSQL Internals* talks and PostgreSQL Wiki: https://wiki.postgresql.org/
- *The Internals of PostgreSQL*, Hironobu Suzuki (free online book): https://www.interdb.jp/pg/
- Brendan Gregg, USE method (applies to the host under the database): https://www.brendangregg.com/usemethod.html

Verify any URL before depending on it, since pages move. On 2026-11-06 the PostgreSQL, PgBouncer, Drizzle, pgBadger, interdb.jp and USE method links loaded. The node-postgres, postgres.js and PostgreSQL wiki links were not checked (unverified).

---

## 19. Glossary

**In plain words.** This table gives short meanings for the hardest terms in this guide. Return here whenever a word is unclear.

| Term | Plain meaning |
|---|---|
| MVCC | Keeping several versions of a row so readers and writers do not block each other |
| Tuple | One stored version of a row |
| Snapshot | A fixed view of which changes were committed at one moment |
| Dead tuple | An old row version that no transaction needs any more |
| Bloat | Wasted space in a table or index from dead tuples |
| VACUUM | A job that removes dead tuples and makes their space reusable |
| Freeze | Mark an old row as visible to everyone, so its transaction ID no longer matters |
| Wraparound | The problem when 32-bit transaction IDs run out of room and old rows would look new |
| HOT update | An update that stays on the same page and needs no index changes |
| fillfactor | How full the database fills each page, leaving room for updates |
| WAL | Write-ahead log: a file where each change is recorded before the data file changes |
| Checkpoint | A moment when changed pages are written to the data files |
| Planner | The part that chooses how to run a query |
| Selectivity | The share of rows a condition keeps |
| Index-only scan | Answering a query from the index without reading the table |
| Isolation level | A rule for what one transaction can see of others' changes |
| Write skew | Two transactions each read the same data, then write different rows, and break a rule together |
| Deadlock | Two transactions each wait for a lock the other holds |
| Advisory lock | A lock your own code defines, not tied to a table or row |
| Replication slot | A marker that makes the primary keep log files until a replica reads them |
| PITR | Point-in-time recovery: restoring the database to a chosen moment |
| RPO / RTO | How much data you can lose, and how long recovery may take |
| Connection pooler | A service that lets many clients share a few database connections |
| Backfill | Filling a new column for old rows, usually in small batches |
| Idempotent | Safe to run twice with the same final result |
| TOAST | PostgreSQL's way of storing large values outside the main row |
| Row-level security | Rules that decide which rows a role may see or change |
| Cold start | The delay when a paused serverless database wakes up |
| Migration | A versioned script that changes the database schema |
