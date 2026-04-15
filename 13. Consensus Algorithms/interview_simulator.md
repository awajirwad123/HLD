# Consensus Algorithms — Interview Simulator

## Scenario 1: "How does Kubernetes use etcd for leader election?"

**Prompt:** "Walk me through how Kubernetes uses etcd. What happens when the etcd leader crashes? What's the impact on the cluster?"

---

**Model answer:**

**etcd's role in Kubernetes:**

```
Kubernetes Control Plane:
  kube-apiserver  ← reads/writes all cluster state to etcd
  kube-scheduler  ← reads pending pods from etcd; writes scheduling decisions
  kube-controller-manager ← watches etcd for desired vs actual state differences
  etcd (3 or 5 nodes) ← single source of truth for ALL cluster state
```

**Data stored in etcd:**
- Pod definitions and status
- Service configurations
- ConfigMaps, Secrets
- Node registrations
- Deployment, StatefulSet, DaemonSet specs

**When etcd leader crashes:**

```
t=0: etcd leader (node 1) crashes
t=50ms: Followers detect missing heartbeat
t=150ms: First follower's election timeout fires → starts election in term N+1
t=250ms: New leader elected (quorum = 2/3 or 3/5)
t=300ms: Kubernetes apiserver reconnects to etcd, detects new leader endpoint
```

**Impact during the 250ms window:**
- `kubectl apply` and all write operations fail transiently
- kube-scheduler cannot schedule new pods (requires etcd write)
- Existing running pods are **unaffected** — kubelet on each node operates independently

**Full resolution:** Within ~1–2 seconds including apiserver reconnect. Users with retry logic won't notice.

**Configuration for HA:**
```yaml
# etcd cluster endpoints in kube-apiserver config
--etcd-servers=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379
# If one endpoint is down, apiserver retries others
```

**Best practices:**
- 3-node etcd across 3 AZs (no single AZ failure kills quorum)
- Snapshots every 30 min → S3 (for disaster recovery)
- Monitor: `etcdctl endpoint health` + alert on quorum loss

---

## Scenario 2: "Design a distributed locking service for your microservices platform"

**Prompt:** "Multiple instances of a payment-processing service are running. Only one should process a payment at a time to prevent double-charges. Design a distributed lock service."

---

**Requirements clarification:**
- How long can a lock be held? (Typically seconds for a payment transaction)
- What happens if the holder crashes while holding the lock?
- Do we need exactly-once semantics for payments, or is at-most-once sufficient?

**Architecture:**

```
Lock Service: etcd cluster (3 nodes, Raft)
  Data model: {lock_key → {holder_id, token, lease_expiry}}

Payment Service instances (N):
  → Each instance: acquire lock → process → release
  → Lock key: payment:{payment_id}
  → Lease TTL: 30 seconds (max payment processing time)
```

**Acquire flow:**
```python
def acquire_lock(payment_id: str, instance_id: str) -> tuple[bool, int]:
    key = f"lock:payment:{payment_id}"
    # Atomic: set only if key doesn't exist, with 30s TTL
    # Returns (acquired: bool, fencing_token: int)
    result = etcd.transaction(
        compare=[etcd.transactions.version(key) == 0],
        success=[etcd.transactions.put(key, instance_id, lease=30)],
        failure=[]
    )
    if result.succeeded:
        token = etcd.get_revision()  # Monotonically increasing cluster revision
        return True, token
    return False, 0
```

**Fencing at the payment DB:**
```sql
-- Include fencing token in the payment write
-- Reject if token is stale (older than last seen)
UPDATE payments
SET status = 'processed', fencing_token = $token
WHERE payment_id = $id
  AND (fencing_token IS NULL OR fencing_token < $token)
```

**Trade-offs discussed:**
- Lock TTL too short: payment not done in time → another instance grabs lock → race condition protected by fencing token
- Lock TTL too long: crashed instance blocks other instances for 30 seconds
- Recommended: 5–10s TTL + heartbeat renewal from payment service while processing

---

## Scenario 3: "Debugging — Raft cluster is losing writes under load"

**Prompt:** "Your 3-node Raft cluster is dropping ~2% of writes under peak load (10K writes/sec). Logs show 'leader changed' errors. Why?"

---

**Symptoms analysis:**

"Leader changed" errors during writes = the current leader stepped down mid-write. The client's write started on the old leader, leader stepped down, write was not committed.

**Root causes:**

**1. Follower disk I/O causing slow AppendEntries acks:**

```
Commit requires quorum (2 of 3 nodes) to ack the log entry.
If follower disk I/O is slow (SSD under load, EBS throttling):
  Follower takes 200–500ms to flush WAL
  Leader's commit timeout: leader waits, eventually suspects followers are dead
  Leader sees no quorum → new election
  Under load: this cascades → frequent leader changes
```

Fix: Use SSDs with predictable I/O (avoid EBS `gp2`; use `io1` or NVMe local disk for etcd).

**2. CPU/memory pressure on leader node:**

Garbage collection pause on the leader JVM (or Go runtime) = heartbeats not sent in time = followers start elections.

Fix: Tune GC, use dedicated nodes for consensus nodes with no co-located services.

**3. Election timeout too close to disk flush latency:**

If election timeout = 150ms and disk flush takes 120ms, any load spike triggers spurious elections.

Fix: Increase election timeout to 500ms–1s. Increase heartbeat interval proportionally (heartbeat = election_timeout / 5).

**4. Network issues (packet loss, jitter):**

TCP retransmissions under high load delay AppendEntries ACKs → leader suspects followers → election.

Fix: Monitor network metrics. Use Priority QoS to give consensus traffic high priority. Ensure consensus nodes are in the same AZ (low latency).

**Diagnostic commands:**
```bash
# etcd metrics
etcdctl endpoint status --cluster
# Look for: leader changes (high = instability), raft_apply_duration (should be < 100ms)

# Check disk latency
etcdctl check perf   # runs I/O benchmark; should show < 10ms sequential write latency
```

**Tuning:**
```yaml
# etcd configuration
heartbeat-interval: 250    # ms (default: 100ms)
election-timeout: 1250     # ms (default: 1000ms, must be 5x heartbeat)
# Increasing these makes elections less trigger-happy under load at cost of slower failover
```
