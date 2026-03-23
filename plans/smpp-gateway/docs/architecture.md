# Architecture

## High-Level System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
│  Enterprise Apps ─── Reseller Platforms ─── Admin Dashboard     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ HTTPS
                    ┌──────▼──────┐
                    │    Nginx    │  TLS termination, rate limiting
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐  ┌─▼──────┐  ┌─▼─────────┐
       │   FastAPI    │  │Next.js │  │  Worker    │
       │  (HTTP API)  │  │ (UI)   │  │ (Async)   │
       └──────┬───────┘  └────────┘  └─────┬─────┘
              │                             │
    ┌─────────┼──────────┐          ┌───────┼───────┐
    │         │          │          │       │       │
┌───▼──┐ ┌───▼───┐ ┌────▼────┐ ┌──▼──┐ ┌─▼────┐  │
│Postgr│ │ Redis │ │RabbitMQ │ │Redis│ │Rabbit│  │
│  eSQL │ │       │ │         │ │     │ │  MQ  │  │
└───┬───┘ └───────┘ └────┬────┘ └─────┘ └──┬───┘  │
    │                     │                 │      │
    │              ┌──────▼──────┐          │      │
    │              │   Jasmin    │◄─────────┘      │
    │              │ SMS Gateway │                  │
    │              └──────┬──────┘                  │
    │                     │ SMPP                    │
    │         ┌───────────┼───────────┐            │
    │    ┌────▼────┐ ┌────▼────┐ ┌────▼────┐      │
    │    │Vendor A │ │Vendor B │ │Telco    │      │
    │    │(Upstream)│ │(Upstream)│ │(Future) │      │
    │    └────┬────┘ └────┬────┘ └────┬────┘      │
    │         │ DLR       │ DLR       │ DLR       │
    │         └───────────┼───────────┘            │
    │              ┌──────▼──────┐                  │
    │              │   Jasmin    │──────────────────┘
    │              │  DLR Path   │   (DLR → Worker → DB + Webhook)
    │              └─────────────┘
    │
    └── Message logs, billing, DLR status updates
```

## Component Breakdown

### 1. FastAPI — HTTP API Server
- Client-facing REST API (send SMS, check balance, view reports)
- Admin API (manage routes, connectors, organizations)
- Authentication (JWT + API key validation)
- Request validation, rate limiting, idempotency enforcement
- Route selection logic (chooses which connector to use)
- Does NOT speak SMPP directly — delegates to Jasmin via RabbitMQ

### 2. Jasmin SMS Gateway — SMPP Transport
- Maintains SMPP binds to upstream providers
- Executes message delivery on the connector chosen by FastAPI
- Receives DLR callbacks from upstream
- Publishes DLR events to RabbitMQ for worker processing
- Future: Exposes SMPP server interface for downstream clients

### 3. Worker — Async Processing
- Consumes message dispatch jobs from RabbitMQ
- Submits to Jasmin via its HTTP API
- Processes DLR events: updates message status, triggers webhooks
- Handles retry logic with exponential backoff
- Runs scheduled message dispatch

### 4. Next.js — Client Portal & Admin Dashboard
- Client view: balance, send history, delivery reports, API docs
- Admin view: connector health, route management, org management, financial reports
- BFF pattern: Next.js API routes proxy to FastAPI

### 5. PostgreSQL — Primary Data Store
- Organizations, users, wallets, wallet_transactions (append-only ledger)
- Messages (partitioned by created_at), delivery_reports
- Routes, connectors, rate_cards, DLT templates, sender IDs, API keys
- ACID guarantees for all billing operations

### 6. Redis + RabbitMQ — Cache & Message Broker
- **Redis:** Session cache, rate limiting counters, DND/NDNC number cache, circuit breaker state
- **RabbitMQ:** Async message dispatch queue, DLR processing queue, webhook delivery queue

---

## Design Patterns

### 1. Strategy Pattern — Route Selection

```python
from typing import Protocol

class RouteStrategy(Protocol):
    def select_route(self, message: MessageRequest, available_routes: list[Route]) -> Route:
        ...

class PriorityRouteStrategy:
    """Select highest-priority active route matching message class and country."""
    def select_route(self, message, available_routes):
        eligible = [r for r in available_routes
                    if r.is_active
                    and r.message_class == message.message_class
                    and r.country_code == message.country_code]
        return sorted(eligible, key=lambda r: r.priority)[0]

class CostOptimizedStrategy:
    """Select cheapest route that meets quality threshold."""
    def select_route(self, message, available_routes):
        eligible = [r for r in available_routes
                    if r.is_active and r.health_score > 0.8]
        return sorted(eligible, key=lambda r: r.cost_per_unit)[0]

class WeightedRoundRobinStrategy:
    """Distribute across routes by weight for load balancing."""
    ...
```

### 2. Circuit Breaker — Connector Health

```python
class CircuitBreaker:
    CLOSED = "closed"       # Normal operation
    OPEN = "open"           # Failing — stop sending
    HALF_OPEN = "half_open" # Testing recovery

    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.state = self.CLOSED
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure_time = None

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = self.OPEN

    def record_success(self):
        self.failure_count = 0
        self.state = self.CLOSED

    def can_send(self) -> bool:
        if self.state == self.CLOSED:
            return True
        if self.state == self.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = self.HALF_OPEN
                return True  # Allow one test message
            return False
        return True  # HALF_OPEN — allow test
```

### 3. Observer / Event-Driven — Message Lifecycle

Events published to RabbitMQ at each stage:

| Event | Trigger | Consumer Action |
|---|---|---|
| `message.accepted` | API receives valid request | Wallet debit, queue for dispatch |
| `message.submitted` | Jasmin confirms SMPP submit | Update status to submitted |
| `message.delivered` | DLR received with success | Update status, trigger webhook |
| `message.failed` | DLR received with failure | Update status, trigger webhook, check retry |
| `message.expired` | TTL exceeded without DLR | Update status, trigger webhook |
| `wallet.low_balance` | Balance drops below threshold | Email/webhook notification |

### 4. Multi-Tenant — Shared Infrastructure

- Every table has `org_id` foreign key
- All queries filtered by org_id (enforced at repository layer)
- Rate cards per-org allow different pricing per client
- Wallet isolation — each org has exactly one wallet
- API keys scoped to org_id
- Reseller hierarchy via `parent_organization_id`

### 5. Repository Pattern — Data Access

```python
class MessageRepository:
    async def create(self, message: MessageCreate) -> Message: ...
    async def get_by_id(self, org_id: UUID, message_id: UUID) -> Message: ...
    async def update_status(self, message_id: UUID, status: str, dlr_data: dict) -> None: ...
    async def list_by_org(self, org_id: UUID, filters: MessageFilter) -> Page[Message]: ...

class WalletRepository:
    async def get_balance(self, org_id: UUID) -> Decimal: ...
    async def debit(self, org_id: UUID, amount: Decimal, reference: str) -> WalletTransaction: ...
    async def credit(self, org_id: UUID, amount: Decimal, reference: str) -> WalletTransaction: ...

class RouteRepository:
    async def get_active_routes(self, message_class: str, country_code: str) -> list[Route]: ...
    async def update_health(self, route_id: UUID, health_score: float) -> None: ...

class ConnectorRepository:
    async def get_by_id(self, connector_id: UUID) -> Connector: ...
    async def update_circuit_state(self, connector_id: UUID, state: str) -> None: ...
```

### 6. Middleware Chain — Request Processing

Every API request passes through:

1. **Rate Limiter** — Redis-based, per API key
2. **Auth** — JWT validation or API key hash lookup
3. **Org Context** — Load org from token, inject into request state
4. **Request ID** — Generate unique request_id for tracing
5. **Idempotency** — Check Idempotency-Key header, return cached response if duplicate
6. **Validation** — Pydantic model validation
7. **Business Logic** — Route selection, wallet check, message creation
8. **Audit** — Log action for compliance trail
9. **Response** — Consistent JSON envelope with request_id

---

## Primary SMS Flow

```
1.  Client → POST /api/v1/sms/send (with API key + Idempotency-Key)
2.  Nginx → FastAPI (TLS terminated, rate limit checked)
3.  Auth middleware → Validate API key, load org context
4.  Idempotency check → Redis lookup by Idempotency-Key
5.  Validate request → Pydantic schema, phone number format
6.  DLT check → Verify template_id matches registered template (if DLT enforced)
7.  Route selection → Strategy pattern picks best route for message_class + country
8.  Cost estimation → Look up rate_card for org + route + message_class
9.  Wallet check → SELECT balance FROM wallets WHERE org_id = ? FOR UPDATE
10. Wallet debit → INSERT wallet_transaction, UPDATE wallet balance
11. Create message → INSERT into messages table (status: 'accepted')
12. Queue dispatch → Publish to RabbitMQ dispatch queue
13. Return 202 → { message_id, status: "accepted", estimated_cost }
14. Worker picks up → Submits to Jasmin HTTP API with chosen connector
15. Jasmin submits → SMPP submit_sm to upstream provider
16. Status update → message status → 'submitted'
```

## DLR Flow

```
1. Upstream provider → SMPP deliver_sm (DLR) → Jasmin
2. Jasmin → Publishes DLR event to RabbitMQ
3. Worker consumes → Parses DLR payload (vendor-specific normalization)
4. Worker → INSERT delivery_report, UPDATE message status
5. Worker → Check if org has webhook_url configured
6. Worker → POST webhook to client (HMAC signed, with retry)
7. Worker → Log webhook delivery attempt
8. If webhook fails → Queue for retry (max 3 attempts, exponential backoff)
```

---

## Security Architecture

### API Security
- **API key auth** — key_prefix for fast lookup, bcrypt hash for verification, pepper in env
- **JWT** — For admin/portal sessions, short expiry (15 min), refresh token rotation
- **RBAC** — 5 roles: super_admin, org_admin, operator, api_user, viewer
- **Request signing** (Phase 3) — HMAC signature on request body for high-value operations
- **Rate limiting** — Per API key, configurable per org, Redis sliding window

### SMPP Security
- **IP whitelist** — Per connector, enforced at Jasmin level
- **Per-connector credentials** — Unique system_id/password per upstream bind
- **TPS limits** — Per connector, enforced in dispatch worker
- **Bind monitoring** — Alert on unexpected unbind or connection drop

### Wallet / Billing Security
- **Decimal arithmetic** — NUMERIC(18,6), never float
- **Append-only ledger** — wallet_transactions are INSERT only, no UPDATE/DELETE
- **Atomic transactions** — Wallet debit + message creation in single DB transaction
- **Cost snapshot** — estimated_cost recorded at submission time, billed_cost at delivery
- **Balance check with lock** — SELECT ... FOR UPDATE prevents race conditions

### Operational Security
- **Secrets in environment** — Never in code or config files
- **Admin isolation** — Admin API endpoints on separate auth middleware
- **Encrypted backups** — PostgreSQL backups encrypted at rest
- **Audit trail** — All admin actions logged with user_id, timestamp, details

---

## Reliability Model

### Failure Handling

| Failure | Mitigation |
|---|---|
| Upstream connector down | Circuit breaker opens, route selection skips, alert fired |
| Duplicate API request | Idempotency-Key returns cached 202 response |
| Worker crash mid-dispatch | RabbitMQ redelivers unacked message |
| Database connection lost | Connection pool retry with backoff |
| Webhook delivery fails | Retry queue: 3 attempts at 30s, 2min, 10min |
| DLR never arrives | TTL expiry job marks message as 'expired' after configurable timeout |

### Observability

Tracked metrics (Prometheus + Grafana):

1. **Messages per second** — by status, org, route
2. **Delivery rate** — delivered / (delivered + failed) per route
3. **Latency** — submit-to-DLR time per route
4. **Queue depth** — RabbitMQ dispatch and DLR queues
5. **Wallet operations** — debits/credits per minute
6. **API response time** — p50, p95, p99
7. **SMPP bind status** — per connector, uptime percentage
8. **Circuit breaker state** — per connector
9. **Error rate** — by error code, per route
10. **DLR match rate** — DLRs matched to messages vs orphaned
11. **Webhook delivery rate** — success vs failure per org

---

## Architecture Decisions

### Why Jasmin + FastAPI (not build SMPP from scratch)
- Jasmin handles SMPP protocol complexity (binds, PDU encoding, DLR correlation)
- FastAPI handles business logic (billing, routing, auth) where we add value
- Clear separation: Jasmin = transport, FastAPI = business
- Can replace Jasmin with direct SMPP library later if needed

### Why shared PostgreSQL (not microservice databases)
- Single source of truth for billing — no distributed transaction complexity
- Simpler operations for Tier 2 stage
- Partitioning handles message volume
- Split databases when we actually need it (Tier 3+)

### Why event-driven via RabbitMQ (not synchronous dispatch)
- API returns 202 immediately — client doesn't wait for SMPP round-trip
- Worker can retry without blocking API
- Natural backpressure via queue depth
- DLR processing decoupled from submission

---

## Anti-Patterns

- **Don't put billing logic in the worker** — Wallet debit happens in the API request transaction, before queuing
- **Don't fan-out message creation** — One message = one DB row = one queue item, even for bulk
- **Don't cache wallet balances** — Always read from DB with row lock for debit operations
- **Don't normalize DLR formats prematurely** — Store raw_payload in delivery_reports, normalize into a known set of statuses
- **Don't skip idempotency for "simple" endpoints** — Every state-changing endpoint needs it
- **Don't log message content** — Log message_id, org_id, route, status — never the SMS body
