# Time-Series Databases — Tricky Interview Questions

## Q1: "We put user_id as a Prometheus label and now Prometheus is OOMing. How do you fix it without changing the client instrumentation code?"

**The trap:** The obvious fix is to change the client code to remove the `user_id` label. But sometimes you can't immediately redeploy every service.

**Better answer:**

First, diagnose: `count({__name__=~".+"}) by (__name__)` — find the metric driving cardinality explosion. A single metric with a user_id label multiplied by 10M users creates 10M series.

Short-term mitigations (no client change required):
1. **Metric relabeling in `prometheus.yml`** — drop the label before it enters the TSDB:
   ```yaml
   metric_relabel_configs:
     - source_labels: [user_id]
       action: labeldrop
   ```
   This drops the `user_id` label from scraped metrics so Prometheus stores aggregated series only.
2. **Vertical sharding:** Use `hashmod` relabeling to split high-cardinality metrics across multiple Prometheus instances.

Long-term fix: remove the label at the instrumentation level. Move `user_id` to a log or use aggregations (e.g., count per endpoint, not per user).

**The insight they're testing:** understanding of Prometheus's `metric_relabel_configs` as a safety valve, and knowing that cardinality problems are index problems — not disk problems.

---

## Q2: "Our P99 latency in Grafana shows 1.8s but users are reporting 5s+ response times. How do you investigate?"

**The trap:** "It's probably a Prometheus bug" or "increase retention." These are wrong.

**Walk through the causes:**

1. **Wrong histogram buckets.** If the highest bucket before `+Inf` is `le=2.0`, then `histogram_quantile(0.99,...)` for 5s responses returns somewhere between 2.0 and `+Inf`—usually it clamps or interpolates to 2.0. Check `histogram_quantile(1.0,...)` — if it returns exactly 2.0, you've hit bucket saturation.
   - Fix: add higher buckets (e.g., `3.0, 5.0, 10.0`) to the histogram definition.

2. **P99 across all endpoints, not the slow one.** `histogram_quantile` applied to a sum over all endpoints blurs the distribution. A fast endpoint at 99th percentile may dominate. Add `by (endpoint)` to isolate the slow path.

3. **Aggregation across replicas.**  `sum(rate(bucket[5m])) by (le)` is correct. `avg(rate(bucket[5m])) by (le)` is almost always wrong.

4. **Client clock skew.** If client timestamps drift, scrape intervals look irregular and `rate()` computes incorrect deltas.

5. **Measurement point location.** Check whether the Prometheus histogram measures server-side time (after load balancer), while users report end-to-end time including network RTT, DNS, TLS handshake, and response streaming.

**The insight they're testing:** histogram_quantile accuracy depends on bucket boundaries; and percentile metrics are only as meaningful as what part of the request cycle they cover.

---

## Q3: "Our on-call team gets paged at 3am for CPU alerts that clear by the time they look. What's wrong with this alert rule?"

```yaml
- alert: HighCPU
  expr: rate(cpu_seconds_total[1m]) > 0.9
  for: 0s
```

**Problems:**

1. **`for: 0s`** means the alert fires immediately on the first sample that crosses the threshold — a single noisy sample triggers a page. Real CPU spikes worth acting on usually persist for > 2–5 minutes.
   - Fix: `for: 5m`

2. **`rate(cpu_seconds_total[1m])`** — a 1-minute window is too short. `rate()` over a narrow window is extremely noisy (resembles `irate()`). A transient garbage collection pause or a single bursty request spikes this.
   - Fix: use `[5m]` or `avg_over_time(rate(cpu_seconds_total[5m])[10m:1m])` for a smoother signal.

3. **No aggregation across CPUs.** On a multi-core machine, Prometheus exports a `cpu` label per core. `rate(cpu_seconds_total{mode="user"}[5m])` per-core averages around 0.0–1.0 per core, not the overall system utilization. Use `avg by (instance) (rate(cpu_seconds_total{mode!="idle"}[5m]))`.

4. **Alert on saturation, not just usage.** High CPU that finishes quickly is not an incident. Alert only if it's sustained AND causing latency degradation (combine with a P99 latency alert).

**The insight:** `for:` duration is the most powerful alert quality knob. Almost every noisy alert is missing it.

---

## Q4: "We're downsampling metrics to reduce storage. After downsampling, our weekly capacity charts look wrong — the peaks are gone. Why, and how do you preserve them?"

**The problem:**

If you downsample using only `mean()`, spike data is lost. Example:
- Raw: 15-second samples — CPU hit 98% for 45 seconds (3 samples)
- 5-minute aggregation with mean: those 3 samples average down to, say, 72% in the 5-minute bucket
- Weekly chart queries the 5-minute tier → the 98% spike is invisible

**Solution:** Store the full **aggregation set** per window: `min, max, avg, sum, count`.

When querying the downsampled 5-minute tier:
- For "average CPU utilization" → use `avg`
- For "did CPU ever spike above X?" → use `max`
- For "total bytes sent" → use `sum`

**Thanos implementation:** Thanos's compactor runs a downsampling task that creates `5m` and `1h` resolution blocks. For each block it stores `chunk_type=raw`, `chunk_type=downsampled_5m`, etc., each with min/max/sum/count per series window.

**The insight they're testing:** downsampling is not just `mean()`. Engineers who haven't designed a downsampling pipeline always get this wrong.

---

## Q5: "We receive push metrics from 10,000 IoT devices via HTTP POST. Should we use Prometheus for this?"

**Short answer:** No, not directly. Prom is pull-based.

**Full answer:**

Prometheus's pull model requires the server to be able to reach each target's HTTP endpoint. For IoT devices behind NAT, cellular networks, or intermittent connectivity, this is impossible or impractical.

**Pushgateway workaround:** You could have all devices push to a central Pushgateway, and Prometheus scrapes the gateway. Problems:
- The gateway is a SPOF
- When a device disappears, its last-pushed metric stays in the gateway forever (stale issue)
- 10,000 devices pushing simultaneously → gateway throughput limits

**Better architecture for IoT push metrics:**
1. Devices push to a **message broker** (Kafka, MQTT, Kinesis)
2. A consumer reads from the broker and writes to **InfluxDB** or **TimescaleDB** (designed for push ingestion)
3. Grafana queries InfluxDB/Timescale directly

Use Prometheus for your **infrastructure monitoring** (the Kafka brokers, the consumer service, the InfluxDB nodes). Use InfluxDB for the IoT telemetry data itself.

---

## Q6: "We're choosing between Prometheus and InfluxDB for our SaaS product metrics. We need to retain 2 years of data at 1-second resolution. Which do you choose and why?"

**The trap:** "Just use Prometheus with a longer retention flag."

**The real answer:**

At 1-second resolution, 2 years = ~63 billion data points per metric. Prometheus TSDB is optimized for **operational metrics** (15s–1min resolution, 15–90 days retention). Extending to 2 years at 1-second resolution is possible but:

- You'd need massive disk (even with compression)
- Prometheus query performance degrades on very large time ranges
- The compaction/retention lifecycle isn't designed for this scale

**Better options:**

1. **InfluxDB + tiered downsampling:** Write raw 1s data, auto-downsample: keep 1s for 7 days, 1m for 90 days, 5m for 2 years. This collapses the storage problem dramatically.

2. **TimescaleDB** if you need SQL and complex joins (like correlating metrics with business events in the same query).

3. **ClickHouse** if you have the engineering bandwidth — MergeTree with `PARTITION BY toYYYYMM(timestamp)` plus `TTL` expressions per resolution tier.

**Thanos/Mimir on top of Prometheus** if you're married to PromQL — they handle long-term storage in S3 and support 2-year retention, but you still need to define downsampling policies.

---

## Q7: "We have a Prometheus alert that calculates error rate: errors / total. It fires spuriously on service startup. Why?"

**The problem:**

At startup, both `error_total` and `request_total` counters start at 0. During the first few seconds:
- `request_total` = 5 (health check requests)
- `error_total` = 1 (one startup error)
- Error rate = 1/5 = 20% → alert fires

**Solutions:**

1. **`for:` duration** — `for: 5m` means the condition must be true for 5 continuous minutes. Startup errors clear in seconds.

2. **Minimum volume guard:**
   ```promql
   sum(rate(error_total[5m])) / sum(rate(request_total[5m])) > 0.05
     and sum(rate(request_total[5m])) > 10
   ```
   The `and` clause suppresses the alert if total traffic is below 10 rps (i.e., during startup or very low-traffic periods).

3. **`absent()` check** — ensure the metric actually exists before computing a ratio.

**The insight they're testing:** alert expressions need guard clauses to handle the zero-denominator and low-traffic startup cases. Raw ratios without volume gates are the most common source of alert noise.
