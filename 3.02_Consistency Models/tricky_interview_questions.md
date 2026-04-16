# Consistency Models — Tricky Interview Questions

## Q1: "Is Causal Consistency the same as Read-Your-Writes? They both seem to track causality."

**Why it's tricky:** Most candidates conflate the two because both deal with "seeing your own changes". The distinction is cross-client vs single-client scope.

**Strong Answer:**

Read-Your-Writes is a single-client, single-session guarantee: if I write X=1, my subsequent read will never return a value older than X=1. No promise is made to any other client.

Causal consistency is a multi-client guarantee. If client A writes "post", client B reads that post and writes "reply" — then any third client C that sees the reply is guaranteed to also see the post. C doesn't need to have been involved in either write.

RYW is actually a special case of causal consistency restricted to a single client. Causal is strictly stronger. When you say "causal consistency" in a system design interview, you imply RYW, monotonic reads, and writes-follow-reads — all three.

**Systems that provide it:** MongoDB causal sessions, CockroachDB, COPS (research system). The cost is carrying causal tokens on every request.

---

## Q2: "Does 'eventual consistency' mean the system will eventually return the correct value?"

**Why it's tricky:** "Eventually correct" sounds reassuring, but the model makes almost no promises about timing, ordering, or what happens before convergence.

**Strong Answer:**

Eventual consistency makes exactly one promise: if all writes stop, all replicas will converge to the same value. Everything else is undefined:

- It does NOT say when convergence happens — it could be milliseconds, seconds, or minutes.
- It does NOT prevent a replica from returning a value that is 10 writes behind.
- It does NOT prevent causal violations (seeing replies before posts).
- It does NOT prevent time-travel (reading older values than you've seen before).

In practice, "eventually consistent" systems with replication lag < 100ms are fine for social feeds, like counts, and CDN caches. They are unacceptable for financial balances, inventory during checkout, or auth tokens.

The key interview move: quantify the staleness window for the specific system and ask whether the business can tolerate it.

---

## Q3: "A candidate says 'Cassandra is eventually consistent'. Is that accurate?"

**Why it's tricky:** Cassandra's consistency is tunable — calling it "eventually consistent" is only true at the default setting and is a dangerous over-simplification.

**Strong Answer:**

It is accurate only for the default write/read consistency level of ONE. But Cassandra offers a full spectrum:

- **ONE**: Eventual consistency — fastest, stalest possible reads.
- **QUORUM**: If W + R > N (e.g., N=3, W=2, R=2), the read set always overlaps the write quorum, guaranteeing the latest value — this approximates linearizability per partition.
- **ALL**: Linearizable for that node, but unavailable if any replica is down.
- **LOCAL_QUORUM**: Strong within a data center, eventual across DCs.

When designing a Cassandra-backed system, you choose the consistency level per operation. For a user-profile update: QUORUM. For an activity feed: ONE. Saying "Cassandra is eventually consistent" in a design interview is a yellow flag unless you immediately follow up with the CL you'd actually use.

---

## Q4: "Can monotonic reads lead to reading stale data?"

**Why it's tricky:** Candidates often think monotonic reads means "fresh reads". It does not — it only prevents going backward.

**Strong Answer:**

Yes. Monotonic reads only guarantee you never see an older version than you've already seen. It says nothing about how old that version was in the first place.

Example: Client A reads key X and gets version 3 (actual version is 10). All future reads in this session will return version ≥ 3 — but you're still 7 versions behind. Monotonic reads will never return version 2 again, but it's perfectly happy returning version 3 indefinitely if no fresher replica is selected.

Monotonic reads is a useful, cheap guarantee that prevents the disorienting "time-travel" UX (seeing data you've already processed disappear). It does not replace read-your-writes for ensuring you see your own changes, and it does not replace linearizability for correctness-sensitive reads.

---

## Q5: "We use Redis as our primary cache. Is it linearizable?"

**Why it's tricky:** Redis single-node is linearizable. Redis Cluster is not. Redis with async replication is definitely not. The answer depends on the deployment.

**Strong Answer:**

It depends on the topology:

| Redis Setup | Consistency |
|------------|-------------|
| Single node, no persistence | Linearizable (single-threaded, no replication lag) |
| Primary + async replica | Eventual — replica can serve stale reads |
| Redis Cluster | Eventual per slot — async replication between primary and replica slots |
| Redis Sentinel (failover) | Eventual during failover window — acknowledged writes may be lost if primary fails before replication |

The dangerous scenario: Redis Sentinel (or Cluster) promotes a replica to primary during a failover. Any writes acknowledged by the old primary but not yet replicated to the new primary are lost. The `WAIT` command can reduce this window but not eliminate it.

For linearizable requirements (distributed locks, leader election), use Redlock with careful understanding of its assumptions, or prefer etcd/ZooKeeper which are designed for strong consistency.

---

## Q6: "A user updates their avatar. 5 seconds later, their friend's feed still shows the old avatar. What consistency model is violated, and how would you fix it?"

**Why it's tricky:** This looks like a replication lag problem but can involve multiple models depending on which user is complaining.

**Strong Answer:**

Two different violations depending on perspective:

1. **The user themselves sees the old avatar** → Read-Your-Writes violation. Their own write is not reflected in their own subsequent read. Fix: route avatar reads for a user to the primary (or a fresh replica) within a session window after a write. A version token in the session cookie is a clean mechanism.

2. **A friend sees the old avatar** → Acceptable eventual consistency. Cross-user propagation under eventual consistency has no time guarantee. Fix: if this is unacceptable, the write must propagate to X replicas before the user gets a success response (increase write quorum/concern). This trades write latency for read consistency.

For most social products, 5s for a friend to see a new avatar is perfectly acceptable eventual consistency. The first case (user can't see their own update) is a UX bug that must be fixed.

---

## Q7: "Interviewer: Design a messaging system. What consistency model do you need for messages?"

**Why it's tricky:** "Messages" is ambiguous. Chat messages, email, notifications, and activity feeds all have different consistency requirements.

**Strong Answer:**

I'd decompose the requirements:

1. **Delivery ordering within a conversation** → Causal consistency minimum. Message replies must never arrive before the messages they reply to. This drives toward vector clocks or monotonic sequence numbers per conversation.

2. **Read-your-own-sends** → Read-Your-Writes. After I send a message, I must see it in my own conversation view immediately. Achieved via writing to primary and reading from primary (or version token routing) for the sender.

3. **Message delivery guarantee** → Not a consistency model question — this is durability. Require synchronous WAL flush or majority write concern before acknowledging delivery to the sender.

4. **Cross-conversation feeds** → Eventual consistency is fine. If my chat list takes 200ms to update with a new conversation badge, that's acceptable.

The design: Cassandra with LOCAL_QUORUM writes per partition (conversation_id as partition key), monotonic sequence numbers per conversation, last-write-wins on version conflicts. This gives causal ordering without needing full vector clocks.
