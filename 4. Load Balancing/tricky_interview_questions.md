# Load Balancing — Tricky Interview Questions

*Deep-dive questions that test real understanding beyond "add a load balancer."*

---

## Q1: "You have a load balancer in front of 10 servers. One server is slow (not down). How does your LB handle it?"

**Trap:** Candidates say "health checks detect it." Health checks only detect server crashes, not slowness.

**Strong answer:**

A slow server (high latency but returning 200) is **invisible to health checks**. The LB keeps routing to it, and it becomes a latency hole.

**Solutions:**

1. **Least Connections** — slow server accumulates high connection count → LB naturally routes less to it
2. **Least Response Time** — explicitly measures response time per backend; slow server gets fewer requests
3. **Circuit breaker at client** — service mesh (Envoy) can detect p99 latency outliers and eject slow pods from the pool ("outlier detection")
4. **Adaptive concurrency** — limit the concurrent requests to any single backend; slow one fills up its limit faster

**Envoy outlier detection config (mention in interview):**
```yaml
outlier_detection:
  consecutive_5xx: 5
  interval: 10s
  base_ejection_time: 30s
  max_ejection_percent: 50  # never eject more than 50% of the pool
```

**Senior signal:**
> *"Health checks detect binary failures. For performance degradation, I'd pair least-response-time LB with outlier detection in the service mesh to automatically eject persistently slow pods from routing."*

---

## Q2: "A user's request hits Server A, updates their cart, then the next request hits Server B which has no knowledge of the cart. What went wrong and what are your options?"

**Root cause:** Session state stored in Server A's local memory — stateful server, no sticky session configured.

**This is a trap** — the interviewer wants to see if you know the *right* fix vs the *easy* fix.

**Option A (wrong fix): Enable sticky sessions**
- Routes user back to Server A forever
- Server A becomes a per-user bottleneck
- If Server A crashes → user loses their cart anyway

**Option B (correct fix): Externalize session state**
```
Any server:  reads/writes cart state from Redis
Redis:       { "cart:user123": {"items": [...], "updated_at": ...} }
```
- Any server can handle any request
- Cart survives server crashes
- True horizontal scaling

**Option C: Client-side state**
- Cart stored in browser (localStorage or encrypted cookie)
- No server storage needed
- Limit: data size (cookie max 4KB), security (client can tamper → must verify server-side)

**Interview answer framework:**
> *"Sticky sessions are a band-aid. The root issue is mutable state in server memory. I'd move cart state to Redis with a session ID in a signed cookie — TTL matches checkout session lifetime (30 min). Any server reads/writes Redis; stickiness is no longer needed."*

---

## Q3: "You're adding a fifth server to your pool. The LB is using IP hash routing. What happens to existing users?"

**Strong answer:**

IP hash: `hash(client_IP) % server_count`

When server_count changes from 4 to 5:
- Nearly **all** clients get remapped to different servers
- `hash(IP) % 4` ≠ `hash(IP) % 5` for most IPs
- If those servers held session state → **mass session loss**
- Even with Redis sessions, there's a burst of cache misses as all users get "new" servers

**Why this matters:**
- IP hash is fragile to pool size changes — not suitable for dynamic auto-scaling
- Consistent hashing solves this: adding 1 server remaps only ~1/N of keys

**When IP hash is OK:**
- Pool is static (no auto-scaling)
- State is fully externalized (misses are tolerable)
- You understand scaling requires a maintenance window

**Senior recommendation:**
> *"I'd use consistent hashing for any system that auto-scales. IP hash is fine for fixed topologies but is dangerous when combined with auto-scaling groups where server count fluctuates."*

---

## Q4: "Your API is behind an AWS ALB. During a peak traffic event, p99 latency jumps from 50ms to 8,000ms. The servers themselves show 40% CPU. What's causing this?"

**Trap:** Candidates jump to "scale up the servers." CPU is fine — so it's not server capacity.

**Systematic diagnosis:**

```
CPU 40% → servers not overloaded
p99 spike → something is queuing or blocking at a specific percentile

Checklist:
1. ALB connection count → is the LB itself at capacity? (unlikely with cloud LB)
2. ALB target response time → is the LB waiting on backends, or is LB itself slow?
3. DB connection pool exhaustion → requests queuing for a DB connection
   (10 servers × 20 connections = 200 DB connections; PostgreSQL default max = 100)
4. Downstream API throttling → payment/3rd party API has rate limit, 429s cause retries
5. GC pauses → Java/Python GC collects at high frequency under load, causing latency spikes
```

**Most common culprit:** DB connection pool exhaustion.

```
40 API servers × 20 connections = 800 connections
PostgreSQL max_connections = 100 (default)
→ 700 connections rejected → requests queue in app → p99 explodes
```

**Fix:**
- Add PgBouncer (connection pooler) between API servers and DB
- PgBouncer maintains 20 real DB connections; 800 app connections multiplex through it
- p99 returns to normal

---

## Q5: "Should the load balancer decrypt HTTPS or should the backend servers handle TLS?"

**There's no single right answer — reason through it:**

**Option A: Terminate at LB (standard)**
- ✅ One cert to manage, auto-renewed (AWS ACM)
- ✅ LB can inspect HTTP content (headers, path routing)
- ✅ Backends are simpler (no cert management)
- ❌ Traffic between LB and backend is plaintext within VPC
- Risk: if VPC is compromised, internal traffic is visible

**Option B: End-to-end TLS (pass-through or re-encrypt)**
- ✅ Encryption in transit everywhere (compliance: PCI-DSS, HIPAA)
- ✅ LB can't see request content (privacy + security)
- ❌ Each backend needs a cert; harder to manage at scale
- ❌ LB cannot do Layer 7 routing in pure pass-through mode
- ❌ Performance overhead: TLS handshake on every connection to backend

**Option C: mTLS via service mesh**
- LB terminates external TLS
- Between services: Envoy sidecar handles mTLS transparently
- App code is unaware; service mesh CA rotates certs automatically
- Best of both worlds: compliance + simplicity + service-to-service auth

**Senior answer:**
> *"For most systems I'd terminate at the ALB — it's simpler and internal VPC traffic is reasonably trusted. For regulated industries needing encryption everywhere, I'd layer on a service mesh with mTLS between services rather than manually managing certs on every backend."*

---

## Q6: "How do you do a zero-downtime deployment with a load balancer?"

**Full procedure — show you understand each step:**

```
Step 1: Pre-deploy
  - Reduce connection draining timeout if requests are short-lived (30s vs 300s)
  - Ensure new version passes health checks in staging

Step 2: Rolling deploy
  a. Deregister Server-1 from LB (LB stops new traffic to it)
  b. Wait for draining (in-flight requests complete)
  c. Deploy new version to Server-1
  d. Server-1 starts, health check passes → LB adds it back
  e. Repeat for Server-2, Server-3...

Step 3: Verify
  - Monitor error rate and p99 latency during deploy
  - Canary: deploy to 1 server first; watch for 5 mins; then proceed

Step 4: Rollback trigger
  - If error rate > 1% or p99 > threshold → halt, deregister new-version servers
  - Previous-version servers still in pool → instant rollback
```

**Kubernetes equivalent:** Rolling update strategy with `maxUnavailable: 1` and readiness probe gate — pod only added to LB after readiness check passes.

---

## Q7: "Your LB is load balancing across 5 database read replicas. One replica is 2 minutes behind the primary (replication lag). What do you do?"

**Trap:** Just removing it from rotation loses read capacity without fixing root cause.

**Strong answer:**

1. **Detect it first** — monitor replication lag (`SHOW SLAVE STATUS`, CloudWatch `ReplicaLag`). Alert if lag > 10 seconds.

2. **Remove from LB rotation immediately** — don't serve stale reads to users who need recent data (e.g., "did my payment succeed?")

3. **Root cause investigation:**
   - Is the replica under heavy read load? → reduce read traffic to it
   - Is the primary generating huge write volume? → replica can't keep up → scale primary writes (queue buffer) or use parallel replication
   - Network partition between primary and replica?

4. **Topology consideration:**
   - For reads that MUST be fresh (financial data, inventory) → always route to primary
   - For reads that can be stale (social feed, search) → use replicas, document acceptable staleness

**Senior signal:**
> *"Replication lag is a consistency signal, not just a performance problem. I'd tag db reads by staleness tolerance at the application layer: `freshness=strong` routes to primary, `freshness=eventual` routes to any replica that's within our acceptable lag threshold."*
