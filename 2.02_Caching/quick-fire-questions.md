# Caching — Quick-Fire Questions

*Short answers for rapid revision.*

---

**Q1: What is the purpose of a cache?**
Store the result of an expensive operation (DB query, API call, computation) so future requests get it cheaply without repeating the work.

---

**Q2: What is the difference between in-process and distributed cache?**
In-process: lives inside one server's RAM — microsecond access, not shared across instances. Distributed: lives in a separate service (Redis) — 1ms network hop, shared by all servers.

---

**Q3: What is cache-aside (lazy loading)?**
On read: check cache first; on miss, read from DB and populate cache. On write: update DB and delete/invalidate the cache key. Cache is populated lazily.

---

**Q4: What is write-through caching?**
On every write, update both the DB and the cache synchronously before returning. Cache is always warm; reads after writes are fast. Writes are slower (two operations).

---

**Q5: What is write-back (write-behind) caching?**
Write to cache immediately (fast response), then asynchronously flush to DB later. Fastest write latency, but risks data loss if cache crashes before DB write completes.

---

**Q6: When would you choose write-back over write-through?**
When write throughput is the bottleneck and slight data loss is acceptable — e.g., view counters, analytics events, leaderboard scores.

---

**Q7: What is LRU eviction?**
Least Recently Used — when cache is full, evict the item that was accessed least recently. O(1) with a doubly-linked list + hashmap. Redis default.

---

**Q8: What is LFU eviction and when is it better than LRU?**
Least Frequently Used — evicts the item accessed fewest times. Better than LRU when viral/trending content has high initial burst then goes cold (LRU would keep it; LFU would evict it sooner).

---

**Q9: What is a TTL in cache context?**
Time To Live — automatic expiry after N seconds. Ensures stale data doesn't live forever. Always set a TTL — even if your eviction policy is LRU.

---

**Q10: What is a cache stampede (thundering herd)?**
When a popular cached item expires, thousands of simultaneous requests all get a cache miss, all hit the DB at once, overwhelming it. Solved by mutex locks, probabilistic early refresh, or stale-while-revalidate.

---

**Q11: How does a distributed lock prevent cache stampede?**
First request to get a cache miss acquires a Redis lock (atomic `SET NX`). It recomputes and populates the cache. All other concurrent requests wait, then read from the now-populated cache.

---

**Q12: What is cache invalidation?**
Removing or updating a cached entry when the underlying data changes, so stale data isn't served. The hardest problem in caching.

---

**Q13: Name 3 cache invalidation strategies.**
1. TTL expiry (time-based). 2. Delete on write (invalidate key when data changes). 3. Event-driven invalidation (subscribe to change events, delete affected keys).

---

**Q14: What is a versioned cache key?**
Embed a version number in the key (e.g., `product:123:v5`). On data change, increment version → old key is orphaned and expires via TTL. New reads get a miss → fresh data → new versioned key populated.

---

**Q15: What is the default Redis maxmemory-policy? Why is it dangerous?**
`noeviction` — Redis returns an error when memory is full instead of evicting keys. Dangerous for caches: it causes write failures rather than gracefully dropping old cache entries.

---

**Q16: What Redis eviction policy should you always set in production caches?**
`allkeys-lru` (evict any key by LRU) with a `maxmemory` limit. Never leave `noeviction` on a cache.

---

**Q17: What is the difference between Redis Sentinel and Redis Cluster?**
Sentinel: HA for a single primary — auto-failover to replica on crash. No sharding.
Cluster: horizontal scaling — shards data across multiple primaries. Both HA and sharding.

---

**Q18: If Redis goes down, what should happen to your system?**
Fall back to the DB — slower but correct. Cache is always optional in cache-aside; DB is source of truth. Design the system to degrade gracefully, not fail completely.

---

**Q19: What is the typical target cache hit rate?**
> 90% for read-heavy systems. Below 80% — investigate your key design, TTL, or eviction policy.

---

**Q20: Name the 3 caching layers closest-to-furthest from the user.**
Browser cache (seconds–minutes) → CDN edge cache (minutes–hours) → distributed cache / Redis (milliseconds).
