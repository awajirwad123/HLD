# System Design Framework — Tricky Interview Questions

*Deep-dive questions with edge cases and follow-ups. These separate good candidates from great ones.*

---

## Q1: "Just walk me through how you'd approach designing any system."

**What they're really testing:** Do you have a repeatable, structured process — or do you freestyle?

**Strong answer:**
> "I follow a 4-step framework: First, I clarify functional and non-functional requirements — I never assume scale or constraints. Second, I do a rough capacity estimation to size storage, QPS, and bandwidth. Third, I sketch the high-level design covering the happy path. Fourth, I deep-dive into 2–3 critical components and discuss trade-offs. Throughout, I call out failure modes and observability needs."

**Follow-up trap:** *"What if the interviewer says 'just start drawing'?"*
> Still ask 2–3 targeted questions. Say: "I'll be quick — just need to know the scale and top 3 features so I design the right thing."

---

## Q2: "How do you decide what to deep-dive into?"

**Trap:** Candidates deep-dive into whatever they know best, not whatever matters most.

**Strong answer:**
> "I prioritize the bottleneck or the hardest constraint. For a URL shortener, the write path is trivial but the redirect read path handles 100x more traffic — so I'd deep-dive the caching strategy. For a chat system, message ordering and delivery guarantees are the hard part — that's where I'd spend time."

---

## Q3: "Your system needs 99.99% availability. What does that mean architecturally?"

**Answer:**
- 99.99% = ~52 minutes downtime/year
- No single point of failure anywhere in the stack
- Every component must be replicated: DB (primary + replicas), cache clusters, multiple load balancers, multi-AZ deployments
- Circuit breakers on inter-service calls
- Health checks + auto-recovery

**Follow-up:** *"How do you test for this?"*
> Chaos engineering (Netflix's Chaos Monkey) — intentionally kill instances in production to verify recovery.

---

## Q4: "A candidate always says 'just add more servers.' Why is that wrong?"

**Answer:**
Horizontal scaling solves stateless compute but doesn't solve:
- Database write bottlenecks (you can't infinitely scale a single primary)
- Network bandwidth limits
- Stateful services (sessions, leader election)
- Data consistency under partitioning

Senior answer: "Adding servers is one tool. The actual bottleneck is usually the DB or the network I/O — you need to design around those constraints specifically."

---

## Q5: "How would you handle a spike in traffic that's 10x your normal QPS?"

**What they're testing:** Resilience, not just scale.

**Strong answer — layers of defense:**
1. **CDN** absorbs static + cached requests at the edge (never hits your servers)
2. **Rate limiting** at API gateway prevents abuse from amplifying the spike
3. **Autoscaling** adds instances within minutes (not seconds)
4. **Message queue** buffers write traffic — workers drain at their own pace
5. **Circuit breaker** sheds load gracefully if downstream services can't keep up
6. **Grace degradation** — serve stale cached data rather than error under load

---

## Q6: "You estimated 600 QPS, but the system is getting 60,000 QPS. What happened, and how do you fix it?"

**Estimation mistake traps:**
- Forgot peak vs average (peak can be 10–50x average)
- Underestimated concurrent users vs total DAU
- Didn't account for retry storms (clients retrying on failure amplify load)

**Fix approach:**
1. Add caching in front of the DB (reduces actual DB QPS by 90%+)
2. Horizontal scale the stateless API layer immediately
3. Introduce a write buffer (queue) to protect the DB
4. Profile: is the bottleneck CPU, network, or DB I/O?

---

## Q7: "When would you NOT use microservices?"

**Trap:** Candidates equate "senior" with "always microservices."

**Strong answer:**
> "Microservices trade deployment independence for operational complexity — you now need service discovery, distributed tracing, network latency between services, and complex deployments. I'd avoid them if: the team is small (<10 engineers), the domain isn't well understood yet (boundaries will change), or the system doesn't have genuinely independent scaling requirements. Start with a well-organized monolith and extract services only when there's a clear, concrete reason."

---

## Q8: "How do you handle the case where you get the requirements wrong mid-design?"

**What they're testing:** Adaptability and communication.

**Strong answer:**
> "In a real interview, I'd acknowledge it explicitly: 'Good point — if we need strong consistency here, my earlier cache-aside approach breaks down. Let me revise the write path.' Pivoting cleanly with justification scores higher than rigidly sticking to a wrong design. In real systems, requirements change — what matters is that trade-offs are re-evaluated explicitly."

---

## Q9: "What's the difference between availability and reliability?"

- **Availability** = fraction of time the system is accessible and operational (SLA %)
- **Reliability** = probability the system performs correctly over a given time period without failure
- High availability doesn't guarantee reliability: a system can be "up" but returning wrong results

**Real-world:** Amazon's shopping cart uses eventual consistency intentionally — it's always available (high availability) but a cart might briefly be inconsistent (not fully reliable in the strict sense).

---

## Q10: "A junior engineer says 'we can solve everything with a bigger database.' How do you respond?"

**Strong senior answer:**
> "A single database is always a single point of failure and a write bottleneck. As QPS grows, even the fastest RDBMS will cap out on writes. The solution isn't a bigger machine — it's redesigning the data access pattern: introduce caching to reduce reads, a write buffer to smooth write spikes, read replicas to distribute reads, and sharding or a horizontally scalable store if the data access pattern warrants it. The database should receive only what it absolutely must persist."

---

## Q11: "Your interview design uses Redis, Kafka, and 3 database clusters. The interviewer asks: what does this cost?"

**What they're testing:** Can you reason about cost as a first-class constraint, not an afterthought?

**Strong answer — framework:**
> "Good question. Cost is a constraint I should have stated upfront. Let me reason through it."

| Component         | Rough Monthly Cost (AWS)         |
|-------------------|----------------------------------|
| Redis (r6g.large) | ~$120/node; cluster of 3 = $360  |
| Kafka (MSK)       | ~$500–$2000 depending on throughput |
| PostgreSQL (RDS)  | ~$200 (primary) + $100 (replica) |
| EC2 API servers   | ~$50–100/instance × 5 = $500     |
| S3 (1 TB)         | ~$23/month                       |
| CDN (100TB egress)| ~$800–$1200/month                |

**Trade-off answer:**
> "Redis is ~10x more expensive per GB than disk. I'd only put data in Redis that's accessed > 100x/day and can tolerate loss. For infrequently accessed data, I'd use PostgreSQL with a caching layer only when QPS justifies it. Kafka has a high fixed cost — for < 10K events/day, SQS would be cheaper. Multi-region doubles infra cost; I'd only justify it if the SLA requires 99.99%+ uptime."

**Senior signal:** Cost awareness = production maturity. Every architecture decision has a $ implication.

---

## Q12: "What is CQRS and when would you use it?"

**CQRS = Command Query Responsibility Segregation**

Split your data model into two sides:
- **Command side** — handles writes (creates, updates, deletes) → optimized for writes
- **Query side** — handles reads → optimized for reads (can be denormalized, different DB)

```
Client
  │
  ├──── Write → [ Command Handler ] → [ Write DB (normalized) ]
  │                                          │
  │                                    [ Event Bus ]
  │                                          │
  │                                   [ Read Model Updater ]
  │                                          │
  └──── Read  → [ Query Handler ]  → [ Read DB (denormalized) ]
```

**When to use CQRS:**
- Read and write workloads have very different scale or shape requirements
- Writes are complex domain operations; reads are simple projections
- System needs multiple read models (different views of the same data)

**Real-world example:**
> Twitter's timeline: writes (new tweet) go to normalized storage. A separate fanout service builds denormalized "read models" (pre-built timelines per user in Redis). Read and write paths are completely independent.

**When NOT to use CQRS:**
- Simple CRUD apps — it adds complexity with no benefit
- Small teams — operational overhead of two models is significant
- Eventual consistency between write and read model is unacceptable

**Key caveat:** CQRS often comes with **eventual consistency** — the read model lags the write model by milliseconds to seconds. Always state this trade-off explicitly.
