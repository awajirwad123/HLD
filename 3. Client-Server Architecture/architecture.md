# Client-Server Architecture — Architecture

## Overview

Client-Server is the foundational model of every distributed system. Every HLD you draw is an evolution of this pattern. Understanding its properties — especially stateless vs stateful, and horizontal vs vertical scaling — is mandatory.

---

## 1. The Request-Response Model

The most fundamental communication pattern:

```
Client                          Server
  │                               │
  │── HTTP Request ──────────────►│
  │   (GET /api/feed)             │
  │                               │── query DB
  │                               │── apply business logic
  │◄── HTTP Response ─────────────│
  │   (200 OK, JSON body)         │
```

### Properties
- **Synchronous** — client blocks and waits for the response
- **Stateless by design** (with HTTP) — each request is independent
- **Simple to reason about** — easy to debug, retry, and scale

### When it breaks down
- Long-running tasks (file processing, video encoding) — client times out
- Real-time updates (chat, notifications) — polling is inefficient
- High fan-out writes — one request triggers thousands of downstream operations

---

## 2. Stateless vs Stateful Servers

This is one of the most important design decisions in any system.

### Stateless Server

```
                 ┌──────────────────────────────────────┐
                 │         Load Balancer                 │
                 └──────────────────────────────────────┘
                    │              │              │
                    ▼              ▼              ▼
              [ Server A ]   [ Server B ]   [ Server C ]
              (no local      (no local      (no local
               state)         state)         state)
                    │              │              │
                    └──────────────┴──────────────┘
                                   │
                              [ Redis / DB ]
                          (shared state store)
```

**Key property:** Any server can handle any request — no "sticky sessions" needed.

**How state is passed:**
- **JWT tokens** — client carries identity/auth state in every request header
- **Cookies with session ID** → session data stored in Redis, not on server
- **Request payload** — client sends all context needed (pagination token, filters)

**Benefits:**
- Horizontal scaling is trivial — add/remove instances freely
- No server affinity needed at load balancer
- Easy auto-scaling and rolling deployments

### Stateful Server

```
Client A ──────────────────────► [ Server A ] (has A's WebSocket connection)
Client B ──────────────────────► [ Server B ] (has B's connection)
```

Server holds state in memory (open connections, in-flight transactions, game state).

**Problems:**
- Load balancer must route same client to same server (**sticky sessions**)
- Server crash = all its clients lose state
- Hard to scale — can't freely redistribute load

**When stateful is unavoidable:**
- WebSocket connections (connection is inherently tied to a server)
- Gaming servers (low-latency state must be in-memory)
- Leader nodes in consensus clusters

**Mitigation:** Use a stateless API layer + push state externally. E.g., WhatsApp routers are stateless — presence/message state is in Mnesia/distributed store.

---

## 3. Horizontal vs Vertical Scaling

### Vertical Scaling (Scale Up)

```
Before:          After:
┌────────┐       ┌────────────┐
│ 4 CPU  │  ──►  │  32 CPU    │
│ 8GB    │       │  128GB RAM │
│ app    │       │  app       │
└────────┘       └────────────┘
```

- Simple — no code changes needed
- Hard upper limit (biggest machine available)
- Single point of failure
- Expensive at high specs
- **Good for:** databases (vertical scale before sharding), early-stage products

### Horizontal Scaling (Scale Out)

```
Before:               After:
┌────────┐            ┌────────┐ ┌────────┐ ┌────────┐
│ app    │  ──►  LB → │ app    │ │ app    │ │ app    │
└────────┘            └────────┘ └────────┘ └────────┘
```

- No theoretical limit
- Requires stateless design
- Increases operational complexity (more instances to monitor/deploy)
- **Good for:** API servers, read replicas, worker pools

### Interview Decision

```
Component          Prefer Scale Up?   Prefer Scale Out?
─────────────────────────────────────────────────────
API / App Server        No                 Yes ✅
Relational DB (write)   Yes ✅             Complex (sharding)
Cache (Redis)           Yes (to a point)   Yes (clustering)
Message Queue           No                 Yes ✅
Search (ES)             No                 Yes ✅
```

---

## 4. The Evolution: 1-Tier → N-Tier Architectures

### 1-Tier (Monolith + Embedded DB)
```
[ Browser → Monolith App → Embedded SQLite ]
```
Suitable only for local tools or tiny prototypes.

### 2-Tier (Client + Server+DB)
```
[ Client App ] ──► [ Server + DB ]
```
Simple. DB and logic on same host. Not scalable.

### 3-Tier (Presentation + Logic + Data)
```
[ Client ] ──► [ App Server ] ──► [ Database ]
```
Standard web app architecture. App server is stateless and scalable.

### N-Tier / Microservices
```
[ Client ]
   │
[ API Gateway ]
   │
   ├──► [ Auth Service ] ──► [ User DB ]
   ├──► [ Feed Service ] ──► [ Feed DB + Cache ]
   └──► [ Media Service ] ──► [ Object Store ]
```
Each service independently deployable and scalable.

---

## 5. Common Client-Server Patterns in HLD

### Pattern 1: Read-Heavy with Cache

```
Client ──► LB ──► API Server ──► [ Redis ] ──► (miss) ──► [ DB ]
                                    │
                               (cache hit) ──► return fast
```

### Pattern 2: Write-Heavy with Queue

```
Client ──► LB ──► API Server ──► [ Message Queue ] ──► [ Worker ] ──► [ DB ]
                     │
                  return 202 Accepted (async)
```

### Pattern 3: Fan-Out on Write

```
User posts tweet
  │
  ▼
Write Service
  │
  ├──► DB (source of truth)
  └──► Fan-out to followers' timeline caches (async via queue)
```

---

## 6. Failure Modes in Client-Server

| Failure            | Symptom                          | Mitigation                       |
|--------------------|----------------------------------|----------------------------------|
| Server crash       | 503 Service Unavailable          | Multiple instances + health checks |
| DB unavailable     | 500 errors                       | Circuit breaker + fallback cache  |
| Network timeout    | Client waits forever             | Set timeout + retry with backoff  |
| Thundering herd    | All clients retry simultaneously | Jitter in retry, queue buffer     |
| Single LB failure  | All traffic lost                 | Active-passive LB pair, DNS failover |
