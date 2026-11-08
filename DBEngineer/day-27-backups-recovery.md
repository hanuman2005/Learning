# Day 27 — Backups & Recovery

> Series: Database Engineering
> **Goal:** Master the safety net every DB engineer owns: **logical** backups (`pg_dump`) vs **physical** backups (`pg_basebackup`), **point-in-time recovery** (PITR) via WAL archiving, the **RPO/RTO** vocabulary that drives every design choice, and the discipline that separates pros from amateurs — **testing your restores**. Lab: back up, corrupt, and restore to a precise moment before the damage.

## Table of Contents
1. [In plain words](#in-plain-words)
2. [RPO and RTO: the two numbers that decide everything](#rpo-and-rto-the-two-numbers-that-decide-everything)
3. [Logical backups: pg_dump](#logical-backups-pg_dump)
4. [Physical backups: pg_basebackup](#physical-backups-pg_basebackup)
5. [Point-in-time recovery with WAL archiving](#point-in-time-recovery-with-wal-archiving)
6. [Disaster-recovery drills & testing restores](#disaster-recovery-drills--testing-restores)
7. [Backup tooling in the real world](#backup-tooling-in-the-real-world)
8. [Lab: back up, corrupt, restore to a point in time](#lab-back-up-corrupt-restore-to-a-point-in-time)
9. [Self-Check](#self-check)
10. [Tomorrow](#tomorrow)

---

## In plain words

**In plain words:** A backup is a *time machine* for your data. There are two kinds of time machine. A **logical** backup (`pg_dump`) is like writing down the recipe to *rebuild* the meal from scratch — a script of `CREATE TABLE` + `INSERT` that recreates the data logically; portable and precise, but slow to cook for a huge kitchen. A **physical** backup (`pg_basebackup`) is like taking a *photograph of the entire kitchen* — every pot, every shelf, byte-for-byte — fast to snap and restore, but only usable in an identical kitchen. And **PITR** (point-in-time recovery) is the killer feature: take a photo, then keep a *continuous journal of every change since* (the WAL), so you can rewind to **9:59:58am** — one second before someone ran `DELETE FROM orders;` — and stop exactly there.

The rule that separates engineers from victims: **a backup you have never restored is not a backup — it is a hope.** (Day 20's warning bears repeating: replicas are *not* backups; they replicate your mistakes instantly.)

**Verify versions when building:** **PostgreSQL 17**. Tools: `pg_dump`/`pg_dumpall`/`pg_restore` (logical), `pg_basebackup` (physical), WAL archiving + `recovery_target_time` for PITR (config in `postgresql.conf` + `recovery.signal` since PG 12). Production tools: **pgBackRest**, **Barman**, **WAL-G**. Cloud snapshots (RDS/Cloud SQL) wrap these.

> Cross-ref: **Day 11 (WAL)** — PITR is WAL replay to a target; you already know the Write-Ahead Log. **Day 20 (Replication)** — replicas ≠ backups. **Day 21 (HA)** — HA protects against *node* failure; backups protect against *logical/human* error. Two different threats, two different tools.

---

## RPO and RTO: the two numbers that decide everything

**In plain words:** Before choosing a backup strategy, answer two questions the business owns:
- **RPO — Recovery Point Objective:** how much *data* can you afford to lose? ("At most 5 minutes of transactions.") Drives *how often/continuously* you back up.
- **RTO — Recovery Time Objective:** how long can you be *down* while recovering? ("Back up within 1 hour.") Drives *how fast* your restore path is.

```mermaid
flowchart LR
    B["last recoverable point"] -->|"RPO = data lost<br/>(gap before the incident)"| I["INCIDENT"]
    I -->|"RTO = downtime<br/>(time to be back up)"| R["service restored"]
```
*RPO is how much data you lose (looking back); RTO is how long you're down (looking forward).*

| | RPO (data loss tolerance) | RTO (downtime tolerance) |
|---|---|---|
| Question | "How much can we lose?" | "How fast must we recover?" |
| Driven by | backup frequency / WAL archiving / sync replication | restore speed / automation / warm standby |
| RPO=0 needs | synchronous replication (Day 21) | — |
| Small RTO needs | — | fast physical restore / hot standby / cloud snapshot |

- **Cheap combos:** nightly `pg_dump` = RPO up to 24h, RTO hours. Fine for a low-value internal tool.
- **Serious combos:** physical base backup + continuous WAL archiving = RPO seconds/minutes (PITR), RTO minutes-to-an-hour. The standard for real products.
- **RPO 0:** requires synchronous replication *in addition* to backups (Day 21) — backups alone always have a gap.

> 🤔 **"We take nightly backups, so we're safe."**
> Nightly backups mean your **RPO is up to 24 hours** — a failure at 5pm loses a whole day of transactions since the 2am dump. That may be catastrophic (an e-commerce day of orders) or fine (a static reference DB). "Safe" is meaningless without stating the RPO/RTO the business accepts. Set the numbers first, *then* design the backup strategy to meet them. Continuous WAL archiving (PITR) is what turns "24-hour RPO" into "5-minute RPO."

> ❌ **Bad scenario:** A company assumed "we have backups" without ever stating RPO/RTO. A corruption at 4pm forced a restore from the 1am dump — 15 hours of orders gone — and the restore itself took 6 hours because nobody had timed it. Both numbers were unknown until the worst possible moment.

**When NOT to:** don't engineer RPO=0 / RTO=minutes for data that doesn't warrant it — continuous archiving, warm standbys, and drills cost money and effort. Match the investment to the data's value.

---

## Logical backups: pg_dump

**In plain words:** `pg_dump` exports a database as a set of SQL commands (or a compressed archive) that *recreates* it: schema + data as `CREATE`/`COPY`/`INSERT`. It's a logical snapshot of one database at one consistent instant (it uses MVCC — Day 14 — so it's a consistent point-in-time view without blocking writers).

- **Why:** portable (restore to a different PG version or even reshape), selective (one table/schema), human-readable, cross-platform.
- **What:** per-database dump; `pg_dumpall` adds cluster-wide objects (roles, tablespaces).
- **When:** migrations, upgrades, moving data, small/medium DBs, grabbing one table, seeding dev.
- **How:** `pg_dump` formats — plain SQL, custom (`-Fc`, compressed + selective restore), directory (`-Fd`, parallel).
- **Where:** dev/test refreshes, version upgrades, exporting a subset.

```bash
# Custom-format dump (compressed, allows selective + parallel restore) — the usual choice
pg_dump -Fc -f app.dump -d app                 # -Fc = custom format
# Directory format with parallel dump (fast for big DBs):
pg_dump -Fd -j 4 -f app_dir -d app             # 4 parallel workers

# Just one table, or just the schema:
pg_dump -Fc -t orders -f orders.dump -d app    # single table
pg_dump --schema-only -f schema.sql -d app     # DDL only (great for diffs/reviews)

# Cluster-wide (roles, grants, all DBs):
pg_dumpall -f whole_cluster.sql

# Restore (pg_restore for custom/dir formats; psql for plain SQL):
createdb app_restored
pg_restore -d app_restored -j 4 app.dump       # parallel restore
psql -d app_restored -f schema.sql             # plain SQL restore
```

| | pg_dump (logical) |
|---|---|
| Granularity | database / schema / table |
| Cross-version | ✔ (restore to newer PG) |
| PITR to a second | ✗ (no — it's one instant) |
| Speed on huge DBs | slow (row-by-row) |
| Restore rebuilds indexes | yes (can be slow) |

> 🤔 **"pg_dump is my backup strategy for our 4TB production database."**
> `pg_dump` is a *logical* export — it reads every row and rebuilds every index on restore. On a 4TB database that can take *many hours to a day* to dump and even longer to restore, and it gives you only the instant you started (no PITR). For large production databases you want a **physical** backup + WAL archiving (PITR), with `pg_dump` reserved for migrations, per-table exports, and small DBs. Right tool, right size.

> ✅ **Good scenario:** Upgrading Postgres 14 → 17 for a 20GB DB: `pg_dump -Fc` on 14, `pg_restore` on 17. Logical portability across versions is exactly `pg_dump`'s superpower.

**When NOT to:** don't rely on `pg_dump` alone for large DBs (slow, no PITR) or for RPO smaller than your dump interval.

---

## Physical backups: pg_basebackup

**In plain words:** `pg_basebackup` copies the *entire data directory* byte-for-byte (all files, all databases) — a physical clone of the cluster at a moment. It's fast (sequential file copy) and restores fast (just start Postgres on the files), but it's version- and architecture-specific and it's all-or-nothing (whole cluster, not one table). It's the *base* on which PITR is built.

- **Why:** fast for large DBs, restore = copy files + start, and it's the foundation for WAL-based PITR.
- **What:** a copy of `$PGDATA` (+ a starting WAL position).
- **When:** large production DBs, setting up standbys (Day 21), the base for continuous archiving.
- **Where:** production DR strategies, replica provisioning.

```bash
# A physical base backup (compressed tar, includes required WAL to be consistent)
pg_basebackup -h primary -U repl -D /backups/base_2026-07-26 \
  -Ft -z -Xs -P
#   -Ft  tar format     -z gzip     -Xs stream WAL alongside     -P progress
# Restore = extract the base into an empty $PGDATA and start Postgres:
#   tar xzf base.tar.gz -C /var/lib/postgresql/data && pg_ctl start
```

| | Logical (pg_dump) | Physical (pg_basebackup) |
|---|---|---|
| Unit | database/table | whole cluster (byte-for-byte) |
| Speed (big DB) | slow | fast |
| Cross-version | ✔ | ✗ (must match major version) |
| Selective (one table) | ✔ | ✗ |
| Basis for PITR | ✗ | ✔ (base + WAL) |
| Restore | rebuilds objects/indexes | copy files + start |

> ✅ **Good scenario:** A 2TB production DB uses a weekly `pg_basebackup` + continuous WAL archiving. A full restore is "extract the base, replay WAL" — an hour, not the day a `pg_dump` restore would take.

> 🤔 **"A physical backup lets me restore a single accidentally-dropped table."**
> No — physical backups are all-or-nothing (the whole cluster). To recover *one* table you either (a) restore the whole physical backup to a *separate* instance and `pg_dump` that one table out of it, or (b) keep logical dumps for granular restores. Physical is for whole-cluster/DR recovery; logical is for surgical, per-object recovery. Serious shops keep both.

**When NOT to:** don't use physical backups when you need cross-version portability or single-table restores — that's logical's job.

---

## Point-in-time recovery with WAL archiving

**In plain words:** This is the crown jewel. A base backup is a photo at 2am. But you need to recover to 3:47:12pm (just before disaster). The trick: after the base backup, **archive every WAL segment** (the change journal, Day 11) as it's produced. To recover, you restore the base and then **replay the WAL forward** — and you can tell Postgres to *stop* at an exact timestamp, transaction ID, or named restore point. That's PITR: base + continuous WAL = rewind to any second in the window.

```mermaid
flowchart LR
    Base["base backup (2:00am)"] --> W1["WAL 2:00-3:00"]
    W1 --> W2["WAL 3:00-3:47"]
    W2 -->|"replay stops here"| T["recovery_target_time = 3:47:11pm"]
    T --> X["DISASTER at 3:47:12pm (not replayed)"]
```
*PITR: restore the base, replay archived WAL forward, and stop one moment before the disaster.*

**Setup (on the primary):**
```bash
# postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'   # copy each WAL segment to safe storage
#   %p = path to the WAL file to archive, %f = its filename
#   (Production: ship to S3/GCS via pgBackRest/WAL-G, not a local cp.)
```

**Recovery (on a fresh instance):**
```bash
# 1) Restore the base backup into an empty $PGDATA
tar xzf /backups/base_2026-07-26.tar.gz -C /var/lib/postgresql/data

# 2) Tell Postgres where to get archived WAL and WHEN to stop
cat >> /var/lib/postgresql/data/postgresql.conf <<'EOF'
restore_command = 'cp /archive/%f %p'                 # fetch archived WAL segments
recovery_target_time = '2026-07-26 15:47:11+00'       # STOP one second before disaster
recovery_target_action = 'promote'                    # become a normal primary when target reached
EOF

# 3) Signal recovery mode and start — Postgres replays WAL up to the target, then promotes
touch /var/lib/postgresql/data/recovery.signal
pg_ctl -D /var/lib/postgresql/data start
# Logs: "recovery stopping before ... target time" then "database system is ready"
```

Recovery targets you can stop at:

| Target | Set with | Use |
|---|---|---|
| A timestamp | `recovery_target_time` | "just before 3:47pm" |
| A transaction id | `recovery_target_xid` | stop right before a known bad txn |
| A named point | `recovery_target_name` (created via `pg_create_restore_point`) | "before_migration_v42" |
| The LSN | `recovery_target_lsn` | precise WAL position |
| End of WAL | (default) | recover everything available |

> ✅ **Good scenario:** At 3:47pm someone runs `DELETE FROM orders;` with no `WHERE`. With PITR you restore the 2am base and replay WAL to `recovery_target_time = '15:46:59'` — the orders are back, losing only the ~1 minute of legitimate activity after that instant. Without PITR, you'd lose everything since the 2am dump.

> 🤔 **"PITR means I never lose any data — I can always rewind."**
> PITR can rewind only within the window you *have* archived WAL for, and only up to the *last archived* segment. If archiving lags or fails silently, your effective RPO is worse than you think. And PITR replays forward to a target — it can't cherry-pick "undo only that one statement" while keeping later good writes (recovery stops at the target; everything after is discarded, unless you extract from a side restore). Monitor archiving, and know PITR's shape: it rewinds the *whole cluster* to a point.

> **Dark Side:** A silently-failing `archive_command` (disk full, S3 creds expired) means WAL isn't being archived — your PITR window quietly stops advancing, and `pg_wal` on the primary may fill up (Day 11) and *halt the database*. Monitor `pg_stat_archiver` (`failed_count`, `last_archived_time`) and archive storage. An unmonitored archive is a false sense of security.

**When NOT to:** don't skip WAL archiving if your RPO is smaller than your base-backup interval — periodic base backups alone leave you at the mercy of "how old is the last full backup."

---

## Disaster-recovery drills & testing restores

**In plain words:** The single highest-leverage backup practice is **regularly restoring your backups into a scratch environment and verifying the data**. Backups fail silently — corrupt archives, missing WAL, a dump that errored, credentials that rotated. You find out either during a scheduled drill (calm, Tuesday afternoon) or during a real disaster (panic, 3am, customers down). Choose the drill.

**A restore drill checklist:**
1. Restore the latest base + WAL to an *isolated* instance (never touch prod).
2. Recover to a target time; confirm the DB starts and is consistent.
3. Run data-integrity checks: row counts, checksums on critical tables, app smoke tests.
4. **Time it** — that measured duration *is* your real RTO (not a guess).
5. Verify RPO — how far back is your latest recoverable point right now?
6. Record results; fix anything broken *before* you need it.

```bash
# Automate a weekly restore verification (sketch)
LATEST=$(ls -t /backups/base_* | head -1)
tar xzf "$LATEST" -C /tmp/restore_test
# ... configure restore_command + a recovery_target_time ...
pg_ctl -D /tmp/restore_test start
psql -p 5599 -c "SELECT count(*) FROM orders;" >> /var/log/restore_drill.log
psql -p 5599 -c "SELECT max(created_at) FROM orders;"   # how fresh? -> effective RPO
pg_ctl -D /tmp/restore_test stop && rm -rf /tmp/restore_test
```

> ❌ **Bad scenario (the classic):** A startup ran nightly `pg_dump` for two years, never tested a restore. When they finally needed it, the dumps had been silently failing for *six months* (a permissions change broke the cron job's DB access) — every recent backup was a 0-byte file. Two years of "backups," zero recoverable data. An untested backup is a placebo.

> ✅ **Good scenario:** A team runs an automated weekly restore-and-verify. One week it flagged that WAL archiving had stalled (a rotated S3 key). They fixed it in an afternoon — a non-event. Had they only discovered it during a real incident, it would have been a data-loss catastrophe.

> 🤔 **"Backups are set up and the job reports success — we're covered."**
> "Job reports success" and "we can actually recover a working database" are different claims. Backups can complete yet be unrestorable (corruption, missing WAL, incompatible version, incomplete dump). The *only* proof of a working backup is a *successful restore*. Test restores on a schedule; a green cron job is necessary but not sufficient.

**When NOT to:** there is no "when not to test restores." If you keep backups at all, you must periodically prove they restore. Untested backups are the most expensive false comfort in operations.

---

## Backup tooling in the real world

| Tool | What it gives you |
|---|---|
| **pgBackRest** | full/incremental/differential physical backups, parallel, compression, S3/GCS/Azure, PITR, retention, verification — the production standard |
| **Barman** | backup + WAL management + PITR, catalog of backups, recovery orchestration |
| **WAL-G** | efficient WAL + base backup archiving to object storage (delta backups) |
| Cloud snapshots (RDS/Cloud SQL/Aurora) | managed automated backups + PITR behind a console/API |

Beyond one tool, professionals follow the **3-2-1 rule**: **3** copies of data, on **2** different media/systems, with **1** copy off-site (a different region/provider) — so a single datacenter loss, ransomware, or region outage can't take out both your data *and* its backups.

> **When NOT to:** don't store backups in the same place/account as production. A region outage, an account compromise, or accidental deletion could destroy both. Off-site (and immutable/versioned where possible) is the point of the "1" in 3-2-1.

---

## Lab: back up, corrupt, restore to a point in time

The full PITR drill on a local PG 17. You will simulate the classic "someone deleted the table" and recover to the second before it.

```bash
# ===== 0) Enable WAL archiving on your local instance =====
mkdir -p /tmp/wal_archive
cat >> "$PGDATA/postgresql.conf" <<'EOF'
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f'
EOF
pg_ctl -D "$PGDATA" restart
```

```sql
-- ===== 1) Create data and take a NAMED restore point BEFORE any damage =====
CREATE TABLE orders (id serial primary key, amt numeric, created_at timestamptz default now());
INSERT INTO orders (amt) SELECT (random()*100)::numeric FROM generate_series(1,1000);
SELECT count(*) FROM orders;             -- 1000
SELECT pg_create_restore_point('before_disaster');   -- a named target we can rewind to
```

```bash
# ===== 2) Take a base backup (the "photo") AFTER the data exists =====
pg_basebackup -D /tmp/base_backup -Ft -z -Xs -P
```

```sql
-- ===== 3) Do MORE legitimate work, then note the exact time, then DISASTER =====
INSERT INTO orders (amt) SELECT (random()*100)::numeric FROM generate_series(1,50);  -- 1050 total
SELECT now();                            -- <-- record this timestamp T_safe (e.g. 15:46:59)
-- ... a moment later, the catastrophe:
DELETE FROM orders;                      -- 🔥 all 1050 rows gone
SELECT count(*) FROM orders;             -- 0  (disaster confirmed)
```

```bash
# ===== 4) RESTORE to a scratch instance, stopping just before the DELETE =====
rm -rf /tmp/restore && mkdir /tmp/restore
tar xzf /tmp/base_backup/base.tar.gz -C /tmp/restore
tar xzf /tmp/base_backup/pg_wal.tar.gz -C /tmp/restore/pg_wal 2>/dev/null || true

cat >> /tmp/restore/postgresql.conf <<'EOF'
restore_command = 'cp /tmp/wal_archive/%f %p'
recovery_target_time = '2026-07-26 15:46:59+00'    # <-- your T_safe: BEFORE the DELETE
recovery_target_action = 'promote'
port = 5599
EOF
touch /tmp/restore/recovery.signal
pg_ctl -D /tmp/restore start
# Watch the log: "recovery stopping before commit ... " then "ready to accept connections"
```

```sql
-- ===== 5) VERIFY recovery on the restored instance (:5599) =====
-- Connect: psql -p 5599
SELECT count(*) FROM orders;        -- 1050  (the DELETE was NEVER replayed — data is back!)
SELECT max(created_at) FROM orders; -- confirm freshness up to T_safe
-- You rewound the cluster to one second before the disaster. RPO here = the gap
-- between T_safe and the DELETE (seconds). RTO = how long steps 4-5 took (time it!).
```

```bash
# ===== 6) Alternative target: stop at the NAMED restore point instead of a time =====
#   recovery_target_name = 'before_disaster'    # (instead of recovery_target_time)
#   This rewinds to exactly when you called pg_create_restore_point() — before the +50 inserts.

# ===== 7) Cleanup =====
pg_ctl -D /tmp/restore stop && rm -rf /tmp/restore /tmp/base_backup
```

> **Reflection:** You just did the operation that saves companies: a base backup + archived WAL let you rewind past a destructive mistake to a precise second, losing only what happened after that instant. Now internalize the two numbers you measured — the data gap (RPO) and the restore duration (RTO) — because those, not the existence of a backup file, are what "are we safe?" actually means.

---

## Self-Check
1. Define RPO and RTO. Which does WAL archiving improve, and which does a warm standby improve?
2. Contrast logical (`pg_dump`) and physical (`pg_basebackup`) backups on granularity, speed, cross-version, and PITR support.
3. Why is `pg_dump` a poor sole strategy for a multi-terabyte production database?
4. How do you recover a single dropped table if you only have physical backups?
5. Explain how PITR works: what are the two ingredients and how do they combine?
6. Name three recovery targets you can stop replay at.
7. Why can a silently-failing `archive_command` both ruin your RPO *and* halt the primary?
8. Why is "the backup job reports success" insufficient proof of safety?
9. State the 3-2-1 rule and the threat each part defends against.
10. Why are replicas (Day 20) not a substitute for backups? What threat do backups uniquely cover?

## Tomorrow
**Day 28 — Zero-Downtime Migrations.** The scariest routine skill in the job: changing the schema of a live, large, busy database without taking it down or locking out users. Tomorrow: the expand/contract pattern, `CREATE INDEX CONCURRENTLY`, adding `NOT NULL` safely in steps, batched backfills, online schema-change tools (gh-ost, pt-online-schema-change), and the lock-timeout danger. Lab: add a `NOT NULL` column to a huge table with zero downtime.
