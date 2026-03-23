# DLT Compliance (India)

Reusable DLT and telecom compliance reference for Indian SMS platforms.

## What is DLT

Distributed Ledger Technology — mandated by TRAI (Telecom Regulatory Authority of India) since 2021. Every commercial SMS in India must be registered on the DLT platform before sending.

**Purpose:** Reduce spam, enforce consent, enable traceability.

---

## Registration Hierarchy

```
Entity (Business) → Sender ID (Header) → Template (Content)
```

### 1. Entity Registration
- Register your business on a DLT platform (Jio, Airtel, Vodafone-Idea, BSNL, MTNL)
- Receive a unique **Entity ID** (also called Principal Entity ID)
- One entity can register on multiple DLT platforms
- Cross-platform templates: register template on one platform, usable across all

### 2. Sender ID (Header) Registration
- 6-character alphanumeric header (e.g., MYCOMP, HDBANK)
- Must be unique and not misleading
- Registered per usage type:
  - **Transactional** — Service messages (e.g., "Your order shipped")
  - **Promotional** — Marketing messages (e.g., "50% off sale")
  - **Service Implicit** — Messages to existing customers
  - **Service Explicit** — Messages with explicit consent

### 3. Template Registration
- Every message pattern must be registered as a template
- Variables marked with `{#var#}` placeholder
- Example: `Your OTP is {#var#}. Valid for {#var#} minutes.`
- Templates have types matching sender ID types
- Approval takes 1-3 business days

---

## Message Classification

| Class | DND Check | Time Restriction | Consent Required | Use Case |
|---|---|---|---|---|
| Transactional | No | None (24/7) | Implied by relationship | Order updates, OTPs, account alerts |
| Promotional | Yes | 9AM-9PM IST only | Explicit opt-in | Marketing, offers, campaigns |
| Service Implicit | No | None | Implied by existing relationship | Renewal reminders, policy updates |
| Service Explicit | No | None | Explicit consent on record | Feedback requests, surveys |
| OTP | No | None (24/7) | Transaction-triggered | One-time passwords |

---

## DND/NDNC Rules

### What is DND
Do Not Disturb — numbers registered by subscribers to block unsolicited messages.

### Scrubbing Rules
- **Promotional messages:** Must check against DND registry before sending
- **Transactional / OTP:** Exempt from DND check
- **Consent override:** If client has explicit, category-specific consent, promotional can be sent to DND numbers
- **Violation penalty:** Fines up to INR 25 lakh for repeated violations

### Implementation
```
1. Obtain DND number list from TRAI / operator
2. Load into Redis SET for O(1) lookup
3. Refresh daily (TRAI updates nightly)
4. For promotional: check Redis before dispatch
5. If DND and no consent → reject with clear error code
```

### Consent Categories (for DND override)
1. Banking/Insurance/Financial
2. Real Estate
3. Education
4. Health
5. Consumer Goods
6. Communication/Broadcasting
7. IT
8. Tourism

If subscriber blocks category 3 (Education), you cannot send promotional education messages even with general consent.

---

## Template Validation

### How It Works
```
Template: "Your OTP is {#var#}. Valid for {#var#} minutes."
Message:  "Your OTP is 123456. Valid for 5 minutes."
                        ^^^^^^              ^
                        variable            variable
```

### Validation Algorithm
```python
import re

def validate_template_match(template: str, message: str) -> bool:
    """
    Convert DLT template to regex pattern and match against message.
    Static text must match exactly. {#var#} accepts any non-empty content.
    """
    # Escape regex special chars in template
    pattern = re.escape(template)
    # Replace escaped placeholder with non-empty wildcard
    pattern = pattern.replace(r"\{\#var\#\}", ".+?")
    # Anchor to full string
    return bool(re.fullmatch(pattern, message))
```

### Edge Cases
- **Whitespace:** Some vendors normalize whitespace. Trim both before comparing
- **Unicode:** UCS-2 encoded messages may have different character representations
- **Line breaks:** Template may have `\n`, message may have actual newlines
- **Empty variables:** `{#var#}` must be non-empty (use `.+?` not `.*?`)

---

## Regulatory Timeline

| Date | Regulation |
|---|---|
| 2018 | TRAI mandates DLT for commercial messaging |
| 2021 Mar | DLT enforcement begins — unregistered messages blocked |
| 2021 Sep | Scrubbing rules tightened — all promotional must be DLT-compliant |
| 2023+ | Template content matching enforcement by some operators |
| Ongoing | Periodic tightening of enforcement, fines increasing |

---

## Platform Integration Checklist

- [ ] Store entity_id per organization
- [ ] Store template_id, entity_id, template_text per template
- [ ] Validate template approval status before dispatch
- [ ] Match message body against template pattern before dispatch
- [ ] Check sender_id usage_type matches message_class
- [ ] Check sender_id approval status
- [ ] Implement DND Redis cache with daily refresh
- [ ] Enforce promotional time window (configurable)
- [ ] Log template validation results for audit
- [ ] Handle template expiry (check expires_at)

---

## Anti-Patterns

- **Don't skip DLT in development** — Build validation from day one; retrofitting is painful
- **Don't cache DND data for >24 hours** — Stale data = compliance violation
- **Don't trust client-declared message class** — Validate template type matches declared class
- **Don't send promotional messages outside time window** — Even 1 minute before 9AM is a violation
- **Don't store consent as a boolean** — Track source, timestamp, category, and revocation
