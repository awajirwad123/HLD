# Observability — Architecture

## What Is Observability?

Observability is the ability to understand the **internal state** of a system by examining its **outputs** — without changing the system.

The three pillars:

| Pillar | What it answers | Primary tools |
|---|---|---|
| **Logs** | "What happened, and when?" | ELK, Loki, Splunk |
| **Metrics** | "How is the system performing right now?" | Prometheus, Datadog, CloudWatch |
| **Traces** | "Where did this request spend its time?" | Jaeger, Zipkin, OpenTelemetry |

---

## Pillar 1: Logging

### Structured vs Unstructured Logs

**Unstructured (bad):**
```
2024-01-15 10:23:45 ERROR Database connection failed for user 42
```

**Structured (good):**
```json
{
  "timestamp": "2024-01-15T10:23:45Z",
  "level": "ERROR",
  "service": "user-service",
  "event": "db_connection_failed",
  "user_id": 42,
  "db_host": "postgres-primary",
  "error": "connection refused",
  "trace_id": "abc123",
  "duration_ms": 5003
}
```

Structured logs are machine-parseable → searchable → aggregatable.

### Log Levels

| Level | When to use |
|---|---|
| DEBUG | Detailed diagnostics (disabled in production) |
| INFO | Normal operations ("user logged in") |
| WARNING | Unusual but recoverable ("retrying DB connection") |
| ERROR | Failed operation that was handled ("payment failed, sending to DLQ") |
| CRITICAL | System-wide failure requiring immediate attention |

### Log Aggregation Pipeline

```
App containers → log driver (stdout/stderr)
    ↓
Log shipper (Filebeat / Fluentd / Vector)
    ↓
Log storage + indexing (Elasticsearch / Loki / Splunk)
    ↓
Query + Dashboards (Kibana / Grafana / Splunk UI)
```

**Loki vs Elasticsearch:**
- Elasticsearch: full-text indexes log content → expensive storage, fast arbitrary search
- Loki: indexes only labels (service, pod, level) → cheap storage, fast label-filtered search, slower full-text

---

## Pillar 2: Metrics

### Types of Metrics

| Type | Description | Example |
|---|---|---|
| **Counter** | Monotonically increasing | `http_requests_total`, `errors_total` |
| **Gauge** | Current value (can go up/down) | `active_connections`, `memory_usage_bytes` |
| **Histogram** | Distribution of values with configurable buckets | `request_duration_seconds{le="0.5"}` |
| **Summary** | Quantiles computed client-side | P50, P95, P99 latency |

### Prometheus Architecture

```
Services → /metrics endpoint (pull model)
    ↑
Prometheus scrapes every 15s
    ↓
Prometheus TSDB (local storage, 15 days default)
    ↓
Grafana for dashboards
    ↓
Alertmanager for alerts → PagerDuty / Slack / OpsGenie
```

**Push vs Pull:** Prometheus uses pull (Prometheus scrapes targets). InfluxDB/StatsD use push (apps push metrics to collector). Pull advantage: Prometheus controls scrape rate, easier to detect dead targets.

### The Four Golden Signals (Google SRE)

| Signal | Definition | Example alert |
|---|---|---|
| **Latency** | Time to service a request | P99 latency > 500ms for 5 min |
| **Traffic** | Request rate | Requests/sec drops by 50% |
| **Errors** | Error rate | HTTP 5xx rate > 1% |
| **Saturation** | How full the system is | CPU > 80%, queue depth > 10K |

---

## Pillar 3: Distributed Tracing

### Why Tracing?

A single user request may touch 10+ microservices. A slow response is hard to debug with logs alone — which service was slow?

```
User request → API Gateway → Auth Service → User Service → DB
                                        ↘ Cache Service

Which step took 800ms out of total 1,000ms? Tracing shows exactly.
```

### How Tracing Works

**Trace:** The entire journey of one request across all services.
**Span:** One unit of work within a trace (e.g., "query DB" or "call User Service").

```
Trace ID: abc123 (unique per request)

Span 1: API Gateway        t=0ms    → t=1000ms  (total)
  Span 2: Auth Service     t=10ms   → t=50ms    (40ms)
  Span 3: User Service     t=60ms   → t=900ms   (840ms)  ← slow!
    Span 4: DB query        t=65ms   → t=860ms   (795ms)  ← root cause
  Span 5: Cache read       t=910ms  → t=920ms   (10ms)
```

### Context Propagation

To link spans across services, a trace context (trace_id + span_id) is propagated via HTTP headers:

```
W3C Trace Context (standard):
  traceparent: 00-{trace_id}-{span_id}-01
  tracestate: vendor-specific-data

OpenTelemetry uses W3C headers automatically.
```

### Sampling

Tracing every request = too much data + performance overhead. Solutions:

| Strategy | How | When |
|---|---|---|
| Head-based (fixed rate) | Sample 1% of traces randomly | General baseline |
| Tail-based | Record all, decide to keep after response time known | Capture all slow/error traces |
| Priority sampling | Always trace errors + high-value transactions | Critical paths |

---

## OpenTelemetry (OTel)

The unified observability framework — single SDK to emit logs, metrics, and traces.

```
App code → OpenTelemetry SDK → OTel Collector
                                     ↓
                    ┌────────────────┼────────────────┐
                    ↓                ↓                ↓
              Prometheus         Jaeger          Elasticsearch
              (metrics)         (traces)           (logs)
```

Auto-instrumentation: OTel can automatically instrument frameworks (FastAPI, requests, SQLAlchemy) without manual code changes.

---

## Alerting Strategy

### Alert Fatigue

Too many alerts → engineers ignore them. Principles:
- Alert on **symptoms**, not causes ("user-facing error rate > 1%" not "DB CPU > 70%")
- Every alert must be **actionable** and have a runbook
- Use different severity tiers: P1 (wake up on-call), P2 (next business day), P3 (informational)

### SLO-Based Alerting

SLO (Service Level Objective): "99.9% of requests complete in < 500ms"

Alert on **error budget burn rate** rather than raw metrics:
- Fast burn (> 14× normal): page immediately (burning 1 month's budget in 2 hours)
- Slow burn (> 2× normal): ticket alert (burning budget slowly, will exhaust in < 30 days)

---

## Key Numbers

| Metric | Value |
|---|---|
| Prometheus default scrape interval | 15 seconds |
| Typical trace sampling rate | 1–5% in production |
| Log retention (hot storage) | 7–30 days |
| Log retention (cold storage/archive) | 1–7 years |
| Trace retention | 7 days |
| P99 latency alert threshold (typical API) | 500ms |
| Error rate alert threshold (typical) | 1% over 5 minutes |
| Metrics cardinality limit (Prometheus) | < 10M series |
