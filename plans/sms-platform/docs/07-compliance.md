# Compliance

## Core Compliance Areas

1. DLT (Distributed Ledger Technology) registration
2. Telemarketer / DOT registration
3. DND/NDNC scrubbing rules
4. Content template matching
5. Consent management
6. Promotional time window enforcement
7. Audit trail
8. Data retention and privacy

---

## 1. DLT Registration

### Entity Registration
- Every organization must have a registered DLT entity ID
- Entity ID stored in `organizations.settings` JSONB and on each `dlt_templates` / `sender_ids` row
- Platform does not submit messages without a valid entity ID

### Template Approval
- All message templates must be registered on the DLT platform before use
- Templates stored locally with `approval_status`: pending → approved → expired
- System rejects messages referencing unapproved or expired templates
- Template text validated against actual message content before dispatch

### Sender ID Registration
- Every sender ID (header) must be registered and approved on DLT
- Stored with `usage_type` (transactional / promotional / OTP) — must match message class
- System rejects messages with mismatched sender_id type vs message_class

---

## 2. Telemarketer / DOT Registration

### Pre-Launch Requirements
- Register as telemarketer with DOT (Department of Telecommunications)
- Obtain UCC (Unsolicited Commercial Communication) registration
- Assign internal compliance contact with authority to suspend traffic

### Ongoing
- Maintain registration validity — renewals tracked in compliance calendar
- Report spam complaints within mandated timeframe

---

## 3. Scrubbing Rules

### Promotional Messages
- **DND/NDNC check required** — Query number against DND registry before sending
- **Implementation:** Redis cache of DND numbers, refreshed daily from TRAI data
- **Consent override:** If client has explicit consent (category-specific), promotional messages can be sent to DND numbers
- **Time window:** Promotional messages only between 09:00 and 21:00 IST (configurable via env)
- **Enforcement:** `PROMOTIONAL_DND_ENFORCEMENT=true` in environment — system rejects promotional messages to DND numbers without consent

### Transactional Messages
- **No DND check required** — but sender type must be transactional
- **Template class must match** — Transactional template on transactional route only
- **Route eligibility** — Message class checked against route's `message_class` field

### OTP Messages
- **No DND check, no time restriction**
- **Must use OTP-registered template and sender ID**
- **TTL enforcement** — OTP messages have shorter delivery timeout

---

## 4. Content Template Matching

### Validation Rules
- Every outbound message must reference a `dlt_template_id`
- System fetches template text, matches against message body using variable placeholder pattern (`{#var#}`)
- **Strict match** — Static text must match exactly; only variable positions may differ
- **Quarantine on mismatch** — Message rejected with `INVALID_TEMPLATE` error code, not silently dropped

### Implementation
```python
def validate_template_match(template_text: str, message_body: str) -> bool:
    """
    Replace {#var#} placeholders with regex wildcard,
    then match against actual message body.
    """
    pattern = re.escape(template_text)
    pattern = pattern.replace(r'\{\#var\#\}', '.+')
    return bool(re.fullmatch(pattern, message_body))
```

### Error Codes
- `INVALID_TEMPLATE` — Template ID not found or not approved
- `TEMPLATE_MISMATCH` — Message body doesn't match template pattern
- `TEMPLATE_EXPIRED` — Template past its validity date
- `SENDER_TEMPLATE_MISMATCH` — Sender ID usage_type doesn't match template type

---

## 5. Consent Management

### What We Track
- Consent is the **client's responsibility** to collect, but we enforce at the platform level
- Organizations must declare consent basis when onboarding
- For promotional messages to DND numbers, explicit consent record required

### Consent Record (Future — Phase 3)
| Field | Description |
|---|---|
| phone_number | The consenting number |
| org_id | Organization that collected consent |
| consent_source | How consent was obtained (web form, SMS opt-in, paper) |
| consent_timestamp | When consent was given |
| consent_channel | Channel consent applies to (SMS, WhatsApp, voice) |
| category | What type of messages consented to |
| opt_out_timestamp | When consent was revoked (if applicable) |

### Opt-Out Enforcement
- Process opt-out keywords (STOP, CANCEL, UNSUBSCRIBE) in inbound messages
- Immediately flag number as opted-out for that organization
- Reject future promotional messages to opted-out numbers
- Provide opt-out webhook to notify client of the event

---

## 6. Audit Trail

### What Gets Logged

| Category | Events Tracked |
|---|---|
| Template changes | Created, updated, status changed, expired |
| Sender ID changes | Registered, approved, rejected, revoked |
| Wallet adjustments | Every credit, debit, refund, adjustment with before/after |
| Message submission | Who submitted, from which API key, to which route |
| Route used | Which connector, route snapshot at submission time |
| Delivery outcome | Final status, DLR data, latency |
| Raw DLR | Original vendor DLR payload stored in `delivery_reports.raw_payload` |

### Implementation Notes
- Wallet audit is built into `wallet_transactions` table (append-only ledger)
- Message audit is built into `messages` table (route_snapshot, timestamps)
- DLR audit is built into `delivery_reports` table (raw_payload)
- Admin action audit via future `audit_logs` table (Phase 3)
- All timestamps in UTC with timezone

---

## 7. Operational Compliance Checklist

- [ ] DOT telemarketer registration completed
- [ ] DLT entity registered for the operating company
- [ ] DND/NDNC data source established (TRAI registry or provider)
- [ ] Redis cache for DND numbers operational and refreshed daily
- [ ] Promotional time window enforcement enabled (`PROMOTIONAL_WINDOW_START/END`)
- [ ] Template validation enabled (`DLT_ENFORCEMENT_ENABLED=true`)
- [ ] Backup and data retention policy documented
- [ ] Compliance contact assigned and documented
- [ ] Incident response procedure for spam complaints documented

---

## 8. Client Onboarding Compliance Checklist

Before activating a new organization:

- [ ] Client has valid DLT entity ID
- [ ] At least one approved sender ID registered
- [ ] At least one approved template registered
- [ ] Client acknowledges consent responsibility (signed terms)
- [ ] Rate card agreed and configured
- [ ] Webhook URL configured for DLR (recommended)

---

## Product Requirements Driven by Compliance

These architecture features exist **because** of regulatory requirements:

| Requirement | Architecture Feature |
|---|---|
| DLT template validation | `dlt_templates` table + validation middleware |
| DND scrubbing | Redis DND cache + promotional check in dispatch |
| Promotional time windows | Env-configurable window + scheduler check |
| Audit trail | Append-only wallet ledger, message snapshots, raw DLR storage |
| Sender ID controls | `sender_ids` table with approval workflow |
| Consent tracking | Per-org consent records (Phase 3) |
| Template expiry | `expires_at` on templates + validation check |
| Message traceability | `request_id` on every API call, `client_message_id` on messages |

---

## Anti-Patterns

- **Don't skip DLT validation "temporarily"** — Once off, it stays off. Build enforcement from day one
- **Don't cache DND data for >24 hours** — TRAI data updates daily, stale cache = compliance violation
- **Don't trust client-declared message_class** — Validate template type matches declared class
- **Don't log message content for debugging** — Log message_id and status, never the SMS body. Content in DB only, access-controlled
- **Don't allow promotional messages without DND check** — Even if the client says "all numbers opted in" — verify at platform level
