# SMS Routing Strategies

Reusable routing patterns for any SMS platform. Not project-specific.

## Why Routing Matters

Route selection is the core competitive advantage of an SMS aggregator. The right route means:
- Higher delivery rates
- Lower cost per message
- Faster delivery
- Better failover

A system with 3 dumb routes is worse than 2 smart ones.

---

## Strategy Pattern for Route Selection

Every routing decision should go through a pluggable strategy. This lets you swap algorithms without touching business logic.

```python
from typing import Protocol

class RouteStrategy(Protocol):
    def select(self, message: Message, routes: list[Route]) -> Route | None:
        """Select best route for this message. Returns None if no route available."""
        ...
```

---

## Common Strategies

### 1. Priority-Based (Start Here)

Simplest strategy. Each route has a priority number (lower = higher priority). Pick the highest-priority active route.

```python
class PriorityStrategy:
    def select(self, message, routes):
        eligible = [r for r in routes
                    if r.is_active
                    and r.circuit_state != "open"
                    and r.message_class == message.message_class]
        if not eligible:
            return None
        return min(eligible, key=lambda r: r.priority)
```

**When to use:** Phase 1. Simple, predictable, easy to debug.

**Limitation:** All traffic goes to one route. No load distribution.

### 2. Weighted Round-Robin

Distribute traffic across routes by weight. Route A (weight 3) gets 3x the traffic of Route B (weight 1).

```python
class WeightedRoundRobinStrategy:
    def __init__(self):
        self._counters: dict[str, int] = {}

    def select(self, message, routes):
        eligible = [r for r in routes if r.is_active and r.circuit_state != "open"]
        if not eligible:
            return None

        # Build weighted pool
        pool = []
        for route in eligible:
            pool.extend([route] * route.weight)

        # Round-robin through pool
        key = message.message_class
        idx = self._counters.get(key, 0) % len(pool)
        self._counters[key] = idx + 1
        return pool[idx]
```

**When to use:** Load balancing across comparable routes.

**Limitation:** Doesn't account for cost or quality differences.

### 3. Least-Cost

Pick the cheapest route that meets a minimum quality threshold.

```python
class LeastCostStrategy:
    def __init__(self, min_health_score: float = 0.8):
        self.min_health_score = min_health_score

    def select(self, message, routes):
        eligible = [r for r in routes
                    if r.is_active
                    and r.circuit_state != "open"
                    and r.health_score >= self.min_health_score]
        if not eligible:
            return None
        return min(eligible, key=lambda r: r.cost_per_unit)
```

**When to use:** When margin optimization matters and you have reliable health scoring.

**Limitation:** Can over-concentrate on one cheap route.

### 4. Quality-First

Pick the route with the highest delivery rate, regardless of cost.

```python
class QualityFirstStrategy:
    def select(self, message, routes):
        eligible = [r for r in routes if r.is_active and r.circuit_state != "open"]
        if not eligible:
            return None
        return max(eligible, key=lambda r: r.health_score)
```

**When to use:** OTP and transactional messages where delivery matters more than cost.

### 5. Failover Chain

Ordered fallback: try primary, if circuit open try secondary, then tertiary.

```python
class FailoverStrategy:
    def select(self, message, routes):
        # Routes must be pre-sorted by failover priority
        for route in sorted(routes, key=lambda r: r.priority):
            if route.is_active and route.circuit_state != "open":
                return route
        return None
```

**When to use:** Critical message classes (OTP) where you always want a working route.

### 6. Hybrid (Production-Grade)

Combine multiple strategies based on message class:

```python
class HybridStrategy:
    def __init__(self):
        self.strategies = {
            "otp": FailoverStrategy(),         # Reliability first
            "transactional": QualityFirstStrategy(),  # Quality first
            "promotional": LeastCostStrategy(), # Cost first
        }
        self.default = PriorityStrategy()

    def select(self, message, routes):
        strategy = self.strategies.get(message.message_class, self.default)
        filtered = [r for r in routes if r.message_class == message.message_class]
        return strategy.select(message, filtered)
```

---

## Route Health Scoring

Health score (0.0 to 1.0) should be computed from real delivery data:

```
health_score = (delivered_count / total_submitted) over rolling window
```

### Factors to include
- **Delivery rate** — Primary signal. Weight: 0.6
- **Latency** — Average submit-to-DLR time. Weight: 0.2
- **Error rate** — Non-delivery errors (throttling, system errors). Weight: 0.2

### Refresh frequency
- Recalculate every 5 minutes from last 1-hour window
- Store in Redis for fast access during route selection
- Persist to DB periodically for historical analysis

### Cold start problem
- New routes have no data → default health_score = 0.5
- Send small percentage of traffic (canary) to build data
- Promote to full traffic after sufficient sample size (e.g., 100 messages)

---

## Circuit Breaker Pattern

Prevents sending to a failing route, gives it time to recover.

### States

```
CLOSED (normal) → failures exceed threshold → OPEN (blocking)
OPEN → recovery timeout expires → HALF_OPEN (testing)
HALF_OPEN → test message succeeds → CLOSED
HALF_OPEN → test message fails → OPEN (reset timer)
```

### Configuration

| Parameter | Recommended Default | Notes |
|---|---|---|
| failure_threshold | 5 consecutive failures | Before opening circuit |
| recovery_timeout | 60 seconds | Before attempting half-open test |
| half_open_max_tests | 3 | Messages to test before closing |
| success_threshold | 2 of 3 | Tests that must succeed to close |

### Implementation Notes
- Store circuit state in Redis (shared across workers)
- Log every state transition for debugging
- Alert on OPEN transitions
- Don't count throttling errors as failures (those are expected under load)

---

## Operator-Based Routing

In India, messages route through specific operators. Advanced routing considers:

| Factor | Impact |
|---|---|
| Destination operator | Some vendors have better rates/quality for specific telcos |
| Number series (MNP) | Number may have ported — HLR lookup needed for accuracy |
| Region | Some vendors have regional coverage gaps |
| Time of day | Promotional messages restricted to 9AM-9PM IST |

### MNP (Mobile Number Portability)
- Phone prefix doesn't guarantee current operator
- HLR (Home Location Register) lookup reveals actual operator
- Cache HLR results (operator rarely changes) — TTL: 30 days
- Use for routing decisions on high-value messages

---

## Anti-Patterns

- **Don't hardcode route selection** — Always use strategy pattern, even in Phase 1
- **Don't route without health data** — Even basic delivery rate tracking beats blind routing
- **Don't ignore circuit breaker** — One bad route can tank your entire delivery rate
- **Don't optimize cost before reliability** — First make it work (priority), then make it cheap (least-cost)
- **Don't skip the hybrid approach** — OTP and promotional have fundamentally different routing needs
- **Don't cache route decisions** — Select fresh for every message (routes change fast)
