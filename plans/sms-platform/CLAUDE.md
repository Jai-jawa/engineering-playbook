# SMS Platform

Multi-tenant SMS aggregation platform. Read relevant docs before starting implementation.

**Rule:** Follow engineering-playbook standards. When this file conflicts with the playbook, this file wins.

## Quick Reference by Task

| Task | Read these files |
|---|---|
| Requirements | `docs/01-requirements.md` |
| Business context | `docs/02-business-model.md` |
| System design | `docs/03-system-architecture.md` |
| Database changes | `docs/04-database-schema.md` |
| New API endpoint | `docs/05-api-specs.md`, `docs/03-system-architecture.md` |
| Routing / failover | `docs/06-routing-logic.md` |
| DLT / compliance | `docs/07-compliance.md` |
| Infrastructure | `docs/08-infrastructure.md` |
| Planning / roadmap | `docs/09-roadmap.md` |
| Architecture decisions | `docs/adr/` |
| SMPP protocol | `../../smsc/smpp-basics.md` |
| Routing patterns | `../../smsc/routing-strategies.md` |
| Telecom concepts | `../../smsc/telecom-concepts.md` |

## Project Conventions

- **Python backend** — FastAPI, async, Pydantic models, ruff for linting, mypy for types
- **Frontend** — Next.js, TypeScript strict mode, ESLint
- **Database** — PostgreSQL 15+, all money as NUMERIC(18,6), UUIDs for primary keys
- **Testing** — pytest for backend, Jest for frontend. Test billing logic thoroughly
- **Docker** — All services in docker-compose.yml. Never hardcode connection strings

## Key Architecture Rules

1. **Wallet debit is atomic** — SELECT FOR UPDATE → verify → UPDATE wallet → INSERT transaction → INSERT message. Single DB transaction
2. **Append-only ledger** — Never UPDATE or DELETE wallet_transactions. Reversals are new rows
3. **Idempotency on all creates** — Every POST endpoint checks Idempotency-Key header
4. **DLT validation before dispatch** — Template must match message body before queueing
5. **FastAPI routes, Jasmin transports** — Business logic in FastAPI, SMPP protocol in Jasmin
6. **Money as strings in API** — All monetary values serialized as strings in JSON responses
7. **org_id on everything** — Every query filters by org_id. No cross-tenant data access
8. **Strategy pattern for routing** — Never hardcode route selection logic
9. **Circuit breaker on connectors** — Never send to a failing route. Auto-recover after timeout

## Environment

- Copy `.env.example` to `.env` for local development
- Never commit `.env` files
- All secrets via environment variables, never in code
