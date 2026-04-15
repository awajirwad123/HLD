# Rate Limiter Service — Interview Simulator

## Scenario 1: "Design a Rate Limiter for a Public REST API"

**Prompt:** "You're building an API platform (like Stripe or Twilio). Design a rate limiting system that enforces per-customer quotas, protects against abuse, and adds minimal latency overhead."

---

**Clarifying questions to ask:**

1. What are the expected traffic levels? (Requests/sec, number of customers)
2. Are limits per customer, per endpoint, or both?
3. What happens when a customer hits the limit — hard reject or graceful degradation?
4. Do different pricing tiers get different limits? (Free: 100/min, Pro: 10,000/min)
5. What's the latency budget for rate limit overhead?

---

**Model answer outline:**

**Scale estimates:**
- 10,000 customers × 1,000 req/min average = 10M req/min = ~167K req/sec
- Redis at 100K ops/sec per core → need a few Redis nodes

**Architecture:**

```
Request arrives
    ↓
API Gateway (N instances, load-balanced)
    ↓
Rate Limit Middleware:
  1. Extract API key from Authorization header
  2. Fetch customer tier → get limit from config
  3. Check Redis rate limit (sliding window counter)
  4. Allow → forward to backend + add headers
     Deny  → return 429 + Retry-After
    ↓
Application Servers
```

**Redis key design:**
```
rl:{hashed_api_key}:{window_60s}     → per-minute limit
rl:{hashed_api_key}:{window_3600s}   → per-hour limit (secondary quota)
```

Hash the API key before using it as a Redis key — avoids leaking the key in Redis memory dumps.

**Tier-based limits:**
```python
TIER_LIMITS = {
    "free":       {"per_minute": 100,    "per_day": 1_000},
    "starter":    {"per_minute": 1_000,  "per_day": 50_000},
    "pro":        {"per_minute": 10_000, "per_day": 1_000_000},
    "enterprise": {"per_minute": None,   "per_day": None},  # No limit
}
```

Tier info cached in Redis (5-minute TTL) to avoid a DB call per request.

**Response headers (Stripe-style):**
```http
X-RateLimit-Limit-Minute: 1000
X-RateLimit-Remaining-Minute: 347
X-RateLimit-Reset-Minute: 1716873600
X-RateLimit-Limit-Day: 50000
X-RateLimit-Remaining-Day: 42100
```

**Latency:**
- Redis in same AZ: ~200µs per check
- Pipeline both minute and day checks: one round-trip = ~200µs total
- Total overhead: < 1ms

**Quota overage handling:**
- Hard block: immediately return 429 for the remainder of the window
- Soft overage (paid tiers): allow up to 110% of limit, charge per request over quota, send weekly usage alert

**Quota replenishment notification:**
- When customer first hits 429, send a webhook to their configured URL
- Include `headers.X-RateLimit-Reset` so their system can self-throttle

---

## Scenario 2: "Login Rate Limiting at Scale"

**Prompt:** "Design rate limiting specifically for a login endpoint to prevent brute-force attacks without blocking legitimate users."

---

**Special considerations for login:**

Unlike a general API, login has asymmetric failure cost:
- Too loose: attackers brute-force passwords
- Too tight: legitimate users locked out (reputation damage, support load)

**Multi-dimensional limits:**

```
1. Per IP: max 20 failed attempts per 15 minutes
   → Protect against single-IP brute force
   
2. Per username: max 10 failed attempts per 15 minutes  
   → Protect against credential stuffing across IPs
   
3. Global: max 10,000 login failures per minute
   → Protect against distributed attacks (botnet)
```

**Fail-closed:** With Redis down, reject all login attempts (no auth = no brute force).

**Progressive backoff (not immediate lockout):**
```
1–5 failures:   no penalty
6–10 failures:  CAPTCHA required
11–20 failures: 30-second lockout between attempts
21+ failures:   30-minute lockout; alert security team
```

This is better than binary lockout (reduces DoS risk for usernames).

**Redis key design:**
```
login:fail:ip:{hashed_ip}:{window_900s}      # 15-minute window
login:fail:user:{hashed_username}:{window_900s}
login:global_fail:{window_60s}
login:locked:user:{hashed_username}          # TTL = lockout duration
```

**Account lockout vs. rate limiting difference:**
- Rate limiting: limits attempt frequency ("max 10 per 15 min")
- Account lockout: hard block after N failures ("account locked until admin reset")

For most consumer apps: rate limiting (no lockout) is preferred — genuine lockout causes support tickets and is a DoS vector (attacker locks all target accounts).

**Monitoring:**
- Alert when global failure rate exceeds 500/sec → coordinated attack
- Alert when any single username has > 50 failures in 1 hour → targeted attack
- Alert when IP failure rate > 200/min → brute force bot

---

## Scenario 3: "Debugging — Legitimate Users Are Getting Rate Limited"

**Prompt:** "Customers are complaining of unexpected 429 errors even though they believe they're within their quota. How do you investigate?"

---

**Investigation framework:**

**Step 1: Quantify the problem**
- How many customers? (One? Many?)
- Which tier? (Free users hitting 100/min is expected)
- What time? (Was there a spike in traffic? Deploy?)
- Which endpoint? (One endpoint or all?)

**Step 2: Pull the actual counter values**

```python
import redis, time, hashlib

r = redis.Redis()
api_key = "customer_reported_key"
h = hashlib.sha256(api_key.encode()).hexdigest()[:8]
window = int(time.time()) // 60

print(r.get(f"rl:{h}:{window}"))       # Current minute count
print(r.get(f"rl:{h}:{window-1}"))     # Previous minute count
```

If both are low: the customer isn't actually over limit. Look at application logs.

**Step 3: Check for clock skew causing narrow windows**

If API gateway servers have different system clocks (NTP drift), `window_id = int(time.time()) // 60` gives different values on different nodes. A request hitting Node A and a request hitting Node B may use different window IDs — request A's count is invisible to Node B.

Test: log `time.time()` on all gateway nodes. If they differ by > 100ms, fix NTP.

**Step 4: Check for shared API key**

One "customer" may actually be multiple backend services all using the same API key. From the rate limiter's view it's one entity. Check: are multiple deployments (staging + prod) using the same key?

**Step 5: Check for retry storms**

Customer's code gets a 429 → retries immediately → gets another 429 → retries immediately → makes the rate limit worse. If client doesn't respect `Retry-After`, they generate 10× the desired traffic.

Fix: ensure clients implement exponential backoff + respect `Retry-After` header.

**Step 6: Check for Redis key collision**

If hashing API keys to short hashes (8 chars = 4 bytes = 4 billion values), two different API keys might produce the same hash. A heavily-loaded customer shares a rate limit counter with another customer.

Fix: use longer hashes (16+ chars) or don't hash (keys are opaque strings, safe to use directly as Redis keys after stripping PII).

**Root causes (priority order):**
1. Shared API key across multiple deployments
2. Client not respecting `Retry-After` (retry storm)
3. Clock skew between gateway nodes (rare with proper NTP)
4. Hash collision (very rare with SHA-256 prefix)
5. Tiering bug: customer on Pro plan but rate limiter reading Free tier limits from stale cache

**Fix for stale tier cache:**
```python
# After customer upgrades plan: immediately invalidate their tier cache
await redis.delete(f"customer_tier:{api_key}")
# Also: set a short TTL (5 min) so stale data self-heals
```
