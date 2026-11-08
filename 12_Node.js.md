# Node.js for Backend Engineers

A working guide to Node.js for people who build and operate backend services. It teaches how the runtime behaves, so you can reason about a slow, leaking or crashing service instead of guessing.

**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.

**Audience:** developers who know JavaScript and want production-level Node.js skill for backend roles.
**Prerequisites:** working JavaScript (functions, closures, promises at a basic level); comfort with a terminal. The Linux guide in this repository (`03_Linux.md`) is a useful companion, especially for signals, limits and services.
**Scope:** server-side Node.js (HTTP services, workers, CLIs that support services). Browser JavaScript and front-end tooling are out of scope.
**Note on accuracy:** the architecture and semantics described here are long-stable. Several features are version-dependent: the built-in test runner, global `fetch`, the permission model, TypeScript type stripping, `require()` of ES modules, `--env-file`, and default heap limits. Where this guide flags a feature as version-dependent, check the release notes and the API docs for the Node version you actually run. Pin and know your version (use an active LTS line in production).

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 21 lists hands-on labs. Chapter 22 collects interview questions with model answers.

**What "good enough" means for a backend role:** you can explain why one slow request can stall every other request, find what is blocking the event loop, apply backpressure when moving data, shut a service down without dropping requests, diagnose a memory leak from a heap snapshot, and decide between worker threads, cluster and more processes. Chapters 2 to 6, 9, 11 and 12 matter most.

## Contents

1. Runtime architecture
2. The event loop
3. The thread pool and blocking the loop
4. Async patterns and error handling
5. EventEmitter
6. Streams
7. Buffers and encodings
8. Modules: CommonJS and ESM
9. Process lifecycle and graceful shutdown
10. Child processes, worker threads and cluster
11. Memory
12. Performance and profiling
13. Networking: http, net, fetch and undici
14. Configuration and environment
15. Security
16. Logging and observability
17. Testing
18. Package managers and workspaces
19. TypeScript in Node
20. Production deployment
21. Practice labs
22. Interview questions
23. References
24. Glossary

---

## 1. Runtime architecture

**In plain words:** Node runs your JavaScript on one main thread. It waits for files and networks without stopping that thread. This chapter explains the parts that make this work.

### Understand

**What Node is.** Node.js is a C++ program that embeds three main parts:

- **V8**, the JavaScript engine (parsing, JIT compilation [turning code into machine code while the program runs], and garbage collection [freeing memory that is no longer used]). It runs your JavaScript on one thread.
- **libuv**, a C library that provides the event loop, asynchronous I/O (sockets, timers, file system operations) and a thread pool for work that has no non-blocking OS interface.
- **Node core**: C++ bindings and a JavaScript standard library (`fs`, `http`, `net`, `crypto`, `stream` and so on) that connect V8 to libuv and the operating system.

**The mental model.** Your JavaScript runs on a single main thread. When code asks for I/O, Node hands the request to the OS (or to a libuv worker thread), returns immediately, and later runs your callback when the result is ready. One thread can therefore serve thousands of mostly-waiting connections, because waiting costs almost nothing. The price is one rule that governs this whole guide: **while your JavaScript is running, nothing else on that thread runs.** A 200 ms synchronous computation delays every other request by up to 200 ms.

**What Node is good at.** I/O-bound work: API gateways, CRUD services, proxies, streaming, WebSocket servers, glue between services. **What it is poor at without extra design:** sustained CPU-bound work (image processing, large JSON transforms, heavy cryptography, big sorts) on the request path. Those need worker threads, separate processes or a different service.

**Not everything is single-threaded.** The main JavaScript thread is one; but libuv's thread pool, V8's garbage collector helper threads and worker threads are real threads. The statement "Node is single-threaded" means "your JavaScript executes on one thread per isolate".

**Related facts.** Each Node process has one V8 **isolate** (an independent heap and JavaScript execution context); each worker thread has its own. `process.versions` reports the Node, V8 and libuv versions. For sockets, libuv uses the OS readiness mechanism (epoll on Linux, kqueue on macOS, IOCP on Windows).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Versions in use | `node -v`, `process.versions`, `node -p "process.versions"` | Report Node, V8 and libuv versions for a running service |
| Seeing syscalls | `strace -f -p <pid>` (Linux) | Confirm which operations are really blocking |
| Thread view | `ps -T -p <pid>`, `top -H -p <pid>` | Show that a Node process has several threads |
| Version management | `nvm`, `fnm`, `volta`, `.nvmrc`, `engines` in `package.json` | Pin the same Node version in dev, CI and production |

### Common mistakes

- Running an end-of-life Node version in production. Choose an active LTS release and track its schedule.
- Believing "async" means "parallel". Async I/O overlaps waiting; it does not make JavaScript execute in parallel.
- Using Node for CPU-heavy request handlers without isolating that work.

### Self-check

- Which parts of a Node process run on threads other than the main JavaScript thread?
- Why can a single Node process serve thousands of concurrent connections?
- What exactly is blocked while a synchronous loop runs for 300 ms?

---

## 2. The event loop

**In plain words:** The event loop is the cycle that runs your code when work is finished. If one piece of code runs too long, everything else waits. Learn the order of the cycle to predict when your code runs.

### Understand

**Why it matters.** Almost every Node production problem (latency spikes, timeouts, "the health check failed but the CPU is only at 100% of one core") traces back to the event loop being kept busy.

**Phases.** libuv runs the loop as a cycle of phases. Each phase has a queue of callbacks to run.

| Phase | What runs |
|---|---|
| timers | callbacks from `setTimeout` and `setInterval` whose time has elapsed |
| pending callbacks | some deferred system-level callbacks (for example certain TCP errors) |
| idle, prepare | internal use |
| poll | retrieves new I/O events and runs most I/O callbacks. The loop may block here waiting for I/O if nothing else is scheduled |
| check | `setImmediate` callbacks |
| close callbacks | `close` events such as `socket.on('close')` |

A timer's delay is a **minimum**, not a guarantee. If the loop is busy, the callback runs late. That lateness is exactly what "event loop lag" measures.

**Microtasks versus macrotasks.** Each callback above is a macrotask. Between callbacks, Node drains two microtask queues completely:

1. the `process.nextTick` queue,
2. the promise job queue (`.then` handlers and code after `await`).

`process.nextTick` callbacks run before promise jobs in CommonJS code, and the whole nextTick queue is drained, then the promise queue, repeatedly until both are empty. A microtask that keeps scheduling more microtasks **starves the loop**: no I/O or timers run until the queues empty.

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
```

Typical output from a CommonJS main script: `sync`, `nextTick`, `promise`, then `timeout` and `immediate`. The order of `timeout` versus `immediate` from the main module is not deterministic, because it depends on process performance and how quickly the loop starts. Inside an I/O callback, `setImmediate` always runs before a `setTimeout(0)`, because the loop moves from poll to check first. In an ES module, top-level code itself runs inside a promise job, so promise callbacks can run before `nextTick` callbacks there. Do not rely on cross-mechanism ordering in production code; rely on it only to explain behaviour.

**Choosing between them.**
- `setImmediate`: run after the current I/O phase, letting I/O proceed. Use it to yield during long loops.
- `process.nextTick`: run before the loop continues. Reserved for cases such as emitting an event after a constructor returns. Overuse starves I/O. Prefer `queueMicrotask` or promises for ordinary deferral.
- `setTimeout(fn, 0)`: effectively a minimum of one millisecond; fine for coarse deferral.

**Yielding in long work.** Splitting a large loop into chunks and calling `await new Promise(r => setImmediate(r))` between chunks lets other requests run. It reduces lag but costs total throughput; real CPU work belongs on a worker thread (chapter 10).

**Event loop lag.** The extra delay a timer experiences. A healthy service shows single-digit milliseconds at p99 (the value that 99 percent of measurements stay below). Node provides `perf_hooks.monitorEventLoopDelay()` which returns a histogram:

```js
const { monitorEventLoopDelay } = require('node:perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  // values are in nanoseconds
  console.log({ p50: h.percentile(50) / 1e6, p99: h.percentile(99) / 1e6, max: h.max / 1e6 });
  h.reset();
}, 10_000).unref();
```

Export these numbers as metrics and alert on p99 lag. Related: `performance.eventLoopUtilization()` reports the fraction of time the loop was busy versus idle.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Ordering | small experiments with `nextTick`, promises, `setImmediate`, `setTimeout` | Predict output and explain each step |
| Lag measurement | `perf_hooks.monitorEventLoopDelay`, `performance.eventLoopUtilization()` | Export p99 lag as a metric |
| Finding blockers | `node --cpu-prof`, `node --inspect` with DevTools, `clinic doctor`, `0x` | Identify the function holding the loop |
| Load generation | `autocannon` | Reproduce lag under concurrency |

### Common mistakes

- Treating `setTimeout(fn, 100)` as exact.
- Recursive `process.nextTick` or recursive promise chains that never yield.
- Assuming code after `await` runs "later" in a way that lets other requests in; it runs as a microtask, so other I/O callbacks are not guaranteed a turn.
- Measuring only average latency. Lag shows in the tail.

### Self-check

- List the event loop phases in order and say which one runs `setImmediate` callbacks.
- Why do `nextTick` callbacks run before promise callbacks, and how can either starve I/O?
- Why does `setImmediate` always beat `setTimeout(0)` inside an I/O callback?
- What does a rising p99 event loop delay tell you?

---

## 3. The thread pool and blocking the loop

**In plain words:** Some jobs, such as file reads and password hashing, run on a small group of helper threads. There are only 4 helpers by default, so they can become a queue. Heavy work in your own code can also freeze the whole service.

### Understand

**Why a pool exists.** Some operations have no portable non-blocking OS interface. libuv runs these on a pool of worker threads and posts the result back to the loop.

**What uses the pool.**
- most `fs` operations (`fs.readFile`, `fs.stat` and the callback and promise forms; the `*Sync` versions block the main thread instead),
- `dns.lookup` (which calls the system resolver, `getaddrinfo`; this is what `http` and `net` use by default to resolve hostnames),
- `crypto` work such as `pbkdf2`, `scrypt`, `randomBytes`, `randomFill` and some key generation,
- `zlib` compression and decompression.

**What does not use the pool.** Network sockets (TCP, HTTP, UDP) use OS readiness notification on the main thread. `dns.resolve*` uses the c-ares library and its own sockets, not the pool.

**Default size.** 4 threads. Set the environment variable `UV_THREADPOOL_SIZE` before the process starts (documented maximum 1024). The consequence of the default: five concurrent `bcrypt`-style or `pbkdf2` operations, or five slow `dns.lookup` calls, make the fifth wait. A DNS slowdown can therefore stall unrelated file reads and hashing. This produces latency with no CPU use, one of the less obvious Node failure modes.

**What blocks the event loop.**
- `*Sync` APIs (`fs.readFileSync`, `child_process.execSync`, `crypto.pbkdf2Sync`) on the request path. They are acceptable at startup, not in handlers.
- `JSON.parse` and `JSON.stringify` of very large values. Cost is linear in size and runs on the main thread.
- Catastrophic regular expressions (chapter 15).
- Large synchronous loops, big array sorts, template rendering of large documents, huge `Buffer` operations, synchronous compression.
- Accidentally quadratic code (nested array scans, repeated spread copies in loops).

**How to detect it.**
1. Metrics: event loop delay p99 and utilization (chapter 2).
2. Symptoms: all endpoints slow at once, health checks time out, CPU on one core near 100%.
3. Profile: `node --cpu-prof` or `clinic flame` or `0x` and look for wide frames.
4. For "why does the process not exit": the `why-is-node-running` package prints active handles. Node also has `process.getActiveResourcesInfo()`.

**Fixes.** Move CPU work to worker threads; stream instead of buffering; paginate; use asynchronous APIs; precompute; increase `UV_THREADPOOL_SIZE` only when the pool is the measured bottleneck and the host has cores to use.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pool tuning | `UV_THREADPOOL_SIZE` | Explain when raising it helps and when it does not |
| Lag detection | `monitorEventLoopDelay`, `autocannon` plus a profiler | Show a blocking handler with a flame graph |
| Hung process | `why-is-node-running`, `process.getActiveResourcesInfo()` | Explain what keeps a process alive after the work is done |
| Linting for sync calls | ESLint rules (for example `n/no-sync`) | Keep `*Sync` calls out of request code |

### Common mistakes

- `fs.readFileSync` or `execSync` inside a route handler.
- Raising `UV_THREADPOOL_SIZE` far beyond the core count without measuring.
- Assuming `dns.lookup` is non-blocking and never saturates the pool.
- Parsing multi-megabyte JSON bodies in the same process that serves latency-sensitive traffic.

### Self-check

- Name four categories of work that run on the libuv thread pool and the default pool size.
- A service shows low CPU but high latency and many outbound calls. How could the thread pool be involved?
- Give three ways to find which function is blocking the loop.

---

## 4. Async patterns and error handling

**In plain words:** Async code lets a program start a job and continue while it waits. Errors in async code are easy to lose if you forget one keyword or one handler. This chapter shows safe patterns and how to limit parallel work.

### Understand

**Evolution.** Callbacks (error-first: `(err, result)`), then promises, then `async/await`. Today use promises and `async/await`. Callback APIs can be wrapped with `util.promisify`, and many core modules offer promise versions (`node:fs/promises`, `node:timers/promises`, `node:stream/promises`).

**An `async` function always returns a promise.** A thrown error becomes a rejection. A `try/catch` catches a rejection only if the promise is awaited inside the `try`.

**Concurrency primitives.**

| Function | Behaviour | Use when |
|---|---|---|
| `Promise.all(ps)` | resolves with all results, rejects at the first rejection (other promises keep running) | independent calls that must all succeed |
| `Promise.allSettled(ps)` | waits for all, reports each outcome | partial failure is acceptable |
| `Promise.race(ps)` | settles with the first to settle | timeouts |
| `Promise.any(ps)` | resolves with the first fulfilment; rejects with `AggregateError` if all reject | redundant sources |

`Promise.all` does not cancel the losers. To cancel work use `AbortController` and pass `signal` to APIs that support it (`fetch`, `fs/promises`, `timers/promises`, `events.once`). `AbortSignal.timeout(ms)` creates a signal that aborts after a delay.

**Sequential versus parallel.** `await` in a `for` loop runs steps one after another. `await Promise.all(items.map(fn))` starts everything at once. Starting 10,000 database calls at once exhausts the pool, file descriptors or a downstream service.

**Bounded concurrency.** Limit in-flight work. Options: a small worker-loop pattern, or a library such as `p-limit` or `p-map`:

```js
async function mapLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  async function worker() {
    while (next < items.length) {
      const i = next++;
      results[i] = await fn(items[i], i);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```

Choose the limit from the downstream capacity (connection pool size, rate limit), not from a guess.

**Error-handling pitfalls.**

- **Forgotten `await`.** `doWork()` without `await` returns a promise that nobody observes. Errors become unhandled rejections and the caller continues before the work finishes. In a `try` block, `return somePromise` without `await` also escapes the `catch`; use `return await` inside `try`. A linter rule (`@typescript-eslint/no-floating-promises`, or ESLint's `no-floating-promises` equivalents) catches this.
- **Unhandled rejections.** Since Node 15 the default is to raise an `uncaughtException`-style failure and crash the process when a promise rejects with no handler (the `--unhandled-rejections` flag selects other modes). Treat this as a feature: log with context and let the supervisor restart you (chapter 9).
- **`async` callbacks passed where a sync callback is expected.** `array.forEach(async ...)` does not wait; `setInterval(async ...)` and event handlers can throw rejections nobody handles.
- **Swallowed errors.** An empty `catch {}` or `.catch(() => {})` hides faults. Log, rethrow or convert to a typed error.
- **Losing the cause.** Wrap with `new Error('message', { cause: err })` so the stack of the original is kept.
- **Errors in event emitters** (chapter 5): an `error` event with no listener throws; an `async` listener's rejection is not caught by the emitter.
- **Throwing non-Error values.** Throw `Error` objects (or subclasses) so stacks exist.
- **Operational versus programmer errors.** Operational errors (timeout, ECONNRESET, validation failure) are expected and handled. Programmer errors (TypeError, broken invariants) mean state is unknown; fail fast.
- **Express and async handlers.** Whether a framework forwards a rejected promise from a handler to its error middleware depends on the framework and version. Check the documentation of the version you use; do not assume.

**Timeouts everywhere.** Every outbound call needs a timeout; a promise that never settles leaks resources and holds requests open. Combine `AbortSignal.timeout()` with retries that use exponential backoff and jitter, and only for idempotent operations (operations that are safe to repeat with the same result).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Promise APIs | `Promise.all`, `allSettled`, `race`, `any`, `util.promisify`, `node:timers/promises` | Pick the right combinator and explain failure behaviour |
| Cancellation | `AbortController`, `AbortSignal.timeout`, `AbortSignal.any` (Node 20.3 and 18.17 or later) | Cancel a fetch and a database call on client disconnect |
| Concurrency control | `p-limit`, `p-map`, hand-written worker loop | Process 100k items with a fixed in-flight limit |
| Detection | `--unhandled-rejections`, `process.on('unhandledRejection')`, ESLint floating-promise rules | Find forgotten awaits before production |

### Common mistakes

- `await` inside `forEach`, or `forEach(async ...)`.
- `Promise.all` over thousands of items with no limit.
- Mixing callbacks and promises in the same function so errors are reported twice or never.
- Retrying non-idempotent operations such as payments.
- No timeout on outbound HTTP or database calls.

### Self-check

- What happens to the other promises when one in `Promise.all` rejects?
- Why does `try { return fetchData(); } catch {}` not catch a rejection?
- Write a function that runs 1000 tasks with at most 10 in flight.
- What is the default behaviour on an unhandled rejection in current Node, and why is crashing reasonable?

---

## 5. EventEmitter

**In plain words:** An EventEmitter lets one part of the code announce that something happened, and other parts react. Listeners run at once, one after another. A missing error listener can crash the program.

### Understand

**What it is.** `events.EventEmitter` is the base of many core objects (servers, sockets, streams, child processes). It is a synchronous publish/subscribe mechanism: `emit` calls each listener in registration order, immediately, on the current stack.

**Key behaviours.**
- Listeners run **synchronously**. A slow listener blocks the emitter.
- Emitting `'error'` with no `'error'` listener throws the error and can crash the process. Always attach an `error` handler to sockets, streams, child processes and custom emitters.
- `on` versus `once`; `off` / `removeListener` to detach.
- More than 10 listeners for one event on one emitter prints a `MaxListenersExceededWarning`. This warning usually reveals a leak, such as registering a listener per request on a long-lived object. Do not simply raise the limit; find the leak.
- `async` listeners: the emitter ignores the returned promise. A rejection becomes an unhandled rejection, and an exception inside is not routed to the `error` event unless you opt in with `captureRejections` (see the `events` documentation).
- `events.once(emitter, 'event')` returns a promise; it rejects if `error` is emitted first. `events.on` returns an async iterator.

```js
const { EventEmitter, once } = require('node:events');

const e = new EventEmitter();
e.on('error', (err) => logger.error({ err }, 'emitter failed'));
setImmediate(() => e.emit('ready', 42));
(async () => {
  const [value] = await once(e, 'ready'); // CommonJS has no top-level await
  console.log(value);
})();
```

**Design guidance.** Use emitters for lifecycle and notification inside a process. For work that must not be lost (jobs, payments), use a durable queue, not an in-memory emitter.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Core API | `on`, `once`, `off`, `emit`, `listenerCount`, `events.once`, `events.on` | Build a small typed emitter and test it |
| Leak detection | `MaxListenersExceededWarning`, `--trace-warnings` | Find the code adding listeners repeatedly |
| Error routing | `'error'` event, `captureRejections` option | Ensure no emitter can crash the process unobserved |

### Common mistakes

- No `error` listener on a stream or socket.
- Adding a listener inside a request handler to a global emitter and never removing it.
- Assuming listeners run asynchronously, or that `emit` waits for async listeners.
- Using an emitter as a job queue.

### Self-check

- What happens if `emit('error', err)` is called with no listener?
- Why are `async` listeners dangerous?
- What does `MaxListenersExceededWarning` usually mean?

---

## 6. Streams

**In plain words:** A stream moves data in small pieces, so a big file does not need big memory. Backpressure tells a fast sender to wait for a slow receiver. Use pipeline so errors close everything cleanly.

### Understand

**Why streams exist.** A file of 5 GB cannot be loaded into a heap that is a fraction of that size. Streams process data in chunks, keeping memory use constant and starting output before input finishes. They also provide **backpressure**, the mechanism that stops a fast producer from overwhelming a slow consumer.

**Types.**
- **Readable**: a source (file, HTTP request body, database cursor).
- **Writable**: a sink (file, HTTP response, socket).
- **Duplex**: both, independent (a TCP socket).
- **Transform**: a duplex that modifies data (gzip, CSV parser).

Streams operate on `Buffer` or string chunks by default, or on arbitrary values in `objectMode`. Readables can be consumed with `for await (const chunk of readable)`. The modern Web Streams API also exists in Node; the classic `node:stream` API remains the most common in server code.

**Backpressure.** A writable has an internal buffer with a size hint, `highWaterMark` (16 KiB for byte streams by default, 16 objects in object mode). `writable.write(chunk)` returns `false` when the buffer is over the mark; the producer must pause until the `'drain'` event. Piping handles this for you. Ignoring the return value of `write()` fills memory and is a classic cause of out-of-memory crashes with slow clients.

**Use `pipeline`, not `.pipe`.** `stream.pipeline` (from `node:stream/promises` for the promise form) wires streams together, propagates errors, and destroys every stream on failure. `.pipe` does not forward errors or clean up, which leaks file descriptors.

```js
const { pipeline } = require('node:stream/promises');
const fs = require('node:fs');
const zlib = require('node:zlib');

await pipeline(
  fs.createReadStream('access.log'),
  zlib.createGzip(),
  fs.createWriteStream('access.log.gz'),
);
```

Custom transform with async generator functions is supported by `pipeline`:

```js
await pipeline(
  source,
  async function* (chunks) { for await (const c of chunks) yield c.toString().toUpperCase(); },
  destination,
);
```

**Stream or buffer?**

| Situation | Choice |
|---|---|
| Small, bounded payload (a JSON body of a few KB) | buffer; simpler |
| Unbounded or large (uploads, exports, log processing, proxying) | stream |
| Need the whole value to validate or parse (JSON) | buffer, but enforce a size limit |
| Slow consumers such as mobile clients | stream with backpressure |

**HTTP and streams.** `http.IncomingMessage` is a readable; `http.ServerResponse` is a writable. Proxying a large download is `pipeline(upstream, res)`. Handle client disconnects: if the client goes away, destroy the upstream request, or work continues for nobody.

**Large file patterns.** Read with `createReadStream` (supports `start`, `end` for ranges), parse line by line with `readline` or a streaming parser, write results incrementally, and use `pipeline` so that errors close everything. For database exports, use a cursor and stream rows rather than loading result sets.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Composition | `stream/promises.pipeline`, `Readable.from`, async generators | Build a multi-stage streaming pipeline with correct error handling |
| Backpressure | `write()` return value, `'drain'`, `highWaterMark` | Explain and demonstrate memory growth without it |
| Line processing | `readline`, `for await` over a readable | Process a multi-gigabyte file in constant memory |
| Observation | `process.memoryUsage()`, `--inspect` memory view | Prove memory stays flat during a stream |

### Common mistakes

- `.pipe()` chains without error handlers.
- Reading a whole file with `fs.readFile` for an export endpoint.
- Ignoring the boolean returned by `write()`.
- Not handling client aborts, leaving upstream work running.
- Buffering an entire upload in memory before saving; enforce size limits and stream to storage.

### Self-check

- What is backpressure, and which stream signals implement it?
- Why prefer `pipeline` over `pipe`?
- How would you serve a 2 GB file to many clients without exhausting memory?
- Where does the buffering happen when a slow client downloads from your server?

---

## 7. Buffers and encodings

**In plain words:** A Buffer holds raw bytes, and a string is bytes read with an encoding such as UTF-8. Many bugs come from mixing these two. This chapter shows how to convert them safely.

### Understand

**What a Buffer is.** A `Buffer` is a fixed-size chunk of raw bytes, a subclass of `Uint8Array`. Network and file I/O deal in bytes; strings are an interpretation of bytes under an encoding. Most encoding bugs come from confusing the two.

**Creation.**
- `Buffer.from('text', 'utf8')`, `Buffer.from(array)`, `Buffer.from(arrayBuffer)`.
- `Buffer.alloc(n)` returns zero-filled memory.
- `Buffer.allocUnsafe(n)` may contain old data from memory; use only if you overwrite it fully. Never expose it to clients uninitialised.
- Never use the deprecated `new Buffer()`.

**Encodings.** `utf8` (default), `hex`, `base64`, `base64url`, `latin1`, `utf16le`, `ascii`. `buf.toString('base64')` and `Buffer.from(str, 'base64')` convert to and from text. `base64url` is the URL-safe variant used in JWTs.

**Multi-byte characters across chunks.** A UTF-8 character can be split between two chunks. Calling `chunk.toString()` on each chunk corrupts it. Concatenate chunks first (`Buffer.concat`), call `readable.setEncoding('utf8')`, or use `StringDecoder`.

**Length versus characters.** `Buffer.byteLength(str)` returns bytes; `str.length` returns UTF-16 code units. Use bytes for `Content-Length` and for size limits.

**Memory.** Buffer memory lives outside the V8 heap (counted as `external` and `arrayBuffers` in `process.memoryUsage()`). A process can run out of memory with a small V8 heap if buffers are retained. Small `Buffer.from` slices can retain a larger underlying pool or parent buffer, so copy a slice (`Buffer.from(slice)` or `Buffer.copyBytesFrom` where available) if you keep it long-term.

**Comparison.** Compare secrets with `crypto.timingSafeEqual` on equal-length buffers, not `===`, to avoid timing leaks.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Conversion | `Buffer.from`, `toString`, `Buffer.concat`, `Buffer.byteLength` | Convert between utf8, hex, base64, base64url correctly |
| Streaming text | `setEncoding`, `string_decoder` | Decode chunked UTF-8 safely |
| Binary parsing | `readUInt32BE`, `writeUInt16LE`, `DataView`, `subarray` | Parse a length-prefixed binary frame |
| Inspection | `buf.toString('hex')`, `process.memoryUsage().external` | Explain off-heap growth |

### Common mistakes

- `chunks += chunk` when assembling a request body, which converts each chunk to a string and can corrupt multi-byte characters.
- Using `str.length` as a byte count.
- `Buffer.allocUnsafe` for data later sent to clients.
- Comparing tokens with `===`.
- Assuming base64 and base64url are interchangeable.

### Self-check

- Why can `chunk.toString()` per chunk corrupt text, and what are two fixes?
- What is the difference between `Buffer.alloc` and `Buffer.allocUnsafe`?
- Where do Buffer bytes live relative to the V8 heap?

---

## 8. Modules: CommonJS and ESM

**In plain words:** Node has two module systems: CommonJS (require) and ES modules (import). They load differently and mix in limited ways. Know which one each file uses and why.

### Understand

**Two module systems.**

| | CommonJS (CJS) | ECMAScript modules (ESM) |
|---|---|---|
| Syntax | `require()`, `module.exports` | `import`, `export` |
| Loading | synchronous | asynchronous, statically analysable |
| Top-level `await` | no | yes |
| `__dirname`, `__filename` | available | not available; use `import.meta.url`. `import.meta.dirname` and `import.meta.filename` exist from Node 20.11 and 21.2 (stable from 22.16 and 24.0), for `file:` modules only |
| Strict mode | optional | always |
| Bindings | copies of exported values at require time | live bindings |

**How Node decides which one a file is.** `.mjs` is ESM and `.cjs` is CJS. For `.js`, the nearest `package.json` `"type"` field decides: `"module"` means ESM; absent or `"commonjs"` means CJS.

**Interop.**
- ESM can `import` CommonJS. The default import is `module.exports`; named imports work only when Node can statically detect the exports.
- CJS loading ESM with `require()` was historically impossible (use dynamic `import()`, which returns a promise). `require()` of ES modules works without a flag from Node 22.12 and 20.19. The ES module must not use top-level `await`; if it does, Node throws `ERR_REQUIRE_ASYNC_MODULE`. Older versions need `import()`.
- Dual packages publish both formats through the `exports` field. Mixing both in one process can load a package twice (the "dual package hazard") and break `instanceof` and singletons.

**Resolution.**
- Bare specifiers (`express`) resolve through `node_modules` directories walking upward from the importing file, using the package's `exports` (or `main`) field.
- The `exports` field in `package.json` defines the public entry points and blocks deep imports not listed; it supports conditions such as `import`, `require`, `node` and `default`.
- ESM relative imports require the full file extension (`./util.js`). CommonJS tries extensions and `index.js` automatically.
- Built-ins use the `node:` prefix (`node:fs`), which makes them unambiguous. Use it consistently.
- `require` caches modules by resolved filename in `require.cache`, so module-level state is a singleton per process. ESM has its own cache.

**Real-world impact.** Choosing ESM or CJS affects tooling (test runners, loaders, bundlers), TypeScript configuration, and how libraries are consumed. New services are commonly written as ESM; check that your critical dependencies and tooling support it.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Format selection | `"type"` in `package.json`, `.mjs`, `.cjs` | Predict how any file will load |
| Resolution | `require.resolve`, `import.meta.resolve`, `node --trace-warnings` | Explain why a specific file was loaded |
| Package entry points | `exports`, `main`, `imports` fields | Publish an internal package with correct conditional exports |
| Dynamic loading | `import()` | Load ESM from CJS and lazy-load heavy modules |

### Common mistakes

- Using `__dirname` in ESM.
- Omitting file extensions in ESM relative imports.
- `require`-ing an ESM-only package on a Node version that does not support it.
- Circular dependencies returning partially initialised exports.
- Reading and mutating `require.cache` in application code.

### Self-check

- How does Node decide whether `app.js` is ESM or CJS?
- Why can an ESM import of a CommonJS package lose named exports?
- What does the `exports` field change compared with `main`?
- What is the dual package hazard?

---

## 9. Process lifecycle and graceful shutdown

**In plain words:** Servers are stopped often: by deployments, scaling and crashes. A good service finishes its current requests before it stops. This chapter shows signals, exit codes and a safe shutdown order.

### Understand

**Why it matters.** Deployments, autoscaling and crashes all stop processes. A service that stops abruptly drops in-flight requests, loses buffered writes and leaves jobs half done. Container and orchestrator behaviour (Kubernetes, Docker, systemd, pm2) is driven by signals and exit codes.

**Signals.** The orchestrator (the tool that starts, stops and scales containers, such as Kubernetes) sends `SIGTERM` (a polite request to stop) and, after a grace period, `SIGKILL` (a forced stop that cannot be caught). In an interactive terminal, Ctrl+C sends `SIGINT`. Node's default action on `SIGTERM` and `SIGINT` is to exit immediately, without cleanup, unless you install a handler. On Windows, signal support is limited.

**Graceful shutdown sequence.**
1. Receive `SIGTERM`; mark the service as not ready (readiness probe fails) so the load balancer stops sending traffic.
2. Stop accepting new connections: `server.close()`.
3. Let in-flight requests finish, with a deadline. Since Node 19, `server.close()` also closes idle keep-alive connections. On older versions, call `server.closeIdleConnections()` (added in 18.2) after `server.close()`. As a last resort after the deadline, call `closeAllConnections()`.
4. Close dependencies in order: stop consumers, flush queues and logs, close database pools and caches.
5. Exit with code 0, or exit non-zero if the deadline passes.

```js
let shuttingDown = false;
async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'shutting down');
  const killer = setTimeout(() => process.exit(1), 25_000);
  killer.unref();
  try {
    const closed = new Promise((res, rej) => server.close((e) => (e ? rej(e) : res())));
    server.closeIdleConnections?.(); // call after close(); needed only before Node 19
    await closed; // resolves when all remaining connections have ended
    await db.end();
    process.exitCode = 0;
  } catch (err) {
    logger.error({ err }, 'shutdown failed');
    process.exitCode = 1;
  }
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

Set the hard-deadline timer shorter than the orchestrator's grace period (for example `terminationGracePeriodSeconds` or `TimeoutStopSec`).

**Exit codes.** `0` is success. Non-zero is failure; the convention for death by signal is 128 plus the signal number (143 for `SIGTERM`, 137 for `SIGKILL`, often seen with out-of-memory kills in containers). Node uses code 1 for an uncaught fatal exception. Prefer `process.exitCode = n` and letting the loop drain over `process.exit(n)`, which terminates immediately and can truncate pending asynchronous writes (including to stdout when it is a pipe).

**`exit` event.** Handlers on `process.on('exit')` must be synchronous; no timers or promises will run. `beforeExit` fires when the loop empties and can schedule more work.

**`uncaughtException` and `unhandledRejection` policy.**
- After an uncaught exception, application state is undefined: a half-completed operation, a leaked lock, an inconsistent cache. The recommended policy is log it with full context, attempt a bounded graceful shutdown or flush, and exit non-zero so the supervisor restarts a clean process. Do not continue serving.
- `unhandledRejection`: current Node defaults to throwing, which routes into `uncaughtException`. Keep the default. A handler that only logs and carries on hides bugs.
- `process.on('uncaughtExceptionMonitor')` observes without altering behaviour (useful for reporting).
- Crash-only design plus a supervisor with backoff is more reliable than attempts to survive arbitrary errors. Keep the request-level error handler for expected failures; reserve the process-level handlers for last-chance logging.

**Process facts.** `process.pid`, `process.uptime()`, `process.argv`, `process.env`, `process.cwd()`. `process.kill(pid, signal)` sends signals.

**Containers.** Run `node server.js` directly, not `npm start`, so that `node` receives signals (npm may not forward them reliably). Node as PID 1 does not reap orphaned zombies; use `--init` or `tini` if you spawn children (see the Linux guide).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Signals | `process.on('SIGTERM')`, `kill -TERM <pid>`, `docker stop`, `kubectl delete pod` | Demonstrate a graceful stop under load with zero failed requests |
| Server drain | `server.close`, `closeIdleConnections`, `closeAllConnections` | Explain why `close()` can hang with keep-alive clients |
| Exit semantics | `process.exitCode`, `process.exit`, `beforeExit`, `exit` | Choose the right one and explain truncation risk |
| Crash handling | `uncaughtException`, `unhandledRejection`, `--unhandled-rejections` | State and defend a crash policy |
| Helper libraries | `stoppable`, `http-terminator`, `lightship` and similar | Know they exist; understand what they automate |

### Common mistakes

- No `SIGTERM` handler, so deploys drop requests.
- Swallowing `uncaughtException` and continuing to serve.
- `process.exit()` right after logging, losing buffered log lines.
- Shutdown with no deadline, so the orchestrator sends `SIGKILL` anyway.
- Closing the database before the HTTP server has drained.
- Health check still reporting ready during shutdown.

### Self-check

- Describe the sequence for a zero-downtime shutdown.
- Why is continuing after an `uncaughtException` unsafe?
- Why does `server.close()` sometimes never call its callback?
- What does exit code 137 usually indicate?

---

## 10. Child processes, worker threads and cluster

**In plain words:** One Node process uses one core for JavaScript. To use more cores or run heavy work, you add processes or threads. This chapter helps you choose between child processes, worker threads and cluster.

### Understand

**Why more than one execution unit.** One Node process uses one core for JavaScript. To use more cores, or to isolate CPU work from request handling, you need more processes or threads.

**`child_process`.**
- `spawn(cmd, args)`: stream-based, no shell by default, any output size. Prefer it.
- `execFile(file, args)`: buffers output, no shell.
- `exec(commandString)`: runs through a shell and buffers output. **Shell injection risk** if the string contains user input.
- `fork(modulePath)`: spawns a new Node process with an IPC channel (`process.send`, `on('message')`).
- Always handle `error`, `exit` and `close`, set timeouts, and limit output. Kill with `child.kill()`.

**`worker_threads`.** Separate threads within one process, each with its own V8 isolate, event loop and heap. Communication is by message passing (structured clone), transferable objects (`ArrayBuffer`, moved without copying) and `SharedArrayBuffer` with `Atomics` for shared memory. Good for CPU-bound JavaScript (image resizing, parsing, hashing, compression). They share the process's memory limit and the libuv pool, and starting one has a cost, so use a **worker pool** (for example `piscina`) instead of a thread per task. Workers do not help with I/O-bound work, which the main thread already handles well.

**`cluster`.** Forks several Node processes that share the same server port. On most platforms the primary distributes incoming connections round-robin. It multiplies HTTP throughput across cores. Workers share nothing: in-memory caches and sessions are per worker, so state must live in an external store.

**Choosing.**

| Need | Choice |
|---|---|
| Use all cores for an HTTP API | multiple processes: `cluster`, pm2 cluster mode, or (better in containers) one process per container with the orchestrator running several replicas |
| CPU-heavy task inside a request | worker thread pool |
| Run an external program (ffmpeg, git) | `spawn` or `execFile` |
| Isolation from crashes and memory leaks | separate processes |
| Share large binary data cheaply | worker threads with `SharedArrayBuffer` or transferables |
| Long, independent background jobs | a separate worker service consuming a queue |

In containerised environments, prefer one Node process per container and scale replicas; the orchestrator then handles restarts, health and resource limits uniformly. `cluster` and pm2 cluster mode remain useful on a plain VM.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Child processes | `spawn`, `execFile`, `fork`, `AbortSignal` for kill | Run an external tool safely with a timeout |
| Worker threads | `worker_threads`, `piscina` | Offload a hashing or parsing task and measure the gain |
| Multi-process | `cluster`, `pm2 -i max`, container replicas | Scale an API across cores without shared state bugs |
| Load test | `autocannon` | Compare one process, cluster and worker pool under load |

### Common mistakes

- `exec` with interpolated user input.
- Creating a worker per request.
- Expecting `cluster` workers to share an in-memory cache or session.
- Using worker threads for I/O-bound tasks.
- Not handling child `error` and exit events, leaving zombies and hung requests.
- Spawning a worker pool larger than the available cores.

### Self-check

- When would you choose a worker thread over cluster, and vice versa?
- Why is `exec` with user input dangerous, and what do you use instead?
- How do worker threads exchange data, and which approaches avoid copying?

---

## 11. Memory

**In plain words:** Memory in Node is managed by a garbage collector, which frees objects that nothing uses. A leak happens when your code keeps objects that it no longer needs. This chapter shows how to measure memory and find leaks.

### Understand

**Where memory goes.** A Node process's RSS (resident set size, the physical memory it uses) includes the V8 heap, off-heap `Buffer` and ArrayBuffer memory, native addon memory, code and the thread stacks. `process.memoryUsage()` reports `rss`, `heapTotal`, `heapUsed`, `external` and `arrayBuffers`. A growing `rss` with a flat `heapUsed` points off-heap (Buffers, native modules, fragmentation).

**Generational garbage collection.**
- **Young generation** (new space): small, collected frequently by a fast copying collector (scavenge). Most objects die here. Objects surviving are promoted.
- **Old generation**: larger, collected by mark-sweep-compact, mostly incrementally and concurrently, with some pauses. Long GC pauses mean a large live heap or high allocation churn.

**Heap limits.** V8 limits the old generation; the default depends on Node version and available memory (newer versions consider container limits). Set it explicitly with `node --max-old-space-size=<MB>`. In a container, set it below the container memory limit with headroom for buffers, stacks and native memory (a common rule of thumb is roughly 70 to 80 percent of the limit, to be validated by measurement). When the heap limit is hit the process dies with a "JavaScript heap out of memory" fatal error; when the container limit is hit the kernel OOM-kills it (exit 137), with no JavaScript error at all.

**Typical leak causes.**
- Unbounded caches or maps (no eviction, no size cap). Use an LRU with a maximum size and TTL.
- Listeners added per request to long-lived emitters.
- Closures retaining large objects (a callback capturing a request or response).
- Global arrays and module-level state that accumulate.
- Timers and intervals never cleared; `setInterval` retains its callback.
- Promises that never settle, holding their closures.
- Retaining request-scoped data in a global structure (metrics labels with unbounded cardinality, per-user keys).
- Buffer slices retaining large parents; accumulating response bodies.
- Per-request creation of agents or clients that keep sockets open.

**Is it a leak?** Look at the trend of `heapUsed` after garbage collection over hours, not at a single snapshot. A sawtooth that returns to the same baseline is healthy; a rising baseline is a leak. Memory that grows under load and plateaus can simply be an undersized heap or cache.

**Finding a leak.**
1. Confirm with metrics (heap used after GC, RSS) over time.
2. Take two or three **heap snapshots** a few minutes apart under steady load: `v8.writeHeapSnapshot()`, the `--heapsnapshot-signal=SIGUSR2` flag, or the DevTools Memory tab attached with `--inspect`.
3. Compare snapshots in Chrome DevTools: look at objects whose count grows, and follow **retainers** to see what holds them.
4. Alternatively use allocation sampling (the DevTools "Allocation sampling" profile) or `clinic heapprofiler`.
5. Fix, then re-measure.

Taking a snapshot pauses the process and needs memory proportional to the heap; do it on an instance taken out of rotation if you can.

**Flags worth knowing.** `--max-old-space-size`, `--heapsnapshot-signal`, `--heapsnapshot-near-heap-limit=<count>` (writes snapshots as the heap nears its limit; added in Node 15.1 and 14.18), `--trace-gc`, `--expose-gc` (for tests only). Exposing the inspector (`--inspect`) on a public interface is a severe security hole; bind to localhost and tunnel.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Memory metrics | `process.memoryUsage()`, `v8.getHeapStatistics()`, `process.resourceUsage()` | Export heap used, RSS and external as metrics |
| Heap snapshots | `node --inspect`, Chrome DevTools Memory tab, `v8.writeHeapSnapshot()`, `--heapsnapshot-signal` | Find a retainer path for a leaking object |
| Profiling allocation | DevTools allocation sampling, `clinic heapprofiler` | Identify which function allocates the most |
| GC behaviour | `node --trace-gc`, `perf_hooks` GC entries | Read pause times and frequency |
| Limits | `--max-old-space-size`, container memory limits | Size heap against container limits |

### Common mistakes

- Setting `--max-old-space-size` equal to or above the container limit.
- Adding RAM or restarting on a schedule instead of finding the leak.
- Using a plain `Map` or object as a cache with no bound.
- Comparing snapshots taken at different load levels.
- Forgetting `external` memory when diagnosing.
- Exposing the debugger port publicly.

### Self-check

- What is the difference between young and old generation collection?
- RSS grows but `heapUsed` is flat. Where do you look?
- Describe how to find a leak using heap snapshots.
- How do you size `--max-old-space-size` inside a container?

---

## 12. Performance and profiling

**In plain words:** Performance work starts with measuring, not guessing. Profilers show where time is spent. Most gains come from fewer or parallel network calls, not from small code tweaks.

### Understand

**Method first.** Measure, find the bottleneck, change one thing, measure again. Typical cost centres for Node services are, in rough order: slow downstream calls (database, HTTP), unbounded concurrency, serialisation (JSON), synchronous work blocking the loop, GC pressure, and DNS or connection setup.

**Profiling tools and what they show.**
- **CPU profile / flame graph**: where JavaScript time is spent. Wide frames at the top are hot. Produce with `node --cpu-prof` (writes `.cpuprofile` loadable in DevTools), `0x`, or `clinic flame`.
- **`clinic doctor`**: diagnoses the category of problem (CPU, event loop, I/O, GC) from a load test.
- **`clinic bubbleprof`**: visualises asynchronous flow and where time is spent waiting.
- **`node --inspect` with DevTools**: live CPU, memory and heap snapshots.
- **`--prof` and `--prof-process`**: V8's built-in tick profiler.
- **Load generators**: `autocannon` for HTTP. Always load test with realistic payloads and concurrency, against a production-like build with `NODE_ENV=production`.

**Flame graph reading.** The x axis is share of samples, not time order. A wide frame means much time inside it and its children. Look for wide plateaus in your own code or in a few hot library functions (JSON, regex, crypto, logging).

**JSON cost.** `JSON.parse` and `JSON.stringify` are synchronous and linear in size. For large or hot payloads: limit body sizes, return less data (field selection, pagination), stream large results (NDJSON), use schema-based serialisers such as `fast-json-stringify` where measured to help, and avoid repeatedly stringifying the same object (for example logging full request bodies). `structuredClone` and deep-cloning via JSON round-trips are also costly.

**Avoid sync APIs** in request paths (chapter 3). **Avoid per-request heavy initialisation:** compile regexes, create HTTP agents, database pools and schema validators once at startup.

**Connection reuse.** Creating a TCP and TLS connection per outbound request is expensive and exhausts ephemeral ports. Use keep-alive with bounded pools (chapter 13). Use a database connection pool sized to the database's capacity, not to the number of requests.

**Other levers.** Response compression at the proxy rather than in Node; caching (HTTP cache headers, in-process LRU, Redis); batching or de-duplicating identical in-flight requests; `Promise.all` for independent calls; keep logs asynchronous and small; upgrade Node for engine improvements (measure).

**Do not micro-optimise before profiling.** The gain is nearly always in I/O structure (fewer calls, parallel calls, caching), not in loop tweaks.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Load testing | `autocannon` (`-c` connections, `-d` duration, `-p` pipelining) | Produce latency percentiles and throughput for an endpoint |
| CPU flame graphs | `0x`, `clinic flame`, `node --cpu-prof`, DevTools | Locate the hot function in a slow endpoint |
| Diagnosis | `clinic doctor`, `clinic bubbleprof` | Classify a slowdown as CPU, loop, I/O or GC |
| Built-in profiler | `node --prof`, `node --prof-process` | Read a tick profile |
| Live inspection | `node --inspect`, `chrome://inspect` | Attach to a running instance safely over a tunnel |
| Tracing | `perf_hooks` (`performance.mark`, `measure`, `PerformanceObserver`) | Time specific code sections in production |

### Common mistakes

- Benchmarking in development mode, or with one connection.
- Optimising code that is not on the flame graph.
- Averages instead of p95 and p99.
- Logging whole request and response bodies.
- Creating clients, agents or regexes per request.
- Load testing the load generator's own bottleneck (run it on a separate machine).

### Self-check

- How do you read a flame graph, and what does the x axis mean?
- List five causes of high tail latency in a Node API.
- Why is a connection pool sized to the downstream capacity?

---

## 13. Networking: http, net, fetch and undici

**In plain words:** Most backend services talk over HTTP. This chapter covers servers, outbound requests, timeouts and DNS. Wrong timeouts and missing connection reuse cause many production errors.

### Understand

**Layers.** `net` provides TCP sockets (`net.Server`, `net.Socket`), with `tls` on top. `http` and `https` implement HTTP/1.1 clients and servers on those sockets. `http2` provides HTTP/2. Frameworks (Express, Fastify, Koa, Nest) add routing and middleware over `http`.

**A minimal server.**

```js
const http = require('node:http');
const server = http.createServer(async (req, res) => {
  res.setHeader('content-type', 'application/json');
  res.end(JSON.stringify({ ok: true }));
});
server.listen(3000, '0.0.0.0');
```

Binding: `127.0.0.1` accepts only local connections; `0.0.0.0` accepts on all IPv4 interfaces. A service that works locally but not in a container is usually bound to localhost.

**Server timeouts (HTTP/1.1).** Know these and set them deliberately:
- `server.headersTimeout`: time to receive the request headers.
- `server.requestTimeout`: time to receive the whole request.
- `server.keepAliveTimeout`: idle time to keep a connection open (default 5 seconds).
- `server.timeout`: socket inactivity timeout (default 0, no timeout, since Node 13).
Current defaults: `headersTimeout` is the smaller of 60 s and `requestTimeout`; `requestTimeout` is 300 s (since Node 18). Defaults changed over time, so check the `http` documentation for your version. A classic production failure: the load balancer's idle timeout is longer than the Node `keepAliveTimeout`, so Node closes a connection the load balancer is about to reuse, producing sporadic 502 errors. Set the Node keep-alive timeout above the load balancer's idle timeout (and keep `headersTimeout` above `keepAliveTimeout`).

**Slow client protection.** Without timeouts, slow-header and slow-body attacks hold sockets open. Terminate TLS and apply request limits at a reverse proxy (nginx, a cloud load balancer) in addition to the app-level limits. Limit body size in your framework.

**Outbound HTTP: keep-alive and pooling.** `http.Agent` manages client sockets. Keep-alive reuses connections between requests, avoiding handshake cost and ephemeral port exhaustion. Since Node 19 the global agent (`http.globalAgent`) uses keep-alive with a 5-second timeout. A new `http.Agent` still defaults to `keepAlive: false`, so configure it explicitly: `new http.Agent({ keepAlive: true, maxSockets: 50 })`. `maxSockets` caps parallel connections per host, which also acts as backpressure. A leaked, unconsumed response body holds a socket; always consume or destroy responses.

**fetch and undici.** `fetch` is a global since Node 18 (no flag needed) and is marked stable from Node 21. It is implemented by **undici**, an HTTP client written for Node, which you can also install and use directly for connection pools (`Agent`, `Pool`), tuning timeouts, `request()` with lower overhead, and pipelining control. Practical points:
- Set a timeout: `fetch(url, { signal: AbortSignal.timeout(5000) })`. Without it, `fetch` can wait for a long time.
- A non-2xx status does not reject; check `res.ok`.
- Consume the body (`await res.text()`, `res.json()` or cancel it) so the connection can be reused.
- Retries are your responsibility: bounded, with backoff and jitter, idempotent requests only.

**DNS.** By default `http` and `net` resolve names with `dns.lookup`, which uses the system resolver and the libuv thread pool (chapter 3). Node does not cache DNS results by default, so every new connection may do a lookup; keep-alive reduces this. Slow or failing DNS shows up as connect delays or `EAI_AGAIN`. Options: connection reuse, a local caching resolver (for example the node's OS-level cache or a sidecar), or a custom `lookup` function with caching, applied carefully so TTLs are respected.

**Common error codes.** `ECONNREFUSED` (nothing listening), `ECONNRESET` (peer closed abruptly, often a stale keep-alive socket or a crash), `ETIMEDOUT`, `EADDRINUSE` (port taken), `ENOTFOUND` and `EAI_AGAIN` (DNS), `EMFILE` (too many open files).

**TCP-level tools.** `net.createServer` for custom protocols; `socket.setKeepAlive`, `setNoDelay`; handle `error` and `close` on every socket.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Servers | `http.createServer`, `server.listen`, timeouts, `close` | Configure timeouts to match the load balancer |
| Outbound clients | `http.Agent`, `fetch`, `undici` `Agent`/`Pool`, `AbortSignal.timeout` | Reuse connections and bound concurrency |
| Load | `autocannon`, `curl -v`, `curl -w` timing variables | Measure connect, TLS and first-byte time |
| Socket state | `ss -tan`, `lsof -i`, `netstat` | Spot `CLOSE_WAIT` and port exhaustion |
| DNS | `dns.lookup` versus `dns.resolve`, `dig`, `getent hosts` | Explain which resolver path Node uses |

### Common mistakes

- No timeouts on outbound requests.
- `keepAliveTimeout` shorter than the load balancer's idle timeout.
- Treating a 4xx or 5xx from `fetch` as an exception.
- Not consuming response bodies.
- Creating a new agent or client for every request.
- Binding to `127.0.0.1` in a container.

### Self-check

- Why can a load balancer idle timeout cause sporadic 502 errors from a Node service?
- Why does `fetch` not reject on a 500 status?
- How does `dns.lookup` differ from `dns.resolve`, and why does it matter?
- What does keep-alive save, and what bounds its pool?

---

## 14. Configuration and environment

**In plain words:** Settings such as database addresses should come from the environment, not from the code. Environment values are always text. Check them at startup and stop early if they are wrong.

### Understand

**Principle (twelve-factor).** Configuration that differs between deployments (database URLs, keys, feature flags, ports) comes from the environment, not from code or committed files. The same build runs in every environment.

**Mechanics.** `process.env` holds string values only. `process.env.PORT` is a string, and `process.env.DEBUG = false` is stored as the string `"false"`, which is truthy. Parse and convert explicitly. Reads of `process.env` are not free; read once at startup.

**Loading `.env` files.** Node has a built-in `--env-file=<path>` flag (since Node 20.6; no longer experimental from 22.21 and 24.10) and `process.loadEnvFile()` (version not re-verified; check the `process` docs). The `dotenv` package is the long-standing alternative. Use `.env` files for local development only; in production inject variables via the orchestrator, secret manager or systemd `EnvironmentFile` with tight permissions.

**Validate at startup, fail fast.** Parse the environment into a typed, validated config object once, and exit with a clear error if anything is missing or malformed. Libraries such as `zod`, `envalid` or `joi` do this. A service that boots with a bad config and fails an hour later is a worse outcome than one that refuses to start.

```js
const { z } = require('zod');
const schema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  DATABASE_URL: z.string().url(),
  REQUEST_TIMEOUT_MS: z.coerce.number().int().positive().default(5000),
});
const config = Object.freeze(schema.parse(process.env));
module.exports = config;
```

**`NODE_ENV`.** Many libraries change behaviour based on `NODE_ENV=production` (caching, fewer debug checks). Do not use it as a general "environment name" for business logic; use a separate variable (for example `APP_ENV`) and never branch security behaviour on it carelessly. `npm ci --omit=dev` (older: `--production`) installs only runtime dependencies.

**Secrets.** Never commit them, never log them, never bake them into images. Use the platform's secret store and short-lived credentials where possible. Redact them in logs (chapter 16). Environment variables are visible to the process and to anyone who can read `/proc/<pid>/environ`; for high-sensitivity secrets consider mounted files or a secret manager with runtime fetching.

**Config design.** Group related settings, provide safe defaults, make timeouts and pool sizes configurable, and log the effective non-secret configuration at startup.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Loading | `--env-file`, `process.loadEnvFile`, `dotenv`, systemd `EnvironmentFile` | Load config the same way across dev, CI and production |
| Validation | `zod`, `envalid`, `joi` | Fail fast on invalid or missing variables |
| Secrets | secret manager of your platform, mounted files | Explain trade-offs of env vars versus files |
| Inspection | `process.env`, `/proc/<pid>/environ` | Verify what a running service actually sees |

### Common mistakes

- Treating env values as booleans or numbers without parsing.
- Defaults that silently point production at a development database.
- Committing `.env` files.
- Logging the whole config object including secrets.
- Reading `process.env` throughout the codebase instead of one validated module.

### Self-check

- Why is `if (process.env.ENABLE_X)` wrong for `ENABLE_X=false`?
- What does fail-fast validation buy you operationally?
- Why is `NODE_ENV` a poor general environment selector?

---

## 15. Security

**In plain words:** A backend receives data from unknown people and uses many third-party packages. This chapter covers common attacks and the habits that reduce risk. No single tool makes a service safe.

### Understand

**Threat model.** A backend processes untrusted input and depends on a very large tree of third-party code. Most real incidents are injection, broken authentication and authorisation, vulnerable or malicious dependencies and secret leakage.

**Prototype pollution** (an attacker changing the shared base object that all objects inherit from). JavaScript objects inherit from `Object.prototype`. Code that merges or sets keys from untrusted input (`_.merge`, deep-assign, `obj[a][b] = c` with user-controlled keys) can be tricked with keys such as `__proto__`, `constructor` or `prototype` into modifying the shared prototype, changing behaviour application-wide (authorisation flags, template options). Mitigations: validate input against a schema that rejects unknown keys; use `Map` or `Object.create(null)` for dictionaries keyed by user data; avoid unsafe recursive merge utilities and keep them patched; freeze prototypes in hardened setups (`Object.freeze(Object.prototype)` can break libraries; test first); the `--disable-proto=delete` flag removes `__proto__` access.

**ReDoS (regular expression denial of service).** Some regular expressions backtrack exponentially on crafted input (nested quantifiers such as `(a+)+$`). V8's regex engine runs on the main thread, so one request can freeze the whole process. Mitigations: avoid nested quantifiers and ambiguous alternations; cap input length before matching; prefer parsers for structured formats; use linear-time engines (for example RE2 bindings) for untrusted patterns; scan with tools such as `safe-regex` or ESLint security plugins. Remember the event loop link: this is a denial of service for all users, not only the attacker's request.

**Injection.**
- **SQL / NoSQL**: use parameterised queries or a query builder; never concatenate input. In MongoDB-style APIs, object-valued input (`{ "$ne": null }`) can change query semantics; validate types.
- **Command injection**: avoid `exec` with strings; use `execFile`/`spawn` with argument arrays and an allowlist.
- **Path traversal**: user-controlled file names containing `..`; resolve with `path.resolve` and verify the result stays inside the intended root.
- **SSRF** (server-side request forgery: tricking the server into calling an address the attacker chose): fetching user-supplied URLs can reach internal services and cloud metadata endpoints. Allowlist hosts, block private ranges after DNS resolution, and disable redirects or re-validate them.
- **XSS** and header injection: encode output for its context; set security headers (for example with `helmet` in Express-style apps).
- **Unsafe deserialisation and `eval`/`new Function`/`vm`**: do not evaluate untrusted input. `vm` is not a security boundary.

**Input validation.** Validate type, shape, length and range at the boundary, with a schema (`zod`, `ajv`, framework-native). Limit body sizes, array lengths and nesting depth. Reject rather than silently fix.

**Authentication and sessions.** Hash passwords with a slow adaptive function (`scrypt`, `argon2`, `bcrypt`); these are CPU-heavy and use the thread pool or native threads, so account for them in capacity. Verify JWT signature, algorithm, issuer, audience and expiry; do not accept `alg: none`. Compare secrets with `crypto.timingSafeEqual`. Use `crypto.randomBytes` or `crypto.randomUUID` for tokens, never `Math.random`. Rate limit login and sensitive routes.

**Dependency risk and supply chain.** A typical service has hundreds of transitive packages. Attacks include typosquatting, compromised maintainer accounts, malicious install scripts and dependency confusion between public and private registries. Hygiene:
- Commit the lockfile and install with `npm ci` (or the pnpm/yarn equivalent in frozen mode) in CI and builds, so exact versions are reproduced.
- Run `npm audit` in CI and triage by reachability and severity; it reports known advisories, not malicious behaviour, and can produce noise. `npm audit signatures` verifies registry signatures and provenance attestations where available.
- Prefer fewer, well-maintained dependencies; review new ones (maintainers, downloads, recent activity, install scripts).
- Disable lifecycle scripts for installs where possible (`--ignore-scripts`) and allow them selectively; pnpm provides finer controls (check its documentation).
- Pin or constrain versions, update through reviewed automated pull requests (Dependabot or Renovate), and do not auto-merge major updates.
- Scope private package names to an organisation and configure registries explicitly to prevent dependency confusion.
- Use two-factor authentication and provenance (`npm publish --provenance`) for your own published packages.
- Keep the runtime patched: subscribe to Node security releases.

**Node's permission model.** Node has an opt-in permission model: start with `--permission` and grant access with flags such as `--allow-fs-read`, `--allow-fs-write`, `--allow-child-process` and `--allow-worker`. It was added in Node 20 (as `--experimental-permission`) and is stable from 22.13 and 23.5. The documentation says it is a "seat belt" against accidental access by trusted code. It is not a sandbox and does not stop malicious code.

**Operational controls.** Run as a non-root user, read-only file system where possible, minimal container images, no debug port exposed, TLS everywhere, secrets out of logs, and least-privilege database accounts.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Dependency audit | `npm audit`, `npm audit signatures`, `npm ls`, `npm outdated`, Dependabot or Renovate | Triage an audit report and justify what to ignore |
| Reproducible installs | `package-lock.json`, `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable` | Guarantee CI installs exactly what was reviewed |
| Static analysis | ESLint security plugins, `safe-regex`, CodeQL, Semgrep | Catch injection and ReDoS patterns in review |
| Validation | `zod`, `ajv` | Reject unknown keys and oversized input at the boundary |
| Hardening | `helmet`, rate limiters, non-root containers | Apply sensible defaults to an HTTP service |
| Secret hygiene | log redaction (`pino` `redact`), secret scanners | Prove secrets cannot reach logs |

### Common mistakes

- Merging request bodies into objects with a recursive utility.
- Regexes on user input with no length cap.
- `exec` with string interpolation.
- Fetching arbitrary user URLs from inside a private network.
- Running `npm install` (which can alter the lockfile) in CI instead of `npm ci`.
- Treating a clean `npm audit` as proof of safety.
- Running with `--inspect=0.0.0.0`.

### Self-check

- Explain prototype pollution and one real consequence.
- Why does ReDoS affect all users of a Node process?
- What does `npm audit` detect and not detect?
- List four supply chain controls you would put in CI.

---

## 16. Logging and observability

**In plain words:** Logs, metrics and traces tell you what your service is doing. Write logs as structured JSON so tools can search them. Add a request id so you can follow one request everywhere.

### Understand

**Logging principles.** Log structured JSON to **stdout** and let the platform ship it. Each event has a level, timestamp, message and context fields (request id, user id, route, duration, error). Structured logs can be searched and aggregated; free-text logs cannot. Never write to files in containers.

**pino.** A fast, low-overhead JSON logger widely used in Node services.
- Levels: `fatal`, `error`, `warn`, `info`, `debug`, `trace`.
- Child loggers add bound context: `logger.child({ reqId })`.
- `redact` removes sensitive paths (`['req.headers.authorization', '*.password']`).
- Serialisers format errors (`err`) so stack traces survive.
- Transports (pretty printing, shipping) run in worker threads so formatting does not block the main loop; use pretty-printing in development only.
- `pino-http` creates per-request loggers and logs each request.

```js
const pino = require('pino');
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  redact: ['req.headers.authorization', 'req.headers.cookie', '*.password'],
});
logger.error({ err, orderId }, 'payment failed');
```

Pass the error under the `err` key so the logger's error serialiser captures message and stack.

**What to log.** Request start or completion with status and duration, outbound call failures, state transitions, startup configuration (non-secret), shutdown, and unexpected errors with the full cause chain. Do not log secrets, tokens, full card or personal data, or entire bodies. Mind log volume: logging is synchronous-ish work that costs CPU, and a runaway error loop can fill disks and bills.

**Request context with `AsyncLocalStorage`.** Passing a request id through every function is tedious. `AsyncLocalStorage` (from `node:async_hooks`) keeps a store bound to the asynchronous execution chain started by `run()`, so any code deeper in the call chain, including after `await`, can read it.

```js
const { AsyncLocalStorage } = require('node:async_hooks');
const als = new AsyncLocalStorage();

app.use((req, res, next) => {
  als.run({ reqId: req.headers['x-request-id'] ?? crypto.randomUUID() }, next);
});

function log(obj, msg) {
  logger.info({ ...als.getStore(), ...obj }, msg);
}
```

Use it for request ids, trace context and tenant ids. It has a small overhead and context can be lost across some callback-based libraries and pools that do not preserve async context; test that the id reaches your database and queue code. This is also the mechanism distributed tracing libraries use.

**Observability pillars.**
- **Logs**: events with detail.
- **Metrics**: numbers over time (request rate, error rate, latency histograms, event loop lag, heap used, active handles, pool usage, queue depth). Prometheus client libraries (`prom-client`) are common.
- **Traces**: a request's path across services. OpenTelemetry provides Node instrumentation (`@opentelemetry/sdk-node`), which relies on async context propagation and must be loaded before other modules (version-specific setup; follow the current documentation).

**Golden signals** for a service: latency, traffic, errors, saturation. For Node, add event loop lag and GC pause time as saturation signals.

**Correlation.** Propagate a request id (or W3C `traceparent`) in outbound calls and include it in every log line, so one id finds the whole story.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Structured logging | `pino`, `pino-http`, `pino-pretty` (development only) | Configure levels, redaction and child loggers |
| Request context | `AsyncLocalStorage` | Make every log line carry the request id without parameter passing |
| Metrics | `prom-client`, `perf_hooks`, `monitorEventLoopDelay` | Expose RED metrics plus loop lag and heap |
| Tracing | OpenTelemetry Node SDK | Follow a request across two services |
| Log analysis | `jq`, your platform's log search | Answer "what failed for request X" in minutes |

### Common mistakes

- `console.log` of strings with interpolated objects instead of structured fields.
- Logging secrets or full request bodies.
- Pretty-printing in production.
- No request id, so incidents cannot be correlated.
- High-cardinality metric labels (labels with very many distinct values, such as user ids or full URLs), which are both a memory leak and a cost problem.
- Logging an error as a message string and losing the stack.

### Self-check

- Why log JSON to stdout rather than to a file in a container?
- How does `AsyncLocalStorage` carry a request id across `await`?
- Which metrics would reveal an event loop problem?

---

## 17. Testing

**In plain words:** Tests give you confidence to change code. Backend tests work best when they use real databases and real HTTP calls. Close servers and timers at the end, or the test run will not exit.

### Understand

**Layers.** Unit tests (pure logic, fast), integration tests (your code with a real database, queue or HTTP stack), contract tests (shape of calls between services) and a small number of end-to-end tests. For backends, integration tests against real dependencies (containers) give the best confidence per test.

**Options in Node.**
- **Built-in runner**: `node:test` with `node --test`, plus `node:assert`. It supports `describe`/`it`/`test`, hooks, mocking and (depending on version) coverage and snapshot features. Its stability and feature set are version-dependent; check the Node release notes and documentation for your version.
- **Jest**: batteries included (runner, assertions, mocking, snapshots, coverage); widely used; ESM support requires configuration.
- **Vitest**: Jest-compatible API, native ESM and TypeScript, fast; popular for newer projects.
- **Mocha** with **chai** or `node:assert`: minimal runner, bring your own pieces.
- **Supertest** (or `fastify.inject`, or `undici` against a listening port) for HTTP-level tests without manual sockets.
- **Testcontainers** to start real Postgres or Redis for integration tests.

**Practices.**
- Test behaviour through public interfaces; avoid asserting on implementation details.
- Control time and randomness (fake timers, injected clocks).
- Make servers testable: export an app factory, let the port be 0 (random), and close the server in teardown.
- Leaked handles (open servers, timers, pools) keep the runner from exiting; close them in `after` hooks. `why-is-node-running` helps find them.
- Isolate tests; do not depend on order or shared mutable state.
- Mock the network boundary (for example with `undici` `MockAgent` or `nock`), not internal functions.
- Test failure modes: timeouts, retries, partial failures, shutdown.
- Run tests in CI with the same Node version as production.

```js
const test = require('node:test');
const assert = require('node:assert/strict');

test('rejects invalid input', async () => {
  await assert.rejects(() => createUser({ email: 'x' }), { name: 'ValidationError' });
});
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Runner | `node --test`, Vitest, Jest | Run, filter, watch and report coverage in CI |
| HTTP testing | `supertest`, `fastify.inject`, `undici` | Test endpoints without a fixed port |
| Real dependencies | Testcontainers, docker compose | Run integration tests against a real database |
| Mocking | `node:test` mocks, `sinon`, `nock`, `MockAgent` | Mock outbound HTTP at the boundary |
| Handle leaks | `why-is-node-running`, `--detectOpenHandles` (Jest) | Explain why a test run will not exit |

### Common mistakes

- Mocking so much that tests only test the mocks.
- Tests that need a fixed port or a shared database state.
- Forgetting to close servers and pools, so the runner hangs.
- Real timers and `sleep` in tests, producing slow and flaky suites.
- Relying on a test runner feature without checking it exists in the Node version used in CI.

### Self-check

- Why does a test process sometimes not exit after all tests pass?
- What would you mock in an integration test, and what would you keep real?
- How do you test graceful shutdown?

---

## 18. Package managers and workspaces

**In plain words:** A package manager downloads the libraries your project needs. The lockfile records exact versions so every install is the same. Choose one tool per repository.

### Understand

**What they do.** Resolve a dependency graph from `package.json`, install packages into `node_modules`, record exact versions in a lockfile and run scripts.

**The main three.**

| | npm | pnpm | Yarn |
|---|---|---|---|
| Lockfile | `package-lock.json` | `pnpm-lock.yaml` | `yarn.lock` |
| Install layout | flat `node_modules` | content-addressable global store with symlinked, strict `node_modules` | Classic: flat; Berry: several modes, including Plug'n'Play |
| Reproducible CI install | `npm ci` | `pnpm install --frozen-lockfile` | `yarn install --immutable` (Berry) or `--frozen-lockfile` (Classic) |
| Workspaces | `workspaces` field | `pnpm-workspace.yaml` | `workspaces` field |

pnpm's strict layout stops code from importing packages it did not declare (phantom dependencies) and saves disk space. npm is the default, bundled with Node. Choose one per repository and enforce it.

**Semver and ranges.** `^1.2.3` allows minor and patch updates; `~1.2.3` allows patches; an exact version allows nothing. The lockfile makes installs reproducible regardless of range. Libraries should use ranges; applications benefit from the lockfile as the source of truth.

**Dependency types.** `dependencies` (runtime), `devDependencies` (build and test), `peerDependencies` (the consumer must provide, common for plugins and frameworks), `optionalDependencies`. Production images install only runtime dependencies (`npm ci --omit=dev`).

**Scripts and lifecycle.** `scripts` in `package.json` run via `npm run`. Lifecycle scripts (`preinstall`, `postinstall`, `prepare`) run on install and are a supply chain vector (chapter 15). `engines` declares supported Node versions (enforced strictly only if configured, such as `engine-strict` in npm).

**Workspaces (monorepos).** Several packages in one repository share one lockfile and link to each other locally. Benefits: atomic changes across packages, shared tooling, one dependency graph. Costs: more complex CI, publishing and versioning. Tools layered on top: Turborepo, Nx, Lerna, Changesets for versioning. Run scripts across packages with the manager's filter option (for example `npm run test --workspaces`, `pnpm --filter <name> ...`).

**Node's Corepack.** Corepack manages pnpm and Yarn versions per project through the `packageManager` field. Corepack is bundled with Node from 14.19 up to, but not including, 25.0. From Node 25 you install it yourself (for example `npm install -g corepack`).

**Good habits.** Commit the lockfile; one package manager per repo; `npm ci` in CI; review lockfile diffs; remove unused dependencies (`npm ls`, `depcheck`); avoid global installs in builds; use `npm pack --dry-run` to see what you would publish.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Install and lock | `npm ci`, `npm install`, `pnpm install`, `yarn install` | Explain the difference between install and a frozen install |
| Inspect | `npm ls <pkg>`, `npm explain`, `npm outdated`, `pnpm why` | Find why a package is in the tree and which version |
| Workspaces | `workspaces`, `pnpm-workspace.yaml`, `--filter`, `--workspaces` | Build and test a multi-package repo in CI |
| Versioning | semver, `npm version`, Changesets | Release packages with correct semver bumps |
| Overrides | `overrides` (npm), `pnpm.overrides`, `resolutions` (Yarn) | Patch a vulnerable transitive dependency temporarily |

### Common mistakes

- Deleting the lockfile to "fix" an install.
- Mixing package managers in one repository.
- Running `npm install` in CI, which may change versions.
- Putting build tools in `dependencies` and bloating production images, or runtime packages in `devDependencies` and breaking production.
- Relying on phantom dependencies that only exist because another package pulled them in.
- Unreviewed lockfile changes in pull requests.

### Self-check

- What is the difference between `npm install` and `npm ci`?
- What problem does pnpm's layout solve?
- When is `peerDependencies` the right choice?

---

## 19. TypeScript in Node

**In plain words:** TypeScript checks types while you write code, but the types disappear at run time. Data from outside still needs checks while the program runs. This chapter shows three ways to run TypeScript in Node.

### Understand

**What TypeScript is and is not.** TypeScript adds static types to JavaScript and checks them at build time. It does not change runtime behaviour, and types are erased. Therefore it cannot validate data that arrives at runtime (request bodies, environment variables, database rows). Use runtime validation (`zod`, `ajv`) at the boundaries and derive types from the schemas.

**How TypeScript runs on Node.** Conceptually three approaches:
1. **Compile first**: `tsc` (or a bundler such as `esbuild`) emits JavaScript to `dist/`; production runs `node dist/server.js`. This is the most conventional and predictable for production.
2. **Run through a loader or runner** (`tsx`, `ts-node`, SWC-based tools): convenient for development and scripts; avoid in production where startup cost and tooling risk matter.
3. **Native type stripping**: recent Node versions can run `.ts` files by stripping type annotations without type checking. It was added in Node 22.6 behind a flag, is on by default from 22.18 and 23.6 (disable with `--no-strip-types`), and is marked stable from 24.12 and 25.2. Features that need code generation (enums, namespaces with values, parameter properties) fail with `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`. You still need `tsc --noEmit` (or an equivalent) in CI for type checking. The `tsconfig` option `erasableSyntaxOnly` flags unsupported syntax at build time.

**Configuration that matters.** `strict: true`; `module` and `moduleResolution` set to match how Node loads your code (`nodenext` is the setting that follows Node's rules for ESM and CommonJS); `target` appropriate for your Node version; `outDir`, `rootDir`, `sourceMap`. Run Node with source map support (`--enable-source-maps`) so stack traces point to TypeScript lines.

**Practices.** Avoid `any`; prefer `unknown` plus validation; type error results; keep types for external APIs generated from the schema or OpenAPI document; run type checks in CI; do not ship `devDependencies` or sources unless needed; watch for ESM and CommonJS mismatches, the main source of confusing build errors.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Compilation | `tsc`, `tsconfig.json`, `esbuild`, `swc` | Produce a production build and explain each tsconfig option you set |
| Dev running | `tsx`, `ts-node`, native stripping (version-dependent) | Run TypeScript quickly in development |
| Runtime validation | `zod`, `ajv`, `typebox` | Derive types from validation schemas |
| Stack traces | `--enable-source-maps` | Read production errors in original source locations |
| Type checking | `tsc --noEmit` in CI | Separate type checking from execution |

### Common mistakes

- Trusting types for data from outside the process.
- Using `as` casts to silence errors.
- A `tsconfig` module setting that disagrees with `package.json` `"type"`.
- Running through a loader in production for convenience.
- Assuming native type stripping type-checks.

### Self-check

- Why can TypeScript not protect you from a malformed request body?
- Name three ways TypeScript code can run on Node and one trade-off of each.
- What does `moduleResolution: nodenext` align with?

---

## 20. Production deployment

**In plain words:** Production means real users, so the service must restart after failure and stop cleanly. Containers, process managers and health checks help with this. Use the checklist at the end.

### Understand

**Goal.** A repeatable build, a process that restarts on failure, safe rollouts, visibility, and defined behaviour on shutdown and overload.

**Process supervision options.**

| Option | Role | Notes |
|---|---|---|
| systemd | OS-level supervisor: restart policy, resource limits, logging via journald, boot start | See chapter 5 of the Linux guide. Set `Restart=on-failure`, `TimeoutStopSec`, `LimitNOFILE`, run as a non-root user |
| pm2 | Node process manager: cluster mode (`-i`), zero-downtime `reload`, ecosystem file, log handling | Commonly started at boot by systemd. In containers use `pm2-runtime`, or better, run Node directly |
| Containers (Docker plus an orchestrator) | Packaging plus restart, health, scaling, limits | One Node process per container; scale replicas |

**Containers: habits.**
- Use a specific LTS base image tag; slim or distroless variants reduce size and attack surface.
- Multi-stage builds: install and build in one stage, copy only production dependencies and compiled output to the final stage.
- `npm ci --omit=dev`, `NODE_ENV=production`.
- Run as a non-root user (official Node images include a `node` user).
- `CMD ["node", "dist/server.js"]` in exec form so `node` is PID 1 and receives `SIGTERM`; do not wrap in `npm start` or a shell script that does not forward signals. Use `--init` or `tini` when spawning children.
- Set the memory limit and a matching `--max-old-space-size` (chapter 11).
- Use a `.dockerignore`, pin the lockfile, and avoid baking secrets.
- Consider the CPU limit: a container limited to under one CPU throttles a Node process, which appears as event loop lag.

**Health checks.** Distinguish:
- **Liveness** (a probe: "is the process alive?"): is the process healthy enough to keep running? Should be cheap and must not depend on downstream services (otherwise a database outage restarts every replica).
- **Readiness**: should this instance receive traffic now? Fails during startup, overload or shutdown, and can reflect critical dependency state.
- **Startup** checks for slow starts.
A health endpoint is served by the same event loop; if the loop is blocked, the check fails, which is a useful signal.

**Rollouts.** Rolling updates with readiness gates; graceful shutdown on termination (chapter 9); backward-compatible database migrations deployed before code that needs them; ability to roll back quickly; canary or blue-green for risky changes. With pm2, `pm2 reload` restarts workers one at a time for zero downtime; `restart` does not.

**Resilience habits.** Timeouts on every call; bounded retries with backoff; circuit breakers or bulkheads for flaky dependencies; rate limiting and load shedding (reject early with 503 when event loop lag is high or queues are full); bounded request body sizes; connection pool limits; idempotency keys for retried writes.

**Operational checklist.**
1. Pinned Node LTS and lockfile build.
2. Config validated at startup; secrets from the platform.
3. Non-root, resource-limited process under a supervisor.
4. Structured logs on stdout with request ids.
5. Metrics: rate, errors, latency, loop lag, heap, pool usage.
6. Liveness and readiness endpoints.
7. Graceful shutdown tested under load.
8. Crash policy: exit and restart, alert on restart loops.
9. Debug access (`--inspect`) only via localhost and tunnels, never public.
10. Runbook for the common failures: OOM, loop lag, connection exhaustion, dependency outage.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Process manager | `pm2` (`start`, `reload`, `ecosystem.config.js`, `logs`, `pm2-runtime`), systemd units | Deploy and reload an app with no dropped requests |
| Containers | Dockerfile multi-stage builds, `docker run --memory`, `--init`, `docker stats` | Build a small non-root image that stops cleanly |
| Health | liveness and readiness endpoints, `curl`, orchestrator probes | Design probes that do not cause cascading restarts |
| Load behaviour | `autocannon`, `clinic doctor` | Show how the service behaves at and beyond capacity |
| Debugging in place | `node --inspect` over an SSH tunnel, `why-is-node-running`, heap snapshot signals | Diagnose a live instance without exposing it |

### Common mistakes

- `npm start` as the container command, so signals are lost and shutdown takes the full grace period.
- Liveness checks that call the database.
- Running as root, or with the debugger bound publicly.
- One huge image containing dev tools and compilers.
- `pm2 restart` for deployments instead of `reload`, causing downtime.
- No memory or CPU limits, then surprise OOM kills.
- Clustering inside a container that already has replicas, multiplying complexity.

### Self-check

- What belongs in a liveness check and what does not?
- Why `CMD ["node", "server.js"]` instead of `npm start`?
- How would you deploy with zero downtime using pm2, and using an orchestrator?

---

## 21. Practice labs

**In plain words:** These labs let you practise each idea by running small experiments. Try each one before you look for answers. Write down what you predicted and what really happened.

Do these on a machine with a recent LTS Node. Each lab states the goal and what "done" looks like. Attempt before looking anything up.

1. **Event loop ordering.** Write a script with `setTimeout`, `setImmediate`, a promise, `process.nextTick` and I/O callbacks. Predict the order, run it, then explain every difference. Repeat as an ES module and observe the change.
2. **Block the loop.** Build an HTTP server with one endpoint that runs a synchronous 300 ms computation. Measure other endpoints with `autocannon`, record p99 latency and `monitorEventLoopDelay`, then fix it with a worker thread and show the improvement.
3. **Thread pool saturation.** Fire eight concurrent `crypto.pbkdf2` calls and time them. Rerun with `UV_THREADPOOL_SIZE=8`. Explain the difference and why raising it beyond the core count stops helping.
4. **Bounded concurrency.** Process 5,000 URLs or items with an in-flight limit you choose. Show that the limit holds by logging active count, and compare with unbounded `Promise.all`.
5. **Unhandled rejection hunt.** Write code with a forgotten `await`, an `async` event listener and `forEach(async ...)`. Observe the failures, then add lint rules and fixes.
6. **Backpressure.** Write a fast producer and a deliberately slow consumer, first ignoring the `write()` return value and then using `pipeline`. Plot `process.memoryUsage()` for both.
7. **Stream a large file.** Generate a multi-gigabyte file, compress and transform it with `pipeline` in constant memory, and serve it over HTTP with abort handling when the client disconnects.
8. **Encoding bug.** Reproduce a multi-byte character corrupted by per-chunk `toString()`, then fix it with `setEncoding` and with `Buffer.concat`.
9. **Graceful shutdown.** Add `SIGTERM` handling with readiness failing, `server.close`, idle connection closing, dependency teardown and a deadline. Prove with `autocannon` running while you send `SIGTERM` that no request fails.
10. **Crash policy.** Trigger an `uncaughtException` and an `unhandledRejection`. Log them with context, exit non-zero, and have a supervisor (pm2 or systemd) restart with backoff.
11. **Leak hunt.** Add an unbounded cache and a per-request listener leak to a service. Find both using heap snapshots and the retainer view, then fix them and show the baseline flattens.
12. **Heap limit versus container limit.** Run a memory-hungry script in a container with a memory limit, once with and once without a matching `--max-old-space-size`. Distinguish a JavaScript heap OOM from a kernel OOM kill by exit code and message.
13. **Flame graph.** Profile a slow endpoint with `0x` or `clinic flame`, identify the hot function, change it and show the before and after graphs.
14. **Keep-alive.** Make many outbound requests with and without keep-alive and a bounded agent. Compare latency and the number of sockets in `ss -tan`.
15. **Timeout mismatch.** Run Node behind nginx or another proxy with a longer idle timeout than Node's `keepAliveTimeout`, reproduce sporadic 502 or reset errors, then fix the timeouts.
16. **ReDoS.** Write a vulnerable regular expression, time it with crafted input, observe the frozen server, then fix it with a safer pattern and an input length cap.
17. **Prototype pollution.** Use a naive recursive merge with `{"__proto__": {"admin": true}}`, observe the impact, then fix it with schema validation or `Object.create(null)`.
18. **Request context.** Add `AsyncLocalStorage` so a request id appears in every pino log line, including inside a database call after several awaits.
19. **Supply chain drill.** Add a dependency, inspect its install scripts and tree, run `npm audit` and `npm audit signatures`, and set CI to use `npm ci`.
20. **Production image.** Containerise a service with a multi-stage build, non-root user, exec-form `CMD`, health endpoints and a memory limit. Verify it stops in under a second on `docker stop`.
21. **Incident write-up.** Cause a failure (leak, loop block or connection exhaustion), diagnose it from metrics and profiles only, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 22. Interview questions

**In plain words:** This chapter lists common interview questions with short model answers. Start with the Top 10, then read your own level. Practise saying each answer aloud with one example.

Grouped by level and tagged [Fresher], [Mid] or [Senior]. Every question has a concise model answer. Answers are intentionally short; expand them with an example when you practise. This guide supports preparation; it does not promise any job outcome.

### Top 10 must-know

Learn these first. Each points to its full answer by number.

1. **[Fresher] Is Node.js single-threaded?** Your JavaScript runs on one thread, but Node also uses a thread pool, garbage collection threads and optional workers. (Q1)
2. **[Fresher] What is the difference between `nextTick`, `setImmediate` and `setTimeout(fn, 0)`?** `nextTick` runs before the loop continues, `setImmediate` runs in the check phase, `setTimeout` runs in the timers phase. (Q5)
3. **[Fresher] Why not use `fs.readFileSync` in a request handler?** It blocks the one JavaScript thread, so every other request waits. (Q10)
4. **[Mid] What uses the libuv thread pool?** `fs`, `dns.lookup`, some `crypto` and `zlib`; the default size is 4. (Q15)
5. **[Mid] How do you find a blocked event loop?** Watch p99 loop delay, then profile with `--cpu-prof` or a flame graph. (Q16)
6. **[Mid] What is backpressure?** A signal that slows a fast producer to the speed of its consumer (`write()` returns `false`, then wait for `'drain'`). (Q17)
7. **[Mid] What happens on an unhandled promise rejection?** Modern Node throws and crashes the process; keep that default and restart under a supervisor. (Q20)
8. **[Mid] How do you shut down gracefully?** Handle `SIGTERM`, fail readiness, `server.close()`, drain with a deadline, close dependencies, exit. (Q25)
9. **[Senior] How do you find a memory leak?** Track post-GC heap trend, compare heap snapshots, follow retainers, fix, re-measure. (Q34)
10. **[Senior] Why does Node give sporadic 502 errors behind a load balancer?** Node's `keepAliveTimeout` is shorter than the balancer's idle timeout. (Q38)

### Fresher

**1. [Fresher] Is Node.js single-threaded?**
Your JavaScript runs on one main thread per isolate. Node itself uses other threads: the libuv thread pool (fs, dns.lookup, crypto, zlib), V8 garbage collection helpers and optional worker threads. Concurrency comes from non-blocking I/O, not parallel JavaScript.

**2. [Fresher] What is the event loop?**
A loop (implemented in libuv) that takes completed I/O events, timers and scheduled callbacks and runs their JavaScript callbacks one at a time on the main thread. It lets one thread handle many connections because waiting costs no CPU.

**3. [Fresher] What does "non-blocking" mean, and what does it not mean?**
An operation returns immediately and notifies you later, so the thread can do other work. It does not mean parallel JavaScript, and it does not protect you from CPU-heavy code, which still blocks everyone.

**4. [Fresher] Explain callbacks, promises and async/await.**
Callbacks pass a function to run on completion (error-first by convention). Promises represent a future value and chain with `then` and `catch`. `async/await` is syntax over promises that reads sequentially; an `async` function always returns a promise, and thrown errors become rejections.

**5. [Fresher] What is the difference between `process.nextTick`, `setImmediate` and `setTimeout(fn, 0)`?**
`nextTick` runs right after the current operation, before promise jobs and before the loop continues. `setImmediate` runs in the check phase after poll. `setTimeout(0)` runs in the timers phase after at least about one millisecond. Overusing `nextTick` can starve I/O.

**6. [Fresher] What is `Promise.all` and how does it behave on failure?**
It waits for all promises and resolves with an array of results, or rejects with the first rejection. The other promises are not cancelled and keep running. Use `allSettled` when you need every outcome.

**7. [Fresher] What are CommonJS and ES modules, in short?**
CommonJS uses `require` and `module.exports`, loads synchronously, and is the older Node format. ES modules use `import` and `export`, load asynchronously, support top-level await and are the language standard. The file extension or the `type` field in `package.json` selects the format.

**8. [Fresher] What is a Buffer?**
A fixed-size sequence of raw bytes (a `Uint8Array` subclass) used for binary data such as network and file I/O. Strings are an encoding of bytes, such as UTF-8, base64 or hex.

**9. [Fresher] What is `package-lock.json` for, and what does `npm ci` do?**
The lockfile records the exact resolved version of every dependency so installs are reproducible. `npm ci` installs exactly what the lockfile says, deletes existing `node_modules` first and fails if `package.json` and the lockfile disagree. Use it in CI and builds.

**10. [Fresher] Why must you not use `fs.readFileSync` in a request handler?**
It blocks the main thread for the duration of the read, so every other request waits. Use the asynchronous or promise API, or a stream. Synchronous calls are acceptable at startup.

**11. [Fresher] What is an `EventEmitter` and what happens on an unhandled `error` event?**
A synchronous publish and subscribe object that many core classes extend. Emitting `error` with no listener throws, which can crash the process. Always attach an `error` handler to sockets, streams and child processes.

**12. [Fresher] How do you read configuration in a Node service?**
From environment variables, read once at startup into a validated config object. Values are strings and need parsing. Secrets come from the platform, not from the repository.

### Mid

**13. [Mid] Describe the event loop phases.**
Timers, pending callbacks, idle/prepare, poll (retrieve and run I/O callbacks, and may wait), check (`setImmediate`), close callbacks. Between each callback Node drains `nextTick` and promise microtask queues.

**14. [Mid] Why does `setImmediate` run before `setTimeout(0)` inside an I/O callback?**
After the poll phase, the loop moves to the check phase, where `setImmediate` runs, before returning to the timers phase. From the main module the order is non-deterministic because it depends on how quickly the loop starts relative to the timer.

**15. [Mid] What uses the libuv thread pool and what is its default size?**
`fs` operations, `dns.lookup`, some `crypto` functions (`pbkdf2`, `scrypt`, `randomBytes`) and `zlib`. Default size is 4, set with `UV_THREADPOOL_SIZE`. Sockets do not use it. A slow `dns.lookup` or many hashes can therefore delay unrelated file and crypto work.

**16. [Mid] How do you detect that the event loop is blocked?**
Measure loop delay with `monitorEventLoopDelay` (p99) and event loop utilization, watch for all endpoints slowing together and health checks timing out, then profile with `--cpu-prof`, `0x`, `clinic flame` or DevTools to find the wide frame.

**17. [Mid] What is backpressure and how is it implemented?**
A mechanism that slows a producer to the speed of its consumer. `writable.write()` returns `false` when the internal buffer exceeds `highWaterMark`; the producer waits for `'drain'`. `pipeline` and `pipe` do this automatically. Ignoring it grows memory.

**18. [Mid] Why prefer `stream.pipeline` over `.pipe()`?**
`pipeline` forwards errors, destroys all streams on failure and supports promises and async generators. `.pipe()` does not clean up on errors, leaking descriptors and leaving streams open.

**19. [Mid] When do you stream and when do you buffer?**
Stream large or unbounded data, slow consumers and proxying. Buffer small bounded payloads and anything you must fully parse (with a size limit enforced).

**20. [Mid] What are common causes of unhandled promise rejections?**
A forgotten `await`, `async` callbacks passed to `forEach`, `setInterval` or event emitters, missing `.catch` on fire-and-forget calls, and errors thrown after the `try` scope has ended. Modern Node crashes the process by default, which is the right policy.

**21. [Mid] How do you limit concurrency?**
Use a pool of workers pulling from a shared index, or `p-limit` or `p-map`, with a limit derived from the downstream capacity (connection pool, rate limit). Avoid unbounded `Promise.all` on large arrays.

**22. [Mid] How do you cancel an operation?**
Use `AbortController`. Pass its `signal` to `fetch`, `fs/promises`, `timers/promises` and other supporting APIs; `AbortSignal.timeout(ms)` gives a timeout signal. Cancel upstream work when the client disconnects.

**23. [Mid] What is the difference between `exec`, `execFile`, `spawn` and `fork`?**
`exec` runs a command string through a shell and buffers output (injection risk). `execFile` runs a file with an argument array, no shell, buffered. `spawn` streams output with no shell by default. `fork` starts a new Node process with an IPC channel.

**24. [Mid] Worker threads, cluster, or more processes: how do you choose?**
Worker threads for CPU-bound JavaScript within a request, using a pool. Cluster or multiple processes (or container replicas) to use more cores for an I/O-bound HTTP service and for fault isolation. Worker threads do not help I/O-bound work; cluster workers share no memory.

**25. [Mid] How do you do a graceful shutdown?**
On `SIGTERM`: fail readiness, stop accepting connections with `server.close()`, close idle keep-alive connections, wait for in-flight requests under a deadline, close pools and consumers, then exit with the right code. Set an overall timer shorter than the orchestrator's grace period.

**26. [Mid] What should happen on `uncaughtException`?**
Log with context, attempt a short, bounded cleanup, and exit non-zero so the supervisor restarts a clean process. State is undefined after an uncaught exception, so continuing is unsafe.

**27. [Mid] What does exit code 137 usually mean?**
128 plus 9: the process was killed with `SIGKILL`, commonly by the kernel out-of-memory killer in a container, or by an orchestrator after the grace period expired. Check memory limits and shutdown time.

**28. [Mid] What is the difference between `http.Agent` keep-alive and not using it?**
Keep-alive reuses TCP and TLS connections across requests, saving handshakes and avoiding port exhaustion. Without it each request opens a new connection. Configure `maxSockets` to bound concurrency, and always consume response bodies so sockets return to the pool.

**29. [Mid] Why does `fetch` not throw on a 404 or 500?**
`fetch` rejects only on network-level failures or aborts. HTTP error statuses are valid responses. Check `res.ok` or `res.status`, and set a timeout with `AbortSignal.timeout`.

**30. [Mid] What is `AsyncLocalStorage` and why use it?**
It stores context tied to an asynchronous execution chain, so a request id set at the start of a request is available in any code reached from it, including after `await`, without passing parameters. It is used for log correlation and tracing.

**31. [Mid] How do you choose between npm, pnpm and Yarn?**
They are largely interchangeable for most services. pnpm gives strict dependency isolation and disk savings, and good workspace support; npm is the default with no extra tooling. Pick one per repository, commit its lockfile and enforce frozen installs in CI.

### Senior

**32. [Senior] A Node API shows high latency on all endpoints but CPU across the host is low. Walk through your diagnosis.**
Check whether one core is saturated (single-threaded process on a multi-core host), event loop delay and utilization, thread pool saturation (DNS, fs, crypto), downstream latency and connection pool waits, GC pauses, and container CPU throttling. Profile with `--cpu-prof` or a flame graph, and compare per-endpoint timings with traces.

**33. [Senior] Explain how a single slow `dns.lookup` can stall unrelated work.**
`dns.lookup` runs on the four-thread libuv pool. If lookups hang, they occupy all threads, and `fs`, `crypto` and `zlib` calls queue behind them. CPU stays low while latency rises. Mitigations: reuse connections, cache or use a local resolver, set timeouts, and raise the pool size if measured.

**34. [Senior] How would you find a memory leak in production?**
Confirm a rising post-GC heap baseline in metrics, distinguish heap from off-heap growth (`external`, `arrayBuffers`), take heap snapshots of an instance out of rotation under steady load, compare them in DevTools, follow retainer paths for growing object types, fix, and verify the baseline flattens. Common culprits are unbounded caches, listeners, closures and timers.

**35. [Senior] How do you size `--max-old-space-size` in a container?**
Below the container memory limit, leaving headroom for off-heap buffers, thread stacks, native memory and the young generation (a starting point of roughly 70 to 80 percent, then validated by measuring RSS under load). The goal is a JavaScript heap error, with a snapshot or clear log, rather than a silent kernel OOM kill.

**36. [Senior] Design a CPU-heavy feature (image resizing) in a Node API.**
Do not run it on the request thread. Use a worker thread pool for moderate load, or a separate worker service fed by a queue for heavy or long jobs, returning a job id. Bound queue and concurrency, apply timeouts, size the pool to the cores, handle failures with retries and dead-letter handling, and expose metrics.

**37. [Senior] Compare cluster, pm2 and container replicas.**
All run multiple Node processes. Cluster is Node's built-in primary and workers sharing a port. pm2 adds management (reload, logs, restarts) on top of cluster or forks. On an orchestrator, one process per container with several replicas gives uniform scaling, health checks and limits, so in-process clustering is unnecessary. All require stateless processes.

**38. [Senior] How do you handle sporadic 502 errors between a load balancer and Node?**
Suspect a keep-alive timing mismatch: Node's `keepAliveTimeout` shorter than the load balancer's idle timeout lets Node close a connection the balancer is reusing. Set Node's timeout longer (and `headersTimeout` above it), then check for crashes, restarts, request timeouts and shutdown behaviour.

**39. [Senior] Describe your policy for errors at the process level.**
Handle expected errors where they occur and return structured errors. Treat programmer errors and uncaught exceptions as fatal: log, shut down gracefully with a deadline, exit non-zero, let the supervisor restart with backoff and alert on restart loops. Keep unhandled-rejection crash behaviour on.

**40. [Senior] How do you protect a Node service from overload?**
Timeouts on all calls, bounded concurrency and pool sizes, request size limits, rate limiting, load shedding (return 503 when event loop lag or queue depth passes a threshold), backpressure in streams, circuit breakers for dependencies, and autoscaling on saturation signals rather than CPU alone.

**41. [Senior] What are the main supply chain risks in the Node ecosystem and how do you reduce them?**
Typosquatting, compromised maintainers, malicious lifecycle scripts, dependency confusion and abandoned packages, multiplied by deep dependency trees. Reduce with lockfiles and `npm ci`, fewer and reviewed dependencies, `npm audit` and signature checks, restricted install scripts, scoped private registries, automated reviewed updates, and provenance on published packages.

**42. [Senior] Explain prototype pollution and ReDoS and why Node makes them severe.**
Prototype pollution lets attacker-controlled keys modify `Object.prototype`, altering behaviour globally; ReDoS makes a regex backtrack exponentially. In Node both run in a shared process, so one request can corrupt behaviour for all users or freeze the single event loop. Defend with schema validation, safe dictionaries, input length caps and linear-time regex engines.

**43. [Senior] How would you propagate request context across services and async code?**
Generate or accept a request id or W3C `traceparent` at the edge, hold it in `AsyncLocalStorage`, attach it to every log line, and forward it on outbound calls. OpenTelemetry automates this and relies on the same async context mechanism.

**44. [Senior] When would you choose ESM versus CommonJS for a new service, and what are the pitfalls?**
ESM is the language standard with top-level await and better tooling direction; choose it if your dependencies and tooling support it. Pitfalls: required file extensions, no `__dirname` (use `import.meta.dirname` on Node 20.11 or later), interop with CommonJS-only packages, the dual package hazard, and `require()` of ESM (works from Node 22.12 and 20.19 when the module has no top-level `await`). Make `tsconfig` and `package.json` agree.

**45. [Senior] How would you design health checks for a service and its dependencies?**
Liveness is cheap and local (the loop responds), with no downstream calls, to avoid cascading restarts. Readiness reflects ability to serve: startup complete, not shutting down, and critical dependencies reachable, possibly with caching. Add a startup probe for slow boots. Include dependency detail in a separate diagnostic endpoint.

**46. [Senior] Walk through a zero-downtime deployment of a Node service.**
Build an immutable artifact, run backward-compatible migrations first, roll out gradually with readiness gating, send `SIGTERM` to old instances so they fail readiness, drain and exit within the grace period, watch error rate and latency, and keep a fast rollback. With pm2, use `reload`, not `restart`.

**47. [Senior] A request handler must call three independent services and one dependent service. How do you structure it and handle failures?**
Start the three independent calls together with `Promise.all` (or `allSettled` if partial results are acceptable), each with a timeout via `AbortSignal`, then make the dependent call using their results. Cancel outstanding work on failure or client disconnect, bound retries to idempotent calls, and return a clear error mapped from the failure type.

---

## 23. References

**In plain words:** This chapter lists official and trusted sources. Prefer these over blog posts. Check the version in the page against the Node version you run.

Primary sources, preferred over blog posts:

- Node.js documentation (API reference for the version you run): https://nodejs.org/docs/latest/api/
- Node.js release schedule and LTS policy: https://nodejs.org/en/about/previous-releases
- Node.js changelogs and release notes: https://github.com/nodejs/node/tree/main/doc/changelogs
- Node.js guides, "The Node.js Event Loop, Timers, and process.nextTick()": https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- Node.js guides on diagnostics (memory, flame graphs, profiling): https://nodejs.org/en/learn/diagnostics/user-journey
- Node.js documentation of `stream`, `worker_threads`, `cluster`, `child_process`, `perf_hooks`, `async_hooks`, `http`, `events`, `modules` and `test`, all under the API reference above
- libuv design overview: https://docs.libuv.org/en/v1.x/design.html
- V8 documentation and blog (garbage collection, "Trash talk"): https://v8.dev/
- undici documentation: https://undici.nodejs.org/
- pino documentation: https://getpino.io/
- Clinic.js documentation: https://clinicjs.org/documentation/
- 0x flame graph tool: https://github.com/davidmarkclements/0x
- autocannon: https://github.com/mcollina/autocannon
- why-is-node-running: https://github.com/mafintosh/why-is-node-running
- pm2 documentation: https://pm2.keymetrics.io/docs/usage/quick-start/
- npm documentation (`npm ci`, `npm audit`, workspaces, `package.json`): https://docs.npmjs.com/
- pnpm documentation: https://pnpm.io/
- Yarn documentation: https://yarnpkg.com/
- OpenTelemetry JavaScript documentation: https://opentelemetry.io/docs/languages/js/
- Node.js security best practices: https://nodejs.org/en/learn/getting-started/security-best-practices
- OWASP Node.js Security Cheat Sheet and Top Ten: https://cheatsheetseries.owasp.org/ and https://owasp.org/www-project-top-ten/
- The Twelve-Factor App (configuration, logs, disposability): https://12factor.net/
- TypeScript handbook and `tsconfig` reference: https://www.typescriptlang.org/docs/
- Docker documentation on Node.js containerisation: https://docs.docker.com/guides/nodejs/

URL status: the nodejs.org, undici and getpino.io links above were checked on 2026-11-06 (the `nodejs.org/docs/latest/api/` pages resolve; the Corepack API page now redirects to the Corepack repository). Third-party links (libuv, V8, Clinic.js, 0x, autocannon, pm2, npm, pnpm, Yarn, OpenTelemetry, OWASP, Twelve-Factor, TypeScript, Docker) were not re-verified.

---

## 24. Glossary

| Term | Plain meaning |
|---|---|
| Event loop | The cycle that picks finished work and runs its JavaScript callback, one at a time. |
| libuv | The C library under Node that provides the event loop, async I/O and the thread pool. |
| V8 | The engine that runs JavaScript in Node and Chrome. |
| Isolate | One independent V8 instance with its own heap. Each worker thread has one. |
| Callback | A function you give to Node so it can call it when work finishes. |
| Microtask | A small job (promise handler, `nextTick`) that runs right after the current callback, before the loop moves on. |
| Thread pool | A small group of helper threads for work that cannot be done without blocking. |
| Blocking | Keeping the thread busy so nothing else can run on it. |
| Backpressure | Slowing a fast data source so a slow receiver is not overloaded. |
| Stream | Data handled in small pieces instead of all at once. |
| Buffer | A fixed block of raw bytes. |
| Idempotent | Safe to repeat: running it twice gives the same result as once. |
| Event loop lag | How late a timer runs because the loop was busy. |
| Percentile (p99) | The value that 99 percent of measurements are below. It shows slow cases that averages hide. |
| Garbage collection | Automatic freeing of memory that the program no longer uses. |
| Heap | The memory area where JavaScript objects live. |
| RSS | Resident set size: the physical memory a process uses. |
| Heap snapshot | A saved copy of all objects in the heap, used to find leaks. |
| Retainer | An object that keeps another object alive, so it cannot be freed. |
| Graceful shutdown | Stopping after finishing current work instead of cutting it off. |
| Signal (SIGTERM, SIGKILL) | A message from the operating system. SIGTERM asks to stop; SIGKILL forces a stop. |
| Orchestrator | A system such as Kubernetes that starts, stops and scales containers. |
| Liveness / readiness | Probes: is the process alive, and may it receive traffic now. |
| Keep-alive | Reusing one network connection for many requests. |
| Cluster | Several Node processes sharing one port. |
| Worker thread | An extra thread with its own isolate, used for CPU-heavy JavaScript. |
| ReDoS | A crafted input that makes a regular expression run for a very long time. |
| Prototype pollution | An attacker changing the shared base object that all objects inherit from. |
| SSRF | Making the server send a request to an address the attacker chose. |
| Lockfile | A file recording the exact version of every dependency. |
| Type stripping | Running TypeScript by removing the types, without checking them. |
