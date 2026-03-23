# ADR-001: Use Jasmin SMS Gateway for SMPP Transport

## Status
Accepted

## Context
We need SMPP connectivity to upstream SMS providers. Options:
1. **Build SMPP client from scratch** using smpplib or python-smpp
2. **Use Jasmin SMS Gateway** as SMPP transport layer
3. **Use a commercial SMPP gateway** (licensed)

## Decision
Use Jasmin SMS Gateway for SMPP transport. FastAPI handles business logic.

## Rationale
- Jasmin is battle-tested open source, handles SMPP 3.4 protocol complexity (bind management, PDU encoding, DLR correlation, enquire_link)
- Building SMPP from scratch adds 2-3 months of protocol work before any business value
- Clear separation: Jasmin = transport, FastAPI = business logic (billing, routing, auth)
- Jasmin integrates natively with RabbitMQ for async DLR processing
- Can replace Jasmin with direct SMPP library later if needed (transport is abstracted)

## Consequences
- **Good:** Fast time-to-market, proven SMPP implementation, focus on business logic
- **Good:** Jasmin's HTTP API allows FastAPI to submit messages without speaking SMPP directly
- **Bad:** Additional operational dependency (Jasmin + RabbitMQ)
- **Bad:** Jasmin's config can be opaque — document all connector configs
- **Risk:** If Jasmin project becomes unmaintained, we may need to replace it. Mitigated by abstracting the transport interface
