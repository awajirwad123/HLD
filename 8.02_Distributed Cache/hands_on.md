# Distributed Cache — Hands-On Exercises

## Exercise 1: Implement LRU and LFU Cache in Python

```python
"""
Implement LRU Cache (O(1) get/put) and LFU Cache from scratch.
Understand why Redis uses these as eviction policies.
"""

from collections import OrderedDict, defaultdict
from typing import Optional


# ─── LRU Cache ────────────────────────────────────────────────────────────────

class LRUCache:
    """
    O(1) get and put using OrderedDict (doubly-linked hashmap).
    Most recently used = end; least recently used = front.
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self._cache: OrderedDict[str, str] = OrderedDict()

    def get(self, key: str) -> Optional[str]:
        if key not in self._cache:
            return None
        self._cache.move_to_end(key)  # Mark as recently used
        return self._cache[key]

    def put(self, key: str, value: str) -> Optional[str]:
        """Returns evicted key (if any), None otherwise."""
        evicted = None
        if key in self._cache:
            self._cache.move_to_end(key)
        else:
            if len(self._cache) >= self.capacity:
                evicted, _ = self._cache.popitem(last=False)  # Evict LRU (front)
            self._cache[key] = value
        return evicted

    def __repr__(self):
        items = list(self._cache.items())
        return f"LRU(LRU→MRU): {items}"


# ─── LFU Cache ────────────────────────────────────────────────────────────────

class LFUCache:
    """
    O(1) get and put using frequency buckets.
    Maintains: key→value, key→freq, freq→OrderedDict[key]
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self._key_val:  dict[str, str] = {}
        self._key_freq: dict[str, int] = {}
        self._freq_keys: dict[int, OrderedDict] = defaultdict(OrderedDict)
        self._min_freq = 0

    def _increment_freq(self, key: str):
        freq = self._key_freq[key]
        self._key_freq[key] = freq + 1

        # Remove from current freq bucket
        del self._freq_keys[freq][key]
        if not self._freq_keys[freq] and self._min_freq == freq:
            self._min_freq += 1

        # Add to next freq bucket
        self._freq_keys[freq + 1][key] = None

    def get(self, key: str) -> Optional[str]:
        if key not in self._key_val:
            return None
        self._increment_freq(key)
        return self._key_val[key]

    def put(self, key: str, value: str) -> Optional[str]:
        if self.capacity == 0:
            return None

        evicted = None
        if key in self._key_val:
            self._key_val[key] = value
            self._increment_freq(key)
        else:
            if len(self._key_val) >= self.capacity:
                # Evict least-frequently-used, then LRU among ties
                lfu_key, _ = next(iter(self._freq_keys[self._min_freq].items()))
                del self._freq_keys[self._min_freq][lfu_key]
                del self._key_val[lfu_key]
                del self._key_freq[lfu_key]
                evicted = lfu_key

            self._key_val[key] = value
            self._key_freq[key] = 1
            self._freq_keys[1][key] = None
            self._min_freq = 1

        return evicted


# ─── Demo ─────────────────────────────────────────────────────────────────────

print("=== LRU Cache Demo ===")
lru = LRUCache(capacity=3)
lru.put("a", "1"); lru.put("b", "2"); lru.put("c", "3")
print("After putting a, b, c:", lru)
lru.get("a")               # Access 'a' → a becomes MRU
print("After get(a):", lru)
print("Evicted on put(d):", lru.put("d", "4"))  # Should evict 'b' (LRU)
print("After put(d):", lru)

print("\n=== LFU Cache Demo ===")
lfu = LFUCache(capacity=3)
lfu.put("a", "1"); lfu.put("b", "2"); lfu.put("c", "3")
lfu.get("a"); lfu.get("a")  # a: freq=3
lfu.get("b")                # b: freq=2
# c: freq=1 → will be evicted next
print("Evicted on put(d):", lfu.put("d", "4"))  # Should evict 'c'
```

---

## Exercise 2: Cache-Aside Pattern with FastAPI

```python
"""
Standard cache-aside (lazy loading) pattern:
  - Read: cache hit → return; miss → DB → populate cache → return
  - Write: write to DB, then DELETE the cache key (not update)

Why DELETE not UPDATE on write?
  Race condition: thread A reads DB, thread B writes newer value + deletes cache,
  thread A writes stale value to cache → stale data survives.
  DELETE ensures next read gets fresh DB value.
"""

import asyncpg
import redis.asyncio as redis
import json
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

DB_DSN    = "postgresql://user:pass@localhost/app"
REDIS_URL = "redis://localhost:6379"
CACHE_TTL = 300  # 5 minutes


@app.on_event("startup")
async def startup():
    app.state.db    = await asyncpg.create_pool(DB_DSN)
    app.state.cache = redis.from_url(REDIS_URL, decode_responses=True)


class UserUpdate(BaseModel):
    name: str
    email: str


@app.get("/users/{user_id}")
async def get_user(user_id: int):
    rdb: redis.Redis       = app.state.cache
    pool: asyncpg.Pool     = app.state.db

    cache_key = f"user:{user_id}"

    # ── Cache read ──────────────────────────────────────────────────────────
    cached = await rdb.get(cache_key)
    if cached:
        return json.loads(cached)

    # ── DB read on miss ─────────────────────────────────────────────────────
    row = await pool.fetchrow("SELECT id, name, email FROM users WHERE id = $1", user_id)
    if not row:
        raise HTTPException(404, "User not found")

    user_dict = dict(row)

    # ── Populate cache ──────────────────────────────────────────────────────
    await rdb.set(cache_key, json.dumps(user_dict), ex=CACHE_TTL)

    return user_dict


@app.put("/users/{user_id}")
async def update_user(user_id: int, body: UserUpdate):
    rdb: redis.Redis   = app.state.cache
    pool: asyncpg.Pool = app.state.db

    # ── Write to DB ─────────────────────────────────────────────────────────
    result = await pool.execute(
        "UPDATE users SET name = $1, email = $2 WHERE id = $3",
        body.name, body.email, user_id,
    )
    if result == "UPDATE 0":
        raise HTTPException(404, "User not found")

    # ── Invalidate cache (DELETE, not update) ────────────────────────────────
    await rdb.delete(f"user:{user_id}")

    return {"status": "updated"}
```

---

## Exercise 3: Distributed Lock with Redis (Redlock Pattern)

```python
"""
Redis-based distributed lock for critical sections.
Redlock: acquire lock on N/2+1 Redis nodes to be safe against single-node failure.
Simple version: acquire lock on single node with SET NX EX.
"""

import asyncio
import time
import uuid
import redis.asyncio as redis


class RedisLock:
    """
    Simple single-node distributed lock using SET NX EX.
    Use for non-critical locks where single Redis node failure is acceptable.
    """

    def __init__(self, rdb: redis.Redis, key: str, ttl_seconds: int = 10):
        self._rdb = rdb
        self._key = f"lock:{key}"
        self._ttl = ttl_seconds
        self._token = str(uuid.uuid4())   # Unique token: only owner can release

    async def acquire(self, timeout: float = 5.0) -> bool:
        """Try to acquire the lock, retrying for up to `timeout` seconds."""
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            # SET key token NX EX ttl  — atomic, returns True if set
            acquired = await self._rdb.set(
                self._key, self._token, nx=True, ex=self._ttl
            )
            if acquired:
                return True
            await asyncio.sleep(0.05)   # Back off 50ms
        return False

    async def release(self):
        """Release lock only if we own it (compare-and-delete via Lua)."""
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        await self._rdb.eval(lua_script, 1, self._key, self._token)

    async def __aenter__(self):
        acquired = await self.acquire()
        if not acquired:
            raise TimeoutError(f"Could not acquire lock {self._key}")
        return self

    async def __aexit__(self, *args):
        await self.release()


# ─── Usage example ────────────────────────────────────────────────────────────

async def process_payment(payment_id: str, rdb: redis.Redis):
    """Ensure payment is processed exactly once even with concurrent requests."""
    async with RedisLock(rdb, f"payment:{payment_id}", ttl_seconds=30):
        # Check idempotency key
        already_done = await rdb.get(f"payment:done:{payment_id}")
        if already_done:
            return {"status": "already_processed"}

        # Process payment (DB write, external API call, etc.)
        await asyncio.sleep(0.1)  # Simulate work

        # Mark as done
        await rdb.set(f"payment:done:{payment_id}", "1", ex=86400)
        return {"status": "processed"}


# ─── Demo ─────────────────────────────────────────────────────────────────────

async def demo_lock():
    rdb = redis.from_url("redis://localhost:6379")

    results = await asyncio.gather(*[
        process_payment("pay-001", rdb)
        for _ in range(5)  # 5 concurrent attempts for the same payment
    ])

    unique_processed = sum(1 for r in results if r["status"] == "processed")
    print(f"Processed exactly once: {unique_processed == 1}")  # Should be True
    await rdb.aclose()
```

---

## Exercise 4: Cache Stampede Prevention

```python
"""
Demonstrate the thundering herd problem and three prevention strategies.
"""

import asyncio
import time
import random
import redis.asyncio as redis


rdb = redis.from_url("redis://localhost:6379", decode_responses=True)


async def fake_db_query(key: str) -> str:
    """Simulate expensive DB query (50ms)."""
    await asyncio.sleep(0.05)
    return f"value_for_{key}_{int(time.time())}"


# ── Strategy 1: Naïve (causes stampede) ───────────────────────────────────

async def get_naive(key: str) -> str:
    val = await rdb.get(key)
    if val:
        return val
    val = await fake_db_query(key)  # All concurrent callers hit DB simultaneously
    await rdb.set(key, val, ex=5)
    return val


# ── Strategy 2: Mutex lock (prevents stampede, adds latency) ──────────────

async def get_with_lock(key: str) -> str:
    val = await rdb.get(key)
    if val:
        return val

    lock_key = f"fill_lock:{key}"
    acquired = await rdb.set(lock_key, "1", nx=True, ex=5)

    if acquired:
        try:
            val = await fake_db_query(key)
            await rdb.set(key, val, ex=30)
            return val
        finally:
            await rdb.delete(lock_key)
    else:
        # Wait for the lock holder to fill the cache
        for _ in range(20):
            await asyncio.sleep(0.05)
            val = await rdb.get(key)
            if val:
                return val
        # Fallback: go to DB directly if lock holder failed
        return await fake_db_query(key)


# ── Strategy 3: Probabilistic early expiry (XFetch algorithm) ─────────────

BETA = 1.0  # Tuning parameter: higher = refresh more eagerly

async def get_with_xfetch(key: str, ttl: int = 30) -> str:
    """
    XFetch: proactively refresh before expiry using probabilistic check.
    P(refresh) increases as key approaches expiration.
    """
    pipe = rdb.pipeline()
    pipe.get(key)
    pipe.ttl(key)
    val, remaining_ttl = await pipe.execute()

    if val is None or remaining_ttl < 0:
        val = await fake_db_query(key)
        await rdb.set(key, val, ex=ttl)
        return val

    # XFetch decision: refresh early with probability that increases near expiry
    elapsed = ttl - remaining_ttl
    if elapsed > 0:
        recompute_time = 0.05  # Estimated DB query time in seconds (50ms)
        if -recompute_time * BETA * (random.random() + 1e-9) > -remaining_ttl:
            # Refresh early (this fires more often as TTL shrinks)
            val = await fake_db_query(key)
            await rdb.set(key, val, ex=ttl)

    return val


# ─── Compare strategies ───────────────────────────────────────────────────────

async def benchmark_strategy(label: str, fn, key: str, n_concurrent: int = 50):
    db_calls = []

    original_fake = globals()["fake_db_query"]
    call_count = 0

    async def counting_db(k):
        nonlocal call_count
        call_count += 1
        return await original_fake(k)

    # Expire the key to simulate cold start
    await rdb.delete(key)

    start = time.perf_counter()
    await asyncio.gather(*[fn(key) for _ in range(n_concurrent)])
    elapsed = time.perf_counter() - start

    print(f"{label}: {n_concurrent} concurrent callers, elapsed={elapsed:.3f}s")
```
