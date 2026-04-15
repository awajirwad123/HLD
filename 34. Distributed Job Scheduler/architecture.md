# Distributed Job Scheduler — Architecture

## Problem Statement

Design a job scheduler that:
- Runs millions of scheduled jobs (cron-like and one-shot) reliably
- Guarantees each job runs at least once (or exactly once) at its scheduled time
- Distributes work across many worker nodes
- Supports retries, priority, job dependencies, and monitoring
- Examples: daily email digest at 8am, retry queue flush every 30s, report generation at midnight

---

## Types of Scheduled Work

| Type | Description | Example |
|---|---|---|
| One-shot (delayed) | Run exactly once at a future time | Send OTP expiry reminder in 10 min |
| Recurring (cron) | Run on a schedule indefinitely | Daily report at midnight |
| Fan-out | One trigger spawns N child jobs | "Process all users" → 1M individual jobs |
| Chained (dependent) | Job B starts only after Job A succeeds | ETL pipeline stages |
| Priority queued | Jobs with different urgency levels | Critical batch vs. low-priority cleanup |

---

## High-Level Architecture

```
Producers (API, cron triggers)
         ↓
  ┌──────────────────┐
  │  Job Registry DB │  ← Stores all job definitions, schedules, state
  │  (PostgreSQL)    │
  └──────────────────┘
         ↓
  ┌──────────────────┐
  │   Scheduler      │  ← Polls DB every second; enqueues due jobs
  │   (single leader)│  ← Uses distributed lock to prevent double-enqueue
  └──────────────────┘
         ↓
  ┌──────────────────┐
  │   Job Queue      │  ← Redis Sorted Set or Kafka or RabbitMQ
  │   (per priority) │
  └──────────────────┘
         ↓
  ┌──────────────────────────────────┐
  │        Worker Pool               │
  │  Worker 1 | Worker 2 | Worker N  │
  │  - Dequeue job                   │
  │  - Execute handler               │
  │  - Mark done / retry on failure  │
  └──────────────────────────────────┘
         ↓
  ┌──────────────────┐
  │   Execution Log  │  ← Every run: start/end time, status, logs
  └──────────────────┘
```

---

## Job Lifecycle

```
SCHEDULED → ENQUEUED → RUNNING → SUCCEEDED
                              ↘ FAILED → retry? → ENQUEUED (attempt+1)
                                              → DEAD (max retries)
         → CANCELLED
```

---

## Job Registry Schema

```sql
CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    handler         TEXT NOT NULL,           -- name of function/class to invoke
    payload         JSONB DEFAULT '{}',
    schedule        TEXT,                    -- NULL = one-shot; cron expression for recurring
    next_run_at     TIMESTAMPTZ NOT NULL,
    priority        SMALLINT DEFAULT 5,      -- 1 (highest) to 10 (lowest)
    max_retries     SMALLINT DEFAULT 3,
    retry_count     SMALLINT DEFAULT 0,
    timeout_seconds INT DEFAULT 300,
    status          TEXT NOT NULL DEFAULT 'scheduled'
                    CHECK (status IN ('scheduled','enqueued','running','succeeded','failed','dead','cancelled')),
    created_by      TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Index for the scheduler poll query
CREATE INDEX idx_jobs_due ON jobs (next_run_at, priority)
    WHERE status = 'scheduled';

CREATE TABLE job_executions (
    id              BIGSERIAL PRIMARY KEY,
    job_id          UUID REFERENCES jobs(id),
    worker_id       TEXT,
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ,
    status          TEXT CHECK (status IN ('running','succeeded','failed')),
    output          TEXT,
    error           TEXT
);
CREATE INDEX idx_executions_job ON job_executions (job_id, started_at DESC);
```

---

## The Scheduler Component

The scheduler's job: poll for due jobs and atomically move them from `scheduled` → `enqueued`.

### Distributed Lock (Preventing Double-Scheduling)

Only one scheduler instance should run at a time. Use a Redis distributed lock:

```
Scheduler A: SET scheduler:lock {instance_id} NX EX 10
  → Success: A is the leader for 10 seconds
  → A polls DB, enqueues due jobs, extends lock
Scheduler B: SET scheduler:lock ... NX → Fails → A already holds the lock → B waits
```

If Scheduler A crashes, the lock TTL of 10 seconds expires → Scheduler B takes over.

### Scheduler Poll Query

```sql
UPDATE jobs
SET status = 'enqueued', updated_at = NOW()
WHERE status = 'scheduled'
  AND next_run_at <= NOW()
RETURNING id, name, handler, payload, priority, timeout_seconds;
```

This atomic `UPDATE ... RETURNING` prevents two schedulers from picking up the same job.

---

## Job Queue (Priority-Based)

Use a Redis Sorted Set per priority tier, or a single set with composite score:

```
score = priority * 10^12 + unix_timestamp_ms

# Lower score = higher priority and earlier submission
ZADD job_queue {score} {job_id}
ZPOPMIN job_queue  → returns job with lowest score (highest priority, earliest time)
```

Or use Kafka with priority topics (separate topics per priority level, consumers round-robin with weights).

### At-Least-Once vs Exactly-Once

**At-least-once (simpler):**
- Worker dequeues job, executes, marks done
- If worker crashes after execution but before marking done: job reruns
- Handler must be idempotent

**Exactly-once (harder):**
- Two-phase: worker locks job with its ID (`UPDATE jobs SET status='running', worker_id={id} WHERE status='enqueued' AND id={job_id}`) → executes → marks done
- If another worker sees `status='running'` with a stale `started_at`, it can steal the job after timeout

---

## Heartbeat and Job Timeout Detection

A job that runs longer than `timeout_seconds` may have crashed. Recovery:

```
Worker: every 30s, update job_executions.heartbeat_at = NOW()

Scheduler (background loop):
  SELECT id FROM jobs
  WHERE status = 'running'
    AND updated_at < NOW() - INTERVAL '1 second' * timeout_seconds
  → Mark these as 'failed', increment retry_count, reschedule
```

---

## Retry Logic

```python
if retry_count < max_retries:
    delay = 2 ** retry_count * base_delay_seconds  # exponential backoff
    next_run_at = NOW() + delay
    UPDATE jobs SET status='scheduled', next_run_at=..., retry_count=retry_count+1
else:
    UPDATE jobs SET status='dead'
    → Publish to DLQ / alert on-call
```

---

## Cron Next-Run Calculation

After a recurring job completes, compute next scheduled time:

```python
from croniter import croniter
from datetime import datetime, timezone

def next_run(cron_expr: str, base: datetime) -> datetime:
    itr = croniter(cron_expr, base)
    return itr.get_next(datetime).replace(tzinfo=timezone.utc)

# "0 8 * * *" → 8:00am daily
next_run("0 8 * * *", datetime.now(timezone.utc))
```

After success: `UPDATE jobs SET next_run_at = next_run(schedule, NOW()), status='scheduled'`

---

## Fan-Out Jobs

"Send daily digest to all 10M users" — don't put this in one job!

Implementation:
1. One parent job runs at midnight, creates 10,000 child jobs (shards of 1,000 users each).
2. Parent records `child_count` and waits.
3. Each child finishes, increments a Redis counter.
4. When counter reaches `child_count`, parent transitions to `succeeded`.

```sql
ALTER TABLE jobs ADD COLUMN parent_id UUID REFERENCES jobs(id);
```

---

## Key Numbers

| Metric | Value |
|---|---|
| Jobs per second (typical) | 100–10,000/sec |
| Scheduler poll interval | 1 second |
| Distributed lock TTL | 10 seconds |
| Job timeout detection lag | Max 30s after heartbeat TTL |
| Postgres jobs table size | 1B rows × 500 bytes = 500GB (with archival) |
| Queue depth alert threshold | 10,000 unprocessed jobs |
| Max retry delays | 0s, 30s, 2min, 10min, 1hr |
