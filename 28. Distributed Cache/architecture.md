# Distributed Cache (Redis Clone) — Architecture

## What Problem Does a Cache Solve?

Every database read has a floor latency: even a perfectly indexed SQL query takes 1–10ms due to network RTT, connection overhead, and disk/memory access. At 100,000 reads/sec, that's 100,000 DB connections or a large connection pool — expensive.

A distributed cache intercepts reads before they hit the DB:
- Redis lookup: < 1ms for in-memory key
- Cache hit ratio 95%+ → 95% of reads never touch the DB

---

## Redis Architecture Internals

### Single-Threaded Event Loop

Redis processes commands using a **single-threaded event loop** (I/O multiplexing via epoll).

```
┌────────────────────────────────────────────────────────┐
│  Redis process (single thread)                         │
│                                                        │
│  Event loop (epoll):                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐ │
│  │ Socket 1 │  │ Socket 2 │  │ Timer events         │ │
│  │ (client) │  │ (client) │  │ (EXPIRE, AOF flush)  │ │
│  └────┬─────┘  └────┬─────┘  └─────────┬────────────┘ │
│       └─────────────┴──────────────────┘              │
│                 Command queue: FIFO executed in order  │
└────────────────────────────────────────────────────────┘
```

**Why single-threaded works:**
- All operations are O(1) or O(log N) and take microseconds
- No CPU bottleneck — bottleneck is network bandwidth
- No locking needed → operations are atomic by design
- A single Redis instance handles ~1M ops/sec on modern hardware

**Caveat:** Since Redis 6.0, I/O is multi-threaded (separate threads for read/write syscalls), but command execution is still single-threaded.

---

## Data Structures and Their Internals

| Structure | Internal Representation | Commands | Use Case |
|---|---|---|---|
| String | SDS (Simple Dynamic String) | GET/SET/INCR/GETEX | Cached objects, counters |
| List | QuickList (linked list of zip lists) | LPUSH/RPUSH/LPOP/LRANGE | Message queues, recent items |
| Hash | ZipList (small) / HashTable (large) | HGET/HSET/HMGET | Object fields |
| Set | IntSet (small) / HashTable (large) | SADD/SMEMBERS/SINTER | Unique visitors, tags |
| Sorted Set | ZipList (small) / SkipList + HashTable | ZADD/ZRANGE/ZRANGEBYSCORE | Leaderboards, rate limiting |
| Bitmap | String (bit array) | SETBIT/GETBIT/BITCOUNT | Feature flags, daily active users |
| HyperLogLog | String (probabilistic) | PFADD/PFCOUNT | Approximate unique count |
| Stream | RadixTree + listpack | XADD/XREAD/XGROUP | Event log, message streaming |

**ZipList → OBJ encoding thresholds:**
- Hash: ≤ 128 fields and all values ≤ 64 bytes → ZipList (compact); else HashTable
- Sorted Set: ≤ 128 members → ZipList; else SkipList

---

## Distributed Cache Cluster: Redis Cluster

### Hash Slots

Redis Cluster divides the keyspace into **16,384 hash slots**.

```
HASH_SLOT = CRC16(key) mod 16384
```

Example 3-node cluster:
```
Node A: slots 0–5460
Node B: slots 5461–10922
Node C: slots 10923–16383
```

When a client writes `SET user:123 data`:
- CRC16("user:123") % 16384 = 7638 → Node B

Each node is primary for its range. Clients maintain a routing table and send commands directly to the correct node.

### Resharding

Adding a 4th node D: move slots 0–4095 from A to D and 5461–9000 from B to D. During migration, reads for migrating slots go to both source and destination. **Minimal downtime.**

### Failover

Each primary has 1+ replica nodes. If primary fails, replicas elect a new primary via Raft-based internal consensus. Client gets `MOVED` or `ASK` errors during failover (brief window).

---

## Replication: Leader-Follower

```
Primary (writes) → asynchronous replication → Replica 1, Replica 2
```

- Replication is **asynchronous** — potential for data loss on primary crash before replica sync
- `WAIT N 0` command: block until N replicas have ACKed the latest write (semi-sync for critical paths)
- Replicas serve reads (read scaling)

### Redis Sentinel (HA without Cluster)

For single-shard deployments needing HA:
```
Sentinel 1, Sentinel 2, Sentinel 3  (quorum = 2)
    ↓ monitor
  Primary ──asynchronous→ Replica
```

If Sentinel quorum agrees primary is down → promotes replica → clients redirect. Failover time: 10–30 seconds.

---

## Eviction Policies

When Redis reaches `maxmemory`:

| Policy | Behavior |
|---|---|
| `noeviction` | Return error on writes (default) |
| `allkeys-lru` | Evict least recently used key from all keys |
| `allkeys-lfu` | Evict least frequently used key |
| `volatile-lru` | LRU eviction only among keys with TTL |
| `volatile-ttl` | Evict key with nearest expiration first |
| `allkeys-random` | Random eviction |

**For a cache:** `allkeys-lru` or `allkeys-lfu`. LFU is better for stable hotspot patterns (Zipf distribution); LRU is better when recency matters more.

---

## Persistence Options

| Mode | Durability | Performance impact | Recovery time |
|---|---|---|---|
| None (pure cache) | Zero | Best | N/A (all keys lost on restart) |
| AOF (Append-Only File) | Per-command or per-second | Moderate | Replay all AOF from beginning |
| RDB (Snapshot) | Point-in-time | Fork overhead every N min | Restore snapshot |
| AOF + RDB (hybrid) | Best | Moderate | RDB + recent AOF tail |

**For a cache (not primary store):** No persistence needed — cache miss → DB refill.  
**For a primary store (session data, counters):** AOF with `fsync=everysec` — max 1 second data loss.

---

## Cache Invalidation Strategies

| Strategy | How it works | Stale window | Write overhead |
|---|---|---|---|
| TTL (passive) | Key expires after N seconds | Up to TTL | None |
| Write-through | Update DB + update cache on every write | None | Double write |
| Write-behind | Update cache; async flush to DB | Possible DB lag | Low (async) |
| Cache-aside (lazy) | Read: miss → DB → set cache. Write: write DB + DEL cache. | None | Extra DEL |
| Event-driven | DB writes emit events → consumer invalidates cache | Near-zero | Complex setup |

**Cache-aside + DELETE-on-write** is the standard pattern for web applications. Never update the cache on write (race condition between cache update and DB write).

---

## Cache Stampede (Thundering Herd)

**Problem:** Popular key TTL expires → 10,000 concurrent requests all miss cache simultaneously → 10,000 concurrent DB queries → DB overloaded.

**Solutions:**

1. **Early expiration / probabilistic refresh:**
```python
# When key will expire within 10% of its TTL, probabilistically refresh it early
remaining_ttl = await redis.ttl(key)
if remaining_ttl < original_ttl * 0.1:
    if random.random() < 0.1:   # 10% of requests trigger early refresh
        await refresh_cache(key)
```

2. **Lock-based refresh (mutex):**
```python
lock = await redis.set(f"lock:{key}", "1", nx=True, ex=5)
if lock:
    value = await db.fetch(...)
    await redis.set(key, value, ex=ttl)
    await redis.delete(f"lock:{key}")
else:
    await asyncio.sleep(0.05)  # Wait for refresher
    value = await redis.get(key)  # Try again
```

3. **Background refresh daemon:** A separate job watches keys with `OBJECT IDLETIME` and proactively refreshes before expiry. Never let popular keys expire cold.
