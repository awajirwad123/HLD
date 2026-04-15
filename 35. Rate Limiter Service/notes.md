# Rate Limiter Service — Notes & Reference

## Algorithm Comparison Cheat Sheet

| Algorithm | Memory per key | Accuracy | Burst tolerance | Use case |
|---|---|---|---|---|
| Fixed window | O(1) | ~OK (boundary issue) | No | Simple APIs, non-critical |
| Sliding window log | O(limit) | Exact | No | Low-volume, exact enforcement |
| Sliding window counter | O(2) = constant | ~99% | No | General purpose, recommended |
| Token bucket | O(2) = constant | Exact | Yes (up to capacity) | APIs with tolerated bursts |
| Leaky bucket | O(queue) | Exact smoothing | Absorbs queue, no extra | Network QoS, smooth output |

---

## Redis Commands Reference

### Fixed Window
```redis
INCR ratelimit:{key}:{window_id}
EXPIRE ratelimit:{key}:{window_id} {window_seconds}
```

### Sliding Window Log
```redis
ZREMRANGEBYSCORE ratelimit:{key} -inf {cutoff_ms}
ZCARD ratelimit:{key}
ZADD ratelimit:{key} {now_ms} {request_uuid}
EXPIRE ratelimit:{key} {window_seconds}
```

### Sliding Window Counter (two keys)
```redis
GET ratelimit:{key}:{window_id}       # current window count
GET ratelimit:{key}:{window_id - 1}   # previous window count
INCR ratelimit:{key}:{window_id}
PEXPIRE ratelimit:{key}:{window_id} {window_ms * 2}
```

### Token Bucket (Lua)
```lua
local tokens = tonumber(redis.call('HGET', KEYS[1], 'tokens') or capacity)
local last   = tonumber(redis.call('HGET', KEYS[1], 'last') or now)
local added  = (now - last) * rate
tokens = math.min(capacity, tokens + added)
if tokens >= 1 then
    redis.call('HMSET', KEYS[1], 'tokens', tokens-1, 'last', now)
    redis.call('EXPIRE', KEYS[1], window)
    return 1   -- allowed
end
return 0   -- denied
```

---

## Standard HTTP Rate Limit Headers

```http
X-RateLimit-Limit: 1000         # Max requests per window
X-RateLimit-Remaining: 723      # Requests left in current window
X-RateLimit-Reset: 1716873600   # Unix timestamp of next window reset
Retry-After: 37                 # Seconds to wait (only on 429)
```

On rejection:
```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

{"error": "rate_limit_exceeded", "message": "Try again in 37 seconds"}
```

---

## Fixed Window Boundary Problem — Illustrated

```
Limit: 5 requests per minute

Timeline:
  :58  → Request 1  ✓  (window 1: count=1)
  :59  → Request 2  ✓  (window 1: count=2)
  :59  → Request 3  ✓  (window 1: count=3)
  :59  → Request 4  ✓  (window 1: count=4)
  :59  → Request 5  ✓  (window 1: count=5)
  1:00 → NEW WINDOW — counter resets!
  1:00 → Request 6  ✓  (window 2: count=1)
  1:00 → Request 7  ✓  (window 2: count=2)
  1:00 → Request 8  ✓  (window 2: count=3)
  1:00 → Request 9  ✓  (window 2: count=4)
  1:00 → Request 10 ✓  (window 2: count=5)

Result: 10 requests in 2 seconds — 2× the intended rate!
```

Sliding window counter fixes this by factoring in the previous window's count proportionally.

---

## Multi-Tier Rate Limiting Design

```
Request arrives
    │
    ├─ Check IP limit (10,000 req/min)          ← DDoS protection
    │       ↓ denied → 429
    │
    ├─ Check API key limit (1,000 req/min)       ← quota per customer
    │       ↓ denied → 429
    │
    ├─ Check user limit (100 req/min)            ← per-user fairness
    │       ↓ denied → 429
    │
    └─ Check endpoint limit (10 req/min)         ← protect expensive endpoint
            ↓ denied → 429
            ↓ allowed → Backend
```

---

## Fail-Open vs Fail-Closed

| Mode | Behavior when Redis is unavailable | Use case |
|---|---|---|
| Fail-open | Allow all requests | General APIs (availability > correctness) |
| Fail-closed | Reject all requests | Security-critical (login, payment) |
| Fail-local | Use local in-process counter | Best of both worlds, but may allow 2× limit |

Implementation for fail-open:
```python
try:
    result = await redis_check(key, limit)
except redis.RedisError:
    logger.error("Rate limiter Redis unavailable, failing open")
    return RateLimitResult(allowed=True, remaining=limit, ...)  # Allow
```

---

## Rate Limiting at Different Layers

| Layer | Mechanism | Granularity | Latency added |
|---|---|---|---|
| DNS / CDN | CDN provider rules | IP, geo | 0 (edge) |
| Load balancer | Nginx `limit_req`, HAProxy | IP | ~0.1ms |
| API gateway | Middleware, Redis | IP, key, user | ~1ms |
| Service level | In-process token bucket | per-user | ~0ms |
| DB level | Max connections pool | N/A | — |

In practice: coarse protection at load balancer, fine-grained at API gateway.

---

## Token Bucket Parameters

| Parameter | Effect | Example |
|---|---|---|
| `capacity` | Max burst size | capacity=10 → burst of 10 requests at once |
| `rate` | Sustained throughput | rate=2/s → 120 req/min continuous |
| `cost` | Tokens per request | cost=1 normally; cost=5 for expensive endpoints |

Burst vs. sustained:
```
capacity=100, rate=10/sec

Scenario A: Idle for 10 seconds → 100 tokens accumulated → send 100 at once ✓
Scenario B: Send steadily at 10/sec → always have ~0 tokens left
Scenario C: Send 150 at once → first 100 allowed, 50 rejected
```

---

## Key Numbers

| Metric | Value |
|---|---|
| Redis INCR latency | ~100–200 µs |
| Lua script overhead | adds ~100–200 µs |
| Total rate-limit middleware overhead | < 1ms |
| Memory per sliding-counter key | ~100 bytes (2 Redis keys) |
| Fixed window memory | ~50 bytes per key |
| Sliding window log memory | O(requests_per_window) × 50 bytes |
| Token bucket memory | ~150 bytes per key (HMSET with tokens + last_refill) |
| Typical API rate limit tiers | 3–4 |
| Common default limits | 1,000 req/min (B2B API), 100 req/min (user), 10 req/min (sensitive) |
