# Observability — Tricky Interview Questions

## Q1: "Our Prometheus is running out of memory. How do you diagnose and fix it?"

**What interviewers probe:** Cardinality, label design, series count growth, federation, Thanos/Cortex for scale.

**Answer:**

First, **identify the source of cardinality explosion:**
```promql
# Find metrics with the most series
topk(10, count by (__name__)({__name__=~".+"}))

# Look for high-cardinality label values
count by (endpoint) (http_requests_total)
```

Common culprits:
- `user_id`, `session_id`, or `request_id` used as labels → unbounded
- URLs with IDs baked in: `/api/users/12345` as `endpoint` label
- Dynamic service names or pod IDs changing every deploy

**Fixes in order of impact:**
1. **Remove unbounded labels** — replace `user_id` label with logs/traces for per-user analysis
2. **Normalise URL patterns** — `/api/users/{id}` not `/api/users/12345`
3. **Increase retention chunking / reduce scrape interval** for high-series metrics
4. **Recording rules** — pre-aggregate in Prometheus, then drop raw metric (`metric_relabel_configs` with `action: drop`)
5. **Scale out** — Thanos / Cortex for horizontal sharding of Prometheus with a global query layer

**Interview trap:** Candidates often say "add more RAM". That's a short-term band-aid, not a fix. The real answer is label cardinality reduction.

---

## Q2: "Tail-based sampling sounds ideal. Why doesn't everyone use it?"

**What interviewers probe:** Trade-offs, stateless vs stateful collectors, buffering cost, head-based propagation constraint.

**Answer:**

Tail-based sampling is operationally expensive because the collector must:

1. **Buffer every span** until the trace is complete — meaning all spans across all services for a trace must route to the same collector node. This requires **affinity routing** (by trace_id), which breaks stateless fan-out.
2. **Hold spans in memory** for the full trace duration (could be seconds for slow operations). At high throughput (10K RPS × 50ms avg trace × 20 spans), the buffer can be large.
3. **Handle the head-based propagation constraint:** The sampling decision at the head is propagated via `traceparent` flags. With tail-based sampling, all spans still arrive, but the decision to emit is made later. If any service drops spans before they reach the tail collector, the decision is lost.

**Practical setup:**
- Use **Jaeger's adaptive sampling** or **OpenTelemetry Collector's tail sampling processor**
- Dedicate collector instances per trace_id hash partition (consistent routing)
- Accept the memory overhead as a cost of getting 100% error/slow-trace capture

**When to use head-based instead:** Any time your traffic is high enough that even 1% sampling gives sufficient cardinality for analysis, and errors are frequent enough to be captured in that 1%.

---

## Q3: "You are on call. A customer reports intermittent slowness. Where do you start?"

**What interviewers probe:** Systematic debugging, correlated signals, incident response.

**Answer (walk through the three pillars):**

**Step 1 — Metrics (what, not why):**
- Check the Four Golden Signals dashboard
- Is P99 elevated for that customer's region/endpoint?
- Is error rate up? Saturation climbing?
- Check "now vs 1 week ago" overlay to spot anomaly

**Step 2 — Traces (where):**
- Filter traces by customer ID, high-latency flag
- Find a slow trace — which span is taking most of the time?
- Is it consistent (always slow in same span) or scattered?

**Step 3 — Logs (why):**
- Filter logs by `trace_id` from the slow trace
- Look for WARN/ERROR events around the timestamp
- Check for retries, lock waits, GC pauses, connection pool exhaustion

**Checklist of common root causes to rule out:**
- N+1 queries (many small DB spans)
- Lock contention (one long DB span, others waiting)
- External API slowness (HTTP call span)
- Hot partition / noisy neighbour in shared cache
- Memory pressure causing GC pauses (correlate with JVM/GC metrics)

**Communication:** While investigating, update the incident channel every 15 min even if you have nothing new — silence creates panic.

---

## Q4: "How would you propagate trace context through a Kafka message queue?"

**What interviewers probe:** Async propagation, context loss, message enrichment, OTel messaging instrumentation.

**Answer:**

Kafka breaks the synchronous HTTP request/response model, so there is no automatic propagation. You must manually carry trace context in the Kafka message headers:

**Producer side (inject context):**
```python
from opentelemetry.propagate import inject

def produce_event(topic, payload):
    headers = {}
    inject(headers)  # Injects traceparent + tracestate into headers dict
    producer.produce(topic, value=payload, headers=list(headers.items()))
```

**Consumer side (extract context and create a new child span):**
```python
from opentelemetry.propagate import extract

def process_message(msg):
    headers = dict(msg.headers())
    ctx = extract(headers)           # Extracts parent trace context
    with tracer.start_as_current_span("kafka.consume", context=ctx):
        # all spans here are children of the producer's trace
        handle_payload(msg.value())
```

**Design decisions:**
- **Link vs parent:** For async processing (fan-out), use trace **links** (not parent-child) — one consumer message can be linked to the producer trace without making them the same trace.
- **Trace ends at Kafka:** For fire-and-forget events, accept that the trace is split. Add a `correlation_id` field to the log so you can join them manually.
- **OTel semantic conventions** define `messaging.*` span attributes: `messaging.system=kafka`, `messaging.destination=topic-name`, `messaging.message_id`.

---

## Q5: "How would you design observability for a 100-service microservices platform from scratch?"

**What interviewers probe:** System design, signal selection, cost vs coverage, operational maturity.

**Answer (structured pillars approach):**

**Phase 1 — Foundation (Day 1):**
- Structured JSON logging with trace_id and service name in every line
- Prometheus metrics on every service (at minimum: request rate, error rate, latency histogram — USE metrics)
- Centralized log aggregation (Loki or ELK based on budget)
- Basic alerting: P99 > 1s, error rate > 1%, service down

**Phase 2 — Tracing:**
- Deploy OTel Collector as DaemonSet (Kubernetes) — apps instrument with OTel SDK
- Head-based 5% sampling + tail-based 100% for errors
- Jaeger or Grafana Tempo as trace backend
- Service map auto-generated from span relationships

**Phase 3 — SLO-based alerting:**
- Define SLOs for every user-facing service (e.g., 99.9% of checkout requests < 500ms)
- Burn rate alerts replacing naive threshold alerts
- Error budget dashboards per team

**Infrastructure:**
```
Apps → OTel Collector (DaemonSet) → Prometheus (metrics via OTLP or scrape)
                                  → Tempo (traces)
                                  → Loki (logs via promtail/vector)
                                  → Grafana (unified dashboard + alerts)
```

**Cost control:**
- Separate long-retention cold storage (S3) from hot query tier
- Sample aggressively for healthy traffic, 100% for errors
- Label cardinality governance (CI check that rejects high-cardinality labels)

**Key principle:** Correlate the three pillars. A Grafana exemplar links a high-latency metric data point directly to the corresponding trace ID. That trace links to logs via trace_id. Full signal correlation = fast incident resolution.
