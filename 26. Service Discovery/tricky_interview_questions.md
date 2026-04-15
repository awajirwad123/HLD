# Service Discovery — Tricky Interview Questions

## Q1: "How do you handle service discovery for a multi-cloud, multi-region deployment?"

**What interviewers probe:** Cross-cloud networking, federation, latency, consistency vs availability trade-offs.

**Answer:**

Single-region discovery (Kubernetes DNS or Consul) doesn't work across cloud boundaries because:
- No shared network namespace
- Different CIDR ranges / private IPs not routable across clouds
- Latency across regions is 10–100ms vs < 1ms in-cluster

**Architecture for multi-region:**

**Option A — Global load balancer + regional registries:**
```
Client → Anycast DNS (Cloudflare / Route53 Latency Routing)
          ↓ (routes to nearest region)
         Regional Service Registry (Consul per region)
          ↓
         Regional service fleet
```
Each region is self-contained. Global DNS handles region routing. Consul federation replicates service health across DCs for cross-DC failover.

**Option B — Service mesh federation (Istio multi-cluster):**
```
Cluster A (us-east) ←── Istio ServiceEntry + mTLS ───→ Cluster B (eu-west)
```
Each cluster's control plane knows about remote services via `ServiceEntry` CRDs. Envoy routes cross-cluster with mTLS. East-west gateway handles the actual traffic.

**Trade-offs:**
- Option A: simpler, less coupling, regional isolation. Harder to do transparent cross-region calls.
- Option B: transparent service-to-service calls across clusters. More complex to operate, requires shared CA or trust federation.

**Key interview point:** "I would start with Option A — regional isolation with cross-region DNS failover. Option B is appropriate when you need transparent call routing without the application being aware of region."

---

## Q2: "A service lookup returns 10 healthy instances but 3 are slow. How does client-side LB know to avoid them?"

**What interviewers probe:** Active vs passive health, outlier detection, circuit breakers, p99 routing.

**Answer:**

Registry health checks are binary (healthy / unhealthy) and fire on a slow interval (10–30s). A slow instance that doesn't fail the health check will keep receiving traffic.

**Passive health / outlier detection (Envoy/Istio):**
```yaml
outlierDetection:
  consecutiveGatewayErrors: 5      # 5 consecutive 5xx → eject
  interval: 10s                     # Check every 10s
  baseEjectionTime: 30s             # Eject for 30s initially
  maxEjectionPercent: 50            # Never eject > 50% of instances
```

This lets the proxy observe actual traffic outcomes and temporarily remove slow/failing instances without waiting for the registry health check.

**Client-side strategies (beyond round-robin):**
1. **Least-connections routing:** Pick the instance with fewest in-flight requests. Naturally avoids slow instances (they accumulate in-flight requests).
2. **Random-pick-2 (Power of Two Choices):** Pick 2 random instances, route to the less-loaded one. Near-optimal load distribution with O(1) cost.
3. **Circuit breaker per instance:** If latency or error rate for an endpoint exceeds threshold, open the circuit for that instance.

**Key interview point:** "The registry tells you who's alive. The proxy/client tells you who's fast. You need both active health checks (registry) and passive outlier detection (proxy) to achieve optimal routing."

---

## Q3: "Consul is down for 5 minutes. How do you ensure services can still communicate?"

**What interviewers probe:** Resilience, cache fallback, graceful degradation, Consul HA.

**Answer:**

**Prevention (HA setup):**
- Run 3 or 5 Consul server nodes — a single node failure doesn't affect quorum
- Consul agents (clients) cache service registrations locally; a server restart doesn't immediately affect queries from agents within the same DC

**In-process fallback for Consul client libraries:**
1. **Stale reads:** Consul client SDK has a `stale=true` option — returns cached data from agent without requiring server quorum. Slightly stale but available.
2. **Last-known-good cache in application:** Application caches the last successful resolution for up to 5 minutes. Falls back to cache on registry unavailability.
3. **Circuit breaker around registry queries:** If 3 consecutive registry queries fail, use cache; re-try registry every 10s.

**For Kubernetes users:**
CoreDNS is highly available (2+ replicas), and EndpointSlice data persists in etcd even if CoreDNS pods restart — DNS restores quickly.

**What happens during the 5-minute outage:**
- Existing connections are unaffected (discovery only happens on new connections)
- New connections use cached discovery data — may route to stale/missing instances
- Rolling deploys should be paused until registry is healthy

**Interview trap:** Candidates often say "Consul being down is catastrophic." It's not — existing connections survive. The real risk is new connection setup to freshly scaled instances being missed.

---

## Q4: "Your service uses Kubernetes DNS but you're seeing connection errors on rolling deploy. Why and how do you fix it?"

**What interviewers probe:** Readiness gates, grace period, connection draining, iptables propagation lag.

**Answer:**

**Root cause — the termination race condition:**

When a pod receives SIGTERM:
1. Kubernetes starts updating iptables rules to remove the pod from the endpoint
2. BUT iptables rule propagation takes 1–5 seconds across all nodes
3. Meanwhile, the pod receives SIGTERM and starts shutting down
4. Clients on nodes that haven't received the iptables update yet still route to the dying pod
5. The pod's server closes the socket → connection refused or RST

**Fixes:**

**Fix 1 — preStop sleep hook:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```
Adds 5s delay before shutdown signal. Gives kube-proxy time to propagate iptables before the socket closes.

**Fix 2 — Graceful shutdown in application:**
```python
# On SIGTERM: stop accepting new connections, drain in-flight requests
signal.signal(signal.SIGTERM, lambda *_: server.shutdown())
```
In-flight requests complete before the process exits.

**Fix 3 — `terminationGracePeriodSeconds`:**
```yaml
spec:
  terminationGracePeriodSeconds: 60   # Default is 30s
```
Kubernetes waits this long after SIGTERM before sending SIGKILL.

**The real fix** is combining all three: preStop sleep (5s) + application graceful drain + reasonable grace period.

---

## Q5: "How would you implement zero-downtime service discovery for a stateful service (e.g., a distributed cache)?"

**What interviewers probe:** Stateful vs stateless discovery, consistent hashing for cache affinity, connection draining.

**Answer:**

Stateless services are easy — any healthy instance is equivalent. Stateful services (caches, databases, partitioned message queues) have **affinity requirements**: a specific key must go to a specific instance.

**For a distributed cache (e.g., custom memcached cluster):**

**Step 1 — Consistent hashing for routing:**
The client uses consistent hashing (not round-robin) to map keys to the same cache node. Adding/removing one node moves only `1/N` keys on average.

**Step 2 — Discovery updates via gossip or registry watch:**
When a node is added or removed, the client's consistent hash ring is updated. The `1/N` affected keys miss cache temporarily (cold start) but don't cause errors.

**Step 3 — Graceful drain for node removal:**
When decommissioning a node:
1. Mark node as "draining" in registry (still receives reads, not writes)
2. Wait for TTL expiry (clients stop routing new writes to it)
3. Remove from registry

This prevents cache stampede on sudden removal.

**Step 4 — Replica awareness:**
For caches with replicas, route reads to replica first, writes to primary. Discovery must expose role metadata (`primary`, `replica`) alongside address.

**Key interview point:** "For stateful services, discovery is not just 'find a healthy node' — it's 'find the right healthy node for this key, and handle topology changes gracefully.'"
