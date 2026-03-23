# System Architecture

## C4 Model

### Level 1 — System Context

Who interacts with the system and what external systems it depends on.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Enterprise  │     │   Reseller   │     │    Admin     │
│   Clients    │     │  Platforms   │     │  Dashboard   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       │         HTTPS      │         HTTPS      │
       └────────────┬───────┘────────────────────┘
                    │
            ┌───────▼───────┐
            │               │
            │  SMS Platform │
            │               │
            └───────┬───────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────▼────┐ ┌───▼────┐ ┌───▼─────┐
   │Vendor A │ │Vendor B│ │ Telco   │
   │  (SMPP) │ │ (SMPP) │ │(Future) │
   └─────────┘ └────────┘ └─────────┘
```

**Key relationships:**
- Clients submit SMS via HTTPS, receive DLR via webhooks
- Platform connects to upstream vendors via SMPP protocol
- Admin manages routes, connectors, orgs via dashboard

### Level 2 — Container Diagram

The major deployable units and how they communicate.

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
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
└───────┘ └───────┘ └────┬────┘ └─────┘ └──┬───┘  │
                         │                  │      │
                  ┌──────▼──────┐           │      │
                  │   Jasmin    │◄──────────┘      │
                  │ SMS Gateway │                   │
                  └──────┬──────┘                   │
                         │ SMPP                     │
             ┌───────────┼───────────┐              │
        ┌────▼────┐ ┌────▼────┐ ┌────▼────┐        │
        │Vendor A │ │Vendor B │ │Telco    │        │
        └────┬────┘ └────┬────┘ └────┬────┘        │
             │ DLR       │ DLR       │ DLR         │
             └───────────┼───────────┘              │
                  ┌──────▼──────┐                   │
                  │   Jasmin    │───────────────────┘
                  │  DLR Path   │  (DLR → RabbitMQ → Worker)
                  └─────────────┘
```

**Containers:**

| Container | Technology | Responsibility |
|---|---|---|
| Nginx | nginx:alpine | TLS termination, rate limiting, static files |
| API Server | FastAPI (Python) | Business logic, auth, routing decisions, billing |
| Frontend | Next.js (React) | Client portal, admin dashboard, BFF |
| Worker | Python | Async dispatch, DLR processing, webhook delivery |
| PostgreSQL | PostgreSQL 15+ | Primary data store, ACID billing |
| Redis | Redis 7 | Cache, rate limiting, DND lookup, circuit breaker state |
| RabbitMQ | RabbitMQ 3 | Message queues (dispatch, DLR, webhook) |
| Jasmin | Jasmin SMS Gateway | SMPP transport, bind management, DLR reception |

### Level 3 — Component Diagram (API Server)

Inside the FastAPI API Server:

```
┌─────────────────────────────────────────────────────┐
│                    FastAPI API Server                │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │              Middleware Chain                  │  │
│  │  Rate Limiter → Auth → Org Context →          │  │
│  │  Request ID → Idempotency → Validation        │  │
│  └──────────────────────┬───────────────────────┘  │
│                         │                           │
│  ┌──────────┐ ┌────────▼───┐ ┌──────────────────┐ │
│  │ Auth     │ │ SMS        │ │ Admin            │ │
│  │ Service  │ │ Service    │ │ Service          │ │
│  │          │ │            │ │                  │ │
│  │ - Login  │ │ - Send     │ │ - Connectors    │ │
│  │ - API key│ │ - Bulk     │ │ - Routes        │ │
│  │ - RBAC   │ │ - Status   │ │ - Organizations │ │
│  └──────────┘ │ - Reports  │ │ - Monitoring    │ │
│               └─────┬──────┘ └──────────────────┘ │
│                     │                               │
│  ┌──────────┐ ┌────▼───────┐ ┌──────────────────┐ │
│  │ Billing  │ │ Routing    │ │ Compliance       │ │
│  │ Service  │ │ Engine     │ │ Service          │ │
│  │          │ │            │ │                  │ │
│  │ - Wallet │ │ - Strategy │ │ - DLT validate  │ │
│  │ - Debit  │ │ - Circuit  │ │ - DND check     │ │
│  │ - Ledger │ │   breaker  │ │ - Time window   │ │
│  │ - Rates  │ │ - Health   │ │ - Sender ID     │ │
│  └──────────┘ └────────────┘ └──────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │            Repository Layer                   │  │
│  │  MessageRepo  WalletRepo  RouteRepo  OrgRepo │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## Design Patterns

### 1. Strategy Pattern — Route Selection

See [06-routing-logic.md](06-routing-logic.md) for full details.

Route selection is pluggable. Different strategies per message class:
- OTP → Failover (reliability first)
- Transactional → Quality-first (delivery rate)
- Promotional → Least-cost (margin optimization)

### 2. Circuit Breaker — Connector Health

Prevents sending to failing connectors. Three states: closed (normal) → open (blocking) → half_open (testing).

See [06-routing-logic.md](06-routing-logic.md) for implementation.

### 3. Event-Driven — Message Lifecycle

Events published to RabbitMQ at each lifecycle stage:

| Event | Trigger | Consumer Action |
|---|---|---|
| `message.accepted` | API validates and queues | Wallet debit, queue for dispatch |
| `message.submitted` | Jasmin confirms submit_sm_resp | Update status |
| `message.delivered` | DLR received with success | Update status, fire webhook |
| `message.failed` | DLR with failure | Update status, fire webhook, check retry |
| `message.expired` | TTL exceeded | Update status, fire webhook |
| `wallet.low_balance` | Balance below threshold | Notify client |

### 4. Repository Pattern — Data Access

All database operations go through repository classes. Enforces:
- org_id filtering on every query
- Consistent error handling
- Clean separation from business logic

```python
class MessageRepository:
    async def create(self, message: MessageCreate) -> Message: ...
    async def get_by_id(self, org_id: UUID, message_id: UUID) -> Message: ...
    async def update_status(self, message_id: UUID, status: str, dlr_data: dict) -> None: ...

class WalletRepository:
    async def get_balance(self, org_id: UUID) -> Decimal: ...
    async def debit(self, org_id: UUID, amount: Decimal, ref: str) -> WalletTransaction: ...
    async def credit(self, org_id: UUID, amount: Decimal, ref: str) -> WalletTransaction: ...
```

### 5. Middleware Chain — Request Processing

Every API request passes through (in order):
1. Rate Limiter (Redis sliding window)
2. Auth (JWT or API key)
3. Org Context (load org, inject into request state)
4. Request ID (unique ID for tracing)
5. Idempotency (check/cache Idempotency-Key)
6. Validation (Pydantic)
7. Business Logic
8. Audit logging
9. Response (consistent JSON envelope)

### 6. Multi-Tenant Isolation

- Every business table has `org_id` foreign key
- All queries filtered by org_id at repository layer
- Rate cards per-org for different pricing
- Wallet per-org for billing isolation
- API keys scoped to org_id
- Reseller hierarchy via `parent_organization_id`

---

## Primary SMS Flow

```
 1. Client → POST /api/v1/sms/send (API key + Idempotency-Key)
 2. Nginx → FastAPI (TLS terminated, rate limit checked)
 3. Auth middleware → Validate API key, load org context
 4. Idempotency check → Redis lookup by Idempotency-Key
 5. Validate request → Pydantic schema, E.164 phone format
 6. DLT check → Template matches message body
 7. Route selection → Strategy picks best route for class + country
 8. Cost estimation → Rate card lookup (org-specific or default)
 9. Wallet lock → SELECT balance FROM wallets WHERE org_id = ? FOR UPDATE
10. Wallet debit → INSERT wallet_transaction, UPDATE wallet balance
11. Create message → INSERT into messages (status: 'accepted')
12. Queue dispatch → Publish to RabbitMQ dispatch queue
13. Return 202 → { message_id, status: "accepted", estimated_cost }
14. Worker picks up → Submit to Jasmin HTTP API with chosen connector
15. Jasmin → SMPP submit_sm to upstream
16. Status → 'submitted'
```

## DLR Flow

```
1. Upstream → SMPP deliver_sm (DLR) → Jasmin
2. Jasmin → Publish DLR to RabbitMQ
3. Worker → Parse + normalize vendor-specific DLR
4. Worker → INSERT delivery_report, UPDATE message status
5. Worker → POST webhook to client (HMAC signed)
6. If webhook fails → Queue for retry (30s, 2min, 10min)
```

## What Happens When a Route Fails

```
1. Worker submits to Jasmin → SMPP error or timeout
2. Worker records failure on connector
3. Circuit breaker evaluates: failure_count >= threshold?
   ├── No → Retry on same route (up to max_retries)
   └── Yes → Circuit opens
4. Circuit OPEN → Alert fired, route marked unavailable
5. Next message → Route selection skips open circuit
6. After recovery_timeout → Circuit enters HALF_OPEN
7. Test message sent through half-open circuit
   ├── Success → Circuit closes, route available again
   └── Failure → Circuit re-opens, timer resets
```

---

## Security Architecture

### API Security
- API key: prefix for fast lookup, bcrypt hash for verification, pepper in env
- JWT: 15-min expiry, refresh token rotation
- RBAC: 5 roles with permission-based access
- Rate limiting: per API key, Redis sliding window

### SMPP Security
- IP whitelist per connector
- Unique credentials per upstream bind
- TPS limits enforced in dispatch worker
- Bind monitoring with alerts on unexpected disconnect

### Billing Security
- NUMERIC(18,6) for all money — never float
- Append-only ledger (INSERT only, never UPDATE/DELETE)
- Atomic transactions (wallet + message in single BEGIN/COMMIT)
- Cost snapshot at submission time
- Balance check with row lock (SELECT FOR UPDATE)

### Operational Security
- Secrets in environment variables only
- Admin API on separate auth middleware
- Encrypted database backups
- Audit trail for all admin actions
- Message content never logged

---

## Anti-Patterns

- **Don't put billing in the worker** — Wallet debit happens in the API request, before queuing
- **Don't fan-out message creation** — One message = one row = one queue item
- **Don't cache wallet balances** — Always read from DB with row lock
- **Don't normalize DLR prematurely** — Store raw payload, normalize into known statuses
- **Don't skip idempotency** — Every state-changing endpoint needs it
- **Don't log message content** — Log message_id, org_id, route, status — never the body
