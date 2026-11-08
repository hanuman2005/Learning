# System Design Interview Question Bank

A practice bank for the 30-day course in this folder. Use it after you finish a topic, not before.

How to use it:

1. Cover the answer. Say your own answer out loud for 1 to 2 minutes.
2. Read the model answer. Note what you missed.
3. Open the "Revise" file for the full explanation and examples.
4. Repeat the missed questions after 3 days.

Level tags: **[Fresher]** = basic idea, **[Mid]** = you can apply it, **[Senior]** = you can weigh trade-offs and failures.

Note on numbers: figures like "about 100k operations per second" are rough rules of thumb. Real values depend on hardware, data size, and settings. Always say "roughly" in an interview.

---

## Contents

1. How to answer a system design question
2. Top 15 must-know questions
3. Question bank by topic (92 questions)
4. Case-study prompts (12)
5. Common mistakes in design interviews

---

## 1. How to answer a system design question

Use the same eight steps every time. The order keeps you calm and shows structured thinking. (Source: day-30-mock-interview-day.md, "universal framework".)

**Step 1. Clarify requirements (about 5 min).**
- Ask who the users are and what the core use case is.
- Split requirements into functional (what the system does) and non-functional (speed, scale, availability, consistency).
- Write an "in scope" and "out of scope" list where the interviewer can see it.

**Step 2. Estimate (about 4 min).**
- Do rough math out loud: users, requests per second (QPS), storage, bandwidth.
- Trick: X per day divided by 100,000 is about X per second. One million requests per day is about 12 per second.
- Add a peak factor (2 to 3 times for normal apps, more for spiky apps).

**Step 3. Define the API (about 4 min).**
- List the 3 to 6 main endpoints with inputs and outputs.
- Mark where idempotency (safe repeat of a request) matters, such as payments.

**Step 4. Design the data model (about 5 min).**
- List the main entities and their keys.
- Pick a database type and give one sentence of reasoning.

**Step 5. Draw the high-level design (about 10 min).**
- Client, load balancer, services, database, cache, queue.
- Walk one request from start to end.

**Step 6. Deep dive (about 12 min).**
- Pick the 1 or 2 hardest parts and go deep.
- Examples: fan-out in a feed, matching in ride-hailing, double-charge in checkout.

**Step 7. Discuss trade-offs.**
- For each key choice, say what you gain, what you lose, and why you accept it.

**Step 8. Failure modes and scaling (about 5 min).**
- Ask "what if this box dies?" for each component.
- Say what changes at 10 times the load.
- End with a short summary. Never stop in the middle of a deep dive.

A good sentence pattern: "I choose X because the requirement is Y. The cost is Z. If Y changes, I would pick W."

---

## 2. Top 15 must-know questions

If you have only one week, learn these first. Full answers are in section 3. The number in brackets is the question number.

1. What are the four pillars of OOP and the five SOLID principles? [Q1, Q3]
2. Singleton, Factory, Strategy, Observer: when do you use each? [Q9 to Q14]
3. What happens when you type a URL and press Enter? [Q23]
4. How do you design a good REST API (status codes, pagination, idempotency)? [Q29 to Q33]
5. SQL or NoSQL: how do you choose? [Q40]
6. What is an index and what does it cost? [Q35]
7. What are the caching patterns and how do you handle invalidation and stampede? [Q47 to Q52]
8. L4 versus L7 load balancing, and common algorithms. [Q54, Q55]
9. Replication versus sharding, and how to pick a shard key. [Q59, Q63]
10. Explain the CAP theorem correctly. [Q66]
11. Kafka-style log versus classic queue, and delivery guarantees. [Q71, Q72]
12. Monolith or microservices, and what is a saga? [Q79, Q80]
13. Timeouts, retries, circuit breaker. [Q83 to Q85]
14. Authentication versus authorization, sessions versus JWT. [Q93, Q94]
15. Estimate QPS and storage for a simple product, and design a rate limiter. [Q97, Q98]

---

## 3. Question bank by topic

### 3.1 OOP and SOLID

**Q1. [Fresher] What are the four pillars of OOP?**
- Encapsulation: hide data inside a class and expose only safe methods.
- Abstraction: show what an object does, hide how it does it.
- Inheritance: a child class reuses a parent class.
- Polymorphism: the same call behaves differently for different types.
- Example: `Shape.area()` works for Circle and Square with different code.

Revise: day-02-oop-solid.md

**Q2. [Fresher] Composition versus inheritance: which is better and why?**
- Inheritance means "is-a". Composition means "has-a".
- Inheritance tightly couples the child to the parent. A parent change can break children.
- Composition lets you swap parts at runtime and is easier to test.
- Rule of thumb: prefer composition. Use inheritance only for a true "is-a" with stable behaviour.

Revise: day-02-oop-solid.md

**Q3. [Fresher] Explain the five SOLID principles in one line each.**
- S: a class has one reason to change.
- O: open for extension, closed for modification.
- L: a subclass must work wherever its parent is expected.
- I: many small interfaces are better than one big one.
- D: depend on abstractions, not on concrete classes.

Revise: day-02-oop-solid.md

**Q4. [Mid] Give an example that violates the Liskov Substitution Principle.**
- Classic case: `Square extends Rectangle`. Setting width also changes height.
- Code that expects a Rectangle with independent sides now gives wrong results.
- Fix: do not model this as inheritance. Use a common `Shape` interface, or make shapes immutable.
- Sign of a violation: a subclass throws "not supported" or breaks the parent's promises.

Revise: day-02-oop-solid.md

**Q5. [Mid] How does Dependency Inversion help testing?**
- A service depends on an interface (for example `PaymentGateway`), not on a real gateway class.
- The real gateway is injected through the constructor (dependency injection).
- In tests, you inject a fake. No network call is needed.
- It also lets you swap vendors without changing business code.

Revise: day-02-oop-solid.md

**Q6. [Senior] When can SOLID hurt a design?**
- Applying every principle early creates many tiny classes and interfaces with one implementation.
- Readers must jump through many files to follow one flow.
- Use SOLID where change is likely. Keep simple code simple.
- Good answer: "I add an abstraction when I see the second real use, not before."

Revise: day-02-oop-solid.md (section "The Dark Side"), day-01-intro-system-design.md

### 3.2 Design patterns

**Q7. [Fresher] What is a design pattern and how is it different from a principle?**
- A principle is general advice, such as "depend on abstractions".
- A pattern is a named, reusable solution to a common design problem, such as Observer.
- Patterns usually apply several principles together.
- Patterns give the team a shared vocabulary.

Revise: day-01-intro-system-design.md

**Q8. [Fresher] Name the three groups of design patterns.**
- Creational: how objects are created (Singleton, Factory, Builder).
- Structural: how objects are combined (Adapter, Decorator, Facade, Proxy).
- Behavioral: how objects talk and share work (Strategy, Observer, State).
- Knowing the group helps you find the right pattern fast.

Revise: day-03-creational-patterns.md, day-04-behavioral-patterns-1.md, day-05-structural-patterns.md

**Q9. [Fresher] What is the Singleton pattern and what are its risks?**
- It ensures one instance of a class and gives global access to it.
- Use it for things like a config holder or a connection manager.
- Risks: hidden global state, hard to test, and thread-safety bugs during creation.
- Safer option: create one instance at startup and pass it in (dependency injection).

Revise: day-03-creational-patterns.md

**Q10. [Mid] Factory Method versus Abstract Factory versus Builder?**
- Factory Method: one method decides which subclass to create.
- Abstract Factory: creates a whole family of related objects (for example a UI theme set).
- Builder: builds one complex object step by step, good when there are many optional fields.
- Use Builder to avoid constructors with ten parameters.

Revise: day-03-creational-patterns.md

**Q11. [Mid] Strategy versus State: both swap behaviour. What is the difference?**
- Strategy: the client picks an algorithm (for example a pricing rule). Strategies do not know each other.
- State: the object changes its own behaviour as its state changes. States often trigger the next state.
- Ask: who decides the switch? Client means Strategy. The object itself means State.

Revise: day-04-behavioral-patterns-1.md

**Q12. [Mid] Explain Observer and one risk of using it.**
- A subject keeps a list of observers and notifies them when something changes.
- Example: an order status change notifies email, SMS, and analytics listeners.
- Risks: memory leaks if observers are never removed, and unclear order of notification.
- At large scale, the same idea becomes a message queue or pub/sub.

Revise: day-04-behavioral-patterns-1.md

**Q13. [Mid] Decorator versus Proxy versus Adapter?**
- Decorator: adds behaviour to an object, same interface (for example compression on a stream).
- Proxy: controls access to an object, same interface (lazy loading, caching, access checks).
- Adapter: converts one interface into another that the client expects (wrapping a vendor SDK).
- Key question: is the goal to add, to control, or to convert?

Revise: day-05-structural-patterns.md

**Q14. [Senior] Which pattern would you use for a payment module that supports many providers, and why?**
- Strategy for the provider choice, so adding a provider needs a new class only.
- Adapter per provider to hide their different SDKs behind one `PaymentGateway` interface.
- Factory to pick the provider from config or region.
- Facade on top gives callers one simple `pay()` call.
- Warn about over-engineering if there is only one provider today.

Revise: day-05-structural-patterns.md, day-03-creational-patterns.md, day-04-behavioral-patterns-1.md

### 3.3 Low-level design (LLD)

**Q15. [Fresher] How do you start an LLD question such as a parking lot?**
- Ask requirements: vehicle types, number of floors, pricing, entry and exit gates.
- List nouns as classes: ParkingLot, Floor, Spot, Vehicle, Ticket.
- List verbs as methods: park, unpark, calculateFee.
- Draw relationships, then code the core flow.

Revise: day-06-lld-parking-lot.md

**Q16. [Mid] Which patterns fit a parking lot design?**
- Strategy for pricing (hourly, flat, weekend) and for spot-allocation rules.
- Factory for creating vehicles or spots by type.
- State for a spot (free, reserved, occupied).
- Singleton (carefully) for the lot itself, or just pass one instance around.

Revise: day-06-lld-parking-lot.md

**Q17. [Mid] How do you handle two cars taking the last spot at the same time?**
- This is a race condition (two threads change shared data at once).
- Make "find and reserve a spot" one atomic step, using a lock or an atomic compare-and-set.
- Lock per floor, not for the whole lot, to keep throughput.
- Test it with many threads in a unit test.

Revise: day-06-lld-parking-lot.md (section "Concurrency")

**Q18. [Senior] How does an elevator controller choose which elevator to send?**
- Model each elevator as a state machine (idle, moving up, moving down, door open).
- Dispatch uses a cost function: distance, direction match, current load.
- Prefer an elevator already moving toward the caller in the same direction.
- Edge cases: overload, emergency stop, many requests at once, door-hold timing.
- Keep the dispatch algorithm behind an interface (Strategy) so it can change.

Revise: day-07-lld-elevator-review.md

### 3.4 Networking

**Q19. [Fresher] What does DNS do?**
- It maps a name like `example.com` to an IP address.
- Lookup goes: browser cache, OS cache, resolver, root, TLD, authoritative server.
- Results are cached based on TTL (time to live).
- DNS can also steer users to the nearest data centre (geo-routing).

Revise: day-08-networking-essentials.md

**Q20. [Fresher] TCP versus UDP?**
- TCP: reliable, ordered, uses a handshake, slower to start. Good for web, payments, files.
- UDP: no guarantees, no handshake, low delay. Good for live video, voice, games, DNS queries.
- If a late packet is worthless, UDP is a good choice.

Revise: day-08-networking-essentials.md

**Q21. [Mid] What is the cost of TLS and how does it get smaller over time?**
- TLS encrypts traffic and proves the server identity with a certificate.
- The handshake adds round trips and CPU work.
- TLS 1.3 reduces the handshake to one round trip, and resumption can reduce it further.
- Connection reuse (keep-alive) spreads the cost over many requests.

Revise: day-08-networking-essentials.md

**Q22. [Mid] HTTP/1.1 versus HTTP/2 versus HTTP/3?**
- HTTP/1.1: one request at a time per connection, so browsers open many connections.
- HTTP/2: many requests share one connection (multiplexing), but TCP packet loss can still stall all of them.
- HTTP/3: runs over QUIC on UDP, so a lost packet blocks only one stream.
- Pick by the client mix and the network quality, usually it is a platform setting.

Revise: day-08-networking-essentials.md

**Q23. [Mid] What happens when you type a URL and press Enter?**
- Browser checks caches, then DNS gives the IP.
- TCP handshake, then TLS handshake.
- HTTP request goes to a CDN or load balancer, then to an app server.
- App may read a cache or database and returns a response.
- Browser renders the page and may load more files.
- Mention where caching and failure can happen at each step.

Revise: day-08-networking-essentials.md (section "The Full Journey of a Request")

**Q24. [Mid] WebSocket versus long polling versus Server-Sent Events?**
- Long polling: client asks, server holds the request until data exists. Simple but wasteful.
- SSE: one-way stream from server to client over HTTP. Good for live feeds.
- WebSocket: two-way, long-lived connection. Good for chat and games.
- WebSocket servers keep state per connection, so scaling needs care (see chat design).

Revise: day-08-networking-essentials.md, day-22-design-chat-app.md

### 3.5 API design

**Q25. [Fresher] What makes a REST API well designed?**
- Nouns for resources (`/orders/42`), verbs come from the HTTP method.
- Correct status codes, such as 201 for created, 404 for not found, 409 for conflict.
- Consistent error body with a code and a message.
- Versioning, pagination, and clear naming.

Revise: day-09-api-design.md

**Q26. [Fresher] Which HTTP methods are idempotent?**
- Idempotent means repeating the call gives the same final state.
- GET, PUT, DELETE are idempotent by definition. POST is not by default.
- Safe means no state change: GET is safe.
- Retry-heavy clients need idempotent endpoints.

Revise: day-09-api-design.md

**Q27. [Mid] How do you make a payment POST safe to retry?**
- Client sends a unique `Idempotency-Key` header with each logical request.
- Server stores the key with the result in the same transaction as the work.
- A repeat with the same key returns the stored result and does not charge again.
- Keys expire after some time, such as 24 hours.

Revise: day-09-api-design.md, day-19-distributed-problems.md

**Q28. [Mid] Offset pagination versus cursor pagination?**
- Offset: `?page=5&size=20`. Simple, but slow on deep pages and rows can shift when data changes.
- Cursor: client sends "give me items after this marker". Stable and fast with an index.
- Cost of cursor: you cannot jump to page 50 directly.
- Use cursor for feeds and large tables.

Revise: day-09-api-design.md

**Q29. [Mid] How do you version an API without breaking clients?**
- Options: version in the URL (`/v2/`), in a header, or in the media type.
- Prefer additive changes: new optional fields do not need a new version.
- Announce deprecation with dates and keep the old version running for a window.
- Track which clients still use the old version before removal.

Revise: day-09-api-design.md

**Q30. [Senior] REST versus gRPC versus GraphQL: how do you choose?**
- REST: simple, cacheable, public APIs, wide tool support.
- gRPC: binary, fast, strong contracts, streaming. Good for internal service calls.
- GraphQL: client chooses fields, fewer round trips for varied clients. Cost: caching and query-cost control are harder.
- Many systems use REST outside and gRPC inside.

Revise: day-09-api-design.md

### 3.6 Databases

**Q31. [Fresher] What is ACID?**
- Atomicity: all steps of a transaction happen or none do.
- Consistency: data rules (constraints) stay valid.
- Isolation: concurrent transactions do not see each other's half-done work.
- Durability: committed data survives a crash.

Revise: day-10-databases.md

**Q32. [Fresher] What is an index and what does it cost?**
- An index (often a B-tree) lets the database find rows without scanning the whole table.
- Reads on the indexed column get much faster.
- Cost: extra disk space and slower writes, because every insert updates the index.
- Index columns used in filters, joins, and sorting. Do not index everything.

Revise: day-10-databases.md

**Q33. [Mid] Name the isolation levels and one problem each prevents.**
- Read uncommitted: allows dirty reads (reading data not yet committed).
- Read committed: prevents dirty reads.
- Repeatable read: the same row reads the same twice in one transaction.
- Serializable: acts as if transactions ran one by one. Safest, slowest.
- Behaviour differs by database product, so check the documentation of the one you use.

Revise: day-10-databases.md

**Q34. [Mid] SQL or NoSQL: how do you choose?**
- SQL: structured data, joins, transactions, flexible queries. Default choice for most products.
- NoSQL: huge scale, simple access by key, flexible schema, or special shapes (graph, time series).
- Ask: what are the access patterns, the consistency need, and the data size?
- Avoid "NoSQL because it scales" without numbers.

Revise: day-10-databases.md

**Q35. [Mid] Name the four NoSQL families with an example use.**
- Key-value: sessions, cache.
- Document: product catalogue with varied fields.
- Wide-column: very high write volume, time-ordered data.
- Graph: relationships such as friends of friends.

Revise: day-10-databases.md

**Q36. [Mid] What is the N+1 query problem?**
- You load a list with one query, then run one extra query per item.
- 100 orders become 101 queries, and each costs a network round trip.
- Fix: join, batch loading (`WHERE id IN (...)`), or eager loading in the ORM.
- Spot it by watching query counts per request.

Revise: day-10-databases.md

**Q37. [Senior] When should you denormalize, and what is the risk?**
- Normalization removes duplicate data and keeps updates simple.
- Denormalize when reads are far more frequent than writes and joins are the bottleneck.
- Example: store the product name inside the order line so history stays correct and reads are fast.
- Risk: copies can become inconsistent, so you need a clear update path.

Revise: day-10-databases.md

### 3.7 Caching

**Q38. [Fresher] Why do we cache?**
- Memory is much faster than disk or network, so reads become quicker.
- It reduces load on the database.
- It saves money at scale.
- Cache data that is read often and changes rarely.

Revise: day-11-caching.md

**Q39. [Mid] Explain the four caching patterns.**
- Cache-aside: app checks cache, on miss reads the database and fills the cache. Most common.
- Read-through: the cache itself loads data on a miss.
- Write-through: write to cache and database together. Safer, slower writes.
- Write-back: write to cache first, database later. Fast, but data can be lost on a crash.

Revise: day-11-caching.md

**Q40. [Mid] LRU versus LFU versus TTL?**
- LRU removes the least recently used item. Good general default.
- LFU removes the least frequently used item. Good when popularity is stable.
- TTL expires items after a set time. It controls staleness, not memory.
- Real caches combine TTL with an eviction policy.

Revise: day-11-caching.md

**Q41. [Mid] How do you keep cache and database consistent?**
- Hard problem: there is no perfect answer, only trade-offs.
- Common approach: update the database, then delete the cache key. The next read refills it.
- Short TTL limits how long stale data lives.
- For critical data (balance, stock), read the database directly.

Revise: day-11-caching.md

**Q42. [Senior] What is a cache stampede and how do you stop it?**
- A popular key expires and thousands of requests hit the database at once.
- Fix 1: lock or "single flight" so only one request rebuilds the value.
- Fix 2: add random jitter to TTLs so keys do not expire together.
- Fix 3: refresh early before expiry, or serve stale data while rebuilding.

Revise: day-11-caching.md

**Q43. [Senior] What are cache penetration and the hot-key problem?**
- Penetration: requests for keys that do not exist always reach the database.
- Fix: cache the "not found" result for a short time, or use a Bloom filter (a compact structure that says "definitely not present").
- Hot key: one key gets a very large share of traffic and overloads one node.
- Fix: local in-process copy, or replicate the key under several names.

Revise: day-11-caching.md

**Q44. [Mid] Redis versus Memcached?**
- Memcached: simple key-value, multi-threaded, good for plain caching.
- Redis: rich data types (lists, sets, sorted sets), persistence options, replication, pub/sub.
- Choose Redis when you need those features, for example leaderboards or rate counters.
- Both are in-memory, so the data size is limited by RAM.

Revise: day-11-caching.md

### 3.8 Load balancing and gateways

**Q45. [Fresher] What does a load balancer do?**
- It spreads requests across many servers.
- It removes unhealthy servers using health checks.
- It lets you add servers without changing clients.
- It can also end TLS and apply simple rules.

Revise: day-12-load-balancing-gateways.md

**Q46. [Mid] L4 versus L7 load balancer?**
- L4 works on TCP/UDP: IP and port only. Very fast and simple.
- L7 reads HTTP: path, headers, cookies. It can route `/api` and `/images` to different pools.
- L7 costs more CPU but gives smart routing and caching.
- Many systems use L4 in front and L7 behind it.

Revise: day-12-load-balancing-gateways.md

**Q47. [Mid] Name common balancing algorithms and when each fits.**
- Round robin: equal servers and equal requests.
- Weighted round robin: servers of different size.
- Least connections: requests with very different durations.
- IP or key hash: same client goes to the same server (sticky).
- Consistent hashing: caches, so adding a node moves few keys.

Revise: day-12-load-balancing-gateways.md

**Q48. [Mid] How is an API gateway different from a plain reverse proxy?**
- A reverse proxy forwards requests and may cache or terminate TLS.
- A gateway adds API concerns: authentication, rate limiting, request shaping, versioned routes, metrics.
- It gives one entry point for many services.
- Risk: it can become a bottleneck or a place where business logic leaks.

Revise: day-12-load-balancing-gateways.md

**Q49. [Senior] Who balances the balancer? How do you avoid a single point of failure?**
- Run at least two load balancers in active-passive or active-active mode.
- Use a shared virtual IP with failover, or DNS and anycast to reach several.
- Health-check the balancers themselves.
- In the cloud, a managed balancer already does this across zones.

Revise: day-12-load-balancing-gateways.md

### 3.9 Replication and sharding

**Q50. [Fresher] Replication versus sharding?**
- Replication: copies of the same data on many nodes. Gives availability and read scale.
- Sharding: splits different data across nodes. Gives write scale and storage scale.
- They solve different problems and are often used together.

Revise: day-13-replication-sharding.md

**Q51. [Mid] What breaks when replicas lag behind the primary?**
- A user writes, then reads from a replica and does not see their own data.
- Fixes: read-your-writes (read from the primary for a short time after a write), or sticky routing.
- Monotonic reads: do not let the user see newer data then older data.
- Monitor lag and remove very slow replicas from the read pool.

Revise: day-13-replication-sharding.md

**Q52. [Senior] What is split-brain and how do you prevent it?**
- Two nodes both believe they are the primary and accept writes. Data diverges.
- It can happen when a network split hides the real primary.
- Prevent with a quorum (majority vote) for leader election and with fencing (the old leader is blocked from writing).
- Prefer fewer, well-tested automatic failover rules.

Revise: day-13-replication-sharding.md, day-19-distributed-problems.md

**Q53. [Mid] Sharding strategies: range, hash, directory?**
- Range: easy range queries, but hot spots (for example latest dates).
- Hash: even spread, but range queries touch all shards.
- Directory: a lookup table maps key to shard. Flexible, but the table is a dependency.
- Consistent hashing reduces data movement when shards change.

Revise: day-13-replication-sharding.md

**Q54. [Senior] How do you choose a shard key?**
- High cardinality (many distinct values) so data spreads out.
- Matches the main query, so most requests hit one shard.
- Avoids hot keys and growth skew.
- Hard to change later, so test the choice against real access patterns.
- Example: orders by `customer_id` fits "my orders", but is poor for "all orders in a city".

Revise: day-13-replication-sharding.md

**Q55. [Senior] How do you handle cross-shard queries, transactions, and hot shards?**
- Cross-shard joins: avoid by design, or merge results in the app, or use a search or analytics store.
- Cross-shard transactions: use sagas or redesign so one shard owns the whole unit of work.
- Hot shard (for example a celebrity user): split that key further, add cache, or give it a dedicated shard.
- Resharding without downtime: dual-write, backfill, verify, then switch reads.

Revise: day-13-replication-sharding.md

### 3.10 CAP and consistency

**Q56. [Mid] Explain the CAP theorem correctly.**
- It says: during a network partition, a system must choose between consistency and availability.
- Partition tolerance is not optional in a distributed system, because networks do fail.
- So the real choice is CP (reject some requests to stay correct) or AP (answer, maybe with stale data).
- Without a partition, you can have both.

Revise: day-14-cap-consistency-url-shortener.md

**Q57. [Senior] What does PACELC add to CAP?**
- If there is a Partition, choose Availability or Consistency (the CAP part).
- Else, when running normally, choose Latency or Consistency.
- It explains why a system with no failure still trades speed for strong consistency.
- Example: waiting for a majority of replicas adds delay.

Revise: day-14-cap-consistency-url-shortener.md

**Q58. [Mid] List consistency levels from strong to weak.**
- Strong (linearizable): every read sees the latest write.
- Sequential or causal: order is kept where cause and effect matter.
- Read-your-writes: you see your own updates.
- Eventual: replicas agree after some time if writes stop.
- Choose per feature: a bank balance needs strong, a like counter can be eventual.

Revise: day-14-cap-consistency-url-shortener.md

**Q59. [Senior] How does quorum work with N, R, W?**
- N replicas hold the data. A write waits for W acknowledgments. A read asks R replicas.
- If R + W > N, a read overlaps with the latest write, so it can see it.
- Example: N=3, W=2, R=2 gives overlap.
- Lower W or R gives speed but weaker guarantees.

Revise: day-14-cap-consistency-url-shortener.md

**Q60. [Mid] How do you generate short IDs in a URL shortener?**
- Option 1: counter converted to base62 (digits, lower and upper letters). No collisions, but needs a safe counter.
- Option 2: hash of the URL, take a prefix, check collisions.
- Option 3: pre-generated key ranges handed to each app server.
- Seven base62 characters give about 3.5 trillion keys (62 to the power 7).
- Use a 301 or 302 redirect depending on whether you need analytics.

Revise: day-14-cap-consistency-url-shortener.md

### 3.11 Message queues

**Q61. [Fresher] Why use a message queue?**
- It decouples producer and consumer, so they can fail and scale separately.
- It absorbs traffic spikes (buffering).
- It allows retries and background work.
- Cost: extra system to run, and the result is no longer immediate.

Revise: day-15-message-queues.md

**Q62. [Mid] Queue versus pub/sub?**
- Queue: each message goes to one consumer in a group. Good for sharing work.
- Pub/sub: each message goes to every subscriber. Good for broadcasting events.
- Log-based systems support both: each consumer group gets all messages, and consumers within a group share them.

Revise: day-15-message-queues.md

**Q63. [Mid] Kafka-style log versus RabbitMQ-style broker?**
- Log: messages stay on disk in ordered partitions. Consumers track an offset. Replay is possible. Very high throughput.
- Broker: smart routing, messages removed after acknowledgment, flexible patterns.
- Pick the log for event streams and replay. Pick the broker for task queues and complex routing.
- Ordering in a log is guaranteed only within one partition.

Revise: day-15-message-queues.md

**Q64. [Mid] Explain at-most-once, at-least-once, and exactly-once delivery.**
- At-most-once: may lose messages, never duplicates.
- At-least-once: never loses, may duplicate. Most common.
- Exactly-once end to end is very hard. In practice, use at-least-once plus idempotent consumers.
- Some platforms offer exactly-once inside their own boundary only.

Revise: day-15-message-queues.md

**Q65. [Senior] What is an idempotent consumer and why do you need it?**
- Duplicates will happen because of retries and rebalances.
- Store a processed message ID, or use natural idempotency (set status to PAID, not add 100).
- Check and apply the effect in one transaction.
- Result: processing a message twice has the same effect as once.

Revise: day-15-message-queues.md

**Q66. [Senior] What are poison messages, dead letter queues, and the outbox pattern?**
- Poison message: always fails, blocks the queue. Move it to a dead letter queue (DLQ) after N retries, and alert.
- Outbox: write the event to an outbox table in the same database transaction as the data change. A relay publishes it later.
- This avoids the "saved to database but event was lost" problem (dual write).
- Watch consumer lag as an early warning.

Revise: day-15-message-queues.md

### 3.12 Event-driven architecture

**Q67. [Mid] Events versus commands?**
- Command: "do this" sent to one receiver (`ChargeCard`). The sender expects an action.
- Event: "this happened" (`OrderPlaced`). The sender does not know who listens.
- Events reduce coupling. Commands make flow easier to follow.

Revise: day-16-event-driven-architecture.md

**Q68. [Senior] Choreography versus orchestration?**
- Choreography: services react to each other's events. No central owner. Loose coupling, hard to see the whole flow.
- Orchestration: a central coordinator calls each step. Easy to see and change the flow, but the coordinator is a dependency.
- For long business flows with money, orchestration is often easier to reason about.

Revise: day-16-event-driven-architecture.md

**Q69. [Senior] What are CQRS and event sourcing?**
- CQRS: separate models for writes and reads, so each can be shaped and scaled on its own.
- Event sourcing: store the list of events as the source of truth, and rebuild state by replaying them.
- Benefits: audit trail, replay. Costs: complexity, eventual consistency, schema evolution of events.
- Use only where the benefit is clear.

Revise: day-16-event-driven-architecture.md

### 3.13 Microservices

**Q70. [Mid] Monolith or microservices: how do you decide?**
- Start with a well-structured monolith for small teams and unclear boundaries.
- Real reasons to split: independent scaling, independent deploys by many teams, fault isolation.
- Weak reasons: "it is modern" or "other companies do it".
- Microservices add network calls, deployment, tracing, and data consistency costs.

Revise: day-17-microservices-vs-monolith.md

**Q71. [Mid] How do you draw service boundaries?**
- Follow business capabilities (orders, payments, inventory), not technical layers.
- Each service owns its data. No shared database.
- If two services always change together, they may be one service.
- Check the team layout: one team should own one or few services.

Revise: day-17-microservices-vs-monolith.md

**Q72. [Senior] What is a saga and how is it different from two-phase commit (2PC)?**
- 2PC: a coordinator locks resources on all participants and commits together. Strong, but blocking and fragile.
- Saga: a series of local transactions. If one fails, run compensating actions (for example refund).
- Sagas are eventually consistent and need idempotent, retry-safe steps.
- Most microservice systems use sagas.

Revise: day-17-microservices-vs-monolith.md

**Q73. [Senior] What is a distributed monolith and how do you migrate safely?**
- Services that must deploy together or share one database. You pay microservice costs and get no benefit.
- Migrate with the strangler fig pattern: put a router in front, move one feature at a time to a new service, and retire the old code slowly.
- Keep rollback possible at each step.

Revise: day-17-microservices-vs-monolith.md

### 3.14 Fault tolerance

**Q74. [Mid] Why must every network call have a timeout?**
- Without a timeout, a slow dependency holds threads and connections forever.
- Soon your service is also stuck, and failure spreads (cascading failure).
- Set timeouts from the real latency data, for example slightly above the 99th percentile.
- Total budget of the whole request must be larger than the sum of nested calls.

Revise: day-18-fault-tolerance.md

**Q75. [Mid] How do you do retries correctly?**
- Retry only safe or idempotent operations, and only for transient errors.
- Use exponential backoff (wait longer each time) with jitter (random spread).
- Limit the number of attempts and total time.
- Otherwise retries cause a retry storm that kills a recovering service.

Revise: day-18-fault-tolerance.md

**Q76. [Mid] How does a circuit breaker work?**
- Closed: calls pass through. If failures pass a threshold, it opens.
- Open: calls fail fast without touching the sick service.
- Half-open: after a wait, a few test calls decide whether to close again.
- Combine with a fallback, such as cached data or a simpler result.

Revise: day-18-fault-tolerance.md

**Q77. [Senior] What are bulkheads, load shedding, and graceful degradation?**
- Bulkhead: separate pools of threads or connections per dependency, so one failure does not use all resources.
- Load shedding: when overloaded, reject low-priority requests early to protect the core.
- Graceful degradation: turn off non-core features (recommendations) but keep the main path (checkout) working.

Revise: day-18-fault-tolerance.md

**Q78. [Mid] Liveness versus readiness checks?**
- Liveness: is the process alive? If not, restart it.
- Readiness: can it serve traffic now (database connected, cache warm)? If not, remove it from the balancer but do not restart.
- Mixing them up can cause restart loops during a dependency outage.

Revise: day-18-fault-tolerance.md

### 3.15 Distributed problems

**Q79. [Senior] Why are distributed locks tricky?**
- A lock holder can pause (garbage collection) or crash, and the lock expires while it still thinks it holds it.
- Use a lease (lock with expiry) plus a fencing token (a rising number checked by the storage).
- Prefer to avoid locks: use database constraints, atomic operations, or single-writer queues.
- Example: "exactly one rider gets the order" can use a conditional update in one row.

Revise: day-19-distributed-problems.md

**Q80. [Senior] What is consensus and where do you meet it?**
- Consensus means several nodes agree on one value even if some fail (algorithms such as Raft and Paxos).
- It needs a majority, so 3 nodes tolerate 1 failure and 5 tolerate 2.
- You usually use it through a coordination service or a database, not write your own.
- Used for leader election, configuration, and locks.

Revise: day-19-distributed-problems.md

**Q81. [Senior] Why can you not trust clocks in distributed systems?**
- Machine clocks drift and are never perfectly in sync.
- Two events on different machines cannot be ordered by timestamps alone.
- Use logical clocks or version counters when order matters.
- Use monotonic time for measuring durations on one machine.

Revise: day-19-distributed-problems.md

**Q82. [Senior] Design idempotency for an operation across services.**
- Create a unique request ID at the edge and pass it through every call.
- Each service stores the ID and result with its state change in one transaction.
- On a repeat, return the stored result.
- Store the keys with a time limit and clean them up.

Revise: day-19-distributed-problems.md, day-09-api-design.md

### 3.16 Observability and security

**Q83. [Fresher] What are the three pillars of observability?**
- Logs: detailed events, best with structure (JSON) and a request ID.
- Metrics: numbers over time (rate, errors, latency) for dashboards and alerts.
- Traces: the path of one request across services, with timing per step.
- Use metrics to detect, traces to locate, logs to explain.

Revise: day-20-observability-security.md

**Q84. [Mid] What are SLI, SLO, and error budget?**
- SLI: a measured indicator, such as the share of requests under 300 ms.
- SLO: the target for it, such as 99.9% over 30 days.
- Error budget: the allowed failure (0.1%). If you spend it, slow down releases and fix reliability.
- Alert on user-visible symptoms and budget burn, not on every CPU spike.

Revise: day-20-observability-security.md

**Q85. [Fresher] Authentication versus authorization?**
- Authentication: who are you? (login, token)
- Authorization: what may you do? (roles, permissions)
- Always check both on the server. Never trust the client.

Revise: day-20-observability-security.md

**Q86. [Mid] Sessions versus JWT?**
- Session: server stores state and the client holds an ID. Easy to revoke, needs shared storage.
- JWT (JSON Web Token): a signed token holding claims. No lookup needed, but hard to revoke before expiry.
- Use short-lived JWTs with refresh tokens, or sessions when instant revoke matters.
- Never store secrets in a JWT payload: it is encoded, not encrypted.

Revise: day-20-observability-security.md

**Q87. [Mid] Name common backend security risks and a fix for each.**
- SQL injection: use parameterized queries.
- Broken access control: check authorization on every object, not only on login.
- Leaked secrets: use a secret manager, never commit keys.
- Weak transport: TLS everywhere; hash passwords with a slow, salted algorithm.
- Abuse: rate limiting.

Revise: day-20-observability-security.md

### 3.17 Estimation and rate limiting

**Q88. [Fresher] Estimate QPS for 10 million requests per day.**
- 10M / 86,400 seconds is about 115 per second, call it 100 to 120.
- Apply a peak factor of 2 to 3, so about 300 per second at peak.
- Compare with one node capacity (a simple app server handles around 1k requests per second, roughly).
- Say your assumptions out loud.

Revise: day-21-estimation-rate-limiter.md

**Q89. [Mid] Estimate yearly photo storage for 10 million uploads per day.**
- Assume an average of 500 KB per photo after compression.
- 10M x 500 KB = 5 TB per day. Times 365 is about 1.8 PB per year.
- Multiply by 3 for replication, so about 5.5 PB. Add thumbnails and headroom.
- Conclusion: use object storage and a CDN, not a database.

Revise: day-21-estimation-rate-limiter.md

**Q90. [Mid] Compare rate limiting algorithms.**
- Fixed window: simple counter per time window. Allows bursts at window edges.
- Sliding window: smoother and more accurate, uses more memory.
- Token bucket: tokens refill at a steady rate, allows small bursts. Very common.
- Leaky bucket: output is smooth at a fixed rate.
- Return 429 and a `Retry-After` header.

Revise: day-21-estimation-rate-limiter.md, day-12-load-balancing-gateways.md

**Q91. [Senior] How do you make a rate limiter work across many servers?**
- Keep counters in a shared fast store, such as Redis, keyed by user or IP.
- Make "check and decrement" atomic (a script or atomic command) to avoid races.
- For lower latency, allow local approximate counting and sync every few seconds.
- Decide fail-open or fail-closed if the store is down. Often fail-open for normal APIs.

Revise: day-21-estimation-rate-limiter.md

### 3.18 Case-study deep questions

**Q92. [Senior] Chat app: how do you deliver a message to an offline user and show receipts?**
- Store the message first, then try live delivery over the user's WebSocket connection.
- If offline, keep it as pending and send a push notification.
- Receipt states: sent, delivered, read. Each is an acknowledgment event.
- Messages get per-conversation sequence numbers for ordering and for sync after reconnect.

Revise: day-22-design-chat-app.md

**Q93. [Senior] News feed: fan-out on write versus fan-out on read?**
- Fan-out on write: when a user posts, push the post ID into each follower's feed. Reads are fast, writes are heavy.
- Fan-out on read: build the feed at read time. Writes are cheap, reads are slow.
- Hybrid: push for normal users, pull for celebrities with millions of followers, then merge.

Revise: day-23-design-news-feed.md

**Q94. [Senior] Video platform: how does upload become smooth playback?**
- Upload to object storage (in chunks, resumable).
- A queue triggers transcoding into several resolutions and formats.
- Video is split into small segments for adaptive bitrate streaming, so the player switches quality by network speed.
- A CDN serves the segments from nearby locations. Metadata lives in a database.

Revise: day-24-design-video-platform.md

**Q95. [Senior] Ride-hailing: how do you find nearby drivers fast?**
- Drivers send location updates every few seconds. Store latest position in memory, indexed by a geo cell (geohash or similar grid).
- Query the rider's cell and neighbours, then rank by distance and ETA.
- Offer the trip to one driver at a time with a timeout, and make the assignment atomic so two riders never get one driver.

Revise: day-25-design-ride-hailing.md

**Q96. [Senior] Notifications: how do you avoid sending twice and handle provider failures?**
- API accepts the request and puts it on a queue, then returns quickly.
- Workers per channel (push, SMS, email) send with retries and backoff.
- Idempotency key per notification prevents duplicates.
- Fall back to another channel or provider, keep a DLQ, respect user preferences and rate limits.

Revise: day-26-design-notification-system.md

**Q97. [Senior] E-commerce checkout: how do you avoid overselling and double charging?**
- Reserve stock with a short-lived hold at checkout, using an atomic conditional update.
- Payment call uses an idempotency key.
- Order flow is a saga: reserve, pay, confirm, with compensation (release stock, refund) on failure.
- Reconcile with the payment provider regularly to catch gaps.

Revise: day-27-design-ecommerce-checkout.md

**Q98. [Senior] Distributed cache: how do you spread keys and survive node loss?**
- Use consistent hashing with virtual nodes so adding or removing a node moves few keys.
- Replicate each key to one or more other nodes for failover.
- Use LRU eviction and TTL; protect against hot keys and stampede.
- Clients or a proxy find the right node. Expect a cold cache after a node is replaced.

Revise: day-28-design-distributed-cache.md

**Q99. [Mid] LRU cache: which data structure and why?**
- A hash map plus a doubly linked list.
- The map gives O(1) lookup. The list keeps usage order, with the most recent at the head.
- On get or put, move the node to the head. On overflow, remove the tail.
- Add a lock for thread safety.

Revise: day-29-lld-revision.md

**Q100. [Fresher] How do you design a vending machine in LLD?**
- Model it as a state machine: idle, has money, dispensing, out of service.
- Use the State pattern: each state class handles insert coin, select item, and cancel.
- Handle edge cases: not enough money, sold out, change not available.
- Keep inventory and payment in separate classes.

Revise: day-29-lld-revision.md

---

## 4. Case-study prompts

For each prompt, a strong answer covers these points. Practice with the 45-minute plan in section 1.

### 4.1 Design a URL shortener
- Requirements: create short link, redirect, optional expiry, analytics. Read-heavy, around 100 reads per write.
- Estimate: writes per second, total links over five years, storage per record.
- API: `POST /links`, `GET /{code}` with a redirect response.
- Key generation: base62 counter, hash, or pre-allocated ranges. Discuss collisions.
- Storage: simple key-value or SQL table by short code.
- Cache hot links in memory; use a CDN or edge for redirects.
- Availability over strict consistency for redirects.
- Extras: abuse protection, rate limits, analytics through a queue.
- Revise: day-14-cap-consistency-url-shortener.md

### 4.2 Design a chat application
- Requirements: one-to-one and group chat, online status, delivery receipts, history, media.
- Estimate: messages per day, connections held at once, bandwidth.
- Connection: WebSocket gateway servers; a mapping of user to gateway server.
- Message path: store first, then route to the receiver's gateway; offline users get push.
- Data model: messages by conversation with a sequence number; wide-column or sharded store.
- Ordering and sync after reconnect; idempotent send with client message ID.
- Group fan-out and presence at scale; limits on group size.
- Failure: gateway dies, clients reconnect and resync from the last sequence.
- Revise: day-22-design-chat-app.md

### 4.3 Design a news feed
- Requirements: post, follow, see a ranked or time-ordered feed, low read latency.
- Estimate: users, follows per user, read to write ratio (very read-heavy).
- Fan-out on write versus read; hybrid for celebrities.
- Feed store: per-user list of post IDs in a cache, with post content fetched separately.
- Ranking: simple recency first, then signals; mention it as a later layer.
- Pagination with a cursor, not offset.
- Failure and scale: backfill when a user follows someone new; cache loss rebuilds from the database.
- Revise: day-23-design-news-feed.md

### 4.4 Design a video streaming platform
- Requirements: upload, process, stream to many devices, search, recommendations (out of scope or brief).
- Estimate: upload volume, storage after multiple resolutions, egress bandwidth (the dominant cost).
- Upload to object storage in resumable chunks.
- Transcoding pipeline through a queue with parallel workers.
- Adaptive bitrate streaming with segmented files.
- CDN for delivery; popular content at the edge.
- Metadata store, view counts through an event stream (approximate is fine).
- Failure: retry failed transcode jobs; keep original file.
- Revise: day-24-design-video-platform.md

### 4.5 Design a ride-hailing service
- Requirements: request ride, match a driver, live tracking, fare, payment, rating.
- Estimate: drivers online, location updates per second (the main write load).
- Location store: in-memory geo index by cell; update frequency trade-off.
- Matching: nearby search, ranking by ETA, offer one driver at a time with timeout.
- Atomic assignment so one driver gets one trip.
- Trip state machine: requested, matched, started, completed, cancelled.
- Real-time updates to rider and driver via WebSocket or push.
- Surge pricing and failure of a region as extra topics.
- Revise: day-25-design-ride-hailing.md

### 4.6 Design a notification system
- Requirements: push, SMS, email; transactional (OTP) and marketing; user preferences.
- Estimate: peak per second, especially for campaigns; provider limits.
- API accepts a request and returns fast; queue per channel and priority.
- Separate high-priority (OTP) traffic from bulk traffic.
- Workers with retries, backoff, DLQ, provider fallback.
- Idempotency key and deduplication; user opt-out and quiet hours.
- Templates and localization; delivery status tracking.
- Revise: day-26-design-notification-system.md

### 4.7 Design an e-commerce checkout
- Requirements: cart, inventory check, payment, order creation, no overselling, no double charge.
- Correctness matters more than speed: consistency over availability for stock and money.
- Estimate: peak orders per second during sales.
- Stock hold with expiry; atomic update.
- Payment with idempotency key; handle timeouts with status lookups.
- Saga with compensation; outbox for reliable events.
- Reconciliation job with the payment provider.
- Graceful degradation: turn off recommendations, keep the checkout path.
- Revise: day-27-design-ecommerce-checkout.md

### 4.8 Design a distributed cache
- Requirements: get, set, delete, TTL; low latency; high availability; scale out.
- Estimate: key count, value size, memory per node, operations per second.
- Partitioning with consistent hashing and virtual nodes.
- Replication for failover; decide leader or leaderless.
- Eviction (LRU), expiry, memory limits.
- Hot keys, stampede, and cold start after node loss.
- Client library versus proxy for routing; membership and failure detection.
- Revise: day-28-design-distributed-cache.md

### 4.9 Design a rate limiter
- Requirements: limit per user, IP, or API key; low added latency; works across servers.
- Algorithm choice: token bucket or sliding window, with the trade-offs.
- Where it runs: gateway, middleware, or a separate service.
- Shared counters in a fast store with atomic operations.
- Response: 429 with `Retry-After` and limit headers.
- Fail-open or fail-closed decision; rule configuration without redeploy.
- Edge cases: clock differences, bursts, distributed race conditions.
- Revise: day-21-estimation-rate-limiter.md, day-12-load-balancing-gateways.md

### 4.10 Design a parking lot (LLD)
- Ask: vehicle types, floors, pricing, payment, entry and exit gates.
- Core classes: ParkingLot, Floor, Spot, Vehicle, Ticket, Payment.
- Patterns: Strategy for pricing and spot choice, Factory for vehicles, State for spots.
- Flow: enter, find spot, issue ticket, exit, compute fee, pay.
- Concurrency: atomic spot reservation.
- Edge cases: lost ticket, full lot, wrong spot type.
- SOLID check at the end.
- Revise: day-06-lld-parking-lot.md

### 4.11 Design an elevator system (LLD)
- Ask: number of elevators and floors, peak behaviour, priority rules.
- Core classes: Elevator, Controller or Dispatcher, Request, Door, Display.
- Elevator state machine and direction handling.
- Dispatch algorithm behind a Strategy interface; cost function.
- Request queues up and down; stop at floors on the way.
- Edge cases: overload, emergency, simultaneous calls.
- Revise: day-07-lld-elevator-review.md

### 4.12 Design a city parking-availability system (HLD)
- Requirements: show free spots near a place in near real time; sensors or gate counts; many cities.
- Estimate: lots, sensors, update rate, read queries per second.
- Ingest events from gates and sensors through a queue.
- Keep the current count per lot in a cache; durable history in a database.
- Geo search for nearby lots; cache by map cell.
- Accept slight staleness; show "last updated" time.
- Handle sensor failure, duplicate and out-of-order events.
- Revise: day-30-mock-interview-day.md

---

## 5. Common mistakes in design interviews

1. **Starting to draw before asking questions.** You may solve a problem nobody asked about.
2. **Skipping numbers.** Without estimates, you cannot justify sharding, caching, or a queue.
3. **Naming tools without reasons.** Say why you pick a database or queue, and what it costs.
4. **Using every buzzword.** Microservices, Kafka, and sharding for 100 requests per second looks like poor judgement.
5. **Ignoring failure.** Always say what happens when a node, network, or dependency fails.
6. **Forgetting idempotency** in payments, retries, and queue consumers.
7. **Mixing up CAP terms.** Say "during a partition" and name the choice per feature.
8. **One consistency level for everything.** Strong for money and stock, eventual for counters and feeds.
9. **No deep dive.** A wide but shallow tour scores lower than two well-explained hard parts.
10. **Silent thinking.** Think aloud so the interviewer can guide you.
11. **Not listening to hints.** A question from the interviewer is often a hint to change direction.
12. **Poor time control.** Do not spend 25 minutes on requirements or end in the middle of a sentence.
13. **Single points of failure left unmentioned** (one load balancer, one database, one queue).
14. **In LLD: God classes and no patterns reasoning.** Keep classes small and explain why a pattern fits.
15. **No summary.** Finish with the design in three sentences and the top two risks.

Practice tip: record yourself doing one case study per week and compare it with the checklist in section 1. This improves your preparation, but no guide can guarantee an interview result.
