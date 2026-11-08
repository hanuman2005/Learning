# FastAPI for Backend Engineers

A working guide to building, testing and operating production HTTP APIs with FastAPI. It teaches the model underneath the framework (ASGI, the event loop, Pydantic, dependency injection), so you can predict how a service behaves under load and when it fails, not just copy decorators.

**Audience:** backend engineers, especially those coming from Node/Express, who want to build and run Python APIs at production level.
**Prerequisites:** working Python (see `04_Python.md`, especially typing, context managers, asyncio and pytest), HTTP basics, SQL basics. Linux and process basics from `03_Linux.md` help in chapter 15.
**Scope:** FastAPI with Starlette, Pydantic v2, SQLAlchemy 2.0 and Uvicorn. It does not teach Python itself, SQL tuning or front-end work.
**Note on accuracy:** FastAPI, Pydantic, SQLAlchemy and their ecosystem change quickly. Concepts here are stable. Names and defaults that changed across versions are flagged "confirm in current docs". Check the official documentation for the versions you pin before relying on a detail.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Each chapter starts with an "In plain words" summary. Hard terms are defined in brackets at first use and in the glossary (chapter 23). Chapter 20 lists hands-on labs. Chapter 21 collects interview questions with model answers. Chapter 22 lists primary references.

**What "good enough" means for a backend role:** you can build a typed, tested, authenticated API with a database, explain why a handler blocks the event loop, deploy it with the right process model behind a proxy, and diagnose it in production. Chapters 1, 4, 5, 9, 15 and 17 matter most.

## Contents

1. The mental model: ASGI, Starlette, Uvicorn, Pydantic
2. The request lifecycle and parameters
3. Pydantic models and response models
4. async def versus def: the event loop and the threadpool
5. Dependency injection
6. Lifespan, middleware, exceptions and error design
7. Authentication and authorisation
8. Configuration
9. Database access with SQLAlchemy 2.0
10. Background tasks versus a real queue
11. Files, streaming, SSE and WebSockets
12. OpenAPI generation and customisation
13. Project layout
14. Testing
15. Performance and deployment
16. Serving ML models safely
17. Observability
18. Security checklist
19. Choosing a framework
20. Practice labs
21. Interview questions
22. References
23. Glossary

---

## 1. The mental model: ASGI, Starlette, Uvicorn, Pydantic

**In plain words.** FastAPI is a thin layer that sits on top of other tools. One tool accepts network connections, one routes requests, and one checks data. Knowing which tool does which job helps you find the cause of a problem.

### Understand

**WSGI versus ASGI.** WSGI (used by Flask and classic Django) is a synchronous contract. A server calls your app once per request, and the call blocks until it returns a response. One worker handles one request at a time. It has no model for long-lived connections. **ASGI** (Asynchronous Server Gateway Interface) is the asynchronous successor. An ASGI app is an `async` callable that receives a `scope` (connection metadata) and two awaitables, `receive` and `send`, which exchange events (request body chunks, response start, response body, WebSocket messages, lifespan events). Because the app awaits instead of blocking, one process can interleave thousands of connections while each waits on I/O. ASGI also standardises WebSockets and lifespan (startup and shutdown).

**The layers.** FastAPI is not a server and not a complete framework. It is a thin, typed layer on top of other libraries:

| Layer | Role | Express analogy |
|---|---|---|
| Uvicorn (or Hypercorn) | ASGI server: accepts sockets, parses HTTP, speaks ASGI to the app | Node's `http` server |
| Starlette | ASGI toolkit: routing, middleware, requests, responses, WebSockets, test client | Express core and Connect middleware |
| Pydantic | data validation, coercion and serialisation from type hints | Joi, Zod or class-validator |
| FastAPI | glue: turns type hints into parameter parsing, validation, dependency injection and an OpenAPI schema | the layer Express lacks |

FastAPI subclasses Starlette's application, so anything true of Starlette (middleware, `Request`, `Response`, `StreamingResponse`, `TestClient`) applies to FastAPI.

**The central idea.** You declare types, and the framework derives behaviour. A function signature becomes a contract: the framework reads it to decide where each value comes from (path, query, body, header), validates it, calls your function, validates and filters the return value, and builds the OpenAPI document from the same information. That single source of truth is the reason to choose it.

**One event loop per process.** Each Uvicorn worker runs one asyncio event loop on one thread. Concurrency is cooperative: a coroutine runs until it awaits. Anything that does not yield (CPU work, a blocking library call) stalls every other request in that process. Chapter 4 is built on this.

A minimal app is `app = FastAPI()` with `@app.get("/health")` on an `async def health()`. Run it with `uvicorn app.main:app`, where the string is module path and attribute name. FastAPI also ships a `fastapi` CLI (`fastapi dev`, `fastapi run`); confirm its current flags in the docs.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Serving | `uvicorn`, `--reload` in dev, `--workers` | Start an app and explain each flag |
| ASGI internals | Starlette docs, a hand-written ASGI callable | Explain `scope`, `receive`, `send` |
| Interactive docs | `/docs` (Swagger UI), `/redoc` | Use them to exercise and review an API |

### Common mistakes

- Thinking `async def` makes code faster. It only allows other work to run during `await`.
- Assuming one async process uses all CPU cores. It uses one.

### Self-check

- What do `scope`, `receive` and `send` represent in ASGI?
- Which responsibilities belong to Uvicorn, Starlette, Pydantic and FastAPI respectively?
- Why does one async worker process use only one CPU core, and how do you use more?

---

## 2. The request lifecycle and parameters

**In plain words.** A request passes through fixed steps: the server, middleware, routing, validation, your function, and the response. FastAPI reads your function signature to find where each value comes from. If a value is invalid, your function never runs.

### Understand

**What happens to a request.**

1. Uvicorn parses the HTTP request and calls the ASGI app.
2. The middleware stack runs, outermost first (chapter 6).
3. Starlette's router matches the path and method to a route.
4. FastAPI resolves the route's **dependencies** and parameters from the request: path, query, headers, cookies, body. Each is validated by Pydantic. Any failure produces a `422` response (`RequestValidationError`) and the handler never runs.
5. Your handler runs (async on the loop, or sync in a threadpool).
6. The return value is validated and filtered through the `response_model`, then serialised to JSON.
7. Middleware unwinds in reverse order. Yield-dependency cleanup and background tasks run. In current FastAPI, the exit code of a `yield` dependency runs after the response is sent by default (`scope="request"`). With `scope="function"` it runs after the handler returns but before the response is sent. This changed between versions, so check the docs for the version you pin.

**Where each parameter comes from.** FastAPI infers the source from the declaration:

- A name that appears in the path template is a **path parameter**.
- A scalar type (`int`, `str`, `bool`, `datetime`) not in the path is a **query parameter**.
- A Pydantic model parameter is a **JSON body**.
- `Header`, `Cookie`, `Query`, `Path`, `Body`, `Form` and `File` make the source explicit and add constraints and metadata.

```python
from typing import Annotated
from fastapi import FastAPI, Query, Path, Header, Cookie

app = FastAPI()

@app.get("/items/{item_id}")
async def read_item(
    item_id: Annotated[int, Path(ge=1)],
    q: Annotated[str | None, Query(max_length=50)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    user_agent: Annotated[str | None, Header()] = None,
    session: Annotated[str | None, Cookie()] = None,
):
    ...
```

The `Annotated[...]` style is the currently recommended form in the docs (confirm for your version). Header names convert underscores to hyphens (`user_agent` reads `User-Agent`).

**Bodies and forms.** A JSON body is one model parameter. Two body models in one handler expect a JSON object keyed by parameter name. Forms and file uploads use `Form` and `File` and require the `python-multipart` package. JSON and form bodies cannot be mixed in one request.

**Semantics.** Set `status_code=201` on creation and `204` for no content. Declare `limit` and `offset` (or a cursor) as bounded query parameters, since unbounded lists are an outage waiting for a large table.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Parameter sources | `Path`, `Query`, `Header`, `Cookie`, `Body`, `Form`, `File` | Choose the source and constrain it |
| Reading raw data | `Request` object, `request.headers`, `await request.body()` | Know when to drop below the model layer |
| Inspecting behaviour | `/docs`, `curl -i`, `httpie` | Reproduce a 422 and read its error body |

### Common mistakes

- Unbounded `limit` or no pagination.
- Returning `200` with an error payload instead of the right status.

### Self-check

- Given a route and a signature, which parameters are path, query and body, and why?
- What status code does validation failure return, and where is it produced?
- Why should every list endpoint have a bounded `limit`?

---

## 3. Pydantic models and response models

**In plain words.** Pydantic turns incoming data into typed Python objects and rejects bad data. Use separate models for input and output, so private fields are never sent to the client. This is a main safety tool in FastAPI.

### Understand

**Pydantic parses, it does not just check.** In its default (lax) mode Pydantic **coerces** (converts a value to the declared type): the string `"5"` becomes `5` for an `int` field, `"true"` becomes `True`. It validates and then produces typed Python objects. This matches HTTP reality, where query strings and form values arrive as text.

**Version note.** This guide uses Pydantic v2 names (`model_validate`, `model_dump`, `model_config`, `field_validator`, `model_validator`). Pydantic v1 used `parse_obj`, `dict()`, `class Config` and `validator`. Older tutorials mix them. The mapping is confirmed in the official migration guide (`parse_obj` to `model_validate`, `dict()` to `model_dump`, `class Config` to `model_config`, `validator` to `field_validator`, `root_validator` to `model_validator`). The v1 names still work but emit deprecation warnings. `BaseSettings` moved to the separate `pydantic-settings` package.

```python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, EmailStr, Field, field_validator

class UserCreate(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    email: EmailStr                       # needs the email-validator package
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=13, le=120)

    @field_validator("name")
    @classmethod
    def no_control_chars(cls, v: str) -> str:
        if not v.isprintable():
            raise ValueError("name must be printable")
        return v
```

**Constraints.** `Field(...)` carries `ge`, `gt`, `le`, `lt`, `min_length`, `max_length`, `pattern`, `default`, `default_factory`, `alias` and documentation metadata. Constraints appear in the OpenAPI schema automatically.

**Strict mode.** Strict mode disables coercion, so `"5"` is rejected for an `int`. Enable it per model with `ConfigDict(strict=True)`, per field, or with strict types such as `StrictInt`. Use it for internal machine-to-machine payloads and money-like fields where silent coercion hides client bugs. Be careful on public JSON APIs: strict mode in JSON mode still accepts some JSON-native conversions and rejects others, so test your exact types against the documented conversion table.

**Validators.** `field_validator` checks or transforms one field. `model_validator` (modes `before` and `after`) checks relationships between fields, for example `end > start`. Keep validators pure and cheap. Do not call the database inside one.

**`model_config`.** Common settings: `extra="forbid"` (reject unknown fields, catches typos and mass-assignment), `from_attributes=True` (build from ORM objects), `str_strip_whitespace`, `frozen`, `populate_by_name`.

**Serialisation.** `model.model_dump()` gives a dict, `model_dump(mode="json")` gives JSON-safe values, `model_dump_json()` gives a string. `exclude_unset=True` helps PATCH semantics: only fields the client sent.

**Separate input, storage and output models.** Never use one model for everything.

```python
class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: EmailStr

@app.post("/users", response_model=UserOut, status_code=201)
async def create_user(body: UserCreate, session: SessionDep): ...
```

**`response_model` filters output.** FastAPI validates the returned value against `UserOut` and serialises only its fields. A returned ORM row containing `password_hash` is stripped because `UserOut` lacks that field. This is a safety net, not a substitute for careful models. You can also declare the return type annotation, and FastAPI uses it as the response model (confirm behaviour in your version). `response_model_exclude_unset` and related options control optional fields.

**PATCH.** Define an all-optional update model and apply `body.model_dump(exclude_unset=True)`. Without `exclude_unset`, omitted fields overwrite stored values with `None`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Validation | `Field`, `field_validator`, `model_validator` | Express cross-field rules declaratively |
| Coercion control | `ConfigDict(strict=True)`, `StrictInt`, `StrictStr` | Know when coercion helps and when it hides bugs |
| Serialisation | `model_dump`, `model_dump_json`, `exclude_unset` | Implement correct PATCH semantics |
| Types | `EmailStr`, `HttpUrl`, `UUID`, `Decimal`, `Literal`, `Enum`, `Annotated` | Model the domain, not just primitives |

### Common mistakes

- One model for input, DB and output, leaking internal fields.
- Using `float` for money. Use `Decimal` or integer minor units.
- Doing I/O in validators, or mixing v1 and v2 names from old tutorials.

### Self-check

- What does lax mode do with `"5"` for an `int`, and what does strict mode do?
- How does `response_model` prevent leaking a password hash?
- Why does PATCH need `exclude_unset=True`?

---

## 4. async def versus def: the event loop and the threadpool

**In plain words.** One process runs one event loop (a loop that switches between tasks while they wait). If your code blocks the loop, every other request in that process waits. Use `async def` only with awaited calls, and use plain `def` for blocking libraries.

### Understand

**The rule.** FastAPI treats the two function kinds differently:

- `async def` endpoint: runs **on the event loop** directly. Fast to start, but must never block.
- plain `def` endpoint: FastAPI runs it **in a threadpool** (via AnyIO), so a blocking call does not stall the loop. The pool is bounded. AnyIO's default thread limiter is 40 threads (confirmed in the AnyIO docs). It can be changed at runtime through `anyio.to_thread.current_default_thread_limiter().total_tokens`. Dependencies follow the same rule.

**The cardinal pitfall.** Calling a blocking function inside `async def` freezes the whole worker. Examples: `requests.get(...)`, `time.sleep(...)`, a synchronous database driver, `open().read()` on a large file, heavy CPU work, `bcrypt` hashing, image processing. While it runs, the loop cannot serve any other connection in that process. Health checks time out and latency of unrelated requests spikes. The symptom looks like the whole service is slow while CPU shows one core busy.

```python
import time
from typing import Annotated
import httpx
from fastapi import Depends, Request

async def get_http(request: Request) -> httpx.AsyncClient:
    return request.app.state.http          # client created in lifespan (chapter 6)

HttpDep = Annotated[httpx.AsyncClient, Depends(get_http)]

@app.get("/bad")
async def bad():
    time.sleep(2)              # blocks the loop: every request waits

@app.get("/ok-async")
async def ok_async(http: HttpDep):
    return (await http.get("https://example.com")).status_code   # shared AsyncClient

@app.get("/ok-sync")
def ok_sync():
    time.sleep(2)              # runs in a worker thread; the loop stays free
```

**Choosing.**

- Use `async def` when every I/O call inside is awaited through an async library (`httpx.AsyncClient`, `asyncpg`, SQLAlchemy async, `redis.asyncio`).
- Use `def` when you must call blocking libraries. The threadpool is the escape hatch.
- Do not choose `async def` "to be modern" and then call blocking code.

**Escape hatches inside async code.** `await asyncio.to_thread(fn, ...)` or `starlette.concurrency.run_in_threadpool` moves a blocking call off the loop. For genuine CPU-bound work, threads do not help because of the GIL (see `04_Python.md`, concurrency). Use a process pool, a separate worker service or a queue (chapter 10).

**Detecting it.** Enable asyncio debug mode in development (it logs slow callbacks), measure loop lag, and load test an endpoint while hitting `/health`. If health latency climbs under load, something blocks the loop.

**Mixing is fine.** A service can contain both kinds. Be deliberate per endpoint. A sync endpoint using a sync SQLAlchemy session is a legitimate, simpler design if the threadpool bound meets your concurrency needs.

**Express contrast.** Node has the same single-threaded loop and the same hazard (`fs.readFileSync`, heavy JSON or crypto in a handler). Node has no automatic threadpool for your handlers, so Python's plain `def` fallback is a convenience you do not have there.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Async HTTP | `httpx.AsyncClient` with a shared client and timeouts | Never use `requests` in async code |
| Offloading | `asyncio.to_thread`, `run_in_threadpool` | Wrap one blocking call correctly |
| Detecting blocking | asyncio debug mode, `py-spy dump`, load test plus health probe | Prove a handler blocks the loop |

### Common mistakes

- `time.sleep`, `requests`, sync DB drivers or file reads in `async def`.
- Creating a new `httpx.AsyncClient` per request instead of sharing one (no connection reuse).
- No timeouts on outbound calls, so one slow dependency exhausts capacity.

### Self-check

- What happens to other requests when an `async def` handler calls `time.sleep(5)`?
- Why do threads not speed up CPU-bound work, and what do you use instead?
- How do you prove, with a measurement, that a handler blocks the loop?

---

## 5. Dependency injection

**In plain words.** A dependency is a small function that FastAPI calls for you before your handler. It can give you a database session, the current user, or settings. This keeps shared logic in one place and makes tests easy, because you can swap a dependency.

### Understand

**What it is.** A dependency is a callable the framework runs before your handler, whose result is passed in. `Depends(fn)` declares it. FastAPI resolves the whole tree: dependencies can have their own dependencies (sub-dependencies), and FastAPI evaluates them in order, validating their parameters like any other function.

**Why it matters.** Dependencies are how you share logic without globals: database sessions, the current user, pagination, permission checks, settings, rate limiters. They appear in OpenAPI where relevant (security schemes), and they are the seam that makes testing easy.

```python
async def pagination(limit: Annotated[int, Query(ge=1, le=100)] = 20, offset: int = 0) -> dict[str, int]:
    return {"limit": limit, "offset": offset}

Pagination = Annotated[dict[str, int], Depends(pagination)]   # alias removes repetition
```

The `Annotated[..., Depends(...)]` style is current in the docs; confirm for your version.

**Yield dependencies for resources.** A dependency that `yield`s acts like a context manager: code before `yield` runs at setup, the yielded value is injected, code after `yield` runs for cleanup.

```python
async def get_session():
    async with SessionLocal() as session:   # opened per request
        yield session                       # injected into the handler
    # leaving the block closes the session

SessionDep = Annotated[AsyncSession, Depends(get_session)]
```

Use this for sessions, transactions and per-request resources. Exceptions raised in the handler propagate into the dependency at the `yield`, so you can roll back. By default the cleanup code runs after the response is sent. Pass `scope="function"` to `Depends` if cleanup must run before the response goes out. A request-scoped dependency may only have request-scoped sub-dependencies. Check the docs for your version, since this timing changed between releases. If you catch an exception in a `yield` dependency, re-raise it, or FastAPI cannot see the error.

**Caching per request.** If the same dependency appears several times in one request's tree, it runs once and its result is reused (`use_cache=True` is the default). Pass `Depends(fn, use_cache=False)` to force a fresh call. This is why `get_current_user` can be required by both a route and a sub-dependency without hitting the database twice. The cache is per request, not across requests.

**Dependencies at other levels.** `dependencies=[Depends(...)]` on a decorator, an `APIRouter`, `include_router` or `FastAPI(...)` runs for those routes without injecting a value. This is the clean way to apply auth to a whole router.

**Overriding in tests.** `app.dependency_overrides[get_session] = fake_get_session` replaces a dependency for the whole app. Tests swap the real database, auth and external clients without monkeypatching. Always clear overrides after the test (chapter 14).

**Express contrast.** Express middleware chains mutate `req`. DI is explicit and typed: the handler's signature declares exactly what it needs, and nothing is attached to a shared mutable object.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Declaring | `Depends`, `Annotated`, aliases | Compose small dependencies |
| Resources | `yield` dependencies, `contextlib.asynccontextmanager` | Open and close a session safely per request |
| Scoping | router-level `dependencies=[...]`, `use_cache` | Apply cross-cutting rules once |
| Testing | `app.dependency_overrides` | Replace DB and auth in tests |

### Common mistakes

- Using module-level global sessions or clients instead of dependencies.
- Forgetting to clear `dependency_overrides` between tests.
- Putting business logic inside dependencies instead of services (chapter 13).

### Self-check

- What does FastAPI do when two sub-dependencies both depend on `get_current_user`?
- How does a `yield` dependency release a database session if the handler raises?
- How do you replace the database dependency in a test, and what must you do afterwards?

---

## 6. Lifespan, middleware, exceptions and error design

**In plain words.** Some things live as long as the whole process: database connections, HTTP clients, loaded models. Create them once in lifespan. Middleware wraps every request, and exception handlers turn errors into one consistent response format.

### Understand

**Lifespan.** Resources that should exist for the whole process (database engine, HTTP client, loaded ML model, Redis pool) are created at startup and closed at shutdown. The current mechanism is a `lifespan` async context manager passed to `FastAPI(...)`. The older `@app.on_event("startup")` and `"shutdown"` decorators are deprecated (confirmed in the official docs). If you pass `lifespan`, the old startup and shutdown handlers are no longer called. Use all lifespan or all events, not both.

```python
from contextlib import asynccontextmanager
import httpx
from fastapi import FastAPI, Request

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = httpx.AsyncClient(timeout=5.0)   # startup
    yield
    await app.state.http.aclose()                     # shutdown

app = FastAPI(lifespan=lifespan)

async def get_http(request: Request) -> httpx.AsyncClient:
    return request.app.state.http
```

Expose lifespan resources to handlers through a dependency that reads `request.app.state`, so tests can override it. If startup raises, the process fails to start, which is what you want for a missing database. Note that test clients run lifespan only in some usage modes (chapter 14).

**Middleware.** Code that wraps every request and response: logging, request IDs, CORS, compression, timing, security headers. Order matters: the last added is the outermost. Common built-ins: `CORSMiddleware`, `GZipMiddleware`, `TrustedHostMiddleware`, `HTTPSRedirectMiddleware`.

The simplest form is `@app.middleware("http")` on an `async def (request, call_next)` that awaits `call_next(request)` and adds headers such as a request ID. The `@app.middleware("http")` decorator is built on Starlette's `BaseHTTPMiddleware`. Starlette documents one limitation: `contextvars` changes do not propagate upward through it, and it can also break `contextvars` for pure ASGI middleware placed later in the stack. Limits with streaming responses and background tasks are widely reported but were not confirmed on the Starlette middleware page (unverified). For high-traffic or streaming services, write a pure ASGI middleware instead.

**Exception handlers.** `HTTPException(status_code=404, detail="...")` produces an error response from anywhere in a handler or dependency. Register handlers with `@app.exception_handler(MyError)` to map domain exceptions to HTTP responses. Override the handler for `RequestValidationError` to reshape 422 bodies. Raise domain exceptions (`OrderNotFound`) from services and translate them at the edge, so services stay HTTP-agnostic.

**Error response design.** Pick one error shape and use it everywhere. A reasonable choice is RFC 9457 "problem details" (`application/problem+json`: `type`, `title`, `status`, `detail`, `instance`) or a simple envelope such as:

```json
{"error": {"code": "order_not_found", "message": "Order 42 does not exist", "request_id": "..."}}
```

Rules: a stable machine-readable `code` clients can branch on; a human `message`; the request ID for support; correct status codes (`400` malformed, `401` unauthenticated, `403` forbidden, `404`, `409` conflict, `422` validation, `429` rate limited); never leak stack traces, SQL or internal paths. Add a catch-all handler for unexpected exceptions that logs with the request ID and returns a generic `500`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Lifespan | `lifespan=`, `app.state`, `asynccontextmanager` | Manage engine, clients, models |
| Middleware | `@app.middleware`, pure ASGI, `CORSMiddleware`, `GZipMiddleware` | Choose the right kind and order |
| Errors | `HTTPException`, `exception_handler`, `RequestValidationError` | Map domain errors to one consistent shape |

### Common mistakes

- `allow_origins=["*"]` together with credentials. Configure CORS explicitly.
- Letting exceptions leak internal detail in `500` bodies.
- HTTP-aware business exceptions, coupling services to the web layer.

### Self-check

- Why is lifespan preferred to creating clients at import time?
- In what order do middlewares run if you add A then B?
- Why do services raise domain exceptions instead of `HTTPException`?

---

## 7. Authentication and authorisation

**In plain words.** Authentication checks who the caller is. Authorisation checks what that caller may do. FastAPI helps you read credentials, but you must verify them and check permissions yourself.

### Understand

**Separate the two.** Authentication answers "who is this?" Authorisation answers "may they do this to that resource?" FastAPI gives you transport and documentation helpers and the dependency mechanism. It does not give you user storage, sessions, password policy or tokens. You assemble those, or use a vetted library or an identity provider.

**Security utilities.** `fastapi.security` provides `OAuth2PasswordBearer` (reads `Authorization: Bearer <token>` and registers the scheme in OpenAPI so Swagger UI shows an Authorize button), `OAuth2PasswordRequestForm` (the form body for the password flow), `HTTPBearer`, `HTTPBasic`, and `APIKeyHeader`, `APIKeyQuery`, `APIKeyCookie`. These only extract credentials. You still verify them.

**Password flow with JWT.** The documented tutorial pattern:

1. Client posts `username` and `password` as form data to a token endpoint.
2. Server looks up the user, verifies the password against a stored hash, and returns a signed JWT with a subject and expiry.
3. Client sends `Authorization: Bearer <jwt>` on later calls.
4. A `get_current_user` dependency decodes the token, checks signature and expiry, loads the user, and raises `401` with a `WWW-Authenticate: Bearer` header on failure.

```python
oauth2 = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(token: Annotated[str, Depends(oauth2)], session: SessionDep) -> User:
    try:
        claims = decode_jwt(token)      # your function: signature, exp, aud, iss
    except InvalidToken:
        raise HTTPException(401, "Invalid credentials", headers={"WWW-Authenticate": "Bearer"})
    user = await session.get(User, int(claims["sub"]))
    if user is None or not user.is_active:
        raise HTTPException(401, "Invalid credentials", headers={"WWW-Authenticate": "Bearer"})
    return user
```

Library choices change. The current official tutorial (confirmed) uses PyJWT (`jwt.encode`, `jwt.decode(..., algorithms=[...])`) for tokens and `pwdlib` with Argon2 (`PasswordHash.recommended()`) for passwords. Older material uses `python-jose` and `passlib`, which are less maintained. Never store plain passwords, never invent a hashing scheme, and verify with a constant-time comparison from the library.

**JWT cautions.** Pin the allowed algorithm explicitly when decoding. Validate `exp`, and `iss` and `aud` where relevant. Keep access tokens short-lived. A JWT cannot be revoked before expiry without a server-side denylist or short lifetimes plus refresh tokens. Store the signing secret in configuration (chapter 8), and rotate it. For consumer or enterprise apps, prefer delegating to an identity provider (OIDC) and validating its tokens against its published keys, rather than running your own credential store.

**API keys.** `APIKeyHeader` extracts the key, a dependency looks up its hash (store hashes, not raw keys) and identifies the caller. Support rotation and revocation.

**Authorisation with dependencies.** Express permissions as dependencies that compose:

```python
def require_role(*roles: str):
    async def checker(user: Annotated[User, Depends(get_current_user)]) -> User:
        if user.role not in roles:
            raise HTTPException(status.HTTP_403_FORBIDDEN, "Insufficient permissions")
        return user
    return checker

@router.delete("/users/{user_id}", dependencies=[Depends(require_role("admin"))])
async def delete_user(user_id: int): ...
```

RBAC (roles to permissions) covers coarse rules. Many real bugs are object-level: a valid user reading another user's order (broken object-level authorisation). Enforce ownership in the query itself (`WHERE id = :id AND owner_id = :user`), not only in a role check. `Security(..., scopes=[...])` with OAuth2 scopes is available when scopes fit your model.

**Fail closed.** Default to denial. Put the authentication dependency on the router (or the whole app) and opt public routes out explicitly, so a new route is protected unless someone deliberately makes it public. An unknown role, a missing claim or a lookup error means deny. Return `404` instead of `403` when revealing existence is itself sensitive.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Credential extraction | `OAuth2PasswordBearer`, `HTTPBearer`, `APIKeyHeader` | Pick the scheme and see it in `/docs` |
| Tokens | PyJWT | Issue and verify with pinned algorithm and expiry |
| Passwords | Argon2 via `pwdlib` (or another maintained library) | Hash and verify correctly, never roll your own |
| Authorisation | dependency factories, `Security` scopes, router-level `dependencies` | Implement RBAC and ownership checks |

### Common mistakes

- Accepting the `none` algorithm or not pinning algorithms, or skipping `exp`.
- Authorising by role but not by object ownership.
- Opt-in security: each route must remember to add auth.
- Sensitive data in JWT payloads (they are encoded, not encrypted).
- Different errors for "no such user" and "wrong password" (user enumeration).

### Self-check

- How do you make every new route in a router authenticated by default?
- A user can fetch `/orders/7` for someone else's order. Which class of bug is this and how do you fix it at query level?
- Why can a JWT not be revoked before it expires, and how do you limit the risk?

---

## 8. Configuration

**In plain words.** Settings such as the database address and secrets change between environments. Read them from environment variables, not from code. Check them at startup, so a missing value stops the program at once.

### Understand

**Twelve-factor configuration.** Settings that differ per environment (database URL, secrets, feature flags) come from environment variables, not code. `pydantic-settings` (a separate package from `pydantic` in v2) provides `BaseSettings`, which reads and validates environment variables and optional `.env` files into a typed object. Failing validation at startup is the point: a missing `DATABASE_URL` stops the process immediately instead of failing at the first query.

```python
from functools import lru_cache
from pydantic import SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="APP_")

    database_url: str
    jwt_secret: SecretStr
    jwt_ttl_minutes: int = 15
    debug: bool = False

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Expose it as a dependency (`Depends(get_settings)`) so tests override it. `SecretStr` prevents accidental logging of the value in reprs. Confirm the exact `SettingsConfigDict` options in the pydantic-settings docs for your version.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Typed settings | `pydantic-settings`, `SettingsConfigDict` | Load, validate and fail fast |
| Secrets | `SecretStr`, platform secret stores | Keep secrets out of code, logs and images |

### Common mistakes

- Reading `os.environ` all over the codebase with no validation.
- Committing `.env` or baking secrets into a Docker image layer.

### Self-check

- Why should a missing required setting crash startup?
- How do you replace settings in a test without touching environment variables?
- Why use `SecretStr` for a secret field?

---

## 9. Database access with SQLAlchemy 2.0

**In plain words.** Use one database session per request and never share it between tasks. Load related data on purpose, because async code cannot load it lazily. Plan the size of the connection pool, because every worker and replica has its own.

### Understand

**The stack.** SQLAlchemy 2.0 provides an async API (`AsyncEngine`, `AsyncSession`) over an async driver (`asyncpg` for PostgreSQL, via the URL scheme `postgresql+asyncpg://`). The 2.0 style uses `select()` statements, `Mapped[...]` type-annotated models and `mapped_column`. Older 1.x `Query` API tutorials are outdated. SQLModel is a related option that combines Pydantic and SQLAlchemy models; evaluate whether its coupling suits you.

```python
from sqlalchemy import select, String
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase): ...

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@db/app",
    pool_size=10, max_overflow=5, pool_pre_ping=True,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_user(session: AsyncSession, user_id: int) -> User | None:
    result = await session.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()
```

`expire_on_commit=False` is recommended in async code (the SQLAlchemy asyncio docs say it should normally be False), because attribute access on an expired object would trigger implicit lazy I/O, which async sessions cannot perform.

**Session per request.** One `AsyncSession` per request via a yield dependency. A session is a unit of work holding an identity map and a connection checked out of the pool when needed. It is **not** thread-safe or task-safe: never share one session across concurrent tasks (for example inside `asyncio.gather`), and never keep one in a global.

**Lazy loading and async.** Accessing an unloaded relationship in async code raises an error (`MissingGreenlet`) because it would need implicit I/O. Load what you need explicitly: `selectinload`, `joinedload` in the query, or `await session.run_sync(...)`, or `AsyncAttrs`. This is also how you avoid N+1 queries: one query for parents, one `IN` query for children.

**Connection pooling.** The pool keeps connections open and reuses them. Budget it: total connections = `(pool_size + max_overflow) x number of worker processes x number of replicas`. That total must stay below the database's `max_connections` with headroom. Too small a pool makes requests queue waiting for a connection (visible as latency spikes under load). Too large exhausts the database. Use `pool_pre_ping` to discard dead connections and `pool_recycle` where a proxy or firewall drops idle connections. For many replicas, put an external pooler (PgBouncer) in front, and check driver compatibility with transaction pooling mode (prepared statements need care; see driver docs).

**Transactions.** Decide the boundary explicitly. A common design: the service layer owns the transaction (`async with session.begin():`), the handler calls services, and repositories never commit. Commit once per unit of business work. On exception the block rolls back. Avoid holding a transaction open across slow outbound HTTP calls: it pins a connection and holds locks.

With `async with session.begin():` the block commits on success and rolls back on error, and `await session.flush()` assigns IDs without committing. If the session already autobegan a transaction, `session.begin()` raises, so be consistent about where transactions start. Confirm semantics in the docs.

**Migrations with Alembic.** Schema changes are versioned scripts. `alembic revision --autogenerate -m "message"` diffs your models against the database and drafts a migration. **Review every autogenerated migration**: it can miss renames (seen as drop plus add), constraint changes and data migrations. `alembic upgrade head` applies. Alembic ships an async template (`alembic init --template async alembic`; the `-t` short form is unverified). Run migrations as a separate deploy step, not on every app start with many replicas racing. Make them backward compatible: add a nullable column, deploy code, backfill, then tighten (expand and contract), so old and new code can run together during a rolling deploy.

**Testing databases.** Prefer a real PostgreSQL (container) over SQLite for tests, since dialect differences hide bugs (chapter 14).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| ORM and Core | SQLAlchemy 2.0 `select`, `Mapped`, `mapped_column` | Write queries and models in 2.0 style |
| Async sessions | `create_async_engine`, `async_sessionmaker`, `AsyncSession` | Session per request with a yield dependency |
| Relationship loading | `selectinload`, `joinedload`, `lazy="raise"` | Eliminate N+1 and async lazy-load errors |
| Pooling | `pool_size`, `max_overflow`, `pool_pre_ping`, PgBouncer | Compute and justify connection budget |
| Migrations | Alembic `revision --autogenerate`, `upgrade`, `downgrade` | Review, order and ship safe migrations |

### Common mistakes

- Sharing a session between concurrent tasks or storing it globally.
- Lazy loading in async code, and N+1 query patterns.
- Pool sizes multiplied by workers and replicas exceeding database limits.
- Committing in repositories, producing partial writes when a later step fails.
- Trusting autogenerate blindly, or dropping a column in the same deploy as the code change.
- Holding transactions open during outbound HTTP calls.

### Self-check

- Compute the maximum database connections for 4 workers, `pool_size=10`, `max_overflow=5`, 3 replicas.
- Why does lazy loading fail in async SQLAlchemy and what do you do instead?
- How do you add a NOT NULL column to a large table with zero downtime?

---

## 10. Background tasks versus a real queue

**In plain words.** `BackgroundTasks` runs a small job after the response, inside the same process. If the process stops, the job is lost. For work that must not be lost, use a queue and a separate worker process.

### Understand

**`BackgroundTasks`.** FastAPI's built-in `BackgroundTasks` runs a function after the response is sent, in the same process.

```python
from fastapi import BackgroundTasks

@app.post("/signup", status_code=201)
async def signup(body: UserCreate, tasks: BackgroundTasks):
    user = await create_user(body)
    tasks.add_task(send_welcome_email, user.email)
    return user
```

It is simple and has no infrastructure. It also has **no durability** (a crash or deploy loses the task), **no retries**, **no visibility**, **no rate control** and it competes with request handling for the same event loop and workers. An async task runs on the loop. A sync function runs in the threadpool.

**A real task queue** (Celery, ARQ, Dramatiq, RQ, or a cloud queue with consumers) puts the job in a broker (Redis, RabbitMQ, SQS) and a **separate worker process** executes it. You gain durability, retries with backoff, scheduling, concurrency limits, dead-letter handling, monitoring and independent scaling.

Celery is mature and feature-rich but sync-oriented and heavier to operate. ARQ is asyncio-native and Redis-only. Dramatiq has simple, reliable defaults on Redis or RabbitMQ. Verify current maintenance status of each before choosing.

**Choosing.**

- `BackgroundTasks`: fast, non-critical, loss-tolerant work (best-effort metrics, a cache warm, a notification you can afford to drop).
- Queue: anything that must happen (payments, emails that matter, webhooks), anything slow or CPU-heavy, anything needing retries, anything that must survive a deploy.

**Design rules for queued work.** Make jobs **idempotent** (retries and duplicates will happen). Pass identifiers, not large objects. Write the database change and enqueue atomically where it matters, using the **transactional outbox** pattern: write an outbox row in the same transaction and have a relay publish it, because "commit then enqueue" can lose the job if the process dies between the two. Return `202 Accepted` with a job or status URL for long operations.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| In-process | `BackgroundTasks` | Know exactly what it does not guarantee |
| Queues | Celery, ARQ, Dramatiq, RQ | Pick one with reasons |
| Reliability | idempotency keys, outbox pattern, retries with backoff | Design jobs that survive duplicates |

### Common mistakes

- Using `BackgroundTasks` for work that must not be lost.
- Non-idempotent jobs combined with at-least-once delivery.
- Enqueueing before the database commit, so the worker runs before the data exists.

### Self-check

- A deploy happens while background tasks run. What is lost, and in which design?
- Why is "commit, then enqueue" unsafe, and what pattern fixes it?
- What does "idempotent" mean for a job, and why do retries require it?

---

## 11. Files, streaming, SSE and WebSockets

**In plain words.** Large files and long responses should be sent in small pieces, not loaded fully into memory. SSE (Server-Sent Events) pushes updates from server to browser. WebSockets allow two-way messages but are harder to scale.

### Understand

**Uploads.** Declare `UploadFile` (requires `python-multipart`). It uses a spooled temporary file, so large uploads spill to disk rather than memory. Do not `await file.read()` an unbounded upload into memory. Read in chunks, enforce a maximum size (also at the reverse proxy, for example nginx `client_max_body_size`), validate the content type by inspecting content rather than trusting the client header, generate your own storage names (never use the client filename as a path), and prefer **presigned URLs** so large files go directly to object storage instead of through your API.

**Downloads and streaming.** `FileResponse` serves a file from disk. `StreamingResponse` takes a (sync or async) generator and sends chunks as produced, so you can export a large dataset without holding it in memory. Use an async generator for async sources. A sync generator is iterated in a threadpool. Streaming means errors after the first byte cannot change the status code, so design for that, and check that the client disconnects stop the generator.

**Server-sent events.** SSE is one-way server-to-client over plain HTTP using `Content-Type: text/event-stream` and `data: ...\n\n` framing. FastAPI has built-in SSE support since version 0.135.0 (confirmed in the official docs): `from fastapi.sse import EventSourceResponse, ServerSentEvent`, used with `response_class=EventSourceResponse` on an async generator endpoint. It sends keep-alive pings and sets `Cache-Control: no-cache` and `X-Accel-Buffering: no` for you. On older versions, use `StreamingResponse` with an async generator, or the `sse-starlette` library. SSE is simpler than WebSockets, reconnects automatically in browsers and passes through ordinary HTTP infrastructure. Disable proxy buffering (nginx `proxy_buffering off`, or the `X-Accel-Buffering: no` response header) or events arrive in bursts. It suits notifications, progress and token streaming from an LLM.

**WebSockets.** Two-way, long-lived. Declare `@app.websocket("/ws")`, call `await websocket.accept()`, then loop with `receive_text` and `send_text`, and handle `WebSocketDisconnect`. Dependencies work, but `Depends` for HTTP-only security schemes does not map neatly, and browsers cannot set custom headers on WebSocket handshakes, so authenticate with a short-lived ticket or token in the query string or first message. Scaling issue: connections are bound to one process, so broadcasting across workers and replicas needs a shared pub/sub layer (Redis Pub/Sub, a message broker). Track and cap connections per worker, set heartbeats and timeouts, and remember that long-lived connections interact with deploys and graceful shutdown (chapter 15).

**Choosing.** Request/response for most things. SSE for server push. WebSockets when the client genuinely sends a stream of messages back (chat, collaborative editing, games).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Uploads | `UploadFile`, `python-multipart`, presigned URLs | Accept large files safely |
| Streaming | `StreamingResponse`, `FileResponse`, async generators | Export large data with flat memory |
| Push | SSE (`sse-starlette` or manual), proxy buffering settings | Stream events through a proxy |
| Realtime | `WebSocket`, Redis pub/sub | Authenticate and scale a socket service |

### Common mistakes

- Reading whole uploads into memory, or no size limit.
- Using the client-supplied filename as a path, or trusting its `content_type`.
- Proxy buffering swallowing SSE events.

### Self-check

- How does an upload of several gigabytes avoid exhausting memory?
- Why must a WebSocket service use shared pub/sub when running several workers?
- Why does proxy buffering break SSE, and how do you turn it off?

---

## 12. OpenAPI generation and customisation

**In plain words.** FastAPI builds a machine-readable description of your API (OpenAPI) from your code. Tools use it to make documentation pages and client code. Treat it as a contract: changing names can break clients.

### Understand

**Generated contract.** FastAPI builds an OpenAPI schema from routes, parameters, Pydantic models, response models, dependencies and security schemes, and serves it at `/openapi.json` with Swagger UI at `/docs` and ReDoc at `/redoc`. The schema is a product: client SDKs, contract tests, gateways and documentation are generated from it.

**What you control.** App metadata (`title`, `version`, `description`, `servers`). Route metadata (`tags`, `summary`, docstring description, `deprecated`, `operation_id`). Alternative responses, such as `responses={404: {"model": ErrorOut}}`, so error shapes appear in the schema. Examples (the mechanism differs between versions; confirm). `openapi_extra` and overriding `app.openapi` for post-processing. Exposure: set `docs_url=None`, `redoc_url=None` and `openapi_url=None`, or protect them, when the API should not document itself publicly.

**Contract hygiene.** Generated clients derive method names from `operation_id`, so set stable IDs early (FastAPI offers a custom unique-ID function; confirm in docs). Diff `openapi.json` in CI to catch breaking changes, and version with a path prefix (`/v1`).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schema | `/openapi.json`, `app.openapi()` | Read and post-process the document |
| Metadata | tags, summaries, `responses`, `operation_id` | Produce a client-friendly schema |
| Contract checks | schema diff in CI, Schemathesis | Detect breaking changes and fuzz the API |

### Common mistakes

- Leaving undocumented error responses, so clients cannot type them.
- Exposing `/docs` on an internal admin API to the internet.
- Changing operation IDs or field names casually.

### Self-check

- How does a typed client get its method names, and what breaks if you rename a route?
- How do you make error responses (for example 404) appear in the OpenAPI schema?
- How would you stop a breaking schema change from reaching production?

---

## 13. Project layout

**In plain words.** Split code into layers: routers handle HTTP, services hold business rules, and repositories run queries. Each layer can then be tested and changed on its own. Do not add layers a small service does not need.

### Understand

**Why structure matters.** A single `main.py` works for a demo. A real service needs boundaries so that HTTP concerns, business rules and data access can change and be tested independently.

```text
app/
  main.py        # create_app(), include routers, lifespan
  core/          # settings, logging, security helpers, exceptions
  db/            # engine, session, base
  users/         # router.py, schemas.py, models.py, service.py, repository.py, deps.py
  orders/ ...
tests/
alembic/
```

Organising by **domain** (users, orders) tends to scale better than by technical type (all routers in one folder, all models in another).

**Layers and their rules.**

- **Router**: thin. Declares the contract, injects dependencies, calls a service, returns a schema. No SQL and no business decisions.
- **Service**: business logic and transaction boundaries. Raises domain exceptions. Knows nothing about HTTP.
- **Repository**: encapsulates queries. Takes a session, returns models. Does not commit.
- **Schemas** (Pydantic) versus **models** (ORM): separate on purpose.

Do not add a repository layer by reflex when a service calling SQLAlchemy directly is clear and small. Add layers when duplication or testing demands them.

**Application factory.** `create_app(settings)` builds the app, which keeps tests isolated and permits different configurations. Use `APIRouter` with `prefix`, `tags` and `dependencies`, and `include_router` to assemble. Avoid circular imports: routers import services, services import repositories, never the reverse.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Routing | `APIRouter`, `include_router`, prefixes, tags | Split an app by domain |
| Layering | routers, services, repositories | State what each layer may import |
| Static analysis | `ruff`, `mypy`, import linters | Enforce layering and typing in CI |
| Factories | `create_app()` | Build isolated app instances for tests |

### Common mistakes

- Business logic in route handlers, untestable without HTTP.
- ORM models returned directly from the API.
- Over-engineering a three-endpoint service with five layers.

### Self-check

- Where should a rule like "an order cannot be cancelled after shipping" live, and why?
- What may a repository import, and what must it never do?
- Why use an application factory (`create_app`)?

---

## 14. Testing

**In plain words.** Test most logic with fast tests, and test the API with a real database. Replace auth and outside services with dependency overrides. Also test failure cases, not only success.

### Understand

**Pyramid.** Many fast unit tests of services and validators, a solid layer of API tests through the app with a real database, a few end-to-end smoke tests. See `04_Python.md` (pytest chapter) for fixtures and parametrisation.

**Clients.**

- `fastapi.testclient.TestClient` (from Starlette, httpx-based in current versions) is synchronous. You call `client.get(...)` in an ordinary test, and it drives the ASGI app for you. Using it as a context manager (`with TestClient(app) as client:`) runs lifespan startup and shutdown. Without that, lifespan may not run.
- `httpx.AsyncClient` with `ASGITransport(app=app)` is the choice when the test itself is async (it needs to await other things, such as the database). It does **not** run lifespan by itself (confirmed in the FastAPI async-tests page). Run lifespan with `LifespanManager` from the `asgi-lifespan` package, or build the needed state in fixtures.

Build it as `httpx.AsyncClient(transport=httpx.ASGITransport(app=app), base_url="http://test")`. The FastAPI docs run async tests with the AnyIO pytest marker (`@pytest.mark.anyio`). `pytest-asyncio` is an alternative. Its default mode is `strict` (only tests marked `asyncio` run, and async fixtures need `@pytest_asyncio.fixture`). In `auto` mode it marks all async tests and fixtures for you. Set the mode with `asyncio_mode`. Fixture event-loop scope options were not confirmed here (unverified), so read the pytest-asyncio docs for your version.

**Dependency overrides.** Replace real dependencies with test doubles:

```python
@pytest.fixture
def client(fake_user):
    app.dependency_overrides[get_current_user] = lambda: fake_user
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()          # never leak between tests
```

Override auth to test business behaviour, and write separate tests that exercise the real auth path, including the failure cases.

**Database fixtures.** Prefer a real PostgreSQL started through a container (Testcontainers or Compose) and the same migrations as production (run `alembic upgrade head` once per session). Isolate tests by wrapping each in a transaction that rolls back, or by truncating tables. When the code under test commits, a rollback wrapper needs the session to use savepoints. SQLAlchemy documents this as "Joining a Session into an External Transaction": bind the session to an open connection that has begun a transaction, pass `join_transaction_mode="create_savepoint"`, and roll back the outer transaction at the end of the test (confirmed for 2.0). SQLite is faster but hides dialect, locking and type differences.

**What to test.** Validation errors (422 shapes), auth failures (401, 403, object-level 404), error contract, pagination bounds, idempotency, transaction rollback on failure, and concurrency-sensitive paths. Use `respx` or a fake to mock outbound HTTP. Contract-test your OpenAPI schema. Run `ruff`, `mypy` and tests in CI.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Sync API tests | `TestClient` | Test routes and lifespan correctly |
| Async API tests | `httpx.AsyncClient`, `ASGITransport`, `pytest-asyncio` | Test async code paths |
| Isolation | `dependency_overrides`, fixtures, Testcontainers | Real DB, fake externals |
| Quality gates | `pytest`, `ruff`, `mypy`, coverage | Run all in CI |

### Common mistakes

- Forgetting that `AsyncClient` does not run lifespan.
- Leaving `dependency_overrides` set, polluting later tests.
- Testing only against SQLite when production uses PostgreSQL.
- Only happy-path tests, no authorisation or failure tests.

### Self-check

- When do you use `TestClient` versus `AsyncClient`, and which runs lifespan by default?
- How do you give each test a clean database cheaply?
- Why test through the real auth path as well as with overridden auth?

---

## 15. Performance and deployment

**In plain words.** One Python process uses one CPU core, so you run several processes to use more cores. Put a proxy in front, shut down gracefully, and add health checks. Measure first, then tune.

### Understand

**Process model.** One Uvicorn process uses one event loop on one core. To use many cores you run **multiple worker processes**. Each worker has its own memory, its own connection pool and its own copy of any loaded model. Two ways:

- `uvicorn app.main:app --workers N`: Uvicorn's own multi-process supervisor.
- Gunicorn as process manager with Uvicorn workers: `gunicorn -k uvicorn.workers.UvicornWorker -w N app.main:app`. Gunicorn provides mature worker supervision, restarts and `--max-requests` recycling. The Uvicorn docs state that the `uvicorn.workers` module is deprecated and will be removed, and tell you to use the separate `uvicorn-worker` package instead (`pip install uvicorn-worker`). The docs page still shows the old class path in its examples and does not give the new import path, so check the `uvicorn-worker` project page for the class name (unverified here).
- In containers, many teams run **one process per container** and scale with replicas through the orchestrator, which keeps health, resource limits and restarts at one level. The FastAPI deployment docs recommend this for containers. Confirm against your platform.

**Process count versus async concurrency.** They are different axes. Async concurrency is how many I/O-bound requests one worker interleaves (thousands). Worker count is how many cores you use for CPU work (typically about the core count, or fewer if the service is I/O bound and memory constrained). Adding workers does not fix a blocked event loop in each of them. Adding async does not use more cores. Size from measurement, and remember the database connection budget multiplies by workers (chapter 9).

**Behind nginx or a load balancer.** The proxy terminates TLS, buffers slow clients, enforces limits and serves static files. Pass client information. Uvicorn enables `--proxy-headers` by default, but it trusts forwarded headers only from the addresses in `--forwarded-allow-ips` (default: `127.0.0.1` and `::1`, or the `FORWARDED_ALLOW_IPS` variable). Set it to your proxy addresses so the app sees the correct scheme and client IP. Keep it narrow, since trusting `*` lets clients spoof `X-Forwarded-For`. Configure upstream timeouts, `client_max_body_size`, and for SSE and WebSockets disable buffering and set the upgrade headers.

**Graceful shutdown.** On `SIGTERM`, Uvicorn stops accepting new connections and lets in-flight requests finish, then runs lifespan shutdown. Set `--timeout-graceful-shutdown` to bound the wait (the option exists; its default was not confirmed, unverified). Your orchestrator's termination grace period must exceed it. Keep PID 1 correct in containers: use the exec form of `CMD` so Uvicorn receives signals (see `03_Linux.md`). Long-lived connections (SSE, WebSockets) delay shutdown, so close them deliberately. Deregister from the load balancer (fail readiness) before stopping so no new traffic arrives during drain.

**Health checks.** Provide two endpoints.

- **Liveness** (`/health/live`): the process is up and the loop is responsive. Keep it trivial. Do not check dependencies, or a database blip restarts every pod.
- **Readiness** (`/health/ready`): the instance can serve now (database reachable, model loaded). Failing it removes the instance from rotation without killing it.

**Docker.** A typical image: slim Python base, install dependencies from a lockfile in an early layer for caching, copy source, run as a non-root user, `PYTHONUNBUFFERED=1`, exec-form `CMD`, a `.dockerignore`, no secrets in layers. Multi-stage builds keep build tools out of the runtime image.

The exec form is `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]`. Pin a Python version you have verified your dependencies on.

**Performance levers, in order.** (1) Fix blocking calls and N+1 queries, which dwarf framework overhead. (2) Add indexes and measure queries. (3) Reuse clients and pooled connections, set timeouts. (4) Cache (Redis, HTTP caching headers). (5) Reduce payload size (pagination, field selection, compression). (6) Choose faster serialisation (`ORJSONResponse`, or check the current docs, since default response handling has changed over versions). (7) Scale workers and replicas. Profile first (`py-spy`, `cProfile`, APM traces).

**Load testing.** Use `locust` (Python scenarios) or `k6` (JavaScript scenarios) against a production-like environment with realistic data. Watch latency percentiles (p95, p99), error rate, saturation (CPU, event-loop lag, pool wait, database), and test beyond expected peak. Include a health probe running during the test to reveal loop blocking.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Serving | `uvicorn`, `gunicorn` with Uvicorn workers | Justify workers, timeouts and shutdown settings |
| Proxying | nginx, `--proxy-headers`, `--forwarded-allow-ips` | Get correct client IP and scheme safely |
| Packaging | Docker multi-stage, non-root user, exec-form `CMD` | Build a small, safe, signal-correct image |
| Load testing | `locust`, `k6` | Produce a latency-versus-load curve and find the knee |
| Profiling | `py-spy`, `cProfile`, APM | Locate hot spots in production-like runs |
| Probes | liveness and readiness endpoints | Distinguish them and explain why |

### Common mistakes

- Setting workers from CPU count with no regard for memory or database connections.
- Liveness probes that depend on the database.
- Trusting forwarded headers from any source.
- `CMD` in shell form, so `SIGTERM` never reaches Uvicorn.
- Measuring only average latency in load tests.

### Self-check

- Why does adding workers not fix a blocking handler?
- What should the liveness probe check, and why not the database?
- How do you size workers and the database pool together?

---

## 16. Serving ML models safely

**In plain words.** Load the model once when the app starts, not on every request. Each worker process holds its own copy, so memory grows with the worker count. Run inference off the event loop so other requests keep working.

### Understand

**Load once, at startup.** Loading a model can take seconds and gigabytes. Do it in lifespan, store it on `app.state`, expose it by dependency, and make readiness fail until it is loaded. Never load per request.

In `lifespan`, assign `app.state.model = load_model(...)` before the `yield`. A blocking load at startup is acceptable.

**Memory per worker.** Every worker process loads its own copy. Four workers with a 3 GB model use about 12 GB. Options: fewer workers with more threads, preloading and relying on copy-on-write (fragile with Python reference counting), shared memory or memory-mapped weights, or a dedicated model server (Triton, TorchServe, vLLM, or a managed endpoint) that the API calls.

**Do not block the loop.** Inference is CPU or GPU bound, so calling `model.predict` directly inside `async def` freezes the worker (chapter 4). Options: a plain `def` endpoint (runs in the threadpool, fine if the library releases the GIL, as many numerical libraries do), `run_in_threadpool` or a dedicated executor with a bounded size, a process pool, or a separate inference service. Bound concurrency with a semaphore so that a burst cannot queue unbounded work and exhaust memory.

**Thread safety.** Do not assume a model object is thread-safe. Some frameworks document that inference is safe concurrently, others keep mutable state. Read the library's guidance, protect shared mutable state with a lock or give each worker its own instance, and never mutate global model state per request.

**Batching.** Accelerators are far more efficient on batches than on single items. **Dynamic batching** collects requests for a short window or until a size limit, runs one batched inference, then returns each caller its slice. Implement with an `asyncio.Queue` and a background task, or rely on an inference server that provides it. The trade-off is added latency (the window) for higher throughput. Set the window and max batch size from measurement.

**Safety and robustness.** Validate input shape, size and ranges with Pydantic (limit text length, image dimensions, batch size) to prevent resource exhaustion. Set timeouts. Return a model version in responses and logs. Never load untrusted pickles or model files: unpickling executes arbitrary code. Prefer safe formats (such as safetensors where applicable) and verify checksums. Version and roll out models with the same care as code, and keep a fast rollback. Log inputs and outputs only within your privacy rules.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Lifecycle | `lifespan`, `app.state`, readiness gating | Load once and gate traffic until ready |
| Offloading | `def` endpoints, `run_in_threadpool`, executors, semaphores | Keep the loop responsive under inference |
| Batching | `asyncio.Queue`, inference servers (Triton, vLLM and similar) | Explain dynamic batching and its trade-off |

### Common mistakes

- Loading the model inside the handler.
- Calling blocking inference in `async def`.
- Ignoring that memory scales with worker count.
- Deserialising untrusted model artefacts.

### Self-check

- Why can four workers exhaust memory when one worker did not?
- How does dynamic batching trade latency for throughput?
- Why is loading an untrusted pickle file dangerous?

---

## 17. Observability

**In plain words.** Observability means you can see what the system is doing from outside. Use logs for events, metrics for numbers over time, and traces for where time was spent. Add a request ID so one request can be followed everywhere.

### Understand

**Three signals.** Logs (what happened), metrics (how much and how often), traces (where time went across services). Design them before an incident, not during.

**Structured logging.** Emit one JSON object per line to stdout (see `04_Python.md`, logging chapter, and `03_Linux.md`, logs) so aggregators can parse and query fields. Use the standard `logging` module with a JSON formatter, or `structlog`. Include timestamp, level, message, service, version, environment, **request ID**, route, method, status, duration, and user or tenant ID where policy allows. Never log secrets, tokens, passwords or full personal data.

**Request IDs and context.** Generate or accept an ID per request in middleware (chapter 6), return it in a response header and in error bodies, and attach it to every log line using `contextvars` so code deep in services logs it without passing it around. Propagate it on outbound calls. Prefer pure ASGI middleware to avoid `BaseHTTPMiddleware` context caveats. For distributed tracing, use W3C `traceparent` propagation, which OpenTelemetry implements.

**Metrics.** Expose request count, latency histogram and error count by route template (not raw path, which explodes label cardinality) and status class. Add runtime and resource metrics: in-flight requests, event-loop lag, DB pool in-use and wait, queue depth, model inference latency. Prometheus client libraries or OpenTelemetry both work (confirm the instrumentation package for your versions). Structure alerting around the **RED** method for services (rate, errors, duration) and **USE** for resources (utilisation, saturation, errors). Alert on symptoms users feel (error rate, p99 latency) rather than every cause.

**Tracing.** OpenTelemetry has FastAPI, SQLAlchemy and httpx instrumentation (confirm for your versions). Traces show where a slow request spent its time. Send unhandled exceptions with the request ID to an error tracker.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Logging | `logging` with JSON formatter, `structlog`, `contextvars` | Correlate every line to a request |
| Metrics | Prometheus client, OpenTelemetry metrics | Expose RED and saturation metrics with bounded cardinality |
| Tracing | OpenTelemetry instrumentation | Read a trace to find the slow span |

### Common mistakes

- Unstructured logs that cannot be queried.
- High-cardinality labels (raw URLs, user IDs) in metrics.
- No request ID, so a user's error report cannot be traced.
- Logging sensitive data.

### Self-check

- Why label metrics by route template and not by URL path?
- How does a request ID get into a log line written deep inside a service?
- What do RED and USE mean, and when do you use each?

---

## 18. Security checklist

**In plain words.** FastAPI checks types, not intent, so security is mostly your job. Use this list as a review before release. The most common serious bug is letting one user read another user's data.

### Understand

Use this as a review list. Each item links back to a chapter.

**Input and output**
- Every input validated by a Pydantic model with bounds (lengths, ranges, list sizes). `extra="forbid"` on writes. (3)
- Separate output models. No ORM rows returned directly. (3)
- Pagination caps on every list. Request body size limits at the proxy and app. (2, 15)
- Uploads: size limit, content inspection, server-generated names, outside the web root. (11)

**Authentication and authorisation**
- Authentication applied at router or app level, public routes explicit (fail closed). (7)
- Object-level authorisation enforced in queries. (7)
- Passwords hashed with a modern algorithm. JWT algorithm pinned, `exp`, `iss`, `aud` checked, short lifetimes. (7)
- Rate limiting and lockout on login and expensive endpoints, at the gateway or with a library.

**Transport, data and secrets**
- TLS at the proxy, HSTS, secure cookie flags if cookies are used. CSRF protection if cookies authenticate users. (Pure bearer-header APIs are not CSRF-prone in the same way.)
- CORS allow-list of explicit origins, never `*` with credentials. (6)
- SQL through the ORM or bound parameters only. (9)
- Secrets from the environment or a secret store, never in code, images or logs. (8)
- Least-privilege database user, separate from the migration user.

**Operations**
- `/docs` and `/openapi.json` disabled or protected where the API is not public. (12)
- Debug off, generic 500 bodies, narrow trusted-proxy settings. (6, 15)
- Dependencies pinned and scanned (`pip-audit` or similar). Non-root minimal container. (15)
- SSRF: if the API fetches user-supplied URLs, allow-list hosts and block internal ranges.
- Audit logging for sensitive actions. Never unpickle or `eval` untrusted data. (16, 17)

Map your review against the OWASP API Security Top 10 (see references).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Dependency scanning | `pip-audit`, Dependabot or similar | Gate builds on known vulnerabilities |
| Dynamic testing | Schemathesis, OWASP ZAP | Fuzz the API from its own schema |

### Common mistakes

- Treating the framework as secure by default. It validates types, not intent.
- Authentication without object-level authorisation.
- Permissive CORS to silence a browser error.
- Shipping the default docs endpoints publicly with no review.

### Self-check

- Which items on this list does FastAPI enforce for you, and which are yours?
- What is broken object-level authorisation, and how do you test for it?
- What is SSRF, and how do you reduce the risk when an API fetches user-supplied URLs?

---

## 19. Choosing a framework

**In plain words.** No framework is best for every job. Pick by team skills, workload and needed features, not by simple speed tests. You can choose differently for each service.

### Understand

An honest comparison. Choose based on the team, workload and ecosystem, not benchmarks of "hello world".

| Aspect | FastAPI | Express | Fastify | Django (with DRF or Ninja) |
|---|---|---|---|---|
| Language | Python | JavaScript or TypeScript | JavaScript or TypeScript | Python |
| Style | typed, declarative, DI | minimal, middleware chain | schema-driven, plugin system | batteries included, MVC-like |
| Validation | Pydantic from type hints, built in | none built in (Joi, Zod, others) | JSON Schema built in | forms, serializers (DRF) |
| API docs | OpenAPI generated automatically | manual or add-ons | OpenAPI via plugins from schemas | DRF schema tooling, Ninja generates OpenAPI |
| Async model | asyncio, plus threadpool for `def` | event loop | event loop | sync core with growing async support (confirm) |
| ORM and admin | none included (SQLAlchemy, others) | none | none | excellent ORM, migrations, admin, auth included |
| Performance | good for I/O, bounded by Python | good | generally higher than Express | lower throughput per process, rarely the bottleneck |
| ML ecosystem | native Python: direct access to models and data libraries | needs a bridge | needs a bridge | native Python |
| Best fit | typed JSON APIs, ML-adjacent services, async I/O | simple services, large JS shops | high-throughput JS services | content-heavy apps, admin-driven products, rapid CRUD |

**Where it does not.** It ships no ORM, admin, auth system, migrations or conventions, so you assemble and own them, and large teams must enforce structure themselves (chapter 13). The ecosystem moves quickly and has a bus-factor concern around a small core maintainer group, as with many frameworks. Python's per-process throughput and GIL limit CPU-bound work: you scale with processes. Async Python has sharp edges (blocking in the loop), which Node developers know as the same hazard. Express and Fastify have the larger deployment ecosystem for JavaScript teams who share types with a front end.

**Where Django wins.** When you need an admin interface, a mature ORM with migrations, authentication, forms and a convention-rich codebase that a large team can navigate, it removes months of assembly. DRF or Django Ninja layer an API on top. Many companies run Django for the product core and FastAPI for specialised services.

**Decision heuristics.** Python team or ML work: FastAPI. Admin-heavy CRUD with a big team: Django. JavaScript team sharing types with a front end: Fastify or a TypeScript framework. Operational familiarity usually outweighs micro-differences, and database and network dominate latency, not the framework.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Evaluation | a spike with realistic endpoints, load test, team review | Decide with evidence for your context |

### Common mistakes

- Choosing by synthetic benchmark.
- Choosing FastAPI for a CRUD-plus-admin product and rebuilding Django's features by hand.
- Choosing a framework nobody on the team can operate.
- Treating the choice as permanent instead of per service.

### Self-check

- Give two workloads where FastAPI is the right choice and two where it is not.
- What does Django give you that FastAPI makes you assemble?
- Why is a "hello world" benchmark a weak reason to choose a framework?

---

## 20. Practice labs

**In plain words.** These labs make you practise the ideas by building. Try each one before you read more.

Do these with Python 3.11 or newer in a virtual environment, and Docker for the database. Each lab states the goal. Attempt before looking anything up.

1. **Typed CRUD API.** Build a resource with separate create, update and output models, `extra="forbid"`, bounded pagination and correct status codes. Show a 422 body for three kinds of bad input.
2. **Block the loop.** Write an `async def` endpoint that calls `time.sleep(3)`. Run a concurrent health request and show it stall. Fix it three ways: `def`, `asyncio.to_thread`, and an async-native call. Measure each with a small load script.
3. **Dependency graph.** Create `get_session`, `get_current_user` and `require_role` as dependencies. Show per-request caching by counting calls. Apply auth at router level and mark one route public.
4. **Lifespan resources.** Create an engine and an `httpx.AsyncClient` in lifespan, expose them via dependencies, and prove they close on `SIGTERM`.
5. **JWT login.** Implement password login, token issue and verification with a pinned algorithm and expiry. Write tests for expired, tampered and wrong-algorithm tokens.
6. **Object-level authorisation.** Add an orders resource. Write a failing test where user A reads user B's order, then fix it in the query.
7. **Async SQLAlchemy.** Model users and orders with a relationship. Trigger a `MissingGreenlet` error through lazy loading, fix it with `selectinload`, and show the query count with `echo=True`.
8. **Migrations.** Use Alembic to add a NOT NULL column to a populated table safely in three steps (nullable, backfill, constrain). Review the autogenerated script and correct it.
9. **Pool exhaustion.** Set `pool_size=2`, `max_overflow=0`, hold sessions open during a slow call, load test, and observe pool wait. Compute a correct pool budget.
10. **Queue it.** Implement the same email job with `BackgroundTasks` and with a queue worker (ARQ, Dramatiq or Celery). Kill the process mid-job and compare what survives. Make the job idempotent.
11. **Streaming.** Export one million rows through a `StreamingResponse` with flat memory, and add an SSE endpoint that works through nginx.
12. **Test suite.** Use `TestClient` and `AsyncClient`, dependency overrides, a PostgreSQL container, migrations applied once, per-test rollback.
13. **Deploy.** Containerise as non-root with exec-form `CMD`, put nginx in front, add probes, and verify graceful shutdown under load with no dropped requests.
14. **Load test.** Use `locust` or `k6` to find the saturation point for one worker, then four. Record p95 and p99 and name the bottleneck.
15. **Serve a model.** Load in lifespan, gate readiness, run inference off the loop with a bounded semaphore, then add dynamic batching.
16. **Observe and audit.** Add JSON logs with request IDs and metrics, then audit against chapter 18.

---

## 21. Interview questions

**In plain words.** These are common questions, grouped by level and tagged [Fresher], [Mid] or [Senior]. Each has a short model answer. Practise answering aloud in your own words. The answers are study aids, and no guide can promise an interview result.

### Top 10 must-know

Learn these first. Each points to its full model answer below.

1. [Fresher] What is ASGI, and how does it differ from WSGI? (Q1) Short answer: ASGI is the asynchronous interface that supports long-lived connections. WSGI is synchronous.
2. [Fresher] How does FastAPI choose path, query or body for a parameter? (Q3) Short answer: path template names are path, scalars are query, models are the JSON body.
3. [Fresher] What happens when validation fails? (Q4) Short answer: a `422` response, and the handler does not run.
4. [Fresher] What is the difference between `async def` and `def` endpoints? (Q6) Short answer: the first runs on the event loop, the second in a threadpool.
5. [Mid] What is the main pitfall of `async def`, and how do you fix it? (Q9) Short answer: blocking calls stall the whole worker, so use async libraries, `def` or `asyncio.to_thread`.
6. [Mid] How do you manage a database session per request? (Q10) Short answer: a `yield` dependency that opens and closes the session.
7. [Mid] When do you use `BackgroundTasks` and when a queue? (Q17) Short answer: `BackgroundTasks` for loss-tolerant work, a queue for anything that must complete.
8. [Mid] Why is RBAC not enough? (Q15) Short answer: you also need object-level checks in the query.
9. [Mid] How do you size workers and the connection pool? (Q18) Short answer: `(pool_size + max_overflow) x workers x replicas` must stay below `max_connections`.
10. [Senior] A service is slow but CPU is low. How do you investigate? (Q22) Short answer: look for waiting (pool, slow dependency, locks, blocked loop) using loop lag, pool metrics, traces and `py-spy`.

### Fresher

1. [Fresher] What is ASGI and how does it differ from WSGI?
2. [Fresher] What do Uvicorn, Starlette, Pydantic and FastAPI each do?
3. [Fresher] How does FastAPI decide whether a parameter is path, query or body?
4. [Fresher] What happens when request validation fails?
5. [Fresher] What is `response_model` for?
6. [Fresher] What is the difference between `async def` and `def` endpoints?
7. [Fresher] What does `Depends` do?
8. [Fresher] How do you return a `404` or `201`?

**Model answers**

**1.** ASGI is an asynchronous server-application interface using `scope`, `receive` and `send`. It supports WebSockets, long-lived connections and lifespan events. WSGI is synchronous and handles one request per worker at a time.

**2.** Uvicorn is the ASGI server handling sockets and HTTP. Starlette provides routing, middleware and request and response objects. Pydantic validates and serialises data. FastAPI connects type hints to parsing, validation, dependency injection and OpenAPI generation.

**3.** Names in the path template are path parameters. Scalar types otherwise are query parameters. A Pydantic model is the JSON body. `Header`, `Cookie`, `Form` and similar make the source explicit.

**4.** FastAPI returns `422` with a structured list of errors (location, message, type). The handler does not run.

**5.** It validates the returned data and serialises only the declared fields, so internal fields such as password hashes are not leaked, and it feeds the OpenAPI response schema.

**6.** `async def` runs on the event loop and must only await non-blocking calls. `def` runs in a threadpool, so blocking calls are safe but bounded by the pool.

**7.** It declares a dependency that FastAPI resolves and injects before the handler, including sub-dependencies and cleanup for `yield` dependencies. It shares logic such as sessions and current user and is overridable in tests.

**8.** Set `status_code=201` on the route decorator, and raise `HTTPException(status_code=404, detail=...)`.

### Mid

9. [Mid] What is the cardinal pitfall of `async def` and how do you detect and fix it?
10. [Mid] How do you manage a database session per request?
11. [Mid] Explain dependency caching within a request.
12. [Mid] How do you use lifespan, and why not `on_event`?
13. [Mid] How would you design consistent error responses?
14. [Mid] Explain the OAuth2 password flow with JWT in FastAPI and its caveats.
15. [Mid] How do you implement role-based access control, and why is it not enough?
16. [Mid] How do you test an app that uses auth and a database?
17. [Mid] When do you use `BackgroundTasks` and when a queue?
18. [Mid] How do you size Uvicorn or Gunicorn workers and the connection pool?
19. [Mid] How do you run Alembic migrations safely in deployments?
20. [Mid] Why does lazy loading fail in async SQLAlchemy?

**Model answers**

**9.** Blocking calls (sync I/O, `time.sleep`, CPU work) inside `async def` stall the event loop and every request in that worker. Detect with asyncio debug mode, loop-lag metrics, or a health probe that stalls under load, and `py-spy`. Fix with async libraries, plain `def`, `asyncio.to_thread`, a process pool, or a queue.

**10.** A `yield` dependency that opens an `AsyncSession` from an `async_sessionmaker`, injects it, and closes it on exit. The transaction boundary sits in the service layer. A session is never shared across concurrent tasks or kept global.

**11.** If several nodes in one request's dependency tree need the same dependency, it runs once and the result is reused (`use_cache=True`). The cache is per request. Disable it with `use_cache=False` when a fresh value is required.

**12.** A `lifespan` async context manager passed to `FastAPI` creates shared resources before the app serves and releases them after. It replaced the deprecated startup and shutdown events, which are ignored when `lifespan` is set. It keeps setup and teardown paired and fails startup loudly.

**13.** One envelope or RFC 9457 problem details with a stable machine code, human message and request ID. Correct status codes. Domain exceptions raised in services and mapped by exception handlers. A catch-all for unexpected errors that logs details and returns a generic 500. Document error responses in OpenAPI.

**14.** The client posts credentials as form data, the server verifies the password hash and returns a signed JWT with `sub` and `exp`. A dependency extracts the bearer token, verifies signature (pinned algorithm), expiry and claims, loads the user and raises 401 on failure. Caveats: no revocation before expiry, so use short lifetimes and refresh tokens or a denylist. Do not store sensitive data in the payload. Consider an external identity provider.

**15.** A dependency factory checks the user's role and raises 403, applied via `dependencies=[...]`. RBAC is coarse. Object-level rules (the user owns this order) must be enforced in the query, or users can read each other's data.

**16.** Override `get_current_user` or the session via `dependency_overrides` for business tests, and add separate tests through the real auth path. Use a real PostgreSQL container with migrations and a per-test rollback or truncation. Clear overrides afterwards.

**17.** `BackgroundTasks` runs after the response in the same process: simple, but not durable, no retries, shares resources. Use it for loss-tolerant quick work. Use a queue with a separate worker for anything that must complete, is slow, or needs retries, scheduling or independent scaling.

**18.** Roughly one worker per core for CPU needs, bounded by memory, or one process per container with replicas. Pool budget: `(pool_size + max_overflow) x workers x replicas` must stay below database `max_connections`. Validate with load tests and watch pool wait and CPU.

**19.** Run them as a separate pre-deploy step, not at app start across replicas. Make them backward compatible (expand and contract), review autogenerated output, take care with long locks on large tables, and have a rollback plan.

**20.** Lazy loading needs implicit I/O at attribute access, which cannot be awaited, so SQLAlchemy raises. Load eagerly with `selectinload` or `joinedload`.

### Senior and design

21. [Senior] Walk through the lifecycle of a request in FastAPI.
22. [Senior] A service gets slow under load but CPU is low. How do you investigate?
23. [Senior] Design authentication and authorisation for a multi-tenant API.
24. [Senior] How do you serve an ML model with FastAPI at scale?
25. [Senior] How do you run FastAPI in Kubernetes, including shutdown and probes?
26. [Senior] FastAPI, Django or Express for a new product? How do you decide?

**Model answers**

**21.** Uvicorn parses HTTP and calls the ASGI app. Middleware runs. The router matches a route. FastAPI resolves dependencies and parameters from the request and validates them, returning 422 on failure. The handler runs (loop or threadpool). The return value is validated and filtered by `response_model`, then serialised. Middleware unwinds, then cleanup and background tasks run.

**22.** Low CPU with high latency suggests waiting: pool exhaustion, a slow dependency, lock contention, or a blocked loop with a single busy core. Check loop lag and health probe latency, pool wait metrics, database slow queries and locks, outbound timeouts, and traces. Use `py-spy dump` to see what the workers are doing. Compare per-route latency and saturation (USE).

**23.** Authenticate through an identity provider or validated JWTs carrying tenant and subject. A dependency builds a request context (user, tenant, roles). Authorise by role at the router and by tenant and ownership in every query, ideally enforced centrally (a tenant-scoped repository or database row-level security) so a forgotten filter cannot leak data. Fail closed by default. Add audit logs and rate limits per tenant.

**24.** Load the model at startup, gate readiness on it, keep inference off the event loop with a bounded executor or separate inference server, add dynamic batching, cap input sizes and concurrency, account for memory per worker, and monitor latency and queue depth. For large models or GPUs use a dedicated serving system and let FastAPI orchestrate and validate. Never load untrusted artefacts.

**25.** One process per container with replicas, liveness endpoint that is dependency-free, readiness that checks the ability to serve, exec-form `CMD` for signals, graceful shutdown timeout below the pod grace period, a preStop or readiness failure to drain traffic, resource requests and limits sized to workers and memory, and migrations as a separate job.

**26.** Start from workload and team: ML or data-adjacent, typed JSON APIs, Python team leads to FastAPI. Admin-heavy CRUD, a large team and need for batteries leads to Django. A JavaScript team sharing types leads to Fastify or a TypeScript framework. Prototype, load test realistically, weigh operational familiarity, and prefer a choice that can be changed per service behind an OpenAPI contract.
---

## 22. References

Primary sources, preferred over blog posts. Confirm version-specific details in each. Reviewed 2026-11-06: FastAPI, Uvicorn (uvicorn.dev), Starlette (starlette.dev), SQLAlchemy and Alembic pages loaded. Pydantic pages moved to pydantic.dev (the old docs.pydantic.dev links redirect). The other links below were not re-checked and may have moved.

- FastAPI documentation (tutorial and advanced guide): https://fastapi.tiangolo.com/
- Starlette documentation: https://starlette.dev/
- Uvicorn documentation (deployment, settings): https://uvicorn.dev/
- Gunicorn documentation: https://docs.gunicorn.org/
- Pydantic documentation (including the v1 to v2 migration guide): https://pydantic.dev/docs/validation/latest/get-started/migration/
- pydantic-settings documentation: https://pydantic.dev/docs/validation/latest/concepts/pydantic_settings/
- SQLAlchemy 2.0 documentation (ORM quickstart, asyncio extension): https://docs.sqlalchemy.org/
- Alembic documentation: https://alembic.sqlalchemy.org/
- HTTPX documentation: https://www.python-httpx.org/
- pytest documentation: https://docs.pytest.org/
- pytest-asyncio documentation: https://pytest-asyncio.readthedocs.io/
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- RFC 9457, Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc9457
- OWASP API Security Top 10: https://owasp.org/API-Security/
- OpenTelemetry documentation (Python): https://opentelemetry.io/docs/languages/python/
- Locust documentation: https://docs.locust.io/
- k6 documentation: https://grafana.com/docs/k6/latest/
- Ruff documentation: https://docs.astral.sh/ruff/
- mypy documentation: https://mypy.readthedocs.io/

Verify any URL before depending on it, since pages move.

---

## 23. Glossary

| Term | Plain meaning |
|---|---|
| ASGI | Standard way for a Python web server to talk to an async app |
| WSGI | Older, synchronous version of that standard |
| Event loop | The loop in one process that switches between tasks while they wait |
| Coroutine | A function defined with `async def` that can pause at `await` |
| Blocking call | A call that holds the thread until it finishes, so nothing else runs |
| Threadpool | A fixed group of threads used to run blocking functions |
| GIL | A Python lock that lets only one thread run Python code at a time |
| Coercion | Converting a value to the declared type, such as `"5"` to `5` |
| Dependency injection | The framework calls helper functions and passes their results to your handler |
| Middleware | Code that wraps every request and response |
| Lifespan | Code that runs once at app startup and once at shutdown |
| ORM | Library that maps database rows to Python objects |
| Session | A SQLAlchemy unit of work holding objects and a database connection |
| Lazy loading | Loading related data only when it is first accessed |
| N+1 queries | One query for a list plus one extra query for each item |
| Connection pool | A set of open database connections that requests reuse |
| Migration | A versioned script that changes the database schema |
| Idempotent | Safe to run many times with the same final result |
| Outbox pattern | Save the message in the same transaction as the data, then send it later |
| JWT | A signed token that carries claims such as user ID and expiry |
| RBAC | Access control by roles |
| Object-level authorisation | Checking that this user may access this specific record |
| CORS | Browser rules about which websites may call your API |
| SSE | One-way server-to-browser event stream over HTTP |
| Graceful shutdown | Stopping after in-flight requests finish |
| Liveness / readiness | Probes: the process is alive, and it is ready to serve |
| Cardinality | Number of distinct values of a metric label |
| SSRF | Tricking a server into requesting internal or unintended addresses |
| OpenAPI | A machine-readable description of an HTTP API |
