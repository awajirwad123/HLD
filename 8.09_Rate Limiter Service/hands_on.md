# Rate Limiter Service — Hands-On Exercises

## Exercise 1: Four Rate Limiting Algorithms Side by Side

```python
import time
import math
from collections import deque


# ─────────────────────────────────────────
# Algorithm 1: Fixed Window Counter
# ─────────────────────────────────────────

class FixedWindowRateLimiter:
    """Simple counter reset at each fixed time window."""

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self._counts: dict[str, tuple[int, int]] = {}  # key → (count, window_id)

    def is_allowed(self, key: str) -> tuple[bool, int]:
        """Returns (allowed, remaining)."""
        window_id = int(time.time()) // self.window
        count, stored_window = self._counts.get(key, (0, window_id))

        if stored_window != window_id:
            count = 0  # New window: reset

        if count < self.limit:
            self._counts[key] = (count + 1, window_id)
            return True, self.limit - count - 1
        return False, 0


# ─────────────────────────────────────────
# Algorithm 2: Sliding Window Log
# ─────────────────────────────────────────

class SlidingWindowLogRateLimiter:
    """Exact sliding window: stores every request timestamp."""

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self._logs: dict[str, deque] = {}

    def is_allowed(self, key: str) -> tuple[bool, int]:
        now = time.time()
        cutoff = now - self.window
        log = self._logs.setdefault(key, deque())

        # Remove expired timestamps
        while log and log[0] <= cutoff:
            log.popleft()

        if len(log) < self.limit:
            log.append(now)
            return True, self.limit - len(log)
        return False, 0


# ─────────────────────────────────────────
# Algorithm 3: Sliding Window Counter
# ─────────────────────────────────────────

class SlidingWindowCounterRateLimiter:
    """Approximate sliding window using two fixed-window counters."""

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        # key → {prev_count, curr_count, window_id}
        self._state: dict[str, dict] = {}

    def is_allowed(self, key: str) -> tuple[bool, int]:
        now = time.time()
        curr_window = int(now) // self.window
        elapsed_in_window = now % self.window
        overlap = (self.window - elapsed_in_window) / self.window

        s = self._state.get(key, {"prev": 0, "curr": 0, "window": curr_window})

        if s["window"] != curr_window:
            # Advance window: current becomes previous
            s = {"prev": s["curr"] if s["window"] == curr_window - 1 else 0,
                 "curr": 0,
                 "window": curr_window}

        effective_count = s["curr"] + s["prev"] * overlap

        if effective_count < self.limit:
            s["curr"] += 1
            self._state[key] = s
            return True, max(0, int(self.limit - effective_count - 1))
        return False, 0


# ─────────────────────────────────────────
# Algorithm 4: Token Bucket
# ─────────────────────────────────────────

class TokenBucketRateLimiter:
    """Allows bursts up to capacity; refills at rate tokens/second."""

    def __init__(self, capacity: int, rate: float):
        """capacity = max burst size; rate = tokens added per second."""
        self.capacity = capacity
        self.rate = rate
        # key → (tokens, last_refill_time)
        self._buckets: dict[str, tuple[float, float]] = {}

    def is_allowed(self, key: str, cost: int = 1) -> tuple[bool, float]:
        """Returns (allowed, tokens_remaining)."""
        now = time.time()
        tokens, last_refill = self._buckets.get(key, (float(self.capacity), now))

        # Refill
        elapsed = now - last_refill
        tokens = min(self.capacity, tokens + elapsed * self.rate)

        if tokens >= cost:
            tokens -= cost
            self._buckets[key] = (tokens, now)
            return True, tokens
        return False, tokens


# ─────────────────────────────────────────
# Comparison Demo
# ─────────────────────────────────────────

def demo_algorithm(name: str, limiter, requests: list[tuple[float, str]]):
    """
    requests: list of (timestamp_offset_seconds, key) pairs
    simulates requests arriving at given offsets from t=0
    """
    print(f"\n{'='*50}")
    print(f"{name}")
    print(f"{'='*50}")
    start = time.time()
    allowed = rejected = 0

    for offset, key in requests:
        # Busy-wait until the proper time (for demo only)
        while time.time() < start + offset:
            pass

        ok, remaining = limiter.is_allowed(key)
        status = "ALLOW" if ok else "DENY "
        elapsed = time.time() - start
        print(f"  t={elapsed:5.2f}s  [{status}]  key={key}  remaining={remaining}")
        if ok:
            allowed += 1
        else:
            rejected += 1

    print(f"\n  Allowed: {allowed}  Rejected: {rejected}")


def main():
    import time

    LIMIT = 5
    WINDOW = 3  # 3 second window for fast demo

    # Test scenario: 8 requests, some clustered near window boundary
    # Offsets in seconds from test start
    requests = [
        (0.0, "user1"), (0.1, "user1"), (0.2, "user1"),
        (0.3, "user1"), (0.4, "user1"),
        # These 5 are in first window → all allowed
        (0.5, "user1"), (0.6, "user1"),
        # These 2 should be rejected (over limit of 5)
        (3.1, "user1"),
        # New window → allowed
    ]

    print(f"Limit: {LIMIT} requests per {WINDOW}s window")
    print(f"Sending {len(requests)} requests for 'user1'\n")

    demo_algorithm(
        "Fixed Window Counter",
        FixedWindowRateLimiter(LIMIT, WINDOW),
        requests
    )
    demo_algorithm(
        "Sliding Window Log",
        SlidingWindowLogRateLimiter(LIMIT, WINDOW),
        requests
    )
    demo_algorithm(
        "Sliding Window Counter",
        SlidingWindowCounterRateLimiter(LIMIT, WINDOW),
        requests
    )
    demo_algorithm(
        "Token Bucket (capacity=5, rate=5/3s)",
        TokenBucketRateLimiter(capacity=LIMIT, rate=LIMIT / WINDOW),
        requests
    )


if __name__ == "__main__":
    main()
```

---

## Exercise 2: Redis-Backed Distributed Rate Limiter (Sliding Window Counter)

```python
import asyncio
import time
import hashlib
from dataclasses import dataclass

import redis.asyncio as aioredis

# Lua script: atomic sliding window counter check + increment
SLIDING_WINDOW_SCRIPT = """
local key_curr = KEYS[1]
local key_prev = KEYS[2]
local now_ms   = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local limit     = tonumber(ARGV[3])

local window_start_ms = now_ms - (now_ms % window_ms)
local elapsed_ms = now_ms - window_start_ms
local overlap = (window_ms - elapsed_ms) / window_ms

local curr = tonumber(redis.call('GET', key_curr) or 0)
local prev = tonumber(redis.call('GET', key_prev) or 0)
local effective = curr + prev * overlap

if effective < limit then
    redis.call('INCR', key_curr)
    redis.call('PEXPIRE', key_curr, window_ms * 2)
    return {1, math.floor(effective + 1), math.floor(limit - effective - 1)}
else
    return {0, math.floor(effective), 0}
end
"""


@dataclass
class RateLimitResult:
    allowed: bool
    current_count: int
    remaining: int
    limit: int
    key: str

    @property
    def retry_after(self) -> int | None:
        """Seconds until next window (for 429 response header)."""
        if not self.allowed:
            # Approximate: time to end of current second
            return 1
        return None

    def headers(self) -> dict:
        h = {
            "X-RateLimit-Limit": str(self.limit),
            "X-RateLimit-Remaining": str(max(0, self.remaining)),
        }
        if not self.allowed and self.retry_after:
            h["Retry-After"] = str(self.retry_after)
        return h


class RedisRateLimiter:
    def __init__(self, redis_client: aioredis.Redis, window_seconds: int = 60):
        self.redis = redis_client
        self.window_ms = window_seconds * 1000
        self._script_sha: str | None = None

    async def _load_script(self):
        if self._script_sha is None:
            self._script_sha = await self.redis.script_load(SLIDING_WINDOW_SCRIPT)

    def _keys(self, identifier: str) -> tuple[str, str]:
        """Generate current and previous window keys."""
        now_ms = int(time.time() * 1000)
        window_id = now_ms // self.window_ms
        h = hashlib.sha256(identifier.encode()).hexdigest()[:8]
        curr_key = f"rl:{h}:{window_id}"
        prev_key = f"rl:{h}:{window_id - 1}"
        return curr_key, prev_key

    async def check(self, identifier: str, limit: int) -> RateLimitResult:
        """Check rate limit. Returns RateLimitResult."""
        await self._load_script()
        now_ms = int(time.time() * 1000)
        curr_key, prev_key = self._keys(identifier)

        result = await self.redis.evalsha(
            self._script_sha, 2,
            curr_key, prev_key,
            now_ms, self.window_ms, limit
        )
        allowed, current_count, remaining = int(result[0]), int(result[1]), int(result[2])

        return RateLimitResult(
            allowed=bool(allowed),
            current_count=current_count,
            remaining=remaining,
            limit=limit,
            key=identifier,
        )


# ---------- Multi-tier rate limiter ----------

class MultiTierRateLimiter:
    """
    Applies multiple rate limit tiers in order.
    Returns the most restrictive rejection (first tier that fails).
    """

    def __init__(self, redis_client: aioredis.Redis):
        self.limiter = RedisRateLimiter(redis_client)

    async def check_request(
        self,
        ip: str,
        api_key: str | None,
        user_id: int | None,
        endpoint: str,
    ) -> RateLimitResult:
        tiers = []

        # Tier 1: IP-level (anti-DDoS)
        tiers.append((f"ip:{ip}", 10_000, 60))           # 10K req/min per IP

        # Tier 2: API key level
        if api_key:
            tiers.append((f"key:{api_key}", 1_000, 60))  # 1K req/min per API key

        # Tier 3: User level
        if user_id:
            tiers.append((f"user:{user_id}", 100, 60))   # 100 req/min per user

        # Tier 4: Per-user per-endpoint
        if user_id:
            ep_slug = endpoint.replace("/", "_")
            tiers.append((f"ep:{user_id}:{ep_slug}", 10, 60))  # 10/min for this endpoint

        for identifier, limit, window_seconds in tiers:
            self.limiter.window_ms = window_seconds * 1000
            result = await self.limiter.check(identifier, limit)
            if not result.allowed:
                return result  # Return first violation

        # All tiers passed
        return result  # Return last result (most granular)


async def demo():
    redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)
    mtl = MultiTierRateLimiter(redis)

    print("=== Multi-Tier Rate Limiter Demo ===\n")

    # Simulate a normal user making requests
    for i in range(5):
        result = await mtl.check_request(
            ip="203.0.113.1",
            api_key="key-abc123",
            user_id=42,
            endpoint="/api/search",
        )
        status = "ALLOW" if result.allowed else f"DENY  (tier: {result.key})"
        print(f"  Request {i+1}: [{status}]  remaining={result.remaining}  "
              f"headers={result.headers()}")

    print("\n=== Simulating IP flood ===\n")
    # Simulate flood from same IP (different users)
    limiter = RedisRateLimiter(redis, window_seconds=1)
    allowed = rejected = 0
    for i in range(20):
        r = await limiter.check(f"ip:10.0.0.1", limit=10)
        if r.allowed:
            allowed += 1
        else:
            rejected += 1
    print(f"  20 rapid requests: {allowed} allowed, {rejected} rejected (limit=10/s)")

    await redis.aclose()


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 3: FastAPI Rate Limiting Middleware

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse
import asyncio
import time


app = FastAPI()

# In-memory store for demo (use Redis in production)
_windows: dict[str, tuple[int, int]] = {}  # key → (count, window_id)
LIMIT = 10
WINDOW = 10  # seconds


def check_rate_limit(key: str) -> tuple[bool, int]:
    window_id = int(time.time()) // WINDOW
    count, stored_window = _windows.get(key, (0, window_id))
    if stored_window != window_id:
        count = 0
    if count < LIMIT:
        _windows[key] = (count + 1, window_id)
        return True, LIMIT - count - 1
    return False, 0


@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Extract rate limit key from request
    api_key = request.headers.get("X-API-Key")
    if api_key:
        rl_key = f"key:{api_key}"
    else:
        # Fall back to IP
        rl_key = f"ip:{request.client.host}"

    allowed, remaining = check_rate_limit(rl_key)

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "Too Many Requests"},
            headers={
                "X-RateLimit-Limit": str(LIMIT),
                "X-RateLimit-Remaining": "0",
                "Retry-After": str(WINDOW),
            },
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(LIMIT)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    return response


@app.get("/api/data")
async def get_data():
    return {"data": "some result"}


# Run: uvicorn filename:app --reload
# Test: curl -H "X-API-Key: test123" http://localhost:8000/api/data
```
