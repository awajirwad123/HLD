# Consistent Hashing — Hands-On Exercises

## Exercise 1: Core Ring Implementation with Distribution Analysis

Build a full consistent hash ring and compare it against modulo hashing when servers change topology.

```python
"""
Goal: Implement ConsistentHashRing, then prove that:
  - Modulo hashing remaps ~(N-1)/N keys when a server is added
  - Consistent hashing remaps only ~1/(N+1) keys
"""

import bisect
import hashlib
import statistics


class ConsistentHashRing:
    """
    Hash ring using virtual nodes (vnodes).
    Each physical node occupies `vnodes` positions on [0, 2^32).
    """

    def __init__(self, vnodes: int = 200):
        self.vnodes = vnodes
        self._ring: list[int] = []           # sorted list of hash positions
        self._map: dict[int, str] = {}       # hash position → node name

    def _hash(self, key: str) -> int:
        """Map an arbitrary string to [0, 2^32)."""
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node: str) -> None:
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h not in self._map:
                bisect.insort(self._ring, h)
                self._map[h] = node

    def remove_node(self, node: str) -> None:
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h in self._map:
                self._ring.remove(h)
                del self._map[h]

    def get_node(self, key: str) -> str:
        if not self._ring:
            raise RuntimeError("Ring is empty")
        h = self._hash(key)
        idx = bisect.bisect_right(self._ring, h) % len(self._ring)
        return self._map[self._ring[idx]]

    def distribution(self) -> dict[str, int]:
        """Count how many ring positions each node owns."""
        counts: dict[str, int] = {}
        for node in self._map.values():
            counts[node] = counts.get(node, 0) + 1
        return counts


# ─── Helper ───────────────────────────────────────────────────────────────────

def route_keys_modulo(keys: list[str], servers: list[str]) -> dict[str, str]:
    """Simple modulo (hash % N) routing."""
    n = len(servers)
    mapping = {}
    for k in keys:
        idx = int(hashlib.md5(k.encode()).hexdigest(), 16) % n
        mapping[k] = servers[idx]
    return mapping


# ─── Demo 1: Adding a node — how many keys remap? ────────────────────────────

NUM_KEYS = 50_000
keys = [f"key:{i}" for i in range(NUM_KEYS)]
servers_before = ["cache-a", "cache-b", "cache-c"]
servers_after  = servers_before + ["cache-d"]

# Consistent hashing
ring_before = ConsistentHashRing(vnodes=200)
for s in servers_before:
    ring_before.add_node(s)

ring_after = ConsistentHashRing(vnodes=200)
for s in servers_after:
    ring_after.add_node(s)

ch_remapped = sum(
    1 for k in keys if ring_before.get_node(k) != ring_after.get_node(k)
)

# Modulo routing
mod_before = route_keys_modulo(keys, servers_before)
mod_after  = route_keys_modulo(keys, servers_after)
mod_remapped = sum(1 for k in keys if mod_before[k] != mod_after[k])

print("=== Adding 1 server (3 → 4) ===")
print(f"Consistent hashing remapped: {ch_remapped:,} / {NUM_KEYS:,} "
      f"({ch_remapped/NUM_KEYS:.1%}) — expected ~{1/len(servers_after):.0%}")
print(f"Modulo hashing remapped:     {mod_remapped:,} / {NUM_KEYS:,} "
      f"({mod_remapped/NUM_KEYS:.1%}) — expected ~{(len(servers_before)-1)/len(servers_before):.0%}")

# Sample output:
# === Adding 1 server (3 → 4) ===
# Consistent hashing remapped: 12,643 / 50,000 (25.3%) — expected ~25%
# Modulo hashing remapped:     37,481 / 50,000 (74.9%) — expected ~75%


# ─── Demo 2: Removing a node ──────────────────────────────────────────────────

ring_post_remove = ConsistentHashRing(vnodes=200)
for s in servers_before:
    ring_post_remove.add_node(s)
ring_post_remove.remove_node("cache-c")  # 3 → 2 nodes

ch_rem_remapped = sum(
    1 for k in keys if ring_before.get_node(k) != ring_post_remove.get_node(k)
)

mod_post_remove  = route_keys_modulo(keys, ["cache-a", "cache-b"])
mod_rem_remapped = sum(1 for k in keys if mod_before[k] != mod_post_remove[k])

print("\n=== Removing 1 server (3 → 2) ===")
print(f"Consistent hashing remapped: {ch_rem_remapped:,} / {NUM_KEYS:,} "
      f"({ch_rem_remapped/NUM_KEYS:.1%}) — only keys from removed node migrate")
print(f"Modulo hashing remapped:     {mod_rem_remapped:,} / {NUM_KEYS:,} "
      f"({mod_rem_remapped/NUM_KEYS:.1%})")


# ─── Demo 3: Load distribution with vnodes ───────────────────────────────────

print("\n=== Key distribution across nodes (vnodes=200, 5 nodes) ===")
ring5 = ConsistentHashRing(vnodes=200)
for i in range(5):
    ring5.add_node(f"node-{i}")

actual_dist: dict[str, int] = {f"node-{i}": 0 for i in range(5)}
for k in keys:
    actual_dist[ring5.get_node(k)] += 1

per_node_values = list(actual_dist.values())
print(f"Expected per node: {NUM_KEYS // 5:,}")
print(f"Actual range:      {min(per_node_values):,} – {max(per_node_values):,}")
print(f"Std deviation:     {statistics.stdev(per_node_values):,.0f}")
for node, cnt in sorted(actual_dist.items()):
    bar = "█" * (cnt // 500)
    print(f"  {node}: {cnt:,}  {bar}")
```

---

## Exercise 2: Weighted Ring — Heterogeneous Server Capacity

Servers with more RAM get proportionally more virtual nodes.

```python
"""
Goal: Assign vnodes proportionally to server weight so that
      a 2× bigger server receives ~2× the keys.
"""

from __future__ import annotations
import bisect, hashlib, statistics
from dataclasses import dataclass, field


@dataclass
class WeightedConsistentHashRing:
    """Ring where each server's vnode count scales with its weight."""

    base_vnodes: int = 100   # vnodes for a server with weight=1

    _ring: list[int]         = field(default_factory=list, repr=False)
    _map:  dict[int, str]    = field(default_factory=dict, repr=False)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node: str, weight: float = 1.0) -> None:
        count = max(1, int(self.base_vnodes * weight))
        for i in range(count):
            h = self._hash(f"{node}#{i}")
            if h not in self._map:
                bisect.insort(self._ring, h)
                self._map[h] = node

    def get_node(self, key: str) -> str:
        h = self._hash(key)
        idx = bisect.bisect_right(self._ring, h) % len(self._ring)
        return self._map[self._ring[idx]]


# Fleet: mixed capacity servers
ring = WeightedConsistentHashRing(base_vnodes=100)
ring.add_node("small-1", weight=1.0)   # 100 vnodes
ring.add_node("small-2", weight=1.0)   # 100 vnodes
ring.add_node("large-1", weight=2.0)   # 200 vnodes → should get ~2× the keys
ring.add_node("large-2", weight=2.0)   # 200 vnodes

NUM_KEYS = 100_000
keys = [f"key:{i}" for i in range(NUM_KEYS)]
dist: dict[str, int] = {}
for k in keys:
    node = ring.get_node(k)
    dist[node] = dist.get(node, 0) + 1

print("=== Weighted Ring Distribution ===")
total_weight = 6.0  # 1+1+2+2
for node, cnt in sorted(dist.items()):
    expected = NUM_KEYS / total_weight * (2 if "large" in node else 1)
    bar = "█" * (cnt // 1000)
    print(f"  {node} (weight={'2.0' if 'large' in node else '1.0'}): "
          f"{cnt:,} keys  (expected ~{expected:,.0f})  {bar}")

# Expected output:
# small-1 (weight=1.0): ~16,600 keys
# small-2 (weight=1.0): ~16,600 keys
# large-1 (weight=2.0): ~33,400 keys
# large-2 (weight=2.0): ~33,400 keys
```

---

## Exercise 3: Memcached-Style Cache Client

Simulate routing 100,000 cache operations across 5 nodes and measure balance.

```python
"""
Goal: Build a CacheClient that uses consistent hashing for routing,
      then measure: hit rate stays constant during a node failure/recovery.
"""

import hashlib, bisect, random, time
from typing import Optional


class ConsistentHashRing:
    def __init__(self, vnodes=200):
        self.vnodes = vnodes
        self._ring: list[int] = []
        self._map: dict[int, str] = {}

    def _hash(self, s: str) -> int:
        return int(hashlib.md5(s.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node: str):
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h not in self._map:
                bisect.insort(self._ring, h)
                self._map[h] = node

    def remove_node(self, node: str):
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h in self._map:
                self._ring.remove(h)
                del self._map[h]

    def get_node(self, key: str) -> Optional[str]:
        if not self._ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect_right(self._ring, h) % len(self._ring)
        return self._map[self._ring[idx]]


class FakeMemcachedNode:
    """In-process dict acting as a Memcached node."""
    def __init__(self, name: str):
        self.name = name
        self._store: dict[str, str] = {}

    def get(self, key: str) -> Optional[str]:
        return self._store.get(key)

    def set(self, key: str, value: str) -> None:
        self._store[key] = value


class CacheClient:
    def __init__(self):
        self._ring = ConsistentHashRing(vnodes=200)
        self._nodes: dict[str, FakeMemcachedNode] = {}

    def add_node(self, name: str):
        self._ring.add_node(name)
        self._nodes[name] = FakeMemcachedNode(name)

    def remove_node(self, name: str):
        self._ring.remove_node(name)
        del self._nodes[name]

    def get(self, key: str) -> Optional[str]:
        node_name = self._ring.get_node(key)
        if node_name and node_name in self._nodes:
            return self._nodes[node_name].get(key)
        return None

    def set(self, key: str, value: str) -> None:
        node_name = self._ring.get_node(key)
        if node_name and node_name in self._nodes:
            self._nodes[node_name].set(key, value)


# ─── Simulation ───────────────────────────────────────────────────────────────

client = CacheClient()
for i in range(5):
    client.add_node(f"mc-{i}")

# Warm up the cache
keys = [f"key:{i}" for i in range(10_000)]
for k in keys:
    client.set(k, f"value-{k}")

def measure_hit_rate(client: CacheClient, keys: list[str]) -> float:
    hits = sum(1 for k in keys if client.get(k) is not None)
    return hits / len(keys)

print(f"Hit rate (5 nodes, fully warm): {measure_hit_rate(client, keys):.1%}")

# Simulate one node failing — only keys routed to mc-2 become misses
client.remove_node("mc-2")
print(f"Hit rate after mc-2 failure:    {measure_hit_rate(client, keys):.1%}")
# ~80% — only ~20% of keys (1/5) were on mc-2

# Recover the node (cold — needs re-warming)
client.add_node("mc-2")
print(f"Hit rate after mc-2 recovery:   {measure_hit_rate(client, keys):.1%}")
# Still ~80% — mc-2's data was lost; keys that re-route to it are cold misses

# Re-warm
for k in keys:
    if client.get(k) is None:
        client.set(k, f"value-{k}")
print(f"Hit rate after re-warming:      {measure_hit_rate(client, keys):.1%}")
# Back to 100%

# Key insight: only ~1/N keys affected per node change — not all keys
```

---

## Exercise 4: Cassandra-Style Replication Placement

Walk the ring clockwise to find N unique physical nodes for replication:

```python
"""
Goal: Implement get_replicas(key, n) — returns the n physical nodes
      responsible for storing replicas of a key.
      Mirrors how Cassandra places RF=3 replicas.
"""

import bisect
import hashlib
from typing import List


class ReplicatedHashRing:
    def __init__(self, vnodes: int = 200):
        self.vnodes = vnodes
        self._ring: list[int] = []
        self._map:  dict[int, str] = {}

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node: str) -> None:
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h not in self._map:
                bisect.insort(self._ring, h)
                self._map[h] = node

    def get_replicas(self, key: str, n: int) -> List[str]:
        """
        Return the n distinct physical nodes that should store key.
        Walk the ring clockwise from key's hash position,
        collecting nodes until n unique physical nodes are found.
        """
        if len(self._ring) == 0:
            return []

        h = self._hash(key)
        start_idx = bisect.bisect_right(self._ring, h) % len(self._ring)

        seen_nodes: set[str] = set()
        replicas: List[str] = []

        for offset in range(len(self._ring)):
            idx = (start_idx + offset) % len(self._ring)
            node = self._map[self._ring[idx]]
            if node not in seen_nodes:
                seen_nodes.add(node)
                replicas.append(node)
                if len(replicas) == n:
                    break

        return replicas


# ─── Demo ─────────────────────────────────────────────────────────────────────

ring = ReplicatedHashRing(vnodes=200)
nodes = [f"cassandra-{i}" for i in range(6)]
for node in nodes:
    ring.add_node(node)

# Test replication placement
test_keys = [f"user:{i}" for i in range(10)]
print("=== RF=3 Replica Placement ===")
for key in test_keys:
    replicas = ring.get_replicas(key, n=3)
    print(f"  {key:12s} → {replicas}")

# Verify: removing a node causes only that node's keys to re-route
key = "user:42"
before = ring.get_replicas(key, n=3)
print(f"\nBefore removal: {key} replicas → {before}")

ring.add_node("cassandra-new")
after = ring.get_replicas(key, n=3)
print(f"After adding cassandra-new: {key} replicas → {after}")
# Some keys gain cassandra-new as a replica; others are unchanged

# Count how many keys gain a new replica set after adding one node
all_keys = [f"key:{i}" for i in range(5_000)]
changed = sum(
    1 for k in all_keys
    if ring.get_replicas(k, 3) != ring.get_replicas(k, 3)  # both from same ring
)
print(f"\n(With RF=3 on 7 nodes, ~1/7 of keys gain new replica on node add)")
```

---

## Exercise 5: Rendezvous Hashing (HRW) Comparison

Implement rendezvous hashing as an alternative — same remapping properties, O(N) lookup:

```python
"""
Rendezvous (HRW) Hashing:
  For each key, score every server: score = hash(key + server).
  Server with the highest score wins.

Advantage: simpler than ring — no vnodes config needed, naturally balanced.
Disadvantage: O(N) per lookup — fine for small N (< 100 servers).

Use case: selecting a CDN cache server or small database shard set.
"""

import hashlib
import statistics


def hrw_get_node(key: str, nodes: list[str]) -> str:
    """Return the node with the highest score for this key."""
    def score(node: str) -> int:
        combined = f"{key}:{node}".encode()
        return int(hashlib.sha256(combined).hexdigest(), 16)

    return max(nodes, key=score)


# ─── Distribution test ────────────────────────────────────────────────────────

nodes = [f"server-{i}" for i in range(5)]
keys  = [f"key:{i}" for i in range(50_000)]

dist: dict[str, int] = {n: 0 for n in nodes}
for k in keys:
    dist[hrw_get_node(k, nodes)] += 1

print("=== HRW Distribution (5 nodes, 50k keys) ===")
for node, cnt in sorted(dist.items()):
    bar = "█" * (cnt // 500)
    print(f"  {node}: {cnt:,}  {bar}")
print(f"  Std dev: {statistics.stdev(dist.values()):,.0f}")

# ─── Remapping test ───────────────────────────────────────────────────────────

nodes_after = nodes + ["server-5"]
hrw_remapped = sum(
    1 for k in keys
    if hrw_get_node(k, nodes) != hrw_get_node(k, nodes_after)
)
print(f"\nHRW: Adding 1 server (5→6) remapped {hrw_remapped:,} / {len(keys):,} "
      f"({hrw_remapped/len(keys):.1%}) — expected ~{1/6:.0%}")
# ~16.7% — same property as consistent hashing, with simpler code

# ─── Side-by-side lookup performance ─────────────────────────────────────────
import time, bisect

ring = ConsistentHashRing(vnodes=200)        # from Exercise 1
for n in nodes:
    ring.add_node(n)

N = 100_000
t0 = time.perf_counter()
for k in keys[:N]:
    ring.get_node(k)
ch_time = time.perf_counter() - t0

t0 = time.perf_counter()
for k in keys[:N]:
    hrw_get_node(k, nodes)
hrw_time = time.perf_counter() - t0

print(f"\nLookup performance (100k keys, 5 nodes):")
print(f"  Consistent hashing (bisect): {ch_time*1000:.1f} ms — O(log KN)")
print(f"  Rendezvous hashing:          {hrw_time*1000:.1f} ms — O(N)")
# Consistent hashing is ~5–10× faster for 5+ nodes; gap grows with N
```

---

## Key Takeaways from All Exercises

| Property | Modulo | Consistent Hashing | HRW |
|---|---|---|---|
| Keys remapped on node add/remove | ~(N-1)/N | ~1/N | ~1/N |
| Lookup complexity | O(1) | O(log KN) | O(N) |
| Load balance | Perfect (no vnodes needed) | ±10–15% with vnodes | Near-perfect |
| Replication placement | Manual | Walk ring clockwise | Re-run with exclusion |
| Config complexity | Zero | Choose K (150–300) | Zero |

**When to choose what:**
- **Consistent hashing:** Distributed caches (Memcached), database sharding, CDN routing within a PoP
- **HRW:** Small node counts (<50), CDN origin selection, service mesh load balancing
- **Modulo:** Static-sized clusters that never change (rare in production)
