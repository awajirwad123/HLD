# Rate Limiting — Architecture & Concepts

> **Scope of this topic:** Rate limiting as a **concept and building block** — algorithms, dimensions, integration patterns, and where it fits inside a larger system. For the full system design of a standalone distributed rate limiter service (including system walk-through, scaling, and database choices), see **[T35 — Rate Limiter Service](../35.%20Rate%20Limiter%20Service/)**.

---

## 1. Why Rate Limiting Exists

Rate limiting controls **how many requests** a client (or system) can make in a given time window. Without it:

- A single misbehaving client can exhaust your server's thread pool, hitting all other users
- Brute-force attacks (password guessing, credential stuffing) become trivial
- Scrapers can drain your entire database
- A bug in a client SDK causes infinite retries → DDoS by accident
- Cost-based APIs (LLM inference, SMS) incur unbounded spend

**Rate limiting is not just availability protection — it is also fairness, cost control, and security.**

---

## 2. The Four Core Algorithms

### 2.1 Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute buckets). Increment a counter per window per client. Reject when the counter exceeds the limit.

```
Window: [00:00 → 01:00]  counter = 0
Request at 00:30         counter = 1
Request at 00:58         counter = 2  ✓
...
Request at 00:59         counter = 100 REJECT
Window resets at 01:00   counter = 0

New window: [01:00 → 02:00]
```

**Implementation:**
```redis
INCR   rate_limit:{user_id}:{window_start}
EXPIRE rate_limit:{user_id}:{window_start}  60
```

**Problem — Boundary burst:**
A client can make 100 requests at 00:59 and 100 more at 01:01 → **200 requests in 2 seconds**, even though the limit is "100/minute."

```
Window 1: ......................[100 at 00:59]
Window 2: [100 at 01:01]..........................
          ↑ 200 requests in a 2-second window
```

---

### 2.2 Sliding Window Log

Store a timestamp log of each request. On each new request, remove expired entries and count the remainder.

```
Request at 01:05 — check log from (01:05 − 60s) = 00:05
  Log: [00:10, 00:30, 00:55, 00:58]
  All within window → count = 4
  If limit = 5: allow, append 01:05
  Log: [00:10, 00:30, 00:55, 00:58, 01:05]
```

**Implementation (Redis Sorted Set):**
```redis
ZREMRANGEBYSCORE key 0 {now - window_ms}       # Remove expired
ZADD             key {now} {request_id}         # Add current
ZCARD            key                            # Count
EXPIRE           key {window_seconds}
```

**Advantage**: Precise — no boundary burst.
**Disadvantage**: Stores every request timestamp. At 1,000 req/min per user × 1M users = 1 billion entries in the sorted set. Memory-intensive at scale.

---

### 2.3 Sliding Window Counter (Hybrid)

Approximates the sliding window log using only two counters: current window count and the previous window count, weighted by how much of the previous window overlaps.

```
Time: 01:35 (35 seconds into the current 60s window)
Previous window [00:00–01:00]: count = 80
Current window  [01:00–02:00]: count = 30

Overlap of previous window: (60 - 35) / 60 = 41.7%
Estimated count = 80 × 0.417 + 30 = 63.4

If limit = 100: 63.4 < 100 → allow
```

**Formula:**
$$\text{rate} = \text{prev\_count} \times \frac{\text{window} - \text{elapsed\_in\_current}}{\text{window}} + \text{curr\_count}$$

**Why it's the practical choice**: Only 2 counters per user (vs N log entries). Very low memory footprint. Accurate to within ~0.3% error in practice (Cloudflare published this).

---

### 2.4 Token Bucket

A bucket holds tokens up to a maximum capacity. Tokens are added at a constant refill rate. Each request consumes one token. If the bucket is empty, the request is rejected.

```
Capacity:     100 tokens
Refill rate:   10 tokens/second
Current:       85 tokens

Request arrives → consume 1 → 84 tokens remaining ✓

Burst of 90 requests → 84 are allowed, 6 are rejected
Next second: refilled to min(84 + 10, 100) = 94 tokens
```

**Key property: allows bursts up to the bucket capacity.**
A client can burst 100 requests instantly, then sustains at the refill rate. Good for APIs where occasional bursts are legitimate (browser page load fetches 20 resources at once).

**Implementation:**
```python
# Conceptual — actual implementation uses last_refill timestamp
tokens = min(capacity, tokens + (now - last_refill) * refill_rate)
if tokens >= 1:
    tokens -= 1
    allow()
else:
    reject()
```

---

### 2.5 Leaky Bucket

Requests enter a fixed-size queue. A worker drains the queue at a constant rate. New requests when the queue is full are dropped.

```
Queue capacity: 100
Drain rate:     10 req/sec

10 requests arrive → queued
Worker processes 10/sec → smooth output

50 requests burst → queued (if < 100 in queue)
Queue full → excess rejected
Output rate: always exactly 10/sec (smooth)
```

**Key property: enforces a constant output rate regardless of input burstiness.**
Good for protecting a downstream service that can only handle a fixed rate (e.g., a DB that handles 100 writes/sec maximum).

**Difference from token bucket:**
- Token bucket: bursty input → bursty output (up to bucket size)
- Leaky bucket: bursty input → **smooth** output (rate = drain rate)

---

## 3. Algorithm Comparison

| Algorithm | Burst handling | Memory | Boundary burst | Smoothness | Best for |
|---|---|---|---|---|---|
| Fixed Window | Burst allowed | O(1) | **Yes — double burst** | No | Simple counters, coarse limits |
| Sliding Window Log | No burst | O(n) requests | No | Yes | High-precision per-user limits |
| Sliding Window Counter | ~Burst | O(1) | Approx. no | Approximate | Production (Cloudflare, NGINX) |
| Token Bucket | **Burst up to capacity** | O(1) | No | No | API rate limits, Stripe |
| Leaky Bucket | **No burst, smooth** | O(queue) | No | **Yes** | Protecting downstream services |

---

## 4. Distributed Rate Limiting

The challenge: with 10 server instances, each has a local counter. A client sending 100 req/s gets 10 req/s per server → all pass, but the total exceeds the limit.

```
Server 1: sees 10 req/s from client X → allows all (limit = 100)
Server 2: sees 10 req/s from client X → allows all
...
Server 10: sees 10 req/s from client X → allows all
Actual: 100 req/s allowed — correct!

But if client X sends 100 req/s to Server 1 only:
Server 1: sees 100 req/s → allows all (counter = 100, limit = 100)
Others: see 0 → counter = 0

With 10 servers but random hashing, 100 req/s could hit server 1→10 unevenly:
Server 1 might see 15 req/s and allow, but actually total is 100 → OK
...or Server 3 might see 95 req/s and allow → limit effectively = 95 on this server
```

### 4.1 Centralised Counter (Redis)

All servers atomically increment one Redis counter. Single source of truth.

```python
INCR   rate:{client_id}:{window}
EXPIRE rate:{client_id}:{window}  60
```

**Advantage**: Perfectly accurate.
**Disadvantage**: Redis is now in the hot path of every request. At 100K RPS: 100K Redis writes/sec. Redis can handle this, but adds ~1ms per request and creates a SPOF.

### 4.2 Local Counter + Periodic Sync

Each server keeps a local counter and periodically syncs a *delta* to Redis.

```
Server 1: local_count = 47
Every 5 seconds: INCRBY rate:{client} 47; local_count = 0
Redis aggregates from all servers → global count
```

**Advantage**: Reduces Redis write pressure.
**Disadvantage**: In the 5-second sync window, the global count is stale. Client could exceed the limit before servers sync → up to `sync_period × N_servers` over-allowance. Acceptable for soft limits.

### 4.3 Token Bucket in Redis (Lua Script)

Atomic check-and-update using a Lua script (executes atomically in Redis):

```lua
-- KEYS[1] = rate_limit key, ARGV = [now, capacity, refill_rate, requested]
local tokens_key = KEYS[1]
local last_refill_key = KEYS[1] .. ":ts"

local now = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local refill_rate = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local last = tonumber(redis.call("GET", last_refill_key) or now)
local current = tonumber(redis.call("GET", tokens_key) or capacity)

local elapsed = now - last
local refilled = math.min(capacity, current + elapsed * refill_rate)

if refilled >= requested then
    redis.call("SET", tokens_key, refilled - requested)
    redis.call("SET", last_refill_key, now)
    return 1   -- allowed
else
    redis.call("SET", tokens_key, refilled)
    redis.call("SET", last_refill_key, now)
    return 0   -- rejected
end
```

Single Redis round-trip. Atomic. No race conditions.

### 4.4 Cell-CRDT / Redis Cell

[`redis-cell`](https://github.com/brandur/redis-cell) is a Redis module implementing the Generic Cell Rate Algorithm (GCRA) — leaky bucket variant — as a single Redis command:

```redis
CL.THROTTLE user_id  100  10  1  1
             key    max  rate period tokens_requested
```

Returns: `[allowed, limit, remaining, retry_after, reset_after]`

---

## 5. Rate Limiting Dimensions

You can rate limit on multiple dimensions simultaneously:

| Dimension | Key | Example |
|---|---|---|
| Per user | `rate:{user_id}` | 1,000 req/min |
| Per IP | `rate:{ip}` | 100 req/min (unauthenticated) |
| Per endpoint | `rate:{user}:{endpoint}` | 10 writes/min, 1000 reads/min |
| Per API key | `rate:{api_key}` | Stripe: 100 req/sec |
| Per tenant (SaaS) | `rate:{org_id}` | Plan-based limits |
| Global | `rate:global:{endpoint}` | Hard ceiling regardless of user |

---

## 6. Where Rate Limiting Lives

```
[Client]
   ↓
[CDN / Edge]          ← Layer 1: IP-based, DDoS protection (Cloudflare)
   ↓
[API Gateway]         ← Layer 2: API key / user limits (Kong, AWS API Gateway)
   ↓
[Microservice]        ← Layer 3: Business logic limits (endpoint-specific)
   ↓
[Downstream service]  ← Layer 4: Protect DB / third-party API
```

Defence in depth: each layer enforces different limits for different purposes.

---

## 7. Rate Limit Response

Industry standard HTTP response for exceeded rate limits:

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit:     100
X-RateLimit-Remaining: 0
X-RateLimit-Reset:     1700000060    ← Unix timestamp when limit resets
Retry-After:           30            ← Seconds to wait before retry

{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded your rate limit. Retry after 30 seconds."
}
```

**Retry-After** is critical — clients that respect it back off correctly instead of retrying immediately (thundering herd prevention).

---

## 8. Real-World Patterns

### Stripe Rate Limiting

Stripe uses multiple overlapping limits:
- **Per API key**: 100 requests/second
- **Per endpoint type**: Write endpoints (charges, refunds) have lower limits than reads
- **Burst allowance**: Token bucket — short bursts are permitted
- **Response**: 429 with `Retry-After` header

The Stripe client library has built-in retry with exponential backoff + jitter that respects `Retry-After`.

### Cloudflare Rate Limiting

- Operates at the edge — rate limiting before traffic reaches the origin
- Sliding window (counter hybrid) per PoP — Cloudflare's blog documents their sliding window implementation
- Per-IP, per-URL, per-cookie/header rules configurable
- Challenge vs block: can serve a CAPTCHA before blocking completely

### GitHub API Rate Limiting

- 5,000 requests/hour for authenticated users
- 60 requests/hour for unauthenticated
- `X-RateLimit-*` headers on every response (cost remaining visible to clients)
- Search API has stricter limits (30 req/min) — endpoint-specific

### Kong API Gateway

Kong implements rate limiting as a plugin per route/service:
- Algorithms: fixed window, sliding window, leaky bucket
- Backends: Redis (distributed), local memory (per node)
- Can limit by: consumer, credential, IP, or header

---

## 9. Soft vs Hard Rate Limits

| Type | Behaviour | Use case |
|---|---|---|
| **Hard limit** | Immediately reject at threshold | Security (login attempts), cost control |
| **Soft limit** | Allow burst, then throttle | Developer-friendly APIs |
| **Quota** | Daily/monthly cap (not per-second) | SaaS billing tiers |
| **Throttle** | Slow down rather than reject | Queue depth management |

---

## 10. Rate Limiting + Idempotency Interaction

A client gets a 429 and retries. If the original request actually succeeded (response lost), and the retry hits the rate limiter (counter already at the limit) → permanent failure even though the operation completed.

**Fix**: Rate limit on *requests received*, not on *operations completed*. The first request that succeeded shouldn't count against the retry of the same idempotency key. Some systems deduplicate: if `Idempotency-Key` is seen again, return the cached response without incrementing the rate limit counter.

---

## 11. Rate Limiting as an Integration Building Block

Rate limiting is embedded into many other system components — it is rarely a standalone feature. Understanding *where* it lives in a system is as important as understanding the algorithms.

### 11.1 Rate Limiting at the CDN / Edge

**Who does it:** Cloudflare, Fastly, AWS CloudFront + WAF

**What is rate limited:** Client IP, URL patterns, geographic origin

**Why at the edge:**
- Requests are filtered *before* they reach your origin servers
- Stops volumetric DDoS attacks at the edge POP (close to the attacker)
- No per-request latency added to legitimate traffic

**Limitation:** CDN rules are coarse-grained (per-IP or per-URL). They cannot enforce per-user or per-API-key limits (no knowledge of your app's identity model).

### 11.2 Rate Limiting at the API Gateway

**Who does it:** Kong, AWS API Gateway, NGINX, Envoy

**What is rate limited:** consumer (user/API key), endpoint, plan tier

**Why at the gateway:**
- Single enforcement point before any service receives traffic
- Centralised policy: change limits without touching service code
- Can enforce SaaS plan tiers (Free: 100/day, Pro: 10K/day)

**Limitation:** Gateway doesn't understand business-level context (e.g., "this user has a payment overdue — suspend their account") — that belongs inside the service.

### 11.3 Rate Limiting Inside a Microservice

**What is rate limited:** Calls to specific *other* services, DB writes, third-party API calls

**Pattern — Client-side rate limiter (protecting downstream):**
```python
# Service A limits its own calls to Service B
limiter = TokenBucket(capacity=50, refill_rate=10)  # max 10 B-calls/sec

def call_service_b(payload):
    if not limiter.consume():
        raise RateLimitError("Service B call throttled")
    return b_client.call(payload)
```

This protects Service B from Service A's own traffic spikes, and protects Service A from cascading failure if B is slow.

**Pattern — DB write throttling:**
```python
# Limit write rate to protect DB from bursts
db_write_limiter = TokenBucket(capacity=500, refill_rate=200)

def write_event(record):
    if not db_write_limiter.consume():
        queue.push(record)   # Buffer to async queue instead of dropping
    else:
        db.insert(record)
```

### 11.4 Rate Limiting Third-Party APIs

When *you* call an external API (Stripe, Twilio, SendGrid, OpenAI), they enforce rate limits on you. Your system must respect them:

```
Third-party limits you → your service must:
  1. Track your own usage (don't rely on hitting 429 to know you're over)
  2. Token bucket on your outbound calls
  3. Retry with exponential backoff + jitter on 429
  4. Honour Retry-After header from the provider
  5. Alert / circuit-break if rate limited for > N minutes
```

**Anti-pattern:** Fire-and-forget retries on 429 without backoff → retry storm makes the problem worse.

---

## 12. Centralized vs Decentralized Enforcement

| Model | How | Pros | Cons |
|---|---|---|---|
| **Centralized** | All nodes share a Redis counter | Exact enforcement, consistent | Redis in hot path, SPOF risk |
| **Decentralized (local)** | Each node tracks locally | No latency, no SPOF | Allows N×limit if N nodes |
| **Semi-centralized** | Local counter + periodic sync to Redis | Low Redis pressure | Allows burst in sync window |
| **Sidecar proxy** | Envoy/Istio enforces per-pod | No app code change, mesh-wide | Sidecar overhead, config complexity |

**Practical recommendation:**

- **Hard security limits** (login attempts, API key abuse): centralised Redis — exactness required
- **Soft throughput limits** (developer API quota): semi-centralised is fine — slight over-allowance acceptable
- **Internal service-to-service throttling**: sidecar (Envoy) or client-side token bucket
