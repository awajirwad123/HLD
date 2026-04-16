# Circuit Breaker & Resilience Patterns — Tricky Interview Questions

## Question 1

**"You add retries to all your service calls to make the system more resilient. Within a week, a downstream service that was having minor trouble is now completely down. What went wrong?"**

### Why it's tricky
Retries feel obviously good — candidates who just "add retries everywhere" without understanding amplification will be surprised by this.

### Strong Answer

Retries amplified a minor problem into a total failure. Here's the math:

```
Downstream: 10% transient errors at 1,000 RPS
Without retries: 1,000 RPS hitting downstream (100 errors returned to user — acceptable)
With 3 retries: each error retries 3×
  1,000 × (1 + 0.1 × 3) = 1,300 RPS effective load on downstream
  At 10% error rate, retries of retries cause further overload
  Positive feedback loop → downstream saturates → 100% errors → 4,000 RPS from all retries
```

This is **retry storm** — retries under partial failure create a positive feedback loop that turns a degraded service into a completely down one.

**Fixes**:
1. **Retry budget**: cap retries at a percentage of total calls (e.g., 10%). When budget is exhausted, fail-fast instead of retrying.
2. **Exponential backoff + jitter**: spread retries over time instead of retrying immediately, giving the downstream breathing room.
3. **Circuit breaker + retry**: the circuit breaker opens when failure rate exceeds the threshold, stopping all retries (and regular calls) until the downstream recovers. Retries contribute to the failure count that trips the circuit.
4. **Only retry idempotent operations**: ensure the retry doesn't cause a different kind of overload (duplicate writes).

The lesson: resilience patterns must be composed correctly. Retry without circuit breaker + budget is sometimes worse than no retry at all.

---

## Question 2

**"Your circuit breaker is configured to open at 50% failure rate with a 30-second window and min 10 calls. Suddenly, after a deployment, 2 out of 3 calls fail (66%). The circuit doesn't open. Why not?"**

### Why it's tricky
Tests understanding of `min_calls` and how it interacts with small sample windows at the start of a rolling period.

### Strong Answer

The circuit requires `min_calls = 10` before it will open. After the deployment, only 3 calls have been made in the current 30-second window. Even though 2/3 = 66% failure rate (above the 50% threshold), the circuit stays CLOSED because the sample size is below `min_calls`.

This is expected and correct behaviour — without `min_calls`, a circuit would open on 1 failure out of 1 call (100% rate), even in completely normal operation during a quiet period.

**The problem**: `min_calls = 10` means you need 10 calls before the circuit can open. If your service only gets 1–2 calls/second on this endpoint, it takes 10 seconds of failures before the circuit opens. During those 10 seconds, users are getting errors.

**How to tune**: Set `min_calls` relative to expected traffic, not a fixed number. For low-traffic endpoints (1 RPS), `min_calls = 3` is more appropriate. For high-traffic endpoints (100 RPS), `min_calls = 20` is fine — you'll collect that in 200ms.

Also consider: a time-based window with a short `min_duration` (e.g., "open if failure rate > 50% for at least 5 seconds") instead of count-based minimum.

---

## Question 3

**"A service has a circuit breaker that opens after 50% failure rate. Service B goes down. The circuit opens. 30 seconds later, the circuit moves to HALF_OPEN and sends one probe. The probe succeeds (Service B came back). Circuit closes. Then Service B crashes again. What design issue is this?"**

### Why it's tricky
Exposes the false-positive recovery problem in circuit breakers — a single probe success is not statistically reliable.

### Strong Answer

This is the **premature close** problem. A single successful probe is insufficient evidence that Service B has truly recovered. One successful call could be:
- A race condition: B came back briefly and crashed again
- Lucky timing: the request hit the one healthy instance in a partially recovered cluster
- A cached/stale response from a load balancer that didn't yet know B was down

**Fix 1 — `success_threshold` > 1**: Require multiple consecutive successes in HALF_OPEN before closing (e.g., 3 successes). Hystrix defaults to 1; Resilience4j allows configuring this.

**Fix 2 — Slow start after close**: After closing from HALF_OPEN, don't immediately pass full traffic. Start at 10% of requests, ramp up over 30 seconds. If failures reappear during the ramp, reopen immediately. This is the "slow recovery" or "gradual recovery" pattern used in Envoy's `outlierDetection`.

**Fix 3 — Monitor after close**: Keep tracking failure rate for 30 seconds after closing. If it immediately exceeds the threshold again, reopen with a longer `recovery_timeout` (double it each time — to avoid oscillation).

The meta-lesson: circuit breakers need a "confidence interval" for recovery, not binary success/fail on one probe.

---

## Question 4

**"You have a timeout of 5 seconds on a call to Service B. Service B internally has a timeout of 10 seconds on its call to Service C. Service C takes 8 seconds. What happens?"**

### Why it's tricky
Tests deadline propagation understanding and what wasted work looks like.

### Strong Answer

From the user's perspective:
- User gets a timeout error at 5 seconds (Service A's timeout fires).
- The user's request is done. They see an error.

From the system's perspective:
- Service B is still running. It's waiting for Service C with a 10-second timeout.
- At 8 seconds, Service C responds. Service B finishes its work.
- Service B tries to respond to Service A — but Service A already closed the connection (5-second timeout fired).
- **Service B did 8 seconds of work for nothing**: CPU, DB queries, downstream calls, latency — all wasted.

**Impact at scale**: If this happens for 10% of requests at 1,000 RPS = 100 wasted operations/second at 8 seconds each = 800 seconds of wasted work/second. At some point, this saturates Service B's thread pool with wasted work, causing real requests to queue.

**Fix — Deadline propagation**:
Service A sets a deadline of "respond by T=5s." When calling Service B, it passes the remaining budget: `X-Deadline: {T}` or gRPC's `context.WithDeadline`. Service B checks: "do I have enough time to call C?" If `remaining < min_threshold`, it returns an error immediately without even calling C. If it calls C, it propagates the remaining budget minus local overhead.

gRPC handles this natively. For HTTP: pass `X-Request-Deadline: <unix_ms>` or `X-Remaining-Timeout-Ms: 4800` and honor it in every service.

---

## Question 5

**"Your team is building a new feature: 'show product recommendations'. The recommendation service has 99.5% uptime. Should you add a circuit breaker to this call?"**

### Why it's tricky
Tests whether candidates apply resilience patterns thoughtfully or reflexively everywhere.

### Strong Answer

Yes — but the reasoning matters as much as the answer.

99.5% uptime = **44 minutes of downtime per month**. Without a circuit breaker:
- During those 44 minutes, every product page request waits for the recommendation service to time out (e.g., 5 seconds)
- Users experience 5-second page loads instead of <200ms
- If the recommendation service is slow (not fully down), thread pool exhaustion occurs and other product page features fail too

With a circuit breaker:
- After N failures, the circuit opens
- Fallback: show an empty recommendations section (or a hardcoded "Popular Products" list from a static config)
- Users see a slightly degraded page but at full speed
- Core product page features (price, images, add-to-cart) are unaffected

**The right fallback**: Not an error message. The fallback should be invisible to users or clearly labelled. Options in priority order:
1. Return results from a Redis cache (ML output from the last successful response, TTL = 1 hour)
2. Return a static "Popular items" list from a config file
3. Return an empty array (hide the section via front-end conditional rendering)

**Configuration for a non-critical feature**: Be more aggressive about opening the circuit (lower threshold, faster open) because the fallback is benign. For a critical feature (e.g., payment service), be more conservative (higher threshold, require `min_calls`).

---

## Question 6

**"You're using both Istio (service mesh) for retries and your application's retry library (Resilience4j). What could go wrong?"**

### Why it's tricky
Tests the retry amplification problem at the infrastructure composition level — a real and common production mistake.

### Strong Answer

**Retry multiplication**: if Istio retries 3 times and Resilience4j also retries 3 times, a single failed request generates up to 3 × 3 = **9 actual attempts** to the downstream service. Under partial failure, 9× amplification from every client request.

**Scenario**:
- 1,000 RPS arriving
- Service B is degraded (30% failure rate)
- Istio: 3 retries each
- Resilience4j: 3 retries each
- Effective load on B: potentially up to 9,000 RPS = Service B gets overwhelmed and reaches 100% failure → your "resilience" caused the outage

**The fix**: pick one layer for retries. Industry standard:
- **Infrastructure retries (Istio)**: for network-level transient errors (connection refused, 503)
- **Application retries**: for business-semantic retries (idempotency-key retry after credit card timeout)

Codify this in an Architecture Decision Record (ADR): "Retries are configured only at the service mesh layer. Do not add HTTP retry logic in application code unless it's for an explicit business-logic case."

Also: ensure Istio retry conditions are specific (`retryOn: "5xx,gateway-error,connect-failure"`) — not blanket retries — and that your application-level code sets `x-envoy-max-retries: 0` on sensitive idempotent calls where you've already configured retries via Resilience4j.

---

## Question 7

**"The circuit breaker for your payment service opens. Your fallback is: 'return a 200 OK and queue the payment for async processing.' Is this a good fallback?"**

### Why it's tricky
Seems like a clever workaround — async queueing sounds resilient. But it trades one problem for several others.

### Strong Answer

It's dangerous unless designed extremely carefully. The specific risks:

**1. User expectation mismatch**: User sees "payment successful" (200 OK), but the payment hasn't happened yet. If the async queue fails or drains slowly, the user gets a service they haven't paid for — a financial liability. Or worse, the retry eventually double-charges.

**2. Queue depth under circuit-open conditions**: If the payment service circuit is open because the service is overwhelmed, adding queue items accelerates the problem on recovery. When the circuit closes, the queue flushes and creates a traffic spike.

**3. Partial order states**: The order is confirmed (DB write happened) but payment is pending. Your system must now track `payment_status = pending_queue`. Every downstream workflow (fulfilment, shipping) must check this status and block until payment confirms.

**When it IS appropriate**:
This pattern works if you've explicitly designed for it as the normal path (not a fallback): an async payment intent model where "order created" and "payment captured" are intentionally decoupled. In this case, the user sees "Order placed — payment processing" and the UI updates via webhook. WhatsApp and Stripe both handle async confirmation explicitly.

**Better fallbacks for a payment service circuit opening**:
- Return a 503 with a `Retry-After` and let the user retry consciously
- Show a "payment temporarily unavailable" message with a clear CTA to retry in 30 seconds
- Never silently queue a payment and claim it was successful

Human-observable, honest, recoverable > silent, "optimistic," potentially inconsistent.
