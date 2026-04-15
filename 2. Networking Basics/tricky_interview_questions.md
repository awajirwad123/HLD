# Networking Basics — Tricky Interview Questions

*Deep-dive questions that separate surface-level knowledge from real understanding.*

---

## Q1: "Why doesn't everyone just use HTTP/3? What are the downsides?"

**Trap:** Candidates say "it's better in every way" without acknowledging real-world constraints.

**Strong answer:**
- UDP is often **blocked or throttled** by enterprise firewalls and middleboxes (they expect TCP)
- **More complex to implement** — QUIC reimplements reliability, congestion control, and TLS inside UDP
- **CPU overhead** — encryption can't be offloaded to NIC as easily as TCP
- **Immature tooling** — less debugging support, fewer libraries compared to HTTP/2
- **Best for**: mobile users, lossy networks, CDN-to-browser traffic
- **Avoid for**: internal service-to-service calls in a controlled DC — HTTP/2 or gRPC is simpler

---

## Q2: "You set DNS TTL to 30 seconds for fast failover. What problems does this create?"

**Strong answer:**
Too-low TTL causes:
1. **Massive DNS query volume** — millions of clients re-querying every 30 seconds → DNS servers under load
2. **DNS amplification risk** — large DNS response to small query; attackers can abuse this
3. **SLA violation on DNS itself** — if your DNS provider has issues, low TTL means outages propagate faster
4. **Not all clients respect TTL** — some OS/browser caches ignore TTL and cache longer (especially Java's JVM was notorious for this)

**Real-world practice:** Keep TTL at 300–3600s normally. Drop to 60s only **before** a planned change (not after an outage starts — it's already too late).

---

## Q3: "A user in Japan is getting 300ms latency to your API. How do you debug and fix this?"

**Debug approach:**
```
1. Check if CDN is even in use → Is the request hitting a CDN edge node in Japan?
2. traceroute / MTR to find where latency is introduced
3. DNS lookup — is GeoDNS resolving to a nearby region?
4. TLS handshake time — HTTP/2 or HTTP/3 would reduce RTT count
5. Time to first byte (TTFB) — is origin processing slow, or just network?
```

**Fix options (layered):**
- Add a CDN PoP in APAC → cache hits serve from ~5ms
- Deploy an API replica in Tokyo → route DB reads to local read replica
- Use HTTP/3 (QUIC) → 0-RTT on reconnect
- Switch from HTTP/1.1 to HTTP/2 → multiplexing reduces sequential round trips

---

## Q4: "If the CDN is serving stale content, how do you fix it without changing TTL?"

**Cache invalidation strategies:**
1. **Cache purge API** — most CDNs (Cloudflare, CloudFront) have an API to immediately invalidate a URL or prefix → `POST /invalidate?path=/product/123`
2. **URL versioning** — embed version in URL: `/static/app.v2.3.js` → new URL = automatic cache miss
3. **ETag + If-None-Match** — client sends previous ETag; CDN/origin returns 304 Not Modified or new content
4. **Surrogate keys / cache tags** — tag cached responses with logical identifiers (e.g., `product:42`), then purge by tag when data changes

**Trade-off to mention:**
> "URL versioning is the most reliable — a new URL is always a cache miss — but requires build tooling to inject version hashes. Purge APIs are simpler but purge propagation across global PoPs takes 1–5 seconds, so there's a brief window of inconsistency."

---

## Q5: "Your service uses HTTP/2 internally between microservices. A new team says 'let's use gRPC instead.' What's your response?"

**They're not mutually exclusive** — gRPC runs *on top of* HTTP/2.

**Reasons TO migrate to gRPC:**
- Protocol Buffers = smaller payload than JSON (~5x compression)
- Strongly-typed contracts (`.proto` files) = less integration bugs
- Built-in bidirectional streaming
- Better tooling for service discovery and load balancing (grpc-lb)
- Auto-generated client code in multiple languages

**Reasons to stick with HTTP/REST:**
- Browser clients can't use gRPC directly (needs gRPC-Web proxy)
- JSON is human-readable and easier to debug
- More familiar to most teams; less tooling setup
- FastAPI + Pydantic is already type-safe with OpenAPI docs

**Strong answer:**
> "I'd use gRPC for internal service-to-service calls where performance and strong contracts matter. I'd keep REST/HTTP for public-facing APIs where browser compatibility and developer ergonomics win."

---

## Q6: "Does HTTPS guarantee that the server is who it says it is?"

**Nuanced answer, not just "yes":**
- HTTPS encrypts traffic AND authenticates the server via TLS certificates
- Certificates are signed by a **Certificate Authority (CA)** — trust is only as strong as the CA
- **Certificate pinning** — hardcode the expected cert/public key in the client to prevent MITM even with a rogue CA
- **HSTS** (HTTP Strict Transport Security) — tells browsers to only connect via HTTPS for a domain, preventing SSL stripping attacks
- **SAN (Subject Alternative Names)** — a single cert can cover multiple domains

**What HTTPS does NOT guarantee:**
- That the server code is secure or honest
- That the certificate hasn't been issued to the wrong party (CT logs help catch this)

---

## Q7: "You have 10 CDN edge nodes globally. A user in Brazil connects and gets a cache miss on every request. What's wrong?"

**Possible root causes:**

1. **`Cache-Control: private` or `no-store`** on responses → CDN correctly not caching
2. **`Vary: Cookie` or `Vary: Authorization`** header → CDN creating a separate cache entry per user (no sharing)
3. **Short TTL plus high content churn** → cache keeps expiring before the next request from the same region
4. **Query strings not normalized** → `/product?id=1&color=red` and `/product?color=red&id=1` treated as different cache keys
5. **POST requests (not GET)** → CDNs don't cache non-idempotent methods by default

**Fix:** Audit your response headers. `Cache-Control: public, s-maxage=3600` with no `Vary: Cookie` on assets that are truly public.

---

## Q8: "Walk me through exactly what happens, in networking terms, when a user opens Netflix on their phone."

**Expected to cover:**
1. DNS resolution → GeoDNS returns nearest CDN IP
2. TCP 3-way handshake + TLS 1.3 handshake (1-RTT)
3. HTTP/2 GET for the app shell (HTML/CSS/JS) → served from CDN edge
4. App makes authenticated API call to Netflix backend → passes CDN (not cached, private)
5. Backend returns content manifest → URLs for video chunks (CDN-hosted)
6. Player fetches video chunks — TCP or QUIC, adaptive bitrate (ABR) → starts low resolution, upgrades as bandwidth is confirmed
7. Simultaneously: CDN edge is streaming, analytics events are being sent async via queue

**Key insight to say:** *"Netflix's Open Connect CDN caches video chunks at ISP level — video bytes literally never leave the ISP's data center in ideal conditions, getting latency below 5ms."*

---

## Q9: "You're designing a global API. Should you use Anycast or GeoDNS for routing users to the nearest region?"

**Trap:** Treating them as alternatives when they solve the problem differently.

**Strong answer:**

| Property              | Anycast                           | GeoDNS (e.g., Route 53)          |
|-----------------------|-----------------------------------|----------------------------------|
| Routing layer         | Network (BGP)                     | DNS layer                        |
| Failover speed        | Seconds (BGP reconvergence)       | Bounded by DNS TTL               |
| Infrastructure needed | BGP peering at PoPs               | Any DNS provider                 |
| Setup complexity      | High                              | Low                              |
| Best for              | Global CDN, DNS, DDoS mitigation  | Multi-region API routing         |

> "For a SaaS API needing global low latency, I'd use GeoDNS with Route 53 — it's simpler, already integrated with ALB/CloudFront, and TTL-based failover is acceptable (60–300s). If we needed DDoS protection and instant failover, I'd move to Cloudflare's anycast network in front of our origin."

---

## Q10: "Your CDN, DNS, and TLS setup is costing $50K/month. An engineer suggests removing the CDN to save money. Your response?"

**This tests whether you can reason about cost vs reliability trade-offs quantitatively.**

**Framework to answer:**

1. **What does removing the CDN actually save?** CDN cost = bandwidth egress charges. Calculate: `traffic_TB × $/TB`
2. **What does it cost us?**
   - Origin servers must absorb 100% of traffic → need 5–10x more compute
   - Cross-region latency increases from ~10ms (CDN edge) to ~150ms (origin)
   - DDoS protection disappears (CDN absorbs volumetric attacks)
   - Cache hit rate for static assets (images, JS) was ~95% → now all hit origin
3. **Real math (example):**
   - CDN cost: $5K/month
   - Origin EC2 to replace CDN capacity: $20K/month + latency SLA breach penalties
   - "Removing CDN saves $5K but costs $20K in compute + customer churn from 10x slower page loads"

**Senior signal:** Cost optimization means reducing total cost, not just one line item. Always model the cascade effects.
