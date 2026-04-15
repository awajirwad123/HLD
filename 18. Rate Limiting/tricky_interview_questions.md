# Rate Limiting — Tricky Interview Questions

## Question 1

**"You implement rate limiting with Redis INCR — check count, if below limit increment, else reject. Your senior engineer says 'this has a race condition.' What is it, and how do you fix it?"**

### Why it's tricky
Candidates who suggest Redis confidently often miss that multi-step operations aren't atomic by default.

### Strong Answer

The race condition:

```
Thread A: GET counter → 99  (one below limit)
Thread B: GET counter → 99  (one below limit)
Thread A: INCR       → 100  ✓ allowed
Thread B: INCR       → 101  ✓ allowed — but limit is 100. Both got in.
```

Two concurrent readers both see the value below the limit; both increment; both exceed the intended threshold.

**Fix 1 — Lua script (preferred)**:
Wrap GET + conditional INCR in a Lua script. Redis guarantees Lua scripts execute atomically — no other command interleaves. One round-trip, no race.

```lua
local count = tonumber(redis.call("GET", KEYS[1]) or 0)
if count >= tonumber(ARGV[1]) then return 0 end
redis.call("INCR", KEYS[1])
redis.call("EXPIRE", KEYS[1], tonumber(ARGV[2]))
return 1
```

**Fix 2 — INCR-then-check**:
Always `INCR` first, then check if the result exceeds the limit. If it does, don't roll back — just reject. Slightly over-counts (last request actually incremented), but no under-counting.

```python
count = await redis.incr(key)
if count == 1:
    await redis.expire(key, window)   # Set TTL only on first request
if count > limit:
    return False  # reject
return True
```

This is correct under concurrency: the INCR is atomic. The only subtlety is setting the TTL: do it only when `count == 1` to avoid resetting it on every request. Use `SET key 1 EX {window} NX` for the initial creation and `INCR` thereafter for a fully atomic approach.

---

## Question 2

**"A user with a 100 req/min limit sends exactly 100 requests at 00:59:59 and 100 more at 01:00:01. Your fixed-window rate limiter allows all 200. Is this a bug or a feature?"**

### Why it's tricky
Tests whether candidates understand the trade-off is intentional, not accidental — and know what to do about it.

### Strong Answer

It's a **known trade-off**, not a bug — but whether it's acceptable depends on what you're protecting.

**Why it happens**: Fixed window resets at discrete boundaries. Both windows are independently at or below their limits. The rate limiter's contract ("100 per window") is technically satisfied — just in two consecutive windows that happen to be adjacent.

**When it matters**:
- **Security (login attempts)**: This double-burst is catastrophic. An attacker can make 200 guesses per effective minute. Use **sliding window** — no reset boundary.
- **API billing/fairness**: Usually acceptable. Plan tiers often use fixed minutes/hours. A tiny burst at boundaries doesn't meaningfully harm the business.
- **Protecting a backend DB at 100 queries/min**: Could be harmful — 200 in 2 seconds might overwhelm the DB. Use sliding window or token bucket.

**What to say in an interview**: "This is a known limitation of fixed-window counters. For security-sensitive limits I'd use sliding window or token bucket. For coarse billing quotas, fixed window is acceptable — simpler and cheaper."

---

## Question 3

**"Your rate limit is 100 req/min per user. You have 1 million active users. How much Redis memory does this require?"**

### Why it's tricky
Tests numerical reasoning and the memory implications of algorithm choice.

### Strong Answer

**It depends entirely on the algorithm:**

**Fixed Window Counter:**
- 1 key per user = 1 million keys
- Each key: ~50 bytes (key name + integer value + expiration metadata in Redis)
- Total: 1M × 50 bytes = **~50 MB** — trivially small

**Sliding Window Log (sorted set per user):**
- Each user: up to 100 entries (their full minute of requests) in a sorted set
- Each sorted set entry: ~16 bytes (score = timestamp, member = request UUID ~40 bytes) = ~56 bytes per entry
- Max: 1M users × 100 entries × 56 bytes ≈ **~5.6 GB** — significant
- For 10M users at high activity: ~56 GB — potentially unaffordable

**Sliding Window Counter (2 integers per user):**
- 2 keys per user (curr + prev window) = 2M keys
- ~50 bytes each = **~100 MB** for 1M users — very practical

**Conclusion**: Fixed window or sliding window counter at O(1) per user. Sliding window log is impractical at 1M+ users. The Cloudflare team explicitly chose sliding window counter to avoid the memory explosion of log-based approaches.

---

## Question 4

**"You add rate limiting middleware to your API Gateway. Within a week, a legitimate enterprise customer calls to say their integration is randomly failing. What happened, and how do you fix it?"**

### Why it's tricky
Tests whether candidates design rate limiting with customer experience in mind, not just security.

### Strong Answer

**Likely causes:**

1. **Shared IP rate limiting**: The enterprise is behind a corporate NAT — 500 employees share one outgoing IP. Your per-IP limit of 100 req/min is being hit by the combined load of all their employees' legitimate requests.

2. **Burst traffic from their integration**: A webhook handler or scheduled job fan-out fires 1,000 requests in a short burst — well within their monthly quota but over the per-minute limit.

3. **Retry amplification**: Their SDK retries on 5xx errors but not on 429s with proper backoff, causing the rate limit to hold while they hammer it.

**Fixes:**

1. **Identify on API key, not IP**: Authenticated enterprise customers should have their limit applied to their API key, not IP. IP limits should only be a fallback for unauthenticated requests.

2. **Plan-based limits**: Give enterprise customers a higher limit — 10,000 req/min vs 100 for free tier. Store limits in a config database keyed to their API key/org.

3. **Burst headroom**: Use token bucket with capacity = 5× steady rate. Allow legitimate bursts from scheduled jobs.

4. **Observability + alerting**: Surface rate limit hit rates in a dashboard. Proactively alert the customer (or your support team) before they notice failures. Include `X-RateLimit-Remaining` in all responses so the customer's SDK can self-throttle.

5. **Communicate limits clearly**: Document your rate limits in the API reference with the `Retry-After` semantics. Provide SDKs that respect it.

---

## Question 5

**"Interviewer: Design a rate limiter for a payment charge API where idempotency keys are used. Should the idempotency key retry count against the rate limit?"**

### Why it's tricky
Intersection of two important patterns. The correct answer is nuanced — not "yes" or "no."

### Strong Answer

Depends on what the rate limit is protecting:

**Case 1 — Rate limit for capacity protection (server load)**:
A retry with the same idempotency key that returns a cached result does essentially zero work on the server — cache lookup only, no DB write, no downstream call. The retry should **not** count against the rate limit (or count at a heavily discounted rate) because the server isn't actually doing the protected work twice.

**Case 2 — Rate limit for fraud/security (charge attempt frequency)**:
If the rate limit exists to prevent a customer from attempting many charges quickly (e.g., 10 charges/hour limit), then a retry with the same idempotency key represents the same charge attempt — it should **not** increment the counter. Counting it would punish a client for doing the right thing (retrying safely).

**Implementation:**
When a request arrives with an idempotency key that's already in the cache:
1. Return the cached response immediately.
2. Do not increment the rate limit counter for this key (`INCR` call is skipped).

When a request arrives with a **new** idempotency key:
1. Run the full rate limit check and increment.
2. Store the response in the idempotency cache.

This way: 1 real charge attempt = 1 counter increment, regardless of how many times the client retries with the same key. Stripe's implementation works this way — retrying a failed charge request with the same idempotency key is safe and doesn't count against your rate limit.

---

## Question 6

**"Your rate limiter uses Redis. Redis goes down for 30 seconds. What should your API do?"**

### Why it's tricky
Forces candidates to reason about fail-open vs fail-closed — a deliberate architectural decision, not a default.

### Strong Answer

This is a **fail-open vs fail-closed** policy decision. Neither is universally correct.

**Fail-closed (reject all requests when Redis is down)**:
- Safer for security-sensitive limits (login endpoints, payment charges)
- Terrible for availability — a 30-second Redis outage takes down every API endpoint
- Acceptable only if the rate limiter is a security control, not just fairness enforcement

**Fail-open (allow all requests when Redis is down)**:
- Maintains availability — clients don't notice the Redis outage
- Temporary unprotected window (30 seconds for us, could be exploited)
- Acceptable for most API rate limits where the risk of a short unprotected window is low vs the cost of an outage

**Best practice in production (hybrid)**:
1. Circuit-break Redis: if Redis is consistently failing, switch to local in-memory limiter with conservative limits (e.g., 50% of normal limit)
2. Set a short timeout on Redis calls (e.g., 10ms): if Redis doesn't respond in 10ms, fall back to local limiter rather than waiting and blocking
3. Alert + auto-remediate: Redis downtime should trigger PagerDuty immediately; this is a P1 infrastructure event

**What most large APIs do**: Fail-open with local fallback limiting. A 30-second unprotected window at Stripe is an acceptable risk; but API availability failing is not.

---

## Question 7

**"A client hits your rate limit and starts retrying with exponential backoff. But because all clients retry on the same schedule, they all hammer you simultaneously every 2 seconds. How do you prevent this?"**

### Why it's tricky
Tests knowledge of thundering herd and jitter, which appears in both client-side and server-side retry design.

### Strong Answer

This is the **thundering herd retry** problem. All clients backed off to exactly the same interval (e.g., 2s base, attempts in sync because they all hit the limit at the same moment) and now retry in lockstep.

**Server-side mitigation:**

1. **`Retry-After` staggering**: Instead of returning a fixed `Retry-After: 60`, return a randomised value within a reasonable range: `Retry-After: {random(55, 65)}`. Now clients that all received 429 at the same moment will retry spread across a 10-second window.

2. **Progressive backoff response**: Return a longer `Retry-After` for clients that have already been rejected multiple times within a window (tracked in Redis per client per window).

**Client-side mitigation (the real fix):**

All well-behaved clients should implement **jitter** in their retry logic:

```python
# Full jitter (AWS recommended)
sleep = random.uniform(0, min(cap, base * 2 ** attempt))

# Equal jitter
v = min(cap, base * 2 ** attempt)
sleep = v / 2 + random.uniform(0, v / 2)
```

Both approaches spread retries over time, breaking the lockstep.

**In the `Retry-After` response**: the server provides the *minimum* wait. Clients should still add jitter on top of it. A well-designed SDK (like Stripe's) does this automatically.
