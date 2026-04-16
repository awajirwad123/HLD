# Distributed Job Scheduler — Notes & Reference

## Job Scheduler vs Message Queue

| Aspect | Job Scheduler | Message Queue (Kafka/SQS) |
|---|---|---|
| Timing | Precise time-based execution | Process as fast as possible |
| State | Tracks job status, retries, history | Fire-and-forget per message |
| Cron | Native support | External trigger needed |
| Visibility | Job execution history, dashboards | Limited per-message tracking |
| Examples | Celery Beat, Quartz, Sidekiq Cron | Kafka, RabbitMQ, SQS |
| Use case | "Run at 8am daily" | "Process this event ASAP" |

---

## Distributed Lock Patterns

### Pattern 1: Redis SET NX EX (Simple Leader Election)

```redis
SET scheduler:lock {instance_id} NX EX 10
```

- `NX` = only set if key does NOT exist
- `EX 10` = auto-expire after 10 seconds (crash protection)
- Returns OK if acquired, nil if already locked

**Renewal (Lua script for atomicity):**
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('EXPIRE', KEYS[1], ARGV[2])
else return 0 end
```

**Release (Lua script for atomicity):**
```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else return 0 end
```

Why Lua? GET + DEL must be atomic. Without Lua, two releases could interleave.

### Pattern 2: PostgreSQL Advisory Locks

```sql
SELECT pg_try_advisory_lock(42);   -- 42 = any agreed integer key
-- Returns TRUE if lock acquired, FALSE otherwise
-- Auto-released when session ends (crash safe)
```

Better for: already using Postgres; simpler than Redis; naturally tied to DB transactions.

### Pattern 3: DB Row-Level Locking (for job stealing)

```sql
SELECT * FROM jobs
WHERE status = 'scheduled' AND next_run_at <= NOW()
ORDER BY priority, next_run_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` = instead of blocking, skip rows already locked by other workers. Each worker atomically grabs a different job. No external lock needed.

---

## Cron Expression Reference

```
┌─ minute (0-59)
│ ┌─ hour (0-23)
│ │ ┌─ day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌─ day of week (0=Sun, 6=Sat)
│ │ │ │ │
* * * * *

Common patterns:
  "*/5 * * * *"      Every 5 minutes
  "0 * * * *"        Every hour at :00
  "0 8 * * *"        Daily at 8am
  "0 8 * * 1-5"      Weekdays at 8am
  "0 0 1 * *"        Monthly: midnight 1st
  "0 0 * * 0"        Weekly: midnight Sunday
  "0 */6 * * *"      Every 6 hours
  "30 3 * * 6,0"     3:30am Saturday and Sunday
```

---

## Job State Machine

```
         schedule()
PENDING ──────────────→ SCHEDULED
                              │
 scheduler poll (due?)        │
                              ▼
           lock acquired → ENQUEUED ──→ worker picks up → RUNNING
                                                               │
                                               ┌──────────────┤
                                               ↓              ↓
                                         SUCCEEDED         FAILED
                                                               │
                                          retry_count ≤ max? ─┤
                                               │               │
                                          SCHEDULED          DEAD
                                        (rescheduled)
```

---

## Exactly-Once vs At-Least-Once

| Guarantee | Implementation | Trade-off |
|---|---|---|
| At-least-once | Dequeue → execute → ack. Crash before ack = re-run | Simple. Handler must be idempotent. |
| Exactly-once | Dequeue + DB lock → execute → commit (same transaction) | Hard with external side effects (API calls, emails) |
| At-most-once | Ack before execute | Risk of lost jobs; use only for best-effort notifications |

**Practical recommendation:** At-least-once with idempotent handlers. For side effects (emails, payments), use an idempotency key:
```python
async def send_email(payload):
    key = payload["idempotency_key"]
    if await redis.set(f"email_sent:{key}", "1", nx=True, ex=86400):
        await smtp.send(...)  # Only send if key was fresh
```

---

## Failure Modes and Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Scheduler crashes | No new jobs enqueued until restart | Distributed lock TTL expires → standby takes over in ≤ 10s |
| Worker crashes mid-job | Job stuck in RUNNING | Heartbeat timeout detection → steal after N seconds |
| DB slow | Scheduler poll delayed | Index on `(next_run_at, priority)` + connection pool |
| Queue overflow | Jobs piling up | Consumer lag alerting; horizontal worker scaling |
| Cron miscalculation | Job fires at wrong time | Always store next_run_at as absolute timestamp in UTC |
| Double-execution | Job runs twice | `UPDATE ... FOR UPDATE SKIP LOCKED` or distributed lock per job_id |

---

## Worker Heartbeat

Workers update a heartbeat timestamp periodically so the scheduler can detect stalled jobs:

```sql
UPDATE job_executions
SET heartbeat_at = NOW()
WHERE id = {execution_id};
```

Scheduler (every 60s):
```sql
SELECT j.id FROM jobs j
JOIN job_executions e ON e.job_id = j.id
WHERE j.status = 'running'
  AND e.heartbeat_at < NOW() - INTERVAL '2 minutes'
→ These jobs are stalled → mark failed, reschedule
```

---

## Queue Depth and Lag Monitoring

| Metric | Alert threshold | Dashboard |
|---|---|---|
| Scheduled jobs past due | > 1000 | Queue depth by priority |
| Job failure rate | > 5% in 5 min | Failure rate rolling window |
| DLQ size | > 10 items | DLQ count |
| Scheduler leader election | Lost + not re-acquired in 30s | Scheduler uptime |
| Worker count | < min_workers | Auto-scale trigger |
| P99 job start latency | > 5s | Time from scheduled → running |

---

## Key Numbers

| Metric | Value |
|---|---|
| Scheduler poll interval | 1 second |
| Distributed lock TTL | 10 seconds |
| Heartbeat interval | 30 seconds |
| Stalled job detection | 2× heartbeat interval = 60s |
| Max retry delays | 0s, 30s, 2min, 10min, 1hr |
| Jobs table index row size | ~50 bytes for (next_run_at, priority) index |
| Fan-out max child jobs | ~10,000 per parent (to avoid single-shard DB write burst) |
| Cron precision | Minute-level (1-second precision if using `*/1 * * * *` with sub-polling) |
