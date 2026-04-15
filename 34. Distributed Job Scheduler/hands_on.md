# Distributed Job Scheduler — Hands-On Exercises

## Exercise 1: In-Memory Job Scheduler with Priority Queue and Retries

```python
import asyncio
import heapq
import time
import uuid
import random
from dataclasses import dataclass, field
from enum import Enum


class JobStatus(str, Enum):
    SCHEDULED = "scheduled"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    DEAD = "dead"


@dataclass
class Job:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    name: str = ""
    handler: str = ""
    payload: dict = field(default_factory=dict)
    priority: int = 5              # 1 = highest, 10 = lowest
    run_at: float = field(default_factory=time.time)
    max_retries: int = 3
    retry_count: int = 0
    status: JobStatus = JobStatus.SCHEDULED
    result: str | None = None
    error: str | None = None

    # Comparator for heapq (by run_at, then priority)
    def __lt__(self, other: "Job") -> bool:
        if self.run_at != other.run_at:
            return self.run_at < other.run_at
        return self.priority < other.priority


# ---------- Simulated job handlers ----------
# Return None on success, raise Exception on failure

async def send_email(payload: dict) -> str:
    await asyncio.sleep(0.1)
    if random.random() < 0.25:  # 25% failure rate for demo
        raise Exception("SMTP connection refused")
    return f"Email sent to {payload.get('to', '?')}"


async def generate_report(payload: dict) -> str:
    await asyncio.sleep(0.2)
    return f"Report {payload.get('report_id', '?')} generated"


async def cleanup_temp_files(payload: dict) -> str:
    await asyncio.sleep(0.05)
    return "Cleaned 42 temp files"


HANDLERS = {
    "send_email":         send_email,
    "generate_report":    generate_report,
    "cleanup_temp_files": cleanup_temp_files,
}

RETRY_DELAYS = [0, 5, 30, 120, 600]  # seconds


class InMemoryJobScheduler:
    def __init__(self):
        self._queue: list[Job] = []     # min-heap
        self._jobs: dict[str, Job] = {}  # id → Job
        self._lock = asyncio.Lock()

    def schedule(self, job: Job):
        """Add a job to the scheduler."""
        self._jobs[job.id] = job
        heapq.heappush(self._queue, job)
        print(f"  [SCHEDULED] {job.name} (id={job.id}) run_at=t+{job.run_at - time.time():.1f}s "
              f"priority={job.priority}")

    async def _next_due(self) -> Job | None:
        """Return the next job that is due, or None if nothing is ready."""
        async with self._lock:
            now = time.time()
            if self._queue and self._queue[0].run_at <= now:
                job = heapq.heappop(self._queue)
                if job.status == JobStatus.SCHEDULED:
                    return job
            return None

    async def _execute(self, job: Job):
        handler = HANDLERS.get(job.handler)
        if not handler:
            job.status = JobStatus.DEAD
            job.error = f"Unknown handler: {job.handler}"
            print(f"  [DEAD]     {job.name} — unknown handler")
            return

        job.status = JobStatus.RUNNING
        print(f"  [RUNNING]  {job.name} (attempt {job.retry_count + 1}/{job.max_retries + 1})")
        try:
            result = await handler(job.payload)
            job.status = JobStatus.SUCCEEDED
            job.result = result
            print(f"  [SUCCESS]  {job.name} → {result}")
        except Exception as exc:
            job.retry_count += 1
            if job.retry_count <= job.max_retries:
                delay = RETRY_DELAYS[min(job.retry_count, len(RETRY_DELAYS) - 1)]
                job.run_at = time.time() + delay
                job.status = JobStatus.SCHEDULED
                async with self._lock:
                    heapq.heappush(self._queue, job)
                print(f"  [RETRY]    {job.name} attempt={job.retry_count} "
                      f"in {delay}s — error: {exc}")
            else:
                job.status = JobStatus.DEAD
                job.error = str(exc)
                print(f"  [DEAD]     {job.name} — all retries exhausted: {exc}")

    async def run(self, workers: int = 3, run_for_seconds: float = 10.0):
        """Run the scheduler loop with N concurrent workers."""
        print(f"\n>>> Scheduler started ({workers} workers, running for {run_for_seconds}s)")
        stop_at = time.time() + run_for_seconds
        running_tasks: set[asyncio.Task] = set()

        while time.time() < stop_at or running_tasks:
            # Spawn new tasks up to worker limit
            while len(running_tasks) < workers:
                job = await self._next_due()
                if job is None:
                    break
                task = asyncio.create_task(self._execute(job))
                running_tasks.add(task)
                task.add_done_callback(running_tasks.discard)

            await asyncio.sleep(0.1)  # Poll interval

        print("\n>>> Scheduler stopped")
        self.print_summary()

    def print_summary(self):
        from collections import Counter
        counts = Counter(j.status for j in self._jobs.values())
        print(f"\n=== Summary: {len(self._jobs)} jobs ===")
        for status, count in sorted(counts.items()):
            print(f"  {status:12s}: {count}")


async def main():
    scheduler = InMemoryJobScheduler()
    now = time.time()

    # Mix of jobs: different priorities, some immediate, some delayed
    jobs = [
        Job(name="weekly-report",    handler="generate_report",    payload={"report_id": "RPT-001"},
            priority=2, run_at=now),
        Job(name="welcome-email-1",  handler="send_email",          payload={"to": "alice@example.com"},
            priority=5, run_at=now),
        Job(name="welcome-email-2",  handler="send_email",          payload={"to": "bob@example.com"},
            priority=5, run_at=now),
        Job(name="urgent-alert",     handler="send_email",          payload={"to": "admin@example.com"},
            priority=1, run_at=now),   # Priority 1 → runs first
        Job(name="cleanup",          handler="cleanup_temp_files",  payload={},
            priority=8, run_at=now + 2),  # Delayed 2s + low priority
        Job(name="monthly-report",   handler="generate_report",     payload={"report_id": "RPT-002"},
            priority=3, run_at=now + 1),
        Job(name="marketing-email",  handler="send_email",          payload={"to": "list@example.com"},
            priority=9, run_at=now,   max_retries=1),  # Low priority, fewer retries
    ]

    for job in jobs:
        scheduler.schedule(job)

    await scheduler.run(workers=3, run_for_seconds=15.0)


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Exercise 2: Cron Expression Evaluator + Next-Run Calculator

```python
from datetime import datetime, timezone
from typing import Iterator


def parse_cron_field(field: str, min_val: int, max_val: int) -> set[int]:
    """Parse a single cron field like '*/5', '1,3,5', '8-20', or '*'."""
    values = set()
    for part in field.split(","):
        if "/" in part:
            base, step = part.split("/", 1)
            step = int(step)
            r = range(min_val, max_val + 1) if base == "*" else \
                range(*map(int, base.split("-")[:2]), 1)
            values.update(x for x in r if (x - min_val) % step == 0)
        elif "-" in part:
            a, b = map(int, part.split("-", 1))
            values.update(range(a, b + 1))
        elif part == "*":
            values.update(range(min_val, max_val + 1))
        else:
            values.add(int(part))
    return values


class CronSchedule:
    """
    Cron expression format: "minute hour day_of_month month day_of_week"
    Examples:
      "0 8 * * *"     → 8:00am every day
      "*/15 * * * *"  → every 15 minutes
      "0 0 1 * *"     → midnight on the 1st of each month
      "30 9 * * 1-5"  → 9:30am weekdays
    """

    def __init__(self, expression: str):
        parts = expression.strip().split()
        if len(parts) != 5:
            raise ValueError(f"Expected 5 fields, got {len(parts)}: '{expression}'")
        self.minutes    = parse_cron_field(parts[0], 0, 59)
        self.hours      = parse_cron_field(parts[1], 0, 23)
        self.days       = parse_cron_field(parts[2], 1, 31)
        self.months     = parse_cron_field(parts[3], 1, 12)
        self.weekdays   = parse_cron_field(parts[4], 0, 6)   # 0=Sun, 6=Sat
        self.expression = expression

    def next_after(self, after: datetime) -> datetime:
        """Find the next run time strictly after `after`."""
        from datetime import timedelta
        # Start one minute after `after`
        dt = after.replace(second=0, microsecond=0) + timedelta(minutes=1)

        # Search up to 4 years (to handle rare leaps)
        for _ in range(2 * 365 * 24 * 60):
            if (dt.month in self.months and
                dt.day in self.days and
                dt.weekday() % 7 in self.weekdays and   # Python: 0=Mon → adjust
                dt.hour in self.hours and
                dt.minute in self.minutes):
                return dt
            dt += timedelta(minutes=1)
        raise ValueError(f"No valid next run found for '{self.expression}'")

    def upcoming(self, count: int, after: datetime | None = None) -> list[datetime]:
        """Return the next `count` scheduled times."""
        result = []
        current = after or datetime.now(timezone.utc)
        for _ in range(count):
            current = self.next_after(current)
            result.append(current)
        return result


def demo():
    now = datetime(2024, 1, 15, 10, 30, 0, tzinfo=timezone.utc)  # Monday 10:30 UTC

    schedules = [
        ("0 8 * * *",     "Daily 8am"),
        ("*/15 * * * *",  "Every 15 minutes"),
        ("0 0 1 * *",     "Midnight 1st of month"),
        ("30 9 * * 1-5",  "9:30am weekdays"),
        ("0 */6 * * *",   "Every 6 hours"),
    ]

    for expr, label in schedules:
        cron = CronSchedule(expr)
        upcoming = cron.upcoming(3, after=now)
        print(f"\n{label} ({expr})")
        for i, dt in enumerate(upcoming, 1):
            print(f"  Next {i}: {dt.strftime('%Y-%m-%d %H:%M UTC')}")


if __name__ == "__main__":
    demo()
```

**Expected output:**
```
Daily 8am (0 8 * * *)
  Next 1: 2024-01-16 08:00 UTC
  Next 2: 2024-01-17 08:00 UTC
  Next 3: 2024-01-18 08:00 UTC

Every 15 minutes (*/15 * * * *)
  Next 1: 2024-01-15 10:45 UTC
  Next 2: 2024-01-15 11:00 UTC
  Next 3: 2024-01-15 11:15 UTC

Midnight 1st of month (0 0 1 * *)
  Next 1: 2024-02-01 00:00 UTC
  ...
```

---

## Exercise 3: Distributed Lock with Leader Election

```python
import asyncio
import time
import uuid

import redis.asyncio as aioredis

LOCK_KEY = "scheduler:leader_lock"
LOCK_TTL = 10        # seconds
RENEW_INTERVAL = 5   # renew every 5s (< TTL)


async def scheduler_leader(redis: aioredis.Redis, instance_id: str):
    """
    Attempt to become the scheduler leader. If successful, run the scheduler loop.
    If another instance holds the lock, wait and retry.
    """
    held = False
    try:
        while True:
            now_leader = await redis.set(LOCK_KEY, instance_id, nx=True, ex=LOCK_TTL)

            if now_leader:
                print(f"[{instance_id}] Acquired leader lock")
                held = True

                # Scheduler loop: renew lock and do work
                for tick in range(1, 100):
                    # Renew lock (atomic: only renew if we still own it)
                    lua_renew = """
                    if redis.call('GET', KEYS[1]) == ARGV[1] then
                        return redis.call('EXPIRE', KEYS[1], ARGV[2])
                    else
                        return 0
                    end
                    """
                    renewed = await redis.eval(lua_renew, 1, LOCK_KEY, instance_id, LOCK_TTL)
                    if not renewed:
                        print(f"[{instance_id}] Lost lock — another instance took over")
                        held = False
                        break

                    print(f"[{instance_id}] Tick {tick}: Running scheduler poll...")
                    await asyncio.sleep(RENEW_INTERVAL)

                if held:
                    break  # Normal exit after demo loop
            else:
                current_holder = await redis.get(LOCK_KEY)
                print(f"[{instance_id}] Lock held by {current_holder} — waiting...")
                await asyncio.sleep(2)

    finally:
        if held:
            # Release lock only if we still own it
            lua_release = """
            if redis.call('GET', KEYS[1]) == ARGV[1] then
                return redis.call('DEL', KEYS[1])
            else
                return 0
            end
            """
            released = await redis.eval(lua_release, 1, LOCK_KEY, instance_id)
            if released:
                print(f"[{instance_id}] Released leader lock")


async def simulate_two_instances():
    redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)

    instance_a = f"sched-{uuid.uuid4().hex[:6]}"
    instance_b = f"sched-{uuid.uuid4().hex[:6]}"

    # Run both concurrently — only one should become leader
    await asyncio.gather(
        scheduler_leader(redis, instance_a),
        scheduler_leader(redis, instance_b),
    )

    await redis.aclose()


if __name__ == "__main__":
    asyncio.run(simulate_two_instances())
```
