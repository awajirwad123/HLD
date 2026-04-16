# Caching — Notes

## One-Line Summary

> Cache = store expensive results closer to the reader; trade consistency for speed.

---

## Cache Strategy Decision Tree

```
Do writes need the cache to be immediately consistent?
  YES → Write-Through (write DB + cache together)
  NO  → Cache-Aside (write DB, invalidate cache)

Is write latency the bottleneck?
  YES → Write-Back (write cache first, async flush to DB)
  NO  → Cache-Aside or Write-Through
```

---

## Strategy Cheat Sheet

| Strategy       | When to use                                | Risk                        |
|----------------|--------------------------------------------|-----------------------------|
| Cache-aside    | Default for most read-heavy systems        | Stale data until invalidated |
| Write-through  | Read-after-write consistency needed        | Slow writes (2 ops)         |
| Write-back     | Write-heavy, can tolerate data loss        | Data loss on crash          |
| Read-through   | ORM/library-managed caching                | Same as cache-aside         |

---

## Eviction Policy Selection

| Policy           | Redis setting          | Use when                                |
|------------------|------------------------|-----------------------------------------|
| LRU              | `allkeys-lru`          | General purpose cache (default choice)  |
| LFU              | `allkeys-lfu`          | Trending/viral content — frequency > recency |
| Volatile-LRU     | `volatile-lru`         | Mix of cached + permanent keys in Redis |
| TTL-based        | `volatile-ttl`         | Short-TTL keys should die first         |
| No eviction      | `noeviction`           | **Never use for caches** — causes write errors |

**Always set in production:**
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

---

## Cache Invalidation Patterns

| Pattern                    | Consistency  | Complexity | Use case                       |
|----------------------------|-------------|------------|--------------------------------|
| TTL expiry                 | Eventual     | Low        | Product catalog, articles      |
| Delete on write            | Near-real-time | Low      | User profiles, settings        |
| Event-driven invalidation  | Near-real-time | High     | Inventory, pricing, financial  |
| Versioned keys             | Strong (by design) | Medium | CDN assets, config             |

---

## Cache Stampede Prevention

| Method                  | How it works                                   | Best for                     |
|-------------------------|------------------------------------------------|------------------------------|
| Distributed lock (mutex)| Only 1 worker recomputes; others wait          | Any critical hot key         |
| Probabilistic early refresh| Refresh before TTL expires (random window) | Gradual, no thundering herd  |
| Stale-while-revalidate  | Serve stale data; refresh async                | Low-consistency-tolerance reads |
| Background refresh      | Cron job refreshes hot keys before expiry      | Known-hot keys (homepage)    |

---

## Redis vs Memcached

| Feature              | Redis                              | Memcached                    |
|----------------------|------------------------------------|------------------------------|
| Data structures      | Strings, hashes, lists, sets, sorted sets | Strings only          |
| Persistence          | RDB snapshots + AOF logs           | None (pure in-memory)        |
| Pub/Sub              | Yes                                | No                           |
| Cluster support      | Native (Redis Cluster)             | Client-side sharding only    |
| TTL per key          | Yes                                | Yes                          |
| Atomic operations    | Yes (INCR, LPUSH, ZADD)            | Basic (CAS)                  |
| Multi-threading      | Single-threaded (I/O multiplexed)  | Multi-threaded               |
| Use Memcached when   | —                                  | Pure string cache, max throughput on simple ops |
| **Choose Redis by default** | ✅ Almost always               | Rarely                       |

---

## Key Numbers

| Metric                              | Value                        |
|-------------------------------------|------------------------------|
| Redis GET/SET latency (same DC)     | < 1ms                        |
| Redis throughput                    | ~100K–200K ops/sec (single)  |
| Redis Cluster throughput            | Near-linear scale per shard  |
| Typical cache hit ratio target      | > 90% for read-heavy systems |
| In-process cache latency            | ~microseconds (no network)   |
| Cache miss penalty                  | DB latency (10–100ms)        |
| LRU eviction check overhead         | Negligible (O(1))            |
| Redis default max memory            | No limit (dangerous!)         |

---

## What to Say in Every Interview That Uses Caching

Whenever you add a cache to your HLD, proactively cover:

1. **Strategy**: "I'll use cache-aside — DB is source of truth, cache accelerates reads"
2. **TTL**: "TTL of 5 minutes — acceptable staleness for this data type"
3. **Eviction**: "allkeys-lru with 4GB cap on Redis"
4. **Invalidation**: "On any product price update, we delete the cache key immediately"
5. **Stampede**: "For hot keys, I'd use a distributed lock on cache miss to prevent thundering herd"
6. **Failure mode**: "If Redis is unavailable, we fall back to the DB — cache is optional"
7. **Hit rate target**: "Expect >90% hit rate given this access pattern — reduces DB QPS by ~90%"
