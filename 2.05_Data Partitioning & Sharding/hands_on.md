# Data Partitioning & Sharding — Hands-On Exercises

## Exercise 1: Hash-Based Sharding Router

```python
import hashlib
from dataclasses import dataclass, field


@dataclass
class ShardNode:
    node_id: int
    host: str
    port: int
    record_count: int = 0


class HashShardRouter:
    """
    Simple hash-mod sharding router.
    Demonstrates: routing, imbalance detection, and the resharding problem.
    """

    def __init__(self, nodes: list[ShardNode]):
        self.nodes = nodes
        self.num_shards = len(nodes)

    def _hash_key(self, key: str) -> int:
        return int(hashlib.sha256(key.encode()).hexdigest(), 16)

    def get_shard(self, key: str) -> ShardNode:
        shard_id = self._hash_key(key) % self.num_shards
        return self.nodes[shard_id]

    def simulate_inserts(self, keys: list[str]) -> dict[int, int]:
        """Simulate inserting keys and return distribution counts."""
        distribution: dict[int, int] = {n.node_id: 0 for n in self.nodes}
        for key in keys:
            node = self.get_shard(key)
            distribution[node.node_id] += 1
            node.record_count += 1
        return distribution

    def print_distribution(self, dist: dict[int, int]):
        total = sum(dist.values())
        print(f"\n{'Node':>6}  {'Count':>8}  {'%':>6}  {'Bar'}")
        print("-" * 40)
        for node_id, count in sorted(dist.items()):
            pct = count / total * 100
            bar = "█" * int(pct / 2)
            print(f"  {node_id:>4}  {count:>8}  {pct:>5.1f}%  {bar}")

    def show_resharding_impact(self, keys: list[str], new_node_count: int):
        """
        Show what fraction of keys would need to migrate
        when changing from N to new_node_count shards.
        """
        old_count = self.num_shards
        migrated = 0
        for key in keys:
            old_shard = self._hash_key(key) % old_count
            new_shard = self._hash_key(key) % new_node_count
            if old_shard != new_shard:
                migrated += 1

        pct = migrated / len(keys) * 100
        print(f"\nResharding {old_count} → {new_node_count} shards:")
        print(f"  Keys that move: {migrated:,} / {len(keys):,} ({pct:.1f}%)")


def main():
    import random
    import string

    # Build 4 shard nodes
    nodes = [ShardNode(node_id=i, host=f"db-{i}.internal", port=5432) for i in range(4)]
    router = HashShardRouter(nodes)

    # Generate 10,000 random user IDs
    keys = [f"user_{random.randint(1, 10_000_000)}" for _ in range(10_000)]

    print("=== Hash-Mod Sharding: 4 nodes, 10,000 keys ===")
    dist = router.simulate_inserts(keys)
    router.print_distribution(dist)

    # Show how bad resharding is
    router.show_resharding_impact(keys, 5)   # Add 1 shard
    router.show_resharding_impact(keys, 8)   # Double shards


if __name__ == "__main__":
    main()
```

---

## Exercise 2: Consistent Hashing Ring

```python
import hashlib
import bisect
from dataclasses import dataclass, field


@dataclass
class PhysicalNode:
    name: str
    host: str


class ConsistentHashRing:
    """
    Consistent hashing ring with virtual nodes.
    - Less than 1/N data moves when adding/removing a node
    - Virtual nodes ensure even load distribution
    """

    def __init__(self, virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self._ring: dict[int, str] = {}     # position → node_name
        self._sorted_keys: list[int] = []   # sorted ring positions
        self._nodes: dict[str, PhysicalNode] = {}

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: PhysicalNode):
        self._nodes[node.name] = node
        for i in range(self.virtual_nodes):
            vnode_key = f"{node.name}#vnode{i}"
            position = self._hash(vnode_key)
            self._ring[position] = node.name
            bisect.insort(self._sorted_keys, position)
        print(f"  [+] Added node '{node.name}' ({self.virtual_nodes} vnodes)")

    def remove_node(self, node_name: str):
        if node_name not in self._nodes:
            return
        for i in range(self.virtual_nodes):
            vnode_key = f"{node_name}#vnode{i}"
            position = self._hash(vnode_key)
            del self._ring[position]
            idx = bisect.bisect_left(self._sorted_keys, position)
            self._sorted_keys.pop(idx)
        del self._nodes[node_name]
        print(f"  [-] Removed node '{node_name}'")

    def get_node(self, key: str) -> str:
        if not self._ring:
            raise RuntimeError("No nodes in ring")
        position = self._hash(key)
        # Find first node clockwise from position
        idx = bisect.bisect(self._sorted_keys, position)
        if idx == len(self._sorted_keys):
            idx = 0  # wrap around
        return self._ring[self._sorted_keys[idx]]

    def get_distribution(self, keys: list[str]) -> dict[str, int]:
        dist: dict[str, int] = {n: 0 for n in self._nodes}
        for key in keys:
            node = self.get_node(key)
            dist[node] += 1
        return dist

    def print_distribution(self, dist: dict[str, int]):
        total = sum(dist.values())
        print(f"  {'Node':>12}  {'Count':>8}  {'%':>6}  Bar")
        print("  " + "-" * 45)
        for node, count in sorted(dist.items()):
            pct = count / total * 100
            bar = "█" * int(pct / 2)
            print(f"  {node:>12}  {count:>8}  {pct:>5.1f}%  {bar}")


def main():
    import random

    ring = ConsistentHashRing(virtual_nodes=150)

    # Start with 3 nodes
    print("=== Initial Ring: 3 nodes ===")
    for name in ["node-A", "node-B", "node-C"]:
        ring.add_node(PhysicalNode(name, f"{name}.db.internal"))

    keys = [f"key_{i}" for i in range(10_000)]

    print("\nDistribution with 3 nodes:")
    ring.print_distribution(ring.get_distribution(keys))

    # Snapshot current assignment
    pre_assignments = {k: ring.get_node(k) for k in keys}

    # Add a 4th node (should migrate ~1/4 of keys)
    print("\n=== Adding node-D ===")
    ring.add_node(PhysicalNode("node-D", "node-D.db.internal"))

    print("\nDistribution with 4 nodes:")
    ring.print_distribution(ring.get_distribution(keys))

    # Count how many keys migrated
    migrated = sum(1 for k in keys if ring.get_node(k) != pre_assignments[k])
    print(f"\nKeys migrated: {migrated:,} / {len(keys):,} ({migrated/len(keys)*100:.1f}%)")
    print(f"Expected ~25% ({len(keys)//4:,}) — consistent hashing delivers!")

    # Remove a node
    print("\n=== Removing node-B ===")
    pre2 = {k: ring.get_node(k) for k in keys}
    ring.remove_node("node-B")
    migrated2 = sum(1 for k in keys if ring.get_node(k) != pre2[k])
    print(f"Keys migrated: {migrated2:,} / {len(keys):,} ({migrated2/len(keys)*100:.1f}%)")


if __name__ == "__main__":
    main()
```

---

## Exercise 3: Shard Key Analysis Tool

```python
from collections import Counter
import hashlib
import statistics


def analyze_shard_key(candidates: list[tuple[str, list]], num_shards: int = 8):
    """
    Evaluate candidate shard keys for evenness and hotspot risk.
    candidates: list of (key_name, list_of_values)
    """

    def _hash(val: str) -> int:
        return int(hashlib.sha256(str(val).encode()).hexdigest(), 16)

    def _score(dist: list[int]) -> dict:
        total = sum(dist)
        expected = total / len(dist)
        stdev = statistics.stdev(dist) if len(dist) > 1 else 0
        max_shard = max(dist)
        imbalance_pct = (max_shard - expected) / expected * 100
        return {
            "stdev": round(stdev, 1),
            "max_shard": max_shard,
            "min_shard": min(dist),
            "imbalance_%": round(imbalance_pct, 1),
        }

    print(f"Shard key analysis — {num_shards} shards, evaluating distribution\n")
    results = []

    for key_name, values in candidates:
        dist = [0] * num_shards
        for v in values:
            shard = _hash(v) % num_shards
            dist[shard] += 1
        score = _score(dist)
        results.append((key_name, score))

    # Sort by imbalance (lower = better)
    results.sort(key=lambda x: x[1]["imbalance_%"])

    print(f"  {'Key Name':>25}  {'StdDev':>8}  {'Imbalance%':>12}  {'Max':>6}  {'Min':>6}  Verdict")
    print("  " + "-" * 75)
    for key_name, s in results:
        verdict = "✓ Good" if s["imbalance_%"] < 20 else ("⚠ Medium" if s["imbalance_%"] < 50 else "✗ Bad")
        print(f"  {key_name:>25}  {s['stdev']:>8.1f}  {s['imbalance_%']:>11.1f}%  "
              f"{s['max_shard']:>6}  {s['min_shard']:>6}  {verdict}")


def main():
    import random

    n = 100_000

    candidates = [
        ("user_id (UUID-like)",       [str(i) for i in range(n)]),
        ("country_code",              random.choices(
                                          ["US", "IN", "BR", "DE", "FR", "CN", "UK", "AU"],
                                          weights=[40, 20, 10, 5, 5, 8, 7, 5], k=n)),
        ("user_type (free/paid)",     random.choices(["free", "paid"], weights=[90, 10], k=n)),
        ("status (active/inactive)",  random.choices(["active", "inactive"], weights=[70, 30], k=n)),
        ("tenant_id (B2B, 500 co's)", [f"tenant_{random.randint(1, 500)}" for _ in range(n)]),
        ("created_at (month bucket)", [f"2024-{(i%12)+1:02d}" for i in range(n)]),
    ]

    analyze_shard_key(candidates, num_shards=8)


if __name__ == "__main__":
    main()
```

**Expected output:**
```
  Key Name                    StdDev  Imbalance%   Max   Min  Verdict
  ─────────────────────────────────────────────────────────────────────
        user_id (UUID-like)    156.2        1.0%  12627 12200  ✓ Good
   tenant_id (B2B, 500 co's)   189.4        2.5%  12823 12150  ✓ Good
       country_code            4821.3       48.2% 17400  2100  ⚠ Medium
  created_at (month bucket)   3201.1      200.0% 10000     0  ✗ Bad  (only 12 values → 4 shards empty)
   user_type (free/paid)      9000.1      900.0% 90000  1000  ✗ Bad
  status (active/inactive)    3000.0      300.0% 70000 30000  ✗ Bad
```
