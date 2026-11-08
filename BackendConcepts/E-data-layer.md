# Section E — Data Layer

> Reference depth for now — will be enriched with the full visual treatment as we teach each topic.

## E1 — Relational modeling / normalization

**Diagram**
```
 BAD (one table)                    GOOD (3NF)
 orders: id, customer_name,         customers: id, name, city
         customer_city, item        orders:    id, customer_id -> customers
 change a city = update 500 rows    change a city = update 1 row
 Denormalize only when a read is measurably slow.
```
**In plain words.** Write each fact once, in one place. Copies drift apart and cause bugs.

- **Normalization (up to 3NF):** every non-key column depends on **the key, the whole key, and nothing
  but the key**. Removes update/insert/delete anomalies and duplicated data.
- **Denormalize deliberately** for read performance (precompute a total, duplicate a label) — accept
  the write-time cost of keeping copies in sync. Normalize first, denormalize with a reason.

> One line: model in 3NF to kill anomalies, denormalize only for measured read wins.

## E2 — Indexes

**Diagram**
```
 no index : scan every row  [1][2][3]...[1,000,000]
 B-tree   : root -> branch -> leaf -> row   (a few hops)
 composite (a,b):  WHERE a=1          uses index
                   WHERE a=1 AND b=2  uses index
                   WHERE b=2          NOT used (leftmost prefix)
 cost: every index slows writes + uses space
```
**In plain words.** An index is the book's index page. You jump to the right page instead of
reading every page. But each new index must be updated on every write.

- **B-tree** (default, ordered — range/equality/sort/prefix). **Hash** (equality only). **GIN**
  (full-text, arrays, jsonb, trigram). **Partial** (index a subset, e.g. `WHERE active`). **Covering**
  (index includes all selected columns → index-only scan). **Composite** — column **order matters**:
  `(a, b)` serves `a` and `a,b`, not `b` alone (leftmost-prefix rule).
- Cost: every index slows writes and uses space. Index for your real query patterns.

> One line: pick the index type for the access pattern; composite order follows leftmost-prefix;
> indexes speed reads and tax writes.

## E3 — Query plans (EXPLAIN)

**Diagram**
```
 EXPLAIN ANALYZE SELECT ...
   Seq Scan on orders   (cost high, rows 1M)   <- bad on big table
   Index Scan using idx_orders_user            <- good
 watch: seq scans, wrong row estimates (run ANALYZE), heavy sorts
```
**In plain words.** EXPLAIN shows the database's route. If it plans to walk every street, add a
shortcut (index).

- `EXPLAIN [ANALYZE]` shows the planner's chosen path. **Seq scan** (read whole table) vs **index
  scan/seek**. Watch for unexpected seq scans on big tables, bad row estimates (stale stats → `ANALYZE`),
  and expensive sorts/joins.

> One line: read EXPLAIN to confirm the query uses an index seek, not a full scan.

## E4 — Migrations

**Diagram**
```
 0001_create_users.sql -> 0002_add_phone.sql -> 0003_add_index.sql
 dev : drizzle push     (sync directly, fast, risky)
 prod: drizzle migrate  (apply committed files in order)
 zero-downtime: EXPAND (add column) -> MIGRATE data -> CONTRACT (drop old)
```
**In plain words.** Migrations are a Git history for your database structure. Every environment
replays the same steps in the same order.

- Schema changes as **versioned, ordered files**. **Forward-only** vs **reversible** (down migrations).
- **Drizzle:** `push` (sync schema, dev) vs `migrate` (apply committed migration files, prod).
- Make them **backward-compatible** for zero-downtime deploys (expand → migrate data → contract).

> One line: version schema changes as ordered migrations; use migrate (not push) in prod and keep
> them backward-compatible.

## E5 — Transactions & isolation

**Diagram**
```
 BEGIN
   debit A -100
   credit B +100
 COMMIT  (both)  or  ROLLBACK (neither)

 isolation: Read Committed < Repeatable Read < Serializable
            more safety  <-->  more waiting
 deadlock: T1 waits T2, T2 waits T1 -> DB kills one
```
**In plain words.** A transaction is "all or nothing". Money must never leave A without arriving at
B.

- **ACID:** Atomicity, Consistency, Isolation, Durability.
- **Isolation levels:** Read Committed (default) → Repeatable Read → Serializable — each prevents more
  anomalies (dirty/non-repeatable/**phantom** reads) at higher contention cost.
- **Deadlocks:** two txns wait on each other → the DB kills one; keep transactions short and lock in a
  consistent order.

> One line: wrap multi-step writes in a transaction; raise the isolation level only as far as the
> anomalies you must prevent; keep txns short to avoid deadlocks.

## E6 — Connection pooling

**Diagram**
```
 requests -> [ POOL: 10 open connections ] -> Postgres
 request 11 waits for a free one (acquire timeout)
 leak: connection taken, never released -> pool runs dry
 many instances x 10 = too many -> use PgBouncer / Neon pooler
```
**In plain words.** Opening a DB connection is slow, like starting a car. A pool keeps a fleet of
running cars and lends them out.

- Opening a DB connection is expensive; a **pool** reuses a fixed set. Tune **max**, **idle timeout**,
  **acquire timeout**; watch for **leaks** (connections never released).
- Serverless/many instances → use a proxy like **PgBouncer** (or Neon's pooler) so you don't exhaust
  Postgres connection limits.

> One line: reuse connections via a bounded pool; use PgBouncer/Neon pooler when many instances would
> otherwise exhaust Postgres.

## E7 — ORM vs raw SQL

**Diagram**
```
 ORM (Drizzle): db.select().from(users).where(eq(users.id, id))
    + safe, typed, quick           - cannot express every query
 Raw SQL: db.execute(sql`...${id}`)
    + full power                   - you own the safety (bind values)
```
**In plain words.** ORM is automatic gear, raw SQL is manual. Use automatic by default, manual for
tricky hills.

- ORM: safety (parameterized), productivity, type-safety (Drizzle). Raw: full control for complex
  queries the ORM can't express — but you own safety (D3) at the escape hatch.

> One line: default to the ORM (safe, typed); drop to raw SQL for complex queries and parameterize
> there yourself.

## E8 — N+1 query problem

**Diagram**
```
 N+1 (bad)                         fixed
 1 query : list 100 orders         1 query : orders JOIN customers
 100 queries: customer per order   or: customers WHERE id IN (...)
 = 101 round trips                 = 1-2 round trips
```
**In plain words.** Do not walk to the shop 100 times for 100 items. Write one shopping list.

- One query for a list + one query **per row** for its relation = N+1 round trips. **Fix:** a **join**
  or an **IN**-batch (dataloader) to fetch relations in one/few queries. Detect via query logs/APM.

> One line: don't query per-row in a loop — batch relations with a join or an IN query.

## E9 — Read replicas & read/write split

**Diagram**
```
 writes -> PRIMARY ==replicates==> REPLICA <- reads
 lag: write on primary, replica not updated yet
 user saves profile, reloads, reads old one!
 fix: read-your-writes from PRIMARY right after a write
```
**In plain words.** The replica is a photocopy that updates a moment late. Right after you write,
read from the original.

- Route **writes to the primary**, **reads to replicas** to scale reads. Beware **replication lag**
  (a read replica may not yet have a just-written row) → read-your-writes from the primary when needed.

> One line: split reads to replicas to scale, but account for replication lag on read-after-write.

## E10 — Soft vs hard delete

**Diagram**
```
 hard delete: DELETE FROM users WHERE id=5      row gone forever
 soft delete: UPDATE users SET deleted_at=now() row stays
              every query: WHERE deleted_at IS NULL  (enforce in repo layer)
 + history/undo/audit   - messier queries, unique constraints
```
**In plain words.** Hard delete is the shredder. Soft delete is moving the file to the recycle
bin, and you must remember to hide the bin everywhere.

- **Hard delete** removes the row. **Soft delete** sets `deleted_at` and every query filters it out.
  Soft keeps history/undo/audit but complicates queries and uniqueness; enforce the filter in a low
  layer (like C10's org filter).

> One line: soft-delete (deleted_at + filter everywhere) preserves history; hard-delete is simpler but
> irreversible.

## E11 — Audit triggers / CDC

**Diagram**
```
 app writes row --> DB --> trigger / CDC (reads WAL)
                                  |
                  audit table  /  search index  /  event stream
 avoids dual-write (app writing to DB and elsewhere, one can fail)
```
**In plain words.** A security camera on the database itself. Whatever changes, it records, no
matter which app made the change.

- **DB triggers** or **Change Data Capture** (read the write-ahead log) capture every change for audit
  or to stream to other systems (search index, warehouse, events). CDC avoids app-level dual-writes.

> One line: capture changes at the DB (triggers/CDC) for audit trails and to feed downstream systems
> without dual-writes.

## E12 — Mongo schema design (embed vs reference)

**Diagram**
```
 EMBED (read together, bounded)    REFERENCE (large/shared/unbounded)
 order: { items:[{...},{...}] }    user: {_id}   orders: {userId, ...}
 limit: 16 MB per document, avoid ever-growing arrays
```
**In plain words.** Embed is stapling the receipt to the order. Reference is writing "see file
#42" because the file is big or used by many.

- **Embed** when data is read together and bounded (order + its line items). **Reference** when data is
  large, shared, or unbounded (user ↔ many orders). Mind the **16 MB document limit** and unbounded
  array growth.

> One line: embed bounded read-together data, reference large/shared/unbounded data; watch the 16 MB
> doc limit.

## E13 — Polyglot persistence

**Diagram**
```
 Postgres : money, orders, KYC     (relations, transactions)
 Mongo    : catalog, flexible docs
 Redis    : cache, queues, OTP     (fast, temporary)
 cost: more moving parts, consistency across stores
```
**In plain words.** Right tool for each job: a ledger book, a flexible folder, and a whiteboard.

- Use the right store per need: **Postgres** for relational/transactional integrity, **Mongo** for
  flexible/denormalized documents, **Redis** for cache/queues/ephemeral. Cost: more moving parts and
  consistency across stores.

> One line: match the datastore to the data shape (Postgres relational, Mongo documents, Redis
> ephemeral) — at the cost of cross-store consistency.

## E14 — Cache layers (Redis)

**Diagram**
```
 read-through : app -> cache? miss -> DB -> fill cache
 write-through: write cache + DB together
 write-behind : write cache now, DB later (fast, risky)
```
**In plain words.** Keep the things you ask for often on your desk instead of in the archive room.

- **Read-through** (app fills cache on miss), **write-through** (write cache + DB together),
  **write-behind** (write cache now, DB async). Cache the hot, expensive, read-heavy data.

> One line: cache hot read-heavy data; choose read-through / write-through / write-behind by your
> consistency needs.

## E15 — Cache invalidation

**Diagram**
```
 TTL: key expires after N sec        event: invalidate on write
 STAMPEDE: key expires -> 1000 requests all miss -> 1000 DB hits
 fix: lock (one rebuilds), jittered TTL, stale-while-revalidate
```
**In plain words.** When the desk copy goes stale, make only one person fetch the new one, while
everyone else waits or uses the old copy.

- The hard part. **TTL** (expire after N) vs **event-driven** (invalidate on write). Watch **stampede /
  thundering herd / dogpile** (many misses hit the DB at once when a key expires) → use locks,
  jittered TTLs, or stale-while-revalidate.

> One line: invalidate by TTL or on write; guard against stampede with locks / jittered TTL /
> stale-while-revalidate.

## E16 — Materialized views / precomputed aggregates

**Diagram**
```
 slow: SELECT SUM(...) over 10M rows, per request
 fast: materialized view (stored result) <- REFRESH on schedule/change
 trade: freshness  <-->  read speed
```
**In plain words.** Do the maths once and keep the answer, instead of recounting every time.

- Store the result of an expensive aggregate and **refresh** it (on schedule or on change) instead of
  recomputing per request. Trades freshness for read speed.

> One line: precompute expensive aggregates into a materialized view and refresh it — freshness traded
> for read speed.
