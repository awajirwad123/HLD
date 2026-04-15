# Distributed Job Scheduler — Interview Simulator

## Scenario 1: "Design a Distributed Job Scheduler (Like Sidekiq / Celery Beat)"

**Prompt:** "Design a job scheduling system that supports cron-based recurring jobs and one-shot delayed jobs for a large-scale web application. It must be fault-tolerant and handle millions of job executions per day."

---

**Clarifying questions to ask:**

1. What's the scale? (Jobs/day, concurrent workers needed)
2. What execution guarantees are needed — at-least-once or exactly-once?
3. Do we need job dependencies (Job B after Job A)?
4. What's the acceptable latency from scheduled time to execution start?
5. Should job outputs/logs be stored for debugging?

---

**Model answer outline:**

**Scale estimates:**
- 1M jobs/day = ~12 jobs/sec average; peak 10× = 120/sec
- 1,000 workers can handle this comfortably at 1 job/sec each

**Core components:**

```
1. Jobs Table (PostgreSQL)
   - job definitions, schedules, state, retry config
   - Indexed on (next_run_at, priority) WHERE status='scheduled'

2. Scheduler Service (single leader via distributed lock)
   - Redis SET NX EX 10 for leader election
   - Polls jobs table every second
   - UPDATE jobs SET status='enqueued' WHERE due → enqueues to job queue
   - Standby scheduler takes over within 10s of leader crash

3. Job Queue (Redis Sorted Set or Kafka)
   - Score = (priority × 10^12) + epoch_ms
   - ZPOPMIN for highest-priority, earliest-scheduled jobs

4. Worker Pool (horizontal scale)
   - Each worker: ZPOPMIN → execute handler → mark succeeded/failed
   - Heartbeat every 30s for stall detection
   - Exponential backoff retries (0s, 30s, 2min, 10min, 1hr)

5. Execution Log
   - job_executions table with started_at, finished_at, status, output
   - Retained for 30 days; archived to cold storage
```

**Cron handling:**
- After each successful run: `next_run_at = croniter.next(schedule, now)`
- For overlapping runs: skip policy by default (check for RUNNING instance before enqueuing)

**Failure modes to mention proactively:**
- Scheduler crash → standby takes over (liveness within 10s)
- Worker crash mid-job → heartbeat timeout → stall detection → reschedule
- DB slow → index on due jobs keeps poll query fast; connection pool with circuit breaker
- Queue overflow → horizontal worker scaling; consumer lag alerting

**Operational features:**
- Pause/resume individual jobs
- Manual trigger (force-run a scheduled job now)
- Job dashboard (per-job execution history, failure rate, avg duration)
- DLQ with alerting when size > threshold

---

## Scenario 2: "GitHub Actions / CI Pipeline Scheduler"

**Prompt:** "Design the job scheduling layer for a CI/CD system (like GitHub Actions). Developers push code → a 'build and test' job runs. Support parallel jobs within a workflow (e.g., test-backend and test-frontend run in parallel, then deploy runs after both succeed)."

---

**Unique challenges vs. a simple cron scheduler:**

1. **DAG execution:** Each workflow is a directed acyclic graph of jobs. `deploy` depends on `test-backend` AND `test-frontend` both succeeding (fan-in dependency).
2. **Git event triggers:** Jobs aren't time-triggered; they're triggered by webhooks (push, PR open, tag created).
3. **Exactly-once required:** A push event should trigger exactly one build, not two (if two webhook deliveries arrive).
4. **Resource isolation:** Different jobs run in different containers (different languages, dependencies). Worker = container runner.
5. **Long-running jobs:** Build jobs can run for 30–60 minutes. Standard 10-minute heartbeat TTL won't work.
6. **Cancellation:** A second push while the first build runs should cancel the in-progress build.

**Model system:**

```
GitHub webhook → Event Ingestor
  → Idempotency check: SET processed:{delivery_id} NX EX 86400
  → Resolve workflow YAML from repo (defines the job DAG)
  → Create Workflow Run record in DB
  → Create individual Job records for each step in the DAG

Workflow Executor:
  - Finds jobs with status='waiting' whose ALL dependencies have status='succeeded'
  - Enqueues them for a container runner

Container Runners:
  - Spin up isolated container per job
  - Execute build/test commands
  - Stream logs to object storage (S3)
  - Report completion → update job status

DAG advancement:
  Job completes → check if any downstream jobs now have all deps met → enqueue them
  All jobs complete → mark Workflow Run succeeded
  Any job fails → cascade cancel downstream dependents (or continue-on-error mode)
```

**DB schema additions:**
```sql
CREATE TABLE workflow_runs (id, repo, commit_sha, trigger_type, status, created_at);
CREATE TABLE workflow_jobs (id, run_id, name, status, depends_on UUID[], runner_id, started_at, finished_at);
CREATE TABLE job_logs (job_id, chunk_order, content, stored_at);  -- S3 reference in practice
```

**Cancellation:**
- New push arrives → cancel all jobs in the previous run for same branch
- `UPDATE workflow_jobs SET status='cancelled' WHERE run_id=? AND status IN ('waiting','enqueued')`
- For `RUNNING` jobs: send SIGTERM to the container runner

---

## Scenario 3: "Debugging — Scheduled Jobs Are Running an Hour Late"

**Prompt:** "Ops team reports that nightly jobs scheduled for midnight are consistently executing at ~1am. How do you debug and fix this?"

---

**Systematic investigation:**

**Step 1: Check what time-zone the timestamps are stored in**

```sql
SELECT next_run_at, timezone('UTC', next_run_at) AS utc_time FROM jobs WHERE name='nightly_report' LIMIT 5;
```

Common bug: `next_run_at` was inserted as local time (midnight EST = 05:00 UTC) but the scheduler interprets it as UTC. So midnight EST appears as 5am UTC → "delayed by 5 hours." Or if server time shifted from UTC to UTC-1 when a VM migrated regions.

**Step 2: Check scheduler clock**

```bash
date -u                         # Current UTC time on scheduler node
SELECT NOW() AT TIME ZONE 'UTC' FROM scheduler_heartbeats;
```

Is the DB and scheduler node in sync? If the scheduler node has wrong NTP config, `next_run_at <= NOW()` comparison is skewed.

**Step 3: Measure actual execution latency**

```sql
SELECT name,
       next_run_at,
       e.started_at,
       EXTRACT(EPOCH FROM (e.started_at - j.next_run_at)) AS lag_seconds
FROM jobs j
JOIN job_executions e ON e.job_id = j.id
WHERE j.name = 'nightly_report'
ORDER BY e.started_at DESC LIMIT 10;
```

If lag is consistently ~3600 seconds: strong indicator of a 1-hour timezone offset error in the `next_run_at` calculation.

**Step 4: Check DST transition**

Did the lag start when Daylight Saving Time changed? If `next_run_at` is calculated using `datetime.now()` in a local timezone instead of `datetime.now(timezone.utc)`, DST adds/removes 1 hour.

**Fix:**
```python
# WRONG: using local time
from datetime import datetime
next_run = calculate_next(cron_expr, datetime.now())

# CORRECT: always UTC
from datetime import datetime, timezone
next_run = calculate_next(cron_expr, datetime.now(timezone.utc))
```

Also verify: all `TIMESTAMPTZ` columns in PostgreSQL store in UTC internally; don't use `TIMESTAMP WITHOUT TIME ZONE` for scheduler columns.

**Post-fix validation:** After deploying the fix, watch the next few midnight runs. The lag should drop from ~3600s to < 5s (just scheduler poll latency).
