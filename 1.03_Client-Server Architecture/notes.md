# Client-Server Architecture — Notes

## Core Concepts Cheat Sheet

---

### Stateless vs Stateful — One-Line Rule

> **Stateless** = server stores nothing between requests. State lives in the client (JWT) or a shared store (Redis, DB).
> **Stateful** = server holds per-client state in memory. Requires sticky sessions. Hard to scale.

---

### Stateless — How State Is Passed

| State Type         | Mechanism                              |
|--------------------|----------------------------------------|
| Identity / Auth    | JWT in Authorization header            |
| Session data       | Session ID in cookie → lookup Redis    |
| Pagination         | Cursor/token in query param            |
| Request context    | All params in request body/headers     |

---

### Horizontal vs Vertical Scaling Summary

| Dimension         | Vertical (Scale Up)           | Horizontal (Scale Out)          |
|-------------------|-------------------------------|----------------------------------|
| How                | Bigger machine                | More machines                   |
| Requires stateless?| No                           | Yes                             |
| Upper limit        | Hardware ceiling              | Almost unlimited                |
| Failure risk       | SPOF                         | Partial failure (one instance)  |
| Cost               | Expensive at high specs       | Cheaper, commodity hardware     |
| Best for           | DB writes, early stage        | API servers, workers             |

---

### 3-Tier Architecture (Standard Web App)

```
Tier 1: Presentation  →  Browser / Mobile App
Tier 2: Logic         →  API Server (FastAPI, etc.)
Tier 3: Data          →  PostgreSQL / Redis / S3
```

Each tier scales independently. Tier 2 is stateless → horizontal scaling free.

---

### Key Failure Mitigations

| Problem               | Solution                                   |
|-----------------------|--------------------------------------------|
| Server crash          | Multiple instances + health check          |
| Sticky session issues | Externalize session to Redis               |
| Client retry storm    | Exponential backoff + jitter               |
| LB single point       | Active-passive LB pair                     |
| DB overwhelm on retry | Circuit breaker + queue buffer             |

---

### Retry Backoff Formula

```
wait = (2 ^ attempt) × random(0.75, 1.25)
attempt 0 → ~1s
attempt 1 → ~2s
attempt 2 → ~4s
attempt 3 → ~8s
attempt 4 → ~16s
```

Always cap max retries (3–5). Always add jitter to avoid thundering herd.

---

### Health Check Best Practices

- Expose `/health` or `/healthz` endpoint on every service
- Check critical dependencies (DB, Redis, downstream services)
- Return 200 = healthy, 503 = unhealthy
- Load balancers AND orchestrators (Kubernetes) use this to route/restart
- Keep it lightweight — do NOT run migrations or heavy queries here

---

### Interview Anchor Phrase

> *"I'll design this service to be stateless — any auth state will live in JWTs, any session data in Redis. That way the load balancer can freely distribute traffic and we can auto-scale horizontally without sticky session concerns."*

---

### When Stateful Is Acceptable

- WebSocket connections (connection = state; use a connection registry to locate them)
- In-memory caches on workers (acceptable if stale data is tolerable)
- Leader nodes in Raft/Paxos clusters (by design)
- Low-latency game servers (state must be in-memory for speed)

---

### Numbers to Remember

| Fact                                   | Value         |
|----------------------------------------|---------------|
| Redis GET latency (same DC)            | < 1 ms        |
| JWT verification cost (CPU)            | ~0.5 ms       |
| Typical LB health check interval       | 5–30 seconds  |
| Instance startup time (container)      | 10–60 seconds |
| Exponential backoff max attempts       | 3–5           |
