# SQL for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to SQL for people who build services on relational databases. It teaches how the database thinks, so you can write correct queries, read a query plan, and explain why a query is wrong or slow, not just recall syntax.

**Audience:** backend developers who have used SQL but want interview-grade fluency and production judgement.
**Prerequisites:** basic programming experience. You have seen a `SELECT` before.
**Scope:** standard SQL with PostgreSQL as the reference dialect. Differences in MySQL and SQL Server are flagged only where they are well established. Analytics warehouses and vendor-specific procedural languages are out of scope.
**Note on accuracy:** syntax shown is standard or PostgreSQL syntax. Version-dependent features are marked. The example queries were reviewed by reading, not executed, so run them on test data. Confirm details against the manual for your database version.

## How to use this guide

Each concept chapter has four parts:

1. **Understand**: the mental model and the reasoning behind it.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 19 contains 26 worked query problems with solutions. Chapter 20 lists labs. Chapter 21 collects interview questions with model answers. Chapter 22 lists primary references. Chapter 23 is a glossary of hard terms. Each chapter starts with a short "In plain words" summary.

**What "good enough" means for a backend role:** you can model data correctly, write queries whose results you can prove are right, explain why a query is slow from its plan, reason about concurrent transactions, and ship schema changes without downtime. Chapters 4 to 9, 12 to 15 and 18 matter most.

**Do not run queries only in your head.** Every pattern here should be typed into a real PostgreSQL instance with a few rows of test data, including awkward rows (NULLs, duplicates, ties).

## Contents

1. The relational model and keys
2. Data types
3. DDL and constraints
4. How a query is evaluated
5. NULL and three-valued logic
6. Joins
7. Aggregation
8. Subqueries, CTEs, LATERAL and recursion
9. Window functions
10. Set operations, expressions, strings, dates and JSON
11. Classic interview patterns
12. Indexes
13. Reading an EXPLAIN plan
14. Transactions and isolation
15. Locking
16. Normalisation and denormalisation
17. Safe schema changes
18. Application-side mistakes
19. Worked query problems
20. Practice labs
21. Interview questions
22. References
23. Glossary

---

## The reference schema

Every example uses this schema (PostgreSQL). It is defined once here and reused everywhere. Amounts are integer minor units (cents).

```sql
CREATE TABLE customers (
  id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email      text        NOT NULL UNIQUE,
  name       text        NOT NULL,
  country    char(2)     NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint      NOT NULL REFERENCES customers (id),
  status      text        NOT NULL
              CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
  total_cents bigint      NOT NULL CHECK (total_cents >= 0),
  currency    char(3)     NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
  order_id         bigint NOT NULL REFERENCES orders (id) ON DELETE CASCADE,
  line_no          int    NOT NULL,
  product_sku      text   NOT NULL,
  quantity         int    NOT NULL CHECK (quantity > 0),
  unit_price_cents bigint NOT NULL CHECK (unit_price_cents >= 0),
  PRIMARY KEY (order_id, line_no)
);

CREATE TABLE payments (
  id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id     bigint      NOT NULL REFERENCES orders (id),
  amount_cents bigint      NOT NULL,
  status       text        NOT NULL,   -- pending, processing, captured, failed
  provider_ref text UNIQUE,            -- external id, used for idempotency
  created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE events (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint      REFERENCES customers (id),
  event_type  text        NOT NULL,    -- e.g. 'login', 'page_view'
  payload     jsonb       NOT NULL DEFAULT '{}',
  occurred_at timestamptz NOT NULL
);
```

Small extra tables are introduced inline where a chapter needs them (for example `categories` in chapter 8).

---

## 1. The relational model and keys

**In plain words:** A database table stores facts as rows. A key is a column (or columns) that identifies a row. You tell the database what you want, and it decides how to get it.

### Understand

**Relations are sets of facts.** A table holds rows, each asserting one fact ("customer 7 placed order 42"). Rows have no inherent order, and a query result has no order unless you say `ORDER BY`. Code that depends on "the first row" without an `ORDER BY` has a latent bug.

**Declarative, not procedural.** You describe the result you want. The **optimizer** (the part that plans execution) chooses the algorithm (scan, index, join order). This is why the same SQL can be fast or slow depending on statistics and indexes, and why you reason about sets, not loops.

**Keys.**
- **Primary key**: uniquely identifies a row, never NULL. One per table.
- **Candidate key**: any column set that could be the primary key (email is a candidate key for customers).
- **Natural key**: a real-world identifier (email, country code). Can change or be reused.
- **Surrogate key**: a generated identifier with no meaning (`bigint` identity, UUID). Stable, and the usual primary key choice, with natural candidates enforced by `UNIQUE`.
- **Foreign key**: a column that must match a primary or unique key elsewhere. It enforces referential integrity: no order without a customer.
- **Composite key**: several columns together (`order_items (order_id, line_no)`).

**Relationship shapes.** One-to-many is a foreign key on the "many" side. Many-to-many needs a junction table holding two foreign keys. One-to-one is a foreign key that is also `UNIQUE`.

**Surrogate key choices.** Identity/sequence integers are compact and index-friendly but leak volume and have gaps after rollbacks (gaps are normal, never rely on contiguity). Random UUIDs avoid coordination and hide counts but make B-tree inserts scatter. Time-ordered UUIDs (UUIDv7) keep locality; recent PostgreSQL versions add a built-in generator, but check your version.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspecting structure | `psql`: `\d table`, `\dt`, `\di`; `information_schema.columns` | Read any schema quickly |
| Drawing relationships | pgAdmin ERD, DBeaver ER diagram | Sketch a schema from existing tables |
| Key design | `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, identity columns | Choose keys and justify the choice |

### Common mistakes

- Using a mutable natural key (email) as the primary key, then needing cascading updates.
- No foreign keys "for speed". Orphaned rows then accumulate and every query must defend against them.
- Assuming sequence values are gapless or reflect commit order.

### Self-check

- Why does a table without `ORDER BY` give no ordering guarantee, even if it "always looks sorted"?
- When would you prefer a surrogate key over a natural key?
- How do you model many-to-many between orders and promotions?

---

## 2. Data types

**In plain words:** Every column has a type, such as number, text or time. The right type stops wrong data and gives exact results. Money and time need special care.

### Understand

Types are constraints you get for free. Choose the narrowest correct type.

**Money.** Never use `real`, `double precision` or `float`. Binary floating point cannot represent 0.1 exactly (`0.1 + 0.2` is not `0.3`), and errors compound across sums. Two correct choices:
- **Integer minor units** (`bigint` cents). Exact, fast, portable. Store the currency code alongside. Conversion to display happens at the edge.
- **`NUMERIC(p, s)`** (for example `NUMERIC(19, 4)`). Exact decimal arithmetic, slower, and the right choice for rates and fractional-cent prices. MySQL calls it `DECIMAL`.

Avoid the PostgreSQL `money` type (locale-dependent formatting and rounding). Never sum amounts across different currencies. Decide a rounding rule (half-up or banker's) once and apply it in one place.

**Timestamps.** Use `timestamptz` (`timestamp with time zone`) for instants. In PostgreSQL it stores a UTC instant; the session `TimeZone` setting only affects input interpretation and display. `timestamp` (without time zone) stores a wall-clock reading with no zone and invites bugs. Rules:
- Store instants in UTC, convert to a local zone at the edge.
- For business-local concepts ("the delivery date in the customer's city"), store a `date` or local time plus an IANA zone name such as `Europe/Berlin`, not a fixed offset, because offsets change with daylight saving.
- `ts AT TIME ZONE 'UTC'` converts a `timestamptz` to a `timestamp` in that zone (and the reverse for a plain `timestamp`). It is easy to apply in the wrong direction, so test with known values.
- MySQL differs: `TIMESTAMP` is stored as UTC and converted by session zone, `DATETIME` is a zone-less wall-clock value.

**Text.** In PostgreSQL `text` and `varchar(n)` perform the same; use `varchar(n)` only when the limit is a real rule. `char(n)` pads with spaces, so reserve it for truly fixed codes (country, currency). Collation controls comparison and sort order and affects index usability.

**Numbers.** `integer` (4 bytes), `bigint` (8 bytes), `smallint`. An `integer` id overflows near 2.1 billion, which has caused real outages, so default to `bigint` for ids on growing tables. Integer division truncates in PostgreSQL (`1/2` is `0`); cast one operand for fractional results.

**Booleans, enums, JSON.** `boolean` for flags. For a closed set of values, either a `CHECK` constraint (easy to change) or a lookup table with a foreign key. Native enum types are hard to change. `jsonb` (binary, indexable) for semi-structured data, covered in chapter 10.

**UUID.** `uuid` is a 16-byte type, not a 36-character string. `gen_random_uuid()` is built in from PostgreSQL 13.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspect a value's type | `SELECT pg_typeof(x)`, `\d` | Know the type of every column you query |
| Casting | `CAST(x AS type)`, `x::type` (PostgreSQL shorthand) | Cast explicitly and predictably |
| Time zones | `SHOW timezone`, `SET TIME ZONE`, `AT TIME ZONE`, `now()` | Predict the output of a conversion |
| Exact arithmetic | `NUMERIC`, `round(x, n)` | Compute and round money correctly |

### Common mistakes

- Storing money in `float`/`double`, or in a `varchar` with a currency symbol.
- Using `timestamp` without zone and mixing server-local times.
- Storing dates as strings, which sorts and compares wrongly in some formats.
- `integer` primary keys on high-volume tables.

### Self-check

- Why is `bigint` cents safer than `double precision` dollars?
- What does a `timestamptz` column actually store, and what does the session time zone change?
- How would you store "9 a.m. local time every day in Tokyo" for a recurring notification?

---

## 3. DDL and constraints

**In plain words:** DDL is the SQL that creates and changes tables. A constraint is a rule that the database checks on every write. Rules in the database cannot be skipped by any program.

### Understand

**DDL** (Data Definition Language: `CREATE`, `ALTER`, `DROP`) defines structure, **DML** (Data Manipulation Language: `INSERT`, `UPDATE`, `DELETE`, `MERGE`) changes data, **DQL** (Data Query Language: `SELECT`) reads it. In PostgreSQL, DDL is transactional: a migration can be rolled back as a unit (MySQL commits implicitly on most DDL).

**Constraints put invariants in the database.** Application checks race with each other and with other writers (scripts, admin tools, other services). A constraint is the only check that cannot be bypassed.

| Constraint | Enforces | Note |
|---|---|---|
| `NOT NULL` | value present | default to `NOT NULL`; allow NULL only when "unknown" is meaningful |
| `PRIMARY KEY` | unique and not null | creates an index |
| `UNIQUE` | no duplicates | creates an index; PostgreSQL permits multiple NULLs unless `NULLS NOT DISTINCT` (version 15+) |
| `FOREIGN KEY` | referenced row exists | PostgreSQL does not index the referencing column automatically |
| `CHECK` | row-level predicate | passes when the predicate is NULL (unknown), so pair with `NOT NULL` |
| `DEFAULT` | value if omitted | `now()`, constants |

**Referential actions.** `ON DELETE RESTRICT`/`NO ACTION` (default), `CASCADE`, `SET NULL`. Use `CASCADE` for true ownership (items belong to an order), not for convenience.

**Other useful DDL.**
- `UNIQUE` partial index for conditional uniqueness: `CREATE UNIQUE INDEX ON payments (order_id) WHERE status = 'captured';` allows at most one captured payment per order.
- `GENERATED ... AS IDENTITY` is the standard replacement for `serial`.
- Exclusion constraints (PostgreSQL) prevent overlapping ranges, for example double-booking.

**Idempotent (safe to run twice) migration style.** `CREATE TABLE IF NOT EXISTS`, `ADD COLUMN IF NOT EXISTS` help scripts, but versioned migration tools are the real mechanism (chapter 17).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schema changes | `CREATE/ALTER/DROP`, `psql -f`, migration tools (Flyway, Liquibase, Alembic, Prisma Migrate, etc.) | Write and review a migration |
| Constraint design | `CHECK`, `UNIQUE`, partial unique indexes | Encode business invariants in the schema |
| Introspection | `\d+ table`, `pg_constraint`, `information_schema.table_constraints` | List all constraints on a table |

### Common mistakes

- Everything nullable, so every query needs defensive `COALESCE`.
- Business rules only in application code.
- No index on a foreign key column, so deletes on the parent scan the child table and joins are slow.
- `CASCADE` on relationships where deleting the parent should be blocked.
- Using `CHECK (x > 0)` and assuming it rejects NULL (it does not).

### Self-check

- Why can a `CHECK (total_cents >= 0)` column still hold NULL?
- How do you enforce "at most one active subscription per customer"?
- What happens to child rows under `ON DELETE CASCADE` versus `RESTRICT`?

---

## 4. How a query is evaluated

**In plain words:** You write `SELECT` first, but the database works in a different order. It reads the tables, filters rows, groups them, and only then builds the output columns. Knowing this order explains many errors.

### Understand

**You write SELECT first, but the database evaluates it last.** The logical order is:

1. `FROM` (and `JOIN`): build the row source.
2. `WHERE`: filter rows (before grouping).
3. `GROUP BY`: form groups.
4. `HAVING`: filter groups.
5. `SELECT`: compute output expressions, including window functions.
6. `DISTINCT`.
7. `ORDER BY`.
8. `LIMIT` / `OFFSET` (`FETCH FIRST`).

This is a **logical** order. The optimizer may physically execute differently as long as the result is identical. The order explains most surprises:

- You cannot use a `SELECT` alias in `WHERE` (it does not exist yet). You can use it in `ORDER BY` in PostgreSQL, and `GROUP BY` allows output-column numbers (`GROUP BY 1`).
- `WHERE` cannot contain aggregates, because groups do not exist yet. Use `HAVING`.
- Window functions run after `WHERE`/`GROUP BY`/`HAVING`, so you cannot filter on a window result in the same query level. Wrap it in a subquery or CTE.
- `LIMIT` applies after `ORDER BY`. Without `ORDER BY` it returns arbitrary rows.

**Basic forms.**

```sql
SELECT country, count(*) AS customers
FROM customers
WHERE created_at >= timestamptz '2026-01-01 00:00:00+00'
GROUP BY country
HAVING count(*) >= 10
ORDER BY customers DESC, country
LIMIT 20;
```

**GROUP BY rule.** Every selected column must be grouped or aggregated. PostgreSQL also accepts columns functionally dependent on a grouped primary key. MySQL's `ONLY_FULL_GROUP_BY` mode (default since 5.7) enforces the standard rule, and older or relaxed settings return arbitrary values, which is a correctness trap.

**Row limiting dialects.** Standard: `FETCH FIRST n ROWS ONLY`. PostgreSQL and MySQL: `LIMIT n`. SQL Server: `TOP n` or `OFFSET ... FETCH` (which requires `ORDER BY`).

**Range predicates on time.** Use half-open ranges: `created_at >= '2026-01-01' AND created_at < '2026-02-01'`. `BETWEEN` is inclusive on both ends and double-counts boundaries.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Interactive work | `psql` (`\timing`, `\x`, `\e`, `\copy`), pgAdmin, DBeaver | Iterate on queries and inspect results fast |
| Evaluation order | reading, then verifying with small data | Predict where an alias is visible |

### Common mistakes

- Using `HAVING` for conditions that belong in `WHERE` (works but filters after grouping, which is slower).
- `BETWEEN` on timestamps.
- Selecting non-grouped columns and trusting the arbitrary value.
- `LIMIT` without `ORDER BY`, then pagination returns duplicates or skips rows.
- `SELECT *` in application code: fragile to schema changes and defeats covering indexes.

### Self-check

- Why does `SELECT price * qty AS total FROM t WHERE total > 100` fail?
- What is the difference between `WHERE` and `HAVING`?
- Why is `WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31'` wrong for timestamps?

---

## 5. NULL and three-valued logic

**In plain words:** NULL means the value is missing or unknown. A test with NULL is neither true nor false, so rows can disappear without warning. Learn the few NULL rules and the traps become easy to avoid.

### Understand

**NULL means "unknown" or "not applicable", not a value.** Any comparison with NULL yields **UNKNOWN**, the third truth value.

- `NULL = NULL` is UNKNOWN, not true. Use `IS NULL` / `IS NOT NULL`.
- `WHERE` keeps only rows where the predicate is TRUE. UNKNOWN rows are dropped, same as FALSE.
- Truth tables: `TRUE AND UNKNOWN` is UNKNOWN; `FALSE AND UNKNOWN` is FALSE; `TRUE OR UNKNOWN` is TRUE; `NOT UNKNOWN` is UNKNOWN.
- Arithmetic and concatenation with NULL give NULL (`'a' || NULL` is NULL in PostgreSQL).
- `IS DISTINCT FROM` / `IS NOT DISTINCT FROM` compare treating NULL as an ordinary value (null-safe equality). MySQL's null-safe operator is `<=>`.

**The famous traps.**

1. **`NOT IN` with a NULL in the list returns no rows.**
   ```sql
   -- if any orders.customer_id is NULL, this returns nothing
   SELECT id FROM customers WHERE id NOT IN (SELECT customer_id FROM orders);
   ```
   `id NOT IN (1, 2, NULL)` expands to `id <> 1 AND id <> 2 AND id <> NULL`, and the last term is UNKNOWN. Use `NOT EXISTS` (chapter 6) or filter the NULLs out.

2. **`COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`.** `COUNT(*)` counts rows. `COUNT(col)` counts non-NULL values. `COUNT(DISTINCT col)` counts distinct non-NULL values.

3. **Aggregates ignore NULLs.** `AVG(x)` divides by the count of non-NULL values. `SUM` over zero rows (or all NULLs) is NULL, not 0, so wrap with `COALESCE(SUM(x), 0)`.

4. **`LEFT JOIN` filter in `WHERE`.** A condition on the right table in `WHERE` rejects the NULL-extended rows, silently turning the outer join into an inner join. Put such conditions in `ON`.

5. **`CHECK` passes on NULL.** A constraint fails only when the predicate is FALSE.

6. **Sorting.** PostgreSQL sorts NULL as larger than any value (last in `ASC`, first in `DESC`); control with `NULLS FIRST`/`NULLS LAST`. MySQL treats NULL as smaller. Do not rely on the default across databases.

7. **`UNIQUE` and NULL.** Standard behaviour in PostgreSQL and MySQL allows multiple NULLs in a unique column. SQL Server treats NULLs as equal in a unique index (one NULL allowed).

**Tools for handling NULL.** `COALESCE(a, b, ...)` returns the first non-NULL. `NULLIF(a, b)` returns NULL when equal, which guards division: `x / NULLIF(y, 0)`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Testing for NULL | `IS [NOT] NULL`, `IS [NOT] DISTINCT FROM` | Never write `= NULL` |
| Defaults | `COALESCE`, `NULLIF` | Guard sums, divisions and optional fields |
| Anti-joins | `NOT EXISTS`, `LEFT JOIN ... IS NULL` | Avoid `NOT IN` against nullable columns |
| Data profiling | `count(*) FILTER (WHERE col IS NULL)` | Measure null rates of a column |

### Common mistakes

- `WHERE col = NULL` or `col <> NULL`.
- `NOT IN (subquery)` on a nullable column.
- Reading `COUNT(col)` as a row count.
- `col <> 'x'` assuming it also returns rows where `col` is NULL. It does not.
- Treating NULL as zero in totals without `COALESCE`.

### Self-check

- What does `SELECT 1 WHERE NULL = NULL` return, and why?
- Why does `NOT IN` misbehave, and what are two safe rewrites?
- A table has 10 rows, 3 with NULL `amount`. What do `COUNT(*)`, `COUNT(amount)` and `AVG(amount)` use as their counts?

---

## 6. Joins

**In plain words:** A join puts rows from two tables side by side when they match. A wrong join can repeat rows and make totals too large. Check row counts before and after every join.

### Understand

**A join pairs rows from two sources by a predicate.** Think of it as a filtered Cartesian product.

| Join | Returns |
|---|---|
| `INNER JOIN` | rows with a match on both sides |
| `LEFT [OUTER] JOIN` | all left rows, right columns NULL when no match |
| `RIGHT [OUTER] JOIN` | mirror of left; usually rewritten as a left join |
| `FULL [OUTER] JOIN` | all rows from both, NULL-extended where unmatched (MySQL does not support it) |
| `CROSS JOIN` | every left row with every right row (`n * m` rows) |
| `JOIN ... USING (col)` | equality join that merges the column into one |

**Semi-join and anti-join.** These answer "does a match exist?" without returning columns from the other side and without multiplying rows.

```sql
-- semi-join: customers with at least one paid order
SELECT c.id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'paid');

-- anti-join: customers with no orders
SELECT c.id FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

`EXISTS` is NULL-safe and stops at the first match. `NOT EXISTS` is the safest anti-join. The equivalent `LEFT JOIN ... WHERE o.id IS NULL` also works. The planner often converts `IN`/`EXISTS` to a semi-join internally.

**Fan-out (the join pitfall that corrupts totals).** Joining a parent to a "many" child repeats the parent's columns once per child row. Summing a parent column after that join counts it several times.

```sql
-- WRONG: each order's total_cents is repeated once per item
SELECT SUM(o.total_cents)
FROM orders o JOIN order_items oi ON oi.order_id = o.id;
```

Fixes: aggregate the child first and then join, join on a one-row-per-key derived table, or use a semi-join if you only need existence. A second symptom is `COUNT(*)` inflated after a join. When joining two one-to-many branches of the same parent (items and payments of one order), both fan out and multiply each other. Aggregate each branch separately.

**Outer join conditions.** For `LEFT JOIN`, a predicate in `ON` limits which right rows match. A predicate in `WHERE` filters the final result. They are not interchangeable (chapter 5).

**Join algorithms (what the planner picks).** Nested loop (good with an index on the inner side and few outer rows), hash join (build a hash table on the smaller side, good for large unsorted inputs), merge join (both inputs sorted). Seeing which one runs, and why, is chapter 13.

**Self-join** compares rows in the same table (alias the table twice). Often a window function is cleaner.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Join types | `INNER`, `LEFT`, `FULL`, `CROSS`, `USING` | Pick the right one from the question's wording |
| Existence tests | `EXISTS`, `NOT EXISTS` | Write semi- and anti-joins without duplicates |
| Row count checks | `count(*)` before and after a join | Detect fan-out by comparing counts |
| Join diagnosis | `EXPLAIN` join nodes | Identify nested loop, hash, merge |

### Common mistakes

- Summing or counting after a one-to-many join (fan-out).
- Moving a right-table filter from `ON` to `WHERE` and losing unmatched rows.
- `SELECT DISTINCT` to hide a fan-out rather than fixing the join.
- Accidental `CROSS JOIN` from a missing join predicate.
- Joining on non-unique columns and assuming one row per key.

### Self-check

- When does a `LEFT JOIN` behave like an `INNER JOIN`?
- How do you total order amounts and payment amounts per order without double counting?
- Why is `NOT EXISTS` preferred over `NOT IN`?

---

## 7. Aggregation

**In plain words:** An aggregate function turns many rows into one value, such as a count or a sum. `GROUP BY` makes one result per group. Aggregates skip NULL values.

### Understand

**Aggregates collapse many rows into one per group.** Common: `count`, `sum`, `avg`, `min`, `max`, `string_agg(expr, sep)`, `array_agg`, `bool_and`/`bool_or` (PostgreSQL). `DISTINCT` inside an aggregate (`count(DISTINCT customer_id)`) de-duplicates inputs, which is more expensive.

**`FILTER` for conditional aggregation.** Standard SQL, supported by PostgreSQL:

```sql
SELECT customer_id,
       count(*)                                  AS orders,
       count(*) FILTER (WHERE status = 'paid')   AS paid_orders,
       sum(total_cents) FILTER (WHERE status = 'paid') AS paid_cents
FROM orders
GROUP BY customer_id;
```

The portable form works in MySQL and SQL Server too: `sum(CASE WHEN status = 'paid' THEN total_cents ELSE 0 END)`. Use `count(CASE WHEN cond THEN 1 END)` for counting, because `count` skips NULL.

**`HAVING`** filters groups: `HAVING count(*) > 1` finds duplicates.

**Grouping sets.** `GROUP BY ROLLUP (a, b)`, `CUBE`, `GROUPING SETS` produce subtotals in one pass.

**Aggregate vs window.** An aggregate returns one row per group. A window function (chapter 9) keeps all rows and attaches the aggregate to each.

**Precision.** `avg` of integers returns `numeric` in PostgreSQL. Integer division elsewhere truncates, so cast before dividing: `100.0 * a / b`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Aggregation | `GROUP BY`, `HAVING`, `FILTER`, `CASE` inside aggregates | Build a multi-metric report in one query |
| Lists | `string_agg`, `array_agg`, `jsonb_agg` | Return children as one column |
| Subtotals | `ROLLUP`, `GROUPING SETS` | Produce totals plus detail |
| Statistics | `percentile_cont`, `percentile_disc` (`WITHIN GROUP`) | Compute medians and percentiles |

### Common mistakes

- Aggregating after a fan-out join.
- `COUNT(col)` when `COUNT(*)` is meant, and the reverse.
- `SUM` returning NULL for empty groups and breaking application arithmetic.
- Integer division producing zero percentages.

### Self-check

- Compute paid and cancelled order counts per customer in a single pass.
- How does `count(DISTINCT x)` differ from `count(x)`, and what does each ignore?
- Why might `avg(amount)` differ from `sum(amount)/count(*)`?

---

## 8. Subqueries, CTEs, LATERAL and recursion

**In plain words:** A subquery is a query inside another query. A CTE (common table expression) gives a name to a step so the query is easier to read. A recursive CTE repeats itself to walk through a tree.

### Understand

**Subqueries** appear as a scalar (`SELECT (SELECT max(...) ...)`), in `IN`/`EXISTS`, or in `FROM` as a derived table. A **correlated** subquery references the outer row and is logically re-evaluated per row, though the optimizer often flattens it into a join.

**CTEs (`WITH`)** name an intermediate result so a query reads top to bottom:

```sql
WITH paid AS (
  SELECT customer_id, sum(total_cents) AS cents
  FROM orders WHERE status = 'paid'
  GROUP BY customer_id
)
SELECT c.name, p.cents
FROM paid p JOIN customers c ON c.id = p.customer_id;
```

Choosing between them:

| Form | Use when | Note |
|---|---|---|
| Derived table / subquery | a single short step | no name, local |
| CTE | several steps, reuse of one result, readability | PostgreSQL 12 and later inlines a non-recursive, side-effect-free CTE referenced once; `MATERIALIZED` / `NOT MATERIALIZED` force the choice. Earlier versions always materialised (an optimization fence) |
| `LATERAL` | the subquery needs values from earlier `FROM` items | per-row subquery with `LIMIT` (top-N per group) |
| Temp table | a large intermediate reused many times, or indexing it | extra write cost |

**LATERAL** lets a subquery in `FROM` reference preceding tables, like a for-each loop:

```sql
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
CROSS JOIN LATERAL (
  SELECT id, created_at FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC LIMIT 3
) o;
```

Use `LEFT JOIN LATERAL (...) ON true` to keep customers with no orders. MySQL supports `LATERAL` from 8.0.14. SQL Server uses `CROSS APPLY` / `OUTER APPLY`. With an index on `orders (customer_id, created_at)` this is a fast top-N per group.

**Recursive CTEs** walk hierarchies and graphs. A non-recursive anchor member is combined by `UNION ALL` with a recursive member that refers to the CTE itself, repeating until no new rows appear. Standard form is `WITH RECURSIVE` (PostgreSQL, MySQL). SQL Server omits the `RECURSIVE` keyword.

```sql
CREATE TABLE categories (
  id        bigint PRIMARY KEY,
  parent_id bigint REFERENCES categories (id),
  name      text NOT NULL
);

WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth, name AS path
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1, t.path || ' > ' || c.name
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY path;
```

Cyclic data loops forever. Guard with a depth limit (`WHERE t.depth < 20`), tracking visited ids in an array, or PostgreSQL 14's `CYCLE` clause. Recursive CTEs also generate series and traverse graphs such as referral chains.

**Data-modifying CTEs** (PostgreSQL): `WITH moved AS (DELETE ... RETURNING *) INSERT ... SELECT * FROM moved` runs as one statement. Used for archiving and queue claims (chapter 15).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Decomposition | CTEs, derived tables | Break a hard problem into named steps |
| Per-row subqueries | `LATERAL`, correlated subqueries | Top-N per group without window functions |
| Trees and graphs | `WITH RECURSIVE`, `CYCLE` | Traverse a hierarchy safely |
| Inspecting inlining | `EXPLAIN` (`CTE Scan` node appears if materialised) | Tell if a CTE was a fence |

### Common mistakes

- Assuming a CTE is always materialised (or never) without checking the version.
- Deep chains of CTEs that hide an unindexed step.
- Recursive CTE with no termination guarantee.
- A correlated subquery in `SELECT` on a large table, causing per-row work. Prefer a join or `LATERAL`.
- Scalar subquery returning more than one row (runtime error).

### Self-check

- When is a CTE not just a readability aid?
- Write a query returning each customer's three most recent orders using `LATERAL`.
- How do you stop a recursive CTE from looping on cyclic data?

---

## 9. Window functions

**In plain words:** A window function calculates a value from nearby rows but keeps every row. Use it for rankings, running totals and the previous row's value. A grouped query would merge those rows instead.

### Understand

**A window function computes over a set of related rows without collapsing them.** Syntax: `function(...) OVER (PARTITION BY ... ORDER BY ... frame)`. `PARTITION BY` splits rows into independent groups, `ORDER BY` orders within the partition, and the **frame** selects which rows around the current row feed the function.

**Ranking.**

| Function | Behaviour on ties |
|---|---|
| `row_number()` | unique 1, 2, 3, arbitrary among ties unless the order is total |
| `rank()` | ties share a rank, then a gap (1, 1, 3) |
| `dense_rank()` | ties share a rank, no gap (1, 1, 2) |
| `ntile(n)` | splits into n buckets |

**Offset and value functions.** `lag(x, n, default)` and `lead(x, n, default)` read previous and next rows. `first_value`, `last_value`, `nth_value`.

**Aggregates as windows.** `sum(x) OVER (...)`, `avg`, `count`, `min`, `max`. With `ORDER BY` they become running values.

**Frames.** Frame units are `ROWS` (physical row offsets), `RANGE` (value-based, includes peers with equal sort value) and `GROUPS`.

```sql
sum(total_cents) OVER (ORDER BY created_at
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

Critical facts:
- **Default frame.** With `ORDER BY` and no explicit frame, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Peers (rows with the same sort value) are all included, so a running total over a non-unique sort key jumps in steps. State `ROWS` explicitly for row-by-row running totals.
- **`last_value` trap.** Under the default frame it returns the current row's peer group, not the partition's last row. Use `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, or flip the order and use `first_value`.
- Without `ORDER BY`, the frame is the whole partition.
- Make the `ORDER BY` total (add `id` as a tie-breaker) when the result must be deterministic.

**Named windows.** `WINDOW w AS (PARTITION BY customer_id ORDER BY created_at)` and `OVER w` avoid repetition.

**Filtering on window results.** Window functions cannot appear in `WHERE`. Compute them in a subquery or CTE, then filter outside. (PostgreSQL has no `QUALIFY`; some warehouses do.)

**Performance.** Each distinct window specification needs a sort of its partition. An index matching `PARTITION BY ... ORDER BY` can avoid the sort. Windows over huge partitions are memory heavy.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Ranking | `row_number`, `rank`, `dense_rank`, `ntile` | Top-N per group and dedupe |
| Time series | `lag`, `lead`, running `sum`, moving `avg` with `ROWS` | Deltas, cumulative totals, moving averages |
| Frames | `ROWS`/`RANGE`/`GROUPS BETWEEN ...` | State frames explicitly |
| Shares | `sum(x) OVER ()`, `x / sum(x) OVER (PARTITION BY ...)` | Percent of total |

### Common mistakes

- Trusting `row_number()` with a non-unique `ORDER BY`.
- Relying on the default frame for running totals or `last_value`.
- Trying to filter on a window function in the same level's `WHERE`.
- Using `rank()` where a strict "one row per group" is required.
- Computing a moving average over days with missing rows (the frame counts rows, not days).

### Self-check

- Difference between `rank`, `dense_rank` and `row_number` for scores 90, 90, 80?
- Why can `last_value(x) OVER (ORDER BY d)` return the current row?
- How do you keep only the latest row per customer?

---

## 10. Set operations, expressions, strings, dates and JSON

**In plain words:** This chapter covers everyday tools. You can combine two result lists, choose a value by condition, work with text and dates, and read JSON data. Each tool has small traps that are listed here.

### Understand

**Set operations** combine results of queries with matching column counts and compatible types.

| Operator | Result | Note |
|---|---|---|
| `UNION` | rows in either, duplicates removed | requires a sort or hash |
| `UNION ALL` | rows in either, duplicates kept | faster; the default choice when duplicates are impossible or acceptable |
| `INTERSECT` | rows in both | MySQL supports it from 8.0.31 |
| `EXCEPT` | rows in the first but not the second | Oracle calls it `MINUS`; MySQL from 8.0.31 |

Set operators treat NULLs as equal for duplicate removal, unlike `=`.

**Conditional expressions.**
```sql
CASE WHEN total_cents >= 10000 THEN 'large'
     WHEN total_cents >= 1000  THEN 'medium'
     ELSE 'small' END
COALESCE(a, b, 'n/a')     -- first non-NULL
NULLIF(a, b)              -- NULL if equal
GREATEST(a, b)  LEAST(a, b)   -- PostgreSQL/MySQL; they skip NULL in PostgreSQL
```
`CASE` evaluates branches in order and returns the first match; `ELSE` omitted yields NULL.

**Strings (PostgreSQL).** `||` concatenates (MySQL uses `CONCAT()` by default; SQL Server uses `+`). `lower`, `upper`, `trim`, `length`, `substring(s FROM 2 FOR 3)`, `position('x' IN s)`, `replace`, `split_part(s, ',', 1)`, `regexp_replace`, `left`, `right`. `LIKE` is case-sensitive, `ILIKE` is PostgreSQL's case-insensitive form, `~` and `~*` are regex matches. `LIKE 'abc%'` can use an index, `LIKE '%abc'` cannot (chapter 12). Always escape user text used in patterns (`%` and `_` are wildcards).

**Dates and times (PostgreSQL).** `now()` (transaction start), `clock_timestamp()` (actual time), `current_date`. Arithmetic with intervals: `created_at + interval '7 days'`. `date_trunc('month', ts)` truncates. `extract(epoch FROM ts)`, `extract(dow FROM ts)`. `age(a, b)`. `generate_series(start, stop, step)` builds calendars. `make_date(y, m, d)`. MySQL uses `DATE_ADD`, `DATE_FORMAT`; SQL Server uses `DATEADD`, `DATEDIFF`. `date_trunc` on a `timestamptz` truncates in the session time zone, so convert first (`date_trunc('day', ts AT TIME ZONE 'UTC')`) when you want UTC days.

**JSON (PostgreSQL `jsonb`).**
```sql
SELECT payload->'device'->>'os'              AS os,       -- -> returns jsonb, ->> returns text
       (payload->>'amount_cents')::bigint    AS cents,
       payload #>> '{geo,city}'              AS city      -- path as text
FROM events
WHERE payload @> '{"source": "mobile"}';                  -- containment
```
Index with GIN for containment: `CREATE INDEX ON events USING gin (payload);`. Index a single hot key with an expression index: `CREATE INDEX ON events ((payload->>'request_id'));`. `jsonb` removes duplicate keys and does not preserve key order; `json` stores text verbatim. `jsonb_build_object`, `jsonb_agg`, `jsonb_array_elements` construct and unnest. MySQL uses `JSON_EXTRACT` and the `->`/`->>` operators; SQL Server uses `JSON_VALUE`/`OPENJSON`.

**Use JSON for sparse, schema-flexible attributes**, not for fields you filter, join or constrain on. Those belong in columns.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Combining results | `UNION ALL`, `INTERSECT`, `EXCEPT` | Pick the correct operator and know the cost |
| Branching | `CASE`, `COALESCE`, `NULLIF` | Bucket and default values correctly |
| Strings | `lower`, `split_part`, `regexp_replace`, `LIKE`/`ILIKE` | Clean and match text safely |
| Dates | `date_trunc`, `interval`, `generate_series`, `AT TIME ZONE` | Build daily/monthly reports in the right zone |
| JSON | `->`, `->>`, `#>>`, `@>`, `jsonb_agg`, GIN index | Query and index semi-structured data |

### Common mistakes

- `UNION` where `UNION ALL` is intended, paying for needless de-duplication or hiding duplicates.
- Grouping by day in the session zone instead of the business zone.
- `LIKE` with unescaped user input.
- Filtering on `payload->>'k'` at scale with no expression or GIN index.
- Storing relational data as JSON and then needing joins.

### Self-check

- When is `UNION` required instead of `UNION ALL`?
- What does `->>` return compared to `->`?
- Why can `date_trunc('day', created_at)` give different days on two servers?

---

## 11. Classic interview patterns

**In plain words:** Most interview SQL tasks repeat a few patterns. Learn to recognise the pattern, then apply its standard solution.

### Understand

Recognise the pattern, then apply the template. Full solutions are in chapter 19 (problem numbers in brackets).

| Pattern | Core technique |
|---|---|
| Top-N per group | `row_number() OVER (PARTITION BY g ORDER BY x DESC)` then filter `rn <= N`, or `LATERAL ... LIMIT N` (4) |
| Dedupe, keep latest | `row_number()` partitioned by the business key ordered newest first, delete `rn > 1`; or `DISTINCT ON` in PostgreSQL (5, 6) |
| Running total | `sum(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` (7) |
| Period over period | `lag()` on an aggregated CTE (8) |
| Gaps and islands | subtract `row_number()` from the value (a date or integer); constant difference identifies one island (10) |
| Sessionization | flag a new session when `lag()` gap exceeds a threshold, then cumulative `sum(flag)` gives the session number (11) |
| Pivot | `count(*) FILTER (WHERE ...)` or `sum(CASE ...)` per column (12) |
| Upsert | `INSERT ... ON CONFLICT ... DO UPDATE` (13) |
| Anti-join | `NOT EXISTS` (1) |
| "Has all / has both" | `GROUP BY ... HAVING count(DISTINCT x) = n`, or double `EXISTS` (15, 16) |
| Fill missing periods | `generate_series` left joined to data (21) |
| Cohort / retention | first-event CTE joined back to later events (22) |
| Nth highest | `dense_rank()` or `DISTINCT ... OFFSET` (14) |

**Upsert detail (PostgreSQL).** `ON CONFLICT (cols) DO UPDATE SET ... ` updates the existing row, with the proposed row available as `EXCLUDED`. `DO NOTHING` skips. The conflict target must match a unique index or constraint. It is atomic and race-safe, unlike "select then insert". MySQL: `INSERT ... ON DUPLICATE KEY UPDATE`. SQL Server and standard SQL: `MERGE` (also PostgreSQL 15 and later). `DO NOTHING` returns no row for conflicts, so use `DO UPDATE` with a no-op set when `RETURNING` must always return the id.

**Method for any new problem.**
1. State the grain: one row per what in the output?
2. Identify the source tables and the join keys. Check for fan-out.
3. Decide where each filter belongs (`WHERE`, `ON`, `HAVING`, outer query).
4. Build in CTE steps and check row counts at each.
5. Test with ties, NULLs, empty groups and boundary dates.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pattern recall | chapter 19 problems | Write each pattern from memory |
| Verification | small hand-built datasets with edge cases | Prove a query correct, not just plausible |

### Common mistakes

- Starting to write before stating the output grain.
- Not testing ties and NULLs.

### Self-check

- Describe the gaps-and-islands trick in two sentences.
- Why is `ON CONFLICT` safer than select-then-insert?

---

## 12. Indexes

**In plain words:** An index is a sorted list that helps the database find rows fast, like the index of a book. It makes reads faster but writes slower. Add indexes only for queries you really run.

### Understand

**An index is a separate, sorted structure that lets the database find rows without scanning the whole table.** The cost: extra storage and slower writes, because every insert, update of an indexed column and delete must maintain each index. Index for the queries you actually run.

**B-tree** (the default) keeps keys sorted in a balanced tree. Lookup is logarithmic. It supports equality, ranges (`<`, `>`, `BETWEEN`), `ORDER BY` in index order (and reverse), `MIN`/`MAX`, and prefix matches (`LIKE 'abc%'`, which in PostgreSQL needs the `C` collation or an `text_pattern_ops` operator class). Other types: hash (equality only), GIN (arrays, `jsonb`, full text), GiST/SP-GiST (geometry, ranges), BRIN (huge, naturally ordered tables).

**Composite indexes and column order.** An index on `(a, b, c)` is sorted by `a`, then `b` within `a`, then `c`. It serves queries that constrain a **leftmost prefix** (`a`; `a, b`; `a, b, c`), not `b` alone. Ordering rule of thumb: equality columns first, then the range or sort column. For `WHERE customer_id = ? AND created_at >= ? ORDER BY created_at DESC`, use `(customer_id, created_at)`.

**Covering and index-only scans.** If every column the query needs is in the index, the table (heap) is not touched. PostgreSQL 11 and later: `CREATE INDEX ... ON t (a) INCLUDE (b)` stores extra payload columns without making them sort keys. Index-only scans also depend on the visibility map, so recently modified tables need `VACUUM`.

**Partial and expression indexes.**
```sql
CREATE INDEX ON payments (created_at) WHERE status = 'pending';  -- small, targeted
CREATE INDEX ON customers (lower(email));                        -- supports WHERE lower(email) = ?
```
The query must repeat the same expression or imply the partial predicate.

**Selectivity.** An index pays off when the predicate selects a small fraction of rows. An index on a low-cardinality column (a boolean, a three-value status) is usually ignored for a common value because reading most of the table sequentially is cheaper. It can still help for a rare value, and a partial index suits that case.

**Why an index is ignored (and sargability).** A predicate is **sargable** (search-argument-able) when the planner can turn it into an index range lookup. Common reasons it is not used:

| Cause | Example | Fix |
|---|---|---|
| Function on the indexed column | `WHERE date_trunc('day', created_at) = ...`, `WHERE lower(email) = ...` | Range predicate on the raw column, or an expression index |
| Type mismatch / implicit cast | comparing a `text` column to a number, or `bigint` to `numeric` params | Match the parameter type |
| Leading wildcard | `LIKE '%smith'` | Trigram index (`pg_trgm`) or full-text search |
| Arithmetic on column | `WHERE total_cents + 100 > 500` | Move arithmetic to the constant side |
| Wrong column order | index `(a, b)`, query filters only `b` | Add a matching index or reorder |
| Low selectivity | query returns 40% of rows | Sequential scan is correct |
| `OR` across columns | `a = 1 OR b = 2` | `UNION ALL` of two indexed queries, or bitmap OR |
| Stale statistics | row counts changed a lot | `ANALYZE` |
| Collation mismatch | prefix `LIKE` with a non-C collation | `text_pattern_ops` |
| Tiny table | a few pages | A sequential scan is faster |

**Foreign keys.** PostgreSQL does not auto-index referencing columns. Index them (`orders (customer_id)`), or joins and parent deletes scan the child table. (MySQL's InnoDB creates one automatically.)

**Operational notes.** `CREATE INDEX CONCURRENTLY` builds without blocking writes but cannot run inside a transaction block and can leave an invalid index on failure. Unused and duplicate indexes cost write performance; find them with `pg_stat_user_indexes`. Indexes bloat over time and may need `REINDEX CONCURRENTLY`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Choosing indexes | `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_statements` | Justify each index with a real query |
| Index usage | `pg_stat_user_indexes` (`idx_scan`), `\di+` | Find unused and oversized indexes |
| Building safely | `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY` | Add an index to a live table |
| Statistics | `ANALYZE`, `pg_stats` | Understand estimate errors |
| Specialised | GIN for `jsonb`, `pg_trgm` for substring search | Match index type to workload |

### Common mistakes

- Indexing every column "just in case".
- Composite index in the wrong column order.
- Wrapping the indexed column in a function.
- Forgetting indexes on foreign keys.
- Creating an index on a live large table without `CONCURRENTLY`.
- Adding an index and never confirming the planner uses it.

### Self-check

- Index on `(customer_id, created_at)`. Which of these use it: `WHERE customer_id = 5`; `WHERE created_at > ...`; `WHERE customer_id = 5 ORDER BY created_at`?
- Why does `WHERE date(created_at) = '2026-03-01'` skip a plain index, and how do you rewrite it?
- What is a covering index and what does `INCLUDE` add?

---

## 13. Reading an EXPLAIN plan

**In plain words:** `EXPLAIN` shows the steps the database will take to run your query. Compare its guesses with what really happened. A big difference between the two usually points to the problem.

### Understand

**`EXPLAIN` shows the plan the optimizer chose. `EXPLAIN (ANALYZE, BUFFERS)` runs the query and reports reality.** Compare estimates with actuals. Most performance problems are visible as a large gap between estimated and actual rows.

**Warning:** `ANALYZE` executes the statement. For `INSERT`/`UPDATE`/`DELETE`, wrap it: `BEGIN; EXPLAIN (ANALYZE) ...; ROLLBACK;`.

A plan is a tree read **from the innermost, most-indented node outward**. Illustrative output (numbers invented):

```
Index Scan using orders_customer_created_idx on orders
    (cost=0.43..8.45 rows=1 width=40) (actual time=0.03..0.04 rows=3 loops=1)
  Index Cond: (customer_id = 7)
  Buffers: shared hit=4
Planning Time: 0.1 ms
Execution Time: 0.06 ms
```

**Fields.**
- `cost=a..b`: planner units (startup..total), not milliseconds. Compare only within one plan.
- `rows`: estimated rows. `actual ... rows=`: real rows **per loop**. Multiply by `loops` for the total.
- `actual time`: milliseconds per loop, as startup..total.
- `Buffers: shared hit` (found in cache) versus `read` (from disk or OS cache). `dirtied`/`written` for writes.
- `Rows Removed by Filter`: rows read but discarded, a sign an index is missing or unselective.

**Scan nodes.**

| Node | Meaning |
|---|---|
| `Seq Scan` | read the whole table; fine for small tables or most of the rows |
| `Index Scan` | walk the index, fetch heap rows |
| `Index Only Scan` | answer from the index alone (watch `Heap Fetches`) |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | collect matching locations, then read pages in order; good for medium selectivity and combining indexes |

**Join and other nodes.** `Nested Loop` (good for few outer rows plus an inner index lookup; terrible if the inner is scanned many times), `Hash Join` (hash the smaller input), `Merge Join` (sorted inputs). `Sort` (watch `Sort Method: external merge Disk`, meaning memory was too small), `HashAggregate`/`GroupAggregate`, `Limit`, `Materialize`.

**Diagnosis checklist.**
1. Find the node with the largest actual time (subtract children's time).
2. Compare estimated versus actual rows. A factor of 10 or more suggests stale or insufficient statistics, correlated columns, or a function hiding the predicate. Try `ANALYZE`, extended statistics (`CREATE STATISTICS`), or rewriting.
3. Large `Rows Removed by Filter` on a scan: missing or wrong index.
4. `loops` very high on a nested-loop inner side: bad join order or missing index on the join key.
5. Sorts or hashes spilling to disk: raise `work_mem` for the session carefully, or reduce the data.
6. High `Buffers: read`: cold cache or a query touching too much data.
7. A `Seq Scan` is not automatically wrong. It is wrong when it scans a large table to return a few rows.

**Find the queries first.** `pg_stat_statements` aggregates execution statistics per normalised query (calls, total and mean time, rows, buffer use). Sort by total time, then explain the worst. Enable the extension via `shared_preload_libraries` and `CREATE EXTENSION pg_stat_statements`. `auto_explain` logs plans of slow statements. `log_min_duration_statement` logs slow queries.

**Plan caveats.** Parameterised or prepared statements can use a generic plan that differs from the plan for a literal value. Test with realistic data volumes. A plan on 1,000 rows says little about 100 million.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Plan reading | `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` | Identify the slowest node and why |
| Plan visualisers | pgAdmin graphical explain, DBeaver plan view, explain.depesz.com, explain.dalibo.com | Read large plans faster |
| Finding slow SQL | `pg_stat_statements`, `auto_explain`, `log_min_duration_statement`, `pg_stat_activity` | Rank queries by total cost |
| Statistics | `ANALYZE`, `CREATE STATISTICS`, `pg_stats` | Fix bad row estimates |
| Test without side effects | `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;` | Profile writes safely |

### Common mistakes

- Reading `cost` as time.
- Forgetting that `EXPLAIN ANALYZE` runs the statement.
- Tuning on a tiny dev dataset.
- Adding indexes without checking the estimated-versus-actual gap.

### Self-check

- A plan shows `rows=10` estimated and `rows=900000` actual. What do you suspect and do?
- What does `Rows Removed by Filter: 4999990` imply?
- Why run `EXPLAIN ANALYZE` on an `UPDATE` inside a transaction you roll back?

---

## 14. Transactions and isolation

**In plain words:** A transaction groups several changes so they all succeed or all fail. When many users work at the same time, isolation rules decide what each user can see. Stronger rules are safer but may force a retry.

### Understand

**A transaction is a unit of work that either fully happens or does not.** `BEGIN` ... `COMMIT` or `ROLLBACK`. Outside an explicit transaction, each statement is its own transaction (autocommit).

**ACID.**
- **Atomicity:** all changes commit or none do. Implemented with a write-ahead log and rollback.
- **Consistency:** a transaction moves the database from one valid state to another with respect to constraints. The database enforces declared constraints, the application enforces the rest.
- **Isolation:** concurrent transactions do not see each other's partial work, to a degree set by the isolation level.
- **Durability:** once committed, data survives a crash (the write-ahead log is flushed).

**MVCC (multi-version concurrency control).** PostgreSQL keeps multiple versions of a row. Readers see a consistent snapshot and do not block writers, and writers do not block readers. The cost is dead row versions that `VACUUM` reclaims. Long-running transactions hold back cleanup, so avoid leaving transactions open (including idle ones held by application code).

**Anomalies.**

| Anomaly | Description |
|---|---|
| Dirty read | reading another transaction's uncommitted change |
| Non-repeatable read | re-reading a row gives a different value because another transaction committed a change |
| Phantom read | re-running a range query returns different rows because another transaction inserted or deleted matching rows |
| Lost update | two transactions read a value, both compute a new one, the second write overwrites the first |
| Write skew | two transactions read overlapping data, each writes a different row based on what it read, and together they violate an invariant (two on-call doctors both go off-call; two bookings both pass a capacity check) |

**Isolation levels (SQL standard) and PostgreSQL behaviour.**

| Level | PostgreSQL behaviour |
|---|---|
| Read Uncommitted | treated as Read Committed; dirty reads never happen |
| Read Committed (default) | each statement sees data committed before it started. Non-repeatable reads, phantoms, lost updates and write skew are possible |
| Repeatable Read | one snapshot for the whole transaction (snapshot isolation). No dirty or non-repeatable reads, and phantoms are also prevented. A concurrent update of a row you modify raises a serialization failure. Write skew is still possible |
| Serializable | behaves as if transactions ran one at a time (serializable snapshot isolation). Prevents write skew. Aborts with SQLSTATE `40001` when it detects a dangerous pattern |

MySQL InnoDB defaults to Repeatable Read with different locking semantics (gap locks) and a different snapshot model, so do not assume PostgreSQL behaviour carries over. SQL Server defaults to Read Committed (locking, or row versioning if enabled).

**Fixing lost updates.**
1. Make the update atomic in SQL: `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;` and check the affected row count.
2. Lock the row: `SELECT ... FOR UPDATE` then update (chapter 15).
3. **Optimistic concurrency**: add a `version` column, `UPDATE ... SET ..., version = version + 1 WHERE id = ? AND version = ?`, and retry if zero rows were updated.
4. Use Repeatable Read or Serializable and retry on `40001`.

**Fixing write skew.** Serializable with retries, or lock the rows that form the invariant (lock a parent row), or enforce with a constraint (unique or exclusion constraint) where possible.

**Retry logic is mandatory** with Repeatable Read, Serializable, and deadlocks (`40P01`). Transactions must be safe to re-run from the start, and must not perform non-idempotent side effects (sending email, charging a card) inside the retried block.

**Transaction hygiene.** Keep transactions short. Do not hold one open across a network call or user think time. Do not mix reads that drive logic with writes in separate transactions when the logic needs consistency.

**Savepoints** (`SAVEPOINT s; ... ROLLBACK TO s;`) allow partial rollback inside a transaction.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Control | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `SET TRANSACTION ISOLATION LEVEL` | Scope transactions deliberately |
| Observe | `pg_stat_activity` (`state`, `xact_start`, `wait_event`), `pg_locks` | Find idle-in-transaction and blocked sessions |
| Reproduce anomalies | two `psql` sessions side by side | Demonstrate each anomaly and its fix |
| Retry patterns | application driver error codes (`40001`, `40P01`) | Implement bounded retry |

### Common mistakes

- Read-modify-write in application code with no lock or version check.
- Assuming Read Committed prevents phantoms or write skew.
- Not retrying serialization failures.
- Long transactions holding locks and blocking vacuum.
- External side effects inside a transaction that may roll back or retry.
- Assuming `COMMIT` ordering equals timestamp or id ordering.

### Self-check

- Walk through a lost update between two requests and give three fixes.
- What is write skew, and which isolation level prevents it in PostgreSQL?
- Why can a transaction at Read Committed see two different results for the same query?

---

## 15. Locking

**In plain words:** A lock makes other sessions wait while one session changes a row or table. Locks keep data correct but can slow the system or block it. Keep transactions short and lock rows in a fixed order.

### Understand

**Locks serialise conflicting access.** In PostgreSQL, ordinary reads take no row locks. Writes lock the rows they modify until commit. Explicit row locks let you claim rows deliberately.

**Row-level locking clauses** (at the end of a `SELECT`):

| Clause | Meaning |
|---|---|
| `FOR UPDATE` | lock the rows against update, delete and other lockers |
| `FOR NO KEY UPDATE` | weaker; allows concurrent foreign-key checks referencing the row |
| `FOR SHARE` / `FOR KEY SHARE` | block writers but allow other shared lockers |
| `NOWAIT` | fail immediately instead of waiting for a lock |
| `SKIP LOCKED` | skip rows already locked by others |

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;   -- hypothetical accounts table
-- compute in the application
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**Work queues with `SKIP LOCKED`.** Several workers claim distinct rows without blocking each other:

```sql
WITH next AS (
  SELECT id FROM payments
  WHERE status = 'pending'
  ORDER BY id
  LIMIT 10
  FOR UPDATE SKIP LOCKED
)
UPDATE payments p
SET status = 'processing'
FROM next
WHERE p.id = next.id
RETURNING p.*;
```

It provides at-least-once processing with a status column and needs a recovery path for rows stuck in `processing` after a worker crash (a timestamp and a reaper). `SKIP LOCKED` returns an inconsistent view by design, so use it for queues only, not for general reads. MySQL 8.0 supports `SKIP LOCKED` and `NOWAIT`. SQL Server uses `READPAST` and `UPDLOCK` hints.

**Deadlocks.** Two transactions each hold a lock the other needs. The database detects it and aborts one (`40P01`). Prevent by acquiring locks in a consistent order (for example, ascending id), keeping transactions short, and retrying. `SELECT ... WHERE id IN (...) FOR UPDATE` with `ORDER BY id` gives a consistent order.

**Table locks and DDL.** `ALTER TABLE` takes strong locks. A DDL statement waiting behind a long transaction blocks every later query on the table, producing an outage from a "small" migration. Use `lock_timeout` (chapter 17).

**Advisory locks.** `pg_advisory_lock(key)` and `pg_try_advisory_lock(key)` provide application-defined locks (for example, "only one instance runs the nightly job"). Session-level advisory locks are released on disconnect, which interacts badly with connection pools, so prefer the transaction-level variants (`pg_advisory_xact_lock`).

**Idempotency instead of locking.** A unique constraint on an external reference (`payments.provider_ref`) with `ON CONFLICT DO NOTHING` makes retries safe without explicit locks.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Claiming rows | `FOR UPDATE`, `SKIP LOCKED`, `NOWAIT` | Build a safe multi-worker queue |
| Diagnosing blocking | `pg_locks`, `pg_stat_activity`, `pg_blocking_pids(pid)` | Identify who blocks whom |
| Timeouts | `lock_timeout`, `statement_timeout`, `idle_in_transaction_session_timeout` | Bound waits and runaway sessions |
| Coordination | advisory locks | Run exactly-one jobs |

### Common mistakes

- Locking rows in inconsistent order across code paths.
- Holding `FOR UPDATE` locks while calling an external service.
- Queue workers without a recovery path for crashed claims.
- Running a migration with no `lock_timeout`.
- Session-level advisory locks behind a pooler.

### Self-check

- How does `SKIP LOCKED` let ten workers share one queue table?
- What causes a deadlock, and how do you prevent it structurally?
- Why can a harmless `ALTER TABLE` take the whole application down?

---

## 16. Normalisation and denormalisation

**In plain words:** Normalisation means storing each fact in one place only. This avoids copies that disagree. Denormalisation adds copies on purpose, but only when measurements show a need.

### Understand

**Normalisation removes redundancy so each fact is stored once**, which prevents update, insert and delete anomalies (changing a customer's address in one place but not another).

- **First normal form (1NF):** every column holds one atomic value, no repeating groups or comma-separated lists, and rows are uniquely identifiable.
- **Second normal form (2NF):** in 1NF, and every non-key column depends on the **whole** key. Relevant for composite keys: `order_items (order_id, line_no, product_name)` where `product_name` depends only on the SKU violates it.
- **Third normal form (3NF):** in 2NF, and non-key columns depend **only on the key**, not on other non-key columns. `orders (id, customer_id, customer_country)` where country depends on the customer violates it. Memory aid: every attribute depends on the key, the whole key, and nothing but the key.
- **BCNF** is a stricter 3NF. Higher forms (4NF, 5NF) address multi-valued and join dependencies and rarely matter day to day.

**Snapshots are not violations.** `order_items.unit_price_cents` copies the product price at purchase time. That is a historical fact, not redundancy: the price may change later and the order must not. Likewise store a shipping address on the order, not only a pointer to a mutable address book.

**Denormalise deliberately, for measured reasons.** Typical cases: read-heavy dashboards needing precomputed counts, avoiding an expensive join on a hot path, search documents, materialised views for reports. Costs: write amplification and drift between copies. Controls:
- Keep a normalised source of truth.
- Derive copies with a trigger, a materialised view (`REFRESH MATERIALIZED VIEW [CONCURRENTLY]`), or a background job.
- Add reconciliation queries that detect drift.
- Document why the denormalisation exists.

**Do not denormalise first.** A correctly indexed join is usually fast enough. Prove the need with `EXPLAIN` and measured latency.

**Other modelling notes.**
- Many-to-many via a junction table with a composite primary key.
- Soft deletes (`deleted_at`) complicate uniqueness and every query. Use partial unique indexes (`WHERE deleted_at IS NULL`) if you adopt them.
- Avoid EAV (entity-attribute-value) tables for core data. Use columns, or `jsonb` for sparse extras.
- Append-only tables (ledgers, events) suit audit and money movement: record entries, derive balances.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Analysis | functional dependency reasoning, ER diagrams | Normalise a messy table to 3NF |
| Derived data | materialised views, triggers, generated columns | Maintain a denormalised copy safely |
| Drift detection | reconciliation queries | Prove a cached total equals the source |

### Common mistakes

- Comma-separated lists or arrays in a column used for joins.
- Copying a mutable attribute (customer name) into many tables without a refresh plan.
- Mistaking a historical snapshot for redundancy and "fixing" it.
- Premature denormalisation with no measurement.

### Self-check

- Normalise `orders (id, customer_id, customer_email, product, product_category)` to 3NF.
- Why is `order_items.unit_price_cents` allowed to duplicate a product price?
- Name two controls that keep a denormalised counter correct.

---

## 17. Safe schema changes

**In plain words:** Changing a table while users are online can block them. Make changes in small steps that old and new code can both handle. Set a lock time limit so a change fails fast instead of waiting.

### Understand

**Production schema changes run while traffic is live, with old and new application versions both running.** Two hazards: locks that stall traffic, and incompatibility between schema and code during deployment.

**Expand and contract (parallel change).** Never make a breaking change in one step.
1. **Expand:** add the new structure (nullable column, new table) in a backward-compatible way.
2. **Migrate:** deploy code that writes both old and new, then backfill existing rows in small batches.
3. **Switch:** deploy code that reads the new structure.
4. **Contract:** after all old code is gone, drop the old structure.

Renaming a column in place breaks the still-running old version. Add a new column, dual-write, backfill, switch, then drop.

**Lock-safe operations (PostgreSQL).**

| Task | Safe approach |
|---|---|
| Add a column | `ADD COLUMN` nullable, or with a constant default (fast since PostgreSQL 11, no table rewrite). Volatile defaults still rewrite |
| Make a column `NOT NULL` | Add `CHECK (col IS NOT NULL) NOT VALID`, then `VALIDATE CONSTRAINT` (weaker lock), then `SET NOT NULL` (PostgreSQL 12 and later can skip the scan using the validated check), then drop the check |
| Add a foreign key | `ADD CONSTRAINT ... FOREIGN KEY ... NOT VALID`, then `VALIDATE CONSTRAINT` |
| Add an index | `CREATE INDEX CONCURRENTLY` |
| Add a unique constraint | build `CREATE UNIQUE INDEX CONCURRENTLY`, then `ADD CONSTRAINT ... UNIQUE USING INDEX` |
| Change a column type | usually rewrites the table and blocks writes. Use expand and contract with a new column |
| Drop a column | after no code references it. It is a fast catalog change, but deploy order matters |
| Backfill | batches by primary key range (a few thousand rows), short transactions, throttle, commit each batch |

```sql
SET lock_timeout = '3s';          -- fail fast instead of queueing behind a long transaction
SET statement_timeout = '15min';  -- for migrations that legitimately run longer, set deliberately
ALTER TABLE orders ADD COLUMN channel text;
```

**Why `lock_timeout` matters.** An `ALTER TABLE` needs an exclusive lock. If a long transaction holds a conflicting lock, the `ALTER` waits, and every query that arrives after it queues behind the `ALTER`. A 50 ms change causes minutes of outage. A short `lock_timeout` makes it fail instead, and you retry at a quieter moment.

**Migration discipline.**
- Use a versioned migration tool and commit migrations with the code. Never hand-edit production schema without a migration file.
- Make each migration small and, where possible, reversible. Keep data backfills separate from schema migrations.
- Test on production-sized data. A statement that takes 10 ms on a dev database may rewrite a billion rows.
- Take and verify backups before destructive changes. Practice restore.
- MySQL specifics differ (online DDL algorithms, implicit commit on DDL). Read your version's online DDL documentation.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Migrations | Flyway, Liquibase, Alembic, Prisma Migrate, Knex, Rails/Django migrations | Review a migration for lock and compatibility risk |
| Lock awareness | `lock_timeout`, `pg_locks`, `pg_stat_activity` | Predict the lock a statement takes |
| Safe patterns | `NOT VALID` plus `VALIDATE`, `CONCURRENTLY`, batched backfills | Apply each without blocking |
| Linting | `sqlfluff`, migration linters such as squawk | Catch risky statements in CI |
| Backup and restore | `pg_dump`, `pg_restore`, point-in-time recovery | Prove a restore works |

### Common mistakes

- Renaming or dropping a column that running code still uses.
- `CREATE INDEX` without `CONCURRENTLY` on a hot table.
- Adding `NOT NULL` with a full-table scan under an exclusive lock.
- One huge `UPDATE` backfill that holds locks and bloats the table.
- No `lock_timeout` on migrations.
- Migrations never tested at production scale.

### Self-check

- Describe expand and contract for renaming `orders.status` to `orders.state`.
- Why does an `ALTER TABLE` waiting for a lock block unrelated queries?
- How do you add a `NOT NULL` constraint to a large live table with minimal locking?

---

## 18. Application-side mistakes

**In plain words:** Most database problems start in the application code, not in SQL. Typical causes are too many small queries, unbounded results and text built from user input. This chapter shows safe habits.

### Understand

Most database incidents are caused by how the application uses SQL, not by SQL itself.

**N+1 queries.** One query fetches N parents, then the code issues one query per parent for its children. 1 + N round trips instead of 1 or 2. ORMs hide it behind lazy loading. Fixes: one join, or two queries using a batch (`WHERE customer_id = ANY($1)` with an array parameter, or `IN (...)`), or ORM eager loading. Detect it by counting queries per request and watching `pg_stat_statements` for a cheap query with enormous `calls`.

**Unbounded queries.** `SELECT * FROM events` with no `LIMIT` loads the whole table into memory. Always bound result sizes, stream large exports with a cursor (`DECLARE ... CURSOR`, or driver streaming) and set `statement_timeout`.

**OFFSET pagination versus keyset.** `OFFSET n` makes the database read and discard n rows, so deep pages get slower linearly. It is also unstable: rows inserted or deleted during browsing cause duplicates or skips. **Keyset (seek) pagination** remembers the last row's sort key and asks for rows after it:

```sql
SELECT id, created_at, total_cents
FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)        -- row-value comparison; last row of previous page
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

It uses an index `(customer_id, created_at, id)` for constant-time pages. Requirements: a deterministic total order (include a unique tie-breaker like `id`), and a cursor that carries the last key. The drawback is no jump to page N. Row-value comparison works in PostgreSQL and MySQL. SQL Server needs the expanded boolean form.

**SQL injection.** Building SQL by string concatenation with user input lets an attacker change the query's structure.

```
-- vulnerable (pseudo-code)
query = "SELECT * FROM customers WHERE email = '" + input + "'"
-- input:  ' OR '1'='1
```

Defence: **parameterised queries (prepared statements)**. The statement text and the values travel separately, so values are never parsed as SQL.

```sql
SELECT id, name FROM customers WHERE email = $1;     -- PostgreSQL placeholder
-- MySQL and many drivers use ?, SQL Server uses @name
```

Rules:
- Placeholders bind **values**, not identifiers (table, column names) or keywords such as sort direction. For dynamic `ORDER BY` or column names, map user input to a fixed allow-list in code, or quote identifiers with the driver's identifier-quoting function.
- Never "escape manually". Use the driver.
- ORMs and query builders are safe when you use their parameter binding, and unsafe when you pass raw interpolated strings.
- Apply least privilege: the application role should not own tables or be a superuser, so injection has limited impact.
- `LIKE` patterns still need wildcard escaping for the user's text even when parameterised.

**Connections.** Each PostgreSQL connection is a process with real memory cost. Use a pool (for example, PgBouncer or an in-app pool) and size it to the database's capacity, not to request concurrency. Release connections promptly. Pool exhaustion looks like a database outage. Beware `idle in transaction` sessions from code that begins a transaction and then waits.

**Other application mistakes.**
- Selecting all columns and filtering in application code instead of in SQL.
- Doing joins and aggregation in application loops.
- Counting with `SELECT count(*)` over a huge table for every page view. Use approximations, cached counters, or avoid showing total counts.
- Using the application clock (`now()` in app code) inconsistently with database time.
- Ignoring the timezone of the connection when reading `timestamptz`.
- Retrying non-idempotent writes with no unique key.
- Swallowing constraint errors instead of translating them (unique violation `23505` to a "conflict" response).
- Long transactions spanning HTTP calls.
- Treating the database as a log of debug output.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Spotting N+1 | ORM query logging, `pg_stat_statements` (`calls`), APM traces | Prove an endpoint issues 1 + N queries |
| Safe SQL from code | prepared statements, driver parameter binding, identifier allow-lists | Never concatenate user input |
| Pagination | keyset queries with composite index | Implement stable cursor pagination |
| Pooling | PgBouncer, driver pool settings, `pg_stat_activity` | Size and monitor a pool |
| Safety nets | `statement_timeout`, `idle_in_transaction_session_timeout`, read-only roles | Bound the damage of mistakes |

### Common mistakes

- Concatenating strings into SQL because "it is just an internal tool".
- Parameterising values but interpolating column names from user input.
- `OFFSET 100000` on a feed table.
- Lazy loading inside a loop in list endpoints.
- A pool size larger than the database can serve.

### Self-check

- Why does a placeholder prevent injection, and what can it not parameterise?
- Convert an `OFFSET`-paginated feed to keyset pagination. What index supports it?
- How would you detect N+1 in a running service?

---

## 19. Worked query problems

**In plain words:** These are 26 practice problems with solutions. Try each one first, then compare. The queries were checked by reading, not by running, so test them on your own data.

All use the reference schema. Each gives the problem, a solution and the reasoning to remember. Assume PostgreSQL, and amounts in one currency (add `AND currency = 'USD'` or group by currency in real work, since summing across currencies is meaningless).

### 1. Customers who never ordered (anti-join)

```sql
SELECT c.id, c.email
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
Equivalent: `LEFT JOIN orders o ON o.customer_id = c.id WHERE o.id IS NULL`. Avoid `NOT IN` because a NULL `customer_id` empties the result.

### 2. Paid revenue per customer, including customers with none

```sql
SELECT c.id, c.name, COALESCE(SUM(o.total_cents), 0) AS revenue_cents
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'paid'
GROUP BY c.id, c.name;
```
The status filter is in `ON`. In `WHERE` it would drop customers with no paid orders.

### 3. Orders whose stored total differs from the sum of their items (avoid fan-out)

```sql
SELECT o.id, o.total_cents, i.items_cents
FROM orders o
JOIN (
  SELECT order_id, SUM(quantity * unit_price_cents) AS items_cents
  FROM order_items GROUP BY order_id
) i ON i.order_id = o.id
WHERE o.total_cents <> i.items_cents;
```
Aggregating items first keeps one row per order, so no total is repeated. (Orders with no items are excluded by the inner join.)

### 4. Top 3 orders by total per customer

```sql
SELECT customer_id, id, total_cents
FROM (
  SELECT o.*, row_number() OVER (PARTITION BY customer_id
                                 ORDER BY total_cents DESC, id) AS rn
  FROM orders o
) t
WHERE rn <= 3;
```
The `id` tie-breaker makes the choice deterministic. Use `rank()` instead if ties should all be kept.

### 5. Latest order per customer

```sql
SELECT DISTINCT ON (customer_id) customer_id, id, created_at, total_cents
FROM orders
ORDER BY customer_id, created_at DESC, id DESC;
```
`DISTINCT ON` is PostgreSQL-specific. Portable form: `row_number() OVER (PARTITION BY customer_id ORDER BY created_at DESC, id DESC)` and filter `rn = 1`. The `ORDER BY` must begin with the `DISTINCT ON` expressions.

### 6. Delete duplicate events, keeping the latest per key

Duplicates share `(customer_id, event_type, payload->>'request_id')`.

```sql
DELETE FROM events
WHERE id IN (
  SELECT id FROM (
    SELECT id,
           row_number() OVER (
             PARTITION BY customer_id, event_type, payload->>'request_id'
             ORDER BY occurred_at DESC, id DESC) AS rn
    FROM events
  ) t
  WHERE rn > 1
);
```
Run the inner `SELECT` first to review what would go. Afterwards add a unique index on the key to prevent recurrence. Rows with NULL `customer_id` are partitioned together (partitioning treats NULLs as equal), which may or may not be intended.

### 7. Running total of daily paid revenue

```sql
WITH daily AS (
  SELECT (created_at AT TIME ZONE 'UTC')::date AS day, SUM(total_cents) AS rev
  FROM orders WHERE status = 'paid'
  GROUP BY 1
)
SELECT day, rev,
       SUM(rev) OVER (ORDER BY day
                      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_cents
FROM daily
ORDER BY day;
```

### 8. Month-over-month revenue change

```sql
WITH m AS (
  SELECT date_trunc('month', created_at AT TIME ZONE 'UTC') AS month,
         SUM(total_cents) AS rev
  FROM orders WHERE status = 'paid'
  GROUP BY 1
), l AS (
  SELECT month, rev, LAG(rev) OVER (ORDER BY month) AS prev_rev FROM m
)
SELECT month, rev, prev_rev,
       ROUND(100.0 * (rev - prev_rev) / NULLIF(prev_rev, 0), 1) AS pct_change
FROM l
ORDER BY month;
```
`LAG` is NULL for the first month, and `NULLIF` avoids division by zero. Months with no orders are absent, so use problem 21 first if gaps matter.

### 9. Seven-day moving average of daily revenue

```sql
WITH days AS (
  SELECT d::date AS day
  FROM generate_series(timestamp '2026-01-01', timestamp '2026-03-31', interval '1 day') AS d
), daily AS (
  SELECT (created_at AT TIME ZONE 'UTC')::date AS day, SUM(total_cents) AS rev
  FROM orders WHERE status = 'paid' GROUP BY 1
)
SELECT days.day,
       AVG(COALESCE(daily.rev, 0)) OVER (ORDER BY days.day
                                         ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg7
FROM days LEFT JOIN daily USING (day)
ORDER BY days.day;
```
A `ROWS` frame counts rows, so the calendar must have one row per day. The first six days average fewer than seven values.

### 10. Longest login streak per customer (gaps and islands)

```sql
WITH d AS (
  SELECT DISTINCT customer_id, (occurred_at AT TIME ZONE 'UTC')::date AS day
  FROM events WHERE event_type = 'login' AND customer_id IS NOT NULL
), g AS (
  SELECT customer_id, day,
         day - (row_number() OVER (PARTITION BY customer_id ORDER BY day))::int AS grp
  FROM d
), streaks AS (
  SELECT customer_id, MIN(day) AS start_day, MAX(day) AS end_day, COUNT(*) AS days
  FROM g GROUP BY customer_id, grp
)
SELECT DISTINCT ON (customer_id) customer_id, start_day, end_day, days
FROM streaks
ORDER BY customer_id, days DESC;
```
Consecutive days minus consecutive row numbers give a constant, which labels each island. De-duplicate days first, or the row numbers drift.

### 11. Sessionise events with a 30-minute inactivity gap

```sql
WITH flagged AS (
  SELECT customer_id, occurred_at,
         CASE WHEN LAG(occurred_at) OVER w IS NULL
                OR occurred_at - LAG(occurred_at) OVER w > interval '30 minutes'
              THEN 1 ELSE 0 END AS is_new
  FROM events
  WHERE customer_id IS NOT NULL
  WINDOW w AS (PARTITION BY customer_id ORDER BY occurred_at)
), numbered AS (
  SELECT *, SUM(is_new) OVER (PARTITION BY customer_id ORDER BY occurred_at
                              ROWS UNBOUNDED PRECEDING) AS session_no
  FROM flagged
)
SELECT customer_id, session_no,
       MIN(occurred_at) AS started, MAX(occurred_at) AS ended, COUNT(*) AS events
FROM numbered
GROUP BY customer_id, session_no;
```
A cumulative sum of "session start" flags numbers the sessions.

### 12. Order counts by status per month (pivot)

```sql
SELECT date_trunc('month', created_at AT TIME ZONE 'UTC')::date AS month,
       COUNT(*) FILTER (WHERE status = 'pending')   AS pending,
       COUNT(*) FILTER (WHERE status = 'paid')      AS paid,
       COUNT(*) FILTER (WHERE status = 'shipped')   AS shipped,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled
FROM orders
GROUP BY 1
ORDER BY 1;
```
Columns are fixed at write time. Truly dynamic pivots need generated SQL or application code. Portable form: `SUM(CASE WHEN status = 'paid' THEN 1 ELSE 0 END)`.

### 13. Upsert a customer by email, and make a payment insert idempotent

```sql
INSERT INTO customers (email, name, country)
VALUES ($1, $2, $3)
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
RETURNING id;

INSERT INTO payments (order_id, amount_cents, status, provider_ref)
VALUES ($1, $2, 'captured', $3)
ON CONFLICT (provider_ref) DO NOTHING;
```
The first always returns the id. The second silently ignores a replayed webhook; check the affected row count (0 means duplicate).

### 14. Second-highest distinct order total

```sql
SELECT total_cents
FROM (
  SELECT total_cents, DENSE_RANK() OVER (ORDER BY total_cents DESC) AS r
  FROM orders
) t
WHERE r = 2
GROUP BY total_cents;
```
`dense_rank` handles ties. Alternative: `SELECT DISTINCT total_cents FROM orders ORDER BY total_cents DESC OFFSET 1 LIMIT 1`. Both return no rows when there is no second value, which is the correct answer to "second highest" (do not return the highest).

### 15. Customers whose orders are all paid (and who have at least one)

```sql
SELECT c.id
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)
  AND NOT EXISTS (SELECT 1 FROM orders o
                  WHERE o.customer_id = c.id AND o.status <> 'paid');
```
"All" is "not exists a counterexample". The first condition excludes customers with no orders, for whom "all" is vacuously true.

### 16. Customers who bought both SKU 'A' and SKU 'B'

```sql
SELECT o.customer_id
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE oi.product_sku IN ('A', 'B')
GROUP BY o.customer_id
HAVING COUNT(DISTINCT oi.product_sku) = 2;
```
The `DISTINCT` matters when a customer bought A twice. Alternative: `INTERSECT` of two selects.

### 17. Paid orders whose captured payments do not equal the total

```sql
SELECT o.id, o.total_cents, COALESCE(p.captured, 0) AS captured_cents
FROM orders o
LEFT JOIN (
  SELECT order_id, SUM(amount_cents) AS captured
  FROM payments WHERE status = 'captured'
  GROUP BY order_id
) p ON p.order_id = o.id
WHERE o.status = 'paid'
  AND o.total_cents <> COALESCE(p.captured, 0);
```
A reconciliation query. Aggregating payments before the join avoids fan-out, and the `LEFT JOIN` catches paid orders with no payment at all.

### 18. Time from signup to first order

```sql
SELECT c.id, MIN(o.created_at) - c.created_at AS time_to_first_order
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.created_at;
```
Subtracting two `timestamptz` values gives an interval with no time-zone ambiguity. Use `LEFT JOIN` to keep customers who never ordered (the result is NULL).

### 19. Each country's share of paid revenue

```sql
SELECT c.country,
       SUM(o.total_cents) AS rev_cents,
       ROUND(100.0 * SUM(o.total_cents) / SUM(SUM(o.total_cents)) OVER (), 2) AS pct
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'paid' AND o.currency = 'USD'
GROUP BY c.country
ORDER BY rev_cents DESC;
```
A window function can wrap an aggregate: the window runs after grouping. `100.0 *` forces decimal division.

### 20. Walk a category tree with depth and path (recursive CTE)

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth, name AS path
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1, t.path || ' > ' || c.name
  FROM categories c
  JOIN tree t ON c.parent_id = t.id
  WHERE t.depth < 20
)
SELECT id, depth, path FROM tree ORDER BY path;
```
The depth bound is a cycle guard. To collect all descendants of one node, change the anchor to `WHERE id = $1`.

### 21. Daily order counts with zero-filled missing days

```sql
SELECT d::date AS day, COALESCE(x.n, 0) AS orders
FROM generate_series(timestamp '2026-01-01', timestamp '2026-01-31', interval '1 day') AS d
LEFT JOIN (
  SELECT (created_at AT TIME ZONE 'UTC')::date AS day, COUNT(*) AS n
  FROM orders GROUP BY 1
) x ON x.day = d::date
ORDER BY day;
```
Use plain `timestamp` arguments so the series has no session-zone surprises. The calendar drives the join, so empty days survive.

### 22. Repeat-purchase rate by first-order-month cohort

Share of customers by first-order month who ordered again within 30 days of their first order.

```sql
WITH first_o AS (
  SELECT customer_id, MIN(created_at) AS first_at
  FROM orders GROUP BY customer_id
), repeaters AS (
  SELECT DISTINCT f.customer_id
  FROM first_o f
  JOIN orders o ON o.customer_id = f.customer_id
               AND o.created_at >  f.first_at
               AND o.created_at <  f.first_at + interval '30 days'
)
SELECT date_trunc('month', f.first_at AT TIME ZONE 'UTC')::date AS cohort,
       COUNT(*)            AS customers,
       COUNT(r.customer_id) AS repeated
FROM first_o f
LEFT JOIN repeaters r ON r.customer_id = f.customer_id
GROUP BY 1
ORDER BY 1;
```
`COUNT(r.customer_id)` counts only matched (non-NULL) rows. Divide in the outer query for a rate.

### 23. Median paid order value

```sql
SELECT percentile_cont(0.5) WITHIN GROUP (ORDER BY total_cents) AS median_cents
FROM orders
WHERE status = 'paid';
```
`percentile_cont` interpolates between the two middle values, and `percentile_disc` returns an actual value. Averages hide skew, so report medians or percentiles for latency and value distributions.

### 24. Keyset pagination of a customer's orders

First page:

```sql
SELECT id, created_at, total_cents
FROM orders
WHERE customer_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 50;
```

Next page, passing the last row's `created_at` and `id`:

```sql
SELECT id, created_at, total_cents
FROM orders
WHERE customer_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 50;
```
Supported by `CREATE INDEX ON orders (customer_id, created_at, id);` (read backward for descending order). Constant cost per page and stable under inserts.

### 25. Claim a batch of pending payments for a worker

```sql
WITH next AS (
  SELECT id FROM payments
  WHERE status = 'pending'
  ORDER BY id
  LIMIT 10
  FOR UPDATE SKIP LOCKED
)
UPDATE payments p
SET status = 'processing'
FROM next
WHERE p.id = next.id
RETURNING p.id, p.order_id, p.amount_cents;
```
Concurrent workers receive disjoint rows. Add a `claimed_at` column and a reaper for crashed workers. A partial index `(id) WHERE status = 'pending'` keeps the claim fast.

### 26. Emails that collide when compared case-insensitively

```sql
SELECT lower(email) AS email_key, COUNT(*) AS n, array_agg(id ORDER BY id) AS ids
FROM customers
GROUP BY lower(email)
HAVING COUNT(*) > 1;
```
Prevent it with `CREATE UNIQUE INDEX ON customers (lower(email));`, after cleaning the existing duplicates.

---

## 20. Practice labs

**In plain words:** Labs are hands-on tasks. You build data, break things on purpose, and fix them. Doing the task teaches more than reading about it.

Do these on a local PostgreSQL instance (a container or a native install). Load the reference schema and generate data with `generate_series` and `random()`, including ties, NULLs and skew. Each lab states what "done" looks like. Attempt before looking anything up.

1. **Build the schema.** Create the reference schema with every constraint. Try to violate each constraint and record the error code. Done: you can name the failing constraint from the message alone.
2. **NULL traps.** Insert a NULL `customer_id` where the schema permits it (temporarily relax it in a scratch table), then show `NOT IN` returning nothing and `NOT EXISTS` working. Compare `COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`.
3. **Fan-out.** Join orders to order_items and to payments, sum order totals, and show the inflated number. Rewrite with pre-aggregation and prove both totals match `SELECT SUM(total_cents) FROM orders`.
4. **Window functions.** Implement problems 4, 5, 7, 8 and 10 from memory on generated data. Add ties and show the difference between `row_number`, `rank` and `dense_rank`, and the `last_value` default-frame trap.
5. **Generate a million orders** and run a customer-lookup query with no index. Capture `EXPLAIN (ANALYZE, BUFFERS)`, add `(customer_id, created_at)`, and compare the plans and timings.
6. **Why is my index ignored?** Create cases for a function on the column, a type mismatch, a leading wildcard and a low-selectivity predicate. Fix each (expression index, cast, trigram index, partial index) and verify in the plan.
7. **Estimate errors.** Create correlated columns, observe a bad row estimate, then fix with `ANALYZE` or `CREATE STATISTICS` and re-read the plan.
8. **Find the slow queries.** Enable `pg_stat_statements`, run a mixed workload (use `pgbench` with a custom script), and rank queries by total time. Explain the top three.
9. **Reproduce anomalies.** With two `psql` sessions, demonstrate a non-repeatable read at Read Committed, a lost update, a write skew (an invariant across two rows) and its prevention with Serializable and retry.
10. **Optimistic locking.** Add a `version` column and implement a conflict-detecting update. Show two clients racing, and one retrying.
11. **Work queue.** Build a worker loop using `FOR UPDATE SKIP LOCKED` (problem 25), run several workers, prove no row is processed twice, then kill one mid-batch and write the recovery query.
12. **Deadlock.** Create a deadlock with two sessions updating two rows in opposite order, read the server log message, and fix it by consistent ordering.
13. **Safe migration.** On a table with several million rows, add a `NOT NULL` column through expand and contract: nullable add, batched backfill, `NOT VALID` check, validate, set not null. Measure lock waits with `pg_stat_activity` while a query load runs. Repeat the naive way and compare.
14. **Blocked DDL.** Hold a transaction open in one session, run `ALTER TABLE` in another with no `lock_timeout`, and show a third session's plain `SELECT` blocking. Repeat with `lock_timeout`.
15. **Pagination.** Implement OFFSET and keyset pagination over a large table. Compare page 1 and page 10,000 timings, and show duplicates under concurrent inserts with OFFSET.
16. **Injection.** In a throwaway app, write a vulnerable query, exploit it, then fix with parameters. Show why a dynamic `ORDER BY` column needs an allow-list.
17. **N+1.** Build a list endpoint with lazy loading, count the queries, then fix with a join and with a batched `ANY($1)` query. Compare latency.
18. **Normalise a mess.** Take a denormalised spreadsheet-style table, normalise it to 3NF, migrate the data with SQL, and write reconciliation queries proving nothing was lost.
19. **Lint and format.** Run `sqlfluff lint` with `--dialect postgres` over your lab SQL, fix findings, and add it to a pre-commit hook.
20. **Incident write-up.** Pick one failure you caused (a slow query, a lock pile-up, a bad migration) and write a one-page timeline: symptom, evidence from plans and views, cause, fix, prevention.

---

## 21. Interview questions

**In plain words:** These are common SQL interview questions. Each has a short model answer. Use the answers to check your own, and say them in your own words. Interviews differ, so this list cannot predict any particular one.

### Top 10 must-know

Start here. The numbers point to the model answers below.

| # | Level | Question |
|---|---|---|
| 3 | [Fresher] | `INNER JOIN` versus `LEFT JOIN` |
| 4 | [Fresher] | What `NULL = NULL` returns, and how to test for NULL |
| 12 | [Fresher] | SQL injection and how to prevent it |
| 13 | [Mid] | Why `NOT IN` can return no rows |
| 14 | [Mid] | The latest row per group |
| 17 | [Mid] | Join fan-out and how to avoid it |
| 18 | [Mid] | Composite index and column order |
| 23 | [Mid] | Isolation levels and anomalies |
| 31 | [Senior] | Diagnosing a query that became slow |
| 34 | [Senior] | A schema migration on a large live table without downtime |

### All questions

Grouped by level: [Fresher], [Mid] or [Senior]. Every question has a concise model answer in the subsection that follows.

### Fresher

1. [Fresher] What is the difference between `WHERE` and `HAVING`?
2. [Fresher] What is a primary key versus a foreign key?
3. [Fresher] Explain `INNER JOIN` versus `LEFT JOIN`.
4. [Fresher] What does `NULL = NULL` return, and how do you test for NULL?
5. [Fresher] What is the difference between `COUNT(*)` and `COUNT(column)`?
6. [Fresher] What is the difference between `UNION` and `UNION ALL`?
7. [Fresher] In what order does the database logically evaluate a `SELECT`?
8. [Fresher] Why should money not be stored as `float`?
9. [Fresher] What is an index and what does it cost?
10. [Fresher] What does `GROUP BY` do, and what is the rule for selected columns?
11. [Fresher] What is the difference between `DELETE`, `TRUNCATE` and `DROP`?
12. [Fresher] What is SQL injection and how do you prevent it?

### Mid

13. [Mid] Why does `NOT IN` sometimes return no rows, and what do you use instead?
14. [Mid] How do you get the latest row per group?
15. [Mid] Explain `rank`, `dense_rank` and `row_number`.
16. [Mid] What is a CTE and how does it differ from a subquery?
17. [Mid] What is join fan-out and how do you avoid it?
18. [Mid] What is a composite index and why does column order matter?
19. [Mid] Why might a query ignore an index?
20. [Mid] What is a covering index?
21. [Mid] How do you implement an upsert?
22. [Mid] What is ACID?
23. [Mid] What is a transaction isolation level, and what anomalies exist?
24. [Mid] OFFSET versus keyset pagination?
25. [Mid] What is the N+1 problem?
26. [Mid] How do you store timestamps correctly?
27. [Mid] How do you find the nth highest salary or order value?
28. [Mid] What is normalisation, and what do 1NF, 2NF and 3NF require?
29. [Mid] How would you pivot rows to columns?
30. [Mid] What does `EXPLAIN` show, and how does `ANALYZE` change it?

### Senior

31. [Senior] A query got slow after data growth. Walk through your diagnosis.
32. [Senior] Explain lost update and write skew, and how to prevent each.
33. [Senior] How does MVCC work and what are its operational costs?
34. [Senior] How do you run a schema migration on a large live table without downtime?
35. [Senior] Design a safe work queue on a relational table.
36. [Senior] Explain gaps and islands, and sessionisation, with approaches.
37. [Senior] When would you denormalise, and how do you keep it correct?
38. [Senior] A table has 10 indexes and writes are slow. How do you decide what to drop?
39. [Senior] Why can an `ALTER TABLE` cause an outage even if it is quick?
40. [Senior] How do you make a payment-recording endpoint idempotent?
41. [Senior] Compare hash, merge and nested loop joins and when each is chosen.
42. [Senior] How would you detect and fix a deadlock in production?
43. [Senior] How do you design and verify a money schema?
44. [Senior] What would you do about a bad row-count estimate in a plan?

### Model answers

**1.** `WHERE` filters rows before grouping and cannot use aggregates. `HAVING` filters groups after aggregation. Put conditions on raw columns in `WHERE`, since it reduces work earlier.

**2.** A primary key uniquely identifies a row in its table and cannot be NULL. A foreign key is a column referencing another table's key, and the database enforces that the referenced row exists.

**3.** `INNER JOIN` returns only rows with a match on both sides. `LEFT JOIN` returns every left row, with NULLs for right columns when no match exists. Filters on the right table belong in `ON`, or the left join degrades to an inner join.

**4.** It returns NULL (unknown), which `WHERE` treats as not true. Use `IS NULL` or `IS NOT NULL`, or `IS NOT DISTINCT FROM` for null-safe equality.

**5.** `COUNT(*)` counts rows. `COUNT(column)` counts rows where the column is not NULL. `COUNT(DISTINCT column)` counts distinct non-NULL values.

**6.** `UNION` removes duplicate rows (extra sort or hash cost). `UNION ALL` keeps all rows and is faster. Use `UNION ALL` unless duplicates must be removed.

**7.** `FROM` and joins, `WHERE`, `GROUP BY`, `HAVING`, `SELECT` (including window functions), `DISTINCT`, `ORDER BY`, `LIMIT`. This is why a `SELECT` alias is not visible in `WHERE`.

**8.** Binary floating point cannot represent most decimal fractions exactly, and errors accumulate. Use integer minor units (cents) or `NUMERIC`, and store the currency.

**9.** A sorted auxiliary structure (usually a B-tree) that finds rows without a full scan. It costs storage and slows writes because each write maintains it. Index for real query patterns.

**10.** It groups rows by the listed expressions so aggregates compute per group. Every selected column must be in `GROUP BY` or inside an aggregate (PostgreSQL also allows columns functionally dependent on a grouped primary key).

**11.** `DELETE` removes selected rows, is logged per row, fires row triggers, and can be filtered. `TRUNCATE` removes all rows quickly with no per-row work and is transactional in PostgreSQL. `DROP` removes the table definition too.

**12.** An attacker alters query structure through concatenated input. Prevent with parameterised queries so values never get parsed as SQL, an allow-list for identifiers and sort directions, and least-privilege database roles.

**13.** If the subquery returns any NULL, `x NOT IN (...)` evaluates to UNKNOWN for every row, so nothing matches. Use `NOT EXISTS`, or filter NULLs inside the subquery.

**14.** Use `row_number() OVER (PARTITION BY key ORDER BY ts DESC, id DESC)` and keep `rn = 1`. In PostgreSQL, `DISTINCT ON (key)` with a matching `ORDER BY` is shorter. A `LATERAL ... LIMIT 1` is efficient with an index.

**15.** All three number rows within a partition. `row_number` gives unique numbers (arbitrary among ties). `rank` gives ties the same number and leaves gaps. `dense_rank` gives ties the same number with no gaps.

**16.** A CTE names an intermediate result for readability and reuse within one statement, and enables recursion. A subquery is inline. In PostgreSQL 12 and later a simple non-recursive CTE referenced once is inlined like a subquery, otherwise it is materialised.

**17.** Joining a parent to a one-to-many child repeats parent columns once per child, inflating sums and counts. Avoid by aggregating the child before the join, using `EXISTS` for existence checks, and checking row counts before and after joins.

**18.** A multi-column index sorted by the first column, then the second within it. It serves queries on a leftmost prefix. Put equality columns first, then the range or sort column. An index on `(a, b)` does not efficiently serve `WHERE b = ?` alone.

**19.** A function or arithmetic on the column, a type mismatch or implicit cast, a leading wildcard `LIKE`, low selectivity, a wrong column order, stale statistics, or a tiny table. Fix with sargable predicates, expression indexes, casts, `ANALYZE`, or accept the sequential scan when it is correct.

**20.** An index containing all columns a query needs, so it can use an index-only scan without visiting the table. In PostgreSQL, `INCLUDE` adds payload columns. It depends on an up-to-date visibility map (`VACUUM`).

**21.** In PostgreSQL, `INSERT ... ON CONFLICT (cols) DO UPDATE SET ...` using `EXCLUDED` for the new values, or `DO NOTHING`. It is atomic and race-free and needs a unique index on the conflict target. MySQL: `ON DUPLICATE KEY UPDATE`. Standard: `MERGE`.

**22.** Atomicity (all or nothing), Consistency (constraints hold across commits), Isolation (concurrent transactions do not interfere beyond the chosen level), Durability (committed data survives a crash).

**23.** The level sets which concurrency anomalies are allowed: dirty read, non-repeatable read, phantom read, and, beyond the standard's list, lost update and write skew. PostgreSQL's default is Read Committed. Repeatable Read gives a transaction-wide snapshot. Serializable also prevents write skew, at the cost of retries on `40001`.

**24.** `OFFSET` reads and discards rows, so deep pages are slow, and results shift when data changes. Keyset pagination filters with `WHERE (sort_key, id) < (last_key, last_id)` and `LIMIT`, using an index, with constant cost and stable pages, but no random page jump.

**25.** One query loads N parents, then one query per parent loads children, giving 1 + N round trips. Fix with a join, a batched query (`WHERE parent_id = ANY($1)`), or eager loading, and detect it by query counts per request.

**26.** Store instants as `timestamptz` (UTC internally), convert to local zones only at display, and store business-local times as local date or time plus an IANA zone name. Avoid zone-less `timestamp` for instants. Group by day in the intended zone explicitly.

**27.** `dense_rank() OVER (ORDER BY x DESC)` and filter on the rank, or `SELECT DISTINCT x ... ORDER BY x DESC OFFSET n-1 LIMIT 1`. Decide how ties and "fewer than n values" behave.

**28.** Normalisation stores each fact once. 1NF: atomic values, no repeating groups. 2NF: no partial dependence on part of a composite key. 3NF: no dependence of non-key columns on other non-key columns, only on the key.

**29.** Conditional aggregation: `count(*) FILTER (WHERE status = 'paid')` (or `SUM(CASE ...)`) per output column, grouped by the row key. Dynamic columns require generating the SQL or pivoting in application code.

**30.** `EXPLAIN` shows the chosen plan with estimated costs and rows. `EXPLAIN ANALYZE` executes the query and adds actual times, row counts and loops. `BUFFERS` adds cache and disk page counts. Compare estimates with actuals to find the problem node.

**31.** Capture the actual slow query (`pg_stat_statements`). Run `EXPLAIN (ANALYZE, BUFFERS)` on realistic parameters. Find the node with the most time. Compare estimated and actual rows, and check `Rows Removed by Filter`, spills and loops. Then fix it with an index, a sargable rewrite, fresh statistics, less data read, or a changed access pattern. Verify with the new plan and under load, and check the cost to writes and other queries.

**32.** Lost update: two transactions read then overwrite the same row. Prevent with atomic `UPDATE x = x + ...`, `SELECT FOR UPDATE`, a version column, or Repeatable Read with retry. Write skew: two transactions read overlapping data and write different rows, breaking an invariant. Prevent with Serializable plus retry, locking a shared parent row, or a constraint.

**33.** Rows have multiple versions identified by transaction ids. A transaction reads a snapshot, so readers do not block writers. Updates create new versions, and `VACUUM` reclaims dead ones. Costs: table and index bloat, vacuum load, and long-running or idle-in-transaction sessions that prevent cleanup and can cause wraparound risk.

**34.** Use expand and contract. Add nullable structures compatibly, dual-write, backfill in small committed batches, validate constraints with `NOT VALID` then `VALIDATE`, build indexes `CONCURRENTLY`, switch reads, then drop old structures later. Set `lock_timeout`, test on production-sized data, have a rollback plan and backups.

**35.** A table with a status column and a claim step using `SELECT ... FOR UPDATE SKIP LOCKED LIMIT n` inside an `UPDATE`. Add `claimed_at` and attempts, a reaper that requeues stale claims, idempotent job handlers (delivery is at least once), a partial index on pending rows, and a dead-letter state after repeated failure.

**36.** Gaps and islands: `value - row_number()` is constant within a run of consecutive values, so group by it. Alternatively compare with `lag()` to flag a break and take a cumulative sum. Sessionisation: flag rows where the time since the previous event exceeds the threshold and cumulatively sum the flags to number sessions.

**37.** When measured latency or load on a hot read path cannot be met by indexes and good queries. Keep a normalised source of truth, derive the copy via a materialised view, trigger or job, add reconciliation checks, and document ownership, because every copy can drift.

**38.** Use `pg_stat_user_indexes` to find indexes with no scans over a representative period, check duplicates and prefix-redundant indexes, confirm none enforces a constraint or supports a foreign key, and check the replica as well as the primary. Drop one at a time with `DROP INDEX CONCURRENTLY` and keep the definition to recreate it.

**39.** `ALTER TABLE` needs a strong lock. If a long transaction holds a conflicting lock, the `ALTER` queues, and all later queries on the table queue behind it. Set `lock_timeout`, retry, and avoid running migrations beside long transactions.

**40.** Require a client-supplied idempotency key or an external provider reference stored under a unique constraint, and insert with `ON CONFLICT DO NOTHING` (or return the stored result). Apply the state change and the record in one transaction, so a retry either finds the existing record or completes once.

**41.** Nested loop: for each outer row, probe the inner side; best with few outer rows and an index on the inner. Hash join: hash the smaller input and probe with the other; good for large unsorted equality joins, needs memory. Merge join: both inputs sorted on the key; good when they are already ordered by an index or the sort is cheap. The planner chooses by estimated cost, so bad estimates produce bad choices.

**42.** The database logs the deadlock with the statements and the aborted transaction (`40P01`). Find the two code paths and fix the lock order (consistent ordering by id), shorten transactions, lock coarser where needed, and add retries. Use `pg_locks` and `log_lock_waits` to see waits.

**43.** Integer minor units or `NUMERIC` with an explicit currency column, `CHECK` constraints, no cross-currency sums, an append-only ledger of entries with a balance derived or maintained under constraint, idempotency keys on movements, and reconciliation queries (ledger sum versus provider records) run regularly. Test rounding rules with explicit cases.

**44.** Run `ANALYZE`, raise the statistics target for the column, add extended statistics (`CREATE STATISTICS`) for correlated columns, remove expressions that hide the column from statistics, or rewrite the query (for example, a CTE or `LATERAL`) to constrain the plan. Re-check the plan, and avoid planner-forcing hacks unless you have no other path.

---

## 22. References

Primary sources, preferred over blog posts:

- PostgreSQL Documentation (select your version): https://www.postgresql.org/docs/
  - SQL Language, queries, window functions, `SELECT` reference: https://www.postgresql.org/docs/current/sql.html
  - Data types and `NUMERIC`: https://www.postgresql.org/docs/current/datatype.html
  - Date/time types and time zones: https://www.postgresql.org/docs/current/datatype-datetime.html
  - Indexes: https://www.postgresql.org/docs/current/indexes.html
  - Using `EXPLAIN`: https://www.postgresql.org/docs/current/using-explain.html
  - Transaction isolation and concurrency control: https://www.postgresql.org/docs/current/mvcc.html
  - Explicit locking (table and row locks, advisory locks): https://www.postgresql.org/docs/current/explicit-locking.html
  - `SELECT` locking clause (`FOR UPDATE`, `NOWAIT`, `SKIP LOCKED`): https://www.postgresql.org/docs/current/sql-select.html
  - `INSERT ... ON CONFLICT`: https://www.postgresql.org/docs/current/sql-insert.html
  - `pg_stat_statements`: https://www.postgresql.org/docs/current/pgstatstatements.html
  - `ALTER TABLE` (`ADD COLUMN` defaults, `SET NOT NULL`, `NOT VALID`): https://www.postgresql.org/docs/current/sql-altertable.html
  - `WITH` queries (CTEs, `MATERIALIZED`, recursion, `CYCLE`): https://www.postgresql.org/docs/current/queries-with.html
  - `psql`: https://www.postgresql.org/docs/current/app-psql.html
- Markus Winand, *Use The Index, Luke* (indexing and SQL performance, multiple databases): https://use-the-index-luke.com/
- Markus Winand, *Modern SQL* (standard SQL features and database support matrix): https://modern-sql.com/
- ISO/IEC 9075-1:2023 (the SQL standard, Part 1: Framework; the full standard is sold by ISO): https://www.iso.org/standard/76583.html (the ISO site blocked automated fetching, but a search confirmed the title)
- MySQL 8.0 Reference Manual: https://dev.mysql.com/doc/refman/8.0/en/ (`INTERSECT` and `EXCEPT` pages: `intersect.html`, `except.html` in the same folder)
- Microsoft SQL Server (Transact-SQL) documentation: https://learn.microsoft.com/en-us/sql/t-sql/
- sqlfluff documentation: https://docs.sqlfluff.com/
- pgAdmin documentation: https://www.pgadmin.org/docs/
- DBeaver documentation: https://dbeaver.com/docs/dbeaver/
- Martin Kleppmann, *Designing Data-Intensive Applications* (transactions, isolation, anomalies; book)
- Hellerstein, Stonebraker and Hamilton, *Architecture of a Database System* (paper)
- Berenson et al., *A Critique of ANSI SQL Isolation Levels* (paper; origin of snapshot isolation and write skew discussion)
- C. J. Date, *SQL and Relational Theory* (book)

URL status (2026-11-06): all PostgreSQL and MySQL links above loaded. Not verified: the Use The Index Luke, Modern SQL, Microsoft, sqlfluff, pgAdmin and DBeaver links (outside the official-docs check). Pages move, so verify a URL before depending on it.

---

## 23. Glossary

**In plain words:** These are the hardest terms in this guide, with simple meanings.

| Term | Plain meaning |
|---|---|
| ACID | Four promises of a transaction: all-or-nothing, rules kept, no interference, and data that survives a crash |
| Anti-join | A query for rows that have no match in another table |
| Atomic | Done completely or not done at all |
| B-tree | The default index structure: a sorted tree used to find values fast |
| CTE | Common table expression: a named step in a query, written with `WITH` |
| Cardinality | The number of distinct values in a column, or of rows in a result |
| Correlated subquery | A subquery that uses a value from the outer row |
| Covering index | An index that holds every column a query needs |
| Deadlock | Two transactions each wait for a lock the other holds, so neither can continue |
| Fan-out | Rows repeated by a join to a "many" side, which can inflate totals |
| Foreign key | A column that must match a key in another table |
| Idempotent | Safe to run again: repeating it gives the same result as running it once |
| Isolation level | A setting that controls what a transaction can see of other transactions' work |
| Keyset pagination | Paging by "rows after the last key seen" instead of skipping rows with `OFFSET` |
| Lateral | A `FROM` item that can use columns from items listed before it |
| Materialised | Computed once and stored, instead of recomputed each time |
| MVCC | Multi-version concurrency control: the database keeps several versions of a row, so readers do not block writers |
| Normalisation | Storing each fact in one place only |
| Optimizer | The part of the database that chooses how to run a query |
| Phantom read | A repeated range query returns new or missing rows because another transaction committed |
| Query plan | The steps the optimizer chose to run a query |
| Sargable | Written so the database can use an index for the condition |
| Selectivity | The share of rows a condition keeps. Few rows kept means high selectivity |
| Semi-join | A query that checks that a match exists, without returning columns from the other side |
| Serializable | The strictest isolation level: the result is as if transactions ran one at a time |
| Surrogate key | A generated identifier with no business meaning |
| Upsert | Insert a row, or update it if it already exists |
| VACUUM | A PostgreSQL job that frees space held by dead row versions |
| Window function | A function that computes over related rows without merging them |
| Write skew | Two transactions read the same data, write different rows, and together break a rule |
