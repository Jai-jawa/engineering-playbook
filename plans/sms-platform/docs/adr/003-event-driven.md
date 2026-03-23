# ADR-003: Event-Driven Architecture via RabbitMQ

## Status
Accepted

## Context
SMS delivery is inherently asynchronous — the API accepts a message, but delivery happens later (DLR arrives seconds to hours after submission). Options:
1. **Synchronous** — API waits for SMPP round-trip before responding
2. **Event-driven** — API queues the message, worker processes asynchronously
3. **Polling** — Worker polls database for pending messages

## Decision
Event-driven architecture with RabbitMQ as the message broker.

## Rationale
- API returns 202 immediately — client doesn't wait for SMPP round-trip (which can be 1-30 seconds)
- Worker can retry failed submissions without blocking the API
- Natural backpressure: queue depth grows when workers can't keep up, visible in monitoring
- DLR processing is inherently async (delivered minutes/hours later) — fits event model naturally
- RabbitMQ provides publisher confirms and consumer acks — no message loss
- Jasmin already uses RabbitMQ internally — reduces infrastructure complexity

## Consequences
- **Good:** API is fast (always 202), workers scale independently, natural backpressure
- **Good:** Separate queues per priority (OTP vs promotional) prevent slow messages blocking fast ones
- **Bad:** Adds operational complexity (RabbitMQ cluster management at scale)
- **Bad:** Debugging async flows is harder than synchronous — need good tracing with request_id
- **Risk:** Message loss if RabbitMQ crashes without persistence. Mitigated by durable queues + publisher confirms
