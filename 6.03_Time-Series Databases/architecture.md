# Time-Series Databases — Architecture & Concepts

## 1. What Makes Time-Series Data Different?

**Time-series data:** A sequence of data points indexed by timestamp, where time is the primary dimension.

```
(timestamp, metric_name, labels, value)
timestamp=2024-04-15T10:00:00Z, metric=cpu_usage, host=web-01, value=73.2
timestamp=2024-04-15T10:00:15Z, metric=cpu_usage, host=web-01, value=75.1
timestamp=2024-04-15T10:00:30Z, metric=cpu_usage, host=web-01, value=74.8
```

**Properties that distinguish it from relational data:**

| Property | Relational DB | Time-Series DB |
|---|---|---|
| Write pattern | Random inserts, updates, deletes | Append-only at current timestamp |
| Delete pattern | By arbitrary conditions | By time range (expire old data) |
| Query pattern | Point lookups, joins | Range scans over time windows |
| Primary index | Primary key | Timestamp + labels (tags) |
| Compression | Generic | Exploit temporal patterns (delta, XOR) |
| Cardinality issue | Not a concern | High-cardinality tags kill performance |

**Why PostgreSQL is wrong for metrics:**
```sql
-- 1 million metrics/second × 86,400 seconds = 86 billion rows/day
-- B-tree index maintenance on every write is O(log N) — prohibitive at this scale
-- Generic row storage: 8+ bytes overhead per row vs. delta-encoded 1–2 bytes per value
```

---

## 2. Time-Series Compression

Efficient storage is the core differentiator of TSDBs.

### Gorilla Compression (Facebook, 2015)

**Timestamp delta-of-delta encoding:**
```
Timestamps: 1000, 1060, 1120, 1180 (60-second intervals)
Deltas:           60,   60,   60
Delta of deltas:       0,    0      → usually 0 for regular intervals
```
Most timestamps have delta-of-delta = 0 → store as a single 1-bit flag (0 = same delta).

**Value XOR encoding (for float64):**
```
Value 1: 0 10000000101 0011001100110011001100110011001100110011001100110011
Value 2: 0 10000000101 0011010001110101110000101000111101011100001010001111
XOR:     0 00000000000 0000011101000110110011010111000000001000001110111100
```
Consecutive float values often differ only in low-order bits → XOR with previous value → many leading zeros → efficient variable-length encoding. Typical compression: **10–20× vs. raw float64 storage**.

### Prometheus Chunk Format
Each series stores chunks of 120 data points (~2 hours at 15s interval). Within a chunk:
- First timestamp stored as absolute value
- Subsequent timestamps: delta-of-delta encoded (variable bit-width)
- Values: XOR-encoded (Gorilla algorithm)

### InfluxDB TSM (Time-Structured Merge Tree)
Inspired by LSM trees. Writes go to WAL + in-memory cache; periodically flushed to immutable TSM files. TSM files use delta encoding + snappy/zstd compression per field. Reads: merge in-memory cache + TSM files (like Cassandra SSTable reads).

---

## 3. Core Time-Series Concepts

### Series (or Metric Stream)
A unique combination of metric name + label set:
```
cpu_usage{host="web-01", datacenter="us-east-1", env="prod"}
cpu_usage{host="web-02", datacenter="us-east-1", env="prod"}
```
Each unique label set = a distinct **series**. Two different series even for the same metric.

### Cardinality
Total number of distinct series. **High cardinality is the #1 TSDB performance killer.**
```
hosts = 10,000
metrics per host = 50
labels per metric = average 3 values

Total series = 10,000 × 50 × ... = potentially millions
```

**Cardinality explosion example:**
```
# Bad: using user_id as a label — millions of unique users = millions of series
http_requests{user_id="u_12345", endpoint="/api/v1/orders"}

# Good: user_id belongs in logs, not in metrics
http_requests{endpoint="/api/v1/orders", status="200"}
```

**Rule:** Never use unbounded values (user IDs, session IDs, request IDs, IP addresses) as metric labels.

### Retention Policy
Time-series data grows unboundedly — retention policies automatically expire old data:
- Prometheus: `--storage.tsdb.retention.time=15d` (default)
- InfluxDB: retention policies per bucket (measurement), e.g., `30d`
- Rule: recent data = full resolution; old data = downsampled

---

## 4. Downsampling

Reducing resolution of historical data to save storage while preserving trends.

```
Raw (15s resolution):
  10:00:00 → 73.2
  10:00:15 → 75.1
  10:00:30 → 74.8
  10:00:45 → 76.0
  ...

1-minute downsampled:
  10:00 → min=73.2, max=76.0, avg=74.775, sum=299.1, count=4

1-hour downsampled:
  10:00 → min=68.1, max=89.2, avg=77.3, ...
```

**Why store min/max/avg/sum/count (not just avg)?**
- `min`, `max`: preserve spikes — a 100ms CPU spike gets lost in a 1-minute average but preserved as `max`
- `sum` + `count` → recalculate `avg` correctly when further aggregating (can't average averages)
- `count`: for rate calculations

**InfluxDB continuous query (automatic downsampling):**
```sql
-- InfluxDB 2.x Flux
from(bucket: "raw_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage")
  |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
  |> to(bucket: "downsampled_1m")
```

**Tiered retention strategy:**
```
Tier 1: Raw (15s) → retain 7 days
Tier 2: 1-min aggregates → retain 30 days
Tier 3: 5-min aggregates → retain 90 days
Tier 4: 1-hour aggregates → retain 2 years
```

---

## 5. InfluxDB Architecture

**Data model:**
```
Measurement: cpu_usage
Tags (indexed, string only): host="web-01", env="prod"    ← filter/group by
Fields (values, not indexed): usage_percent=73.2, idle=26.8  ← what you measure
Timestamp: 2024-04-15T10:00:00Z

"Line protocol" format:
cpu_usage,host=web-01,env=prod usage_percent=73.2,idle=26.8 1713175200000000000
```

**Key InfluxDB concepts:**
- **Bucket:** Named data store with a retention policy
- **Measurement:** Similar to a table
- **Tag set:** Indexed metadata — used for filtering; stored in series key
- **Field set:** Data values — NOT indexed; read from TSM files

**TSM (Time-Structured Merge Tree) storage:**
```
Write → WAL (append-only, durability) + In-memory cache
Cache full → flush to TSM file (immutable, compressed)
Background: compact multiple TSM files → larger, more compressed files
Read: merge cache + TSM files in time order
```

**Query with Flux (InfluxDB 2.x):**
```python
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage" and r.host == "web-01")
  |> aggregateWindow(every: 1m, fn: mean)
  |> yield(name: "mean_cpu")
```

---

## 6. Prometheus Architecture

Prometheus is a **pull-based** monitoring system — it scrapes targets on a schedule.

```
Targets (services/exporters)
  ↑ scrape (HTTP GET /metrics every 15s)
Prometheus Server
  ├── TSDB (local storage, default 15-day retention)
  ├── Rules engine (recording rules, alerting rules)
  └── Query layer (PromQL)
  ↕
Alertmanager (route alerts → PagerDuty, Slack)
  ↕
Grafana (visualization, dashboards)
```

**Metrics format (exposition format):**
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET", endpoint="/api/users", status="200"} 1234
http_requests_total{method="POST", endpoint="/api/orders", status="201"} 567
http_requests_total{method="GET", endpoint="/api/users", status="404"} 12
```

**Metric types:**
| Type | Description | Use |
|---|---|---|
| **Counter** | Monotonically increasing, resets to 0 on restart | Request count, error count |
| **Gauge** | Can go up or down | CPU %, memory usage, active connections |
| **Histogram** | Sampled observations in configurable buckets | Request latency buckets |
| **Summary** | Quantiles calculated on client | P50, P99 latency (client-computed) |

**PromQL — Prometheus Query Language:**

```python
# HTTP error rate over last 5 minutes
rate(http_requests_total{status=~"5.."}[5m])

# P99 latency (from histogram)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# memory usage % by pod
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100

# Alert: CPU > 80% for 5 minutes
# (alerting rule)
# expr: avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) < 0.2
# for: 5m
# labels: severity: warning
```

**Prometheus storage — local TSDB:**
- WAL → memory-mapped chunks → block files (2-hour aligned blocks)
- After 2 hours: chunk compaction into block (efficient storage + query)
- After `retention.time`: blocks deleted

**Prometheus limitations and Thanos/Cortex:**
- Single-server: limited to local disk (typically 500GB–2TB)
- No long-term storage natively
- Thanos: sidecar ships blocks to S3; query layer federates across Prometheus instances
- Cortex/Mimir: horizontally scalable Prometheus-compatible cluster (multi-tenant)

---

## 7. Retention Policies in Practice

### InfluxDB
```python
# Create a bucket with 30-day retention
client.buckets_api().create_bucket(
    bucket=BucketRequest(
        name="application_metrics",
        retention_rules=[BucketRetentionRules(type="expire", every_seconds=30*86400)]
    )
)
```

### Prometheus
```bash
# Start Prometheus with 15-day retention
prometheus \
  --storage.tsdb.retention.time=15d \
  --storage.tsdb.retention.size=50GB   # also limit by disk size
```

### The Tiered Downsampling + Retention Recipe
```
1. Ingest: raw data → Tier-1 bucket (raw, 7d retention)
2. Continuous downsampling job: every hour
   → 1-min aggregates → Tier-2 bucket (30d retention)
   → 5-min aggregates → Tier-3 bucket (1y retention)
3. Query routing: choose tier based on time range requested
   - Last 24h: query Tier-1 (raw, full resolution)
   - Last 30 days: query Tier-2 (1-min)
   - Last year: query Tier-3 (5-min)
```

---

## 8. When to Use a TSDB vs. Other Stores

| Use Case | Best Store |
|---|---|
| Application performance monitoring | Prometheus + Grafana |
| IoT sensor data at massive scale | InfluxDB, TimescaleDB, or ClickHouse |
| Financial tick data | KDB+, QuestDB |
| Event analytics with SQL | TimescaleDB (PostgreSQL extension) |
| Log analytics (structured) | Elasticsearch, ClickHouse |
| Ad-hoc analytical queries on time-series | ClickHouse |
| Short-term operational metrics | Prometheus (default) |
| Long-term metrics with federation | Thanos, Cortex/Mimir, VictoriaMetrics |
