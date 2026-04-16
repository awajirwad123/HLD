# Distributed Job Scheduler — Tricky Interview Questions

## Q1: "Your cron job is supposed to run daily at 8am, but sometimes it runs twice or not at all. What are the root causes and fixes?"

**What they're testing:** Understanding of distributed systems failure modes, clock skew, race conditions.

**Double-execution causes:**
1. Two scheduler instances both acquired "the lock" briefly (split-brain: Redis failover during lock transition, Redlock partial failure).
2. The scheduler poll runs every 500ms and the job's `next_run_at` gets picked up on two consecutive polls before the first update commits.
3. DB replication lag: scheduler reads from a replica that hasn't seen the `status='enqueued'` write from 50ms ago.

**Missed-execution causes:**
1. Scheduler crashed precisely at 8am and the standby didn't take over in time.
2. `next_run_at` was stored in local time; DST transition caused it to skip.
3. Job table is partitioned or archived; the due-job index doesn't cover the relevant partition.
4. Scheduler poll query is slow (no index): by the time it finishes, the window passed.

**Fixes:**
- Store all timestamps as UTC; never local time.
- Use `UPDATE ... WHERE status='scheduled' AND id=? RETURNING id` — only one update succeeds.
- Read from DB primary, not replica, for the scheduler poll (no replication lag).
- For split-brain: use fencing tokens. When lock is acquired, get an incremented token. Include token in the job-update query; the DB rejects updates with stale tokens.
- Add post-execution audit: after midnight, verify that every expected cron run actually happened (execution log check). Alert if any were missed.

---

## Q2: "Design a system where Job B must run only after Job A succeeds. How do you implement job dependencies?"

**What they're testing:** DAG scheduling, dependency tracking, ETL workflows.

**Simple dependency model:**

```sql
ALTER TABLE jobs ADD COLUMN depends_on UUID[] DEFAULT '{}';
-- Job B's depends_on = ARRAY[job_A_id]
```

Scheduler rule: a job is only eligible to be enqueued if ALL jobs in `depends_on` have `status = 'succeeded'`.

```sql
SELECT j.id FROM jobs j
WHERE j.status = 'scheduled'
  AND j.next_run_at <= NOW()
  AND NOT EXISTS (
    SELECT 1 FROM jobs dep
    WHERE dep.id = ANY(j.depends_on)
      AND dep.status != 'succeeded'
  );
```

**DAG validation:**
- On job creation, validate that the dependency graph has no cycles (topological sort).
- Store the full execution graph in a `job_dependencies` table for fan-in N-to-1 relationships.

**Failure propagation:**
- If Job A fails and exhausts retries (becomes `dead`): what happens to Job B?
- Two policies: (1) **fail cascade** — mark B as `cancelled` (it can never run). (2) **wait** — B stays `waiting` until A is manually re-run or overridden.

**Real-world:** Apache Airflow uses this exact DAG model. Each task in an Airflow DAG is a "job" with dependency edges. Airflow's scheduler does the same `depends_on` check before enqueuing a task.

**For interviews:** Mention start with a simple linked `depends_on` array, then evolve to a dedicated `job_edges` table for complex DAGs. Add cycle detection at creation time.

---

## Q3: "10 million daily digest jobs are created at midnight. Your DB falls over. How do you fix this and prevent it?"

**What they're testing:** Write amplification, batching, fan-out bottlenecks.

**Root cause:** 10M individual `INSERT INTO jobs ...` at midnight = massive DB write storm. Even at 100K inserts/sec, takes 100 seconds. Index maintenance and MVCC bloat make it worse.

**Fix 1: Fan-out job model**
Don't create 10M individual jobs. Create one parent job with `type='send_digest_to_all'`, which spawns batched child jobs on the fly:
- Parent starts, paginates users 1000 at a time
- Creates child jobs in batches of 1000 per DB transaction
- Each child handles 1000 users
- Total: 10,000 child jobs instead of 10,000,000

**Fix 2: Rate-limited fan-out**
Spread the fan-out over a window:
- Parent creates child jobs at a rate of 1,000/sec
- Workers process children as they're created
- No burst at midnight

**Fix 3: Avoid explicit job records for bulk operations**
Store a single job with `payload = {shard: all}`. The worker itself is responsible for pagination:
```python
async def send_digest(payload):
    cursor = payload.get("cursor", 0)
    while True:
        users = await db.fetch("SELECT id FROM users WHERE id > $1 LIMIT 1000", cursor)
        if not users: break
        for u in users: await send_digest_to_user(u.id)
        cursor = users[-1].id
        await db.update_job_cursor(job_id, cursor)  # checkpoint progress
```

**DB protection:**
- Bulk inserts in transactions of 10K rows, with `pg_sleep(0.1)` between batches (throttle)
- Use a separate jobs table partition for fan-out children (easy cleanup)

---

## Q4: "A job that should run every 5 minutes is taking 6 minutes to complete. What happens, and how should the system behave?"

**What they're testing:** Overlap policy, job queueing semantics, operational safety.

**What happens without overlap protection:**
- t=0: Job A starts (attempt #1)
- t=5: Scheduler fires again → creates attempt #2, which starts while #1 is still running
- t=6: Job A (attempt #1) finishes
- t=10: Job A (attempt #2) finishes
- Two instances running simultaneously: race conditions on shared resources, double processing

**Options:**

1. **Skip policy (default for most cron):** If the previous run is still active when next trigger fires, skip this trigger. Update `next_run_at` only after completion. Simple, but if the job always runs > interval, it effectively becomes "run as fast as possible."

2. **Queue policy:** Allow overlap. Each trigger is independent. Use only for stateless, truly idempotent handlers (log aggregation, metrics collection).

3. **Alert + auto-skip:** Skip overlapping run AND emit an alert: "job X is consistently running longer than its interval." Triggers investigation.

**DB implementation:**
```sql
-- Skip policy: don't enqueue if a RUNNING instance exists
SELECT COUNT(*) FROM jobs
WHERE name = 'my_job' AND status = 'running';
-- If count > 0: skip this trigger cycle
```

**Scheduling adjustment:** After each run, recalculate `next_run_at` based on *after completion* rather than absolute cron schedule. This prevents drift accumulation.

---

## Q5: "How would you implement a distributed scheduler that guarantees sub-second job execution latency for time-sensitive jobs?"

**What they're testing:** Deep understanding of scheduling bottlenecks, alternatives to poll-based schedulers.

**Problem with poll-based schedulers:** Even with 1-second polling, a job might wait up to ~1 second to be picked up. For sub-second targets, this is too long.

**Approach 1: Redis Sorted Set with ZPOPMIN + blocking wait**

```python
# Worker: non-blocking ZPOPMIN loop at 100ms intervals
while True:
    result = await redis.zpopmin("job_queue", count=1)
    if result:
        score, job_id = result[0]
        if score <= time.time():
            await execute(job_id)
    await asyncio.sleep(0.05)  # 50ms poll
```

With workers polling at 50ms, maximum scheduling latency = 50ms. Trade-off: Redis CPU increases with fast polling.

**Approach 2: Scheduler pushes to queue early + worker waits**

Scheduler runs at 100ms intervals (instead of 1s). For high-priority jobs running in the next 500ms, pre-enqueue them. Workers use blocking BLPOP with short timeout.

**Approach 3: Dedicated fast-path for critical jobs**

Separate queue + dedicated workers for `priority=CRITICAL` jobs. These workers poll at 10ms. Majority of jobs (low priority, daily crons) use the normal 1s scheduler. Resource isolation prevents low-priority load from impacting critical job latency.

**Approach 4: Webhook-based triggers (not cron)**

For sub-second triggers on predictable events (API call completed → trigger job), use event-driven triggers (Kafka consumer) rather than a time-based scheduler. The "job" becomes a message handler; latency is Kafka consumer lag, typically 5–50ms.

**Practical answer:** 99% of job schedulers don't need sub-second latency. For the 1% that do, combine a 100ms-polled Redis Sorted Set + dedicated worker pool for critical priority + SLA-based alerting if any critical job exceeds 200ms from scheduled to running.
