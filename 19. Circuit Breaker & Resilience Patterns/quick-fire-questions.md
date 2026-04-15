# Circuit Breaker & Resilience Patterns — Quick-Fire Q&A

## Circuit Breaker

**Q1. What are the three states of a circuit breaker and what happens in each?**
- **CLOSED**: Normal operation. Requests pass through. Failures are counted in a rolling window.
- **OPEN**: All requests are fast-failed immediately without calling the downstream. The circuit opened because failure rate exceeded the threshold.
- **HALF_OPEN**: After the recovery timeout, one probe request is allowed through. Success → CLOSED. Failure → back to OPEN.

**Q2. Why is a `min_calls` parameter important in a circuit breaker?**
Without it, a circuit opens on tiny samples: 1 failure out of 1 call = 100% failure rate → circuit opens. `min_calls` (e.g., 10) ensures there's a statistically meaningful sample before the circuit can open.

**Q3. What is the difference between count-based and rate-based failure thresholds?**
Count-based: open after N absolute failures (e.g., 5 failures). Simple but misleading under low traffic — 5 failures at 1,000 RPS is noise; 5 failures at 5 RPS is total breakdown. Rate-based: open when failure *percentage* exceeds the threshold (e.g., >50%) AND minimum calls are met. More accurate across different traffic levels.

**Q4. How does a circuit breaker prevent cascading failure?**
When open, it fast-fails in microseconds instead of waiting for a 30-second timeout. The calling service's thread pool stays free to handle other requests unrelated to the unhealthy downstream. The downstream gets time to recover without being further bombarded.

**Q5. What triggers the transition from OPEN to HALF_OPEN?**
The `recovery_timeout` (e.g., 30 seconds) elapsing. The circuit breaker polls: "has enough time passed to try the downstream again?" If yes, it allows one probe request. This is a time-based recovery, not an external notification.

---

## Retry

**Q6. Why must retries on a POST request require extra care?**
POST is not guaranteed to be idempotent. If the original request was received and processed but the response was lost, a retry causes the operation to execute twice (double charge, duplicate order). Retrying POST is only safe if the endpoint is explicitly idempotent via idempotency keys.

**Q7. What is exponential backoff with jitter, and why is jitter critical?**
Exponential backoff: each retry waits twice as long as the previous (`base × 2^attempt`). Jitter: multiply by a random factor in `[0, 1]` to randomise the delay. Without jitter, all clients that hit a failure at the same time retry at the same intervals in lockstep — a thundering herd that re-floods the recovering service. Jitter staggers them.

**Q8. What is a retry budget and why does it matter?**
A retry budget caps the ratio of retries allowed (e.g., "retries may not exceed 10% of total calls"). Without it, when a service is down, every request retries 3× → the downstream receives 4× the normal load at exactly the worst time. A retry budget bounds the amplification factor.

**Q9. Should you retry on HTTP 500?**
Not by default. A 500 means the server encountered an error — it may have already processed the request. Retrying could cause duplicate side effects. Retry 500 only if the endpoint is explicitly idempotent. Contrast with 503 (server is temporarily unavailable, never started processing) — safer to retry.

---

## Fallback

**Q10. What are the four main fallback strategies?**
1. **Cached response**: return the last known good value (e.g., stale product price)
2. **Default value**: return a safe hardcoded response (e.g., empty recommendations list)
3. **Degraded feature**: skip the feature entirely (e.g., hide "recommended for you" section)
4. **Alternative source**: call a secondary service or read replica

**Q11. What is the "fallback that calls the failing service" anti-pattern?**
A fallback that calls the same failing downstream just defers the failure by one level. E.g., primary payment service fails → fallback calls *a different method on the same payment service* → also fails. The fallback must use a genuinely different, more resilient data source (cache, DB, static response).

**Q12. When should you show an explicit error vs silent degradation?**
Show explicit error when the user needs to take action or the degraded result is misleading (wrong price, wrong availability). Silent degradation (e.g., hiding a recommendations widget) is acceptable when the feature is non-critical. Silently showing stale data as if it's current without informing the user is the most dangerous — it can cause business errors.

---

## Bulkhead

**Q13. What problem does the bulkhead pattern solve?**
Resource pool exhaustion: without isolation, a slow downstream A can consume all threads/connections in a shared pool, starving unrelated calls to downstream B and C. Bulkhead assigns separate pools per downstream so A's slowness is contained in A's pool.

**Q14. What is the difference between a thread pool bulkhead and a semaphore bulkhead?**
Thread pool bulkhead: each downstream gets its own thread pool — true isolation. A slow call in pool A never touches pool B's threads. Semaphore bulkhead: a semaphore limits concurrent calls, but all calls run on the same shared thread pool. It limits concurrency but doesn't give true thread isolation. Semaphore is lighter; thread pool is stronger isolation.

---

## Timeout

**Q15. Why should you never set a timeout to infinity?**
Under load, if the downstream becomes unresponsive, threads/goroutines accumulate waiting forever. The service's thread pool or goroutine limit is exhausted. New requests can't be handled. The service itself becomes unresponsive — effectively a self-inflicted DoS.

**Q16. How do you calibrate a timeout for a downstream service?**
Measure the P99 latency of the downstream under normal load. Set the timeout to 3–5× P99. Never set it at P50 (rejects half of normal traffic) or P100 (one outlier blocks your thread indefinitely). P99 captures legitimate slow calls while bounding thread occupancy.

**Q17. What is deadline propagation and which framework handles it natively?**
Deadline propagation: the total time budget set by the user-facing request is decremented at each service hop. Downstream services check remaining budget before starting work and abort if it's expired. gRPC handles this natively via `context.deadline` — each `CallOption` propagates the deadline to downstream RPCs. HTTP requires manually forwarding a header (e.g., `Grpc-Timeout` or a custom `X-Deadline`).

---

## Tools & Real-World

**Q18. What is Hystrix, and why is it in maintenance mode?**
Netflix Hystrix is the original Java circuit breaker library (2012) that popularised the bulkhead + circuit breaker + fallback composition model. It's in maintenance mode (no new features) because it uses thread pool bulkheads which are heavyweight for reactive/async programming. Resilience4j is its lightweight successor.

**Q19. What does Resilience4j offer over Hystrix?**
Modular (use only what you need), supports both thread pool and semaphore bulkheads, designed for Java 8+ functional style, works with reactive frameworks (RxJava, Reactor, WebFlux), lightweight (no transitive dependencies), and emits Micrometer metrics natively for Prometheus/Grafana integration.

**Q20. When should you use a service mesh (Istio) for resilience instead of application-level libraries?**
Mesh for: transport-level retries, timeouts, and basic circuit breaking applied uniformly across all services — no code changes per service, language-agnostic. Application library for: business-aware fallbacks (return stale cache, degrade a specific feature), nuanced error classification (retry 503 but not 500), and when the fallback requires application context. Production systems use both: mesh for infrastructure-level consistency, application library for business logic.
