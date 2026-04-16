# System Design Framework — Architecture

## Overview

System design interviews test your ability to take an **ambiguous problem** and produce a **scalable, reliable architecture**. The framework is your structured approach to avoid panic and demonstrate senior-level thinking.

---

## The 4-Step Interview Framework

```
┌─────────────────────────────────────────────────┐
│  Step 1: Clarify Requirements   (~5 min)         │
│  Step 2: Capacity Estimation    (~5 min)         │
│  Step 3: High-Level Design      (~15 min)        │
│  Step 4: Deep Dive & Trade-offs (~20 min)        │
└─────────────────────────────────────────────────┘
```

---

## Step 1: Clarify Requirements

Never start drawing boxes without asking questions. Interviewers give vague prompts intentionally.

### Functional Requirements (What the system does)
Ask:
- **Who** are the users? (consumers, businesses, internal tools)
- **What** are the core features? (write down top 3–5 only)
- **What is out of scope?** (anchor scope early)

**Example — "Design Twitter":**
- Core: post tweets, follow users, view timeline
- Out of scope: ads, DMs, analytics

### Non-Functional Requirements (How the system behaves)
Ask about:
- **Scale** — How many users? Reads vs writes?
- **Latency** — Is it real-time? Acceptable p99?
- **Availability** — 99.9% or 99.99%?
- **Consistency** — Strong vs eventual?
- **Durability** — Can data be lost?

> Senior signal: You proactively ask about consistency vs availability trade-offs, not just "how many users".

---

## Step 2: Capacity Estimation

Rough maths to size your system. Interviewers don't expect exact answers — they want to see **your reasoning process**.

### Key Numbers to Memorize

| Resource       | Approximate Value           |
|----------------|-----------------------------|
| 1 million users × 1 req/day | ~12 QPS |
| 1 million users × 10 req/day | ~120 QPS |
| Read-heavy systems | Read:Write = 100:1 typical |
| Avg tweet size  | ~280 chars ≈ 300 bytes      |
| 1 TB            | 10⁶ MB = 10⁹ KB             |
| SSD read latency | ~0.1 ms                   |
| Network RTT (same DC) | ~0.5 ms              |

### Estimation Formula

```
QPS = (Daily Active Users × Requests per user per day) / 86,400
Storage = QPS × avg_object_size × seconds_per_year × retention_years
Bandwidth = QPS × avg_response_size
```

**Example — URL Shortener:**
- 100M DAU, 1 write per 10 users/day → 10M writes/day → ~115 write QPS
- 10:1 read ratio → 1,150 read QPS
- Each URL = 500 bytes → 10M × 500B = 5 GB/day storage

---

## Step 3: High-Level Design

Sketch the **happy path** first — don't optimize prematurely.

### Standard HLD Template

```
Client
  │
  ▼
[ CDN / DNS ]
  │
  ▼
[ Load Balancer ]
  │
  ▼
[ API Gateway / Auth ]
  │
  ├──► [ Service A ]──► [ DB / Cache ]
  │
  ├──► [ Service B ]──► [ Message Queue ]──► [ Worker ]
  │
  └──► [ Object Storage / Blob Store ]
```

### Components to Always Consider

| Component        | When to include                          |
|------------------|------------------------------------------|
| CDN              | Static assets, geo-distributed reads    |
| Load Balancer    | Any service with >1 instance            |
| Cache (Redis)    | Read-heavy, repeated queries            |
| Message Queue    | Async processing, decoupling            |
| DB (Primary)     | Persistent structured data              |
| Object Store     | Files, images, videos                   |
| Search Index     | Full-text search needs                  |

---

## Step 4: Deep Dive & Trade-offs

Pick **2–3 components** to deep-dive. Show you can reason about choices.

### Trade-off Dimensions

```
Consistency  ◄──────────────► Availability
   Strong                        Eventual

Latency      ◄──────────────► Throughput
  Low (ms)                    High (TPS)

Cost         ◄──────────────► Performance
  Cheap                        Fast
```

### How to Present Trade-offs

Never say "I'll use Redis." Say:

> *"I'll use Redis for caching here because reads are 100x more frequent than writes, and our use case can tolerate slight staleness (eventual consistency). If we needed strong consistency, I'd reconsider or add cache-invalidation logic."*

---

## Real-World Anchors

| Company  | Key System Design Decision                          |
|----------|-----------------------------------------------------|
| Netflix  | Eventual consistency in content metadata; CDN-heavy |
| Uber     | Consistent hashing for driver location routing      |
| WhatsApp | Message fanout via async queues; strong ordering    |
| Twitter  | Pre-computed timelines (push model) for hot users   |
| Amazon   | Cart service uses eventual consistency deliberately |

---

## Common Mistakes to Avoid

- Starting to draw without clarifying requirements
- Designing for 1 billion users when the problem says 100K
- Ignoring failure modes (what happens if DB goes down?)
- Jumping to microservices without justifying the decomposition
- Not mentioning monitoring/observability

---

## Step 5: Failure Paths & Graceful Degradation

A complete HLD shows what happens when things go wrong — not just the happy path.

### Every Component Can Fail — Design For It

```
Normal path:
  Client → LB → API → Cache → DB → Response ✅

Failure paths:
  DB down       → serve from cache (stale OK?) or return 503
  Cache down    → fallback to DB (slower, but correct)
  LB down       → secondary LB via DNS failover
  API crash     → LB detects via health check, routes away
  Queue full    → backpressure: reject new requests or drop with 429
```

### Graceful Degradation Patterns

| Scenario               | Degraded Behavior                              |
|------------------------|------------------------------------------------|
| Recommendation service down | Show most-popular items instead         |
| Search service slow    | Return cached results from last hour           |
| Payment processor down | Show "retry later" instead of 500 error        |
| DB replicas lagging    | Route reads to primary (slower but consistent) |
| Rate limit exceeded    | Return 429 with `Retry-After` header           |

> Senior signal: *"Rather than failing the whole request, I'd return a degraded response — trending content instead of personalized — while the recommendation service recovers. Users see something useful, not an error screen."*

### Circuit Breaker States

```
CLOSED (normal) → failures accumulate → OPEN (reject all) → timeout → HALF-OPEN (probe) → success → CLOSED
```
- **Closed**: requests flow normally
- **Open**: fast-fail without hitting the failing service
- **Half-open**: send 1 probe request — if it succeeds, close the circuit

---

## Step 6: Multi-Region Design

For 99.99%+ availability or global latency requirements, single-region is not enough.

### Active-Passive vs Active-Active

```
Active-Passive (simpler):
  Region A (primary, handles all traffic)
  Region B (standby, replicates data, takes over on failure)
  → Failover time: 1–5 minutes (DNS TTL + health check)
  → Risk: data loss during failover window (depends on replication lag)

Active-Active (harder, better):
  Region A  ←──── Users in US/EU ────►  Region B
  Both regions handle traffic simultaneously
  → No failover needed; traffic shifts on health check
  → Requires: handling write conflicts or single-write-region per entity
```

### Data Replication Across Regions

| Strategy             | Consistency    | Latency Impact            | Use Case                    |
|----------------------|----------------|---------------------------|-----------------------------|
| Async replication    | Eventual       | None on writes            | Social feeds, content       |
| Sync replication     | Strong         | Write latency = cross-region RTT | Payments, banking    |
| Multi-master         | Conflict-prone | Low                       | Collaborative apps (CRDTs)  |

### RTO and RPO

- **RTO (Recovery Time Objective)** — How fast must the system recover? (e.g., "back online in < 5 min")
- **RPO (Recovery Point Objective)** — How much data loss is acceptable? (e.g., "lose no more than 30 seconds of data")

Low RPO requires synchronous replication → higher write latency. Trade-off is explicit.

---

## Step 7: Deployment Strategies

Safe deployments are part of system reliability. Know these for design discussions.

### Blue-Green Deployment

```
Traffic 100% → [ Blue (v1) ]   [ Green (v2, idle) ]
Deploy v2 to Green →
Switch traffic → [ Blue (v1, idle) ]   [ Green (v2, 100%) ]
Rollback = flip switch back to Blue (instant)
```
- **Pro**: instant rollback, zero downtime
- **Con**: requires 2x infrastructure cost during switch

### Canary Deployment

```
Traffic → 95% → [ v1 servers ]
              → 5%  → [ v2 canary ]
Monitor metrics/errors for v2 canary →
Gradually shift: 5% → 20% → 50% → 100%
```
- **Pro**: real traffic validates v2 before full rollout; blast radius is small
- **Con**: need good observability to detect problems in small traffic slice

### Rolling Deployment

```
Replace instances one-by-one:
[v1, v1, v1, v1] → [v2, v1, v1, v1] → [v2, v2, v1, v1] → [v2, v2, v2, v2]
```
- **Pro**: simple, no extra infrastructure
- **Con**: briefly runs mixed versions; harder to roll back

---

## Step 8: Observability Checklist

Every system design should mention these three pillars:

```
Logging   → What happened? (structured JSON logs, correlation ID)
Metrics   → How is the system performing? (QPS, error rate, p99 latency)
Tracing   → Where did this request spend time? (distributed trace across services)
```

### Must-Have Metrics to Mention

| Metric            | Alert Threshold (example)     |
|-------------------|-------------------------------|
| Error rate        | > 1% of requests → page       |
| p99 latency       | > 500ms → investigate         |
| DB connection pool | > 80% utilized → scale         |
| Queue depth       | > 10K messages → add workers  |
| Cache hit rate    | < 85% → investigate miss reason|
