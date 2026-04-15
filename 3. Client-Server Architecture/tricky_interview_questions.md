# Client-Server Architecture — Tricky Interview Questions

*Deep-dive questions with edge cases, follow-ups, and senior-level reasoning.*

---

## Q1: "Your API is stateless and uses JWTs. A user logs out — how do you invalidate their token?"

**The trap:** JWTs are stateless by design — the server has no record of issued tokens. You can't "delete" them.

**Strong answer — options with trade-offs:**

**Option A: Short-lived tokens + refresh tokens**
- Access token TTL = 15 minutes. At logout, discard the refresh token.
- At worst, attacker has 15 minutes. Tradeoff: every auth check involves a clock check, no network call.

**Option B: Token blocklist in Redis**
- On logout, store `jti` (JWT ID claim) in Redis with TTL = token's remaining lifetime
- On every request, check Redis: `if jti in blocklist → reject`
- Tradeoff: now every auth check hits Redis — partial statefulness. But Redis at ~1ms is acceptable.

**Option C: Token versioning in DB**
- Store `token_version` per user in DB. JWT embeds this version at issuance.
- On logout, increment `token_version` in DB. All old tokens immediately invalid.
- Tradeoff: every auth requires a DB lookup. Slower, but solves the problem completely.

**Senior signal:** Present all three, state which you'd choose and why for the given scale.

---

## Q2: "Why can't we just make the database horizontally scalable the same way we do API servers?"

**Strong answer:**
- API servers are stateless — any instance can handle any request independently
- A database is *inherently stateful* — it holds the authoritative version of your data
- **Write conflicts**: if two DB instances both accept writes, which one is correct? You need consensus/locking — expensive
- **Read replicas** are easy to scale horizontally (reads are idempotent)
- **Write scaling** requires either:
  - Vertical scale (bigger primary) — simple, limited
  - Sharding (partition data across primaries) — complex, must route correctly
  - Multi-primary replication (Galera, CockroachDB) — consensus overhead on every write

---

## Q3: "A user reports they see different data when refreshing the page. What could cause this, and is it a bug?"

**Root cause: Eventual consistency between replicas**

```
Write → Primary DB (updated)
         │
         └──► Replica A (lag: 50ms) ← user's request hits here → stale
         └──► Replica B (lag: 200ms)
```

**Is it a bug?** Depends on requirements:
- For a social feed: **No** — slight staleness is acceptable
- For a bank balance: **Yes** — strong consistency required; always read from primary

**Fix options:**
1. **Read-your-writes consistency** — after a write, route that user's next read to the primary for N seconds (set a cookie/flag)
2. **Sticky reads** — always route a user to the same replica
3. **Wait for replica to catch up** — use `WAIT` command in MySQL/PostgreSQL to block until replicas confirm the write
4. **Design choice** — state upfront: "For this social app, eventual consistency on reads is acceptable; for payments, we always read from the primary"

---

## Q4: "You're designing an e-commerce checkout. Should the checkout service be stateless?"

**Nuanced answer — mostly yes, but with careful design:**

The checkout *process* has state (items, shipping, current step), but the server holding it shouldn't.

**Correct approach:**
- Store checkout session in Redis: `checkout:{session_id}` → `{items, address, payment_intent}`
- Each step POSTs idempotently with a checkout session ID
- Server is stateless — retrieves and updates Redis on each call
- If server crashes mid-checkout, any other instance can resume from Redis state

**Why this matters:**
- Checkout flows involve payment providers (Stripe webhooks can hit any server)
- Must be idempotent — retry a payment step shouldn't double-charge

**Follow-up:** *"The payment step fails. How do you retry safely?"*
> Use an idempotency key (UUID from client) on the payment API call. Stripe/payment processors support this — same key = same result, no double charge.

---

## Q5: "Your service handles 10K QPS fine, but spikes to 100K for 30 seconds during a flash sale. How do you handle this?"

**This tests layered resilience, not just "add servers":**

1. **CDN** — absorb repeat reads (product pages, images) at edge. Reduces origin load dramatically.
2. **Rate limiting** — cap requests per user/IP at API gateway. Flash sale bots are real.
3. **Queue the writes** — order placement hits a queue; workers drain at safe pace. Return 202 Accepted immediately.
4. **Pre-scale** — if flash sale is scheduled, manually trigger autoscaling 5 minutes early (autoscaling has 2–5min lag for instance startup)
5. **Circuit breaker** — if inventory DB is at capacity, fail fast with "sold out" rather than letting the queue pile up forever
6. **Serve stale stock counts** — 5-second stale cache of "N items left" is fine; payment step does the real atomic reservation

**Anti-pattern to avoid:** "Just increase DB connection pool" — this just shifts the bottleneck, not removes it.

---

## Q6: "What's wrong with using a single load balancer?"

**It's a single point of failure (SPOF).**

**Strong answer:**
- If the LB goes down, 100% of traffic is lost regardless of how many healthy servers you have
- **Solutions:**
  1. **Active-Passive**: standby LB takes over via VRRP/keepalived if primary fails. Failover time ~1–5s.
  2. **Active-Active**: both LBs receive traffic (via DNS round-robin or anycast). No failover needed; traffic just shifts.
  3. **Cloud-managed LBs** (AWS ALB, GCP LB): built-in HA by the provider; no single hardware node
- **DNS TTL matters**: if LB IP changes on failover, low TTL ensures clients pick up the new address quickly

---

## Q7: "A junior engineer suggests that since we use Redis for sessions, we should store everything users do in Redis. What's your concern?"

**Strong answer:**

Redis is an in-memory store — great for ephemeral/fast-access data, but:
1. **Memory is expensive** — Redis cost scales linearly; disk is 100x cheaper per GB
2. **No persistence guarantee by default** — data can be lost if Redis restarts without RDB/AOF configured
3. **No complex querying** — Redis is key-value; you can't do `SELECT * WHERE user.country = 'IN'`
4. **Not designed for large blobs** — storing images/documents in Redis is an anti-pattern

**Right model:**
- Redis = session data, rate limit counters, short-lived caches, pub/sub
- PostgreSQL/Cassandra = persistent user records, orders, events
- S3 = files, images, video

> "The rule isn't 'put everything in Redis' — it's 'put in Redis only what needs to be fast AND is acceptable to lose or reconstruct.'"

---

## Q8: "A user writes a post and immediately refreshes — they see the old data. Is this a bug? How would you fix it?"

**This is about CQRS and read-your-writes consistency.**

**Root cause:**
```
User writes post → Primary DB (updated immediately)
                        │
                   [ Async replication, 50–200ms lag ]
                        │
User reads feed  → Read Replica or Read Model (stale)
                   └──► Returns old data ← this is the "bug"
```

**Is it a bug?** Depends on what you designed:
- If you explicitly chose eventual consistency → it's expected behavior, not a bug
- If users expect to see their own writes immediately → it's a gap in design

**Fix options:**

1. **Read-your-writes consistency** — after a write, sticky-route that user's reads to the primary for 5 seconds (via a short-lived cookie/flag)
2. **CQRS with synchronous read model update** — the write handler updates both the write DB and the read cache atomically before returning 200
3. **Client-side optimistic update** — client UI shows the new post immediately before the server confirms; correct on save

**CQRS in context:**
```
POST /post → Write Service → Write DB → event on bus
                                             │
                               Read Model Builder (async)
                                             │
                               User's timeline cache (Redis)
                                             │
GET /feed → Read Service ──► Redis timeline  ← may be stale by 50–200ms
```

> *"In a social feed, 200ms staleness on someone else's posts is fine. But for the user's own post, I'd use the read-your-writes pattern — route their next read to primary for 3 seconds so they see their own write immediately."*

---

## Q9: "Your architecture has primary DB + 3 read replicas + Redis + Kafka. Estimate the monthly cost and suggest one cost optimization."

**Tests:** Cost awareness as a production engineering skill.

**Rough AWS estimates (us-east-1):**

| Component              | Spec              | Monthly Cost     |
|------------------------|-------------------|------------------|
| RDS PostgreSQL primary | db.r6g.2xlarge    | ~$900            |
| RDS read replicas ×3   | db.r6g.xlarge     | ~$450 × 3 = $1,350 |
| Redis (ElastiCache)    | cache.r6g.large   | ~$180            |
| Kafka (MSK)            | 3 brokers         | ~$800            |
| EC2 API servers ×5     | t3.medium         | ~$35 × 5 = $175  |
| **Total**              |                   | **~$3,405/month** |

**Optimization to suggest:**

> "The 3 read replicas are a significant cost. I'd check the actual read QPS distribution — if Redis cache hit rate is >90%, the DB read replicas are mostly idle. I could reduce to 1 replica and save ~$900/month. Alternatively, if the read pattern is cacheable, I'd cache more aggressively and right-size the replicas.
>
> For Kafka, if the event volume is low (<10K events/day), SQS would cost ~$0.40/million vs Kafka's ~$800/month fixed — a significant saving at low scale."

**Senior signal:** Don't just design for scale — right-size for actual load, and know when a simpler tool is cheaper.
