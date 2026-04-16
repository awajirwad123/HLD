# Rate Limiter Service — Architecture

## Problem Statement

Design a rate limiting service that:
- Limits how many requests a client (user/IP/API key) can make in a time window
- Works across many API gateway instances (distributed)
- Adds < 5ms latency per request
- Supports multiple algorithms depending on use case

---

## Why Rate Limiting?

| Goal | Example |
|---|---|
| Prevent abuse / DoS | Block IP sending 10,000 req/sec |
| Fair resource sharing | Each user gets 1,000 req/min, not one user consuming all |
| Cost control | Limit expensive LLM API calls per customer |
| Compliance | Payment processing max N transactions per hour |
| Protect downstream services | Don't overwhelm a database with burst traffic |

---

## Rate Limiting Algorithms

### 1. Fixed Window Counter

```
Window: [0s–60s), [60s–120s), ...
Counter per window per client key

Allow if count < limit → increment counter
```

**Pros:** Simple, low memory (one counter per window).

**Cons:** Edge case — burst at window boundary. If limit=100/min, a user can send 100 at 0:59 and 100 more at 1:01, getting 200 in 2 seconds.

```
Key: ratelimit:{user_id}:{window}
window = floor(unix_timestamp / 60)

INCR ratelimit:user123:28300
EXPIRE ratelimit:user123:28300 60
```

---

### 2. Sliding Window Log

```
Keep a sorted set of request timestamps
On each request:
  - Remove timestamps older than window
  - Count remaining
  - If count < limit: allow + add timestamp
```

**Pros:** Perfectly accurate; no boundary burst problem.

**Cons:** Memory-intensive — stores every timestamp. 1,000 req/min limit = up to 1,000 entries per user.

```redis
ZREMRANGEBYSCORE ratelimit:{user_id} -inf (now - window_ms)
count = ZCARD ratelimit:{user_id}
if count < limit:
    ZADD ratelimit:{user_id} {now_ms} {request_id}
    EXPIRE ratelimit:{user_id} {window_seconds}
    → allow
else:
    → reject
```

---

### 3. Sliding Window Counter (Hybrid — Most Practical)

Approximates exact sliding window using two fixed-window counters:

```
current_window_count + previous_window_count × (overlap_fraction)
```

```
If window=60s and we're 15s into the current window:
  overlap = (60-15)/60 = 0.75  (75% of previous window is still in our 60s lookback)
  effective_count = current_count + prev_count × 0.75
```

**Pros:** Two counters per key (minimal memory). < 1% error vs exact sliding window.

**Cons:** Approximate (good enough for rate limiting).

---

### 4. Token Bucket

```
Bucket holds up to `capacity` tokens.
Tokens refill at `rate` tokens/second.
Each request consumes 1 token.
If bucket empty: reject.
```

**Allows bursts** up to `capacity` tokens, then smooths to `rate`/sec.

Use case: allow brief bursts but enforce average rate. Good for APIs that allow some burstiness.

```
last_refill = stored in Redis
tokens_remaining = stored in Redis

On request:
  elapsed = now - last_refill
  add = elapsed × rate
  tokens = min(capacity, tokens + add)
  last_refill = now
  if tokens >= 1:
      tokens -= 1
      → allow
  else:
      → reject
```

---

### 5. Leaky Bucket

```
Requests enter a queue (the "bucket").
Requests exit the queue at a fixed rate (the "leak").
If queue is full: drop new request.
```

Smooths bursty traffic to a constant output rate. Used in network traffic shaping.

**Pros:** Guarantees constant processing rate.

**Cons:** Added latency (requests queue); harder to implement in a stateless API gateway.

---

## Fixed Window vs Token Bucket vs Sliding Window

| Algorithm | Memory | Accuracy | Burst handling | Complexity |
|---|---|---|---|---|
| Fixed window | O(1) per key | Boundary burst issue | Not handled | Trivial |
| Sliding window log | O(requests) per key | Exact | No burst allowance | Moderate |
| Sliding window counter | O(2) per key | ~99% accurate | Not handled | Low |
| Token bucket | O(2) per key | Exact | Allows burst up to capacity | Moderate |
| Leaky bucket | O(queue size) | Exact smoothing | No burst | High |

**Recommendation:**
- Most APIs: **sliding window counter** (accurate, low memory)
- APIs needing burst tolerance: **token bucket**
- Fixed window: only for simple use cases or when exact accuracy doesn't matter

---

## Distributed Rate Limiting Architecture

```
API Request
    ↓
API Gateway (horizontal, N instances)
    ↓
Rate Limiter Middleware
    ├── Local cache (in-memory, ~1s TTL): fast path for "clearly over limit"
    └── Redis (centralized): authoritative counter shared across all gateway nodes
    ↓ (allowed)
Backend Service
```

### Redis Commands for Sliding Window Counter

```python
now_ms = int(time.time() * 1000)
window_ms = 60_000  # 60 seconds

# Lua script (atomic execution):
script = """
local key_curr = KEYS[1]
local key_prev = KEYS[2]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

local curr = tonumber(redis.call('GET', key_curr) or 0)
local prev = tonumber(redis.call('GET', key_prev) or 0)
local window_start = now - (now % window)
local elapsed = now - window_start
local overlap = (window - elapsed) / window
local count = curr + prev * overlap

if count < limit then
    redis.call('INCR', key_curr)
    redis.call('PEXPIRE', key_curr, window * 2)
    return {1, math.floor(count + 1)}  -- allowed, new count
else
    return {0, math.floor(count)}      -- rejected, current count
end
"""
```

### Response Headers (Standard Practice)

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 347
X-RateLimit-Reset: 1716873600      # Unix timestamp of window reset
Retry-After: 23                    # Seconds until retry (only on 429)
HTTP/1.1 429 Too Many Requests
```

---

## Multi-Tier Rate Limiting

Real APIs often apply limits at multiple granularities:

| Level | Key | Limit | Example |
|---|---|---|---|
| Global | IP | 10,000 req/min | Block scraper bots |
| API key | api_key | 1,000 req/min | Per-customer quota |
| User | user_id | 100 req/min | Per-account fair use |
| Endpoint | user_id + endpoint | 10 req/min | Expensive ML inference |

Check all applicable limits; reject if any one is exceeded. Return the most specific error.

---

## Edge Cases

**1. Redis unavailable:** fail-open (allow all requests) or fail-closed (reject all). Most APIs fail-open — brief Redis outage shouldn't take down the API. But for security-critical endpoints (login), fail-closed is safer.

**2. Large number of unique keys:** Key expiry keeps Redis memory bounded. With TTL = window × 2, only active clients use memory. At 1M requests/min: ~1GB Redis memory.

**3. Clock skew between nodes:** Use a single Redis clock (`TIME` command) as ground truth rather than application server timestamps. Prevents inconsistency when multiple API gateway nodes have slightly different system clocks.

---

## Key Numbers

| Metric | Value |
|---|---|
| Redis INCR latency | ~100 microseconds |
| Lua script execution | ~200–500 microseconds |
| Total rate-limit overhead | < 1ms (Redis in same datacenter) |
| Memory per rate limit key | ~50 bytes (sliding counter = 2 Redis strings) |
| 1M active users (60s window) | ~100MB Redis |
| Typical API rate limit tiers | 3 tiers (IP, API key, user) checked sequentially |
