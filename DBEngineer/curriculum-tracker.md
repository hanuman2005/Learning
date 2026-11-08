# 🗄️ Database Engineering — 30-Day Curriculum Tracker

> **Goal:** Go from *using* a database to *engineering* one — schemas, internals, query plans, concurrency, scaling, and safe production operations. Product-company grade for backend/DB engineers.
> **Primary DB:** PostgreSQL (the industry default), with MySQL differences and NoSQL systems where relevant.
> **Rule:** 3 hrs/day = ~1.5h learn + ~1.5h hands-on (run real queries, read real plans).
> **Teaching style (same as your other courses):** In plain words → Why/What/When/How/Where → 🤔 misconception hook → ✅ good / ❌ bad scenario → When NOT to.
> **Sufficiency:** This core = 100% for backend, ~90–95% for DB app/infra roles. Kernel-engineer topics live in the optional Advanced Track at the end.

**Progress: S1 ⬜ · S2 ⬜ · S3 ⬜ · S4 ⬜ · S5 ⬜ · S6 ⬜ · S7 ⬜ · S8 ⬜ · Advanced ⬜**

---

## Stage 1 — Relational Foundations (Days 1–5)
### Day 1 — Why Databases; RDBMS; OLTP vs OLAP
- [ ] Why DBs exist (vs files); the relational model; OLTP vs OLAP
- [ ] ACID at a glance; the DB engineer's mindset
- [ ] **Lab:** set up PostgreSQL, load a sample schema

### Day 2 — Data Modeling & Keys
- [ ] ER diagrams; entities/relationships/cardinality
- [ ] Keys: primary, foreign, composite, surrogate vs natural
- [ ] **Lab:** model an e-commerce domain end to end

### Day 3 — Normalization & Denormalization
- [ ] 1NF → 2NF → 3NF → BCNF (with the anomaly each fixes)
- [ ] Deliberate denormalization and its trade-offs
- [ ] **Lab:** normalize a messy table, then justify one denormalization

### Day 4 — SQL Core
- [ ] DDL/DML; all join types; filtering; aggregates; GROUP BY/HAVING
- [ ] **Lab:** 20-query drill against the sample DB

### Day 5 — SQL Advanced
- [ ] Subqueries, CTEs, window functions, set operations
- [ ] **Lab:** rewrite slow subqueries as window functions

---

## Stage 2 — Database Internals (Days 6–10) ⭐ *what makes you an "engineer"*
### Day 6 — Storage Layout
- [ ] Pages, rows, heap vs index-organized tables, TOAST, fill factor
- [ ] **Lab:** inspect page/tuple layout with pageinspect

### Day 7 — Indexes Deep
- [ ] B-tree, hash, GIN/GiST/BRIN; composite, covering, partial; index-only scans
- [ ] **Lab:** build each index type, measure the difference

### Day 8 — When Indexes Hurt
- [ ] Write amplification, bloat, wrong index chosen; maintenance
- [ ] **Lab:** show a workload where an index slows things down

### Day 9 — Query Execution & the Planner
- [ ] Parser → planner/optimizer → executor; scan/join/sort nodes
- [ ] Reading EXPLAIN / EXPLAIN ANALYZE
- [ ] **Lab:** diagnose 3 query plans

### Day 10 — Buffer Pool, WAL & MVCC intro
- [ ] Shared buffers, WAL, checkpoints; MVCC overview
- [ ] **Lab:** watch WAL + checkpoint behavior under load

---

## Stage 3 — Transactions & Concurrency (Days 11–14)
### Day 11 — ACID In Depth
- [ ] Atomicity/consistency/isolation/durability with real failure cases
- [ ] **Lab:** force a rollback mid-transaction, observe

### Day 12 — Isolation Levels & Anomalies
- [ ] Read uncommitted → serializable; dirty/non-repeatable/phantom reads
- [ ] **Lab:** reproduce each anomaly with two sessions

### Day 13 — Locking & Deadlocks
- [ ] Shared/exclusive, row vs table locks; deadlock detection & avoidance
- [ ] **Lab:** create and then resolve a deadlock

### Day 14 — MVCC Deep & VACUUM
- [ ] Tuple versions, snapshots, optimistic vs pessimistic; VACUUM/autovacuum, bloat
- [ ] **Lab:** cause bloat, then VACUUM it away

---

## Stage 4 — Performance Engineering (Days 15–19)
### Day 15 — Query Optimization
- [ ] Reading plans deeply; fixing slow queries; sargability
- [ ] **Lab:** take a 10s query under 100ms

### Day 16 — Index Tuning & N+1
- [ ] Index strategy; the N+1 problem; connection pooling (PgBouncer)
- [ ] **Lab:** find & kill an N+1 from an ORM

### Day 17 — Partitioning & Maintenance
- [ ] Range/list/hash partitioning; bloat, REINDEX, ANALYZE, statistics
- [ ] **Lab:** partition a large time-series table

### Day 18 — Caching Layers
- [ ] App cache, Redis, materialized views; invalidation (reuse System Design Day 11)
- [ ] **Lab:** add a materialized view + refresh strategy

### Day 19 — Benchmarking
- [ ] pgbench/load testing; finding the real bottleneck (CPU/IO/lock)
- [ ] **Lab:** benchmark before/after a tuning change

---

## Stage 5 — Scaling & Distribution (Days 20–23)
### Day 20 — Replication
- [ ] Sync vs async; primary-replica; streaming; replication lag
- [ ] **Lab:** set up a read replica, measure lag

### Day 21 — Read/Write Scaling & HA
- [ ] Read replicas, write scaling limits, failover, HA topologies
- [ ] **Lab:** simulate a failover

### Day 22 — Sharding
- [ ] Sharding strategies; choosing a shard key; cross-shard queries/pain (reuse System Design Day 13)
- [ ] **Lab:** shard a table by key, run a cross-shard query

### Day 23 — CAP & Consistency
- [ ] CAP, PACELC, consistency models; quorum reads/writes (reuse System Design Day 14)
- [ ] **Lab:** reason through a partition scenario

---

## Stage 6 — NoSQL & Polyglot Persistence (Days 24–26)
### Day 24 — Document & Key-Value
- [ ] MongoDB (document modeling), Redis/DynamoDB (key-value); access-pattern-first modeling
- [ ] **Lab:** model the same domain in Postgres vs MongoDB

### Day 25 — Wide-Column, Graph, NewSQL
- [ ] Cassandra (wide-column), Neo4j (graph), CockroachDB/Spanner (NewSQL)
- [ ] **Lab:** design a Cassandra table for a query pattern

### Day 26 — Choosing the Right Store + Vector DBs
- [ ] Polyglot persistence; decision framework; vector DBs (ties to your AI course)
- [ ] **Lab:** justify DB choices for 5 workloads

---

## Stage 7 — Operations / DBRE (Days 27–29)
### Day 27 — Backups & Recovery
- [ ] Logical vs physical backups; PITR; disaster-recovery drills; RPO/RTO
- [ ] **Lab:** back up, corrupt, restore to a point in time

### Day 28 — Zero-Downtime Migrations ⭐ *the skill that scares everyone*
- [ ] Expand/contract pattern; online schema change; backfills; migration tools (Flyway/Liquibase, gh-ost)
- [ ] **Lab:** add a NOT NULL column to a big table with no downtime

### Day 29 — Security & Observability
- [ ] Access control/roles, encryption at rest/in transit, SQL injection defense
- [ ] Monitoring (pg_stat_*, slow-query log), key metrics & alerts
- [ ] **Lab:** lock down a role to least privilege + set a slow-query alert

---

## Stage 8 — Advanced & Capstone (Day 30)
### Day 30 — Warehousing, CDC & Capstone
- [ ] OLAP vs OLTP, columnar stores, data warehouses (Snowflake/BigQuery/Redshift)
- [ ] CDC & streaming (Debezium/Kafka), time-series DBs
- [ ] **Capstone:** design + tune a schema end-to-end
- [ ] **Interview prep:** "design a schema for X", "this query is slow — fix it", "migrate a 500M-row table with zero downtime"

---

## 🎓 Advanced Track — Database Systems Engineering (OPTIONAL / specialist)
> **Read only if** you want to build databases (Postgres/CockroachDB/Snowflake internals) or become a DB *kernel* engineer. NOT needed for backend or DB-application/infra roles. Skip unless this is your goal — per the "don't rabbit-hole on the rarely-used" rule.

- [ ] **A1 — Engine architecture:** parser → planner → executor, buffer manager, lock manager, WAL internals, checkpoint algorithm, storage manager
- [ ] **A2 — Distributed algorithms:** Raft, Paxos, leader election, quorum, consensus, distributed transactions, 2PC/3PC, Saga
- [ ] **A3 — Storage engine design:** LSM trees, SSTables, MemTables, Bloom filters, B+ tree implementation, compaction, write/read amplification (Cassandra/RocksDB/ScyllaDB)
- [ ] **A4 — DB ↔ OS interaction:** mmap(), fsync(), page cache, NUMA, huge pages, SSD vs HDD optimization
- [ ] **A5 — Memory management:** shared buffers, buffer replacement (LRU, clock sweep), memory contexts, temp memory
- [ ] **A6 — DB networking:** Postgres wire protocol, connection multiplexing, PgBouncer internals, TCP optimization
- [ ] **A7 — Advanced optimizer theory:** cost estimation, cardinality estimation, join ordering, predicate pushdown, statistics/histograms, adaptive query optimization
- [ ] **A8 — Multi-region databases:** global replication, clock sync, TrueTime (Spanner), multi-master, conflict resolution
- [ ] **A9 — Building a database:** implement a SQL parser, planner, execution engine, index manager, storage layer, transaction manager

*If/when you want depth here, tell me and I'll build out these as full files too.*

---

## 📁 File Index

| Day | File | Studied? |
|---|---|---|
| — | `curriculum-tracker.md` | (this file) |
| 1–30 | *day files created in this batch* | ⬜ |
| Advanced | `advanced-track-db-systems.md` | ⬜ |

---

## Note on overlap with System Design
Indexing, replication, sharding, CAP appear in both courses — but here at the *database engineer's* depth (query plans, MVCC, VACUUM, migration mechanics), not the architect's altitude. Complementary. Move fast through anything you already know from System Design.

## Daily Ritual
- [ ] Every day ends with real queries run + a plan read — DB skill is hands-on.
- [ ] Explain the day's concept out loud in 2 min (Feynman test).
- [ ] Commit lab SQL to git: `git commit -m "db-day XX: <topic>"`.
