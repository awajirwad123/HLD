# Networking Basics — Quick-Fire Questions

*Short answers. Rapid revision.*

---

**Q1: What is the main improvement HTTP/2 brought over HTTP/1.1?**
Multiplexing — multiple requests can be sent over a single TCP connection simultaneously, eliminating head-of-line blocking at the HTTP layer.

---

**Q2: What transport protocol does HTTP/3 use, and why?**
UDP (via QUIC). Eliminates TCP-level head-of-line blocking and enables 0-RTT reconnection on mobile/lossy networks.

---

**Q3: What's the difference between TCP and UDP?**
TCP = reliable, ordered, connection-based. UDP = fast, no delivery guarantee, connectionless. Use TCP when correctness matters; UDP when speed > reliability.

---

**Q4: Why would you use UDP for live video calls (Zoom/WebRTC)?**
A dropped frame from 100ms ago is useless. It's better to skip it and show the next frame than to retransmit and cause freezing. Latency matters more than completeness.

---

**Q5: What is a CDN and why is it used?**
A Content Delivery Network caches content at geographically distributed edge nodes. Reduces latency for global users and offloads origin server traffic.

---

**Q6: What is the difference between Pull CDN and Push CDN?**
Pull CDN = cache populated on first request (lazy). Push CDN = content proactively uploaded to edge nodes. Pull is simpler; push is better for known large content like video releases.

---

**Q7: What HTTP header controls CDN caching behavior?**
`Cache-Control`. Key values: `public, max-age=N` (cacheable), `private, no-store` (don't cache), `s-maxage=N` (CDN-specific TTL).

---

**Q8: What is DNS TTL and why does it matter in system design?**
TTL = how long a DNS record is cached. Low TTL enables fast failover (but more DNS queries). High TTL reduces DNS load but slows down record changes.

---

**Q9: What is a CNAME record?**
An alias — maps one hostname to another. Commonly used by CDNs so `assets.example.com` points to `example.cdn-provider.net`.

---

**Q10: What is TLS termination?**
Decrypting HTTPS traffic at a load balancer or CDN edge, so backend services communicate over plain HTTP internally (within a trusted network).

---

**Q11: What is mTLS?**
Mutual TLS — both client and server present certificates to authenticate each other. Used for service-to-service authentication in microservices.

---

**Q12: What latency should you expect between two services in the same data center?**
~0.5 ms. Cross-region (e.g., US to Europe) is ~80–100 ms.

---

**Q13: What is GeoDNS?**
A DNS technique that returns different IP addresses based on the user's geographic location — used to route traffic to the nearest data center or CDN node.

---

**Q14: What is the difference between a CDN cache miss and cache hit?**
Hit = content served from edge node (fast, ~5–20ms). Miss = edge node fetches from origin (slow, adds cross-region latency).

---

**Q15: What does `stale-while-revalidate` mean in Cache-Control?**
Serve the stale cached content immediately for low latency, while asynchronously fetching a fresh copy in the background for the next request.

---

**Q16: What HTTP version does gRPC use?**
HTTP/2. gRPC requires HTTP/2 for bidirectional streaming and multiplexing.

---

**Q17: What is the DNS resolution order on a client machine?**
Browser cache → OS cache (/etc/hosts) → Recursive resolver (ISP/8.8.8.8) → Root NS → TLD NS → Authoritative NS.

---

**Q18: When does a CDN NOT help?**
For personalized/user-specific responses (e.g., user dashboards, authenticated API responses) — these can't be shared across users and must hit origin every time.

---

**Q19: What does the `Vary` header do in HTTP caching?**
Tells caches to store separate copies of the response for different request header values (e.g., `Vary: Accept-Encoding` → separate cache for gzip vs non-gzip).

---

**Q20: What is head-of-line blocking?**
When a slow/lost packet blocks all subsequent packets on the same TCP connection from being processed, even if they already arrived. HTTP/2 solved it at the HTTP layer; HTTP/3 (QUIC) solved it at the transport layer.
