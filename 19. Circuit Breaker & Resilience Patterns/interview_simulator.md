# Circuit Breaker & Resilience Patterns — Interview Simulator

## Scenario 1: Resilient Payment Gateway

**Prompt:**
> "Your payment service calls three external dependencies: a fraud-detection API, a card-processing gateway, and a notification service. Design the resilience strategy for each call. The user expects synchronous confirmation at checkout."

**Time limit:** 30 minutes

---

### Clarifying Questions You Should Ask

1. What is the acceptable latency for the checkout response? (e.g., 2s P99)
2. Can fraud detection be async (pre-authorize, decide later) or must it block?
3. What is the SLA on each dependency? (fraud: 99.9%, card processor: 99.95%, notifications: 99.5%)
4. What happens if fraud check is unavailable — fail open (allow) or fail closed (deny)?

---

### Expected Architecture Walkthrough

**Dependency analysis:**

| Dependency | Criticality | Fallback available? | Latency target |
|---|---|---|---|
| Fraud Detection API | High | Fail-open (allow with flag) | 200ms |
| Card Processor | Critical (no fallback) | Queue for retry only | 1s |
| Notification Service | Low | Log and retry async | N/A |

**Resilience strategy per dependency:**

**Fraud Detection API:**
- Circuit breaker: `failure_threshold=40%`, `min_calls=10`, `recovery=20s`
- Timeout: 300ms (P99 latency historically 150ms → 2×P99)
- Retry: 1 retry on timeout only (not on 5xx — fraud check may have already processed)
- Fallback: fail-open with `fraud_status="unchecked"`, flag transaction for manual review
- Rationale: denying all transactions when fraud API is down costs more than the small fraud risk

**Card Processor:**
- Circuit breaker: `failure_threshold=20%` (much tighter — financial impact of errors is high), `min_calls=5`, `recovery=60s`
- Timeout: 3s (card processors are slow; P99 is 1.5s)
- Retry: NO automatic retry — card charge is not idempotent without explicit idempotency key. Client provides `Idempotency-Key`; retry is safe only if key is sent.
- Fallback: **409 — no silent fallback**. Return clear user-visible error "Payment temporarily unavailable. Please try again."
- Bulkhead: dedicated connection pool (max 20 concurrent) — isolated from fraud/notification pools

**Notification Service:**
- No circuit breaker needed on the hot path — call is fire-and-forget async
- Publish to Kafka `payment.completed` topic immediately after card charge succeeds
- Notification service consumes from Kafka — failure is isolated from checkout path
- DLQ for failed notification deliveries

**Thread structure at checkout:**
```
Checkout Request Timeline (2s budget)
  t=0ms    → Fraud check (parallel if possible, or sequential if required)
  t=~300ms → Card processor call (sequential after fraud pass)
  t=~1500ms→ Return success to user
  t=async  → Kafka publish → Notification service (after HTTP response returned)
```

---

### Key Decisions to Justify

**"Why fail-open on fraud detection?"**
99.9% uptime = 44 minutes downtime/month. Blocking all checkouts for 44 minutes/month loses more revenue than the fraud that slips through in that window. Flag unchecked transactions for async review post-payment. This is how Stripe handles it.

**"Why no auto-retry on card processor?"**
Without verifying idempotency key support end-to-end, retrying a charge risks a double charge. The correct approach: ensure the endpoint is idempotent (return same result for same idempotency key), then enable exactly 1 retry on network timeout only.

---

### Follow-up Questions

**"Fraud API is down for 20 minutes. 10,000 payments were flagged as 'unchecked'. How do you handle the backlog?"**
Kafka consumer reads `payment.completed` events with `fraud_status="unchecked"` and submits them to the fraud API asynchronously. Idempotent submission: use `payment_id` as idempotency key. Results update the `fraud_status` in the payments DB. High-risk transactions can be paused for fulfilment pending review.

**"What metrics do you expose for this system?"**
- `circuit_state{service="fraud_api"}` — CLOSED=0, OPEN=1, HALF_OPEN=2
- `fraud_unchecked_count` — current backlog of unreviewed transactions
- `payment_fallback_rate` — % of payments that hit a fallback path
- `cb_rejection_rate{service="card_processor"}` — alerts if card processor circuit opens

---

## Scenario 2: E-Commerce Order Placement Under Dependency Failures

**Prompt:**
> "Design the resilience for an order placement flow with 5 service calls: inventory check, price validation, cart service, order DB write, and post-order event publishing. Some failures should prevent the order; others shouldn't. Which patterns apply where?"

**Time limit:** 25 minutes

---

### Expected Architecture Walkthrough

**Step 1 — Classify each call by criticality:**

| Step | Blocks order? | Fallback | Pattern |
|---|---|---|---|
| Inventory check | Yes (can't sell unavailable) | No | Timeout + CB + no fallback |
| Price validation | Yes (can't confirm wrong price) | Stale cache (if fresh < 5min) | Timeout + CB + cache fallback |
| Cart service | No (cart already submitted) | Proceed without cart update | Timeout + CB + skip fallback |
| Order DB write | Yes (no order = nothing happened) | No (must succeed) | Retry (idempotent write) + timeout |
| Event publish (Kafka) | No (async) | Outbox pattern | Outbox → guaranteed eventual delivery |

**Step 2 — Design per call:**

**Inventory Check (critical, no fallback):**
```
Circuit Breaker: failure_rate > 50%, min_calls=10, recovery=30s
Timeout: 500ms
Retry: 2× on network timeout (inventory read is idempotent)
Fallback: NONE — return "Item temporarily unavailable" to user
```

**Price Validation (critical with cache fallback):**
```
Circuit Breaker: failure_rate > 50%, min_calls=10, recovery=30s
Timeout: 300ms
Retry: 1× on timeout
Fallback: Redis cache — use cached price if age < 5 minutes, reject if older
Cache key: price:{product_id}:{variant}  TTL: 10 minutes
```

**Cart Service (non-critical):**
```
No circuit breaker on critical path — call is best-effort
Timeout: 200ms
Retry: NONE (if cart update fails, order still proceeds)
Fallback: log warning + continue
Background job: reconcile cart state from order events
```

**Order DB Write (critical, idempotent):**
```
No circuit breaker needed — DB is your own infrastructure (not third-party)
But: connection pool bulkhead (max 30 connections shared for order writes)
Retry: 3× on transient DB errors (deadlock, timeout) — ON CONFLICT DO NOTHING for idempotency
Timeout: 2s
```

**Event Publish — Outbox Pattern:**
```
1. Write order + outbox entry in same DB transaction
   INSERT INTO orders ...
   INSERT INTO outbox (event_type, payload) VALUES ('order.created', {...})

2. Outbox poller (runs independently):
   SELECT * FROM outbox WHERE published_at IS NULL
   → Publish to Kafka
   → UPDATE outbox SET published_at = now()

3. Result: event delivery guaranteed even if process crashes between DB write and Kafka publish
```

---

### Follow-up Questions

**"Price cache is 4 minutes old. Inventory check succeeds. User places order. 30 seconds later, price was updated. What do you do?"**
This is an eventual consistency window. Options: (a) honour the cached price — the user confirmed at that price, the business eats the difference (Amazon's approach for small differences), or (b) hold the fulfilment, validate current price, send a "price changed" notification. For large discrepancies, option (b). Define a "material threshold" (e.g., >5% change → hold for review).

**"Order DB write fails after retry. User's card was already charged. What happens?"**
This is the worst-case partial failure. The circuit breaker doesn't help here — it's your own DB. Mitigation: write the DB and charge the card in the right order. Never charge before order is confirmed. If the card charge succeeded but DB write failed → immediately trigger a refund (compensating transaction) before returning an error to the user.

---

## Scenario 3: Designing for Resilience Across a Microservice Graph

**Prompt:**
> "You're a platform engineer. 30 microservices are using ad-hoc retry logic — some retry 5×, some don't retry, some don't have timeouts. You've seen two incidents where retries caused cascading failures. Design a company-wide resilience standard."

**Time limit:** 25 minutes

---

### Expected Architecture Walkthrough

**Problem analysis from incidents:**
- Service A retries 5× → amplification under partial failure of B
- Service C has no timeout → thread pool saturation under B slowness
- No circuit breakers → slow cascade not stopped; recovery impossible without manual intervention
- No observability → incidents found by users, not monitoring

**Proposed standard layer by layer:**

**Layer 1 — Istio (mesh-enforced baseline, zero code change):**
```yaml
# Default VirtualService applied to all services via namespace policy
retryPolicy:
  attempts:       2
  perTryTimeout:  2s
  retryOn:        "5xx,gateway-error,connect-failure,reset"
timeout: 10s   # Global fallback — per-service overrides allowed
```

**Layer 2 — DestinationRule (circuit breaker via outlier detection):**
```yaml
outlierDetection:
  consecutive5xxErrors:    5
  interval:               10s
  baseEjectionTime:       30s
  maxEjectionPercent:     50   # Max 50% of hosts ejected at once
```

**Layer 3 — Application-level (language-specific, optional override):**
- Business-logic fallbacks (stale cache, default values) — can't be done in mesh
- Idempotency-key-aware retry (don't count idempotent retries as new requests)
- Custom failure classification (retry 503 differently from 409)

**Enforcement standard:**
```
Non-negotiable:
  ✓ Every service call has a timeout (mesh enforced)
  ✓ Max 2 retries via mesh (teams cannot configure >3)
  ✓ Circuit breaker via outlier detection on all services

Team-configurable (documented in runbook):
  - Timeout value (default 10s; must justify if >10s)
  - CB sensitivity (default; override requires approval)
  - Application-level fallback logic
```

**Observability requirements:**
```
Mandatory metrics (Istio sidecar emits automatically):
  istio_requests_total{response_code, source, destination}
  istio_request_duration_milliseconds{quantile="0.99"}
  
Mandatory dashboards:
  - Circuit state per service
  - Retry rate per service (alert if >5%)
  - P99 latency heatmap across the service graph
  - Upstream error rate (alert if >1%)
```

**Rollout plan:**
1. Week 1–2: Enable mesh observability for all services (read-only, no policy enforcement)
2. Week 3–4: Apply timeout + retry policy to 5 non-critical services as pilot
3. Week 5–8: Rollout to all services, measure incident reduction
4. Week 9+: Enforce circuit breaker outlier detection

---

### Follow-up Questions

**"A team says 'our service calls an external legacy API with variable latency up to 30 seconds. We can't use a 10s timeout.' How do you handle this?"**
Two options: (a) isolate the legacy API call to an async worker — the service publishes a job to Kafka, the worker calls the legacy API and publishes results; main service path returns immediately with `status: processing`. (b) Allow a team-specific timeout override (30s) but mandate a dedicated connection pool bulkhead for this one call, so the 30s waits don't consume shared connections. Both options must be documented with a ticket justifying the deviation.

**"How do you retroactively test each service's resilience?"**
Chaos engineering: use Istio's fault injection (`HTTPFaultInjection`) to simulate failures in staging:
```yaml
fault:
  delay: {fixedDelay: 5s, percentage: {value: 50}}
  abort: {httpStatus: 503, percentage: {value: 10}}
```
Run the existing test suite with faults injected. A resilient service should pass or degrade gracefully. A non-resilient service will fail outright — revealing where fallbacks are missing.
