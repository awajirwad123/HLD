# Replication — Notes & Cheat Sheets

## Replication Models — One-Line Each

| Model            | Who accepts writes?         | Who accepts reads?          | Used by                       |
|------------------|-----------------------------|-----------------------------|-------------------------------|
| Single-leader    | One primary only            | Primary + replicas          | PostgreSQL, MySQL, MongoDB, Redis |
| Multi-leader     | Multiple leaders            | Any leader or replica       | CockroachDB, MySQL Group Replication, active-active setups |
| Leaderless       | Any node                    | Any node                    | Cassandra, DynamoDB, Riak     |

---

## Sync vs Async — Decision Table

| Signal                                   | Choose Async              | Choose Sync                  |
|------------------------------------------|---------------------------|------------------------------|
| Write latency is critical                | ✅                        | ❌ (adds RTT per write)      |
| Zero data loss required                  | ❌                        | ✅                           |
| Replica can be slow/unreliable           | ✅ (primary unaffected)   | ❌ (primary blocks)          |
| Financial / audit data                   | ❌                        | ✅                           |
| Social / content data                    | ✅                        | Overkill                     |
| Multi-AZ durability (e.g., RDS Multi-AZ) | —                        | ✅ (AWS uses sync for Multi-AZ) |

---

## Quorum Math Cheat Sheet

```
N = Replication Factor (total copies)
W = Write quorum (nodes that must acknowledge write)
R = Read quorum (nodes that must agree on read)

Strong consistency → W + R > N

Common configurations with N=3:
  W=2, R=2: Strong consistency, tolerates 1 node failure ✅ (recommended)
  W=3, R=1: Extra durable writes, reads fast        (writes fail if any node down)
  W=1, R=3: Fastest writes, reads from all          (reads fail if any node down)
  W=1, R=1: High availability, eventual consistency (any single node survives)
  W=1, R=2: Faster writes, slightly more consistent (not strong)
```

---

## Replication Lag Consequences + Fixes

| Problem                    | Description                                     | Fix                                                  |
|----------------------------|-------------------------------------------------|------------------------------------------------------|
| Read-your-writes           | User sees stale data after their own write      | Route that user's reads to primary for N seconds     |
| Monotonic reads            | User sees older value than previously seen      | Route same user to same replica consistently         |
| Consistent prefix reads    | Causal ordering violated (reply before message) | Write causally related data to same shard/partition  |
| Phantom reads              | Count changes between two reads in same query   | Use transactions; read from primary                  |
| Stale secondary index      | Query on replica returns outdated indexed data  | Tolerate lag or query primary for indexed reads      |

---

## Replication in PostgreSQL — Key Config

```ini
# postgresql.conf (primary)
wal_level = replica          # minimum for streaming replication
max_wal_senders = 10         # how many replicas can connect
synchronous_standby_names = 'replica1'  # for sync replication

# postgresql.conf (replica)
hot_standby = on             # allow reads on replica

# recovery.conf / postgresql.conf (replica)
primary_conninfo = 'host=primary port=5432 user=replication password=...'
```

```sql
-- Monitor lag (on primary)
SELECT client_addr, state, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- Check if I am primary or replica
SELECT pg_is_in_recovery();  -- false = primary, true = replica
```

---

## Failover Checklist (Leader-Follower)

1. **Detect failure:** Heartbeat timeout (default 30s in PostgreSQL, ~10s in MongoDB)
2. **Choose new leader:** Replica with highest LSN / most up-to-date oplog
3. **Prevent split-brain:** Old primary must not accept writes (STONITH or lease expiry)
4. **Reconfigure clients:** Update load balancer or DNS; clients retry
5. **Reconfigure old primary:** When it recovers, demote to replica; resync from new primary
6. **Monitor for data loss:** Compare WAL positions; alert if any commits were lost

---

## Version Vectors vs Timestamps for Conflict Detection

| Mechanism         | Detects concurrent writes? | Clock sync required? | Used by                  |
|-------------------|----------------------------|-----------------------|--------------------------|
| Wall-clock timestamp (LWW) | ❌ (one always "wins")  | ✅ (NTP, imprecise) | Cassandra (LWW by default) |
| Logical clock (Lamport)    | Partial order only      | ❌                  | Theoretical             |
| Version vector             | ✅ Precisely detects concurrent writes | ❌ | Riak, DynamoDB (internal), CouchDB |
| MVCC versions              | ✅ Within single node   | ❌                  | PostgreSQL, MySQL       |

---

## Key Numbers to Memorize

| Fact                                         | Number                     |
|----------------------------------------------|----------------------------|
| PostgreSQL async replication lag (same AZ)   | < 10ms typical             |
| PostgreSQL sync replication overhead         | = network RTT (5–20ms)     |
| MongoDB replica set failover time            | ~10–30 seconds             |
| PostgreSQL failover (Patroni)                | ~10–30 seconds             |
| Redis Sentinel failover time                 | ~5–10 seconds              |
| Cassandra hinted handoff storage default     | 3 hours                    |
| PostgreSQL WAL segment size                  | 16MB                       |
| Max replication lag before ops alert         | 1–5 seconds (tune per SLA) |
| RDS Multi-AZ RPO                             | ~0 (sync replication)      |
| RDS Multi-AZ failover RTO                    | 1–2 minutes                |

---

## Interview Answer Framework — "How does your system handle X node failure?"

Structure every replication HA answer around:
1. **Detection:** How is the failure noticed? (heartbeat, timeout, health check)
2. **Election/Promotion:** Who becomes the new primary? (quorum, highest LSN, manual)
3. **Client rerouting:** How do clients discover the new leader? (DNS TTL, Service Discovery, driver protocol)
4. **Data safety:** Was any committed data lost? (async vs sync; RPO)
5. **Recovery:** How does the failed node rejoin? (resync from current primary)
6. **Prevention:** What prevents split-brain? (quorum > N/2, STONITH, lease)
