# Section A — Runtime & Node Internals

## A1 — The event loop

**Diagram: one loop turn**
```
 sync code (your JS)
        v
 microtasks: nextTick, then Promises
        v
 timers (setTimeout)  -> drain microtasks
 pending callbacks    -> drain microtasks
 poll (I/O results)   -> drain microtasks
 check (setImmediate) -> drain microtasks
 close callbacks      -> drain microtasks
        v
 next loop turn
```
**Diagram: who does the work**
```
 your JS (1 thread)
   | await db.query()
   +--> network I/O ---------> OS (no thread used)
   +--> fs / bcrypt / zlib --> libuv pool (4 threads)
   +--> for(10M) loop -------> STAYS on the 1 thread = everyone blocked
```
**In plain words.** One cashier, many customers. The cashier takes an order, hands it to the
kitchen, and serves the next person. Only if the cashier cooks personally (CPU loop) does the
whole line freeze.

**Core idea.** Node runs your JS on **one thread**. Concurrency comes from *not waiting*: I/O is
handed off, the thread keeps working, results come back later.

**Mechanism.** Execution order per tick:
1. Run the current synchronous stack to completion.
2. Drain **microtasks**: `process.nextTick` queue first, then the Promise (`.then`) queue.
3. Enter **macrotask phases** in order: **timers** (`setTimeout`/`setInterval`) → **pending
   callbacks** → **poll** (I/O) → **check** (`setImmediate`) → **close** callbacks.
4. Microtasks are drained **between every phase**, not just once per loop.

**libuv thread pool.** True blocking work (fs, DNS, crypto like bcrypt, zlib) is offloaded to a
**thread pool** (default **4** threads, `UV_THREADPOOL_SIZE`). The pool does the blocking work; the
loop stays free. Network I/O uses the OS event system, not the pool.

**The trap.** *CPU-bound* JS (a big loop, `JSON.parse` of 50 MB, sync crypto) runs **on the one
thread** and **blocks every other request** until it finishes. `process.nextTick` recursion can
**starve** the loop (microtasks never drain to the next phase).

**Analogy.** A single cashier (thread) who takes your order and, instead of standing idle while the
kitchen (libuv/OS) cooks, serves the next customer. A customer who makes the cashier *personally*
count a million beans (CPU work) freezes the whole line.

**Stack note.** bcrypt/argon2 use pool threads (good — won't block the loop, but 4 at a time by
default). Never do heavy CPU work inline in a request handler; offload to a job (Bull) or a worker.

> **One line:** one JS thread + microtasks(nextTick>Promise) drained between ordered macrotask
> phases + a 4-thread libuv pool for blocking work; I/O offloads and scales, CPU work blocks everyone.

---

## A2 — Heap vs stack, and garbage collection

**Diagram**
```
 STACK (auto-freed)        HEAP (GC cleans)
 +--------------+          +-----------------------+
 | fn frame     |--ref---> | object / array        |
 | fn frame     |          | young space: scavenge |
 +--------------+          | old space: mark-sweep |
                           +-----------------------+
 ROOT (global Map) --keeps reachable--> grows forever = LEAK
 Fix: cap size or TTL
```
**In plain words.** The stack is a desk that clears itself when you finish a task. The heap is a
storeroom with a cleaner. The cleaner only throws away things nobody points at. A global Map you
keep adding to is always pointed at, so it never gets cleaned and just grows.

**Core idea.** The **stack** holds call frames and references and is freed automatically as functions
return. The **heap** holds objects and must be reclaimed by the **garbage collector (GC)**.

**GC mechanism.** V8 is **generational**:
- **Nursery / young space** — new objects; collected by cheap, frequent **scavenges**.
- **Old space** — survivors; collected by **mark-sweep**, which includes **stop-the-world** pauses
  that block the event loop briefly.
GC keeps anything **reachable from a root** (globals, the stack, closures) — not "anything still
logically used."

**Leaks = reachable + unbounded.** A leak is not "forgot to free" — it's an object that stays
**reachable** and **grows without bound**. Classic: a module-scope `Map`/array you push into forever.
A bounded cache/pool survives GC *correctly* (still reachable, but capped); an unbounded structure
survives *incorrectly* — same rule, the difference is bounded vs unbounded.

**Stack note.** Watch module-level collections, event listeners you never remove, and closures
capturing large objects. Bound your caches (size or TTL).

> **One line:** stack = auto-freed frames/refs; heap = objects reclaimed by generational GC (cheap
> nursery scavenge vs stop-the-world old-space mark-sweep); a leak is anything reachable-and-unbounded.

---

## A3 — HTTP/1.1 vs 2 vs 3 (head-of-line blocking)

**Diagram**
```
 H1.1: [req1 ......][req2 ......]   one at a time, slow one blocks
 H2  : [s1][s2][s3] on 1 TCP pipe   lost packet stalls ALL streams
 H3  : [s1][s2][s3] on QUIC/UDP     lost packet stalls only its own
```
**In plain words.** H1 is a single-lane road. H2 is many cars on one lane that must stay in
order, so one breakdown stops all. H3 gives each car its own lane.

**Core idea.** Each HTTP version reduces a different kind of **head-of-line (HOL) blocking**.

- **HTTP/1.1** — one request per connection at a time; a slow response blocks everything behind it
  on that connection (**application-level HOL**). Browsers open ~6 connections to compensate.
- **HTTP/2** — **multiplexing**: many streams over one connection, so it kills *HTTP-level* HOL. But
  it rides on **TCP**, and TCP demands in-order delivery, so **one lost packet stalls all streams**
  (**TCP-level HOL** remains).
- **HTTP/3 (QUIC over UDP)** — independent streams at the transport layer, so a lost packet only
  stalls its own stream. Kills TCP HOL.

**Stack note.** Usually terminated at the **CDN / load balancer**, not at Node — so your Node
process typically speaks HTTP/1.1 to the LB while clients get HTTP/2/3 at the edge.

> **One line:** H1 = per-connection HOL; H2 multiplexing kills HTTP-level HOL but TCP-level HOL
> remains; H3/QUIC over UDP kills TCP HOL — and it's usually terminated at the edge, not in Node.

---

## A4 — Request lifecycle (byte → response)

**Diagram**
```
 client
   | TCP + TLS handshake
   v
 socket -> HTTP parser -> req/res objects (heap)
   v
 middleware: parser -> security -> auth -> validate
   v
 route -> handler -> await DB (thread free meanwhile)
   v
 res.json() -> same keep-alive connection -> client
```
**In plain words.** A request is a parcel that goes through checkpoints (middleware) in a fixed
order, reaches the right desk (route), waits for the DB, and the reply goes back on the same road.

**Core idea.** A request is a journey from raw bytes to a written response over one kept-alive
connection.

**Steps.**
1. **TCP + TLS handshake** establishes the encrypted connection.
2. Node's **socket** receives bytes; the HTTP parser builds `req`/`res` objects on the **heap**.
3. **Express middleware chain** runs in registration order (parsers → security → auth → validation).
4. **Route match** dispatches to the handler.
5. Handler **`await`s the DB** — the thread is freed to serve others while the DB works (A1).
6. `res.json(...)` **writes the response** back over the same keep-alive connection.

> **One line:** TCP+TLS → socket parses bytes into req/res on the heap → ordered middleware → route
> → handler awaits I/O (freeing the thread) → response written on the same connection.
