# CAP Theorem — Notes & Cheat Sheets

## The One-Sentence Definitions

| Property              | One sentence                                                                    |
|-----------------------|---------------------------------------------------------------------------------|
| Consistency (CAP)     | Every read returns the most recent write — or an error, never stale data        |
| Availability          | Every request to a non-failing node returns a non-error response                |
| Partition Tolerance   | The system continues operating despite network message loss between nodes        |

> **Key distinction:** CAP's Consistency ≠ ACID's Consistency. CAP C = linearizability. ACID C = constraint invariants.

---

## The Real Trade-off

```
P is not optional in any distributed system → partitions WILL happen

Real choice: during a partition, do you sacrifice C or A?

Sacrifice C (AP):  Always respond, may serve stale data
Sacrifice A (CP):  Refuse to respond if can't guarantee freshness
```

---

## CP vs AP Quick Reference

| Dimension                 | CP                              | AP                                    |
|---------------------------|---------------------------------|---------------------------------------|
| Partition behavior        | Minority nodes go unavailable   | All nodes stay available              |
| Data freshness            | Always consistent               | May be stale                          |
| Write during partition    | May be rejected                 | Always accepted (may diverge)         |
| Conflict resolution       | No conflicts (one side blocked) | App-level or LWW after partition heals|
| Error mode                | 503 / timeout                   | Stale data (no error)                 |
| Best for                  | Financial, coordination, locks  | Social, feeds, DNS, caching           |

---

## System Classification Reference Card

| System                         | Classification    | Notes                                      |
|--------------------------------|-------------------|--------------------------------------------|
| ZooKeeper                      | CP                | ZAB consensus; minority nodes read-only    |
| etcd                           | CP                | Raft; majority quorum required always      |
| HBase                          | CP                | Single HMaster; region servers wait        |
| Google Spanner                 | CP                | TrueTime + 2PC global linearizability      |
| PostgreSQL (sync replication)  | CP                | Sync replica must confirm before ack       |
| MongoDB (w:majority)           | CP                | Majority ack required                      |
| Cassandra (CL=ONE)             | AP                | Any node responds immediately              |
| Cassandra (CL=QUORUM, N=3)     | CP (effectively)  | W+R=4 > N=3, but availability drops       |
| DynamoDB (eventual)            | AP                | Default reads eventually consistent       |
| DynamoDB (strong reads)        | CP (effectively)  | Extra RCU cost for strong reads           |
| Riak                           | AP                | Leaderless; always available              |
| CouchDB                        | AP                | Multi-master; conflict via MVCC           |
| Redis (primary, async replica) | CP write + AP read| Primary writes CP; replica reads AP       |
| DNS                            | AP                | Cached responses served even if stale     |

---

## PACELC — The Extended Model

```
IF Partition:   choose between A (availability) vs C (consistency)  ← same as CAP
ELSE (normal):  choose between L (latency) vs C (consistency)

The ELSE case is the everyday reality for most systems (partitions are rare):
  EL: Optimize for low latency → accept potential staleness on reads
  EC: Optimize for consistency  → accept higher read/write latency
```

| System              | PACELC classification | Notes                                      |
|---------------------|------------------------|--------------------------------------------|
| DynamoDB            | PA/EL                  | AP in partition, low-latency in normal ops |
| Cassandra           | PA/EL                  | AP, optimized for low latency too          |
| Spanner             | PC/EC                  | CP + linearizability even in normal ops    |
| CRDT-based systems  | PA/EL                  | Always available, low latency, merge later |
| ZooKeeper           | PC/EC                  | CP + strong consistency normal ops         |

---

## CAP Misconceptions Quick-Correct

| Misconception                          | Correction                                              |
|----------------------------------------|---------------------------------------------------------|
| "Pick any 2 of 3"                      | P is required; real choice is C vs A during partition   |
| "CA systems exist (distributed)"       | CA requires no partitions — impossible to guarantee     |
| "CAP C = ACID C"                       | CAP C = linearizability. ACID C = constraint invariants |
| "Cassandra is always AP"               | Tunable; QUORUM makes it CP-like                       |
| "MongoDB is CP"                        | Only with w:majority; w:1 is AP                        |
| "CAP means C and A are incompatible"   | During normal operation, you CAN have both             |

---

## Decision Framework — Choosing CP vs AP

```
Ask these questions:
  1. What happens if a user receives stale data?
     → Minor UX issue (feed, catalog) → AP is fine
     → Financial error / double charge possible → CP required

  2. What happens if the system returns an error?
     → User retried, no harm → CP acceptable
     → Downtime cost > data inconsistency cost → AP preferred

  3. How often do partitions happen in your infrastructure?
     → Same-AZ, single DC: rare (minutes per year) → CP is low-cost
     → Cross-region, multi-DC: more frequent → AP may be necessary

  4. Can you reconcile conflicts programmatically?
     → Yes (CRDTs, last-write-wins, app-level merge) → AP viable
     → No (financial ledger, distributed lock state) → CP required
```

---

## Key Numbers and Facts

| Fact                                               | Value                            |
|----------------------------------------------------|----------------------------------|
| Quorum formula (strong consistency)                | W + R > N                        |
| Typical partition duration in same DC              | Seconds (switch reboot, cable)   |
| Typical partition healing time                     | < 1 minute (automatic detection) |
| Cassandra strong consistency CL (N=3)              | QUORUM (W=2, R=2)                |
| etcd cluster size for majority quorum (3 nodes)    | 2 nodes must agree               |
| DynamoDB strong read cost vs eventual              | 2× read capacity units           |
| ZooKeeper session timeout (default)                | 5–10 seconds                     |
| Network partition frequency (same AZ)              | ~4 minutes/year (AWS SLA)        |

---

## Interview Checklist — CAP Questions

When any CAP question comes up, structure your answer this way:
1. **Define the terms precisely** — especially that CAP C ≠ ACID C
2. **State that P is mandatory** — you always have to tolerate partitions
3. **Reframe as C vs A during partitions** — this is the real choice
4. **Classify the system** — with justification (not just a label)
5. **Mention tunability** — Cassandra, DynamoDB are not binary
6. **State PACELC if applicable** — shows advanced understanding
7. **Apply to the design question** — what does your system's CAP choice imply for the user experience?
