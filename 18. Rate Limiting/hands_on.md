# Rate Limiting — Hands-On Exercises

## Exercise 1: All Four Algorithms — Pure Python

```python
"""
Side-by-side implementations of Fixed Window, Sliding Window Log,
Sliding Window Counter, Token Bucket, and Leaky Bucket.
All in-memory — no Redis dependency.

Run the demo to see burst behaviour differences.
"""
import time
import collections
from threading import Lock


# ── 1. Fixed Window Counter ────────────────────────────────────────────────────
class FixedWindowLimiter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit   = limit
        self.window  = window_seconds
        self._counts: dict[str, tuple[int, int]] = {}  # key → (count, window_start)
        self._lock   = Lock()

    def allow(self, key: str) -> bool:
        now           = int(time.time())
        window_start  = now - (now % self.window)

        with self._lock:
            count, stored_window = self._counts.get(key, (0, window_start))
            if stored_window != window_start:
                # New window
                count        = 0
                stored_window = window_start

            if count >= self.limit:
                return False

            self._counts[key] = (count + 1, stored_window)
            return True


# ── 2. Sliding Window Log ──────────────────────────────────────────────────────
class SlidingWindowLogLimiter:
    def __init__(self, limit: int, window_seconds: int):
        self.limit  = limit
        self.window = window_seconds
        self._logs: dict[str, collections.deque] = {}
        self._lock  = Lock()

    def allow(self, key: str) -> bool:
        now    = time.time()
        cutoff = now - self.window

        with self._lock:
            log = self._logs.setdefault(key, collections.deque())
            # Remove entries outside the window
            while log and log[0] <= cutoff:
                log.popleft()

            if len(log) >= self.limit:
                return False

            log.append(now)
            return True


# ── 3. Sliding Window Counter (Hybrid) ────────────────────────────────────────
class SlidingWindowCounterLimiter:
    """
    Approximates sliding window using previous and current window counters.
    Only 2 integers per key — memory-efficient.
    """
    def __init__(self, limit: int, window_seconds: int):
        self.limit  = limit
        self.window = window_seconds
        # key → (prev_count, curr_count, curr_window_start)
        self._state: dict[str, tuple[int, int, int]] = {}
        self._lock  = Lock()

    def allow(self, key: str) -> bool:
        now          = time.time()
        window_start = int(now // self.window) * self.window
        elapsed      = now - window_start

        with self._lock:
            prev_count, curr_count, stored_start = self._state.get(key, (0, 0, window_start))

            if stored_start != window_start:
                # Advance: previous = last current, reset current
                prev_count   = curr_count
                curr_count   = 0
                stored_start  = window_start

            # Weighted estimate: how much of the prev window overlaps
            prev_weight = (self.window - elapsed) / self.window
            estimated   = prev_count * prev_weight + curr_count

            if estimated >= self.limit:
                return False

            self._state[key] = (prev_count, curr_count + 1, stored_start)
            return True


# ── 4. Token Bucket ───────────────────────────────────────────────────────────
class TokenBucketLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity:    max tokens (burst ceiling)
        refill_rate: tokens added per second
        """
        self.capacity    = capacity
        self.refill_rate = refill_rate
        # key → (tokens, last_refill_time)
        self._buckets: dict[str, tuple[float, float]] = {}
        self._lock    = Lock()

    def allow(self, key: str, tokens_requested: int = 1) -> bool:
        now = time.time()

        with self._lock:
            tokens, last_refill = self._buckets.get(key, (float(self.capacity), now))

            # Refill
            elapsed = now - last_refill
            tokens  = min(self.capacity, tokens + elapsed * self.refill_rate)

            if tokens < tokens_requested:
                self._buckets[key] = (tokens, now)
                return False

            self._buckets[key] = (tokens - tokens_requested, now)
            return True

    def remaining(self, key: str) -> float:
        now = time.time()
        tokens, last_refill = self._buckets.get(key, (float(self.capacity), now))
        elapsed = now - last_refill
        return min(self.capacity, tokens + elapsed * self.refill_rate)


# ── 5. Leaky Bucket ───────────────────────────────────────────────────────────
import asyncio

class LeakyBucketLimiter:
    """
    Enqueues requests and drains at a constant rate.
    New requests when queue is full are dropped.
    """
    def __init__(self, capacity: int, drain_rate: float):
        """
        capacity:   max queue size
        drain_rate: requests processed per second
        """
        self.capacity   = capacity
        self.drain_rate = drain_rate
        self._queues: dict[str, asyncio.Queue] = {}

    async def allow(self, key: str) -> bool:
        q = self._queues.setdefault(key, asyncio.Queue(maxsize=self.capacity))
        try:
            q.put_nowait(1)
            return True     # Queued successfully — will be processed at drain_rate
        except asyncio.QueueFull:
            return False    # Queue full — drop request

    async def drain(self, key: str, handler):
        """Background task: drain the queue at constant rate."""
        q = self._queues.setdefault(key, asyncio.Queue(maxsize=self.capacity))
        interval = 1.0 / self.drain_rate
        while True:
            await q.get()
            await handler()
            await asyncio.sleep(interval)


# ── Demo: Compare burst behaviour ─────────────────────────────────────────────
def burst_test(name: str, limiter, key: str = "user1", n: int = 15):
    allowed = sum(1 for _ in range(n) if limiter.allow(key))
    print(f"  {name:30s}: {allowed}/{n} requests allowed")


if __name__ == "__main__":
    print("=== Burst of 15 requests, limit = 10/10s ===\n")

    burst_test("Fixed Window",           FixedWindowLimiter(10, 10))
    burst_test("Sliding Window Log",     SlidingWindowLogLimiter(10, 10))
    burst_test("Sliding Window Counter", SlidingWindowCounterLimiter(10, 10))
    burst_test("Token Bucket (cap=10)",  TokenBucketLimiter(10, 1))

    print("\n=== Test boundary burst issue (Fixed Window) ===")
    fw = FixedWindowLimiter(10, 10)
    # Fill end of first window
    for _ in range(10):
        fw.allow("u")
    # Simulate window rollover
    fw._counts["u"] = (0, fw._counts["u"][1] + 10)  # Advance window manually
    # Burst at start of new window
    new_allowed = sum(1 for _ in range(10) if fw.allow("u"))
    print(f"  Fixed Window allows {new_allowed} at start of new window → 20 in 2 windows")

    print("\n=== Token Bucket: burst then sustain ===")
    tb = TokenBucketLimiter(capacity=5, refill_rate=2)       # 2 tokens/sec refill
    for i in range(8):
        ok = tb.allow("u")
        remaining = tb.remaining("u")
        print(f"  Request {i+1}: {'✓' if ok else '✗'} | remaining: {remaining:.1f}")
        time.sleep(0.3)                                       # 0.3s between requests
```

---

## Exercise 2: Redis-Backed Distributed Rate Limiter (Production-Grade)

```python
"""
Production-grade distributed rate limiter using Redis.
Implements the Sliding Window Counter algorithm with atomic Lua scripts.
Exposes FastAPI middleware that reads limits from a config dict.

pip install fastapi redis asyncio uvicorn
"""
import time
import redis.asyncio as aioredis
from dataclasses import dataclass
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse

REDIS_URL = "redis://localhost:6379"

# ── Rate limit config per route ───────────────────────────────────────────────
RATE_LIMITS: dict[str, dict] = {
    "/api/charge":      {"limit": 10,   "window": 60},   # 10/min — sensitive
    "/api/search":      {"limit": 100,  "window": 60},   # 100/min
    "/api/users":       {"limit": 1000, "window": 60},   # 1000/min
    "default":          {"limit": 500,  "window": 60},   # catch-all
}

# ── Lua script: atomic sliding window counter ─────────────────────────────────
# Two keys per client per route: curr_window_count and prev_window_count
SLIDING_WINDOW_LUA = """
local curr_key   = KEYS[1]
local prev_key   = KEYS[2]
local limit      = tonumber(ARGV[1])
local window     = tonumber(ARGV[2])
local now        = tonumber(ARGV[3])

-- Determine position within current window
local window_start  = math.floor(now / window) * window
local elapsed       = now - window_start
local prev_weight   = (window - elapsed) / window

local curr_count = tonumber(redis.call("GET", curr_key) or 0)
local prev_count = tonumber(redis.call("GET", prev_key) or 0)

local estimated = prev_count * prev_weight + curr_count

if estimated >= limit then
    return {0, math.ceil(estimated), limit, window - elapsed}
end

redis.call("INCR", curr_key)
redis.call("EXPIRE", curr_key, window * 2)
return {1, math.ceil(estimated) + 1, limit, window - elapsed}
"""


@dataclass
class RateLimitResult:
    allowed:    bool
    count:      int
    limit:      int
    retry_after: float   # Seconds until window resets (if rejected)


class DistributedRateLimiter:
    def __init__(self, redis: aioredis.Redis):
        self._redis  = redis
        self._script = None

    async def _get_script(self):
        if self._script is None:
            self._script = self._redis.register_script(SLIDING_WINDOW_LUA)
        return self._script

    async def check(self, key: str, limit: int, window: int) -> RateLimitResult:
        now          = time.time()
        window_start = int(now // window) * window
        curr_key     = f"rl:{key}:{window_start}"
        prev_key     = f"rl:{key}:{window_start - window}"

        script = await self._get_script()
        result = await script(
            keys=[curr_key, prev_key],
            args=[limit, window, now],
        )

        allowed, count, limit_v, retry_after = result
        return RateLimitResult(
            allowed=bool(allowed),
            count=int(count),
            limit=int(limit_v),
            retry_after=float(retry_after),
        )


# ── FastAPI middleware ────────────────────────────────────────────────────────
redis_client: aioredis.Redis = None
rate_limiter: DistributedRateLimiter = None

app = FastAPI()


@app.on_event("startup")
async def startup():
    global redis_client, rate_limiter
    redis_client = aioredis.from_url(REDIS_URL, decode_responses=False)
    rate_limiter = DistributedRateLimiter(redis_client)


@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Identify client: prefer authenticated user ID, fall back to IP
    client_id = request.headers.get("X-User-Id") or request.client.host

    # Find the most specific rate limit config for this path
    path = request.url.path
    config = RATE_LIMITS.get(path) or RATE_LIMITS["default"]

    key    = f"{client_id}:{path}"
    result = await rate_limiter.check(key, config["limit"], config["window"])

    # Always include rate limit headers (even on success)
    headers = {
        "X-RateLimit-Limit":     str(result.limit),
        "X-RateLimit-Remaining": str(max(0, result.limit - result.count)),
        "X-RateLimit-Reset":     str(int(time.time() + result.retry_after)),
    }

    if not result.allowed:
        headers["Retry-After"] = str(int(result.retry_after) + 1)
        return JSONResponse(
            status_code=429,
            content={
                "error":   "rate_limit_exceeded",
                "message": f"Rate limit exceeded. Retry after {int(result.retry_after)+1}s.",
            },
            headers=headers,
        )

    response = await call_next(request)
    for k, v in headers.items():
        response.headers[k] = v
    return response


# ── Sample endpoints ──────────────────────────────────────────────────────────
@app.post("/api/charge")
async def charge():
    return {"status": "charged"}


@app.get("/api/search")
async def search(q: str = ""):
    return {"results": [], "query": q}


@app.get("/api/users")
async def list_users():
    return {"users": []}
```

---

## Exercise 3: Token Bucket in Redis (Lua — Atomic)

```python
"""
Token bucket implemented entirely in Redis using an atomic Lua script.
Correct under concurrent access — no race conditions.

pip install redis
"""
import time
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

TOKEN_BUCKET_LUA = """
local key          = KEYS[1]
local ts_key       = KEYS[1] .. ":ts"
local capacity     = tonumber(ARGV[1])
local refill_rate  = tonumber(ARGV[2])   -- tokens per second
local now          = tonumber(ARGV[3])
local requested    = tonumber(ARGV[4])

local last_ts  = tonumber(redis.call("GET", ts_key) or now)
local tokens   = tonumber(redis.call("GET", key) or capacity)

-- Refill based on elapsed time
local elapsed  = math.max(0, now - last_ts)
local refilled = math.min(capacity, tokens + elapsed * refill_rate)

if refilled < requested then
    -- Update tokens (so we don't lose the refill) but reject
    redis.call("SET",  key,    refilled)
    redis.call("SET",  ts_key, now)
    redis.call("EXPIRE", key,    3600)
    redis.call("EXPIRE", ts_key, 3600)
    local wait = (requested - refilled) / refill_rate
    return {0, math.floor(refilled), math.ceil(wait * 1000)}  -- {allowed, tokens, wait_ms}
end

local new_tokens = refilled - requested
redis.call("SET",  key,    new_tokens)
redis.call("SET",  ts_key, now)
redis.call("EXPIRE", key,    3600)
redis.call("EXPIRE", ts_key, 3600)
return {1, math.floor(new_tokens), 0}
"""

token_bucket_script = r.register_script(TOKEN_BUCKET_LUA)


def check_token_bucket(
    client_id: str,
    capacity: int,
    refill_rate: float,
    tokens_requested: int = 1,
) -> dict:
    key    = f"tb:{client_id}"
    result = token_bucket_script(
        keys=[key],
        args=[capacity, refill_rate, time.time(), tokens_requested],
    )
    allowed, remaining, wait_ms = result
    return {
        "allowed":   bool(allowed),
        "remaining": int(remaining),
        "wait_ms":   int(wait_ms),
    }


# ── Demo ──────────────────────────────────────────────────────────────────────
if __name__ == "__main__":
    print("=== Token bucket: capacity=5, refill=2/s ===\n")

    for i in range(10):
        result = check_token_bucket("user_alice", capacity=5, refill_rate=2.0)
        status = "✓ ALLOW" if result["allowed"] else f"✗ REJECT (wait {result['wait_ms']}ms)"
        print(f"  Request {i+1:2d}: {status} | {result['remaining']} tokens remaining")
        time.sleep(0.4)    # 0.4s between requests; refill = 0.8 token/interval

    print("\n=== Burst: 5 rapid requests ===")
    r.delete("tb:burst_user", "tb:burst_user:ts")   # Reset
    for i in range(7):
        result = check_token_bucket("burst_user", capacity=5, refill_rate=1.0)
        status = "✓" if result["allowed"] else "✗"
        print(f"  Burst request {i+1}: {status} | {result['remaining']} tokens")
```

---

## Exercise 4: FastAPI Rate Limiter — Multi-Tier (Per-IP + Per-User + Per-Endpoint)

```python
"""
Multi-tier rate limiting:
  Tier 1: Per-IP limit   (unauthenticated protection)
  Tier 2: Per-User limit (plan-based authenticated limit)
  Tier 3: Per-endpoint   (write endpoints stricter than reads)

All checks run in parallel via asyncio.gather.

pip install fastapi redis asyncio uvicorn pydantic
"""
import asyncio
import time
from enum import Enum

import redis.asyncio as aioredis
from fastapi import FastAPI, Request, Depends, Header
from fastapi.responses import JSONResponse
from typing import Optional

REDIS_URL = "redis://localhost:6379"

# ── Plan-based limits ─────────────────────────────────────────────────────────
class Plan(str, Enum):
    FREE       = "free"
    PRO        = "pro"
    ENTERPRISE = "enterprise"


PLAN_LIMITS = {
    Plan.FREE:       {"limit": 60,    "window": 60},
    Plan.PRO:        {"limit": 1000,  "window": 60},
    Plan.ENTERPRISE: {"limit": 10000, "window": 60},
}

ENDPOINT_MULTIPLIERS = {
    "write": 0.1,   # Write endpoints are 10× more restricted
    "read":  1.0,
}

IP_LIMIT   = {"limit": 30,  "window": 60}    # Strict for unauthenticated IPs

redis_pool: aioredis.Redis = None
app = FastAPI()


@app.on_event("startup")
async def startup():
    global redis_pool
    redis_pool = aioredis.from_url(REDIS_URL, decode_responses=True)


async def sliding_window_check(key: str, limit: int, window: int) -> tuple[bool, int]:
    """Returns (allowed, remaining)."""
    now          = time.time()
    window_start = int(now // window) * window
    curr_key     = f"rl:{key}:{window_start}"
    prev_key     = f"rl:{key}:{window_start - window}"

    pipe = redis_pool.pipeline()
    pipe.get(prev_key)
    pipe.get(curr_key)
    prev_str, curr_str = await pipe.execute()

    prev_count = int(prev_str or 0)
    curr_count = int(curr_str or 0)
    elapsed    = now - window_start
    weight     = (window - elapsed) / window
    estimated  = prev_count * weight + curr_count

    if estimated >= limit:
        return False, 0

    await redis_pool.incr(curr_key)
    await redis_pool.expire(curr_key, window * 2)
    return True, max(0, limit - int(estimated) - 1)


async def check_all_tiers(
    ip: str,
    user_id: Optional[str],
    plan: Plan,
    endpoint_type: str,
) -> tuple[bool, dict]:
    """
    Runs all applicable rate limit checks concurrently.
    Returns (all_allowed, headers_dict).
    """
    checks = []

    # Tier 1: Per-IP (always)
    checks.append(("ip", sliding_window_check(
        f"ip:{ip}", IP_LIMIT["limit"], IP_LIMIT["window"]
    )))

    # Tier 2: Per-user (if authenticated)
    if user_id:
        plan_cfg    = PLAN_LIMITS[plan]
        user_limit  = int(plan_cfg["limit"] * ENDPOINT_MULTIPLIERS[endpoint_type])
        checks.append(("user", sliding_window_check(
            f"user:{user_id}:{endpoint_type}", user_limit, plan_cfg["window"]
        )))

    # Run checks concurrently
    labels, coros = zip(*checks)
    results = await asyncio.gather(*coros)

    # All tiers must pass
    headers = {}
    all_allowed = True
    for label, (allowed, remaining) in zip(labels, results):
        headers[f"X-RateLimit-{label.title()}-Remaining"] = str(remaining)
        if not allowed:
            all_allowed = False

    return all_allowed, headers


def classify_endpoint(path: str, method: str) -> str:
    if method in ("POST", "PUT", "PATCH", "DELETE"):
        return "write"
    return "read"


async def get_current_user(
    x_user_id: Optional[str] = Header(None),
    x_user_plan: Optional[str] = Header(None),
) -> tuple[Optional[str], Plan]:
    plan = Plan(x_user_plan) if x_user_plan in Plan._value2member_map_ else Plan.FREE
    return x_user_id, plan


@app.middleware("http")
async def multi_tier_rate_limit(request: Request, call_next):
    ip      = request.client.host
    user_id = request.headers.get("X-User-Id")
    plan    = Plan(request.headers.get("X-User-Plan", "free"))
    ep_type = classify_endpoint(request.url.path, request.method)

    allowed, rl_headers = await check_all_tiers(ip, user_id, plan, ep_type)

    if not allowed:
        return JSONResponse(
            status_code=429,
            content={"error": "rate_limit_exceeded"},
            headers=rl_headers,
        )

    response = await call_next(request)
    for k, v in rl_headers.items():
        response.headers[k] = v
    return response


@app.post("/api/orders")
async def create_order():
    return {"order_id": "ord-123", "status": "created"}


@app.get("/api/products")
async def list_products():
    return {"products": []}
```

---

## Exercise 5: Rate-Limit-Aware Client with Retry

```python
"""
HTTP client that respects 429 responses, parses Retry-After,
and uses exponential backoff with jitter.

pip install httpx
"""
import asyncio
import random
import time
import httpx


class RateLimitAwareClient:
    def __init__(self, base_url: str, max_retries: int = 5):
        self.base_url    = base_url
        self.max_retries = max_retries
        self._client     = httpx.AsyncClient(base_url=base_url, timeout=10.0)

    async def request(self, method: str, path: str, **kwargs) -> httpx.Response:
        attempt  = 0
        backoff  = 1.0

        while attempt <= self.max_retries:
            try:
                resp = await self._client.request(method, path, **kwargs)

                if resp.status_code == 429:
                    # Parse Retry-After header (seconds or HTTP-date)
                    retry_after = self._parse_retry_after(resp)
                    print(f"  [Client] 429 received — waiting {retry_after:.1f}s (attempt {attempt+1})")
                    await asyncio.sleep(retry_after)
                    attempt += 1
                    continue

                if resp.status_code >= 500 and attempt < self.max_retries:
                    # Server error — exponential backoff with jitter
                    sleep_for = min(backoff * (0.5 + random.random()), 30)
                    print(f"  [Client] {resp.status_code} error — backoff {sleep_for:.1f}s")
                    await asyncio.sleep(sleep_for)
                    backoff  *= 2
                    attempt  += 1
                    continue

                return resp

            except (httpx.ConnectError, httpx.TimeoutException) as e:
                if attempt >= self.max_retries:
                    raise
                sleep_for = min(backoff * (0.5 + random.random()), 30)
                print(f"  [Client] Connection error: {e} — backoff {sleep_for:.1f}s")
                await asyncio.sleep(sleep_for)
                backoff *= 2
                attempt += 1

        raise RuntimeError(f"Max retries ({self.max_retries}) exceeded")

    def _parse_retry_after(self, resp: httpx.Response) -> float:
        header = resp.headers.get("Retry-After", "5")
        try:
            return float(header)
        except ValueError:
            # HTTP-date format
            from email.utils import parsedate_to_datetime
            retry_dt = parsedate_to_datetime(header)
            return max(0.0, retry_dt.timestamp() - time.time())

    async def get(self, path: str, **kwargs) -> httpx.Response:
        return await self.request("GET", path, **kwargs)

    async def post(self, path: str, **kwargs) -> httpx.Response:
        return await self.request("POST", path, **kwargs)

    async def close(self):
        await self._client.aclose()


# ── Demo ──────────────────────────────────────────────────────────────────────
async def demo():
    client = RateLimitAwareClient("http://localhost:8000")

    print("=== Sending 15 requests (limit = 10/min) ===\n")
    for i in range(15):
        try:
            resp = await client.get("/api/products")
            remaining = resp.headers.get("X-RateLimit-Remaining", "?")
            print(f"  Request {i+1:2d}: {resp.status_code} | remaining: {remaining}")
        except RuntimeError as e:
            print(f"  Request {i+1:2d}: FAILED — {e}")

    await client.close()


if __name__ == "__main__":
    asyncio.run(demo())
```
