# Distributed Cache — Notes & Reference

## Redis Data Structure Cheat Sheet

### String
```redis
SET key value EX 300          # Set with 5-min TTL
GET key
INCR counter                  # Atomic increment
GETEX key EX 600              # Get + reset TTL
SETNX key value               # Set only if not exists (NX = "not exists")
```

### Hash (object fields)
```redis
HSET user:1 name "Alice" age 30
HGET user:1 name
HMGET user:1 name age         # Multi-field get
HGETALL user:1
HINCRBY user:1 login_count 1
```

### List (queue / recent items)
```redis
LPUSH queue job1 job2         # Push to head
RPUSH queue job3              # Push to tail
LPOP queue                    # Pop from head
BRPOP queue 5                 # Blocking pop (wait up to 5s)
LRANGE list 0 9               # Get first 10 items
LLEN list
```

### Sorted Set (leaderboard / rate limiting)
```redis
ZADD leaderboard 1500 player1
ZADD leaderboard 2300 player2
ZRANGE leaderboard 0 9 REV WITHSCORES   # Top 10
ZRANGEBYSCORE leaderboard 1000 2000
ZINCRBY leaderboard 100 player1
ZRANK leaderboard player1               # 0-indexed rank
```

### Set
```redis
SADD tags:post:1 python web backend
SMEMBERS tags:post:1
SISMEMBER tags:post:1 python            # O(1) membership test
SUNIONSTORE result tags:post:1 tags:post:2
SCARD tags:post:1                       # Cardinality
```

### Pub/Sub
```redis
SUBSCRIBE channel1
PUBLISH channel1 "message"
```

---

## Eviction Policy Quick Reference

| Policy | Evicts | When to use |
|---|---|---|
| `noeviction` | Nothing (OOM error) | Primary store where data loss is unacceptable |
| `allkeys-lru` | LRU from all keys | General cache |
| `allkeys-lfu` | LFU from all keys | Stable hotspot patterns |
| `volatile-lru` | LRU from keys with TTL | Cache + persistent data mix |
| `volatile-ttl` | Soonest-to-expire | When expiring data is least valuable |

---

## Persistence Modes Summary

| Mode | Command | When |
|---|---|---|
| No persistence | — | Pure cache (allow full loss on restart) |
| RDB snapshot | `SAVE` / `BGSAVE` | Periodic backup, fast restart |
| AOF | `appendonly yes` | Per-second durability (max 1s loss) |
| AOF + RDB | `aof-use-rdb-preamble yes` | Best: fast restart + recent changes |

**AOF fsync options:**
- `always` — max durability, slowest
- `everysec` — max 1 second loss, near-zero performance impact (recommended)
- `no` — OS decides, fastest but ~30s potential loss

---

## Replication and HA Patterns

### Single Primary + Replica (Sentinel)
```
Primary ──async→ Replica 1, Replica 2
Sentinel quorum promotes replica on primary failure (~30s failover)
```

Use when: single shard, HA required, simple setup.

### Redis Cluster (Sharded)
```
16,384 hash slots divided across N primary nodes
Each primary has 1+ replicas
Client routes directly to correct node based on hash slot
```

Use when: dataset > single node memory, or > 1M ops/sec.

---

## Cluster Slot Formula

```
HASH_SLOT = CRC16(key) mod 16384
```

**Hash tags** — force multiple keys to same slot:
```redis
SET {user:1}.profile  ...   # Both keys hash on {user:1} → same slot
SET {user:1}.orders   ...   # → same node → MGET is atomic
```

---

## Common Cache Patterns

### Cache-Aside (Lazy Loading)
```
read(key):
  val = cache.get(key)
  if not val:
    val = db.read(key)
    cache.set(key, val, TTL)
  return val

write(key, val):
  db.write(key, val)
  cache.delete(key)       # Invalidate, not update
```

### Write-Through
```
write(key, val):
  cache.set(key, val)
  db.write(key, val)      # Synchronous DB write
```

### Write-Behind
```
write(key, val):
  cache.set(key, val)     # Return immediately
  queue.enqueue(db_write) # Async DB write
```

---

## Key Numbers

| Fact | Value |
|---|---|
| Redis single-thread ops/sec | ~1M ops/sec |
| Redis GET latency (local) | < 0.1ms |
| Redis GET latency (network) | 0.5–2ms |
| Redis Cluster max slots | 16,384 |
| Default max memory policy | noeviction |
| Replication lag (LAN) | < 1ms |
| Sentinel failover time | ~10–30 seconds |
| AOF everysec data loss max | 1 second |
| ZipList threshold (Hash) | 128 fields, 64 bytes/value |

---

## Common Mistakes

1. **Caching mutable data without invalidation** — cache becomes stale permanently
2. **Using KEYS \* in production** — blocks event loop for seconds (use SCAN instead)
3. **Not setting TTL** — memory grows unbounded; eviction kicks in unpredictably
4. **Storing huge values** — Redis is fast for small values; 1MB+ values cause latency spikes
5. **Using SELECT (multiple DBs) for isolation** — Redis Cluster doesn't support multiple DBs; use key prefixes instead
6. **Not handling cache miss race condition** — two threads populate cache with different values
7. **Storing sensitive data without encryption** — Redis stores plaintext in memory; enable AUTH + TLS
