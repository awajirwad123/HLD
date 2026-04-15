# Time-Series Databases — Hands-On Exercises

## Exercise 1: Build a Mini Time-Series Store in Python

Understand TSDBs from first principles — write, compress, query, downsample:

```python
"""
Goal: Implement a minimal TSDB supporting:
  - Append-only writes (timestamp, tags, value)
  - Delta-of-delta timestamp compression
  - Time-range queries
  - In-memory downsampling (min/max/avg/count per window)
"""

import time
import math
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Optional


# ─── Data structures ──────────────────────────────────────────────────────────

@dataclass
class DataPoint:
    timestamp: int     # Unix milliseconds
    value: float


@dataclass
class Series:
    """A single time series: unique by (metric_name, tags_frozen)."""
    metric: str
    tags: dict[str, str]
    points: list[DataPoint] = field(default_factory=list)

    def append(self, timestamp: int, value: float):
        if self.points and timestamp <= self.points[-1].timestamp:
            raise ValueError(f"Out-of-order write: {timestamp} ≤ {self.points[-1].timestamp}")
        self.points.append(DataPoint(timestamp, value))

    def range_query(self, start_ms: int, end_ms: int) -> list[DataPoint]:
        """Binary search for start, linear scan to end."""
        import bisect
        times = [p.timestamp for p in self.points]
        lo = bisect.bisect_left(times, start_ms)
        hi = bisect.bisect_right(times, end_ms)
        return self.points[lo:hi]

    def downsample(self, start_ms: int, end_ms: int,
                   window_ms: int) -> list[dict]:
        """
        Aggregate points into fixed-size windows.
        Returns: [{timestamp, min, max, avg, sum, count}]
        """
        points = self.range_query(start_ms, end_ms)
        if not points:
            return []

        windows: dict[int, list[float]] = defaultdict(list)
        for p in points:
            bucket = (p.timestamp // window_ms) * window_ms
            windows[bucket].append(p.value)

        result = []
        for bucket_start in sorted(windows):
            vals = windows[bucket_start]
            result.append({
                "timestamp": bucket_start,
                "min": min(vals),
                "max": max(vals),
                "avg": sum(vals) / len(vals),
                "sum": sum(vals),
                "count": len(vals),
            })
        return result


class MiniTSDB:
    """
    In-memory TSDB.
    Series identified by (metric, frozen tags dict).
    """

    def __init__(self):
        self._series: dict[tuple, Series] = {}

    def _series_key(self, metric: str, tags: dict[str, str]) -> tuple:
        return (metric, tuple(sorted(tags.items())))

    def write(self, metric: str, tags: dict[str, str],
              timestamp_ms: int, value: float) -> None:
        key = self._series_key(metric, tags)
        if key not in self._series:
            self._series[key] = Series(metric=metric, tags=tags)
        self._series[key].append(timestamp_ms, value)

    def query(self, metric: str, tags: dict[str, str],
              start_ms: int, end_ms: int) -> list[DataPoint]:
        key = self._series_key(metric, tags)
        series = self._series.get(key)
        if not series:
            return []
        return series.range_query(start_ms, end_ms)

    def downsample(self, metric: str, tags: dict[str, str],
                   start_ms: int, end_ms: int, window_ms: int) -> list[dict]:
        key = self._series_key(metric, tags)
        series = self._series.get(key)
        if not series:
            return []
        return series.downsample(start_ms, end_ms, window_ms)

    def cardinality(self) -> int:
        return len(self._series)


# ─── Delta-of-Delta compression demonstration ─────────────────────────────────

def demo_delta_compression():
    """Show how timestamps compress with delta-of-delta encoding."""
    import struct

    # Simulate 1 hour of 15-second interval data
    base_ts = 1713175200000  # ms
    interval = 15_000        # 15 seconds in ms
    timestamps = [base_ts + i * interval for i in range(240)]  # 240 points = 1 hour

    # Raw size: 8 bytes × 240 = 1,920 bytes
    raw_size = len(timestamps) * 8

    # Delta: store only differences
    deltas = [timestamps[0]] + [timestamps[i] - timestamps[i-1] for i in range(1, len(timestamps))]
    # All deltas are 15000 — could store as one value + count

    # Delta-of-delta
    dod = [timestamps[0], deltas[1]] + [deltas[i] - deltas[i-1] for i in range(2, len(deltas))]
    non_zero_dod = sum(1 for x in dod[2:] if x != 0)
    # Regular intervals → all delta-of-deltas = 0 → can represent each with 1 bit

    # Compressed: first 2 timestamps (16 bytes) + 238 × 1 bit = ~46 bytes
    compressed_size = 16 + math.ceil(238 / 8)

    print("=== Timestamp Delta-of-Delta Compression ===")
    print(f"  Raw timestamps:       {raw_size} bytes")
    print(f"  First 5 delta-of-deltas: {dod[:7]}")
    print(f"  Non-zero delta-of-deltas: {non_zero_dod} of {len(dod)-2}")
    print(f"  Compressed size:      ~{compressed_size} bytes")
    print(f"  Compression ratio:    {raw_size / compressed_size:.0f}×")


# ─── XOR float compression demonstration ──────────────────────────────────────

def demo_xor_compression():
    """Show Gorilla XOR encoding for floats."""
    import struct

    values = [73.2, 73.5, 74.1, 73.9, 74.3, 74.8, 75.1, 74.6, 74.0, 73.8]

    def float_to_bits(f: float) -> int:
        return struct.unpack(">Q", struct.pack(">d", f))[0]

    prev_bits = None
    xor_values = []
    for v in values:
        bits = float_to_bits(v)
        if prev_bits is not None:
            xor = bits ^ prev_bits
            leading_zeros = (64 - xor.bit_length()) if xor else 64
            xor_values.append((xor, leading_zeros))
        prev_bits = bits

    print("\n=== XOR Float Compression (Gorilla) ===")
    print(f"  Values: {values}")
    print(f"  {'Value':>8} {'XOR (hex)':>20} {'Leading zeros':>15}")
    for (xor, lz), v in zip(xor_values, values[1:]):
        print(f"  {v:>8.1f} {hex(xor):>20} {lz:>15}")
    print(f"  Avg leading zeros: {sum(lz for _, lz in xor_values) / len(xor_values):.1f}")
    print(f"  Fewer bits needed → smaller stored representation")


# ─── Full demo ─────────────────────────────────────────────────────────────────

def demo_tsdb():
    db = MiniTSDB()

    base_ms = 1713175200000
    tags_web01 = {"host": "web-01", "env": "prod"}
    tags_web02 = {"host": "web-02", "env": "prod"}

    # Write 1 hour of synthetic CPU data for 2 hosts
    import random
    random.seed(42)

    for i in range(240):   # 240 × 15s = 1 hour
        ts = base_ms + i * 15_000
        db.write("cpu_usage", tags_web01, ts, 70 + random.gauss(0, 5))
        db.write("cpu_usage", tags_web02, ts, 60 + random.gauss(0, 3))

    print(f"\n=== TSDB Demo ===")
    print(f"Total series in memory: {db.cardinality()}")

    # Query last 5 minutes
    start = base_ms + 55 * 60_000
    end   = base_ms + 60 * 60_000
    pts = db.query("cpu_usage", tags_web01, start, end)
    print(f"\nweb-01 CPU last 5 min ({len(pts)} points):")
    for p in pts[:5]:
        print(f"  {p.timestamp} → {p.value:.2f}%")

    # Downsample to 5-minute buckets
    windows = db.downsample("cpu_usage", tags_web01, base_ms, base_ms + 3600_000, 5 * 60_000)
    print(f"\nweb-01 CPU 5-min downsampled ({len(windows)} windows):")
    for w in windows[:4]:
        print(f"  {w['timestamp']}: avg={w['avg']:.1f} min={w['min']:.1f} max={w['max']:.1f}")


demo_tsdb()
demo_delta_compression()
demo_xor_compression()
```

---

## Exercise 2: Prometheus Metrics with Python

Instrument a FastAPI service and write PromQL queries for it:

```python
"""
Goal: Instrument a FastAPI service with all 4 Prometheus metric types:
  Counter, Gauge, Histogram, Summary.
  Then write PromQL expressions to query them.
"""

from fastapi import FastAPI, Request, Response
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    generate_latest, CONTENT_TYPE_LATEST,
    CollectorRegistry, REGISTRY,
)
import time, asyncio, random

app = FastAPI()

# ─── Metric definitions ───────────────────────────────────────────────────────

# Counter: total requests by method, path, status
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

# Gauge: currently active connections (up and down)
ACTIVE_CONNECTIONS = Gauge(
    "http_active_connections",
    "Number of active HTTP connections",
)

# Histogram: request latency with configurable buckets
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
)

# Gauge: queue depth (for background workers)
QUEUE_DEPTH = Gauge(
    "background_queue_depth",
    "Number of tasks in the background processing queue",
    ["queue_name"],
)

# Summary: client-side quantile calculation (useful for small volumes)
PAYMENT_LATENCY = Summary(
    "payment_processing_duration_seconds",
    "Payment processing duration",
)


# ─── Instrumentation middleware ───────────────────────────────────────────────

@app.middleware("http")
async def instrumentation_middleware(request: Request, call_next):
    ACTIVE_CONNECTIONS.inc()
    start = time.perf_counter()

    try:
        response = await call_next(request)
        duration = time.perf_counter() - start

        # Record metrics
        REQUEST_COUNT.labels(
            method=request.method,
            endpoint=request.url.path,
            status_code=str(response.status_code),
        ).inc()

        REQUEST_LATENCY.labels(
            method=request.method,
            endpoint=request.url.path,
        ).observe(duration)

        return response
    finally:
        ACTIVE_CONNECTIONS.dec()


# ─── Metrics endpoint (scraped by Prometheus) ─────────────────────────────────

@app.get("/metrics")
async def metrics():
    return Response(generate_latest(REGISTRY), media_type=CONTENT_TYPE_LATEST)


# ─── Application endpoints ────────────────────────────────────────────────────

@app.get("/api/users")
async def list_users():
    await asyncio.sleep(random.uniform(0.01, 0.05))  # Simulate DB
    return {"users": []}


@app.post("/api/payments")
async def create_payment():
    with PAYMENT_LATENCY.time():
        await asyncio.sleep(random.uniform(0.05, 0.5))  # Simulate payment processor
    return {"payment_id": "pay_123", "status": "captured"}


@app.get("/api/slow")
async def slow_endpoint():
    await asyncio.sleep(random.uniform(1.0, 3.0))   # Intentionally slow
    return {"ok": True}


# Background job simulation
async def background_worker():
    while True:
        QUEUE_DEPTH.labels(queue_name="email").set(random.randint(0, 50))
        QUEUE_DEPTH.labels(queue_name="export").set(random.randint(0, 10))
        await asyncio.sleep(5)
```

**PromQL queries to write against this service:**

```promql
# ─── Rate and throughput ───────────────────────────────────────────────────────

# Request rate (requests per second) over last 5 minutes
rate(http_requests_total[5m])

# Per-endpoint request rate
rate(http_requests_total[5m]) by (endpoint)

# HTTP 5xx error rate
rate(http_requests_total{status_code=~"5.."}[5m])

# HTTP error percentage
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# ─── Latency ───────────────────────────────────────────────────────────────────

# P50 / P95 / P99 latency using histogram
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)

# Average latency per endpoint
rate(http_request_duration_seconds_sum[5m])
  / rate(http_request_duration_seconds_count[5m])

# ─── Saturation ────────────────────────────────────────────────────────────────

# Active connections gauge
http_active_connections

# Queue depth trend
http_active_connections > 100   # Alert: active connections > 100

# Background queue depth
background_queue_depth{queue_name="email"}

# ─── Alerting rules (prometheus rules file) ────────────────────────────────────
# groups:
# - name: app_alerts
#   rules:
#   - alert: HighErrorRate
#     expr: |
#       sum(rate(http_requests_total{status_code=~"5.."}[5m]))
#       / sum(rate(http_requests_total[5m])) > 0.05
#     for: 2m
#     labels:
#       severity: critical
#     annotations:
#       summary: "HTTP 5xx error rate > 5% for 2 minutes"
#
#   - alert: HighP99Latency
#     expr: |
#       histogram_quantile(0.99,
#         sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
#       ) > 2.0
#     for: 5m
#     labels:
#       severity: warning
```

---

## Exercise 3: InfluxDB Write and Query with Python

```python
"""
Goal: Write IoT sensor metrics to InfluxDB 2.x using the Python client.
      Then query with Flux including downsampling.
"""

from influxdb_client import InfluxDBClient, Point, WritePrecision
from influxdb_client.client.write_api import SYNCHRONOUS
import random
import time
from datetime import datetime, timezone, timedelta


INFLUX_URL   = "http://localhost:8086"
INFLUX_TOKEN = "my-super-secret-token"
INFLUX_ORG   = "my-org"
INFLUX_BUCKET = "iot_sensors"


client = InfluxDBClient(url=INFLUX_URL, token=INFLUX_TOKEN, org=INFLUX_ORG)
write_api = client.write_api(write_options=SYNCHRONOUS)
query_api = client.query_api()


# ─── Write sensor data ────────────────────────────────────────────────────────

def write_sensor_data(num_points: int = 500):
    """Write synthetic temperature/humidity readings for 3 sensors."""
    sensors = ["sensor-a", "sensor-b", "sensor-c"]
    locations = {"sensor-a": "office", "sensor-b": "server-room", "sensor-c": "warehouse"}

    base_time = datetime.now(timezone.utc) - timedelta(hours=2)
    points = []

    for i in range(num_points):
        ts = base_time + timedelta(seconds=i * 15)  # 15-second intervals
        for sensor_id in sensors:
            loc = locations[sensor_id]
            # Simulate different baselines per sensor
            base_temp = {"sensor-a": 22, "sensor-b": 28, "sensor-c": 18}[sensor_id]
            base_hum  = {"sensor-a": 45, "sensor-b": 60, "sensor-c": 70}[sensor_id]

            p = (
                Point("environment")
                .tag("sensor_id", sensor_id)      # Tags: indexed, filterable
                .tag("location", loc)
                .field("temperature", round(base_temp + random.gauss(0, 0.5), 2))
                .field("humidity", round(base_hum + random.gauss(0, 2), 2))
                .time(ts, WritePrecision.SECONDS)
            )
            points.append(p)

        if len(points) >= 300:   # Batch writes
            write_api.write(bucket=INFLUX_BUCKET, record=points)
            points = []
            print(f"  Written {i*3} points...")

    if points:
        write_api.write(bucket=INFLUX_BUCKET, record=points)

    print(f"Wrote {num_points * 3} data points")


# ─── Query with Flux ──────────────────────────────────────────────────────────

def query_raw(sensor_id: str, minutes: int = 30):
    """Query raw data for a specific sensor."""
    query = f"""
    from(bucket: "{INFLUX_BUCKET}")
      |> range(start: -{minutes}m)
      |> filter(fn: (r) => r._measurement == "environment")
      |> filter(fn: (r) => r.sensor_id == "{sensor_id}")
      |> filter(fn: (r) => r._field == "temperature")
      |> sort(columns: ["_time"])
    """
    result = query_api.query(query)
    points = [(r.get_time(), r.get_value()) for table in result for r in table.records]
    print(f"\n=== Raw temperature for {sensor_id} (last {minutes} min) ===")
    for ts, val in points[-5:]:
        print(f"  {ts}: {val}°C")
    return points


def query_downsampled(window: str = "5m"):
    """Downsample all sensors to window aggregates."""
    query = f"""
    from(bucket: "{INFLUX_BUCKET}")
      |> range(start: -2h)
      |> filter(fn: (r) => r._measurement == "environment"
                         and r._field == "temperature")
      |> aggregateWindow(
           every: {window},
           fn: (tables=<-, column) => tables
             |> mean(),
           createEmpty: false
         )
      |> group(columns: ["sensor_id"])
    """
    # Note: For min/max/avg simultaneously, use multiple queries or task API
    result = query_api.query(query)
    print(f"\n=== Temperature {window} averages per sensor (last 2h) ===")
    for table in result:
        sensor = table.records[0].values.get("sensor_id", "?")
        recent = table.records[-3:]
        for r in recent:
            print(f"  {sensor} @ {r.get_time()}: avg={r.get_value():.2f}°C")


def query_max_temperature_alert(threshold: float = 30.0):
    """Find time windows where server-room temperature exceeded threshold."""
    query = f"""
    from(bucket: "{INFLUX_BUCKET}")
      |> range(start: -2h)
      |> filter(fn: (r) => r._measurement == "environment"
                         and r.location == "server-room"
                         and r._field == "temperature")
      |> filter(fn: (r) => r._value > {threshold})
      |> sort(columns: ["_time"])
    """
    result = query_api.query(query)
    alerts = [(r.get_time(), r.get_value()) for table in result for r in table.records]
    if alerts:
        print(f"\n⚠ Server room temperature exceeded {threshold}°C {len(alerts)} times:")
        for ts, val in alerts[:5]:
            print(f"  {ts}: {val:.2f}°C")
    else:
        print(f"\n✓ Server room temperature stayed below {threshold}°C")


# ─── Run ──────────────────────────────────────────────────────────────────────
print("Writing sensor data...")
write_sensor_data(500)

query_raw("sensor-b", minutes=30)
query_downsampled("5m")
query_max_temperature_alert(30.0)

client.close()
```

---

## Exercise 4: Downsampling Pipeline

Implement a downsampling job that reduces raw data to multiple tiers:

```python
"""
Goal: Simulate a tiered downsampling pipeline:
  Tier 1: Raw 15s data
  Tier 2: 1-minute aggregates (min/max/avg/count)
  Tier 3: 5-minute aggregates
  Query router: select appropriate tier based on requested time range
"""

import time
from datetime import datetime, timezone
from typing import Literal


class TieredTSDB:
    """Multi-resolution TSDB with automatic tier selection for queries."""

    def __init__(self):
        # Each tier: list of (timestamp_ms, min, max, avg, count)
        self.raw: list[tuple] = []           # 15s, last 7 days
        self.tier_1m: list[tuple] = []       # 1-min, last 30 days
        self.tier_5m: list[tuple] = []       # 5-min, last 1 year

    def write(self, ts_ms: int, value: float):
        self.raw.append((ts_ms, value, value, value, 1))

    def downsample_to_1m(self):
        """Aggregate raw points into 1-minute buckets."""
        buckets = {}
        for ts_ms, mn, mx, avg, cnt in self.raw:
            bucket = (ts_ms // 60_000) * 60_000
            if bucket not in buckets:
                buckets[bucket] = []
            buckets[bucket].append(avg)   # avg of raw = the raw value itself

        for bucket_ms, vals in sorted(buckets.items()):
            n = len(vals)
            self.tier_1m.append((
                bucket_ms,
                min(vals), max(vals), sum(vals) / n, n
            ))
        print(f"Downsampled {len(self.raw)} raw points → {len(self.tier_1m)} 1-min buckets")

    def downsample_to_5m(self):
        """Aggregate 1-minute buckets into 5-minute buckets."""
        buckets = {}
        for ts_ms, mn, mx, avg, cnt in self.tier_1m:
            bucket = (ts_ms // 300_000) * 300_000
            if bucket not in buckets:
                buckets[bucket] = {"min": float("inf"), "max": float("-inf"),
                                   "sum": 0, "count": 0}
            b = buckets[bucket]
            b["min"] = min(b["min"], mn)
            b["max"] = max(b["max"], mx)
            b["sum"] += avg * cnt    # Weighted sum to correctly recompute avg
            b["count"] += cnt

        for bucket_ms, b in sorted(buckets.items()):
            self.tier_5m.append((
                bucket_ms,
                b["min"], b["max"], b["sum"] / b["count"], b["count"]
            ))
        print(f"Downsampled {len(self.tier_1m)} 1-min points → {len(self.tier_5m)} 5-min buckets")

    def query(self, start_ms: int, end_ms: int,
              force_tier: str = None) -> tuple[list[tuple], str]:
        """
        Automatically select the appropriate resolution tier.
        Returns (points, tier_name).
        """
        duration_ms = end_ms - start_ms
        duration_hours = duration_ms / 3_600_000

        # Tier selection based on query range
        if force_tier:
            tier_name = force_tier
        elif duration_hours <= 24:
            tier_name = "raw"
        elif duration_hours <= 24 * 7:
            tier_name = "1m"
        else:
            tier_name = "5m"

        tier_data = {"raw": self.raw, "1m": self.tier_1m, "5m": self.tier_5m}[tier_name]
        results = [(ts, mn, mx, avg, cnt) for ts, mn, mx, avg, cnt in tier_data
                   if start_ms <= ts <= end_ms]
        return results, tier_name


# ─── Demo ─────────────────────────────────────────────────────────────────────

import random, math

db = TieredTSDB()

# Write 2 weeks of synthetic data (15-second intervals)
base_ms = int(time.time() * 1000) - 14 * 24 * 3600 * 1000
for i in range(14 * 24 * 240):   # 2 weeks × 240 points/hour
    ts = base_ms + i * 15_000
    # Diurnal pattern: higher during business hours
    hour = (ts // 3_600_000) % 24
    base_val = 50 + 30 * math.sin(math.pi * hour / 12)  # Simple sine wave
    db.write(ts, base_val + random.gauss(0, 5))

print(f"Total raw points: {len(db.raw):,}")

db.downsample_to_1m()
db.downsample_to_5m()

now_ms = base_ms + 14 * 24 * 3600 * 1000

# Query last 1 hour → raw tier
pts, tier = db.query(now_ms - 3_600_000, now_ms)
print(f"\nLast 1 hour → {tier} tier: {len(pts)} points")

# Query last 3 days → 1-min tier
pts, tier = db.query(now_ms - 3 * 24 * 3_600_000, now_ms)
print(f"Last 3 days  → {tier} tier: {len(pts)} points")

# Query last 10 days → 5-min tier
pts, tier = db.query(now_ms - 10 * 24 * 3_600_000, now_ms)
print(f"Last 10 days → {tier} tier: {len(pts)} points")
print("\n(Notice: each tier returns a manageable number of points for the client)")
```
