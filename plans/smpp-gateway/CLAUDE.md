# SMPP Gateway

Multi-tenant SMS aggregation platform. Read relevant docs before starting implementation.

**Rule:** Follow engineering-playbook standards. When this file conflicts with the playbook, this file wins.

## Quick Reference by Task

| Task | Read these files |
|---|---|
| SMS send flow | `docs/architecture.md`, `docs/api-specs.md` |
| Database changes | `docs/database-schema.md` |
| New API endpoint | `docs/api-specs.md`, `docs/architecture.md` |
| Billing / wallet | `docs/database-schema.md` (wallet tables + transaction rules) |
| Route / connector | `docs/architecture.md` (strategy pattern, circuit breaker) |
| DLT / compliance | `docs/compliance.md` |
| Infrastructure | `docs/infrastructure.md` |
| Planning / roadmap | `docs/roadmap.md` |

## Project Conventions

- **Python backend** — FastAPI, async, Pydantic models, ruff for linting, mypy for types
- **Frontend** — Next.js, TypeScript strict mode, ESLint
- **Database** — PostgreSQL 15+, all money as NUMERIC(18,6), UUIDs for primary keys
- **Testing** — pytest for backend, Jest for frontend. Test billing logic thoroughly
- **Docker** — All services in docker-compose.yml. Never hardcode connection strings

## Key Architecture Rules

1. **Wallet debit is atomic** — SELECT FOR UPDATE → verify balance → UPDATE wallet → INSERT transaction → INSERT message. Single DB transaction, no exceptions
2. **Append-only ledger** — Never UPDATE or DELETE wallet_transactions. Reversals are new rows
3. **Idempotency on all creates** — Every POST endpoint checks Idempotency-Key header
4. **DLT validation before dispatch** — Template must match message body before queueing
5. **FastAPI routes, Jasmin transports** — Business logic in FastAPI, SMPP protocol in Jasmin
6. **Money as strings in API** — All monetary values serialized as strings in JSON responses
7. **org_id on everything** — Every query filters by org_id. No cross-tenant data access

## Environment

- Copy `.env.example` to `.env` for local development
- Never commit `.env` files
- All secrets via environment variables, never in code
