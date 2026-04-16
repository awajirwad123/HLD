# Client-Server Architecture — Quick-Fire Questions

*Short answers for rapid revision.*

---

**Q1: What is a stateless server?**
A server that stores no per-client state between requests. All necessary context comes from the request itself (JWT, cookies, params).

---

**Q2: What is the main benefit of stateless servers?**
They can be freely scaled horizontally — any server instance can handle any request without sticky sessions.

---

**Q3: Where does "state" go if not on the server?**
In the client (JWT carrying identity) or in a shared external store (Redis for sessions, DB for persistent data).

---

**Q4: What is horizontal scaling?**
Adding more server instances to distribute load. Requires stateless design. No theoretical ceiling.

---

**Q5: What is vertical scaling?**
Upgrading the existing machine (more CPU/RAM). Simple, but has a hardware ceiling and remains a single point of failure.

---

**Q6: When is vertical scaling preferred?**
For databases (writing to multiple primaries is complex), or early-stage products where simplicity matters.

---

**Q7: What is a sticky session? Why is it a problem?**
A load balancer directive that pins a client to a specific server. Problem: that server becomes a dependency — if it fails, the client loses state. Prevents free horizontal scaling.

---

**Q8: How do you fix the sticky session problem?**
Move session data to an external shared store (Redis). All servers can then access it, removing the affinity requirement.

---

**Q9: What is a JWT and why does it help stateless design?**
JSON Web Token — a self-contained, signed token carrying claims (user ID, roles). Any server can verify it without a central session store.

---

**Q10: What is the 3-tier architecture?**
Presentation (client) → Logic (API server) → Data (database). Each tier is independently scalable. The standard model for web applications.

---

**Q11: What is a health check endpoint?**
A lightweight HTTP endpoint (e.g., `/health`) that returns 200 if the server is operational. Load balancers poll this to route traffic only to healthy instances.

---

**Q12: What should a health check verify?**
Its own process is running, and critical dependencies are reachable (DB ping, Redis ping). It should NOT run heavy queries.

---

**Q13: What is thundering herd in client-server systems?**
When many clients simultaneously retry a failed request at the same time, overwhelming a recovering server with a sudden traffic spike.

---

**Q14: How do you prevent thundering herd?**
Exponential backoff + random jitter. Each client waits a different amount of time before retrying, spreading the load.

---

**Q15: What is the request-response model?**
Client sends a request, waits synchronously for the server to respond. The simplest and most common communication pattern.

---

**Q16: When does the request-response model break down?**
Long-running operations (client times out), real-time updates (polling is inefficient), and high fan-out writes.

---

**Q17: What is a 1-tier architecture?**
App and DB on the same machine (e.g., desktop app with embedded SQLite). Only suitable for local tools, not web systems.

---

**Q18: What does 202 Accepted mean and when do you use it?**
HTTP status indicating the request was received but processing is async (not yet complete). Used for write-heavy endpoints that offload work to a queue.

---

**Q19: How does auto-scaling work conceptually?**
Monitor CPU/memory/QPS metrics. When utilization exceeds threshold, spin up new instances (scale out). When it drops, remove instances (scale in). Works only with stateless services.

---

**Q20: Name two real-world systems that are stateless at the API layer.**
Netflix API Gateway, Twitter API servers — both are stateless; session/user state is in Redis or DB, not in server RAM.
