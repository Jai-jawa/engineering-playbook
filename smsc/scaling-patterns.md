# SMS Platform Scaling Patterns

Reusable scaling strategies for SMS aggregation platforms.

## Scaling Philosophy

Build with Tier 3 shape from day one, run at Tier 2 cost. Every component should be independently scalable, but don't split until traffic demands it.

---

## Phase-Based Scaling

### Phase 1: Single Machine (0-50 TPS)

```
One VPS → Docker Compose → All services
```

- All services on one machine
- PostgreSQL, Redis, RabbitMQ as Docker containers
- Suitable for <10 clients, <50 TPS
- Cost: $50-100/month

**Scaling trigger:** CPU consistently >70%, or queue depth growing faster than drain rate.

### Phase 2: Separated Services (50-200 TPS)

```
App Server ──── DB Server ──── Queue Server
```

- Database on dedicated instance (or managed service)
- Jasmin + RabbitMQ on separate server (SMPP stability)
- API and workers can scale independently
- Cost: $200-500/month

**Scaling trigger:** Single worker can't keep up with dispatch queue, or need read replicas.

### Phase 3: Horizontal Scale (200-1000 TPS)

```
LB → API x3 → Workers x5 → Jasmin x2 → PG Primary + Replica
```

- Load balancer in front of API instances
- Workers auto-scale based on queue depth
- PostgreSQL primary/replica (writes to primary, reads from replica)
- Redis cluster for session/cache
- Multiple Jasmin instances for redundancy
- Cost: $500+/month

---

## Component-Specific Scaling

### API Server (FastAPI)

| Bottleneck | Solution |
|---|---|
| CPU | Add instances behind load balancer |
| DB connections | Connection pooling (PgBouncer) |
| Auth overhead | Cache JWT validation results in Redis (short TTL) |

**Stateless by design:** No session state in API. All state in DB/Redis. Any instance can handle any request.

### Workers (Dispatch + DLR)

| Bottleneck | Solution |
|---|---|
| Queue growing | Add worker instances (horizontal) |
| DB write contention | Batch DLR updates, use COPY for bulk |
| Webhook delivery slow | Separate webhook workers from dispatch workers |

**Auto-scaling rule:** Scale workers when `queue_depth / drain_rate > 30 seconds`.

### PostgreSQL

| Bottleneck | Solution |
|---|---|
| Read load | Read replicas for reports/dashboards |
| Write load | Table partitioning (messages by month) |
| Connection exhaustion | PgBouncer in front of PostgreSQL |
| Large table scans | Proper indexing, partition pruning |

**Partitioning strategy:**
```sql
-- Monthly partitions for messages table
-- Auto-create next month's partition via cron job
-- Detach and archive partitions older than retention period
```

### Redis

| Bottleneck | Solution |
|---|---|
| Memory | Eviction policies, separate instances per concern |
| Throughput | Redis Cluster (sharding) |

**Separation of concerns:** Consider separate Redis instances for:
- Rate limiting (can tolerate data loss)
- Session cache (can tolerate data loss)
- DND cache (rebuild from source daily)
- Circuit breaker state (critical, small data)

### RabbitMQ

| Bottleneck | Solution |
|---|---|
| Single queue | Separate queues per message priority |
| Throughput | Cluster with mirrored queues |
| Message loss risk | Publisher confirms + consumer acks |

**Queue topology:**
```
dispatch.otp       → Priority queue, dedicated workers
dispatch.transactional → Standard queue
dispatch.promotional   → Can tolerate slight delay
dlr.processing     → High throughput, batch-friendly
webhook.delivery   → Separate from main dispatch path
```

### SMPP (Jasmin)

| Bottleneck | Solution |
|---|---|
| TPS limit per bind | Open multiple binds (negotiate with vendor) |
| Single point of failure | Multiple Jasmin instances, sticky routing per connector |
| Connection instability | Health monitoring, auto-reconnect |

---

## Database Scaling Patterns

### Partitioning (Critical for Messages)

```sql
-- Create partitions automatically
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    next_month DATE := date_trunc('month', now() + interval '1 month');
    partition_name TEXT := 'messages_' || to_char(next_month, 'YYYY_MM');
    start_date TEXT := to_char(next_month, 'YYYY-MM-DD');
    end_date TEXT := to_char(next_month + interval '1 month', 'YYYY-MM-DD');
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF messages FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
END;
$$ LANGUAGE plpgsql;
```

Run via cron: create next month's partition on the 25th of each month.

### Read Replica Usage

| Query Type | Target |
|---|---|
| Message submission | Primary (write) |
| Wallet debit | Primary (write, with row lock) |
| Dashboard stats | Replica (read) |
| Message reports | Replica (read) |
| DLR status update | Primary (write) |
| Financial reports | Replica (read) |

### Connection Pooling

```
API (20 connections) → PgBouncer → PostgreSQL (100 max connections)
Workers (20 connections) → PgBouncer → PostgreSQL
```

PgBouncer in transaction mode: connections returned to pool after each transaction. Handles 100+ application connections with 20 actual DB connections.

---

## Queue-Based Backpressure

When upstream can't handle the load:

```
API accepts at any rate
  → RabbitMQ buffers
  → Workers consume at TPS limit
  → Jasmin submits at negotiated TPS
```

**Key insight:** The queue absorbs burst traffic. Workers enforce TPS limits per connector. API never blocks — always returns 202.

### TPS Enforcement in Workers

```python
import asyncio
from collections import defaultdict

class TPSLimiter:
    def __init__(self):
        self._counts: dict[str, int] = defaultdict(int)
        self._last_reset: float = time.time()

    async def acquire(self, connector_id: str, max_tps: int):
        """Block until TPS budget available for this connector."""
        while True:
            now = time.time()
            if now - self._last_reset >= 1.0:
                self._counts.clear()
                self._last_reset = now

            if self._counts[connector_id] < max_tps:
                self._counts[connector_id] += 1
                return
            await asyncio.sleep(0.05)  # Wait 50ms, try again
```

---

## Monitoring for Scale Decisions

Track these metrics to know when to scale:

| Metric | Scale When |
|---|---|
| API response time p95 | > 500ms sustained |
| Queue depth (dispatch) | Growing for > 5 minutes |
| Worker CPU | > 80% sustained |
| DB connection pool usage | > 80% |
| Message processing lag | > 30 seconds |
| SMPP throttle errors | > 1% of submissions |

---

## Anti-Patterns

- **Don't scale prematurely** — Measure first, then scale the bottleneck
- **Don't scale by making things bigger** — Scale by adding instances (horizontal > vertical)
- **Don't share database connections** — Use connection pooling (PgBouncer)
- **Don't forget partition maintenance** — Automate partition creation and archival
- **Don't mix queue priorities** — OTP and promotional in the same queue = OTP delayed by bulk campaign
- **Don't skip backpressure** — Without TPS enforcement, upstream will throttle you and messages will fail
