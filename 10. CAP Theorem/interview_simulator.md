# CAP Theorem — Interview Simulator

## How to use this file
Set a 35-minute timer. Answer each scenario out loud or in writing before reading the model answer. Score yourself using the rubric at the end.

---

## Scenario 1: Classify and Justify the CAP Position for a Banking System

**Prompt:**
"You're designing the core banking ledger for a fintech startup. It handles account balances, transfers, and transaction history. The system operates across 3 AWS availability zones in the same region. Your CTO asks: 'Should we be CP or AP? What does the choice actually mean for our users and our engineers?'"

**Time box:** 12 minutes

---

### Model Answer

**Start with precision — define your terms:**

"Before answering, I want to be clear on what CAP's Consistency means here: it means linearizability — after a confirmed write, every subsequent read returns that value, never a stale one. This is distinct from ACID's Consistency which is about constraint enforcement."

**The business requirement maps directly to CP:**

```
Scenario: Two transfers happen concurrently
  Transfer A: debit account X $100
  Transfer B: debit account X $80
  Account X balance: $120

AP system (stale reads possible):
  Both transfers read balance = $120, both see sufficient funds
  Both succeed → balance goes to -$60 ← OVERDRAFT
  The stale read caused a real financial error

CP system:
  Transfer A reads balance = $120, acquires lock, deducts $100 → $20 remaining
  Transfer B reads balance = $20 → insufficient funds → returns error to user
  → User sees an error (momentary unavailability) vs a financial loss
  → For a bank: 503 on one transfer is infinitely preferable to -$60 balance
```

**What CP means for users:**
```
Normal operation (99.99% of requests):
  No difference — CP overhead is < 10ms per write
  Users see no availability difference

During AZ failure (rare):
  CP behavior: writes may return errors for < 30 seconds during failover
  Users see: "Transfer failed, please retry" ← recoverable, honest
  AP behavior: system accepts writes but may produce inconsistent state
  Users see: inconsistent balance, overdraft, double-debit ← unrecoverable without manual intervention
```

**What CP means for engineers:**
```
Must implement:
  1. Synchronous replication to at least 1 AZ-b standby (RPO = 0)
  2. Automatic failover (Patroni / RDS Multi-AZ) for RTO < 30s
  3. Distributed locking for concurrent transfers (SELECT FOR UPDATE or optimistic locking)
  4. Idempotency for all transaction endpoints (prevent double-submit on retry)
  5. Monitoring: alert on replication lag > 100ms

Do NOT need:
  Conflict resolution logic, compensating transactions for balance inconsistency,
  or application-level merge of concurrent writes
```

**Database recommendation:**
```
PostgreSQL with synchronous replication (RDS Multi-AZ or Patroni):
  - Intra-region: sync to standby AZ-b → RPO = 0
  - Replica in AZ-c: async for read scale (balance reads, transaction history)
  - PgBouncer for connection pooling
  - Write concern = sync → CP
```

**Honest trade-off to state:**
"The cost of CP for a bank is accepting ~30 seconds of write unavailability during a primary failure, approximately 2–3 times per year. The cost of AP is accepting financial errors that require customer service resolution, fraud investigations, and regulatory exposure. For a bank, CP is the only defensible choice."

---

## Scenario 2: Justify the AP Choice for a Social Network's Feed

**Prompt:**
"The engineering team wants to use Cassandra for storing post data in a social news feed. A junior engineer raises concerns: 'Cassandra is AP — users could see stale data! Shouldn't we use a CP database?' How do you respond and defend the AP choice?"

**Time box:** 12 minutes

---

### Model Answer

**Open with reframing the concern:**

"The concern is valid — users CAN see stale data in an AP system. But the real question is: what does stale data mean for a news feed, and what is the cost of choosing CP instead?"

**Characterize the acceptable staleness:**

```
News feed scenario:
  User A posts "Hello world" at 10:00:00.000
  User B refreshes feed at 10:00:00.010 (10ms later)
  
  AP system (CL=ONE): B may not see A's post yet (replication in progress)
  CP system (CL=QUORUM): B is guaranteed to see A's post

  User experience:
    AP: B's feed doesn't show A's post for ~10ms → B refreshes again → sees it
    CP: B always sees A's post immediately
    
  Business impact of AP staleness: None — a 10ms delay on a social post is invisible to users
  Business impact of CP: Cassandra at QUORUM is 2–3x slower per read; at peak (500M DAU),
                         this turns 2ms reads into 5ms reads → real impact on p99 latency
```

**Quantify the scale argument:**

```
500M daily active users
Average: 20 feed reads per day = 10B feed reads/day = ~115K reads/sec
At p99 latency: CP at QUORUM → might require 3x the cluster size to maintain SLA

AP at CL=ONE:
  - Any of 3 replicas responds immediately
  - Reads consistently < 2ms
  - Occasional ~10ms staleness: imperceptible to users

CP cost: millions of dollars in additional infrastructure to maintain the same SLA
AP benefit: same latency, 3x cluster efficiency, zero user-visible downside
```

**Address the junior engineer's concern directly:**

"You're right that AP allows stale reads. But CAP's Consistency means linearizability — not 'posts show up instantly.' A news feed has no transactional requirement across posts. Whether you see a post at time T or T+10ms doesn't create any invalid state in the system. Compare this to a bank balance: a 10ms stale read there could result in double-spending. Context determines whether AP is acceptable."

**When AP would NOT be acceptable for feeds:**

```
If the product requires:
  1. Guaranteed sequential story ordering (read your latest post immediately after posting)
     → Fix: optimistic UI update (show the post locally before server confirms)
     → Or: route the author's read-after-write to primary for 2 seconds

  2. Real-time counts (likes must be accurate to the millisecond)
     → This is a CQRS problem, not a CAP problem
     → Use Redis sorted sets for real-time counts; Cassandra for durable storage
```

**Summary statement:**

"Cassandra AP at CL=ONE is the right choice for a news feed because: (1) the business has no correctness requirement for millisecond-level freshness on social content, (2) at scale, the latency and infrastructure cost of CP would be significant and provide no user benefit, (3) the one place we need read-your-writes (user's own posts) can be solved with optimistic UI without changing the database choice."

---

## Scenario 3: Design a Distributed Rate Limiter — CAP Implications

**Prompt:**
"You're designing a distributed rate limiter for an API gateway. The limiter must prevent a user from making more than 100 requests per minute. The system runs across 3 nodes. Analyze the CAP implications of your approach."

**Time box:** 11 minutes

---

### Model Answer

**Frame the CAP trade-off for this specific system:**

```
What does "Consistency" mean here?
  → The counter must accurately reflect the true total request count across all nodes
  → If Node A and Node B both think the user has made 50 requests, but the true count
     (across all nodes) is 100, the user is over-limit but won't be blocked → CP violated

What does "Availability" mean here?
  → The rate limiter must respond for every incoming request (never block requests
     because the counter store is unavailable)
```

**Option A: Redis single primary (CP-adjacent)**

```
All 3 API gateway nodes read/write a single Redis counter:
  INCR rate:user_123
  → Atomic counter: always consistent
  → All nodes see the same count

CAP position: CP
  → If Redis primary fails, reads/writes to the counter fail → must choose:
    a) Fail open (allow all traffic during Redis failure) ← AP-like fallback
    b) Fail closed (block all traffic) ← extreme CP, terrible UX

Trade-offs:
  + Exact rate limiting: 100 requests = 100, always
  + Simple implementation
  - Redis primary is a SPOF (use Sentinel or Cluster for HA)
  - Cross-node latency: each request requires a Redis round trip (~1ms)
```

**Option B: Per-node counting (AP)**

```
Each API node maintains its own local counter:
  Node A: user_123 = 30 requests
  Node B: user_123 = 40 requests
  Node C: user_123 = 35 requests
  Total: 105 → over limit, but each node thinks the user is under limit

CAP position: AP
  → Always available (no coordination), but inconsistent counts
  → User can make ~300 requests (100 per node) before being limited
  → Rate limit is approximate

Use when: exact limiting is less important than zero overhead
Example: CDN rate limiting where ±3x is acceptable
```

**Option C: Sliding window with Redis + fallback (recommended)**

```python
import redis
import time

r = redis.Redis(host="redis-primary", port=6379)

WINDOW_SECONDS = 60
MAX_REQUESTS = 100

def is_allowed(user_id: str) -> bool:
    key = f"ratelimit:{user_id}"
    now = time.time()
    pipe = r.pipeline()

    try:
        pipe.zremrangebyscore(key, 0, now - WINDOW_SECONDS)  # remove old entries
        pipe.zadd(key, {str(now): now})                       # add current request
        pipe.zcard(key)                                        # count in window
        pipe.expire(key, WINDOW_SECONDS + 1)
        results = pipe.execute()

        count = results[2]
        return count <= MAX_REQUESTS

    except redis.RedisError:
        # Fail open: if Redis is unreachable, allow the request
        # Log the failure; alert the on-call engineer
        # Brief Redis unavailability should not block all API traffic
        return True  # ← AP fallback behavior
```

**CAP analysis of the recommended approach:**

```
Normal operation: CP-like (Redis atomic ops give exact counts)
During Redis failure: AP fallback (fail open, allow traffic)

This is a deliberate hybrid:
  → Choose C (exact limiting) during normal operation
  → Choose A (no blocking) during partition/failure
  → The business rationale: a brief window of unenforced rate limits
    during Redis outage is less bad than blocking all API traffic
  → Rate abuse during outage: detectable after the fact (logs), can be blocked retroactively
```

**State this clearly in the interview:**

"For a rate limiter, I'd choose CP for accuracy under normal conditions, but explicitly fail open during partition/failure events. This is a business decision: the cost of allowing a brief window of excess traffic is lower than the cost of blocking legitimate users during an infrastructure incident. This is documented in the system design so operators know what to expect."

---

## Self-Assessment Scorecard

| Criterion                                                              | Score /5 |
|------------------------------------------------------------------------|----------|
| Defined CAP terms precisely (especially C = linearizability, not ACID) |          |
| Stated that P is mandatory; real choice is C vs A during partition     |          |
| Correctly classified systems without just memorizing labels            |          |
| Applied CAP to a concrete user-visible impact (not abstract)           |          |
| Mentioned tunable consistency (Cassandra, DynamoDB) as a spectrum      |          |
| Referenced PACELC or normal-operation trade-offs                       |          |
| Proposed practical mitigations for AP consistency gaps                 |          |
| Did NOT treat CAP as binary — acknowledged hybrid/context-dependent    |          |

**Total: /40**

| Score | Interpretation                                                          |
|-------|-------------------------------------------------------------------------|
| 35–40 | Excellent — ready to discuss CAP at principal/staff engineer level      |
| 25–34 | Good — review architecture.md misconceptions table and PACELC section   |
| 15–24 | Revisit the CP vs AP section; redo Scenario 1                           |
| < 15  | Full re-read of architecture.md, then complete the CP/AP simulation     |
