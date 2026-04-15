# Circuit Breaker & Resilience Patterns — Architecture & Concepts

## 1. Why Resilience Patterns Exist

In a distributed system, any downstream dependency can be slow, overloaded, or unavailable at any moment. Without explicit resilience, a slow dependency causes:

```
User Request
  → Service A (waits 30s for Service B)
  → A's threads pile up waiting for B
  → A's thread pool exhausts
  → A now fails for ALL users, not just B-dependent ones
  → Services C and D that call A now also fail
```

This is **cascading failure** — one unhealthy node takes down the whole graph.

Resilience patterns intercept this at the call boundary and define **controlled degradation** rather than uncontrolled collapse.

The four core patterns: **Circuit Breaker**, **Retry**, **Fallback**, **Bulkhead**. They compose — a real system uses all four together.

---

## 2. Circuit Breaker

Named after the electrical circuit breaker: "stop the flow when conditions are dangerous."

### State Machine

```
             N failures in window
CLOSED ──────────────────────────────► OPEN
(normal)                               (fast-fail)
   ▲                                       │
   │    M consecutive successes            │ recovery_timeout elapsed
   │                                       ▼
   └──────────────────────────── HALF_OPEN
                                  (probe: 1 request through)
```

| State | Behaviour |
|---|---|
| **CLOSED** | Requests pass through. Failures are counted. Normal operation. |
| **OPEN** | All requests fail immediately (fast-fail) without calling the downstream. |
| **HALF_OPEN** | One probe request is sent. Success → CLOSED. Failure → OPEN again. |

### Key Configuration Parameters

| Parameter | Typical value | Meaning |
|---|---|---|
| `failure_threshold` | 50% or N failures | Open circuit when failures exceed this in the rolling window |
| `rolling_window` | 10–60 seconds | Time window for counting failures |
| `min_calls` | 10–20 | Minimum calls before circuit can open (avoids opening on 1 failure) |
| `recovery_timeout` | 30–60 seconds | How long to stay OPEN before probing with HALF_OPEN |
| `success_threshold` | 3–5 | Successes needed in HALF_OPEN to close the circuit |

### Why Fast-Fail is Better Than Waiting

Without a circuit breaker, each failed call takes `timeout` seconds. With 100 threads waiting for a 30s timeout:
- 30 seconds of degraded service per call
- 100 threads blocked → thread pool exhausted → system unresponsive

With a circuit breaker OPEN, each rejected call takes microseconds. The system stays responsive to other users with an explicit error, and the downstream has time to recover.

### Failure Counting Strategies

**Count-based**: Open after N absolute failures in the window.
**Rate-based (Hystrix/Resilience4j default)**: Open when failure *rate* > threshold (e.g., >50%) AND minimum call count is met. Avoids opening on 1 failure out of 2 calls (100% rate but tiny sample).

```
Window: last 10 calls
Failures: 7/10 = 70% → threshold 50% → OPEN
```

**Time-based rolling window**: Failures counted only within last N seconds, not last N calls. More intuitive under variable load.

---

## 3. Retry

Retrying a failed request when the failure is transient (network blip, momentary overload).

### What Is Safe to Retry

| Error type | Safe to retry | Reason |
|---|---|---|
| Network timeout | Yes (with idempotency) | Transient — server may not have received it |
| 503 Service Unavailable | Yes | Temporary overload |
| 429 Too Many Requests | Yes (after `Retry-After`) | Transient rate limit |
| 500 Internal Server Error | **No** by default | Server may have already processed it |
| 409 Conflict | No | Business logic conflict, not transient |
| 400 Bad Request | No | Client error — retrying won't help |

**Idempotency requirement**: Only retry operations that are safe to re-execute. `GET` is always safe. `POST /charge` is only safe if the endpoint supports idempotency keys.

### Retry with Exponential Backoff + Jitter

```python
# Exponential backoff
delay = base_delay * (2 ** attempt)   # 1s → 2s → 4s → 8s → 16s

# With full jitter (AWS recommendation) — spreads out retry storms
delay = random.uniform(0, min(cap, base_delay * 2 ** attempt))

# With decorrelated jitter (slightly better distribution)
delay = min(cap, random.uniform(base_delay, prev_delay * 3))
```

**Why jitter matters**: Without jitter, all clients that failed at the same moment retry at the same interval → thundering herd at `t + delay`. Jitter staggers them across time.

### Retry Budget

A retry amplifies load: 3 retries per call = 3× the load on a struggling service. Under saturation, retries worsen the situation.

**Retry budget**: Limit retries to a percentage of total calls (e.g., 10%). If the retry budget is exceeded, fail immediately rather than retry.

```
Service B is down: 1,000 req/sec to Service A
Each request retries 3×: 4,000 req/sec hitting Service B (already overloaded)
With retry budget capped at 10%: max 1,100 req/sec — much safer
```

---

## 4. Fallback

When a call fails (or circuit is open), execute an alternative behaviour rather than propagating an error.

### Fallback Strategies

| Strategy | Description | Example |
|---|---|---|
| **Cached response** | Return the last known good value from cache | Return stale product price |
| **Default value** | Return a safe hardcoded default | Empty recommendations list |
| **Degraded feature** | Skip the feature entirely | Hide "recommended for you" section |
| **Alternative service** | Call a secondary/replica service | Fall back to read replica |
| **Queue for later** | Accept the request and process asynchronously | Queue the payment for retry |
| **Fail fast with error** | Return a clear error message | "Service temporarily unavailable" |

### Fallback Design Principles

1. **Fallback must not call the same failing service** — a fallback that calls the same downstream just defers the failure
2. **Fallbacks can also fail** — wrap the fallback in its own resilience (Hystrix supports fallback-of-fallback)
3. **Silent degradation vs visible degradation** — sometimes it's better to show a clear "this feature is temporarily unavailable" than to show stale/wrong data silently

---

## 5. Bulkhead

Named after ship bulkheads: watertight compartments that limit flooding to one section.

In software: **isolate resource pools** so that failure in one section doesn't consume resources needed by others.

### Thread Pool Bulkhead

Each downstream dependency gets its own dedicated thread pool:

```
Service A thread pools:
  ├── Pool for Service B: 50 threads
  ├── Pool for Service C: 20 threads
  └── Pool for DB calls:  30 threads

Service B becomes slow → Pool B's 50 threads fill up → Pool B requests queued/rejected
Pool C and DB pool are unaffected → those features keep working
```

Without bulkhead: Service A has 100 total threads. Service B uses all 100. Service C requests fail because no threads available.

### Semaphore Bulkhead

Limits the number of *concurrent* calls to a dependency (not a thread pool — calls run on the caller's thread but are bounded by a semaphore):

```python
semaphore = asyncio.Semaphore(20)   # Max 20 concurrent calls to Service B

async with semaphore:
    result = await call_service_b()
# If 20 are already in flight, this raises immediately (or after optional timeout)
```

**Trade-off**: Thread pool bulkhead = better isolation but higher thread overhead. Semaphore bulkhead = lighter but not true isolation (same thread pool, just limited concurrency).

### Connection Pool Bulkhead

Each downstream service has its own DB/HTTP connection pool:

```python
# Separate connection pools per downstream
payment_pool = asyncpg.create_pool(max_size=10, dsn=PAYMENT_DB_URL)
inventory_pool = asyncpg.create_pool(max_size=5,  dsn=INVENTORY_DB_URL)
```

If payment DB connection pool saturates (max 10), inventory calls still have 5 connections available.

---

## 6. Timeout Strategies

Timeouts are the most fundamental resilience tool — every network call must have a timeout.

### Types of Timeouts

| Timeout type | What it bounds |
|---|---|
| **Connection timeout** | Time to establish the TCP connection |
| **Request timeout** | Time to send the full request |
| **Response timeout** | Time to receive the first byte of response |
| **Read timeout** | Time to receive the complete response body |
| **Overall / deadline** | Total time cap for the entire call chain |

### Deadline Propagation

In a microservice chain, each service has a timeout. But if the overall user-facing timeout is 1 second, and Service A calls B with a 5-second timeout, the user gets their response after 1 second, but Service B is still doing 5 seconds of wasted work.

**Solution — propagate deadlines** (gRPC does this natively):

```python
# Remaining time budget propagated in header
async def call_downstream(remaining_budget_ms: float):
    service_b_timeout = min(0.5, remaining_budget_ms / 1000 - 0.05)  # Reserve 50ms
    async with httpx.AsyncClient(timeout=service_b_timeout) as client:
        resp = await client.get("http://service-b/api")
```

gRPC: deadline is set per RPC and decremented across the call chain via `context.deadline`.

### Timeout Calibration

```
P99 latency of downstream = 200ms
→ Set timeout = 3–5× P99 = 600ms–1s
→ Do NOT set timeout = P50 (50% of calls would time out!)
→ Do NOT set timeout = infinity ("waits forever under load")
```

Use percentile-based calibration: the timeout should be above P99 of normal latency, well below the point at which holding threads becomes harmful.

---

## 7. How the Patterns Compose

A production call from Service A to Service B uses all four patterns:

```python
result = await resilience_pipeline(
    fn            = call_service_b,
    timeout       = 1.0,              # Timeout: fail if B doesn't respond in 1s
    retry         = RetryConfig(      # Retry: up to 3 times with backoff
        max_attempts=3,
        backoff=ExponentialBackoff(base=0.5, cap=10, jitter=True),
        retry_on=[503, 429],
    ),
    circuit_breaker = CircuitBreakerConfig(
        failure_threshold=50,         # Open if >50% failure rate
        rolling_window=60,
        min_calls=10,
        recovery_timeout=30,
    ),
    bulkhead      = BulkheadConfig(max_concurrent=20),
    fallback      = lambda: cached_response or default_value,
)
```

**Order of execution:**
1. Bulkhead: check capacity → reject if full
2. Circuit breaker: check state → fast-fail if OPEN
3. Timeout: wrap the call with a deadline
4. Execute the call
5. On failure: retry (if configured) with backoff
6. After max retries exhausted: invoke fallback

---

## 8. Real-World Tools

### Netflix Hystrix (Legacy — maintenance mode)

The original circuit breaker library for Java. Introduced the "bulkhead + circuit breaker + fallback" composition model. No longer actively developed but widely referenced in interviews as the conceptual origin.

Key contributions:
- Thread pool bulkhead per downstream command
- Metrics dashboard (Hystrix Dashboard) for real-time circuit state
- Command pattern: each downstream call is wrapped in a `HystrixCommand`

### Resilience4j (Current Java/Spring standard)

Hystrix's successor. Lightweight, modular, functional:

```java
// Circuit breaker
CircuitBreaker cb = CircuitBreaker.ofDefaults("paymentService");
Supplier<String> decoratedSupplier = CircuitBreaker.decorateSupplier(cb, this::callPayment);

// Retry
Retry retry = Retry.ofDefaults("paymentService");
decoratedSupplier = Retry.decorateSupplier(retry, decoratedSupplier);

// Bulkhead (semaphore-based)
Bulkhead bulkhead = Bulkhead.ofDefaults("paymentService");
decoratedSupplier = Bulkhead.decorateSupplier(bulkhead, decoratedSupplier);

// Compose and execute
Try<String> result = Try.ofSupplier(decoratedSupplier)
    .recover(throwable -> fallbackMethod(throwable));
```

Resilience4j exposes Micrometer metrics → Prometheus → Grafana dashboards.

### Microsoft Polly (.NET)

The .NET equivalent. Pipeline-based:

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddTimeout(TimeSpan.FromSeconds(1))
    .AddRetry(new RetryStrategyOptions {
        MaxRetryAttempts = 3,
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 10,
        BreakDuration = TimeSpan.FromSeconds(30),
    })
    .Build();
```

### Service Mesh Alternatives (Istio/Envoy)

Move resilience out of application code entirely into the sidecar proxy. Configured via `VirtualService` (retries, timeouts) and `DestinationRule` (circuit breaker) — see Topic 17 for details.

**Trade-off**: Mesh-level resilience is infrastructure policy (consistent, no language dependency), but can't implement business-logic fallbacks (returning stale data requires application knowledge).

---

## 9. Resilience + Observability

Patterns are only useful if you can see their state in production.

**Metrics to emit from circuit breaker:**

| Metric | Why it matters |
|---|---|
| `circuit_state{service="B"}` | CLOSED=0, OPEN=1, HALF_OPEN=2 — dashboard shows circuit health |
| `circuit_failure_rate{service="B"}` | Trending failure rate — leading indicator before opening |
| `retry_count{service="B", attempt="2"}` | High retry count → upstream is struggling |
| `fallback_invocations{service="B"}` | Frequent fallbacks → degraded mode — alert if threshold exceeded |
| `bulkhead_rejected{service="B"}` | Requests dropped by bulkhead — capacity issue |

**Alert thresholds:**
- Circuit OPEN for > 5 minutes → PagerDuty
- Fallback rate > 1% of traffic → warning
- Retry rate > 10% of traffic → warning (possible cascading failure incoming)
