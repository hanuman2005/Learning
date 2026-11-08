# 📋 System Design Mastery — 30-Day Curriculum Tracker

> **Rule:** 3 hrs/day = 2h learn + 1h apply. A day is ✅ only when its "Apply" box is checked too.
> **Legend:** ☑ = done · Day 0 was UML (already completed ✅)

**Progress: Week 0 ✅ · Week 1 ⬜ · Week 2 ⬜ · Week 3 ⬜ · Week 4 ⬜**

> **📘 Language: Java.** All code examples are Java. Every topic starts with a plain-English **"In plain words:"** primer (basics + everyday analogy) and then goes deep — read the primer first, then the Why/What/When/How/Where.

---

## Week 0 — Prerequisite (Done)

- [x] UML: Class, Sequence, Use Case, Activity, Component, Deployment diagrams → `uml-study-guide.md`
- [ ] UML: State Machine diagram (added later — read Part 7 of the guide)
- [x] Day 1 intro: What is system design, HLD, LLD, HLD vs LLD, SOLID intro, patterns intro → `day-01-intro-system-design.md`

---

## Week 1 — LLD Foundations

### Day 1 ✅ — Introduction (completed)
- [x] What is system design + 5 key attributes (scalability, reliability, performance, maintainability, security)
- [x] HLD — why/what/when/how/where
- [x] LLD — why/what/when/how/where
- [x] HLD vs LLD differences
- [x] SOLID — names + one-liners only
- [x] Design patterns — 3 categories + star patterns
- [x] Principles vs patterns
- [x] When bad practice wins (trade-off thinking)

### Day 2 — OOP Deep-Dive + SOLID In Depth
- [ ] Encapsulation, inheritance, polymorphism, abstraction (with real examples)
- [ ] Composition vs inheritance — when each wins
- [ ] SRP — violation + refactoring + code
- [ ] OCP — violation + refactoring + code
- [ ] LSP — violation + refactoring + code
- [ ] ISP — violation + refactoring + code
- [ ] DIP — violation + refactoring + code
- [ ] Dark side: when SOLID over-abstraction hurts
- [ ] **Apply:** model a library system in classes applying SOLID

### Day 3 — Creational Patterns
- [ ] Singleton (+ why it's controversial, DI alternative)
- [ ] Factory Method
- [ ] Abstract Factory (light — less used)
- [ ] Builder
- [ ] **Apply:** code Factory + Builder + Singleton once each

### Day 4 — Behavioral Patterns I
- [ ] Strategy
- [ ] Observer
- [ ] State
- [ ] **Apply:** code Strategy + Observer; class diagram for State

### Day 5 — Structural Patterns
- [ ] Decorator
- [ ] Adapter
- [ ] Facade
- [ ] Proxy
- [ ] **Apply:** code any 2; spot 2 in a real framework you use

### Day 6 — LLD Problem: Parking Lot
- [ ] Requirements gathering + assumptions
- [ ] Class diagram (entities, relationships, patterns used)
- [ ] Code skeleton (working core flow)
- [ ] Edge cases list

### Day 7 — LLD Problem + Week Review
- [ ] Elevator system OR BookMyShow (full: requirements → diagram → code)
- [ ] Re-explain all 5 SOLID out loud (Feynman test)
- [ ] Redo any weak pattern from memory

---

## Week 2 — HLD Building Blocks

### Day 8 — Networking Essentials
- [ ] HTTP/HTTPS — methods, status codes, headers, HTTP/2 vs 1.1
- [ ] TCP vs UDP — guarantees, when each
- [ ] DNS — resolution flow
- [ ] TLS — handshake at high level
- [ ] WebSockets vs polling vs SSE
- [ ] **Apply:** trace a full request (browser → DNS → TCP → TLS → server) on paper

### Day 9 — API Design
- [ ] REST — resources, verbs, status codes, versioning
- [ ] Idempotency — why it matters, idempotency keys
- [ ] Pagination — offset vs cursor
- [ ] gRPC vs REST vs GraphQL — when each
- [ ] **Apply:** design the full API contract for a todo app (payloads + error codes)

### Day 10 — Databases
- [ ] SQL vs NoSQL — real decision criteria
- [ ] Indexing — B-tree basics, when indexes hurt
- [ ] Transactions & ACID
- [ ] NoSQL types — document, key-value, wide-column, graph (one use case each)
- [ ] **Apply:** pick DBs for Instagram's services, justify each choice

### Day 11 — Caching
- [ ] Cache-aside vs write-through vs write-behind
- [ ] Eviction policies — LRU, LFU, TTL
- [ ] Cache invalidation problems (the hard part)
- [ ] CDN — how it works, what to put on it
- [ ] Redis vs Memcached
- [ ] **Apply:** add caching layers to the Swiggy design, mark stale-data risks

### Day 12 — Load Balancing & Gateways
- [ ] L4 vs L7 load balancing
- [ ] Algorithms — round robin, least connections, consistent hashing
- [ ] Health checks & failover
- [ ] Reverse proxy vs API gateway
- [ ] Rate limiting — token bucket, sliding window
- [ ] **Apply:** draw Instagram's traffic path with LB + gateway + rate limits

### Day 13 — Replication & Sharding
- [ ] Primary-replica replication, replication lag
- [ ] Failover — how promotion works
- [ ] Sharding strategies — range, hash, geo
- [ ] Hot key/celebrity problem
- [ ] Resharding pain
- [ ] **Apply:** shard the Swiggy orders DB — pick a shard key, defend it

### Day 14 — CAP + First Full HLD
- [ ] CAP theorem — what it actually says (and doesn't)
- [ ] Strong vs eventual consistency — real examples
- [ ] PACELC (light)
- [ ] **Apply:** full HLD — **URL shortener** (requirements → estimates → diagram → data model → bottlenecks)

---

## Week 3 — Distributed Systems

### Day 15 — Message Queues
- [ ] Why async — decoupling, buffering, smoothing spikes
- [ ] Kafka vs RabbitMQ — log vs broker model
- [ ] Delivery guarantees — at-most-once, at-least-once, exactly-once (why it's hard)
- [ ] Dead letter queues, consumer groups
- [ ] **Apply:** add a queue to Swiggy order flow — what breaks if consumer is down?

### Day 16 — Event-Driven Architecture
- [ ] Events vs commands
- [ ] Pub/sub vs point-to-point
- [ ] CQRS — concept + when it's worth it
- [ ] Event sourcing — concept only (rarely fully used — don't go deep)
- [ ] **Apply:** redesign Instagram fanout as event-driven, diagram it

### Day 17 — Microservices vs Monolith
- [ ] Monolith → microservices: real triggers to split
- [ ] Service boundaries — how to draw them (DDD light)
- [ ] Inter-service communication — sync vs async choices
- [ ] Saga pattern — choreography vs orchestration
- [ ] Distributed monolith anti-pattern
- [ ] **Apply:** split (or defend not splitting) a food-delivery monolith

### Day 18 — Fault Tolerance
- [ ] Timeouts + retries + exponential backoff & jitter
- [ ] Circuit breaker
- [ ] Bulkheads & backpressure
- [ ] Graceful degradation — feature flags, fallbacks
- [ ] **Apply:** payment service is down — design Swiggy's behavior end to end

### Day 19 — Distributed Problems
- [ ] Idempotency keys in practice (double-payment problem)
- [ ] Distributed locks — Redis lock, pitfalls
- [ ] Consensus — what Raft/Paxos solve (high level only)
- [ ] Clock skew — why "what happened first" is hard
- [ ] **Apply:** design "exactly one rider gets assigned" for Swiggy

### Day 20 — Observability & Security
- [ ] Logs vs metrics vs traces — the 3 pillars
- [ ] Alerting — SLI/SLO basics
- [ ] AuthN vs AuthZ
- [ ] OAuth2 + JWT — flows, storage, expiry
- [ ] Encryption at rest / in transit, secrets management
- [ ] OWASP top risks (injection, XSS — awareness level)
- [ ] **Apply:** add auth + observability to the URL shortener design

### Day 21 — Estimation + Rate Limiter Design
- [ ] Back-of-envelope math — powers of 2, latency numbers everyone should know
- [ ] QPS / storage / bandwidth estimation drill ×3
- [ ] **Apply:** full HLD — **Rate limiter** (algorithms, distributed counters, where it sits)

---

## Week 4 — Full Design Practice

> Format each day (3h): 30min requirements & estimates → 1h HLD diagram → 1h deep-dive one component → 30min bottlenecks & trade-offs. Save each as `day-XX-design-<system>.md`.

### Day 22 — Chat App (WhatsApp)
- [ ] Requirements + estimates
- [ ] HLD diagram (WebSockets, message delivery, online status)
- [ ] Deep-dive: message storage + delivery guarantees
- [ ] Bottlenecks & trade-offs

### Day 23 — News Feed (Twitter/Instagram)
- [ ] Requirements + estimates
- [ ] HLD (fanout on write vs read — the classic)
- [ ] Deep-dive: celebrity problem
- [ ] Bottlenecks & trade-offs

### Day 24 — Video Platform (YouTube)
- [ ] Requirements + estimates
- [ ] HLD (upload pipeline, transcoding, CDN)
- [ ] Deep-dive: storage + streaming
- [ ] Bottlenecks & trade-offs

### Day 25 — Ride-Hailing (Uber)
- [ ] Requirements + estimates
- [ ] HLD (location updates, matching)
- [ ] Deep-dive: geo-indexing (geohash/quadtree)
- [ ] Bottlenecks & trade-offs

### Day 26 — Notification System
- [ ] Requirements + estimates
- [ ] HLD (queues, fan-out, multi-channel: push/SMS/email)
- [ ] Deep-dive: retries + dedup + rate limits per user
- [ ] Bottlenecks & trade-offs

### Day 27 — E-commerce Checkout
- [ ] Requirements + estimates
- [ ] HLD (cart, inventory, order, payment)
- [ ] Deep-dive: payment idempotency + inventory consistency
- [ ] Bottlenecks & trade-offs

### Day 28 — Your Weakest Area
- [ ] Pick: Distributed cache OR Google Docs (collaboration/OT) OR revisit any Week 2–3 gap
- [ ] Full design or focused re-study
- [ ] Note what was still shaky

### Day 29 — LLD Revision Day
- [ ] Redo Parking Lot from memory (no notes) — compare with Day 6 version
- [ ] New LLD: vending machine OR LRU cache (code it)
- [ ] Skim all pattern class diagrams once

### Day 30 — Mock Interview Day 🎯
- [ ] Unseen HLD problem — strict 45 min
- [ ] Unseen LLD problem — strict 45 min
- [ ] Self-review: score clarity, structure, trade-off reasoning
- [ ] Write final gaps list → becomes your revision plan

---

## 📁 File Index — all 30 days created ✅ (study one per day, tick boxes above)

| Day | File | Studied? |
|---|---|---|
| 0 | `uml-study-guide.md` | ✅ |
| 1 | `day-01-intro-system-design.md` | ✅ |
| 2 | `day-02-oop-solid.md` | ⬜ |
| 3 | `day-03-creational-patterns.md` | ⬜ |
| 4 | `day-04-behavioral-patterns-1.md` | ⬜ |
| 5 | `day-05-structural-patterns.md` | ⬜ |
| 6 | `day-06-lld-parking-lot.md` | ⬜ |
| 7 | `day-07-lld-elevator-review.md` | ⬜ |
| 8 | `day-08-networking-essentials.md` | ⬜ |
| 9 | `day-09-api-design.md` | ⬜ |
| 10 | `day-10-databases.md` | ⬜ |
| 11 | `day-11-caching.md` | ⬜ |
| 12 | `day-12-load-balancing-gateways.md` | ⬜ |
| 13 | `day-13-replication-sharding.md` | ⬜ |
| 14 | `day-14-cap-consistency-url-shortener.md` | ⬜ |
| 15 | `day-15-message-queues.md` | ⬜ |
| 16 | `day-16-event-driven-architecture.md` | ⬜ |
| 17 | `day-17-microservices-vs-monolith.md` | ⬜ |
| 18 | `day-18-fault-tolerance.md` | ⬜ |
| 19 | `day-19-distributed-problems.md` | ⬜ |
| 20 | `day-20-observability-security.md` | ⬜ |
| 21 | `day-21-estimation-rate-limiter.md` | ⬜ |
| 22 | `day-22-design-chat-app.md` | ⬜ |
| 23 | `day-23-design-news-feed.md` | ⬜ |
| 24 | `day-24-design-video-platform.md` | ⬜ |
| 25 | `day-25-design-ride-hailing.md` | ⬜ |
| 26 | `day-26-design-notification-system.md` | ⬜ |
| 27 | `day-27-design-ecommerce-checkout.md` | ⬜ |
| 28 | `day-28-design-distributed-cache.md` | ⬜ |
| 29 | `day-29-lld-revision.md` | ⬜ |
| 30 | `day-30-mock-interview-day.md` | ⬜ |

---

## Daily Ritual (don't skip)

- [ ] End every session: explain today's topic out loud in 2 min (Feynman test)
- [ ] Every "Apply" ends with a diagram
- [ ] Check the boxes here + commit to git: `git add . && git commit -m "day XX: <topic>"`
