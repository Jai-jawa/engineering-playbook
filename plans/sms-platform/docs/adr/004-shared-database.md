# ADR-004: Shared PostgreSQL (Not Microservice Databases)

## Status
Accepted

## Context
Could split data stores per service (billing DB, messaging DB, routing DB) or use a single shared PostgreSQL instance. Options:
1. **Single shared PostgreSQL** — all tables in one database
2. **Database per service** — separate instances for billing, messaging, etc.
3. **Hybrid** — shared DB now, split later

## Decision
Single shared PostgreSQL for all services. Split when we actually need it.

## Rationale
- **Billing atomicity** — Wallet debit + message creation MUST be in the same transaction. Cross-database transactions are fragile and complex
- **Simpler operations** — One database to backup, monitor, and maintain. Critical for a small team
- **Table partitioning** handles message volume without splitting databases
- **No distributed transaction complexity** — saga patterns add months of work for a problem we don't have yet
- At Tier 2 (<50 TPS), a single PostgreSQL handles all load comfortably

## Consequences
- **Good:** ACID guarantees for billing without distributed transactions
- **Good:** Simple operations, single backup, single connection pool
- **Good:** JOIN queries across domains (e.g., message + wallet + org) are trivial
- **Bad:** All services share connection pool limits
- **Bad:** Schema migrations affect all services
- **Risk:** At high scale, single DB becomes bottleneck. Mitigated by read replicas (Phase 3) and partitioning. Full split only if >500 TPS sustained
