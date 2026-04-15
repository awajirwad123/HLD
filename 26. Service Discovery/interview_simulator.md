# Service Discovery — Interview Simulator

## Scenario 1: Design Service Discovery for a Large Microservices Platform

**Prompt:** "You're building a microservices platform with 200+ services, 5 regions, 10K pods total. Design the service discovery system. What technology do you choose and why?"

---

**Clarifying Questions:**
- All Kubernetes-based or mixed with VMs?
- Cross-region communication frequency (intra-region mostly, or heavy cross-region calls)?
- Existing tooling (HashiCorp stack, AWS-native, GCP-native)?
- Any non-HTTP protocols (gRPC, Kafka, custom TCP)?

*Assume: Kubernetes, mostly intra-region calls, gRPC between services, some cross-region failover needed.*

---

**Decision: Kubernetes DNS + Istio service mesh per region, Consul for cross-region**

**Intra-region (per cluster):**
```
Services use gRPC with headless Services
         ↓
CoreDNS returns pod IPs (headless)
         ↓
gRPC client picks connection from pool (client-side LB)
         ↓
Envoy sidecar (Istio) handles:
  - mTLS between all services
  - Retry policy, circuit breaker
  - Outlier detection (eject slow pods)
  - Load balancing: LEAST_REQUEST
```

**Cross-region failover:**
```
Consul deployed with 3 nodes per region
Consul federation (WAN gossip) links all regions
Services register with both Kubernetes and Consul (via Consul Sync)
On primary region failure:
  Route53 health check → update DNS → traffic shifts to secondary
```

**Health checks:**
- Kubernetes readiness probe handles pod-level health → EndpointSlice
- Consul TCP/HTTP check for cross-region health awareness
- Envoy outlier detection for sub-check-interval passive health

---

**Scale numbers:**
- 10K pods × 200 services = 50 pods/service average
- CoreDNS handles ~100K queries/sec per cluster (tune with `--cache-size`)
- Envoy xDS updates propagated < 500ms per topology change
- Consul WAN gossip convergence < 2s for cross-region health updates

---

**Trade-offs to mention:**
- Istio adds ~5ms latency per call (sidecar overhead) — acceptable at scale for the mTLS + observability benefit
- Consul adds operational complexity — needed only for cross-region; skip for single-cloud single-region
- HeadlessService + gRPC client-side LB is better than ClusterIP for gRPC (which multiplexes over one HTTP/2 connection — ClusterIP picks one pod and sticks)

---

## Scenario 2: Debug — Service Cannot Find Upstream

**Prompt:** "The payment service has been deployed for an hour but is logging 'connection refused' to the inventory service. The inventory service pods appear healthy in Kubernetes. Debug the issue."

---

**Step 1: Verify the Service and endpoints exist**
```bash
kubectl get svc inventory-service -n production
kubectl get endpoints inventory-service -n production
```
If endpoints are empty: readiness probe is failing. Check pod status:
```bash
kubectl describe pod -l app=inventory-service -n production | grep -A5 Readiness
kubectl logs -l app=inventory-service -n production | tail -50
```

**Step 2: Verify DNS resolution**
```bash
# Exec into a pod in the same namespace
kubectl exec -it payment-service-xxx -n production -- sh
nslookup inventory-service.production.svc.cluster.local
```
- If NXDOMAIN: Service doesn't exist or is in the wrong namespace
- If resolves but connection refused: Service resolves but no healthy pods behind it

**Step 3: Check network policy**
```bash
kubectl get networkpolicy -n production
```
A NetworkPolicy might block port 8080 from payment-service to inventory-service. Verify the selector matches.

**Step 4: Check Service port mapping**
```bash
kubectl get svc inventory-service -n production -o yaml
```
Common mistake: `port: 80 → targetPort: 8080` but the pod is actually listening on 8081.
```bash
kubectl exec -it inventory-service-xxx -- ss -tlnp
```
Verify what port the process is actually listening on.

**Step 5: Check Istio (if mesh is enabled)**
```bash
istioctl proxy-status
istioctl analyze -n production
```
Misconfigured VirtualService or DestinationRule can blackhole traffic.

---

**Root causes in order of likelihood:**
1. Readiness probe failing → pod exists but not in endpoints
2. Wrong namespace or Service name typo
3. NetworkPolicy blocking the port
4. targetPort mismatch (pod on 8081, service maps to 8080)
5. Istio PeerAuthentication requiring mTLS but client sending plain HTTP

---

## Scenario 3: Design Service Discovery for a Hybrid Cloud Migration

**Prompt:** "You're migrating from on-prem bare metal to AWS. During the 18-month migration, you'll have services on both sides that need to talk to each other. How do you handle service discovery?"

---

**Key constraints:**
- On-prem: legacy services, mostly DNS-based discovery, some hard-coded IPs
- AWS: new services on EKS, native Kubernetes
- Connectivity: existing Direct Connect (private VPC to on-prem VLAN)
- Duration: 18 months — both sides live simultaneously

---

**Architecture: Consul as the bridge**

```
On-Prem                              AWS (EKS)
┌──────────────────────┐            ┌──────────────────────┐
│ Legacy Services      │            │ New Services (K8s)    │
│   ↓ register         │            │   ↓ register          │
│ Consul Agent         │            │ Consul Agent          │
│   ↕ WAN gossip       │◄──Direct──►│   ↕ WAN gossip       │
│ Consul Server (3)    │ Connect    │ Consul Server (3)     │
└──────────────────────┘            └──────────────────────┘

All services → consul-agent → DNS lookup via search domain: .service.consul
```

**Discovery:**
- On-prem service queries `payment-api.service.aws.consul` → gets AWS pod IPs
- AWS service queries `inventory.service.onprem.consul` → gets on-prem server IPs
- DNS is the universal interface — legacy apps need zero code changes

**Kubernetes integration:**
- Run Consul on EKS with `consul-k8s` Helm chart
- Enable `connectInject` for new services (Consul Connect mTLS)
- `sync-catalog` syncs Kubernetes Services → Consul registry automatically
- Legacy on-prem services register manually or via Ansible/Chef

**Migration strategy:**
1. Run Consul on-prem alongside existing discovery
2. Deploy AWS Consul cluster, establish WAN link
3. Register all services in Consul (Kubernetes side auto-synced)
4. Update on-prem services to use Consul DNS (`.service.consul`) instead of hard-coded IPs
5. As services migrate to AWS, they automatically pick up from Consul
6. After migration complete: decommission on-prem Consul cluster

**Key interview point:** "Consul is the right choice here because it federates across environments and provides a vendor-neutral DNS interface. Kubernetes-native tools (CoreDNS, Istio) are limited to the cluster — they can't discover on-prem services."
