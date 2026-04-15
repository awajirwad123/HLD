# Observability — Quick-Fire Questions

**Q1: What are the three pillars of observability and what does each answer?**

Logs, metrics, and traces.
- **Logs** — What happened (discrete events with context)?
- **Metrics** — How much/how fast (aggregated numbers over time)?
- **Traces** — Where did the time go (request flow across services)?

They are complementary: metrics tell you *something* is wrong, traces tell you *where*, logs tell you *why*.

---

**Q2: What is the difference between observability and monitoring?**

Monitoring is checking known failure modes (you know what to ask). Observability is the ability to understand internal system state from external outputs — you can answer questions you have never anticipated. A highly observable system lets you debug novel failures with no prior knowledge of them.

---

**Q3: What is structured logging and why does it matter?**

Structured logging emits log records as key-value pairs (typically JSON) rather than free-form strings. This makes logs machine-parseable: you can filter, aggregate, and alert on specific fields. Example: `{"event": "payment_failed", "user_id": 123, "error": "card_declined"}` vs `"payment failed for user 123 – card declined"`. The structured form can be queried with `event=payment_failed AND error=card_declined` without regex.

---

**Q4: What are the Four Golden Signals? Who defined them?**

Defined by Google's SRE book:
1. **Latency** — How long requests take (P95/P99, not average)
2. **Traffic** — Request rate (RPS, QPS)
3. **Errors** — Rate of failed requests (5xx, application errors)
4. **Saturation** — How "full" the service is (CPU, memory, queue depth)

These four signals cover virtually all user-impacting issues.

---

**Q5: What is the difference between a Prometheus Histogram and Summary?**

- **Histogram** — Buckets are pre-declared on the server. Quantiles calculated at query time (`histogram_quantile`). **Can be aggregated** across instances with PromQL.
- **Summary** — Quantiles computed on the client. Pre-calculated, cannot be re-aggregated. Use only when you need exact quantiles for a single process.

In practice: always use Histogram for anything you want to aggregate across pods or regions.

---

**Q6: What is a trace_id and a span_id in distributed tracing?**

- **trace_id** — 128-bit ID shared by every span in a single request/transaction, across all services.
- **span_id** — 64-bit ID unique to one unit of work (one service/operation). Each span also records its parent's span_id, forming a tree.

The root span has no parent. The combination of trace_id + parent_id allows reconstruction of the full request flow.

---

**Q7: What is the difference between head-based and tail-based sampling?**

- **Head-based sampling** — Sampling decision is made at the beginning of the trace and propagated. Simple, low overhead. Problem: you might discard a rare but important slow/error trace.
- **Tail-based sampling** — All spans are buffered; the decision is made after the trace completes (e.g., keep all traces with errors or P99 latency). Captures high-value traces. Problem: requires buffering the full trace, higher memory/complexity.

Most production systems use head-based probabilistic sampling (1–5%) plus tail-based 100% capture for errors.

---

**Q8: What does OpenTelemetry (OTel) provide?**

OTel is a vendor-neutral CNCF project that provides:
1. A unified instrumentation SDK (traces, metrics, logs) for multiple languages
2. A collector/agent for receiving, processing, and exporting telemetry
3. Standardised propagation format (W3C TraceContext)

It decouples instrumentation from backends — you instrument once and can send to Jaeger, Prometheus, Datadog, or any compatible backend without code changes.

---

**Q9: What is high cardinality and why is it a problem in Prometheus?**

Cardinality = number of unique time series = number of unique label combinations. Prometheus stores each time series separately. Adding a label with many unique values (e.g., `user_id`, `request_id`) multiplies the series count by millions. This causes:
- Out-of-memory crashes (each series uses ~3–8 KB RAM)
- Slow queries
- Extremely high ingestion rate

Solution: never use unbounded values as Prometheus labels. Bucket them (status_code groups, endpoint groups) or move to logs/traces.

---

**Q10: What is an SLO, SLA, and SLI?**

- **SLI** (Service Level Indicator) — The metric you measure. E.g., "% of requests with latency < 200ms".
- **SLO** (Service Level Objective) — The internal target for the SLI. E.g., "99.9% of requests < 200ms over 30 days".
- **SLA** (Service Level Agreement) — External contractual commitment, with financial penalty for breach. E.g., "99.9% uptime SLA with 10% credit if breached".

SLO is what you aim for internally. SLA is what you promise customers.

---

**Q11: What is an error budget and how do burn rate alerts work?**

Error budget = time allowed to violate the SLO.
At 99.9% SLO: error budget = 0.1% × 30 days = ~43.8 minutes/month.

**Burn rate** = current error rate ÷ error budget rate.
- Burn rate 1× = you'll exactly exhaust the budget at month-end
- Burn rate 14× = you'll exhaust it in ~2 hours

Burn rate alerts fire when you're consuming the budget too fast. Typical: 14× burn over 1 hour = P1 page.

---

**Q12: What is a Prometheus pull model and how does it differ from push?**

- **Pull:** Prometheus server scrapes (HTTP GET `/metrics`) each target on a schedule. Prometheus is in control; targets don't need to know where Prometheus is.
- **Push:** Targets push metrics to a central aggregator (StatsD, InfluxDB push, Graphite). Good for short-lived jobs (use `pushgateway` as a bridge).

Pull model advantages: easy to detect dead targets (scrape fails), simpler service discovery, firewall-friendly (Prometheus initiates). Disadvantage: not suitable for batch jobs (use pushgateway instead).
