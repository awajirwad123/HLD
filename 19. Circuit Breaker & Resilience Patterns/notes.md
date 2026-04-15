# Circuit Breaker & Resilience Patterns — Notes & Cheat Sheet

## 1. Pattern Summary

| Pattern | Problem it solves | Key mechanism |
|---|---|---|
| **Circuit Breaker** | Cascading failure from slow/down dependency | Fast-fail when failure rate exceeds threshold |
| **Retry** | Transient network errors and momentary overload | Re-execute with exponential backoff + jitter |
| **Fallback** | Dependency unavailable; user shouldn't see raw error | Return cached/default value or degrade gracefully |
| **Bulkhead** | One slow dependency exhausting all shared resources | Isolate resource pools per downstream |
| **Timeout** | Threads blocked forever on unresponsive dependency | Hard deadline on every network call |

---

## 2. Circuit Breaker State Machine

```
    N failures / failure_rate exceeded
CLOSED ─────────────────────────────────► OPEN
  ▲    (all requests pass through)        (all requests fast-fail)
  │                                           │
  │  M consecutive successes                  │ recovery_timeout elapsed
  │                                           ▼
  └───────────────────────────── HALF_OPEN
                                 (1 probe request allowed)
```

**Config parameters:**

| Parameter | Typical value | Effect |
|---|---|---|
| `failure_threshold` | 50% rate or N absolute | Trigger to open |
| `rolling_window` | 60s | Window for rate calculation |
| `min_calls` | 10–20 | Prevents opening on 1/2 failures (100% rate but 2 calls) |
| `recovery_timeout` | 30–60s | Time in OPEN before probing |
| `success_threshold` | 3 | Successes in HALF_OPEN to close |

---

## 3. Retry — What to Retry and What Not To

| HTTP Status / Error | Retry? | Reason |
|---|---|---|
| Network timeout | ✓ (idempotent only) | Transient |
| 503 Service Unavailable | ✓ | Temporary overload |
| 429 Too Many Requests | ✓ after `Retry-After` | Transient rate limit |
| 502 Bad Gateway | ✓ | Transient proxy error |
| 500 Internal Server Error | ✗ by default | Server may have processed the request |
| 400 Bad Request | ✗ | Client bug — retry won't help |
| 409 Conflict | ✗ | Business conflict |
| 404 Not Found | ✗ | Resource doesn't exist |

**Always check idempotency before retrying mutating operations (POST, PUT).**

---

## 4. Backoff Formulas

```python
# Exponential backoff (no jitter — DO NOT USE alone)
delay = min(cap, base * 2 ** attempt)   # 0.5 → 1 → 2 → 4 → 8s

# Full jitter (AWS recommendation) — best for thundering herd prevention
delay = random.uniform(0, min(cap, base * 2 ** attempt))

# Equal jitter — balanced burst smoothing
v     = min(cap, base * 2 ** attempt)
delay = v / 2 + random.uniform(0, v / 2)

# Decorrelated jitter (slightly better statistical distribution)
delay = min(cap, random.uniform(base, prev_delay * 3))
```

**Rule**: always combine exponential backoff with jitter. Never retry at a fixed interval.

---

## 5. Bulkhead Comparison

| Type | Mechanism | Isolation | Overhead |
|---|---|---|---|
| **Thread pool** | Separate thread pool per downstream | True (diff threads) | Higher (thread context switch) |
| **Semaphore** | Semaphore limits concurrent calls | Partial (same thread pool) | Low |
| **Connection pool** | Separate DB/HTTP pool per downstream | True | Medium |

**Decision**: In async Python/Go: semaphore bulkhead is the right level. In Java with blocking I/O: thread pool bulkhead (as Hystrix used).

---

## 6. Timeout Calibration

```
P99 latency of downstream = 200ms
→ Timeout = 3–5× P99 = 600ms–1s

NEVER set timeout = P50  → rejects half of normal traffic
NEVER set timeout = ∞   → threads block forever under load
NEVER set timeout > user-facing SLA → user gets error anyway
```

**Types of timeouts to set on every HTTP client:**

| Timeout type | What it bounds |
|---|---|
| `connect_timeout` | TCP handshake |
| `write_timeout` | Sending request |
| `read_timeout` | Receiving response body |
| `pool_timeout` | Waiting for connection from pool |
| `overall` / deadline | Total budget for entire call |

---

## 7. Deadline Propagation Pattern

```python
# User-facing SLA: 1 second total
# Service A → B → DB: each gets a shrinking slice

deadline = now + 1.0s
→ Service A: uses 50ms, passes deadline - 50ms - overhead to Service B
→ Service B: uses 50ms, passes deadline - 100ms - overhead to DB
→ DB: has ~800ms — checks before starting long query: if remaining < 0, abort
```

**Why it matters**: Without propagation, a cancelled user request still causes downstream work (CPU, DB queries, money) to complete unnecessarily.

gRPC handles this natively via `context.deadline`. HTTP: pass `X-Deadline: <unix_ms>` or `X-Request-Timeout: <remaining_ms>` as a custom header.

---

## 8. Pattern Composition Order

```
Request arrives
  ↓
1. Bulkhead check    (are resources available?)
  ↓
2. Circuit breaker   (is downstream healthy?)
  ↓
3. Timeout wrapper   (set deadline)
  ↓
4. Execute call
  ↓
5. On failure: Retry (if retryable + budget available)
  ↓
6. On exhaustion: Fallback
```

**Why this order?**
- Bulkhead first: no point checking CB or setting a timeout if no threads/connections available
- CB before timeout: fast-fail with microsecond overhead is better than setting up a timer
- Retry inside CB: each retry attempt is counted by the CB; too many retries trip the circuit naturally

---

## 9. Resilience4j Quick Reference (Java)

```java
// Register via registry
CircuitBreakerRegistry cbRegistry = CircuitBreakerRegistry.ofDefaults();
CircuitBreaker cb = cbRegistry.circuitBreaker("payment");

// Decorate a call
Supplier<String> decorated = CircuitBreaker.decorateSupplier(cb, this::callPayment);
decorated = Retry.decorateSupplier(Retry.ofDefaults("payment"), decorated);
decorated = Bulkhead.decorateSupplier(Bulkhead.ofDefaults("payment"), decorated);

Try<String> result = Try.ofSupplier(decorated)
    .recover(ex -> "fallback");

// Metrics → Micrometer → Prometheus
io.github.resilience4j.micrometer.tagged.TaggedCircuitBreakerMetrics
    .ofCircuitBreakerRegistry(cbRegistry)
    .bindTo(meterRegistry);
```

---

## 10. Service Mesh vs Application-Level Resilience

| Concern | Service Mesh (Istio/Envoy) | Application (Resilience4j / Polly) |
|---|---|---|
| HTTP retries | ✓ `VirtualService.retries` | ✓ |
| Timeouts | ✓ `VirtualService.timeout` | ✓ |
| Circuit breaker | ✓ `DestinationRule.outlierDetection` | ✓ |
| Business fallback (stale cache) | ✗ | ✓ |
| Language-agnostic | ✓ | ✗ (per language) |
| Visibility in code | ✗ | ✓ |

**Practical answer**: Use mesh for transport-level retries/timeouts (consistency, zero code change). Use application-level for business-logic fallbacks and nuanced failure handling.

---

## 11. Key Numbers

| Metric | Value | Context |
|---|---|---|
| Timeout recommendation | 3–5× P99 latency | Calibration baseline |
| CB recovery timeout | 30–60s typical | Time in OPEN before HALF_OPEN |
| Retry jitter range | ±50% of backoff delay | Enough to stagger thundering herd |
| Retry max attempts | 3 (rarely 5) | More = amplification risk |
| Bulkhead semaphore limit | 10–50 concurrent | Depends on downstream capacity |
| Fast-fail overhead | < 1μs | vs 30s timeout wait |

---

## 12. Interview Key Phrases

- *"Without a circuit breaker, a single slow dependency exhausts my thread pool and takes down the entire service — cascading failure."*
- *"A circuit breaker fast-fails in microseconds when OPEN — I stay responsive while the downstream recovers."*
- *"Retries must use jitter to prevent thundering-herd re-storms. Without it, all clients retry at the same interval and re-overwhelm the recovering service."*
- *"Bulkhead isolates thread/connection pools: slow calls to Service B can't starve Service C calls."*
- *"Every timeout should be set at 3–5× P99 of the downstream. Never infinity, never P50."*
- *"Deadline propagation cancels downstream work when the user request is already cancelled — saves CPU and money."*
- *"Resilience4j/Polly for application-level fallbacks with business logic; Istio for transport-level retries across all services uniformly."*
