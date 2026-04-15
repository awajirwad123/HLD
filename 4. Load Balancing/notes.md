# Load Balancing — Notes

## One-Line Summary

> A load balancer distributes traffic across multiple backends, eliminates SPOF, enables horizontal scaling, and detects failures automatically.

---

## Layer 4 vs Layer 7 — Decision Rule

| Use Layer 4 when...                          | Use Layer 7 when...                        |
|----------------------------------------------|---------------------------------------------|
| Non-HTTP protocol (TCP, UDP, gRPC raw)        | HTTP/HTTPS traffic (web apps, REST APIs)   |
| Absolute minimum latency is required          | You need URL-based or header-based routing |
| You don't need to inspect request content     | You need SSL termination at LB              |
| Examples: DB TCP, gaming UDP, raw Kafka       | Examples: API gateway, web apps, microservices |

---

## Algorithm Selection Guide

| Algorithm            | Best For                                              | Avoid When                        |
|----------------------|-------------------------------------------------------|-----------------------------------|
| Round Robin          | Homogeneous servers, short requests                   | Long-lived connections, mixed server specs |
| Weighted RR          | Servers with different capacities                     | Dynamic load changes              |
| Least Connections    | Long-lived connections (WebSocket, streaming, uploads)| Very short requests (overhead not worth it) |
| Least Response Time  | Mixed request durations, heterogeneous servers        | Simple setups (overhead)          |
| IP Hash              | Legacy stateful app, need sticky without cookies      | Dynamic scaling (hash changes)    |
| Consistent Hashing   | Distributed caches, CDN routing                      | Small server counts               |

---

## Health Check Config Reference

```
Interval:            10s   (how often to probe)
Timeout:             2s    (how long to wait for response)
Healthy threshold:   2     (consecutive successes to mark healthy)
Unhealthy threshold: 3     (consecutive failures to mark unhealthy)
Protocol:            HTTP
Path:                /healthz
Expected status:     200
```

**Time to detect failure:** `interval × unhealthy_threshold` = 10s × 3 = 30 seconds worst case.

To detect faster: lower interval (5s) + lower threshold (2) = 10s detection.

---

## SSL Termination Models

```
Model A (standard):  Client → HTTPS → LB → HTTP → Backend   (1 cert at LB)
Model B (passthrough): Client → HTTPS → LB → HTTPS → Backend (cert at each backend)
Model C (re-encrypt): Client → HTTPS → LB → HTTPS → Backend (different internal cert)
```

- **Model A**: simplest, fine for most use cases (HTTP in trusted VPC)
- **Model B**: needed when LB cannot inspect traffic (Layer 4 passthrough)
- **Model C**: required for compliance (PCI-DSS, HIPAA) — encrypt in transit everywhere

---

## Key Numbers

| Metric                                  | Value                                 |
|-----------------------------------------|---------------------------------------|
| AWS ALB max requests/sec                | ~50K–100K+ (auto-scales)             |
| Nginx OSS throughput                    | ~50K RPS on 4-core machine           |
| Health check detection time             | 30–60s (with 10s interval, 3 failures)|
| Connection draining timeout (default)   | 300s (AWS ALB); reduce to 30s for fast deploy |
| LB latency overhead                     | ~1ms (Layer 7); < 0.1ms (Layer 4)   |
| Sticky session cookie TTL               | Match session timeout (e.g., 30 min) |

---

## Real-World Tools

| Tool              | Layer | Managed? | Notes                                   |
|-------------------|-------|----------|-----------------------------------------|
| AWS ALB           | L7    | Yes      | Best for HTTP; path/host routing, WAF   |
| AWS NLB           | L4    | Yes      | Ultra-low latency, static IP            |
| Nginx             | L4+L7 | No       | OSS, highly configurable                |
| HAProxy           | L4+L7 | No       | High performance, fine-grained control  |
| Envoy             | L7    | No       | Service mesh sidecar, very powerful     |
| Traefik           | L7    | No       | Kubernetes-native, auto-discovers services |
| Cloudflare LB     | L7    | Yes      | Global, anycast, built-in DDoS          |

---

## Common Mistakes to Avoid in Interviews

- ❌ Never mention LB without also mentioning health checks
- ❌ Don't suggest sticky sessions as the first solution — always prefer stateless
- ❌ Don't say "the LB handles everything" — LB itself can fail; mention HA
- ❌ Don't ignore connection draining when discussing rolling deployments
- ❌ Don't confuse L4 and L7 — know when each is appropriate

---

## Interview Anchor Phrases

> *"I'd use an AWS ALB here — it's Layer 7, so it can route `/api/*` to API servers and `/static/*` to S3 origin. It also handles TLS termination, health checks, and scales automatically."*

> *"For the load balancer itself being a SPOF — cloud-managed LBs like ALB are inherently HA across AZs, so that concern is handled by the provider."*

> *"I'd use least connections instead of round robin here because we're dealing with WebSocket connections — some users maintain 30-minute sessions, so equal request count doesn't mean equal load."*
