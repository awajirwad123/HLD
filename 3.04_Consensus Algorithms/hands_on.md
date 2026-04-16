# Consensus Algorithms — Hands-On Exercises

## Exercise 1: Raft Leader Election Simulation

```python
import asyncio
import random
import time
from dataclasses import dataclass, field
from enum import Enum


class NodeState(str, Enum):
    FOLLOWER  = "follower"
    CANDIDATE = "candidate"
    LEADER    = "leader"


@dataclass
class LogEntry:
    term: int
    index: int
    command: str


@dataclass
class RaftNode:
    node_id: int
    peers: list["RaftNode"] = field(default_factory=list, repr=False)

    # Persistent state (would be on disk in production)
    current_term: int = 0
    voted_for: int | None = None
    log: list[LogEntry] = field(default_factory=list)

    # Volatile state
    state: NodeState = NodeState.FOLLOWER
    commit_index: int = -1
    last_applied: int = -1

    # Leader state
    next_index: dict[int, int] = field(default_factory=dict)
    match_index: dict[int, int] = field(default_factory=dict)

    # Timers
    last_heartbeat: float = field(default_factory=time.time)
    election_timeout: float = field(default_factory=lambda: random.uniform(0.5, 1.5))

    # Simulation
    alive: bool = True
    votes_received: int = 0

    def __post_init__(self):
        self.last_heartbeat = time.time()

    def log_info(self, msg: str):
        state_emoji = {"follower": "📥", "candidate": "🗳️", "leader": "👑"}
        print(f"  [{state_emoji.get(self.state, '?')} Node {self.node_id} | "
              f"Term {self.current_term:2d} | {self.state.upper():9s}] {msg}")

    async def run(self):
        """Main Raft loop."""
        while self.alive:
            if self.state == NodeState.LEADER:
                await self._send_heartbeats()
                await asyncio.sleep(0.2)
            elif self.state == NodeState.FOLLOWER:
                elapsed = time.time() - self.last_heartbeat
                if elapsed > self.election_timeout:
                    await self._start_election()
                await asyncio.sleep(0.05)
            elif self.state == NodeState.CANDIDATE:
                # Election already in progress
                await asyncio.sleep(0.05)

    async def _start_election(self):
        self.state = NodeState.CANDIDATE
        self.current_term += 1
        self.voted_for = self.node_id
        self.votes_received = 1  # Vote for self
        self.election_timeout = random.uniform(0.5, 1.5)  # Reset timeout

        self.log_info(f"Election timeout — starting election for term {self.current_term}")

        # Request votes from peers
        votes_needed = len(self.peers) // 2 + 1  # Quorum (not counting self)

        vote_tasks = [self._request_vote(peer) for peer in self.peers if peer.alive]
        results = await asyncio.gather(*vote_tasks, return_exceptions=True)

        granted = sum(1 for r in results if r is True)
        self.votes_received = 1 + granted  # self + peers

        if self.state == NodeState.CANDIDATE and self.votes_received > len(self.peers) // 2:
            self._become_leader()
        elif self.state == NodeState.CANDIDATE:
            self.log_info(f"Election failed ({self.votes_received} votes) → back to follower")
            self.state = NodeState.FOLLOWER
            self.last_heartbeat = time.time()

    async def _request_vote(self, peer: "RaftNode") -> bool:
        """Simulate RequestVote RPC."""
        await asyncio.sleep(random.uniform(0.01, 0.05))  # Network delay

        if not peer.alive:
            return False

        # Raft vote granting rules:
        vote_granted = False
        if self.current_term > peer.current_term:
            peer.current_term = self.current_term
            peer.voted_for = None
            peer.state = NodeState.FOLLOWER

        if (self.current_term == peer.current_term and
                (peer.voted_for is None or peer.voted_for == self.node_id)):
            peer.voted_for = self.node_id
            vote_granted = True
            peer.last_heartbeat = time.time()  # Reset timeout on vote grant

        if vote_granted:
            peer.log_info(f"Voted for node {self.node_id} in term {self.current_term}")
        return vote_granted

    def _become_leader(self):
        self.state = NodeState.LEADER
        self.log_info(f"🎉 WON ELECTION with {self.votes_received}/{len(self.peers)+1} votes!")
        for peer in self.peers:
            self.next_index[peer.node_id] = len(self.log)
            self.match_index[peer.node_id] = -1

    async def _send_heartbeats(self):
        """Leader sends heartbeats to followers."""
        for peer in self.peers:
            if peer.alive and peer.state != NodeState.LEADER:
                asyncio.create_task(self._send_heartbeat_to(peer))

    async def _send_heartbeat_to(self, peer: "RaftNode"):
        await asyncio.sleep(random.uniform(0.01, 0.03))
        if not peer.alive:
            return
        if self.current_term >= peer.current_term:
            peer.current_term = self.current_term
            peer.state = NodeState.FOLLOWER
            peer.last_heartbeat = time.time()
            peer.voted_for = None  # Reset for new elections

    def crash(self):
        self.alive = False
        self.log_info("CRASHED")
        self.state = NodeState.FOLLOWER

    def recover(self):
        self.alive = True
        self.state = NodeState.FOLLOWER
        self.last_heartbeat = time.time()
        self.election_timeout = random.uniform(0.5, 1.5)
        self.log_info("RECOVERED")


async def demo():
    print("=== Raft Leader Election Demo ===\n")

    # Create 5-node cluster
    nodes = [RaftNode(node_id=i) for i in range(1, 6)]
    for n in nodes:
        n.peers = [other for other in nodes if other.node_id != n.node_id]

    # Run all nodes concurrently
    tasks = [asyncio.create_task(node.run()) for node in nodes]

    # Let election happen
    await asyncio.sleep(2.0)

    leader = next((n for n in nodes if n.state == NodeState.LEADER), None)
    print(f"\n--- Current leader: Node {leader.node_id if leader else '(none)'} ---\n")

    # Crash the leader
    if leader:
        print(f"\n*** Crashing leader Node {leader.node_id} ***\n")
        leader.crash()
        await asyncio.sleep(2.5)

    new_leader = next((n for n in nodes if n.state == NodeState.LEADER and n.alive), None)
    print(f"\n--- New leader after crash: Node {new_leader.node_id if new_leader else '(none)'} ---")
    print(f"    Terms advanced: {max(n.current_term for n in nodes)}")

    for task in tasks:
        task.cancel()


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 2: Quorum Calculator & Fault Tolerance Analyzer

```python
def quorum_analysis(n_nodes: int) -> dict:
    """Calculate quorum, fault tolerance, and availability trade-offs for N nodes."""
    quorum = n_nodes // 2 + 1
    max_failures = n_nodes - quorum  # = floor((N-1)/2)

    return {
        "nodes": n_nodes,
        "quorum": quorum,
        "max_tolerated_failures": max_failures,
        "availability": f"{(1 - (1-0.99)**max_failures*100 if max_failures > 0 else 0.99):.5f}",
    }


def print_quorum_table():
    print(f"{'N nodes':>8}  {'Quorum':>8}  {'Max failures':>13}  {'Write cost':>12}")
    print("-" * 50)
    for n in [1, 2, 3, 4, 5, 6, 7, 9, 11]:
        q = quorum_analysis(n)
        write_cost = f"{q['quorum']}/{n} nodes"
        print(f"{n:>8}  {q['quorum']:>8}  {q['max_tolerated_failures']:>13}  {write_cost:>12}")

    print("""
Insight: 
  Even N nodes give no extra fault tolerance over (N-1) odd nodes
  N=4 tolerates 1 failure (same as N=3) but costs more
  → Always use ODD cluster sizes: 3, 5, 7 nodes
  → 3 nodes: simplest (1 failure OK)
  → 5 nodes: standard production (2 failures OK)
  → 7 nodes: high availability, more write latency
""")


def simulate_network_partition(n_nodes: int, partition_sizes: tuple[int, int]):
    """
    Given a cluster split into two groups, which group can elect a leader?
    """
    quorum = n_nodes // 2 + 1
    group_a, group_b = partition_sizes
    assert group_a + group_b <= n_nodes

    print(f"\nNetwork partition: {n_nodes}-node cluster split {group_a} | {group_b}")
    print(f"  Quorum needed: {quorum}")

    for label, size in [("Group A", group_a), ("Group B", group_b)]:
        can_elect = size >= quorum
        print(f"  {label} ({size} nodes): {'✓ CAN elect leader' if can_elect else '✗ CANNOT elect leader (no quorum)'}")

    if group_a < quorum and group_b < quorum:
        print("  → BOTH groups lack quorum → cluster is unavailable (CAP: consistent, not available)")


if __name__ == "__main__":
    print("=== Quorum Analysis ===\n")
    print_quorum_table()

    print("=== Network Partition Scenarios ===")
    simulate_network_partition(5, (3, 2))   # Majority + minority
    simulate_network_partition(5, (2, 3))   # Same thing
    simulate_network_partition(4, (2, 2))   # Even split: no quorum either side
    simulate_network_partition(7, (4, 3))   # 7-node cluster
```
