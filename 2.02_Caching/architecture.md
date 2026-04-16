# Caching — Architecture

## Overview

Caching is the single highest-leverage optimization in distributed systems. A well-designed cache can reduce database load by 90%, cut latency from 50ms to < 1ms, and allow systems to serve 10–100x more traffic with the same infrastructure.

> **Core idea:** Store the result of an expensive operation so the next request gets it for free.

---

## 1. Cache Types

### In-Process Cache (Local Cache)

```
[ API Server Process ]
    ├── Application logic
    └── [ In-memory dict / LRU cache ]   ← lives inside the same process
```

- Data stored in application memory (RAM of the server)
- Fastest possible access (~microseconds, no network)
- **Problem:** Not shared between server instances — Server A's cache is invisible to Server B
- **Problem:** Cache lost on server restart or crash
- **Use when:** Single-server apps, CPU-bound computations (e.g., parsed config), session-local data

### Distributed Cache

```
[ Server A ] ──┐
[ Server B ] ──┼──► [ Redis / Memcached Cluster ]   ← shared cache
[ Server C ] ──┘
```

- All server instances share the same cache
- Consistent view across the entire fleet
- Network hop required (~0.5–1ms)
- **Redis** — supports strings, hashes, lists, sets, sorted sets, TTL, pub/sub, Lua scripting
- **Memcached** — simpler, pure key-value, multi-threaded, slightly faster for simple ops
- **Use when:** Horizontally scaled stateless services, session stores, rate limiters, leaderboards

### CDN as Cache

- Edge nodes cache HTTP responses at network level
- Acts as cache for static assets and cacheable API responses
- Covered in Topic 2 (Networking Basics)

---

## 2. Cache Strategies (Write Policies)

This is the most important section — interviewers test this heavily.

### Strategy 1: Cache-Aside (Lazy Loading) ★ Most Common

```
Read path:
  App → check cache
          │ HIT  → return data
          │ MISS → read from DB → write to cache → return data

Write path:
  App → write to DB → invalidate (delete) cache key
```

**Properties:**
- Cache is populated lazily — only when data is requested
- Cache miss = 3 operations (read miss + DB read + cache write) → slightly higher latency on first access
- **Cache is always optional** — DB is source of truth; cache just accelerates reads
- Most resilient: if cache goes down, system falls back to DB

**Implementation:**
```python
def get_user(user_id: str) -> dict:
    key = f"user:{user_id}"
    cached = redis.get(key)
    if cached:
        return json.loads(cached)              # cache HIT

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(key, 3600, json.dumps(user))  # cache MISS → populate
    return user

def update_user(user_id: str, data: dict):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.delete(f"user:{user_id}")            # invalidate cache
```

**When to use:** Read-heavy systems, data that can tolerate brief staleness, most general-purpose caching.

---

### Strategy 2: Write-Through

```
Write path:
  App → write to cache AND write to DB (synchronously, both before returning)

Read path:
  App → check cache → always HIT (cache always has latest data)
```

**Properties:**
- Cache is always in sync with DB (no stale reads)
- Writes are slower — two writes per operation
- Cache is always warm — reads are fast immediately after write
- Wasted cache space: data written to cache that may never be read

**Use when:** Write latency is acceptable, and read consistency is critical (e.g., user profile that's read immediately after update).

---

### Strategy 3: Write-Back (Write-Behind)

```
Write path:
  App → write to cache immediately → return to client (fast!)
                │
                └──► [ async queue ] ──► DB write (later)

Read path:
  App → cache HIT (always fresh in cache)
```

**Properties:**
- **Fastest writes** — client gets response before DB is updated
- Risk: data loss if cache crashes before async write completes
- DB may temporarily be out of sync with cache
- Complexity: need reliable async write queue

**Use when:** Write-heavy workloads where write latency matters more than durability (e.g., analytics counters, view counts, gaming leaderboards).

**Real-world:** Redis as a write buffer for InfluxDB (metrics), social media like counts (eventually persisted to DB in batches).

---

### Strategy 4: Read-Through

```
Read path:
  App → requests from cache library only
  Cache library: HIT → return / MISS → fetch from DB → populate → return
  App never talks to DB directly
```

- Abstraction layer handles all cache logic
- Same pattern as cache-aside but the cache library manages the DB fallback
- Used by ORM cache layers (Hibernate 2nd level cache, Django cache framework)

---

### Strategy Comparison

| Strategy      | Read latency | Write latency | Consistency | Data loss risk | Best for           |
|---------------|-------------|--------------|-------------|---------------|--------------------|
| Cache-aside   | Slow on miss | Normal       | Eventual    | None (DB is truth) | General reads  |
| Write-through | Fast (always warm) | Slow (2 writes) | Strong | None      | Read-after-write   |
| Write-back    | Fast        | Fastest      | Eventual    | Yes (cache crash) | Write-heavy    |
| Read-through  | Slow on miss | Normal       | Eventual    | None          | ORM/library-level  |

---

## 3. Cache Eviction Policies

When cache is full, which keys to remove?

### LRU (Least Recently Used) ★ Default for most systems

```
Access order: A → B → C → A → D
Cache full, must evict: remove B (least recently accessed)
```
- Evicts the item that hasn't been accessed for the longest time
- Good proxy for "least likely to be needed again"
- Redis implements as approximated LRU (samples 5 random keys, evicts oldest)

### LFU (Least Frequently Used)

```
Access counts: A=100, B=2, C=50, D=1
Must evict: remove D (fewest accesses)
```
- Evicts the item accessed fewest times over its lifetime
- Better for items with high initial burst but then go cold (trending content)
- Redis 4.0+ supports LFU via `maxmemory-policy allkeys-lfu`

### TTL (Time To Live)

```
redis.setex("product:123", 3600, data)  # expires in 1 hour
```
- Item automatically expires after a set duration
- Not an eviction competing with LRU — it's a freshness guarantee
- Use TTL always — even alongside LRU — to prevent stale data living forever

### FIFO / Random

- FIFO: evict oldest inserted (regardless of access)
- Random: evict a random key
- Rarely the right choice — use LRU or LFU

---

## 4. Cache Invalidation (The Hard Problem)

> *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

### Invalidation Strategies

**1. TTL Expiry — simplest**
- Set a reasonable TTL; data is eventually consistent
- Risk: stale data shown until TTL expires
- Good for: product catalog, news articles, user profiles (minutes of staleness OK)

**2. Event-Driven Invalidation — stronger**
```
Event: product price updated
  → publish "product:123:updated" event
  → cache invalidation service subscribes
  → deletes cache key "product:123"
```
- Near-real-time consistency
- Complexity: need a reliable event bus
- Good for: inventory levels, pricing (where staleness has business consequences)

**3. Cache-through invalidation (on write)**
```python
def update_product(product_id, data):
    db.update(product_id, data)
    redis.delete(f"product:{product_id}")   # invalidate immediately
```
- Simplest for write paths you control
- Cache miss on next read → DB fetches fresh data → re-populates

**4. Versioned cache keys**
```python
# Instead of invalidating, change the key
cache_key = f"user:{user_id}:v{user.version}"
```
- Old key is orphaned (expires via TTL), new key is always fresh
- No invalidation needed — new version = new key = cache miss → fresh data
- Used by CDN URL versioning (`app.v2.1.js`)

---

## 5. Cache Stampede (Thundering Herd)

One of the most dangerous cache failure modes in production.

### The Problem

```
TTL expires for popular key "popular_product:999"
Simultaneously, 10,000 requests arrive (all cache MISS)
All 10,000 hit the DB at once
DB overloaded → cascade failure
```

### Solutions

**1. Probabilistic early expiration (PER)**
```python
import random, time

def get_with_per(key: str, ttl: int, recompute_fn):
    data, expiry = redis.get_with_expiry(key)
    remaining_ttl = expiry - time.time()
    # Exponentially increasing probability of early refresh as TTL approaches 0
    if data is None or random.random() > (remaining_ttl / ttl):
        data = recompute_fn()
        redis.setex(key, ttl, data)
    return data
```

**2. Mutex / Distributed Lock**
```python
def get_with_lock(key: str, recompute_fn):
    cached = redis.get(key)
    if cached:
        return cached

    lock_key = f"lock:{key}"
    acquired = redis.set(lock_key, "1", nx=True, ex=10)  # atomic NX lock

    if acquired:
        # This worker does the DB query
        data = recompute_fn()
        redis.setex(key, 3600, data)
        redis.delete(lock_key)
        return data
    else:
        # Other workers wait and retry
        time.sleep(0.05)
        return redis.get(key)   # hopefully populated by now
```

**3. Background refresh (stale-while-revalidate)**
```python
# Serve stale data immediately; refresh in background before TTL expires
# Redis: store with a "soft TTL" shorter than actual TTL
# On soft TTL hit → trigger background refresh, return stale data
```

---

## 6. Cache Warming Strategies

A cold cache = all traffic hits DB. Plan for it.

```
Scenarios requiring cache warming:
  - New server deployment (local cache is empty)
  - Redis cluster restart or failover
  - After a planned cache flush (bug fix, data corruption)
```

**Strategies:**
1. **Pre-warm on startup** — background job reads top-N most accessed keys from DB and populates cache before server accepts traffic
2. **Read-through warm** — accept cold start, LB health check delays until cache is warm enough
3. **Traffic replay** — replay recent production traffic against new cluster
4. **Consistent hashing minimizes re-warming** — when adding cache nodes, only ~1/N keys need to be re-fetched (vs flushing entire cache on topology change)

---

## 7. Redis Architecture Patterns

### Standalone
Single Redis instance. Simple. SPOF. For dev/small workloads.

### Sentinel (High Availability)
```
[ Redis Primary ] ──► automatic failover ──► [ Redis Replica becomes Primary ]
[ Sentinel 1,2,3 ] — monitors and orchestrates failover
```
- Automatic failover within 10–30 seconds
- No data sharding — one primary handles all writes
- Good for: moderate scale, HA without sharding

### Cluster (Horizontal Scale)
```
16,384 hash slots split across 3+ primary nodes:
  Primary 1: slots 0–5460    (+ 1 replica)
  Primary 2: slots 5461–10922 (+ 1 replica)
  Primary 3: slots 10923–16383 (+ 1 replica)
```
- Horizontal write scaling — each primary owns a partition
- `CLUSTER KEYSLOT user:123` → routes to correct shard automatically
- Good for: datasets > single machine RAM, very high write throughput

---

## 8. Cache at Every Layer of the Stack

```
Browser cache       → HTTP Cache-Control (seconds to minutes)
CDN edge cache      → HTTP Cache-Control (minutes to hours/days)
API gateway cache   → response cache (seconds to minutes)
In-process cache    → application memory (microseconds, hot data)
Distributed cache   → Redis (milliseconds, shared)
DB query cache      → PostgreSQL's internal buffer pool (automatic)
```

Each layer has different TTL, consistency, and granularity. Caching closest to the user delivers the most latency savings.
