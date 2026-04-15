# Consensus Algorithms — Notes & Reference

## Raft Protocol Summary

```
Three states per node:
  FOLLOWER   → passive; receives heartbeats from leader
  CANDIDATE  → trying to become leader; requesting votes
  LEADER     → handles client requests; replicates log; sends heartbeats

State transitions:
  FOLLOWER  → CANDIDATE  (election timeout; no heartbeat received)
  CANDIDATE → LEADER     (received quorum votes)
  CANDIDATE → FOLLOWER   (discovered higher term or lost election)
  LEADER    → FOLLOWER   (discovered higher term)

Key timers:
  Election timeout:  150–300ms (randomized to avoid split votes)
  Heartbeat interval: 50ms (well below election timeout)
```

---

## Raft RPCs

| RPC | Direction | Purpose |
|---|---|---|
| `RequestVote(term, candidateId, lastLogIndex, lastLogTerm)` | Candidate → all | Request votes during election |
| `AppendEntries(term, leaderId, entries[], commitIndex)` | Leader → followers | Replicate log + heartbeat (empty entries = heartbeat) |

---

## Log Safety Property (Critical)

Raft guarantees: **if a log entry is committed, it will never be lost**.

Mechanism:
1. Entry is committed only when quorum of nodes have it in their log.
2. A candidate can only win election if its log is at least as up-to-date as any voter's log.
3. Any two quorums share at least one node.
4. That shared node voted for the winner → winner has all committed entries.

---

## Paxos Phases Reference

```
Phase 1 (Prepare):
  Proposer → Acceptors: Prepare(n)  [n = proposal number]
  Acceptors → Proposer: Promise(n, accepted_value)  [or reject if n is too low]

Phase 2 (Accept):
  Proposer → Acceptors: Accept(n, v)
    [v = highest previously accepted value OR proposer's own value]
  Acceptors → Learners: Accepted(n, v)

Consensus reached when: quorum of acceptors accepted same (n, v)
```

---

## CAP Theorem Application to Consensus Systems

Consensus systems are **CP** (Consistent + Partition Tolerant):

- During a network partition: the minority partition becomes **unavailable** (refuses reads/writes) because it can't achieve quorum.
- The majority partition remains **available** with strong consistency.

This is a deliberate trade-off: it's safer to reject requests than to accept potentially inconsistent data.

```
5-node cluster partitioned 3|2:
  Group of 3: has quorum → still available, still consistent ✓
  Group of 2: no quorum → refuses all writes and reads (availability sacrificed) ✗

→ Split brain (two leaders) is impossible → consistency preserved
```

---

## etcd — Raft in Practice

etcd is the production-grade Raft implementation used by Kubernetes.

```
etcd cluster: 3 or 5 nodes
  - Stores: Kubernetes cluster state (pods, services, config maps, secrets)
  - All writes go through leader → replicated to quorum → committed
  - Reads: from leader (strong consistency) or follower (stale = eventually consistent)

Key commands:
  etcdctl put /config/db_host "10.0.0.5"    → writes to leader → replicated
  etcdctl get /config/db_host                → reads from leader
  etcdctl watch /config/db_host              → watches for changes (uses Raft events)

Leader election use case:
  Multiple app instances: each tries to acquire a distributed lock via etcd lease
  SET /locks/master {instance_id} WITH LEASE 10s
  Only one succeeds → that instance is the active master
  Lease expires on crash → others compete for lock
```

---

## ZooKeeper vs etcd

| Aspect | ZooKeeper | etcd |
|---|---|---|
| Algorithm | ZAB (Paxos-like) | Raft |
| Language | Java | Go |
| Data model | Hierarchical znodes (filesystem-like) | Flat KV |
| Watches | Native znodes watches | Native key watches |
| Performance | ~10K writes/sec | ~10K–50K writes/sec |
| Use in ecosystem | Kafka (legacy), HBase, older Hadoop | Kubernetes, Consul, modern systems |

---

## Distributed Lock via Consensus (etcd/Consul)

```python
# Using etcd for distributed leader election
import etcd3

e = etcd3.client()

# Try to acquire lock with TTL (auto-released if process crashes)
lock = e.lock("my-service/primary", ttl=30)
if lock.acquire():
    print("I am the primary — start processing")
    # Periodically renew: lock.refresh() every 10 seconds
else:
    print("Another instance is primary — I am standby")
```

---

## Liveness vs Safety

| Property | Definition | Consensus guarantee |
|---|---|---|
| Safety | Bad things never happen | ✓ Never elects two leaders simultaneously |
| Liveness | Good things eventually happen | Best-effort (requires partial synchrony) |

Raft guarantees **safety** always. **Liveness** (eventually a new leader is elected) requires that at least `floor(N/2) + 1` nodes can communicate within bounded time. During a network partition severe enough to prevent a majority, no progress is made — but no incorrect state is created.

---

## Key Numbers

| Metric | Value |
|---|---|
| Raft heartbeat interval | ~50ms |
| Raft election timeout | 150–300ms (randomized) |
| quorum(N) | floor(N/2) + 1 |
| Failures tolerated with N nodes | floor((N-1)/2) |
| etcd recommended cluster size | 3 nodes (dev) or 5 nodes (prod) |
| etcd max recommended cluster | 7 nodes (more = slower quorum) |
| Raft write latency overhead (vs no replication) | 1 RTT extra (~1–5ms same DC) |
| Typical ZooKeeper + Kafka leader election time | 5–30 seconds |
