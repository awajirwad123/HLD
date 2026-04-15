# Consistent Hashing — Tricky Interview Questions

Questions designed to expose shallow understanding. Each tests a specific misconception or edge case that interviewers frequently probe.

---

## Question 1: "You said consistent hashing remaps only 1/N keys. But a key that used to go to Server A now goes to Server D instead of Server A. Isn't that a problem for cache consistency?"

**The trap:** Conflating *routing change* with *data corruption*. Interviewer wants to see you distinguish the cache cold-miss problem from a correctness problem.

**Strong answer:**

It's not a consistency problem — it's a **cold-miss problem**. Cache entries have no inherent owner; they're just copies of data from the source of truth. When a key remaps from A to D, D simply has a cold miss on the first request. It fetches from the database, caches, and all subsequent requests hit. The data is never incorrect.

The operational concern is: with modulo hashing, ~(N-1)/N keys remap simultaneously → **cache stampede**, which can overwhelm the database. With consistent hashing, only ~1/N keys remap → a controlled warm-up period.

If I needed to avoid even these cold misses, I would **pre-warm** the new server by reading the keys from the predecessor node's cache before bringing the new node live. This is what some cache management systems call "cache migration."

---

## Question 2: "If virtual nodes fix the load imbalance, why not just use K=10,000 vnodes instead of K=200? More is better, right?"

**The trap:** Assuming there's no cost to increasing K.

**Strong answer:**

There are real trade-offs:

1. **Memory:** Ring occupies `K × N × (hash_size + pointer)` bytes in memory. At K=10,000 and N=100 servers, that's 1 million ring entries — a few MB, but it's held in every client process.

2. **Add/remove latency:** `add_node` does K `bisect.insort` calls — O(K log KN) per operation. At K=10,000, adding a node requires 10,000 sorted inserts. This is a paused operation on the client.

3. **Diminishing returns:** Load balance variance decreases with K as O(1/√K). Going from K=100 to K=200 cuts variance significantly. Going from K=1,000 to K=10,000 gives marginal improvement. K=150–300 is the practical sweet spot where diminishing returns begin.

Practical choice: K=150–200 for typical clusters. Increase slightly for heterogeneous capacity (so the minimum weight server still has ≥50 vnodes).

---

## Question 3: "Redis Cluster has 16,384 slots and handles node changes. Is it doing the same thing as consistent hashing?"

**The trap:** Assuming all "sharding" systems are equivalent. Interviewers often ask this specifically because candidates frequently say "Redis uses consistent hashing."

**Strong answer:**

No — Redis Cluster uses **hash slots**, which is a different model:

| Property | Consistent Hashing | Redis Cluster Hash Slots |
|---|---|---|
| Slot assignment | Implicit (ring position) | Explicit (cluster configuration) |
| Node change | Automatic key remapping | Manual `CLUSTER REBALANCE` or explicit slot migration |
| Multi-key operations | Not guaranteed co-location | Use `{tag}` to force same slot |
| Routing info | Each client maintains ring | Slots gossip — any node returns MOVED/ASK redirect |

Redis chose hash slots because: (1) explicit slot assignment enables deterministic multi-key atomics, (2) slot migration can be rate-limited and monitored, and (3) 16,384 slots are a small enough table to gossip efficiently across the cluster.

The trade-off: adding or removing a node requires operational intervention (rebalancing). In a consistent hash ring, it's transparent. For a cache, consistent hashing is usually better. For a stateful database like Redis, the explicit control of hash slots is worth it.

---

## Question 4: "In your Cassandra replication design, you walk the ring clockwise to find RF=3 replicas. What happens when two adjacent virtual nodes on the ring belong to the same physical node?"

**The trap:** Vnode-aware replication requires skipping duplicate physical nodes — many candidates describe the basic ring walk without this guard.

**Strong answer:**

That's exactly the case that vnodes create constantly — a single physical node has K=200 positions scattered around the ring, so adjacent vnodes often belong to the same server.

The `get_replicas` walk **must track seen physical nodes** and skip any vnode belonging to an already-selected physical node:

```python
def get_replicas(self, key, n):
    h = self._hash(key)
    start = bisect.bisect_right(self._ring, h) % len(self._ring)
    seen_physical = set()
    replicas = []
    for offset in range(len(self._ring)):
        idx = (start + offset) % len(self._ring)
        physical_node = self._map[self._ring[idx]]
        if physical_node not in seen_physical:
            seen_physical.add(physical_node)
            replicas.append(physical_node)
        if len(replicas) == n:
            break
    return replicas
```

Without this skip, a key could end up with all 3 replicas on the same physical machine — defeating the entire purpose of replication. In production Cassandra this is further extended to **rack-aware** and **data-center-aware** placement: skip additional vnodes until a replica lands on a different rack (and in multi-DC mode, different data centers).

---

## Question 5: "Your distributed cache service receives 500,000 requests/second. An operations engineer accidentally removes a cache node from the ring config. What exactly happens in the next 5 seconds, and how would you prevent or mitigate it?"

**The trap:** Tests operational understanding, not just algorithmic knowledge.

**Strong answer:**

**What happens immediately:**

1. All client instances that receive the updated ring config now route the affected ~1/N of keys (say 100,000 RPS) to the neighbor node clockwise.
2. That neighbor node has a cold miss for every one of those keys → 100,000 RPS hits the database in the next few seconds while the neighbor warms up.
3. The database, designed for perhaps 10,000 RPS cache-miss traffic, receives 10× its intended load → latency spikes, possibly connection pool exhaustion → cascading failures.

**Mitigations:**

- **Staggered config rollout:** Don't propagate ring changes to all clients simultaneously. Roll out to 1% of clients first, monitor miss rate spike, then continue.
- **Read-through with circuit breaker:** If the database latency spikes, open a circuit breaker on cache misses to prevent complete database saturation.
- **Two-layer caching:** L1 (local in-process) cache absorbs hot keys. The ring change only affects L2 (distributed Memcached). L1 continues serving hot keys for 30–60 seconds during L2 remapping.
- **Shadow routing:** New node serves requests but also forwards to the old node if it has a miss, silently warming itself from the neighbor before the config is fully committed.
- **Pre-warming script:** Before removing a node, copy its top-N keys to the successor node.

---

## Question 6: "Consistent hashing guarantees ~1/N remapping. But you just said we should use K=200 vnodes. With 200 vnodes, isn't the variance in which keys remap much higher — some servers absorbing 5% of keys from the removed node, others absorbing 15%?"

**The trap:** Distinguishes the *average* remapping guarantee from the *distribution* of absorbed load.

**Strong answer:**

Correct — vnodes introduce variance in which server absorbs the load from a removed server. The 1/N is an *expected value*. With K=200 vnodes and N=10 servers, each server owns about 200 of the 2,000 ring positions. When a server is removed, its 200 positions are scattered around the ring, and its keys route to whichever server was clockwise of each position.

In theory, each of the 9 remaining servers should absorb about 200/1800 ≈ 11% of the removed server's keys. In practice, variance over 200 samples means one server might absorb 14% and another 8% — not catastrophic, but unequal.

This is why at high K values (K=500+), the distribution becomes substantially more uniform. For high-stakes productions (e.g., a Memcached fleet powering a large payment page), operators sometimes use explicit **slot assignment** (Redis-style) rather than virtual nodes — manually assigning equal-sized arc segments to each server — to eliminate variance entirely at the cost of automation.

---

## Question 7: "Interviewer shows you this code: `server = servers[hash(key) % len(servers)]`. You mention consistent hashing as a fix. They ask: isn't this over-engineering? The servers never change."

**The trap:** Tests whether you can argue proportionality and think about failure modes, not just mechanics.

**Strong answer:**

If the cluster topology truly never changes — static number of servers, no auto-scaling, no node failures — then yes, modulo hashing is simpler and sufficient. Don't over-engineer.

But I'd push back on "never changes" assumptions:

1. **Node failures:** Even in a "static" cluster, a server will eventually crash, be rebooted for patches, or fail a health check. When that happens with modulo hashing, the application must either (a) remove it from the list (causing mass remapping) or (b) keep it in the list and return errors for 1/N of keys. Neither is graceful.

2. **Scaling events:** Even manually-scaled systems eventually need to change size. Consistent hashing makes that operation near-zero-impact; modulo requires cache warm-up planning.

3. **The cost of consistent hashing is very low:** ~50 lines of code, O(log KN) lookup instead of O(1). For a cache cluster this is the right default even in "stable" systems.

I'd frame it as: consistent hashing is the standard for distributed caches not because topology changes frequently, but because when it *does* change — even due to failure — the impact should be minimal by design.
