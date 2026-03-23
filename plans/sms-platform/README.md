# SMS Platform

Multi-tenant SMS aggregation platform. Tier 3 architecture from day one, operating at Tier 2 cost.

## What This Is

Production-grade SMPP gateway that aggregates upstream SMS providers and exposes a clean HTTP API + SMPP interface to downstream clients. Start as a Tier 2 reseller, graduate to Tier 3 with direct telco connections.

## Documentation

| # | Document | Purpose |
|---|---|---|
| 01 | [Requirements](docs/01-requirements.md) | Functional and non-functional requirements |
| 02 | [Business Model](docs/02-business-model.md) | Revenue model, pricing, personas, scope |
| 03 | [System Architecture](docs/03-system-architecture.md) | C4 model, components, design patterns, flows |
| 04 | [Database Schema](docs/04-database-schema.md) | Full SQL schema, indexes, transaction rules |
| 05 | [API Specs](docs/05-api-specs.md) | Complete REST API contract with examples |
| 06 | [Routing Logic](docs/06-routing-logic.md) | Route selection strategies, circuit breaker, health scoring |
| 07 | [Compliance](docs/07-compliance.md) | DLT, DND/NDNC, consent, audit trail |
| 08 | [Infrastructure](docs/08-infrastructure.md) | Docker, deployment, CI/CD, monitoring |
| 09 | [Roadmap](docs/09-roadmap.md) | 6-phase plan with go/no-go gates |

## Architecture Decision Records

| ADR | Decision |
|---|---|
| [001](docs/adr/001-use-jasmin.md) | Use Jasmin SMS Gateway for SMPP transport |
| [002](docs/adr/002-use-fastapi.md) | Use FastAPI for HTTP API server |
| [003](docs/adr/003-event-driven.md) | Event-driven architecture via RabbitMQ |
| [004](docs/adr/004-shared-database.md) | Shared PostgreSQL (not microservice databases) |
| [005](docs/adr/005-prepaid-wallet.md) | Prepaid wallet with append-only ledger |

## Tech Stack

| Component | Technology |
|---|---|
| API Server | FastAPI (Python) |
| SMPP Engine | Jasmin SMS Gateway |
| Database | PostgreSQL 15+ |
| Cache | Redis |
| Queue | RabbitMQ |
| Frontend | Next.js (React) |
| Reverse Proxy | Nginx |
| Containers | Docker Compose → Kubernetes |

## Quick Start

```bash
cp .env.example .env
docker compose up -d
# API at http://localhost:8000
# Admin at http://localhost:3000
```

## Related

- [Engineering Playbook — SMSC Knowledge](../../smsc/) — Reusable telecom patterns
- [Engineering Playbook — Architecture](../../architecture/) — General architecture patterns
