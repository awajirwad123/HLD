# URL Shortener — Tricky Interview Questions

## Q1: "If Redis goes down, the redirect latency spikes to 50ms. Our SLA is 10ms P99. How do you solve this?"

**The trap:** "Add Redis Sentinel" — that handles Redis HA but doesn't help if Redis is simply slow or partially degraded.

**Full answer:**

The root cause: Redis miss → DB hit → 50ms. The solution has two layers:

**Layer 1: Multi-tier caching**
Add a local in-process LRU cache (Python `cachetools.LRUCache`) in each redirect service instance, sized for ~10,000 most-recently-seen codes:
```python
from cachetools import LRUCache, TTLCache
local_cache = TTLCache(maxsize=10_000, ttl=300)  # 5-minute TTL

async def redirect(code):
    if code in local_cache:
        return local_cache[code]  # L1: ~0.01ms
    
    val = await redis.hgetall(f"url:{code}")  # L2: ~1ms
    if not val:
        val = await db.fetchrow(...)          # L3: ~5ms
        await redis.hset(...)
    
    local_cache[code] = val["long_url"]
    return val["long_url"]
```
With power-law URL access (top 1% of codes = 90% of traffic), local cache absorbs most load.

**Layer 2: Redis Cluster with read replicas.** Even if primaries are degraded, read replicas serve cached data. Configure client to read from replicas for GET operations.

**Layer 3: Circuit breaker on Redis.** If Redis latency > 5ms for 3 consecutive calls, open circuit and go directly to DB read replica for a 30-second window. This prevents Redis slow responses from causing cascades.

**The insight:** The question tests whether you know that HA ≠ low-latency. You need defense in depth: L1 local cache → L2 Redis → L3 DB replica.

---

## Q2: "How would you scale this to 1 billion URL creations per day?"

Current design handles 1,160 writes/sec. At 1B/day that's ~11,560/sec — 10× higher.

**Bottlenecks to address:**

1. **Token uniqueness check.** Random token + `UNIQUE` constraint + retry loop works, but at 11k wps, collision check contention grows. Switch to Snowflake IDs — zero DB roundtrip for ID generation, guaranteed unique.

2. **Single primary DB.** At 11k wps PostgreSQL write throughput is still manageable (~50k wps possible with proper tuning), but INSERT hot spot on the `short_code` index is a concern. Partition strategy: shard DB by `short_code[0]` (first character) → 62 shards, each handling ~187 wps.

3. **Redirect path.** 10× redirects = 1.15M rps. Redis Cluster handles 1M+ ops/sec. Scale horizontally by adding shards. CDN becomes even more critical — push to 3-4 global PoPs.

4. **Click analytics at 1.15M/s.** Redis INCR is fine, but the flush worker pushing to DB must also shard. Alternatively, each counter flush writes to Kafka partitioned by `short_code`. ClickHouse consumers process click events.

5. **Storage.** 1B × 365 × 250 bytes × 5 years = 450 TB. Time-partition by `created_at` quarter. Archive partitions older than 1 year to cold object storage (S3 Glacier). DB only holds recent 1–2 years online.

---

## Q3: "What's wrong with using `MD5(long_url)[0:7]` as the short code?"

**Multiple problems:**

1. **Same URL always gets the same code.** User A and User B share the same short URL even if they both want separate analytics tracking. Not suitable for business use.

2. **Collisions.** MD5 outputs 128 bits. Taking the first 43 bits means birthday collision probability is ~50% after √(2^43) ≈ 3 million URLs. Real TinyURL at 10M+ URLs would have thousands of collisions daily — each collision silently redirects to the wrong URL.

3. **Pre-image attacks.** Famous URLs (google.com, amazon.com) produce known MD5 prefixes — easy to reverse. If you want to hide which URL is shortened, cryptographic unpredictability matters.

4. **No natural way to support custom aliases.** Custom aliases don't have a "long URL" to hash.

5. **Index performance.** Storing 7-byte CHAR vs CHAR derived from hash — same cost. But MD5 requires computing the hash on every write AND on every GET (to look up by long URL for deduplication). An O(|url_table|) index scan on `long_url TEXT` is expensive.

---

## Q4: "A user reports their link is broken — it returns 404. They created it 3 days ago and it worked yesterday. You find the row still exists in the DB. What's happening?"

**The investigation path:**

1. Check `is_active` — was it manually deactivated? Check audit log.  
2. Check `expires_at` — did it expire? `expires_at = now() - 1 day`? The redirect API returns 410 for expired, not 404. But if the error message says 404, something else is wrong.
3. Check Redis. The issue: a bug in the cleanup job set `is_active = false` for non-expired URLs. Or: the cleanup job soft-deleted the entry but the service returns 404 instead of 410 for `is_active = false`.
4. Check for cache inconsistency — Redis has a stale `is_active = false` flag cached with TTL 1h. DB was corrected but Redis still serves the bad data.

**Root cause in this scenario:** The Redis cache stores a stale `is_active = false` value. The fix:
- On write path (URL creation), always invalidate relevant cache keys
- Cleanup job must also invalidate Redis, not just update DB
- Admin deactivation must flush Redis key immediately (`DEL url:{code}`)

**Mitigation pattern:** Cache should only store `long_url` + `expires_at`, not `is_active`. Active check should always go through DB if `is_active` matters (or store it with a short TTL of 60s, accept 1-min stale reads for deactivations).

---

## Q5: "Design the analytics system so we can answer: which country do most clicks come from for URL X, broken down by day?"

**Naive approach:** Store per-click events in PostgreSQL. At 115k clicks/sec, that's 9.9 billion rows/day. Postgres cannot handle that write throughput, and analytical queries on 9.9B rows are slow.

**Scalable design:**

**Ingestion:** On every redirect, publish to Kafka topic `url-clicks`:
```json
{
  "short_code": "aB3xZ",
  "ts": 1713175200000,
  "ip": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "referer": "https://twitter.com"
}
```

**Enrichment consumer:** Resolve IP → (country, city, ISP) using MaxMind GeoIP2. Write enriched events to ClickHouse:

```sql
CREATE TABLE clicks (
    short_code  LowCardinality(String),
    ts          DateTime,
    country     LowCardinality(String),
    city        String,
    referer     String,
    device_type LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (short_code, ts);
```

**Query:**
```sql
SELECT country, toDate(ts) AS day, count() AS clicks
FROM clicks
WHERE short_code = 'aB3xZ'
  AND ts >= today() - 30
GROUP BY country, day
ORDER BY day, clicks DESC;
```

ClickHouse executes this over billions of rows in seconds using columnar storage and vectorized execution.

**Dashboard:** Grafana or a custom API queries ClickHouse. Pre-compute daily country rollups via ClickHouse materialized views for sub-100ms dashboard loads.
