# Circuit Breaker & Resilience Patterns — Hands-On Exercises

## Exercise 1: Full Circuit Breaker — Rate-Based, All Three States

```python
"""
Production-quality circuit breaker with:
  - Rate-based failure threshold (not just count-based)
  - Rolling time window
  - Minimum call count before opening
  - CLOSED → OPEN → HALF_OPEN → CLOSED transitions
  - Metrics emission hooks

pip install asyncio
"""
import asyncio
import time
import collections
from enum import Enum
from typing import Callable, Any, Optional
from dataclasses import dataclass, field


class CircuitState(Enum):
    CLOSED    = "closed"
    OPEN      = "open"
    HALF_OPEN = "half_open"


@dataclass
class CircuitBreakerConfig:
    name:               str
    failure_threshold:  float = 0.5    # Open when failure rate > 50%
    rolling_window:     float = 60.0   # Seconds — rolling window for rate calculation
    min_calls:          int   = 10     # Min calls before circuit can open
    recovery_timeout:   float = 30.0   # Seconds in OPEN before trying HALF_OPEN
    success_threshold:  int   = 3      # Successes in HALF_OPEN to close


@dataclass
class _Call:
    """Represents one call result in the rolling window."""
    ts:      float
    success: bool


class CircuitBreaker:
    def __init__(self, config: CircuitBreakerConfig):
        self.cfg              = config
        self._state           = CircuitState.CLOSED
        self._window:         collections.deque[_Call] = collections.deque()
        self._last_opened_at: float = 0.0
        self._half_open_successes: int = 0
        self._lock            = asyncio.Lock()

        # Metrics hooks (replace with Prometheus counters in production)
        self.on_state_change: Optional[Callable] = None
        self.on_rejected:     Optional[Callable] = None

    @property
    def state(self) -> CircuitState:
        return self._state

    def _prune_window(self):
        """Remove entries older than the rolling window."""
        cutoff = time.monotonic() - self.cfg.rolling_window
        while self._window and self._window[0].ts < cutoff:
            self._window.popleft()

    def _failure_rate(self) -> float:
        if not self._window:
            return 0.0
        failures = sum(1 for c in self._window if not c.success)
        return failures / len(self._window)

    def _should_open(self) -> bool:
        self._prune_window()
        if len(self._window) < self.cfg.min_calls:
            return False     # Not enough data to make a decision
        return self._failure_rate() >= self.cfg.failure_threshold

    def _should_try_reset(self) -> bool:
        elapsed = time.monotonic() - self._last_opened_at
        return elapsed >= self.cfg.recovery_timeout

    async def _transition(self, new_state: CircuitState):
        old = self._state
        self._state = new_state
        print(f"[CB:{self.cfg.name}] {old.value} → {new_state.value}")
        if self.on_state_change:
            await self.on_state_change(old, new_state)

    async def call(self, fn: Callable, *args, **kwargs) -> Any:
        async with self._lock:
            # ── OPEN state ──────────────────────────────────────────────────
            if self._state == CircuitState.OPEN:
                if self._should_try_reset():
                    await self._transition(CircuitState.HALF_OPEN)
                    self._half_open_successes = 0
                else:
                    if self.on_rejected:
                        await self.on_rejected()
                    raise CircuitOpenError(
                        f"Circuit '{self.cfg.name}' is OPEN. "
                        f"Retry in {self.cfg.recovery_timeout - (time.monotonic() - self._last_opened_at):.0f}s"
                    )

            # ── HALF_OPEN: only one probe allowed at a time ──────────────────
            if self._state == CircuitState.HALF_OPEN:
                pass  # Allow the one probe request through

        try:
            result = await fn(*args, **kwargs)
            await self._record_success()
            return result

        except Exception as e:
            await self._record_failure()
            raise

    async def _record_success(self):
        async with self._lock:
            self._window.append(_Call(ts=time.monotonic(), success=True))

            if self._state == CircuitState.HALF_OPEN:
                self._half_open_successes += 1
                if self._half_open_successes >= self.cfg.success_threshold:
                    self._window.clear()   # Fresh start after recovery
                    await self._transition(CircuitState.CLOSED)

    async def _record_failure(self):
        async with self._lock:
            self._window.append(_Call(ts=time.monotonic(), success=False))

            if self._state == CircuitState.HALF_OPEN:
                # Single failure in HALF_OPEN → reopen immediately
                self._last_opened_at = time.monotonic()
                await self._transition(CircuitState.OPEN)
                return

            if self._state == CircuitState.CLOSED and self._should_open():
                self._last_opened_at = time.monotonic()
                await self._transition(CircuitState.OPEN)

    def metrics(self) -> dict:
        self._prune_window()
        total    = len(self._window)
        failures = sum(1 for c in self._window if not c.success)
        return {
            "state":        self._state.value,
            "total_calls":  total,
            "failure_count": failures,
            "failure_rate": f"{self._failure_rate():.1%}",
        }


class CircuitOpenError(Exception):
    pass


# ── Demo ──────────────────────────────────────────────────────────────────────
call_num = 0

async def flaky_service():
    global call_num
    call_num += 1
    # Fail calls 5–15, then recover
    if 5 <= call_num <= 15:
        raise ConnectionError(f"Service down (call #{call_num})")
    return f"OK (call #{call_num})"


async def demo():
    cfg = CircuitBreakerConfig(
        name="payment",
        failure_threshold=0.5,
        rolling_window=60,
        min_calls=5,
        recovery_timeout=2.0,   # Short for demo
        success_threshold=2,
    )
    cb = CircuitBreaker(cfg)

    for i in range(25):
        if i == 16:
            print(f"\n[Demo] Waiting {cfg.recovery_timeout}s for recovery timeout...")
            await asyncio.sleep(cfg.recovery_timeout + 0.1)

        try:
            result = await cb.call(flaky_service)
            print(f"  [{i+1:2d}] ✓ {result:25s} | {cb.metrics()['state']:10s} | rate={cb.metrics()['failure_rate']}")
        except CircuitOpenError as e:
            print(f"  [{i+1:2d}] ⚡ FAST-FAIL                   | {cb.metrics()['state']:10s}")
        except ConnectionError as e:
            print(f"  [{i+1:2d}] ✗ {str(e):25s} | {cb.metrics()['state']:10s} | rate={cb.metrics()['failure_rate']}")


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 2: Retry with Exponential Backoff + Jitter

```python
"""
Configurable retry decorator with:
  - Exponential backoff with full jitter (AWS recommendation)
  - Retry budget (max retry ratio to prevent amplification)
  - Per-error-type retry decision
  - Hooks for metrics

pip install asyncio httpx
"""
import asyncio
import random
import time
from dataclasses import dataclass, field
from typing import Callable, Type


@dataclass
class RetryConfig:
    max_attempts:     int          = 3
    base_delay:       float        = 0.5       # Seconds
    max_delay:        float        = 30.0      # Cap on backoff
    jitter:           bool         = True
    retryable_errors: tuple        = (ConnectionError, TimeoutError, OSError)
    retryable_http_codes: set[int] = field(default_factory=lambda: {429, 503, 502, 504})


def compute_backoff(attempt: int, base: float, cap: float, jitter: bool) -> float:
    """Full jitter exponential backoff — recommended by AWS for distributed systems."""
    exponential = min(cap, base * (2 ** attempt))
    if jitter:
        return random.uniform(0, exponential)
    return exponential


class RetryBudget:
    """
    Limits retries to a percentage of total calls.
    Prevents retry storms when a service is overloaded.
    """
    def __init__(self, max_retry_ratio: float = 0.1, window: float = 60.0):
        self.max_ratio   = max_retry_ratio
        self.window      = window
        self._total:   int = 0
        self._retries: int = 0
        self._reset_at: float = time.monotonic() + window

    def has_budget(self) -> bool:
        now = time.monotonic()
        if now > self._reset_at:
            self._total   = 0
            self._retries = 0
            self._reset_at = now + self.window

        self._total += 1
        if self._total == 0:
            return True
        return (self._retries / self._total) < self.max_ratio

    def record_retry(self):
        self._retries += 1


async def retry(
    fn: Callable,
    *args,
    config: RetryConfig = None,
    budget: RetryBudget = None,
    **kwargs,
) -> any:
    """
    Execute fn(*args, **kwargs) with retry logic.
    Raises the last exception if all attempts exhausted.
    """
    cfg      = config or RetryConfig()
    last_exc = None

    for attempt in range(cfg.max_attempts):
        try:
            return await fn(*args, **kwargs)

        except Exception as e:
            last_exc = e
            is_last  = attempt == cfg.max_attempts - 1

            # Check if this error type is retryable
            if not isinstance(e, cfg.retryable_errors):
                raise   # Non-retryable — propagate immediately

            if is_last:
                break   # Exhausted retries

            # Check retry budget
            if budget and not budget.has_budget():
                print(f"  [Retry] Budget exhausted — not retrying (attempt {attempt + 1})")
                raise

            budget and budget.record_retry()

            delay = compute_backoff(attempt, cfg.base_delay, cfg.max_delay, cfg.jitter)
            print(f"  [Retry] Attempt {attempt + 1} failed: {e}. Waiting {delay:.2f}s...")
            await asyncio.sleep(delay)

    raise last_exc


# ── Decorator version ─────────────────────────────────────────────────────────
def with_retry(config: RetryConfig = None):
    def decorator(fn: Callable):
        async def wrapper(*args, **kwargs):
            return await retry(fn, *args, config=config, **kwargs)
        return wrapper
    return decorator


# ── Demo ──────────────────────────────────────────────────────────────────────
call_counter = 0

@with_retry(RetryConfig(max_attempts=4, base_delay=0.2, max_delay=5.0))
async def unstable_api(endpoint: str) -> str:
    global call_counter
    call_counter += 1
    if call_counter <= 3:
        raise ConnectionError(f"Connection refused (attempt {call_counter})")
    return f"200 OK from {endpoint}"


async def demo():
    global call_counter
    call_counter = 0

    print("=== Retry with exponential backoff + jitter ===\n")
    try:
        result = await unstable_api("/api/orders")
        print(f"\nFinal result: {result}")
    except ConnectionError as e:
        print(f"\nAll retries exhausted: {e}")

    # ── Retry budget demo ──
    print("\n=== Retry budget limiting ===")
    budget       = RetryBudget(max_retry_ratio=0.2, window=60)
    call_counter = 0

    async def always_fails():
        global call_counter
        call_counter += 1
        raise ConnectionError("Always down")

    for i in range(5):
        try:
            await retry(always_fails, config=RetryConfig(max_attempts=3, base_delay=0.01), budget=budget)
        except Exception as e:
            retries      = budget._retries
            total        = budget._total
            ratio        = retries / max(total, 1)
            print(f"  Call {i+1}: failed | retries={retries} total={total} ratio={ratio:.1%}")


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 3: Bulkhead — Semaphore-Based Concurrency Limit

```python
"""
Bulkhead pattern: limit concurrent calls to a downstream service.
Prevents one slow dependency from exhausting all available resources.

Two downstream services: payment (max 5 concurrent) and inventory (max 10 concurrent).
Demonstrates isolation: slow payment calls don't starve inventory calls.

pip install asyncio
"""
import asyncio
import time
import random
from dataclasses import dataclass
from typing import Callable, Any


class Bulkhead:
    """
    Semaphore-based bulkhead: limits concurrent calls.
    Requests exceeding max_concurrent are rejected immediately (no queuing).
    """
    def __init__(self, name: str, max_concurrent: int, timeout: float = 0.0):
        """
        name:           identifier for metrics/logging
        max_concurrent: max simultaneous calls
        timeout:        seconds to wait for a slot (0 = reject immediately if full)
        """
        self.name        = name
        self.max         = max_concurrent
        self._semaphore  = asyncio.Semaphore(max_concurrent)
        self._timeout    = timeout
        self._active     = 0
        self._rejected   = 0

    async def call(self, fn: Callable, *args, **kwargs) -> Any:
        try:
            acquired = await asyncio.wait_for(
                self._semaphore.acquire(),
                timeout=self._timeout if self._timeout > 0 else None
            )
        except asyncio.TimeoutError:
            self._rejected += 1
            raise BulkheadFullError(
                f"Bulkhead '{self.name}' full ({self.max} concurrent). Request rejected."
            )

        if not self._timeout:   # Immediate rejection path
            if not self._semaphore._value < self.max:  # Already released by wait_for
                pass

        self._active += 1
        try:
            return await fn(*args, **kwargs)
        finally:
            self._active -= 1
            self._semaphore.release()

    @property
    def metrics(self) -> dict:
        return {
            "name":         self.name,
            "max":          self.max,
            "available":    self._semaphore._value,
            "rejected":     self._rejected,
        }


class BulkheadFullError(Exception):
    pass


# ── Simulated downstream services ────────────────────────────────────────────
async def call_payment(amount: float) -> dict:
    """Simulate slow payment service (200ms–2s)."""
    delay = random.uniform(0.2, 2.0)
    await asyncio.sleep(delay)
    return {"status": "charged", "amount": amount}


async def call_inventory(product_id: str) -> dict:
    """Simulate fast inventory service (10–50ms)."""
    delay = random.uniform(0.01, 0.05)
    await asyncio.sleep(delay)
    return {"available": random.randint(0, 100), "product": product_id}


# ── Demo: without bulkhead, slow payment starves inventory ────────────────────
async def demo_without_bulkhead():
    print("=== WITHOUT Bulkhead: 20 payment + 20 inventory calls ===")
    start = time.time()

    payment_tasks   = [asyncio.create_task(call_payment(99.99))   for _ in range(20)]
    inventory_tasks = [asyncio.create_task(call_inventory("SKU-001")) for _ in range(20)]

    await asyncio.gather(*payment_tasks, *inventory_tasks)
    print(f"  All completed in {time.time() - start:.2f}s")


async def demo_with_bulkhead():
    print("\n=== WITH Bulkhead: payment (max 3) + inventory (max 10) ===")

    payment_bh   = Bulkhead("payment",   max_concurrent=3)
    inventory_bh = Bulkhead("inventory", max_concurrent=10)

    start = time.time()

    async def safe_payment(amount):
        try:
            return await payment_bh.call(call_payment, amount)
        except BulkheadFullError as e:
            return {"status": "rejected", "reason": str(e)}

    async def safe_inventory(pid):
        try:
            return await inventory_bh.call(call_inventory, pid)
        except BulkheadFullError as e:
            return {"status": "rejected", "reason": str(e)}

    payment_tasks   = [asyncio.create_task(safe_payment(99.99))     for _ in range(20)]
    inventory_tasks = [asyncio.create_task(safe_inventory("SKU-001")) for _ in range(20)]

    p_results = await asyncio.gather(*payment_tasks)
    i_results = await asyncio.gather(*inventory_tasks)

    elapsed = time.time() - start
    p_ok      = sum(1 for r in p_results if r.get("status") == "charged")
    p_rej     = sum(1 for r in p_results if r.get("status") == "rejected")
    i_ok      = sum(1 for r in i_results if r.get("status") != "rejected")

    print(f"  Payment:   {p_ok} OK, {p_rej} rejected (bulkhead protected) | {payment_bh.metrics}")
    print(f"  Inventory: {i_ok} OK  (unaffected by slow payments)         | {inventory_bh.metrics}")
    print(f"  Total time: {elapsed:.2f}s")


if __name__ == "__main__":
    asyncio.run(demo_without_bulkhead())
    asyncio.run(demo_with_bulkhead())
```

---

## Exercise 4: Composing All Patterns — Resilience Pipeline

```python
"""
A composable resilience pipeline that applies:
  timeout → circuit breaker → retry → bulkhead → fallback
in the correct order.

This is the pattern used by Resilience4j and Polly.

pip install asyncio
"""
import asyncio
import time
import random
from typing import Callable, Any, Optional


class ResiliencePipeline:
    """
    Composes resilience patterns in the correct order.
    Usage:
        pipeline = ResiliencePipeline("payment")
            .with_timeout(2.0)
            .with_circuit_breaker(failure_rate=0.5, window=60, min_calls=5, recovery=30)
            .with_retry(max_attempts=3, base_delay=0.5)
            .with_bulkhead(max_concurrent=10)
            .with_fallback(lambda: {"status": "cached"})
    """

    def __init__(self, name: str):
        self.name      = name
        self._timeout  = None
        self._cb       = None
        self._retry_cfg = None
        self._bulkhead  = None
        self._fallback  = None

    def with_timeout(self, seconds: float) -> "ResiliencePipeline":
        self._timeout = seconds
        return self

    def with_circuit_breaker(
        self,
        failure_rate:     float = 0.5,
        window:           float = 60.0,
        min_calls:        int   = 10,
        recovery_timeout: float = 30.0,
        success_threshold: int  = 3,
    ) -> "ResiliencePipeline":
        # Import CB from exercise 1 — inlined here for self-containment
        from collections import deque
        from enum import Enum

        class State(Enum):
            CLOSED = "closed"; OPEN = "open"; HALF_OPEN = "half_open"

        class CB:
            def __init__(self):
                self.state       = State.CLOSED
                self.window      = deque()
                self.opened_at   = 0.0
                self.ho_successes = 0
                self.lock        = asyncio.Lock()

            def prune(self):
                cut = time.monotonic() - window
                while self.window and self.window[0][0] < cut:
                    self.window.popleft()

            def rate(self):
                if not self.window: return 0.0
                return sum(1 for _, ok in self.window if not ok) / len(self.window)

            async def check_and_record(self, success: bool):
                async with self.lock:
                    self.window.append((time.monotonic(), success))
                    self.prune()
                    if self.state == State.HALF_OPEN:
                        if success:
                            self.ho_successes += 1
                            if self.ho_successes >= success_threshold:
                                self.state = State.CLOSED
                                self.window.clear()
                                print(f"[CB:{self.name}] HALF_OPEN → CLOSED")
                        else:
                            self.opened_at = time.monotonic()
                            self.state = State.OPEN
                            print(f"[CB:{self.name}] HALF_OPEN → OPEN (probe failed)")
                        return
                    if self.state == State.CLOSED:
                        if len(self.window) >= min_calls and self.rate() >= failure_rate:
                            self.opened_at = time.monotonic()
                            self.state = State.OPEN
                            print(f"[CB:{self.name}] CLOSED → OPEN (rate={self.rate():.0%})")

            def is_open(self) -> bool:
                if self.state == State.OPEN:
                    if time.monotonic() - self.opened_at >= recovery_timeout:
                        self.state = State.HALF_OPEN
                        self.ho_successes = 0
                        print(f"[CB:{self.name}] OPEN → HALF_OPEN")
                        return False
                    return True
                return False

        cb = CB()
        cb.name = self.name   # Attach name for logging
        self._cb = cb
        return self

    def with_retry(self, max_attempts: int = 3, base_delay: float = 0.5) -> "ResiliencePipeline":
        self._retry_cfg = (max_attempts, base_delay)
        return self

    def with_bulkhead(self, max_concurrent: int = 10) -> "ResiliencePipeline":
        self._bulkhead = asyncio.Semaphore(max_concurrent)
        self._bulkhead_max = max_concurrent
        return self

    def with_fallback(self, fb: Callable) -> "ResiliencePipeline":
        self._fallback = fb
        return self

    async def execute(self, fn: Callable, *args, **kwargs) -> Any:
        # ── 1. Bulkhead ────────────────────────────────────────────────────
        if self._bulkhead and self._bulkhead.locked():
            # Non-blocking check
            acquired = self._bulkhead._value > 0
            if not acquired:
                if self._fallback:
                    print(f"[{self.name}] Bulkhead full → fallback")
                    return self._fallback() if callable(self._fallback) else self._fallback
                raise RuntimeError(f"[{self.name}] Bulkhead full")

        semaphore_ctx = self._bulkhead if self._bulkhead else asyncio.Semaphore(999999)

        async with semaphore_ctx:
            return await self._run_with_cb(fn, *args, **kwargs)

    async def _run_with_cb(self, fn, *args, **kwargs):
        # ── 2. Circuit breaker ─────────────────────────────────────────────
        if self._cb and self._cb.is_open():
            if self._fallback:
                print(f"[{self.name}] Circuit OPEN → fallback")
                return self._fallback() if callable(self._fallback) else self._fallback
            raise RuntimeError(f"[{self.name}] Circuit OPEN")

        # ── 3. Retry + timeout ─────────────────────────────────────────────
        max_a, base = self._retry_cfg or (1, 0)
        last_exc    = None

        for attempt in range(max_a):
            try:
                if self._timeout:
                    result = await asyncio.wait_for(fn(*args, **kwargs), self._timeout)
                else:
                    result = await fn(*args, **kwargs)

                if self._cb:
                    await self._cb.check_and_record(True)
                return result

            except Exception as e:
                last_exc = e
                if self._cb:
                    await self._cb.check_and_record(False)
                if attempt < max_a - 1:
                    delay = min(base * (2 ** attempt) * (0.5 + random.random()), 30)
                    print(f"[{self.name}] Attempt {attempt+1} failed ({type(e).__name__}), retry in {delay:.2f}s")
                    await asyncio.sleep(delay)

        # All retries exhausted
        if self._fallback:
            print(f"[{self.name}] All retries failed → fallback")
            return self._fallback() if callable(self._fallback) else self._fallback
        raise last_exc


# ── Demo ──────────────────────────────────────────────────────────────────────
attempt_n = 0

async def payment_service(amount: float) -> dict:
    global attempt_n
    attempt_n += 1
    # Simulate intermittent failures
    if 3 <= attempt_n <= 12:
        await asyncio.sleep(0.01)
        raise ConnectionError(f"Payment service down (attempt {attempt_n})")
    return {"status": "charged", "amount": amount}


async def demo():
    global attempt_n

    pipeline = (
        ResiliencePipeline("payment")
        .with_timeout(2.0)
        .with_circuit_breaker(failure_rate=0.5, window=60, min_calls=4, recovery_timeout=2.0)
        .with_retry(max_attempts=2, base_delay=0.1)
        .with_bulkhead(max_concurrent=5)
        .with_fallback(lambda: {"status": "queued_for_retry", "cached": True})
    )

    print("=== Resilience Pipeline Demo ===\n")
    for i in range(20):
        if i == 14:
            print(f"\n[Demo] Sleeping 2s for circuit recovery...\n")
            await asyncio.sleep(2.1)

        result = await pipeline.execute(payment_service, amount=99.99 * (i + 1))
        print(f"  [{i+1:2d}] {result}")


if __name__ == "__main__":
    asyncio.run(demo())
```

---

## Exercise 5: Timeout + Deadline Propagation

```python
"""
Demonstrates timeout calibration and deadline propagation across service calls.
At each hop, the remaining deadline is passed forward and respected.

pip install asyncio httpx
"""
import asyncio
import time


class Deadline:
    """
    Propagatable deadline. Passed through service call chains.
    Each service checks remaining time before making downstream calls.
    """
    def __init__(self, total_seconds: float):
        self._deadline = time.monotonic() + total_seconds

    @property
    def remaining(self) -> float:
        return max(0.0, self._deadline - time.monotonic())

    @property
    def expired(self) -> bool:
        return time.monotonic() >= self._deadline

    def child(self, reserve: float = 0.05) -> "Deadline":
        """Create a child deadline with a small overhead reserved for bookkeeping."""
        return Deadline(max(0.0, self.remaining - reserve))

    async def run(self, coro):
        """Execute coro with this deadline as a timeout."""
        r = self.remaining
        if r <= 0:
            raise asyncio.TimeoutError("Deadline already expired")
        return await asyncio.wait_for(coro, timeout=r)


# ── Service call chain: API → Service A → Service B → DB ─────────────────────

async def query_db(deadline: Deadline) -> list:
    if deadline.expired:
        raise asyncio.TimeoutError("Deadline expired before DB query")
    print(f"    [DB]        remaining={deadline.remaining:.3f}s → querying...")
    await asyncio.sleep(0.05)  # Simulate 50ms DB query
    return [{"id": 1}, {"id": 2}]


async def service_b(deadline: Deadline) -> dict:
    print(f"  [Service B]  remaining={deadline.remaining:.3f}s")
    # Reserve 20ms for overhead; give rest to DB
    db_deadline = deadline.child(reserve=0.02)
    rows = await db_deadline.run(query_db(db_deadline))
    return {"rows": rows, "count": len(rows)}


async def service_a(deadline: Deadline) -> dict:
    print(f"[Service A]   remaining={deadline.remaining:.3f}s")
    # Reserve 30ms for local processing; give rest to Service B
    b_deadline = deadline.child(reserve=0.03)
    result = await b_deadline.run(service_b(b_deadline))
    return {"from_a": True, **result}


async def demo():
    print("=== Deadline Propagation Demo ===\n")

    # Scenario 1: Generous deadline — all calls succeed
    print("--- Scenario 1: 500ms total deadline ---")
    deadline = Deadline(0.5)
    try:
        result = await deadline.run(service_a(deadline.child(0.01)))
        print(f"  Result: {result}\n")
    except asyncio.TimeoutError as e:
        print(f"  TIMEOUT: {e}\n")

    # Scenario 2: Tight deadline — times out in DB
    print("--- Scenario 2: 60ms total deadline (DB takes 50ms, overhead is 50ms) ---")
    deadline = Deadline(0.06)
    try:
        result = await deadline.run(service_a(deadline.child(0.01)))
        print(f"  Result: {result}\n")
    except asyncio.TimeoutError as e:
        print(f"  TIMEOUT at DB level (deadline expired) — upstream services cancelled ✓\n")

    # Scenario 3: Calibration helper
    print("--- Scenario 3: Timeout calibration based on P99 latency ---")
    latencies = []
    for _ in range(20):
        start = time.monotonic()
        await asyncio.sleep(0.05 + 0.03 * (asyncio.get_event_loop().time() % 1))
        latencies.append(time.monotonic() - start)

    latencies.sort()
    p50  = latencies[int(0.50 * len(latencies))]
    p99  = latencies[int(0.99 * len(latencies))]
    rec  = p99 * 3
    print(f"  P50={p50*1000:.0f}ms  P99={p99*1000:.0f}ms  → Recommended timeout={rec*1000:.0f}ms (3×P99)")
    print(f"  Never set timeout = P50 ({p50*1000:.0f}ms) — that rejects half of normal traffic!")


if __name__ == "__main__":
    asyncio.run(demo())
```
