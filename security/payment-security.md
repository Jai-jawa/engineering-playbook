# Payment Security

## Signature Verification (CRITICAL)

Every payment callback must verify the gateway's HMAC signature before updating order status.

### Razorpay Example

```python
import hmac
import hashlib

def verify_razorpay_signature(order_id: str, payment_id: str, signature: str, secret: str) -> bool:
    message = f"{order_id}|{payment_id}"
    expected = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Why:** Without signature verification, an attacker can send fake payment success callbacks to mark orders as paid without actually paying.

### Always use `hmac.compare_digest`
```python
# CORRECT — constant-time comparison (prevents timing attacks)
hmac.compare_digest(expected, received)

# WRONG — short-circuits on first mismatch (leaks information)
expected == received
```

## Currency Handling

```python
# Store in primary currency (INR)
amount_inr = Decimal("1250.00")

# Gateway expects smallest unit (paise)
amount_paise = int(amount_inr * 100)  # 125000

# Never float for conversion
amount_paise = int(Decimal("1250.00") * 100)  # Correct
amount_paise = int(1250.00 * 100)              # Risky
```

## Webhook Idempotency

Payment webhooks can fire multiple times. Handle this gracefully:

```python
async def handle_payment_webhook(payment_data: dict, db: AsyncSession):
    result = await db.execute(
        update(orders)
        .where(orders.c.id == payment_data["order_id"])
        .where(orders.c.status == "pending")  # Only transition from pending
        .returning(orders.c.id)
    )
    updated = result.fetchone()

    if not updated:
        # Already processed — return success, not error
        return {"status": "already_processed"}

    # First time: send confirmation email, update inventory, etc.
    await send_confirmation_email(payment_data["order_id"])
    return {"status": "confirmed"}
```

**Key patterns:**
- Use SQL WHERE condition: `status = 'pending'` — first update wins
- If no rows returned → already processed → return success (not error)
- Both client verification and webhook may fire — first one wins

## Order Status Flow

```
pending → confirmed → processing → shipped → delivered
pending → cancelled (customer cancels before payment)
confirmed → refunded (after payment)
```

- Only allow forward transitions (never confirmed → pending)
- Each transition should be an atomic DB operation
- Log all status changes for audit trail

## Anti-patterns

- Trusting payment status from client-side redirect (verify server-side)
- Using string comparison for HMAC (timing attack)
- Raising errors on duplicate webhooks (causes retries)
- Float math for currency conversion
- Skipping signature verification in development
