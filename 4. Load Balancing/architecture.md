# Load Balancing — Architecture

## Overview

A load balancer distributes incoming traffic across multiple backend instances. It is the single most universal component in any HLD — present in every system at every scale. Understanding it deeply is non-negotiable.

---

## 1. Why Load Balancing Exists

Without a load balancer:
```
All users ──────────────────────► [ Single Server ] ← SPOF + bottleneck
```

With a load balancer:
```
All users ──► [ Load Balancer ] ──► [ Server A ]
                                ──► [ Server B ]
                                ──► [ Server C ]
```

Benefits:
- **Eliminates SPOF** — if one server fails, LB routes around it
- **Horizontal scalability** — add/remove servers without changing client config
- **Health checking** — automatically removes unhealthy instances
- **SSL termination** — offloads TLS from application servers
- **DDoS mitigation** — absorbs volumetric attacks before they reach the app

---

## 2. Layer 4 vs Layer 7 Load Balancing

This is the most important distinction to know for interviews.

### Layer 4 (Transport Layer — TCP/UDP)

```
Client TCP packet → LB reads: [src IP, src port, dst IP, dst port]
                             → selects server by connection tuple or IP hash
                             → forwards the TCP stream (without inspecting payload)
```

- Operates at the TCP/UDP level — **does not read HTTP headers or body**
- Extremely fast — minimal processing overhead
- Cannot route based on URL path, headers, or cookies
- Examples: AWS NLB, HAProxy (TCP mode), hardware F5

**Use when:**
- Non-HTTP protocols (gRPC, database connections, game servers)
- Ultra-low latency required (microsecond overhead matters)
- You only need TCP-level health checks (is port 80 open?)

### Layer 7 (Application Layer — HTTP/HTTPS)

```
Client HTTP request → LB reads: [method, URL path, headers, cookies]
                               → routes based on content
                               → example: /api/* → API servers, /static/* → CDN origin
```

- **Content-aware routing** — can route by URL path, hostname, header values
- Full request inspection → slightly higher overhead (~1ms)
- Can do SSL termination, request rewriting, rate limiting, WAF integration
- Examples: AWS ALB, Nginx, Envoy, Traefik

**Use when:**
- HTTP/HTTPS traffic (99% of web systems)
- You need path-based or host-based routing
- You need sticky sessions with cookie affinity
- You need advanced health checks (HTTP 200 on `/health`)

### Comparison Table

| Feature                  | Layer 4 (TCP)       | Layer 7 (HTTP)          |
|--------------------------|---------------------|-------------------------|
| Routing basis            | IP + port           | URL, headers, cookies   |
| Throughput               | Higher              | Slightly lower          |
| Protocol awareness       | None                | Full HTTP understanding |
| SSL termination          | Pass-through only   | Yes                     |
| Content-based routing    | No                  | Yes                     |
| Session stickiness       | IP-based only       | Cookie-based            |
| Use case                 | DB, gRPC, raw TCP   | REST, GraphQL, web apps |

---

## 3. Load Balancing Algorithms

### Round Robin
```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A (wraps around)
```
- Simple, equal distribution
- Problem: ignores server load — a busy Server A gets the same traffic as an idle one
- Best when all servers are identical and requests are similar duration

### Weighted Round Robin
```
Server A (weight 3): gets 3 of every 5 requests
Server B (weight 2): gets 2 of every 5 requests
```
- Useful when servers have different specs (2x RAM server gets 2x weight)

### Least Connections
```
Server A: 100 active connections
Server B: 50 active connections  ← new request goes here
Server C: 75 active connections
```
- Routes to the server with fewest active connections
- Better than round robin for long-lived requests (WebSockets, streaming)
- Requires LB to track connection counts → slightly more overhead

### Least Response Time
- Routes to the server with lowest combination of active connections + average response time
- Most accurate but most overhead
- Used by AWS ALB under the hood

### IP Hash
```
hash(client_IP) % server_count → always same server for same client
```
- Provides **sticky routing without cookies** — same client always hits same server
- Problem: if one IP generates huge traffic (NAT/proxy), that server overloads
- Fails if server count changes (all hashes remap)

### Consistent Hashing (advanced)
```
Servers and requests mapped to a ring
Each request routes to the nearest server clockwise on the ring
Adding/removing a server only remaps ~1/N of requests
```
- Solves the remapping problem of IP hash — minimal disruption when scaling
- Used in: distributed caches (Redis Cluster), CDN routing, Dynamo/Cassandra
- Full deep-dive in Topic 20

---

## 4. Health Checks

The LB must know which backends are healthy. Two types:

### Passive Health Checks
- LB observes real traffic responses
- If server returns 5xx or times out → mark unhealthy after N failures
- No extra traffic overhead
- Slower to detect failure (waits for real requests to fail)

### Active Health Checks
- LB periodically sends synthetic requests to `/health` endpoint
- Fast failure detection (detect down server in 5–10 seconds)
- Small overhead (one probe per interval per server)
- Best practice: always use active health checks

```
Health check config:
  Path: /healthz
  Interval: 10 seconds
  Timeout: 2 seconds
  Healthy threshold: 2 consecutive successes
  Unhealthy threshold: 3 consecutive failures
```

---

## 5. SSL/TLS Termination at LB

```
Internet (HTTPS) ──► [ Load Balancer ] ──► Backend servers (HTTP, internal)
                         ↑
                  TLS terminated here
                  One cert to manage

vs.

Internet (HTTPS) ──► [ Load Balancer ] ──► Backend servers (HTTPS, end-to-end)
                     (pass-through)             ↑
                                         More secure, more overhead
                                         Each backend needs cert
```

**Standard practice:** Terminate TLS at LB. Backend traffic is plain HTTP on a private VPC network. One cert, managed centrally (AWS ACM auto-renews). Backends don't need cert management.

**When to do end-to-end TLS:** Regulated industries (PCI-DSS, HIPAA) requiring encryption in transit even on internal networks. Use mTLS for service mesh (see Topic 2).

---

## 6. Sticky Sessions (Session Affinity)

Forces the same client to always hit the same backend:

```
Cookie-based stickiness:
  First request → LB routes to Server B, sets cookie: AWSALB=serverB_id
  Next requests → LB reads cookie → always routes to Server B
```

**When needed:** Legacy stateful apps that store session data in local memory (can't be refactored to external session store quickly).

**Problems:**
- Uneven load distribution (one user's heavy session = one server's problem)
- If that server crashes, session is lost anyway
- Prevents true horizontal scaling

**Senior answer:**
> *"Sticky sessions are a band-aid. The real fix is externalizing session state to Redis so any server can handle any request. I'd only use sticky sessions as a short-term migration strategy while refactoring to stateless."*

---

## 7. Global Server Load Balancing (GSLB)

For multi-region deployments, a GSLB routes users to the correct region before they hit the regional LB:

```
User (India)
  │
  ▼
[ GSLB / GeoDNS ] → resolves to Mumbai region IP
  │
  ▼
[ Regional Load Balancer — Mumbai ]
  │
  ├──► Server A
  └──► Server B
```

- Implemented via GeoDNS (Route 53 latency routing, Cloudflare) or anycast
- Handles regional failover: if Mumbai is unhealthy, redirect to Singapore
- TTL determines failover speed (short TTL = faster but more DNS traffic)

---

## 8. Load Balancer High Availability

The LB itself can be a SPOF. Solutions:

### Active-Passive (Failover)
```
LB-Primary (active) ◄── VRRP/keepalived ──► LB-Secondary (standby)
       │                    (heartbeat)
   All traffic                                  Takes over in ~1-5s on failure
```

### Active-Active
```
User traffic → DNS round-robin → LB-1 (active, handles 50%)
                               → LB-2 (active, handles 50%)
```
Both handle traffic simultaneously. No failover needed — traffic just shifts.

### Cloud-Managed (recommended)
- AWS ALB / NLB, GCP Load Balancer, Azure LB are inherently HA
- Managed by the provider; no single hardware node; built-in multi-AZ
- Best choice unless you have specific on-prem requirements

---

## 9. Load Balancing in Microservices

In microservices, load balancing exists at two levels:

### External LB (North-South Traffic)
```
Internet → [ API Gateway / External LB ] → backend services
```
Single entry point. Handles client-facing traffic. Usually Layer 7.

### Internal / Service-Mesh LB (East-West Traffic)
```
Service A → [ Envoy Sidecar ] → [ Service B instances ]
```
- Each service has a sidecar proxy (Envoy/Istio) that load-balances outbound calls
- No central LB bottleneck for service-to-service traffic
- Enables per-service routing policies, retries, circuit breaking, mTLS

---

## 10. Failure Modes

| Scenario                        | Impact                             | Mitigation                              |
|---------------------------------|------------------------------------|------------------------------------------|
| LB crash (single instance)      | 100% traffic loss                  | Active-passive or cloud-managed LB       |
| Backend server crash            | Requests in-flight lost, new OK   | Active health check → fast removal       |
| Health check false positive      | Healthy server removed             | Tune thresholds (require 3 failures)     |
| Thundering herd on server add   | New server overwhelmed immediately | Slow start: ramp up traffic gradually    |
| Sticky session server crash     | Session lost for affected users    | Externalize session state to Redis       |
| LB becomes bottleneck (CPU)     | Latency spike, drops               | Scale LB horizontally or use cloud LB    |
