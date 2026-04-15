# Observability — Reference Notes

## Three Pillars at a Glance

| Pillar | Purpose | Storage | Query |
|---|---|---|---|
| Logs | Events with context | Elasticsearch, Loki, CloudWatch | Full-text search, filters |
| Metrics | Aggregated numbers | Prometheus TSDB, InfluxDB | PromQL, range queries |
| Traces | Request flow across services | Jaeger, Zipkin, Tempo | Trace ID lookup, service map |

Each pillar is complementary — you need all three:
- **Alert fires** → go to metrics (what is broken?)
- **Metrics show spike** → go to traces (which requests are slow?)
- **Trace shows error** → go to logs (what exactly happened?)

---

## Metric Types

| Type | Description | Example | Can decrease? |
|---|---|---|---|
| Counter | Monotonically increasing | `http_requests_total` | No (reset to 0 on restart) |
| Gauge | Current snapshot value | `active_connections` | Yes |
| Histogram | Bucketed distribution | `request_duration_seconds` | — |
| Summary | Client-side quantiles | Request P50/P99 | — |

**Histogram vs Summary:**
- **Histogram** — buckets on server, quantiles calculated at query time via PromQL `histogram_quantile(0.99, ...)`. Aggregatable across instances.
- **Summary** — quantiles computed client-side. Cannot aggregate across instances. Use only when exact quantile accuracy matters for one instance.

> **Rule:** Prefer Histogram for anything aggregated across pods/nodes.

---

## PromQL Cheat Sheet

```promql
# Request rate over 5 minutes
rate(http_requests_total[5m])

# Error rate
rate(http_requests_total{status=~"5.."}[5m])

# Error ratio
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])

# P99 latency from histogram
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)

# CPU saturation
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)

# Memory usage %
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Throughput: requests per second
sum(rate(http_requests_total[1m]))
```

---

## Four Golden Signals

| Signal | Metric Example | Alert Threshold (typical) |
|---|---|---|
| **Latency** | P99 of request duration | > 500ms sustained for 5 min |
| **Traffic** | Requests per second | Sudden drop > 50% (possible outage) |
| **Errors** | 5xx rate | > 1% error ratio |
| **Saturation** | CPU, memory, queue depth | CPU > 80% sustained, queue > 10k |

> **Latency rule:** Always alert on percentile (P95/P99), not average. Averages hide tail latency.

---

## Structured Logging Guidelines

**Log levels and when to use them:**

| Level | When | Example |
|---|---|---|
| DEBUG | Detailed flow for local dev only | `"cache_hit: key=user:123"` |
| INFO | Normal operational events | `"order_placed: order_id=456"` |
| WARN | Unusual but handled | `"retry_attempt: n=3 allowed=5"` |
| ERROR | Failed operation, investigation needed | `"db_query_failed: table=orders"` |
| FATAL | Unrecoverable — service cannot continue | Crash with full context |

**Fields every structured log should include:**
```json
{
  "timestamp": "2024-01-15T14:23:00Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "a1b2c3d4",
  "span_id": "e5f6g7h8",
  "event": "payment_failed",
  "user_id": 12345,
  "amount": 49.99,
  "error": "card_declined",
  "error_code": "CARD_001"
}
```

**Anti-patterns:**
- `logger.error("Error: " + e.getMessage())` — no context
- Logging PII in plaintext (passwords, full card numbers, SSNs)
- Using string concatenation instead of structured fields
- Logging inside tight loops without sampling

---

## Distributed Tracing — W3C TraceContext

Standard HTTP headers for trace propagation:
```http
traceparent: 00-{trace-id}-{parent-span-id}-{flags}
tracestate:  vendor-specific-data
```

Example:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ↑  ↑                                ↑                ↑
             version  128-bit trace ID           64-bit span ID   sampled flag
```

**Trace propagation flow:**
```
Browser → API Gateway → Order Service → Inventory Service → DB
           (injects)      (extracts+       (extracts+
                           re-injects)      re-injects)
```

All services share the same `trace_id`. Each creates a new `span_id` and records its parent's `span_id`.

---

## Sampling Strategies

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Head-based** | Decision at trace start, propagated | Low overhead, consistent | Misses rare slow/error spans |
| **Tail-based** | Buffer, decide at end | Captures all errors and slow traces | Requires buffering entire trace |
| **Probabilistic** | X% of traces | Simple, predictable volume | May miss rare events |
| **Rate limiting** | N traces/second | Caps storage cost | Same sampling regardless of content |
| **Adaptive** | Adjust rate by endpoint/error | Best signal-to-noise | Complex to operate |

> **Typical production setup:** Head-based 1–5% for all traffic, plus tail-based 100% sampling for any trace with an error or latency > threshold.

---

## Alerting Strategy

**SLO-based alerting (burn rate alerts):**
```
Error budget = 1 - SLO target
For SLO = 99.9%: error budget = 0.1% = 43.8 minutes/month

Burn rate = current error rate / error budget rate
Burn rate 1x = consuming exactly your budget
Burn rate 14x = will exhaust monthly budget in ~2 hours
```

**Multi-window burn rate (recommended):**
| Window | Burn Rate | Severity |
|---|---|---|
| 1hr + 5min | 14× | Page (P1) |
| 6hr + 30min | 6× | Page (P2) |
| 3d + 6hr | 1× | Ticket (P3) |

**Symptom-based vs cause-based alerts:**
- Alert on symptoms (user impact): high error rate, slow P99, payment failures
- Do NOT alert on causes unless they're guaranteed to lead to user impact
- Example: "CPU at 90%" is NOT an alert. "P99 > 2s for 5min" is.

---

## OpenTelemetry (OTel) Architecture

```
Your Application
    ↓ (OTel SDK — vendor-neutral instrumentation)
OTel Collector  ──────→  Prometheus (metrics)
    │            ──────→  Jaeger / Tempo (traces)
    └────────────→  Loki / Elasticsearch (logs)
```

**OTel SDK key concepts:**
- **Tracer:** Creates and manages spans
- **Meter:** Creates metric instruments (Counter/Gauge/Histogram)
- **Logger:** Emits log records (bridges to existing loggers)
- **Propagator:** Injects/extracts context from HTTP headers, queues

**Signal correlation in OTel:**
OTel allows linking traces and logs via shared `trace_id`, and traces and metrics via exemplars (a trace ID embedded in a Prometheus metric sample).

---

## Log Aggregation Pipeline

```
Apps (structured JSON) → Filebeat/Fluentd (collect + forward)
                          → Kafka (buffer, high throughput)
                          → Logstash/Vector (parse, enrich, route)
                          → Elasticsearch (index + store)
                          → Kibana (search + dashboard)
                               OR
                          → Grafana Loki (log labels, cheaper)
```

**Loki vs Elasticsearch trade-off:**
| | Elasticsearch | Loki |
|---|---|---|
| Index | Full-text index every field | Only labels (no full-text index) |
| Query | Fast full-text search | Faster by label filter, then grep |
| Cost | High (index storage) | Low (compressed blob) |
| Use when | Need regex/full-text search | Labels-based filtering is enough |

---

## Key Numbers

| Metric | Value |
|---|---|
| Prometheus scrape interval | 15s (default) |
| Typical trace sampling rate prod | 1–5% |
| P99 alert threshold (web APIs) | 500ms–1s |
| OTel collector memory (light load) | ~256MB |
| Elasticsearch shard size (recommended) | 10–50 GB |
| Logs retention (compliance) | 90 days–7 years |
| Prometheus TSDB retention (default) | 15 days |
| Max recommended Prometheus cardinality | ~10M active series |
| Error budget @ 99.9% SLO | 43.8 min/month |
