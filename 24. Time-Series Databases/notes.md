# Time-Series Databases — Notes & Reference

## What Makes Time-Series Different

| Property | Relational DB | TSDB |
|---|---|---|
| Write pattern | Random inserts/updates | Append-only at wall clock |
| Query pattern | Arbitrary joins/filters | Time-range scans |
| Update/delete | Common | Rare (bulk expiry) |
| Primary key | Any column(s) | (series_id, timestamp) |
| Compression | General | Temporal (DoD, XOR) |
| Retention | Manual | Automatic tiered |

**Core assumption:** Older timestamps are never inserted (mostly). Monotonically increasing time → blocks can be sealed, compressed, and tiered.

---

## Gorilla Compression (Facebook, 2015)

### Delta-of-Delta Timestamp Encoding

```
t0          → 64 bits (full Unix timestamp)
t1 − t0     → 14 bits (initial delta, usually scrape interval)
Δ(delta)    → VLE bits based on magnitude:
  '0'       → 1 bit  (delta unchanged, i.e., regular interval)
  '10'      → 7 bits  (delta changed by ≤ 63)
  '110'     → 9 bits  (delta changed by ≤ 255)
  '1110'    → 12 bits
  '11110'   → 32 bits (any value)
```

**Key insight:** Most scrapes/samples are at a fixed interval. Delta-of-delta = 0 → stored as a single `0` bit. Regular-interval data compresses to near nothing for timestamps.

### XOR Float Encoding

```
v0          → 64 bits (raw IEEE 754 double)
v1 XOR v0   → leading_zeros, trailing_zeros, meaningful_middle_bits
```

Similar consecutive values have large common prefix. The XOR is small → can be encoded with few bits. CPU metrics that hover around 75% → same upper bits → XOR is tiny.

**Overall result:** 10–20× compression versus raw binary.

---

## InfluxDB Reference

### Line Protocol Format

```
<measurement>[,<tag_key>=<tag_val>...] <field_key>=<field_val>[...] [<timestamp>]

cpu_usage,host=web-01,env=prod value=72.3,idle=27.7 1713175200000000000
           ──Tags (indexed)────  ───────Fields──────   ─Nanosecond Unix timestamp
```

**Rules:**
- Tags: strings only, indexed as B-tree, used in WHERE clauses
- Fields: int, float, string, bool — NOT indexed, must scan
- Timestamp: nanoseconds by default (can configure ms/s)
- Omitting timestamp → uses server ingestion time

### Tags vs Fields — Critical Distinction

| | Tags | Fields |
|---|---|---|
| Indexed | Yes (B-tree) | No |
| Data type | String only | int/float/string/bool |
| Cardinality impact | Each unique value creates a series | None |
| Use for | Host, region, env, service | Measurement values, counters |
| Query performance | Fast (index lookup) | Slow (scan) |

**High cardinality anti-pattern:**
```
# BAD: user_id as tag → millions of series
cpu_usage,user_id=u123456,host=web-01 value=72.3

# GOOD: user_id as field, aggregate by host
cpu_usage,host=web-01 value=72.3,active_users=1234
```

### InfluxDB TSM Storage Engine

```
Write path:
  Client → WAL (write-ahead log, immediate fsync) → In-memory Cache
                ↓ when cache full / time-based flush
  TSM Files (immutable, sorted, compressed blocks)
                ↓ background compaction
  Fewer, larger TSM files (Gorilla-compressed chunks)
```

### Flux Query Language

```flux
// Basic range + filter
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage" and r.host == "web-01")

// Aggregation window
from(bucket: "metrics")
  |> range(start: -24h)
  |> filter(fn: (r) => r._field == "value")
  |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)

// Multiple aggregations on same data
data = from(bucket: "metrics") |> range(start: -1h) |> filter(...)
union(tables: [data |> min(), data |> max(), data |> mean()])
```

---

## Prometheus Reference

### Metric Types

| Type | Description | Use When | Example |
|---|---|---|---|
| Counter | Monotonically increasing, resets to 0 on restart | Total requests, errors, bytes processed | `http_requests_total` |
| Gauge | Snapshot value, can go up or down | Current connections, memory usage, queue depth | `memory_usage_bytes` |
| Histogram | Samples → configurable buckets, exposes `_bucket`, `_count`, `_sum` | Latency, payload size (when percentiles needed) | `request_duration_seconds` |
| Summary | Client-side quantile computation | When quantile accuracy > label requirements | `rpc_duration_seconds` |

### Histogram vs Summary

| | Histogram | Summary |
|---|---|---|
| Percentile calculation | Server-side (PromQL) | Client-side (library) |
| Aggregatable across instances | Yes | No |
| Configurable at query time | Yes (any quantile) | No (fixed at instrument time) |
| Accurate for | Approximate | Exact |
| Use when | Multiple replicas, unknown quantiles | Single instance, known quantiles |

**Rule of thumb: Always prefer Histogram for new services.**

### Prometheus Exposition Format

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status_code="200"} 1234.0
http_requests_total{method="POST",endpoint="/api/orders",status_code="201"} 456.0
http_requests_total{method="GET",endpoint="/api/users",status_code="500"} 12.0
```

### PromQL Cheat Sheet

```promql
# Rate of counter (per second) over time window
rate(http_requests_total[5m])

# irate = instant rate (last 2 samples only — reactive, noisy)
irate(http_requests_total[5m])

# Use rate() for dashboards, irate() for alerts

# P99 latency from histogram
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)
# le = "less than or equal" — the histogram bucket label

# Aggregation functions
sum(rate(http_requests_total[5m])) by (endpoint)   # Sum across all instances
avg(memory_usage_bytes) by (host)
max(cpu_usage) without (cpu)                        # Drop 'cpu' dimension

# Ratio / percentage
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# Threshold alert
rate(http_requests_total[5m]) > 1000               # Alert condition
```

### Prometheus Architecture

```
scrape targets (HTTP /metrics)
    ↓ pull every 15s (default scrape_interval)
Prometheus server
    ├── TSDB (2-hour blocks → compaction)
    ├── Rule engine (recording + alerting rules)
    └── HTTP API (PromQL)
         ↓
Alertmanager → PagerDuty/Slack/email
         ↓
Grafana (dashboards)
```

**Retention:** Default 15 days (`--storage.tsdb.retention.time=15d`). Longer-term → Thanos/Cortex/Mimir.

---

## Downsampling Strategy

### Why Not Just Store Averages?

```
Raw data (1-min window): [85, 87, 91, 93, 97, 95, 88, 82]
If you store only avg=89.75:
  - You can't know the spike hit 97%
  - Downstream alert on "CPU > 95%" fires on raw but not on downsampled avg

Must store: min=82, max=97, avg=89.75, sum=718, count=8
```

### Standard Downsampling Tiers

| Tier | Resolution | Retention | Use |
|---|---|---|---|
| Raw | 15s (Prometheus default) | 7 days | Real-time dashboards |
| 1-minute | 1m aggregates | 30 days | Recent trends |
| 5-minute | 5m aggregates | 90 days | Weekly views |
| 1-hour | 1h aggregates | 2 years | Long-term capacity planning |

Each tier stores: **min, max, avg, sum, count** per window.

---

## Thanos / Cortex / Mimir (Long-Term Prometheus)

**Problem with standalone Prometheus:**
- Default retention: 15 days
- Single node: no HA, no horizontal scale
- No cross-datacenter federation

**Thanos architecture (sidecar model):**
```
Prometheus A  →  Thanos Sidecar  →  Object Store (S3/GCS)
Prometheus B  →  Thanos Sidecar  ↗
                                ↓
                 Thanos Query (fan-out query across everything)
```

**Cortex / Mimir:** microservices version, fully sharded ingestion, better at very high cardinality.

---

## TSDB Selection Guide

| Scenario | Best Fit | Why |
|---|---|---|
| Kubernetes / app metrics | Prometheus | Native pull model, PromQL, vast ecosystem |
| IoT / high-rate sensor data | InfluxDB | Line protocol, fast ingest, Flux queries |
| Need SQL / complex joins | TimescaleDB (PG extension) | Standard SQL, hypertable time partitioning |
| Analytics at massive scale | ClickHouse | Columnar, MergeTree, fast aggregations |
| Financial tick data | QuestDB or TimescaleDB | ASOF JOIN, microsecond precision |

---

## Key Numbers

| Fact | Value |
|---|---|
| Prometheus default scrape interval | 15 seconds |
| Prometheus TSDB 2-hour block | 120 samples per chunk |
| Gorilla compression ratio | 10–20× vs raw binary |
| Prometheus default retention | 15 days |
| InfluxDB WAL max size before flush | 25 MB |
| Histogram bucket count recommendation | 10–20 buckets |
| High cardinality threshold (practical) | > 1 million series = problem |
| P99 vs avg latency gap (rule of thumb) | P99 is often 3–10× the average |
