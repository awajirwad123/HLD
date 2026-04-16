# Load Balancing — Interview Simulator

*Full mock interview scenarios. Set a 20-minute timer per simulation. Work through it before reading the guided answer.*

---

## Simulation 1: Design a Load Balancing Strategy for a Chat App

### Scenario
> Design the load balancing layer for a real-time chat app (WhatsApp-like). 50M DAU. Each user maintains a persistent WebSocket connection. 10K messages/second average, 100K peak.

---

### Guided Walkthrough

**Why this is tricky:**
Regular round robin fails for WebSockets — the LB doesn't know which server holds a user's connection. If user A is connected to Server-1 and user B is connected to Server-3, a message from A to B must be routed correctly.

**Clarify first:**
- Mobile-only or also web?
- Group chats or 1:1 only?
- Message delivery guarantee (at-least-once OK)?

**Estimate:**
```
50M DAU, assume 10% online at peak = 5M concurrent connections
5M connections / 10 servers = 500K connections/server (feasible with async I/O)
100K msg/sec × 200 bytes = 20 MB/s aggregate throughput
```

**LB strategy decision:**

```
Problem: WebSockets are stateful connections — once established,
         ALL messages for that connection must go to the SAME server.

Option A: IP hash at L4 LB
  Same user IP → always same server
  Problem: NATed users (office network) share one IP → uneven load

Option B: Consistent hashing on user_id at L7 LB
  Route based on user_id header/cookie → always same server for same user
  Problem: server failure remaps some users → brief reconnect storm

Option C (best): L7 LB + Connection Registry
  - LB does NOT need to be sticky — connections register themselves
  - WebSocket connects to ANY server (LB round-robin initial HTTP upgrade)
  - That server stores [user_id → server_id] in Redis
  - Message routing: Server-1 receives msg for user_B → lookup Redis → forward to Server-3
```

**Final architecture:**
```
Client ──► [ L4 NLB ] (static IP, pass-through) ──► [ WebSocket Servers ]
                                                          │
                                               [ Redis Pub/Sub ]
                                               (cross-server message routing)
                                                          │
                                               [ Connection Registry ]
                                               (user_id → server_id)
```

**Health checks:**
- Active: HTTP probe on `/health` alongside WebSocket port
- Unhealthy server: drain WebSocket connections (clients reconnect to healthy server)
- Connection draining: 0s for WebSockets (can't drain; force reconnect)

**Key decision to state:**
> *"I'd use an NLB (Layer 4) for the initial TCP/WebSocket connection — minimal overhead for persistent connections. Message routing between servers uses Redis Pub/Sub as a lightweight message bus. The LB doesn't need to be sticky because the Connection Registry in Redis handles message delivery to the correct server."*

---

## Simulation 2: Debug Load Balancer Behavior in Production

### Scenario
> Your monitoring shows that 1 of your 5 backend servers is receiving 60% of all traffic while the other 4 split the remaining 40%. The LB is configured for round robin. What's wrong?

---

### Answer

Round robin distributes *connections* equally, but connection count ≠ traffic.

**Likely root cause: long-lived connections**

```
Server A: 200 connections (each open for 30 minutes, from API clients)
         → receives continuous traffic for each connection
Servers B–E: receive short HTTP request/response connections
            → connection opens, request processed, connection closes
            → lower average active connections despite equal new-connection rate
```

Round robin assigns equal *new connections* but Server A's connections persist longer → disproportionate ongoing traffic.

**Fix options:**

1. **Switch to Least Connections** — routes new connections to least-loaded server, naturally balancing Server A's long-lived load
2. **Switch to Least Response Time** — AWS ALB default behavior; considers both connections and latency
3. **If it's one specific client** (like a bulk API consumer) — rate limit that client or give it a dedicated connection pool

**Lesser-known cause to mention:**
> "Another cause: if your LB uses per-process worker affinity (some Nginx configs with `worker_connections`), new connections distribute per-worker, not globally. Verify the LB itself is distributing evenly before assuming algorithmic issues."

---

## Simulation 3: Design LB Strategy for a Multi-Tenant SaaS

### Scenario
> You have a SaaS platform with free and paid tiers. Free users can tolerate up to 2s latency; paid users must have < 200ms. Both hit the same API. How do you use load balancing to enforce this SLA difference?

---

### Answer

**This requires traffic-class-aware routing — a Layer 7 LB capability.**

**Strategy: Dedicate backend pools per tier**

```
Client request → [ L7 ALB ]
                     │
           read JWT → extract tier claim
                     │
                     ├── tier=paid  → [ Paid Server Pool (5 servers, high-spec) ]
                     └── tier=free  → [ Free Server Pool (3 servers, lower-spec) ]
```

**Implementation:**
- JWT contains `{"tier": "paid"}` or `{"tier": "free"}`
- ALB listener rule: `if header X-User-Tier == paid → target group: paid-pool`
- API gateway (or Lambda@Edge) verifies JWT, sets `X-User-Tier` header before ALB routing rule evaluates

**Isolation benefits:**
- Free tier traffic spike cannot degrade paid tier performance
- Can apply different rate limits, queue priorities, cache TTLs per tier
- Can independently scale paid pool during peak without scaling free pool

**Health checks per pool:**
- Paid pool: aggressive health check (5s interval, 2 failure threshold) — faster failover
- Free pool: relaxed (30s interval) — tolerate brief slowness

**Cost optimization:**
> "Free tier servers run on spot/preemptible instances — 60–80% cheaper. If a spot instance is reclaimed, free users see brief retry delay (acceptable per SLA). Paid servers run on reserved instances — predictable cost, never interrupted."

**Interview signal:** This shows you understand LB as a *business tier enforcement mechanism*, not just a traffic splitter.
