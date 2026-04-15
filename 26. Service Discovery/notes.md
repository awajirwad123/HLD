# Service Discovery — Reference Notes

## Discovery Pattern Comparison

| Pattern | Who does LB? | Extra hop? | LB logic per language? | Example |
|---|---|---|---|---|
| Client-side | Client | No | Yes | Ribbon + Eureka, gRPC |
| Server-side | LB / proxy | Yes | No | AWS ALB, Kubernetes Service |
| Service mesh | Sidecar proxy | Negligible | No | Istio + Envoy, Linkerd |
| DNS round-robin | Client (dumb) | No | No | Route53, Consul DNS |

---

## Registration Patterns

| Pattern | Who registers | Deregistration | Use when |
|---|---|---|---|
| Self-registration | App itself | On shutdown (or TTL) | Non-Kubernetes, full control |
| Platform registration | Orchestrator (K8s) | On pod termination | Kubernetes — preferred |

---

## Health Check Types (Consul)

| Type | Mechanism | Appropriate for |
|---|---|---|
| HTTP | GET to `/healthz` → 200 OK | HTTP/REST services |
| TCP | TCP connect attempt | Non-HTTP, DB |
| gRPC | gRPC health protocol | gRPC services |
| TTL | App POSTs heartbeat within TTL | Background workers |
| Script | Shell command returns 0 | Legacy/custom checks |

**Important:** Set `deregister_critical_service_after` to remove dead instances automatically.

---

## Kubernetes Service Discovery Summary

```
Pod starts → kubelet sets pod Ready → kube-controller-manager
                                      updates EndpointSlice
                                         → kube-proxy updates iptables
                                         → CoreDNS serves A record
                                       (propagation ~1s)
```

**Service types:**
| Type | Access scope | Use case |
|---|---|---|
| ClusterIP | Within cluster only | Inter-service communication |
| NodePort | External (node:port) | Dev/test, simple external access |
| LoadBalancer | External (cloud LB IP) | Production external services |
| ExternalName | DNS alias (CNAME) | Routing to external services |
| Headless (clusterIP: None) | Direct pod IPs | gRPC, Cassandra, StatefulSets |

---

## Consul Key Commands

```bash
# Register service via HTTP API
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d '{"Name":"payment-api","Port":8080,"Check":{"HTTP":"http://localhost:8080/health","Interval":"10s"}}'

# Query healthy instances
curl http://localhost:8500/v1/health/service/payment-api?passing=true

# DNS lookup
dig @127.0.0.1 -p 8600 payment-api.service.consul
dig @127.0.0.1 -p 8600 _payment-api._tcp.service.consul SRV

# KV store
consul kv put config/payment-api/timeout 5s
consul kv get config/payment-api/timeout

# Watch for changes (blocks until change)
consul watch -type=service -service=payment-api cat
```

---

## Service Mesh (Istio + Envoy) Key Concepts

**Control plane (Istiod):**
- Holds the authoritative service registry (synced from Kubernetes API)
- Pushes xDS config to all Envoy sidecars
- Manages TLS certificate issuance (SPIFFE/SVID)

**Data plane (Envoy sidecar):**
- Intercepts all inbound + outbound traffic via iptables rules
- Performs service discovery, load balancing, retries, circuit breaking
- Reports telemetry (metrics, traces) for every call

**xDS API (discovery service protocols):**
- **EDS** — Endpoint Discovery Service: list of healthy pod IPs
- **CDS** — Cluster Discovery Service: service definitions
- **LDS** — Listener Discovery Service: port/protocol config
- **RDS** — Route Discovery Service: traffic routing rules
- **SDS** — Secret Discovery Service: TLS certificates

**Benefits over pure DNS:**
- Zero-code mTLS between all services
- Traffic shifting (canary: 90/10 split) via VirtualService
- Circuit breaking, outlier detection at proxy level
- Distributed tracing automatically (Envoy injects trace headers)

---

## Common Failure Modes

| Failure | Impact | Mitigation |
|---|---|---|
| Registry outage | Clients can't discover new instances | Cache last-known-good addresses; hard-code fallback |
| Stale DNS TTL | Requests to dead instances | Short TTL (5–30s) + client retry |
| Thundering herd on registry startup | All clients re-query simultaneously | Jittered backoff on client startup |
| Health check false positive | Healthy instance removed | Multiple consecutive failures before deregister |
| Health check false negative | Unhealthy instance receives traffic | Combine passive (circuit breaker) and active health checks |
| Consul agent crash | Local service queries fail | Consul DNS cache | graceful fallback to stale |

---

## Key Numbers

| Metric | Value |
|---|---|
| Consul health check default interval | 10s |
| K8s EndpointSlice update latency | < 1s |
| Envoy xDS config propagation | 100–500ms |
| DNS TTL for discovery | 5–30s |
| Consul minimum server nodes | 3 (Raft quorum) |
| Consul maximum recommended cluster | 5–9 server nodes |
| K8s Service DNS format | `<svc>.<namespace>.svc.cluster.local` |
| Consul DNS format | `<service>.service.<datacenter>.consul` |
