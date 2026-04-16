# Client-Server Architecture — Interview Simulator

*Mock interview scenarios testing stateless/stateful design, scaling decisions, and resilience. Set a timer per scenario.*

---

## How to Use

1. Read the **scenario** only
2. Set timer: **20 minutes** per scenario
3. Design on paper, then compare with guided answer

---

## Simulation 1: Design a Ride-Sharing App Driver Location Service

### Scenario
> Uber needs to store and query real-time driver locations. 5M active drivers globally, each updating location every 4 seconds. Riders query "find nearest 10 drivers." How do you design this?

---

### Guided Walkthrough

**Clarify:**
- Global or regional? (regional makes data isolation cleaner)
- Accuracy needed? (100m is fine for matching)
- History needed? (for this service: only current location)
- How fast must "find nearest drivers" respond? (< 100ms)

**Estimate:**
```
5M drivers × 1 update/4s = 1.25M updates/second ← massive write load
Reads: "find nearest 10 drivers" — assume 10M rider requests/min = ~167K reads/sec
Data per driver: 50 bytes (driver_id + lat/lng + timestamp)
Total in-memory: 5M × 50B = 250 MB (fits in Redis!)
```

**Architecture:**
```
Driver App → [ Driver Location API ] → [ Redis GEO ]
                                           │
Rider App → [ Matching Service ] ──────► GEORADIUS query
                                           │
                              returns top 10 nearest drivers
```

**Why Redis GEO?**
- `GEOADD drivers <lng> <lat> <driver_id>` — O(log N) insert
- `GEORADIUS drivers <lng> <lat> 5 km ASC COUNT 10` — find nearest N in radius
- Entire 5M driver dataset = 250MB in RAM — fully in-memory, < 1ms queries
- High write throughput: Redis handles >100K writes/sec

**Stateless design:**
- Location API servers are stateless — any instance handles any driver update
- State lives entirely in Redis GEO sorted set
- If a location server crashes, driver just reconnects and resumes updates

**Failure mode:**
- Redis primary down → Redis Sentinel promotes replica within seconds
- During failover: accept slight location staleness (< 10 seconds), no rides are blocked
- Driver location data has natural TTL: if no update for 30s, driver is considered offline

---

## Simulation 2: Design an Authentication Service

### Scenario
> Build an auth service used by 20 microservices in your platform. 50M users, 500K logins/day, each authenticated request must be verified in < 5ms. Design it.

---

### Guided Walkthrough

**Clarify:**
- JWT or session-based?
- SSO required (OAuth2/OIDC)?
- MFA support?
- Token refresh strategy?

**Agreed scope:** JWT-based, internal microservices, no SSO, MFA out of scope initially.

**Estimate:**
```
500K logins/day = ~6 login QPS (low — DB lookup acceptable)
500M API calls/day = ~5,800 token verifications/second ← must be fast
JWT verification: cryptographic check, no network call → ~0.5ms CPU ✅
```

**Architecture:**
```
Client → [ Auth Service ]
  Login:  validate password → issue access_token (15min) + refresh_token (7d)
  Verify: any microservice verifies JWT locally (no roundtrip to auth service)
  Refresh: swap refresh_token → new pair (rotation)

       [ Auth Service ]
           │
           ├──► [ PostgreSQL ] — user accounts, hashed passwords
           └──► [ Redis ] — refresh token store + blocklist
```

**Why JWT for verification at scale?**
- 5,800 verifications/sec × 20 services = each service verifies locally using public key
- No central auth service hit on every request → scales independently
- Auth service only hit on login + refresh (~6 + ~3K QPS combined) → easy to handle

**Stateless API servers:**
- Each microservice has the auth service's public key (distributed at startup)
- Token verification = local CPU operation, no network
- If Redis (refresh store) is down: new logins and refresh blocked, but existing access tokens still work for 15 minutes

**Security considerations to mention:**
- Passwords: bcrypt with cost factor 12 (slow hashing intentional)
- Refresh token rotation: each refresh invalidates the old token
- Blocklist in Redis for explicit logout
- HTTPS everywhere; tokens never in URL params (only headers/httpOnly cookies)

---

## Simulation 3: Scale a Monolithic App to Handle 100x Traffic (20 min)

### Scenario
> Your Python/FastAPI app currently handles 1K QPS. The business is growing and you need to plan for 100K QPS. The current architecture is: single FastAPI server + single PostgreSQL + no cache. What's your roadmap?

---

### Guided Roadmap

**Phase 1: Quick wins (1 week, 0–10x traffic)**
```
1. Add a load balancer (Nginx/ALB) → 2 stateless API instances instead of 1
2. Add Redis caching for top read queries (eliminate 90% of DB reads)
3. Add a read replica to PostgreSQL (route all reads there)
Current state: 1K → ~10K QPS handled
```

**Phase 2: Reliability (2–4 weeks, 10–30x traffic)**
```
4. Add health checks + auto-scaling group (3–10 instances)
5. Move session state to Redis (ensure true statelessness)
6. Add circuit breakers on external calls
7. Add structured logging + tracing
Current state: ~30K QPS handled
```

**Phase 3: Scale writes (1–2 months, 30–100x traffic)**
```
8. Identify write bottlenecks: what endpoints hammer the DB?
9. Async heavy writes: order processing, notifications → Kafka queue → workers
10. Add connection pooling (PgBouncer) — FastAPI × 10 instances × 20 connections = 200 DB connections → pooler reduces to 20 actual DB connections
11. Vertical scale DB primary (bigger machine buys time)
12. Evaluate: does the monolith need to be split? (only if teams conflict or scaling needs differ)
Current state: ~100K QPS handled
```

**What NOT to do:**
- Don't start with microservices — premature decomposition creates operational overhead
- Don't shard the DB first — exhaust vertical scale and caching before sharding
- Don't rewrite in a new language for performance — optimize the bottleneck, not the innocent code

**Interview signal:** Show the interviewer a phased approach. Big Bang rewrites fail. Incremental improvements with measurable outcomes show production maturity.