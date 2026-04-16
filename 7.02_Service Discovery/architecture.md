# Service Discovery — Architecture

## The Core Problem

In a static world, services know each other's addresses at deploy time. In a dynamic world (containers, auto-scaling, rolling deploys) that breaks immediately:

- A service instance starts on a random port on a random host
- Instances crash and restart with different IP addresses
- Auto-scaling adds and removes instances in seconds
- Blue/green deploys swap entire service fleets

**Service discovery** is the mechanism that lets services find each other dynamically, without hard-coded addresses.

---

## Client-Side vs Server-Side Discovery

### Client-Side Discovery

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Client → Service Registry (query "order-service")     │
│                                                         │
│  Registry returns: ["10.0.1.1:8080", "10.0.1.2:8080"] │
│                                                         │
│  Client picks one (load balancing logic in client)      │
│  Client → 10.0.1.1:8080 (direct connection)            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Pros:**
- Client has full control over load balancing (round-robin, least-connections, zone-aware)
- No extra network hop
- Can implement circuit breaking, retries inside the client

**Cons:**
- Load balancing logic must be in every client (language-specific libraries)
- Client must handle stale registrations, health filtering
- Every language needs its own SDK

**Example:** Netflix Eureka + Ribbon, Consul + client SDK, gRPC service config.

---

### Server-Side Discovery

```
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  Client → Load Balancer / API Gateway / Service Mesh Proxy   │
│                                                               │
│  Load Balancer queries registry → picks healthy instance     │
│  Load Balancer → 10.0.1.1:8080                               │
│                                                               │
│  Client only knows the load balancer address (stable VIP)    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Pros:**
- Client is simple — just talks to a stable VIP
- Load balancing logic centralised — all clients get it automatically
- Language-agnostic

**Cons:**
- Extra network hop
- Load balancer becomes a bottleneck / single point of failure (mitigated by HA setup)

**Examples:** AWS ALB/ELB, Kubernetes Service + kube-proxy, Envoy as sidecar proxy, Nginx/HAProxy with Consul template.

---

## Service Registry

The authoritative store of "what instances are healthy and where."

**Core operations:**
1. **Register** — instance announces itself with address, port, metadata
2. **Deregister** — instance removes itself on shutdown (or TTL expiry)
3. **Query** — client/LB asks "give me all healthy instances of X"
4. **Health check** — registry actively probes or instances heartbeat

**Registration patterns:**

| Pattern | Who registers | Risk |
|---|---|---|
| Self-registration | App registers itself on boot | App must include SDK, can forget to deregister |
| Third-party registration | Orchestrator (Kubernetes) registers on app's behalf | No SDK needed; platform handles lifecycle |

Kubernetes uses third-party registration — kubelet handles pod lifecycle, Endpoints/EndpointSlice API is the service registry.

---

## Consul

HashiCorp Consul is the most widely-used dedicated service registry. It offers:

### Service Registration (via config or API)
```json
{
  "service": {
    "name": "payment-api",
    "port": 8080,
    "tags": ["v2", "primary"],
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s",
      "timeout": "3s",
      "deregister_critical_service_after": "90s"
    }
  }
}
```

### Health Check Types
| Type | How | Use when |
|---|---|---|
| HTTP | GET to `/health` endpoint | HTTP services |
| TCP | TCP connection attempt | Non-HTTP services |
| gRPC | gRPC health protocol | gRPC services |
| TTL | Service sends heartbeat; must renew within TTL | Services that self-assess health |
| Script/Exec | Run a shell command | Legacy systems |

### Consul Architecture
```
Consul Agent (client) — runs on every host, handles local service registrations
         ↕ RPC
Consul Server (3 or 5 nodes) — stores registry data, uses Raft for consensus
```

Consul agents forward queries to server cluster. Clients read from local agent (cached) for speed.

### DNS Interface
Consul exposes service records via DNS:
```
payment-api.service.consul    → A record for each healthy instance
payment-api.service.dc1.consul → scope to datacenter
_payment-api._tcp.service.consul → SRV record with port info
```

Applications can use standard DNS lookups — no SDK required.

---

## Kubernetes Service Discovery

Kubernetes is the dominant orchestration platform and has built-in service discovery.

### Kubernetes Service (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-api
spec:
  selector:
    app: payment-api
  ports:
    - port: 80
      targetPort: 8080
```

Creates a stable `ClusterIP` VIP. kube-proxy installs iptables/ipvs rules on every node to load-balance to healthy pods.

**DNS:** `payment-api.default.svc.cluster.local` resolves to the ClusterIP.

### EndpointSlice API

The actual registry backing a Service. Kubernetes automatically maintains EndpointSlice objects with the list of healthy pod IPs. Updated within ~1s of a pod becoming unhealthy.

```
Service → EndpointSlice → [ {ip: "10.0.1.1", port: 8080, ready: true},
                            {ip: "10.0.1.2", port: 8080, ready: true},
                            {ip: "10.0.1.3", port: 8080, ready: false} ]
```

### Headless Services

For client-side load balancing (e.g., gRPC, Cassandra, StatefulSets):
```yaml
spec:
  clusterIP: None  # Headless
```
DNS returns the individual pod IPs instead of a VIP — clients perform their own load balancing.

---

## DNS-Based Discovery

The simplest form. Services register via DNS; clients do DNS lookup.

**TTL trade-off:**
- Short TTL (5–30s) → faster propagation of changes, but more DNS queries
- Long TTL (300s) → fewer DNS queries, but stale records after instance failure

**The DNS negative caching problem:** If DNS returns NXDOMAIN or times out, many clients cache the failure for the negative TTL. Hard to recover from.

**DNS round-robin:** DNS returns multiple A records; clients typically pick the first. Not a reliable load balancing mechanism — no health awareness, no session persistence.

---

## Service Mesh

A service mesh (Istio, Linkerd) handles service discovery, load balancing, mTLS, and observability via sidecar proxies — zero application code changes.

```
Service A → [Envoy sidecar] ──────────────→ [Envoy sidecar] → Service B
                ↑                                   ↑
                └────── Control Plane (Istiod) ─────┘
                         (pushes xDS config)
```

**xDS protocol:** Envoy sidecars connect to Istiod over gRPC and receive:
- **CDS** (Cluster Discovery Service): list of upstream services
- **EDS** (Endpoint Discovery Service): list of healthy endpoints per cluster
- **LDS/RDS/SDS**: listeners, routes, TLS certificates

**Discovery is pushed:** When a new pod starts, Kubernetes notifies Istiod, which pushes updated endpoint lists to all Envoy sidecars watching that service — in ~100–500ms.

---

## Key Numbers

| Metric | Typical Value |
|---|---|
| Consul health check interval | 10s |
| Kubernetes Endpoint update latency | < 1s (pod ready → DNS updated) |
| Envoy xDS propagation latency | 100–500ms |
| DNS TTL for service discovery | 5–30s |
| Consul cluster minimum nodes | 3 (for Raft quorum) |
| Consul agent cache TTL | 0 (always consistent) or configurable stale |
| kube-proxy iptables rules (large cluster) | 10K–100K rules → consider ipvs |
