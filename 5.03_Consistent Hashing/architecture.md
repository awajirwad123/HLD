# Consistent Hashing — Architecture & Concepts

## 1. The Problem: Naive Modulo Hashing at Scale

The simplest way to route a key to a server is:

```python
server_index = hash(key) % num_servers
```

This works when the server count is fixed. When it changes:

**Adding 1 server (3 → 4):**
```
hash("user:42") % 3 = 1  → Server 1
hash("user:42") % 4 = 2  → Server 2  ← different server!
```

With N servers and we add 1, changing `% N` to `% (N+1)` remaps roughly:
$$\frac{N}{N+1} \approx \text{all keys}$$

For 10 servers: ~91% of keys remap. For 100 servers: ~99% remap. This is catastrophic for caches (mass cache misses), session stores (users logged out), and sharded databases (wrong shard reads stale data).

**Consistent hashing solves this:** adding or removing one server remaps only $\frac{1}{N}$ of keys on average.

---

## 2. The Hash Ring

Map every server and key to a position on a ring `[0, 2^32)` using the same hash function.

```
Hash ring [0 ──────────────────────────────── 2^32)
               S1    K1  S2    K2    S3   K3
               ↑──────↑  ↑─────↑   ↑────↑
              Key K1 routes to S2 (next server clockwise)
              Key K2 routes to S3
              Key K3 routes to S1 (wraps around)
```

**Rule:** A key is owned by the first server encountered clockwise from the key's position on the ring.

### Adding a Server

```
Before: S1 ──── S2 ──── S3
Key K  sits between S1 and S2 → owned by S2

Insert S_new between S1 and S2:
After: S1 ──── S_new ──── S2 ──── S3
Keys between S1 and S_new → remapped from S2 to S_new
Keys between S_new and S2 → stay with S2 ✓
All other keys → unchanged ✓
```

Only the keys in the arc `(S1, S_new]` remap — approximately `1/N` of the total.

### Removing a Server

When S2 fails, its arc `(S1, S2]` is absorbed by S3 (next server clockwise). Only S2's keys remap; all others are unchanged.

---

## 3. Virtual Nodes (VNodes)

**Problem with basic ring:** If server positions are arbitrary hash values, the arcs are unequal in length. Some servers own large arcs (heavy load), others small arcs (underutilized).

```
Physical ring                Virtual node ring (3 vnodes each)
    S1 ──── big arc               S1a  S2b  S1b  S3a  S2a  S1c  S3b  S3c  S2c
    S2 ── small arc              (interleaved, balanced arcs per server)
    S3 ─── medium arc
```

**Virtual nodes:** Each physical server is placed at K positions on the ring using different hash inputs (e.g., `hash("S1#0")`, `hash("S1#1")`, ..., `hash("S1#K-1")`).

**Effect:**
1. **Load balance**: K vnodes per server → each server owns K arcs, averaging out to an equal share
2. **Smooth rebalancing**: when a server is added, it gets K vnodes that take keys from K different donors — no single server loses a large chunk
3. **Heterogeneous capacity**: a server with 2× the capacity gets 2K vnodes → 2× the keys

**Typical K values:** 100–300 vnodes per server (Cassandra uses 256 by default).

**Memory cost:** K vnodes × N servers entries in the sorted ring structure = K×N entries. At K=256, N=100 → 25,600 entries — trivial.

---

## 4. Consistent Hashing in Distributed Caches

### Memcached (libketama)

The original production use case. Before libketama (2007), adding a Memcached server caused ~100% cache miss storm (all keys remapped with modulo). With consistent hashing, adding one node causes only `1/N` cache misses.

**Hot path:**
```
Client hash library:
  1. hash(cache_key) → ring position
  2. Binary search for next server clockwise in sorted vnode list
  3. Send request directly to that server
No coordinator — client-side routing (O(log N) lookup)
```

### Redis Cluster

Redis Cluster does NOT use consistent hashing — it uses **hash slots** (16,384 fixed slots, `CRC16(key) % 16384`). Slots are assigned to masters. When resharding, individual slots move between masters. Keys within a slot move together.

**Why not consistent hashing in Redis Cluster?**
- Hash slots make location deterministic and auditable
- Moving a slot is a discrete, manageable operation
- The 16,384 slot granularity gives enough flexibility without the variable-arc problem

Consistent hashing IS used in Redis client-side sharding (Twemproxy, early Redis setups).

---

## 5. Consistent Hashing in Load Balancers

**Use case: stateful backends (sticky sessions, WebSocket connections)**

Round-robin and least-connections don't preserve request-to-server affinity. Consistent hashing on `client_ip` or `session_token` routes the same client to the same backend consistently:

```
NGINX upstream hash (consistent):
  hash $remote_addr consistent;
  server backend1;
  server backend2;
  server backend3;
```

When a backend is removed: only that backend's clients are remapped to other servers. Other clients stay with their server.

**Use case: L7 API Gateway routing**

Consistent hashing on `user_id` header → same user's requests always hit the same microservice instance (useful for local caches, connection state).

---

## 6. Consistent Hashing in CDNs

CDNs use consistent hashing to route requests to caching nodes within a PoP:

```
Edge PoP (20 cache servers)
Request arrives for asset X
  hash(asset_X) → cache server 7 (always)
  cache server 7 has it cached → serve from memory
  if server 7 added/removed → only 1/20 of assets remapped
```

**Why this matters:** Without consistent hashing, adding a cache node causes a miss storm on re-routed keys. Many CDNs (Akamai, Fastly) use variants of consistent hashing internally.

**Real-Aorld: Varnish Cache** uses a consistent hash ring for backend routing and cache key distribution.

---

## 7. Data Replication + Consistent Hashing (Cassandra / DynamoDB)

In Cassandra, the hash ring is used not just for routing but for **replication placement**:

```
Ring: S1 ── S2 ── S3 ── S4 ── S5
Replication factor = 3

Write key K → hash(K) lands at S2
  Primary replica:  S2
  Replica 2:        S3 (next clockwise)
  Replica 3:        S4 (next after S3)
```

Adding a node S_new between S2 and S3:
- S_new becomes primary for keys in arc `(S1, S_new]`
- Replicas automatically shift — Cassandra triggers a streaming rebalance to transfer the relevant data

This is why Cassandra can scale out with minimal disruption: consistent hashing + replication placement are co-designed.

---

## 8. Implementation: Sorted Array + Binary Search

```python
import hashlib
import bisect
from typing import Optional

class ConsistentHashRing:
    def __init__(self, nodes: list[str] = None, vnodes: int = 150):
        self.vnodes = vnodes
        self._ring: dict[int, str] = {}      # hash → node
        self._sorted_keys: list[int] = []    # sorted hash values for binary search

        for node in (nodes or []):
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        for i in range(self.vnodes):
            vnode_key = f"{node}#{i}"
            h = self._hash(vnode_key)
            self._ring[h] = node
            bisect.insort(self._sorted_keys, h)

    def remove_node(self, node: str):
        for i in range(self.vnodes):
            vnode_key = f"{node}#{i}"
            h = self._hash(vnode_key)
            self._ring.pop(h, None)
            idx = bisect.bisect_left(self._sorted_keys, h)
            if idx < len(self._sorted_keys) and self._sorted_keys[idx] == h:
                self._sorted_keys.pop(idx)

    def get_node(self, key: str) -> Optional[str]:
        if not self._ring:
            return None
        h   = self._hash(key)
        idx = bisect.bisect(self._sorted_keys, h)
        if idx == len(self._sorted_keys):
            idx = 0   # Wrap around the ring
        return self._ring[self._sorted_keys[idx]]

    def distribution(self) -> dict[str, int]:
        from collections import Counter
        return dict(Counter(self._ring.values()))
```

**Time complexity:**
- `add_node`: O(K log KN) — K inserts into sorted list of KN entries
- `remove_node`: O(K log KN)
- `get_node`: O(log KN) — binary search

For N=100 servers, K=150 vnodes: `get_node` is `O(log 15,000) ≈ 14` comparisons — negligible.

---

## 9. Hotspot Mitigation

Even with virtual nodes, a "hotkey" problem can occur: many different keys hash to the same server due to key distribution (not node distribution). Consistent hashing addresses **server load imbalance** but not **key-level hotspots**.

**Separate problem — key hotspot solutions:**
- Replication of hot keys across multiple nodes with random selection on read
- Adding a random suffix to hot keys: `user:celebrity#0`, `user:celebrity#1`, ..., `user:celebrity#9` → spreads across 10 servers

---

## 10. Comparison with Alternatives

| Approach | Remapping on add/remove | Implementation | Use case |
|---|---|---|---|
| Modulo hashing | ~100% of keys | Trivial | Fixed topology only |
| Consistent hashing | ~1/N of keys | Sorted ring + binary search | Distributed caches, LBs |
| Hash slots (Redis) | Only affected slots | Discrete slot assignment | Redis Cluster |
| Rendezvous hashing | ~1/N of keys | Score each server per key | Alternative to consistent hashing |

**Rendezvous hashing (HRW):** For each key, compute `score(key, server_i)` for all servers; assign to the server with the highest score. Adding a server only displaces keys where the new server scores highest — approximately 1/N. Simpler but O(N) per lookup vs O(log N) for ring.
