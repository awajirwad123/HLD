# Networking Basics — Interview Simulator

*Full mock interview scenarios focused on networking decisions. Set a timer and work through each before reading the guided answer.*

---

## How to Use

1. Read the **scenario** only
2. Set timer: **15 minutes** per scenario
3. Write your answer, then compare with the guided walkthrough

---

## Simulation 1: Design the Content Delivery for a Video Streaming Platform

### Scenario
> You're designing the video delivery layer for a Netflix-like platform. 10M DAU, average 2 hours of video/day, video library of 100K titles. How do you architect the delivery network?

---

### Guided Walkthrough

**Clarify first:**
- What regions? Global?
- What resolution profiles? (360p, 720p, 1080p, 4K)
- Acceptable buffering? (real-time: no, 2-second buffer: fine)

**Estimate:**
```
10M DAU × 2h × avg bitrate 4 Mbps = 10M × 7,200s × 0.5 MB/s = ~36 PB/day egress
Peak: 3x avg = ~108 PB/day peak
Cache-able: same top 1K titles = 80% of traffic → cache them at edge
```

**Architecture decision:**
```
Video Upload:
  Creator → [ Transcoding Service ] → multiple bitrates/resolutions
                                     → [ Object Storage (S3) ] ← origin

Video Delivery:
  User → GeoDNS / Anycast → [ CDN Edge Node (nearest PoP) ]
           MISS ──────────────────────────────────────────────► [ Origin S3 ]
           HIT  ──────────────────────────► User (5ms)

  CDN serves: DASH/HLS chunks (2-second segments)
  Player: adapts bitrate based on bandwidth (ABR)
```

**Protocol choices:**
- HTTP/2 for chunk delivery (multiplexed, persistent connection)
- HTTP/3 (QUIC) at CDN edge for mobile users on lossy networks
- No WebSockets needed — video segments are pull-based

**Cache strategy:**
- Popular top 1% of titles: cached at ALL PoPs (pre-warmed push CDN)
- Long-tail titles: pull CDN (cached on first request per region)
- TTL: 24–48h (content rarely changes once published)

**Failure handling:**
- CDN PoP down → DNS/anycast routes to next nearest PoP (~100ms degradation)
- Origin S3 unavailable → CDN serves from warm cache; new cache misses fail (503)

---

## Simulation 2: Debug a Real-World Latency Incident

### Scenario
> Users in Southeast Asia are reporting 3–5 second page load times. Your monitoring shows p50 latency is 200ms (normal), but p95 is 4,500ms. What's your investigation and fix plan?

---

### Guided Debug Approach

**Step 1 — Isolate the distribution**
```
p50 = 200ms (most users fine)
p95 = 4,500ms (5% are severely affected)
→ Not a global issue — specific subset of users or requests
```

**Step 2 — Check each layer**
```
Question to ask: Is the 4,500ms at network, DNS, TLS, or backend?

Tool: Real User Monitoring (RUM) — browser timing breakdown:
  DNS resolution:  50ms  (normal)
  TLS handshake:   3,800ms ← ANOMALY
  Time to first byte: 200ms (normal once connected)
  Content download: 300ms
```

**Step 3 — TLS handshake anomaly**
- TLS 1.2 = 2 RTT handshake × 200ms cross-region = 400ms expected
- 3,800ms → something is wrong: certificate chain too long? OCSP stapling not configured?

**Root cause found:** OCSP (certificate revocation check) not cached/stapled. Browser fetches OCSP from CA's server for every new connection → CA's OCSP server has high latency to SE Asia.

**Fix:**
1. Enable **OCSP stapling** at the CDN/LB — server includes OCSP response in TLS handshake, browser doesn't need to fetch it separately
2. Enable **TLS 1.3** — reduces to 1-RTT (halves handshake latency)
3. Verify CDN has PoPs in SE Asia (Jakarta, Singapore, Bangkok)

**Result:** p95 drops from 4,500ms to ~600ms.

---

## Simulation 3: Design a Globally Distributed API (15 min)

### Scenario
> Design a REST API for a fintech app used globally (US, EU, Asia). Strict requirement: EU data must not leave the EU (GDPR). p99 latency < 200ms globally.

---

### Guided Answer

**Key constraints:**
- Data residency (EU data stays in EU)
- Low global latency
- Consistency: financial data needs strong consistency within a region

**Architecture:**
```
User (EU)  → GeoDNS → [ EU Region ] (Frankfurt/Dublin)
User (US)  → GeoDNS → [ US Region ] (us-east-1)
User (Asia)→ GeoDNS → [ APAC Region ] (Singapore)

Each region:
  LB → API Server → Redis Cache (read) → Regional PostgreSQL primary
                                          (EU data ONLY in EU region)
```

**GDPR compliance:**
- User accounts tagged with region at registration
- API gateway routes based on user region, enforced by JWT claims
- No cross-region DB replication for EU data (data residency hard boundary)
- Audit logs for all EU data access (required by GDPR)

**Latency path:**
```
User (EU) → CDN edge ~10ms → API in eu-west-1 ~50ms → DB ~5ms = ~65ms ✅
User (Asia) → CDN edge ~10ms → APAC API ~80ms → DB ~5ms = ~95ms ✅
```

**Failure mode:**
- EU region down → EU users get 503 (cannot failover to US — GDPR)
- US region down → can failover to APAC (no data residency restriction)
- Strategy: 3 AZ deployment within each region for fault tolerance without cross-region data movement