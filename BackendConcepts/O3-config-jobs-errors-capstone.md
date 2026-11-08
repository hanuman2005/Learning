# Section O — Gaps Supplement (3/3): Config, Jobs Depth, Errors & Capstone Map

> The remaining smaller gaps, plus how everything maps onto your capstone.

---

## O12 — Configuration & feature flags

**Diagram**
```
 one build -> env vars per environment -> app
 boot: required var missing? -> crash immediately (fail fast)
 feature flag: code deployed OFF -> flip ON at runtime -> flip OFF = instant rollback
```
**In plain words.** A light switch for features. You can turn one off without rebuilding the
house.

**Core idea.** Config that changes per environment or over time shouldn't be hardcoded or require a
redeploy.

- **12-factor config** — all environment-specific values via **env vars** (D11); one build, many
  environments.
- **Feature flags** — toggle features at runtime (gradual rollout, kill-switch, A/B). Lets you deploy
  code dark and enable it later; instant rollback without redeploy.
- **Config validation on boot** — validate required env vars at startup and **fail fast** if missing
  (don't discover a missing secret mid-request).

> **One line:** externalize config via env vars, use feature flags for runtime toggles/kill-switches, and
> validate required config at boot so the app fails fast.

---

## O13 — Graceful shutdown & lifecycle

**Diagram**
```
 SIGTERM
   -> readiness = not ready (LB drains you)
   -> server.close(): no new requests
   -> finish in-flight + current Bull job
   -> close DB pool, Redis, workers -> exit(0)
```
**In plain words.** Do not walk out mid-sentence. Finish what you are doing, then leave.

**Core idea.** When an instance is told to stop (deploy, scale-down), it must **not drop in-flight work**.

- Catch `SIGTERM` → **stop accepting new** requests → **finish in-flight** ones → close DB pool, Redis,
  queue workers → exit.
- Workers (Bull) must finish or safely re-queue the current job, not vanish mid-processing.
- Pairs with health checks (H5): flip **readiness** to "not ready" first so the LB stops routing before
  you drain.

> **One line:** on SIGTERM, stop taking new requests, drain in-flight work, close pools/workers cleanly,
> then exit — coordinated with readiness so the LB drains you first.

---

## O14 — Error handling strategy (taxonomy)

**Diagram**
```
 operational (expected): validation, 404, timeout -> handle, clean envelope
 programmer (bug)      : undefined.x -> crash + restart, do not limp on
 Redis down  -> degrade (use DB)        Primary DB down -> fail
 client sees code+message; logs hold stack + req-id; retry idempotent only
```
**In plain words.** Some errors are normal weather. Some mean the machine is broken. Handle each
type differently.

**Core idea.** Not all errors are equal; handle them by **class**, consistently.

- **Operational errors** (expected: validation fail, 404, upstream timeout) → handle gracefully, return a
  clean error envelope (B7), don't crash.
- **Programmer errors** (bugs: undefined access) → let them surface/crash-and-restart rather than limp on
  corrupt state.
- **Fail fast vs degrade** — decide per dependency: if Redis (cache) is down, **degrade** (serve from DB);
  if the primary DB is down, **fail**.
- **Never leak internals** to clients (stack traces, SQL errors — D3); log them with a correlation id
  (H2), return a generic message + stable code.
- **Retry only idempotent** operations, with backoff (G1); don't retry a non-idempotent charge blindly.

> **One line:** separate operational errors (handle + clean envelope) from bugs (fail fast), degrade on
> non-critical dependency loss, never leak internals, and retry only idempotent ops with backoff.

---

## O15 — Job/queue depth (extends F)

**Diagram**
```
 queue: [KYC (high)] [KYC] ... [bulk email x 10,000 (low)]
 separate priorities so email flood does not starve KYC
 at-least-once -> consumers idempotent
 watch: queue depth, time in queue, DLQ rate -> alert if consumers lag
```
**In plain words.** The express lane and the normal lane at a supermarket. If the queue keeps
growing, open more tills.

**Core idea.** A few queue concepts beyond F worth naming.

- **Delivery semantics** — most queues are **at-least-once** → consumers must be **idempotent** (F2).
  Exactly-once is effectively idempotency + dedup, not a free network guarantee.
- **Priorities & fairness** — separate queues or priorities so a flood of low-priority jobs (bulk email)
  doesn't starve high-priority ones (KYC).
- **Concurrency & rate control** — cap worker concurrency to protect downstreams (an external API's rate
  limit); backpressure when the queue grows.
- **Visibility** — monitor queue depth, processing time, failure/DLQ rate (H3); alert when depth grows
  (consumers falling behind).
- **Scheduled vs immediate** — delayed jobs, cron with a distributed lock (F5).

> **One line:** queues are at-least-once (be idempotent), need priorities so critical jobs aren't starved,
> capped concurrency to protect downstreams, and monitoring of depth/DLQ so you notice consumers falling
> behind.

---

## O16 — Backpressure & flow control

**Diagram**
```
 incoming 1000/s, capacity 200/s
   reject early (429) -> bounded queue (full = reject)
   shed low-priority work -> circuit breaker stops calls to the sick dependency
 degrade instead of collapse
```
**In plain words.** When the shop is full, the bouncer stops letting people in. Better than the
building collapsing.

**Core idea.** When work arrives faster than you can process it, you must **push back** rather than fall
over (unbounded queues, OOM, cascading failure).

- **Reject early** — rate-limit (C16) and return 429 before doing expensive work.
- **Bounded queues** — cap in-memory buffers; shed or reject when full.
- **Load shedding** — under overload, drop low-priority work to protect the core.
- **Circuit breakers** (G1) — stop calling a struggling dependency so you don't pile on.

> **One line:** under overload, push back — rate-limit/429 early, bound your queues, shed low-priority
> load, and break circuits so you degrade instead of collapsing.

---

## O17 — Capstone → curriculum map
Your capstone tasks each exercise specific sections. Use this to know *why* each step exists and where to
revise:

| Capstone step | Revise |
|---|---|
| Define scope & failure modes | J (architecture), O14/O16 (failure) |
| Architecture diagram + tech choices | J1–J2, E13 (polyglot) |
| Build core service (1–3) | B (Express), B6 layering, B7 envelope |
| Add persistence layer | E1–E7 |
| Add auth layer | C1–C17 |
| Review + refactor core | B6, K (testing) |
| Add caching layer | E14–E15, I3 |
| Add message queue / async | F1–F6, O15 |
| Simulate scaling (shard/replica) | E9, J9–J10 (light — not full distributed) |
| Circuit breakers + retries | G1, O14 |
| Rate limiting + backpressure | C16, D10, O16 |
| Review reliability layer | F, G, O14–O16 |
| Logging + tracing | H1–H2, H4 |
| Metrics + alerting | H3, H5–H6 |
| Load test + fix bottlenecks | I1–I6, E2/E8 |
| Security review + hardening | all of D, O9/O10 correctness |
| Final polish + writeup + mock interview | M (API/docs), everything |

**Verdict on coverage.** A–N + this O supplement + the capstone = a genuinely complete backend engineer
foundation for building and interviewing. The one area intentionally left for *later* (post-capstone) is
**deep distributed systems**: real brokers (Kafka, consumer groups, partitions), consensus/replication
internals, event sourcing, CAP in depth, multi-region, and exactly-once semantics. The capstone's
"simulate scaling" step gives you a taste; go deep there once you've felt the pain it creates.

> **One line:** you're not missing foundations — build the capstone to cement A–O, and reserve deep
> distributed systems for after it.
