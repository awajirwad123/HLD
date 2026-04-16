# Time-Series Databases — Interview Simulator

## Scenario 1: IoT Fleet Monitoring at Scale

**Interviewer prompt:** "You're joining a company that manages 2 million connected industrial machines — pumps, compressors, HVAC units. Each machine pushes temperature, vibration, pressure, and RPM readings every 5 seconds to a central API. The current architecture uses PostgreSQL and is falling over. Design a new time-series platform."

---

### Clarifying Questions You Should Ask

- What's the retention requirement? (Machines run 24/7 — probably 1–2 years with downsampling)
- Are queries real-time dashboards, batch analytics, or anomaly detection?
- Are queries per-machine, per-machine-type, or across the fleet?
- Do we need millisecond precision or is 5s fine?
- Are machines directly internet-connected or via gateways?

### Expected Answer Walkthrough

**Ingest path:**

2M machines × 4 fields × 1 point per 5 seconds = **1.6M writes/second**.

This is too fast for any HTTP-based TSDB to absorb directly. Use a message broker as a buffer:

```
Machines → MQTT broker (e.g., EMQX / HiveMQ)
              ↓
         Kafka (partition by machine_id % N)
              ↓
     Consumer fleet (Python/Go — writes in batch)
              ↓
         InfluxDB Cluster (or TimescaleDB distributed)
```

Why Kafka in the middle? Absorbs ingest spikes, decouples ingestion from TSDB write pressure, enables replay on TSDB failures, and allows multiple consumers (anomaly detection, alerting, analytics).

**Data model (InfluxDB Line Protocol):**

```
machine_telemetry,machine_id=m_123,type=pump,facility=plant-B
  temperature=72.3,vibration=0.05,pressure=14.7,rpm=1800
  1713175200000000000
```

Key design decision: `machine_id`, `type`, `facility` are tags (indexed, for WHERE filtering). Field values are the measurements.

**CARDINALITY CHECK:** 2M machines × 3 tag dimensions (machine_id, type, facility). Since `machine_id` alone has 2M values → 2M series minimum. At 4 fields per machine → 8M series total. This is high. Solutions:
1. Use `machine_id` as a tag (it IS your primary dimension — needed for per-machine dashboards)
2. Ensure `type` and `facility` have low cardinality (50 types × 200 facilities = 10,000 combinations, acceptable)
3. Monitor series count continuously

**Retention and downsampling:**

```
Raw (5s):   7 days    → ~2.5TB per day → 17.5TB for 7 days (with compression ~3TB)
1-minute:   90 days   → ~86× reduction from raw → 35GB
1-hour:     2 years   → ~720× reduction from 1m → <1GB
```

Implement with InfluxDB Tasks (scheduled Flux jobs):
```flux
option task = {name: "1-min downsample", every: 1m}
from(bucket: "telemetry_raw")
  |> range(start: -2m, stop: -1m)
  |> aggregateWindow(every: 1m, fn: mean)
  |> to(bucket: "telemetry_1m")
```
Run separate tasks for min, max, mean, count and union the results.

**Query routing:** Grafana variables control which bucket to query based on selected time range. Raw bucket for last 1 hour, 1-minute bucket for last 30 days, 1-hour bucket for longer.

**Follow-up: How do you detect anomalies across the fleet?**

Don't do this in the TSDB. Feed the Kafka stream to a stream processor (Flink or Spark Streaming) that runs a rolling z-score or isolation forest per machine-type. Write anomaly events to a separate metrics/events store (likely PostgreSQL with a small schema) and surface them in dashboards alongside the telemetry.

---

## Scenario 2: Kubernetes Infrastructure Observability

**Interviewer prompt:** "Your company has 50 Kubernetes clusters across 3 cloud regions. Engineers can't debug production incidents because metrics from different clusters are siloed in separate Prometheus instances. The ops team wants a single pane of glass with 1-year history, and SLA reporting across clusters. Design the observability platform."

---

### Clarifying Questions

- What volume of metrics? (Number of pods × metrics per pod × scrape interval)
- Is 1-year retention at full resolution or downsampled?
- Should this be deployed on-prem or in cloud?
- SLA reports — is that just uptime percentages or complex calculations?

### Expected Answer Walkthrough

**Problem statement:**

50 clusters × (say) 5,000 pods per cluster × 200 series per pod = **50M active series**. Prometheus default memory usage ≈ 2–3 bytes per sample × 50M series × 120 samples per chunk = several hundred GB just for the in-memory Series Head. That's not a single-Prometheus problem.

**Architecture: Thanos (or Mimir)**

```
[Cluster 1  Prometheus] ──→ Thanos Sidecar ──→ S3 (cluster-1-blocks)
[Cluster 2  Prometheus] ──→ Thanos Sidecar ──→ S3 (cluster-2-blocks)
   ...
[Cluster 50 Prometheus] ──→ Thanos Sidecar ──→ S3 (cluster-50-blocks)
                                                     ↓
                                          Thanos Store Gateway
                                               (serves S3 blocks)
                                                     ↓
                                          Thanos Query Frontend
                                               (caching, splitting)
                                                     ↓
                                          Thanos Querier
                                         (fan-out + dedup across all clusters)
                                                     ↓
                                              Grafana
```

**Key components:**

| Component | Role |
|---|---|
| Prometheus per cluster | Local TSDB, scrapes all pods, 2h–24h local retention |
| Thanos Sidecar | Uploads 2-hour blocks to S3, serves local data to Querier |
| Thanos Store Gateway | Translates S3 REST calls to remote_read API |
| Thanos Compactor | Downsamples old blocks (5m at 90 days, 1h at 1 year) |
| Thanos Query Frontend | Query splitting (long range → parallel sub-queries), result caching |
| Thanos Querier | Merges results from Store and live Prometheus sidecars |

**Deduplication:** When you run 2× Prometheus replicas per cluster for HA, Thanos deduplicates using `--query.replica-label=prometheus_replica`. Same data from both replicas is merged; missing samples from one fill from the other.

**1-year retention math:**

Thanos Compactor creates:
- Raw blocks: 7 days
- 5-minute downsampled: 90 days
- 1-hour downsampled: 1 year

Grafana's time picker automatically directs to the appropriate resolution via Thanos query routing.

**SLA reporting:**

Record Prometheus recording rules that pre-compute availability per service:
```yaml
- record: job:slo_availability:ratio_rate5m
  expr: |
    sum(rate(http_requests_total{status_code!~"5.."}[5m])) by (job)
      / sum(rate(http_requests_total[5m])) by (job)
```

These pre-computed metrics are themselves stored in Prometheus/Thanos, making 1-year SLA lookback queries fast (recorded series vs raw metric queries).

**Alerting:** Alertmanager per cluster, with routing to a central Alertmanager cluster that deduplicates and routes to PagerDuty/Slack.

---

## Scenario 3: Financial Trading Tick Data

**Interviewer prompt:** "We run an algorithmic trading platform. We need to store every trade tick — price, volume, order book snapshots — for every instrument on 50 exchanges. Each exchange generates ~5,000 events/second. We need sub-millisecond precision. We need to run backtesting queries over 10 years of tick data. What do you use?"

---

### Clarifying Questions

- Read/write ratio? (Backtesting = heavy read)
- Do we need SQL? (Join ticks with corporate actions, dividends?)
- Latency requirement for live queries vs batch backtests?
- Is this OLTP (live trading decisions) or OLAP (research/backtesting)?

### Expected Answer Walkthrough

**Volume calculation:**

50 exchanges × 5,000 events/sec = **250,000 writes/second**.
With order book depth (50 levels × 2 sides = 100 fields per snapshot) → ~25M field writes/second. This is extreme.

**Why Prometheus is wrong for this:** Designed for metrics aggregation, not tick storage. Doesn't support microsecond precision queries or complex financial joins.

**Why standard InfluxDB is limited:** Can handle ingest, but Flux query language is weak for financial math (ASOF JOINs, lead/lag, complex window functions).

**Primary database: QuestDB (or TimescaleDB)**

QuestDB's strengths for this use case:
- SIMD-accelerated sequential scan — full column scan in microseconds
- `ASOF JOIN` — join two time series where left ts ≤ right ts (essential for aligning tick data with reference data)
- SQL + time-series extensions
- Partitioned by day/month natively
- Sub-microsecond timestamp precision

Schema:
```sql
CREATE TABLE trades (
    ts          TIMESTAMP,
    exchange    SYMBOL,        -- SYMBOL type = low-cardinality string (indexed)
    symbol      SYMBOL,
    price       DOUBLE,
    volume      LONG,
    side        SYMBOL         -- 'BUY' | 'SELL'
) TIMESTAMP(ts) PARTITION BY DAY;

CREATE TABLE order_book_snapshots (
    ts          TIMESTAMP,
    exchange    SYMBOL,
    symbol      SYMBOL,
    bid_price   DOUBLE[50],    -- Array of 50 levels
    bid_size    LONG[50],
    ask_price   DOUBLE[50],
    ask_size    LONG[50]
) TIMESTAMP(ts) PARTITION BY DAY;
```

**Backtesting query example:**
```sql
-- Compute 1-minute VWAP (Volume Weighted Average Price)
SELECT
    ts,
    symbol,
    sum(price * volume) / sum(volume) AS vwap,
    sum(volume) AS total_vol
FROM trades
WHERE symbol = 'AAPL' AND ts BETWEEN '2023-01-01' AND '2023-12-31'
SAMPLE BY 1m;   -- QuestDB time aggregation syntax
```

**10-year data retention strategy:**

- Hot tier (last 90 days): NVMe SSD-backed QuestDB (or TimescaleDB chunks on NVMe)
- Warm tier (1–3 years): columnar Parquet files on object storage (S3), queryable via DuckDB or ClickHouse external tables
- Cold tier (3–10 years): compressed Parquet in S3 Glacier (access for regulatory review only)

**Data integrity:** Financial data has strict requirements. Each tick should have a `source_checksum` (exchange-provided sequence number). A validation job compares sequence numbers for gaps or duplicates. Immutable: ticks are never updated. WORM-style storage for regulatory compliance.

**ASOF JOIN example (aligning different frequencies):**
```sql
-- Align minute bars with daily reference data
SELECT t.ts, t.close_price, r.pe_ratio
FROM minute_bars t
ASOF JOIN daily_fundamentals r ON (t.symbol = r.symbol)
WHERE t.symbol = 'MSFT';
-- Returns each minute bar with the pe_ratio from the most recent daily snapshot ≤ the bar's timestamp
```
