# Redis for Backend Engineers

Accuracy status: Reviewed against official documentation on 2026-11-06; items still unverified are marked.

A working guide to Redis for people who build and operate backend services. It teaches how Redis behaves and where it fails, so you can choose the right data structure, design a cache that survives load, and diagnose Redis in production, not just recall commands.

**Audience:** backend developers who have used Redis as a cache or queue and want production-level judgement.
**Prerequisites:** basic command-line comfort, TCP and HTTP basics, some Node.js. Read the Linux chapter on memory and processes first (`03_Linux.md`, chapters 4 and 6).
**Scope:** Redis as a server-side data store: data structures, caching, coordination, messaging, persistence, high availability, Cluster, operations and client usage from Node with ioredis. Redis modules and managed-service specifics are out of scope.
**Note on accuracy:** the data model and command semantics described here are long-stable. Defaults, config names and newer commands change between versions, and Redis has had forks and licensing changes (Valkey is a fork that keeps the same protocol and core commands). Where behaviour is version-dependent this guide says so, and you should confirm in the documentation for the version you run (`INFO server` shows it, and `COMMAND DOCS <name>` or the commands reference describes each command).

## How to use this guide

Each chapter has four parts:

1. **Understand**: the concepts and the reasoning behind them.
2. **Tools to master**: what to learn to use fluently, and the level to reach.
3. **Common mistakes**: what goes wrong in practice.
4. **Self-check**: questions you should be able to answer without notes.

Chapter 18 lists hands-on labs. Chapter 19 collects interview questions with model answers for every question.

**What "good enough" means for a backend role:** you can pick the right structure for a problem, build a cache with correct expiry and stampede protection, implement a rate limiter and a lock while knowing their limits, explain what persistence and replication do and do not guarantee, and find out why Redis is slow. Chapters 1, 2, 5, 6, 7, 11 and 14 matter most.

## Contents

1. The mental model: why Redis is fast, and what that costs
2. Data types and their real uses
3. Key design, TTL and expiration
4. Memory management and eviction
5. Caching patterns
6. Cache failure modes and invalidation
7. Atomicity: Lua, transactions and pipelining
8. Rate limiting
9. Distributed locks
10. Pub/sub, streams and job queues
11. Persistence
12. Replication, Sentinel and failover
13. Redis Cluster
14. Operating Redis: dangerous commands and monitoring
15. Security
16. Using Redis from Node with ioredis
17. When not to use Redis
18. Practice labs
19. Interview questions
20. References
21. Glossary

---

## 1. The mental model: why Redis is fast, and what that costs

**In plain words.** Redis keeps its data in memory, so it answers very quickly. It runs commands one by one on a single main thread (a thread is one line of work inside a program). This is simple and fast, but one slow command makes every other client wait.

### Understand

**Redis is a data structure server.** It holds typed values (strings, hashes, lists, sets, sorted sets and more) in memory and exposes operations on those structures over a simple text protocol (RESP) on TCP. You do not fetch a value, change it in your application and write it back. You ask the server to change it in place (`HINCRBY`, `ZADD`, `LPUSH`). This is the main reason Redis is useful for coordination and counting, not only for caching.

**Why it is fast.**
- **Data lives in RAM.** A lookup is a hash table access in memory, microseconds rather than a disk seek. Disk is used only for persistence (chapter 11), not for serving reads.
- **Efficient structures.** Each type has compact encodings for small values and tuned structures for large ones. `OBJECT ENCODING <key>` shows which is in use.
- **No locking inside the command path.** Commands execute one at a time on a single thread, so there are no mutexes around data and no context-switch overhead between concurrent commands.
- **Event loop with I/O multiplexing.** One thread watches thousands of sockets using the OS readiness mechanism (`epoll` on Linux), reads whichever have data, runs the commands, and writes replies. This is the same model as the Node.js event loop. A connection costs a file descriptor and a small buffer, not a thread.

**Single-threaded command execution, and what it implies.** Command execution is serialised: while one command runs, no other client's command runs. Newer versions can use extra threads for socket reads and writes (the `io-threads` setting, off by default). Background threads handle some work such as closing files, lazy freeing and fsync. The commands themselves still run one at a time on the main thread. The official benchmark page confirms that modern Redis uses threads for different things. The exact io-threads behaviour is unverified here, so check the documentation for your version. Consequences you must design around:

1. **One slow command blocks everyone.** A `KEYS *` over ten million keys, an `SMEMBERS` on a set with millions of members, `LRANGE 0 -1` on a huge list or a heavy Lua script stalls every other client for its whole duration. Redis latency is shared, so a single careless caller degrades the whole service.
2. **Every command is atomic.** Because nothing interleaves, a single command (or a Lua script, or a `MULTI`/`EXEC` block) is isolated from other clients for free. This is what makes `INCR`, `SET NX` and scripts safe to use for coordination.
3. **Throughput is limited by one core for command execution.** A single instance serves very high request rates, often in the hundreds of thousands per second for simple commands, but it does not scale CPU across cores. To scale writes you shard (Cluster, chapter 13) or run several instances.
4. **Network round trips dominate.** A command takes microseconds on the server and hundreds of microseconds to milliseconds on the network. Issuing 1000 sequential commands costs 1000 round trips. Pipelining (chapter 7) fixes this.
5. **Big values hurt.** Sending a 50 MB value occupies the thread for the copy and the network for the transfer, and delays other clients.

**Redis is not a database replacement by default.** It can persist and replicate, but its guarantees are weaker and its data must fit in memory (chapter 17).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Talking to a server | `redis-cli`, `redis-cli -h host -p port`, `redis-cli --tls`, `PING`, `INFO server` | Connect, identify version and mode, run any command |
| Seeing real latency | `redis-cli --latency`, `--latency-history`, `--intrinsic-latency <seconds>` | Separate network latency from server latency |
| Throughput and capacity | `redis-benchmark -c 50 -n 100000 -t set,get -P 16 -d 256` | Measure your instance with realistic payload size and pipelining |
| Inspecting structures | `TYPE`, `OBJECT ENCODING`, `MEMORY USAGE <key>`, `DEBUG` (avoid in production) | See how a key is stored and what it costs |
| Server facts | `INFO`, `CONFIG GET`, `COMMAND DOCS` | Read configuration and command complexity from the server itself |

### Common mistakes

- Believing "Redis is fast" means any command is safe. Speed is per command on small data. Complexity per command still applies (chapter 2).
- Benchmarking from the same host with no pipelining and concluding Redis is the bottleneck, when the client is.
- Treating Redis as multi-core and putting heavy Lua or large sorts on the primary during peak traffic.

### Self-check

- Why can a single-threaded server handle tens of thousands of concurrent clients?
- Why is `INCR` safe from two clients at the same moment with no lock?
- What happens to every other client when one client runs `KEYS *` on a large keyspace?

---

## 2. Data types and their real uses

**In plain words.** Redis has several data types, and each one fits a different job. Choose the type first, because it decides memory use and speed. Complexity (how the cost of a command grows as the data grows) is written O(1), O(log N) or O(N).

### Understand

Choosing the right type is the central Redis skill. The type determines memory use, command complexity and what can be done atomically. Complexity below uses N for the size of the collection and M for the number of items returned or touched. Verify exact complexity on each command's page, which lists it.

| Type | Mental model | Typical use | Key commands | Notes on complexity |
|---|---|---|---|---|
| String | Byte blob, up to 512 MB | Cache values (serialised JSON), counters, flags, locks, session tokens | `GET`, `SET`, `MSET`, `INCR`, `INCRBY`, `SET ... NX`, `GETEX` | `GET`/`SET`/`INCR` are O(1). Keep values small. |
| Hash | Map of field to value inside one key | An object with several fields (user profile, cart), partial updates | `HSET`, `HGET`, `HMGET`, `HINCRBY`, `HGETALL`, `HSCAN` | `HGET`/`HSET` O(1). `HGETALL` is O(N) in fields. |
| List | Ordered sequence, fast at both ends | Simple queues, recent-items feeds, work hand-off | `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `BLPOP`, `LMOVE`, `LRANGE`, `LTRIM` | Push/pop at the ends O(1). `LRANGE` is O(S+M). `LINDEX` in the middle is O(N). |
| Set | Unordered unique members | Tags, unique visitors, membership checks, set algebra | `SADD`, `SISMEMBER`, `SREM`, `SINTER`, `SUNION`, `SCARD`, `SSCAN` | `SADD`/`SISMEMBER` O(1). `SMEMBERS` is O(N). `SINTER` cost depends on set sizes. |
| Sorted set (ZSET) | Unique members, each with a numeric score, kept ordered | Leaderboards, priority queues, time-ordered indexes, sliding-window rate limits, delayed jobs | `ZADD`, `ZINCRBY`, `ZRANGE` (with `BYSCORE`, `REV`, `LIMIT` in newer versions), `ZRANK`, `ZREM`, `ZREMRANGEBYSCORE`, `ZCARD` | `ZADD`/`ZREM` O(log N). Range queries O(log N + M). |
| Bitmap | Bit operations on a string | Compact per-id boolean flags (daily active users by numeric id) | `SETBIT`, `GETBIT`, `BITCOUNT`, `BITOP`, `BITPOS` | `SETBIT`/`GETBIT` O(1). `BITCOUNT` is O(N) in bytes scanned. Sparse high ids waste memory. |
| HyperLogLog | Probabilistic distinct counter | Approximate unique counts at fixed small memory (unique visitors) | `PFADD`, `PFCOUNT`, `PFMERGE` | Fixed memory of about 12 KB per key. Standard error is about 0.81 percent. Cannot list members. |
| Stream | Append-only log of entries with IDs, with consumer groups | Durable-ish event log, work queues with acknowledgement, event fan-out | `XADD`, `XREAD`, `XGROUP CREATE`, `XREADGROUP`, `XACK`, `XPENDING`, `XAUTOCLAIM`, `XTRIM` | `XADD` O(1). Reads by ID range are O(log N + M). |
| Geo | Locations stored as a sorted set with a geohash score | "Nearby" lookups (drivers, stores) | `GEOADD`, `GEOSEARCH`, `GEODIST` | Built on sorted sets. `GEOSEARCH` supersedes the older `GEORADIUS` in recent versions. |

**Reading the table.** Three questions pick the type: what is the access pattern (point lookup, range, membership, ordering), how large can the collection get, and do I need atomic compound updates.

**Strings and counters.** `INCR` is atomic and creates the key at zero if missing, so it is the right primitive for counters and ID generation. `SET key value EX 60` sets a value and TTL in one command. Never do get-then-set for a counter in application code, because two clients read the same value.

**Hashes versus JSON strings.** A hash allows updating one field without rewriting the object and reading selected fields with `HMGET`, so it suits objects updated often in part. A JSON string is simpler and is better when you always read and write the whole object. Small hashes use a compact encoding that is very memory efficient. Hashes cannot nest. Per-field expiry (`HEXPIRE`, `HTTL`) exists since Redis 7.4, so check your version before relying on it.

**Lists.** `LPUSH` plus `RPOP` gives a queue. `BLPOP`/`BRPOP` block until an item arrives, which avoids polling. `LMOVE` (older name `RPOPLPUSH`) moves an item atomically to a second list, which is the basis of a reliable queue where a worker's in-progress list survives its crash. `LTRIM` keeps a list capped, for example the latest 100 events. Lists do not support acknowledgement, which is why streams are preferred for serious queues.

**Sorted sets.** The score gives ordering and the member gives uniqueness. Leaderboard: `ZADD board 1500 user:42`, `ZINCRBY board 10 user:42`, top ten with `ZRANGE board 0 9 REV WITHSCORES` (older versions use `ZREVRANGE`). Scheduling: use the run-at timestamp as the score and fetch due items with a score range. Score is a double, so integers beyond 2^53 lose precision.

**Bitmaps and HyperLogLog.** For "was user 123456 active today", a bitmap uses one bit per id and `BITCOUNT` gives the daily total. For "how many distinct visitors", HyperLogLog trades about one percent error for constant memory. If you need the exact set or membership, use a set.

**Streams** are covered in chapter 10. **Geo** stores points in a sorted set, so `ZRANGE` and `ZREM` also work on it.

**Encodings.** Small collections use compact encodings (listpack in recent versions, older ones used ziplist) and switch to full structures past size thresholds. This is automatic, but it explains why many small hashes can be far cheaper than many separate string keys. `MEMORY USAGE` measures it for a real key.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Picking a type | the commands reference at redis.io, complexity noted per command | State the complexity of any command before using it in a hot path |
| Atomic counters | `INCR`, `INCRBY`, `HINCRBY`, `ZINCRBY` | Build counters with no read-modify-write |
| Iterating safely | `SCAN`, `SSCAN`, `HSCAN`, `ZSCAN` with `COUNT` | Walk large collections without blocking |
| Measuring cost | `MEMORY USAGE`, `OBJECT ENCODING`, `redis-cli --bigkeys` | Estimate memory for a design before shipping it |
| Experimenting | `redis-cli`, RedisInsight browser | Inspect keys by type and TTL |

### Common mistakes

- Storing a large collection in one key and reading it whole (`SMEMBERS`, `HGETALL`, `LRANGE 0 -1`) on every request.
- Using a list as a set (checking membership with `LRANGE`) or a set where order matters.
- Using bitmaps with very large sparse ids, which allocates a huge string for one set bit.
- Using HyperLogLog and then needing to know who the visitors were.
- Choosing a string of JSON for an object that is updated field by field by several writers, losing updates.

### Self-check

- Which type would you use for a top-100 leaderboard that updates constantly, and why?
- What is the complexity of `SISMEMBER` versus `LRANGE`, and how does that affect the choice?
- When is a hash better than a JSON string, and when is it worse?
- How would you count distinct daily users for 50 million ids with bounded memory?

---

## 3. Key design, TTL and expiration

**In plain words.** A key is the name of a stored value. A TTL (time to live, the number of seconds before a key is deleted automatically) stops old data from filling memory. Redis removes expired keys in two ways, and you should know both.

### Understand

**Key naming.** Keys are binary-safe strings. A readable, consistent convention matters because keys are your schema and your only index.

- Use a delimiter, conventionally colon: `app:entity:id:attribute`, for example `shop:user:42:profile`.
- Include a short prefix for the application or service so several services can share an instance safely, and consider an environment prefix if instances are shared.
- Include a version in cached shapes (`v2:product:17`) so a deploy that changes the serialised format does not read old data. Bumping the version is also a cheap whole-cache invalidation.
- Keep keys short enough to be cheap but never cryptic. Millions of keys make key length a real memory cost, since the key is stored on every entry.
- Never build keys from unbounded user input without limits, because attackers can create unbounded distinct keys.
- Design for the lookup you need. There is no secondary index, so if you need to find users by email, maintain a second key (`email:<addr>` to user id) yourself.
- In Cluster, multi-key operations require keys in the same slot. Put a **hash tag** in braces so related keys land together (`cart:{42}:items`, `cart:{42}:meta`). Only the part inside the first braces is hashed (chapter 13).
- Prefer a single logical database (db 0). `SELECT` numbered databases are not supported in Cluster and give weak isolation (they share the same memory and thread).

**TTL mechanics.** `EXPIRE key seconds`, `PEXPIRE` (milliseconds), `EXPIREAT`, `SET key v EX seconds`, `TTL`, `PTTL` (returns -1 if no expiry, -2 if the key does not exist), `PERSIST` to remove it. Important details:

- Overwriting a key with `SET` clears its TTL unless you pass `KEEPTTL`. Forgetting this creates keys that never expire.
- `INCR`, `HSET` and similar modify values without touching the key's TTL.
- Expiry is stored as an absolute time, so it keeps counting while a server is down and applies at load time.
- On replicas, expiry decisions are driven by the primary, which sends a `DEL` when a key expires, so replicas stay consistent. A replica may still hold a logically expired key until the delete arrives, though recent versions hide it from reads.

**How expiration actually works: lazy and active.** Redis does not run a timer per key. It uses two cooperating mechanisms:

1. **Lazy expiration.** When a command touches a key, Redis first checks whether it is expired. If so, it deletes it and behaves as if it was missing. Reads are therefore always correct.
2. **Active expiration.** A background cycle runs frequently, samples a batch of keys that have a TTL, deletes the expired ones, and repeats while a large fraction of the sample was expired. This reclaims memory for keys nobody reads again.

Implication: expired keys are not removed at the exact moment of expiry, and memory use can lag. If a huge number of keys share the same expiry second, the active cycle has a burst of deletions to do and can add latency. This is one reason to add jitter to TTLs (chapter 6). Memory held by expired but not yet reclaimed keys still counts toward `used_memory`, and `INFO stats` reports `expired_keys`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspecting expiry | `TTL`, `PTTL`, `EXPIRETIME`, `OBJECT IDLETIME` | Verify a key has the TTL you intended |
| Setting expiry correctly | `SET k v EX n`, `SET ... KEEPTTL`, `EXPIRE k n NX` (conditional options exist in newer versions) | Set value and TTL in one atomic command |
| Finding keys safely | `SCAN 0 MATCH prefix:* COUNT 1000`, `redis-cli --scan --pattern 'prefix:*'` | Enumerate a namespace without blocking |
| Expiry health | `INFO stats` (`expired_keys`, `evicted_keys`), `INFO keyspace` (keys, expires, avg_ttl) | Notice keys without TTL growing unbounded |

### Common mistakes

- `SET` then `EXPIRE` as two commands. A crash between them leaves a key forever. Use `SET ... EX`.
- Overwriting a key and losing its TTL.
- Caching with no TTL "for now", then running out of memory.
- Same fixed TTL for every key written in a batch, so they all expire together.
- Using `KEYS pattern` to find keys to delete (chapter 14).

### Self-check

- Explain lazy and active expiration and why both exist.
- What does `TTL` return for a key without expiry and for a missing key?
- Why does a plain `SET` on an existing key with a TTL make it permanent?
- How would you give related keys the same slot in Cluster?

---

## 4. Memory management and eviction

**In plain words.** Redis stores data in RAM, and RAM is limited. Eviction (deleting keys automatically to free space) starts when the memory limit is reached. Choose the policy that matches the kind of data you store.

### Understand

**Memory is the limit.** Redis keeps the dataset in RAM, so capacity planning is about memory. `maxmemory` caps the memory Redis may use for data. If unset on a 64-bit system, Redis can grow until the OS kills it (chapter 6 of the Linux guide covers the OOM killer). Always set `maxmemory` below the machine or container limit, leaving headroom for:

- the copy-on-write cost of `fork` during snapshots and AOF rewrites (under heavy writes, much of memory can be duplicated),
- client and replication output buffers,
- allocator fragmentation.

**Eviction policy.** When `maxmemory` is reached and a write needs memory, `maxmemory-policy` decides what happens.

| Policy | Behaviour |
|---|---|
| `noeviction` | Reject writes with an error. Reads still work. Default in recent versions. |
| `allkeys-lru` | Evict least recently used keys from the whole keyspace. |
| `allkeys-lfu` | Evict least frequently used keys from the whole keyspace. |
| `allkeys-random` | Evict random keys. |
| `volatile-lru` / `volatile-lfu` / `volatile-random` | Same, but only among keys that have a TTL. |
| `volatile-ttl` | Evict keys with the shortest remaining TTL first, among keys with a TTL. |

LRU and LFU in Redis are approximations: Redis samples a few keys (`maxmemory-samples`) and evicts the best candidate among them, trading accuracy for speed.

**How to choose.**
- **Pure cache** (every key can be rebuilt): `allkeys-lru` is the usual choice. `allkeys-lfu` helps when a stable hot set should survive scans of cold data.
- **Mixed instance** (cache keys with TTL plus some keys that must never be evicted, such as locks or configuration): `volatile-*` policies protect keys without TTL, but writes fail with errors if no evictable key exists. Mixing is fragile. Prefer separate instances.
- **Queues, locks, source-of-truth data:** `noeviction`, so memory pressure becomes a visible error instead of silent data loss. BullMQ documents that it requires `noeviction`, because evicting queue keys corrupts queue state.
- With `noeviction` you need alerting on memory long before the limit, because the first symptom is failed writes.

**Fragmentation.** `INFO memory` reports `used_memory` (what Redis allocated), `used_memory_rss` (what the OS gave the process) and `mem_fragmentation_ratio` (their ratio). A ratio well above 1 means the allocator holds memory it cannot reuse, often after deleting many differently sized keys. A ratio below 1 can mean the OS swapped Redis memory out, which is a severe performance problem. Active defragmentation (`activedefrag`) exists and depends on build and version, so check the docs. Restarting a replica and failing over is another remedy.

**Big keys.** A single key with millions of elements or tens of megabytes is a hazard: reading it blocks the thread, deleting it with `DEL` blocks while memory is freed, and in Cluster it unbalances a shard. Use `UNLINK` (frees memory in the background) instead of `DEL` for large keys, and split big collections across keys (bucket by hash of the member). Find them with `redis-cli --bigkeys` and `MEMORY USAGE`.

**Memory reduction practices.** Short keys at scale, compact serialisation, hashes for many small related fields, TTLs on everything cache-like, and avoiding storing data you can recompute cheaply.

**Linux settings Redis warns about.** Redis logs warnings at startup for settings such as transparent huge pages (can cause latency spikes and large copy-on-write) and `vm.overcommit_memory` (affects whether `fork` for a snapshot can fail when memory is tight). Treat startup warnings as real. See the Linux guide for how to inspect kernel settings.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Memory overview | `INFO memory`, `MEMORY STATS`, `MEMORY DOCTOR` | Read used vs RSS vs fragmentation and explain them |
| Per-key cost | `MEMORY USAGE key [SAMPLES n]` | Estimate cost of a structure before and after a change |
| Finding big keys | `redis-cli --bigkeys`, `redis-cli --memkeys` (exists in current docs; first version unverified), `--keystats` (newer), `SCAN` plus `MEMORY USAGE` | Find the largest keys without blocking |
| Eviction behaviour | `CONFIG GET maxmemory*`, `INFO stats` (`evicted_keys`), `INFO keyspace` | Know whether Redis is evicting and which policy applies |
| Hot keys | `redis-cli --hotkeys` (requires an LFU policy) | Identify the most accessed keys |
| Non-blocking delete | `UNLINK`, lazy-free settings | Remove big keys without a stall |

### Common mistakes

- No `maxmemory`, so Redis competes with the OS and gets OOM-killed.
- Using an eviction policy on a queue or lock instance.
- Using `volatile-lru` with keys that have no TTL and then seeing "OOM command not allowed" errors.
- Setting `maxmemory` equal to container memory and ignoring fork and buffer overhead.
- Reading `used_memory_rss` and assuming a leak when it is fragmentation.

### Self-check

- Which eviction policy for a pure cache, and which for a job queue? Why?
- What does `mem_fragmentation_ratio` of 0.6 tell you?
- Why is `UNLINK` preferred over `DEL` for a big key?
- Why should `maxmemory` be set below the container limit?

---

## 5. Caching patterns

**In plain words.** A cache is a fast copy of data that is slow to get. The patterns here decide who fills the cache and when. The main risk is stale data (a copy that no longer matches the source).

### Understand

A cache stores a copy of data that is expensive to compute or fetch, trading **freshness** for latency and load reduction. Every pattern is a decision about who populates the cache, when, and what happens when the source changes.

**Cache-aside (lazy loading).** The most common pattern. The application owns the logic.

```ts
async function getUser(id: string): Promise<User> {
  const key = `v1:user:${id}`;
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);

  const user = await db.users.findById(id);        // source of truth
  if (user) await redis.set(key, JSON.stringify(user), 'EX', 300);
  return user;
}
// on write: update the database first, then delete the cache key
async function updateUser(id: string, patch: Partial<User>) {
  await db.users.update(id, patch);
  await redis.del(`v1:user:${id}`);
}
```

- Strengths: resilient (if Redis is down, fall back to the database), only requested data is cached, simple.
- Weaknesses: first request is slow (cold miss), and there is a window of staleness between a database write and the cache delete. Races exist: a reader may load the old row, a writer then updates the database and deletes the key, and the reader finally writes the old value into the cache. TTLs bound how long such stale data lives, which is why cache-aside always needs a TTL.
- **Delete on write, do not update on write.** Deleting is safer than writing the new value, because concurrent writers can set the cache in a different order from the database.

**Read-through.** The cache layer itself loads from the source on a miss. The application only talks to the cache abstraction. Redis does not do this natively, so it is implemented in a library or service wrapper. Behaviour is like cache-aside with the loading logic centralised, which also makes single-flight protection (chapter 6) easier to apply in one place.

**Write-through.** Writes go to the cache and the database together, synchronously, through one code path. The cache is always warm and consistent with the database after a successful write. Cost: write latency includes both, data never read still occupies memory, and the two writes are not atomic, so you must decide what happens if one fails (usually: database first, then cache, and treat a cache failure as a delete).

**Write-behind (write-back).** Writes go to the cache and are persisted to the database asynchronously, often batched. Writes are very fast and the database load is smoothed. The risk is **data loss**: if Redis fails before the flush, acknowledged writes are gone, and the database lags. Use it only for data that can tolerate loss (counters, view counts, analytics), with a durable queue (for example a stream) between cache and database if loss matters.

**Choosing.**

| Pattern | Consistency | Write latency | Failure behaviour | Typical fit |
|---|---|---|---|---|
| Cache-aside | Eventual, bounded by TTL and invalidation | Database only | Falls back to database | Default for most read-heavy services |
| Read-through | Same as cache-aside | Database only | Depends on the wrapper | Same, with centralised loading |
| Write-through | Strong between cache and database when both succeed | Database plus cache | Partial failure must be handled | Read-heavy, must not be stale after write |
| Write-behind | Database lags cache | Cache only | Can lose acknowledged writes | High write rates of loss-tolerant data |

**What to cache.** Data that is read far more often than it changes, is expensive to compute, and tolerates bounded staleness: rendered fragments, permission lookups, reference data, session data, aggregated counts. Do not cache data that must be exact at every read (account balances used to authorise a payment) without a clear invalidation story.

**Serialisation.** JSON is readable but larger and slower. Binary formats (MessagePack, Protocol Buffers) are smaller. Include a version in the key when the format changes. Compress only large values, since compression costs CPU on the client.

**Local plus remote caching.** A small in-process cache in front of Redis removes network hops for the hottest keys, at the cost of per-instance staleness and more complex invalidation. Keep local TTLs short.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Hit-rate analysis | `INFO stats` (`keyspace_hits`, `keyspace_misses`), application metrics | Compute and track hit ratio, explain a drop |
| Pattern implementation | ioredis `get`/`set ... EX`/`del`, wrapper libraries | Implement cache-aside with correct TTL and delete-on-write |
| Cache inspection | `redis-cli`, RedisInsight, `SCAN` with prefix | Verify what is cached, TTLs and sizes |
| Load testing the miss path | `redis-benchmark`, application load tools | Know what the database sees when the cache is empty |

### Common mistakes

- Cache-aside with no TTL, relying only on explicit invalidation that will eventually be missed.
- Updating the cache value on write instead of deleting it, and creating ordering races.
- Caching `null` results inconsistently (chapter 6) and either hammering the database or serving stale "not found".
- Caching per-user or permission-dependent data under a shared key.
- Treating the cache as required for correctness, so that an outage of Redis becomes an outage of the product.

### Self-check

- Describe the race in cache-aside between a reader and a writer, and what bounds its damage.
- Why do we delete rather than update the cache on a write?
- Which pattern risks losing acknowledged writes, and when is that acceptable?
- How do you measure whether a cache is worth having?

---

## 6. Cache failure modes and invalidation

**In plain words.** Caches fail in known ways, often when many requests arrive together. A stampede (many requests rebuild the same value at the same time) can overload the database. Each failure here has a standard fix.

### Understand

Caches fail in recognisable ways, usually under load and usually by moving the load onto the database. Each has a name and a standard mitigation.

**Cache stampede (thundering herd, dog-piling).** A popular key expires. Hundreds of concurrent requests miss at once, all run the expensive query, and the database is overwhelmed. Mitigations:

- **Single-flight (request coalescing).** Only one caller per key rebuilds. Others wait for it. In one process, keep a map from key to in-flight promise. Across processes, take a short lock (`SET lock:key token NX PX 5000`), let the winner rebuild and write, and let the losers briefly retry the cache or serve a stale value.
- **Probabilistic early refresh.** Instead of everyone refreshing at expiry, each reader may refresh a little early with a probability that increases as expiry approaches and with the cost of recomputation. This spreads refreshes without coordination. The technique is described in "Optimal Probabilistic Cache Stampede Prevention" (Vattani, Chierichetti, Lowenstein, VLDB 2015), often called XFetch.
- **Stale-while-revalidate.** Store a soft expiry inside the value. After the soft expiry, serve the stale value and refresh in the background, while the hard TTL is longer.
- **Pre-warming** known hot keys before they expire, for example by a scheduled job.

**Cache avalanche.** Many keys expire at the same time (a batch loaded together with one TTL, or a Redis restart that loses the cache), so the database receives a wave of misses. Mitigations: **TTL jitter** (add a random offset, for example base 300 s plus up to 60 s), staggered warm-up, rate-limiting or circuit-breaking calls to the database, and high availability so that a Redis failure does not wipe the cache for everyone.

```ts
const ttl = 300 + Math.floor(Math.random() * 60);
await redis.set(key, value, 'EX', ttl);
```

**Cache penetration.** Requests for keys that do not exist anywhere (random ids, malicious scans) always miss the cache and always hit the database. Mitigations:

- **Negative caching.** Cache the "not found" result with a short TTL (for example 30 to 60 seconds) using a sentinel value that is distinct from a real value.
- **Input validation** before the lookup.
- A **Bloom filter** of valid ids in front of the cache, which answers "definitely not present" cheaply. A Bloom filter may give false positives but never false negatives. Bloom filters are a data type in Redis 8 and later (earlier through the RedisBloom module), and availability in other distributions such as Valkey differs, so check before depending on it. An in-process filter is the alternative.
- Rate limiting by client (chapter 8).

**Hot keys.** A single key receives a disproportionate share of traffic (a viral item, a global config key). One Redis thread, or one Cluster shard, becomes the bottleneck, and adding nodes does not help because the key lives on one node. Mitigations: a short-lived **in-process cache** in each application instance, **replicating the key under several names** (`item:17#0` to `item:17#7`) and reading a random one, and read replicas for read-heavy keys (with replication-lag awareness). Detect with `redis-cli --hotkeys` (needs an LFU policy), application metrics, or `MONITOR` for a very short sample.

**Cache inconsistency from races** is covered in chapter 5.

**Invalidation.** "There are only two hard things in computer science: cache invalidation and naming things." The strategies, in order of preference:

1. **TTL only.** Accept staleness up to the TTL. Simplest and always correct as a safety net.
2. **Delete on write.** Delete the key in the same code path that writes the source. Fast, but depends on every writer doing it, including background jobs and manual changes.
3. **Event-driven invalidation.** Publish a change event (from the service, a change-data-capture feed or a message broker) and have consumers delete keys. Decouples writers from caches. Pub/sub is lossy (chapter 10), so keep TTLs as a backstop.
4. **Versioned keys (generation numbers).** Store a version counter (`INCR user:42:ver`) and include it in cache keys (`user:42:v{ver}:profile`). Bumping the counter invalidates a whole group at once without scanning. Old entries expire by TTL or get evicted.
5. **Tag or set-based invalidation.** Keep a set of keys per tag, delete them together. Works but the set itself must be bounded and cleaned.

Never invalidate by `KEYS pattern` and `DEL`. Use versioned keys or `SCAN` in a background job with `UNLINK`.

**Graceful degradation.** Decide what the application does when Redis is slow or down: fall back to the database with protection (concurrency limits, circuit breaker), serve stale data, or fail fast. Set client timeouts so a Redis incident does not become a request-thread incident (chapter 16).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Stampede protection | `SET NX PX` lock, in-process promise map, soft-expiry fields | Implement single-flight and explain when it fails |
| Jitter and TTL policy | application code, per-class TTL constants | Apply jitter to every bulk-written key |
| Negative caching | sentinel values with short TTL | Cache "not found" without poisoning real data |
| Detecting hot keys | `redis-cli --hotkeys`, per-key application metrics | Find and mitigate a hot key |
| Observing invalidation | `INFO stats`, `MONITOR` (briefly, on a non-critical instance), logs | Verify an invalidation reached the cache |

### Common mistakes

- Fixing a stampede by lengthening TTLs, which delays it and increases staleness.
- Single-flight locks with no expiry, so a crashed rebuilder blocks everyone.
- Negative caching with a long TTL, so a newly created record looks missing.
- Invalidating by pattern scan on the request path.
- No fallback and no timeout, so a slow Redis stalls all requests.

### Self-check

- A popular key expires at peak traffic and the database falls over. List three mitigations.
- How does TTL jitter prevent an avalanche?
- How do you stop a flood of requests for nonexistent ids from reaching the database?
- Why is a hot key not solved by adding Cluster nodes?
- What is the benefit of versioned keys over deleting by pattern?

---

## 7. Atomicity: Lua, transactions and pipelining

**In plain words.** Sometimes several steps must run without other clients getting in between. Atomic (done as one step that cannot be split) is the word for this. A pipeline only saves network time. A transaction or a Lua script (a small program that runs inside Redis) keeps steps together.

### Understand

Three different tools solve three different problems. Mixing them up is a classic interview and production error.

**Pipelining** is about **round trips**, not atomicity. The client sends many commands without waiting for each reply and then reads all replies. Commands from other clients may be interleaved between yours. It can improve throughput by an order of magnitude on high-latency links. Batch sizes should stay bounded (hundreds to a few thousand commands), since replies are buffered in memory on the server.

**Transactions: `MULTI` / `EXEC` / `WATCH`.**

```
MULTI
INCR stock:42
LPUSH orders:42 "order-9"
EXEC
```

- Commands after `MULTI` are queued and executed together by `EXEC`, with no other client's commands interleaved. That is isolation.
- **There is no rollback.** If a command fails at execution (for example `INCR` on a non-numeric string), the others still run. Syntax errors found while queuing cause `EXEC` to abort the whole block. This is different from a SQL transaction.
- **You cannot branch on a value read inside the transaction**, because commands are queued and their results arrive only at `EXEC`.
- **`WATCH key`** gives optimistic locking (check-and-set). If a watched key is modified by another client before `EXEC`, `EXEC` returns a null reply and nothing runs, and you retry.

```
WATCH balance:42
GET balance:42          -- read the value (outside MULTI)
MULTI
SET balance:42 <new>    -- computed in the client
EXEC                    -- null if balance:42 changed meanwhile
```

Under contention this retry loop wastes work. A Lua script (below) is usually simpler. In Cluster, a transaction's keys must be in one slot.

**Lua scripting (`EVAL`, `EVALSHA`).** A script runs on the server as one atomic unit, can read values, branch, and write, with no network round trip between steps. This is the tool for check-and-set logic: rate limiters, safe lock release, conditional updates.

```
-- compare-and-delete (used for lock release)
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:order:42 <token>
```

Rules and facts:
- Pass key names through `KEYS` and other values through `ARGV`. Cluster uses declared keys to route the script, and all keys must be in the same slot.
- Scripts block the server while they run, so they must be short and bounded. A loop over a million items is an outage. Redis has a script time limit setting after which other clients get `BUSY` errors, and the script can only be stopped with `SCRIPT KILL` if it has not written yet.
- Scripts are replicated by their effects (the write commands they produce), not by their source. This is the default since Redis 5.0 and the only mode since 7.0. Because of this, `TIME` and random commands are allowed in scripts on those versions. On older versions they were restricted.
- `EVALSHA` runs a cached script by its SHA1 and avoids resending the body. Load with `SCRIPT LOAD`. If the script cache is flushed or you fail over to a node that never loaded it, `EVALSHA` returns `NOSCRIPT`, and clients typically fall back to `EVAL`. ioredis `defineCommand` handles this for you.
- Newer versions add **Functions** (`FUNCTION LOAD`, `FCALL`), a persisted and replicated alternative to ad hoc scripts. Check your version.

```ts
redis.defineCommand('releaseLock', {
  numberOfKeys: 1,
  lua: `if redis.call('GET', KEYS[1]) == ARGV[1]
        then return redis.call('DEL', KEYS[1]) else return 0 end`,
});
await (redis as any).releaseLock('lock:order:42', token);
```

**Choosing.**

| Need | Use |
|---|---|
| Many independent commands, fewer round trips | Pipeline |
| Several writes that must apply together with no interleaving, no branching | `MULTI`/`EXEC` |
| Check-and-set on one value with retry tolerated | `WATCH` + `MULTI` |
| Read, decide, write atomically | Lua script |
| A single operation that already exists | The single command (`INCR`, `SET NX`, `HSETNX`, `GETDEL`) |

Atomicity in Redis is about **isolation between clients**, not durability. A script that completed is not necessarily on disk (chapter 11).

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pipelining | ioredis `pipeline()`, `enableAutoPipelining`, `redis-benchmark -P` | Measure the gain and bound batch size |
| Transactions | `MULTI`, `EXEC`, `DISCARD`, `WATCH`, `UNWATCH`, ioredis `multi()` | Explain no-rollback and use `WATCH` correctly |
| Lua | `EVAL`, `EVALSHA`, `SCRIPT LOAD/EXISTS/FLUSH/KILL`, ioredis `defineCommand` | Write a safe, short script and handle `NOSCRIPT` |
| Debugging scripts | `redis-cli --eval script.lua key , arg`, `SLOWLOG` | Run and time a script from the CLI |

### Common mistakes

- Expecting `MULTI`/`EXEC` to roll back on error.
- Using a pipeline and assuming it is atomic.
- Doing read-modify-write in the application (`GET`, compute, `SET`) for anything shared.
- Long-running Lua that blocks all clients.
- Scripts that touch keys not passed in `KEYS`, which breaks in Cluster.

### Self-check

- What is the difference between a pipeline and `MULTI`/`EXEC`?
- A command inside `MULTI` fails at `EXEC`. What happens to the others?
- Why is a Lua script often better than `WATCH` under contention?
- What is `NOSCRIPT` and when do you see it?

---

## 8. Rate limiting

**In plain words.** A rate limiter controls how many requests a client can make in a period of time. Redis is a good fit because all servers can share one counter. The counter update must be atomic, or two requests can both pass the check.

### Understand

A rate limiter protects a service from abuse and from overload. Redis suits it because counters are shared across all application instances and `INCR` and scripts are atomic.

**Fixed window.** Count requests per client per time window.

```
INCR  rl:user:42:28561234      -- window id = floor(now / windowSeconds)
EXPIRE rl:user:42:28561234 60  -- only needed on first increment
```

If the result exceeds the limit, reject. Do the `INCR` and `EXPIRE` atomically (a Lua script, or `SET key 0 EX 60 NX` followed by `INCR`), otherwise a crash between them leaves a key without expiry. Simple and cheap. The flaw is the **boundary burst**: a client can send the full limit at the end of one window and again at the start of the next, doubling the effective rate for a moment.

**Sliding window log (sorted set).** Store each request timestamp as a member of a sorted set, scored by time. Count only entries in the last window.

```
-- KEYS[1] = rl:user:42, ARGV = now_ms, window_ms, limit, unique_member
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1] - ARGV[2])
if redis.call('ZCARD', KEYS[1]) < tonumber(ARGV[3]) then
  redis.call('ZADD', KEYS[1], ARGV[1], ARGV[4])
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
end
return 0
```

Accurate, no boundary burst. Memory is proportional to the limit per client (one entry per request in the window), so it is costly for high limits or many clients. The member must be unique per request (for example timestamp plus random suffix), or two requests in the same millisecond collapse into one.

**Sliding window counter (approximation).** Keep counters for the current and previous fixed windows and weight the previous one by how much of it still overlaps the sliding window. Memory is constant, and accuracy is good enough for most uses.

**Token bucket.** A bucket holds up to `capacity` tokens and refills at a fixed rate. Each request consumes a token. It allows bursts up to capacity while enforcing an average rate. Store two values per client (tokens and last refill time) in a hash and update them in one Lua script, which computes the refill from elapsed time.

```
-- KEYS[1]=bucket  ARGV: capacity, refill_per_sec, now_ms, cost
local d = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(d[1]) or tonumber(ARGV[1])
local ts = tonumber(d[2]) or tonumber(ARGV[3])
local elapsed = math.max(0, tonumber(ARGV[3]) - ts) / 1000
tokens = math.min(tonumber(ARGV[1]), tokens + elapsed * tonumber(ARGV[2]))
local allowed = 0
if tokens >= tonumber(ARGV[4]) then tokens = tokens - tonumber(ARGV[4]); allowed = 1 end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('PEXPIRE', KEYS[1], math.ceil(tonumber(ARGV[1]) / tonumber(ARGV[2]) * 1000) * 2)
return allowed
```

Passing `now` from the client makes the script simple, but clocks across application hosts differ slightly. Using the server's clock inside the script (`redis.call('TIME')`) avoids that. It is allowed since Redis 5.0, because scripts replicate by their effects. Also `HSET` with multiple field-value pairs needs Redis 4.0 or later.

**Atomicity is the key point.** A limiter written as `GET`, compare, `INCR` in application code lets concurrent requests all pass the check. The check and update must be one atomic step (`INCR` result, or a Lua script).

**Design questions that matter in practice.**
- **Key identity.** By user id, API key, IP address, or a combination. IP limits behind a proxy need the real client IP from a trusted header (see the Linux guide on reverse proxies).
- **Fail open or fail closed.** If Redis is down, do you allow traffic (protects availability, loses protection) or reject it (protects the backend, hurts users)? Decide deliberately and set a short client timeout.
- **Response.** Return HTTP 429 with `Retry-After`, and expose limit headers.
- **Cluster.** Each limiter key is independent, so limiters shard well. Keep all keys of one script in one slot.
- **Layering.** Coarse limits at the edge or gateway, fine-grained per-user limits in the application.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Counters with expiry | `INCR`, `EXPIRE`, `SET ... EX NX`, `PTTL` | Implement a correct fixed window |
| Sliding window | `ZADD`, `ZREMRANGEBYSCORE`, `ZCARD`, Lua | Implement it atomically and state its memory cost |
| Token bucket | Lua, `HMGET`/`HSET`, `TIME` | Write the script and reason about refill math |
| Testing limiters | `redis-benchmark`, load tools, unit tests with fake time | Prove behaviour at the boundary and under concurrency |
| Existing libraries | `rate-limiter-flexible` and similar | Know what they implement before adopting |

### Common mistakes

- Non-atomic check and increment.
- `INCR` without ever setting an expiry, creating permanent keys.
- Fixed windows with the boundary burst unnoticed.
- Sorted-set members that are not unique per request.
- Rate limiting by IP while behind a load balancer, so every user shares the proxy's address.

### Self-check

- Compare fixed window, sliding window log and token bucket on accuracy, memory and burst behaviour.
- Why must the token bucket update be one Lua script?
- Should the limiter fail open or closed when Redis is unreachable? Argue both sides.

---

## 9. Distributed locks

**In plain words.** A distributed lock lets only one process do a task at a time, even across many servers. A Redis lock is easy to build but has limits. A fencing token (a number that grows with each lock, which the protected system checks) makes it safer.

### Understand

A distributed lock lets only one of many processes perform an action at a time. Redis is often used because it is available and fast. Be precise about what it can and cannot guarantee.

**The basic correct lock on a single Redis instance.**

Acquire:
```
SET lock:invoice:42 <random-token> NX PX 30000
```
- `NX`: set only if the key does not exist. This is the atomic "test and take".
- `PX 30000`: expiry in milliseconds, so a crashed holder cannot block everyone forever.
- The value is a **unique token** (a random string per acquisition) that identifies the holder.

Release, only if you still own it:
```
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```
This must be a Lua script, because `GET` then `DEL` as separate commands has a race: the lock could expire and be taken by someone else between them, and you would delete their lock. Without the token check, a slow process whose lock expired would release another process's lock. Redis 8.4 adds `DELEX key IFEQ <token>` as a single-command alternative to this script. Use the script on older versions.

**Why the TTL creates the real problem.** The lock expires on a timer, but the holder does not know that. Consider:

1. Process A takes the lock with a 30 s TTL.
2. A pauses (a long garbage collection, a stalled network, a VM freeze) for longer than 30 s.
3. The lock expires. Process B takes it and starts working.
4. A resumes, believes it still holds the lock, and writes. A and B are now both in the critical section.

No timing assumption in the lock service can prevent this, because the pause happens in the client after the check.

**Fencing tokens.** The robust fix is for the lock to hand out a monotonically increasing number with each acquisition (for a Redis lock, `INCR` a counter at acquisition), and for the **protected resource** to reject any operation carrying a token lower than the highest it has seen. Then a stale holder's late write is refused by the resource. This only works if the resource (the database, the storage service) can check the token. For example, a database update with `WHERE fencing_token < :token`, or a version column.

**Lock extension.** For work longer than the TTL, a watchdog can extend the TTL periodically while the work runs (with a check-and-extend Lua script). That reduces expiry during normal operation but does not remove the pause problem above.

**Redlock and the controversy.** Redlock is an algorithm proposed in the Redis documentation to acquire a lock on a majority of N independent Redis instances (commonly five), within a time budget, to tolerate failure of some instances. Two issues matter:

- A single Redis primary with asynchronous replication can lose a lock on failover: the primary grants the lock, crashes before replicating, a replica is promoted without the key, and a second client takes the same lock. Redlock tries to address this with multiple independent masters.
- Martin Kleppmann published a critique ("How to do distributed locking") arguing that Redlock depends on timing assumptions (bounded network delay, process pauses and clock drift) that real systems do not satisfy, and that it provides no fencing token, so it is neither safe enough for correctness nor needed for efficiency. Salvatore Sanfilippo (antirez), the author of Redis, replied that the timing assumptions are acceptable in practice. The disagreement is real and unresolved in the community. Read both primary sources (see References) rather than a summary.

**A practical position.** Classify what the lock protects:

- **Efficiency locks:** the lock prevents duplicate work (two workers rebuilding the same cache entry, two cron instances sending the same report). Occasionally two holders is harmless. A single-instance `SET NX PX` lock is appropriate.
- **Correctness locks:** two holders would corrupt data or cause a wrong outcome. **Do not use a Redis lock as the only protection for money or any invariant that must hold.** Charging a card twice, double-spending a balance or overselling the last unit must be guarded by the database: transactions, row locks (`SELECT ... FOR UPDATE`), unique constraints, idempotency keys, optimistic version checks, or a consensus-based system (ZooKeeper, etcd) with fencing. A Redis lock may reduce contention in front of that, but the database is the authority.

For idempotent payments the correct tools are an idempotency key stored with a unique constraint and atomic state transitions in the database, not a lock.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Acquire | `SET key token NX PX ttl` | Acquire with unique token and TTL in one command |
| Safe release | Lua compare-and-delete, ioredis `defineCommand` | Write it from memory and explain the race it prevents |
| Extension | Lua compare-and-`PEXPIRE` | Implement a watchdog and state its limit |
| Fencing | `INCR` counter, version column, conditional `UPDATE` | Make the resource reject stale writers |
| Libraries | `redlock` for Node and similar | Read what the library guarantees and what it does not |

### Common mistakes

- `SETNX` then `EXPIRE` as two commands (crash in between leaves a permanent lock). Use `SET NX PX`.
- Releasing with a plain `DEL`, deleting another holder's lock.
- Using a constant value instead of a unique token.
- TTL shorter than the worst-case work duration, with no fencing.
- Guarding payments or inventory correctness with a Redis lock alone.
- Assuming Redlock makes timing problems disappear.

### Self-check

- Why must the release be a Lua script and why does the value need to be unique?
- Walk through how a lock holder and the lock can disagree about who owns it after a pause.
- What is a fencing token and what must the protected resource do with it?
- Summarise the Redlock disagreement and say which kinds of lock are safe on Redis.
- How would you prevent double charging if two workers process the same payment?

---

## 10. Pub/sub, streams and job queues

**In plain words.** Redis can pass messages in three ways, and each has a different safety level. Pub/sub loses a message if nobody is listening at that moment. Streams and lists keep messages until a worker handles them. Handlers should be idempotent (running them twice gives the same result as running them once).

### Understand

Redis offers several messaging primitives with very different guarantees. Choosing wrongly loses messages silently.

**Pub/sub.** `SUBSCRIBE channel`, `PSUBSCRIBE pattern`, `PUBLISH channel message`.
- **Fire-and-forget, at-most-once.** A message is pushed to the subscribers connected at that instant. If a subscriber is disconnected, slow or restarting, it misses the message permanently. There is no storage, no replay and no acknowledgement.
- A subscribing connection is in a special mode and can only run subscription commands, so it needs its own connection.
- A slow subscriber causes the server to buffer output for it, and the client output buffer limit for pub/sub clients can disconnect it.
- In Cluster, plain pub/sub broadcasts messages across the cluster. Newer versions add sharded pub/sub (`SSUBSCRIBE`, `SPUBLISH`), so check the docs.
- Good for: cache invalidation hints (with TTL as backstop), presence, live dashboards, websocket fan-out across app instances. Not for work that must not be lost.

**Streams.** An append-only log, with entries that have an ID (a millisecond timestamp and a sequence number, auto-generated by `*`) and field-value pairs.

```
XADD orders:events MAXLEN ~ 100000 * orderId 42 status paid
XGROUP CREATE orders:events billing $ MKSTREAM
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS orders:events >
XACK orders:events billing 1700000000000-0
```

Concepts:
- **Persistent log.** Entries stay until trimmed (`XTRIM`, or `MAXLEN` on `XADD`; the `~` form trims approximately and is cheaper). Consumers can read from any ID and replay history.
- **Consumer groups.** Several consumers in one group share the stream, and each entry is delivered to exactly one consumer in the group. Different groups each see all entries, which gives fan-out to independent services.
- **Pending Entries List (PEL).** When a consumer reads an entry with `XREADGROUP`, it becomes pending for that consumer until acknowledged with `XACK`. This gives **at-least-once** delivery.
- **Recovery.** If a consumer dies, its pending entries stay pending. `XPENDING` lists them, and `XAUTOCLAIM` (or `XCLAIM`) lets another consumer take over entries idle longer than a threshold. A reaper loop that claims stuck entries is part of any production consumer.
- **Handling duplicates.** At-least-once means a message can be processed twice (consumer crashed after processing but before `XACK`). Consumers must be idempotent, for example by recording processed ids or using natural idempotency.
- **Poison messages.** An entry that always fails will be retried forever. Track the delivery count (available through `XPENDING`) and move it to a dead-letter stream after a limit.
- **Reading new versus pending.** `>` asks for entries never delivered to the group, and `0` reads this consumer's own pending history.

Streams are bounded by memory, so trim deliberately. Retention in a stream is not a substitute for a log system such as Kafka when you need very long retention, huge volumes or partitioned ordering at scale.

**Comparison.**

| Feature | Pub/sub | List queue (`LPUSH`/`BRPOP`) | Stream with groups |
|---|---|---|---|
| Stored | No | Yes, until popped | Yes, until trimmed |
| Delivery | At-most-once | Popped item is gone, lost if worker crashes (unless `LMOVE` pattern) | At-least-once with ack |
| Replay | No | No | Yes |
| Multiple independent consumers | Yes (all subscribers) | No (competing) | Yes (groups) |
| Recovery of crashed worker's items | No | Manual with `LMOVE` | `XAUTOCLAIM` |

**Job queues on Redis.** Most Node systems use a library rather than raw commands. **BullMQ** is a widely used example. Concepts to understand regardless of library:

- Jobs are stored in Redis structures (lists, sorted sets for delayed jobs, hashes for job data) with state transitions: waiting, active, delayed, completed, failed.
- Workers take jobs, process them, and mark them complete. Features include retries with backoff, delayed jobs, priorities, rate limits, repeatable jobs and concurrency control.
- **Stalled jobs.** If a worker dies while processing, the job must be detected as stalled (the worker failed to renew a lock) and re-queued. This gives at-least-once semantics, so job handlers must be idempotent.
- **Durability depends on your Redis configuration.** The queue is only as durable as Redis. With no persistence, a restart loses all pending jobs. With RDB only, you lose changes since the last snapshot. With AOF `everysec`, up to about a second of acknowledged jobs can be lost. Replication is asynchronous, so a failover can drop recent jobs. For jobs that matter (emails, payment webhooks), enable AOF, use a replicated deployment, and consider keeping the source of truth in your database with an outbox pattern so that lost jobs can be recreated.
- **Eviction must be `noeviction`.** BullMQ documents this requirement, because evicting a queue's keys corrupts it. Do not share a cache instance (with eviction) with a queue.
- **Do not co-locate** a queue and a cache with a high churn on one instance if you can avoid it, since memory pressure from the cache can break the queue.
- Monitor queue depth, oldest waiting job age, failed counts and stalled counts. Queue depth growth is an early warning signal.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Pub/sub | `SUBSCRIBE`, `PUBLISH`, `PUBSUB CHANNELS`, ioredis `subscribe` with a dedicated connection | Explain exactly what is lost and when |
| Streams | `XADD`, `XLEN`, `XRANGE`, `XGROUP`, `XREADGROUP`, `XACK`, `XPENDING`, `XAUTOCLAIM`, `XINFO` | Build a consumer group with ack, reclaim and dead-letter handling |
| List queues | `LPUSH`, `BRPOP`, `LMOVE`/`BLMOVE` | Build a reliable queue with an in-progress list |
| Queue libraries | BullMQ, its dashboards and events | Configure retries, backoff, stalled handling and concurrency |
| Observing | `XINFO GROUPS`, `XINFO CONSUMERS`, `INFO persistence`, queue metrics | Alert on lag and pending counts |

### Common mistakes

- Using pub/sub for jobs or events that must not be lost.
- Using a list with `RPOP` and losing the item when the worker crashes mid-job.
- Never acknowledging stream entries, so the PEL grows without bound.
- Never trimming a stream, so it fills memory.
- Non-idempotent job handlers on an at-least-once system.
- Running a queue on a no-persistence, eviction-enabled instance.
- Sharing one subscriber connection for normal commands.

### Self-check

- What exactly happens to a pub/sub message when a subscriber is briefly disconnected?
- Explain consumer groups, the pending list and how a crashed consumer's work is recovered.
- Why must job handlers be idempotent?
- What persistence configuration would you choose for an instance that holds a payments-related job queue, and what risk remains?

---

## 11. Persistence

**In plain words.** Persistence means saving data to disk so it survives a restart. RDB saves snapshots (full copies of the data at one moment). AOF (append-only file) records each write. Neither one is a backup.

### Understand

Redis serves data from memory, but it can write data to disk so that a restart does not start empty. There are two mechanisms, and you may use both or neither.

**RDB (snapshot).** A point-in-time binary dump of the dataset.
- Triggered by rules (`save <seconds> <changes>`), by `BGSAVE`, and on shutdown. `SAVE` blocks the server and should not be used in production.
- Redis `fork`s a child process, which writes the snapshot while the parent keeps serving. Copy-on-write means memory use grows with the write rate during the snapshot, and the `fork` call itself can pause the server noticeably on large datasets (`INFO stats` field `latest_fork_usec`).
- Compact, fast to load, good for backups and fast restarts.
- Data written after the last snapshot is lost on a crash. With snapshots every few minutes, that window is minutes.

**AOF (append-only file).** A log of every write command, replayed at startup.
- Enabled with `appendonly yes`. The `appendfsync` policy controls durability:

| `appendfsync` | Behaviour | Loss on crash | Cost |
|---|---|---|---|
| `always` | fsync after every write | Essentially none of acknowledged writes (see caveat) | Slowest, limits write throughput |
| `everysec` | fsync once per second | Up to about one second | Common default, good balance |
| `no` | leave flushing to the OS | Whatever the OS had not flushed, often tens of seconds | Fastest |

- Caveat: fsync reaches the disk controller, and storage that lies about flushes or ignores barriers can still lose data. Hardware and filesystem matter (see the Linux guide on the page cache and `fsync`).
- The log grows, so Redis periodically **rewrites** it in the background into a compact form (`BGREWRITEAOF`, and automatic rewrite thresholds). Rewrites also fork.
- In recent versions the AOF is split into multiple files (a base file and incremental files) managed through a manifest. Do not hand-edit them, and check the docs for your version.
- Tools: `redis-check-aof` can repair a truncated AOF.

**RDB plus AOF.** Many deployments enable both. On restart, if AOF is enabled Redis loads the AOF because it is the more complete record. RDB remains useful for backups and fast replica seeding.

**Trade-offs.**

| Goal | Choice |
|---|---|
| Pure cache, can rebuild | No persistence, or RDB only for warm restarts |
| Accept minutes of loss, fast restarts | RDB |
| Lose at most about a second | AOF `everysec` |
| Minimise loss, accept throughput hit | AOF `always` |
| Backups and disaster recovery | Copy RDB snapshots off-host regularly |

**Persistence is not replication, and neither is a backup.**
- **Persistence** protects against a process or machine restart on the same disk.
- **Replication** copies data to another running node for availability and read scaling. It is asynchronous, so a failover can lose recent writes. It also replicates mistakes: a `FLUSHALL` is replicated to every replica immediately.
- **Backups** are separate copies, off the host, that survive deletion and corruption, and must be restore-tested.

An instance configured with persistence still loses data in these cases: the disk is lost with the machine, the last second or minutes of writes, and an operator error replicated everywhere. A common failure: a replica is promoted with an empty dataset, then the old primary, restarted, replicates from it and wipes its own data. Know the restart and failover procedure.

**Operational concerns.** Disk space for RDB and AOF (an AOF rewrite temporarily needs extra), disk throughput during rewrites, memory headroom for fork copy-on-write, and the effect on latency. Check `INFO persistence` for `rdb_last_bgsave_status`, `aof_last_write_status`, `rdb_last_save_time` and rewrite state, and alert when statuses are not `ok`.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Snapshots | `BGSAVE`, `LASTSAVE`, `CONFIG GET save`, `redis-check-rdb` | Trigger and verify a snapshot |
| AOF | `CONFIG GET appendonly appendfsync`, `BGREWRITEAOF`, `redis-check-aof` | Choose fsync policy and justify it |
| Health | `INFO persistence`, `INFO stats` (`latest_fork_usec`) | Detect failing saves and slow forks |
| Backups | copy RDB off-host, test restore | Restore into a fresh instance and verify data |
| Durability checks | disk and filesystem settings, monitoring | Explain what `fsync` guarantees on your storage |

### Common mistakes

- Assuming replication equals backup.
- AOF `no` or RDB-only on a queue, then surprise at data loss.
- No headroom for fork copy-on-write, causing OOM during snapshot.
- Never testing a restore.
- Ignoring `MISCONF` write errors that appear when a background save fails (by default Redis stops accepting writes if the last save failed).

### Self-check

- Compare RDB and AOF on durability, restart time and cost.
- What do `always`, `everysec` and `no` guarantee after a crash?
- Why is replication not a substitute for backups?
- Why can a snapshot cause a memory spike?

---

## 12. Replication, Sentinel and failover

**In plain words.** Replication copies data from a primary (the node that accepts writes) to replicas (the copies). Failover (switching to a replica when the primary fails) can lose the newest writes. Sentinel is a separate tool that detects the failure and does the switch.

### Understand

**Replication.** A replica connects to a primary, receives a full copy (an RDB transfer, or a diskless transfer in some configurations), then receives a stream of the primary's write commands. Reconnection after a short interruption uses **partial resynchronisation** from the replication backlog buffer. If the backlog no longer covers the gap, a full resync (expensive) happens.

Key facts:
- Replication is **asynchronous** by default. The primary acknowledges a write to the client before replicas have it. If the primary dies, writes not yet replicated are lost.
- **Replication lag** is the delay between a write on the primary and its appearance on a replica. Reads from replicas may be stale, which breaks read-your-own-writes. Check `INFO replication` (`master_repl_offset` on the primary against each replica's offset, and the `lag` shown for each replica).
- `WAIT numreplicas timeout` blocks until a number of replicas have acknowledged the writes so far. It narrows the loss window but does not make Redis a strongly consistent store.
- `min-replicas-to-write` and `min-replicas-max-lag` make a primary refuse writes if too few replicas are in contact, limiting loss during a partition.
- Replicas are read-only by default. Use them to scale reads, take backups without loading the primary, and as failover targets.
- A big full resync forks the primary and moves a large file, which can hurt production. Size the backlog (`repl-backlog-size`) for your write rate and expected disconnect time.

**Sentinel.** Redis Sentinel is a separate set of processes that monitor a primary and its replicas and provide automatic failover and service discovery. It is for a single shard with high availability, not for sharding.
- Run at least three Sentinels, on separate failure domains, so a majority can agree.
- A Sentinel marks a primary as subjectively down (it cannot reach it) and, when a **quorum** of Sentinels agree, as objectively down. A majority of all Sentinels then elects a leader which promotes the best replica and reconfigures the others.
- Clients ask Sentinel for the current primary address (`SENTINEL get-master-addr-by-name`), and a Sentinel-aware client (ioredis supports Sentinel configuration) reconnects after a failover.
- **Limits.** Failover takes seconds. Writes acknowledged by the old primary but not yet on the promoted replica are lost. During a network partition, an isolated old primary may accept writes that are later discarded. `min-replicas-to-write` limits this.
- Sentinel does not protect against logical errors or provide backups.

**Managed services** (cloud-hosted Redis) usually provide their own failover mechanism and endpoint. Read their documentation for the guarantees and the failover behaviour, and test your client against a failover.

**Application consequences.**
- Clients must reconnect and retry sensibly after failover (chapter 16).
- Operations must be safe to retry (idempotent) because a command may have been executed before the connection dropped.
- Do not assume a write that succeeded survives failover, unless you accept the `WAIT` semantics and their limits.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Replication status | `INFO replication`, `ROLE`, `REPLICAOF host port`, `CONFIG GET repl-*` | Read lag and offsets and spot a broken replica |
| Reducing loss | `WAIT`, `min-replicas-to-write`, `min-replicas-max-lag` | State what each does and does not guarantee |
| Sentinel | `redis-cli -p 26379 SENTINEL masters`, `SENTINEL replicas`, `SENTINEL failover`, `sentinel.conf` | Operate and test a failover |
| Client behaviour | ioredis `sentinels`, `name`, `role` options | Configure a client that survives failover |
| Failure drills | stop the primary in a test environment | Observe promotion time, lost writes and client recovery |

### Common mistakes

- Reading from replicas where stale data is not acceptable.
- Running Sentinels in two places only (no majority possible when one side is lost) or all on one host.
- Treating failover as lossless.
- Never testing failover or client reconnection.
- A tiny replication backlog causing repeated full resyncs.

### Self-check

- Why can a failover lose writes, and how do you limit the loss?
- What is the Sentinel quorum, and why do you need three Sentinels?
- How does a client discover the new primary after a failover?
- What does replication lag mean for a user who writes then immediately reads?

---

## 13. Redis Cluster

**In plain words.** Cluster splits data across many nodes, so you can store more and serve more traffic. The key space is divided into 16384 hash slots (numbered groups of keys). A command that uses several keys needs all of them in one slot.

### Understand

**Why Cluster.** A single primary is limited by one CPU core for command execution and by one machine's memory. Redis Cluster shards the keyspace over several primaries, each with replicas for failover.

**Hash slots.** The keyspace is split into **16384 hash slots**. A key's slot is `CRC16(key) mod 16384`. Each primary owns a subset of slots. Clients learn the slot map and send each command to the owning node.

**Routing.** If a client contacts the wrong node, it receives a `MOVED <slot> <host:port>` redirect and updates its slot map. During resharding it may receive `ASK` for a slot being migrated. Cluster-aware clients (ioredis `Redis.Cluster`) handle both. Redirects add latency, so clients cache the slot map.

**Hash tags.** If a key contains `{...}`, only the text inside the first pair of braces is hashed. `cart:{42}:items` and `cart:{42}:meta` share a slot. This enables multi-key commands and scripts on related keys. Overusing one tag (`{app}`) puts everything into one slot and one node, defeating sharding.

**Multi-key limits.** Commands that touch several keys (`MGET`, `SINTER`, `MULTI`/`EXEC`, Lua scripts, `RENAME`) require all keys to be in the same slot, otherwise the server returns a `CROSSSLOT` error. Pipelines are fine across slots only if the client splits them per node. Cluster supports only database 0.

**Resharding.** Slots can be moved between nodes while the cluster serves traffic. Keys are migrated slot by slot, and clients are redirected with `ASK`/`MOVED`. Tools: `redis-cli --cluster` commands (`create`, `check`, `add-node`, `reshard`, `rebalance`) and the `CLUSTER` command family. Migrating very large keys is slow and blocks, which is another reason to avoid big keys.

**Failover and availability.** Nodes gossip over a cluster bus (an additional TCP port). If a majority of primaries agree that a primary is unreachable, one of its replicas is promoted. If a primary with no remaining replica is lost, its slots become unavailable, and by default (`cluster-require-full-coverage yes`) the cluster stops accepting writes until slots are covered, a setting you may relax at the cost of partial availability. Replication is asynchronous in Cluster too, so the same loss caveats apply.

**Design implications.**
- Design keys and hash tags up front for the multi-key operations you need.
- Hot keys and big keys still live on one shard.
- Scripts must declare all keys and keep them in one slot.
- Pub/sub and some commands behave differently, so check the docs.
- Operational cost is higher: more nodes, more monitoring, a harder upgrade. Use Cluster when you need the scale, not by default. A single instance with replicas and Sentinel, or a managed service, is simpler for most workloads.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Inspecting the cluster | `CLUSTER INFO`, `CLUSTER NODES`, `CLUSTER SLOTS` (or `CLUSTER SHARDS`), `CLUSTER KEYSLOT key` | Map a key to its slot and node |
| Administration | `redis-cli --cluster check`, `reshard`, `rebalance`, `add-node`, `del-node`, `fix` | Add a node and rebalance safely in a test cluster |
| Client | ioredis `Redis.Cluster`, `scaleReads`, `natMap` and options | Configure a cluster client and handle redirects |
| Key design | hash tags | Keep related keys together without creating one hot slot |
| Diagnosing errors | `MOVED`, `ASK`, `CROSSSLOT`, `CLUSTERDOWN` | Explain what each means and the fix |

### Common mistakes

- Multi-key commands across slots (`CROSSSLOT`).
- One global hash tag that concentrates traffic on one node.
- Using a non-cluster client against a cluster.
- Ignoring that a transaction or script must be single-slot.
- Resharding with big keys during peak traffic.

### Self-check

- How many hash slots exist and how is a key's slot computed?
- What does a hash tag do and what is the risk of overusing it?
- Why does `MGET a b` fail in Cluster, and how do you fix it?
- What do `MOVED` and `ASK` mean?

---

## 14. Operating Redis: dangerous commands and monitoring

**In plain words.** Some commands are slow on big data and can freeze Redis for everyone. This chapter shows safe replacements and what to watch. Latency (how long a request takes) and memory use are the main warning signs.

### Understand

**O(N) commands and why they bite.** Because execution is single-threaded, a command whose cost grows with data size is dangerous on large data.

| Dangerous | Why | Use instead |
|---|---|---|
| `KEYS pattern` | Walks the entire keyspace in one go | `SCAN` with `MATCH` and `COUNT`, in a loop |
| `SMEMBERS` on a big set | Returns everything at once | `SSCAN`, or restructure |
| `HGETALL` on a big hash | Same | `HSCAN`, or `HMGET` for specific fields |
| `LRANGE 0 -1` on a big list | Same | Bounded ranges |
| `ZRANGE 0 -1` on a big sorted set | Same | Bounded ranges with `LIMIT` |
| `DEL` on a big key | Frees memory synchronously | `UNLINK` |
| `FLUSHALL`, `FLUSHDB` | Destroys data | Restrict via ACL, use the async option if intended |
| `SINTER`/`SUNION` of big sets | Cost depends on set sizes | Precompute, or bound sizes |
| `SORT` on large input | Sorts in the main thread | Maintain a sorted set instead |
| Long Lua scripts | Block like any command | Keep them short |
| `MONITOR` | Streams every command, large overhead | Short, controlled debugging only |

`SCAN` is cursor-based. Each call returns a cursor and a batch, and you continue until the cursor returns to 0. It does not block for long, but it gives no snapshot: elements added or removed during iteration may or may not appear, and the same element can be returned more than once, so handle duplicates. `COUNT` is a hint about work per call, not a guarantee on the number returned.

```ts
let cursor = '0';
do {
  const [next, keys] = await redis.scan(cursor, 'MATCH', 'v1:session:*', 'COUNT', 500);
  cursor = next;
  if (keys.length) await redis.unlink(...keys);
} while (cursor !== '0');
```

**Monitoring. What to watch.** Track these continuously, not only during incidents.

- **Latency:** client-side command latency percentiles, plus `redis-cli --latency`. Use `LATENCY DOCTOR` and `LATENCY LATEST` (need `latency-monitor-threshold` set) for server-side event analysis.
- **Slow commands:** `SLOWLOG GET 10` shows the slowest recent commands, with execution time in microseconds (it measures command execution, not network or queueing). Configure `slowlog-log-slower-than` (microseconds) and `slowlog-max-len`. Review regularly.
- **Memory:** `used_memory`, `maxmemory`, `mem_fragmentation_ratio`, `evicted_keys`.
- **Throughput and hit rate:** `instantaneous_ops_per_sec`, `keyspace_hits` and `keyspace_misses`, `expired_keys`.
- **Clients:** `connected_clients`, `blocked_clients`, `rejected_connections`, and `maxclients` headroom.
- **Persistence:** `rdb_last_bgsave_status`, `aof_last_write_status`, `latest_fork_usec`.
- **Replication:** role, `connected_slaves`, replica offsets and lag, `master_link_status` on replicas.
- **Per command:** `INFO commandstats` shows call counts and time per command, which reveals which command dominates.
- **Keyspace shape:** `INFO keyspace`, `redis-cli --bigkeys`, `--memkeys`, `--hotkeys`.

**`MONITOR` caveats.** It prints every command in real time, which is invaluable for seeing what an application sends, but it can reduce throughput substantially on a busy server, it exposes all data and credentials passed in commands (for example `AUTH`), and it stays on until the client disconnects. Use briefly, on a non-critical node, with care about sensitive data.

**Latency investigation order.** (1) Is it the network or the client (compare `redis-cli --latency` locally and remotely)? (2) Slow commands (`SLOWLOG`, `commandstats`). (3) Big keys and O(N) calls. (4) Fork pauses (`latest_fork_usec`, snapshot and rewrite times) and disk fsync with AOF. (5) Swapping (the fragmentation ratio below 1, `vmstat` on the host) and transparent huge pages. (6) Expiry bursts of many keys at once. (7) CPU saturation of the single thread (`top` shows the process near 100 percent). Use the Linux guide's one-minute checklist for the host.

**Operational hygiene.** Set `maxmemory`, a sensible `maxclients` and `timeout`, enable `latency-monitor`, store configuration in version control, test restore and failover, and upgrade deliberately with the release notes read.

### Tools to master

| Tool | Learn | Target level |
|---|---|---|
| `redis-cli` | interactive mode, `--scan`, `--bigkeys`, `--memkeys`, `--hotkeys`, `--latency`, `--stat`, `-n`, `--pipe`, `-x`, `--eval` | Do routine inspection and diagnosis from one tool |
| `INFO` | sections (`server`, `clients`, `memory`, `persistence`, `stats`, `replication`, `commandstats`, `keyspace`, `cluster`) | Read the fields listed above and explain each |
| `SLOWLOG` | `GET`, `LEN`, `RESET`, `slowlog-log-slower-than` | Find the commands that block the server |
| `LATENCY` | `LATENCY DOCTOR`, `LATENCY LATEST`, `LATENCY HISTORY` | Attribute a spike to fork, fsync, expiry or a command |
| `redis-benchmark` | `-c`, `-n`, `-P`, `-d`, `-t`, `--threads` (version dependent) | Produce a repeatable baseline before and after a change |
| RedisInsight | browse keys, workbench, slow log, memory analysis, profiler | Use the GUI to explore data and see command streams, with care in production |
| Metrics exporters | Prometheus `redis_exporter` or the cloud provider metrics | Build dashboards and alerts on the list above |

### Common mistakes

- Running `KEYS *` in production, even once, "just to look".
- Looking only at average latency. Percentiles and `SLOWLOG` show the spikes.
- Leaving `MONITOR` running.
- No alert for memory near `maxmemory` or for failing background saves.
- Treating `SLOWLOG` times as end-to-end latency.
- Running `redis-cli --bigkeys` on a busy primary with no awareness that it scans the whole keyspace (run it on a replica when possible).

### Self-check

- What replaces `KEYS` and what are the caveats of `SCAN`?
- Order the checks you would run when Redis latency doubles.
- What does `SLOWLOG` measure, and what does it not?
- Why is `MONITOR` risky on a production primary?

---

## 15. Security

**In plain words.** Redis was built for trusted networks and has little protection by default. Never expose it to the internet. Use passwords, ACLs (rules that say which user may run which commands) and TLS (encryption of network traffic).

### Understand

Redis was designed for use inside trusted networks. Its protocol is simple and powerful, and an exposed instance is a serious breach: attackers have used exposed Redis servers to read data, wipe data and, through configuration commands, to write files on the host.

**Do not expose Redis to the internet.**
- Bind to a private interface (`bind` setting) and keep the port closed in the host firewall and the cloud security group (the Linux guide's three layers: security group, host firewall, binding).
- Keep `protected-mode` enabled. It limits access when no password is set and the instance is reachable from outside loopback, but it is a last-resort safeguard, not a control to rely on.
- Place Redis in a private subnet reachable only from the application tier.

**Authentication.**
- Set a strong password. Legacy `requirepass` sets one for the default user.
- Redis 6 and later have **ACLs**: named users with their own passwords, allowed commands, allowed key patterns and allowed channels. Example (syntax verified against the ACL docs for your version):

```
ACL SETUSER app on >S3cretFromSecretManager ~app:* +@read +@write -@dangerous
ACL SETUSER readonly on >AnotherSecret ~* +@read
ACL LIST
```

- Give each application its own user restricted to its key prefix and to the commands it needs. Disable or restrict the default user.
- Keep passwords in a secret manager or a root-only environment file. Never commit them.
- Rotate credentials, and do not log connection URLs that contain passwords.

**Dangerous commands.** `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `KEYS`, `SHUTDOWN`, `REPLICAOF`, `MODULE`, `SCRIPT` and `EVAL` (where untrusted input could reach a script), `MONITOR`. Restrict them through ACL categories (`-@dangerous`) or specific denials. Older setups used `rename-command` for this, and ACLs are the current approach.

**Encryption in transit.** Redis supports TLS (`tls-port`, `tls-cert-file`, `tls-key-file`, `tls-ca-cert-file`, optional client certificates). Use TLS when traffic crosses any network you do not fully control, and between hosts in a cloud. Clients enable it with the `tls` option in ioredis, or a `rediss://` URL. Replication and Cluster traffic can be configured for TLS as well.

**Application-level concerns.**
- **Injection.** Never build commands or Lua scripts by concatenating untrusted input. Pass values as arguments (`ARGV`) and validate key components. Key names built from user input can collide with other keys or be used to read others' data (`user:${input}` where input contains `:`).
- **Sensitive data in cache.** Cached values are plain in memory and in RDB and AOF files. Do not cache secrets or regulated data without considering encryption, retention TTLs and access control. Backups inherit the sensitivity.
- **Multi-tenancy.** Key prefixes and ACLs separate tenants softly, but they share memory, CPU and the single thread. A noisy tenant affects everyone.
- **File permissions and OS user.** Run Redis as a non-root service user and restrict permissions on the data directory and configuration file, which may contain passwords.
- **Denial of service.** Unbounded key creation, huge values, expensive commands and many connections exhaust Redis. Use ACLs, `maxclients`, `maxmemory` and limits at the application layer.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Access control | `ACL SETUSER`, `ACL LIST`, `ACL WHOAMI`, `ACL LOG`, `ACL CAT` | Create a least-privilege user for one service |
| Network | `bind`, `protected-mode`, `ufw`/security groups, `ss -tulpn` | Prove the port is reachable only from the app tier |
| TLS | `tls-*` config, `redis-cli --tls`, ioredis `tls` option, `openssl s_client` | Enable TLS and verify the certificate chain |
| Secrets | secret managers, environment files, rotation | Keep credentials out of repos, logs and images |
| Auditing | `ACL LOG`, `CONFIG GET`, `CLIENT LIST` | See denied commands and who is connected |

### Common mistakes

- Redis bound to `0.0.0.0` with no password and an open security group.
- One shared password and the default user for every service.
- Leaving `CONFIG`, `FLUSHALL` and `DEBUG` available to application users.
- Passwords in URLs printed in logs.
- Assuming a private network needs no authentication or TLS.

### Self-check

- List the layers that keep a Redis instance off the internet.
- What does an ACL give you that `requirepass` does not?
- Why is an exposed Redis more dangerous than an exposed read-only service?
- How would you restrict a service to its own key prefix and no dangerous commands?

---

## 16. Using Redis from Node with ioredis

**In plain words.** ioredis is a Node.js library that talks to Redis. Most problems come from connection settings: timeouts, retries and sharing one client. Backoff (waiting longer after each failed attempt) prevents a flood of reconnects when Redis comes back.

### Understand

ioredis is a widely used Redis client for Node with support for Cluster, Sentinel, pipelines, transactions, Lua scripts and automatic reconnection. Redis clients differ in details, so confirm the options below against the ioredis documentation for your version.

**Connection model.** A Redis connection is a single TCP socket that carries commands in order. Node is single-threaded, so one shared client per process is normally enough. Commands from concurrent requests are pipelined over it automatically. Create the client once at startup and reuse it, never per request. Exceptions that need a separate connection:
- **Pub/sub subscribers.** A connection in subscriber mode cannot run normal commands. Use `redis.duplicate()` or a second client.
- **Blocking commands** (`BLPOP`, `XREADGROUP BLOCK`) hold the connection until they return, so they would delay other commands on that socket. Give blocking consumers their own connection.

```ts
import Redis from 'ioredis';

export const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  connectTimeout: 5_000,
  commandTimeout: 2_000,          // fail a stuck command instead of waiting forever
  maxRetriesPerRequest: 2,        // bounded: reject a queued command after N reconnect attempts
  retryStrategy: (times) => Math.min(times * 200, 2_000), // backoff capped at 2 s
  enableOfflineQueue: true,       // see below
});

redis.on('error', (err) => logger.error({ err }, 'redis error'));   // never leave unhandled
redis.on('reconnecting', () => logger.warn('redis reconnecting'));
redis.on('ready', () => logger.info('redis ready'));
```

**Error events.** An ioredis client is an event emitter. ioredis emits `error` events only when at least one listener exists, so without a listener errors are silent and the process does not crash. Always attach a handler to log and record metrics. Connection errors during an outage are emitted as events repeatedly while the client retries, so rate-limit the logging.

**Reconnection and retry.** On a connection loss ioredis reconnects using `retryStrategy`, a function that receives the attempt count and returns the delay in milliseconds (or a non-number to stop retrying). Use exponential or capped backoff, ideally with jitter, so that hundreds of clients do not reconnect in lockstep and flood a recovering server. The default policy retries indefinitely, so unbounded retry is the default, and you must consciously bound it for the application behaviour you want.

**Offline queue and bounded retries.** While disconnected, commands are queued in memory (`enableOfflineQueue`, on by default) and sent after reconnection. `maxRetriesPerRequest` bounds how many reconnect attempts a pending command survives before it fails (the default is 20 according to the ioredis README, and `null` disables the limit). Without bounds, a Redis outage becomes a memory growth and latency problem in your service, since request handlers hang on queued commands. Choose per use case:
- **Cache and rate limiting:** fail fast (low `commandTimeout`, small `maxRetriesPerRequest`) and degrade gracefully, so the request continues without cache.
- **Queue workers:** a blocking worker may reasonably retry indefinitely, since there is nothing else for it to do.
- **Reads and idempotent writes** are safe to retry. A non-idempotent command (`INCR`, `LPUSH`) may have executed before the connection dropped, so a blind retry can double-apply.

**Timeouts.** Set `connectTimeout`, and `commandTimeout` for request paths. Also keep application-level timeouts and circuit breakers, so a slow Redis does not tie up all request handlers. TCP keep-alive (`keepAlive` option) helps detect dead connections behind NAT or load balancers.

**Pipelines, transactions and scripts.**
```ts
const pipeline = redis.pipeline();
ids.forEach((id) => pipeline.get(`v1:user:${id}`));
const results = await pipeline.exec();      // [[err, value], ...] one per command

const tx = await redis.multi().incr('a').lpush('b', 'x').exec();
```
`exec()` on a pipeline returns an array of `[error, result]` pairs, and one command failing does not reject the whole call, so inspect each entry. Auto-pipelining (`enableAutoPipelining`) batches commands issued in the same event loop tick, with a trade-off in complexity.

**Cluster and Sentinel.** Use `new Redis.Cluster([...nodes], options)` for Cluster, which maintains the slot map and follows redirects. Use the `sentinels` and `name` options for Sentinel. In both, verify behaviour by failing a node in a test environment. Cloud and NAT setups may need address mapping (`natMap` for Cluster).

**Shutdown.** On `SIGTERM`, stop accepting work, then `await redis.quit()` (sends `QUIT` and waits for pending replies) so in-flight commands finish. `disconnect()` closes immediately. Remember the Linux guide on graceful shutdown.

**Data handling.** Redis returns strings. Serialise and parse explicitly, and handle parse failures from older cached shapes (versioned keys help). Use `set(key, value, 'EX', seconds)` for atomic value plus expiry, `unlink` for large deletes, `scanStream` for iteration, and `defineCommand` for Lua.

**Testing.** Use a real Redis in a container for integration tests, since Redis semantics (expiry, atomicity, Lua) are hard to mock faithfully. Reserve mocks for unit tests of cache wrappers. Flush a dedicated test database or prefix between tests, never against shared instances.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Connection config | ioredis `connectTimeout`, `commandTimeout`, `retryStrategy`, `maxRetriesPerRequest`, `enableOfflineQueue`, `keepAlive` | Configure bounded retries and timeouts for a stated failure policy |
| Events | `error`, `connect`, `ready`, `close`, `reconnecting`, `end` | Log and measure connection lifecycle |
| Batching | `pipeline()`, `multi()`, `enableAutoPipelining` | Batch safely and read per-command results |
| Scripts | `defineCommand`, `eval`, `evalsha` | Wrap a Lua script as a typed method |
| Iteration | `scanStream`, `sscanStream`, `hscanStream` | Iterate large keyspaces with bounded memory |
| HA clients | `Redis.Cluster`, Sentinel options | Survive failover in a test drill |
| Debugging | `redis.monitor()` (briefly), `DEBUG=ioredis:*` environment variable, RedisInsight | Trace what the client sends |

### Common mistakes

- Creating a new client per request, exhausting connections and file descriptors.
- No `error` handler, no timeouts, unbounded queue, so an outage freezes the service.
- Using one connection for subscribing and normal commands.
- Using a blocking command on the shared connection.
- Retrying non-idempotent commands blindly.
- Ignoring per-command errors in pipeline results.
- Treating Cluster as a single node, with multi-key commands across slots.

### Self-check

- Why is a single shared client usually enough, and when do you need more connections?
- What happens to commands issued while ioredis is disconnected, and how do you bound it?
- Why are retries with backoff and jitter important after an outage?
- How do you inspect a pipeline result for partial failure?
- How should a service shut down its Redis connection on `SIGTERM`?

---

## 17. When not to use Redis

**In plain words.** Redis is not the right tool for every job. It is a poor choice for data that must never be lost, or data larger than your RAM. Decide from your requirements, not from habit.

### Understand

Redis is excellent at what it does, and it is often adopted for things it does badly. Decide by the requirements, not familiarity.

**Do not use Redis as the primary store when:**
- **Data must be durable and correct.** Even with AOF `always`, replication is asynchronous and failover can lose acknowledged writes. Financial records, orders and user accounts belong in a transactional database. Use Redis in front of them.
- **The dataset exceeds affordable RAM.** Memory is the most expensive storage tier. Large, rarely accessed datasets belong on disk-based stores.
- **You need rich queries.** Joins, ad hoc filtering, secondary indexes, aggregations and full-text search are what relational databases and search engines do. Redis 8 includes search, JSON, probabilistic and time-series types in Redis Open Source (earlier they were modules), but you take on their operational cost, and availability differs in other distributions, so verify it for the one you run.
- **You need multi-key ACID transactions** with rollback and constraints. Redis transactions are isolated but do not roll back, and Cluster limits them to one slot.
- **You need a long-retention, high-volume event log with partitioning and replay at scale.** A dedicated log system fits better than a stream (chapter 10).
- **You need strong consistency across failover** or a coordination service with proper consensus. Use a consensus-based system.
- **A lock protects money or a hard invariant** (chapter 9).

**Redis fits well for:** caching, sessions, rate limiting and counters, leaderboards and rankings, short-lived coordination, real-time presence and fan-out, lightweight queues and streams where bounded loss is acceptable or configured for, deduplication sets, and geo lookups.

**Alternatives to consider.** In-process cache for tiny, single-instance needs (no network hop). The primary database's own features (materialised views, a cache table, advisory locks) when load is moderate. A message broker for durable messaging. A search engine for text. A relational or document store for data of record.

**Licensing and forks.** Redis 7.2 and earlier use the BSD 3-clause licence. From Redis 7.4 the source is under RSALv2 or SSPLv1 (source-available, not OSI-approved). Redis 8 added AGPLv3 as a third option. Valkey is a community fork of the BSD-licensed code with a compatible protocol and core commands. If licence terms matter to your organisation, check the licence of the exact version you plan to run, and this is not legal advice. Newer features and modules differ between Redis and Valkey, so verify the commands you depend on.

**Operational cost.** Each Redis instance is another system to monitor, secure, upgrade and back up, and another dependency in your failure analysis. If the database handles the load with an index, adding Redis may only add a consistency problem.

### Tools to master

| Concept | Tools | Target level |
|---|---|---|
| Deciding | requirements list: durability, size, query shape, consistency | Justify the choice in a design review |
| Sizing | `MEMORY USAGE` sampling, key count estimates | Estimate RAM for a dataset with growth |
| Alternatives | database features, message brokers, search engines | Name the better tool for a non-fit workload |

### Common mistakes

- Storing the only copy of important data in Redis.
- Adding a cache before measuring, then maintaining invalidation logic for no gain.
- Choosing Redis streams or pub/sub where a broker with durability is needed.
- Ignoring the cost of RAM when data grows.

### Self-check

- Give three workloads that suit Redis and three that do not, with reasons.
- A product manager wants the shopping cart stored only in Redis. What are the risks and mitigations?
- When is adding a cache the wrong first step for a slow endpoint?

---

## 18. Practice labs

**In plain words.** These labs let you try each idea on a local Redis. Try each lab before you read the answers elsewhere. Never run them on a shared or production server.

Do these against a local Redis (a container is the easiest way). Each lab states the goal and what "done" looks like. Attempt before looking anything up. Never run them against a shared or production instance.

1. **Type selection.** For six problems (counter, session object, top-100 leaderboard, daily active users, unique visitors, work queue) choose a type, write the commands, and state the complexity of each. Check `OBJECT ENCODING` and `MEMORY USAGE` on realistic data.
2. **Single-thread demonstration.** Fill a set with a million members. In one terminal run `SMEMBERS` on it, and in another run `redis-cli --latency`. Observe the stall. Then iterate with `SSCAN` and compare.
3. **Expiry mechanics.** Create 100000 keys with the same TTL and watch `INFO keyspace` and `expired_keys`. Repeat with jittered TTLs. Show that `SET` without `KEEPTTL` removes a TTL.
4. **Eviction.** Set a small `maxmemory`. Compare `noeviction`, `allkeys-lru` and `volatile-ttl` on the same workload and record which keys survive and which errors appear.
5. **Cache-aside with metrics.** Implement cache-aside for a slow function. Measure hit ratio, add delete-on-write, and demonstrate the stale-read race with a deliberate delay.
6. **Stampede.** Expire a hot key under concurrent load and count database calls. Add single-flight (in-process, then a `SET NX PX` lock), then jittered TTLs, and compare.
7. **Penetration.** Hit the service with random ids. Add negative caching and show the database call count drop.
8. **Rate limiters.** Implement fixed window, sliding window (sorted set in Lua) and token bucket. Test each with a concurrent load script and show the boundary burst in the fixed window.
9. **Lock correctness.** Implement `SET NX PX` and Lua release. Reproduce the failure where a plain `DEL` releases another holder's lock after expiry. Add a fencing token enforced by a conditional update in a database table.
10. **Transactions.** Show that `MULTI`/`EXEC` does not roll back by putting a failing command inside. Implement check-and-set with `WATCH`, then replace it with a Lua script under contention.
11. **Streams.** Build a producer and a consumer group with two workers. Kill one worker mid-job, then recover its pending entries with `XAUTOCLAIM`. Add a dead-letter stream after three failed attempts.
12. **Pub/sub loss.** Publish while a subscriber is disconnected and show the message is gone. Repeat with a stream and show it can be read.
13. **Persistence.** Write data, `kill -9` the server, and restart under each of: no persistence, RDB only, AOF `everysec`, AOF `always`. Record what survives. Run `redis-check-aof` on a truncated AOF.
14. **Replication and failover.** Start a primary and a replica, write continuously, stop the primary, promote the replica, and measure what was lost. Add Sentinel with three instances and observe automatic failover and client recovery.
15. **Cluster.** Create a local three-primary cluster. Trigger `CROSSSLOT`, fix it with a hash tag, add a node, and reshard while a load script runs.
16. **Slow command forensics.** Introduce a slow Lua script and a big key. Find them using `SLOWLOG`, `LATENCY`, `INFO commandstats` and `redis-cli --bigkeys`.
17. **ioredis outage drill.** Run a service with bounded retries and timeouts, stop Redis mid-load, and show it degrades gracefully, then recovers without a reconnect storm. Repeat with unbounded queueing to see the failure mode.
18. **Security baseline.** Configure an ACL user restricted to a prefix and denied dangerous commands, enable TLS, and prove with `ACL LOG` and a failed command that the restrictions work.
19. **Incident write-up.** Fill memory on a `noeviction` instance, observe the write errors, diagnose from `INFO` and `SLOWLOG` only, and write a one-page timeline: symptom, evidence, cause, fix, prevention.

---

## 19. Interview questions

**In plain words.** These questions test whether you understand how Redis works, not only its commands. Each question has a short model answer and a level tag: [Fresher], [Mid] or [Senior]. Practice answering aloud in your own words.

Grouped by level. Every question is followed by a concise model answer in the same section.

### Top 10 must-know

Start here. The numbers point to the full model answers below.

| # | Level | Question | Key point of the answer |
|---|---|---|---|
| 2 | [Fresher] | What does "single-threaded" imply? | Atomic commands, but one slow command blocks all clients |
| 5 | [Fresher] | How does Redis expire keys? | Lazy check on access plus an active background sampling cycle |
| 12 | [Fresher] | What are RDB and AOF? | Snapshot versus write log, with an fsync policy |
| 10 | [Fresher] | Pub/sub versus streams? | Pub/sub stores nothing; streams store, replay and acknowledge |
| 13 | [Mid] | Cache stampede and prevention? | Single-flight, early refresh, jitter, stale-while-revalidate |
| 17 | [Mid] | `MULTI`/`EXEC` versus pipeline versus Lua? | Round trips, isolation without rollback, atomic logic |
| 19 | [Mid] | A safe single-instance lock? | `SET NX PX` with unique token, Lua compare-and-delete |
| 21 | [Mid] | Persistence versus replication? | Local disk versus another node; neither is a backup |
| 30 | [Senior] | Lock holder pauses past the TTL? | Two holders; fix with fencing tokens checked by the resource |
| 32 | [Senior] | Latency spike investigation? | Network, SLOWLOG, big keys, fork, fsync, swap, expiry, CPU |

### Fresher

**1. [Fresher] Why is Redis fast?**
Data is in memory, structures are efficient, commands run on one thread with no locking, and an event loop with I/O multiplexing serves many connections without a thread per connection. The network round trip usually costs more than the command.

**2. [Fresher] What does "single-threaded" imply for how you use Redis?**
Commands execute one at a time, so each is atomic, but one slow command (`KEYS *`, `SMEMBERS` on a huge set, a long Lua script) blocks every client. Avoid O(N) commands on big data.

**3. [Fresher] Name the main Redis data types and one use for each.**
String (counter, cached JSON), hash (object with fields), list (simple queue, recent items), set (unique tags, membership), sorted set (leaderboard, time-ordered index), bitmap (daily active flags), HyperLogLog (approximate unique count), stream (event log with consumer groups), geo (nearby search).

**4. [Fresher] How do you set a key with an expiry in one atomic command?**
`SET key value EX 60` (or `PX` for milliseconds). Doing `SET` then `EXPIRE` risks a permanent key if the process dies between them.

**5. [Fresher] How does Redis expire keys?**
Lazily (a key found expired on access is deleted) and actively (a background cycle samples keys with a TTL and removes expired ones). Both are needed so reads are correct and memory is eventually reclaimed.

**6. [Fresher] What is the difference between `DEL` and `UNLINK`?**
`DEL` frees memory synchronously and can block on a big key. `UNLINK` removes the key immediately and frees memory in a background thread.

**7. [Fresher] Why not use `KEYS *` in production, and what do you use?**
It scans the whole keyspace in one blocking call. Use `SCAN` with `MATCH` and `COUNT`, looping until the cursor is 0, and handle possible duplicates.

**8. [Fresher] What is cache-aside?**
The application checks the cache, on a miss loads from the database and stores the result with a TTL, and on writes updates the database and deletes the cache key.

**9. [Fresher] What happens when a key is read after its TTL elapses?**
Redis treats it as missing, because expiry is checked on access, and deletes it.

**10. [Fresher] What is the difference between pub/sub and streams?**
Pub/sub is fire-and-forget: only currently connected subscribers receive a message and nothing is stored. Streams persist entries and support replay, consumer groups and acknowledgement.

**11. [Fresher] What does `INCR` do if the key does not exist?**
It creates it with value 0 and increments to 1. It is atomic, so concurrent calls never lose counts.

**12. [Fresher] What are RDB and AOF?**
RDB is a periodic point-in-time snapshot. AOF is a log of write commands replayed on startup, with an fsync policy (`always`, `everysec`, `no`) controlling durability.

### Mid-level

**13. [Mid] Explain cache stampede and how to prevent it.**
When a popular key expires, many requests miss together and all recompute, overloading the database. Prevent with single-flight (one rebuilder, others wait or serve stale), probabilistic early refresh, stale-while-revalidate, TTL jitter and pre-warming.

**14. [Mid] What are cache avalanche and cache penetration?**
Avalanche is mass expiry or loss of cache causing a wave of misses, fixed with TTL jitter, staggered warm-up and high availability. Penetration is repeated requests for non-existent keys that always reach the database, fixed with negative caching, validation, Bloom filters and rate limiting.

**15. [Mid] Why delete the cache on write rather than update it?**
Concurrent writers can update the cache in a different order than the database, leaving a stale value indefinitely. Deleting makes the next read reload the current database value. A TTL still bounds any remaining race.

**16. [Mid] Compare fixed-window, sliding-window and token-bucket rate limiting.**
Fixed window is cheapest but allows a double-rate burst at window edges. Sliding-window log (sorted set) is accurate but uses memory per request. Token bucket allows controlled bursts with a steady average rate and uses constant memory. All need atomic check-and-update, typically via `INCR` or Lua.

**17. [Mid] How does `MULTI`/`EXEC` differ from a pipeline and from a Lua script?**
A pipeline only batches round trips, with no isolation. `MULTI`/`EXEC` runs the queued commands without interleaving but has no rollback and cannot branch on intermediate results. A Lua script runs atomically with logic and reads, so it handles read-decide-write.

**18. [Mid] What does `WATCH` do?**
It marks keys for optimistic locking. If any watched key changes before `EXEC`, the transaction aborts (null reply) and the client retries.

**19. [Mid] How do you implement a safe single-instance lock?**
`SET key token NX PX ttl` with a unique random token to acquire, and a Lua script that deletes the key only if its value equals the token to release. Optionally extend the TTL with a similar script.

**20. [Mid] Choose an eviction policy for a cache and for a queue.**
Cache: `allkeys-lru` (or `allkeys-lfu`). Queue or locks: `noeviction`, so memory pressure appears as an error instead of silently deleting needed keys. Keep them on separate instances.

**21. [Mid] What is the difference between persistence and replication?**
Persistence writes data to local disk to survive a restart. Replication copies data to other running nodes for availability and read scale. Replication is asynchronous and copies mistakes, and neither replaces off-host backups.

**22. [Mid] What is replication lag and why does it matter?**
It is the delay between a primary write and its presence on a replica. Reads from replicas can be stale, breaking read-your-own-writes. A failover can also lose the unreplicated writes.

**23. [Mid] How do you find a slow command in production?**
`SLOWLOG GET` (execution time only), `INFO commandstats` for per-command totals, `LATENCY DOCTOR`, and `redis-cli --bigkeys` for big keys. Then correlate with application traces.

**24. [Mid] What are hash tags in Cluster and why use them?**
A `{tag}` in a key makes only the tagged part determine the slot, so related keys share a slot and can be used in multi-key commands, transactions and scripts. Overusing a single tag concentrates load on one node.

**25. [Mid] How would you configure ioredis to fail fast during an outage?**
Set `connectTimeout`, `commandTimeout`, a low `maxRetriesPerRequest`, a capped `retryStrategy` with backoff, an `error` handler, and decide on `enableOfflineQueue`, then degrade gracefully in the caller (for example, skip the cache).

**26. [Mid] Why does a pub/sub subscriber need its own connection?**
A connection in subscribe mode can only run subscription commands, so normal commands need another connection.

**27. [Mid] What does `mem_fragmentation_ratio` tell you?**
The ratio of OS-resident memory to allocated memory. Much above 1 suggests allocator fragmentation. Below 1 suggests parts of Redis memory were swapped out, which seriously hurts latency.

**28. [Mid] What are the failure modes of BullMQ-style queues if Redis is misconfigured?**
With no persistence a restart loses pending jobs, with RDB only you lose recent changes, with eviction enabled queue keys can be deleted and corrupt the queue, and with asynchronous replication a failover can drop recent jobs. Use `noeviction`, AOF, replication and idempotent jobs.

### Senior

**29. [Senior] Explain the Redlock controversy and your position on using Redis locks.**
Redlock acquires a lock on a majority of independent instances to survive a node failure. Kleppmann argued it relies on timing assumptions (bounded delay, pauses, clock drift) and lacks fencing tokens, so it cannot guarantee safety. The author of Redis replied that the assumptions are reasonable. The pragmatic position: use Redis locks for efficiency (avoiding duplicate work), and never as the only guard for correctness. Protect money and invariants in the database with transactions, constraints, idempotency keys, or fencing enforced by the resource.

**30. [Senior] A lock holder pauses past the TTL. Describe the failure and the fix.**
The lock expires, another process acquires it, and the first resumes and writes believing it still holds the lock, so two processes are in the critical section. The fix is a fencing token (monotonic number per acquisition) that the protected resource checks, rejecting writes with an older token, plus idempotent operations.

**31. [Senior] Design a rate limiter for 50,000 requests per second across many app servers.**
Per-client keys (user or API key), token bucket or sliding-window counter implemented in one Lua script for atomicity, TTLs on keys, Redis Cluster with the key as the hash input so limiters shard, a short client timeout with an explicit fail-open or fail-closed decision, 429 with `Retry-After`, a coarse edge limit in front, and optionally a small local pre-limit to reduce Redis load. Monitor Redis CPU, since the single thread per shard is the limit.

**32. [Senior] Redis latency has spiked. Walk through the investigation.**
Separate network from server (`redis-cli --latency` locally and remotely), then `SLOWLOG` and `commandstats` for slow or heavy commands, big keys and O(N) calls, fork pauses (`latest_fork_usec`, snapshots, AOF rewrite), fsync with AOF, swap and transparent huge pages, bursts of expiry, connection storms, and CPU saturation of the main thread. Fix the cause (restructure data, move work, add shards) and add alerts.

**33. [Senior] How do replication and failover affect data safety, and how do you reduce loss?**
Replication is asynchronous, so a failover loses writes not yet replicated, and a partitioned old primary can accept writes that are discarded. Reduce with `WAIT`, `min-replicas-to-write` and `min-replicas-max-lag`, AOF, and idempotent clients. Do not claim durability beyond this. For stronger guarantees, keep the data of record in a transactional database.

**34. [Senior] When would you choose Cluster over Sentinel, and what does it cost?**
Choose Cluster when one node's CPU (single-threaded commands) or memory is insufficient and the data can be partitioned by key. It costs multi-key restrictions (`CROSSSLOT`), hash tag design, single-slot transactions and scripts, harder operations and a cluster-aware client. If the data fits on one node, Sentinel or a managed HA offering is simpler.

**35. [Senior] How do you handle a hot key?**
Detect it (`--hotkeys` with LFU, metrics, a short `MONITOR`). Mitigate with an in-process cache with short TTL, replicating the key under several suffixed names and reading randomly, read replicas where staleness is acceptable, and splitting counters into shards that are summed on read.

**36. [Senior] Choose between pub/sub, a list, a stream and a dedicated broker for an order-processing pipeline.**
Orders must not be lost, so not pub/sub. A plain list loses an item if a worker crashes after popping it. A stream with consumer groups gives at-least-once delivery, acknowledgement, recovery of pending entries and dead-lettering, if Redis is configured for durability (AOF, replication) and handlers are idempotent. If you need long retention, strict durability, multi-consumer replay at scale or cross-team contracts, use a dedicated broker or log, and keep the order in the database with an outbox.

**37. [Senior] Design a cache for a read-heavy product catalogue with updates from several services.**
Cache-aside with versioned keys and TTL with jitter, delete on write plus event-driven invalidation from the catalogue service (pub/sub or a broker, with TTL as backstop), single-flight on rebuild, negative caching for unknown ids, a local cache for the hottest items, metrics on hit ratio and miss latency, a graceful fallback with a circuit breaker, and `allkeys-lru` with `maxmemory` set below the container limit.

**38. [Senior] How do you make a queue on Redis safe enough for payment webhooks?**
Do not make Redis the only record. Persist the event to the database first (outbox), enqueue a job referencing it, run AOF (`everysec` or `always`) with replication and `noeviction`, make handlers idempotent using an idempotency key with a unique constraint, retry with backoff, dead-letter after a limit, reconcile periodically against the database to recover lost jobs, and alert on queue age and failures.

**39. [Senior] What are the operational risks of RDB snapshots and AOF rewrites on a large instance?**
`fork` can pause the server in proportion to the memory page-table size, and copy-on-write can double memory use under heavy writes, risking OOM, while disk I/O can contend with AOF fsync. Mitigate with memory headroom, snapshots on replicas, scheduling, monitoring `latest_fork_usec` and disk throughput, and tuning kernel settings Redis warns about.

**40. [Senior] How do you secure a Redis deployment in a cloud environment?**
Private subnet, no public exposure, security group and host firewall limited to the application tier, `bind` to a private address, per-service ACL users restricted by key pattern and command categories with `-@dangerous`, TLS in transit, secrets in a secret manager with rotation, no passwords in logs, non-root service account, encrypted backups with retention, and audit through `ACL LOG` and monitoring.

---

## 20. References

Primary sources, preferred over blog posts:

- Redis documentation: https://redis.io/docs/latest/
- Redis commands reference (complexity and version notes per command): https://redis.io/docs/latest/commands/
- Redis data types overview: https://redis.io/docs/latest/develop/data-types/
- Redis persistence, replication, Sentinel, Cluster and security pages are under the management section of the documentation: https://redis.io/docs/latest/operate/oss_and_stack/management/ (the section index page loads, but its sub-page paths were not individually checked)
- Redis distributed locks pattern (Redlock): https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/
- Martin Kleppmann, "How to do distributed locking": https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- Salvatore Sanfilippo, "Is Redlock safe?": https://antirez.com/news/101
- Redis Lua scripting and replication: https://redis.io/docs/latest/develop/programmability/eval-intro/
- redis-cli reference: https://redis.io/docs/latest/develop/tools/cli/
- Redis licensing announcement (2024): https://redis.io/blog/redis-adopts-dual-source-available-licensing/
- Vattani, Chierichetti, Lowenstein, "Optimal Probabilistic Cache Stampede Prevention", VLDB 2015
- ioredis repository and README (options, Cluster, Sentinel, pipelines, scripting): https://github.com/redis/ioredis
- BullMQ documentation (queues, workers, stalled jobs, going to production): https://docs.bullmq.io
- Valkey project documentation (compatible fork): https://valkey.io/docs/
- Martin Kleppmann, *Designing Data-Intensive Applications* (book), chapters on replication, consistency and distributed locks
- Brendan Gregg, the USE method, for host-level diagnosis: https://www.brendangregg.com/usemethod.html
- Companion chapter in this repository: `03_Linux.md`

Verify any URL before depending on it, since pages move.

---

## 21. Glossary

**In plain words.** This table explains the hardest words in this guide in simple language. Return to it whenever a term is unclear.

| Term | Plain meaning |
|---|---|
| Atomic | Done as one step; no other client can act in the middle |
| AOF | Append-only file: a log of every write, replayed at startup |
| RDB | A snapshot file: a full copy of the data at one moment |
| fsync | A call that forces data from the OS buffer onto the disk |
| fork | Creating a child process that shares memory with its parent |
| Copy-on-write | Shared memory is copied only when one side changes it |
| TTL | Time to live: seconds left before a key is deleted |
| Eviction | Removing keys automatically when memory is full |
| LRU / LFU | Least recently used / least frequently used: rules to choose which key to evict |
| Fragmentation | Memory the allocator holds but cannot reuse |
| Pipelining | Sending many commands without waiting for each reply |
| Lua script | A small program that runs inside Redis as one atomic unit |
| Idempotent | Safe to run twice; the result is the same as running once |
| Stampede | Many requests rebuilding the same expired cache value together |
| Jitter | A small random amount added to a time value |
| Fencing token | A growing number that the protected resource checks to reject stale holders |
| Redlock | An algorithm that takes a lock on a majority of independent Redis nodes |
| Consumer group | Workers that share the entries of one stream |
| PEL | Pending Entries List: entries delivered but not yet acknowledged |
| Replica | A copy of a primary node, usually read-only |
| Replication lag | Delay before a write on the primary appears on a replica |
| Failover | Switching to a replica when the primary fails |
| Quorum | The number of Sentinels that must agree a primary is down |
| Hash slot | One of 16384 numbered groups that Cluster uses to place keys |
| Hash tag | Text in braces in a key; only it decides the slot |
| Big key | A key holding a very large value or collection |
| Hot key | A key that receives a very large share of traffic |
| ACL | Access control list: per-user rules for commands and keys |
| TLS | Encryption of network traffic |
| Backoff | Waiting longer after each failed attempt before retrying |
