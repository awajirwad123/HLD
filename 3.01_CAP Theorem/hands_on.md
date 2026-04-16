# CAP Theorem — Hands-On Exercises

## Exercise 1: Simulate CP vs AP Behavior Under a Partition

```python
"""
Simulates a two-node distributed key-value store.
Toggle CP vs AP mode to see how each behaves during a network partition.
"""
import asyncio
import time
from enum import Enum
from dataclasses import dataclass, field


class Mode(Enum):
    CP = "CP"   # Refuse reads/writes if can't reach peer (prefer consistency)
    AP = "AP"   # Always respond, may serve stale data (prefer availability)


@dataclass
class KVNode:
    node_id: str
    mode: Mode
    store: dict = field(default_factory=dict)
    versions: dict = field(default_factory=dict)  # key → version number
    peer: "KVNode" = None
    partitioned: bool = False  # True = can't talk to peer

    def _can_reach_peer(self) -> bool:
        return not self.partitioned and self.peer is not None and not self.peer.partitioned

    def write(self, key: str, value) -> dict:
        new_version = self.versions.get(key, 0) + 1

        if self.mode == Mode.CP:
            # CP: require peer acknowledgment before confirming write
            if not self._can_reach_peer():
                return {
                    "success": False,
                    "error": "503: Cannot reach peer during partition — refusing write to preserve consistency",
                    "node": self.node_id
                }
            # Write to self and peer atomically (simplified)
            self.store[key] = value
            self.versions[key] = new_version
            self.peer.store[key] = value
            self.peer.versions[key] = new_version
            return {"success": True, "key": key, "value": value, "version": new_version, "node": self.node_id}

        else:  # AP mode
            # AP: accept write locally; will sync with peer later (eventual consistency)
            self.store[key] = value
            self.versions[key] = new_version
            synced = False
            if self._can_reach_peer():
                self.peer.store[key] = value
                self.peer.versions[key] = new_version
                synced = True
            return {
                "success": True,
                "key": key, "value": value,
                "version": new_version,
                "node": self.node_id,
                "peer_synced": synced,
                "warning": None if synced else "Peer not synced — eventual consistency applies"
            }

    def read(self, key: str) -> dict:
        if self.mode == Mode.CP:
            # CP: only return value if we can confirm it's fresh with peer
            if not self._can_reach_peer():
                return {
                    "success": False,
                    "error": "503: Cannot confirm freshness — refusing read to preserve consistency",
                    "node": self.node_id
                }
            # Compare versions with peer; return only the latest
            local_ver = self.versions.get(key, 0)
            peer_ver  = self.peer.versions.get(key, 0)
            if local_ver >= peer_ver:
                return {"success": True, "key": key, "value": self.store.get(key), "version": local_ver, "node": self.node_id}
            else:
                return {"success": True, "key": key, "value": self.peer.store.get(key), "version": peer_ver, "node": f"{self.node_id}→peer"}

        else:  # AP mode
            # AP: always respond with local data, even if stale
            return {
                "success": True,
                "key": key,
                "value": self.store.get(key),
                "version": self.versions.get(key, 0),
                "node": self.node_id,
                "freshness": "guaranteed" if self._can_reach_peer() else "potentially stale"
            }

    def sync_from_peer(self):
        """Reconcile after partition heals — take newest version of each key."""
        if not self.peer:
            return
        for key, peer_ver in self.peer.versions.items():
            local_ver = self.versions.get(key, 0)
            if peer_ver > local_ver:
                self.store[key] = self.peer.store[key]
                self.versions[key] = peer_ver
        print(f"[{self.node_id}] Reconciled with peer after partition healed")


def run_demo(mode: Mode):
    print(f"\n{'='*60}")
    print(f"Mode: {mode.value}")
    print('='*60)

    node_a = KVNode("A", mode)
    node_b = KVNode("B", mode)
    node_a.peer = node_b
    node_b.peer = node_a

    # Normal operation
    print("\n[Normal operation]")
    print("Write X=10 via Node A:", node_a.write("X", 10))
    print("Read X from Node B:   ", node_b.read("X"))

    # Introduce partition
    print("\n[Partition introduced: A and B cannot communicate]")
    node_a.partitioned = True

    print("Write X=99 via Node A:", node_a.write("X", 99))
    print("Read X from Node B:   ", node_b.read("X"))

    # Heal partition
    print("\n[Partition healed]")
    node_a.partitioned = False
    node_a.sync_from_peer()
    node_b.sync_from_peer()
    print("Read X from Node A:", node_a.read("X"))
    print("Read X from Node B:", node_b.read("X"))


if __name__ == "__main__":
    run_demo(Mode.CP)
    run_demo(Mode.AP)
```

---

## Exercise 2: Observe Cassandra Consistency Levels (Conceptual Simulation)

```python
"""
Simulates Cassandra's tunable consistency to demonstrate CP vs AP behavior.
Mirrors the real Cassandra consistency level semantics.
"""
import random
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class CassandraNode:
    node_id: str
    data: dict = field(default_factory=dict)
    versions: dict = field(default_factory=dict)
    available: bool = True   # False = node is down or partitioned

    def write(self, key, value, version) -> bool:
        if not self.available:
            return False
        self.data[key] = value
        self.versions[key] = version
        return True

    def read(self, key) -> Optional[tuple]:
        if not self.available:
            return None
        return self.data.get(key), self.versions.get(key, 0)


class CassandraCluster:
    def __init__(self, replication_factor: int = 3):
        self.rf = replication_factor
        self.nodes = [CassandraNode(f"node_{i}") for i in range(replication_factor)]
        self._version = 0

    def write(self, key, value, consistency: str = "QUORUM") -> dict:
        self._version += 1
        v = self._version
        required = self._required_acks(consistency)

        successes = sum(n.write(key, value, v) for n in self.nodes)

        if successes >= required:
            return {"ok": True, "acks": successes, "required": required,
                    "consistency": consistency, "version": v}
        else:
            # Roll back partial writes (simplified — real Cassandra uses hinted handoff)
            return {"ok": False, "acks": successes, "required": required,
                    "error": f"Write failed: only {successes}/{required} acks",
                    "consistency": consistency}

    def read(self, key, consistency: str = "QUORUM") -> dict:
        required = self._required_acks(consistency)
        responses = [n.read(key) for n in self.nodes if n.available and n.read(key) is not None]

        if len(responses) < required:
            return {"ok": False,
                    "error": f"Read failed: only {len(responses)}/{required} nodes responded",
                    "consistency": consistency}

        # Pick newest version
        newest = max(responses, key=lambda r: r[1])
        return {"ok": True, "value": newest[0], "version": newest[1],
                "responses": len(responses), "required": required,
                "consistency": consistency}

    def _required_acks(self, consistency: str) -> int:
        return {
            "ONE": 1,
            "TWO": 2,
            "QUORUM": self.rf // 2 + 1,
            "ALL": self.rf,
        }.get(consistency, self.rf // 2 + 1)


def demo():
    cluster = CassandraCluster(replication_factor=3)

    print("=== Normal operation ===")
    print(cluster.write("balance", 1000, "QUORUM"))
    print(cluster.read("balance", "QUORUM"))

    print("\n=== One node goes offline ===")
    cluster.nodes[2].available = False

    print("Write with QUORUM (needs 2):", cluster.write("balance", 1100, "QUORUM"))
    print("Read  with QUORUM (needs 2):", cluster.read("balance", "QUORUM"))
    print("Read  with ONE    (needs 1):", cluster.read("balance", "ONE"))
    print("Write with ALL    (needs 3):", cluster.write("balance", 1200, "ALL"))  # ← fails

    print("\n=== Two nodes offline ===")
    cluster.nodes[1].available = False

    print("Write with QUORUM (needs 2):", cluster.write("balance", 1300, "QUORUM"))  # ← fails
    print("Read  with ONE    (needs 1):", cluster.read("balance", "ONE"))   # ← may be stale


if __name__ == "__main__":
    demo()
```

---

## Exercise 3: PACELC Trade-off Analyzer

```python
"""
Decision helper: given a system's requirements, suggest the best CAP/PACELC position.
Useful to run through mentally before any system design interview.
"""

def analyze_requirements(
    data_loss_acceptable: bool,
    stale_reads_acceptable: bool,
    must_always_respond: bool,
    latency_sensitive: bool,
    cross_region: bool
) -> dict:
    """
    Classifies system requirements into CAP + PACELC recommendation.
    Returns suggested design choices.
    """
    recommendation = {}

    # CAP choice
    if not data_loss_acceptable and not stale_reads_acceptable:
        recommendation["CAP"] = "CP"
        recommendation["partition_behavior"] = "Return error rather than stale or lost data"
        recommendation["examples"] = ["PostgreSQL sync replication", "ZooKeeper", "etcd", "Spanner"]
    elif must_always_respond:
        recommendation["CAP"] = "AP"
        recommendation["partition_behavior"] = "Always respond; reconcile conflicts after partition heals"
        recommendation["examples"] = ["Cassandra (CL=ONE)", "DynamoDB default", "Riak"]
    else:
        recommendation["CAP"] = "CP (tunable)"
        recommendation["partition_behavior"] = "Default CP; downgrade to AP under severe partition"
        recommendation["examples"] = ["Cassandra (CL=QUORUM)", "MongoDB w:majority"]

    # PACELC choice (normal operation, no partition)
    if latency_sensitive and stale_reads_acceptable:
        recommendation["PACELC"] = "EL — optimize for Latency"
        recommendation["normal_operation"] = "Read from local replica; tolerate brief staleness"
    elif not stale_reads_acceptable:
        recommendation["PACELC"] = "EC — optimize for Consistency"
        recommendation["normal_operation"] = "Require quorum or primary reads; accept higher latency"
    else:
        recommendation["PACELC"] = "EL (balanced)"
        recommendation["normal_operation"] = "Read replicas for most traffic; primary for critical reads"

    # Cross-region consideration
    if cross_region:
        recommendation["cross_region_note"] = (
            "Cross-region replication adds 50–200ms RTT. "
            "Strong consistency across regions requires consensus latency on every write. "
            "Consider geo-partitioning write ownership to reduce cross-region sync."
        )

    return recommendation


# --- Run it like a decision framework ---
scenarios = [
    {
        "name": "Payment processing",
        "data_loss_acceptable": False,
        "stale_reads_acceptable": False,
        "must_always_respond": False,
        "latency_sensitive": False,
        "cross_region": True
    },
    {
        "name": "Social media feed",
        "data_loss_acceptable": True,
        "stale_reads_acceptable": True,
        "must_always_respond": True,
        "latency_sensitive": True,
        "cross_region": True
    },
    {
        "name": "Distributed lock / leader election",
        "data_loss_acceptable": False,
        "stale_reads_acceptable": False,
        "must_always_respond": False,
        "latency_sensitive": False,
        "cross_region": False
    },
    {
        "name": "Product catalog",
        "data_loss_acceptable": False,
        "stale_reads_acceptable": True,
        "must_always_respond": True,
        "latency_sensitive": True,
        "cross_region": True
    },
]

if __name__ == "__main__":
    for s in scenarios:
        print(f"\n--- {s['name']} ---")
        name = s.pop("name")
        result = analyze_requirements(**s)
        for k, v in result.items():
            print(f"  {k}: {v}")
```

---

## Exercise 4: Detect Linearizability Violations

```python
"""
Demonstrates a linearizability violation — what CAP's C guards against.
Shows the kind of anomaly that AP systems can produce after a partition.
"""
import threading
import time
import random

# Simulated eventual-consistency key-value store
# Two nodes with async replication (AP mode)

class APNode:
    def __init__(self, name: str, replication_delay_ms: float = 50):
        self.name = name
        self.store = {}
        self.peer = None
        self.delay_ms = replication_delay_ms
        self._lock = threading.Lock()

    def write(self, key, value):
        with self._lock:
            self.store[key] = value
        # Async replication to peer (with delay)
        if self.peer:
            t = threading.Timer(self.delay_ms / 1000, self._replicate_to_peer, args=(key, value))
            t.start()

    def _replicate_to_peer(self, key, value):
        with self.peer._lock:
            # Only apply if peer doesn't have a newer version
            if self.peer.store.get(key) != value:
                self.peer.store[key] = value

    def read(self, key):
        with self._lock:
            return self.store.get(key)


def demonstrate_linearizability_violation():
    """
    Scenario:
      Thread 1 writes X=1 to Node A. Replication to Node B is delayed 50ms.
      Thread 2 reads X from Node B immediately after → sees None or old value.
      Thread 3 reads X from Node A → sees X=1.
      
    → Two readers see different values at the same logical time.
    → This violates linearizability (CAP Consistency).
    → This is EXPECTED and ACCEPTABLE in an AP system.
    → It must be handled at the application layer.
    """
    node_a = APNode("A", replication_delay_ms=50)
    node_b = APNode("B", replication_delay_ms=50)
    node_a.peer = node_b
    node_b.peer = node_a

    violations = []

    def writer():
        node_a.write("user_status", "online")

    def reader_from_b(results: list, label: str):
        time.sleep(0.005)  # 5ms after write
        val = node_b.read("user_status")
        results.append((label, val))

    def reader_from_a(results: list, label: str):
        time.sleep(0.005)  # same delay
        val = node_a.read("user_status")
        results.append((label, val))

    results = []
    threads = [
        threading.Thread(target=writer),
        threading.Thread(target=reader_from_b, args=(results, "Reader-B (may be stale)")),
        threading.Thread(target=reader_from_a, args=(results, "Reader-A (sees latest)")),
    ]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    time.sleep(0.1)  # Wait for replication

    print("\nLinearizability violation demonstration:")
    for label, val in results:
        print(f"  {label}: user_status = {val!r}")

    # After replication heals
    print(f"\nAfter 50ms replication:")
    print(f"  Node A: {node_a.read('user_status')}")
    print(f"  Node B: {node_b.read('user_status')}")
    print("  → Nodes converged (eventual consistency achieved)")
    print("\nApplication impact: user appeared offline to some clients for ~50ms")
    print("This is acceptable for a presence indicator, NOT for a payment status.")


if __name__ == "__main__":
    demonstrate_linearizability_violation()
```

---

## Exercise 5: CAP Classification Quiz (Interactive)

```python
"""
Interactive quiz: classify systems into CP or AP and justify.
Run before interviews to sharpen CAP classification.
"""

SYSTEMS = [
    {
        "name": "Apache ZooKeeper",
        "answer": "CP",
        "justification": "Uses ZAB (ZooKeeper Atomic Broadcast) consensus. Leader must have quorum. Minority partition nodes go into read-only mode — they refuse writes. Used for distributed coordination where correctness is critical."
    },
    {
        "name": "Apache Cassandra (CL=ONE)",
        "answer": "AP",
        "justification": "Any node responds with its local data immediately. If partitioned, all nodes continue serving. CL=ONE means one node's ack suffices — fast but potentially stale."
    },
    {
        "name": "Apache Cassandra (CL=QUORUM, N=3)",
        "answer": "CP (effectively)",
        "justification": "W=2, R=2, W+R=4 > N=3. Reads and writes overlap, guaranteeing linearizability. BUT: if 2 nodes are in the partitioned minority, writes fail → sacrifices availability for consistency."
    },
    {
        "name": "Amazon DynamoDB (default eventual consistency)",
        "answer": "AP",
        "justification": "Default reads are eventually consistent — served from any AZ. Strongly consistent reads are optional (and cost 2x RCU). Default = AP for availability and latency."
    },
    {
        "name": "MongoDB (w:majority, j:true)",
        "answer": "CP",
        "justification": "Write must be acknowledged by majority of replica set before confirming. On minority partition, writes fail rather than risk data divergence."
    },
    {
        "name": "DNS",
        "answer": "AP",
        "justification": "DNS servers respond with cached records even when out of date (stale TTL). Availability is paramount; a few seconds of stale DNS is acceptable."
    },
    {
        "name": "etcd",
        "answer": "CP",
        "justification": "Raft consensus. Cluster requires majority quorum for reads and writes. Minority partition nodes reject all writes. Used as the source of truth for Kubernetes cluster state."
    },
    {
        "name": "Redis (single primary, async replica)",
        "answer": "CP primary / AP with replica",
        "justification": "Primary is the only write source — fail → writes blocked (CP-like). Reads from async replica may be stale (AP-like). Redis Cluster auto-failover favors availability."
    },
]

def run_quiz():
    import random
    shuffled = random.sample(SYSTEMS, len(SYSTEMS))
    score = 0

    print("CAP Classification Quiz — type CP or AP (or your answer)\n")
    for i, s in enumerate(shuffled, 1):
        user = input(f"Q{i}: {s['name']} → ").strip().upper()
        correct = s["answer"].split()[0].upper()  # First word
        if user == correct or user in s["answer"].upper():
            print(f"  ✅ Correct! {s['justification']}\n")
            score += 1
        else:
            print(f"  ❌ Answer: {s['answer']}")
            print(f"  Why: {s['justification']}\n")

    print(f"Score: {score}/{len(shuffled)}")


if __name__ == "__main__":
    run_quiz()
```
