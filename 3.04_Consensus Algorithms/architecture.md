# Consensus Algorithms — Architecture

## Why Consensus?

In a distributed system, multiple nodes need to agree on a single value or a sequence of operations — even when some nodes crash or messages are delayed.

**Problems that require consensus:**
- Which node is the leader? (Leader election)
- What is the committed order of writes? (Replicated log)
- Is a configuration change (adding/removing nodes) valid? (Membership)
- Did a 2PC transaction commit or abort?

---

## The Core Problem: Split Brain

Without consensus, two nodes can independently think they're both the leader:

```
Node A: "I'm the primary, I'll accept writes"
Node B: "I'm the primary, I'll accept writes"
  → Writes go to A and B independently → data diverges → split brain
```

Consensus prevents this by requiring a **quorum** (majority) to agree before any decision is final.

---

## Quorum

A quorum is a majority of nodes: `floor(N/2) + 1`

```
N=3 nodes → quorum = 2
N=5 nodes → quorum = 3
N=7 nodes → quorum = 4
```

**Why majority?** Any two majorities overlap by at least one node. That overlapping node carries the latest information from both groups, preventing split decisions.

```
5-node cluster, quorum = 3
  Group 1 (leader election): nodes [1, 2, 3] — majority
  Group 2 (leader election): nodes [3, 4, 5] — majority
  Overlap: node 3 is in both → carries info that group 1 already elected a leader
  → Group 2's election attempt fails (node 3 votes no)
```

---

## Paxos

The original consensus algorithm (Leslie Lamport, 1989). Foundation for all production systems.

### Roles
- **Proposer:** Initiates a proposal (wants to make a decision)
- **Acceptor:** Votes on proposals (typically all nodes)
- **Learner:** Observes the final decided value

### Two-Phase Protocol

**Phase 1: Prepare**
```
Proposer → Acceptors: "Prepare(n)" where n is a proposal number
Acceptors respond:
  - If n > any previously seen proposal number: "Promise(n, last_accepted_value)"
  - Else: ignore
Proposer waits for promises from quorum (n/2 + 1 nodes)
```

**Phase 2: Accept**
```
Proposer → Acceptors: "Accept(n, value)"
  - value = highest previously accepted value if any acceptor reported one
  - value = proposer's own value if no acceptor has accepted anything yet
Acceptors: accept if n >= anything they've seen, notify learners
Learners: once quorum of acceptors have accepted accepted value → consensus reached
```

### Why Paxos is hard

- Multi-round communication
- Leader changes (dueling proposers) can livelock
- Doesn't specify the full protocol for logs, just single-value consensus
- Hard to understand and implement correctly

---

## Raft: Understandable Consensus

Raft (Ongaro & Ousterhout, 2014) was designed to be understandable. It's what etcd, CockroachDB, TiKV, and Consul use.

### Raft's Three Sub-problems

1. **Leader election:** Elect exactly one leader per term
2. **Log replication:** Leader replicates log entries to followers
3. **Safety:** If a log entry is committed, it will be present on all future leaders

### Terms

Time is divided into **terms** (monotonically increasing integers). Each term starts with an election. If election succeeds: one leader for the term. If split vote: new term immediately.

```
Term 1: Node A elected leader
Term 2: A crashes → election → Node B wins
Term 3: B network partition → election → Node C wins
```

### Leader Election

```
All nodes start as Followers.
Follower timeout (150–300ms random): no heartbeat from leader → become Candidate
Candidate: vote for self, send RequestVote to all nodes
Node grants vote if: hasn't voted this term AND candidate's log is at least as up-to-date
Candidate receives majority votes → becomes Leader
Leader sends heartbeats every 50ms to reset follower timeouts
```

**Random timeout:** The reason election timeslots are randomized (150–300ms). Without randomization, all followers would time out simultaneously and all become candidates → split vote → no leader elected.

### Log Replication

```
Client → Leader: "Set x=5"
Leader: append to own log (index=7, term=3, cmd="Set x=5"), mark as UNCOMMITTED
Leader → Followers: AppendEntries(index=7, term=3, cmd="Set x=5")
Followers: append to their log, respond OK to leader
Leader: once quorum (majority) respond OK → mark entry as COMMITTED
Leader: respond to client "OK, x=5 committed"
Leader → Followers: next heartbeat carries commit index → followers apply to state machine
```

### Log Replication Safety Guarantee

A log entry is **committed** only when a quorum has it. A new leader can only win election if it has all committed entries (its log must be at least as up-to-date as any voter's). Therefore: every committed entry survives leader crashes.

```
Committed entry = quorum has it
Candidate can only win = quorum votes for it
Any two quorums overlap → the overlapping node verified the candidate has all committed entries
→ No committed entry is ever lost
```

---

## Raft vs Paxos

| Aspect | Paxos | Raft |
|---|---|---|
| Design goal | Correct | Correct + understandable |
| Leader election | Implied | Explicit, first-class |
| Log management | Single-decree (per-value) | Full replicated log |
| Implementation | Very hard | Hard but documented clearly |
| Real implementations | ZooKeeper (ZAB, Paxos variant) | etcd, CockroachDB, TiKV, Consul |

---

## ZAB: ZooKeeper Atomic Broadcast

ZooKeeper uses ZAB (ZooKeeper Atomic Broadcast) — a Paxos variant tailored for ordered state machine replication. Key difference: ZAB treats recovery (catching up after crash) as a first-class phase, not an afterthought.

---

## Where Consensus Appears in Practice

| System | Uses | Algorithm |
|---|---|---|
| **etcd** | Kubernetes control plane, config, leader election | Raft |
| **CockroachDB** | Distributed SQL, per-range replication | Raft |
| **TiKV** | Distributed KV store (TiDB) | Raft |
| **Consul** | Service discovery, distributed lock | Raft |
| **ZooKeeper** | Distributed coordination, HBase leader election | ZAB (Paxos-like) |
| **Kafka** | Metadata replication (pre-KRaft), controller election | ZooKeeper → KRaft (Raft) |
| **Google Spanner** | Global distributed SQL | Paxos |

---

## 2-Phase Commit vs Consensus

Confusion: 2PC (used in distributed transactions) is NOT a consensus algorithm.

| Property | 2PC | Consensus (Raft) |
|---|---|---|
| Coordinator crash | Transaction stuck (blocked) | New leader takes over |
| Participant crash | Transaction stuck | Log entry committed if quorum |
| Fault tolerance | None (coordinator SPOF) | Tolerates f failures with 2f+1 nodes |
| Use case | Atomic commit across shards | Replicated state machine |

2PC requires all participants to agree + coordinator to coordinate. If the coordinator crashes after Phase 1 but before Phase 2, participants are stuck. Raft tolerates coordinator (leader) crashes by electing a new leader.

---

## FLP Impossibility

Fisher, Lynch, Paterson (1985): In a purely asynchronous distributed system, there is no deterministic algorithm that can guarantee consensus if even one node may fail.

**Why it doesn't stop us:** Real systems use timeouts (which assume partial synchrony). Raft uses heartbeat timeouts. If a node doesn't hear from the leader for 150–300ms, it assumes the leader is dead. This is a practical workaround to FLP.

---

## Key Numbers

| Metric | Value |
|---|---|
| Raft heartbeat interval | 50ms typical |
| Raft election timeout (random) | 150–300ms |
| Quorum formula | floor(N/2) + 1 |
| Minimum cluster size for 1 fault tolerance | 3 nodes |
| Minimum cluster size for 2 fault tolerance | 5 nodes |
| etcd recommended cluster size | 3 or 5 |
| Raft write latency overhead | 1–2 extra round-trips (propose + commit) |
