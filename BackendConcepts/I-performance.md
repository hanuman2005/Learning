# Section I — Performance

> Reference outline.

## I1 — Measure before optimizing

**Diagram**
```
 slow? -> MEASURE -> where is time spent?
          DB 70% | CPU 10% | network 15% | GC 5%
 fix the biggest slice first, measure again
```
**In plain words.** Do not guess why the car is slow. Look at the gauges, then fix the worst part.

- Profile to find the real bottleneck (DB, CPU, network, GC). Don't guess. Optimize the hot path.
- One line: profile first; optimize the measured hot path, not a guess.

## I2 — Database is usually the bottleneck

**Diagram**
```
 checklist: missing index? (E2)  N+1? (E8)  SELECT * ?  deep OFFSET? (B8)
```
**In plain words.** Most slow apps are slow because of the database, so check it first.

- Missing indexes (E2), N+1 (E8), fetching too much data, deep offset pagination (B8). Fix these first.
- One line: most backend slowness is the DB — indexes, N+1, and over-fetching.

## I3 — Caching (E14/E15)

**Diagram**
```
 request -> Redis hit? yes -> return (fast)
                     no  -> DB -> store in Redis (TTL) -> return
```
**In plain words.** Keep the answer to a common question on a sticky note.

- Cache hot, expensive, read-heavy results in Redis; handle invalidation and stampede.
- One line: cache the hot expensive reads; mind invalidation.

## I4 — Avoid blocking the event loop (A1)

**Diagram**
```
 CPU-heavy in handler : all users wait
 CPU-heavy in job/worker: handler replies, worker grinds
 big file: stream it, never load all into memory; cap body size
```
**In plain words.** Never let one customer hog the only cashier.

- Offload CPU-bound work to jobs/workers; stream large payloads; cap body/array sizes.
- One line: never block the single thread — offload CPU work, stream, cap sizes.

## I5 — Payload & network

**Diagram**
```
 BAD : GET /orders -> 50,000 rows, every column, uncompressed
 GOOD: GET /orders?limit=20 -> DTO fields only -> gzip
```
**In plain words.** Send only what the screen needs.

- Paginate, select only needed fields (DTOs), compress responses, use keyset pagination for large lists.
- One line: send less — paginate, project fields, compress.

## I6 — Concurrency & batching

**Diagram**
```
 sequential: await a; await b; await c   = 300 ms
 parallel  : await Promise.all([a,b,c])  = 100 ms  (independent calls only)
 batch     : 1 call with 50 ids instead of 50 calls
```
**In plain words.** Send three people to three shops at once instead of one after another.

- Batch external calls, use connection pools (E6), parallelize independent awaits (Promise.all).
- One line: batch and parallelize independent I/O; pool connections.
