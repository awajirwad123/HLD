# Networking Basics — Notes

## Quick Reference

---

### HTTP Version Comparison

| Feature                  | HTTP/1.1       | HTTP/2         | HTTP/3 (QUIC)    |
|--------------------------|----------------|----------------|------------------|
| Transport                | TCP            | TCP            | UDP              |
| Multiplexing             | ❌ (one req/conn) | ✅           | ✅               |
| Header compression       | ❌             | ✅ (HPACK)     | ✅ (QPACK)       |
| HOL blocking             | Yes (TCP)      | Yes (TCP)      | ❌ Eliminated    |
| Server push              | ❌             | ✅             | ✅               |
| 0-RTT reconnect          | ❌             | ❌             | ✅               |
| Adoption                 | Ubiquitous     | ~65% of web    | Growing fast     |

---

### TCP vs UDP — One-Line Summary

- **TCP** = reliable, ordered, slower — use when data must arrive correctly
- **UDP** = fast, lossy, no guarantees — use when speed > reliability (video, DNS, games)

---

### DNS Key Records

| Record | Purpose                              | Example                          |
|--------|--------------------------------------|----------------------------------|
| A      | Hostname → IPv4                      | `api.example.com → 1.2.3.4`     |
| AAAA   | Hostname → IPv6                      | `api.example.com → ::1`         |
| CNAME  | Alias → another hostname             | `www → example.com`             |
| NS     | Authoritative nameserver for domain  | `example.com NS ns1.cloudflare` |
| MX     | Mail server for domain               | Email routing                   |
| TXT    | Arbitrary text (verification, SPF)   | `"v=spf1..."`                   |

---

### CDN Key Concepts

- **Pull CDN** — lazy population (first request hits origin, result cached for TTL)
- **Push CDN** — proactive upload (for predictable large content like video)
- **Cache-Control: s-maxage** — CDN-specific TTL (overrides max-age for CDNs)
- **Stale-while-revalidate** — serve stale content instantly, refresh in background
- **Cache hit ratio** — target >90% for static assets; lower is normal for dynamic

---

### TLS Key Points

- TLS 1.3 = 1-RTT handshake (vs 2-RTT in TLS 1.2)
- TLS 1.3 with session resumption = 0-RTT
- **TLS termination** at load balancer = backend traffic is HTTP (faster, simpler certs)
- **mTLS (mutual TLS)** = both client AND server present certificates — used for service-to-service auth in microservices

---

### Network Latency Reference

| Hop                              | Approx Latency    |
|----------------------------------|-------------------|
| Same server (loopback)           | < 0.1 ms          |
| Same data center (LAN)           | ~0.5 ms           |
| Same region (different AZ)       | ~1–2 ms           |
| Cross-continent (US → Europe)    | ~80–100 ms        |
| Cross-Pacific (US → Asia)        | ~150–200 ms       |
| CDN edge (user near PoP)         | ~5–20 ms          |

---

### Cache-Control Header Cheat Sheet

```
Cache-Control: public, max-age=86400        # cacheable, 1 day
Cache-Control: private, no-store            # never cache (user data)
Cache-Control: no-cache                     # revalidate every time
Cache-Control: s-maxage=3600               # CDN TTL = 1 hour
Cache-Control: stale-while-revalidate=60   # serve stale, refresh async
```

---

### Interview Checklist — Networking

When designing any system, ask yourself:
- [ ] Where is TLS terminated? (CDN/LB or end-to-end?)
- [ ] What protocol between services? (HTTP/2? gRPC?)
- [ ] Is a CDN justified? (any static/cacheable content?)
- [ ] What are the cross-region latency implications?
- [ ] Does DNS TTL support fast failover if needed?
