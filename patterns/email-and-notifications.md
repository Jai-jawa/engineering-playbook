# Email & Notifications

## Architecture

Email is a fire-and-forget operation. It should never block or break the main flow.

```python
async def send_order_confirmation(order_id: str, email: str) -> bool:
    """Returns True if sent, False if skipped/failed. Never raises."""
    if not settings.resend_api_key:
        logger.info(f"Email skipped (no API key): order {order_id}")
        return False

    try:
        await resend_client.send(
            from_email="orders@yourdomain.com",
            to=email,
            subject=f"Order #{order_id} Confirmed",
            html=render_template("order_confirmation", order_id=order_id),
        )
        return True
    except Exception as e:
        logger.error(f"Email failed for order {order_id}: {e}")
        return False
```

## Key Patterns

### Never block the main flow
```python
# Order confirmation flow
order = await create_order(db, request)  # Critical — must succeed
await db.commit()

# Email is non-critical — failure doesn't affect order
await send_order_confirmation(order.id, user.email)
# No try/except needed here — the function handles its own errors
```

### Graceful degradation without API key
```python
if not settings.resend_api_key:
    logger.info(f"Email skipped: {subject}")
    return False
```

In development, emails are logged but not sent. No email service configuration needed for local dev.

### Return bool, not raise
The email function returns `True`/`False` instead of raising exceptions. The caller doesn't need to handle email errors — it's informational only.

## Email Service Setup

### Resend (recommended for transactional email)
```python
import resend

resend.api_key = settings.resend_api_key

async def _send_email(to: str, subject: str, html: str) -> bool:
    try:
        resend.Emails.send({
            "from": "noreply@yourdomain.com",
            "to": to,
            "subject": subject,
            "html": html,
        })
        return True
    except Exception as e:
        logger.error(f"Email error: {e}")
        return False
```

## Email Types

| Type | Trigger | Priority |
|---|---|---|
| Order confirmation | Payment verified | Medium (fire-and-forget) |
| Shipping notification | Status → shipped | Medium |
| Password reset | User request | High (user-blocking) |
| Welcome email | Account creation | Low |

**Only password reset is user-blocking** — the user is waiting for it. All others are fire-and-forget.

## Anti-patterns

- Raising exceptions from email functions (blocks main flow)
- Sending email inside a database transaction
- Requiring email service for local development
- Synchronous email sending in async route handlers
- No logging when email is skipped or fails
