# Consistent Hashing — Quick-Fire Interview Q&A

**Format:** Read the question, answer out loud, then check the answer below.

---

**Q1. What is the key problem with `hash(key) % N` when a server is added?**

When N changes, almost all keys remap to different servers. Adding 1 server to a 10-server cluster remaps ~91% of all keys — causing a cache avalanche where all remapped requests simultaneously miss and hit the database.

---

**Q2. How does consistent hashing limit key remapping when N changes?**

Keys map to a ring `[0, 2^32)`. A key routes to the first server clockwise from its position. When you add a server, only the arc between that server and its predecessor changes ownership — about 1/(N+1) of keys. Removing remaps only 1/N keys (those that were on the removed server).

---

**Q3. What is a virtual node and why is it needed?**

A virtual node (vnode) is one of K ring positions assigned to a physical server. Without vnodes, servers land at non-uniform positions on the ring — some own large arcs, some tiny ones — causing load imbalance. With K=150–300 vnodes, the law of large numbers produces near-uniform load distribution.

---

**Q4. You have three servers: small (4 GB RAM), medium (8 GB RAM), large (16 GB RAM). How do you weight them in a consistent hash ring?**

Assign vnodes proportionally to capacity. If base=100, set small=100, medium=200, large=400 vnodes. The large server will own ~4/7 ≈ 57% of the ring and receive 57% of keys — proportional to its 16 GB vs. 28 GB total capacity.

---

**Q5. What is the time complexity of a key lookup in a consistent hash ring with K vnodes and N servers?**

`O(log KN)` — a binary search (`bisect_right`) over `K × N` sorted ring positions. For K=200, N=10 that's log₂(2000) ≈ 11 comparisons.

---

**Q6. Does Redis Cluster use consistent hashing? Why or why not?**

No. Redis Cluster uses **hash slots**: `CRC16(key) % 16384` → 16,384 slots, each assigned to a specific node. Unlike consistent hashing, slot assignment is explicit and managed by the cluster. This enables multi-key atomic operations: use `{tag}` to force related keys into the same slot.

---

**Q7. You add a new server to a Memcached cluster. How many cache requests will miss immediately after the change?**

Approximately 1/(N+1) of all requests — only those whose keys remap to the new server. With 10 servers, ~9% of traffic will cold-miss while the new server warms up. Compare to modulo hashing where ~91% would miss.

---

**Q8. In Cassandra with RF=3 on a 6-node ring, how does the ring determine which nodes store a key?**

The primary replica is the first node clockwise from the key's hash position. Then walk clockwise, skipping vnodes of already-seen physical nodes, until 2 more unique physical nodes are found — those are the secondary replicas. All 3 together form the replica set.

---

**Q9. What is rendezvous hashing (HRW)?**

For each key, score every server with `score = hash(key + server_name)`. The server with the highest score wins. No ring structure needed — in Python, `max(servers, key=lambda s: hash(key+s))`. It provides the same ~1/N remapping property as consistent hashing with simpler code, but O(N) lookup instead of O(log KN).

---

**Q10. When would you prefer HRW over consistent hashing?**

When N is small (< 50 nodes). HRW's O(N) is perfectly fine for 5–20 nodes and requires zero configuration (no vnode tuning). Consistent hashing's O(log KN) advantage matters at hundreds of nodes, e.g., a large Memcached fleet.

---

**Q11. Two vnode positions hash to exactly the same value on the ring. What happens?**

A collision. The implementation must handle this — the simple `list.remove(h)` will only remove one occurrence. Robust implementations use a dict with `position → node` and skip duplicate hash values (`if h not in self._map`) during `add_node`. In practice, MD5 collisions for less than 10,000 vnodes are astronomically rare.

---

**Q12. A server goes down in a consistent hash ring. Where do its keys go?**

Each of its keys routes to the next server clockwise on the ring. Because the failed server's vnodes are scattered around the ring, its load redistributes across all remaining servers roughly equally — each surviving server absorbs ~1/(N-1) of the failed server's traffic.

---

**Q13. How does NGINX use consistent hashing in its upstream module?**

`hash $remote_addr consistent;` in the upstream block routes each client IP to a specific backend using a consistent hash ring. If a backend goes down, only the ~1/N of clients that were hashed to it will re-route; the rest continue hitting their usual backend, preserving connection affinity.

---

**Q14. What is the "hot spot" problem in consistent hashing and how do you mitigate it?**

A specific key or small key range receives disproportionate traffic (e.g., a celebrity post). Virtual nodes help distributional imbalance across servers, but they don't fix logical hot keys. Mitigation at the key level: shard the hot key across multiple nodes using a suffix (`hot_key:0`, `hot_key:1`, ... `hot_key:N-1`) and fan out reads, or cache the hot key at the edge (CDN/L1).

---

**Q15. What is the minimum number of vnodes K to achieve reasonable load balance?**

Empirically, K ≥ 100 per server gives ≤15% load imbalance (std dev / mean). K=150–200 is standard in production. Below K=50, you'll see occasional servers with 2–3× the expected load depending on hash clustering. Libketama (Memcached) uses K=160.

---

**Q16. If you're designing a distributed cache for 1 million keys across 10 servers, and you need to add servers one at a time during traffic, what is the maximum number of simultaneous cache misses you should expect per addition?**

~100,000 keys (1/N+1 ≈ 1/11 ≈ 9%). With consistent hashing, these 100k keys will cold-miss and hit origin once each. You can mitigate this by "warming" the new server from neighbor nodes before switching traffic.

---

**Q17. Why might you hash both the key AND the server name when computing scores in HRW?**

To ensure statistical independence — if only the key is hashed and compared across servers, all servers would score identically for any key. Combining both produces a unique score per (key, server) pair, creating a deterministic total ordering across all servers for each key.

---

**Q18. In a CDN, how is consistent hashing used within a single Point of Presence?**

A PoP has multiple cache servers. Incoming requests for a URL are routed to a specific cache server via consistent hashing on the URL. This ensures the same URL is always served by the same server, maximizing the cache hit rate. If a cache server fails, only its ~1/N of URLs need to re-fill from origin.

---

**Q19. What happens to Cassandra's replica placement if you have 3 nodes but RF=5?**

Cassandra will place as many replicas as available physical nodes (3 replicas on 3 nodes). The `get_replicas` ring walk stops once it has visited all physical nodes. Theoretically RF should not exceed N — the Cassandra documentation warns against this as it provides no additional fault tolerance.

---

**Q20. How do you detect whether a consistent hash implementation is balanced without running traffic?**

Build the ring, route a large sample of synthetic keys (e.g., 100,000), count keys per node, and compute coefficient of variation (std dev / mean). A well-tuned ring with K=200 should show CV < 0.10 (10%). Alternatively, count the arc lengths (distances between consecutive vnode positions on the ring) owned by each physical server.
