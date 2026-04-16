# Consistent Hashing — Interview Simulator

Three full mock interview scenarios. For each: read the prompt, close the answer, explain your design out loud for 10–15 minutes, then compare.

---

## Scenario 1: Design a Distributed Cache for a Social Feed (Caching Layer Focus)

**Interviewer prompt:**

> "We have a social network with 50 million daily active users. Each user's feed is cached after computation. We have 20 cache servers. Occasionally we need to add or remove cache servers for maintenance or scaling. Walk me through how you'd ensure cache keys route to the right server, and what happens operationally when a server changes."

---

### Strong Answer

**Problem framing:**

50M DAU with feed caches means our cache tier likely handles hundreds of thousands of reads per second. Cache efficiency is critical — a high miss rate sends expensive feed computation requests to the database / ML ranking service.

The core challenge: cache routing must be deterministic (same key → same server every time), but must also gracefully handle servers being added, removed, or failing.

**Reject modulo hashing immediately with a number:**

`server = hash(user_id) % 20` — if we change to 21 servers, `hash(x) % 20 ≠ hash(x) % 21` for ~95% of keys. With 50M users and an 80% cache hit rate that means ~40M users would experience a cold miss simultaneously → complete database overwhelm. This is a cache stampede.

**Consistent hashing design:**

Map the 20 servers onto a hash ring `[0, 2^32)`:
- Each server gets K=200 virtual nodes: `MD5("server-i#j")` for j in 0..199
- Each key maps to `MD5(user_id)` → route clockwise to the first vnode → that's the cache server
- Ring lives in client memory (e.g., in our cache client library) — no central coordinator needed

**Adding a server:**
- New server's K=200 vnodes are inserted into the ring
- ~1/21 ≈ 4.8% of user IDs now route to the new server — those will cold-miss once and then warm up
- All other 95.2% of users are unaffected — no cache stampede
- Gradual warm-up: the new server goes from 0% to steady-state hit rate over ~30 minutes as keys populate

**Removing a server:**
- Server's 200 vnodes are removed from the ring
- That server's keys (~5%) route to their clockwise neighbor — those neighbors cold-miss and warm up
- Graceful way: before removing, copy the server's hot keys to its successor (pre-warming)

**Client configuration:**
Each app instance (Python/Java/Go) holds the ring in memory. Config changes (add/remove server) are distributed via a configuration service (e.g., ZooKeeper or a simple Redis config key + version number). Clients watch for config updates and reload the ring — use a small window (e.g., 5-second propagation delay) since brief client inconsistency during transition is acceptable for a cache.

**Failure handling:**
Health checks run every 5 seconds. When a server fails health checks 3× consecutive, it's removed from the ring config. This triggers the same 1/N remapping as intentional removal — but the operation is automatic and the impact is bounded.

**Replication (if asked about durability):**
For feeds, we probably don't need cache replication — a cache miss just means we recompute the feed, which is expensive but not catastrophically wrong. If we did want replicas, we could use `get_replicas(user_id, n=2)` to route reads and writes to 2 nodes — a trade-off between redundancy cost and simplicity.

**Summary numbers to mention:**
- 20 servers × 200 vnodes = 4,000 ring positions; lookup O(log 4000) ≈ 12 comparisons
- Topology change affects ~1/N ≈ 5% of traffic vs ~95% for modulo
- Ring fits in ~100KB of memory per client process

---

## Scenario 2: Design a Database Sharding Strategy (Cassandra-Style)

**Interviewer prompt:**

> "We're building a time-series telemetry database — billions of sensor readings per day. Data is distributed across 12 nodes. We need strong partition tolerance, and we need RF=3 (3 replicas of every record). How would you shard data and place replicas? What happens when we add a 13th node?"

---

### Strong Answer

**Sharding key selection first:**

Sensor readings should be sharded by `(sensor_id, timestamp_bucket)` — co-locating a sensor's recent readings on the same node enables efficient range scans for time-series queries. The partition key is `sensor_id` for ring placement; within a partition, rows are sorted by timestamp.

**Consistent hashing for partition placement:**

`MD5(sensor_id) % 2^32` maps each sensor to a position on the ring. The sensor's data lives on whichever node's arc contains that hash position.

With 12 nodes × 200 vnodes = 2,400 ring positions, the ring is partitioned into 2,400 small arcs, each owned by a physical node.

**RF=3 replica placement:**

Starting from the key's hash position, walk clockwise and collect the next 3 **distinct physical nodes**. This is critical: because of vnodes, adjacent ring positions may belong to the same physical node — must skip them:

```
for each vnode clockwise from key.hash:
    if vnode.physical_node not in replica_set:
        replica_set.add(vnode.physical_node)
    if len(replica_set) == 3:
        break
```

Result: every write is sent to 3 different physical machines. With `QUORUM` write consistency (`floor(RF/2)+1 = 2`), the write succeeds once 2 of 3 replicas acknowledge.

**Rack awareness (production consideration):**

If servers span 3 racks, extend the walk to ensure each replica lands on a different rack. This prevents rack-level failure from taking down all 3 replicas. Cassandra's `NetworkTopologyStrategy` implements this. Multi-DC: ensure 1+ replica per data center.

**Adding a 13th node:**

1. New node joins, announces its ID to the cluster (gossip protocol in Cassandra)
2. Its K=200 vnodes are assigned positions on the ring via consistent hashing on its node ID
3. For each of its new ring positions, the predecessor node transfers the data it owns up to that position → only ~1/13 ≈ 7.7% of data migrates
4. During migration: new node serves requests from its position but forwards misses to the predecessor (the predecessor is still authoritative for those keys until migration completes)
5. Migration can be throttled to avoid overwhelming network I/O — Cassandra's `nodetool rebuild` does this

**Query routing:**

Any node can serve as a query coordinator. Client sends read for `sensor_id=X` to any node. That coordinator:
1. Computes hash of `sensor_id=X`
2. Identifies the 3 replica nodes from its local ring view
3. Forwards read to the replica with lowest latency (Cassandra's dynamic snitch)
4. With `LOCAL_QUORUM`, waits for 2 responses — returns the result with the highest timestamp

**What interviewers look for:**
- You explicitly mention *skipping same-physical-node vnodes* in replica placement
- You know the difference between partition key (for ring lookup) and clustering key (for within-partition ordering)
- You mention data migration throttling and the transient dual-ownership period

---

## Scenario 3: CDN Cache Routing Within a PoP (Google / Cloudflare Style)

**Interviewer prompt:**

> "You're designing the internal routing logic for a CDN's Point of Presence (PoP). The PoP has 8 cache servers. When a request arrives for a URL, you need to route it to a specific cache server so the content is cached consistently. How do you approach this, and what happens during cache server maintenance?"

---

### Strong Answer

**Why consistent routing matters in a CDN PoP:**

A PoP handles, say, 200,000 requests/second for thousands of distinct URLs. If each request could go to any server (round-robin), each server would need a full copy of every cached object → 8× storage cost, and the cache hit rate per server is lower.

With consistent routing: each URL is always served by the same server → that server maintains a warm cache for that URL → high hit rate, 1/8 of the storage per URL.

**Consistent hashing design:**

```
server = ring.get_node(URL)
```

- 8 servers × K=200 vnodes = 1,600 ring positions
- Lookup: `bisect_right(ring, MD5(URL)) → clockwise first server`
- Client: the PoP's request router (reverse proxy — nginx, Varnish); maintains the ring in memory
- Config: ring updated via local config management (Ansible, Puppet) when servers change

**URL normalization (important detail):**

Before hashing, normalize the URL: lowercase scheme and host, strip default port, sort query parameters canonically. This ensures `HTTP://Example.com/page` and `http://example.com/page` route to the same server. Inconsistent normalization → cache splits → reduced hit rate.

**Cache server added (horizontal scale-out):**

- New server's vnodes are inserted into the ring
- ~1/9 ≈ 11% of URLs now map to the new server — those URLs are cold on it
- The new server fetches those URLs from origin on the first miss, caches, and serves subsequent requests from cache
- The previously warm server no longer serves those URLs, potentially freeing memory for other content
- Total impact: ~11% of requests experience one additional origin fetch — controlled and bounded

**Cache server maintenance (graceful drain):**

1. Before removing, mark server as `draining` — new requests route to its ring-successor
2. Outstanding connections complete naturally
3. Actively pre-warm the successor: proxy tells it to pre-fetch the top-N objects from the draining server
4. Remove the server's vnodes from the ring
5. Full cut-over — only the trailing long-tail objects miss (infrequently requested URLs)

**Cache server failure (ungraceful):**

Health check fails → server removed from ring immediately. Its 11% of URLs cold-miss until the successor warms up. Rate of origin fetches spikes by 11% temporarily. If origin can't absorb the spike, serve stale content (if TTL allows) — `Cache-Control: stale-if-error` header.

**Alternative to consistent hashing for small N:**

With only 8 servers, rendezvous hashing (HRW) is equally valid: `score = hash(URL + server_name)`, pick highest score. Zero configuration, same remapping property, O(N=8) lookup per request — negligible cost. Some CDN implementations (Varnish) use this exact approach.

**What interviewers look for:**
- URL normalization mentioned (common miss)
- Graceful drain / pre-warming vs. crash failure distinguished
- You can argue for either consistent hashing or HRW for 8 nodes — interviewers respect both if justified
- You mention stale-on-error as a resilience mechanism
