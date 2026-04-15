# Networking Basics — Hands-On

## Exercise 1: Observe HTTP Versions in the Wild

Run this in your terminal to see which HTTP version a server uses:

```bash
# Check HTTP version negotiation
curl -sI --http2 https://www.google.com | grep -i "HTTP"
curl -sI --http1.1 https://www.google.com | grep -i "HTTP"

# Check if HTTP/3 is advertised (Alt-Svc header)
curl -sI https://www.cloudflare.com | grep -i "alt-svc"
```

Expected output for HTTP/3 capable servers:
```
alt-svc: h3=":443"; ma=86400
```
`h3` = HTTP/3, `ma=86400` = advertise for 1 day.

---

## Exercise 2: DNS Lookup Inspection

```bash
# Basic DNS lookup
nslookup google.com

# Trace full DNS resolution path
nslookup -type=NS google.com        # find authoritative nameservers
nslookup google.com 8.8.8.8         # query Google's public resolver

# Check TTL values
nslookup -debug google.com | grep ttl

# On Linux/Mac:
dig google.com            # full DNS answer with TTL
dig +trace google.com     # full resolution chain
```

**Insight to internalize:** Low TTL = you can change DNS quickly during failover. High TTL = DNS caches longer, reducing DNS server load but slowing down updates.

---

## Exercise 3: Simulate a CDN Cache Decision — Python

This simulates how a CDN edge node decides to serve from cache or fetch from origin:

```python
import time
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class CacheEntry:
    content: str
    cached_at: float
    ttl_seconds: int

    def is_expired(self) -> bool:
        return (time.time() - self.cached_at) > self.ttl_seconds

class CDNEdgeNode:
    """Simulates a CDN edge cache (Pull model)."""

    def __init__(self, origin_url: str):
        self.origin_url = origin_url
        self._cache: dict[str, CacheEntry] = {}
        self.cache_hits = 0
        self.cache_misses = 0

    def _fetch_from_origin(self, path: str) -> tuple[str, int]:
        """Simulate origin fetch. Returns (content, ttl)."""
        # In reality: HTTP GET to origin server
        print(f"  [ORIGIN FETCH] {self.origin_url}{path}")
        content = f"<content of {path}>"
        ttl = 3600  # 1 hour cache
        return content, ttl

    def get(self, path: str) -> str:
        entry = self._cache.get(path)

        if entry and not entry.is_expired():
            self.cache_hits += 1
            print(f"  [CACHE HIT]  {path}")
            return entry.content
        
        # Miss or stale
        self.cache_misses += 1
        content, ttl = self._fetch_from_origin(path)
        self._cache[path] = CacheEntry(
            content=content,
            cached_at=time.time(),
            ttl_seconds=ttl
        )
        return content

    def stats(self):
        total = self.cache_hits + self.cache_misses
        hit_rate = (self.cache_hits / total * 100) if total else 0
        print(f"\nHits: {self.cache_hits}, Misses: {self.cache_misses}, Hit rate: {hit_rate:.1f}%")


# Simulate traffic
cdn = CDNEdgeNode(origin_url="https://origin.example.com")

paths = ["/video/intro.mp4", "/img/logo.png", "/video/intro.mp4", "/video/intro.mp4"]
for path in paths:
    cdn.get(path)

cdn.stats()
```

**Expected output:**
```
  [ORIGIN FETCH] /video/intro.mp4
  [ORIGIN FETCH] /img/logo.png
  [CACHE HIT]  /video/intro.mp4
  [CACHE HIT]  /video/intro.mp4

Hits: 2, Misses: 2, Hit rate: 50.0%
```

---

## Exercise 4: Inspect TLS Handshake

```bash
# See TLS version and cipher negotiated
openssl s_client -connect google.com:443 -tls1_3 2>&1 | grep -E "Protocol|Cipher"

# Check certificate details
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -dates -subject
```

In FastAPI, HTTPS is handled by a reverse proxy (Nginx/Caddy). The app itself runs HTTP internally:

```
Internet (HTTPS:443) → Nginx (TLS termination) → FastAPI (HTTP:8000, internal)
```

---

## Exercise 5: Build a Simple FastAPI Service with Proper HTTP Headers

```python
from fastapi import FastAPI, Response

app = FastAPI()

@app.get("/static-resource")
def get_static(response: Response):
    # Tell CDN/browser to cache for 1 day
    response.headers["Cache-Control"] = "public, max-age=86400, s-maxage=86400"
    response.headers["ETag"] = '"v1-abc123"'
    return {"data": "This can be cached by CDN"}

@app.get("/user-profile/{user_id}")
def get_profile(user_id: str, response: Response):
    # Private — never cache at CDN
    response.headers["Cache-Control"] = "private, no-store"
    return {"user_id": user_id, "name": "Alice"}

@app.get("/feed")
def get_feed(response: Response):
    # Short CDN cache — acceptable staleness
    response.headers["Cache-Control"] = "public, max-age=30, stale-while-revalidate=60"
    return {"posts": []}
```

**Key takeaway:** Correct `Cache-Control` headers are what make CDN caching work. Missing or wrong headers = CDN fetches origin on every request.

---

## Exercise 6: Mental Model — Network Latency in Your Architecture

Annotate every hop in your HLD with expected latency:

```
User (Sydney)
  │
  │ ~150ms (cross-Pacific)
  ▼
CDN Edge (Sydney)  ──► Cache HIT → ~5ms total ✅
  │ Miss
  │ ~20ms (Sydney → Singapore CDN origin shield)
  ▼
Origin Shield (Singapore)
  │ Miss
  │ ~180ms (Singapore → US-East origin)
  ▼
Load Balancer → API Server → Redis → DB

Total cold path: ~350ms
Total warm (CDN hit): ~5ms
```

**Interview habit:** Always annotate latency per hop when drawing your HLD. Shows you understand network reality.

---

## Exercise 7: Propagating Trace Headers Across Services (FastAPI)

In distributed systems, a single user request touches many services. Trace headers tie them together.

```python
import httpx
import uuid
import logging
from fastapi import FastAPI, Request

app = FastAPI()
logger = logging.getLogger(__name__)

# --- Service A: API Gateway (public-facing) ---
@app.get("/user/{user_id}/timeline")
async def get_timeline(user_id: str, request: Request):
    # Accept trace ID from upstream (e.g., CDN or client), or generate new
    trace_id = request.headers.get("X-Trace-ID", str(uuid.uuid4()))

    # Call downstream Feed Service — propagate the trace ID
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            f"http://feed-service/feed/{user_id}",
            headers={
                "X-Trace-ID": trace_id,           # propagate
                "X-Calling-Service": "api-gateway" # optional, for debugging
            },
            timeout=2.0
        )

    logger.info({"trace_id": trace_id, "service": "api-gateway", "action": "get_timeline", "user_id": user_id})
    return {"trace_id": trace_id, "timeline": resp.json()}


# --- Service B: Feed Service (internal) ---
feed_app = FastAPI()

@feed_app.get("/feed/{user_id}")
async def get_feed(user_id: str, request: Request):
    trace_id = request.headers.get("X-Trace-ID", "unknown")

    # Log with same trace_id → searchable across both services
    logger.info({"trace_id": trace_id, "service": "feed-service", "action": "get_feed", "user_id": user_id})

    # Call DB, cache, etc.
    return {"posts": []}
```

**What this enables:**
```
User request → trace_id: abc-123
  └── api-gateway: GET /timeline  [trace: abc-123]
    └── feed-service: GET /feed   [trace: abc-123]
      └── DB query                [trace: abc-123]

Search logs for "abc-123" → see the full distributed trace in one view
```

**Tools that build on this:**
- **Jaeger / Zipkin** — open source distributed tracing
- **OpenTelemetry** — vendor-neutral standard for traces, metrics, logs
- **Datadog APM / AWS X-Ray** — managed tracing with flame graphs

**Interview phrase:**
> *"I'd propagate a correlation ID via `X-Trace-ID` header across all service calls and include it in every structured log entry. This way, a Grafana Loki or Datadog query on one trace ID gives you the full request journey across all services."*
