# MongoDB for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to MongoDB for people who build and operate backend services. It teaches how the database behaves and why. You will learn to model data, diagnose a slow query or a failing cluster, and defend your design in a review, not just recall syntax.

**Audience:** backend developers, most likely Node.js, who have used a database and want production-level MongoDB skill.
**Prerequisites:** basic JSON and JavaScript, and a working idea of what a relational database does (tables, joins, indexes, transactions). The Linux guide in this repository helps for the operations chapters.
**Scope:** MongoDB server (self-managed and Atlas), `mongosh`, the Node.js driver and Mongoose. Other drivers behave similarly but are out of scope. Enterprise-only features are mentioned only where they change a decision.
**Note on accuracy:** the concepts here are long-stable. Defaults, limits and feature availability change between server versions. Where a detail is version-dependent this guide says so and tells you to confirm in the MongoDB manual for the version you run. Run `db.version()` first, and read the manual page that matches it.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 22 lists hands-on labs. Chapter 23 collects interview questions with model answers. Chapter 24 lists primary references. Chapter 25 is a glossary of hard terms.

Each chapter starts with an "In plain words" summary in simple English.

**What "good enough" means for a backend role:** you can design a schema from access patterns, create the right compound index, read an `explain()` plan and fix a collection scan, explain write concern and read concern in terms of data loss, and describe what happens when a primary fails. Chapters 3, 7, 8, 11 and 12 matter most.

## Contents

1. The mental model and BSON types
2. When MongoDB is, and is not, the right choice
3. Data modelling: embed versus reference
4. Schema design patterns
5. CRUD and update operators
6. Query operators and cursors
7. Indexes
8. Reading explain output
9. The aggregation pipeline
10. Transactions
11. Replica sets
12. Consistency and durability: read and write concern
13. Sharding
14. Change streams
15. Atlas operational features
16. Mongoose
17. The Node driver and connection pooling
18. Security
19. Backups and restore
20. Monitoring and diagnostics
21. Common production mistakes
22. Practice labs
23. Interview questions
24. References
25. Glossary

---

## 1. The mental model and BSON types

**In plain words.** MongoDB keeps data as documents, which look like JSON objects. Each value has a type, such as text, number or date, and the type changes how queries match. Learn the types first, because many silent query bugs come from a wrong type.

### Understand

**A document database.** MongoDB stores **documents** (ordered sets of field and value pairs) in **collections**, inside **databases**. A document is the unit of storage, of reading, and, importantly, of **atomicity** (a change applies fully or not at all). Documents in one collection need not share a shape, though in practice you keep them consistent and enforce it with validation (chapter 16 and chapter 3).

**BSON.** Documents are stored and transmitted as **BSON**, a binary superset of JSON. It adds types JSON lacks, and it is typed, so `"5"` and `5` are different values and sort and compare differently. This is the source of many silent query bugs: a query for the number `5` does not match the string `"5"`.

Types you must know:

| Type | Notes |
|---|---|
| String | UTF-8. |
| Int32, Int64 (Long), Double | JavaScript numbers become Double by default in many drivers. Large integer IDs need Int64 or a string. |
| Decimal128 | 128-bit decimal floating point. Use for money and exact decimals. Doubles cannot represent 0.1 exactly. |
| Boolean, Null | `null` and a missing field are different, but `{ a: null }` matches both. |
| Date | A UTC instant stored as milliseconds since the Unix epoch. Store dates as Date, never as strings, or range queries, TTL indexes and date aggregation operators will not work. |
| ObjectId | 12-byte identifier: 4-byte timestamp (seconds), 5-byte random value, 3-byte incrementing counter. Roughly increasing over time, generated client-side, no coordination needed. |
| Array | Ordered list, can hold mixed types, can be indexed (multikey, chapter 7). |
| Embedded document | A document as a field value. Fields are addressed with dot notation (`address.city`). |
| BinData, UUID | Binary data. UUIDs stored as binary use less space than strings but need care with driver representation. |
| Timestamp | Internal type used by the replication machinery. Do not use it as an application date. |

**The `_id` field.** Every document has a unique `_id`, indexed automatically. If you do not supply one the driver generates an ObjectId. You can use any immutable unique value (a natural key, a UUID). `_id` cannot be changed after insert. Because an ObjectId embeds its creation time, `_id` order approximates insertion order, which is convenient for sorting and pagination and dangerous as a shard key (chapter 13).

**Storage engine.** The default engine is **WiredTiger**. It provides document-level concurrency control, compression, an internal cache and a write-ahead journal (a log written before the data, used for crash recovery). You rarely tune it, but you must know that the working set (hot data plus the indexes in use) should fit in RAM for good latency, and that the cache is separate from the operating system's file cache.

**Document size limit.** A single BSON document is limited to 16 MiB. This is a design constraint, not a tuning knob: it is the reason unbounded arrays are a defect (chapter 3). Nesting depth is also limited. Confirm exact limits in the manual page "MongoDB Limits and Thresholds".

**Wire path.** The application uses a driver, which keeps connections to the servers it discovers from your connection string. On a replica set the driver learns the topology and routes operations to the primary or to secondaries according to read preference. On a sharded cluster it talks to `mongos` routers instead (chapter 13).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Shell | `mongosh`: `show dbs`, `use`, `db.coll.find()`, `db.version()`, `help()` | Work fluently in the shell, write multi-line scripts |
| GUI | MongoDB Compass | Browse documents, view schema distribution, run explain visually |
| Types | `typeof`, `$type`, `Object.prototype.toString`, `ObjectId().getTimestamp()` | Identify why a query misses documents of a different type |
| Decimal | `Decimal128.fromString()` or driver equivalent | Store money without float error |

### Common mistakes

- Storing dates as ISO strings, then discovering TTL and date operators do not apply.
- Storing money as Double and accumulating rounding error.
- Querying `{ userId: "123" }` when stored values are numbers.
- Treating `null` and "field missing" as the same thing in a query without checking.
- Using the application Timestamp type as a date.

### Self-check

- Why does a query for the number 5 not return a document with the string "5"?
- What three pieces make up an ObjectId, and what does that imply about ordering?
- Why is Decimal128 preferred for currency?
- What does `{ field: null }` match?

---

## 2. When MongoDB is, and is not, the right choice

**In plain words.** MongoDB fits well when you read and write a whole object at once and the shape of the data changes. A relational database fits better when many tables must be joined in flexible ways. Choose from your real queries, not from habit.

### Understand

Choose a database from the data and the access patterns, not from fashion.

**MongoDB fits well when:**
- Data is naturally aggregate-shaped: an order with its line items, a user profile with preferences, a product with variable attributes, a content item with metadata. You read and write the aggregate as a whole.
- Schemas vary between records or change often (catalogues, event payloads, configuration).
- You need horizontal scale for large collections through sharding.
- Documents map directly to your application objects and you want few transformation layers.
- High write volume of semi-structured records (logs, events, telemetry), especially with time series collections or the bucket pattern.

**A relational database fits better when:**
- Data is highly interconnected and queries routinely join many entities in unpredictable ways.
- You need strong declarative integrity across entities: foreign keys, `CHECK` constraints, cascades.
- Reporting and ad hoc analytics need flexible joins and mature SQL tooling.
- Workloads are dominated by multi-row, multi-table transactions, such as a double-entry ledger.

**What the trade really is.** In MongoDB you pay at design time: you must decide the access patterns and shape the data to serve them, and you duplicate some data deliberately. In exchange, common reads touch one document and one index lookup. In a relational system you normalise first and pay at query time through joins. Neither is faster in general. MongoDB being "schemaless" is misleading: the schema exists, in your application code, and without discipline it decays.

**Misconceptions to correct.**
- "MongoDB has no transactions." It has multi-document ACID transactions (chapter 10). They cost more than single-document operations and good modelling avoids most of the need.
- "MongoDB is not consistent." Single-document operations are atomic and, with the right read and write concern, strongly consistent (chapter 12). Defaults and your settings decide the guarantee.
- "MongoDB cannot join." `$lookup` exists, but it is not free and not the way you should shape routine queries (chapter 9).
- "Schemaless means no validation." You can enforce structure with `$jsonSchema` validation.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Access pattern analysis | List the top reads and writes with frequency and latency target | Produce a one-page access-pattern table before modelling |
| Comparison | Write the same feature as a relational and as a document model | Explain which is simpler and why |

### Common mistakes

- Choosing MongoDB to "avoid schema design", then building a relational model with references everywhere and manual joins in application code.
- Choosing it for a transaction-heavy, relational domain and fighting it.
- Skipping the access-pattern analysis.

### Self-check

- Give two workloads where you would pick MongoDB and two where you would not, with the reason.
- What does "schema-less" really move, and to where?

---

## 3. Data modelling: embed versus reference

**In plain words.** Data modelling means deciding how to arrange your data. Put data in one document when you read it together. Keep it in separate documents when it is large or grows without limit.

### Understand

**The central principle.** Data that is accessed together should be stored together. A single-document read is one index lookup and one disk location. Document-level atomicity means an update to one document, including its embedded data, is all-or-nothing without needing a transaction.

**Embed when:**
- The child data is read with the parent nearly every time.
- The child belongs to one parent and has no life of its own.
- The child set is bounded and small.
- Child and parent change together and you want atomic updates.

**Reference (store the other document's `_id`) when:**
- The child is large, or accessed independently.
- The relationship is many-to-many.
- The child set is unbounded or grows without limit.
- The child changes frequently and is duplicated in many parents.

**Relationship cardinality.** A useful rule of thumb:

| Cardinality | Typical model |
|---|---|
| One-to-few (a user's addresses) | Embed an array in the parent. |
| One-to-many (a product's reviews, hundreds) | Embed if bounded and read together. Otherwise store parent `_id` in each child, or an array of child ids in the parent. |
| One-to-squillions (log events per host, readings per sensor) | Never embed. Store the parent's `_id` in each child document and index it. Consider bucketing (chapter 4). |
| Many-to-many (students and courses) | Arrays of references on one or both sides, or a join collection when the relationship has its own attributes. |

**Unbounded arrays.** An array that grows without limit is the most common modelling failure. Consequences: the document approaches 16 MiB and writes start failing, updates rewrite larger and larger documents, multikey index entries multiply, and reads transfer the whole array. If you cannot state the maximum size of an array, reference instead of embedding, or cap it (the subset pattern, chapter 4).

**Duplication is acceptable when it is controlled.** Copying a user's display name into each comment avoids a lookup on every read. The price is updating copies when the source changes. Duplicate data that is read often and changes rarely.

**Referential integrity is your job.** MongoDB has no foreign keys. Deleting a parent can orphan children unless the application, a transaction or a cleanup job handles it.

**Schema validation.** Add a `$jsonSchema` validator to critical collections to reject malformed writes at the database:

```js
db.createCollection("orders", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["customerId", "status", "total"],
    properties: {
      status: { enum: ["new", "paid", "shipped", "cancelled"] },
      total: { bsonType: "decimal" }
    }
  }},
  validationLevel: "strict",
  validationAction: "error"
})
```

`validationLevel` controls which writes are checked, and `validationAction` is `error` (reject) or `warn` (log). Adding validation to an existing collection does not fix existing documents.

**Schema evolution.** Documents of old and new shape coexist during a migration. A `schemaVersion` field lets code handle both and migrate lazily or in batches. Plan this from the start.

**Design method.**
1. List entities and their relationships.
2. List the queries and writes, with frequency and latency needs.
3. Embed what is read together and bounded. Reference the rest.
4. Check every document for growth bounds.
5. Design indexes for the queries (chapter 7).
6. Test with realistic data volumes, not ten documents.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Modelling | Access-pattern table, entity sketch, sample documents | Justify each embed or reference by a read or write pattern |
| Validation | `$jsonSchema`, `collMod`, `validationLevel`, `validationAction` | Add validation to a live collection safely |
| Size check | `Object.bsonsize(doc)`, Compass schema tab | Measure real document sizes and array lengths |
| Analysis | `$bsonSize` in aggregation, `$size` | Find the largest documents and arrays in a collection |

### Common mistakes

- Embedding an array that grows forever (comments, events, audit trail, followers).
- Normalising everything into references and joining in application code, producing N+1 round trips.
- Embedding data that is queried on its own in huge numbers, then needing to query or update it independently.
- No validation, so malformed documents appear and every reader needs defensive code.
- Designing the schema before knowing the queries.

### Self-check

- A blog post has comments. When do you embed them, and when do you reference them?
- What is wrong with a `followers` array inside a user document for a social product?
- How do you change the schema of a collection with 200 million documents while the app is running?
- Which operations on an embedded structure are atomic without a transaction?

---

## 4. Schema design patterns

**In plain words.** A schema pattern is a proven answer to a common design problem. Each pattern gives faster reads in exchange for harder writes or some repeated data. Use a pattern only when you have the problem it solves.

### Understand

Patterns are named solutions to recurring modelling problems. Learn the problem each one solves, not just the name.

**Subset pattern.** Problem: a document has a large array of which only the recent or top items are shown (a product with thousands of reviews, a page showing the latest ten). Solution: embed only the subset needed for the common read, keep the rest in a separate collection. Benefit: smaller documents, smaller working set. Cost: some duplication, and keeping the subset in sync.

**Extended reference pattern.** Problem: a frequent read needs a few fields from a referenced document, forcing a `$lookup` or second query each time. Solution: copy those few fields (for example customer name and city) into the referencing document alongside the id. Benefit: avoids joins. Cost: copies can go stale, so copy only fields that rarely change or tolerate eventual update.

**Computed pattern.** Problem: an expensive calculation (total, count, average) is repeated on every read. Solution: compute once on write or by a scheduled job, and store the result. Example: maintain `commentCount` with `$inc` when a comment is added. Benefit: cheap reads. Cost: write complexity and the risk of drift, so provide a reconciliation job.

**Bucket pattern.** Problem: very high volume of small time-ordered records (sensor readings, metrics), where one document per reading makes the collection and indexes huge. Solution: group a time window or fixed count of readings into one document with an array of measurements, plus summary fields. Benefit: fewer documents, smaller indexes, efficient range reads. Cost: more complex writes and queries, and the bucket size must be bounded (for example per hour or at most N entries). For time series data, check whether the **time series collections** feature in your server version fits, since it implements bucketing for you (available from MongoDB 5.0). Confirm limits in the manual.

```js
// one document per sensor per hour, with a bounded array
db.readings.updateOne(
  { sensorId: 7, hour: ISODate("2025-01-01T10:00:00Z"), count: { $lt: 200 } },
  { $push: { samples: { t: new Date(), v: 21.4 } }, $inc: { count: 1 } },
  { upsert: true }
)
```

**Outlier pattern.** Problem: a few documents have far more related data than the rest (a celebrity with millions of followers) and would break a design that suits the typical case. Solution: handle the typical case with the simple model (embed), and flag outliers (`hasOverflow: true`) with overflow data in additional documents. Benefit: the common path stays simple. Cost: application code must handle the flag.

**Attribute pattern.** Problem: many similar but varying fields to index and query (product specs). Solution: store them as an array of `{ k, v }` pairs and index `{ "specs.k": 1, "specs.v": 1 }`. This avoids dozens of sparse indexes. A wildcard index (chapter 7) is an alternative for some cases.

**Schema versioning pattern.** Store `schemaVersion` on each document so code can read multiple shapes during migration.

**Approximation pattern.** For counters where exactness is not required (page views), update the stored value only every N events to reduce write load.

**Choosing.** Patterns trade write complexity and duplication for read speed. Apply one only when a measured or clearly predicted read, size or write problem justifies it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Counters and computed values | `$inc`, `$set`, scheduled reconciliation | Maintain a computed field and recompute it |
| Bounded arrays | `$push` with `$each` and `$slice`, conditional update filter | Cap an array at a fixed length |
| Bucketing | upsert with a bound condition | Write the bucket upsert above from memory |
| Time series | time series collections (confirm version) | Decide between manual bucketing and the built-in collection |

### Common mistakes

- Applying patterns by name without a problem to solve.
- Computed fields with no way to rebuild them when they drift.
- Buckets with no size bound, recreating the unbounded array problem.
- Extended reference copies of fields that change often.

### Self-check

- Which pattern would you use for a product page that shows the latest 10 of 50,000 reviews, and what does it cost you?
- How do you keep a stored `commentCount` correct under concurrent writes?
- Why does bucketing shrink indexes?

---

## 5. CRUD and update operators

**In plain words.** CRUD means create, read, update and delete. Update operators change one field without rewriting the whole document. Use an operator such as `$inc` instead of reading a value, changing it in your code and writing it back.

### Understand

**Create.** `insertOne` and `insertMany`. `insertMany` is **ordered** by default: it stops at the first error. Pass `ordered: false` to continue and collect errors, which is usually right for bulk loads with possible duplicates. Duplicate `_id` or unique-index violations raise error code 11000.

**Read.** `find(filter, projection)` returns a cursor. `findOne` returns one document. Use projection to return only needed fields, which cuts network and memory cost and can enable covered queries (chapter 8).

**Update.** `updateOne`, `updateMany`, `replaceOne`, `findOneAndUpdate`. An update without update operators in `replaceOne` replaces the whole document. An update document with operators changes only the named fields.

**Delete.** `deleteOne`, `deleteMany`. `deleteMany({})` removes all documents but keeps the collection and indexes. `drop()` removes the collection.

**Upsert.** `upsert: true` inserts a document if none matches. Combine with a unique index to make it race-safe (correct when two clients act at the same time): two concurrent upserts for the same key can otherwise both insert, and the unique index makes the loser fail with a duplicate key error that the application should retry.

**Atomic read-modify-write.** `findOneAndUpdate` with `returnDocument: "after"` returns the updated document atomically. This is the right tool for claiming a job from a queue or incrementing and reading a sequence. Never read, modify in application memory, and write back for state that concurrent clients change.

**Update operators.**

| Operator | Effect |
|---|---|
| `$set`, `$unset` | Set or remove fields. |
| `$inc`, `$mul` | Increment or multiply a number atomically. |
| `$min`, `$max` | Set only if the new value is lower or higher. |
| `$rename` | Rename a field. |
| `$currentDate` | Set to the server date. |
| `$setOnInsert` | Apply only when an upsert inserts. |
| `$push`, `$addToSet`, `$pull`, `$pop`, `$pullAll` | Array add and remove. `$addToSet` avoids duplicates. |
| `$each`, `$slice`, `$sort`, `$position` | Modifiers for `$push`. |
| `$` , `$[]`, `$[<id>]` with `arrayFilters` | Positional updates in arrays: first match, all elements, filtered elements. |

```js
// capped recent-activity array: keep the newest 50 entries
db.users.updateOne(
  { _id: id },
  { $push: { recent: { $each: [entry], $slice: -50 } } }
)

// update only matching elements of an array
db.orders.updateOne(
  { _id: id },
  { $set: { "items.$[i].shipped": true } },
  { arrayFilters: [{ "i.sku": "A-100" }] }
)
```

**Aggregation pipeline updates.** An update can take an array of pipeline stages (`$set`, `$unset`, `$replaceWith`), letting you compute a field from other fields in the same document. Confirm the supported version in the manual.

**Bulk operations.** `bulkWrite` sends many mixed operations in few round trips. It is ordered by default, with an `ordered: false` option.

**Atomicity.** Each single-document write is atomic, including all its embedded changes. A `updateMany` is not atomic across documents: it can partly apply if it fails midway.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Writes | `insertMany` (ordered or not), `bulkWrite`, upsert, `findOneAndUpdate` | Pick the correct call for each case |
| Operators | `$set`, `$inc`, `$push` with `$each` and `$slice`, `$addToSet`, `$pull`, `arrayFilters` | Update nested and array data without reading first |
| Safety | Dry-run the filter with `find` and `countDocuments` before `updateMany` or `deleteMany` | Never run a broad write unverified |
| Error handling | Duplicate key code 11000, `writeErrors`, `WriteConcernError` | Interpret and retry correctly |

### Common mistakes

- Read, modify, write in application code for counters and state, losing concurrent updates.
- `replaceOne` or an update without operators wiping fields unintentionally.
- `updateMany` or `deleteMany` with a wrong or empty filter.
- Using `$push` with no bound, growing an array forever.
- Upsert on a field with no unique index, creating duplicates under concurrency.
- Ignoring the result: `modifiedCount` and `matchedCount` differ, and a 0 match is not an error.

### Self-check

- What is the difference between `matchedCount` and `modifiedCount`?
- How do you atomically take the next job from a queue collection?
- Why can two concurrent upserts create duplicates, and how do you prevent it?
- What does `$[<id>]` with `arrayFilters` do?

---

## 6. Query operators and cursors

**In plain words.** A query filter is a document that describes which records you want. Some filters are slow, and some array filters give surprising results. For large results, move through pages with a range condition instead of `skip`.

### Understand

**Filter documents.** A filter is a document of conditions, implicitly ANDed. Equality is `{ status: "paid" }`. Operators are prefixed with `$`.

| Group | Operators |
|---|---|
| Comparison | `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin` |
| Logical | `$and`, `$or`, `$nor`, `$not` |
| Element | `$exists`, `$type` |
| Evaluation | `$expr`, `$regex`, `$mod`, `$text`, `$where` (JavaScript, avoid) |
| Array | `$all`, `$elemMatch`, `$size` |

**Performance character of operators.**
- Negations (`$ne`, `$nin`, `$not`) generally match most of the collection and use indexes poorly.
- `$regex` can use an index efficiently only for case-sensitive prefix patterns anchored with `^`. A leading wildcard or case-insensitive regex scans the whole index or collection. For real text search use a text index or Atlas Search.
- `$where` and JavaScript expressions are slow and a security risk. Prefer `$expr` or normal operators.
- `$or` can use one index per clause. Each clause needs an index.

**Arrays.** A condition on an array field matches if any element matches. With several conditions on the same array of subdocuments, the conditions may be satisfied by different elements. Use `$elemMatch` when all conditions must hold on one element:

```js
// matches if some single element has sku "A" AND qty > 5
db.orders.find({ items: { $elemMatch: { sku: "A", qty: { $gt: 5 } } } })
```

Without `$elemMatch`, `{ "items.sku": "A", "items.qty": { $gt: 5 } }` can match when one element has the sku and a different one has the quantity.

**Null and missing.** `{ x: null }` matches documents where `x` is null or absent. Use `{ x: { $exists: false } }` or `{ x: { $type: "null" } }` to separate them.

**Sort, limit, skip.** Order of application is sort, then skip, then limit regardless of how you write the call. A sort without an index supporting it happens in memory and is bounded by a memory limit (100 MB per sort stage in the manual; confirm for your version). Before MongoDB 6.0 a large sort fails unless disk use is allowed. From 6.0 the server may spill to disk by default.

**Pagination.** `skip(n)` still walks and discards n entries, so deep pages get slow. Prefer **range (keyset) pagination**: sort by an indexed unique-ordered field and ask for the next page with a condition on the last seen value.

```js
db.events.find({ _id: { $gt: lastSeenId } }).sort({ _id: 1 }).limit(50)
```

For a non-unique sort field, include a tiebreaker (`_id`) in both the sort and the condition.

**Cursors.** `find` returns a cursor that fetches in batches. A cursor that stays idle is closed by the server after a timeout (10 minutes by default, `cursorTimeoutMillis`). Do not hold cursors open across long processing, and never load an entire large result into memory with `toArray()`. Iterate or stream.

**Counting.** `countDocuments(filter)` does an accurate count by running an aggregation. `estimatedDocumentCount()` uses metadata and is fast but ignores filters.

**Collation.** A collation defines locale-aware comparison and case or accent insensitivity. Queries and indexes must use matching collations to use the index.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Filters | Comparison, logical, element, array operators | Write any filter without notes |
| Arrays | `$elemMatch`, `$all`, `$size`, dot notation | Explain the multi-condition array trap |
| Projection | inclusion, exclusion, `$slice`, `$elemMatch` projection | Return only the fields needed |
| Pagination | keyset pagination with tiebreaker | Replace `skip` for deep paging |
| Streaming | cursor iteration, `for await` in Node | Process large results in constant memory |

### Common mistakes

- Deep `skip` pagination on large collections.
- Case-insensitive regex for lookups at scale (`/^john/i`), which cannot use an ordinary index efficiently.
- `$ne` filters expecting index use.
- Missing `$elemMatch` on arrays of subdocuments.
- Calling `toArray()` on an unbounded query.
- Comparing across types (string versus number) and getting no results.

### Self-check

- Why can `{ "items.sku": "A", "items.qty": { $gt: 5 } }` return a wrong match?
- How do you paginate 10 million documents efficiently?
- Why is a case-insensitive regex slow, and what are the alternatives?

---

## 7. Indexes

**In plain words.** An index is a sorted structure that lets the database find records without reading everything. It makes reads faster and writes slower, so keep only the indexes you need. The order of fields in a compound index (an index on several fields) decides which queries it can serve.

### Understand

**Why.** Without an index a query performs a **collection scan** (`COLLSCAN`): it reads every document. An index is a sorted B-tree structure on one or more fields that lets the server find matching documents and return them in order without scanning. Indexes speed reads and cost writes (every insert, delete and indexed-field update maintains each index), RAM and disk. The aim is the smallest set of indexes that serves your real queries.

**Index types.**

| Type | Use |
|---|---|
| Single field | Equality, range and sort on one field. Direction matters little for a single field. |
| Compound | Several fields in a defined order. The workhorse. |
| Multikey | Created automatically when an indexed field holds an array. One entry per element. |
| Text | Word search with stemming and stop words in one language setting. At most one text index per collection. For richer search consider Atlas Search. |
| TTL | Deletes documents after a period. The field must be a Date (or an array of Dates). |
| Partial | Indexes only documents matching `partialFilterExpression`. Smaller and cheaper. |
| Sparse | Indexes only documents that contain the field. Prefer partial in new designs. |
| Unique | Rejects duplicates. Enforces a business key. |
| Hashed | Hash of a field. Supports equality only and hashed sharding. |
| Wildcard | Indexes unknown or varying field names, for example `{ "attrs.$**": 1 }`. For dynamic structures. Not a substitute for compound indexes on known queries. |
| Geospatial | `2dsphere` for GeoJSON queries. |

**Compound index order and the prefix rule.** An index on `{ a: 1, b: 1, c: 1 }` supports queries on `a`, on `a, b`, and on `a, b, c` (its **prefixes**), but not efficiently a query on `b` alone. An index on `{ a: 1, b: 1 }` makes a separate `{ a: 1 }` index redundant.

**The ESR rule (Equality, Sort, Range).** Order compound index fields as:
1. **Equality** fields first (exact match conditions).
2. **Sort** fields next, in the sort direction.
3. **Range** fields last (`$gt`, `$lt`, `$ne`). A short `$in` list can behave like equality.

Reason: equality narrows to a contiguous index region. If the next fields are the sort keys, the entries in that region are already in sorted order, avoiding an in-memory sort. A range field placed before the sort key fragments the index traversal and forces a sort. Example for `find({ status: "open", amount: { $gt: 100 } }).sort({ createdAt: -1 })`:

```js
db.orders.createIndex({ status: 1, createdAt: -1, amount: 1 })
```

ESR is a strong default, not a law: confirm with `explain()` on realistic data. Also consider selectivity: a field that matches few documents is worth placing where it narrows most.

**Sort direction in compound indexes.** An index can serve a sort in the exact specified directions or fully reversed. Mixed directions need a matching index.

**Covered queries.** If all fields in the filter and the projection are in the index and `_id` is excluded (unless indexed), the server answers from the index alone and examines zero documents. Fast, but only for narrow queries.

**Multikey limits.** A compound index can have at most one array field per document. Large arrays mean large indexes.

**TTL.** `createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })`. A background task removes expired documents about once a minute, so deletion is not instant. TTL works on a single field and not on capped collections. Use it for sessions, tokens and temporary data.

**Partial and unique together.** A unique partial index enforces uniqueness only among matching documents, for example one active subscription per user.

```js
db.subs.createIndex(
  { userId: 1 },
  { unique: true, partialFilterExpression: { status: "active" } }
)
```

**Building indexes.** Index builds on a large collection take time and resources. In current versions a build holds exclusive locks only briefly at the start and end, so reads and writes continue. From 4.4 the build runs on all data-bearing members at once, and the primary waits for a commit quorum (by default all voting members), so an unreachable member can stall it. A build still competes for I/O, and the index must fit in RAM to be useful. Build during a quiet period and watch replication lag. Never build indexes ad hoc on a loaded production primary without a plan.

**How many indexes.** Each index slows writes and consumes memory. Remove unused indexes. `$indexStats` shows usage counts since the last restart. Hide an index (`hideIndex`, confirm version) to test removal safely before dropping.

**Index size and RAM.** Indexes you use frequently should fit in memory. Check with `db.coll.stats()` or `totalIndexSize`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Create and inspect | `createIndex`, `getIndexes`, `dropIndex`, `hideIndex`, `db.coll.stats()` | Create and remove indexes safely |
| Design | ESR rule, prefix rule, selectivity | Design a compound index for a given query and sort |
| Verify | `explain("executionStats")` | Prove an index is used and no in-memory sort occurs |
| Usage | `$indexStats` | Find unused indexes |
| Special | TTL, partial, unique, text, wildcard, 2dsphere | Know the use case and limit of each |
| Compass | Indexes tab, explain plan view | Review index size and usage visually |

### Common mistakes

- No index on a field used in a frequent filter or sort.
- Many single-field indexes instead of one well-ordered compound index.
- Compound order that ignores ESR, so the sort happens in memory.
- A new index built straight on a busy production primary.
- Over-indexing, which slows every write.
- Expecting TTL to delete at the exact second.
- Unique index on a field that has many missing values (missing counts as null, so only one such document is allowed). Use a partial index.
- Case-insensitive lookups with a case-sensitive index.

### Self-check

- State the ESR rule and why a range field before a sort field hurts.
- Which queries can an index on `{ a: 1, b: 1, c: 1 }` serve?
- When is a query covered?
- How would you enforce one active record per user but allow many historical ones?
- How do you find indexes nobody uses?

---

## 8. Reading explain output

**In plain words.** `explain()` shows how the database ran a query. Compare how many index entries and documents it read with how many it returned. A big gap means the index does not narrow the search enough.

### Understand

`explain()` shows how the server plans and runs a query. It is the central performance tool.

```js
db.orders.find({ status: "open" }).sort({ createdAt: -1 }).explain("executionStats")
```

**Verbosity modes.**
- `queryPlanner`: shows the winning plan and rejected plans, without running the query.
- `executionStats`: runs the query and reports real counts and timings. Use this.
- `allPlansExecution`: also reports statistics for the candidate plans the optimiser compared.

**Plan stages to recognise.**

| Stage | Meaning |
|---|---|
| `COLLSCAN` | Reads the whole collection. A red flag on large collections. |
| `IXSCAN` | Scans an index range. Good. |
| `FETCH` | Reads full documents using index pointers. |
| `SORT` | An in-memory sort. Means no index provided the order. |
| `IXSCAN` with no `FETCH` stage above it | Covered query. |
| `LIMIT`, `SKIP` | Applied after sort. |

Newer versions may show the plan in different shapes (for example slot-based execution). The stage ideas are the same. Confirm stage names in the manual for your version.

**The key numbers in `executionStats`.**
- `nReturned`: documents returned.
- `totalKeysExamined`: index entries read.
- `totalDocsExamined`: documents read from storage.
- `executionTimeMillis`: time.

**The efficiency rule.** The ideal is `nReturned` close to `totalKeysExamined` and `totalDocsExamined`. If a query returns 10 documents but examined 2,000,000 keys or documents, the index does not narrow enough. A covered query shows `totalDocsExamined: 0`.

Reading patterns:
- `COLLSCAN`, large `totalDocsExamined`: add an index.
- `IXSCAN` but `totalKeysExamined` far above `nReturned`: index is on a low-selectivity prefix or the order is wrong. Revisit ESR.
- A `SORT` stage: the index does not provide the order. Add the sort field after equality fields.
- `FETCH` with a filter after it examining many documents: part of the filter is not in the index, so documents are fetched then rejected.
- Index used but slow: check for large documents, cold cache, or a slow range.

**Plan cache.** The server caches the winning plan per query shape. A plan that was best for one parameter value can be poor for another (skewed data). If a query is erratically slow, look at plan choice with `allPlansExecution` and consider an index hint (`hint`) as a diagnostic or last-resort fix. Confirm plan cache tools in the manual.

**Method.** Run explain with realistic data volume and a realistic parameter, change one thing at a time, and re-run.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Explain | `explain("executionStats")`, `explain("allPlansExecution")` | Diagnose any slow query from the plan |
| Hints | `.hint({ field: 1 })` | Force an index to compare plans |
| Compass | Explain tab with visual plan | Share a plan with a colleague |
| Profiler | `system.profile` entries | Find queries worth explaining (chapter 20) |

### Common mistakes

- Running explain on a tiny dev collection and concluding the query is fine.
- Looking only at execution time, not at keys and documents examined.
- Accepting a `SORT` stage on a large result.
- Forgetting that explain with `executionStats` actually runs the query, including a heavy one.
- Judging a query with one parameter value when the data is skewed.

### Self-check

- What do `totalKeysExamined` and `totalDocsExamined` tell you, and what is the ideal relationship to `nReturned`?
- A query shows `IXSCAN` yet is slow. List two reasons.
- How do you recognise a covered query?
- What does a `SORT` stage mean?

---

## 9. The aggregation pipeline

**In plain words.** An aggregation is a pipeline: documents pass through stages, and each stage changes them. Put filters first so the database can use an index and handle less data. Heavy stages such as `$lookup` and `$group` need care.

### Understand

**Mental model.** An aggregation is a pipeline: documents flow through ordered stages, each transforming the stream. It is the way to group, reshape, join and compute inside the database, replacing "fetch everything and loop in the app".

```js
db.orders.aggregate([
  { $match: { status: "paid", createdAt: { $gte: start } } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" }, n: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

**Common stages.**

| Stage | Purpose |
|---|---|
| `$match` | Filter documents. Can use indexes when early. |
| `$project`, `$set` / `$addFields`, `$unset` | Reshape and compute fields. |
| `$group` | Group by a key and accumulate (`$sum`, `$avg`, `$min`, `$max`, `$push`, `$addToSet`, `$first`, `$last`). |
| `$sort`, `$limit`, `$skip` | Order and page. |
| `$unwind` | Emit one document per array element. |
| `$lookup` | Left outer join with another collection. |
| `$facet` | Run several sub-pipelines over the same input. |
| `$bucket`, `$bucketAuto` | Group into ranges. |
| `$count`, `$sortByCount` | Counting shortcuts. |
| `$replaceRoot` / `$replaceWith` | Promote an embedded document. |
| `$unionWith` | Combine with another collection. |
| `$merge`, `$out` | Write results to a collection (materialised views, reports). |
| `$setWindowFields` | Window functions such as running totals (confirm version). |

**Put `$match` and `$sort` first.** Only an early `$match` (and a `$sort` that matches an index) can use an index. A `$match` after `$group` or `$project` works on already transformed data and scans the stream. The optimiser reorders some stages, for example moving a `$match` ahead of a `$project` when safe, and merging a `$sort` with a following `$limit` into a top-k, but do not depend on it. Reduce data early: `$match`, then `$project` only needed fields, then heavier stages.

**`$lookup` and its cost.** `$lookup` joins each input document to matching documents in another collection. Without an index on the foreign field it scans the other collection for every input document. Rules:
- Index the `foreignField` (or the fields used in the sub-pipeline).
- Filter and limit before `$lookup` so fewer documents join.
- Treat `$lookup` as a tool for reporting and occasional joins, not for hot request paths. If a hot path needs one on every read, change the model (embed or use extended reference).
- The `from` collection can be sharded from MongoDB 5.1. Before 8.0, a `$lookup` inside a transaction could not target a sharded collection. Confirm for your version.

**`$group` memory and `$facet`.**
- Blocking stages (`$group`, `$sort` without index, `$bucket`) have a per-stage memory limit of 100 MB. Before MongoDB 6.0, a stage over the limit failed unless you passed `allowDiskUse: true`. From 6.0 the server parameter `allowDiskUseByDefault` (default `true`) lets stages write temporary files to disk, and `allowDiskUse: false` overrides it for one query. Design to avoid it: filter first, index the sort, group on fewer keys.
- `$facet` runs sub-pipelines on one input but its output is a single document, so it is subject to the 16 MiB document limit. Stages inside `$facet` cannot use indexes. Use it for small result sets such as search facet counts, not for large data.
- An aggregation result returned to the client is a cursor, but a `$group` that collects with `$push` into one array can build a huge document and fail on the size limit.

**Aggregation expressions.** `$cond`, `$switch`, `$ifNull`, date operators (`$dateToString`, `$dateTrunc`, confirm version), string, array (`$map`, `$filter`, `$reduce`) and math operators. Field paths are written `"$field"`, variables `"$$var"`.

**Time zones.** Dates are stored in UTC. Date grouping operators accept a `timezone` option. Group by day in the business time zone, not UTC, if reports must match local days.

**Materialising.** For repeated heavy reports, compute with `$merge` into a summary collection on a schedule and read the summary.

**Analytics load.** Heavy aggregations compete with transactional traffic. Run them on a secondary with an appropriate read preference or a dedicated analytics node, accepting slightly stale data.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Stages | `$match`, `$group`, `$project`, `$lookup`, `$unwind`, `$facet`, `$merge` | Build multi-stage pipelines from a requirement |
| Optimisation | `aggregate(..., { explain: true })`, stage ordering | Show that `$match` uses an index and that no stage blows memory |
| Expressions | `$cond`, `$map`, `$filter`, `$reduce`, date operators | Compute derived fields in the database |
| Compass | Aggregation builder | Prototype a pipeline and inspect each stage's output |
| Materialised views | `$merge`, `$out` | Schedule a summary collection |

### Common mistakes

- Late `$match`, so a full scan feeds every stage.
- `$lookup` with no index on the foreign field.
- `$unwind` followed by `$group` on large arrays, multiplying documents, when an array operator would do.
- `$facet` over a huge input.
- Grouping by a high-cardinality key and exceeding memory.
- Running a heavy report on the primary during peak traffic.
- Doing joins in a hot request path that a better model would avoid.

### Self-check

- Why must `$match` come early, and what can the optimiser do for you?
- What are the costs of `$lookup`, and how do you reduce them?
- Why can `$facet` fail on large data?
- How would you compute daily totals in a specific time zone?

---

## 10. Transactions

**In plain words.** A transaction makes several changes succeed or fail together. It costs speed and can conflict with other writes. Use it only when one document cannot hold all the data, and retry on the transient errors the driver reports.

### Understand

**What exists.** Single-document operations are always atomic. **Multi-document transactions** (ACID across several documents, collections and, on sharded clusters, shards) are available on replica sets from MongoDB 4.0 and on sharded clusters from 4.2 (the minimum `featureCompatibilityVersion`). A standalone server does not support transactions.

**What they cost.** A transaction holds a snapshot, takes locks, and writes through the replication machinery. It is slower than single-document writes, increases write conflicts, and long transactions pressure the cache. A transaction has a time limit (default 60 seconds, configurable by the server parameter `transactionLifetimeLimitSeconds`). Large transactions have size constraints too. Keep transactions short, small, and free of user interaction or external calls.

**Write conflicts.** If two transactions modify the same document, one aborts with a `WriteConflict` error, labelled `TransientTransactionError`. The application should retry the whole transaction.

**API.** Use the driver's callback API, which handles retries of transient errors and of an unknown commit result:

```js
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await accounts.updateOne({ _id: from }, { $inc: { bal: -amt } }, { session });
    await accounts.updateOne({ _id: to },   { $inc: { bal:  amt } }, { session });
  }, { readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } });
} finally {
  await session.endSession();
}
```

Every operation must pass the `session`, otherwise it runs outside the transaction. The callback may run more than once, so it must be safe to repeat and must not have side effects outside the database (sending email, calling an API).

**Error labels.** `TransientTransactionError` means retry the whole transaction. `UnknownTransactionCommitResult` means retry the commit. `withTransaction` handles both.

**Avoid by modelling.** Most uses of transactions in MongoDB arise from normalised designs spread over several collections. If the data that must change together lives in one document, a single atomic update suffices and no transaction is needed. Examples: an order with its line items and totals in one document, or a counter and a flag updated in one `$inc` and `$set`.

**When to use them.** Genuine multi-entity invariants that cannot be put in one document: moving funds between two account documents, reserving stock across an order and an inventory collection, keeping a summary consistent with detail rows.

**Alternatives.** Idempotent operations (safe to repeat, with the same result) with a unique key, an outbox collection written atomically with the business document and processed asynchronously, sagas with compensation, and optimistic concurrency using a `version` field in the filter.

```js
// optimistic concurrency: fails (matchedCount 0) if someone else changed it
await coll.updateOne({ _id, version: 7 }, { $set: { ...changes }, $inc: { version: 1 } })
```

**Idempotency.** Retries cause duplicates unless operations are idempotent. Use a client-generated idempotency key with a unique index.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Sessions | `startSession`, `withTransaction`, `commitTransaction`, `abortTransaction` | Write a correct retrying transaction |
| Errors | error labels, `WriteConflict` | Handle transient and unknown-commit errors |
| Alternatives | optimistic concurrency, outbox, idempotency keys | Replace a transaction with a single-document design |
| Diagnosis | `db.currentOp()`, transaction metrics in `serverStatus` | Find long-running transactions |

### Common mistakes

- Using transactions everywhere because the model is over-normalised.
- Forgetting to pass the session to each operation.
- Long transactions with network calls or user waits inside.
- Side effects inside the callback that is retried.
- Hot-spot documents (a global counter) updated in every transaction, causing constant conflicts.
- Not retrying on transient errors.

### Self-check

- Which operations are atomic without a transaction?
- Why might `withTransaction`'s callback run twice, and what does that demand of your code?
- Give an example where you remove the need for a transaction by changing the model.
- What does optimistic concurrency look like in MongoDB?

---

## 11. Replica sets

**In plain words.** A replica set is a group of servers that keep the same data. One server takes the writes and the others copy them. If it fails, the group elects (chooses by vote) a new one.

### Understand

**What a replica set is.** A group of `mongod` processes holding the same data, giving redundancy and automatic failover. One member is the **primary**, which receives all writes. The others are **secondaries**, which copy the primary's changes. Production runs a replica set, never a standalone. Use an odd number of voting members (typically three) so a majority can be formed.

**The oplog.** The primary records every write operation in the **oplog** (`local.oplog.rs`), a capped collection. Secondaries read and apply it asynchronously. The oplog has a fixed size, so it covers a time window. A secondary that falls behind further than the window can no longer catch up and needs a full resync. Watch the oplog window and replication lag.

**Elections.** Members exchange heartbeats. If a secondary cannot reach the primary within the election timeout (`electionTimeoutMillis`, 10 seconds by default), an election begins. A candidate needs votes from a majority of voting members and must be sufficiently up to date. The new primary takes over, and drivers rediscover it. During an election (typically seconds) there is no primary and writes fail or wait. Applications must tolerate it with retryable writes and reasonable timeouts.

**Majority arithmetic.** With 3 members you tolerate 1 failure, with 5 you tolerate 2. Two members tolerate none (losing one leaves no majority). An **arbiter** votes but holds no data. Arbiters reduce the safety of majority write concern and are generally discouraged. Confirm current guidance in the manual.

**Member options.** Priority (influences who becomes primary), hidden members (for reporting or backups, not read by clients), delayed members (a rolling window behind, protection against a bad deployment), and tags (for targeting reads and writes by location).

**Rollback.** If a primary accepted writes that did not reach a majority and then fails, the new primary will not have them. When the old primary rejoins, it **rolls back** those writes. They are removed from its dataset and saved to rollback files for manual recovery. This is the central reason writes you care about need `w: "majority"` (chapter 12).

**Read preference.** Chooses which member serves reads:

| Mode | Behaviour |
|---|---|
| `primary` | Default. Always the primary. Reads are current. |
| `primaryPreferred` | Primary, a secondary if none is available. |
| `secondary` | Only secondaries. May be stale. |
| `secondaryPreferred` | Secondaries, primary if none available. |
| `nearest` | Lowest network latency, regardless of role. May be stale. |

Reading from secondaries scales read capacity only for reads that tolerate staleness, and it can serve data that is later rolled back. Do not read your own recent write from a secondary without a causally consistent session. Secondary reads do not reduce write load, since every member applies all writes.

**Connection string.** List several hosts and the replica set name, so the driver discovers the topology:

```
mongodb://h1:27017,h2:27017,h3:27017/app?replicaSet=rs0&w=majority&retryWrites=true
```

`mongodb+srv://` fetches the host list from DNS (used by Atlas).

**Maintenance.** Rolling upgrades and restarts: restart secondaries one at a time, step down the primary (`rs.stepDown()`), then restart it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Status | `rs.status()`, `rs.conf()`, `rs.printReplicationInfo()`, `rs.printSecondaryReplicationInfo()` | Read member states, lag and oplog window |
| Control | `rs.stepDown()`, `rs.reconfig()`, member priority | Perform a planned failover |
| Driver | replica set connection string, read preference, retryable writes | Configure an application to survive a failover |
| Logs | election and heartbeat messages in `mongod` logs | Explain why an election happened |

### Common mistakes

- Two data members (no failure tolerance) or arbiter-heavy designs.
- Running all members on one host or in one availability zone.
- Reading from secondaries and assuming read-your-writes.
- Oplog too small for a long maintenance window or a heavy bulk load.
- Applications with no retry or timeout behaviour, failing for the whole duration of an election.
- Not alerting on replication lag.

### Self-check

- Walk through what happens from the moment a primary crashes to the moment writes succeed again.
- Why is a two-member replica set a bad design?
- What is rolled back, and which write concern prevents data from being rolled back?
- Why can reading from a secondary return stale or since-rolled-back data?

---

## 12. Consistency and durability: read and write concern

**In plain words.** Write concern says how many servers must have a write before you are told it worked. Read concern says what kind of data a read may return. Together they decide whether you can lose data or read data that later disappears.

### Understand

Two settings decide your guarantees: **write concern** for durability of writes and **read concern** for the isolation of reads. Since MongoDB 5.0 the implicit default write concern is `w: "majority"`, except in some replica sets with arbiters, where it is `w: 1`. The default read concern is `local`. Set both explicitly for important data.

**Write concern.**
- `w: 1`: acknowledged by the primary only. Fast, but the write can be rolled back if the primary fails before replication.
- `w: "majority"`: acknowledged after a majority of voting members have the write. Survives failover and will not be rolled back. Recommended for data you cannot lose.
- `w: 0`: no acknowledgement. Errors are invisible. Avoid.
- `j: true`: wait until the write is in the on-disk journal on the acknowledging members. With `w: "majority"` the journal is flushed before acknowledgement while `writeConcernMajorityJournalDefault` is `true` (its default).
- `wtimeout` (or the operation time limit, `maxTimeMS` and driver `timeoutMS`): bounds how long you wait. A timeout does not mean the write failed, only that it was not confirmed in time.

**Read concern.**
- `local`: returns the data on the member with no guarantee it is majority-committed. Could be rolled back.
- `available`: like local, used mainly with sharding, and may return orphaned documents.
- `majority`: returns only data acknowledged by a majority, so it will not be rolled back.
- `linearizable`: reflects all majority-acknowledged writes before the read, on the primary, at higher latency.
- `snapshot`: for transactions, a consistent point-in-time view.

**Causal consistency.** A causally consistent session guarantees reads reflect the session's earlier writes and monotonic ordering, even across members. Use it when you write and then read from a secondary within a flow.

**Combination for strong guarantees.** Write `w: "majority"` and read `majority` (or `linearizable` where needed) from the primary. This avoids lost acknowledged writes and avoids reading data that later vanishes.

**Retryable writes and reads.** The driver retries a write once after a transient network error or failover when `retryWrites` is on (the default in drivers for MongoDB 4.2 and later). The server deduplicates by transaction number, so a retried write does not apply twice. Multi-document writes such as `updateMany` and `deleteMany` are not retryable.

**What "acknowledged" means.** An acknowledged write with `w: 1` means one node has it in memory. Do not describe that as durable. The trade is latency against safety: choose per operation (for example majority for payments, `w: 1` for ephemeral metrics).

**Single-node durability.** Journaling protects against a crash of one node with a bounded window of loss, and replication with majority protects against loss of the node.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Write concern | `w`, `j`, `wtimeout` on connection string, client, collection, operation | Pick correct settings per data class |
| Read concern | `majority`, `linearizable`, `snapshot`, causal sessions | Explain what each prevents |
| Failure testing | step down the primary during a write load | Show which writes survive |
| Driver options | `retryWrites`, `retryReads`, `timeoutMS` | Configure failure behaviour explicitly |

### Common mistakes

- Relying on defaults without knowing them for your version and topology.
- `w: 1` for critical data on a replica set, then losing writes after a failover.
- Reading from a secondary immediately after writing.
- Treating a write timeout as a failure and retrying non-idempotently.
- Using `linearizable` everywhere and paying the latency.

### Self-check

- What is the difference between `w: 1` and `w: "majority"` after a primary failure?
- What does read concern `majority` protect against?
- Why is a write-concern timeout ambiguous?
- When would you use a causally consistent session?

---

## 13. Sharding

**In plain words.** Sharding splits one large collection across several replica sets. The shard key is the field that decides where each document goes. It is the most important choice here and is hard to change, so try other fixes first.

### Understand

**What and why.** **Sharding** partitions a collection across several replica sets (**shards**) to scale beyond one machine's storage or throughput. It is a last resort for data size or write volume that vertical scaling cannot meet. It adds operational and design complexity. Try indexing, modelling, archiving (TTL) and a larger instance first.

**Components.**
- **Shards**: each is a replica set holding a subset of the data.
- **`mongos`**: stateless routers. Applications connect to them.
- **Config servers**: a replica set storing cluster metadata (which ranges live on which shard).

**Shard key.** The field (or compound fields) that decides placement. It is the most important and hardest-to-change decision. It must be present in documents (and generally immutable in practice, confirm version rules on changing shard key values). Resharding exists in newer versions but is a heavy operation. Confirm availability and limits in the manual.

**Partitioning styles.**
- **Ranged**: contiguous ranges of key values live together. Good for range queries on the key. Risk of hot spots with monotonically increasing keys.
- **Hashed**: the hash of the key is partitioned. Spreads writes evenly. Range queries on the key become scatter-gather.

**Chunks and balancing.** Data is divided into **chunks** (ranges of the shard key). A **balancer** moves chunks between shards to even the distribution, and chunks split as they grow. Migrations use I/O and network, so balancing can be scheduled into windows. The default chunk (range) size is 128 MB. Confirm balancer behaviour for your version.

**What makes a good shard key.**
1. **High cardinality**: many distinct values, so data can split finely.
2. **Even write distribution**: not monotonic (ObjectId, timestamp), which sends every insert to the last chunk and one hot shard.
3. **Query isolation**: your common queries include the shard key, so `mongos` sends them to one shard (**targeted**) instead of all shards (**scatter-gather**).

Common fixes: a hashed `_id` for even writes, or a compound key such as `{ tenantId: 1, _id: 1 }` or `{ customerId: 1, createdAt: 1 }` so one tenant's data is together (targeted queries) while many tenants spread across shards. A low-cardinality key (such as `status` or `country`) creates **jumbo chunks** that cannot split.

**Query routing.** A query with the shard key (or a prefix) is targeted to the relevant shard or shards. A query without it is sent to all shards and merged, which is slower and does not scale with the number of shards. Unique indexes across shards must include the shard key. Sorting and `$group` across shards merge results at `mongos` or on a shard.

**Hot shards.** Symptoms: one shard with high CPU and disk, others idle, balancer unable to help because the traffic is for one key range. Causes: monotonic key, one very large tenant, a celebrity key. Fix by changing or refining the key, using hashed or compound keys, or isolating large tenants.

**Zones.** Zone sharding pins ranges of the key to specific shards, for data locality or tiering.

**Operational points.** Test the shard key with realistic data and query mix before committing. Monitor chunk distribution and balancer state. Avoid operations that touch all shards on hot paths.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspect | `sh.status()`, `db.coll.getShardDistribution()` | Read data distribution across shards |
| Define | `sh.shardCollection()`, hashed versus ranged | Choose and defend a shard key |
| Balancer | `sh.getBalancerState()`, balancer windows | Know what the balancer does and its cost |
| Diagnosis | `explain()` on `mongos` (shows shards touched) | Tell targeted from scatter-gather queries |

### Common mistakes

- Sharding too early, or not needing to shard at all.
- Monotonic shard key (ObjectId or timestamp) causing a hot shard.
- Low-cardinality shard key causing jumbo chunks.
- Frequent queries without the shard key, giving scatter-gather.
- Choosing a shard key without testing real queries.
- Treating the shard key as easy to change later.

### Self-check

- Why does a timestamp shard key cause a hot shard?
- Compare hashed and ranged sharding for a multi-tenant application.
- What are the three properties of a good shard key?
- What is scatter-gather and how do you see it in `explain()`?

---

## 14. Change streams

**In plain words.** A change stream lets your application listen for changes in the database. Each event has a resume token, which lets you continue after a restart. An event can arrive more than once, so your code must handle repeats.

### Understand

**What.** A **change stream** lets an application subscribe to data changes (insert, update, replace, delete, and others) on a collection, database or whole deployment, in order. It is built on the oplog. It requires a replica set or sharded cluster (a standalone has no oplog).

**Use cases.** Cache invalidation, search index updates, event-driven integrations, audit feeds, syncing to another store, triggering workflows. It replaces polling and fragile tailing of the oplog.

```js
const stream = coll.watch(
  [{ $match: { operationType: { $in: ["insert", "update"] } } }],
  { fullDocument: "updateLookup" }
);
for await (const change of stream) {
  // change._id is the resume token
}
```

**Key mechanics.**
- Each event has a **resume token** (`_id`). Persist the last processed token. After a crash or disconnect, resume with `resumeAfter` or `startAfter` so no event is missed. Resuming only works if the token is still within the oplog window.
- By default an update event contains the changed fields, not the full document. `fullDocument: "updateLookup"` fetches the current document, which may already reflect later changes. From MongoDB 6.0 you can enable pre-images and post-images per collection (`changeStreamPreAndPostImages`) and request them with `fullDocumentBeforeChange` and `fullDocument`.
- Events are majority-committed data (they will not be rolled back).
- Filter inside the stream with a pipeline so only relevant events cross the network.

**Delivery semantics.** Treat processing as **at least once**. After a resume, an event may be redelivered, so handlers must be idempotent.

**Caveats.**
- Each open stream holds a cursor and resources. Do not open thousands. Fan out events yourself.
- A slow consumer can fall behind the oplog window and lose the ability to resume.
- On sharded clusters, events merge across shards, with extra cost.
- Large events can hit the 16 MiB limit on full documents.
- Use a single logical consumer per stream with leader election, or events are processed in duplicate by multiple app instances.

**Alternative: outbox.** For reliable event publishing, write an event document in the same atomic write (or transaction) as the business change, and have a worker publish it, optionally driven by a change stream on the outbox collection.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Subscribe | `watch()` with pipeline filter | Stream filtered changes in Node |
| Resume | resume token storage, `resumeAfter`, `startAfter` | Survive a restart without losing events |
| Idempotency | dedupe by event or document id | Make handlers safe to repeat |
| Compass / mongosh | `db.coll.watch()` | Observe events while testing |

### Common mistakes

- Not persisting the resume token, so events are lost on restart.
- Non-idempotent handlers.
- Expecting `updateLookup` to return the document as of the event.
- Running the same stream in every app instance.
- Letting the consumer lag beyond the oplog window.

### Self-check

- What is a resume token, and what limits resuming?
- Why must handlers be idempotent?
- Why does a standalone server not support change streams?

---

## 15. Atlas operational features

**In plain words.** Atlas is the managed cloud service for MongoDB. It runs the servers, backups and monitoring for you. You still own the schema, the indexes, the queries and the access rules.

### Understand

**MongoDB Atlas** is the managed cloud service. It runs replica sets and sharded clusters on cloud providers, and handles provisioning, patching, backups and monitoring. Managing it well still needs the concepts in this guide. Feature names and availability change, so confirm details in the Atlas documentation.

**Core operational features.**
- **Cluster tiers and scaling**: choose instance size, storage and IOPS. Auto-scaling can raise or lower compute and storage within limits you set. Scaling events and failovers are rolling operations.
- **Backups**: cloud backup snapshots with retention policies, and continuous backup with point-in-time restore (confirm which tiers include it). Test restores.
- **Monitoring and alerts**: built-in metrics (operations, connections, replication lag, cache, disk, slow queries) and configurable alerts to email, chat or paging.
- **Performance Advisor and Query Profiler**: show slow queries and suggest indexes. Treat suggestions as input and verify with `explain()` and write cost.
- **Network security**: IP access list, VPC or VNet peering, private endpoints. Do not open `0.0.0.0/0`.
- **Authentication and access**: database users with roles, and integration with identity providers for human access. Use separate users per application.
- **Encryption**: TLS in transit, encryption at rest, optional customer-managed keys, and client-side field level encryption for the most sensitive fields.
- **Atlas Search and Vector Search**: search integrated with the database. Worth evaluating before building your own search pipeline.
- **Data Federation, online archive**: query or tier older data cheaply. Confirm availability.
- **Maintenance windows and version upgrades**: schedule patching to quiet periods. Test upgrades in a non-production cluster first.
- **Connection string**: `mongodb+srv://` with TLS on by default.

**Shared responsibilities.** Atlas manages the servers. You still own schema design, indexes, query efficiency, connection management, access control and restore testing. A managed cluster does not fix a collection scan.

**Cost levers.** Over-provisioned tiers, unused indexes, large documents, unneeded backups retention, and cross-region traffic.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Console | Atlas UI: Metrics, Real-Time panel, Performance Advisor, Query Profiler | Find the slow query and the saturated resource |
| Alerts | Alert configuration | Alert on replication lag, connections, disk, CPU |
| Access | IP access list, private networking, database users | Lock a cluster down correctly |
| Backups | Snapshot policies and restore to a new cluster | Perform and verify a restore |
| CLI and automation | Atlas CLI, infrastructure as code | Reproduce cluster configuration from code (confirm tool availability) |

### Common mistakes

- IP access list open to the world.
- A single shared admin user for every application.
- Never testing a restore.
- Applying every Performance Advisor index suggestion blindly.
- Scaling up a tier to hide a missing index.
- No alerts until users complain.

### Self-check

- What does Atlas manage for you, and what does it not?
- How do you validate that backups actually work?
- Which network exposure options does Atlas offer, and which would you choose for production?

---

## 16. Mongoose

**In plain words.** Mongoose is a Node.js library that adds schemas, validation and helpers on top of the driver. Some of its rules do not run on every operation, for example validators on updates. Learn where its rules apply and where they do not.

### Understand

**What it is.** Mongoose is an object-document mapper for Node.js. It adds schemas, validation, defaults, middleware, population and a query builder on top of the driver. It brings structure to a flexible database and costs some performance and some indirection.

**Schemas and models.**

```js
const orderSchema = new mongoose.Schema({
  customerId: { type: mongoose.Schema.Types.ObjectId, ref: "Customer", required: true, index: true },
  status: { type: String, enum: ["new", "paid", "shipped"], default: "new" },
  total: { type: mongoose.Schema.Types.Decimal128, required: true },
  items: { type: [{ sku: String, qty: { type: Number, min: 1 } }], default: [] }
}, { timestamps: true });

orderSchema.index({ customerId: 1, createdAt: -1 });
const Order = mongoose.model("Order", orderSchema);
```

**Validation.** Mongoose validates in the application on `save()` and `create()`, before the database sees the write. Built-ins include `required`, `min`, `max`, `enum`, `match`, and custom `validate` functions. Key limits:
- Validation does not run by default on `updateOne`, `findOneAndUpdate` and similar. Pass `runValidators: true`, and note validators then run only on updated paths with different `this` semantics.
- Mongoose validation protects only writes through Mongoose. Other writers bypass it. For hard guarantees add `$jsonSchema` validation in the database.
- `strict` mode (default) drops fields not in the schema on write. `strictQuery` controls whether unknown filter fields are dropped, and its default has changed across major versions, so check yours.

**Middleware (hooks).** `pre` and `post` hooks on `save`, `validate`, `find`, `updateOne`, `deleteOne` and others. Use for derived fields, audit stamps and cascading cleanup. Pitfalls: document middleware (`save`) does not run for query operations such as `updateMany` and `findOneAndUpdate`, and bulk operations like `insertMany` and `bulkWrite` bypass many hooks. Do not hide business rules in hooks that some code paths skip.

**Population.** `populate` resolves references by running **additional queries** and merging in the application. It is not a database join.

```js
const orders = await Order.find({ status: "paid" }).populate("customerId", "name email").lean();
```

Pitfalls:
- It issues one extra query per populated path (batched with `$in`), so deep populate chains multiply round trips and memory.
- Populating many documents with large referenced documents bloats memory. Always select fields.
- The populated field is a query on the target collection that must be indexed (`_id` is).
- A frequent populate on every read signals a model that should embed or copy fields (extended reference).
- `$lookup` in aggregation is the in-database alternative, with its own costs.
- Populate with `limit` across many parents does not behave as a per-parent limit. Read the manual for `perDocumentLimit`.

**`lean()`.** By default Mongoose returns full document instances with change tracking, getters, virtuals and methods. `lean()` returns plain objects. Use it for read-only paths. It is markedly faster and lighter. Lean results have no `save()`, no virtuals by default and no casting on read.

**Indexes and `autoIndex` in production.** Mongoose calls `ensureIndexes`-style creation at model initialisation when `autoIndex` is true (default). Consequences in production: every app start tries to build declared indexes, a heavy build can run on a busy primary at deploy time, and a mismatch with existing indexes errors or leaves drift. Recommended practice: set `autoIndex: false` in production (schema option or connection option), manage indexes through versioned migrations or an explicit deployment step, and use `syncIndexes()` deliberately, knowing it drops indexes not in the schema. Confirm behaviour for your Mongoose version.

**Other points.**
- **Buffering**: by default Mongoose buffers commands until connected. This can hide a failed connection as hanging operations. Consider `bufferCommands: false` and fail fast.
- **Transactions**: `connection.transaction()` or sessions passed with `{ session }` on every call.
- **Operator injection**: `sanitizeFilter` option converts nested objects in filters into `$eq` so request bodies cannot inject operators (chapter 18).
- **Schemas for arrays and subdocuments** create `_id` for every subdocument by default, which costs space. Set `_id: false` where unneeded.
- **Versioning**: Mongoose adds `__v` to protect array updates. Know it exists.
- **`save()` rewrites only modified paths** but loads the whole document first. For hot counters use atomic `updateOne` with `$inc`.
- **Discriminators** and virtuals exist. Use sparingly.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schema | types, defaults, `timestamps`, `enum`, custom validators, `_id: false` | Model a collection with proper validation |
| Hooks | `pre`/`post` for document and query middleware | Know exactly which operations trigger which hooks |
| Reading | `lean()`, `select()`, `populate()` with field selection | Choose the lightest read for each endpoint |
| Indexes | `schema.index`, `autoIndex`, `syncIndexes`, `Model.collection.getIndexes()` | Manage indexes outside application start |
| Debug | `mongoose.set("debug", true)` in development | See the real queries Mongoose sends |

### Common mistakes

- Leaving `autoIndex` on in production.
- Expecting validators and hooks to run on `updateOne`, `updateMany` and `insertMany`.
- Populating several levels deep on list endpoints.
- Returning full Mongoose documents from read-only endpoints instead of `lean()`.
- Assuming Mongoose schema validation protects the data from other clients.
- A unique index declared in the schema but never actually created because `autoIndex` was off.
- Ignoring buffering, so requests hang when the database is down.

### Self-check

- Which operations skip Mongoose validators and hooks by default?
- Explain how `populate` works and why it can be slow.
- When do you use `lean()` and what do you lose?
- Why disable `autoIndex` in production, and how do you create indexes instead?

---

## 17. The Node driver and connection pooling

**In plain words.** The driver is the library your app uses to talk to MongoDB. Create one client for the whole process, because it keeps a pool (a set of reusable connections). Set timeouts so a failover does not hang your requests.

### Understand

**One client per process.** `MongoClient` represents the connection to the deployment and owns a **connection pool for each server** it monitors. Create it once at start-up, share it across the application, and close it on shutdown. Creating a client per request defeats pooling, exhausts connections and slows every call with new TCP and TLS handshakes.

```js
const client = new MongoClient(process.env.MONGODB_URI, {
  maxPoolSize: 50,
  minPoolSize: 5,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000
});
await client.connect();
const db = client.db("app");
process.on("SIGTERM", async () => { await client.close(); process.exit(0); });
```

**Pool settings (confirm defaults for your driver version).**
- `maxPoolSize`: maximum concurrent connections per server (the default is 100 in current drivers). Operations beyond it wait in a queue.
- `minPoolSize`: connections kept warm.
- `maxIdleTimeMS`: how long an idle connection stays open.
- `waitQueueTimeoutMS`: how long an operation waits for a connection from the pool before erroring. The default is 0, meaning no limit, so set a value.
- `serverSelectionTimeoutMS`: how long to wait to find a suitable server (for example during an election) before failing.
- `connectTimeoutMS`, `socketTimeoutMS`: network timeouts.
- `maxTimeMS` per operation (or `timeoutMS` in newer drivers): bounds server-side execution.

**Sizing the pool.** Total connections to the database equal instances multiplied by pool size per server. With 40 app instances at a pool of 100 the server can see 4,000 connections, each costing memory on the server (and the server has a connection limit, with cloud tiers capping it). Size `maxPoolSize` to the real concurrency, not a large guess, and account for every process, including workers and scheduled jobs. Pool exhaustion shows as rising latency and `waitQueueTimeoutMS` errors, and is often caused by slow queries holding connections, not by too small a pool.

**Serverless and short-lived runtimes.** Each function instance holds its own pool. Reuse the client across invocations by creating it outside the handler, keep `maxPoolSize` small, and watch total connections. Confirm vendor guidance.

**Cursors and streaming.** Iterate with `for await` or `.stream()`. Avoid `toArray()` on large results. Close cursors you abandon.

**Error handling.** Handle `MongoServerSelectionError` (cannot reach a suitable server), `MongoNetworkError`, duplicate key (code 11000), and timeouts. Distinguish retryable from non-retryable errors. Use `retryWrites` (on by default in drivers for MongoDB 4.2 and later) and `retryReads` (also on by default; unverified here, confirm in your driver docs), and still design operations to be idempotent.

**Observability.** Drivers emit monitoring events (command started, succeeded, failed, pool events) for logging slow commands and tracking pool saturation. Use them to log operation names and durations (never full filter values containing personal data).

**Graceful shutdown.** On `SIGTERM`, stop accepting requests, finish in-flight work, then `client.close()`. This ties to the signal handling in the Linux guide.

**ObjectId handling.** Convert string ids from requests with `new ObjectId(str)` after validating with `ObjectId.isValid` and a strict format check. An invalid id throws. Comparing an ObjectId with a string never matches in queries.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Client lifecycle | singleton `MongoClient`, `close()` on shutdown | Set up a correct client module |
| Pool | `maxPoolSize`, `minPoolSize`, `waitQueueTimeoutMS`, pool events | Size and monitor the pool |
| Timeouts | `serverSelectionTimeoutMS`, `maxTimeMS` or `timeoutMS` | Make failures fast and bounded |
| Monitoring | `monitorCommands: true` with command events | Log slow commands from the app |
| Server side | `db.serverStatus().connections` | Compare app-side and server-side connection counts |

### Common mistakes

- A new `MongoClient` per request or per function call.
- Pool sized without multiplying by instance count.
- No timeouts, so requests hang during failover.
- Not closing the client on shutdown, leaving connections to time out on the server.
- Loading huge result sets into memory.
- Passing unvalidated request strings straight to `new ObjectId`.

### Self-check

- Why one `MongoClient` per process?
- What is the total number of connections the database may see, and how do you compute it?
- A latency spike with `waitQueueTimeoutMS` errors appears. What are likely causes?
- What happens to your app during an election, and which settings govern it?

---

## 18. Security

**In plain words.** Security has several layers: network, login, permissions, encryption and input checks. Two common serious problems are a database open to the internet and unchecked user input in queries. Apply every layer, not only one.

### Understand

**Defence in layers.** Network exposure, authentication, authorisation, encryption, and input handling. Most real MongoDB breaches came from databases exposed to the internet with no authentication. Treat that as the baseline failure to avoid.

**Network exposure.**
- Bind `mongod` only to the interfaces it needs (`net.bindIp`). Do not bind to all interfaces on a public host.
- Put databases in private networks, allow only application hosts through security groups and firewalls, and use private endpoints or peering with Atlas.
- Never expose port 27017 to the internet.

**Authentication.** Enable access control (`security.authorization: enabled`). The default mechanism is SCRAM (username and password). Other mechanisms (x.509 certificates, LDAP, cloud identity) exist, some Enterprise-only. Create an administrator, then create separate application users.

**Authorisation: role-based access control.** Users get roles on databases. Built-in roles include `read`, `readWrite`, `dbAdmin`, `userAdmin`, `clusterMonitor` and `root`. Principle of least privilege: an application user gets `readWrite` on its database only, never `root` or `userAdmin`. Custom roles can restrict to specific collections and actions.

```js
db.getSiblingDB("app").createUser({
  user: "orders_svc",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "app" }]
})
```

**Encryption.** TLS for connections between clients and servers and between cluster members. Encryption at rest through the storage engine (Enterprise) or through disk or cloud-level encryption (Atlas). **Client-side field level encryption** and **Queryable Encryption** protect specific fields so the server never sees plaintext. Confirm availability and limits.

**Operator injection (NoSQL injection).** MongoDB queries are data structures. If you put untrusted JSON straight into a filter, an attacker can supply operators instead of values. Classic login bypass:

```js
// request body: { "email": "a@b.com", "password": { "$ne": null } }
const user = await users.findOne({ email: req.body.email, password: req.body.password });
// password condition becomes "not null", which is true, and authentication is bypassed
```

Other vectors: `{ "$gt": "" }`, `$regex` for data extraction by timing or content, `$where` with JavaScript, and Express's `qs` parsing turning `?password[$ne]=x` into an object.

Defences:
1. **Validate types and shape** of all input with a schema validator (for example Zod, Joi, or Ajv). Require strings where strings are expected, so an object is rejected.
2. **Coerce explicitly**: `String(req.body.email)`.
3. **Strip operator keys** from untrusted objects (keys starting with `$` or containing `.`), with a sanitiser, or use Mongoose's `sanitizeFilter`.
4. **Never pass `req.body` or `req.query` directly as a filter or update**. Pick known fields.
5. **Never store or compare plaintext passwords.** Look up by identifier, then verify a password hash (bcrypt, argon2, or scrypt) in application code. This makes the `$ne` trick useless for passwords.
6. **Disable server-side JavaScript** (`security.javascriptEnabled: false`) if you do not use `$where`, `$function`, `$accumulator` or `mapReduce`.
7. **Restrict update bodies**: mass assignment (`updateOne(filter, { $set: req.body })`) lets clients set `role: "admin"`.

**Auditing and secrets.** Store connection strings and passwords in a secret manager or protected environment, never in the repository. Rotate credentials. Audit logging is available in Enterprise and Atlas. Do not log full filters with personal data.

**Hardening checklist.** Access control on, private network, TLS, least-privilege users per application, no default or shared admin credentials, JavaScript disabled where unused, current patched version, encrypted and tested backups, and alerts on authentication failures.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Users and roles | `createUser`, `createRole`, `db.getUsers()`, built-in roles | Create a least-privilege application user |
| Network | `net.bindIp`, firewall, Atlas IP list and private endpoints | Prove the database is unreachable from the public internet |
| TLS | `net.tls` options, `tls=true` in the connection string | Enable and verify TLS end to end |
| Input safety | Zod, Joi or Ajv, Mongoose `sanitizeFilter` | Reject operator objects in request bodies |
| Testing | send `{"$ne": null}` and `{"$gt": ""}` in login and filter endpoints | Demonstrate and fix operator injection |

### Common mistakes

- Running without authentication, bound to a public interface.
- Application connecting as an admin or `root` user.
- Passing `req.body` directly as a filter or `$set`.
- Comparing plaintext or reversible passwords in the query.
- Credentials committed to a repository or logged.
- Leaving JavaScript execution enabled when unused.
- Atlas access list at `0.0.0.0/0`.

### Self-check

- Show a login bypass using `$ne` and give three independent defences.
- What does `net.bindIp` control, and what is a safe value?
- Which role does an application service account need, and which must it never have?
- What is mass assignment in a MongoDB update?

---

## 19. Backups and restore

**In plain words.** A backup is useful only after you have tested a restore. Replication copies mistakes too, so it is not a backup. Choose a method from how much data you can lose and how long you can be down.

### Understand

A backup is only real after a restore has been tested. Define the **recovery point objective** (how much data you can lose) and **recovery time objective** (how long you can be down), then choose methods accordingly.

**Methods.**
- **Logical backups: `mongodump` and `mongorestore`.** Export data as BSON. Portable across versions and good for small to medium datasets, selective collections and migrations. They read every document, so they load the database and are slow on large datasets. On a replica set, use the `--oplog` option to capture a consistent point-in-time view (confirm the option behaviour for your tool version). `mongorestore` rebuilds data and indexes. Use `--gzip` and `--archive` to compress and package. `mongodump` is not the recommended route for large sharded clusters. Confirm in the manual.
- **Filesystem or volume snapshots.** Fast for large data. They require the snapshot to be consistent (journaling on the same volume, or briefly locking writes with `fsyncLock` on setups where required). Confirm the exact conditions for your storage layout.
- **Managed backups.** Atlas cloud backup provides snapshots and, on suitable tiers, continuous backup with point-in-time restore. Self-managed Enterprise tooling (Ops Manager and similar) offers similar features.
- **Delayed replica set member.** Protects against a bad deployment or accidental deletion within its delay window. It is not a backup.
- **Replication is not backup.** A `deleteMany({})` replicates to every member within seconds.

**Point-in-time recovery.** Restore a base snapshot, then replay the oplog up to a chosen moment (just before the incident). Needs continuous oplog capture.

**Restoring safely.** Restore to a new, isolated cluster first, validate counts and sample queries, then promote or copy back the needed data. Be careful with `mongorestore --drop`, which removes existing collections before restoring.

**Backup hygiene.** Encrypt backups, store them off the cluster and in another account or region, protect them with access control (they contain all your data), set retention, monitor job success, and run restore drills on a schedule.

**Mongodump example (conceptual).**

```
mongodump --uri="$MONGODB_URI" --gzip --archive=backup.gz
mongorestore --uri="$TARGET_URI" --gzip --archive=backup.gz --nsInclude="app.*"
```

Check exact flags with `mongodump --help` for your version.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Logical | `mongodump`, `mongorestore` (`--gzip`, `--archive`, `--oplog`, `--nsInclude`) | Back up and restore a database and one collection |
| Snapshots | cloud volume snapshots, `fsyncLock` and `fsyncUnlock` | Explain consistency requirements |
| Managed | Atlas cloud backup and point-in-time restore | Restore to a new cluster |
| Verification | restore drill with count and checksum comparison | Prove a backup works |

### Common mistakes

- Treating replication or a delayed member as a backup.
- Never testing a restore.
- Backups stored on the same host or account as the database.
- `mongodump` of a huge production primary during peak load.
- Backups that omit indexes or users and surprise you at restore time.
- Unencrypted backups readable by many people.

### Self-check

- Why is a replica set not a backup?
- When would you choose snapshots over `mongodump`?
- How does point-in-time recovery work?
- What is your restore drill, and how do you know it succeeded?

---

## 20. Monitoring and diagnostics

**In plain words.** Monitoring means watching the database so you see problems early. Use the profiler, `currentOp` and `serverStatus` to find slow work. Follow a fixed order when the database is slow, and change one thing at a time.

### Understand

Watch the database as a system: **throughput, latency, saturation and errors** for each resource, the method used in the Linux guide.

**What to watch.**
- **Operations**: reads, writes and commands per second (`opcounters`).
- **Latency**: slow operations and tail latency.
- **Connections**: current versus available, and growth.
- **Replication**: lag, oplog window, member states, election count.
- **Cache and memory**: WiredTiger cache usage and evictions, page faults, working set versus RAM.
- **Disk**: free space, IOPS, queue depth, latency.
- **Locks and queues**: operations waiting, ticket availability (confirm the metric names for your version).
- **Query efficiency**: scanned-to-returned ratio, collection scans, in-memory sorts.

**Database profiler.** Records operations slower than a threshold into the capped collection `system.profile` per database.

```js
db.setProfilingLevel(1, { slowms: 100 })   // level 1: slow ops only
db.system.profile.find().sort({ ts: -1 }).limit(5)
db.setProfilingLevel(0)                    // turn off
```

Level 2 profiles everything and is expensive. Each entry shows the namespace, command, `keysExamined`, `docsExamined`, `nreturned`, duration and plan summary. Group by query shape and look at the highest total time, not only the slowest single call. Atlas offers a profiler UI with the same idea. Slow operations also appear in the server log according to `slowms`.

**`db.currentOp()`.** Lists in-progress operations, with duration, client, lock state and plan. Use it to find long-running queries, index builds, transactions and operations waiting for locks. Filter, for example by `secs_running`. `db.killOp(opid)` stops one. Kill carefully.

**`db.serverStatus()`.** Returns server-wide counters: `connections`, `opcounters`, `mem`, `wiredTiger.cache`, `globalLock`, `network`, `repl`, and more. Sample it over time (counters accumulate since start) and compute rates, or let a monitoring system do so.

**`mongostat`.** A command-line tool printing a line per second of operations, queued reads and writes, connections, cache dirty and used percentages, and network. It is like `vmstat` for MongoDB. `mongotop` shows per-collection read and write time.

**Other useful commands.**
- `rs.status()` and `rs.printSecondaryReplicationInfo()`: replication state and lag.
- `db.coll.stats()`, `db.stats()`: sizes, index sizes.
- `db.coll.aggregate([{ $indexStats: {} }])`: index use.
- `db.coll.aggregate([{ $collStats: { storageStats: {} } }])`: storage detail.
- The `mongod` log: slow queries, elections, connection events.

**Investigation flow for "the database is slow".**
1. Check the resource picture: `mongostat`, host CPU, disk and memory (Linux guide, chapter 6).
2. Check `currentOp` for long operations, lock waits, index builds.
3. Check the profiler or Atlas profiler for top query shapes by total time.
4. `explain("executionStats")` the worst shapes. Look for collection scans, big `docsExamined`, in-memory sorts.
5. Check connections and the pool, then replication lag if reading from secondaries.
6. Compare with a baseline and with recent deploys. Fix one thing, verify.

**Alert thresholds.** Replication lag, connection count near limit, disk free, cache pressure, a sustained rise in scanned-to-returned ratio, elections, and failed authentication spikes.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Live view | `mongostat`, `mongotop` | Read a live health snapshot |
| Slow queries | profiler (`setProfilingLevel`, `system.profile`), slow log | Rank query shapes by total time |
| In-flight work | `db.currentOp()`, `db.killOp()` | Find and stop a runaway operation safely |
| Counters | `db.serverStatus()`, `rs.status()` | Compute rates and read replication health |
| Index use | `$indexStats`, `db.coll.stats()` | Find unused and oversized indexes |
| Dashboards | Atlas metrics, Prometheus exporters or vendor tools (confirm tooling) | Build alerts on the key signals |

### Common mistakes

- Profiling level 2 left on in production.
- Reading `serverStatus` counters as instantaneous values rather than cumulative.
- Looking only at the slowest single query, not the aggregate cost of a frequent one.
- Killing an index build or a long operation without understanding the consequence.
- No baseline, so nothing looks abnormal.
- Alerting on CPU only, missing replication lag and cache pressure.

### Self-check

- How do you find the top three query shapes by total time?
- What does `currentOp` reveal that the profiler does not?
- Which `serverStatus` fields would you graph, and why compute rates?
- Where do you look first when replication lag climbs?

---

## 21. Common production mistakes

**In plain words.** This chapter is a short checklist of mistakes seen in production. Use it to review a design or an incident. Each group points to the chapters with the reasoning.

- **Modelling (chapters 3, 4):** unbounded arrays growing toward 16 MiB. Joins done in application code. No schema validation. No plan for migrating documents.
- **Indexing and queries (chapters 6, 7, 8):** missing indexes and collection scans. Wrong compound index order causing in-memory sorts. Deep `skip` pagination. Case-insensitive or leading-wildcard regex at scale. Too many or unused indexes. A large index built on a loaded primary with no plan.
- **Correctness (chapters 5, 10, 12):** read-modify-write in application code. `w: 1` for critical data. Secondary reads expecting read-your-writes. Long or non-retried transactions. Non-idempotent retries creating duplicates.
- **Topology and scale (chapters 11, 13):** two-member or single-zone replica sets. Oplog too small and no lag alert. Sharding without need, or with a monotonic or low-cardinality shard key.
- **Application (chapters 16, 17):** a `MongoClient` per request. Pool size not matched to instance count. No timeouts or retry strategy. Mongoose `autoIndex` on in production, deep `populate`, no `lean()`. Unbounded results loaded into memory.
- **Security (chapter 18):** database reachable from the internet. No authentication. Application using an admin user. `req.body` passed into filters and updates. Credentials in the repository.
- **Operations (chapters 19, 20):** backups never restored in a test, or replication mistaken for backup. No alerts on lag, disk, connections or cache. Heavy reports on the primary at peak. No TTL or archival, so the disk fills. Upgrades without release notes and a test on a copy.

---

## 22. Practice labs

**In plain words.** These labs let you practise each idea on a real server. Use a test cluster, never production. Try each lab before you look up the answer.

Do these on a local replica set (a Docker-based three-node set or a free Atlas cluster). Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Type traps.** Insert the same value as string, Int32 and Double. Show how `find`, sort and `$type` treat each, and fix a query that missed documents.
2. **Model from access patterns.** Pick a small product (shop, blog, ticketing). Write the access-pattern table, then design collections. Justify each embed or reference and show the maximum size of every array.
3. **Unbounded array.** Push entries into one document until it fails near 16 MiB, record the failure and the slowdown on the way, then remodel with a separate collection or the bucket pattern.
4. **Bucket pattern.** Load 5 million readings as one document each, then as hourly buckets. Compare collection size, index size and the time of a range query.
5. **ESR in practice.** With 1 million documents, create indexes in three orders for one query with equality, sort and range. Use `explain("executionStats")` to show which avoids a `SORT` stage and which examines the fewest keys.
6. **Collection scan to covered query.** Start from a `COLLSCAN`, fix it with an index, then make the query covered. Record `totalKeysExamined`, `totalDocsExamined` and time at each step.
7. **Partial and unique.** Enforce "one active subscription per user" with a partial unique index. Show that a second active one fails and an inactive one is allowed.
8. **TTL.** Create a session collection with a TTL index and observe the deletion delay. State why it is not exact.
9. **Pagination.** Compare `skip` and keyset pagination at page 100,000. Record `docsExamined` and latency.
10. **Aggregation report.** Write a pipeline for top customers by revenue per month in a specific time zone. Show that the early `$match` uses an index, then add `$lookup` for customer names with and without an index on the foreign field and compare.
11. **Operator injection.** Build a small login endpoint that is vulnerable to `{ "password": { "$ne": null } }`. Exploit it, then fix it with schema validation and password hashing, and re-test.
12. **Atomic claim.** Implement a job queue with `findOneAndUpdate` and run several workers concurrently. Prove no job is processed twice.
13. **Transaction versus single document.** Implement a funds transfer with a transaction, force a write conflict, and observe the retry. Then design a case where a single-document model removes the transaction.
14. **Failover drill.** Run a write loop with `w: 1`, then with `w: "majority"`, kill the primary mid-run and compare which acknowledged writes survive. Observe the election in `rs.status()` and in the driver's behaviour.
15. **Secondary read staleness.** Read from a secondary right after a write, with and without a causally consistent session. Show the difference.
16. **Change stream consumer.** Build a consumer that stores its resume token, kill it, restart it and verify no event is lost and handlers are idempotent.
17. **Shard key experiment.** On a test sharded cluster, shard a collection by ObjectId, then by a hashed key, then by a compound tenant key. Compare chunk distribution and the shards touched by your main queries.
18. **Mongoose pitfalls.** Demonstrate that `updateOne` skips validation without `runValidators`, that `insertMany` skips `save` hooks, and measure `lean()` against full documents on 100,000 rows.
19. **Pool exhaustion.** Set `maxPoolSize` low, run slow queries concurrently, and observe `waitQueueTimeoutMS` errors. Then size the pool for N app instances and compare with `serverStatus().connections`.
20. **Profiler hunt.** Turn on the profiler, run a mixed workload with two bad queries, and find them by total time. Fix them and show the improvement.
21. **Backup and restore drill.** Take a `mongodump`, restore it into a fresh instance, and verify counts, indexes and a sample query. Write down the time taken.
22. **Incident write-up.** Break something on purpose (remove an index, fill the disk, stop the primary), diagnose it using only chapter 20, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 23. Interview questions

**In plain words.** This chapter has interview questions with model answers. Start with the Top 10, then read each level. Every question is tagged [Fresher], [Mid] or [Senior]. Answers are short, so prepare an example from your own work for each one.

### Top 10 must-know

Read these first. The model answer for each is in the numbered question given.

| Question | Level | Model answer |
|---|---|---|
| When do you embed, and when do you reference? | [Fresher] | Q10 |
| What does an index do, and what does it cost? | [Fresher] | Q6 |
| What does `w: "majority"` mean? | [Fresher] | Q14 |
| What is operator injection and how do you stop it? | [Fresher] | Q15 |
| Explain the ESR rule. | [Mid] | Q16 |
| How do you read `explain("executionStats")`? | [Mid] | Q18 |
| Why is `skip` pagination slow, and what replaces it? | [Mid] | Q20 |
| When do you use a transaction, and what does it cost? | [Mid] | Q27 |
| What happens when a primary fails? | [Mid] | Q30 |
| How do you guarantee an acknowledged write is not lost? | [Senior] | Q39 |

### Fresher level

**1. [Fresher] What is a document, and how does a collection differ from a table?**
A document is a BSON record of field and value pairs, which can contain arrays and nested documents. A collection groups documents and does not enforce a fixed set of columns by default, though you can add validation. A document often holds data a relational design would spread over several tables.

**2. [Fresher] What is BSON and why does it matter?**
Binary JSON with extra types (ObjectId, Date, Decimal128, Int64, binary). It is typed, so a string `"5"` and number `5` are different for queries and sorting. It also defines the 16 MiB document limit in terms of its encoded size.

**3. [Fresher] What is an ObjectId?**
A 12-byte identifier: a 4-byte timestamp, a 5-byte random value and a 3-byte counter. It is generated by the client, is unique without coordination, and roughly increases with time.

**4. [Fresher] What is `_id`?**
The required, unique, automatically indexed primary key of each document. It is immutable. If not supplied, the driver generates an ObjectId.

**5. [Fresher] How do you store money?**
Use Decimal128 (or integer minor units such as cents). Avoid Double because of binary rounding error.

**6. [Fresher] What does an index do, and what does it cost?**
It lets the server find and order documents without scanning the collection. It costs write time, memory and disk, since every write updates every index.

**7. [Fresher] What is a collection scan and how do you find one?**
A read of every document in the collection. `explain()` shows `COLLSCAN`, and the profiler shows high `docsExamined` relative to `nreturned`.

**8. [Fresher] What is the difference between `updateOne` with `$set` and `replaceOne`?**
`$set` changes only the named fields. `replaceOne` replaces the whole document except `_id`, discarding fields not in the replacement.

**9. [Fresher] What does `upsert` do?**
Updates the matching document, or inserts one if none matches. Pair it with a unique index to be race-safe.

**10. [Fresher] When do you embed, and when do you reference?**
Embed data that is read with the parent, bounded in size and owned by it. Reference data that is large, shared, accessed on its own or unbounded.

**11. [Fresher] What is the 16 MiB limit, and what does it imply?**
It is the maximum BSON document size. It forbids unbounded embedded arrays and large blobs inside documents (use references or GridFS).

**12. [Fresher] What is the difference between `find().limit(10)` with and without an index on the sort field?**
With a matching index the server walks the index in order and stops after 10. Without one it must read and sort all matches in memory before returning 10.

**13. [Fresher] What is a replica set and why use one?**
Several `mongod` members with the same data: one primary taking writes and secondaries replicating. It gives redundancy and automatic failover.

**14. [Fresher] What does `w: "majority"` mean?**
The write is acknowledged only after a majority of voting members have it, so it survives a primary failure.

**15. [Fresher] What is operator injection?**
Untrusted JSON placed into a query carries operators like `$ne`, changing the query's meaning, for example bypassing a login check. Validate types and never pass request bodies directly as filters.

### Mid level

**16. [Mid] Explain the ESR rule.**
Order compound index fields as Equality, then Sort, then Range. Equality narrows to a contiguous region, sort fields in that region are already ordered so no in-memory sort is needed, and range fields last avoid breaking the order. Verify with `explain()`.

**17. [Mid] Which queries can `{ a: 1, b: 1, c: 1 }` serve?**
Queries using a prefix: `a`, `a, b`, and `a, b, c`, with sorts that follow the index order after equality fields. It does not efficiently serve a filter on `b` or `c` alone.

**18. [Mid] How do you read `explain("executionStats")`?**
Check the winning plan stages (`IXSCAN`, `COLLSCAN`, `FETCH`, `SORT`), then compare `nReturned` with `totalKeysExamined` and `totalDocsExamined`. A large gap means poor selectivity. A `SORT` stage means no index gave the order. A covered query has zero documents examined.

**19. [Mid] What is a covered query?**
One answered entirely from an index: filter and projected fields are all in the index and `_id` is excluded or indexed. `totalDocsExamined` is 0.

**20. [Mid] Why is `skip` pagination slow, and what is the alternative?**
The server walks and discards `n` entries for each page. Use keyset pagination: sort on an indexed unique-ordered field and filter by the last seen value, adding `_id` as a tiebreaker.

**21. [Mid] Why can a case-insensitive regex be slow?**
It cannot use an ordinary index efficiently and scans index keys or documents. Use a case-insensitive collation with a matching index, a normalised lowercase field, or a text or Atlas Search index.

**22. [Mid] How does a TTL index behave?**
It removes documents once a Date field is older than `expireAfterSeconds`. A background task runs about once a minute, so deletion is approximate. It is single-field and not for capped collections.

**23. [Mid] How do you enforce uniqueness only for some documents?**
A unique partial index with `partialFilterExpression`. Alternatively handle missing fields, since a unique index treats missing as null.

**24. [Mid] Where should `$match` go in a pipeline, and why?**
First, so it can use an index and reduce the data flowing through later stages. A `$match` after `$group` or `$project` cannot use the collection's indexes.

**25. [Mid] What are the costs of `$lookup`?**
Each input document triggers a lookup in the other collection, so it is expensive without an index on the foreign field and with many inputs. Filter first, index the foreign field, and avoid it on hot paths by embedding or copying data.

**26. [Mid] What limits does `$group` and `$facet` have?**
Blocking stages have a per-stage memory limit, with disk spill depending on the `allowDiskUse` setting and version. `$facet` produces a single document, so it is bound by 16 MiB and its sub-pipelines cannot use indexes.

**27. [Mid] When do you use a transaction, and what does it cost?**
For invariants across several documents that cannot be placed in one document. It adds latency, write conflicts and cache pressure, has a time limit, and must be retried on transient errors. Prefer modelling that keeps related data in one document.

**28. [Mid] How do you handle a `WriteConflict`?**
It arrives as a `TransientTransactionError`. Retry the whole transaction, which `session.withTransaction` does, so the callback must be idempotent and side-effect free.

**29. [Mid] What are read preference and read concern, and how do they differ?**
Read preference chooses which member serves a read (primary, secondary, nearest). Read concern sets what data the read may return (local, majority, linearizable, snapshot). One is about where, the other about isolation.

**30. [Mid] What happens when a primary fails?**
Secondaries miss heartbeats, hold an election, and a candidate with the most up-to-date data and a majority becomes primary, typically within seconds. Drivers rediscover it, retry retryable writes, and writes acknowledged with `majority` survive. Unreplicated `w: 1` writes can be rolled back.

**31. [Mid] What is the oplog?**
A capped collection on each data member recording write operations. Secondaries replay it to stay in sync, and change streams read it. Its size sets the window a lagging member can recover from.

**32. [Mid] How does Mongoose `populate` work, and what is wrong with overusing it?**
It runs additional queries and merges results in Node, not a database join. Deep or wide populate multiplies round trips and memory. Select fields, use `lean()`, and consider embedding or an extended reference.

**33. [Mid] Why disable Mongoose `autoIndex` in production?**
Every app start would attempt to build indexes, possibly a heavy build on a busy primary, with drift and surprises. Manage indexes through controlled migrations and a deployment step.

**34. [Mid] How does the Node driver pool connections?**
One `MongoClient` keeps a pool per server, up to `maxPoolSize`. Operations queue when it is full. Create one client per process, size the pool for real concurrency times instance count, and set timeouts.

**35. [Mid] What backup options exist and what are their trade-offs?**
`mongodump`/`mongorestore` are portable but slow and load the database. Volume snapshots are fast but need consistency handling. Atlas cloud backup and continuous backup give snapshots and point-in-time restore. Replication and delayed members are not backups. Always test restores.

### Senior

**36. [Senior] Design a schema for an e-commerce order system. What do you embed and what do you reference?**
Embed line items and a shipping address snapshot in the order (read together, bounded, historical copy needed). Reference customer and product by id, copying display fields needed on order views (extended reference). Keep a computed `total` and `status` in the order with a `version` for optimistic concurrency. Inventory lives in its own collection, with a transaction or an atomic conditional `$inc` to reserve stock. Index `{ customerId, createdAt }` and `{ status, createdAt }`. Document the maximum items per order.

**37. [Senior] How do you choose a shard key for a multi-tenant SaaS with a few very large tenants?**
Need high cardinality, even writes and targeted queries. A hashed or compound key such as `{ tenantId: 1, _id: 1 }` keeps a tenant's data together but a huge tenant can create a hot shard or jumbo chunk. Options are a hashed compound with a high-cardinality secondary field, isolating the largest tenants through zones, and testing with real data. Avoid monotonic keys and low-cardinality keys.

**38. [Senior] A query is fast in staging and slow in production. How do you investigate?**
Compare data volume and distribution first. Run `explain("executionStats")` in production with the real parameter values, check plan choice (skewed data and the plan cache), keys and docs examined, `SORT` stages and cache state. Check `currentOp` for contention, the profiler for the real shape, replication lag if reading secondaries, and a recent index or deploy change.

**39. [Senior] How do you guarantee an acknowledged write is not lost?**
Use `w: "majority"` (with journal durability per version defaults) on a replica set with data-bearing members across failure domains, handle timeouts as unknown outcomes using idempotent retries, and read with `majority` where you need to see committed data. Keep enough members that a majority survives a zone failure.

**40. [Senior] Describe consistency when reading from secondaries.**
Secondary reads are asynchronous and can be stale, and with `local` read concern can return data later rolled back. Use causally consistent sessions for read-your-writes, `majority` read concern for committed data, and restrict secondary reads to staleness-tolerant workloads such as reports. They do not reduce write load.

**41. [Senior] How would you perform a zero-downtime schema change on a large collection?**
Add `schemaVersion`. Deploy code that reads both shapes and writes the new one, backfill in throttled batches with a resumable cursor, verify with counts and queries, add new indexes during a quiet period, then deploy code removing the old path and clean up. Use validation in `warn` mode before switching to `error`.

**42. [Senior] How do you build a reliable event pipeline from MongoDB changes?**
Either an outbox collection written atomically with the business document (single document or transaction) and processed by a worker, or a change stream with a persisted resume token. Handlers must be idempotent because delivery is at least once. Watch the oplog window so a slow consumer can still resume, and run one active consumer per stream.

**43. [Senior] What would make you choose sharding, and what would you try first?**
Sharding is for data volume or write throughput beyond a single replica set. First fix queries and indexes, remodel to shrink documents and the working set, archive or TTL old data, scale vertically, and add secondaries for tolerant reads. Sharding adds shard key risk and operational complexity.

**44. [Senior] Explain how you would secure a MongoDB deployment end to end.**
Private network only, `bindIp` restricted, access control on, TLS, per-application least-privilege users, secrets in a secret manager, JavaScript disabled when unused, encryption at rest and optionally client-side field encryption for sensitive data, input validation to prevent operator injection and mass assignment, patching, audit logs, and encrypted, tested backups.

**45. [Senior] When would you pick PostgreSQL over MongoDB?**
For highly relational data with many ad hoc joins, strict cross-entity integrity (foreign keys, constraints), heavy multi-row transactional logic such as ledgers, and SQL-based analytics. MongoDB is stronger for aggregate-shaped, variable, high-volume data and horizontal scale, but the right answer depends on access patterns and team skill, and a hybrid can be correct.

**46. [Senior] A deploy caused connection counts to spike and the database to slow. What do you check?**
Whether a `MongoClient` is created per request or per function invocation, the new `maxPoolSize` times instance count, an autoscaling event multiplying instances, slow queries holding connections, and missing timeouts. Compare `serverStatus().connections` with app-side pool metrics, then fix client lifecycle and pool sizing.

**47. [Senior] How do you design for graceful behaviour during a primary election?**
Replica-set-aware connection string, `retryWrites` and `retryReads`, bounded `serverSelectionTimeoutMS` and operation timeouts, idempotent writes, a queue or circuit breaker for non-critical writes, and alerts on elections. Test it by stepping down the primary under load.

**48. [Senior] Describe a safe approach to adding an index to a very large live collection.**
Verify the need with `explain` and the profiler, estimate size and RAM impact, schedule a quiet period, build with the platform's supported low-impact method (rolling builds on replica sets where applicable, confirm for your version), monitor I/O, replication lag and `currentOp`, then validate the plan and remove the old index after hiding it first.

---

## 24. References

**In plain words.** These are official sources. When this guide and the manual differ, follow the manual for your server version.

Primary sources, preferred over blog posts. URLs were checked on 2026-11-06, except the top-level pages for indexes, aggregation, sharding, BSON types, `mongosh`, the Node.js driver root and Atlas, which are unchecked. Pages move and the manual is versioned.

- MongoDB Manual (select the version you run): https://www.mongodb.com/docs/manual/
- Data modelling: https://www.mongodb.com/docs/manual/data-modeling/
- Schema design patterns: https://www.mongodb.com/docs/manual/data-modeling/design-patterns/
- Indexes (ESR guidance is under the indexing strategies pages): https://www.mongodb.com/docs/manual/indexes/
- Interpret explain results: https://www.mongodb.com/docs/manual/reference/explain-results/
- Aggregation pipeline and stages: https://www.mongodb.com/docs/manual/aggregation/
- Transactions: https://www.mongodb.com/docs/manual/core/transactions/
- Replication: https://www.mongodb.com/docs/manual/replication/
- Read concern, write concern and read preference: https://www.mongodb.com/docs/manual/reference/read-concern/ and https://www.mongodb.com/docs/manual/reference/write-concern/ and https://www.mongodb.com/docs/manual/core/read-preference/
- Sharding and shard keys: https://www.mongodb.com/docs/manual/sharding/
- Change streams: https://www.mongodb.com/docs/manual/changeStreams/
- Security checklist and authentication: https://www.mongodb.com/docs/manual/administration/security-checklist/
- Backup methods (self-managed): https://www.mongodb.com/docs/manual/core/backups/
- Database profiler: https://www.mongodb.com/docs/manual/tutorial/manage-the-database-profiler/
- MongoDB Limits and Thresholds: https://www.mongodb.com/docs/manual/reference/limits/
- BSON types: https://www.mongodb.com/docs/manual/reference/bson-types/
- Database tools (`mongodump`, `mongorestore`, `mongostat`): https://www.mongodb.com/docs/database-tools/
- MongoDB Shell (`mongosh`): https://www.mongodb.com/docs/mongodb-shell/
- Node.js driver documentation: https://www.mongodb.com/docs/drivers/node/current/
- MongoDB Atlas documentation: https://www.mongodb.com/docs/atlas/
- Mongoose documentation: https://mongoosejs.com/docs/
- Building with Patterns (MongoDB blog series on schema patterns) and the MongoDB University courses: https://learn.mongodb.com/
- Kleppmann, *Designing Data-Intensive Applications* (book), for replication, partitioning and consistency background.

---

## 25. Glossary

**In plain words.** These are the hardest terms in this guide, with simple meanings. Each term is also explained where it first appears.

| Term | Plain meaning |
|---|---|
| Atomic | A change that applies fully or not at all. |
| BSON | The binary format MongoDB uses to store documents. It has more types than JSON. |
| Collection scan | Reading every document in a collection to answer a query. |
| Compound index | An index on several fields, in a fixed order. |
| Covered query | A query answered from the index alone, without reading documents. |
| Election | The vote in a replica set that chooses a new primary. |
| ESR rule | Order index fields as Equality, Sort, Range. |
| Idempotent | Safe to repeat: running it twice gives the same result as once. |
| Journal | A log written before the data, used to recover after a crash. |
| Jumbo chunk | A chunk of shard data that is too big to split or move. |
| Multikey index | An index on an array field, with one entry for each element. |
| Oplog | The capped log of writes that secondaries copy and change streams read. |
| Pipeline | An ordered list of stages that transform documents in an aggregation. |
| Primary | The replica set member that accepts writes. |
| Read concern | The setting for what data a read may return. |
| Read preference | The setting for which member serves a read. |
| Resume token | A marker in a change stream that lets you continue after a stop. |
| Rollback | Removal of writes from a former primary that never reached a majority. |
| Scatter-gather | A query sent to all shards because it has no shard key. |
| Secondary | A replica set member that copies the primary. |
| Shard | One replica set that holds part of a sharded collection. |
| Shard key | The field that decides which shard stores a document. |
| TTL index | An index that deletes documents after a set time. |
| Working set | The data and indexes in regular use, which should fit in RAM. |
| Write concern | The setting for how many members must confirm a write. |
| Write conflict | Two transactions change the same document, so one must retry. |
