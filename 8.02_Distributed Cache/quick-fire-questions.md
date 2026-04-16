# Distributed Cache — Quick-Fire Questions

**Q1: Why is Redis single-threaded? How does it handle high concurrency?**
Redis executes commands in a single thread using an I/O multiplexing event loop (epoll). Commands take microseconds (O(1)/O(log N)), so the CPU rarely blocks. The bottleneck is network bandwidth, not CPU. Since Redis 6.0, I/O threads handle socket reads/writes in parallel, while command execution remains single-threaded (no locking needed → atomic operations by design).

---

**Q2: What is cache-aside vs write-through?**
Cache-aside (lazy loading): on read miss, populate cache from DB. On write, delete the cache key — next read repopulates. Write-through: on every write, update both cache and DB synchronously. Cache-aside gives better read performance for rarely-written data; write-through guarantees cache freshness but adds write latency.

---

**Q3: Why do you DELETE the cache key on write instead of updating it?**
Race condition: Thread A reads DB (slow), Thread B writes new value + DELETEs cache. Thread A finishes and writes the stale old value to cache — now cache has stale data until TTL expires. With DELETE-on-write, the next read always fetches fresh data from DB.

---

**Q4: What is the thundering herd / cache stampede problem?**
When a popular cached key expires simultaneously, thousands of concurrent requests all miss the cache and hit the DB at the same time. Solutions: mutex lock (one thread fills, others wait), probabilistic early refresh (XFetch algorithm: refresh with increasing probability as TTL dwindles), or staggered TTLs (add random jitter: `TTL + random(0, 60)` to prevent mass simultaneous expiries).

---

**Q5: How does Redis Cluster assign keys to nodes?**
`HASH_SLOT = CRC16(key) mod 16384`. Each node owns a range of hash slots. The client maintains a routing table and sends commands directly to the correct node. On resharding, slots migrate between nodes — clients get `MOVED` redirects during migration.

---

**Q6: What is `allkeys-lru` and when would you use `allkeys-lfu` instead?**
`allkeys-lru` evicts the key that was accessed least recently. Good when recent access predicts future access. `allkeys-lfu` evicts the key accessed least frequently overall. Better for stable hotspot patterns (popular content that stays popular). LFU can keep popular-but-not-recently-accessed keys alive; LRU cannot.

---

**Q7: What's the difference between Redis Sentinel and Redis Cluster?**
Sentinel: HA without sharding. Single primary + replicas; Sentinel monitors and promotes replica on failure (~30s failover). Use when dataset fits in one node. Cluster: both HA and sharding. 16,384 hash slots distributed across N primaries, each with replicas. Use when you need horizontal scale or >1M ops/sec.

---

**Q8: What's the danger of using `KEYS *` in production?**
`KEYS *` is O(N) over all keys and runs synchronously on the single event loop thread — it blocks all other commands for the entire scan duration. On a Redis instance with 50M keys, `KEYS *` can stall Redis for seconds. Use `SCAN cursor MATCH pattern COUNT 100` instead — it's O(1) per call and incremental.

---

**Q9: What is a distributed lock and how do you implement it in Redis?**
`SET key uuid NX EX 10` — atomic: set only if not exists, with 10-second TTL. Release: Lua script `if GET key == uuid then DEL key end` — ensures only the lock owner releases it (prevents releasing another thread's lock after TTL expiry and re-acquisition). For critical operations across multiple Redis nodes, use the Redlock algorithm (acquire on majority of N nodes).

---

**Q10: What does `WAIT 2 500` do in Redis?**
Blocks the current connection until at least 2 replicas have acknowledged the latest write, or 500ms timeout elapses. Turns async replication semi-synchronous for a specific write. Used for critical paths where you can't afford to lose data (e.g., after writing an idempotency key before charging a credit card).

---

**Q11: What are Hash Tags and why are they needed in Redis Cluster?**
By default, every key hashes independently — `user:1:profile` and `user:1:orders` may land on different nodes. `MGET user:1:profile user:1:orders` across different nodes is not atomic. Hash tags: `{user:1}.profile` and `{user:1}.orders` — only the content inside `{}` is hashed, ensuring both map to the same slot, enabling atomic multi-key operations.

---

**Q12: Explain the write-behind (write-back) pattern and its risks.**
Write-behind: writes go to cache immediately and return to the caller. A background worker asynchronously flushes dirty cache entries to the DB. Risk: if the Redis node crashes before flush, those writes are lost. Mitigation: use AOF persistence on Redis so the cache itself is durable. Use when write latency matters more than durability (e.g., counters, leaderboard scores).
