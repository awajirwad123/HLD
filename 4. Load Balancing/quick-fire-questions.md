# Load Balancing — Quick-Fire Questions

*Short answers for rapid revision.*

---

**Q1: What is the primary purpose of a load balancer?**
Distribute incoming traffic across multiple backend servers to eliminate SPOF, enable horizontal scaling, and automatically route around unhealthy instances.

---

**Q2: What is the difference between Layer 4 and Layer 7 load balancing?**
Layer 4 routes based on IP/port (TCP/UDP level) — fast but content-blind. Layer 7 routes based on HTTP content (URL, headers, cookies) — more powerful but slightly more overhead.

---

**Q3: When would you choose Layer 4 over Layer 7?**
When the traffic is non-HTTP (raw TCP, UDP, gRPC direct), when minimum latency is critical, or when you can't inspect packet content.

---

**Q4: Name 4 common load balancing algorithms.**
Round Robin, Weighted Round Robin, Least Connections, IP Hash (Consistent Hashing is an advanced variant).

---

**Q5: When is Least Connections better than Round Robin?**
When connections are long-lived (WebSockets, streaming, file uploads). Round Robin assumes equal request duration; Least Connections routes based on actual current load.

---

**Q6: What is a health check in the context of load balancing?**
A probe the LB sends to backends to verify they're alive and able to serve traffic. Failed health checks cause the LB to remove that backend from rotation automatically.

---

**Q7: What is the difference between active and passive health checks?**
Active: LB sends synthetic requests at intervals (`GET /health`). Passive: LB observes real traffic — if errors occur, marks unhealthy. Active detects failures faster but adds probe overhead.

---

**Q8: What does "SSL termination at the load balancer" mean?**
The LB decrypts HTTPS traffic and forwards plain HTTP to backend servers. Centralizes cert management; backends don't need certs. Fine for trusted internal networks.

---

**Q9: What is a sticky session? What's wrong with it?**
A sticky session (session affinity) pins a client to a specific backend using a cookie or IP. Problem: breaks horizontal scaling, causes uneven load, and the session is lost if that server fails.

---

**Q10: What is the correct long-term fix for sticky sessions?**
Externalize session state to a shared store (Redis). Makes all servers stateless — any can handle any request.

---

**Q11: What is connection draining?**
A grace period (e.g., 30s) where the LB stops sending new requests to a deregistering server but lets existing in-flight requests complete. Prevents 500 errors during deployments.

---

**Q12: How does a load balancer avoid being a SPOF?**
Use Active-Passive pair (VRRP failover), Active-Active (both handle traffic simultaneously), or a cloud-managed LB (AWS ALB/NLB are inherently HA across AZs).

---

**Q13: What is GSLB?**
Global Server Load Balancing — routes users to the correct geographic region (data center) before regional LB routing. Implemented via GeoDNS or anycast.

---

**Q14: What is path-based routing? Give an example.**
Layer 7 LB routes requests based on URL path. Example: `/api/*` → API server cluster, `/static/*` → CDN origin, `/admin/*` → admin service.

---

**Q15: What is the "slow start" feature in load balancers?**
Gradually ramps up traffic to a newly added server instead of immediate full load. Prevents overwhelming a fresh instance that's still warming up (cache cold, JVM JIT compiling, etc.).

---

**Q16: What is Weighted Round Robin and when do you use it?**
A server with weight=3 gets 3x the requests of a server with weight=1. Use it when servers have different hardware specs (bigger instance should handle more traffic).

---

**Q17: What real AWS service is an example of a Layer 7 LB?**
AWS Application Load Balancer (ALB). For Layer 4: AWS Network Load Balancer (NLB).

---

**Q18: In a service mesh, where does load balancing happen for inter-service traffic?**
At the sidecar proxy (Envoy) of the calling service — each service's sidecar does client-side load balancing to the target service's instances. No central LB for east-west traffic.

---

**Q19: What is IP hash load balancing? What breaks it?**
Hash of client IP determines which server they're routed to — same IP always hits same server. It breaks when the server count changes: all hashes remap, destroying session affinity.

---

**Q20: What should a `/health` endpoint check vs not check?**
Check: critical dependencies (DB ping, Redis ping, required config). Don't check: optional/degraded dependencies, or run heavy queries. Keep it fast (< 100ms) — LBs poll it frequently.
