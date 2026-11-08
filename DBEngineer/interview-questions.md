# Database Engineer Interview Question Bank

This file helps you prepare for database interviews. It follows the 30-day course in this folder.
Each answer is short. Each answer points to the day file where you can learn the full detail.

How to use this file:

1. Read the "Top 15" section first.
2. Cover the answer, say your own answer aloud, then compare.
3. Open the "Revise" file when your answer was weak.
4. Do the SQL problems on paper first, then run them on a real database.

Tags: [Fresher] = basic, [Mid] = 2 to 5 years of work, [Senior] = design and incident level.

Notes on scope:

- Most answers use PostgreSQL, because the course uses it. Other systems differ in details. Check your own version.
- Terms in brackets after a word are short definitions.
- This file gives no promise about job results. It is only practice material.

## Table of Contents

1. Top 15 must-know questions
2. Question bank by topic
   - A. Data modelling and normalisation
   - B. SQL core and advanced
   - C. Storage and indexes
   - D. Query planner and optimisation
   - E. Transactions, MVCC, locking, vacuum
   - F. Performance in practice
   - G. Replication, scaling, sharding, CAP
   - H. NoSQL and choosing a store
   - I. Operations: backup, migration, security, warehouse
3. SQL practice problems (25)
4. Common mistakes in database interviews

---

# 1. Top 15 must-know questions

If you have only one evening, study these.

**T1. What is the difference between OLTP and OLAP?**
OLTP (online transaction processing) handles many small reads and writes, such as orders and logins.
OLAP (online analytical processing) scans large data to make reports.
OLTP wants low latency and row storage. OLAP wants high scan speed and often column storage.
Revise: day-01-why-databases-oltp-olap.md

**T2. What does ACID mean?**
Atomicity: all steps of a transaction succeed, or none does.
Consistency: rules such as keys and checks always hold after commit.
Isolation: concurrent transactions do not see each other's half-done work.
Durability: after commit, data survives a crash. The write-ahead log (WAL) gives this.
Revise: day-11-acid-in-depth.md

**T3. How does a B-tree index speed up a query?**
It keeps keys sorted in a balanced tree. A lookup needs about log(n) page reads instead of a full table scan.
It also serves range filters and ORDER BY on the indexed columns.
It costs extra disk space and slows writes.
Revise: day-07-indexes-deep.md

**T4. What is the leftmost-prefix rule?**
A composite index on (a, b, c) helps queries that filter on a, or a and b, or a, b and c.
It does not help a query that filters only on b or c.
Put equality columns first, then the range or sort column.
Revise: day-07-indexes-deep.md

**T5. Why might the database ignore your index?**
The query wraps the column in a function, the types do not match, the filter is not selective, or statistics are stale.
For a small table, a sequential scan can be cheaper.
Check with EXPLAIN. Do not guess.
Revise: day-08-when-indexes-hurt.md and day-15-query-optimization.md

**T6. What do EXPLAIN and EXPLAIN ANALYZE show?**
EXPLAIN shows the plan and estimated cost without running the query.
EXPLAIN ANALYZE runs the query and adds real timings and real row counts.
A big gap between estimated and actual rows is the main clue for a bad plan.
Revise: day-09-query-execution-planner.md

**T7. What is MVCC?**
MVCC (multi-version concurrency control) keeps several versions of a row.
Each transaction reads a snapshot, so readers do not block writers and writers do not block readers.
Old versions pile up and must be cleaned by VACUUM in PostgreSQL.
Revise: day-14-mvcc-vacuum.md

**T8. Name the isolation levels and the anomalies they stop.**
Read Committed stops dirty reads. Repeatable Read also stops non-repeatable reads (and phantoms in PostgreSQL).
Serializable also stops write skew. Higher levels cost more and may abort transactions that you must retry.
Revise: day-12-isolation-levels-anomalies.md

**T9. What is a deadlock and how do you prevent it?**
Two transactions each hold a lock the other needs, so both wait forever.
The database detects it and aborts one. Prevent it by locking rows in the same order and keeping transactions short.
Revise: day-13-locking-deadlocks.md

**T10. What is the N+1 query problem?**
The code runs one query for a list, then one more query for each item. That is N+1 queries.
Fix it with a join, a batch query using IN or ANY, or eager loading in the ORM.
Revise: day-16-index-tuning-n-plus-1.md

**T11. What is normalisation, and when do you denormalise?**
Normalisation removes repeated facts so each fact lives in one place. It prevents update, insert and delete anomalies.
Denormalise on purpose for read speed, such as reports, when you accept extra storage and update work.
Revise: day-03-normalization.md

**T12. Explain replication, sharding and partitioning.**
Replication copies the same data to many servers (for reads and safety).
Partitioning splits one table into parts inside one server.
Sharding splits data across many servers, each owning a part.
Revise: day-22-sharding.md and day-20-replication.md

**T13. What does the CAP theorem say?**
During a network partition (a split between nodes), a system must choose between consistency and availability.
It does not say you can only have two of three at all times.
PACELC adds: without a partition, you still trade latency for consistency.
Revise: day-23-cap-consistency.md

**T14. How do you add a NOT NULL column to a huge table without downtime?**
Add the column as nullable. Backfill in small batches. Add a CHECK constraint with NOT VALID, then VALIDATE it.
Then SET NOT NULL, which can reuse the valid check in PostgreSQL 12 and later. Set a lock_timeout.
Revise: day-28-zero-downtime-migrations.md

**T15. A query became slow after a deploy. What do you check first?**
Find the exact query (pg_stat_statements or logs). Run EXPLAIN (ANALYZE, BUFFERS). Compare with the old plan.
Check for a dropped index, new unsargable filter, stale statistics, a new N+1 pattern, or a lock wait.
Fix the cause, then measure again.
Revise: day-15-query-optimization.md and day-29-security-observability.md

---

# 2. Question bank by topic

## A. Data modelling and normalisation

**Q.** [Fresher] Why use a relational database instead of plain files?
A database gives you transactions, constraints, indexes, a query language, and safe concurrent access.
With files, you must build all of these yourself, and bugs lose data.
The engine also recovers after a crash using its log.
Revise: day-01-why-databases-oltp-olap.md

**Q.** [Fresher] What is the difference between a primary key, a unique key and a foreign key?
A primary key identifies each row. It is unique and not null, and there is one per table.
A unique key is also unique, but a table can have many, and it may allow nulls.
A foreign key points to a key in another table and keeps the link valid.
Revise: day-02-data-modeling-keys.md

**Q.** [Fresher] How do you model a many-to-many relationship?
Use a third table, called a junction table (or link table).
It holds the two foreign keys, for example student_id and course_id.
Make the pair a composite primary key so the same link cannot repeat. Extra columns, such as enrolled_on, go here.
Revise: day-02-data-modeling-keys.md

**Q.** [Mid] Surrogate key or natural key? Which do you choose?
A natural key is real data, such as an email or a tax number. A surrogate key is made by the system, such as an identity number or UUID.
Natural keys can change or be wrong. Surrogate keys are stable and small.
Common choice: surrogate primary key, plus a unique constraint on the natural key.
Revise: day-02-data-modeling-keys.md

**Q.** [Mid] What do ON DELETE CASCADE, SET NULL and RESTRICT do?
CASCADE deletes child rows when the parent is deleted. SET NULL clears the child's foreign key.
RESTRICT (or NO ACTION) blocks the delete while children exist.
Use CASCADE only for true ownership, such as order lines. A wrong cascade can delete a lot of data quietly.
Revise: day-02-data-modeling-keys.md

**Q.** [Mid] Does PostgreSQL index a foreign key column automatically?
No. It indexes the referenced primary key, but not the foreign key column in the child table.
Without an index, joins and parent deletes scan the child table. Create the index yourself.
MySQL InnoDB does create one automatically, so know which system you use.
Revise: day-02-data-modeling-keys.md and day-16-index-tuning-n-plus-1.md

**Q.** [Fresher] Explain 1NF, 2NF and 3NF in simple words.
1NF: each cell holds one atomic value, with no repeating groups.
2NF: every non-key column depends on the whole key, not only part of a composite key.
3NF: non-key columns depend only on the key, not on other non-key columns.
A quick phrase: "the key, the whole key, and nothing but the key".
Revise: day-03-normalization.md

**Q.** [Fresher] What are update, insert and delete anomalies?
They are errors caused by repeated data. Update anomaly: you change a fact in one row but not in others.
Insert anomaly: you cannot store a fact without storing another one. Delete anomaly: deleting a row also loses an unrelated fact.
Normalisation removes them.
Revise: day-03-normalization.md

**Q.** [Mid] When is denormalisation a good idea?
When reads are much more frequent than writes and joins are too slow, such as dashboards or search pages.
Examples: a stored order total, a counter column, or a copied customer name on an invoice (which should not change later).
You must then keep copies in sync, with a trigger, job or app code.
Revise: day-03-normalization.md

**Q.** [Senior] An invoice shows the product name and price. Should it join to the products table?
No for the price and the name at sale time. An invoice is a historical record, so copy the values into the line item.
If it joins to the product, a later price change would rewrite history.
Store product_id too, for reporting. This is deliberate denormalisation.
Revise: day-03-normalization.md

## B. SQL core and advanced

**Q.** [Fresher] What is the difference between WHERE and HAVING?
WHERE filters rows before grouping. HAVING filters groups after aggregation.
You cannot use an aggregate such as COUNT(*) in WHERE.
Use WHERE where possible, because it reduces rows early.
Revise: day-04-sql-core.md

**Q.** [Fresher] In what logical order does SQL run a SELECT?
FROM and JOIN, then WHERE, GROUP BY, HAVING, SELECT, DISTINCT, ORDER BY, and LIMIT.
This is why you cannot use a SELECT alias in WHERE, but you can in ORDER BY.
The physical plan may differ, but the result must match this order.
Revise: day-04-sql-core.md

**Q.** [Fresher] Why does `WHERE col = NULL` return nothing?
NULL means "unknown". Any comparison with NULL gives unknown, not true. Use `IS NULL` or `IS NOT NULL`.
SQL uses three-valued logic: true, false, unknown. WHERE keeps only true rows.
Revise: day-04-sql-core.md

**Q.** [Mid] What is the danger of NOT IN with a subquery?
If the subquery returns even one NULL, `x NOT IN (...)` never becomes true, so you get no rows.
Use NOT EXISTS, or a LEFT JOIN with an IS NULL check. Both handle NULLs in the expected way.
Revise: day-04-sql-core.md and day-05-sql-advanced.md

**Q.** [Fresher] What is the difference between INNER JOIN and LEFT JOIN?
INNER JOIN returns only rows that match in both tables.
LEFT JOIN returns all rows from the left table, and fills right columns with NULL when there is no match.
Use LEFT JOIN to find "rows with no related rows".
Revise: day-04-sql-core.md

**Q.** [Mid] You LEFT JOIN orders and put `WHERE orders.status = 'paid'`. What goes wrong?
The WHERE removes the NULL rows from unmatched customers, so the query acts like an INNER JOIN.
Move the condition into the ON clause: `LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'paid'`.
Revise: day-04-sql-core.md

**Q.** [Fresher] UNION or UNION ALL?
UNION removes duplicates, which needs a sort or hash step. UNION ALL keeps all rows and is faster.
Use UNION ALL unless you really need to remove duplicates.
Revise: day-05-sql-advanced.md

**Q.** [Mid] When do you use EXISTS, IN or a JOIN?
EXISTS checks "is there at least one match" and stops at the first. It is good for semi-joins and handles NULL safely.
IN works well with a short list or a small subquery. A JOIN is right when you need columns from both tables.
A JOIN can multiply rows if the match is one-to-many. Modern planners often give the same plan for all three.
Revise: day-05-sql-advanced.md

**Q.** [Mid] What is a CTE, and is it always a separate step in PostgreSQL?
A CTE (common table expression) is a named subquery from `WITH name AS (...)`. It makes long queries easier to read.
Since PostgreSQL 12, a simple non-recursive CTE used once is inlined, not forced to run first.
You can force it with `AS MATERIALIZED` or prevent it with `AS NOT MATERIALIZED`.
Revise: day-05-sql-advanced.md

**Q.** [Mid] What is a recursive CTE, and what do you use it for?
It has an anchor query (the start) and a recursive part that joins to the CTE itself, joined by UNION ALL.
It walks trees and graphs: org charts, category trees, bill of materials.
Guard against cycles with a depth limit or a path array.
Revise: day-05-sql-advanced.md

**Q.** [Fresher] What is the difference between ROW_NUMBER, RANK and DENSE_RANK?
ROW_NUMBER gives unique numbers, even for ties: 1, 2, 3.
RANK gives ties the same number and skips the next: 1, 1, 3.
DENSE_RANK gives ties the same number and does not skip: 1, 1, 2.
Revise: day-05-sql-advanced.md

**Q.** [Mid] How is a window function different from GROUP BY?
GROUP BY collapses many rows into one row per group. A window function keeps every row and adds a computed value.
Example: show each order and the customer's total at the same time.
The window is defined with OVER (PARTITION BY ... ORDER BY ...).
Revise: day-05-sql-advanced.md

**Q.** [Senior] What is the default frame of `SUM(x) OVER (ORDER BY d)`, and what trap does it have?
The default frame is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW.
With RANGE, rows that tie on d are peers, so they all get the same running total.
If you want a strict row-by-row total, write `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, or add a unique tie-breaker.
Revise: day-05-sql-advanced.md

**Q.** [Mid] How do you write an upsert in PostgreSQL?
Use `INSERT ... ON CONFLICT (key) DO UPDATE SET ...` or `DO NOTHING`.
It is atomic, so it avoids the race of "select, then insert or update".
The conflict target must match a unique index or constraint.
Revise: day-04-sql-core.md

**Q.** [Fresher] DELETE, TRUNCATE and DROP: what are the differences?
DELETE removes chosen rows, can have a WHERE, fires row triggers, and is fully logged.
TRUNCATE removes all rows quickly and keeps the table. DROP removes the table itself.
In PostgreSQL, TRUNCATE can be rolled back inside a transaction, but it takes a strong lock.
Revise: day-04-sql-core.md

## C. Storage and indexes

**Q.** [Mid] What is a page, and why does it matter?
A page (or block) is the fixed unit of disk and memory I/O. PostgreSQL uses 8 KB by default.
Rows live inside pages. A query reads whole pages, not single rows.
So fewer, narrower rows per page mean fewer reads, and row width matters for speed.
Revise: day-06-storage-layout.md

**Q.** [Mid] What is the difference between a heap table and a clustered (index-organised) table?
A PostgreSQL table is a heap: rows sit in no special order, and indexes point to them.
In MySQL InnoDB, the table is stored in primary key order, so the primary key is a clustered index.
In a clustered table, the primary key choice affects every secondary index.
Revise: day-06-storage-layout.md

**Q.** [Mid] What is TOAST in PostgreSQL?
TOAST (The Oversized-Attribute Storage Technique) stores large values, such as long text or JSON, outside the main row.
It compresses them or moves them to a side table, so the main row stays small.
Selecting those wide columns costs extra reads. Do not use SELECT * if you do not need them.
Revise: day-06-storage-layout.md

**Q.** [Fresher] What is an index and why is it fast?
An index is a separate sorted structure that maps column values to row locations.
The database finds a row by walking a tree, not by reading the whole table.
The cost is more storage and slower INSERT, UPDATE and DELETE.
Revise: day-07-indexes-deep.md

**Q.** [Mid] Compare B-tree, hash, GIN, GiST and BRIN indexes.
B-tree: default, good for equality, ranges and sorting. Hash: equality only.
GIN (generalised inverted index): many values per row, such as arrays, JSONB and full-text search.
GiST: geometry and range types. BRIN (block range index): tiny index for huge tables whose values follow disk order, such as timestamps.
Revise: day-07-indexes-deep.md

**Q.** [Mid] Which queries use an index on (country, city, created_at)?
Filters on country; on country and city; and on all three. A filter only on city or only on created_at cannot use it well.
That is the leftmost-prefix rule. Sorting by created_at after equality on country and city can also use the index.
Revise: day-07-indexes-deep.md

**Q.** [Mid] How do you choose column order in a composite index?
Put equality columns first. Put the range or ORDER BY column last.
Among equality columns, put the more commonly used column first. Selectivity matters, but query shape matters more.
Check the plan with EXPLAIN on real queries.
Revise: day-07-indexes-deep.md and day-16-index-tuning-n-plus-1.md

**Q.** [Mid] What is a covering index and an index-only scan?
A covering index contains every column the query needs. Then the database reads only the index, not the table.
In PostgreSQL, use `INCLUDE (cols)` for extra payload columns.
It works well only if the visibility map is current, so regular VACUUM matters.
Revise: day-07-indexes-deep.md

**Q.** [Mid] When is a partial index useful?
When you query a small, known slice of the table, for example `WHERE status = 'pending'`.
`CREATE INDEX ON jobs (created_at) WHERE status = 'pending'` is small and fast.
The query must include a predicate that implies the index predicate, or the planner will not use it.
Revise: day-07-indexes-deep.md

**Q.** [Mid] Why does `WHERE lower(email) = 'a@b.com'` ignore an index on email?
The index stores email, not lower(email). The planner cannot match the function result to it.
Create an expression index: `CREATE INDEX ON users (lower(email))`. Or use the citext type.
Revise: day-07-indexes-deep.md and day-15-query-optimization.md

**Q.** [Mid] Name other reasons an index is not used.
Type mismatch (a text column compared to a number), a leading wildcard like `LIKE '%abc'`, low selectivity, and stale statistics.
Also an OR across different columns, or a very small table where a sequential scan is cheaper.
Use EXPLAIN to see the planner's choice, and ANALYZE to refresh statistics.
Revise: day-08-when-indexes-hurt.md

**Q.** [Mid] How can you speed up `LIKE '%term%'` searches?
A normal B-tree cannot help with a leading wildcard. Use a trigram index (pg_trgm extension) with GIN or GiST.
For real text search, use full-text search with a GIN index on a tsvector.
Revise: day-07-indexes-deep.md

**Q.** [Mid] How do indexes slow down writes?
Every insert must add an entry to each index. An update that changes an indexed column updates that index too.
This is called write amplification. It also creates dead index entries that need cleanup.
So index only what your queries really use.
Revise: day-08-when-indexes-hurt.md

**Q.** [Senior] Scenario: design an index for `SELECT * FROM orders WHERE customer_id = ? AND status = 'open' ORDER BY created_at DESC LIMIT 20`.
Use `CREATE INDEX ON orders (customer_id, status, created_at DESC)`. Two equality columns come first, then the sort column.
The database reads the first 20 entries in order and stops, with no sort step.
If most queries use status 'open', a partial index `WHERE status = 'open'` on (customer_id, created_at DESC) is smaller.
Revise: day-07-indexes-deep.md and day-16-index-tuning-n-plus-1.md

**Q.** [Senior] How do you find unused or duplicate indexes safely?
Look at `pg_stat_user_indexes` for indexes with idx_scan near zero over a full business cycle (including month-end jobs).
Duplicates have the same columns in the same order. Check replicas too, because stats are per server.
Drop in a quiet window; with CONCURRENTLY, and keep the CREATE statement so you can rebuild.
Revise: day-08-when-indexes-hurt.md

**Q.** [Senior] What happens to a B-tree when the primary key is a random UUID?
Inserts land at random places in the tree. This causes page splits, poor cache use, and more WAL (write-ahead log) volume.
Time-ordered keys (UUIDv7, or bigint identity) append at the right edge and behave better.
Random UUIDs are fine for small tables; the cost shows on large, write-heavy tables.
Revise: day-07-indexes-deep.md and day-02-data-modeling-keys.md

## D. Query planner and optimisation

**Q.** [Fresher] What are the steps from SQL text to result?
Parse (check syntax), rewrite (expand views), plan (choose the cheapest way), then execute.
The planner is cost-based: it estimates cost using table statistics.
Revise: day-09-query-execution-planner.md

**Q.** [Fresher] What is the difference between EXPLAIN and EXPLAIN ANALYZE? Any danger?
EXPLAIN only estimates. EXPLAIN ANALYZE actually runs the query and shows real time and rows.
Danger: for INSERT, UPDATE or DELETE it really changes data. Wrap it in BEGIN and ROLLBACK.
Add BUFFERS to see page reads.
Revise: day-09-query-execution-planner.md

**Q.** [Mid] Explain sequential scan, index scan and bitmap scan.
Sequential scan reads the whole table. Index scan follows the index and fetches rows one by one.
Bitmap scan collects matching row locations from the index, sorts them by page, then reads pages in order.
It is good when many rows match, but not enough for a sequential scan.
Revise: day-09-query-execution-planner.md

**Q.** [Mid] What are the three join algorithms and when does each win?
Nested loop: good when the outer side is small and the inner side has an index.
Hash join: builds a hash table of one input; good for large, unsorted equality joins.
Merge join: both inputs sorted on the join key; good for large sorted inputs.
Revise: day-09-query-execution-planner.md

**Q.** [Mid] The plan shows estimated rows = 10 but actual rows = 2,000,000. What does it mean?
The planner's statistics are wrong, so it picked a poor plan, such as a nested loop on a huge input.
Run ANALYZE. If the data is skewed or columns are correlated, raise the statistics target or create extended statistics.
Revise: day-15-query-optimization.md

**Q.** [Mid] What does "sargable" mean? Give examples.
A sargable predicate (search-argument-able) lets the engine use an index.
Bad: `WHERE date(created_at) = '2025-01-01'` or `WHERE amount + 1 > 10`.
Good: `WHERE created_at >= '2025-01-01' AND created_at < '2025-01-02'` and `WHERE amount > 9`.
Revise: day-15-query-optimization.md

**Q.** [Mid] Why is `OFFSET 100000 LIMIT 20` slow, and what is better?
The database must read and throw away 100,000 rows before it returns 20.
Use keyset pagination (seek method): `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20`.
It needs a matching index and gives stable speed on any page.
Revise: day-15-query-optimization.md

**Q.** [Senior] What are extended statistics, and when do you need them?
By default, the planner treats columns as independent. If city and zip are strongly linked, it multiplies selectivities and under-estimates rows.
`CREATE STATISTICS ... (dependencies) ON city, zip FROM t` lets it learn the link. Run ANALYZE afterwards.
Revise: day-15-query-optimization.md

**Q.** [Senior] Scenario: a query got slow after a deploy. What do you check?
1. Identify the query with pg_stat_statements or slow-query logs. Compare time before and after.
2. Run EXPLAIN (ANALYZE, BUFFERS). Look for a new sequential scan, a big estimate gap, or a sort spilling to disk.
3. Check what changed: new ORM code (N+1), a function on a column, a dropped or invalid index, a migration that took locks.
4. Check statistics freshness, bloat, and waiting locks (pg_stat_activity).
5. Fix the cause, then measure again. Do not add random indexes.
Revise: day-15-query-optimization.md and day-29-security-observability.md

**Q.** [Mid] Why avoid SELECT * in application code?
It reads and sends columns you do not need, including TOASTed (large) ones.
It blocks index-only scans. It also breaks code when a column is added or the order changes.
Name the columns you need.
Revise: day-15-query-optimization.md

## E. Transactions, MVCC, locking, vacuum

**Q.** [Fresher] What is a transaction? Explain each ACID letter with an example.
A transaction is a group of statements that succeed or fail as one unit.
Bank transfer: both the debit and the credit happen (atomicity), balances never go below the CHECK rule (consistency),
another session sees either the old or the new state (isolation), and a committed transfer survives a crash (durability).
Revise: day-11-acid-in-depth.md

**Q.** [Mid] What is the buffer pool (shared buffers)?
It is a memory cache of table and index pages. Reads check memory first; a miss goes to disk.
Changed pages stay in memory as "dirty" and are written later by the background writer and checkpoints.
A good cache hit ratio is key to speed.
Revise: day-10-buffer-wal-mvcc.md

**Q.** [Mid] What is the WAL and why is it written first?
The write-ahead log records each change before the data page changes. A commit waits only for the log flush to disk.
Log writes are sequential and fast. After a crash, the database replays the log to restore a consistent state.
WAL is also the base for replication and point-in-time recovery.
Revise: day-10-buffer-wal-mvcc.md and day-11-acid-in-depth.md

**Q.** [Mid] What is a checkpoint?
A checkpoint writes all dirty pages to disk and marks a point in the WAL. Crash recovery starts from the last checkpoint.
Frequent checkpoints shorten recovery but cause more I/O. Rare ones do the reverse and use more WAL space.
Revise: day-10-buffer-wal-mvcc.md

**Q.** [Mid] How does the WAL give atomicity and durability?
Durability: the commit record is flushed to disk before the client gets "OK".
Atomicity: on recovery, changes of transactions with no commit record are ignored (in PostgreSQL, their versions stay invisible).
Revise: day-11-acid-in-depth.md

**Q.** [Fresher] What is MVCC and what problem does it solve?
It keeps multiple row versions so each transaction sees a consistent snapshot.
Readers do not block writers, and writers do not block readers. Only two writers on the same row wait for each other.
The price is dead row versions that need cleaning.
Revise: day-14-mvcc-vacuum.md

**Q.** [Fresher] Explain dirty read, non-repeatable read and phantom read.
Dirty read: you see data that another transaction has not committed.
Non-repeatable read: you read a row twice and get different values, because another transaction committed in between.
Phantom read: you run the same range query twice and get different sets of rows.
Revise: day-12-isolation-levels-anomalies.md

**Q.** [Mid] What are the default isolation levels in PostgreSQL and MySQL InnoDB, and how does PostgreSQL Repeatable Read behave?
PostgreSQL default: Read Committed. MySQL InnoDB default: Repeatable Read.
In PostgreSQL, Repeatable Read uses one snapshot for the whole transaction, so it also prevents phantoms.
If two such transactions update the same row, one fails with a serialization error, and you must retry.
Revise: day-12-isolation-levels-anomalies.md

**Q.** [Senior] What is write skew? Give an example and a fix.
Two transactions read the same data, each decides it is safe, and each writes a different row, which together break a rule.
Example: two doctors both check "at least one doctor is on call" and both go off call.
Repeatable Read does not stop it. Use Serializable (and retry), or lock the shared rows with SELECT FOR UPDATE.
Revise: day-12-isolation-levels-anomalies.md

**Q.** [Mid] How do you avoid a lost update?
Lost update: two sessions read a value, both add to it, and one write overwrites the other.
Best fix: do it in one statement, `UPDATE acc SET bal = bal - 10 WHERE id = 1`.
Other fixes: SELECT FOR UPDATE, optimistic locking with a version column, or Repeatable Read with retry.
Revise: day-12-isolation-levels-anomalies.md and day-13-locking-deadlocks.md

**Q.** [Mid] What are SELECT FOR UPDATE and SKIP LOCKED used for?
FOR UPDATE locks the selected rows so others cannot change them until you commit.
SKIP LOCKED skips rows that other sessions already locked.
Together they make a safe job queue: many workers each grab different rows without waiting.
Revise: day-13-locking-deadlocks.md

**Q.** [Mid] How does a deadlock happen and how do you reduce them?
Session A locks row 1 then wants row 2. Session B locks row 2 then wants row 1. Both wait.
PostgreSQL detects this after deadlock_timeout (default 1 second) and aborts one session.
Reduce them: lock in a fixed order, keep transactions short, and retry on the deadlock error.
Revise: day-13-locking-deadlocks.md

**Q.** [Senior] Scenario: the app is hanging. How do you find who blocks whom?
Query `pg_stat_activity` for sessions with a wait event of type Lock, and use `pg_blocking_pids(pid)` to find the blocker.
`pg_locks` shows the lock details. Often the blocker is "idle in transaction", an open transaction that did nothing.
Fix the app code first. Use `pg_terminate_backend` only as an emergency step, and set idle_in_transaction_session_timeout.
Revise: day-13-locking-deadlocks.md

**Q.** [Mid] Why is a long "idle in transaction" session dangerous?
It holds locks and, more importantly, an old snapshot. VACUUM cannot remove dead rows newer than that snapshot.
So tables bloat and queries slow down. Set timeouts and keep transactions short.
Revise: day-14-mvcc-vacuum.md and day-13-locking-deadlocks.md

**Q.** [Mid] What do dead tuples and bloat mean? What does VACUUM do?
An UPDATE or DELETE leaves the old row version in place. These are dead tuples (unneeded old row versions).
VACUUM marks their space reusable, updates the visibility map and freezes old transaction IDs.
Bloat is the wasted space when dead tuples are not cleaned fast enough.
Revise: day-14-mvcc-vacuum.md and day-08-when-indexes-hurt.md

**Q.** [Mid] What is the difference between VACUUM and VACUUM FULL?
Plain VACUUM runs alongside normal traffic and makes space reusable inside the table, but usually does not shrink the file.
VACUUM FULL rewrites the table into a new file, returns space to the OS, and takes an ACCESS EXCLUSIVE lock (blocks all access).
Prefer plain VACUUM. For online repair of bloat, use tools such as pg_repack.
Revise: day-14-mvcc-vacuum.md

**Q.** [Senior] How do you tune autovacuum for a very busy, big table?
By default, it starts after about 20 percent of rows change (autovacuum_vacuum_scale_factor = 0.2). On huge tables that is too late.
Set per-table values: a lower scale factor, a fixed threshold, and a higher cost limit.
Also check for old transactions and replication slots that hold back cleanup.
Revise: day-14-mvcc-vacuum.md

**Q.** [Senior] What is transaction ID wraparound?
PostgreSQL transaction IDs are 32-bit and compared in a circle. After about 2 billion transactions, old rows could look as if they are in the future.
Autovacuum "freezes" old rows to prevent this. If it falls behind, the database warns and finally stops accepting writes.
Monitor the age of datfrozenxid and fix blocked vacuums early.
Revise: day-14-mvcc-vacuum.md

**Q.** [Senior] What are HOT updates and fillfactor?
A HOT (heap-only tuple) update stores the new row version on the same page and needs no new index entries.
It works only if no indexed column changes and the page has free room.
Lowering fillfactor (for example 80) leaves room on each page, which helps update-heavy tables.
Revise: day-14-mvcc-vacuum.md and day-06-storage-layout.md

## F. Performance in practice

**Q.** [Fresher] What is the N+1 problem and how do you detect it?
One query loads N parent rows, then the code runs one query per parent for its children.
Signs: hundreds of similar fast queries in one request, with high total time. Use query logs, pg_stat_statements, or ORM debug output.
Fix: join, `WHERE parent_id = ANY(...)`, or eager loading.
Revise: day-16-index-tuning-n-plus-1.md

**Q.** [Mid] Why use a connection pooler such as PgBouncer?
Each PostgreSQL connection is a process that uses memory. Thousands of connections slow the server.
A pooler keeps a small set of real connections and shares them among many clients.
In transaction pooling mode, session features like prepared statements and SET need care.
Revise: day-16-index-tuning-n-plus-1.md

**Q.** [Mid] What is table partitioning and what is partition pruning?
Partitioning splits one large table into smaller child tables by range, list or hash of a key.
Pruning means the planner skips partitions that cannot match the filter. This needs the partition key in the WHERE clause.
Revise: day-17-partitioning-maintenance.md

**Q.** [Mid] Partitioning, indexing or sharding: how do you choose?
First fix queries and indexes. Use partitioning when a table is huge and you drop or archive old data by time.
Use sharding only when one server can no longer handle the write load or storage.
Partitioning does not add servers; sharding does.
Revise: day-17-partitioning-maintenance.md and day-22-sharding.md

**Q.** [Mid] How do partitions help with deleting old data?
With monthly range partitions, you detach or drop the oldest partition. This is instant, and creates no dead rows.
A big DELETE would be slow, create bloat and generate lots of WAL.
Revise: day-17-partitioning-maintenance.md

**Q.** [Mid] Compare cache-aside and write-through caching.
Cache-aside: the app checks the cache, on a miss reads the database and fills the cache. Simple and common.
Write-through: every write goes to the cache and the database together. The cache is fresher but writes are slower.
Both need an invalidation plan (TTL, or delete on write).
Revise: day-18-caching-materialized-views.md

**Q.** [Senior] What is a cache stampede and how do you reduce it?
A popular key expires and many requests miss at once, so all of them hit the database.
Reduce it with a lock so only one request rebuilds the value, random TTL jitter, or refreshing before expiry.
Revise: day-18-caching-materialized-views.md

**Q.** [Mid] What is a materialized view and how do you refresh it without blocking readers?
A materialized view stores the result of a query on disk. It is fast to read but can be stale.
`REFRESH MATERIALIZED VIEW` locks readers. `REFRESH MATERIALIZED VIEW CONCURRENTLY` does not, but needs a unique index on the view.
Revise: day-18-caching-materialized-views.md

**Q.** [Mid] What mistakes make a benchmark misleading?
Too-small data (everything fits in cache), too-short runs, a cold cache or a warm cache not stated, and averages instead of percentiles.
Also a test client that is slower than the database, and unrealistic query mixes.
Report p95 and p99 latency, change one thing at a time, and repeat the run.
Revise: day-19-benchmarking.md

**Q.** [Senior] How do you find the bottleneck: CPU, memory, disk or locks?
Look at CPU and I/O wait on the host, then cache hit ratio, then wait events in pg_stat_activity.
High I/O wait with low hit ratio points to memory or missing indexes. Many Lock waits point to contention.
Test the change with a repeatable load, such as pgbench, before and after.
Revise: day-19-benchmarking.md and day-29-security-observability.md

## G. Replication, scaling, sharding, CAP

**Q.** [Mid] What is the difference between synchronous and asynchronous replication?
Synchronous: the primary waits until a replica confirms the write. No data loss on failover, but higher commit latency, and a dead replica can stall writes.
Asynchronous: the primary does not wait. It is faster, but the replica may be behind, so a failover can lose recent commits.
Revise: day-20-replication.md

**Q.** [Mid] What is the difference between physical and logical replication?
Physical (streaming) copies WAL bytes, so the replica is a full copy of the whole cluster. It must run the same major version.
Logical sends row changes by table, so you can copy part of the data, or replicate across versions.
Revise: day-20-replication.md

**Q.** [Mid] What is replication lag and the "read-your-writes" problem?
Lag is the delay before a change reaches a replica. A user saves data, then reads from a lagging replica and sees old data.
Fixes: read from the primary for a short time after a write, route by session, or wait for the replica to reach a given WAL position.
Revise: day-20-replication.md

**Q.** [Senior] What is split brain, and how do HA tools avoid it?
Split brain means two nodes both think they are primary and accept writes. Data then diverges and is hard to merge.
Tools use a consensus store (a quorum vote), leader leases and fencing (shutting the old primary down) to allow only one primary.
Revise: day-21-read-write-scaling-ha.md

**Q.** [Mid] Why do read replicas not scale writes?
Every write still goes to one primary, and every replica must replay all of those writes.
Replicas add read capacity and safety. For more write capacity, you need better hardware, less write work, or sharding.
Revise: day-21-read-write-scaling-ha.md

**Q.** [Senior] How do you choose a shard key?
Pick a key with high cardinality (many values), even spread, and which appears in most queries, such as tenant_id or user_id.
Avoid keys that create hot shards, such as a date for fresh data. Queries without the key must contact all shards.
Plan for resharding at the start.
Revise: day-22-sharding.md

**Q.** [Senior] What are the pains of sharding?
Cross-shard joins and transactions are slow or limited. Unique constraints and sequences are hard across shards.
Resharding moves data and needs care. Operations, backup and schema changes multiply by the shard count.
So delay sharding until simpler steps no longer work.
Revise: day-22-sharding.md

**Q.** [Senior] Compare hash, range and directory-based sharding.
Hash: spreads data evenly, but range scans hit all shards, and adding shards moves much data unless you use consistent hashing.
Range: good for range scans, but can create hot spots.
Directory (lookup table): flexible, but the lookup service is a new dependency.
Revise: day-22-sharding.md

**Q.** [Mid] Explain the CAP theorem correctly.
When a network partition happens, a system can keep consistency (refuse some requests) or availability (answer, maybe with stale data).
"Partition tolerance" is not optional in a real network. So the real choice is CP or AP during a partition.
Revise: day-23-cap-consistency.md

**Q.** [Senior] What does PACELC add?
If there is a Partition, choose Availability or Consistency. Else (normal operation), choose Latency or Consistency.
It shows that even without failures, strong consistency costs latency, since nodes must coordinate.
Revise: day-23-cap-consistency.md

**Q.** [Senior] Explain quorum reads and writes. What does R + W > N mean?
With N replicas, a write must reach W of them and a read must ask R of them.
If R + W > N, every read set overlaps every write set, so a read sees at least one copy of the latest write.
Higher W or R means stronger consistency but more latency.
Revise: day-23-cap-consistency.md

## H. NoSQL and choosing a store

**Q.** [Mid] When would you choose a document database over PostgreSQL?
When data is naturally one nested object, the shape changes often, and you read it as a whole, such as a product catalogue or a content page.
If you need many joins, strong multi-row transactions and reports, a relational database fits better.
PostgreSQL also has JSONB, so mixed designs are common.
Revise: day-24-document-keyvalue.md

**Q.** [Mid] Embedding or referencing in a document model?
Embed when the child data is read with the parent, is bounded in size, and does not change alone, such as address lines.
Reference when the child is large, unbounded, shared, or updated on its own, such as comments on a popular post.
Model for your access patterns first.
Revise: day-24-document-keyvalue.md

**Q.** [Fresher] What are typical uses of a key-value store such as Redis?
Cache, sessions, counters, rate limits, leaderboards (sorted sets), and simple queues.
Access is by key and very fast, because data is in memory. It is not for complex queries.
Check the persistence settings before you store data that you cannot lose.
Revise: day-24-document-keyvalue.md

**Q.** [Mid] How do you model data in a wide-column database such as Cassandra?
Start from the queries. Each table is built for one query, and the partition key decides which node stores the rows.
Rows in a partition are sorted by clustering columns. Data is duplicated across tables on purpose.
Avoid very large partitions and joins.
Revise: day-25-widecolumn-graph-newsql.md

**Q.** [Mid] When is a graph database a good fit?
When relationships are the main question: friends of friends, fraud rings, recommendations, route finding.
In SQL these need many self-joins or recursive CTEs, which get slow with depth. A graph database follows links directly.
Revise: day-25-widecolumn-graph-newsql.md

**Q.** [Mid] What is NewSQL (distributed SQL)?
It gives SQL and ACID transactions on a database that spreads across many nodes, usually with consensus replication.
It aims to give horizontal scale without giving up the relational model.
The cost is higher latency for cross-node writes, so test your workload.
Revise: day-25-widecolumn-graph-newsql.md

**Q.** [Senior] Scenario: you must pick a store for a shopping app with orders, a product catalogue, sessions and search. What do you choose?
Orders and payments: PostgreSQL (ACID, constraints). Sessions and cache: Redis. Catalogue with flexible attributes: JSONB in PostgreSQL first.
Search: PostgreSQL full-text, or a search engine if needs grow.
Start with the fewest systems. Each new store adds operations work and sync problems (polyglot persistence).
Revise: day-26-choosing-store-vector-dbs.md

**Q.** [Mid] What is a vector database and when do you use pgvector?
It stores embeddings (lists of numbers that represent meaning) and finds nearest neighbours, for semantic search and RAG.
Exact search is slow at scale, so indexes such as HNSW or IVFFlat give approximate nearest neighbour (ANN) results.
pgvector keeps vectors next to your relational data. A dedicated store may suit very large scale.
Revise: day-26-choosing-store-vector-dbs.md

## I. Operations: backup, migration, security, warehouse

**Q.** [Fresher] What are RPO and RTO?
RPO (recovery point objective): how much data you can afford to lose, measured in time.
RTO (recovery time objective): how long you can afford to be down.
They decide your backup type, frequency and replication design.
Revise: day-27-backups-recovery.md

**Q.** [Mid] Compare logical and physical backups.
Logical (pg_dump): SQL or data export. It is portable across versions and can restore one table, but is slow for big databases.
Physical: copy of data files plus WAL. It is fast to restore and supports point-in-time recovery, but it is tied to the major version.
Revise: day-27-backups-recovery.md

**Q.** [Senior] What is point-in-time recovery (PITR)?
You keep a base backup and a continuous archive of WAL. To recover, restore the base backup and replay WAL up to a chosen time.
Use it to undo a bad DELETE made at 14:03 by recovering to 14:02. Replication does not help here, because the bad change is copied too.
Revise: day-27-backups-recovery.md

**Q.** [Fresher] A team has backups but never tested a restore. What is the risk?
A backup that was never restored is not proven. It may be corrupt, incomplete or too slow to meet the RTO.
Run regular restore tests and time them. Write and practise the runbook (step-by-step guide).
Revise: day-27-backups-recovery.md

**Q.** [Senior] Scenario: add a NOT NULL column to a table with 500 million rows without downtime.
1. `ALTER TABLE t ADD COLUMN c type;` (nullable, no volatile default). This is a metadata change.
2. Deploy app code that writes c for new rows.
3. Backfill old rows in small batches (for example 10,000 rows per batch, with short pauses).
4. `ADD CONSTRAINT ... CHECK (c IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT`, which allows writes.
5. `ALTER COLUMN c SET NOT NULL` (PostgreSQL 12+ uses the valid check), then drop the check. Set lock_timeout on each step.
Note: a constant default with ADD COLUMN is also a fast metadata change in PostgreSQL 11 and later.
Revise: day-28-zero-downtime-migrations.md

**Q.** [Mid] How do you create an index on a busy table without blocking writes?
Use `CREATE INDEX CONCURRENTLY`. It takes longer and cannot run inside a transaction block.
If it fails, it leaves an INVALID index. Drop it and try again.
A plain CREATE INDEX blocks writes for the whole build.
Revise: day-28-zero-downtime-migrations.md

**Q.** [Senior] What is the expand and contract pattern?
Expand: add the new structure beside the old one (new column or table), and make the app write to both.
Migrate: backfill old data and move reads to the new structure.
Contract: remove the old structure after all code uses the new one. Each step is safe to roll back.
Revise: day-28-zero-downtime-migrations.md

**Q.** [Senior] Why is `lock_timeout` important for migrations?
An ALTER TABLE needs a strong lock. If it waits behind a long query, all later queries queue behind the ALTER, and the app freezes.
Set a short lock_timeout, so the migration fails quickly, then retry at a calm time.
Revise: day-28-zero-downtime-migrations.md

**Q.** [Mid] What is least privilege and row-level security (RLS)?
Least privilege: each role gets only the permissions it needs, such as an app role without DROP or superuser rights.
RLS adds policies so a role sees only certain rows, for example only its own tenant. The database enforces it, not the app.
Revise: day-29-security-observability.md

**Q.** [Fresher] How do you prevent SQL injection?
Use parameterised queries (prepared statements). Never build SQL by joining user input into a string.
Also use least-privilege roles and validate identifiers from a fixed list when they must be dynamic.
Revise: day-29-security-observability.md

**Q.** [Mid] Which database metrics and alerts matter most?
Query latency (p95, p99), connections used, cache hit ratio, replication lag, lock waits, bloat and autovacuum health, disk space and WAL size.
Alert on symptoms users feel and on things that run out, such as disk.
Enable pg_stat_statements to find the top queries by total time.
Revise: day-29-security-observability.md

**Q.** [Mid] Why are analytic queries not run on the main OLTP database?
They scan much data and can use all I/O and CPU, slowing the app. They also hold old snapshots, which delay vacuum.
Use a read replica for light reports, or a warehouse for heavy ones.
Revise: day-30-warehousing-cdc-capstone.md and day-01-why-databases-oltp-olap.md

**Q.** [Mid] Why is columnar storage fast for analytics?
Data is stored by column, so a query reading 3 of 100 columns reads only those 3.
Similar values sit together, so compression is strong. Engines also process data in batches.
Row storage is better for fetching or changing a whole row.
Revise: day-30-warehousing-cdc-capstone.md

**Q.** [Mid] What is CDC (change data capture) and why use it?
CDC reads the database's change log (WAL) and sends each insert, update and delete as an event, for example to Kafka.
It feeds a warehouse, search index or cache with low load and low delay. It is better than repeated full table copies.
Watch for replication slots that keep WAL on disk if the consumer is down.
Revise: day-30-warehousing-cdc-capstone.md

**Q.** [Senior] Compare B-tree and LSM-tree storage engines.
B-tree updates pages in place: good reads, steady write cost, more random I/O.
LSM-tree (log-structured merge tree) writes to memory and appends sorted files, then merges them (compaction). Writes are fast, but reads may check many files, and compaction costs I/O.
Use LSM for write-heavy loads. Use B-tree for balanced, read-heavy loads.
Revise: advanced-track-db-systems.md (A3 Storage Engine Design) and day-25-widecolumn-graph-newsql.md

**Q.** [Senior] What role does a consensus algorithm such as Raft play in a database?
It lets a group of nodes agree on one ordered log, even when some nodes fail. A leader is elected, and a write counts after a majority stores it.
Distributed SQL stores and HA tools use it for replication and leader choice.
It needs a majority (for example 2 of 3), so it tolerates a minority of failures.
Revise: advanced-track-db-systems.md (A2 Distributed Database Algorithms)

**Q.** [Senior] How does the query optimiser pick a join order, and why can it fail?
It estimates the cost of many orders, using statistics, with dynamic programming for small joins. For many tables it uses heuristics, such as PostgreSQL's genetic optimiser above a threshold.
It fails when row estimates are wrong, especially after several joins, since errors multiply.
Fixes: fresh and extended statistics, simpler queries, or splitting a query with a temporary table.
Revise: advanced-track-db-systems.md (A7 Advanced Optimizer Theory) and day-09-query-execution-planner.md

**Q.** [Senior] What is the difference between a database being "consistent" in ACID and in CAP?
ACID consistency means a transaction moves data from one valid state to another, so constraints hold.
CAP consistency means linearizability: all clients see the latest write in a single order, as if there were one copy.
They are different ideas that share a name.
Revise: day-23-cap-consistency.md and day-11-acid-in-depth.md

**Q.** [Senior] How do you reduce the risk of a failover?
Test it often. Use automatic failover with a quorum, set sensible timeouts, and check replication lag before promoting.
Applications must reconnect and retry. Plan for the old primary to rejoin as a replica, and rebuild it if it diverged.
Revise: day-21-read-write-scaling-ha.md and day-20-replication.md

**Q.** [Mid] What is a covering approach to reduce I/O for a hot read path?
Check whether a small index with INCLUDE can answer the query without reading the table.
Otherwise cache the result in Redis, or precompute it in a materialized view, depending on how fresh the data must be.
Revise: day-07-indexes-deep.md and day-18-caching-materialized-views.md

---

# 3. SQL practice problems (25)

Use this schema for all problems. Types are PostgreSQL.

```sql
employees   (emp_id int PK, name text, dept_id int, manager_id int, salary numeric, hired_on date)
departments (dept_id int PK, dept_name text)
customers   (customer_id int PK, name text, email text, country text, created_at timestamptz)
orders      (order_id int PK, customer_id int, order_date date, status text, total numeric)
order_items (order_id int, product_id int, qty int, unit_price numeric)
products    (product_id int PK, name text, category text)
logins      (user_id int, login_date date)   -- may repeat the same day
```

Each problem has the approach first, then the query. Try it yourself before reading.

**P1. Second highest salary.**
Approach: take the maximum of the salaries that are lower than the overall maximum. It returns NULL if there is no second value.

```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

**P2. Top 3 salaries in each department (ties share a rank).**
Approach: DENSE_RANK partitioned by department, then filter in an outer query. You cannot filter on a window function in the same level.

```sql
SELECT dept_id, name, salary
FROM (
  SELECT dept_id, name, salary,
         DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
  FROM employees
) t
WHERE rnk <= 3
ORDER BY dept_id, salary DESC;
```

**P3. Find duplicate emails.**
Approach: group by email and keep groups with more than one row.

```sql
SELECT email, COUNT(*) AS cnt
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;
```

**P4. Delete duplicate customers, keeping the lowest customer_id.**
Approach: join the table to itself on the same email and delete the row that has the larger id.

```sql
DELETE FROM customers c
USING customers d
WHERE c.email = d.email
  AND c.customer_id > d.customer_id;
```

**P5. Customers who never placed an order.**
Approach: NOT EXISTS is safe with NULLs and easy to read. A LEFT JOIN with `o.order_id IS NULL` also works.

```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);
```

**P6. Departments with more than 5 employees, with their average salary.**
Approach: group by department, filter groups with HAVING, and join for the name.

```sql
SELECT d.dept_name, COUNT(*) AS headcount, AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON d.dept_id = e.dept_id
GROUP BY d.dept_name
HAVING COUNT(*) > 5;
```

**P7. Employees who earn more than their manager.**
Approach: self-join. One copy is the employee, the other is the manager.

```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON m.emp_id = e.manager_id
WHERE e.salary > m.salary;
```

**P8. Employees who earn more than the average of their own department.**
Approach: use a window average per department, then filter in an outer query.

```sql
SELECT name, dept_id, salary
FROM (
  SELECT name, dept_id, salary,
         AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg
  FROM employees
) t
WHERE salary > dept_avg;
```

**P9. Running total of revenue per day.**
Approach: first sum per day, then a window SUM over those daily rows. Use ROWS to avoid tie issues.

```sql
WITH daily AS (
  SELECT order_date, SUM(total) AS revenue
  FROM orders
  GROUP BY order_date
)
SELECT order_date, revenue,
       SUM(revenue) OVER (ORDER BY order_date
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM daily
ORDER BY order_date;
```

**P10. Month-over-month revenue growth in percent.**
Approach: sum by month, use LAG to get the previous month, and guard against divide by zero with NULLIF.

```sql
WITH monthly AS (
  SELECT date_trunc('month', order_date)::date AS month, SUM(total) AS revenue
  FROM orders
  GROUP BY 1
)
SELECT month, revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
             / NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) AS growth_pct
FROM monthly
ORDER BY month;
```

**P11. Top 3 products by revenue in each category.**
Approach: compute revenue per product, rank inside each category, then keep rank 3 or lower.

```sql
WITH rev AS (
  SELECT p.category, p.name, SUM(oi.qty * oi.unit_price) AS revenue
  FROM order_items oi
  JOIN products p ON p.product_id = oi.product_id
  GROUP BY p.category, p.name
)
SELECT category, name, revenue
FROM (
  SELECT *, RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
  FROM rev
) t
WHERE rnk <= 3;
```

**P12. First order of each customer.**
Approach: in PostgreSQL, DISTINCT ON keeps the first row per group by the ORDER BY. The portable form uses ROW_NUMBER.

```sql
SELECT DISTINCT ON (customer_id) customer_id, order_id, order_date
FROM orders
ORDER BY customer_id, order_date, order_id;
```

**P13. Longest login streak (in days) per user.**
Approach: this is a "gaps and islands" problem. Remove duplicate days. Subtract the row number (in days) from the date. Days in one streak give the same result.

```sql
WITH d AS (
  SELECT DISTINCT user_id, login_date FROM logins
),
g AS (
  SELECT user_id, login_date,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM d
)
SELECT user_id, MAX(streak) AS longest_streak
FROM (
  SELECT user_id, grp, COUNT(*) AS streak
  FROM g
  GROUP BY user_id, grp
) s
GROUP BY user_id;
```

**P14. Users who logged in on at least 3 consecutive days.**
Approach: same island idea, but keep islands with a size of 3 or more.

```sql
WITH d AS (
  SELECT DISTINCT user_id, login_date FROM logins
),
g AS (
  SELECT user_id,
         login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
  FROM d
)
SELECT DISTINCT user_id
FROM g
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

**P15. Count of orders per status for each month (pivot).**
Approach: conditional aggregation with FILTER. Each status becomes a column.

```sql
SELECT date_trunc('month', order_date)::date AS month,
       COUNT(*) FILTER (WHERE status = 'paid')      AS paid,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
       COUNT(*) FILTER (WHERE status = 'open')      AS open_orders
FROM orders
GROUP BY 1
ORDER BY 1;
```

**P16. Find gaps in order_id (missing numbers).**
Approach: for each id, look at the next id with LEAD. If the difference is more than 1, there is a gap.

```sql
SELECT order_id + 1 AS gap_start, next_id - 1 AS gap_end
FROM (
  SELECT order_id, LEAD(order_id) OVER (ORDER BY order_id) AS next_id
  FROM orders
) t
WHERE next_id - order_id > 1;
```

**P17. All direct and indirect reports of employee 1.**
Approach: a recursive CTE. The anchor is employee 1. The recursive step finds people whose manager is already in the result.

```sql
WITH RECURSIVE reports AS (
  SELECT emp_id, name, manager_id, 1 AS depth
  FROM employees
  WHERE manager_id = 1
  UNION ALL
  SELECT e.emp_id, e.name, e.manager_id, r.depth + 1
  FROM employees e
  JOIN reports r ON e.manager_id = r.emp_id
)
SELECT * FROM reports ORDER BY depth, emp_id;
```

**P18. Median salary per department.**
Approach: use the ordered-set aggregate `percentile_cont(0.5)`. It interpolates for an even count.

```sql
SELECT dept_id,
       percentile_cont(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY dept_id;
```

**P19. Customers who bought from every product category (relational division).**
Approach: count the distinct categories each customer bought and compare with the total number of categories.

```sql
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p     ON p.product_id = oi.product_id
GROUP BY o.customer_id
HAVING COUNT(DISTINCT p.category) = (SELECT COUNT(DISTINCT category) FROM products);
```

**P20. Orders whose stored total differs from the sum of their items.**
Approach: aggregate items per order in a subquery, join, and compare. Use IS DISTINCT FROM to treat NULL safely. A LEFT JOIN catches orders with no items.

```sql
SELECT o.order_id, o.total, COALESCE(i.items_total, 0) AS items_total
FROM orders o
LEFT JOIN (
  SELECT order_id, SUM(qty * unit_price) AS items_total
  FROM order_items
  GROUP BY order_id
) i ON i.order_id = o.order_id
WHERE o.total IS DISTINCT FROM COALESCE(i.items_total, 0);
```

**P21. Latest order of each customer, using ROW_NUMBER.**
Approach: number orders per customer from newest to oldest, then keep number 1. Add order_id as a tie-breaker so the result is stable.

```sql
SELECT customer_id, order_id, order_date, total
FROM (
  SELECT o.*,
         ROW_NUMBER() OVER (PARTITION BY customer_id
                            ORDER BY order_date DESC, order_id DESC) AS rn
  FROM orders o
) t
WHERE rn = 1;
```

**P22. 7-day moving average of daily revenue.**
Approach: first make one row per day. Then average over the current row and the 6 rows before it. Note: this assumes no missing days. For missing days, generate a calendar with generate_series and LEFT JOIN.

```sql
WITH daily AS (
  SELECT order_date, SUM(total) AS revenue
  FROM orders
  GROUP BY order_date
)
SELECT order_date, revenue,
       AVG(revenue) OVER (ORDER BY order_date
                          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS moving_avg_7
FROM daily
ORDER BY order_date;
```

**P23. Each category's share of total revenue (percent).**
Approach: sum per category, then divide by the sum of all categories using a window SUM over the whole result.

```sql
WITH cat AS (
  SELECT p.category, SUM(oi.qty * oi.unit_price) AS revenue
  FROM order_items oi
  JOIN products p ON p.product_id = oi.product_id
  GROUP BY p.category
)
SELECT category, revenue,
       ROUND(100.0 * revenue / SUM(revenue) OVER (), 2) AS pct_of_total
FROM cat
ORDER BY revenue DESC;
```

**P24. Users who logged in during both January and February 2025.**
Approach: write one query per month and use INTERSECT, which returns rows found in both results.

```sql
SELECT user_id FROM logins
WHERE login_date >= DATE '2025-01-01' AND login_date < DATE '2025-02-01'
INTERSECT
SELECT user_id FROM logins
WHERE login_date >= DATE '2025-02-01' AND login_date < DATE '2025-03-01';
```

**P25. Safe "create or add" counter (upsert).**
Approach: use a table with a unique key. ON CONFLICT makes the insert-or-update atomic, so two sessions cannot create duplicate rows.

```sql
CREATE TABLE IF NOT EXISTS daily_hits (
  day  date PRIMARY KEY,
  hits bigint NOT NULL DEFAULT 0
);

INSERT INTO daily_hits (day, hits)
VALUES (CURRENT_DATE, 1)
ON CONFLICT (day) DO UPDATE
SET hits = daily_hits.hits + 1;
```

Tips for SQL rounds:

- Say the grain (what one row means) of the result before you write code.
- State how you handle ties, NULLs and empty groups.
- After writing, test on a tiny table in your head, with one tie and one NULL.

---

# 4. Common mistakes in database interviews

**Answering with a tool name, not a reason.**
Say why: "I add a composite index because the filter has two equality columns and a sort".
A reason shows that you understand the system.

**Saying "add an index" for every slow query.**
First read the plan. The cause may be a stale statistic, a function on a column, an N+1 pattern, or a lock wait.
Also mention the write cost of a new index.

**Forgetting NULL behaviour.**
`= NULL`, `NOT IN` with NULLs, `COUNT(col)` versus `COUNT(*)`, and NULLs in unique constraints are common traps.

**Mixing up the CAP theorem.**
Do not say "pick any two of three". Say what happens during a partition, and mention latency (PACELC) for normal time.

**Mixing up the meanings of "consistency".**
ACID consistency and CAP consistency are different. Name which one you mean.

**Ignoring the isolation level.**
"The transaction is safe" is not enough. Say which anomalies the level allows, and how you retry on serialization errors.

**Proposing sharding too early.**
Interviewers expect the order: fix queries, add indexes, cache, add read replicas, partition, then shard.
Say that sharding adds big operational cost.

**Unsafe migration answers.**
Do not say "just run ALTER TABLE". Mention locks, lock_timeout, batched backfills, CONCURRENTLY, and a rollback plan.

**No mention of backups and restore tests.**
Always give RPO and RTO, and say that you test restores.

**Claiming one database is always best.**
Answer with trade-offs and the access pattern. "It depends, and here is what it depends on" is a good start, if you then decide.

**Not asking clarifying questions in design rounds.**
Ask about data size, read and write ratio, latency target, consistency need and growth. Then design.

**Writing SQL without checking the grain.**
A join on a one-to-many relation multiplies rows, and sums become wrong. Check the row count after each join.

**Using SELECT * and OFFSET paging in examples.**
They signal weak habits. Name your columns and use keyset pagination for deep pages.

**Guessing numbers.**
If you do not know a default or limit, say "I would check the documentation for my version". Version details change.

**Silent thinking.**
Say your plan aloud. State assumptions. If you make a mistake, notice it and fix it calmly.

---

End of question bank. Return to the day files named in each "Revise" line, and practise the lab exercises there.
