# CAP Theorem — Tricky Interview Questions

---

## Q1: "Is it possible to build a system that's both CP and AP? Why or why not?"

**What they're testing:** Whether you understand CAP as a scenario-based constraint, not an immutable label.

**Strong answer:**

Strictly speaking, no — during a network partition you must choose one. But the way the question is framed reveals an important nuance:

**A system can be CP under some conditions and AP under others.** Cassandra is the canonical example:

```
Cassandra CL=ONE:   AP — any single node responds, even during partition
Cassandra CL=QUORUM (N=3): CP-like — W+R=4 > N=3, strong consistency
                             but: if quorum is unreachable, writes fail → sacrifices A

You choose per query. You can even choose differently for:
  → User login (CL=QUORUM — must be correct)
  → User feed (CL=ONE — eventual is fine)
```

**The deeper insight to mention:** CAP is a worst-case model for partition scenarios. In the normal case (no partition), you can have both C and A. The real engineering challenge is deciding what to do in the partition scenario — which in practice occurs for seconds to minutes per year in a well-operated datacenter.

**What I would NOT say:** "You always have to choose 2 of 3" — this is an oversimplification that misses tunability and the spectrum of real-world behaviors.

---

## Q2: "Our database cluster is 5 ZooKeeper nodes. 2 nodes fail. Is the cluster still functional?"

**What they're testing:** Application of quorum math with real systems.

**Strong answer:**

Yes — fully functional. Here's why:

```
ZooKeeper cluster: 5 nodes
Quorum required: majority = floor(5/2) + 1 = 3 nodes

With 2 failures: 3 nodes remain → exactly meets quorum threshold ✅
  → Reads: any node can serve (or just leader-coordinated reads)
  → Writes: require quorum ack → 3 nodes confirm → system continues
```

**What if 3 nodes fail?**
```
2 nodes remain → below majority (need 3) → cluster stops accepting writes
→ ZooKeeper becomes read-only (some implementations) or fully unavailable
→ Kubernetes (if using etcd, same quorum math) cannot update cluster state
```

**Why odd numbers?**
```
3 nodes: tolerates 1 failure  (quorum = 2)
5 nodes: tolerates 2 failures (quorum = 3)
7 nodes: tolerates 3 failures (quorum = 4)

Even numbers add no benefit and create split-brain risk:
4 nodes: tolerates 1 failure (quorum = 3) — same as 3 nodes
         If partitioned 2|2: neither side has quorum → full outage
```

In production: always use odd-numbered cluster sizes.

---

## Q3: "If your service uses Cassandra with CL=ONE, a user updates their email and immediately refreshes. They see the old email. Is this a bug?"

**What they're testing:** AP system behavior awareness and how to handle it in product design.

**Strong answer:**

Not a bug — it is the expected behavior of an AP system with eventual consistency. Here's exactly what happened:

```
Write:  User updates email → Write goes to Node A (CL=ONE: 1 node acks)
Read:   User refreshes    → Read hits Node B (CL=ONE: first responding node)
        Node B hasn't received the write yet (replication lag ~5ms)
        → Returns old email

After ~5ms: All nodes converge → subsequent reads see new email
```

**Is it a problem?** Depends on the use case:
- User sees old email for 5ms → minor UX annoyance, not a real error ← **acceptable**
- User "can't log in" because auth system still has old email → **not acceptable**

**How to fix it at the application layer:**

```python
# Option 1: Optimistic UI update
# Don't re-fetch from server after save — update local UI state immediately
# Backend will converge within milliseconds

# Option 2: Read-your-writes routing
# After write: set a cookie/flag
# Next read within 2 seconds: route to primary or use CL=LOCAL_QUORUM

# Option 3: For critical paths (auth, billing): use CL=QUORUM
# Accept the slightly higher latency for these specific queries
```

**Key interview point:** In AP systems, the application is responsible for handling eventual consistency. The database won't make this decision for you. Know which reads in your system require freshness and route them accordingly.

---

## Q4: "Explain CAP in the context of a microservices architecture — not just databases."

**What they're testing:** Ability to generalize CAP beyond storage systems.

**Strong answer:**

Every inter-service communication faces a CAP-equivalent trade-off when a service is unavailable:

**Scenario:** Order Service needs inventory stock count from Inventory Service to process an order.

```
Network partition: Order Service → ✂ → Inventory Service

CP response (synchronous call):
  Order Service calls Inventory Service → 503 → rejects the order request
  → User gets error: "Unable to process order, please try again"
  → No orders placed on incorrect stock data
  → Sacrifices availability: users can't order during partition

AP response (async / cached):
  Order Service uses last known stock count from Redis cache
  → Accepts the order, deducts from cached stock
  → Inventory Service processes the event when reconnected
  → If stock was actually 0: compensating transaction (cancel, refund, notify)
  → Always available, but occasional oversell is possible
```

**Which is correct?** Depends on the business:
- Exclusive high-demand items (1 left in stock): CP — can't oversell
- Products with deep inventory (1000 units): AP — occasional oversell is recoverable
- Payments: always CP — never proceed without confirmed available balance

**The microservices pattern mapping:**
```
CP microservices  → synchronous calls (gRPC, HTTP) with circuit breaker
AP microservices  → async messaging (Kafka, SQS) + eventual reconciliation + compensating transactions
```

**SAGA pattern** is the AP microservices solution: each service does its local transaction and publishes an event. If a downstream step fails, compensating transactions roll back prior steps. Always available, eventually consistent.

---

## Q5: "HBase is classified as CP. But what actually happens to the region with the failed RegionServer during a partition — and how is it recovered?"

**What they're testing:** Real depth on CP internals, not just label memorization.

**Strong answer:**

HBase uses a single HMaster + multiple RegionServers. Each Region (a contiguous range of row keys) is owned by exactly one RegionServer at a time.

**During a partition / RegionServer failure:**

```
HMaster
  │ detects RS failure via ZooKeeper session expiry (ZK heartbeat timeout ~30s)
  │
  ├── Region assignment: the failed RS's regions are UNASSIGNED
  │   → Those regions are temporarily unavailable ← CP: sacrifices A for safety
  │
  └── HMaster assigns orphaned regions to healthy RegionServers
      Healthy RS recovers the region's WAL (HLog) from HDFS
      Replays WAL → applies all uncommitted writes → region comes online
```

**Why this is CP:**
- During the (typically 30s–2min) recovery window, rows in the failed RS's regions return errors
- HBase does NOT route to another RS (no replication to another RS for the same region)
- No stale data served; instead, errors until recovery completes

**How to reduce the unavailability window:**
1. Increase ZooKeeper session timeout detection reliability (reduce false positives)
2. Enable HBase region replication (experimental): allows read-only replicas for hot regions
3. Increase RegionServer heap to reduce GC pauses (common cause of false death detection)
4. Use smaller regions: faster WAL replay = faster recovery

**Interview insight:** This is why HBase is not ideal for SLAs requiring < 1 minute RTO for any single node failure. It's excellent for write-heavy batch workloads (analytics) where a 1-2 minute recovery is acceptable.

---

## Q6: "A candidate says 'We'll use an AP system so we never have downtime.' What's wrong with this reasoning?"

**What they're testing:** Critical thinking about the trade-offs both models involve.

**Strong answer:**

This reasoning confuses "no errors" with "no downtime" — and ignores the operational costs of AP.

**Problems with the reasoning:**

1. **AP systems can still have "application downtime":**
   If a user receives stale data and acts on it (e.g., sees a product as "in stock" when it's not), the resulting error — out-of-stock notification, failed checkout, forced cancellation — is effectively downtime from the user's perspective. It's just a different kind of failure.

2. **Eventual consistency requires compensating mechanisms:**
   AP systems push the correctness problem from the DB to the application. You now need:
   - Conflict detection (version vectors, CAS)
   - Conflict resolution logic (application-specific)
   - Compensating transactions for invalid states
   This is often MORE engineering work than tolerating occasional write failures in a CP system.

3. **"No downtime" overstates AP guarantees:**
   AP means: non-failing nodes respond. If the node your client is connected to fails, you get an error. AP just means more nodes remain available — not that your specific connection never errors.

4. **The real question is: which failure is worse?**
   - CP failure: user gets a 503, retries, likely succeeds → annoying but recoverable
   - AP failure: user receives a wrong answer, acts on it (double-spend, oversell) → harder to fix

**Corrected reasoning:** "We'll use AP for data where eventual convergence is acceptable (feeds, catalog) and CP for data where correctness is non-negotiable (balances, inventory allocations, locks)." The answer is usually per-entity, not a system-wide choice.

---

## Q7: "How does Google Spanner achieve global linearizability while still being a distributed system?"

**What they're testing:** Advanced knowledge of how modern distributed databases overcome CAP constraints through hardware.

**Strong answer:**

Spanner achieves global external consistency (linearizability across all nodes, globally) through **TrueTime** — a hardware-assisted globally synchronized clock.

**The problem with distributed transactions normally:**
```
To order events globally, you need to know the wall-clock time of each transaction.
But clocks on different servers drift → can't definitively say which happened first.
→ Either use consensus (slow: cross-planet round trip) or accept uncertainty.
```

**TrueTime — Spanner's solution:**
```
Google's datacenters have atomic clocks + GPS receivers
TrueTime API returns: [earliest_possible, latest_possible] for "now"
  → The true current time is within this interval
  → Interval is typically 1–7ms wide

Commit wait:
  When Spanner commits a transaction, it assigns a timestamp = TrueTime.now.latest
  Then WAITS until TrueTime.now.earliest > commit_timestamp
  → Guarantees that when the commit is visible, its timestamp is definitively in the past
  → Any transaction started after will see a later timestamp
  → Global ordering without cross-planet communication
```

**Trade-off Spanner makes:**
- Write latency is higher by the commit wait (1–7ms extra per write) + cross-region consensus latency
- But reads are always consistent, globally, without extra overhead

**Why this matters for CAP:**
Spanner is still subject to CAP — during a partition, it cannot guarantee both C and A. It chooses C (linearizability, always). The TrueTime insight is that, during normal operation, it achieves "EC" (consistency over latency) in PACELC terms with a bounded, small latency penalty — rather than accepting eventual consistency.

**Key quote to remember:** "Spanner essentially eliminates the latency vs consistency trade-off for normal operation, at the cost of requiring purpose-built hardware."
