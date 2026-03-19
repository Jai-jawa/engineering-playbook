# Error Handling Patterns

## HTTP Error Responses

Use specific status codes with human-readable messages:

```python
raise HTTPException(400, "Quantity must be greater than zero")
raise HTTPException(401, "Authentication required")
raise HTTPException(403, "Admin access required")
raise HTTPException(404, f"Product '{slug}' not found")
# 422 — returned automatically by Pydantic for validation errors
```

| Code | When |
|---|---|
| 400 | Invalid input the client can fix |
| 401 | Missing or invalid authentication |
| 403 | Authenticated but insufficient permissions |
| 404 | Resource not found |
| 422 | Request shape/type errors (Pydantic) |
| 503 | External dependency unavailable |

**Rule:** Error messages should help the client fix the problem. Never leak internal details (stack traces, SQL errors, file paths).

## Fire-and-Forget for Non-Critical Operations

Some operations should never block the main flow:

```python
# Critical: DB update (inside transaction)
async with db.begin():
    await db.execute(update_order_status)

# Non-critical: email (outside transaction, fire-and-forget)
try:
    await send_confirmation_email(order_id)
except Exception as e:
    logger.error(f"Email failed for order {order_id}: {e}")
    # Don't re-raise — order is still confirmed
```

### What is fire-and-forget
- Email sending
- Analytics/tracking events
- Cache invalidation
- Notification pushes

### What is NOT fire-and-forget
- Payment verification
- Database writes
- Price calculation
- Auth checks

## Graceful Fallbacks

When a non-critical operation fails, continue with degraded functionality:

```python
# Cart merge on login — failure doesn't block login
try:
    await merge_guest_cart(session_id, user_id)
except Exception as e:
    logger.warning(f"Cart merge failed: {e}")
    # User logs in fine, guest cart items stay as guest items

# JWKS fetch failure — fall back to stale cache
try:
    key = await fetch_jwks(kid)
except Exception:
    if cached_key:
        return cached_key  # Stale > nothing
    raise  # No cache at all — must fail
```

## Idempotency for Webhooks

Payment webhooks can fire multiple times. Handle gracefully:

```python
result = await db.execute(
    update(orders)
    .where(orders.c.id == order_id)
    .where(orders.c.status == "pending")  # Only process once
    .returning(orders.c.id)
)

if not result.fetchone():
    return {"status": "already_processed"}  # Success, not error
```

**Key:** Use SQL WHERE conditions to ensure only the first call takes effect. Return success on duplicates — returning an error would trigger retries.

## Validation Strategy

Three layers, each catching different problems:

1. **Pydantic** — request shape (automatic 422)
2. **Route handler** — business rules (manual 400)
3. **Database** — constraints as last safety net (unique, FK, check)

## Anti-patterns

- Catching all exceptions silently (`except: pass`)
- Leaking stack traces in production error responses
- Raising errors on duplicate webhook calls (causes infinite retries)
- Fire-and-forget for critical operations (payments, DB writes)
- Generic error messages that don't help the client ("Something went wrong")
