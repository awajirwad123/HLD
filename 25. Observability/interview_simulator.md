# Observability — Interview Simulator

## Scenario 1: Design Observability for a Payment Platform

**Prompt:** "We're building a payment processing platform handling 50K transactions/second at peak. Design the observability system. I want to know when payments fail, when they're slow, and be able to debug any incident within 5 minutes."

---

**Clarifying Questions to Ask:**
- Number of microservices? (helps size the collector fleet)
- Cloud platform or on-prem? (affects managed vs self-hosted tooling)
- Compliance requirements? (PCI-DSS may limit what you can log — no card numbers in logs)
- Target MTTR? ("5 minutes" is the stated goal — we'll design to it)

---

**Step 1 — Define the Critical Signals**

Payment platform Four Golden Signals:
- **Latency:** P99 of `payment_authorisation_duration_seconds`
- **Traffic:** `transactions_per_second` by payment_method
- **Errors:** `payment_failures_total{reason="declined|timeout|fraud"}` / total
- **Saturation:** DB connection pool utilisation, queue depth, card network API rate limit remaining

Business-level SLOs:
- 99.95% of payment authorisations complete within 3 seconds
- < 0.1% transaction error rate (excluding user-caused errors)

---

**Step 2 — Logging**

**Rules for payment logs:**
- Structured JSON with: `trace_id`, `service`, `event`, `payment_id`, `merchant_id`, `amount`, `currency`, `status`
- **Never log:** full card number, CVV, raw PIN — PCI-DSS prohibits it
- Log `bin` (first 6 digits of card) for debugging card network routing
- Retention: 90 days hot (Elasticsearch), 7 years cold (S3 + Glacier) for compliance

**Log pipeline:**
```
Services → Filebeat → Kafka (buffer 24h) → Logstash (redact PII) → Elasticsearch
```
Redaction in Logstash before indexing: mask `card_number` field matching digit patterns.

---

**Step 3 — Metrics**

Key Prometheus metrics:
```
# Payment success/failure by reason
payment_attempts_total{method, status, reason}

# Card network API call latency
card_network_request_duration_seconds{network, endpoint}

# Fraud check latency
fraud_check_duration_seconds{result}

# Queue depth (async processing)
payment_queue_depth{priority}
```

**Cardinality rules:**
- `payment_id` — never a label (unbounded). Use logs/traces for per-payment analysis.
- `merchant_id` — label only if < 10K active merchants. Otherwise aggregate.

---

**Step 4 — Distributed Tracing**

Every payment spans multiple services:
```
API Gateway → Payment Service → Fraud Service → Card Network API
                             → Ledger Service
                             → Notification Service
```

Trace design:
- Root span: `payment.authorise` with `payment_id` as a **tag** (not label)
- Child spans: `fraud.check`, `card_network.charge`, `ledger.debit`, `user.notify`
- Sampling: 10% of successful payments, 100% of failures and slow (>1s) payments
- Tail sampling in OTel Collector: keep all traces with `payment.status=failed`

---

**Step 5 — Alerting**

```yaml
# P1 - Page immediately
- alert: PaymentFailureRateCritical
  expr: rate(payment_attempts_total{status="failed"}[5m]) /
        rate(payment_attempts_total[5m]) > 0.01
  for: 2m
  severity: page

# P1 - Latency SLO burn
- alert: PaymentLatencyBudgetBurn
  expr: histogram_quantile(0.99, rate(payment_duration_seconds_bucket[5m])) > 3
  for: 5m
  severity: page

# P2 - Card network degradation
- alert: CardNetworkHighLatency
  expr: histogram_quantile(0.95, rate(card_network_request_duration_seconds_bucket[10m])) > 1
  for: 10m
  severity: slack
```

---

**Trade-offs:**
- **Managed vs self-hosted:** Datadog/New Relic is faster to set up; Prometheus + Grafana + Tempo is cheaper at scale but more ops burden
- **Sampling:** 100% tracing at 50K TPS = ~5M spans/sec. Too expensive — use tail-based with error capture
- **Log retention:** Full 7-year compliance retention requires tiered storage. Hot tier (90 days) for fast queries; cold tier (Glacier) for audit

---

## Scenario 2: Debug a P99 Latency Spike in Production

**Prompt:** "Your P99 latency for the checkout API just jumped from 200ms to 2.5 seconds. You have 10 minutes. Walk me through your debugging process."

---

**Minute 0–2: Scope the impact**
```promql
# What changed?
increase(http_request_duration_seconds_bucket{endpoint="/checkout", le="0.5"}[10m])
  - increase(http_request_duration_seconds_bucket{endpoint="/checkout", le="0.5"}[10m] offset 30m)

# Is it all regions or one?
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{endpoint="/checkout"}[5m])) by (region)

# All endpoints or just checkout?
top5 histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) by (endpoint)
```

Answers: "Only /checkout, only EU-west-1 region."

**Minute 2–4: Trace analysis**
- Filter traces: service=checkout, region=eu-west-1, duration > 1s, last 5 min
- Look at the waterfall: which span is slow?
- Find: `payment.card_network_call` is consistently 2.2s. All other spans normal.

**Minute 4–6: Cross-reference metrics**
```promql
histogram_quantile(0.99, rate(card_network_request_duration_seconds_bucket{network="VISA", region="eu-west-1"}[5m]))
```
Result: VISA European gateway API P99 = 2.4s. Normal = 150ms. VISA has a regional incident.

**Minute 6–8: Logs for confirmation**
```
service=payment-service region=eu-west-1 event=card_network_call
→ {"event": "card_network_timeout", "network": "VISA", "timeout_ms": 2000, "count_last_5min": 847}
```

**Minute 8–10: Mitigation**
- Enable fallback: route VISA EU transactions via global gateway (slightly higher cost)
- Update status page: "Degraded checkout performance in EU-West"
- Set incident resolved timer: check VISA status page, re-evaluate in 20 min

**Post-incident:** Add runbook link in alert, add card-network-specific SLO with independent budget.

---

## Scenario 3: Set Up Observability for a New Service

**Prompt:** "You're launching a new recommendation service. It calls ML model inference, reads from Redis, writes results to PostgreSQL. What observability will you add before go-live?"

---

**Minimum viable observability (must-haves before launch):**

**1. Four USE Metrics per resource:**
```python
# Redis
redis_commands_total{command, status}
redis_command_duration_seconds{command}  # Histogram

# PostgreSQL
pg_query_duration_seconds{query_type}  # Histogram
pg_pool_connections{state}             # Gauge (active/idle/waiting)

# ML model inference
ml_inference_duration_seconds{model_version}  # Histogram
ml_inference_errors_total{error_type}
```

**2. Request-level instrumentation:**
```python
# OTel SDK auto-instrumentation for FastAPI
# Traces every incoming request + outgoing HTTP/DB calls automatically
app = FastAPI()
FastAPIInstrumentor.instrument_app(app)
RedisInstrumentor().instrument()
```

**3. Business metric:**
```python
# Did we actually serve a recommendation, or fallback?
recommendations_served_total{type="personalised|fallback|cached"}
```

**4. Pre-launch dashboard** (4 panels):
- Request rate + error rate (dual axis)
- P50/P95/P99 latency over time
- Redis hit ratio + miss rate
- ML inference latency per model version + error count

**5. SLO and alerts:**
- SLO: 99.5% of recommendation requests < 100ms
- Alert: P99 > 200ms for 5 min → Slack
- Alert: error rate > 2% for 2 min → Page
- Alert: Redis hit ratio < 70% → Slack (cache warming issue)

**Launch checklist:**
- [ ] Structured logging with trace_id in all log lines
- [ ] `/health` and `/ready` endpoints returning status
- [ ] Prometheus metrics endpoint at `/metrics`
- [ ] OTel SDK initialised pointing to collector
- [ ] Dashboard deployed and reviewed with team
- [ ] Runbook linked in alert annotations
