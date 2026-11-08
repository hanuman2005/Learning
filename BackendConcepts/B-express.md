# Section B — Express Layer

## B1 — Middleware order & `next()`

**Diagram: the chain**
```
 request
   v
 [json parser] -> [helmet] -> [auth] -> [joi validate] -> [handler]
      |              |          |            |               |
      +----------- next(err) on any failure ----------------+
                          v
        [error handler (err,req,res,next)]   <- registered LAST
 Each step: next() OR send response OR next(err). None = request hangs.
```
**Diagram: the async trap**
```
 async handler throws
   +- no wrapper  : Promise rejects, Express never sees it -> unhandledRejection
   +- asyncHandler: .catch(next) -> error handler -> clean error JSON
```
**In plain words.** An airport: each checkpoint either waves you on (next), turns you away
(response), or calls security (next(err)). A checkpoint that does nothing leaves you stuck
forever. The wrapper makes sure errors from async handlers reach security.

**Core idea.** Express is a **pipeline** of functions `(req, res, next)`. Each has exactly three legal
moves: call `next()` (pass on), **send a response** (end it), or `next(err)` (jump to error handling).

- Forget `next()` and don't respond → the request **hangs** forever.
- **Order matters**: body parser before body readers; auth before the handler; security headers early.
- **Error handlers** are recognized by **4-arg arity** `(err, req, res, next)` and must be registered
  **last**.

**The async trap (B1/B4).** An `async` handler returns a **Promise immediately**. Express 4 doesn't
`await` it, so a `throw` inside becomes a **rejected promise Express never sees** → `unhandledRejection`
(not your error handler). Fix with a wrapper:
```js
const asyncHandler = fn => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);
```
Express 5 handles this natively.

> **One line:** middleware is an ordered pipeline; each step calls next(), responds, or next(err);
> error handlers are 4-arg and last; async throws need asyncHandler in Express 4.

---

## B2 — Sub-routers & mount paths

**Diagram**
```
 app.use('/orgs/:orgId/users', usersRouter)
 request: GET /orgs/7/users/42
   req.originalUrl = /orgs/7/users/42   (full)
   req.url         = /42                (prefix stripped)
 usersRouter needs express.Router({ mergeParams: true }) to read :orgId
```
**In plain words.** The router is a department that only sees the part of the address after its
own name. To also see the parent's `:orgId`, switch on `mergeParams`.

**Core idea.** `express.Router()` is a mountable mini-app. Express **strips the mount prefix** before
the router matches, so router paths are **relative**.

- `req.url` = stripped path (inside the router); `req.originalUrl` = full path.
- Middleware scope: `app.use()` (all routes) / `router.use()` (that branch) / route-level (one route).
- **`mergeParams: true`** is required for a nested router to see the parent's `:params` — classic bug
  when a child router can't read `:orgId` from the parent.

> **One line:** a Router is a mountable mini-app whose prefix Express strips before matching; use
> mergeParams:true so nested routers can read parent params.

---

## B3 — param vs query vs body vs header vs cookie
- **param** (`/orders/:id`) — *which* resource.
- **query** (`?sort=asc`) — *how* to shape the response. Never put secrets in the URL (logged).
- **body** — the payload.
- **header** — request metadata (content-type, auth).
- **cookie** — browser auto-sends it.

**All five are untrusted / attacker-controlled.** Validate every source (D1).

> **One line:** param=which, query=how (no secrets), body=payload, header=metadata, cookie=auto-sent —
> all five untrusted.

---

## B4 — Async error propagation
**Core idea.** In an awaited chain, errors **bubble up for free** — no try/catch at every layer.

- Missing an `await` (fire-and-forget) → the response may send `200`, then the later rejection becomes
  an `unhandledRejection` and can crash the process.
- `Promise.all` **rejects on the first failure**; use `Promise.allSettled` when you want partial
  success.

> **One line:** awaited errors bubble to the error handler; a missing await orphans the rejection;
> Promise.all is all-or-nothing (use allSettled for partial).

---

## B5 — The validation gate (Joi) = the trust boundary

**Diagram**
```
 UNTRUSTED                    TRUSTED
 body/query/params --> [Joi: whitelist + coerce + defaults] --> controller
 { name, isAdmin:true }        stripUnknown drops isAdmin       { name }
```
**In plain words.** A security guard at the door of your code. Only fields on his list get in.
Anything extra, like `isAdmin:true`, is removed before your logic sees it.

**Core idea.** A validation layer that runs **after** body-parsing and **before** the controller. It's
where untrusted input becomes trusted.

- **Whitelist** with `stripUnknown` → stops **mass-assignment** (attacker sends `isAdmin:true`).
- Validates + **coerces** + fills **defaults**.
- `abortEarly:false` to collect all errors.
- Validate **body + query + params** (B3), on every route, in every service.

> **One line:** the Joi gate whitelists/coerces input after parsing and before the controller — it's
> the trust boundary that stops mass-assignment.

---

## B6 — Controller / Service / Repository

**Diagram**
```
 Route -> Controller -> Service -> Repository -> DB
          (HTTP only)   (rules)    (queries)
                           ^
          jobs / CLI call the Service directly (no req/res needed)
```
**In plain words.** Restaurant: the waiter (controller) takes the order, the chef (service)
decides how to cook, the storekeeper (repository) fetches ingredients. The chef never talks to
customers, so anyone can ask the chef, including a scheduled job.

- **Controller** — HTTP only (read `req`, write `res`). No business logic.
- **Service** — business logic; **no HTTP** — reusable from jobs, CLI, other services.
- **Repository** — data access only.

Layering buys testability, reusability, swappability, and clean transaction boundaries.

> **One line:** controller = HTTP, service = business logic (HTTP-free, reusable), repo = data access.

---

## B7 — DTO + response envelope
**Core idea.** Never `res.json` a raw DB row (it leaks `password_hash`, internal fields).

- **DTO (output whitelist)** decouples your API shape from the DB schema.
- **Consistent envelope**: `{ success, data }` / `{ success, error: { code, message } }`.
- `error.code` = a **stable machine-readable string** the client can switch on.

> **One line:** map rows to DTOs (never leak raw rows) and return a consistent {success,data}/{error}
> envelope with a stable error.code.

---

## B8 — Pagination (offset vs keyset)

**Diagram**
```
 OFFSET 1000000 LIMIT 10         WHERE id > :last ORDER BY id LIMIT 10
 +-------------------+           +-------------------+
 | read 1M rows      |           | index seek to id  |
 | throw them away   |           | read 10 rows      |
 | return 10         |           +-------------------+
 +-------------------+           fast, stable, no page jump
 slow, rows shift under writes
```
**In plain words.** Offset: to find page 100 you flip through 99 pages first. Keyset: you kept a
bookmark, so you open straight at it. The catch is you cannot jump to a random page number.

- **Offset/limit** — `OFFSET n` **scans and discards** the first n rows: O(n), so deep pages are slow
  and a **DoS vector**; also **drifts** under concurrent writes (rows shift between pages).
- **Keyset / cursor** — `WHERE id > :last ORDER BY id LIMIT n`: **O(log n)** index seek, stable under
  writes — but **no random page jumps** and no easy total count. Needs a **unique, indexed, sortable**
  key.
- Offset is fine for **small bounded** tables; use keyset for large or infinite-scroll lists.

> **One line:** offset scans+discards a prefix (slow, drifts); keyset seeks by a unique indexed key
> (fast, stable, no page-jump) — offset for small tables, keyset for large.

---

## B9 — Filter / sort / search safely

**Diagram**
```
 ?sort=price     -> SORTABLE['price']   = 'price'            -> ORDER BY price
 ?sort=1;DROP..  -> SORTABLE[...] undefined ?? 'created_at'  -> safe
 values: bind ($1)   |   column names: whitelist only
```
**In plain words.** A form letter: the blanks (values) can be filled by anyone. The wording
(column names) is fixed by you from an approved list.

**Core idea (the deep one).** You can **bind values** but you must **whitelist identifiers**.

- **Values** → parameterize (`WHERE status = $1`, values array).
- **Identifiers** (column names, `ASC`/`DESC`) **cannot be parameterized** — the parser needs the query
  *structure* before the value arrives, so a placeholder can only fill a value hole, never change the
  shape. Map user input to a **whitelist**: `SORTABLE[input] ?? 'created_at'`.
- **Search**: leading-wildcard `LIKE '%x%'` **can't use a B-tree index** (index is ordered by prefix).
  Use a prefix match, or **trigram** (`pg_trgm` + GIN), or **full-text** (`tsvector` + GIN).
- Always **cap** `limit`.

> **One line:** bind values, whitelist identifiers (parser locks shape before the value arrives);
> leading-wildcard LIKE can't use a B-tree — use trigram/full-text; cap the limit.
