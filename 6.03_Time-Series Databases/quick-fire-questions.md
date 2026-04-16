# Time-Series Databases — Quick-Fire Questions

**Format:** Question → Answer. Use these to drill before an interview.

---

**Q1: Why can't you just store time-series data in PostgreSQL?**

At scale you can, but: PostgreSQL's B-tree index requires random writes for each insert, generating heavy write amplification. Append-only TSDBs exploit the monotonic nature of timestamps to seal blocks and compress sequentially. At 1 million data points/second a general-purpose RDBMS requires massive over-provisioning. Also, TSDBs auto-compress 10–20× and auto-expire old data; Postgres needs manual partitioning and custom logic for both.

---

**Q2: What is a "series" in Prometheus?**

A series is the unique combination of a metric name and a label set. Example: `http_requests_total{method="GET", endpoint="/api/users", status="200"}` is one series. `http_requests_total{method="POST", endpoint="/api/orders", status="201"}` is a different series. Cardinality = the total number of distinct series.

---

**Q3: What is high cardinality and why is it dangerous?**

High cardinality means you have a very large number of distinct series. Example: adding a `user_id` label to a metric when you have 10 million users creates 10 million series just for that one metric. Each series requires memory for its index entry and chunk head. Above ~1M active series, Prometheus starts OOMing and query performance degrades severely. The fix: never use high-cardinality values (user IDs, request IDs, session tokens) as labels.

---

**Q4: Explain delta-of-delta timestamp compression.**

Step 1 — Delta: instead of storing raw timestamps, store `t[n] - t[n-1]`. For 15-second scrapes, all deltas = 15000 ms.
Step 2 — Delta-of-delta: instead of storing each delta, store `delta[n] - delta[n-1]`. For perfectly regular scrapes, all deltas-of-deltas = 0.
Storing 0 repeatedly requires only 1 bit per timestamp after the first two. Result: regular-interval timestamp data compresses from 8 bytes to ~1 bit per point.

---

**Q5: How does Prometheus handle a Counter resetting to 0 after a process restart?**

`rate()` automatically detects counter resets. If `v[n] < v[n-1]`, Prometheus assumes a reset and adds `v[n-1]` to the running total before continuing. This is why you should always use `rate(counter[window])` and never do arithmetic on raw counter values directly.

---

**Q6: Histogram vs Summary — when do you choose each?**

Use **Histogram** when: you have multiple replicas (because histograms from different instances can be summed with `sum()` before calling `histogram_quantile()`), or when you don't know which quantiles you'll need at instrumentation time.

Use **Summary** when: you need highly accurate quantiles on a single-instance service and the quantiles are known at code-write time. Summaries cannot be aggregated across instances.

**Rule of thumb:** Almost always use Histogram.

---

**Q7: What does `rate()` compute vs `irate()`?**

`rate(metric[5m])` — computes the per-second rate averaged across the entire 5-minute window. Smooth, good for dashboards.

`irate(metric[5m])` — computes the per-second rate from the last two samples only. Highly responsive to spikes, but noisy. Good for alerting on sudden changes.

---

**Q8: Why should downsampled data store min AND max, not just the average?**

Because averaging destroys spike information. A 15-second CPU spike to 99% averaged into a 1-minute bucket becomes ~85%, which may not trigger alerts. Storing max=99 separately allows range queries to faithfully detect spikes even in downsampled tiers. The rule: **always downsample to (min, max, avg, sum, count)**.

---

**Q9: What is Thanos and when do you need it?**

Thanos extends Prometheus for long-term storage and multi-cluster federation. It adds:
- **Sidecar:** ships 2-hour Prometheus blocks to object storage (S3/GCS)
- **Store:**  serves historical data from object storage
- **Query:** fan-out query router that transparently merges live Prometheus data with historical object store data
- **Compactor:** downsamples historical data in object storage

You need it when: Prometheus's 15-day retention is insufficient, you have multiple Prometheus servers across clusters, or you need HA for Prometheus.

---

**Q10: What is the series cardinality formula?**

`total_series = product_of_distinct_label_values_per_metric`

Example: metric with labels `{method: 3, endpoint: 50, status_code: 10}` → 3 × 50 × 10 = 1,500 series.

Add `user_id` with 1M users → 3 × 50 × 10 × 1,000,000 = 1.5 billion series. That kills the TSDB.

---

**Q11: Explain InfluxDB's tags vs fields distinction.**

- **Tags:** string key-value pairs, **indexed** in an inverted index. Used in WHERE clauses. Form the series key. High cardinality tags are dangerous.
- **Fields:** typed values (int/float/string/bool), **NOT indexed**. Queries on fields require a full scan of all points in the time range. Use fields for measurement values that you aggregate but don't filter on.

Tags define WHAT you're measuring. Fields are the actual measured values.

---

**Q12: What is the InfluxDB Line Protocol?**

```
<measurement>[,tag_key=tag_val ...] field_key=field_val[,...] [unix_timestamp_ns]
```
Example: `cpu_usage,host=web-01,env=prod value=72.3,idle=27.7 1713175200000000000`

Commas separate tags, spaces separate the three sections (measurement+tags, fields, timestamp). Timestamp is optional (defaults to server time), in nanoseconds.

---

**Q13: What happens if you query without using the time index in a TSDB?**

You lose all TSDB benefits. Time-series queries that include a time range use the block/chunk metadata to skip irrelevant time windows in O(log n). Without a time filter you must scan all blocks. This is why TSDB GUIs and APIs always require a start/end time.

---

**Q14: What's the difference between `sum by` and `avg by` in PromQL?**

`sum by (endpoint) (rate(requests_total[5m]))` — totals the request rate across all instances *per endpoint*. Useful for throughput: if web-01 and web-02 each get 100 rps on /api, sum = 200 rps.

`avg by (endpoint) (rate(requests_total[5m]))` — averages across instances. Good for CPU usage where adding two instances' CPU percentages makes no sense; 75% + 80% shouldn't equal 155%.

---

**Q15: When does histogram_quantile return garbage results?**

When your bucket boundaries are wrong for your data distribution. If your P99 latency is 3 seconds but your highest histogram bucket is `le=1.0`, then `histogram_quantile(0.99, ...)` will return `+Inf`. Always include `+Inf` bucket (added automatically) and choose bucket boundaries based on expected value range. A common mistake: copying default `[0.005, 0.01, 0.025, ...]` latency buckets for a service that processes batch jobs taking 10–30 seconds.

---

**Q16: What is the "pull vs push" model in Prometheus and what breaks with pull?**

**Pull:** Prometheus scrapes each target's `/metrics` HTTP endpoint on a schedule. Targets are discovered via service discovery (k8s, consul, etc.)

**Push breaks when:**
- Target is a batch job that runs for 30 seconds and exits (no HTTP server)
- Target is behind NAT / firewall that Prometheus can't reach
- You're running serverless functions

**Solution for batch jobs:** Prometheus Pushgateway — jobs push to the gateway, Prometheus scrapes the gateway. Not for general use; it becomes a SPOF and loses per-instance tracking.

---

**Q17: In InfluxDB, what is the consequence of high series cardinality on the TSM engine?**

The series index (in-memory inverted index over tag values) must fit in RAM. High cardinality → index grows beyond available memory → disk spills → query performance degrades catastrophically. InfluxDB also opens a file handle per active series chunk during compaction; exceeding OS file descriptor limits causes write failures. Thresholds vary, but > 10M series on a single node is typically problematic.
