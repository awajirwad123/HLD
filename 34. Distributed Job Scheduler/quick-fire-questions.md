# Distributed Job Scheduler — Quick-Fire Questions

**Q1: What is the difference between a job scheduler and a message queue?**

A job scheduler is time-aware: it executes jobs at specific scheduled times (cron, one-shot delay). It tracks execution state, history, and retry counts. A message queue (Kafka, SQS) processes messages as fast as possible without built-in time awareness. Job schedulers typically use message queues internally as the execution backend.

---

**Q2: How do you prevent two scheduler instances from enqueuing the same job twice?**

Use a distributed lock (Redis `SET NX EX`) so only one scheduler instance is active at a time (leader election). Additionally, use an atomic DB update: `UPDATE jobs SET status='enqueued' WHERE status='scheduled' AND next_run_at <= NOW()` — no two workers can enqueue the same row because they're racing on a state transition that only one can win.

---

**Q3: What is `FOR UPDATE SKIP LOCKED` and when is it useful?**

A PostgreSQL clause on `SELECT ... FOR UPDATE` that skips rows already locked by other transactions instead of blocking. Multiple workers can poll the same jobs table concurrently and each atomically grabs a different row. This removes the need for an external distributed lock for job assignment — the DB provides the coordination.

---

**Q4: A worker crashes mid-execution. How do you detect and recover the job?**

Workers write a heartbeat timestamp to the `job_executions` table every 30 seconds. The scheduler runs a background query to find jobs in `RUNNING` status whose heartbeat is older than 2× the heartbeat interval. These are considered stalled, marked as `FAILED`, and rescheduled (with retry_count incremented). The stalled detection lag is at most 60–90 seconds.

---

**Q5: What does it mean for a job handler to be idempotent? Give an example.**

An idempotent handler produces the same outcome whether it runs once or many times. Example: `send_welcome_email` — before sending, check `SELECT EXISTS (SELECT 1 FROM emails_sent WHERE user_id=$1 AND type='welcome')`. Insert + send only if that row doesn't exist. Even if the job reruns (at-least-once delivery), the user receives only one email.

---

**Q6: How do you implement at-exactly-once job execution?**

True exactly-once requires both job dequeue and the side effect to be in the same atomic transaction — only possible when the side effect is a DB write. Example: dequeue job + `INSERT INTO reports (...)` in one transaction. For external side effects (API calls, emails), exactly-once is impossible; use at-least-once with idempotency keys at the target system.

---

**Q7: What are the components of a cron expression? How would you calculate the next run time?**

Five space-separated fields: minute, hour, day-of-month, month, day-of-week. Each can be `*`, a number, a range (`1-5`), a list (`1,3,5`), or a step (`*/15`). To calculate next run: start from (now + 1 minute), advance minute by minute, and check whether all five fields match. Libraries like `croniter` (Python) or `node-cron` (Node.js) implement this efficiently.

---

**Q8: How do you handle a cron job that takes longer than its interval?**

Two strategies: (1) Skip — if the previous run is still executing when the next trigger fires, skip that trigger (prevents overlap). (2) Queue — allow overlap; each firing creates a new independent execution. Skip is safer for most cron jobs (reports, cleanup). Queue-up is only acceptable if the logic is truly idempotent and stateless. Mark policy in the job definition: `overlap_policy: skip | queue`.

---

**Q9: What is a fan-out job and why is it a problem if done naively?**

A fan-out job creates N child jobs — e.g., "send digest to all 10M users." If the parent creates 10M individual jobs in a tight loop, it overwhelms the DB with inserts and floods the job queue. Proper approach: create batched child jobs (10K users each = 1,000 child jobs), track completion via a counter, and transition the parent to `SUCCEEDED` when all children complete.

---

**Q10: How do you prioritize urgent jobs over background jobs in the same queue?**

Use a priority queue: Redis Sorted Set with `score = priority × 10^12 + scheduled_at_ms`. Lower score = higher priority. Workers always `ZPOPMIN`, so urgent jobs (priority=1) are always processed before cleanup jobs (priority=10). Alternatively, use separate Kafka topics or queues per priority tier with a weighted consumer.

---

**Q11: How do you prevent a cron job from being scheduled multiple times due to clock drift between scheduler nodes?**

Two protections: (1) Leader election — only one scheduler node is active (holds the distributed lock). (2) Atomic DB state transition — `UPDATE ... WHERE status='scheduled' AND next_run_at <= NOW()` can only be applied once per job. Even if two schedulers somehow both try to enqueue at the same instant, the DB UPDATE is atomic; only one succeeds.

---

**Q12: How do you implement a "run at most once" job (at-most-once semantics)?**

Acknowledge (delete) the job from the queue before executing it. If the worker crashes after ack but before execution completes, the job is lost — which is the intended at-most-once behavior. Use case: fire-and-forget telemetry, cache warm-up. Not suitable for critical operations. Most job schedulers default to at-least-once; at-most-once requires explicit design.

---

**Q13: What is a dead letter queue in the context of job scheduling?**

A DLQ is where jobs land after exhausting all retry attempts. Instead of deleting failed jobs, they're moved to a separate store (a dedicated DB table, SQS DLQ, or Kafka topic) for human review, alerting, and manual reprocessing. The DLQ size is a critical health metric: a growing DLQ means systemic failures that aren't self-healing.
