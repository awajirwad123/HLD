# System Design Framework — Notes

## The Framework (Cheat Sheet)

```
1. Clarify  →  2. Estimate  →  3. Design  →  4. Deep Dive
   (5 min)       (5 min)        (15 min)       (20 min)
```

---

## Key Definitions

- **Functional requirements** — What the system must do (features)
- **Non-functional requirements** — How it behaves (latency, scale, availability)
- **QPS (Queries Per Second)** — Request throughput the system must handle
- **SLA** — Service Level Agreement; availability promise (e.g., 99.9%)
- **p99 latency** — 99th percentile response time; worst case for 99% of users
- **Trade-off** — Every design decision has advantages AND costs

---

## Numbers Every Engineer Must Know

| Fact                           | Value                    |
|--------------------------------|--------------------------|
| Seconds in a day               | 86,400                   |
| 1M req/day in QPS              | ~12 QPS                  |
| 1B req/day in QPS              | ~11,574 QPS              |
| Typical read:write ratio       | 100:1 (read-heavy)       |
| Avg web request payload        | ~10 KB                   |
| Avg photo size (compressed)    | ~200 KB                  |
| Avg video (1 min, 720p)        | ~50 MB                   |
| Redis throughput               | ~100K ops/sec            |
| PostgreSQL write throughput    | ~5K–10K TPS              |
| Cassandra write throughput     | ~100K+ TPS               |
| SSD read latency               | ~0.1 ms                  |
| Network same-DC                | ~0.5 ms                  |
| Network cross-region           | ~50–150 ms               |

---

## Availability SLA Reference

| SLA     | Downtime per year |
|---------|-------------------|
| 99%     | 3.65 days         |
| 99.9%   | 8.7 hours         |
| 99.99%  | 52 minutes        |
| 99.999% | 5 minutes         |

---

## Component Selection Quick Guide

| Need                             | Use                        |
|----------------------------------|----------------------------|
| Reduce read latency              | Redis / Memcached           |
| Decouple services asynchronously | Kafka / RabbitMQ / SQS     |
| Store unstructured files         | S3 / GCS                   |
| Full-text search                 | Elasticsearch               |
| High write throughput            | Cassandra / DynamoDB        |
| Complex queries + ACID           | PostgreSQL / MySQL          |
| Real-time updates to clients     | WebSockets / SSE            |
| Global content delivery          | CDN (CloudFront, Fastly)    |
| Rate limiting                    | Token bucket + Redis        |

---

## Trade-off Cheat Sheet

| Decision                  | Choose A when...                    | Choose B when...               |
|---------------------------|--------------------------------------|----------------------------------|
| SQL vs NoSQL              | ACID needed, relational data         | Scale > consistency, flexible schema |
| Push vs Pull (fanout)     | Celebrities excluded, low fanout     | Simple reads, high write fanout |
| Sync vs Async comm.       | Immediate response needed            | Fire-and-forget, decoupling     |
| Monolith vs Microservices | Small team, early product            | Independent scaling, large teams|
| Strong vs Eventual cons.  | Financial/critical data              | Social feeds, metrics, counters |

---

## Interview Red Flags to Avoid

- ❌ Starting to design before asking questions
- ❌ "I'll use microservices" without justification
- ❌ Ignoring the non-functional requirements
- ❌ Picking a technology without explaining why
- ❌ Never mentioning failure handling or monitoring
- ❌ Over-engineering for traffic you haven't estimated

---

## Senior-Level Signals to Show

- ✅ Ask about consistency vs availability trade-offs upfront
- ✅ Mention bottlenecks before the interviewer does
- ✅ Propose alternatives and explain why you rejected them
- ✅ Quantify your decisions (e.g., "cache reduces DB load by 95%")
- ✅ Bring up observability (logging, metrics, alerts)

---

## "When to Scale" Thresholds

Use these to justify architecture choices during estimation:

| Trigger                          | Action                                      |
|----------------------------------|---------------------------------------------|
| > 1K write QPS on single DB      | Add write buffer (queue) or read replicas   |
| > 10K read QPS on single DB      | Add Redis cache layer                       |
| > 100K total QPS                 | Consider DB sharding / NoSQL                |
| > 1M DAU                         | CDN becomes mandatory                       |
| > 10M DAU                        | Multi-region deployment needed              |
| Cache hit rate < 80%             | Review key design or increase TTL           |
| p99 latency > 500ms              | Profile and optimize or add cache layer     |
| DB CPU > 70% sustained           | Vertical scale or offload reads to replica  |
| Queue depth growing unbounded    | Add more consumers / workers                |

---

## Cross-Cutting Concerns Checklist

Add these to EVERY design before you're done:

### Observability
- [ ] Structured logging with correlation/trace IDs
- [ ] Metrics: QPS, error rate, p99 latency, saturation
- [ ] Distributed tracing across service boundaries
- [ ] Alerting thresholds defined

### Security
- [ ] Auth: JWT (stateless) or session (Redis-backed)
- [ ] TLS everywhere (HTTPS externally, mTLS between services if needed)
- [ ] Input validation at API boundaries
- [ ] Secrets in env vars / vault, never hardcoded
- [ ] Rate limiting to prevent abuse

### Reliability
- [ ] Health check endpoints on every service
- [ ] Circuit breaker on inter-service calls
- [ ] Retry with exponential backoff + jitter
- [ ] Graceful degradation for non-critical paths
- [ ] No single points of failure

### Cost (mention in senior-level interviews)
- [ ] In-memory (Redis) is ~10x more expensive per GB than disk (PostgreSQL)
- [ ] CDN bandwidth costs vary by provider; usually worth it above 10M DAU
- [ ] Kafka/SQS queue depth → worker cost scales with throughput
- [ ] Multi-region doubles infrastructure cost; justify with SLA requirements

---

## Deployment Strategy Reference

| Strategy    | Rollback Speed  | Infrastructure Cost  | Risk              |
|-------------|-----------------|----------------------|-------------------|
| Blue-Green  | Instant          | 2x during switch     | Low               |
| Canary      | Minutes (reroute)| +5–10% extra         | Very low          |
| Rolling     | Slow (redeploy)  | None extra           | Mixed versions    |
| Recreate    | N/A (downtime)   | None                 | Downtime          |
