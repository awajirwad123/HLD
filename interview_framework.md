# System Design Interview Framework

A universal script for any system design interview. Work through these phases in order — every interview covers the same ground, just at different depths.

---

## The 45-Minute Breakdown

| Phase | Time | Goal |
|---|---|---|
| 1. Requirements | 5 min | Scope the problem clearly |
| 2. Estimates | 5 min | Establish scale and constraints |
| 3. API Design | 5 min | Define the interface |
| 4. Data Model | 5 min | Define storage schema |
| 5. High-Level Design | 15 min | Core architecture + components |
| 6. Deep Dive | 10 min | Interviewer picks one area |
| 7. Trade-offs & Wrap-Up | 5 min | Show maturity |

---

## Phase 1: Requirements (5 min)

**Goal:** Eliminate ambiguity. Never assume — ask.

### Functional Requirements
Ask what the system *must* do (write → store, use case 1, 2, 3):
- "What are the core user-facing features we're building?"
- "Any out-of-scope features I should explicitly skip?"
- "Read-heavy or write-heavy?"

### Non-Functional Requirements
Ask about scale and quality attributes:
- "How many users / DAU?"
- "Expected QPS (read and write)?"
- "Data retention? How much storage?"
- "What are the latency SLAs?" (e.g., P99 < 200ms?)
- "Consistency requirements? Can we tolerate eventual consistency?"
- "Availability target? 99.9%? 99.99%?"
- "Global or single-region?"

### Clarifying Questions by Topic
| System type | Key questions |
|---|---|
| Social feed | "Does the feed rank posts or is it chronological?" |
| Search | "Fuzzy matching, typo tolerance, or exact?" |
| Messaging | "One-to-one, group? Message retention?" |
| Payments | "Idempotent payments? Fraud detection in-scope?" |
| File storage | "What file sizes? Is streaming required?" |
| Rate limiter | "Per user, per IP, or per endpoint?" |

> **Tip:** Write the requirements down on the whiteboard. This signals discipline and gives you an artifact to refer back to.

---

## Phase 2: Capacity Estimation (5 min)

**Goal:** Justify your architecture choices with numbers. Round aggressively — precision is not the point.

### Quick Formulas

```
QPS = DAU × requests_per_day / 86400
Write QPS = QPS × write_fraction
Read QPS = QPS × read_fraction

Storage per day  = write_QPS × payload_size_bytes × 86400
Storage per year = storage_per_day × 365

Bandwidth in  = write_QPS × payload_size
Bandwidth out = read_QPS × response_size
```

### Common Benchmarks to Know

| Resource | Rule of thumb |
|---|---|
| SQL server (simple reads) | 1K–5K QPS per node |
| NoSQL Cassandra | 10K–50K writes/s per node |
| Redis | 100K–1M ops/s per node |
| CDN edge | 100K–1M RPS per POP |
| Typical user | 1–10 requests/day for most apps |
| Image size | ~300 KB |
| Short video (1 min) | ~50 MB |
| Log event | ~1 KB |
| Database row | 100 bytes–1 KB |

### Example Walk-Through (Twitter-scale)

```
500M DAU, each user reads feed 10× and posts 0.1×/day
Write QPS: 500M × 0.1 / 86400 ≈ 580/s
Read QPS:  500M × 10 / 86400  ≈ 58,000/s

Storage (tweets, 280 chars ≈ 300B + metadata ≈ 1KB):
  580 writes/s × 1KB × 86400s ≈ 50 GB/day → ~18 TB/year

Media (50% tweets have images):
  290 × 300KB × 86400 ≈ 7 TB/day → ~2.5 PB/year
```

> **Tip:** Don't agonise over exact numbers. Say "I'll round to 1K writes/s to size the system." Interviewers reward correct reasoning, not precision.

---

## Phase 3: API Design (5 min)

**Goal:** Define the exact contract — what inputs and outputs matter.

### Pattern
Design 3–5 core endpoints or methods. For each, state:
- Method (GET/POST/PUT/DELETE) or RPC name
- Parameters (required vs optional)
- Response structure
- Any auth or rate limit headers

### Example (URL shortener)

```http
# Create short URL
POST /api/v1/urls
Body: { "long_url": "https://...", "custom_alias": "abc123" (opt), "expire_at": "2025-01-01" (opt) }
Response 201: { "short_url": "https://short.io/abc123", "id": "uuid" }

# Redirect
GET /:alias
Response 301: Location: <long_url>

# Get stats (authenticated)
GET /api/v1/urls/:id/stats
Response 200: { "clicks": 1234, "unique_visitors": 890, "top_countries": [...] }

# Delete (authenticated owner)
DELETE /api/v1/urls/:id
Response 204
```

### Common API Design Flags
- **Idempotency key** for payments and mutations: `Idempotency-Key: uuid` header
- **Pagination** for list endpoints: cursor-based (`?cursor=abc&limit=20`) over page/offset for large datasets
- **Versioning**: `/api/v1/` in path
- **Rate limit headers**: `X-RateLimit-Remaining`, `X-RateLimit-Reset`

---

## Phase 4: Data Model (5 min)

**Goal:** Define what you store, where, and in what shape.

### Framework
For each entity, decide:
1. **What** data (fields, types)
2. **Where** to store it (SQL / NoSQL / blob / cache)
3. **How** to access it (primary key, secondary index, query patterns)

### Storage Decision Matrix

| Requirement | Storage |
|---|---|
| ACID transactions, complex joins | PostgreSQL / MySQL |
| High write throughput, horizontal scale | Cassandra / DynamoDB |
| Flexible schema, document model | MongoDB |
| Low-latency cache, leaderboards | Redis |
| Full-text search | Elasticsearch |
| Immutable events / audit log | Kafka + object storage |
| Binary blobs (images, videos) | S3 / GCS + CDN |
| Time-series metrics | Prometheus / InfluxDB |

### Example (Instagram feed)

```sql
-- Users
users: id (UUID PK), username, email, created_at

-- Posts
posts: id (UUID PK), user_id FK, image_url, caption, created_at

-- Follows
follows: follower_id FK, followee_id FK, PRIMARY KEY (follower_id, followee_id)
INDEX: (followee_id)  -- "who follows me"

-- Feed (pre-computed, Redis sorted set)
ZSET feed:{user_id}  →  score=timestamp  member=post_id
```

> **Tip:** Mention your access patterns explicitly. "I'm designing this schema to support fetching a user's last 20 posts efficiently — hence the post_id + user_id composite index."

---

## Phase 5: High-Level Design (15 min)

**Goal:** Draw the core architecture. Hit every major component. Be systematic.

### Standard Component Checklist

```
[ ] Client (web / mobile)
[ ] CDN (static assets, geographic distribution)
[ ] Load balancer / API gateway
[ ] Application servers (microservices or monolith)
[ ] Cache layer (Redis, Memcached)
[ ] Primary database
[ ] Message queue / event bus (Kafka, SQS) — if async processing needed
[ ] Background workers / job queue
[ ] Object storage (S3) — if binary data
[ ] Search index — if full-text search
[ ] Monitoring / logging (brief mention)
```

### Drawing Order
1. Client → Load Balancer → Application server
2. Application server → Primary DB
3. Add cache in front of DB
4. Add async path (queue + workers) if needed
5. Add CDN + object store for media
6. Add search if needed

### Language to Use
- "I'd put a Redis cache in front of the DB to absorb the **58K read QPS** we estimated."
- "For writes, I'd use Kafka to decouple the upload pipeline from the notification system."
- "I'll use S3 for binary storage and CloudFront as the CDN to serve with < 20ms latency globally."

---

## Phase 6: Deep Dive (10 min)

The interviewer will pick one area: "Tell me more about how the feed generation works" / "How do you handle hot users?" / "Walk me through the write path."

### Common Deep Dive Areas

| Area | What to cover |
|---|---|
| Caching | Eviction policy, cache invalidation, stampede prevention |
| Database | Indexing strategy, sharding approach, replication lag |
| Feed generation | Fan-out on write vs read, celebrity problem |
| Search | Inverted index, tokenisation, ranking |
| Rate limiting | Algorithm (token bucket, sliding window), storage (Redis), distributed |
| Messaging | At-least-once vs exactly-once, consumer groups, offset commits |
| Failure modes | What happens when DB is down? What if queue is full? |

### Deep Dive Framework
For any component: **Normal path → Failure path → Trade-offs**

Example (caching):
- Normal path: check Redis, hit → return, miss → query DB, write to cache with TTL
- Failure path: Redis down → fallback to DB directly (circuit breaker), alert on elevated DB QPS
- Trade-offs: Write-through (consistency) vs write-behind (performance), TTL (too short = too many misses, too long = stale data)

---

## Phase 7: Trade-offs & Wrap-Up (5 min)

**Goal:** Show engineering maturity. No design is perfect — acknowledge it.

### Standard Trade-offs to Address
- **Consistency vs Availability:** Did you pick eventual consistency anywhere? Why?
- **Latency vs throughput:** Did you batch writes? Async any operations?
- **Complexity vs simplicity:** Did you introduce Kafka? Justify it.
- **Cost vs performance:** Is your CDN / cache strategy cost-proportionate to the benefit?

### Future Improvements Script
"Given more time, I would also add:
- [Multi-region replication] for global availability
- [Read replicas] to separate read and write load
- [Backpressure on the queue] to handle traffic spikes gracefully
- [Graceful degradation] — if the recommendation system is down, serve chronological feed as fallback"

---

## Interview Red Flags to Avoid

| Mistake | Fix |
|---|---|
| Jump straight to design without requirements | Always spend 5 min on Phase 1 |
| Use average latency in estimates | Use P99 / P95 |
| Design without mentioning failure modes | "What happens when X is down?" |
| Pick a specific technology without justification | "I choose Cassandra because our access pattern is write-heavy with known primary keys — no JOINS needed" |
| Overengineer from the start | Start simple, add complexity when the need is justified by scale |
| Forget about data size | Always estimate storage alongside compute |
| Ignore security | Mention auth (JWT/OAuth), HTTPS, rate limiting, input validation at minimum |

---

## Topic Cross-Reference

When you need to go deeper on any component, find the matching topic:

| Component | Topic # |
|---|---|
| Load balancing | T2 — Load Balancing |
| Caching | T3 — Caching |
| Database scaling | T4 — Database Replication, T8 — Data Sharding |
| Message queue | T6 — Message Queues |
| CDN | T9 — CDN |
| Rate limiting | T18 — Rate Limiting |
| Search | T21 — Search Index |
| Distributed lock | T22 — Distributed Locking |
| Consensus / leader election | T13 — Consensus Algorithms |
| Observability | T25 — Observability |
| Service discovery | T26 — Service Discovery |
| File storage | T36 — File Storage System |
