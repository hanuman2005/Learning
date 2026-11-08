# Fastify for Backend Engineers

A working guide to building production HTTP services with Fastify 5 and TypeScript. It teaches how the framework behaves, so you can design, test and operate a service, not only copy snippets.

**Audience:** developers who know JavaScript or TypeScript and have seen Express or a similar framework.
**Prerequisites:** TypeScript basics, `async`/`await`, HTTP basics, basic SQL.
**Scope:** Fastify 5 on Node.js 20 or later, with zod, pino, Vitest and Drizzle on PostgreSQL. Front-end work is out of scope.
**Example domain:** a neutral event-ingestion service. Clients send events (for example `payment.failed`). The service stores them, groups related ones and runs background jobs.
**Accuracy status:** Reviewed against official documentation on 2026-11-06; items still unverified are marked.
**Note on accuracy:** package versions move fast. The code was checked against official pages (see References). If a name differs in your installed version, trust that package's own README and types. Items marked **(unverified)** could not be confirmed in official docs during this review.

## How to use this guide

Each chapter has five parts:

1. **In plain words**: a short summary in everyday language.
2. **Understand**: the concepts and the reasoning behind them.
3. **Tools to master**: what to learn to use fluently, and the level to reach.
4. **Common mistakes**: what goes wrong in practice.
5. **Self-check**: questions you should be able to answer without notes.

Chapter 17 is a glossary of hard terms. Chapter 18 lists hands-on labs. Chapter 19 collects interview questions with model answers. A hard term is explained in brackets the first time it appears.

**What "good enough" means for a backend role:** you can build a validated, tested API; split it into modules; run it safely in production; and explain your choices. Chapters 3, 4, 7, 9, 12 and 14 matter most.

## Contents

1. Fastify and Express
2. Server basics and handlers
3. Lifecycle and hooks
4. Routes, schemas and zod validation
5. Serialisation
6. Error handling
7. Plugins and encapsulation
8. fastify-plugin and decorators
9. Modular monolith structure
10. Config validation
11. Logging with pino and redaction
12. Graceful shutdown and health checks
13. Testing with inject and Vitest
14. PostgreSQL with Drizzle on a pooled connection
15. Security plugins
16. Cross-cutting mistakes to avoid
17. Glossary
18. Practice labs
19. Interview questions
20. References

---

## 1. Fastify and Express

**In plain words.** Fastify is a Node.js web framework. It is built around plugins and schemas. A schema is a description of the data a route accepts and returns. If you know Express, the main change is how you organise code and where you check data.

### Understand

| Express | Fastify |
|---|---|
| `res.json(x)` | `return x` from an async handler |
| `next()` and middleware order | hooks at named lifecycle points |
| Async errors need a wrapper (in Express 4) | errors thrown in async handlers are caught and passed to the error handler |
| Validation through add-on libraries | schema on the route, built in |
| Global middleware | plugins with their own scope (encapsulation) |
| `console.log` or morgan | built-in pino logger with a request id |

The mental shift: **everything is a plugin**, and **the schema is part of the route**.

**Why people choose Fastify.** It has a low-overhead core. A response schema lets it use a fast serialiser (a function that turns an object into JSON text). Plugin scopes keep modules apart. Built-in structured logging (logs as JSON objects) suits production.

**Fastify 5 changes that affect you** (from the official migration guide):
- It requires Node.js 20 or later.
- `querystring`, `params`, `body` and `response` schemas must be full JSON Schema objects with an explicit `type`. The `jsonShortHand` option was removed. Zod schemas through a type provider are compiled by the provider, so this mostly matters for hand-written schemas.
- `.listen()` takes an options object, for example `{ port: 3000 }`.
- `reply.getResponseTime()` was removed. Use `reply.elapsedTime`.
- The `logger` option no longer accepts a logger instance. Use `loggerInstance`.
- A plugin cannot mix callback and async styles. Pick one.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Install | `npm i fastify fastify-plugin zod fastify-type-provider-zod` | Start a project from scratch |
| Official docs | fastify.dev Reference section | Find a hook, option or error code in minutes |
| Migration | Migration Guide V5 | List the breaking changes from Express habits |

### Common mistakes

- Porting Express middleware order directly. Fastify uses hooks and scopes, not one global chain.
- Calling `res.send` style APIs from memory. In Fastify the reply is `reply`, and the usual style is to return a value.
- Following v4 tutorials and using removed v5 options.

### Self-check

- Name three differences between Express middleware and Fastify hooks and plugins.
- Why does a response schema make serialisation faster?
- Which Fastify 5 change breaks `app.listen(3000, "0.0.0.0")`?

---

## 2. Server basics and handlers

**In plain words.** A route is a URL plus a function. The function gets a request and returns the answer. In Fastify, you usually return the answer instead of calling a send function.

### Understand

```ts
import Fastify from "fastify";

const app = Fastify({ logger: true });

app.get("/ping", async () => {
  return { status: "ok" };          // returned value becomes the JSON body, status 200
});

app.get<{ Params: { key: string } }>("/events/:key", async (request) => {
  return { key: request.params.key, requestId: request.id };
});

app.post("/echo", async (request, reply) => {
  reply.code(201);
  return { youSent: request.body };
});

await app.listen({ port: 3000, host: "0.0.0.0" });
```

- The default `host` is `localhost`. Inside a container (Docker or similar) use `0.0.0.0` or `::`, or the port is unreachable from outside.
- Handler styles: `async` returning a value (preferred), or `reply.send(x)`. Do not mix them. If you call `reply.send` in a callback inside an async handler, you must `return reply` (or await it) so Fastify knows the reply is handled.
- `request.id` is a per-request id (a value that identifies one request). It is already in every log line made with `request.log`.
- Logging is off by default. Turn it on with `logger: true` or a pino options object.
- `reply.code(n)` sets the status. `reply.status(n)` is an alias. The default is 200.
- To fail with a given status, throw an error that has `statusCode` and `message`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Routes | `app.get/post/put/delete`, `app.route` | Write all five verbs with params, query and body |
| Reply | `reply.code`, `reply.header`, `reply.send`, `reply.redirect` | Know when `return` is enough |
| Start-up | `app.listen`, `app.ready` | Explain what each one triggers |

### Common mistakes

- Using `host: "localhost"` in a container and wondering why the port is closed.
- Mixing `return value` and `reply.send(value)` in one handler.
- Using the removed `app.listen(3000, host, cb)` style.

### Self-check

- What status does a handler return if you only return an object?
- When must an async handler `return reply`?
- Why does a service in Docker need `0.0.0.0`?

---

## 3. Lifecycle and hooks

**In plain words.** Every request passes through fixed steps in a fixed order. A hook is a function you attach to one step. This lets you add checks and timing without changing each route.

### Understand

Request order (from the official Lifecycle page):

`onRequest` then `preParsing` then body parsing then `preValidation` then validation then `preHandler` then **your handler** then `preSerialization` then `onSend` then the response is sent then `onResponse`.

Other hooks: `onError` runs when an error is thrown. `onTimeout` and `onRequestAbort` cover timeouts and clients that disconnect.

**Application hooks** are tied to the server, not a request: `onReady` (before listening), `onListen`, `preClose`, `onClose` (when `close()` runs), `onRoute` and `onRegister`.

```ts
app.addHook("onRequest", async (request) => {
  request.log.info({ url: request.url }, "incoming");
});

app.addHook("onResponse", async (request, reply) => {
  request.log.info({ ms: reply.elapsedTime, status: reply.statusCode }, "done");
});
```

**Where things usually go:**
- Authentication (an API key check): `onRequest` or `preHandler`.
- Timing and metrics: `onResponse`.
- Setting a tenant id for row-level security: `preHandler`.
- Reading the parsed body: `preHandler` (the body is not parsed in `onRequest`).

**Rules:**
- With `async` hooks, do not use the `done` callback. Using it can run things twice.
- All hooks are encapsulated except `onClose`. A hook added on the root applies to every route. A hook added inside a plugin applies only inside that plugin (chapter 7).
- To stop a request early from a hook, send a reply (`reply.code(401).send(...)`) and return `reply`, or throw an error.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Request hooks | `addHook("onRequest" ...)`, `preHandler`, `onSend`, `onResponse` | Place each concern at the right step |
| App hooks | `onReady`, `onClose` | Open and close resources correctly |
| Timing | `reply.elapsedTime` | Log request duration |
| Auth | a hook that checks a header | Block a request before the handler runs |

### Common mistakes

- Reading `request.body` in `onRequest`. It is not parsed yet.
- Adding `done` to an `async` hook.
- Putting a hook on the root when it belongs to one module.
- Doing slow work in `onSend`, which delays every response.

### Self-check

- In what order do `preValidation`, `preHandler` and `onSend` run?
- Which hook is not encapsulated?
- Where would you set a per-request database tenant, and why?

---

## 4. Routes, schemas and zod validation

**In plain words.** A schema says what a request must look like. Fastify checks the request against it before your code runs. With a type provider, one zod schema also gives you TypeScript types, so you write the rule once.

### Understand

Fastify validates `body`, `querystring`, `params` and `headers` (request parts) against schemas. By default it uses Ajv (a JSON Schema checker). A failed check returns status 400 with code `FST_ERR_VALIDATION`.

A **type provider** infers TypeScript types from the schema. Without one you keep a JSON Schema and a TypeScript type in sync by hand. Two zod packages exist:

- `fastify-type-provider-zod` (the package this guide uses; its README was checked).
- `@fastify/type-provider-zod`, listed on the official Type Providers page as a fork of the first. Its API was **(unverified)** here. Check its README before switching.

README facts for `fastify-type-provider-zod`:
- Versions up to 4.x work with zod 3. Versions 5.x and 6.x work with zod 4. Version 7 and later needs zod 4.2 or newer. From version 7, response serialisation uses `z.output<T>` instead of `z.input<T>`.
- Exports include `validatorCompiler`, `serializerCompiler`, `createSerializerCompiler`, `ZodTypeProvider`, `FastifyPluginAsyncZod`, `hasZodFastifySchemaValidationErrors`, `isResponseSerializationError`, `jsonSchemaTransform` and `createJsonSchemaTransform`.
- The README imports zod from `zod/v4`. With a zod 4 install, the main `zod` import also gives v4. **(unverified)** that both paths behave the same in your version. Use the path your installed README shows.

```ts
import Fastify from "fastify";
import {
  serializerCompiler,
  validatorCompiler,
  type ZodTypeProvider,
} from "fastify-type-provider-zod";
import { z } from "zod";

const app = Fastify({ logger: true }).withTypeProvider<ZodTypeProvider>();

app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);

app.post(
  "/v1/events",
  {
    schema: {
      body: z.object({
        idempotencyKey: z.string().min(1),
        eventType: z.string().regex(/^[a-z_]+\.[a-z_]+$/),
        correlationKey: z.string().min(1),
        occurredAt: z.iso.datetime(),
        payload: z.record(z.string(), z.unknown()).default({}),
      }),
      response: {
        202: z.object({ accepted: z.literal(true), duplicate: z.boolean() }),
      },
    },
  },
  async (request, reply) => {
    request.body.eventType; // typed as string and already validated
    return reply.code(202).send({ accepted: true, duplicate: false });
  },
);
```

Query strings arrive as text, so convert them with `z.coerce`:

```ts
schema: {
  querystring: z.object({
    severity: z.enum(["low", "medium", "high"]).optional(),
    limit: z.coerce.number().int().min(1).max(100).default(20),
  }),
  params: z.object({ id: z.uuid() }),
}
```

**Idempotency.** An operation is idempotent (safe to repeat) if repeating it gives the same result. For ingestion, store an `idempotencyKey` with a unique constraint. A repeat returns the same 202 and marks `duplicate: true`. Client retries then do no harm.

**Type providers do not carry across scopes.** Inside each plugin, call `app.withTypeProvider<ZodTypeProvider>()` again, or your routes lose their types.

**OpenAPI.** Register `@fastify/swagger` with `transform: jsonSchemaTransform` to generate API docs from the same zod schemas.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Schemas | zod objects, `z.coerce`, `z.enum`, `.default` | Model body, query, params and headers |
| Provider | `withTypeProvider<ZodTypeProvider>()`, both compilers | Get typed `request.body` |
| Docs | `@fastify/swagger` with `jsonSchemaTransform` | Generate OpenAPI from schemas |
| Limits | `bodyLimit` route or server option | Set a size limit on ingestion |

### Common mistakes

- Forgetting `setValidatorCompiler` or `setSerializerCompiler`. Then zod schemas are not used correctly.
- Mixing a zod package version with the wrong zod major version (see the table above).
- Validating query numbers without `z.coerce`. They stay strings.
- Trusting the client for fields like tenant id. Take those from auth, not from the body.

### Self-check

- What does a type provider remove from your workflow?
- A query `?limit=10` fails validation as a number. What is missing?
- Why must you call `withTypeProvider` again inside a plugin?

---

## 5. Serialisation

**In plain words.** Serialisation means turning your data into the text that is sent back. A response schema tells Fastify which fields to send. Fields not in the schema are left out, which helps stop accidental leaks.

### Understand

When a route has a `response` schema, Fastify serialises with a schema-based serialiser (`fast-json-stringify`). The official docs say this "can drastically increase throughput and help prevent accidental disclosure of sensitive information". With the zod provider, `serializerCompiler` does the work through zod.

- Schemas are chosen by status code: `response: { 200: ..., 404: ... }`.
- Output properties that are not in the schema are not sent. This is why a missing field in a response often means a missing schema field.
- Keep **internal** database columns out of response schemas (password hashes, internal ids).
- A response that does not match its schema raises a serialisation error. With this provider, `isResponseSerializationError(err)` detects it and the issues are on `err.cause.issues`. Treat this as a server bug (status 500), not a client error.
- A response schema can expose fields like `stack` if you declare them. Do not declare them.
- With provider version 7 and later, the shape is checked with `z.output`, so transforms and defaults in zod affect what is sent.

```ts
const EventOut = z.object({
  id: z.uuid(),
  eventType: z.string(),
  receivedAt: z.iso.datetime(),
});

app.get("/v1/events/:id", { schema: { params: z.object({ id: z.uuid() }), response: { 200: EventOut } } },
  async (request) => repo.findEvent(request.params.id));   // extra columns are not sent
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Response schemas | `schema.response[status]` | Define one per status code you return |
| Serialiser | `serializerCompiler`, `createSerializerCompiler` | Know it runs after your handler |
| Failures | `isResponseSerializationError` | Map to a 500 and log details |

### Common mistakes

- Forgetting that unknown fields are removed, then debugging "missing" data.
- Declaring a schema for 200 only, then returning 201 with no schema.
- Putting database rows straight into responses without a response schema.

### Self-check

- How does a response schema reduce the risk of leaking data?
- A field is in your database row but not in the response. Where do you look first?
- Is a serialisation failure a 4xx or a 5xx, and why?

---

## 6. Error handling

**In plain words.** Throw an error when something is wrong. One central function decides what the client sees. Clients should never see internal error text for server faults.

### Understand

The default handler returns JSON with `statusCode`, `error` and `message` (and `code` if present). For a 500 it uses the thrown error's message, which can leak library or database details. So write your own handler with `setErrorHandler`.

```ts
import { hasZodFastifySchemaValidationErrors, isResponseSerializationError } from "fastify-type-provider-zod";

app.setErrorHandler((error, request, reply) => {
  if (hasZodFastifySchemaValidationErrors(error)) {
    return reply.code(400).send({
      error: "validation_failed",
      issues: error.validation.map((v) => ({ path: v.instancePath, message: v.message })),
    });
  }
  if (isResponseSerializationError(error)) {
    request.log.error({ err: error }, "response did not match schema");
    return reply.code(500).send({ error: "internal_error", requestId: request.id });
  }
  const status = (error as { statusCode?: number }).statusCode ?? 500;
  if (status >= 500) {
    request.log.error({ err: error }, "request failed");
    return reply.code(status).send({ error: "internal_error", requestId: request.id });
  }
  return reply.code(status).send({ error: (error as Error).message, requestId: request.id });
});
```

Facts from the official Errors page:
- A custom handler does not set the status for you. Set it yourself.
- Error handlers are encapsulated. A handler inside a plugin covers only that plugin. The nearest one wins.
- If a plugin handler throws, the new error goes to the parent handler. Throw `new Error(...)`, not a string, or the parent is skipped.
- A status below 500 does not prove the message is safe to show. Narrow that branch if your errors can carry internal text.
- Fastify has its own error codes (for example `FST_ERR_VALIDATION`, `FST_ERR_CTP_BODY_TOO_LARGE`). They are available through `errorCodes`.

**Rules:**
- Never send internal text for 5xx. Log it. Return a generic message plus the request id.
- Expected business outcomes (a duplicate event, not found) are normal responses, not thrown errors.
- A duplicate ingest returns the same success response, so client retries are safe.
- Use one error body shape across the API.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Central handler | `setErrorHandler`, `error.validation` | Return one shape for every failure |
| Codes | `errorCodes`, `error.code`, `error.statusCode` | React to specific Fastify errors |
| Not found | `setNotFoundHandler` | Return JSON for unknown routes |
| Hook | `onError` | Know it runs once, for the first error |

### Common mistakes

- Sending `error.message` for a 500.
- Forgetting `reply.code(...)` in a custom handler, which sends a wrong status.
- Throwing a string from a plugin handler.
- Using exceptions for normal outcomes such as "duplicate".

### Self-check

- Why is the default 500 response risky?
- What happens if an error handler inside a plugin throws?
- How do you tell a validation error from another error in your handler?

---

## 7. Plugins and encapsulation

**In plain words.** A plugin is a function that adds routes and features to the app. Each plugin gets its own private area. What it adds is not visible to its neighbours. This keeps modules separate.

### Understand

```ts
import type { FastifyPluginAsync } from "fastify";

export const healthModule: FastifyPluginAsync = async (app) => {
  app.get("/health", async () => ({ status: "ok" }));
};

app.register(healthModule, { prefix: "/v1" });   // route becomes /v1/health
```

**Encapsulation** (an isolated scope per plugin). By default `register` creates a new scope. Facts from the official Encapsulation page:
- Children inherit decorators, hooks and plugins from their ancestors.
- Inheritance only flows down. A parent cannot see what a child adds.
- Siblings do not share anything. A hook or auth plugin in one sibling does not protect the other.
- Settings on the root (validator, serialiser, error handler) are inherited by children.

The plugin tree is a directed acyclic graph (a tree-like structure with no loops).

**Rules for plugins:**
- Use the `async` style without `done`. Do not mix the two styles (a v5 rule).
- `await app.register(...)` is optional. Fastify loads plugins in order when you call `listen`, `ready` or `inject`.
- Pass dependencies (database, bus, config) through plugin options. Do not import globals.
- `prefix` adds a path start to all routes in the plugin.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Register | `app.register(plugin, opts)`, `prefix` | Nest plugins and prefixes |
| Types | `FastifyPluginAsync<Opts>` | Type plugin options |
| Scope | hooks inside a plugin vs on root | Predict which routes a hook affects |
| Debug | `app.printPlugins()`, `app.printRoutes()` | Inspect the plugin tree and routes |

### Common mistakes

- Expecting a decorator added in one plugin to appear in a sibling.
- Adding the error handler inside one module when you meant it for the whole app.
- Mixing callback and async plugin styles.
- Losing types inside a plugin because `withTypeProvider` was not called again.

### Self-check

- A hook in plugin A: does it run for routes in plugin B? Why?
- What does `prefix` do?
- What do children inherit from the root?

---

## 8. fastify-plugin and decorators

**In plain words.** Sometimes you want to share something on purpose, such as a database client. A wrapper called `fastify-plugin` removes the private area for that one plugin. A decorator is a named value you attach to the app, a request or a reply.

### Understand

**`fastify-plugin` (`fp`).** Wrapping a plugin with `fp` stops Fastify from creating a new scope. Its decorators and hooks reach the parent scope and siblings. Facts from the official docs and README:
- The Plugins page recommends this for shared utilities.
- A wrapped plugin ignores `prefix`, `logLevel` and `logSerializers`.
- Options include `name` (checked for collisions), `fastify` (a version range such as `5.x`), `decorators`, `dependencies` and `encapsulate`.
- Plugins registered inside a wrapped plugin, without `fp`, still get their own scope.

**Decorators** (from the Decorators page):
- `app.decorate(name, value)` adds to the server instance. `decorateRequest` and `decorateReply` add to each request or reply.
- Reference types (objects) are not allowed as initial values in `decorateRequest` and `decorateReply`. They would be shared between requests. Use `null` or `""` and set the value per request in an `onRequest` hook.
- Defining the same name twice in one scope throws.
- `hasDecorator`, `hasRequestDecorator` and `hasReplyDecorator` check whether a name exists.
- Arrow functions do not bind `this` to Fastify.

```ts
import fp from "fastify-plugin";

declare module "fastify" {
  interface FastifyInstance {
    db: DbClient;          // type augmentation so app.db is typed everywhere
  }
}

export const dbPlugin = fp(async (app) => {
  const db = await connect();
  app.decorate("db", db);
  app.addHook("onClose", async () => { await db.close(); });
}, { name: "db" });
```

The `declare module` block adds the type to Fastify's interface. The Decorators page links to the TypeScript docs for this, and does not show it, so check the TypeScript reference for your version.

**Use `fp` sparingly.** Encapsulation is the default for modules. Wrap only cross-cutting services (database client, config, auth).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Share | `fp(plugin, { name })` | Share one client with all siblings |
| Decorate | `decorate`, `decorateRequest`, `decorateReply` | Add typed values safely |
| Types | `declare module "fastify"` | Make `app.db` typed |
| Order | `dependencies` option | Declare "needs plugin X first" |

### Common mistakes

- Using `fp` on every module. This removes isolation.
- Passing an object to `decorateRequest`. It throws at start-up.
- Forgetting the type augmentation, then casting to `any`.
- Expecting `prefix` to work on an `fp` plugin.

### Self-check

- When is `fp` correct and when is it harmful?
- Why can `decorateRequest("user", {})` not be used?
- What does `declare module "fastify"` do?

---

## 9. Modular monolith structure

**In plain words.** A modular monolith is one program made of clear parts. Each part owns its data and exposes a small public face. Parts talk through events or exported functions. Later you can move one part to its own service with small changes.

### Understand

One deployable app. Many modules. Each module is a Fastify plugin.

```
src/
  server.ts            process entry for the API: build app, listen, handle signals
  worker.ts            process entry for background jobs
  app.ts               composition root: builds the app and registers modules
  config.ts            env parsing and validation (fail fast)
  shared/
    event-bus.ts       typed in-process bus
    domain-events.ts   event types that cross module boundaries
  modules/
    ingestion/
      index.ts         PUBLIC API of the module (the only file others may import)
      routes.ts        HTTP only: parse, call service, reply
      service.ts       business rules
      repo.ts          database access only
      schemas.ts       zod schemas
    incidents/
    notifications/
    reports/
```

**Rules:**
1. A module exposes one public file, `index.ts`. Other modules import only from it.
2. A module owns its tables. No module queries another module's tables.
3. Modules talk through events on the bus, or through exported functions.
4. `routes.ts` calls `service.ts`. `service.ts` calls `repo.ts`. Never skip a layer.
5. Dependencies (db, bus, config) come in through plugin options.
6. Only `app.ts` knows about every module.

**Composition root** (the one place where parts are assembled):

```ts
// app.ts
export function buildApp(deps: { db: Db; bus: EventBus<DomainEvents> }) {
  const app = Fastify({ logger: true }).withTypeProvider<ZodTypeProvider>();
  app.setValidatorCompiler(validatorCompiler);
  app.setSerializerCompiler(serializerCompiler);
  app.setErrorHandler(errorHandler);

  app.register(ingestionModule, { prefix: "/v1", ...deps });
  app.register(incidentsModule, { prefix: "/v1", ...deps });
  return app;   // build and listen are separate: tests use buildApp without a port
}
```

```ts
// modules/ingestion/index.ts
export const ingestionModule: FastifyPluginAsync<{ db: Db; bus: EventBus<DomainEvents> }> =
  async (app, { db, bus }) => {
    const service = createIngestionService(createEventRepo(db), bus);
    registerIngestionRoutes(app.withTypeProvider<ZodTypeProvider>(), service);
  };
```

**Typed in-process event bus:**

```ts
export type EventBus<M extends Record<string, unknown>> = {
  on<K extends keyof M>(name: K, handler: (payload: M[K]) => void | Promise<void>): void;
  emit<K extends keyof M>(name: K, payload: M[K]): Promise<void>;
};
```

`emit("event.ingested", wrongShape)` then fails to compile. Decide what happens when a handler throws: log it and continue, so one listener cannot break the request.

**Limit of an in-process bus.** If the process crashes after the database write but before the event is handled, the event is lost. For work that must not be lost, store a row in the database and let the worker read it. This is the **outbox pattern** (write the event to a table in the same transaction as the data). Use the bus only for non-critical reactions.

**Two entry points, one codebase.** An HTTP server (`api`) and a background process (`worker`) import the same modules. Each has its own entry file (`server.ts`, `worker.ts`). You can scale and restart them separately. The worker uses `SKIP LOCKED` claiming (chapter 14).

**Why this eases a later split.** A module that talks only through events and its own tables can move to its own process. You replace the in-process bus with a queue and give it its own database. Callers do not change.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Boundaries | one `index.ts` per module, lint rule for imports | Enforce "import only from index" |
| Composition | `buildApp(deps)` | Build the app with fake dependencies in tests |
| Events | typed bus, outbox table | Choose bus vs outbox by "can we lose it" |
| Processes | separate `server.ts` and `worker.ts` | Run and scale them apart |

### Common mistakes

- One module reading another module's tables "just this once".
- Calling `app.listen` inside `buildApp`.
- Using the in-process bus for work that must survive a crash.
- Importing a module's internal files from outside.
- Global singletons (a shared `db` import) that make tests hard.

### Self-check

- Why do `build` and `listen` live in different files?
- What is lost if the process dies between a write and an event handler?
- Name the two entry points and why they are separate.

---

## 10. Config validation

**In plain words.** Configuration comes from environment variables, which are text. Check them once when the program starts. If something is wrong, stop immediately with a clear message.

### Understand

```ts
import { z } from "zod";

const ConfigSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().default(3000),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
  DATABASE_URL: z.url(),          // pooled URL for the app
  DATABASE_URL_DIRECT: z.url(),   // direct URL for migrations and session features
});

export const config = ConfigSchema.parse(process.env);   // throws with a clear message
```

- `z.url()` is the zod 4 form. In zod 3 use `z.string().url()`.
- Parse once in `config.ts`, then pass values down. Do not read `process.env` all over the code.
- Never log secrets or a full `DATABASE_URL` (it contains a password).
- Use `.default(...)` for safe defaults only. Do not default secrets.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Parsing | zod schema, `z.coerce` | Turn env text into typed values |
| Failing fast | `parse` at start-up | Crash with a readable error |
| Files | `.env` for local use, real env in production | Keep secrets out of git |

### Common mistakes

- Reading `process.env.X` inside business code.
- Forgetting `z.coerce` for numbers. `PORT` stays a string.
- Printing the whole config object to logs.

### Self-check

- Why parse config at start-up instead of on first use?
- What is the risk of logging `DATABASE_URL`?
- Why keep two database URLs in config?

---

## 11. Logging with pino and redaction

**In plain words.** Fastify writes logs as JSON with pino. Each line carries a request id, so you can follow one request. Redaction hides secret values before they are written.

### Understand

- Enable with `Fastify({ logger: true })` or `logger: { level: "info", ... }`. If the logger is off, it cannot be enabled later.
- Use `request.log` inside handlers. It includes the request id. Use `app.log` outside requests.
- `req`, `res` and `err` use standard serialisers by default. Custom serialisers must never throw.
- The body is not parsed when the `req` serialiser runs. Log `request.body` yourself in `preHandler` if needed, and be careful with secrets.
- Request ids: by default Fastify makes an incrementing number. Set `genReqId` for random ids. If you enable `requestIdHeader`, callers can set any id and there is no validation, so do not trust it for security.
- Put structured data first, message second: `request.log.info({ eventId }, "stored event")`.

**Redaction** uses pino's `redact` option (built on `fast-redact`):

```ts
Fastify({
  logger: {
    level: "info",
    redact: {
      paths: ["req.headers.authorization", "req.headers.cookie", "body.password", "body.apiKey"],
      censor: "[Redacted]",   // default; use remove: true to delete the key
    },
  },
});
```

Facts from the pino docs:
- Paths use JavaScript property syntax, are case sensitive, and may use `*` as a wildcard. Hyphenated keys need brackets: `req.headers["x-api-key"]`.
- Wildcards cost more CPU than explicit paths, so prefer explicit paths.
- Set `redact` at start-up. Never build paths from user input.
- Whether a path may start with a wildcard is **(unverified)** in the docs read. Use explicit paths to be safe.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Logger | `logger` option, `request.log`, `app.log` | Log with context objects |
| Redaction | `redact.paths`, `censor`, `remove` | Prove secrets never reach logs |
| Ids | `genReqId`, `requestIdHeader` | Follow a request across services |
| Reading | `pino-pretty` in development only | Read JSON logs easily |

### Common mistakes

- Logging full request headers or bodies with secrets.
- Using `console.log`. It has no request id and no structure.
- Using `pino-pretty` in production. It slows output and removes JSON.
- Trusting an incoming request-id header without checks.

### Self-check

- Why log objects instead of building strings?
- How do you keep the `authorization` header out of logs?
- Why are wildcard paths costly?

---

## 12. Graceful shutdown and health checks

**In plain words.** When a platform stops your program, it sends a signal. A good service stops taking new requests, finishes current ones, closes its connections and then exits. Health checks let the platform know if the service is alive and ready.

### Understand

```ts
for (const signal of ["SIGINT", "SIGTERM"] as const) {
  process.on(signal, async () => {
    app.log.info({ signal }, "shutting down");
    try {
      await app.close();      // stops listening, drains requests, runs onClose hooks
      process.exit(0);
    } catch (err) {
      app.log.error({ err }, "shutdown failed");
      process.exit(1);
    }
  });
}
```

Facts from the Server docs:
- `close()` runs `onClose` hooks after the server stops listening and connections drain. Use it to close database pools.
- `return503OnClosing` defaults to `true`. New requests during close get 503 with `Connection: close`.
- `forceCloseConnections` defaults to `"idle"`: idle keep-alive connections are closed so shutdown does not hang.
- A platform usually waits a fixed time after `SIGTERM`, then sends `SIGKILL`. Finish within that time.
- The worker needs the same logic: stop claiming new jobs, finish the current job, then close the pool.

**Health checks:**
- **Liveness** `/health/live`: returns 200 if the process runs. Do not check the database here. A database outage should not make the platform restart a healthy process.
- **Readiness** `/health/ready`: checks the database (for example `select 1`) and returns 503 if it fails. The platform stops sending traffic until it recovers.

```ts
app.get("/health/live", async () => ({ status: "ok" }));
app.get("/health/ready", async (_req, reply) => {
  try {
    await db.execute(sql`select 1`);
    return { status: "ready" };
  } catch {
    return reply.code(503).send({ status: "unavailable" });
  }
});
```

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Signals | `process.on("SIGTERM")` | Handle stop signals once |
| Close | `app.close()`, `onClose` | Release every resource |
| Probes | live and ready routes | Explain why they differ |
| Timeouts | platform grace period | Keep shutdown shorter than it |

### Common mistakes

- Calling `process.exit()` straight away, which drops in-flight requests.
- Checking the database in the liveness probe.
- Forgetting to close the database pool, so the process hangs.
- Registering signal handlers more than once.

### Self-check

- What does `app.close()` do in order?
- Why should liveness not call the database?
- What happens to a request that arrives during shutdown?

---

## 13. Testing with inject and Vitest

**In plain words.** `inject` sends a fake request straight into the app, with no network port. Tests are fast and need no free port. You test the whole route, including validation.

### Understand

`app.inject` uses `light-my-request`. It waits for plugins to load, so the app is ready when it runs. It accepts `method`, `url`, `query`, `payload`, `headers` and `cookies`.

```ts
import { describe, it, expect, afterEach } from "vitest";
import { buildApp } from "./app.js";

describe("ingestion", () => {
  let app: ReturnType<typeof buildApp>;
  afterEach(async () => { await app.close(); });

  it("accepts a valid event", async () => {
    app = buildApp(testDeps());
    const res = await app.inject({
      method: "POST",
      url: "/v1/events",
      payload: {
        idempotencyKey: "k1",
        eventType: "sms.sent",
        correlationKey: "m1",
        occurredAt: "2026-11-06T10:00:00Z",
      },
    });
    expect(res.statusCode).toBe(202);
  });

  it("rejects a bad body with 400", async () => {
    app = buildApp(testDeps());
    const res = await app.inject({ method: "POST", url: "/v1/events", payload: { eventType: "BAD" } });
    expect(res.statusCode).toBe(400);
  });
});
```

- Separate `build` from `listen`. Tests call `buildApp(fakeDeps)`. The official Testing guide recommends this.
- Always call `app.close()` so connections are released.
- Use `res.json()` to read a JSON body.
- For database logic, test against a real PostgreSQL (a container or a test branch), not a mock. `SKIP LOCKED` and transaction behaviour only show up on a real database.
- Unit-test services with fake repos. Integration-test routes with `inject` and a real database.
- Test the unhappy paths: bad input, duplicates, unauthorised, a failing dependency.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Fake requests | `app.inject`, `res.statusCode`, `res.json()` | Test any route without a port |
| Runner | Vitest `describe`, `it`, `expect`, `afterEach` | Structure suites with clean-up |
| Dependencies | `buildApp(deps)` | Swap real parts for fakes |
| Real DB | container or test branch, migrations before tests | Test concurrency for real |

### Common mistakes

- Starting a real server on a port in unit tests.
- Forgetting `app.close()`, so tests hang or leak.
- Mocking the database, then missing locking bugs.
- Sharing one database state between tests with no clean-up.

### Self-check

- Why use `inject` instead of an HTTP client?
- What must `buildApp` not do, so tests work?
- Which behaviours need a real database in tests?

---

## 14. PostgreSQL with Drizzle on a pooled connection

**In plain words.** Serverless-style databases put a pooler in front of PostgreSQL. A pooler shares a few real connections among many clients. It hands a connection back after each transaction, so some session features stop working. Use the pooled URL for the app and the direct URL for migrations.

### Understand

```bash
npm i drizzle-orm postgres
npm i -D drizzle-kit
```

**Pooled vs direct.** Facts from the Neon connection pooling docs (one example of this style of provider):
- The pooler is PgBouncer in **transaction mode**. A pooled host name includes `-pooler`.
- Use **pooled** for web apps and serverless functions. Use **direct** for migrations, `pg_dump`/`pg_restore`, logical replication and admin tasks.
- In transaction mode, session state does not persist. Not supported: plain `SET`/`RESET`, `LISTEN`/`NOTIFY`, session-level advisory locks, temporary tables, SQL-level `PREPARE`.
- Protocol-level prepared statements are supported from PgBouncer 1.22.0, through the driver. Older or other poolers may not support them.
- A client that waits over 120 seconds for a pooled connection fails with `query_wait_timeout`.
- Scale-to-zero computes can make the first query after idle slow. That is not a bug.

**Driver setup.** Drizzle supports `postgres` (postgres.js) and `node-postgres`. The Drizzle docs warn that postgres.js uses prepared statements by default, which can be a problem in some environments. Turn them off for transaction-mode poolers:

```ts
import postgres from "postgres";
import { drizzle } from "drizzle-orm/postgres-js";
import { sql } from "drizzle-orm";

const client = postgres(config.DATABASE_URL, { prepare: false });
export const db = drizzle({ client });
```

The `prepare: false` option name is from the postgres.js README, which was not fetched in this review **(unverified)**. Confirm it there. The `drizzle({ client })` form is shown in the Drizzle docs.

**Transactions and `SET LOCAL`.** `db.transaction(async (tx) => { ... })` runs queries in one transaction. A transaction pins one real connection, so settings made inside it are safe on a pooler. For row-level security (RLS, rules that limit which rows a database user can see), set a setting with `set_config` and `true` as the last argument. That makes it local to the transaction, like `SET LOCAL`:

```ts
await db.transaction(async (tx) => {
  await tx.execute(sql`select set_config('app.tenant_id', ${tenantId}, true)`);
  // queries here see the tenant; an RLS policy can read current_setting('app.tenant_id')
});
```

Drizzle's docs show `db.transaction`, `tx.rollback()` (it throws) and isolation options. They do not show `tx.execute` **(unverified)**. Check that your Drizzle version offers it. The `set_config` function is standard PostgreSQL. Do not interpolate user text into SQL strings. Use the `sql` tag, which sends parameters.

**Claiming work with `SKIP LOCKED`.** Many workers must not take the same job. The PostgreSQL docs say `SKIP LOCKED` skips rows that are already locked, which avoids lock contention with multiple consumers of a queue-like table. They also say it gives an inconsistent view of data, so it suits queues, not general reads.

```ts
const claimed = await db.transaction(async (tx) => {
  const rows = await tx.execute(sql`
    select id from jobs
    where status = 'pending' and due_at <= now()
    order by due_at
    limit 50
    for update skip locked
  `);
  // mark claimed rows inside the same transaction, then commit
  return rows;
});
```

Mark rows as claimed (status or lease time) in the same transaction, then commit. Do the slow work after commit, or use a lease timeout so a crashed worker's jobs are retried.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pools | pooled vs direct URL, `prepare: false` | Pick the URL by use case |
| Migrations | `drizzle-kit` on the direct URL | Run them without the pooler |
| Transactions | `db.transaction`, `set_config(..., true)` | Scope tenant settings safely |
| Queues | `for update skip locked`, lease or status column | Claim jobs without duplicates |
| Indexes | partial index on pending rows | Keep claim queries fast |

### Common mistakes

- Running migrations through the pooled URL.
- Using `LISTEN/NOTIFY` or session advisory locks through the pooler.
- Using plain `SET` and expecting it to last across queries.
- Leaving prepared statements on with a pooler that does not support them.
- Claiming jobs without `SKIP LOCKED`, so workers block each other or double-process.
- Doing slow work while holding the claim transaction open.

### Self-check

- Why do plain `SET` statements fail through a transaction-mode pooler?
- How do you set a tenant id for RLS safely on a pooled connection?
- What does `SKIP LOCKED` change for two workers running the same query?

---

## 15. Security plugins

**In plain words.** Plugins can add safe default headers and limit how often one client may call you. They are quick to add. They do not replace good authentication and input checks.

### Understand

**`@fastify/helmet`** wraps the `helmet` library. It adds an `onRequest` hook and a `reply.helmet` decorator and sets security headers. It applies to all routes by default. Plugin version 12 and later matches Fastify 5.

```ts
import helmet from "@fastify/helmet";
await app.register(helmet, { contentSecurityPolicy: false }); // example: turn one middleware off
```

For a pure JSON API, the default headers are usually fine. Turn off or tune the Content Security Policy only if you know why.

**`@fastify/rate-limit`** limits requests per client. Plugin version 10 and later matches Fastify 5. Facts from its README:
- `max` defaults to 1000 requests. `timeWindow` defaults to one minute.
- `global` defaults to `true` (applies to all routes in scope). Set `global: false` to limit only chosen routes.
- Counters live in memory by default. With several instances, pass an `ioredis` client so all instances share counters. The README warns that ioredis defaults are not ideal here. Tune `connectTimeout` and `maxRetriesPerRequest`.
- `keyGenerator` picks the client identity. The default is the IP address. Behind a proxy, set Fastify's `trustProxy` correctly, or all clients look like the proxy.
- `allowList` excludes some clients. Per-route limits go in `config: { rateLimit: { ... } }`. `rateLimit: false` exempts a route.

```ts
import rateLimit from "@fastify/rate-limit";
await app.register(rateLimit, { max: 100, timeWindow: "1 minute" });

app.post("/v1/events", { config: { rateLimit: { max: 1000, timeWindow: "1 minute" } }, schema }, handler);
```

**Other security habits:**
- Set `bodyLimit` (default 1 MiB) on ingestion routes explicitly.
- Prefer keying ingestion limits on the API key, not the IP.
- Add `@fastify/cors` only when a browser front end needs it.
- Keep `trustProxy` at its default `false` unless you run behind a known proxy.
- Redact secrets in logs (chapter 11).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Headers | `@fastify/helmet` | Explain what it sets and when to tune it |
| Limits | `@fastify/rate-limit`, `keyGenerator`, Redis store | Limit per key, share state across instances |
| Proxy | `trustProxy` | Get the real client IP |
| Size | `bodyLimit` | Reject oversized bodies |

### Common mistakes

- Using the in-memory rate limit store with several instances.
- Rate limiting by IP behind a proxy without `trustProxy`.
- Copying a Content Security Policy that breaks the API docs page.
- Thinking helmet or rate limits replace authentication.

### Self-check

- Why does an in-memory limiter fail with three instances?
- What does `trustProxy` change for the limiter?
- Which version lines of helmet and rate-limit match Fastify 5?

---

## 16. Cross-cutting mistakes to avoid

**In plain words.** These mistakes appear in many projects. Read them before a code review or before you go live.

- Mixing `return reply.send(x)` and `return x` in one handler.
- Putting a hook, decorator or error handler on the root when it belongs to one module.
- Using `fp` everywhere, which removes the isolation you want.
- Forgetting that a response schema removes unknown fields. If a field is missing, check the schema first.
- Calling `app.listen` inside `buildApp`.
- Reading config from `process.env` in many files.
- Sending internal error text to clients.
- Running migrations on the pooled URL.
- Using `LISTEN/NOTIFY` or session locks through a pooler.
- Not closing the database pool on shutdown.
- Losing events by trusting an in-process bus for must-not-lose work.
- Mocking the database in tests for queue and locking code.
- Following Fastify 4 examples with removed v5 options.
- Not calling `app.close()` in tests.

### Self-check

- Pick five mistakes above. For each, say what symptom you would see.
- Which two of them can silently lose data?

---

## 17. Glossary

| Term | Plain meaning |
|---|---|
| Plugin | A function that adds routes or features to a Fastify app |
| Encapsulation | Each plugin gets a private area; others cannot see its additions |
| Decorator | A named value or function attached to the app, request or reply |
| Hook | A function that runs at a fixed step of a request or of the server life |
| Lifecycle | The fixed order of steps a request passes through |
| Schema | A description of the data a route accepts or returns |
| Type provider | A tool that turns a schema into TypeScript types |
| Validation | Checking input against rules before using it |
| Serialisation | Turning an object into the text that is sent back |
| Coercion | Converting text into another type, such as "10" to the number 10 |
| Idempotent | Safe to repeat; repeating gives the same result |
| Composition root | The one place where all parts are created and connected |
| Modular monolith | One program built from clear modules that own their data |
| Event bus | A small tool where parts send and receive named messages |
| Outbox pattern | Saving an event in a database table, in the same transaction as the data |
| Redaction | Hiding secret values in logs |
| Structured logging | Logs written as JSON objects, not free text |
| Graceful shutdown | Finishing current work and closing resources before exit |
| Liveness probe | A check that says "the process is running" |
| Readiness probe | A check that says "the service can handle traffic now" |
| Connection pool | A set of reusable database connections |
| Pooler (PgBouncer) | A server that shares few real connections among many clients |
| Transaction mode | Pooler mode that gives a connection only for one transaction |
| Prepared statement | A query the database parses once and runs many times |
| Row-level security (RLS) | Database rules that limit which rows a user can see |
| SKIP LOCKED | A SQL option that skips rows already locked by others |
| Lease | A time-limited claim on a job; it expires if the worker dies |
| Rate limit | A cap on requests per client in a time window |
| Reverse proxy | A server in front of your app that forwards requests |
| Fail fast | Stop at start-up when something is wrong, instead of failing later |

---

## 18. Practice labs

1. **Hello and schemas.** Start a server. Add `POST /v1/events` with a zod body, a query schema with `z.coerce`, and a response schema. Confirm a bad body gives 400 and an extra response field is dropped.
2. **Error handler.** Add one `setErrorHandler`. Return a single error shape. Force a 500 and confirm the client sees no internal text but the log does.
3. **Two modules.** Build `ingestion` and `incidents` as plugins with the folder layout in chapter 9. Ingestion emits `event.ingested` on a typed bus. Incidents groups events whose type ends in `.failed` by type and provider and keeps a count.
4. **Scale test.** Send 1000 failing events. Confirm `GET /v1/incidents` shows one incident with count 1000.
5. **Idempotency.** Add a unique constraint on `idempotencyKey`. Send the same event twice. The second response is 202 with `duplicate: true`.
6. **fastify-plugin.** Move the database client into an `fp` plugin with `name`. Prove a sibling can use `app.db`. Then remove `fp` and observe the failure.
7. **Config.** Add a zod config with two database URLs. Start with a missing value and read the error.
8. **Logging.** Enable redaction for `authorization`. Send a request with the header and prove it is hidden in logs.
9. **Shutdown and health.** Add signal handling, `/health/live` and `/health/ready`. Stop the database and see ready return 503 while live returns 200.
10. **Tests.** Write `inject` tests: valid (202), invalid (400), duplicate. Use a real PostgreSQL for the duplicate test.
11. **Pooled connection.** Connect through a pooled URL with `prepare: false`. Run migrations on the direct URL. Try plain `SET` across two queries and see it fail to persist.
12. **RLS tenant.** Create a table with an RLS policy on `current_setting('app.tenant_id')`. Use `set_config(..., true)` in a transaction and prove two tenants see only their rows.
13. **Claiming.** Create a `jobs` table. Start two workers that claim with `for update skip locked`. Prove no job runs twice. Kill a worker mid-job and confirm a lease lets another worker retry.
14. **Security.** Add helmet and rate limiting. Prove the 101st request in a minute gets 429 when `max` is 100.
15. **Separate processes.** Add `worker.ts` as a second entry point that shares modules. Stop and start each one independently.

---

## 19. Interview questions

Tags: **[Fresher]** basics, **[Mid]** working knowledge, **[Senior]** design and trade-offs. Answers are concise models, not scripts.

### Top 10 must-know

**1. [Fresher] How does Fastify differ from Express?**
Fastify uses plugins with their own scope and hooks at named lifecycle points. Validation and serialisation are schema-based and built in. Async handlers can simply return a value, and errors thrown in them are caught. Logging is built in with pino.

**2. [Fresher] What is the Fastify request lifecycle?**
`onRequest`, `preParsing`, parsing, `preValidation`, validation, `preHandler`, handler, `preSerialization`, `onSend`, response, `onResponse`. Errors go to `onError` and then the error handler.

**3. [Mid] What is encapsulation?**
Each `register` creates a new scope. Children inherit from parents. Parents and siblings cannot see a child's hooks, decorators or routes. This keeps modules isolated.

**4. [Mid] When do you use `fastify-plugin`?**
When you want a plugin's decorators or hooks to be visible to the parent and siblings, for example a shared database client. Use it sparingly, because it removes isolation. A wrapped plugin ignores `prefix`.

**5. [Mid] How do you validate input with zod in Fastify?**
Set a type provider (`withTypeProvider<ZodTypeProvider>()`), set `validatorCompiler` and `serializerCompiler`, and put zod schemas under `schema.body`, `querystring`, `params` and `response`. Types are inferred. Failures return 400.

**6. [Mid] How should errors be handled?**
Throw errors. Use one `setErrorHandler` that maps validation errors to 400 and hides internal text for 5xx. Log the detail and return a generic message with the request id. Expected outcomes are normal responses.

**7. [Mid] Why separate `buildApp` from `listen`?**
Tests can build the app with fake dependencies and use `inject` without a port. It also lets different entry points (API and worker) reuse the same setup.

**8. [Senior] How do you shut down a Fastify service gracefully?**
Handle `SIGTERM` and `SIGINT`, call `await app.close()` (stops listening, drains requests, runs `onClose` hooks that close pools), then exit. Finish within the platform's grace period. Workers stop claiming jobs first.

**9. [Senior] How do you use PostgreSQL safely behind a transaction-mode pooler?**
Use the pooled URL for the app and the direct URL for migrations and session features. Disable prepared statements if the pooler does not support them. Do not rely on plain `SET`, `LISTEN/NOTIFY` or session advisory locks. Scope settings with `set_config(..., true)` inside a transaction.

**10. [Senior] How do several workers claim jobs without duplicates?**
Select due rows with `for update skip locked` inside a transaction, mark them claimed (status or lease), and commit. Other workers skip locked rows. Add a lease timeout so jobs from a crashed worker are retried.

### Fresher

**11. [Fresher] What does a handler return in Fastify?**
A value, which becomes the response body with status 200. Use `reply.code(n)` to change the status.

**12. [Fresher] Why use `host: "0.0.0.0"` in Docker?**
The default `localhost` is reachable only inside the container. `0.0.0.0` listens on all interfaces, so the published port works.

**13. [Fresher] What is a hook?**
A function attached to a lifecycle step, such as `onRequest` or `preHandler`, used for auth, timing and similar concerns.

**14. [Fresher] What does a response schema do?**
It shapes and serialises the output. Fields not in the schema are not sent, which helps avoid leaking data.

**15. [Fresher] What is `app.inject`?**
A way to send a fake request in-process for tests, with no network port.

**16. [Fresher] What is `request.id`?**
A per-request identifier that appears in logs from `request.log`, so you can trace one request.

**17. [Fresher] Why parse environment config at start-up?**
To fail immediately on missing or wrong values, and to get typed values everywhere else.

### Mid

**18. [Mid] Why must you call `withTypeProvider` again inside a plugin?**
Type provider types do not carry across scopes. Without it, routes in the plugin lose their inferred types.

**19. [Mid] Why can `decorateRequest("user", {})` fail?**
Reference types are not allowed as initial values because one object would be shared across requests. Start with `null` and set the value in an `onRequest` hook.

**20. [Mid] How do you hide secrets in logs?**
Use pino `redact` with explicit paths such as `req.headers.authorization`. Never log full URLs with passwords. Prefer explicit paths over wildcards for cost.

**21. [Mid] What is the difference between liveness and readiness?**
Liveness says the process runs. Readiness says it can serve traffic, so it checks dependencies like the database. A failed readiness stops traffic. A failed liveness can restart the process.

**22. [Mid] Why does a response schema sometimes "lose" a field?**
Output properties not declared in the schema are removed. Add the field to the schema if it should be sent.

**23. [Mid] How do you make ingestion safe for client retries?**
Require an idempotency key stored under a unique constraint. A repeat returns the same success response and marks `duplicate`.

**24. [Mid] What belongs in routes, service and repo layers?**
Routes handle HTTP only (parse, call service, reply). Services hold business rules. Repos hold database access. Layers are not skipped.

**25. [Mid] How do you rate limit across several instances?**
Use `@fastify/rate-limit` with a shared Redis client, so counters are not per process. Tune the Redis client timeouts as the README advises.

**26. [Mid] How do you test routes that use the database?**
Use `inject` plus a real PostgreSQL, with migrations and clean-up. Use fakes for unit tests of services. Do not mock locking behaviour.

**27. [Mid] What changed in Fastify 5 that affects migration from 4?**
Node 20 or later, full JSON Schema required for hand-written schemas, `.listen` takes an options object, `loggerInstance` for custom loggers, `reply.elapsedTime` replaces `getResponseTime`, and no mixing of callback and async plugins.

### Senior

**28. [Senior] How do you keep modules independent in a modular monolith?**
One public `index.ts` per module, each module owns its tables, cross-module work goes through events or exported functions, dependencies come in by options, and only the composition root knows all modules. Enforce imports with a lint rule.

**29. [Senior] What are the limits of an in-process event bus?**
Events are lost if the process dies before handling. Handlers run in the same process, so a slow one can hurt latency. For must-not-lose work, write an outbox row in the same transaction and let a worker process it. Use the bus for non-critical reactions.

**30. [Senior] How would you split a module into its own service later?**
Because it only uses events and its own tables, replace the in-process bus with a queue, move its tables to its own database, and expose its public functions as an API. Callers should not change.

**31. [Senior] How do you run an API and a worker from one codebase?**
Two entry files that build from the same modules and config. Each has its own signal handling, health checks and scaling. Only the API opens an HTTP port.

**32. [Senior] Why is `set_config(..., true)` used for row-level security on a pooler?**
A transaction holds one real connection, and the setting is local to it, so it cannot leak to another client after the connection returns to the pool. A plain session `SET` would not persist or could leak between clients.

**33. [Senior] What can go wrong with `SKIP LOCKED` queues?**
The view is inconsistent by design, so it only suits queue-like tables. Jobs claimed by a crashed worker need a lease or status timeout. Ordering is not strict across workers. Slow work inside the claim transaction holds locks and a connection.

**34. [Senior] How do you decide where an error handler lives?**
Put the main handler on the root so all modules share one error shape. Add a module handler only when that module needs a different format. Remember handlers are encapsulated and the nearest wins.

**35. [Senior] How do you protect against abuse on a public ingestion endpoint?**
Authenticate with API keys, rate limit by key, set an explicit `bodyLimit`, validate every field with a schema, return idempotent responses, redact secrets in logs and set `trustProxy` correctly behind a proxy.

---

## 20. References

Official documentation, checked on 2026-11-06.

**Fastify**
- Getting Started: https://fastify.dev/docs/latest/Guides/Getting-Started/
- Lifecycle: https://fastify.dev/docs/latest/Reference/Lifecycle/
- Hooks: https://fastify.dev/docs/latest/Reference/Hooks/
- Server (factory options): https://fastify.dev/docs/latest/Reference/Server/
- Reply: https://fastify.dev/docs/latest/Reference/Reply/
- Validation and Serialization: https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/
- Errors: https://fastify.dev/docs/latest/Reference/Errors/
- Plugins: https://fastify.dev/docs/latest/Reference/Plugins/
- Encapsulation: https://fastify.dev/docs/latest/Reference/Encapsulation/
- Decorators: https://fastify.dev/docs/latest/Reference/Decorators/
- Type Providers: https://fastify.dev/docs/latest/Reference/Type-Providers/
- Logging: https://fastify.dev/docs/latest/Reference/Logging/
- Testing guide: https://fastify.dev/docs/latest/Guides/Testing/
- Migration guide to v5: https://fastify.dev/docs/latest/Guides/Migration-Guide-V5/

**Fastify packages**
- fastify-type-provider-zod: https://github.com/turkerdev/fastify-type-provider-zod
- @fastify/type-provider-zod (fork, API unverified): https://github.com/fastify/fastify-type-provider-zod
- fastify-plugin: https://github.com/fastify/fastify-plugin
- @fastify/helmet: https://github.com/fastify/fastify-helmet
- @fastify/rate-limit: https://github.com/fastify/fastify-rate-limit

**Logging, database and SQL**
- Pino redaction: https://github.com/pinojs/pino/blob/main/docs/redaction.md
- Drizzle with PostgreSQL: https://orm.drizzle.team/docs/get-started-postgresql
- Drizzle transactions: https://orm.drizzle.team/docs/transactions
- Neon connection pooling: https://neon.com/docs/connect/connection-pooling
- PostgreSQL SELECT (locking clause): https://www.postgresql.org/docs/current/sql-select.html

**Not fetched in this review (unverified):** the zod documentation, the postgres.js README (`prepare` option) and the Vitest documentation.
