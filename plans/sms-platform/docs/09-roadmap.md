# Implementation Roadmap

## Planning Assumption

Launch fast without fragile shortcuts. Every phase produces working software that can serve real traffic. No "refactor everything" phase — do it right incrementally.

---

## Phase 1 — Core Infrastructure (Week 1-3)

### Goals
- Docker environment running all services
- Database schema deployed with migrations
- Basic auth (JWT + API keys)
- Single SMS send endpoint working end-to-end
- 2 upstream vendor connections via Jasmin

### Deliverables
- [ ] Docker Compose with all 8 services running locally
- [ ] PostgreSQL schema (all 12 tables) via init.sql
- [ ] FastAPI project structure with auth middleware
- [ ] POST /api/v1/sms/send → Jasmin → upstream vendor → DLR back
- [ ] JWT login + API key creation endpoints
- [ ] 2 SMPP connectors configured in Jasmin
- [ ] Basic route selection (priority-based)
- [ ] Health check endpoint
- [ ] CI pipeline: lint + test + build

### Exit Criteria
- [ ] Can send an SMS via API and receive DLR callback
- [ ] Auth works (reject unauthorized, accept valid token/key)
- [ ] Docker Compose starts cleanly from scratch
- [ ] All tests pass in CI

---

## Phase 2 — Business Logic (Week 3-6)

### Goals
- Prepaid wallet with append-only ledger
- Rate card pricing per org/route/class
- DLT template validation
- Intelligent routing with circuit breaker
- DLR processing and webhook delivery

### Deliverables
- [ ] Wallet create/credit/debit with atomic transactions
- [ ] Balance check before SMS dispatch (SELECT ... FOR UPDATE)
- [ ] Rate cards: per-org, per-route, per-message_class pricing
- [ ] DLT template registration and validation middleware
- [ ] Sender ID registration and validation
- [ ] Circuit breaker on connectors (closed/open/half_open)
- [ ] Route selection strategy pattern (priority + cost-optimized)
- [ ] Worker: DLR processing → status update → webhook delivery
- [ ] Idempotency enforcement on all POST endpoints
- [ ] Bulk SMS endpoint (POST /api/v1/sms/bulk)

### Exit Criteria
- [ ] Wallet correctly debits on send, records balance_before/after
- [ ] Messages rejected when balance insufficient
- [ ] DLT validation rejects mismatched templates
- [ ] Circuit breaker opens after 5 consecutive failures, recovers after timeout
- [ ] Webhooks delivered to client URL with HMAC signature

---

## Phase 3 — Client Portal (Week 6-9)

### Goals
- Next.js dashboard for clients and admins
- Delivery reports and analytics
- Organization management UI
- API documentation page

### Deliverables
- [ ] Client dashboard: balance, recent messages, delivery stats
- [ ] Message reports: filterable by status, date, message_class
- [ ] Admin dashboard: connector health, route status, queue depth
- [ ] Admin: organization CRUD, wallet credit, user management
- [ ] Admin: connector and route management UI
- [ ] Admin: financial reports (revenue, cost, margin)
- [ ] Swagger/ReDoc API docs page
- [ ] Role-based access (admin sees everything, client sees own org)

### Exit Criteria
- [ ] Client can log in, see balance, view message history
- [ ] Admin can manage all resources via UI
- [ ] Financial reports match wallet transaction ledger
- [ ] All UI actions backed by existing API endpoints

---

## Phase 4 — Scale (Week 9-12)

### Goals
- SMPP server for downstream clients
- Scheduled/deferred messages
- Webhook retry system
- Performance optimization

### Deliverables
- [ ] SMPP server interface via Jasmin (downstream bind support)
- [ ] Scheduled messages: accept schedule_at, dispatch at specified time
- [ ] Webhook delivery queue with 3-attempt retry
- [ ] Message partitioning automation (monthly partition creation)
- [ ] Connection pooling and query optimization
- [ ] Load testing: establish baseline TPS per connector
- [ ] API rate limiting per org (configurable)

### Exit Criteria
- [ ] Downstream client can connect via SMPP and send messages
- [ ] Scheduled messages dispatch within 5 seconds of scheduled time
- [ ] System handles 100+ TPS sustained
- [ ] No message loss under load (verified by reconciliation)

---

## Phase 5 — Growth (Month 4-6)

### Goals
- Third vendor integration
- Client onboarding workflow
- Performance benchmarking
- WhatsApp Business API planning

### Deliverables
- [ ] 3rd upstream vendor connected and route-tested
- [ ] Client self-service onboarding: signup → entity setup → first message
- [ ] Benchmark suite: TPS, latency, delivery rate per route
- [ ] DND/NDNC Redis cache with daily TRAI data refresh
- [ ] Consent management (basic opt-out tracking)
- [ ] WhatsApp Business API technical evaluation document
- [ ] Cost optimization: auto-route based on delivery rate + cost

### Exit Criteria
- [ ] 3 active upstream routes with health monitoring
- [ ] New client can go from signup to first SMS in <30 minutes
- [ ] Published benchmark results per route
- [ ] DND enforcement operational for promotional messages

---

## Phase 6 — Tier 3 Maturity (Month 6-12)

### Goals
- Direct telco connections
- High availability deployment
- Advanced analytics
- Regional expansion readiness

### Deliverables
- [ ] At least 1 direct telco SMPP connection
- [ ] PostgreSQL primary/replica setup
- [ ] Multi-instance API and worker deployment
- [ ] Real-time analytics dashboard (TPS, delivery, revenue)
- [ ] Automated failover runbooks
- [ ] Infrastructure-as-code (Terraform/Ansible)
- [ ] International route support (country_code routing)
- [ ] Audit log system for all admin actions

### Exit Criteria
- [ ] Direct telco route delivering messages
- [ ] System recovers from single-node failure within 15 minutes
- [ ] Historical analytics available for >6 months of data
- [ ] Infrastructure reproducible from code

---

## Dependency Order

```
1. Docker + PostgreSQL schema
2. Auth (JWT + API keys)
3. SMS send endpoint + Jasmin integration
4. Wallet + billing
5. DLT + routing intelligence
6. DLR processing + webhooks
7. Client portal
8. SMPP server + scaling
```

Each step depends on the previous. Do not skip ahead.

---

## Go/No-Go Gates

### Gate 1 — After Phase 1 (Week 3)
- [ ] End-to-end SMS flow works (API → Jasmin → vendor → DLR → DB)
- [ ] Auth and security middleware operational
- [ ] CI/CD pipeline green
- [ ] **Decision:** Proceed to business logic or fix infrastructure gaps

### Gate 2 — After Phase 2 (Week 6)
- [ ] Billing is atomic and ledger is consistent
- [ ] DLT validation catches real violations
- [ ] Circuit breaker proven under simulated failure
- [ ] **Decision:** Proceed to UI or prioritize reliability

### Gate 3 — After Phase 3 (Week 9)
- [ ] First paying client can onboard and send messages
- [ ] Admin has full visibility into operations
- [ ] Financial reports balance correctly
- [ ] **Decision:** Proceed to scaling or acquire more clients first

---

## MVP Summary

### Required for First Client
- HTTP API (send, status, balance)
- JWT + API key auth
- Prepaid wallet with ledger
- 2 upstream routes
- DLT template validation
- DLR tracking
- Basic admin dashboard

### Not Required for MVP
- SMPP server (downstream)
- Scheduled messages
- Client self-service portal
- WhatsApp
- Advanced analytics
- Multi-region
- Direct telco connections

---

## Anti-Patterns

- **Don't build Phase 3 features during Phase 1** — No UI work until API is solid
- **Don't skip go/no-go gates** — If Phase 1 isn't clean, Phase 2 will be worse
- **Don't optimize before measuring** — Load test first, then optimize the bottleneck
- **Don't add vendors before routing is smart** — 3 dumb routes is worse than 2 smart ones
