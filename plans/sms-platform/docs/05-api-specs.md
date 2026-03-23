# API Specifications

## API Principles

1. **Versioned** — All endpoints under `/api/v1/`
2. **Money as strings** — All monetary values serialized as strings to prevent floating point issues
3. **Consistent errors** — Every error follows the standard error envelope
4. **Idempotency required** — All POST endpoints that create resources require `Idempotency-Key` header
5. **Pagination** — List endpoints use cursor-based pagination with `page` and `per_page`
6. **Request ID** — Every response includes `request_id` for support and debugging

---

## Authentication

### POST /api/v1/auth/login

Request:
```json
{
  "email": "admin@example.com",
  "password": "secure_password"
}
```

Response (200):
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 900,
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "admin@example.com",
    "role": "org_admin",
    "org_id": "660e8400-e29b-41d4-a716-446655440000"
  }
}
```

### POST /api/v1/auth/api-keys

Creates an API key. The full key is returned **once** — it cannot be retrieved again.

Request:
```json
{
  "name": "Production SMS Key",
  "permissions": ["sms:send", "sms:read", "balance:read"],
  "expires_at": "2027-01-01T00:00:00Z"
}
```

Response (201):
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "name": "Production SMS Key",
  "key": "smsgw_live_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "key_prefix": "smsgw_li",
  "permissions": ["sms:send", "sms:read", "balance:read"],
  "expires_at": "2027-01-01T00:00:00Z",
  "created_at": "2026-03-23T10:00:00Z"
}
```

---

## SMS

### POST /api/v1/sms/send

Headers:
```
Authorization: Bearer smsgw_live_a1b2c3d4...
Idempotency-Key: client-unique-id-12345
```

Request:
```json
{
  "sender_id": "MYCOMP",
  "recipient": "+919876543210",
  "message": "Your OTP is 123456. Valid for 5 minutes.",
  "message_class": "otp",
  "client_message_id": "order-otp-789",
  "dlt_template_id": "1107161234567890123",
  "callback_url": "https://client.example.com/dlr"
}
```

Response (202):
```json
{
  "message_id": "880e8400-e29b-41d4-a716-446655440000",
  "client_message_id": "order-otp-789",
  "status": "accepted",
  "segments": 1,
  "estimated_cost": "0.250000",
  "route": {
    "id": "990e8400-e29b-41d4-a716-446655440000",
    "name": "premium-otp"
  },
  "request_id": "req_abc123def456"
}
```

### POST /api/v1/sms/bulk

Request:
```json
{
  "sender_id": "MYCOMP",
  "message_class": "promotional",
  "dlt_template_id": "1107161234567890456",
  "messages": [
    {
      "recipient": "+919876543210",
      "message": "Hi Rahul, check out our new offers!",
      "client_message_id": "camp-001-001"
    },
    {
      "recipient": "+919876543211",
      "message": "Hi Priya, check out our new offers!",
      "client_message_id": "camp-001-002"
    }
  ],
  "schedule_at": "2026-03-24T09:00:00+05:30"
}
```

Response (202):
```json
{
  "batch_id": "aa0e8400-e29b-41d4-a716-446655440000",
  "total": 2,
  "accepted": 2,
  "rejected": 0,
  "estimated_cost": "0.500000",
  "scheduled_at": "2026-03-24T09:00:00+05:30",
  "request_id": "req_def789ghi012"
}
```

### GET /api/v1/sms/{message_id}/status

Response (200):
```json
{
  "message_id": "880e8400-e29b-41d4-a716-446655440000",
  "client_message_id": "order-otp-789",
  "status": "delivered",
  "sender_id": "MYCOMP",
  "recipient": "+919876543210",
  "segments": 1,
  "estimated_cost": "0.250000",
  "billed_cost": "0.250000",
  "submitted_at": "2026-03-23T10:00:01Z",
  "delivered_at": "2026-03-23T10:00:03Z",
  "request_id": "req_ghi345jkl678"
}
```

### GET /api/v1/sms/reports

Query params: `status`, `date_from`, `date_to`, `page`, `per_page`

Response (200):
```json
{
  "data": [
    {
      "message_id": "880e8400-e29b-41d4-a716-446655440000",
      "client_message_id": "order-otp-789",
      "sender_id": "MYCOMP",
      "recipient": "+919876543210",
      "status": "delivered",
      "message_class": "otp",
      "segments": 1,
      "billed_cost": "0.250000",
      "created_at": "2026-03-23T10:00:00Z",
      "delivered_at": "2026-03-23T10:00:03Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 1,
    "total_pages": 1
  },
  "request_id": "req_mno901pqr234"
}
```

---

## Balance

### GET /api/v1/balance

Response (200):
```json
{
  "org_id": "660e8400-e29b-41d4-a716-446655440000",
  "balance": "15420.500000",
  "currency": "INR",
  "credit_limit": "0.000000",
  "low_balance_threshold": "100.000000",
  "is_low": false,
  "request_id": "req_stu567vwx890"
}
```

### GET /api/v1/balance/transactions

Query params: `tx_type`, `date_from`, `date_to`, `page`, `per_page`

Response (200):
```json
{
  "data": [
    {
      "id": "bb0e8400-e29b-41d4-a716-446655440000",
      "tx_type": "debit",
      "amount": "0.250000",
      "balance_before": "15420.750000",
      "balance_after": "15420.500000",
      "reference_type": "message",
      "reference_id": "880e8400-e29b-41d4-a716-446655440000",
      "description": "SMS to +919876543210",
      "created_at": "2026-03-23T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 1,
    "total_pages": 1
  },
  "request_id": "req_yza123bcd456"
}
```

---

## DLT

### GET /api/v1/dlt/templates

Response (200):
```json
{
  "data": [
    {
      "id": "cc0e8400-e29b-41d4-a716-446655440000",
      "template_id": "1107161234567890123",
      "entity_id": "1101234567890",
      "template_text": "Your OTP is {#var#}. Valid for {#var#} minutes.",
      "template_type": "otp",
      "approval_status": "approved",
      "approved_at": "2026-03-15T00:00:00Z",
      "expires_at": null
    }
  ],
  "request_id": "req_efg789hij012"
}
```

### POST /api/v1/dlt/templates

Request:
```json
{
  "template_id": "1107161234567890789",
  "entity_id": "1101234567890",
  "template_text": "Dear {#var#}, your order {#var#} has been shipped.",
  "template_type": "transactional"
}
```

Response (201):
```json
{
  "id": "dd0e8400-e29b-41d4-a716-446655440000",
  "template_id": "1107161234567890789",
  "approval_status": "pending",
  "created_at": "2026-03-23T10:00:00Z",
  "request_id": "req_klm345nop678"
}
```

### GET /api/v1/dlt/sender-ids

Response (200):
```json
{
  "data": [
    {
      "id": "ee0e8400-e29b-41d4-a716-446655440000",
      "sender_id": "MYCOMP",
      "entity_id": "1101234567890",
      "usage_type": "transactional",
      "approval_status": "approved"
    }
  ],
  "request_id": "req_qrs901tuv234"
}
```

---

## Admin — Connectors

### GET /api/v1/admin/connectors

Response (200):
```json
{
  "data": [
    {
      "id": "ff0e8400-e29b-41d4-a716-446655440000",
      "name": "Vendor A - Transactional",
      "jasmin_connector_id": "vendor_a_trx",
      "direction": "upstream",
      "bind_mode": "trx",
      "host": "smpp.vendora.com",
      "port": 2775,
      "system_id": "mycompany",
      "max_binds": 2,
      "tps_limit": 100,
      "health_score": 0.95,
      "circuit_state": "closed",
      "is_active": true
    }
  ],
  "request_id": "req_wxy567zab890"
}
```

### POST /api/v1/admin/connectors

Request:
```json
{
  "name": "Vendor B - Promotional",
  "jasmin_connector_id": "vendor_b_promo",
  "direction": "upstream",
  "bind_mode": "trx",
  "host": "smpp.vendorb.com",
  "port": 2775,
  "system_id": "mycompany_promo",
  "password": "secure_password",
  "max_binds": 1,
  "tps_limit": 50
}
```

### PUT /api/v1/admin/connectors/{connector_id}

### DELETE /api/v1/admin/connectors/{connector_id}

---

## Admin — Routes

### GET /api/v1/admin/routes

### POST /api/v1/admin/routes

Request:
```json
{
  "name": "Premium OTP Route",
  "connector_id": "ff0e8400-e29b-41d4-a716-446655440000",
  "route_type": "premium",
  "message_class": "otp",
  "country_code": "IN",
  "dlt_required": true,
  "priority": 10,
  "weight": 1,
  "max_tps_override": 50
}
```

### POST /api/v1/admin/routes/{route_id}/test

Request:
```json
{
  "recipient": "+919876543210",
  "message": "Route test message",
  "sender_id": "MYCOMP",
  "message_class": "transactional",
  "dlt_template_id": "1107161234567890123"
}
```

Response (200):
```json
{
  "validation_status": "pass",
  "route_name": "Premium OTP Route",
  "connector_name": "Vendor A - Transactional",
  "estimated_cost": "0.250000",
  "checks": {
    "dlt_template": "valid",
    "sender_id": "approved",
    "route_active": true,
    "connector_healthy": true,
    "balance_sufficient": true
  },
  "request_id": "req_cde123fgh456"
}
```

### PUT /api/v1/admin/routes/{route_id}

### DELETE /api/v1/admin/routes/{route_id}

---

## Admin — Organizations

### GET /api/v1/admin/organizations

### POST /api/v1/admin/organizations

Request:
```json
{
  "name": "Acme Corp",
  "slug": "acme-corp",
  "legal_name": "Acme Corporation Pvt Ltd",
  "billing_email": "billing@acme.com",
  "timezone": "Asia/Kolkata",
  "webhook_url": "https://acme.com/webhooks/sms",
  "initial_credit": "10000.000000"
}
```

Response (201):
```json
{
  "id": "110e8400-e29b-41d4-a716-446655440000",
  "name": "Acme Corp",
  "slug": "acme-corp",
  "status": "active",
  "wallet": {
    "id": "220e8400-e29b-41d4-a716-446655440000",
    "balance": "10000.000000",
    "currency": "INR"
  },
  "created_at": "2026-03-23T10:00:00Z",
  "request_id": "req_ijk789lmn012"
}
```

### POST /api/v1/admin/organizations/{org_id}/credit

Request:
```json
{
  "amount": "5000.000000",
  "description": "Monthly top-up via bank transfer"
}
```

Response (200):
```json
{
  "transaction_id": "330e8400-e29b-41d4-a716-446655440000",
  "amount": "5000.000000",
  "balance_before": "10000.000000",
  "new_balance": "15000.000000",
  "request_id": "req_opq345rst678"
}
```

### GET /api/v1/admin/organizations/{org_id}

### PUT /api/v1/admin/organizations/{org_id}

---

## Admin — Monitoring

### GET /api/v1/admin/monitoring/dashboard

Response (200):
```json
{
  "live_tps": 42,
  "queue_depth": {
    "dispatch": 150,
    "dlr": 23,
    "webhook": 5
  },
  "delivery_rate": {
    "last_1h": 0.967,
    "last_24h": 0.954
  },
  "route_health": [
    {
      "route_id": "990e8400-e29b-41d4-a716-446655440000",
      "name": "Premium OTP Route",
      "connector": "Vendor A - Transactional",
      "health_score": 0.95,
      "circuit_state": "closed",
      "current_tps": 28,
      "max_tps": 100
    }
  ],
  "active_connectors": 3,
  "total_connectors": 4,
  "request_id": "req_uvw901xyz234"
}
```

### GET /api/v1/admin/monitoring/reports

Query params: `date_from`, `date_to`, `group_by` (org, route, message_class)

Response (200):
```json
{
  "period": {
    "from": "2026-03-01",
    "to": "2026-03-23"
  },
  "summary": {
    "total_messages": 150000,
    "delivered": 144150,
    "failed": 4500,
    "delivery_rate": 0.961,
    "revenue": "37500.000000",
    "cost": "22500.000000",
    "gross_margin": "15000.000000",
    "margin_percentage": 0.40
  },
  "by_message_class": [
    {
      "message_class": "transactional",
      "count": 80000,
      "revenue": "20000.000000",
      "cost": "12000.000000"
    },
    {
      "message_class": "promotional",
      "count": 50000,
      "revenue": "10000.000000",
      "cost": "6000.000000"
    },
    {
      "message_class": "otp",
      "count": 20000,
      "revenue": "7500.000000",
      "cost": "4500.000000"
    }
  ],
  "request_id": "req_abc567def890"
}
```

---

## Webhooks

When a message status changes, the system POSTs to the organization's `webhook_url`:

```
POST https://client.example.com/webhooks/sms
Content-Type: application/json
X-Webhook-Signature: sha256=a1b2c3d4e5f6...
X-Webhook-Timestamp: 1711180800
```

```json
{
  "event": "message.delivered",
  "message_id": "880e8400-e29b-41d4-a716-446655440000",
  "client_message_id": "order-otp-789",
  "status": "delivered",
  "sender_id": "MYCOMP",
  "recipient": "+919876543210",
  "delivered_at": "2026-03-23T10:00:03Z",
  "segments": 1,
  "billed_cost": "0.250000",
  "error_code": null,
  "timestamp": "2026-03-23T10:00:04Z"
}
```

**Signature verification:**
```python
import hmac, hashlib

expected = hmac.new(
    webhook_secret.encode(),
    f"{timestamp}.{body}".encode(),
    hashlib.sha256
).hexdigest()

assert hmac.compare_digest(f"sha256={expected}", signature_header)
```

**Retry policy:** 3 attempts at 30s, 2min, 10min intervals. Failed deliveries logged in webhook_deliveries table.

---

## Error Format

All errors follow this structure:

```json
{
  "error": {
    "code": "INSUFFICIENT_BALANCE",
    "message": "Wallet balance too low to process this request",
    "details": {
      "current_balance": "0.500000",
      "required_amount": "0.250000",
      "currency": "INR"
    },
    "request_id": "req_err123abc456"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Request body/params failed validation |
| `INVALID_TEMPLATE` | 400 | DLT template ID not found or not approved |
| `INVALID_SENDER_ID` | 400 | Sender ID not approved for this message class |
| `UNAUTHORIZED` | 401 | Missing or invalid auth token/API key |
| `FORBIDDEN` | 403 | Valid auth but insufficient permissions |
| `NOT_FOUND` | 404 | Requested resource does not exist |
| `DUPLICATE_REQUEST` | 409 | Idempotency-Key already used (returns original response) |
| `INSUFFICIENT_BALANCE` | 402 | Wallet balance below required amount |
| `RATE_LIMITED` | 429 | Too many requests — include Retry-After header |
| `ROUTE_UNAVAILABLE` | 503 | No healthy route available for this message class |
| `INTERNAL_ERROR` | 500 | Unexpected server error — request_id for support |

---

## OpenAPI

The API will be documented with OpenAPI 3.1 spec, auto-generated from FastAPI:

- **Docs page** at `/docs` (Swagger UI) and `/redoc` (ReDoc)
- **Spec file** at `/openapi.json`
- SDK generation from spec for Python, Node.js, PHP clients
- Contract tests generated from spec to prevent drift

---

## Anti-Patterns

- **Don't return money as numbers** — Always string. `"0.250000"` not `0.25`
- **Don't skip Idempotency-Key on creates** — Every POST that creates a resource must require it
- **Don't expose internal IDs** — Jasmin message IDs, DB sequence numbers stay internal
- **Don't return different error shapes** — Every error uses the standard envelope, even 500s
