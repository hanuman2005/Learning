# Express for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to Express as it is used to build and operate production HTTP APIs. It teaches the mental model first, so you can reason about why a request behaves the way it does. Then it covers the mechanics and the habits that separate a demo from a service you can run on-call.

**Audience:** backend developers who know JavaScript or TypeScript and basic HTTP and want production-level Express skill.
**Prerequisites:** working Node.js (async/await, promises, modules), HTTP basics (methods, status codes, headers), and the Linux chapter in this repository for process and signal concepts.
**Scope:** Express on Node.js as a JSON API server behind a reverse proxy. Server-side rendering, template engines and WebSockets are out of scope.
**Note on accuracy:** Express 5 changed several behaviours compared with Express 4 (async error handling, path matching syntax, query parser default, removed methods). The Express 5 notes in this guide were checked against the official migration guide. Before you upgrade, read that guide for the full list. Package option names (helmet, cors, express-rate-limit, multer, pino-http) change between major versions, so confirm against each package's README. Still unverified against official pages: the default values of Node's `keepAliveTimeout`, `headersTimeout` and `requestTimeout`, and the exact `max` to `limit` rename history of express-rate-limit.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Each chapter starts with an "In plain words" summary. Chapter 19 lists hands-on labs. Chapter 20 collects interview questions with model answers. Chapter 21 lists primary references. Chapter 22 is a glossary of hard terms.

**What "good enough" means for a backend role:** you can build a small API from an empty directory with correct middleware ordering, validated input, a single error handler, authentication, structured logs and graceful shutdown. You can write tests for it without starting a real network port or database. You can explain what happens to a request from the load balancer to the response. Chapters 3, 5, 7, 10 and 14 matter most.

## Contents

1. What Express is
2. The request and response objects
3. The middleware chain
4. Routing
5. Request parsing
6. Validation at the boundary
7. Error handling
8. Project structure
9. Authentication and authorisation
10. Security middleware
11. File uploads
12. Streaming, SSE, compression and caching
13. Logging and request IDs
14. Graceful shutdown and load balancers
15. Performance
16. Testing
17. API versioning and OpenAPI
18. Express versus Fastify, Koa and NestJS
19. Practice labs
20. Interview questions
21. References
22. Glossary

---

## 1. What Express is

**In plain words.** Express is a small library for building web servers with Node.js. It helps you match a URL to a function and run shared steps for every request. It leaves everything else, such as the database and login, to you.

### Understand

**A thin layer over Node's `http` module.** Node gives you a server that emits a request event with an `IncomingMessage` (request) and a `ServerResponse` (response). Express adds three things: a **router** that maps method plus path to functions, a **middleware pipeline** (a list of functions that run in order for each request), and **helpers** on the request and response objects (`res.json`, `req.params`, `res.redirect`). It adds no database layer, no validation, no authentication and no dependency injection (passing a function its tools instead of letting it create them). Everything else comes from packages you choose.

**Why that matters.** Express is unopinionated (it does not force a structure). The quality of an Express service therefore depends on its authors: ordering, structure, error handling, security. Two Express apps can look completely different. A strong backend engineer brings the conventions Express does not.

**An `app` is a function.** `express()` returns a function `(req, res)` that you can hand to `http.createServer`. `app.listen(port)` is a shortcut that does exactly that. This matters for testing (you can pass `app` to a test client without listening on a port) and for shutdown (you need the `http.Server` handle that `listen` returns).

```js
import express from 'express';

const app = express();
app.get('/health', (req, res) => res.json({ status: 'ok' }));

const server = app.listen(3000);   // returns http.Server
```

**Single-threaded event loop.** Node runs your JavaScript on one thread. The event loop is the cycle that picks the next piece of work to run. While a handler does CPU work, no other request on that process makes progress. Express handlers should mostly wait on I/O (database, network) and return quickly. Chapter 15 covers the consequences.

**Version note.** Express 5 is the current major version line. Express 4 is still widely deployed, so you will meet both. Sections flag the differences you need to know.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Bootstrapping | `npm init`, `express`, TypeScript with `tsx` or `ts-node`, `@types/express` | Create a typed API skeleton from nothing |
| Dev loop | `nodemon` or `node --watch` | Auto-restart on change, understand that it kills and restarts the process |
| Raw http | `node:http` `createServer` | Write a tiny server without Express to see what Express adds |
| Inspecting requests | `curl -v`, `httpie`, a REST client | Reproduce any request and read headers |

### Common mistakes

- Treating Express as a framework with structure. It has none. Without conventions, handlers grow into large files mixing HTTP, business logic and SQL.
- Calling `app.listen` inside the module that defines `app`, which makes the app impossible to test without binding a port (chapter 16).
- Running `nodemon` in production. It is a development tool.

### Self-check

- What does `app.listen` return, and why do you need it for graceful shutdown?
- What three things does Express add on top of `http.createServer`?
- Why can one CPU-heavy handler slow every other request on the same process?

---

## 2. The request and response objects

**In plain words.** Every request gives you two objects. `req` holds what the client sent, and `res` is how you answer. Treat everything in `req` as unchecked input from strangers.

### Understand

`req` and `res` are the Node objects extended by Express. Know what lives where.

**Request (`req`).**

| Property | Meaning |
|---|---|
| `req.method`, `req.url`, `req.path` | Verb, raw URL, and the path without the query string |
| `req.originalUrl` | Full URL as received, before any mount-path stripping |
| `req.baseUrl` | The mount path of the current router |
| `req.params` | Values captured from path segments (`/users/:id`) |
| `req.query` | Parsed query string |
| `req.body` | Parsed body. Only exists after a body parser ran |
| `req.headers`, `req.get('name')` | Headers (names lowercased in `req.headers`) |
| `req.ip`, `req.ips`, `req.protocol`, `req.secure`, `req.hostname` | Derived from the socket, or from `X-Forwarded-*` headers when `trust proxy` is configured (chapter 10) |
| `req.cookies` | Only with a cookie parser |
| `res.locals` and `req.app` | `res.locals` carries per-request data between middleware. `req.app` is the app instance |

Everything in `req.params`, `req.query`, `req.body` and `req.headers` is **untrusted input**. Query values may be strings, arrays or (with the extended parser) nested objects. A single `?id=1&id=2` makes `req.query.id` an array. Express 5 changed the default query parser from "extended" (Express 4) to "simple". Set the `query parser` setting explicitly if you need nested objects, and validate rather than assume the shape.

**Response (`res`).**

| Method | Behaviour |
|---|---|
| `res.status(code)` | Sets status, chainable |
| `res.json(obj)` | Serialises with `JSON.stringify`, sets `Content-Type`, ends the response |
| `res.send(body)` | Sends string, Buffer or object, sets type and a weak ETag, ends |
| `res.sendStatus(code)` | Sets status and sends its text as body |
| `res.set(name, value)` | Sets a header |
| `res.cookie(name, value, opts)` | Sets a cookie |
| `res.redirect(url)` | 302 by default |
| `res.end()` | Ends with no body |
| `res.sendFile`, `res.download` | Send a file from disk (use absolute paths, never user-controlled) |

**One response per request.** After the first `res.json` or `res.send`, headers are sent and the response is finished. Trying to send again produces `ERR_HTTP_HEADERS_SENT` ("Cannot set headers after they are sent to the client"). This is the symptom of a missing `return` or a handler that both responds and calls `next()`.

**Status codes you must use correctly.** 200 OK, 201 Created (with `Location`), 204 No Content, 304 Not Modified, 400 Bad Request (malformed), 401 Unauthorized (no or invalid credentials), 403 Forbidden (authenticated but not allowed), 404 Not Found, 409 Conflict, 413 Payload Too Large, 415 Unsupported Media Type, 422 Unprocessable Content (valid syntax, failed semantic validation, a common choice for validation errors), 429 Too Many Requests, 500, 502, 503, 504. Choose one validation status (400 or 422) and use it consistently.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Reading input | `req.params`, `req.query`, `req.get()` | Know the source and type of each value |
| Writing output | `res.status().json()`, `res.set()`, `res.cookie()` | Produce consistent response shapes |
| Debugging | `curl -i`, browser devtools Network tab, `DEBUG=express:*` | See exactly what went over the wire |
| Typing | `Request<Params, ResBody, ReqBody, Query>` generics | Type handlers without trusting runtime data |

### Common mistakes

- Reading `req.body.field` without validation, then writing it to a database.
- Using `res.send` for JSON in some routes and `res.json` in others, giving inconsistent content types.
- Returning 200 with `{ "error": ... }`. Clients, caches and monitoring rely on the status code.
- Passing `req.params` or `req.query` values to `res.sendFile` or `fs` paths, which allows path traversal.

### Self-check

- What is the difference between 401 and 403?
- Why might `req.query.tag` be a string in one request and an array in another?
- What produces "Cannot set headers after they are sent to the client"?

---

## 3. The middleware chain

**In plain words.** A request passes through a list of functions, one after another. Each function can answer the client, pass the request on, or report an error. The order of the list decides how your app behaves.

### Understand

**This is the core mental model.** A request enters an ordered list of functions. Each function has the signature `(req, res, next)`. It may:

1. **Respond**, ending the chain (short-circuit).
2. **Call `next()`**, passing control to the next function.
3. **Call `next(err)`**, skipping to the first error-handling function.
4. **Do nothing**, in which case the request hangs until a client or proxy timeout.

Everything in Express is this: `express.json()` is a middleware, `helmet()` is a middleware, a route handler is a middleware that happens to match a path.

```js
app.use(requestId);        // 1. runs for every request
app.use(express.json());   // 2. parses the body
app.use('/api', apiRouter);// 3. only for paths under /api
app.use(notFound);         // 4. nothing matched
app.use(errorHandler);     // 5. four arguments, runs on errors
```

**Order is behaviour.** Functions run in the order they were registered. Consequences:

- A body parser registered after a route never affects that route. In Express 5, `req.body` is `undefined` there (Express 4 gave `{}`).
- A rate limiter registered after the route it should protect protects nothing.
- A 404 handler registered before your routers swallows every request.
- Authentication registered after a public route does not protect it, and registered before it blocks everything.

**Short-circuiting.** A middleware that responds must not call `next()` afterwards, and it must `return` so the code below does not continue.

```js
function requireApiKey(req, res, next) {
  if (req.get('x-api-key') !== process.env.API_KEY) {
    return res.status(401).json({ error: { code: 'unauthenticated' } });
  }
  next();
}
```

Without the `return`, execution continues to `next()` and a second response is attempted.

**Error-handling middleware has four parameters.** Express identifies it by `fn.length === 4`. The parameters must all be declared even if unused.

```js
// eslint-disable-next-line no-unused-vars
function errorHandler(err, req, res, next) {
  if (res.headersSent) return next(err);   // delegate to Express default
  res.status(err.status ?? 500).json({ error: { code: 'internal_error' } });
}
```

It must be registered **after** the routes and middleware it should cover. Calling `next(err)` (with a truthy argument other than the special string `'route'` or `'router'`) skips normal middleware and runs error handlers in order.

**Special `next` values.** `next('route')` skips the remaining handlers of the current route. `next('router')` exits the current router. Rarely needed. Know they exist.

**Scope.** `app.use(fn)` runs for all paths. `app.use('/path', fn)` runs for requests whose path starts with `/path`, with the prefix stripped from `req.url` inside the function (`req.originalUrl` keeps it). `app.get('/path', fn)` runs only for GET with an exact path match.

**Cross-cutting concerns belong here.** Request IDs, logging, security headers, CORS, authentication, rate limiting, parsing, timeouts, metrics. Business rules do not.

**Per-request state.** Attach data with `res.locals` or typed extensions of `req` (for example `req.user`). Do not use module-level variables for per-request data, because requests interleave on one thread.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Writing middleware | `(req, res, next)`, factory pattern `(options) => (req, res, next)` | Write a configurable middleware from memory |
| Ordering | Reading `app._router` is not stable. Use explicit comments and a single `app.ts` that lists the stack | State the order of any app's pipeline |
| Debugging flow | `DEBUG=express:router`, log lines per middleware | Find which middleware swallowed or hung a request |
| Third-party stack | `helmet`, `cors`, `compression`, `express-rate-limit`, `pino-http` | Know the correct position of each |

A conventional order for an API:

```
1. request ID and logging          (so every later log line has an ID)
2. security headers (helmet)
3. CORS
4. compression (or leave to the proxy)
5. rate limiting
6. body parsers (with limits)
7. routes (auth applied per router or route)
8. 404 handler
9. error handler (last)
```

Raw-body routes such as webhooks must be mounted before the global JSON parser (chapter 5).

### Common mistakes

- Forgetting `return` after `res.status(...).json(...)` inside a guard clause.
- Registering the error handler before the routes.
- Declaring an error handler with three arguments, so Express treats it as ordinary middleware and it never receives errors.
- A middleware that neither responds nor calls `next()`, so requests hang.
- Calling `next()` and also responding, producing a duplicate response.
- Doing slow, blocking work in a middleware that runs on every request.

### Self-check

- How does Express know a function is an error handler?
- Why does `app.use(express.json())` placed after `app.use('/orders', ordersRouter)` break POST `/orders`?
- What happens to a request when a middleware neither responds nor calls `next()`?
- What is the difference between `app.use('/api', fn)` and `app.get('/api', fn)`?

---

## 4. Routing

**In plain words.** Routing means choosing which function answers which method and URL. Routers let you split a large API into small files, one per resource. The first matching route that answers wins, so order matters here too.

### Understand

**Routes** bind a method and path pattern to one or more handlers: `app.get('/users/:id', handler)`. Handlers run in order and each may call `next()`, so route-level middleware is just extra functions before the final handler.

```js
router.get('/:id', requireAuth, requireRole('admin'), getUser);
```

**Routers.** `express.Router()` creates a mini application with its own middleware and routes. You mount it at a path. This is how an API is divided by resource.

```js
// routes/orders.js
const router = express.Router();
router.get('/', listOrders);
router.post('/', createOrder);
router.get('/:orderId', getOrder);
export default router;

// app.js
app.use('/api/v1/orders', ordersRouter);
```

**Chaining on one path.** `router.route('/:id').get(a).put(b).delete(c)` avoids repeating the path.

**Path parameters.** `/users/:id` fills `req.params.id` with a string. Parameters from a parent mount are not visible in a child router unless the child is created with `express.Router({ mergeParams: true })`. `router.param('id', fn)` runs a function whenever that parameter appears, which can load and validate a resource once. Use it sparingly, because it hides data loading from the handler.

**Query strings** carry filters, pagination and sorting: `GET /orders?status=paid&limit=20&cursor=abc`. They are strings, and need validation and coercion (chapter 6).

**Path syntax changed in Express 5.** Express 5 uses a newer `path-to-regexp` with stricter syntax (confirmed in the migration guide):
- Wildcards must be named: `'/*'` becomes `'/*splat'`. `*splat` does not match `/`, so use `'/{*splat}'` to include it. The captured value is an array.
- The `?` for optional parts is gone. Use braces: `'/:file.:ext?'` becomes `'/:file{.:ext}'`. An unmatched optional parameter is omitted from `req.params`.
- Regular expression characters in path strings are not supported. Use an array of paths instead. The characters `()[]?+!` are reserved.
- Plain parameterised paths like `/users/:id` are the same in both.

**Route order matters.** The first matching route that responds wins. A catch-all or a parameterised route registered before a specific one hides it: `/users/:id` registered before `/users/me` captures `me` as an id.

**HTTP method semantics.** GET is safe and cacheable and must not change state. PUT replaces and is idempotent. PATCH partially updates. DELETE is idempotent. POST creates or triggers an action and is not idempotent by default. Design retries and idempotency keys around these properties.

**Resource design for APIs.** Nouns in paths (`/orders/:id/items`), verbs from HTTP methods, plural collections, consistent pagination (prefer cursor-based for large or changing data), and consistent error shape.

**Mounting and `trust`.** Apps can also be mounted: a sub-application via `app.use('/admin', adminApp)`. Prefer routers for a single deployable API.

**404.** If nothing matches, Express's default responds with an HTML 404. An API should register its own final middleware that returns a JSON 404.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Routers | `express.Router`, `app.use(path, router)`, `mergeParams` | Split an API by resource with nested routes |
| Parameters | `req.params`, `router.param`, `router.route()` | Capture, validate and reuse path parameters |
| Route-level middleware | multiple handlers per route | Attach auth or validation to specific routes |
| Inspecting routes | `express-list-endpoints` or manual listing in tests | Verify the route table in a test |
| Query handling | `qs` behaviour via the `query parser` setting | Explain how nested and repeated parameters parse |

### Common mistakes

- Registering `/:id` before a static sibling route.
- Forgetting `mergeParams: true` so `req.params.userId` is undefined in a nested router.
- Using GET for actions with side effects, which crawlers and prefetchers can trigger.
- Copying Express 4 wildcard or optional-parameter patterns into Express 5.
- No JSON 404, so clients receive an HTML page.

### Self-check

- Why does `/users/:id` before `/users/me` break `/users/me`?
- When do you need `mergeParams: true`?
- Which HTTP methods are idempotent, and why does that matter for client retries?
- How do you write a wildcard route and an optional segment in Express 5?

---

## 5. Request parsing

**In plain words.** A request body arrives as a stream of bytes. A body parser reads it and puts the result in `req.body`. Always set a size limit, and use the raw bytes when you must check a signature.

### Understand

Node delivers the body as a stream. Express does not read it by default. A **body parser** middleware reads the stream, decodes it according to `Content-Type`, and sets `req.body`.

| Parser | Handles | Notes |
|---|---|---|
| `express.json()` | `application/json` | Rejects invalid JSON with a 400. Strict by default (objects and arrays only) |
| `express.urlencoded({ extended })` | HTML form posts | `extended: true` uses `qs` allowing nested objects. The default is `false` in Express 5. Set it explicitly |
| `express.raw()` | Bytes as a `Buffer` | Needed for signature verification |
| `express.text()` | Plain text | Rare in APIs |

Since Express 4.16 these parsers are built in (they previously came from the `body-parser` package, which still exists).

**Always set limits.** The default JSON limit is 100 kilobytes. Set it deliberately: `express.json({ limit: '100kb' })`. An unlimited or very large limit lets one client exhaust memory. Oversized bodies produce a 413. Apply larger limits only on routes that need them, by mounting a parser on a specific router.

**Content-Type gating.** A parser only acts when the request `Content-Type` matches its `type` option. A client that sends JSON with the wrong or missing `Content-Type` is not parsed. In Express 5, `req.body` is `undefined` whenever the body has not been parsed (Express 4 gave `{}`). Let validation reject it. You can return 415 for unexpected content types.

**Raw bodies for webhook signature verification.** A webhook is a request that another service sends to your URL. Providers sign (add a keyed hash, called an HMAC, over) the exact bytes they sent. If you parse the JSON and re-serialise it, key order or whitespace can change and the signature no longer matches. The solution is to read the raw bytes for that route.

```js
// Mount BEFORE the global express.json() so the stream is not consumed
app.post('/webhooks/payments',
  express.raw({ type: 'application/json', limit: '1mb' }),
  (req, res) => {
    const signature = req.get('x-signature');
    if (!verifyHmac(req.body /* Buffer */, signature, process.env.WEBHOOK_SECRET)) {
      return res.sendStatus(401);
    }
    const event = JSON.parse(req.body.toString('utf8'));
    // ... enqueue, then respond fast
    res.sendStatus(200);
  });

app.use(express.json({ limit: '100kb' }));
```

An alternative is the `verify` option of `express.json({ verify: (req, res, buf) => { req.rawBody = buf; } })`, which captures the buffer while still parsing. Either way, compare signatures with a constant-time function (`crypto.timingSafeEqual`), check a timestamp to limit replay, and respond quickly, doing heavy work asynchronously. Follow the provider's own documented verification method.

A body can be consumed only once. If a parser already read the stream, a later parser sees nothing. This is the "body parser ordering" bug.

**Query and params parsing.** `req.query` is parsed by the `query parser` setting. Do not rely on nested structures from query strings unless you chose that setting deliberately.

**Malformed requests.** Invalid JSON, wrong charset or oversized bodies raise errors from the parser with a `status` (400, 413, 415). Your central error handler should map them to clean responses instead of leaking a stack trace.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| JSON and forms | `express.json`, `express.urlencoded` | Configure limits and types explicitly |
| Raw bodies | `express.raw`, the `verify` option | Verify an HMAC webhook signature correctly |
| Constant-time compare | `crypto.timingSafeEqual`, `crypto.createHmac` | Compute and compare a signature safely |
| Testing | `curl -d`, `curl -H 'Content-Type: ...'` | Send wrong types and oversized bodies deliberately |

### Common mistakes

- Registering `express.json()` globally before a webhook route that needs the raw body.
- Leaving the body limit at default without thinking, or raising it globally to make one route work.
- Comparing signatures with `===` (timing leak) or on re-serialised JSON.
- Assuming `req.body` is always an object.
- Parsing a body on routes that never use one.

### Self-check

- Why must a webhook route see the raw bytes, and how do you arrange it?
- What happens if the client sends JSON without a JSON `Content-Type`?
- Why can a body be read only once?
- Why set a body size limit, and what status does exceeding it produce?

---

## 6. Validation at the boundary

**In plain words.** Data from the network can be anything. Check its shape and values once, at the edge of your app, and then use only the checked result. This stops many bugs and attacks.

### Understand

**The boundary rule.** Data crossing from the network into your code is `unknown`. Validate and convert it once, at the edge, into a typed value. Inside the boundary, services work with trusted types. This prevents injection, type confusion, mass assignment and a large class of crashes.

**Validate everything you read:** body, query, params, and sometimes headers. Validate shape (fields, types), constraints (lengths, ranges, formats, enums) and, separately, business rules (a uniqueness check is a service concern, not a schema concern).

**Libraries.** **zod** is widely used in TypeScript projects because one schema gives both runtime validation and a static type. Alternatives include Joi, Ajv with JSON Schema, and Valibot. JSON Schema has the advantage of also feeding OpenAPI (chapter 17).

```js
import { z } from 'zod';

const createOrderBody = z.object({
  customerId: z.string().uuid(),
  items: z.array(z.object({
    sku: z.string().min(1).max(64),
    quantity: z.number().int().positive().max(100),
  })).min(1).max(50),
}).strict();                       // reject unknown keys

const listQuery = z.object({
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
});

export const validate = (schema, source = 'body') => (req, res, next) => {
  const result = schema.safeParse(req[source]);
  if (!result.success) {
    return res.status(422).json({
      error: { code: 'validation_failed', details: result.error.issues },
    });
  }
  req.validated = { ...req.validated, [source]: result.data };
  next();
};

router.post('/', validate(createOrderBody), createOrder);
```

Notes on that snippet:
- Query and path values arrive as strings, so numeric fields need coercion (`z.coerce`) or explicit parsing.
- `.strict()` rejects unexpected keys. Alternatively strip them. The point is to never pass an arbitrary client object into a database update (**mass assignment**: a client setting `isAdmin: true`).
- Handlers read from the validated, typed result, not from the raw `req.body`.
- Validation errors are client errors (4xx) and should list field issues in a stable machine-readable shape. They must not echo secrets.
- Express 5 makes `req.query` a getter (a read-only property), so you cannot assign coerced values back to it. Store results on a separate property such as `req.validated`.
- In zod 4, `.strict()` on an object still works, and `z.strictObject()` and `z.uuid()` are the preferred forms. Check the zod docs for your version.

**Validate responses in tests, and sometimes in production.** A response schema catches accidental leakage of fields such as password hashes. Explicit DTO mapping (building the response object field by field) is the most reliable guard.

**Where not to validate.** Do not rely on client-side validation. Do not duplicate the same rules in every handler. Centralise schemas per resource.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schemas | `zod` (`parse`, `safeParse`, `coerce`, `refine`, `infer`) | Define and reuse schemas, derive TypeScript types |
| Alternatives | `ajv` with JSON Schema, `joi`, `valibot` | Know the trade-offs, especially OpenAPI reuse |
| Middleware pattern | a `validate(schema, source)` factory | Apply validation per route in one line |
| Error shape | a consistent `{ error: { code, message, details } }` body | Give clients actionable, stable errors |

### Common mistakes

- Trusting `req.body` shape because the frontend "always sends it correctly".
- Passing `req.body` directly to an ORM `create` or `update`.
- Forgetting to coerce query strings, then comparing `"20" > 5` in a surprising way.
- Returning raw validation library internals that change between versions.
- Validating in the controller after already using the data.

### Self-check

- What is mass assignment and how does validation plus DTO mapping prevent it?
- Why do query values need coercion?
- Why is a uniqueness check not a schema rule?

---

## 7. Error handling

**In plain words.** Some errors are normal, such as a missing record. Others are bugs. Send the client a safe, short message, keep the full details in your logs, and handle all errors in one place.

### Understand

**Two kinds of error.**

| Type | Examples | Right response |
|---|---|---|
| **Operational** | invalid input, record not found, upstream timeout, database connection refused, permission denied | Expected at runtime. Map to a status code, log at the right level, continue serving |
| **Programmer** | `TypeError`, undefined property, wrong argument, broken invariant | A bug. Log with full detail, return a generic 500, and treat the process state as suspect |

For programmer errors that escape (an `uncaughtException`, or an unhandled rejection), the safe action is to log, stop accepting traffic, finish in-flight work if possible and exit, letting the supervisor restart a clean process. Continuing with unknown corrupted state is riskier than a restart.

**A centralised handler.** Throw or pass typed errors from anywhere. Translate them in exactly one place.

```js
export class AppError extends Error {
  constructor(status, code, message, details) {
    super(message);
    this.status = status;
    this.code = code;
    this.details = details;
    this.expose = true;           // safe to show the message to the client
  }
}
export const notFound = (what) => new AppError(404, 'not_found', `${what} not found`);

// last middleware
export function errorHandler(err, req, res, next) {
  if (res.headersSent) return next(err);

  const isApp = err instanceof AppError;
  const status = isApp ? err.status : (err.status >= 400 && err.status < 500 ? err.status : 500);

  req.log.error({ err, status }, 'request failed');     // full detail goes to logs only

  res.status(status).json({
    error: {
      code: isApp ? err.code : 'internal_error',
      message: isApp || status < 500 ? err.message : 'Internal Server Error',
      requestId: req.id,
    },
  });
}
```

The client receives a code, a safe message and a request ID. The log receives the stack and context. Support staff join the two through the request ID.

**Do not leak internals.** Never return stack traces, SQL text, file paths, dependency names or raw upstream error bodies to clients. In production, `NODE_ENV=production` makes Express's default handler omit stack traces, but your own handler should be explicit regardless. A distinct, generic 500 body avoids revealing what failed.

**Async errors: Express 4 versus Express 5.** This is a key difference.

- **Express 4** does not catch rejected promises from async handlers. If an `async` handler throws or awaits a rejected promise, Express 4 never learns about it, the request hangs and the process may log an unhandled rejection. Synchronous throws are caught, async ones are not. The usual fixes are a wrapper that forwards rejections to `next`, or the `express-async-errors` package.
- **Express 5** handles promises returned by handlers and middleware: if the returned promise rejects, Express passes the error to `next(err)` automatically.

```js
// Express 4: needed
const wrap = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
router.get('/:id', wrap(async (req, res) => {
  res.json(await service.get(req.params.id));
}));

// Express 5: this is enough, a rejection reaches the error handler
router.get('/:id', async (req, res) => {
  res.json(await service.get(req.params.id));
});
```

Callbacks and event emitters are still not covered. An error thrown inside a `setTimeout`, a stream `'error'` event with no listener, or a callback-style API must be forwarded manually with `next(err)` (use `try...catch` inside timers). With a promise chain that is not `async`, return the promise or add `.catch(next)`. The official error handling guide confirms this.

**Process-level safety net.** Register handlers for `unhandledRejection` and `uncaughtException` that log and start a controlled shutdown. These are for last resort diagnostics, not for continuing normal operation.

```js
process.on('unhandledRejection', (reason) => { logger.fatal({ reason }, 'unhandled rejection'); shutdown(1); });
process.on('uncaughtException',  (err)    => { logger.fatal({ err }, 'uncaught exception');   shutdown(1); });
```

**Status mapping for common causes.** Validation 422 or 400, not found 404, conflict (unique violation) 409, unauthenticated 401, forbidden 403, rate limited 429, upstream failure 502, upstream timeout 504, overloaded or draining 503. Distinguish database error classes inside the repository layer and translate them to domain errors there, so controllers never inspect driver error codes.

**Timeouts and cancellation.** A handler waiting on a slow upstream ties up a request. Set timeouts on outbound calls (HTTP clients, database queries) and surface failures as 504 or 502. Respect client disconnects where cheap (`req.on('close')`, `AbortSignal`).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Error classes | custom `AppError` hierarchy, `err.cause` | Distinguish operational from programmer errors in code |
| Central handler | four-argument middleware, `res.headersSent` | Produce one consistent error contract |
| Async | Express 5 promise handling, a wrapper or `express-async-errors` for Express 4 | Explain precisely what each version catches |
| Process safety | `process.on('unhandledRejection')`, `uncaughtException`, supervisor restarts | Fail loudly and restart cleanly |
| Monitoring | error-rate alerts, Sentry-style capture of 5xx only | Alert on 5xx, not on every 4xx |

### Common mistakes

- Sending `err.stack` or `err.message` of unknown errors to the client.
- Swallowing errors with an empty `catch`, then returning 200.
- Catching everything and returning 500 for what is actually a 4xx.
- Relying on async handler errors reaching the handler in Express 4 without a wrapper.
- Continuing to run after an `uncaughtException`.
- Logging the same error at several layers, producing duplicate noise.
- Alerting on every 404 and 401.

### Self-check

- Distinguish operational and programmer errors with two examples each, and the right reaction to each.
- Why does an async handler that rejects hang a request in Express 4?
- What belongs in the log line for a 500 that must not be in the response body?
- Why register the error handler last?

---

## 8. Project structure

**In plain words.** Express does not tell you how to organise files. Split the code into small layers: routes, controllers, services and data access. Then each part is easy to test and change.

### Understand

**Why structure at all.** Express will let you put validation, SQL, business rules and HTTP details in one function. That is fast to write and expensive to change and test. Separation lets you test business rules without HTTP and swap infrastructure without touching rules.

**A layered layout.**

```
src/
  app.ts               build the Express app from dependencies (no listen)
  server.ts            create deps, call app.listen, handle signals
  config/              environment parsing and validation
  middleware/          requestId, auth, errorHandler, validate
  modules/
    orders/
      orders.routes.ts       HTTP mapping only
      orders.controller.ts   parse validated input, call service, shape response
      orders.service.ts      business rules, transactions, no req/res
      orders.repository.ts   all SQL or ORM calls
      orders.schemas.ts      zod schemas and types
  lib/                 logger, db pool, http client
test/
```

Organise by **feature module** (orders, users) rather than by technical type (all controllers together). A module owns its routes, rules and data access, which keeps changes local as the code base grows.

**Layer responsibilities.**

| Layer | Knows about | Must not |
|---|---|---|
| Route | paths, middleware order | contain logic |
| Controller | `req`, `res`, validated input | contain business rules or SQL |
| Service | domain rules, transactions, other services | import `express`, read `req` |
| Repository | the database and its driver | contain business decisions |

A test of the structure: you can call a service function from a script or a queue worker with no HTTP in sight.

**Dependency injection by hand.** You do not need a container. Build dependencies at the application edge and pass them in through factory functions. This gives test seams.

```js
// orders.service.js
export const makeOrdersService = ({ repo, payments, clock = () => new Date() }) => ({
  async create(input) { /* uses repo, payments, clock */ },
});

// orders.routes.js
export const makeOrdersRouter = ({ ordersService }) => {
  const r = express.Router();
  r.post('/', validate(createOrderBody), async (req, res) => {
    const order = await ordersService.create(req.validated.body);
    res.status(201).json(toOrderDto(order));
  });
  return r;
};

// app.js
export const buildApp = (deps) => {
  const app = express();
  /* middleware ... */
  app.use('/api/v1/orders', makeOrdersRouter(deps));
  /* 404 and error handler ... */
  return app;
};

// server.js: the composition root, the only place that creates real dependencies
const pool = createPool(config.db);
const app = buildApp({ ordersService: makeOrdersService({ repo: makeOrdersRepo(pool), payments }) });
```

`buildApp(deps)` is the dependency seam that makes tests simple: pass fakes for `payments` and an in-memory repository.

**Configuration.** Read environment variables in one module, validate them at start (with the same schema library), and fail fast with a clear message when something is missing. Never read `process.env` scattered through code. Keep secrets out of the repository.

**Response shaping.** Map database rows to response DTOs explicitly. Do not return ORM entities directly, since new columns then leak to clients automatically.

**Avoid premature abstraction.** A repository wrapper around a trivial query is noise. Add layers when a rule must be tested without a database or when two entry points (HTTP, worker) share logic. Do not copy a heavy enterprise layout into a three-endpoint service.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Composition | `buildApp(deps)`, factory functions, a single composition root | Wire an app so every external dependency can be replaced in tests |
| Config | `zod` or `envalid` for environment parsing | Crash at startup on invalid configuration |
| Separation | modules per feature, DTO mapping | Keep `express` imports out of services |
| Tooling | TypeScript strict mode, ESLint, `dependency-cruiser` | Enforce layer boundaries automatically |

### Common mistakes

- Fat controllers containing SQL.
- Services importing `req` or `res`, making them impossible to reuse.
- Creating the database pool inside a module at import time, which blocks testing and shutdown.
- Reading `process.env` everywhere.
- Returning database rows directly.

### Self-check

- What does a service function gain from not knowing about `req`?
- How does `buildApp(deps)` help testing?
- Why organise by feature instead of by layer?

---

## 9. Authentication and authorisation

**In plain words.** Authentication checks who the caller is. Authorisation checks what that caller may do. Check both on the server for every request, and refuse by default.

### Understand

**Authentication (authn)** answers who the caller is. **Authorisation (authz)** answers what the caller may do. Mixing them causes security bugs. Authn failure is 401. Authz failure is 403 (or 404 when you must not reveal that a resource exists).

**Where it lives.** Authentication is a middleware that runs before protected routes, verifies a credential and attaches an identity to the request. Authorisation is a check, per route or inside the service, that uses that identity.

**JWT bearer tokens.** A JWT (JSON Web Token) is a signed token that carries claims, such as the user ID and expiry time. The client sends `Authorization: Bearer <token>`. The middleware verifies the signature and claims, then sets `req.user`.

```js
import jwt from 'jsonwebtoken';

export function authenticate(req, res, next) {
  const header = req.get('authorization') ?? '';
  const [scheme, token] = header.split(' ');
  if (scheme !== 'Bearer' || !token) return res.status(401).json({ error: { code: 'unauthenticated' } });

  try {
    const claims = jwt.verify(token, publicKey, {
      algorithms: ['RS256'],            // pin the algorithm
      issuer: 'https://auth.example.com',
      audience: 'orders-api',
    });
    req.user = { id: claims.sub, roles: claims.roles ?? [] };
    next();
  } catch {
    res.status(401).json({ error: { code: 'invalid_token' } });
  }
}
```

Rules that prevent real vulnerabilities:
- **Pin the allowed algorithms.** Never accept the algorithm named in the token header. Reject `none`.
- **Verify, do not just decode.** `jwt.decode` does not check a signature.
- Check `exp`, `iss` and `aud`. Use short lifetimes for access tokens and a separate refresh mechanism.
- Key management: prefer asymmetric signing so the API holds only the public key, and fetch keys from the issuer's JWKS endpoint with caching. The `jose` library supports remote key sets.
- A JWT cannot be revoked before expiry without extra state. Keep it short-lived, or keep a deny list for sensitive cases.
- Do not store sensitive data in the payload. It is signed, not encrypted.

**Sessions versus tokens.** Server-side sessions (a session ID in a cookie, state in Redis or a database) are simple to revoke and well suited to browser applications. Stateless tokens suit service-to-service and mobile clients. Choose deliberately. If you use cookies for authentication, you need CSRF defences (chapter 10).

**Passwords.** Hash with a slow, salted algorithm designed for passwords: argon2id, scrypt or bcrypt. Never a fast hash such as SHA-256. Rate limit login and return the same message for unknown user and wrong password.

**RBAC.** Role-based access control maps roles to permissions. Keep a permission check as small, explicit middleware or service guard.

```js
export const requireRole = (...allowed) => (req, res, next) => {
  const ok = req.user && req.user.roles.some((r) => allowed.includes(r));
  if (!ok) return res.status(403).json({ error: { code: 'forbidden' } });
  next();
};

router.delete('/:id', authenticate, requireRole('admin'), deleteUser);
```

Roles alone are not enough for **object-level authorisation**: "this user may read this order" depends on the resource owner. Check ownership in the service or the query (`WHERE id = $1 AND customer_id = $2`). Missing object-level checks (insecure direct object reference, IDOR, or "broken object level authorisation") is a top API vulnerability class.

**Fail closed.** The default must be denial. Protect routers by default and open specific public routes, rather than the reverse. If the token cannot be verified, the key service is down, or the role claim is missing, deny. A bug that skips the check must result in 401 or 403, not in access.

```js
app.use('/api', authenticate);              // everything under /api requires auth
app.use('/api/public', publicRouter);       // mount public routes BEFORE the line above if they must be open
```

Place the public routes first deliberately and write a test that asserts every route not on a short allow-list returns 401 without credentials.

**Service-to-service.** Use mutual TLS, signed tokens with an audience, or an API gateway. Do not use a shared static API key in a header as the only control for anything sensitive. If you must, compare in constant time and rotate.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| JWT | `jsonwebtoken`, `jose`, JWKS | Verify tokens with pinned algorithm, issuer and audience |
| Passwords | `argon2`, `bcrypt`, `crypto.scrypt` | Store and verify password hashes correctly |
| Sessions | `express-session` with a Redis store, cookie flags | Explain trade-offs against tokens |
| RBAC | role middleware, policy functions | Implement role and ownership checks |
| Testing | tests with no token, bad token, wrong role, other user's resource | Prove every protected route fails closed |

### Common mistakes

- Using `jwt.decode` and trusting the result.
- Accepting any algorithm, or a shared secret in an HS256 setup that is also known to clients.
- Long-lived access tokens with no revocation plan.
- Checking role but not ownership, so any logged-in user can read any record by changing an ID.
- Returning 200 with partial data when authentication fails.
- Authentication registered after some routes in the file.
- Putting tokens in query strings, where they land in logs.

### Self-check

- What is the difference between a 401 and a 403 here?
- Why must you pin the accepted JWT algorithms?
- What is object-level authorisation and why is a role check not enough?
- What does "fail closed" mean for an authentication middleware whose key server is unreachable?

---

## 10. Security middleware

**In plain words.** Security comes in layers. Packages such as helmet, cors and a rate limiter add cheap protection. They do not replace checking input and permissions.

### Understand

Security is layered. Middleware gives you cheap, broad protection, and none of it replaces authorisation or validation.

**helmet.** Sets a collection of HTTP response headers with safer defaults (for example `X-Content-Type-Options`, `Strict-Transport-Security`, a Content-Security-Policy, and removal of `X-Powered-By`). The helmet documentation lists 12 headers set by default. It reduces exposure in browsers. For a pure JSON API many headers matter less, but it is cheap and expected. Also call `app.disable('x-powered-by')` if you do not use helmet.

**CORS.** Cross-Origin Resource Sharing is a **browser** policy. It does not protect your server from non-browser clients and is not authentication. With the `cors` package, configure an explicit allow-list of origins.

```js
import cors from 'cors';
app.use(cors({
  origin: ['https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,          // only if cookies are required
  maxAge: 600,
}));
```

A wide-open `origin: '*'` with credentials is not permitted by browsers, and reflecting any `Origin` back with credentials enabled is a vulnerability. Preflight (`OPTIONS`) requests must succeed before your authentication runs, which is why CORS middleware comes before auth.

**Rate limiting.** `express-rate-limit` counts requests per key (by default the client IP) in a window and answers 429 when exceeded.

```js
import { rateLimit } from 'express-rate-limit';
app.use('/api', rateLimit({ windowMs: 60_000, limit: 100, standardHeaders: true, legacyHeaders: false }));
app.use('/api/auth/login', rateLimit({ windowMs: 15 * 60_000, limit: 10 }));
```

The current README documents `limit` (older versions used `max`; this rename is not verified here), `windowMs`, `standardHeaders` and `legacyHeaders`. Important points:
- The default store is **in memory, per process**. With several instances or restarts the counters do not agree. Use a shared store (Redis) for real limits.
- Stricter limits on login, password reset and expensive endpoints. Key by user or API key as well as IP when possible.
- Rate limiting at the edge (nginx, a CDN or gateway) is cheaper than in Node. Use both for defence in depth.

**`trust proxy` and why it matters.** In production your app usually sits behind nginx or a load balancer. The TCP connection your app sees comes from the proxy, so `req.ip` would be the proxy's address for every client. The real client address arrives in `X-Forwarded-For`, and the original scheme in `X-Forwarded-Proto`. Express reads those headers **only** if you set the `trust proxy` setting.

```js
app.set('trust proxy', 1);   // trust exactly one proxy hop in front of the app
```

Why it matters:
- **Rate limiting** by `req.ip` without it treats all users as one client (everyone blocked together) or, if misconfigured the other way, lets attackers rotate a spoofed header to evade limits.
- **Logs and audit trails** record the wrong IP.
- **`req.secure` and `req.protocol`** become wrong, so `secure` cookies may not be set and redirects may use `http`.

Do not set `trust proxy` to `true` blindly. `X-Forwarded-For` is client-controlled until a trusted proxy appends to it. Trusting it from the open internet lets any client choose its apparent IP. Set the number of hops (or the specific proxy addresses or subnets) you actually have, and make sure the proxy overwrites or appends the header correctly. The Express "behind proxies" guide confirms this: `true` takes the left-most `X-Forwarded-For` entry, and a number `n` trusts `n` hops. It also warns that a number assumes one consistent path to the app. The `express-rate-limit` package warns when the setting looks inconsistent with the header it sees.

**Cookies.** For cookies carrying a session or token set `httpOnly` (not readable by page scripts), `secure` (HTTPS only), `sameSite` (`lax` or `strict`, or `none` with `secure` for cross-site use), a bounded `maxAge`, and a restrictive `path` and `domain`. The `__Host-` name prefix adds protection. Behind a proxy `secure` cookies need `trust proxy` to work.

**CSRF.** Cross-site request forgery (a malicious site makes the user's browser send a request to your API) applies when the browser attaches credentials **automatically**, which means cookie-based authentication. APIs that authenticate with an `Authorization` header set by script code are not vulnerable in the same way, since a malicious page cannot make the browser add that header. If you use cookies: use `SameSite` cookies as a baseline, require a custom header or a CSRF token on state-changing requests (synchronizer token or double-submit cookie), check `Origin` or `Referer`, and never change state on GET. The older `csurf` package is deprecated, so pick a maintained approach or implement the token pattern deliberately.

**Other baseline controls.**
- Validate all input (chapter 6) and use parameterised queries to prevent SQL injection.
- Limit body sizes and request timeouts.
- Avoid user-controlled file paths and regular expressions with catastrophic backtracking.
- Keep dependencies patched: `npm audit`, Dependabot or Renovate, lockfiles, `npm ci` in builds.
- Prototype pollution: be careful merging untrusted objects.
- Do not log secrets or full tokens.
- TLS terminates at the proxy. Enforce HTTPS and HSTS there.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Headers | `helmet` | Enable with defaults, tune CSP only when serving HTML |
| CORS | `cors` with an origin allow-list | Explain preflight and why `*` with credentials is wrong |
| Rate limits | `express-rate-limit`, a Redis store, edge limits | Configure per-route limits with a shared store |
| Proxy | `app.set('trust proxy', n)`, `req.ip`, `req.ips` | Get correct client IPs behind nginx |
| Cookies and CSRF | cookie flags, `SameSite`, CSRF tokens | Decide when CSRF applies and defend it |
| Dependencies | `npm audit`, `npm ci`, Renovate or Dependabot | Keep a patched, reproducible build |
| Scanning | `curl -I`, securityheaders-style checks, OWASP ZAP | Verify headers and probe common issues |

### Common mistakes

- `cors()` with no options (allows any origin) on an API with credentials.
- Believing CORS blocks non-browser requests.
- Forgetting `trust proxy`, so every client shares one IP and rate limits lock out everyone.
- Setting `trust proxy` to `true` on an app exposed directly to the internet.
- An in-memory rate limit on a multi-instance deployment.
- Non-`httpOnly` session cookies, or no `secure` flag in production.
- Treating helmet as a complete security solution.

### Self-check

- What does `req.ip` return behind nginx without `trust proxy`, and what is the consequence for rate limiting?
- Why is trusting `X-Forwarded-For` from the public internet dangerous?
- When does CSRF apply, and why does bearer-token authentication largely avoid it?
- Why does an in-memory rate limiter fail with two instances?

---

## 11. File uploads

**In plain words.** File uploads can fill your memory or disk. Set strict limits, check the real file type, and never trust the file name from the client. For large files, let the client upload straight to storage.

### Understand

Uploads are a classic source of denial of service and storage abuse. `multipart/form-data` bodies are not handled by the JSON or urlencoded parsers. The common middleware is **multer**, built on busboy, a streaming multipart parser. Multer passes its `limits` object to busboy (keys include `fileSize`, `files`, `fields`, `parts`).

```js
import multer from 'multer';

const upload = multer({
  storage: multer.memoryStorage(),
  limits: { fileSize: 5 * 1024 * 1024, files: 1, fields: 10 },
  fileFilter: (req, file, cb) => {
    const ok = ['image/png', 'image/jpeg'].includes(file.mimetype);
    cb(ok ? null : new AppError(415, 'unsupported_media_type', 'Only PNG or JPEG'), ok);
  },
});

router.post('/avatar', authenticate, upload.single('file'), handler);
```

**Design rules.**
- **Always set `limits`** (file size, file count, field count). Without them a client can fill memory or disk.
- **Memory storage keeps the whole file in RAM.** Fine for small files, dangerous for large ones or many concurrent uploads. **Disk storage** writes to a temporary directory, which you must clean up.
- **For large files, stream to the destination** (object storage) instead of buffering. Streaming means the Node process never holds the whole file. Options: a multer storage engine that streams to the target, using busboy directly, or avoiding the API server for the bytes entirely with a **pre-signed upload URL**: your API authorises the upload and returns a short-lived URL, and the client uploads directly to object storage. This removes load, bandwidth and risk from your servers.
- **Do not trust the client's metadata.** `mimetype` and the filename come from the client. Check the actual file signature (magic bytes), limit extensions, and for images re-encode them. Generate your own storage key. Never use the original filename as a path (path traversal, overwriting, odd characters).
- **Store outside the web root**, serve through a controlled route or storage with correct `Content-Type` and `Content-Disposition`, and set `X-Content-Type-Options: nosniff`.
- **Scan** untrusted files for malware if they will be shared with other users.
- **Authenticate before parsing the body.** Put `authenticate` before `upload`, so anonymous clients cannot make the server buffer large bodies.
- **Handle multer errors.** Limit violations raise a `MulterError` with codes such as `LIMIT_FILE_SIZE`. Map them to 413 or 400 in the error handler.
- **Reverse proxy limits.** nginx `client_max_body_size` rejects oversized bodies before they reach Node. Align it with your application limit.
- **Timeouts.** Slow uploads hold a connection open. Keep request and header timeouts sensible (chapter 14).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Parsing | `multer` (`single`, `array`, `fields`, `limits`, `fileFilter`) | Accept a bounded upload safely |
| Streaming | `busboy`, `stream.pipeline`, object storage SDK streams | Stream a file to storage with no full buffering |
| Direct upload | pre-signed URLs | Explain why they are preferable for large files |
| Validation | magic-byte detection such as `file-type` | Verify content, not the declared type |
| Edge limits | nginx `client_max_body_size` | Reject oversize requests at the proxy |

### Common mistakes

- No `limits`, so one request can exhaust memory.
- Memory storage for large files.
- Trusting `file.mimetype` and `file.originalname`.
- Writing to a path built from the original filename.
- Leaving temporary files on disk after failures.
- Authenticating after the upload is parsed.

### Self-check

- Why are pre-signed URLs often better than proxying uploads through the API?
- What are the risks of using `file.originalname` as a path?
- What happens to memory when 100 clients upload 50 MB files with memory storage?

---

## 12. Streaming, SSE, compression and caching

**In plain words.** Large responses should be sent in small pieces, not built in memory first. Server-Sent Events keep one response open to push updates to a browser. Cache headers tell clients and proxies how long they may reuse a response.

### Understand

### Streaming responses

`res` is a writable stream (a destination that accepts data piece by piece). Instead of building a large payload in memory, pipe a readable source to it. Use `stream.pipeline` so errors and client disconnects clean up both sides.

```js
import { pipeline } from 'node:stream/promises';

router.get('/exports/:id', authenticate, async (req, res) => {
  res.set({ 'Content-Type': 'text/csv', 'Content-Disposition': 'attachment; filename="export.csv"' });
  await pipeline(await storage.openReadStream(req.params.id), res);
});
```

Streaming gives low memory use, fast time to first byte and natural back pressure (a slow client makes the reader wait instead of growing a buffer). After headers are sent you can no longer change the status code, so an error mid-stream can only abort the connection. Validate and authorise before you start writing.

### Server-Sent Events (SSE)

SSE is a one-way, long-lived HTTP response of `text/event-stream`, delivering events from server to browser with automatic reconnection. It is simpler than WebSockets for notifications and progress updates.

```js
router.get('/events', authenticate, (req, res) => {
  res.set({
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    'X-Accel-Buffering': 'no',     // ask nginx not to buffer this response
  });
  res.flushHeaders();

  const send = (event) => res.write(`data: ${JSON.stringify(event)}\n\n`);
  const heartbeat = setInterval(() => res.write(': ping\n\n'), 25_000);
  const unsubscribe = bus.subscribe(req.user.id, send);

  req.on('close', () => { clearInterval(heartbeat); unsubscribe(); });
});
```

Points to know:
- Always clean up on `close`, or you leak timers and subscribers for every disconnected client.
- Proxies buffer or time out idle connections. Send periodic comment heartbeats and disable proxy buffering for the route.
- Each connection holds a socket and some memory. Plan capacity and connection limits.
- Compression middleware can buffer SSE output so events arrive late. Exclude the route or flush explicitly.
- The browser `EventSource` cannot set custom headers, which affects how you authenticate (cookies, or a short-lived token). Be careful about tokens in URLs.
- With multiple instances you need a shared pub/sub (Redis, a broker) to fan out events.

### Compression

The `compression` middleware compresses responses based on `Accept-Encoding`, mainly for text (JSON, HTML). It costs CPU on the single Node thread. The Express performance guide says compression is best done at the reverse proxy for high-traffic sites. If you compress in Node, do not compress already compressed content (images, archives) and set a size threshold. Compressing a response that mixes secrets with attacker-chosen text has enabled attacks such as BREACH. So do not reflect authentication tokens in response bodies.

### Caching headers and ETags

HTTP caching saves bandwidth and load, and is a performance feature you can add cheaply.

- **`Cache-Control`** states policy: `no-store` (never keep, for sensitive responses), `no-cache` (store but revalidate), `private` (only the browser), `public, max-age=3600` (shared caches may keep for an hour), `immutable` (fingerprinted assets that never change).
- **ETag and conditional requests.** Express generates a weak ETag for `res.send` and `res.json` responses by default. A client sends `If-None-Match`, and if it still matches, Express replies **304 Not Modified** with no body. This saves bandwidth, but your handler still ran and built the response, so it does not save server work. Real savings need caching at the application or a CDN, or checking a version or `updated_at` before doing the expensive query.
- **`Last-Modified` / `If-Modified-Since`** works similarly.
- **Static assets** via `express.static` support `maxAge` and ETags. Serve versioned file names with long `max-age` and `immutable`.
- **Personalised or sensitive API responses** should be `Cache-Control: private` or `no-store`, so shared caches never serve one user's data to another. Use `Vary` (for example `Vary: Authorization` or `Accept-Encoding`) when responses differ by a request header.
- Set `app.set('etag', false)` or a custom function if the default weak ETag is not wanted.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Streaming | `stream.pipeline`, `Readable.from`, backpressure | Stream a large export with flat memory |
| SSE | `text/event-stream`, heartbeats, `req.on('close')`, `EventSource` | Build a leak-free event endpoint |
| Compression | `compression` middleware, nginx `gzip`/`brotli` | Decide where compression belongs |
| Caching | `Cache-Control`, `ETag`, `Vary`, `express.static` options | Set correct policy for public, private and immutable responses |
| Verification | `curl -I`, `curl -H 'If-None-Match: ...'`, `curl -N` | Observe 304s and streamed output |

### Common mistakes

- Loading a whole large result into memory before sending.
- Not cleaning up SSE resources on client disconnect.
- Proxy buffering that makes streaming or SSE appear to hang.
- Public caching of per-user responses.
- Thinking a 304 saves database work.
- Compressing in Node under high load when the proxy could do it.

### Self-check

- Why must you authorise before starting to stream?
- What leaks if you forget `req.on('close')` in an SSE handler?
- Does an ETag-based 304 reduce server computation? Why or why not?
- What does `Vary: Authorization` tell a cache?

---

## 13. Logging and request IDs

**In plain words.** Logs tell you what happened in production. Write them as JSON, put the same request ID on every line of one request, and never log secrets.

### Understand

**Why.** In production, logs are how you reconstruct what happened. They must be **structured** (JSON), **correlated** (a request ID on every line) and **safe** (no secrets).

**Request ID.** Assign or accept an ID at the edge, put it in every log line and return it in a response header (for example `X-Request-Id`). Accept an incoming ID from a trusted upstream (the proxy or gateway) and generate one otherwise. Pass it on to downstream calls so a request can be traced across services. Standards-based alternatives include W3C Trace Context (`traceparent`) with OpenTelemetry.

**pino and pino-http.** `pino` is a fast JSON logger. `pino-http` is Express middleware that logs one line per request (method, URL, status, duration) and attaches a child logger as `req.log`.

```js
import pino from 'pino';
import pinoHttp from 'pino-http';
import { randomUUID } from 'node:crypto';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'],
});

app.use(pinoHttp({
  logger,
  genReqId: (req, res) => {
    const id = req.headers['x-request-id'] ?? randomUUID();   // req is the raw Node request here
    res.setHeader('x-request-id', id);
    return id;
  },
  customLogLevel: (req, res, err) => (err || res.statusCode >= 500 ? 'error' : res.statusCode >= 400 ? 'warn' : 'info'),
}));

// in a handler
req.log.info({ orderId }, 'order created');
```

Practices:
- Log to **stdout** and let the platform (systemd, Docker, Kubernetes) collect it.
- **Redact** authorisation headers, cookies, tokens and personal data. Do not log whole request bodies by default.
- Use levels consistently: `error` for failures needing attention, `warn` for degraded or client errors worth noticing, `info` for normal events, `debug` only when needed. A very high rate of identical errors can fill a disk and hide the real fault, so rate-limit or deduplicate at the source.
- Log **events with fields**, not interpolated strings: `{ orderId, userId }`, not `"order " + id`. Structured fields are searchable.
- Log once at the point where an error is handled (the central handler), not at every layer.
- Measure latency and status, and expose metrics (requests, errors, duration histograms) with a metrics library such as `prom-client`, alongside logs. Alert on symptoms (error rate, latency) rather than on single log lines.
- Do not use `console.log` for production logs. It is synchronous on some destinations, unstructured and uncorrelated.
- Health endpoints: a cheap `/healthz` liveness check (the process is up) and a `/readyz` readiness check (dependencies reachable, not draining). Keep health requests out of access logs if they are noisy (`autoLogging.ignore`).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Logger | `pino`, `redact`, child loggers | Emit structured, redacted JSON |
| HTTP logging | `pino-http`, `genReqId`, `customLogLevel` | One correlated line per request |
| Pretty output | `pino-pretty` (development only) | Read logs locally without changing production format |
| Tracing and metrics | OpenTelemetry, `prom-client` | Connect logs to traces and expose RED metrics |
| Querying | `jq`, your log platform | Find a request by ID across services |

### Common mistakes

- Logging the `Authorization` header or full request bodies containing passwords.
- Generating a new request ID per service, breaking correlation.
- Trusting a client-supplied request ID from the open internet for anything but correlation (it can contain anything, so bound its length and characters).
- Logging at every layer so each error appears five times.
- Using `pino-pretty` in production.

### Self-check

- Why return the request ID in a response header?
- What does `redact` protect against?
- What is the difference between liveness and readiness?

---

## 14. Graceful shutdown and load balancers

**In plain words.** When a server is told to stop, it should finish the requests it already has and refuse new ones. Then it closes its connections and exits. Timeouts between your app and the load balancer must match, or users see random 502 errors.

### Understand

**Why.** Deployments, autoscaling and crashes stop processes constantly. If a process dies while serving requests, clients see errors. A **graceful shutdown** stops accepting new work, finishes in-flight requests, releases resources and then exits. Linux chapter 4 covers signals: orchestrators and systemd send `SIGTERM`, wait a grace period, then send `SIGKILL`.

**Procedure.**

1. On `SIGTERM`, mark the service as **not ready** so the load balancer stops sending new requests (readiness endpoint returns 503).
2. Optionally wait a few seconds for the load balancer to notice and deregister.
3. Call `server.close()`, which stops accepting new connections and calls back when existing connections end.
4. Close idle keep-alive connections (idle connections kept open for reuse). Since Node 19, `server.close()` closes idle connections itself. `server.closeIdleConnections()` (Node 18.2 and later) does the same on demand.
5. Wait for in-flight requests, with an overall timeout shorter than the platform's kill timeout. After the timeout, force-close remaining connections (`server.closeAllConnections()`).
6. Close the database pool, message clients and flush logs.
7. Exit with a code (0 for a clean stop).

```js
let shuttingDown = false;

app.get('/readyz', (req, res) => res.sendStatus(shuttingDown ? 503 : 200));

async function shutdown(code = 0) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info('shutdown started');

  const force = setTimeout(() => { logger.error('forced exit'); process.exit(1); }, 25_000);
  force.unref();

  server.close(async () => {
    try { await pool.end(); } finally { process.exit(code); }
  });
  server.closeIdleConnections();
}

process.on('SIGTERM', () => shutdown(0));
process.on('SIGINT',  () => shutdown(0));
```

**Keep-alive timeouts behind load balancers.** This is a frequent source of intermittent 502 and 504 errors.

- Load balancers and proxies reuse connections to your app (keep-alive). Each side has an **idle timeout**.
- Node's HTTP server closes an idle keep-alive connection after `server.keepAliveTimeout` (the default is 5 seconds, not verified against the Node docs here).
- If Node closes the connection **before** the load balancer's idle timeout, then the balancer may send a request on a connection Node is just closing. The request fails with a reset and the client sees an intermittent 502. This shows up as rare errors under steady traffic, unrelated to load.
- The fix: set Node's `keepAliveTimeout` **higher** than the idle timeout of the load balancer in front of it. Many cloud load balancers default to 60 seconds, so a Node value of 65 seconds is a common choice. Check your balancer's documented idle timeout.
- Also set `server.headersTimeout` above `keepAliveTimeout`, as Node's own guidance links them. Modern Node also has `server.requestTimeout` to bound how long a client may take to send a whole request, which defends against slow clients (confirm defaults for your version).

```js
server.keepAliveTimeout = 65_000;
server.headersTimeout   = 66_000;
```

**Other deployment facts.**
- Run **under a supervisor** (systemd, a container orchestrator, pm2) that restarts crashes. Make the app the PID 1 process or use an init in containers, so signals reach Node (Linux chapter 4). A shell wrapper (`npm start` in some setups) can swallow `SIGTERM`, so run `node` directly.
- **Health checks** should be cheap and should not depend on every downstream, otherwise a database blip takes all instances out of rotation at once. Distinguish "alive" from "ready".
- During a rolling deploy, in-flight long requests (SSE, streaming) are cut off at shutdown. Clients must reconnect, and servers should send an end or a retry hint.
- Set a sensible `terminationGracePeriod` or `TimeoutStopSec` greater than your shutdown timeout.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Signals | `process.on('SIGTERM')`, `server.close`, `closeIdleConnections` | Write a shutdown that drains and exits in bounded time |
| Timeouts | `keepAliveTimeout`, `headersTimeout`, `requestTimeout` | Align Node timeouts with the load balancer |
| Probes | `/healthz`, `/readyz`, platform probes | Separate liveness from readiness |
| Supervision | systemd, Docker, Kubernetes, pm2 | Explain who sends the signal and the kill deadline |
| Verification | `curl` loops during a restart, `autocannon` while deploying | Show zero failed requests during a rollout |

### Common mistakes

- No `SIGTERM` handler, so deploys cut requests mid-flight.
- Leaving `keepAliveTimeout` at the default behind a 60 second balancer, producing sporadic 502s.
- Calling `process.exit()` immediately, skipping draining and pool closure.
- A shutdown with no upper time limit, which the platform then kills.
- A readiness check that stays healthy during shutdown.
- Running through `npm start` so the signal never reaches Node.

### Self-check

- Why does a mismatch between Node's keep-alive timeout and the load balancer's idle timeout cause intermittent 502s?
- List the steps of a graceful shutdown in order.
- Why separate liveness from readiness?

---

## 15. Performance

**In plain words.** Node runs your code on one thread, so slow synchronous work blocks every request. Keep handlers short, reuse database connections, and run several copies of the app to use all CPU cores. Measure before you optimise.

### Understand

**The event loop is the constraint.** One thread runs all JavaScript for a process. While it is busy, nothing else on that process progresses, including health checks. Throughput is high for I/O-bound work and poor for CPU-bound work.

**Avoid synchronous and CPU-heavy work in handlers.**
- Do not use `fs.readFileSync`, `crypto.pbkdf2Sync`, `zlib` sync methods or similar on request paths. Startup is fine.
- Heavy computation (image processing, large JSON transformation, PDF generation, big sorts) should go to **worker threads** (`node:worker_threads`), a separate service, or a job queue. Offload slow work and respond with 202 and a job ID when the client does not need to wait.
- `JSON.parse` and `JSON.stringify` of very large payloads block. Keep payloads small and paginate.
- Catastrophic regular expressions on user input can freeze a process (ReDoS).
- Password hashing with argon2 or bcrypt runs in the native thread pool in the common libraries, not on the main thread, but the pool is small (default 4 threads shared with file system and DNS work), so heavy hashing load can delay other pool-dependent operations.

**Measure the event loop.** Use `perf_hooks.monitorEventLoopDelay()` and export the delay as a metric. Rising p99 delay means something is blocking.

**Connection pooling.** Opening a database connection per request is slow and exhausts the database. Create **one pool per process at startup** and reuse it. Size it deliberately: total connections across all instances (instances times pool size) must stay under the database limit. Always release connections (`finally`), and use timeouts so a stuck query cannot hold a connection forever. An undersized pool shows up as requests waiting for a connection. Similarly, reuse HTTP agents with keep-alive for outbound calls instead of a new connection per call.

**Scaling across cores: cluster versus multiple instances.**
- Node's built-in `cluster` module forks workers that share a listening port, one per core.
- A modern, simpler approach is to run **multiple independent instances** (containers or processes under a process manager) behind a load balancer. Each is isolated, restarts independently, and works the same locally and in production. Containers or an orchestrator handle replicas, and you scale horizontally.
- Both need **stateless** processes: sessions, rate-limit counters, caches and SSE subscriptions that live in one process's memory will not be shared. Use Redis or a database for shared state.
- Per-process memory limits and pool sizes multiply by instance count.

**Cache and reduce work.** Cache expensive reads (in a shared cache such as Redis, or a CDN for public data), paginate, select only needed columns, avoid N+1 query patterns, and batch calls. Most API latency comes from the database and downstream calls rather than from Express.

**Express overhead.** Routing and middleware cost is small compared to I/O. A very long middleware stack or many regex routes adds up, but profile before optimising. Frameworks such as Fastify are faster in raw benchmarks (chapter 18), though rarely the real bottleneck.

**Load testing.** Use `autocannon` (Node) or similar tools (k6, wrk) against a production-like environment. Test with realistic payloads and concurrency, watch latency percentiles and error rate rather than only requests per second, and find the saturation point. Always test the same code path your users hit (through the proxy, with auth).

```
npx autocannon -c 100 -d 30 http://localhost:3000/api/v1/orders
```

`-c` is concurrent connections and `-d` is duration in seconds. Run the load generator on a separate machine when you can, since it competes for CPU with the server otherwise.

**Profiling.** `node --inspect` with Chrome DevTools, `node --prof`, `clinic` (doctor, flame) and 0x produce flame graphs that show where CPU time goes. Heap snapshots find memory leaks. Look at RSS growth over time (Linux chapter 6).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Load testing | `autocannon`, `k6`, `wrk` | Find saturation point and read latency percentiles |
| Profiling | `clinic`, `0x`, `--inspect`, `--cpu-prof` | Produce and read a flame graph |
| Event loop health | `monitorEventLoopDelay`, `clinic doctor` | Detect blocking work |
| Pools | `pg.Pool`, ORM pool settings, HTTP keep-alive agents | Size pools against database limits |
| Offloading | `worker_threads`, queues (BullMQ and similar) | Move CPU or slow work off the request path |
| Scaling | multiple instances, `cluster`, pm2 cluster mode | Explain trade-offs and the statelessness requirement |

### Common mistakes

- Synchronous file, crypto or compression calls inside handlers.
- Creating a new database connection per request, or a pool per module.
- In-memory state that breaks when you add a second instance.
- Benchmarking `hello world` and concluding about real services.
- Optimising Express routing when the real cost is an unindexed query.
- Pool size times instance count exceeding the database's connection limit.

### Self-check

- Why does CPU-heavy work in one handler affect unrelated requests?
- What must be true of your application before you can run several instances?
- How would you tell whether a slow endpoint is bound by the database, by downstream calls or by the event loop?

---

## 16. Testing

**In plain words.** You can test an Express app without opening a network port. Build the app in a function, pass fake dependencies, and call it with supertest. Test the failure cases as much as the success cases.

### Understand

**Test strategy.** Most tests should be fast and need no network: unit tests for services with fakes, and HTTP-level tests of the app through an in-process client. A smaller set of integration tests uses a real database (often a container). A few end-to-end tests exercise a deployed environment.

**Make the app testable.** Export a function that builds the app and keep `listen` in a separate entry file (chapter 8). The test passes fakes for external dependencies.

**supertest** drives an Express app in-process. It starts an ephemeral server for the call, so you do not manage ports.

```js
import request from 'supertest';
import { buildApp } from '../src/app.js';

describe('POST /api/v1/orders', () => {
  const fakeService = { create: async (input) => ({ id: 'o1', ...input }) };
  const app = buildApp({ ordersService: fakeService, authenticate: (req, res, next) => { req.user = { id: 'u1', roles: [] }; next(); } });

  it('returns 201 for a valid order', async () => {
    const res = await request(app)
      .post('/api/v1/orders')
      .send({ customerId: '8d1d1f5e-7e3a-4c3e-9b0e-2f5f2a8f7a11', items: [{ sku: 'A1', quantity: 2 }] })
      .expect(201);
    expect(res.body.id).toBe('o1');
  });

  it('returns 422 for an invalid body', async () => {
    const res = await request(app).post('/api/v1/orders').send({}).expect(422);
    expect(res.body.error.code).toBe('validation_failed');
  });

  it('returns 401 without credentials', async () => {
    const realAuthApp = buildApp({ ordersService: fakeService });
    await request(realAuthApp).post('/api/v1/orders').send({}).expect(401);
  });
});
```

**What to test.**
- Success paths and the response shape (status, headers, body).
- Validation failures: missing fields, wrong types, extra fields, oversized body, wrong content type.
- **Authentication and authorisation:** no token, malformed token, expired token, wrong role, another user's resource. Assert that protection fails closed.
- The error handler: a thrown error in a handler yields the generic error shape and **no stack trace**. Include an async rejection test, since its behaviour differs between Express 4 and 5.
- Unknown routes return the JSON 404.
- Webhook signature acceptance and rejection with the raw body.
- Business rules in services with plain unit tests and no HTTP.

**Dependency seams.** Database, clock, random IDs, payment and email clients, and the queue are injected. Tests substitute fakes, or in-memory implementations, or a real database in a container for repository tests. Avoid mocking Express itself (`req` and `res` mocks) when supertest gives a more faithful test.

**Test runners.** Jest, Vitest or the built-in `node:test` all work. Pick one and keep tests deterministic: freeze time, seed IDs, isolate data, no shared mutable state between tests.

**Contract tests and API checks.** Validate responses against the OpenAPI document (chapter 17) to keep implementation and documentation in agreement.

**Close handles.** Tests that open a real server or pool must close them, or the test process hangs. Supertest handles the server it creates, but your own pool or timers must be shut down in a teardown hook.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| HTTP tests | `supertest` (`.get/.post`, `.set`, `.send`, `.expect`) | Test any endpoint in-process |
| Runners | `vitest`, `jest`, `node:test` | Configure and run a deterministic suite |
| Seams | factory functions, fakes, `buildApp(deps)` | Replace every external dependency in a test |
| Real dependencies | Testcontainers or Docker Compose for the database | Run repository tests against the real database |
| Coverage | the runner's coverage tool | Use it to find untested branches, not as a goal in itself |
| Load and smoke | `autocannon`, `curl` against a staging URL | Verify a deployed build |

### Common mistakes

- Starting the real server on a fixed port in tests.
- Tests that depend on execution order or shared database rows.
- Only testing the success path, never 401, 403, 422 or errors.
- Mocking so much that the test verifies the mocks.
- Forgetting to close pools and timers, producing hanging test runs.
- Not testing the error handler's no-leak guarantee.

### Self-check

- Why separate `buildApp` from `listen`?
- List four authentication and authorisation tests every protected route needs.
- How do you test that an unexpected exception does not expose a stack trace?

---

## 17. API versioning and OpenAPI

**In plain words.** Clients depend on your API, so changes must not break them. Add new fields instead of changing old ones, and use a version in the URL for real breaking changes. OpenAPI is a standard file format that describes your API for people and tools.

### Understand

**Versioning.** Once clients depend on your API, you cannot change it freely. Prefer **additive, backward-compatible** changes (new optional fields, new endpoints) and keep a version for breaking changes. Common schemes:

| Scheme | Example | Trade-off |
|---|---|---|
| Path | `/api/v1/orders` | Most visible and easiest to route, cache and document. Most common |
| Header or media type | `Accept: application/vnd.example.v2+json` | Cleaner URLs, harder to test by hand |
| Query parameter | `?version=2` | Easy but easy to forget, complicates caching |

In Express, path versioning is a router mount: `app.use('/api/v1', v1Router)` and `app.use('/api/v2', v2Router)`. Share services between versions and keep differences in the controller and DTO layer, so you do not duplicate business logic. Announce deprecation with a date, use `Deprecation` and `Sunset` headers if useful, log version usage, and remove a version only when metrics show it is unused.

Breaking changes include removing or renaming a field, changing a type, tightening validation, changing a status code or an error shape, and changing default behaviour. Clients also break when you add a value to an enum they switch over, so document that clients must tolerate unknown values.

**Idempotency and pagination as part of the contract.** Support `Idempotency-Key` for non-idempotent POSTs that clients retry (payments, order creation). Use cursor pagination with a stable sort, and document limits.

**OpenAPI.** The OpenAPI Specification describes an HTTP API (paths, parameters, schemas, responses, security) in a machine-readable document. It enables documentation (Swagger UI, Redoc), client and server code generation, mock servers, contract tests and linting. Two workflows:

- **Design first:** write the OpenAPI document, then implement and validate against it. Tools such as `express-openapi-validator` can validate requests and responses against the document at runtime or in tests.
- **Code first:** derive the document from your schemas. With zod, libraries such as `@asteasolutions/zod-to-openapi` generate OpenAPI from the same schemas you validate with, giving one source of truth. Its README says v8 targets zod 4, and zod 3 needs v7.3.4. With JSON Schema and Ajv the schemas map directly into OpenAPI.

Serve the document at a known path and optionally a docs UI (`swagger-ui-express`), protected in production if the API is internal. Whatever route you take, the important property is a **single source of truth** so documentation cannot drift from behaviour, and a CI check that fails when the generated document changes unexpectedly (breaking-change detection with a diff tool).

Confirm each library's current API in its own documentation, since these ecosystems move.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Versioning | router mounts per version, deprecation headers | Introduce `v2` without breaking `v1` |
| Spec | OpenAPI 3.x documents, `swagger-ui-express`, Redoc | Read and write an OpenAPI document |
| Generation | `zod-to-openapi`-style libraries, `openapi-typescript` | Derive docs or types from one source |
| Validation | `express-openapi-validator`, contract tests | Check responses against the spec in CI |
| Linting and diff | Spectral, an OpenAPI diff tool | Catch breaking changes before release |

### Common mistakes

- Changing a response shape in place for existing clients.
- Copy-pasting whole v1 controllers into v2, then fixing bugs in only one.
- Documentation maintained by hand, drifting from the code.
- Versioning too early or too often, instead of evolving additively.
- Removing an old version without usage data.

### Self-check

- Which changes are breaking and which are additive?
- Why share services across API versions and keep differences at the edge?
- What is the benefit of generating OpenAPI from the validation schemas?

---

## 18. Express versus Fastify, Koa and NestJS

**In plain words.** Express, Fastify, Koa and NestJS all build web servers in Node. They differ in speed, structure and how much they decide for you. Choose by team skill and project size, not by benchmark charts.

### Understand

An honest comparison, because a framework choice is a trade-off about team, ecosystem and risk rather than benchmarks.

| | Express | Fastify | Koa | NestJS |
|---|---|---|---|---|
| Philosophy | Minimal, middleware, huge ecosystem | Performance and schema-driven, plugin system | Minimal core by the Express authors, promise-based middleware with a context object | Opinionated framework with decorators, modules, dependency injection and conventions |
| Validation | Bring your own | Built in via JSON Schema (Ajv) | Bring your own | Built in through pipes (class-validator or zod integrations) |
| Async errors | Native in Express 5 (wrapper needed in 4) | Native promise support | Native (async by design) | Native with exception filters |
| TypeScript | Types via `@types/express`, less typed by default | Good, with type providers | Fair | First class, TypeScript-centred |
| Structure | You decide | Plugins and encapsulation guide you | You decide | Enforced modular structure |
| Speed | Adequate for most APIs | Generally faster in benchmarks | Fast, small core | Depends on the adapter (Express or Fastify underneath) |
| Ecosystem and hiring | Largest, most tutorials and middleware | Large and growing | Smaller | Large, especially in enterprise teams |
| Learning curve | Lowest to start, highest to do well | Moderate | Low | Highest at first, consistent afterwards |

**Express.** Strengths: ubiquity, every library offers Express middleware, easy hiring, simple mental model. Weaknesses: no structure, no built-in validation or serialisation, slower than Fastify in micro-benchmarks, and a long period of slow maintenance before Express 5 matured. Right when: team knowledge matters, the service is small to medium, or you need a specific Express-only middleware.

**Fastify.** Strengths: schema-first design (request validation and fast response serialisation from JSON Schema), an encapsulated plugin model, good logging by default (pino) and strong performance. Weaknesses: smaller middleware ecosystem and a different plugin and lifecycle model to learn. Right when: you want performance, schema validation and OpenAPI generation as a design centre.

**Koa.** Strengths: small, clean async middleware model ("onion" flow where code after `await next()` runs on the way back out). Weaknesses: you assemble everything yourself (router, body parser), with a smaller community, and few advantages over Express 5 for most teams. Right when: you value its minimalism or already use it.

**NestJS.** Strengths: structure for large teams, dependency injection, modules, guards, interceptors and pipes that map to the layers in chapter 8, TypeScript-first, good testing support, Express or Fastify as the HTTP layer. Weaknesses: decorator and DI abstraction to learn, heavier, and more magic that can obscure what is happening. Right when: a large team needs enforced conventions and consistency across many services.

**Practical advice.** The concepts in this guide (middleware ordering, boundary validation, central error handling, graceful shutdown, observability) transfer across all four. Throughput differences rarely decide real systems, since databases and downstream calls dominate latency. Choose by team familiarity, the shape of the codebase and the need for enforced structure. Be able to defend a choice without tribalism, and be able to work in any of them.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Comparison | a small endpoint written in each framework | Explain the differences from experience |
| Benchmarking | `autocannon` on equivalent routes with a database call | Show the gap shrinks when I/O dominates |
| Migration | adapters, running both behind one proxy | Plan an incremental move rather than a rewrite |

### Common mistakes

- Choosing a framework from synthetic benchmark charts.
- Rewriting a working Express service for marginal gains.
- Adopting a heavy framework for a tiny service, or building a large system on ad hoc Express with no conventions.

### Self-check

- When is Express the right choice, and when would you pick Fastify or NestJS?
- Why do benchmark gaps between frameworks often disappear in real services?
- What does "onion" middleware flow mean in Koa, and how does Express differ?

---

## 19. Practice labs

**In plain words.** These labs make you break and fix a small API on purpose. Each one has a clear "done" condition. Try them before you read any answer.

Do these in a scratch project. Each states the goal and what "done" looks like. Attempt before looking anything up.

1. **Raw http versus Express.** Write the same two-route server with `node:http` and with Express. List what Express added. Done when you can explain `app.listen` returning an `http.Server`.
2. **Middleware order experiments.** Build a stack with a logger, a JSON parser and a route, then reorder them deliberately (parser after route, 404 before routers, error handler first). Record the symptom of each mistake.
3. **Error handler contract.** Implement an `AppError` hierarchy and a central handler. Throw a domain error, a validation error, a `TypeError` and a rejected promise. Assert clients never see stacks, and that logs include the request ID. Repeat the async rejection under Express 4 and Express 5 and note the difference.
4. **Validated resource.** Build `/orders` with zod schemas for body, query and params, `.strict()` bodies, coerced pagination and a consistent 422 shape. Prove mass assignment is blocked.
5. **Layered module.** Split a feature into routes, controller, service and repository with `buildApp(deps)`. Done when the service runs from a plain script and the HTTP tests use an in-memory repository.
6. **Webhook with signature.** Implement an HMAC-signed webhook with a raw body, constant-time comparison and a timestamp window. Show that mounting the global JSON parser first breaks verification.
7. **Authentication and RBAC.** Add JWT verification (pinned algorithm, issuer, audience), role middleware and an ownership check. Write tests for no token, expired token, wrong role and another user's record. Add a test that enumerates all routes and fails if an unprotected one is not on an allow-list.
8. **Trust proxy.** Put nginx (or a small proxy) in front of the app. Log `req.ip` with and without `trust proxy`, then apply `express-rate-limit` and show the effect on all clients sharing one IP. Show a spoofed `X-Forwarded-For` working when the setting is too permissive.
9. **CORS and cookies.** Build a tiny browser page on another origin. Demonstrate a failed preflight, fix it with an origin allow-list, then set a session cookie with the right flags and demonstrate a CSRF attempt and your defence.
10. **Bounded upload.** Implement an upload route with multer limits and a content check by magic bytes. Send an oversized file, a wrong type and a path-like filename, and show each rejected. Then stream a large file to disk or storage and show flat memory.
11. **SSE and streaming.** Build an SSE endpoint with heartbeats and cleanup on close. Connect and disconnect many clients and show no leaked timers or subscribers. Stream a large CSV with `pipeline` and show flat memory.
12. **Caching.** Add `Cache-Control` and ETags. Use `curl -H 'If-None-Match: ...'` to observe a 304. Show that a per-user endpoint is marked `private` and that `Vary` is set where needed.
13. **Structured logging.** Add `pino-http` with request IDs, redaction and an incoming ID header. Follow one request ID through a handler, an outbound call and the central error handler with `jq`.
14. **Graceful shutdown.** Implement readiness, `SIGTERM` handling, `server.close` and a forced-exit timer. While running `autocannon`, send `SIGTERM` and show zero failed requests. Then reproduce the keep-alive mismatch with a proxy whose idle timeout exceeds `keepAliveTimeout` and fix it.
15. **Blocking the loop.** Add a handler with a synchronous heavy computation. Measure `monitorEventLoopDelay` and `autocannon` latency on another route. Move the work to a worker thread and compare.
16. **Pool sizing.** Run several instances against a database with a low connection limit. Reproduce "too many connections", then size pools so instances times pool size stays within the limit.
17. **Load test report.** Use `autocannon` on three routes (static, database read, database write). Record p50, p99 and error rate at increasing concurrency, and identify the saturation point and its cause.
18. **Versioned API.** Introduce `/v2` that changes a response field while sharing the service layer. Generate an OpenAPI document from your schemas and add a CI check that diffs it.
19. **Framework comparison.** Implement the same endpoint (validated input, database read, JSON response) in Express and Fastify. Compare lines of code, validation approach and benchmark results, and write down your conclusion.
20. **Incident write-up.** Break the service (a missing `return` causing duplicate responses, an unhandled rejection in Express 4, a mismatched keep-alive timeout) and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 20. Interview questions

Every question is tagged [Fresher], [Mid] or [Senior], and has a model answer in the final subsection (the answer number matches the question number). This guide makes no promise about interview or job results. It only helps you practise the ideas.

### Top 10 must-know

Learn these first. Answers are in "Model answers" under the same numbers.

1. [Fresher] **Q2.** What is middleware, and what can it do?
2. [Fresher] **Q7.** What causes "Cannot set headers after they are sent to the client"?
3. [Fresher] **Q8.** How do you write an error-handling middleware, and where do you register it?
4. [Mid] **Q13.** Why does middleware order matter?
5. [Mid] **Q15.** How do you validate input at the boundary?
6. [Mid] **Q16.** How do async errors behave in Express 4 versus Express 5?
7. [Mid] **Q20.** Why does `trust proxy` matter behind nginx?
8. [Mid] **Q21.** How do you verify a JWT correctly?
9. [Senior] **Q29.** Walk through a graceful shutdown.
10. [Senior] **Q42.** How would you migrate an Express 4 service to Express 5?

### Fresher

1. [Fresher] What is Express, and how does it relate to Node's `http` module?
2. [Fresher] What is middleware, and what are the three things a middleware can do?
3. [Fresher] What is the difference between `app.use` and `app.get`?
4. [Fresher] How do you read path parameters, query strings and the request body?
5. [Fresher] Why does `req.body` come back undefined or empty?
6. [Fresher] What does `res.json` do that `res.send` does not necessarily do?
7. [Fresher] What causes "Cannot set headers after they are sent to the client"?
8. [Fresher] How do you write an error-handling middleware, and where do you register it?
9. [Fresher] What is the difference between a 401 and a 403?
10. [Fresher] What is CORS, and does it protect your server from curl?
11. [Fresher] What does `express.Router` give you?
12. [Fresher] Why should you not run `nodemon` in production?

### Mid-level

13. [Mid] Explain why middleware order matters, with two concrete failures.
14. [Mid] How do you verify a webhook signature in Express?
15. [Mid] How do you validate input at the boundary, and why not trust `req.body`?
16. [Mid] How do async errors behave in Express 4 versus Express 5?
17. [Mid] Distinguish operational errors from programmer errors and say how you handle each.
18. [Mid] How do you avoid leaking internals in error responses?
19. [Mid] How do you structure a medium-sized Express API?
20. [Mid] Why does `trust proxy` matter behind nginx, and what is the risk of setting it wrongly?
21. [Mid] How do you verify a JWT correctly, and what mistakes do people make?
22. [Mid] What cookie flags would you set for a session cookie, and when does CSRF apply?
23. [Mid] How do you handle file uploads safely?
24. [Mid] How do you attach a request ID to every log line?
25. [Mid] How do you test an Express app without starting a server on a port?
26. [Mid] How do you version an API in Express?
27. [Mid] How does an ETag work in Express, and does a 304 save server work?
28. [Mid] How do you implement Server-Sent Events, and what must you clean up?

### Senior and design

29. [Senior] Walk through a graceful shutdown for an Express service on Kubernetes or behind a cloud load balancer.
30. [Senior] Why do some deployments see intermittent 502 errors, and how does `keepAliveTimeout` relate?
31. [Senior] How do you scale an Express service across cores and instances, and what must be true of the application?
32. [Senior] How would you find and fix an event loop blocking problem in production?
33. [Senior] How do you size database connection pools across many instances?
34. [Senior] Design the authentication and authorisation approach for a multi-tenant API, including fail-closed behaviour and object-level checks.
35. [Senior] How would you design rate limiting for a distributed deployment?
36. [Senior] How do you keep OpenAPI documentation and implementation in agreement?
37. [Senior] How would you design uploads of multi-gigabyte files?
38. [Senior] How would you handle a large export that exceeds memory?
39. [Senior] When would you choose Fastify, Koa or NestJS over Express, and when not?
40. [Senior] A service intermittently returns 5xx during deploys. How do you diagnose and prevent it?
41. [Senior] How do you design the error contract and logging so on-call engineers can debug a customer report quickly?
42. [Senior] How would you migrate a large Express 4 service to Express 5 safely?

### Model answers (condensed)

**1.** Express is a small routing and middleware layer on top of Node's `http` server. It adds a router, an ordered middleware pipeline and helpers on `req` and `res`. It does not provide validation, authentication or structure.

**2.** Middleware is a function `(req, res, next)` that runs in registration order. It can respond and end the chain, call `next()` to continue, or call `next(err)` to jump to error handling. If it does none of these, the request hangs.

**3.** `app.use(path, fn)` matches any method and any path starting with the prefix, and strips the prefix inside the function. `app.get(path, fn)` matches only GET and the specific path pattern.

**4.** `req.params` for path segments, `req.query` for the query string, `req.body` after a body parser such as `express.json()`. All are untrusted and should be validated.

**5.** No parser ran before the route, the parser was registered after the route, or the `Content-Type` did not match the parser. In Express 5 `req.body` is `undefined` (not `{}`) whenever the body has not been parsed. Check order and the client header.

**6.** `res.json` serialises the value with `JSON.stringify`, sets `Content-Type: application/json` and sends. `res.send` guesses from the argument type, so using `res.json` for JSON gives an explicit, consistent content type.

**7.** A second attempt to write headers after the response was already sent, typically a missing `return` after `res.json` in a guard clause, or a handler that both responds and calls `next()`.

**8.** A function with four parameters `(err, req, res, next)`. Register it after all routes and other middleware. Check `res.headersSent` and delegate to `next(err)` if headers were already sent.

**9.** 401 means the caller is not authenticated (missing or invalid credentials). 403 means the caller is authenticated but not permitted. Sometimes 404 is used for forbidden resources to avoid revealing existence.

**10.** CORS is a browser-enforced policy controlling which origins page scripts may call. It does not stop curl, servers or other non-browser clients, so it is not an access control mechanism.

**11.** A mountable mini-application with its own middleware and routes, used to group routes by resource and mount them at a prefix such as `/api/v1/orders`.

**12.** It watches files and restarts the process, which is useful in development, but production needs a supervisor with proper restart policy, signal handling and no file watching overhead.

**13.** Functions run in registration order, so a body parser registered after a route leaves `req.body` unparsed there, and a 404 handler registered before routers answers every request with 404. Likewise a rate limiter or authentication placed after a route does not protect it.

**14.** Use `express.raw({ type: 'application/json' })` on the webhook route (mounted before the global JSON parser) or capture the buffer with the `verify` option. Compute an HMAC over the exact bytes with the shared secret, compare with `crypto.timingSafeEqual`, check a timestamp to limit replay, and respond quickly after queueing the work.

**15.** Define a schema (for example with zod) for body, query and params, run it in a validation middleware, reject with 4xx and a stable error shape, and pass only the parsed result onward. Client data is unknown until validated, and unvalidated input enables injection, mass assignment and crashes.

**16.** Express 4 does not catch rejected promises from async handlers, so the request hangs unless you wrap handlers or use a helper package. Express 5 forwards a rejected promise returned by a handler to the error handling chain. Callbacks, timers and event emitters still need manual forwarding with `next(err)`.

**17.** Operational errors are expected runtime conditions (bad input, not found, upstream timeout). Map them to proper statuses and keep running. Programmer errors are bugs (a TypeError). Log in full, return a generic 500 and, if the process state may be corrupt, shut down gracefully and let the supervisor restart.

**18.** Use a central error handler that returns a stable code, a safe message and the request ID. Log stack and context only server-side. Return a generic message for 5xx, never include stacks, SQL or paths, and set `NODE_ENV=production`.

**19.** Organise by feature module with routes, controller, service and repository. Controllers handle HTTP only, services hold rules without importing Express, repositories hold data access. Wire dependencies in one composition root with `buildApp(deps)` so tests can replace them. Read configuration once and validate it at startup.

**20.** Behind a proxy the socket address is the proxy's, so `req.ip` is wrong for every client, which breaks rate limiting, logging and the secure-cookie and protocol detection. `trust proxy` tells Express to read `X-Forwarded-*`. Set it to the actual number of proxy hops. Setting it too broadly lets clients forge their IP with a header.

**21.** Verify the signature and algorithm (pinned allow-list, no `none`), check `exp`, `iss` and `aud`, and obtain keys from a trusted source such as a cached JWKS. Common mistakes: using `decode` instead of `verify`, accepting any algorithm, long-lived tokens without revocation, and storing secrets in the payload.

**22.** `httpOnly`, `secure`, `sameSite` (lax or strict where possible), a bounded `maxAge` and narrow path and domain. CSRF applies when the browser attaches credentials automatically, as with cookie authentication. Defend with SameSite, CSRF tokens or custom headers, origin checks and no state change on GET. Bearer tokens in an `Authorization` header set by script avoid most CSRF.

**23.** Set multer `limits`, authenticate first, filter by declared type but verify magic bytes, generate your own storage keys, avoid memory storage for large files, stream to storage or use pre-signed URLs, clean up temporary files, scan if shared, and align the proxy body size limit.

**24.** Use `pino-http` with `genReqId`, accepting a bounded incoming ID header from a trusted proxy or generating a UUID, echoing it in a response header and using `req.log` so every line carries the ID. Pass it to downstream calls.

**25.** Export a builder for the app without `listen`, then use `supertest` with the app. It starts an ephemeral listener per call. Inject fakes for databases and external services through the builder.

**26.** Mount routers under version prefixes (`/api/v1`, `/api/v2`), share the service layer, keep differences in controllers and DTOs, prefer additive changes, and announce deprecation with usage metrics before removing a version.

**27.** Express generates a weak ETag from the response body, and when the client's `If-None-Match` matches it answers 304 with no body. This saves bandwidth, but the handler already ran, so it does not save computation. Server work is saved only by caching earlier or checking a version first.

**28.** Set `Content-Type: text/event-stream` and `Cache-Control: no-cache`, flush the headers, write `data: ...\n\n` frames, send heartbeats and disable proxy buffering. On `req.on('close')` clear timers and unsubscribe. With multiple instances use a shared pub/sub.

**29.** On `SIGTERM` flip readiness to failing, wait briefly for the balancer to drain the instance, call `server.close()`, close idle connections, wait for in-flight requests within a deadline shorter than the platform kill time, force-close stragglers, close pools and clients, flush logs and exit.

**30.** If Node closes an idle keep-alive connection sooner than the balancer's idle timeout, the balancer may reuse a connection Node is closing and the request fails. Set `keepAliveTimeout` higher than the balancer's idle timeout (and `headersTimeout` above that), commonly 65 seconds for a 60 second balancer.

**31.** Run several stateless instances behind a load balancer (containers or processes, or Node's `cluster` on one host). Sessions, counters, caches and subscriptions must live in shared stores. Account for per-instance memory and pool size multiplying.

**32.** Watch event loop delay metrics and latency percentiles, take a CPU profile or flame graph (`clinic`, `--cpu-prof`) of the live or staging process, find synchronous or CPU-heavy code and regexes, then move work to worker threads, queues or another service and re-measure.

**33.** Total connections equal instances times pool size and must stay under the database limit with headroom for admin and migrations. Size pools from query concurrency needs, set timeouts, always release connections and consider a pooler such as PgBouncer.

**34.** Authenticate in middleware with verified, short-lived tokens carrying tenant and subject. Authorise by role and also check tenant and ownership in the service or query. Default deny, mount public routes explicitly, deny on verification failure or missing claims, and test that every route fails closed.

**35.** Use a shared store such as Redis behind `express-rate-limit` or limit at a gateway or proxy. Key by user or API key and IP, set stricter limits on sensitive routes, return 429 with `Retry-After`, set `trust proxy` correctly, and decide whether to fail open or closed if the store is down.

**36.** Make one source of truth: derive the document from validation schemas or validate requests and responses against a design-first document, run contract tests and diff the spec in CI to flag breaking changes.

**37.** Avoid proxying through the API. Issue a pre-signed or multipart upload URL for direct upload to object storage, with size and type conditions, then process the object asynchronously and notify the client. Include resumable uploads and integrity checks.

**38.** Stream from the database cursor or storage through a transform into `res` with `pipeline`, paginate, or run the export as a background job that writes to storage and returns a download link. Authorise before streaming, since errors mid-stream cannot change the status.

**39.** Fastify when schema-driven validation, serialisation speed and plugin encapsulation matter. NestJS when a large team needs enforced structure and dependency injection. Koa for a minimal modern core. Express when ecosystem and team familiarity dominate, or the service is small. Do not rewrite a working service for benchmark gains.

**40.** Likely causes are missing graceful shutdown, readiness staying true while draining, keep-alive timeout mismatch, forced kills before draining finishes, or startup readiness reporting true too early. Check load balancer and application logs around deploy times, reproduce under load with a rollout, and fix shutdown, probes and timeouts.

**41.** A stable error body with code, safe message and request ID, a request ID header on every response, structured JSON logs with that ID on every line and redaction, status and latency metrics, and alerts on 5xx rate. Support can search by the ID.

**42.** Read the migration guide, upgrade on a branch, run the full test suite and add tests for affected behaviour (async errors, path patterns, query and body handling, removed methods). Audit routes for old wildcard and optional-parameter syntax (`'/*'` to `'/*splat'`, `?` to braces), check the `req.query` default parser and `express.urlencoded` default change, replace removed methods such as `app.del()` and `res.redirect(url, status)`, check middleware compatibility, deploy gradually behind a flag or canary and watch error rates.

---

## 21. References

Primary sources, preferred over blog posts:

- Express documentation (guide and API reference): https://expressjs.com/
- Express migrating to 5 guide: https://expressjs.com/en/guide/migrating-5.html
- Express error handling guide: https://expressjs.com/en/guide/error-handling.html
- Express behind proxies guide: https://expressjs.com/en/guide/behind-proxies.html
- Express production best practices (security and performance): https://expressjs.com/en/advanced/best-practice-security.html and https://expressjs.com/en/advanced/best-practice-performance.html
- Node.js documentation (`http`, `stream`, `worker_threads`, `cluster`, `perf_hooks`): https://nodejs.org/docs/latest/api/
- Node.js guides on not blocking the event loop: https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop
- supertest: https://github.com/forwardemail/supertest
- helmet: https://helmetjs.github.io/ (redirects to https://helmet.js.org/)
- cors: https://github.com/expressjs/cors
- express-rate-limit: https://github.com/express-rate-limit/express-rate-limit and https://express-rate-limit.mintlify.app/overview
- multer: https://github.com/expressjs/multer
- pino and pino-http: https://github.com/pinojs/pino and https://github.com/pinojs/pino-http
- zod: https://zod.dev/
- zod-to-openapi: https://github.com/asteasolutions/zod-to-openapi
- autocannon: https://github.com/mcollina/autocannon
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
- HTTP Caching, RFC 9111: https://www.rfc-editor.org/rfc/rfc9111
- JSON Web Token, RFC 7519, and JWT best current practices, RFC 8725: https://www.rfc-editor.org/rfc/rfc7519 and https://www.rfc-editor.org/rfc/rfc8725
- OWASP API Security Top 10 and Cheat Sheet Series: https://owasp.org/API-Security/ and https://cheatsheetseries.owasp.org/
- HTML Living Standard, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- Fastify, Koa and NestJS documentation: https://fastify.dev/ , https://koajs.com/ , https://docs.nestjs.com/

URL status (2026-11-06): the Express, Node.js, helmet, cors, multer, pino-http, supertest, express-rate-limit and zod-to-openapi pages above loaded. The RFC, OWASP, WHATWG, zod, autocannon, OpenAPI, Fastify, Koa and NestJS links were not fetched. Verify any URL before depending on it, since pages move.

---

## 22. Glossary

**In plain words.** This table explains the hardest words in the guide in everyday language.

| Term | Plain meaning |
|---|---|
| Middleware | A function that runs in order on each request and can answer it, pass it on, or report an error |
| Router | An object that groups routes and mounts them at a path prefix |
| Event loop | The cycle that lets Node run one piece of work at a time while it waits for slow I/O |
| Idempotent | Doing it once or many times has the same final effect |
| Mass assignment | A client sets fields it should not, such as `isAdmin`, because the server copies the whole body |
| Validation | Checking that input has the expected shape and values |
| Coercion | Converting a value to the right type, such as the string "20" to the number 20 |
| Operational error | An expected problem at runtime, such as a missing record |
| Programmer error | A bug in the code, such as calling a function on `undefined` |
| Authentication | Proving who the caller is |
| Authorisation | Deciding what an authenticated caller may do |
| JWT | A signed token that carries claims such as user ID and expiry time |
| IDOR | A user reads another user's record by changing an ID because the server did not check ownership |
| CORS | A browser rule about which web pages may call your API from another origin |
| CSRF | A malicious site makes a logged-in browser send a request to your API |
| Preflight | An `OPTIONS` request that a browser sends first to ask if a cross-origin call is allowed |
| Rate limiting | Allowing only a fixed number of requests per time window |
| Reverse proxy | A server such as nginx that receives requests first and forwards them to your app |
| `trust proxy` | An Express setting that makes it read the client address from `X-Forwarded-For` |
| HMAC | A hash made with a shared secret, used to prove a message was not changed |
| Webhook | A request that another service sends to your URL when an event happens |
| Backpressure | A slow receiver makes the sender wait instead of filling memory |
| SSE | Server-Sent Events: one long HTTP response that pushes events to a browser |
| ETag | A label for a response version, used so a client can ask "has it changed?" |
| Keep-alive | Reusing one TCP connection for several requests |
| Graceful shutdown | Stopping by finishing current requests first, then exiting |
| Liveness and readiness | Liveness: the process is running. Readiness: it can take traffic now |
| DTO | Data transfer object: a plain object shaped for the response |
| Composition root | The one place where real dependencies are created and connected |
| ReDoS | A crafted input that makes a regular expression run for a very long time |
