# Routing Logic

## Overview

Route selection is the core competitive advantage. The routing engine decides which upstream connector handles each message, based on message class, cost, quality, and connector health.

**Key principle:** Design the routing engine as a pluggable strategy from day one. Even in Phase 1 with simple priority routing, the interface should support swapping strategies without changing business logic.

---

## Route Selection Interface

```python
from typing import Protocol
from decimal import Decimal
from uuid import UUID

class RouteStrategy(Protocol):
    def select(self, message: MessageRequest, routes: list[Route]) -> Route | None:
        """Select best route. Returns None if no route available."""
        ...

class RouteSelector:
    """Main entry point. Delegates to strategy based on message class."""

    def __init__(self):
        self.strategies: dict[str, RouteStrategy] = {
            "otp": FailoverStrategy(),
            "transactional": QualityFirstStrategy(),
            "promotional": LeastCostStrategy(),
        }
        self.default = PriorityStrategy()

    def select_route(self, message: MessageRequest) -> Route | None:
        # 1. Get active routes for this message class + country
        routes = self._get_eligible_routes(message)

        # 2. Filter out open circuits
        routes = [r for r in routes if r.circuit_state != "open"]

        if not routes:
            return None

        # 3. Delegate to strategy
        strategy = self.strategies.get(message.message_class, self.default)
        return strategy.select(message, routes)

    def _get_eligible_routes(self, message):
        """Fetch routes matching message_class and country_code."""
        ...
```

---

## Strategies

### Phase 1: Priority-Based (MVP)

Simplest. Each route has a priority number (lower = higher). Pick highest-priority active route.

```python
class PriorityStrategy:
    def select(self, message, routes):
        if not routes:
            return None
        return min(routes, key=lambda r: r.priority)
```

**When:** Phase 1. Simple, predictable.
**Limitation:** All traffic on one route. No load distribution.

### Phase 2: Quality-First (for OTP/Transactional)

Pick the route with the highest delivery rate.

```python
class QualityFirstStrategy:
    def select(self, message, routes):
        if not routes:
            return None
        return max(routes, key=lambda r: r.health_score)
```

### Phase 2: Least-Cost (for Promotional)

Pick cheapest route above minimum quality threshold.

```python
class LeastCostStrategy:
    def __init__(self, min_health: float = 0.8):
        self.min_health = min_health

    def select(self, message, routes):
        eligible = [r for r in routes if r.health_score >= self.min_health]
        if not eligible:
            # Fall back to any route if none meets quality threshold
            eligible = routes
        return min(eligible, key=lambda r: r.cost_per_unit)
```

### Phase 2: Failover Chain (for Critical Messages)

Ordered fallback: try primary, if unavailable try next.

```python
class FailoverStrategy:
    def select(self, message, routes):
        for route in sorted(routes, key=lambda r: r.priority):
            if route.circuit_state != "open":
                return route
        return None
```

### Phase 3: Weighted Round-Robin (for Load Balancing)

Distribute traffic by weight across comparable routes.

```python
class WeightedRoundRobinStrategy:
    def __init__(self):
        self._counters: dict[str, int] = {}

    def select(self, message, routes):
        if not routes:
            return None
        pool = []
        for route in routes:
            pool.extend([route] * route.weight)
        key = f"{message.message_class}:{message.country_code}"
        idx = self._counters.get(key, 0) % len(pool)
        self._counters[key] = idx + 1
        return pool[idx]
```

---

## Circuit Breaker

Prevents sending to a failing connector. Shared state in Redis.

### States

```
CLOSED (normal)
  │ failures >= threshold
  ▼
OPEN (blocking all traffic)
  │ recovery_timeout expires
  ▼
HALF_OPEN (allow test messages)
  │ success → CLOSED
  │ failure → OPEN (reset timer)
```

### Implementation

```python
import time
import redis

class CircuitBreaker:
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

    def __init__(self, redis_client: redis.Redis,
                 failure_threshold: int = 5,
                 recovery_timeout: int = 60):
        self.redis = redis_client
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout

    def _key(self, connector_id: str) -> str:
        return f"circuit:{connector_id}"

    def get_state(self, connector_id: str) -> str:
        data = self.redis.hgetall(self._key(connector_id))
        if not data:
            return self.CLOSED

        state = data.get(b"state", b"closed").decode()
        if state == self.OPEN:
            last_failure = float(data.get(b"last_failure", 0))
            if time.time() - last_failure > self.recovery_timeout:
                return self.HALF_OPEN
        return state

    def record_failure(self, connector_id: str):
        key = self._key(connector_id)
        pipe = self.redis.pipeline()
        pipe.hincrby(key, "failures", 1)
        pipe.hset(key, "last_failure", str(time.time()))
        pipe.execute()

        failures = int(self.redis.hget(key, "failures") or 0)
        if failures >= self.failure_threshold:
            self.redis.hset(key, "state", self.OPEN)
            # TODO: Fire alert

    def record_success(self, connector_id: str):
        key = self._key(connector_id)
        self.redis.hmset(key, {"state": self.CLOSED, "failures": 0})

    def can_send(self, connector_id: str) -> bool:
        state = self.get_state(connector_id)
        return state != self.OPEN
```

### Configuration

| Parameter | Default | Notes |
|---|---|---|
| failure_threshold | 5 | Consecutive failures before OPEN |
| recovery_timeout | 60s | Wait before testing HALF_OPEN |
| half_open_tests | 3 | Messages to test before closing |
| success_threshold | 2/3 | Tests that must succeed to close |

### What Counts as Failure
- SMPP timeout (no submit_sm_resp)
- SMPP system error (0x00000008)
- Connection lost during submission

### What Does NOT Count as Failure
- Throttling (0x00000058) — expected under load
- Invalid number — client's problem, not route's
- DLR failure — may be handset issue, not route issue

---

## Route Health Scoring

Health score (0.0 to 1.0) computed from real delivery data over a rolling window.

### Formula

```
health_score = (0.6 × delivery_rate) + (0.2 × latency_score) + (0.2 × error_score)

delivery_rate = delivered / total_submitted  (last 1 hour)
latency_score = 1.0 - (avg_latency / max_acceptable_latency)  (capped at 0)
error_score   = 1.0 - (error_count / total_submitted)
```

### Implementation

```python
async def calculate_health_score(connector_id: str, window_hours: int = 1) -> float:
    stats = await get_connector_stats(connector_id, window_hours)

    if stats.total_submitted == 0:
        return 0.5  # Cold start — unknown health

    delivery_rate = stats.delivered / stats.total_submitted
    latency_score = max(0, 1.0 - (stats.avg_latency_seconds / 30.0))
    error_score = 1.0 - (stats.errors / stats.total_submitted)

    return (0.6 * delivery_rate) + (0.2 * latency_score) + (0.2 * error_score)
```

### Refresh
- Recalculate every 5 minutes
- Cache in Redis for fast access during route selection
- Persist to DB hourly for historical analysis

### Cold Start
- New routes: default health_score = 0.5
- Send canary traffic (5% of messages) to build data
- Promote to full traffic after 100+ messages

---

## Route Decision Flow (Complete)

```
1. Receive message request
2. Filter routes by message_class + country_code + is_active
3. Filter out routes with open circuit breakers
4. Check DLT: sender_id type matches message_class
5. Select strategy based on message_class:
   - otp → FailoverStrategy
   - transactional → QualityFirstStrategy
   - promotional → LeastCostStrategy
6. Strategy returns selected route (or None)
7. If None → return ROUTE_UNAVAILABLE error
8. Snapshot route config into message.route_snapshot
9. Look up rate_card for org + route + message_class
10. Proceed to billing (wallet debit)
```

---

## Anti-Patterns

- **Don't hardcode route selection** — Use strategy pattern from day one
- **Don't route without health data** — Even basic delivery rate beats blind routing
- **Don't ignore circuit breaker** — One bad route tanks overall delivery rate
- **Don't optimize cost before reliability** — Priority first, then cost optimization
- **Don't cache route decisions** — Select fresh per message (route health changes fast)
- **Don't count throttling as failure** — Throttle = expected, failure = broken
- **Don't skip route snapshot** — Record which route was used at submission time for audit
