# Section O — Gaps Supplement (1/3): Communication Styles

> Topics that sit outside A–N. This file covers **how services talk** beyond plain REST.
> Same format: core idea → mechanism/why → real-world variations → stack note → one-line summary.

---

## O1 — Real-time: WebSockets vs SSE vs polling

**Diagram**
```
 short poll : client -ask-> server (every 5s, mostly "nothing new")
 long poll  : client -ask-> server holds... answers when data -> ask again
 SSE        : client <======= server  (one-way stream, auto-reconnect)
 WebSocket  : client <=======> server  (two-way, chat/presence)

 scaling: user on instance B, event on instance A
    A --publish--> [Redis pub/sub] --> B --> user
```
**In plain words.** Polling is you phoning every minute. SSE is the shop calling you when your
order is ready. WebSocket is an open phone line where both talk. Open lines belong to one
instance, so you need Redis to pass messages between instances.

**Core idea.** REST is request→response: the client must **ask** for updates. For **server-pushed**
updates (notifications, live status, chat) you need a different transport.

**The options (ascending capability):**
- **Short polling** — client re-requests every N seconds. Simple, but wasteful and laggy.
- **Long polling** — request hangs open until the server has data, then reconnects. Better latency, still
  HTTP overhead per message.
- **Server-Sent Events (SSE)** — a **one-way** persistent stream (server → client) over plain HTTP. Auto
  reconnect, simple, great for **notifications / live progress**. Can't send client → server on the same
  channel.
- **WebSockets** — a **full-duplex** persistent connection (both directions). For **chat, presence,
  collaborative** features. More moving parts (connection state, scaling, heartbeats).

**Scaling reality.** Persistent connections are **stateful** — a WebSocket lives on **one instance**. To
scale across your instances you need a **pub/sub backplane** (Redis pub/sub, or a managed service) so a
message published on instance A reaches a client connected to instance B. Also: auth the handshake
(validate the JWT on connect), heartbeat/ping to detect dead connections, and cap connections.

**Stack note.** For your **notifications fan-out (N8)** and the **WhatsApp/real-time** surface: SSE is
often enough (server → client only) and far simpler than WebSockets. Use Redis pub/sub as the backplane
so any instance can push to any connected client.

> **One line:** poll (ask repeatedly), SSE (one-way server push over HTTP, simple), or WebSockets
> (full-duplex, for chat/presence) — persistent connections are stateful, so scale them with a Redis
> pub/sub backplane and auth the handshake.

---

## O2 — GraphQL (conceptual)

**Diagram**
```
 REST   : GET /orders/1, GET /orders/1/items, GET /customers/9  (3 trips)
 GraphQL: one query { order(1){ items{name} customer{name} } }  (1 trip, exact fields)
 costs : N+1 built in (use DataLoader), per-field authz, limit query depth
```
**In plain words.** REST is a set menu. GraphQL is "I will have only these items". The kitchen
must work harder to avoid repeating trips.

**Core idea.** One endpoint where the **client specifies exactly what data it wants** (fields, nested
relations) in a query language, and gets back exactly that shape — no over/under-fetching.

**Mechanism.** A typed **schema** defines types and fields; **resolvers** fetch each field. Client sends
a query; the server walks resolvers to build the response.

**Trade-offs.**
- **Wins:** no over-fetching, one round trip for nested data, strong typed contract, great for varied
  clients (web/mobile want different fields).
- **Costs:** the **N+1 problem is built-in** (each nested field can trigger a query) → you need
  **DataLoader** batching (your E8). Caching is harder than REST (can't just cache by URL). Complexity;
  rate-limiting/authorization must be **per-field**, and a malicious deep/nested query is a DoS (need
  query depth/cost limits).

**When to use.** Many client types with divergent data needs, or deeply nested/related data. For a
straightforward B2B API, REST is usually simpler and fine.

> **One line:** GraphQL lets clients request the exact shape they need from one typed endpoint — powerful
> for varied clients and nested data, but you own N+1 batching, per-field authz, and query-cost limits.

---

## O3 — gRPC (conceptual) — internal service-to-service

**Diagram**
```
 public edge : browser <-- REST/JSON --> gateway
 internal    : central <== gRPC (binary, .proto contract, HTTP/2) ==> buyer
 .proto -> generated typed client/server code
```
**In plain words.** Public speaking uses plain language (JSON). Colleagues use a fast, strict
shorthand (gRPC).

**Core idea.** A high-performance **RPC** framework: you define the service and messages in a **`.proto`**
schema, and it generates typed client/server stubs. Uses **Protocol Buffers** (compact binary) over
**HTTP/2**.

**Why it exists.** For **internal service-to-service** calls (your central ↔ buyer ↔ warehouse), gRPC is
faster and stricter than JSON/REST: binary payloads (smaller/faster than JSON), a hard typed contract
(the `.proto` is the source of truth), HTTP/2 multiplexing (A3), and **streaming** (client/server/
bidirectional).

**Trade-offs.** Not human-readable, not natively browser-friendly (needs a proxy), more tooling. Great
**behind the gateway**; keep REST/JSON at the **public edge**.

**Stack note.** A common pattern: **REST/JSON at the edge for clients, gRPC between services** internally.
You don't need it now (JSON between your services is fine), but it's the standard answer for "how do I make
internal calls faster and more strictly typed."

> **One line:** gRPC is schema-first binary RPC over HTTP/2 for fast, strictly-typed internal
> service-to-service calls — REST at the public edge, gRPC between services when you need the speed/contract.

---

## O4 — Outbound webhooks (you as the sender)

**Diagram**
```
 event (order placed) -> delivery job -> sign HMAC + timestamp + event id
   -> POST customer URL
        200 -> done
        fail -> retry backoff+jitter -> ... -> DLQ + customer log + manual replay
 customer dedupes by event id. Never POST to internal IPs (SSRF).
```
**In plain words.** You now send the letters. You seal them, keep trying if nobody answers, and
tell the receiver "if you get two copies, use the ID".

**Core idea.** The mirror of D16/G2: instead of *receiving* webhooks, **you notify your B2B customers'
systems** when events happen (order placed, KYC approved, payment received). This is a real product
feature for a B2B platform.

**What you must build (the hard parts):**
- **Signing** — sign each payload with **HMAC** (per-customer secret) so *they* can verify it's really you
  (the C17/D16 pattern, now outbound). Include a **timestamp** so they can guard replay.
- **Retries with backoff** — the customer's endpoint will be down sometimes. Retry with exponential
  backoff + jitter; give up after N and surface a failure.
- **Delivery guarantees & idempotency** — you deliver **at-least-once**, so include a unique **event id**
  and tell customers to dedupe (F2). Never guarantee exactly-once over the network.
- **Dead-letter / visibility** — failed deliveries go to a **DLQ** (F6); give customers a dashboard/log of
  deliveries and a **manual replay**.
- **Security** — only POST to customer-registered HTTPS URLs; beware this is an **SSRF surface** if the
  URL is user-supplied (D6) — validate and don't let it hit internal IPs.
- **Ordering** — networks reorder; if order matters, include a sequence number rather than assuming.

**Stack note.** Build it on **Bull** (F1): an event enqueues a delivery job; the worker signs, POSTs,
retries, and DLQs. Store delivery attempts for the customer-facing log.

> **One line:** sending webhooks means signing each payload (HMAC + timestamp), retrying with backoff,
> delivering at-least-once with an event id for dedupe, DLQ-ing failures with replay, and treating the
> target URL as an SSRF surface.

---

## O5 — Search infrastructure (when Postgres FTS isn't enough)

**Diagram**
```
 Postgres (truth) --events/CDC--> Search engine (inverted index copy)
 user types "lether bag" -> typo-tolerant, ranked, faceted results
 writes go to Postgres only; search can lag, never authoritative
```
**In plain words.** The search engine is a book index kept up to date from the real library.

**Core idea.** Postgres full-text (`tsvector` + GIN, your B9) handles a lot. When you need typo-tolerance,
relevance ranking, faceting, and scale, you reach for a **search engine** (Elasticsearch / OpenSearch /
Typesense / Meilisearch).

**Mechanism & cost.** The search engine holds a **denormalized copy** of your data in an inverted index.
That means a **sync problem**: you must keep it updated from your source of truth (via events/CDC — E11 —
not dual-writes). It's a **secondary index, not your system of record** — never treat search results as
authoritative for writes.

**When to use.** Product/catalog search with fuzzy matching, ranking, filters/facets at scale. For your
supplier catalog, this is a plausible future need beyond Postgres FTS.

> **One line:** use a search engine for typo-tolerant, ranked, faceted search at scale — it's a
> denormalized secondary index you keep in sync via events, never your source of truth.

---

## O6 — File streaming & large/resumable uploads (extends N9)

**Diagram**
```
 BAD : upload 500MB -> your server buffers it all -> memory gone, loop blocked
 OK  : stream: request --pipe--> Blob  (constant memory)
 BEST: client --SAS URL--> Blob directly (your server not in the path)
 huge/flaky: multipart/resumable chunks
```
**In plain words.** Do not carry the whole sofa through your house. Let the delivery van unload at
the loading dock.

**Core idea.** Big files must be **streamed**, not buffered fully in memory (buffering a 500 MB upload
blocks the event loop and blows memory — A1/D1 size limits).

**Techniques.**
- **Streaming** — pipe the request/response stream to/from storage; constant memory regardless of size.
- **Direct-to-storage** — issue a **signed SAS URL** (G5) so the client uploads **straight to Azure Blob**,
  bypassing your server entirely (no bandwidth/memory cost to you).
- **Resumable / multipart** — for very large or flaky-network uploads, chunked/multipart uploads let a
  failed upload resume instead of restarting.
- **Validate** type/size/content even with direct upload (check after, or via storage events).

> **One line:** stream large files (never buffer them), prefer signed direct-to-blob uploads to bypass
> your server, and use resumable/multipart for big or flaky uploads.
