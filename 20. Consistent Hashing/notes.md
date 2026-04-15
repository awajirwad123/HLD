# Consistent Hashing — Notes & Reference Sheet

## 1. The Problem with Modulo Hashing

```
server = hash(key) % N
```

Adding or removing one server changes N → almost every key remaps:

| N (before) | N (after) | Keys remapped |
|---|---|---|
| 3 | 4 | ~75% |
| 10 | 11 | ~91% |
| 100 | 101 | ~99% |

For a cache cluster this means a **cache avalanche** — essentially all requests miss simultaneously and hammer the database.

---

## 2. The Hash Ring

- Map server names to positions on a circular ring `[0, 2^32)` using `MD5(server_name)`
- Map each key to a position using `MD5(key)`
- Route key to the **first server clockwise** from the key's position

```
                0
              ┌───┐
    A (hash)  │   │  D (hash)
              │   │
   ─── key ──►│   │──► routes to D (first clockwise server)
              │   │
              │   │  C (hash)
              │   │
              └───┘
            2^32 - 1
```

When a server is added/removed, only the keys between the predecessor and that server's position are affected → ~1/N remapping.

---

## 3. Virtual Nodes (Vnodes)

**Problem with basic ring:** Servers land at non-uniform positions → some own large arcs, some small → unbalanced load.

**Solution:** Give each physical server K positions on the ring:

```
Physical server A → positions: MD5("A#0"), MD5("A#1"), ..., MD5("A#199")
```

**Effect:**
- Law of large numbers evens out load across all physical servers
- K=150–300 gives ~5–10% load imbalance in practice
- Larger K = better balance, higher memory usage (K×N ring entries)

**Capacity weighting:** `vnodes = base_vnodes × capacity_weight` — a server with 2× RAM gets 2× the vnodes → 2× the keys.

---

## 4. Key Math

| Scenario | Fraction of keys remapped |
|---|---|
| Add 1 server (N→N+1) | ≈ 1/(N+1) |
| Remove 1 server (N→N-1) | ≈ 1/N (only keys on removed server) |
| Modulo add 1 server | ≈ (N-1)/N |
| Modulo remove 1 server | ≈ (N-1)/N |

**Ring size:** `|ring| = K × N` total vnode positions
- K=200 vnodes, N=10 nodes → 2,000 ring entries
- Each `get_node` lookup: `bisect.bisect_right` → **O(log KN)**

---

## 5. Python Implementation Cheat Sheet

```python
import bisect, hashlib

class ConsistentHashRing:
    def __init__(self, vnodes=200):
        self.vnodes = vnodes
        self._ring: list[int] = []       # sorted positions
        self._map:  dict[int, str] = {}  # position → node name

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)

    def add_node(self, node: str) -> None:
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h not in self._map:
                bisect.insort(self._ring, h)   # O(N) insert — amortized OK
                self._map[h] = node

    def remove_node(self, node: str) -> None:
        for i in range(self.vnodes):
            h = self._hash(f"{node}#{i}")
            if h in self._map:
                self._ring.remove(h)           # O(N) — acceptable for rare ops
                del self._map[h]

    def get_node(self, key: str) -> str:
        h = self._hash(key)
        idx = bisect.bisect_right(self._ring, h) % len(self._ring)
        return self._map[self._ring[idx]]      # O(log KN)

    def get_replicas(self, key: str, n: int) -> list[str]:
        """Walk ring clockwise to find n distinct physical nodes."""
        h = self._hash(key)
        start = bisect.bisect_right(self._ring, h) % len(self._ring)
        seen, result = set(), []
        for i in range(len(self._ring)):
            node = self._map[self._ring[(start + i) % len(self._ring)]]
            if node not in seen:
                seen.add(node); result.append(node)
            if len(result) == n:
                break
        return result
```

---

## 6. Real-World Reference

### Memcached / libketama
- Client-side consistent hashing; `ketama_points = 160` vnodes per server
- Client library routes based on MD5 of key
- **No server coordination** — each client independently routes to the same node for the same key

### Redis Cluster — NOT consistent hashing
- Uses **hash slots**: `CRC16(key) % 16384` → 16,384 slots
- Slots are assigned to nodes; the cluster gossips slot-to-node mapping
- Adding a node: slots are *manually migrated* (CLUSTER REBALANCE command)
- Why not consistent hashing? Redis wanted deterministic slot assignment and multi-key operations (all keys in same slot can be atomic — use `{tag}` to force slot)

### NGINX Consistent Hash (Load Balancer)
```nginx
upstream backend {
    hash $request_uri consistent;   # or $remote_addr for session affinity
    server backend-1;
    server backend-2;
    server backend-3;
}
```
- Upstream server added/removed: only 1/N requests re-route (minimizes cache misses on upstream caches)

### Cassandra / DynamoDB — Replication on Ring
- RF=3: key routes to its primary node, then the next 2 distinct physical nodes clockwise
- Replication is implicit in ring position — no central metadata needed
- Data center awareness: rack-aware placement skips same-rack nodes while walking clockwise

### CDN — Intra-PoP Cache Routing
- A PoP has 5–10 cache servers; content is striped across them via consistent hashing on URL
- When a cache server is added, only ~1/N of cached objects need to migrate

---

## 7. Rendezvous Hashing (HRW) vs. Consistent Hashing

| Property | Consistent Hashing | HRW |
|---|---|---|
| Lookup time | O(log KN) | O(N) |
| Load balance | ±10–15% (vnodes) | Near-perfect, no config |
| Config | Choose K | None |
| Remapping on change | ~1/N | ~1/N |
| Replication | Walk ring clockwise | Run with exclusion set |
| Best for | Large N (>50 nodes), caches | Small N (<50 nodes), CDN origin selection |

**HRW rule:** `server = argmax_s hash(key || s)` — highest score wins.

---

## 8. Common Interview Numbers

| Metric | Value |
|---|---|
| Typical vnode count K | 150–300 |
| Ring positions (K=200, N=10) | 2,000 |
| Lookup complexity | O(log KN) |
| Keys remapped on +1 node | ~1/(N+1) |
| Redis hash slots | 16,384 (CRC16 % 16384) |
| Cassandra default RF | 3 |
| libketama vnodes per server | 160 |

---

## 9. Quick Diagram

```
      Key X hashes here ──┐
                           ▼
  ──────────────────────── Ring ─────────────────────────►
                           │ (clockwise)
  ... vnode C2 ─── vnode A7 ─── vnode B3 ─── vnode A8 ...
                   ▲
                   Key X routes to Node A (first server clockwise)

  Adding Node D: inserts D's K vnodes into the ring.
  Keys between predecessor vnode and each D-vnode that previously
  went to another node now go to D. Only these keys are affected.
```
