# Replication — Interview Simulator

## How to use this file
Set a 35-minute timer. Answer each scenario out loud or in writing before reading the model answer. Score yourself using the rubric at the end.

---

## Scenario 1: Design the Replication Strategy for a Global E-Commerce Platform

**Prompt:**
"You're designing the database layer for an e-commerce platform serving customers in the US, EU, and Asia. You have orders, inventory, and user accounts. Write throughput is 50K TPS global at peak. Reads are 10x that. Design the replication strategy that meets: 99.99% availability, RPO < 5 seconds, latency < 50ms for reads."

**Time box:** 15 minutes

---

### Model Answer

**Step 1: Classify data by write ownership**

```
Not all data has the same replication requirement:

User accounts:  User modifies only their own account
                → Geo-partition: US users' accounts owned by US region
                → Low conflict risk, easy to geo-replicate

Orders:         Created in user's region, read globally (merchants, ops)
                → Write to local region, async replicate globally

Inventory:      SHARED MUTABLE across all regions
                → The hard problem (concurrent stock decrements)
                → Needs special handling
```

**Step 2: Architecture per region**

```
Each of 3 regions (US / EU / APAC):

  ┌─────────────────────────────────────────┐
  │ PostgreSQL cluster (per region)          │
  │                                          │
  │  [ Primary ] ← writes                   │
  │        │  sync replication               │
  │  [ Replica AZ-b ] ← same-region failover│
  │        │  async replication              │
  │  [ Replica AZ-c ] ← read scale          │
  └─────────────────────────────────────────┘
        │
        │ Cross-region async replication
        ▼
  EU and APAC regions (read-only replicas for non-owned data)
```

**RPO calculation:**
```
Same-region (sync): RPO = 0 ms ✅
Cross-region (async with 50ms RTT): RPO ≈ 50–200ms ✅ (< 5s requirement)
```

**Step 3: Inventory — the hard problem**

```
Naive approach: replicate inventory globally, decrement on purchase
Problem: US and EU concurrently sell the last unit → oversell

Option A: Single global inventory primary (US-East)
  → All inventory reads/writes to US → high latency for EU/APAC
  → Not compatible with < 50ms read latency globally

Option B: Inventory reservation in user's region + central reconciliation
  → User's region holds a "reservation" of N units
  → Purchase decrements local reservation
  → Central inventory service periodically reconciles
  → Works if per-region reservations are sized > local expected demand

Option C: Centralized inventory with Redis cache
  → Inventory READ from Redis (< 5ms, replicated to each region)
  → Inventory WRITE (purchase) always goes to central primary
  → Accept that purchase write latency for EU/APAC = cross-region RTT (~100ms)
  → But read latency (product page) is always local Redis = fast ✅
```

**Step 4: RTO for 99.99% availability**

```
99.99% = 52.5 minutes downtime per year = ~4.4 minutes per month

Failover budget: RTO must be << 4.4 min (otherwise a single failover event
                 burns several months of budget)

Target RTO: < 30 seconds
Implementation: Patroni + etcd per region, auto-promotion

Cross-region failover (region-level disaster):
  1. Route traffic away from failed region (Route 53 health checks, < 60s)
  2. Promote cross-region replica to primary (async lag = up to 200ms data loss)
  3. Update DNS → global traffic resumes
  Total: ~1–2 minutes ← acceptable for region-level disaster
```

**Final summary to state:**
```
1. Geo-partition user-owned data: 3 per-region primaries
2. Synchronous intra-region replication: RPO=0 within region
3. Asynchronous cross-region: RPO < 5s ✅
4. Read replicas in each AZ: read latency < 10ms local ✅
5. Inventory: central primary + Redis cache per region
6. Patroni auto-failover: RTO < 30s ✅
7. Route 53 cross-region failover: RTO < 2min for region disaster
```

---

## Scenario 2: Debug a Replication Lag Incident

**Prompt:**
"3am alert: replication lag on your PostgreSQL replica has climbed from < 10ms to 90 seconds over the past 30 minutes. Your site's analytics dashboard (which reads from the replica) is showing data from 90 seconds ago. The primary is unaffected. Walk me through your investigation."

**Time box:** 10 minutes

---

### Model Answer

**Immediate: assess blast radius**

```
Is the lagging replica serving any critical user-facing reads?
  → If yes: immediately reroute those reads to primary
  → Analytics dashboard reads from replica = acceptable to leave stale temporarily
  → Priority: prevent lag from becoming permanent, diagnose root cause
```

**Step 1: Check what's happening on the replica**

```sql
-- Is WAL apply stuck? (replica side)
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY wait_event_type;

-- Is the WAL receiver still connected?
SELECT status, received_lsn, latency_ms FROM pg_stat_wal_receiver;
```

**Step 2: Check replication from primary**

```sql
-- How much WAL is being generated? (primary)
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS unsent_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_behind_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication
WHERE client_addr = '<replica_ip>';
```

**Step 3: Diagnose common root causes**

```
A) Long-running query on replica blocking WAL apply
   Symptom: hot_standby_feedback = on, a long SELECT is blocking apply
   Check: pg_stat_activity shows a long-running SELECT on replica
   Fix:  pg_cancel_backend(<pid>), adjust max_standby_streaming_delay

B) Write burst on primary (batch import, large update)
   Symptom: primary WAL generation spiked in last 30 min
   Check: SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') for growth rate
   Fix:  Wait — replica will catch up once burst subsides; throttle batch import

C) Replica CPU/disk bottleneck
   Symptom: replica CPU at 100%, or disk I/O maxed
   Check: OS metrics on replica node
   Fix:  Upgrade replica instance, move to faster SSD

D) Network saturation between primary and replica
   Symptom: write_lag >> 0 (primary can't even send WAL fast enough)
   Check: network bandwidth utilization between AZs
   Fix:  wal_compression = on on primary (reduces bandwidth 50-70%)
```

**30 minutes of gradual drift indicates (most likely):**
A long-running analytics query holding a lock on the replica, or a sustained write burst on the primary from a batch job that started ~30 min ago.

**Resolution:**
```
1. Kill the blocking query on replica (if found)
2. Monitor: lag should start decreasing within minutes
3. If batch job on primary: contact DBA/team to throttle it (add pg_sleep between batches)
4. Set alert: lag > 10s → page; lag > 30s → wake engineer
5. Post-incident: set max_standby_streaming_delay = '30s' to auto-cancel slow replica queries
```

---

## Scenario 3: Choose Between Cassandra and PostgreSQL Replication for a Ride-Sharing App

**Prompt:**
"You're building the location tracking service for a ride-sharing app like Uber. Drivers update their GPS coordinates every 3 seconds. There are 500K active drivers. The service needs to answer 'give me all drivers within 5km of this user' in < 100ms. Design the storage and replication strategy."

**Time box:** 10 minutes

---

### Model Answer

**Write load calculation:**
```
500K drivers × 1 write/3 seconds = ~167K writes/sec
→ This is the dominant constraint: extreme write throughput
```

**Why standard PostgreSQL single-primary replication won't work:**
```
PostgreSQL single primary: ~10K–20K TPS writes comfortably
→ 167K writes/sec requires 8–16 primary instances
→ Sharding PostgreSQL is complex and adds cross-shard query overhead
```

**The right tool: Redis for hot location data**

```
Write path:
  Driver app → API → Redis GEOADD driver_locations <lng> <lat> <driver_id>
  167K ops/sec → Redis handles ~500K ops/sec on a single node ✅

Read path (nearby drivers):
  Redis GEORADIUS driver_locations <user_lng> <user_lat> 5 km ASC WITHCOORD
  → Returns sorted list of drivers within 5km
  → O(N+log M) where M = entries in radius → typically < 5ms ✅
```

**Replication strategy for Redis:**
```
Redis Cluster (Geospatial data):
  - Shard by driver_id prefix (geo commands work within a shard)
  - Cluster: 6 nodes (3 primary + 3 replica, 1 replica per shard)
  - Replication: async (RPO = a few ms — acceptable for location data)
  - Failover: Redis Cluster auto-failover in < 10 seconds

Why async is acceptable here:
  - Location data is ephemeral (a 3-second-old location is "close enough")
  - If a node fails and we lose 3 seconds of driver positions:
    → Driver will send a new update within 3 seconds anyway
  - RPO of seconds is fine; availability is the priority
```

**Persistent storage (historical trips, billing):**
```
Redis: hot current location data → high-frequency reads/writes, TTL
PostgreSQL: completed trips, billing records → strong ACID, fewer writes

Replication for PostgreSQL:
  - Write volume: ~10K trips/day = ~0.1 TPS → trivial for PostgreSQL
  - Standard leader-follower, async replication is fine
  - Read replicas for analytics queries
```

**Full architecture:**
```
Driver app (every 3s)
    │
    ▼
  API Gateway
    │
    ├── Redis Cluster (write GEOADD) ← current locations, 60s TTL
    │       │ async replication
    │   [Replicas × 3]
    │
    └── PostgreSQL (write trip records when trip completes)
            │ async replication
        [Replicas × 2] ← analytics, reporting

User search request:
    → Redis GEORADIUS → driver locations < 5ms ✅
```

---

## Self-Assessment Scorecard

| Criterion                                                              | Score /5 |
|------------------------------------------------------------------------|----------|
| Correctly distinguished sync vs async and their trade-offs             |          |
| Applied RPO/RTO framing to the scenario requirements                   |          |
| Named quorum formula (W + R > N) correctly                             |          |
| Included read-your-writes consistency in relevant designs              |          |
| Identified replication lag root causes and systematic triage           |          |
| Chose correct replication model per use case (not one-size-fits-all)  |          |
| Mentioned failover mechanism (Patroni, Redis Sentinel, Raft election)  |          |
| Did NOT say "just add replicas" to solve write scale                   |          |

**Total: /40**

| Score | Interpretation                                                          |
|-------|-------------------------------------------------------------------------|
| 35–40 | Excellent — replication design internalized, interview-ready            |
| 25–34 | Good — review architecture.md sync/async and leaderless sections        |
| 15–24 | Re-read tricky_interview_questions.md Q1 and Q5, redo Scenario 2        |
| < 15  | Full re-read of architecture.md + quorum simulation exercise            |
