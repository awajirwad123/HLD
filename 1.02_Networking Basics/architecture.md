# Networking Basics for System Design — Architecture

## Why Networking Matters in HLD

Every component in a distributed system communicates over a network. Your design decisions — sync vs async, REST vs gRPC, CDN placement — all depend on understanding latency, protocols, and failure modes at the network layer.

---

## 1. HTTP Protocol Evolution

```
HTTP/1.1  →  HTTP/2  →  HTTP/3
  1997          2015        2022
```

### HTTP/1.1
- One request per TCP connection (by default)
- Keep-alive added persistent connections, but still **head-of-line blocking** (next request waits for previous response)
- Text-based headers (verbose, uncompressed)

### HTTP/2
- **Multiplexing** — multiple requests over a single TCP connection simultaneously
- **Header compression** (HPACK) — reduces overhead
- **Server push** — server can proactively send resources
- Still has TCP-level head-of-line blocking

### HTTP/3
- Built on **QUIC** (UDP-based) instead of TCP
- Eliminates head-of-line blocking at transport layer
- Faster connection setup (0-RTT for known servers)
- Better performance on lossy networks (mobile)
- Used by: Google, Cloudflare, YouTube

### Interview Decision Tree

```
Need browser-facing API?
  └─► HTTP/2 (default modern choice)

Need ultra-low latency internal service calls?
  └─► gRPC over HTTP/2

High packet loss environment (mobile)?
  └─► HTTP/3 / QUIC

Legacy compatibility needed?
  └─► HTTP/1.1
```

---

## 2. TCP vs UDP

| Property          | TCP                         | UDP                         |
|-------------------|-----------------------------|-----------------------------|
| Connection        | Connection-oriented (3-way handshake) | Connectionless      |
| Reliability       | Guaranteed delivery, ordering | No guarantee              |
| Flow control      | Yes (congestion control)    | No                          |
| Latency overhead  | Higher (ACKs, retransmits)  | Lower                       |
| Use cases         | HTTP, DB, file transfer     | DNS, video streaming, gaming|

### Real-world Mapping

- **TCP** — PostgreSQL connections, Redis, Kafka, HTTP/1.1 and HTTP/2
- **UDP** — DNS queries, WebRTC (video calls), live streaming (where a dropped frame is OK), QUIC (HTTP/3 runs on UDP but adds its own reliability layer)

### When to mention UDP in an interview

> *"Video streaming like Netflix uses adaptive bitrate streaming over HTTP/TCP for reliability, but real-time video calls (Zoom, WebRTC) use UDP because a 50ms old frame is useless — it's better to drop it and move on."*

---

## 3. DNS Resolution Flow

Understanding DNS matters for: CDN design, geo-routing, failover, and blue/green deployments.

```
User types: www.example.com
  │
  ▼
[ Browser Cache ] ──► hit → done
  │ miss
  ▼
[ OS Cache / /etc/hosts ]
  │ miss
  ▼
[ Recursive Resolver ] (ISP or 8.8.8.8)
  │
  ├──► [ Root Nameserver ] → "try .com TLD server"
  │
  ├──► [ TLD Nameserver (.com) ] → "try example.com NS"
  │
  └──► [ Authoritative Nameserver ] → "www = 93.184.216.34"
  │
  ▼
IP returned to browser → TCP connection established
```

### Key DNS Concepts for HLD

- **TTL (Time To Live)** — how long DNS records are cached; low TTL = faster failover but more DNS load
- **A record** — maps hostname → IPv4
- **CNAME** — alias to another hostname (used by CDNs)
- **GeoDNS** — returns different IPs based on user location → used to route to nearest data center
- **DNS-based load balancing** — multiple A records; limited (no health checks, relies on TTL)

### Real-world: How Netflix uses DNS
Netflix uses GeoDNS + AWS Route53 to route users to the nearest Open Connect CDN node. No single global IP — different users resolve to different IPs.

---

## 4. CDN (Content Delivery Network)

CDNs are one of the highest-leverage tools in any large-scale HLD.

### How a CDN Works

```
User (India)
  │
  ▼
[ CDN Edge Node — Mumbai ] ←── Cache HIT → response in ~5ms
  │ Cache MISS
  ▼
[ CDN Origin Shield — Singapore ]
  │ Miss
  ▼
[ Origin Server — US-East ]
  (response cached at edge for future requests)
```

### What CDNs Cache
- Static assets: JS, CSS, images, fonts
- Video segments (Netflix, YouTube)
- API responses (with proper Cache-Control headers)

### CDN Cache Control

```
# Response headers that control CDN behavior
Cache-Control: public, max-age=86400        # cache for 1 day
Cache-Control: private, no-store            # don't cache (user-specific)
Cache-Control: s-maxage=3600               # CDN-specific TTL
Vary: Accept-Encoding                       # cache separate copies per encoding
```

### CDN Trade-offs

| Benefit                              | Cost / Risk                             |
|--------------------------------------|-----------------------------------------|
| Reduced origin load (>90% cache hit) | Stale content if TTL too high           |
| Lower latency for global users       | Cache invalidation complexity           |
| DDoS protection at edge              | Cost per GB transferred                 |
| SSL termination offload              | Not suitable for personalized responses |

### Push vs Pull CDN

- **Pull CDN** — cache is populated on first request (miss → origin → cache). Simple. Most common.
- **Push CDN** — you proactively upload content to CDN nodes. Used for known large assets (video releases, software updates).

---

## 5. HTTP vs HTTPS

- **HTTP** — plaintext; vulnerable to MITM
- **HTTPS** — HTTP + TLS; encrypted + authenticated

### TLS Handshake (simplified)

```
Client                          Server
  │── ClientHello (TLS version, ciphers) ──►│
  │◄── ServerHello + Certificate ──────────│
  │── Verify cert (via CA chain) ──────────│
  │── Key Exchange ────────────────────────►│
  │◄────────────── Encrypted session ───────│
```

### Interview relevance
- TLS adds ~1 RTT latency overhead (TLS 1.3 reduces to 1-RTT or 0-RTT)
- Certificates must be managed (auto-renewal via Let's Encrypt / ACM)
- Load balancers and CDNs act as **TLS termination** points — backend traffic can use plain HTTP internally (within VPC)

---

## 6. Long Polling, SSE, WebSockets (Preview)

| Protocol       | Direction         | Use Case                       |
|----------------|-------------------|--------------------------------|
| HTTP polling   | Client → Server   | Simple, inefficient            |
| Long polling   | Client → Server   | Simulated push, higher latency |
| SSE            | Server → Client   | News feeds, live scores        |
| WebSockets     | Bidirectional     | Chat, real-time collab         |

> Full deep-dive in Topic 16 (WebSockets & Real-Time Communication).

---

## 7. mTLS in Microservices

In microservices, simply trusting any internal service call is insecure (lateral movement attacks). mTLS solves this.

### What is mTLS?

```
Standard TLS:
  Client verifies Server cert → encrypted tunnel
  Server does NOT verify client identity

mTLS (Mutual TLS):
  Client verifies Server cert
  Server verifies Client cert
  Both parties proved who they are
```

### Why it matters in system design

| Without mTLS                          | With mTLS                              |
|---------------------------------------|----------------------------------------|
| Attacker who gets into VPC can call any service | Only cert-holding services can call each other |
| Service identity = IP address (spoofable) | Service identity = cryptographic cert |
| Hard to audit who called what         | Full auditability via cert identities  |

### How it works in practice (service mesh)

```
Service A                               Service B
  │                                        │
[ Envoy Sidecar ] ──── mTLS ────────► [ Envoy Sidecar ]
  │                                        │
[ App (plain HTTP on localhost) ]   [ App (plain HTTP on localhost) ]
```

- App code writes plain HTTP; the sidecar proxy (Envoy/Istio) handles mTLS transparently
- Certs are rotated automatically by the service mesh CA (SPIFFE/SVID standard)
- Zero code change in the application

**When to mention in interviews:**
> *"For service-to-service auth within the cluster, I'd use mTLS via a service mesh like Istio rather than application-level API keys. This gives us automatic cert rotation, zero-trust networking, and full observability on inter-service traffic."*

---

## 8. Anycast Routing

Anycast is how CDN providers and DNS services serve millions of users with ultra-low latency.

### How Anycast Works

```
Same IP address advertised from multiple locations:

User in US-East ─────► 1.2.3.4 → routed to New York PoP
User in EU      ─────► 1.2.3.4 → routed to Frankfurt PoP
User in Asia    ─────► 1.2.3.4 → routed to Singapore PoP
```

- **Same IP, different physical destinations** — BGP routing directs users to the nearest node
- No DNS tricks needed — routing is at the network layer

### Use Cases

| Use Case               | Example                              |
|------------------------|--------------------------------------|
| Global DNS resolution  | Cloudflare (1.1.1.1), Google (8.8.8.8) |
| CDN edge routing       | Cloudflare, Fastly, Akamai           |
| DDoS mitigation        | Attack traffic absorbed by nearest PoP |
| Low-latency APIs       | Cloudflare Workers, AWS Global Accelerator |

### Anycast vs GeoDNS

| Property          | Anycast                          | GeoDNS                           |
|-------------------|----------------------------------|----------------------------------|
| Routing decision  | Network layer (BGP)              | DNS layer (A record per region)  |
| Failover speed    | Instant (BGP reconverge ~seconds)| Depends on DNS TTL               |
| Granularity       | Network topology                 | Geographic location              |
| Complexity        | Requires BGP infrastructure      | Easy via DNS provider            |

**Interview answer:**
> *"Cloudflare uses anycast — all their edge IPs are the same globally. BGP routes packets to the nearest PoP. GeoDNS (like Route 53 latency routing) is simpler to set up but failover is bounded by DNS TTL."*
