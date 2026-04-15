# Observability — Hands-On Exercises

## Exercise 1: Structured Logging with Correlation IDs

```python
import logging
import json
import uuid
import time
from contextvars import ContextVar
from functools import wraps
from typing import Any

# Thread/coroutine-safe context variable for request context
_request_ctx: ContextVar[dict] = ContextVar("request_ctx", default={})


class StructuredLogger:
    def __init__(self, service_name: str):
        self.service = service_name
        self._logger = logging.getLogger(service_name)
        handler = logging.StreamHandler()
        handler.setFormatter(logging.Formatter("%(message)s"))
        self._logger.addHandler(handler)
        self._logger.setLevel(logging.DEBUG)

    def _build_record(self, level: str, event: str, **kwargs) -> str:
        ctx = _request_ctx.get({})
        record = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "level": level,
            "service": self.service,
            "event": event,
            **ctx,
            **kwargs,
        }
        return json.dumps(record)

    def info(self, event: str, **kwargs):
        self._logger.info(self._build_record("INFO", event, **kwargs))

    def warning(self, event: str, **kwargs):
        self._logger.warning(self._build_record("WARNING", event, **kwargs))

    def error(self, event: str, **kwargs):
        self._logger.error(self._build_record("ERROR", event, **kwargs))

    def debug(self, event: str, **kwargs):
        self._logger.debug(self._build_record("DEBUG", event, **kwargs))


log = StructuredLogger("payment-service")


def with_request_context(func):
    """Decorator: injects trace_id and request_id into log context for entire request."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        token = _request_ctx.set({
            "trace_id": str(uuid.uuid4())[:8],
            "request_id": str(uuid.uuid4())[:8],
        })
        try:
            return func(*args, **kwargs)
        finally:
            _request_ctx.reset(token)
    return wrapper


@with_request_context
def process_payment(user_id: int, amount: float, currency: str):
    log.info("payment_started", user_id=user_id, amount=amount, currency=currency)

    # Simulate DB lookup
    start = time.time()
    time.sleep(0.01)
    db_latency = (time.time() - start) * 1000
    log.debug("db_user_fetched", user_id=user_id, latency_ms=round(db_latency, 2))

    # Simulate occasional failure
    import random
    if random.random() < 0.3:
        log.error("payment_failed",
                  user_id=user_id,
                  amount=amount,
                  error="insufficient_funds",
                  error_code="PAYMENT_001")
        return False

    log.info("payment_succeeded", user_id=user_id, amount=amount, currency=currency)
    return True


if __name__ == "__main__":
    for i in range(5):
        process_payment(user_id=100 + i, amount=49.99, currency="USD")
```

---

## Exercise 2: Prometheus-Style Metrics Collector

```python
import time
import math
from collections import defaultdict
from dataclasses import dataclass, field


# ─── Metric Types ───────────────────────────────────────────────

class Counter:
    """Monotonically increasing value."""
    def __init__(self, name: str, help: str, labels: list[str] = ()):
        self.name = name
        self.help = help
        self._values: dict[tuple, float] = defaultdict(float)

    def inc(self, amount: float = 1, **label_values):
        key = tuple(sorted(label_values.items()))
        self._values[key] += amount

    def collect(self) -> list[tuple]:
        return list(self._values.items())


class Gauge:
    """Current value that can go up or down."""
    def __init__(self, name: str, help: str):
        self.name = name
        self.help = help
        self._values: dict[tuple, float] = defaultdict(float)

    def set(self, value: float, **label_values):
        key = tuple(sorted(label_values.items()))
        self._values[key] = value

    def inc(self, amount: float = 1, **label_values):
        key = tuple(sorted(label_values.items()))
        self._values[key] += amount

    def dec(self, amount: float = 1, **label_values):
        key = tuple(sorted(label_values.items()))
        self._values[key] -= amount

    def collect(self) -> list[tuple]:
        return list(self._values.items())


class Histogram:
    """Distribution of values — stores in configurable buckets."""
    DEFAULT_BUCKETS = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]

    def __init__(self, name: str, help: str, buckets: list[float] = None):
        self.name = name
        self.help = help
        self.buckets = sorted(buckets or self.DEFAULT_BUCKETS) + [math.inf]
        self._observations: list[float] = []

    def observe(self, value: float):
        self._observations.append(value)

    def percentile(self, p: float) -> float:
        if not self._observations:
            return 0.0
        sorted_obs = sorted(self._observations)
        idx = int(p / 100 * len(sorted_obs))
        return sorted_obs[min(idx, len(sorted_obs) - 1)]

    def bucket_counts(self) -> dict[float, int]:
        counts = {}
        for b in self.buckets:
            counts[b] = sum(1 for o in self._observations if o <= b)
        return counts

    def summary(self) -> dict:
        if not self._observations:
            return {}
        return {
            "count": len(self._observations),
            "sum": sum(self._observations),
            "p50": self.percentile(50),
            "p95": self.percentile(95),
            "p99": self.percentile(99),
            "p999": self.percentile(99.9),
        }


# ─── Demo ─────────────────────────────────────────────────────

import random

http_requests_total = Counter("http_requests_total", "Total HTTP requests")
active_connections  = Gauge("active_connections", "Current open DB connections")
request_duration    = Histogram("http_request_duration_seconds", "Request latency",
                                 buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.0])


def simulate_traffic(n_requests: int = 1000):
    endpoints = ["/api/search", "/api/checkout", "/api/users", "/health"]
    statuses  = ["200", "200", "200", "200", "429", "500"]  # Weighted: mostly 200s

    for _ in range(n_requests):
        endpoint = random.choice(endpoints)
        status = random.choice(statuses)
        # Latency: /checkout is slow, /health is fast
        base_latency = 0.5 if "checkout" in endpoint else 0.05
        latency = random.lognormvariate(math.log(base_latency), 0.5)

        http_requests_total.inc(endpoint=endpoint, status=status)
        request_duration.observe(latency)

    active_connections.set(random.randint(5, 50))


def print_metrics():
    print("\n=== Simulated Metrics Report ===\n")

    print("─ http_requests_total:")
    for labels, count in sorted(http_requests_total.collect(), key=lambda x: -x[1]):
        print(f"  {dict(labels)} → {count}")

    print(f"\n─ active_connections: {list(active_connections.collect())}")

    print("\n─ http_request_duration_seconds:")
    s = request_duration.summary()
    for k, v in s.items():
        if k not in ("count", "sum"):
            print(f"  {k:6s}: {v*1000:7.1f}ms")
    print(f"  count : {s['count']:,}")


if __name__ == "__main__":
    simulate_traffic(2000)
    print_metrics()
```

---

## Exercise 3: Simple Distributed Trace Builder

```python
import time
import uuid
from contextlib import contextmanager
from dataclasses import dataclass, field


@dataclass
class Span:
    name: str
    trace_id: str
    span_id: str = field(default_factory=lambda: uuid.uuid4().hex[:8])
    parent_id: str | None = None
    start_time: float = field(default_factory=time.time)
    end_time: float | None = None
    tags: dict = field(default_factory=dict)
    status: str = "ok"

    def finish(self, status: str = "ok"):
        self.end_time = time.time()
        self.status = status

    @property
    def duration_ms(self) -> float:
        end = self.end_time or time.time()
        return (end - self.start_time) * 1000


class Tracer:
    def __init__(self):
        self._spans: list[Span] = []
        self._active: list[Span] = []   # Stack of active spans

    def start_trace(self, name: str) -> Span:
        trace_id = uuid.uuid4().hex[:8]
        span = Span(name=name, trace_id=trace_id)
        self._spans.append(span)
        self._active.append(span)
        return span

    @contextmanager
    def span(self, name: str, **tags):
        parent = self._active[-1] if self._active else None
        trace_id = parent.trace_id if parent else uuid.uuid4().hex[:8]
        s = Span(
            name=name,
            trace_id=trace_id,
            parent_id=parent.span_id if parent else None,
            tags=tags,
        )
        self._spans.append(s)
        self._active.append(s)
        try:
            yield s
            s.finish("ok")
        except Exception as e:
            s.finish("error")
            s.tags["error"] = str(e)
            raise
        finally:
            self._active.pop()

    def print_trace(self):
        if not self._spans:
            return
        trace_id = self._spans[0].trace_id
        print(f"\n[Trace {trace_id}]")
        print(f"{'Span Name':30s}  {'Duration':>10}  {'Status':8}  {'Tags'}")
        print("-" * 70)
        for span in self._spans:
            indent = "  " if span.parent_id else ""
            indent2 = "    " if span.parent_id and any(
                s.span_id == span.parent_id and s.parent_id for s in self._spans
            ) else indent
            duration = f"{span.duration_ms:.1f}ms"
            tags_str = str(span.tags) if span.tags else ""
            print(f"{indent2}{span.name:30s}  {duration:>10}  {span.status:8}  {tags_str}")


tracer = Tracer()


def handle_request(user_id: int):
    with tracer.span("handle_request", user_id=user_id) as root:
        time.sleep(0.005)  # Parsing

        with tracer.span("auth.validate_token"):
            time.sleep(0.010)

        with tracer.span("user.get_profile", db="postgres") as span:
            time.sleep(0.080)  # Slow DB call
            span.tags["rows_returned"] = 1

        with tracer.span("cache.get_feed", cache="redis"):
            time.sleep(0.003)

        with tracer.span("response.serialize"):
            time.sleep(0.002)

    tracer.print_trace()


if __name__ == "__main__":
    handle_request(user_id=42)
```

**Expected output:**
```
[Trace a1b2c3d4]
Span Name                       Duration    Status    Tags
─────────────────────────────────────────────────────────────
handle_request                   102.3ms  ok        {'user_id': 42}
  auth.validate_token             10.2ms  ok        {}
  user.get_profile                80.5ms  ok        {'db': 'postgres', 'rows_returned': 1}
  cache.get_feed                   3.1ms  ok        {'cache': 'redis'}
  response.serialize               2.0ms  ok        {}
```
