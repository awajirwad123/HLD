# Replication — Tricky Interview Questions

---

## Q1: "You use async replication. Primary crashes. How much data do you lose?"

**What they're testing:** Understanding of RPO (Recovery Point Objective) in async setups.

**Strong answer:**

It depends on the **replication lag at the moment of the crash** — not some fixed number.

```
Best case:  Lag = 0ms → zero data loss (replica had everything)
Typical:    Lag = 5–50ms in same AZ → small number of recent writes lost
Worst case: Lag spike during write burst → seconds or minutes of data lost
```

**Exactly what is lost:** Any write that was committed on the primary (client received `OK`) but whose WAL record had not yet been applied on the replica at the time of crash. If the primary's disk is unrecoverable, those writes are permanently gone.

**How to quantify:**
```sql
-- On primary (before crash): record current WAL LSN
SELECT pg_current_wal_lsn();   -- e.g., 0/5A4B2C10

-- On replica after promotion: check what was the last applied LSN
SELECT pg_last_wal_apply_lsn();   -- e.g., 0/5A4B2900

-- Difference in bytes = 0x5A4B2C10 - 0x5A4B2900 = 0x310 = ~784 bytes of lost WAL
```

**Mitigation options:**
1. Set up monitoring: alert if `replay_lag > 1s` (gives time to make decision)
2. Use synchronous replication for critical tables only (mix sync + async)
3. Accept the risk: for social content, a few lost rows are tolerable; for payments, not

**Interview insight:** Always state that RPO = maximum acceptable data loss. If RPO = 0, you must use sync replication. If RPO = "a few seconds is fine," async is acceptable.

---

## Q2: "How would you design a global active-active database with zero write conflicts?"

**What they're testing:** Multi-leader conflict awareness and the strategies to avoid it.

**Strong answer:**

Active-active across regions means both regions accept writes — the hardest replication problem. The key insight is: **avoiding conflicts is better than resolving them.**

**Strategy 1: Shard by geography (geo-partition)**
```
US users always write to US region (primary for US data)
EU users always write to EU region (primary for EU data)

Each region is primary for its own users' data, replica for others.
→ No conflicts: users are never "concurrent writers" to their own data on two regions
→ Works for: Uber (rides belong to a city), Stripe (merchants belong to a country)
→ Fails for: truly global shared data (e.g., a global counter)
```

**Strategy 2: CRDTs for shared data**
```
For data that must be globally writable:
  Use CRDT (Conflict-free Replicated Data Types) structures:
    G-Counter: monotonically growing counter (likes, views, downloads)
    LWW-Register: last-write-wins for single values (with logical clocks)
    OR-Set: add/remove elements with tombstones

CRDTs guarantee convergence — all replicas reach the same state after sync
regardless of write order → no conflicts by design
```

**Strategy 3: CockroachDB / Google Spanner (Distributed SQL)**
```
Single logical database, multi-region writes.
Achieves global linearizability via consensus (Paxos/Raft) on each write.
Writes are slower (cross-region consensus = 50–200ms), but no conflicts.
Used when: strong correctness > write latency.
```

**What I would NOT do:**
- LWW timestamps for conflict resolution — clock skew causes silent data loss
- Application-level merge for every entity type — operationally unsustainable

---

## Q3: "Your replica is 60 seconds behind the primary. What do you do?"

**What they're testing:** Operational reasoning, not just theory.

**Immediate triage (first 5 minutes):**

```
1. Is it a sudden spike or gradual drift?
   → Sudden: primary had a write burst (batch import, migration) — replica catching up
   → Gradual: replica is resource-constrained — CPU, disk, network

2. Check replica resource metrics:
   → CPU: applying WAL is CPU-bound for complex transactions
   → Disk I/O: replica writing to a slower disk than primary?
   → Network: bandwidth saturated between primary and replica?

3. Check if a long-running query on the replica is blocking WAL apply:
   →  SELECT query, wait_event_type, wait_event
      FROM pg_stat_activity
      WHERE state = 'active';
   → A SELECT holding a lock can block WAL apply in some configs

4. Check pg_stat_replication on primary:
   → write_lag > flush_lag > replay_lag?
   → If write_lag is large: network bottleneck to replica
   → If replay_lag >> flush_lag: replica processing bottleneck
```

**Immediate mitigations:**
1. If replica has long-running queries: cancel them (`pg_cancel_backend`)
2. Set `max_standby_streaming_delay` to limit how long WAL apply waits for a query
3. If it's a burst: wait for replica to catch up; don't promote it while lagging
4. Route reads away from the lagging replica to protect users from stale reads

**Long-term fixes:**
- Scale replica vertically (CPU, faster disk)
- Enable `hot_standby_feedback = off` to prevent WAL apply delays from long queries
- Add a second replica and route reads to the faster one

---

## Q4: "Explain how Cassandra achieves eventual consistency without a leader."

**What they're testing:** Deep understanding of leaderless replication mechanics.

**Full answer:**

Cassandra uses three mechanisms to converge all replicas to the same state:

**1. Gossip Protocol (Cluster Membership)**
```
Every second, each node:
  - Picks 1–3 random peers
  - Exchanges information: which nodes are alive, which are down, which data they hold

Result: Within ~O(log N) rounds, all nodes know the cluster state
        N=100 nodes → ~7 gossip rounds → all nodes aware of a failure
```

**2. Read Repair (On-Demand Reconciliation)**
```
On a QUORUM read:
  - Coordinator queries R nodes
  - Compares version timestamps across responses
  - If a node returns stale data → coordinator sends the latest value back to that node
  
→ Reads act as a repair mechanism; frequently-read data converges quickly
→ Infrequently-read data may stay stale longer
```

**3. Anti-Entropy (Background Reconciliation via Merkle Trees)**
```
Cassandra runs "nodetool repair" (manually or automatically):
  - Each node builds a Merkle tree of its data (hash tree of hash tree of rows)
  - Exchange tree roots with peer → if roots match, data matches
  - If differ: drill into the tree to find the divergent range
  - Sync only the differing rows

→ Finds and fixes divergence that read repair misses
→ Essential after node recovery, after adding nodes
```

**Why there's no leader needed:**
- Writes go to any coordinator node; coordinator uses consistent hashing to find the correct replica nodes
- Conflicts are resolved by timestamp (LWW) or application logic
- Convergence is guaranteed eventually because all three mechanisms continuously push replicas toward the same state

---

## Q5: "A primary and one replica are in a sync replication setup. The replica goes offline. What happens to your application?"

**What they're testing:** Understanding of the availability trade-off of sync replication.

**Strong answer:**

With synchronous replication, the primary **must wait for replica confirmation before acknowledging writes**. If the sync replica is offline, this wait never resolves.

**Default behavior in PostgreSQL with `synchronous_standby_names`:**
```
All writes will BLOCK indefinitely, waiting for a replica that never responds.
Your application's write requests will timeout or run until their statement_timeout.
→ Writes are completely unavailable until the replica comes back or sync is disabled.
```

**This is an intentional trade-off:** You configured sync replication for zero data loss. The system honors that: it will not acknowledge a write it cannot guarantee won't be lost.

**Operational escape hatch:**
```sql
-- Temporarily downgrade to async while replica is down:
ALTER SYSTEM SET synchronous_standby_names = '';
SELECT pg_reload_conf();
-- Now writes proceed asynchronously → application recovers
-- ⚠️ Data loss risk is now active until replica returns
```

**Better design: Semi-sync (at least one):**
```
synchronous_standby_names = 'ANY 1 (replica1, replica2)'
-- Requires acknowledgment from ANY 1 of the listed replicas.
-- If replica1 is down, replica2's ack suffices.
-- Only fails if ALL listed replicas are down simultaneously.
```

**Interview key point:** This is the core availability cost of `C > A` (CAP theorem). Sync replication = CP. When the sync replica fails, the primary sacrifices availability to preserve consistency guarantees.

---

## Q6: "Is it safe to read from a replica for a payment confirmation page?"

**What they're testing:** Practical understanding of which reads must go to primary.

**Strong answer:**

No — it depends on the freshness requirement, but for payment confirmation: **read from the primary**.

**Why replicas are unsafe here:**

1. The payment write just happened (< 1 second ago)
2. The replica may have 5–50ms of lag
3. If the read goes to the lagging replica, the user sees "payment pending" even though it was confirmed
4. User refreshes, hits a different replica, may see different state (non-monotonic reads)
5. In the worst case: user re-submits the payment assuming it failed → double charge

**What I would implement:**
```python
# After writing a payment:
# Option 1: Always read from primary for payment status
async def get_payment_status(payment_id: str, db=Depends(get_primary_session)):
    return await db.execute("SELECT * FROM payments WHERE id = :id", {"id": payment_id})

# Option 2: Redirect after write (POST-Redirect-GET pattern)
# After payment confirmed: 302 redirect to /payment/{id}/confirmation
# That GET request reads from primary (tagged as post-write)
# → Prevents duplicate submission AND ensures fresh read
```

**Pattern for read routing decision:**
```
Payment status page      → Primary (user just made this action)
Order history (list)     → Replica (eventual consistency ok)
Inventory check at checkout → Primary (must be accurate before charging)
Product catalog          → Replica (a few seconds stale is fine)
```

**General rule:** Any read whose staleness could directly cause a financial or safety error → primary. Everything else → replica.

---

## Q7: "Design the replication setup for a payments database that requires RPO=0 and RTO < 1 minute."

**What they're testing:** Applying theory to a real requirement.

**Constraints translation:**
```
RPO = 0     → zero data loss → synchronous replication mandatory
RTO < 1min  → automatic failover, not manual → auto-promotion required
```

**Architecture:**

```
                  ┌──────────────────────────────────┐
                  │         Patroni cluster          │
                  │   (etcd-backed HA manager)        │
                  └──────────────────────────────────┘
                       │           │           │
               ┌───────┘     ┌─────┘     ┌────┘
               ▼             ▼           ▼
     [ Primary/Leader ]  [ Replica 1 ] [ Replica 2 ]
          AZ-1               AZ-2          AZ-3

Replication:  Primary → Replica 1: SYNCHRONOUS (RPO=0)
              Primary → Replica 2: ASYNCHRONOUS (read scale, hot standby)

Failover:
  Patroni + etcd: leader election in < 10s
  DNS update or HAProxy reload: further 10–20s
  Total RTO: ~30s ✅ (< 1 minute)
```

**PostgreSQL config:**
```ini
# primary postgresql.conf
synchronous_standby_names = 'FIRST 1 (replica1)'
# FIRST 1: synchronous acknowledgment required from the first available in the list
# replica2 remains async → no write block if replica2 is slow
```

**Why three nodes?**
- Quorum for Patroni election requires majority: 2 of 3 Patroni agents must agree
- If primary fails: replica1 (sync) is immediately promotable with zero data loss
- If replica1 also fails: fall back to replica2 (async, possible small data loss — acceptable as a degraded mode)

**Monitoring SLAs:**
- Alert: `replay_lag > 100ms` on the sync replica (investigate before it worsens)
- Alert: `pg_is_in_recovery() = true` on what should be primary (failover happened)
- Alert: Patroni leader change (PagerDuty)
