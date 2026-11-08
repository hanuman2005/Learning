# Section J — Architecture

## J1 — The monolith → microservices spectrum (FULL)

**Diagram**
```
 monolith -> modular monolith -> services -> microservices -> serverless
 gain: independent scale/deploy      cost: network failures, no shared txn, ops
 you are here: central | buyer | warehouse (3 services, own DB each)
 do next: harden them (C17 service auth, F4 sagas, G1 breakers)
```
**In plain words.** Splitting into more services is like opening more branches. It scales, but you
need more managers and communication. Do not open more until needed.

- Spectrum: **monolith → modular monolith → SOA/services → microservices → serverless**. Moving right
  buys **independent scale/deploy** but adds **network failure, no shared transaction, more ops**.
- **Default to a modular monolith**; split only when a real driver appears (team scaling, independent
  scale, isolation).
- **A multi-service setup is already microservices** (service-A / service-B / service-C). Advice given: don't add more
  services — solidify the three (service-to-service auth C17, sagas F4, circuit breakers G1).
- "Logout not instant" with stateless JWT → short access token + stored refresh at central + optional
  Redis denylist for instant ban (C4/C5).
- One line: pick the least-distributed architecture that meets the need; you're already at
  microservices, so harden the three services rather than adding more.

## J2 — Layered architecture (FULL)

**Diagram**
```
 route/middleware -> controller -> service -> repository -> DB
 each layer calls only the layer below; errors bubble up
```
**In plain words.** A chain of command: each person only talks to the person directly under them.

- routes/middleware → controller → service → repository → DB (down); results/errors flow up. Each layer
  calls only the one below. (Mirrors B6.)
- One line: strict downward layering (controller→service→repo→DB) with each layer calling only the next.

## J3 — Hexagonal / ports & adapters / clean (reference)

**Diagram**
```
          +-------------- domain core --------------+
          |  business rules, depends on PORTS only  |
          +--------+-----------------------+--------+
                   | port: Repo            | port: Notifier
             adapter: Postgres       adapter: email provider/SMS provider
 swap adapters, core never changes; easy to test
```
**In plain words.** Your business logic is a laptop. Ports are sockets (USB). Adapters are the
devices you plug in and swap.

- Business core depends on **ports** (interfaces); **adapters** implement them (DB, HTTP, queue). Swap
  infrastructure without touching the core; highly testable.
- One line: isolate the domain behind ports; adapters plug in infrastructure.

## J4 — Event-driven architecture (reference)

**Diagram**
```
 order-service --publishes--> [OrderPlaced] --> warehouse reacts
                                            --> notify reacts
 needs: outbox (F3), idempotent consumers (F2), stable event schema
```
**In plain words.** Instead of phoning each person, you announce on the loudspeaker and whoever
cares reacts.

- Services communicate via **events** (pub/sub) instead of direct calls → loose coupling, async. Needs
  outbox (F3), idempotency (F2), and event schema care.
- One line: decouple services with events; pair with outbox + idempotency.

## J5 — CQRS (reference)

**Diagram**
```
 commands (writes) -> write model (normalized) --events--> read model (denormalized view)
 queries  (reads)  ------------------------------------> read model
```
**In plain words.** One counter for taking orders, another for answering questions, each set up
for its own job.

- Separate the **write model** from the **read model** (optimized read stores/denormalized views).
  Complexity for read-scale/clarity.
- One line: split reads from writes when read and write shapes diverge enough to justify it.

## J6 — API gateway / BFF (reference)

**Diagram**
```
 web app   -> BFF-web    --\
 mobile    -> BFF-mobile --- API GATEWAY (auth, routing, rate-limit) -> services
```
**In plain words.** A gateway is the reception desk. A BFF is a receptionist who speaks each
client's language.

- A **gateway** fronts services (auth, routing, rate-limit); a **BFF** (backend-for-frontend) tailors an
  API per client (web/mobile).
- One line: front services with a gateway; add a BFF to tailor APIs per client.

## J7 — API versioning (reference)

**Diagram**
```
 /v1/orders (old clients)     /v2/orders (new shape)
 add a field = safe (additive). rename/remove = breaking -> new version
```
**In plain words.** Adding a room is fine. Moving the front door needs a new address and notice.

- Version via URL (`/v1`) or header. Additive changes are backward-compatible; breaking changes need a
  new version + deprecation window.
- One line: version your API and make changes additive where possible.

## J8 — Deprecation strategy (reference)

**Diagram**
```
 announce -> run old+new together -> watch old usage -> notice -> sunset
```
**In plain words.** Close a road only after signs, a detour, and traffic has dropped to zero.

- Announce, dual-run old+new, monitor usage, sunset with notice. Never silently break clients.
- One line: deprecate with notice and a dual-run window, never silently.

## J9 — Service boundaries (reference)

**Diagram**
```
 split by CAPABILITY: [Onboarding] [Orders] [Warehouse]  each owns its data
 smell: chatty calls (10 per request), or two services sharing one DB
```
**In plain words.** Draw lines around things that change together. If two services call each other
constantly, they probably belong together.

- Draw boundaries around **business capabilities** that change together and own their data. Avoid chatty
  cross-service calls and shared databases.
- One line: split by business capability with owned data; avoid shared DBs and chatty calls.

## J10 — Data ownership across services (reference)

**Diagram**
```
 buyer needs supplier info
   NO : buyer reads central's database directly
   YES: buyer calls central API  (or consumes its events)
```
**In plain words.** You ask the owner of the data. You do not break into their filing cabinet.

- Each service owns its data; others get it via API/events, not by reaching into its DB. (You already do
  this — service-A / service-B / service-C each own a DB.)
- One line: one service owns each dataset; others read via API/events, never its DB directly.
