# Service Discovery — Quick-Fire Questions

**Q1: What problem does service discovery solve?**

In dynamic environments (containers, auto-scaling), service instances start and stop on different hosts/ports. Hard-coded addresses break immediately. Service discovery provides a mechanism to locate healthy instances dynamically at runtime, without configuration changes.

---

**Q2: What is the difference between client-side and server-side service discovery?**

- **Client-side:** The client queries the registry directly and selects an instance (load balancing logic in the client). Example: Ribbon + Eureka.
- **Server-side:** A load balancer or proxy sits in front of the service fleet. The client only knows the LB address. Example: Kubernetes Service + kube-proxy, AWS ALB.

Client-side has less overhead (no extra hop) but requires LB logic in every language SDK. Server-side is language-agnostic but adds a hop.

---

**Q3: What is a Kubernetes headless service and when do you use it?**

A headless service has `clusterIP: None`. Instead of a single VIP, DNS returns the individual pod IP addresses. Use it when the client needs direct connections to specific pods — e.g., gRPC (which does its own connection pooling), Cassandra/Kafka clients (which need to know individual broker addresses), StatefulSets (which use pod DNS names like `pod-0.svc`).

---

**Q4: How does Kubernetes know which pods are healthy endpoints?**

kubelet runs the pod's readiness probe (HTTP, TCP, or exec). When the probe passes, kubelet sets the pod condition `Ready: True`. The kube-controller-manager watches for this and updates the EndpointSlice API object for the matching Service. kube-proxy and CoreDNS watch EndpointSlice and update iptables rules and DNS records accordingly. Propagation from pod-ready to DNS update is typically < 1 second.

---

**Q5: What is the DNS TTL trade-off in service discovery?**

- **Short TTL (5–30s):** Clients get fresh instance lists quickly after changes, but generate many DNS queries (higher load on DNS server).
- **Long TTL (300s+):** Fewer DNS queries, but clients may cache stale records pointing to dead instances for minutes after they fail.

For dynamic environments, 5–30s TTL is typical. Clients should also implement retry with fresh DNS lookup on connection failure.

---

**Q6: What are the three health check failure modes to understand?**

1. **False positive (healthy but removed):** Health check fails transiently due to GC pause or slow response, removing a healthy instance from the pool. Mitigate with: require N consecutive failures before deregistering.
2. **False negative (unhealthy but kept):** Health check passes even though the service is functionally broken (e.g., deep dependency is down). Mitigate with: deep health checks that verify dependencies.
3. **Health check latency hides cascading failure:** Under heavy load, instances respond slowly to health checks but appear "healthy" while serving degraded traffic. Mitigate with: latency-based health + passive circuit breaking.

---

**Q7: What is Consul DNS and how does it work?**

Consul agents expose a DNS interface on port 8600. Lookups follow the pattern `<service>.service.<dc>.consul`. DNS returns A records for healthy instances — Consul internally filters by health check status before returning results. Services can query via standard DNS if they set the search domain to `consul`. Example: `payment-api.service.dc1.consul → [10.0.1.1, 10.0.1.2]`.

---

**Q8: What is a service mesh and how does it differ from a traditional API gateway?**

- **API Gateway:** Handles north-south traffic (external → internal). Single ingress point. Usually at the edge.
- **Service mesh:** Handles east-west traffic (internal service → service). Deploys a sidecar proxy next to every service. Provides mTLS, observability, retries, circuit breaking for all inter-service calls transparently.

A service mesh is like deploying a smart proxy everywhere, not just at the edge. Istio + Envoy is the dominant implementation.

---

**Q9: What happens to traffic during a rolling deploy in Kubernetes?**

1. New pods start, run their readiness probe
2. Once ready, they are added to EndpointSlice
3. kube-proxy updates iptables — new pods start receiving traffic
4. Old pods are sent a SIGTERM, enter Terminating state
5. kube-proxy removes them from iptables rules
6. Kubernetes waits `terminationGracePeriodSeconds` (default 30s) for in-flight requests to complete, then sends SIGKILL

The window between steps 4 and 5 can cause in-flight request failures. Mitigate with: graceful shutdown (`server.close()`), `preStop` hook to wait, and application-side connection draining.

---

**Q10: What is the difference between Consul and etcd for service discovery?**

| | Consul | etcd |
|---|---|---|
| Primary purpose | Service discovery + health checks | Distributed key-value store |
| Health checks | Built-in (HTTP, TCP, TTL, gRPC) | No built-in; external sidecars |
| DNS interface | Yes (native) | No (needs external adapter) |
| UI | Yes | No (external Kubernetes Dashboard) |
| Multi-datacenter | Yes (built-in federation) | No (single cluster |
| Who uses it | HashiCorp stack, Nomad | Kubernetes, CockroachDB |

Use Consul when health checking and DNS are requirements. Use etcd when you need a strongly consistent KV store (often embedded in the platform).

---

**Q11: How do you prevent a thundering herd when your service registry restarts?**

When the registry restarts, all clients may simultaneously re-register and re-query. Mitigations:
1. **Jittered startup delay** — clients sleep `random(0, backoff_cap)` before registering
2. **Local cache / warm standby** — clients keep a local cache of last-known-good addresses and continue using them until the registry is healthy
3. **High-availability registry** — Consul 3-node cluster means one node restart doesn't affect clients
4. **Graceful drain** — Consul sends notifications of leader change; clients hold off on bulk queries

---

**Q12: Can you use service discovery without a dedicated registry? How?**

Yes — **platform-native discovery:**
- **Kubernetes DNS:** CoreDNS is the built-in registry via Kubernetes API. No Consul or etcd needed for service discovery within a cluster.
- **AWS Cloud Map:** Managed service discovery integrated with Route53 and ECS/EKS.
- **Envoy / Istio:** Control plane polls Kubernetes API directly (no separate registry).

For simple in-cluster communication in Kubernetes, `ClusterIP` Services + DNS is all you need. A dedicated registry like Consul adds value primarily for multi-cluster, multi-cloud, or non-Kubernetes workloads.
