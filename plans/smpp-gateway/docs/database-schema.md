# Database Schema

## Schema Principles

1. **Multi-tenant by org_id** — Every business table has an `org_id` foreign key. All queries filter by org_id
2. **NUMERIC not float** — All money columns use `NUMERIC(18,6)`. No floating point ever
3. **Ledger-based billing** — `wallet_transactions` is append-only. Balance is denormalized on `wallets` but the ledger is the source of truth
4. **Partitioned messages** — `messages` table partitioned by `created_at` for query performance and archival
5. **Auditable status history** — Message status changes tracked via `delivery_reports`. Every wallet change has a transaction record
6. **UUIDs for public IDs** — All primary keys are UUIDs. No auto-increment IDs exposed to clients

---

## Tables

### organizations

```sql
CREATE TABLE organizations (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_organization_id  UUID REFERENCES organizations(id),  -- reseller hierarchy
    name                    VARCHAR(255) NOT NULL,
    slug                    VARCHAR(100) NOT NULL UNIQUE,
    legal_name              VARCHAR(255),
    billing_email           VARCHAR(255),
    timezone                VARCHAR(50) DEFAULT 'Asia/Kolkata',
    webhook_url             TEXT,
    webhook_secret          VARCHAR(255),
    settings                JSONB DEFAULT '{}',
    status                  VARCHAR(20) NOT NULL DEFAULT 'active'
                            CHECK (status IN ('active', 'suspended', 'pending', 'closed')),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    role            VARCHAR(20) NOT NULL DEFAULT 'viewer'
                    CHECK (role IN ('super_admin', 'org_admin', 'operator', 'api_user', 'viewer')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'invited', 'disabled')),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, email)
);
```

### wallets

```sql
CREATE TABLE wallets (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id                  UUID NOT NULL UNIQUE REFERENCES organizations(id),
    balance                 NUMERIC(18,6) NOT NULL DEFAULT 0,
    credit_limit            NUMERIC(18,6) NOT NULL DEFAULT 0,
    low_balance_threshold   NUMERIC(18,6) DEFAULT 100,
    currency                VARCHAR(3) NOT NULL DEFAULT 'INR',
    allow_negative          BOOLEAN NOT NULL DEFAULT false,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (allow_negative OR balance >= -credit_limit)
);
```

### wallet_transactions

```sql
CREATE TABLE wallet_transactions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organizations(id),
    wallet_id           UUID NOT NULL REFERENCES wallets(id),
    tx_type             VARCHAR(20) NOT NULL
                        CHECK (tx_type IN ('credit', 'debit', 'refund', 'adjustment', 'bonus')),
    amount              NUMERIC(18,6) NOT NULL,
    balance_before      NUMERIC(18,6) NOT NULL,
    balance_after       NUMERIC(18,6) NOT NULL,
    reference_type      VARCHAR(50),        -- 'message', 'admin_credit', 'refund', etc.
    reference_id        UUID,               -- FK to messages.id or other
    description         TEXT,
    created_by_user_id  UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No updated_at — append-only table
);
```

### rate_cards

```sql
CREATE TABLE rate_cards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID REFERENCES organizations(id),  -- NULL = default rate card
    route_id        UUID REFERENCES routes(id),
    message_class   VARCHAR(20) NOT NULL
                    CHECK (message_class IN ('transactional', 'promotional', 'otp')),
    country_code    VARCHAR(5) NOT NULL DEFAULT 'IN',
    buy_price       NUMERIC(18,6) NOT NULL,
    sell_price      NUMERIC(18,6) NOT NULL,
    billing_unit    VARCHAR(20) NOT NULL DEFAULT 'per_segment'
                    CHECK (billing_unit IN ('per_segment', 'per_message')),
    effective_from  DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to    DATE,                               -- NULL = no expiry
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### smpp_connectors

```sql
CREATE TABLE smpp_connectors (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(100) NOT NULL,
    jasmin_connector_id VARCHAR(100),
    direction           VARCHAR(20) NOT NULL
                        CHECK (direction IN ('upstream', 'downstream')),
    bind_mode           VARCHAR(5) NOT NULL DEFAULT 'trx'
                        CHECK (bind_mode IN ('tx', 'rx', 'trx')),
    host                VARCHAR(255) NOT NULL,
    port                INTEGER NOT NULL DEFAULT 2775,
    system_id           VARCHAR(100) NOT NULL,
    password_encrypted  TEXT NOT NULL,
    system_type         VARCHAR(50),
    max_binds           INTEGER NOT NULL DEFAULT 1,
    tps_limit           INTEGER NOT NULL DEFAULT 100,
    health_score        NUMERIC(3,2) NOT NULL DEFAULT 1.00
                        CHECK (health_score >= 0 AND health_score <= 1),
    circuit_state       VARCHAR(10) NOT NULL DEFAULT 'closed'
                        CHECK (circuit_state IN ('closed', 'open', 'half_open')),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    metadata            JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### routes

```sql
CREATE TABLE routes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(100) NOT NULL,
    connector_id        UUID NOT NULL REFERENCES smpp_connectors(id),
    route_type          VARCHAR(20) NOT NULL DEFAULT 'standard'
                        CHECK (route_type IN ('standard', 'premium', 'backup')),
    message_class       VARCHAR(20) NOT NULL
                        CHECK (message_class IN ('transactional', 'promotional', 'otp')),
    country_code        VARCHAR(5) NOT NULL DEFAULT 'IN',
    sender_id_pattern   VARCHAR(50),        -- regex pattern for allowed sender IDs
    dlt_required        BOOLEAN NOT NULL DEFAULT true,
    priority            INTEGER NOT NULL DEFAULT 100,   -- lower = higher priority
    weight              INTEGER NOT NULL DEFAULT 1,     -- for weighted round-robin
    max_tps_override    INTEGER,                        -- overrides connector TPS for this route
    cost_per_unit       NUMERIC(18,6),
    failure_rate        NUMERIC(5,4) NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### dlt_templates

```sql
CREATE TABLE dlt_templates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organizations(id),
    template_id         VARCHAR(100) NOT NULL,  -- DLT platform template ID
    entity_id           VARCHAR(100) NOT NULL,  -- DLT entity ID
    template_text       TEXT NOT NULL,
    template_type       VARCHAR(20)
                        CHECK (template_type IN ('transactional', 'promotional', 'otp', 'service_implicit', 'service_explicit')),
    approval_status     VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (approval_status IN ('pending', 'approved', 'rejected', 'expired')),
    approved_at         TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, template_id)
);
```

### sender_ids

```sql
CREATE TABLE sender_ids (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organizations(id),
    sender_id           VARCHAR(20) NOT NULL,
    entity_id           VARCHAR(100) NOT NULL,
    usage_type          VARCHAR(20) NOT NULL
                        CHECK (usage_type IN ('transactional', 'promotional', 'otp')),
    approval_status     VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (approval_status IN ('pending', 'approved', 'rejected', 'revoked')),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(org_id, sender_id, usage_type)
);
```

### api_keys

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    name            VARCHAR(100) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,        -- First 8 chars for fast lookup
    key_hash        VARCHAR(255) NOT NULL,       -- bcrypt hash of full key
    permissions     JSONB DEFAULT '["sms:send", "sms:read", "balance:read"]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_prefix ON api_keys(key_prefix) WHERE is_active = true;
```

### messages (partitioned)

```sql
CREATE TABLE messages (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    org_id              UUID NOT NULL REFERENCES organizations(id),
    user_id             UUID REFERENCES users(id),
    api_key_id          UUID REFERENCES api_keys(id),
    client_message_id   VARCHAR(100),
    idempotency_key     VARCHAR(255),
    sender_id           VARCHAR(20) NOT NULL,
    recipient           VARCHAR(20) NOT NULL,
    message_body        TEXT NOT NULL,
    encoding            VARCHAR(10) DEFAULT 'gsm7'
                        CHECK (encoding IN ('gsm7', 'ucs2')),
    segments_count      INTEGER NOT NULL DEFAULT 1,
    message_class       VARCHAR(20) NOT NULL
                        CHECK (message_class IN ('transactional', 'promotional', 'otp')),
    status              VARCHAR(20) NOT NULL DEFAULT 'queued'
                        CHECK (status IN ('queued', 'accepted', 'submitted', 'delivered',
                                         'failed', 'rejected', 'expired', 'unknown')),
    route_id            UUID REFERENCES routes(id),
    connector_id        UUID REFERENCES smpp_connectors(id),
    route_snapshot      JSONB,                  -- Snapshot of route config at submission time
    jasmin_message_id   VARCHAR(255),           -- Jasmin's internal message ID
    dlt_template_id     UUID REFERENCES dlt_templates(id),
    estimated_cost      NUMERIC(18,6),
    billed_cost         NUMERIC(18,6),
    fail_reason_code    VARCHAR(50),
    fail_reason_message TEXT,
    submit_time         TIMESTAMPTZ,
    delivery_time       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (example)
CREATE TABLE messages_2026_01 PARTITION OF messages
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE messages_2026_02 PARTITION OF messages
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE messages_2026_03 PARTITION OF messages
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

### delivery_reports

```sql
CREATE TABLE delivery_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id          UUID NOT NULL,
    message_created_at  TIMESTAMPTZ NOT NULL,   -- needed for partitioned FK
    org_id              UUID NOT NULL REFERENCES organizations(id),
    raw_payload         JSONB NOT NULL,          -- original DLR from upstream
    normalized_status   VARCHAR(20) NOT NULL
                        CHECK (normalized_status IN ('delivered', 'failed', 'expired',
                                                      'rejected', 'unknown')),
    error_code          VARCHAR(50),
    operator_response   TEXT,
    received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    FOREIGN KEY (message_id, message_created_at) REFERENCES messages(id, created_at)
);
```

---

## Indexes

```sql
-- Messages: lookup by org + status
CREATE INDEX idx_messages_org_status ON messages(org_id, status, created_at DESC);

-- Messages: idempotency check
CREATE UNIQUE INDEX idx_messages_idempotency ON messages(org_id, idempotency_key)
    WHERE idempotency_key IS NOT NULL;

-- Messages: client message ID lookup
CREATE INDEX idx_messages_client_id ON messages(org_id, client_message_id)
    WHERE client_message_id IS NOT NULL;

-- Messages: Jasmin correlation
CREATE INDEX idx_messages_jasmin_id ON messages(jasmin_message_id)
    WHERE jasmin_message_id IS NOT NULL;

-- Delivery reports: by message
CREATE INDEX idx_dlr_message ON delivery_reports(message_id, message_created_at);

-- Wallet transactions: by org + time
CREATE INDEX idx_wallet_tx_org ON wallet_transactions(org_id, created_at DESC);

-- Routes: active route lookup
CREATE INDEX idx_routes_active ON routes(message_class, country_code, priority)
    WHERE is_active = true;

-- Rate cards: pricing lookup
CREATE INDEX idx_rate_cards_lookup ON rate_cards(org_id, route_id, message_class, country_code)
    WHERE is_active = true AND (effective_to IS NULL OR effective_to >= CURRENT_DATE);
```

---

## Transaction Rules

### Wallet Debit Pattern (SMS Submission)

Every message submission follows this exact sequence in a single DB transaction:

```sql
BEGIN;

-- 1. Lock the wallet row
SELECT balance, credit_limit, allow_negative
FROM wallets WHERE org_id = $1 FOR UPDATE;

-- 2. Verify sufficient balance
-- Application checks: balance >= estimated_cost (or balance + credit_limit >= estimated_cost)

-- 3. Update wallet balance
UPDATE wallets
SET balance = balance - $estimated_cost, updated_at = now()
WHERE org_id = $1;

-- 4. Insert ledger entry
INSERT INTO wallet_transactions (org_id, wallet_id, tx_type, amount, balance_before, balance_after, reference_type, reference_id)
VALUES ($org_id, $wallet_id, 'debit', $estimated_cost, $old_balance, $new_balance, 'message', $message_id);

-- 5. Insert message
INSERT INTO messages (org_id, user_id, sender_id, recipient, message_body, ...)
VALUES (...);

COMMIT;
```

### Message Acceptance Pattern

```
1. Check Idempotency-Key → if exists, return cached response
2. Validate DLT template (if enforced)
3. Select route via strategy pattern
4. Snapshot route config into route_snapshot JSONB
5. Estimate cost from rate_card (org-specific or default)
6. Execute wallet debit transaction (above)
7. Queue message for dispatch via RabbitMQ
8. Cache idempotency response in Redis (TTL: 24h)
9. Return 202 Accepted
```

---

## Future Tables

These tables are planned but not in MVP:

- **audit_logs** — All admin actions with before/after snapshots
- **webhook_deliveries** — Track webhook delivery attempts and responses
- **scheduled_jobs** — For scheduled message dispatch
- **route_health_events** — Historical circuit breaker state changes
- **support_tickets** — Basic ticket system for client issues

---

## Anti-Patterns

- **Don't use SERIAL/auto-increment for public IDs** — UUIDs prevent enumeration attacks
- **Don't UPDATE wallet_transactions** — It's an append-only ledger. If you need to reverse, insert a new 'refund' or 'adjustment' row
- **Don't store money as FLOAT** — Always NUMERIC(18,6)
- **Don't skip the wallet lock** — SELECT ... FOR UPDATE prevents double-spend race conditions
- **Don't store raw API keys** — Store bcrypt hash + plaintext prefix for lookup
- **Don't query messages without org_id** — Always filter by org_id to leverage partitioning and enforce tenancy
