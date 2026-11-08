# Section O — Gaps Supplement (2/3): Concurrency & Correctness

> The bugs that quietly corrupt data under load. Directly relevant to your **loyalty ledger (N5)**,
> **inventory**, **pricing (N6)**, and **banking (N3)**.

---

## O7 — Optimistic vs pessimistic locking (the lost-update problem)

**Diagram: lost update**
```
 balance 100
 A reads 100 --------------> writes 70
 B reads 100 ---> writes 50      (A's -30 lost, should be 20)

 fix 1 PESSIMISTIC: BEGIN; SELECT ... FOR UPDATE   B waits for A, reads 70
 fix 2 OPTIMISTIC : UPDATE ... SET bal=?, version=version+1
                    WHERE id=? AND version=:read    0 rows -> retry
 fix 3 ATOMIC     : UPDATE ... SET bal=bal-30 WHERE bal>=30
 best : append-only ledger, nothing to overwrite
```
**In plain words.** Two people edit the same Google Doc offline and the second save wipes the
first. Locking makes the second person wait, versioning makes them redo.

**Core idea.** Two requests read the same row, both modify it, both write — the **second overwrites the
first**. This is the **lost update** problem, and E5's isolation levels don't fully solve it for
read-modify-write. You need explicit **locking**.

**The scenario (why it matters for you):**
```
balance = 100
Req A reads 100 →  wants to subtract 30 → writes 70
Req B reads 100 →  wants to subtract 50 → writes 50   (A's -30 is LOST)
correct answer: 100 - 30 - 50 = 20
```
Same bug for inventory ("2 people buy the last item"), loyalty points, wallet debits.

**Two solutions:**

**Pessimistic locking — "assume conflict, lock first."**
- `SELECT ... FOR UPDATE` inside a transaction locks the row; the second request **waits** until the
  first commits, then reads the fresh value.
- **Good** for high-contention, must-not-conflict writes (money). **Cost:** holds a lock → reduced
  concurrency, deadlock risk (E5) if you lock in inconsistent order.

**Optimistic locking — "assume no conflict, detect it."**
- Add a **`version`** column (or use `updated_at`). Read it; on write do
  `UPDATE ... SET balance=?, version=version+1 WHERE id=? AND version=:readVersion`.
- If **0 rows updated**, someone else changed it first → **retry** the whole read-modify-write.
- **Good** for low-contention (most web traffic) — no locks held, high concurrency. **Cost:** you must
  handle the retry; bad under heavy contention (many retries).

**Rule of thumb.** Optimistic for mostly-independent updates (default); pessimistic for genuinely
contended hot rows (a shared balance/counter). For a **counter**, an atomic DB operation
(`UPDATE ... SET balance = balance - 30 WHERE balance >= 30`) sidesteps both by doing the read+write in
one statement — the cleanest fix when it fits.

**Stack note.** Loyalty ledger (N5): prefer an **append-only ledger** (insert an entry, derive balance)
so there's **no row to overwrite** — concurrency-safe by design. Where you must mutate a balance/inventory
row, use `FOR UPDATE` (pessimistic) or a `version` column (optimistic) or an atomic conditional update.

> **One line:** concurrent read-modify-write loses updates — fix with pessimistic locks
> (`SELECT FOR UPDATE`, waits), optimistic versioning (`WHERE version=?`, retry on conflict), or an atomic
> conditional update; append-only ledgers avoid the problem entirely.

---

## O8 — Race conditions & atomicity beyond one row

**Diagram**
```
 check-then-act:  A: name free? yes      B: name free? yes
                  A: insert              B: insert   -> duplicate!
 fix: UNIQUE constraint (DB refuses the 2nd), or
      INSERT ... ON CONFLICT DO NOTHING, or lock / idempotency key
```
**In plain words.** Two people see one empty seat and both sit. Let the chair itself refuse the
second person.

**Core idea.** Anytime "check then act" spans two statements, a concurrent request can slip between them
(TOCTOU — time-of-check to time-of-use). E.g. "is this username taken? → no → insert" — two requests both
pass the check and both insert.

**Fixes.**
- **Let the database enforce it** — a **unique constraint** turns the race into a catch-and-handle
  (insert, catch the duplicate error) instead of a check-then-act.
- **Do it in one atomic statement** — `INSERT ... ON CONFLICT DO NOTHING`, upserts, atomic increments.
- **Distributed lock** (Redis) when the critical section spans systems (your F5 cron lock is this).
- **Idempotency keys** (F2) so a retried "create" doesn't create twice.

> **One line:** "check then act" across statements races — push the invariant into the DB (unique
> constraint, atomic upsert) or guard with a lock/idempotency key rather than checking in app code.

---

## O9 — Money handling (never float)

**Diagram**
```
 0.1 + 0.2 = 0.30000000000000004  (float)  -> cents lost
 store 12345 (paise) as integer, or NUMERIC(12,2); store currency too
 format as Rs 123.45 only when DISPLAYING; define rounding rule
```
**In plain words.** Count money in whole paise like counting coins. Never in fractions of a coin.

**Core idea.** Floating-point can't represent money exactly (`0.1 + 0.2 !== 0.3`). Using `float`/`double`
for currency **loses cents** and fails audits — unacceptable for your banking/loyalty/pricing.

**Rules.**
- **Store money as integers in the smallest unit** (paise/cents) — `12345` = ₹123.45 — or use the DB
  **`NUMERIC`/`DECIMAL`** type. Never `float`/`double`.
- Do arithmetic in integer/decimal; format to currency only at the **display** edge.
- **Always store the currency** alongside the amount (never assume). Handle rounding **explicitly** and
  consistently (define the rounding rule).
- Financial operations must be **idempotent** (F2) and **transactional** (E5) — a double-processed
  payment or a half-applied transfer is a serious bug.
- Keep a **ledger** (append-only entries), derive balances — auditable and concurrency-safe (O7).

> **One line:** represent money as integer minor units or DECIMAL (never float), always store the
> currency, round explicitly, and make financial ops idempotent, transactional, and ledger-based.

---

## O10 — Time & timezones (store UTC)

**Diagram**
```
 store: timestamptz in UTC   -> convert to user's zone only at display
 never trust the client clock for expiry/OTP; use server time
 DST: a local time can happen twice or never; use a date library
```
**In plain words.** Keep one world clock inside, and translate to local time only when showing it.

**Core idea.** Time bugs are silent and everywhere: wrong timezone, DST jumps, clock skew across servers,
ambiguous local times.

**Rules.**
- **Store everything in UTC** (`timestamptz`); convert to the user's timezone only at the **display**
  edge.
- Never trust the **client's clock** for security decisions (token expiry, OTP windows) — use the
  **server's** time (ties to C4/C12).
- Beware **DST** (some local times don't exist or exist twice) and **month/day math** (adding "1 month"
  is ambiguous). Use a solid date library; don't hand-roll.
- For "expires in 15 min" logic, compare server-side UTC timestamps, not local strings.
- Log timestamps in UTC + ISO-8601 with offset so cross-service logs (H2) line up.

> **One line:** store and compute in UTC, convert to local only for display, trust the server clock for
> security windows, and respect DST/calendar edge cases with a real date library.

---

## O11 — Enums, states & invariants

**Diagram**
```
 states: PLACED -> PAID -> SHIPPED -> DELIVERED
 PLACED -> DELIVERED ? illegal, rejected
 DB also guards: enum type, CHECK constraint, foreign key
```
**In plain words.** Allowed moves are written down. Even if your code has a bug, the DB refuses an
impossible state.

**Core idea.** Model finite sets (order status, KYC state) as **explicit enums/state machines** with
**guarded transitions**, not free-form strings you compare ad hoc. Prevents "impossible" states.

**Practices.**
- Define allowed **states** and legal **transitions** (N1's supplier onboarding is this); reject illegal
  jumps.
- Enforce invariants at the **DB** where possible (check constraints, enums, foreign keys) so bad data
  can't exist even if app code has a bug — the same "make the safe thing automatic" principle as C10's
  RLS.
- Audit state changes (D15).

> **One line:** model finite lifecycles as explicit state machines with guarded transitions and DB-level
> constraints, so illegal states can't be reached or stored.
