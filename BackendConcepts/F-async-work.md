# Section F — Async Work & Jobs

> Reference outline — to be deepened with visuals as we teach it. Stack: Bull + Redis.

## F1 — Background jobs (Bull + Redis)

**Diagram**
```
 request -> API: enqueue job, reply 202 at once
                 |
              [Redis queue] --> worker: send email / call KYC / resize image
 knobs: concurrency, priority, delay, retries + backoff
```
**In plain words.** A restaurant: the waiter takes your order and walks away. The kitchen cooks
in the background. You are not blocked at the counter.

- Move slow/non-critical work (emails, KYC calls, image processing, exports) **out of the request** into
  a queue so the response returns fast. A worker processes jobs; Redis holds the queue.
- Concepts: producers/consumers, concurrency, priorities, delayed/scheduled jobs, retries with backoff.
- One line: offload slow work to a Bull queue so requests stay fast; a worker drains it.

## F2 — Idempotency

**Diagram**
```
 job "charge order 77" delivered TWICE (retry)
   run 1: key=order77 not seen -> charge, store key
   run 2: key=order77 seen     -> skip
 result: charged once
```
**In plain words.** Pressing a lift button five times still calls one lift. Design handlers so
running twice equals running once.

- A job/webhook may run **more than once** (retries, at-least-once delivery). Make handlers **idempotent**:
  key on a unique id so processing twice == once (don't double-charge/double-send).
- One line: design handlers so re-running a job/event has the same effect as running it once.

## F3 — Outbox pattern

**Diagram**
```
 BAD : save order to DB ... publish event  (crash between = lost event)
 GOOD: BEGIN
         insert order
         insert outbox(event)        <- same transaction
       COMMIT
       relay: read outbox -> publish -> mark sent
```
**In plain words.** Write the letter and the "to-post" note in the same notebook page. A relay
posts it later, so a crash never loses it.

- Writing to the DB **and** publishing an event isn't atomic (one can fail). **Outbox:** write the event
  to an `outbox` table in the **same transaction** as the data; a relay publishes it later. Guarantees
  the event iff the data committed.
- One line: persist events in the same txn as the data (outbox), then relay — no lost/ghost events.

## F4 — Sagas (distributed transactions)

**Diagram**
```
 order -> reserve stock -> take payment -> book shipping
                              |
                          payment FAILS
           undo: release stock  <-- compensating actions
 choreography: services react to events
 orchestration: one coordinator tells each step
```
**In plain words.** No single "undo all" button across services, so you plan an undo step for
each step.

- No cross-service ACID txn. A **saga** is a sequence of local txns with **compensating actions** to undo
  earlier steps on failure. Choreography (events) vs orchestration (a coordinator).
- One line: replace cross-service transactions with a saga — local steps + compensations on failure.

## F5 — Cron / scheduled jobs & locking

**Diagram**
```
 3 instances, cron at 02:00
 no lock : all 3 run the job -> triple work
 with Redis lock: instance A gets lock, runs; B and C skip
```
**In plain words.** One key to the supply cupboard. Whoever holds it does the job.

- Scheduled tasks (nightly cleanup, reconciliation). With **multiple instances**, use a **distributed
  lock** (Redis) so only one instance runs the job — else it runs N times.
- One line: guard scheduled jobs with a distributed lock so only one instance runs them.

## F6 — Dead-letter queue (DLQ)

**Diagram**
```
 job fails -> retry 1 -> retry 2 -> retry 3 -> still fails
                                       |
                              DEAD-LETTER QUEUE -> alert, inspect, replay
```
**In plain words.** The "returned mail" box. Failed jobs wait there for a human instead of looping
forever or vanishing.

- Jobs that fail after all retries go to a **DLQ** for inspection/replay instead of being lost or
  looping forever. Alert on DLQ growth.
- One line: send exhausted-retry jobs to a DLQ for inspection/replay, and alert on it.
