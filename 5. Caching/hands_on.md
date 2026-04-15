# Caching — Hands-On

## Exercise 1: Cache-Aside Pattern with FastAPI + Redis

Full implementation of the most common caching pattern:

```python
import json
import redis
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Simulated DB
_db: dict[str, dict] = {
    "user:1": {"id": "1", "name": "Alice", "email": "alice@example.com"},
    "user:2": {"id": "2", "name": "Bob",   "email": "bob@example.com"},
}

CACHE_TTL = 300  # 5 minutes

def get_user_from_db(user_id: str) -> dict | None:
    """Simulates a slow DB query."""
    import time; time.sleep(0.05)   # 50ms DB latency
    return _db.get(f"user:{user_id}")

# ── READ: Cache-Aside ────────────────────────────────────────────────────────
@app.get("/users/{user_id}")
def get_user(user_id: str):
    cache_key = f"user:{user_id}"

    # 1. Try cache first
    cached = r.get(cache_key)
    if cached:
        return {"source": "cache", "data": json.loads(cached)}

    # 2. Cache miss → go to DB
    user = get_user_from_db(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")

    # 3. Populate cache
    r.setex(cache_key, CACHE_TTL, json.dumps(user))
    return {"source": "db", "data": user}


# ── WRITE: Invalidate on update ──────────────────────────────────────────────
class UserUpdate(BaseModel):
    name: str | None = None
    email: str | None = None

@app.patch("/users/{user_id}")
def update_user(user_id: str, update: UserUpdate):
    if f"user:{user_id}" not in _db:
        raise HTTPException(status_code=404, detail="User not found")

    # 1. Update DB
    _db[f"user:{user_id}"].update(update.model_dump(exclude_none=True))

    # 2. Invalidate cache — next read will re-populate from DB
    deleted = r.delete(f"user:{user_id}")
    return {
        "status": "updated",
        "cache_invalidated": bool(deleted),
        "data": _db[f"user:{user_id}"]
    }


# ── DEMO stats endpoint ──────────────────────────────────────────────────────
@app.get("/cache/stats")
def cache_stats():
    info = r.info("stats")
    return {
        "hits": info.get("keyspace_hits", 0),
        "misses": info.get("keyspace_misses", 0),
        "hit_rate": round(
            info.get("keyspace_hits", 0) /
            max(info.get("keyspace_hits", 0) + info.get("keyspace_misses", 0), 1) * 100, 2
        )
    }
```

---

## Exercise 2: Write-Through vs Cache-Aside — Benchmark Comparison

```python
import time
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Fake DB
db: dict[str, dict] = {}

def slow_db_write(key: str, value: dict):
    time.sleep(0.01)  # 10ms DB write latency
    db[key] = value

def slow_db_read(key: str) -> dict | None:
    time.sleep(0.05)  # 50ms DB read latency
    return db.get(key)


# ── Cache-Aside write ────────────────────────────────────────────────────────
def write_cache_aside(key: str, value: dict):
    slow_db_write(key, value)              # write DB only
    r.delete(key)                          # invalidate cache

def read_cache_aside(key: str) -> dict:
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    data = slow_db_read(key)
    if data:
        r.setex(key, 60, json.dumps(data))
    return data


# ── Write-Through ────────────────────────────────────────────────────────────
def write_through(key: str, value: dict):
    slow_db_write(key, value)              # write DB
    r.setex(key, 60, json.dumps(value))    # also write cache (sync)

def read_write_through(key: str) -> dict:
    cached = r.get(key)
    if cached:
        return json.loads(cached)
    return slow_db_read(key)


# ── Benchmark ────────────────────────────────────────────────────────────────
def benchmark(label: str, write_fn, read_fn, key: str, value: dict):
    # Write
    t0 = time.perf_counter()
    write_fn(key, value)
    write_ms = (time.perf_counter() - t0) * 1000

    # Read 1 (cold or miss)
    t0 = time.perf_counter()
    read_fn(key)
    read_cold_ms = (time.perf_counter() - t0) * 1000

    # Read 2 (warm)
    t0 = time.perf_counter()
    read_fn(key)
    read_warm_ms = (time.perf_counter() - t0) * 1000

    print(f"\n{label}")
    print(f"  Write:      {write_ms:.1f}ms")
    print(f"  Read cold:  {read_cold_ms:.1f}ms")
    print(f"  Read warm:  {read_warm_ms:.1f}ms")

r.flushdb()
benchmark("Cache-Aside", write_cache_aside, read_cache_aside, "item:1", {"price": 42})
benchmark("Write-Through", write_through, read_write_through, "item:2", {"price": 99})
```

**Expected output:**
```
Cache-Aside
  Write:       10ms   (DB only)
  Read cold:   51ms   (DB miss + populate)
  Read warm:    1ms   (cache hit)

Write-Through
  Write:       11ms   (DB + cache, but cache is async ~1ms extra)
  Read cold:    1ms   (cache already warm from write!)
  Read warm:    1ms   (cache hit)
```

---

## Exercise 3: LRU Cache Implementation from Scratch

Understand the data structure before relying on Redis:

```python
from collections import OrderedDict

class LRUCache:
    """
    O(1) get and put using OrderedDict (doubly-linked list + hash map).
    Most recently used item is at the END; LRU is at the START.
    """
    def __init__(self, capacity: int):
        self.capacity = capacity
        self._cache: OrderedDict[str, any] = OrderedDict()
        self.hits = 0
        self.misses = 0

    def get(self, key: str):
        if key not in self._cache:
            self.misses += 1
            return None
        self._cache.move_to_end(key)   # mark as recently used
        self.hits += 1
        return self._cache[key]

    def put(self, key: str, value) -> bool:
        """Returns True if eviction occurred."""
        evicted = False
        if key in self._cache:
            self._cache.move_to_end(key)
        else:
            if len(self._cache) >= self.capacity:
                oldest = next(iter(self._cache))   # first item = LRU
                del self._cache[oldest]
                evicted = True
        self._cache[key] = value
        return evicted

    def hit_rate(self) -> float:
        total = self.hits + self.misses
        return round(self.hits / total * 100, 1) if total else 0.0

    def __repr__(self):
        return f"LRU({list(self._cache.keys())}) hit_rate={self.hit_rate()}%"


# Demo
cache = LRUCache(capacity=3)
for key in ["a", "b", "c", "a", "d", "b"]:
    cache.get(key) or cache.put(key, f"val_{key}")
    print(cache)

# Output shows eviction order and hit rate
```

---

## Exercise 4: Distributed Lock for Cache Stampede Prevention

```python
import redis
import time
import json
import threading

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def expensive_db_query(key: str) -> dict:
    """Simulates a slow DB operation (200ms)."""
    print(f"  [DB] Computing {key}...")
    time.sleep(0.2)
    return {"key": key, "value": "computed", "ts": time.time()}

def get_with_mutex(key: str) -> dict:
    """Cache-aside with distributed lock to prevent stampede."""
    # 1. Try cache
    cached = r.get(key)
    if cached:
        print(f"  [CACHE HIT] {key}")
        return json.loads(cached)

    # 2. Cache miss — try to acquire lock
    lock_key = f"lock:{key}"
    acquired = r.set(lock_key, "1", nx=True, ex=5)  # NX = only if not exists

    if acquired:
        try:
            # Winner: compute and populate
            data = expensive_db_query(key)
            r.setex(key, 60, json.dumps(data))
            print(f"  [LOCK WINNER] Populated {key}")
            return data
        finally:
            r.delete(lock_key)
    else:
        # Loser: wait and retry
        for _ in range(10):
            time.sleep(0.05)
            cached = r.get(key)
            if cached:
                print(f"  [LOCK WAITER] Got {key} after waiting")
                return json.loads(cached)
        # Fallback: just query DB directly (lock holder may have crashed)
        return expensive_db_query(key)


# Simulate stampede: 10 concurrent requests for same key
r.delete("hot_key")   # ensure cold cache

threads = [threading.Thread(target=get_with_mutex, args=("hot_key",)) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
# Only ONE [DB] line should print — the rest get the cached result
```

---

## Exercise 5: Cache Eviction Policy Comparison (Redis)

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Check current eviction policy
print("Current policy:", r.config_get("maxmemory-policy"))

# Common policies:
# allkeys-lru     — evict any key, LRU order (most common)
# volatile-lru    — evict only keys WITH TTL set, LRU order
# allkeys-lfu     — evict any key, LFU order (good for frequency-based)
# volatile-ttl    — evict key with shortest remaining TTL first
# noeviction      — return error when memory full (default — dangerous!)

# Recommended for caches:
r.config_set("maxmemory-policy", "allkeys-lru")
r.config_set("maxmemory", "100mb")

# Test: write many keys and observe eviction
for i in range(1000):
    r.setex(f"key:{i}", 3600, json.dumps({"data": "x" * 100}))

print(f"Keys stored: {r.dbsize()}")
# Under 100mb limit: all keys present
# Over limit: LRU keys are evicted automatically
```

**Interview insight:**
> *"I always set `maxmemory-policy allkeys-lru` and `maxmemory` on production Redis. The default `noeviction` policy turns Redis into a system that starts failing writes when full — that's worse than evicting old cache entries."*
