# Rate Limiter Service — Tricky Interview Questions

## Q1: "Two API gateway nodes each have a local Redis cache for rate limit counters. User sends 100 requests across both nodes. How do they see a consistent limit of 100?"

**What they're testing:** Distributed counter accuracy, Redis consistency, local vs shared state.

**Problem:** If Node A caches `count=50` locally and Node B caches `count=50` locally, each thinks the user is at 50/100 and allows another 50. Total: 150 requests pass, 50% over limit.

**Root cause:** Local in-process caches are per-node. With N nodes each caching independently, the effective limit becomes `limit × N`.

**Solutions:**

**Option 1: Redis as single source of truth (no local cache)**
Every request hits Redis. At ~200µs per check, this adds ~1ms of overhead. Acceptable for most APIs. Simple, accurate.

**Option 2: Local approximate limiter (accept some over-counting)**
Each node tracks a local counter. Every 500ms, nodes synchronize with Redis (batch INCR, read total). Accept up to `limit × N × 0.5s / window` over-counting. Works if limit >> N.

**Option 3: Divide the limit by N (token pre-allocation)**
Each gateway node pre-allocates `limit/N` tokens from Redis at startup. When its local allocation is exhausted, fetches more. Precise, but complex. Uneven traffic distribution means some nodes exhaust early while others have tokens left.

**Option 4: Sticky routing**
Load balancer routes all traffic for a given API key/user to the same gateway node. Single node owns the counter — no sharing needed. Downside: uneven load if some users are more active, loss of sticky assignment when node restarts.

**Strong answer:** For most use cases, central Redis is correct and fast enough (< 1ms). Only optimize if benchmarks show Redis is the bottleneck.

---

## Q2: "Design a rate limiter that applies different limits to different API endpoints: /search gets 10 req/min, /checkout gets 3 req/min, everything else gets 100 req/min."

**What they're testing:** Configuration-driven rate limiting, key design, flexibility.

**Approach: Route-aware rate limiting with configuration**

```python
ENDPOINT_LIMITS = {
    "/search":    (10,  60),   # (limit, window_seconds)
    "/checkout":  (3,   60),
    "/login":     (5,   60),
    "default":    (100, 60),
}

def get_limit(path: str) -> tuple[int, int]:
    # Exact match first, then prefix, then default
    for pattern, (limit, window) in ENDPOINT_LIMITS.items():
        if pattern == "default":
            continue
        if path.startswith(pattern):
            return limit, window
    return ENDPOINT_LIMITS["default"]
```

**Key design for per-endpoint per-user:**
```
ratelimit:{user_id}:{endpoint_slug}:{window_id}
ratelimit:42:search:28300
ratelimit:42:checkout:28300
```

**Configuration management:**
- Store endpoint limit config in Redis or a config service (not hardcoded).
- Hot-reload config without restarting gateways.
- Example using Redis Hash: `HSET endpoint_limits /search "10:60" /checkout "3:60"`
- Gateway reads config at startup + listens for updates via Redis Pub/Sub.

**Regex/wildcard matching:**
```
/api/v*/items  → 50 req/min
/admin/*       → 5 req/min (admin endpoints are special)
```
Use a priority-ordered list of patterns. First match wins.

**Interview tip:** Mention that in production this is usually handled at the API gateway (Kong, NGINX) through route-level plugin configuration, not custom code.

---

## Q3: "Your rate limiter is causing high latency at peak traffic. Every request has to wait 5ms for Redis. How do you fix this?"

**What they're testing:** Performance optimization, caching strategies, bottleneck analysis.

**Root causes of 5ms Redis latency:**
1. Redis is in a different availability zone (network RTT = 3–4ms)
2. Redis CPU is saturated (too many concurrent EVALSHA calls)
3. Connection pool exhaustion (requests queue waiting for a Redis connection)

**Optimizations:**

**1. Move Redis to same AZ:**
Redis round-trip in same AZ = 0.1–0.3ms. Different AZ = 1–4ms. Simple infra fix that gives 5–10× improvement.

**2. Redis pipelining / batching:**
If a single request checks 3 rate limit tiers, pipeline all 3 Lua scripts into one network round-trip:
```python
async with redis.pipeline() as pipe:
    pipe.evalsha(script, ..., ip_key, limit_1)
    pipe.evalsha(script, ..., api_key, limit_2)
    pipe.evalsha(script, ..., user_key, limit_3)
    results = await pipe.execute()
```

**3. Local approximate limiter (50ms sync):**
Maintain a local in-process token bucket. Sync with Redis every 50ms. Most requests served locally at ~0µs. Redis sync amortizes over all requests in the 50ms window. Accept ~5% over-limit during sync window.

**4. Shadow mode / pre-check:**
For clearly over-limit clients (counter >> limit), use a fast Redis GET (not Lua) to check. Only run the full Lua script for borderline cases. Reduces Lua script overhead by 80%.

**5. Redis Cluster for horizontal scaling:**
Shard rate limit keys across multiple Redis nodes. Each node handles a subset of users. Throughput scales linearly with nodes.

**6. Redis 7+ Function Scripts:**
Same as EVALSHA but pre-compiled server-side. Marginally faster for complex scripts.

**Trade-off to state:** The local approximation (option 3) is usually the best answer — 10–100× latency improvement, < 5% accuracy loss, acceptable for most rate limiting use cases.

---

## Q4: "A scraper is bypassing your rate limiter by using 10,000 different IP addresses. What do you do?"

**What they're testing:** Adversarial thinking, rate limiting limitations, complementary defenses.

**The problem:** IP-based rate limiting is easily bypassed with IP rotation (botnets, residential proxies, cloud VMs). Scraper can distribute requests across thousands of IPs, never hitting per-IP limits.

**Additional signals beyond IP:**

1. **User-agent pattern:** Bots often use recognizable user-agent strings or rotate through a known set. Block known bot UAs.

2. **Browser fingerprint:** JavaScript challenge (CAPTCHA, proof-of-work) that real browsers pass and bots typically fail.

3. **Behavioral analysis:**
   - Bots have perfectly regular inter-request timing (inhuman consistency).
   - Bots don't follow normal navigation patterns (e.g., no referrer headers, never loading CSS/images).
   - Bots often access sitemap URLs directly.

4. **Request pattern fingerprinting:** If all 10,000 IPs request the same URL pattern, same time window, same query structure → clearly coordinated. Aggregate at the pattern level, not just the IP level.

5. **API key requirement:** Make scrapers register for an API key. Scraping without a key → blocked. Scrapers with keys are tied to an account → can be suspended. This also distinguishes legitimate third-party integrations from scrapers.

6. **Rate limiting by ASN / organization:**
   - Many cloud scraper IPs belong to the same ASN (AS15169 = Google, AS14618 = AWS).
   - Rate limit at ASN level: `ratelimit:asn:{asn_number}`. Flags unusual concentration from one provider.

7. **Honeypot URLs:** Pages that only bots visit (invisible to real users). IPs that hit these get banned.

**Practical answer:** Rate limiting alone can't stop sophisticated scrapers. Combine rate limiting with: API key authentication, behavioral analysis, CAPTCHA challenges, and ASN-level blocking. CAPTCHAs are the most effective tool against botnet scraping.

---

## Q5: "Should a rate limiter fail-open or fail-closed, and what is a 'circuit breaker' pattern?"

**What they're testing:** Operational reliability, circuit breaker pattern, security vs availability trade-offs.

**Fail-open:** When the rate limiter (Redis) is down, allow all requests. The API stays up at the cost of no limiting during the outage.
**Fail-closed:** When the rate limiter is down, reject all requests. The API is protected at the cost of downtime.

**Which to choose:**
- General API endpoints: **fail-open**. A 30-second Redis outage shouldn't take down your API for all users.
- Login/auth endpoints: **fail-closed**. A brief Redis outage is preferable to allowing unlimited auth attempts (brute force risk).
- Payment endpoints: **fail-closed**. Unlimited $1,000 API calls during an outage could be catastrophic.

**Circuit breaker pattern:**
The circuit breaker sits between the rate limiter and Redis. It tracks Redis call failures:

```
States:
  CLOSED   → Normal: all Redis calls go through
  OPEN     → Redis is broken: fail-open (or closed) immediately, don't call Redis
  HALF-OPEN → After timeout: allow 1 probe call to test if Redis recovered
                If probe succeeds → CLOSED
                If probe fails   → OPEN again
```

Implementation:
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.failures = 0
        self.state = "CLOSED"
        self.last_failure = 0
        self.threshold = failure_threshold
        self.timeout = timeout

    async def call(self, fn, *args, fallback=None):
        now = time.time()
        if self.state == "OPEN":
            if now - self.last_failure > self.timeout:
                self.state = "HALF-OPEN"
            else:
                return fallback  # Fast-fail: don't call Redis
        try:
            result = await fn(*args)
            self.failures = 0
            self.state = "CLOSED"
            return result
        except Exception:
            self.failures += 1
            self.last_failure = now
            if self.failures >= self.threshold:
                self.state = "OPEN"
            return fallback
```

With a circuit breaker, Redis failures are detected quickly, and the fallback (fail-open or local limiter) activates automatically. Redis is given time to recover before probe calls restart.
