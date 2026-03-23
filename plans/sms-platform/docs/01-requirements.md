# Requirements

## Functional Requirements

### FR-1: SMS Submission
- FR-1.1: Accept single SMS via HTTP API (POST /api/v1/sms/send)
- FR-1.2: Accept bulk SMS via HTTP API (POST /api/v1/sms/bulk) — up to 1000 per request
- FR-1.3: Accept scheduled SMS with future delivery time
- FR-1.4: Validate phone number format (E.164)
- FR-1.5: Calculate segment count (GSM7: 160 chars, UCS2: 70 chars per segment)
- FR-1.6: Return message_id and estimated_cost in response
- FR-1.7: Support Idempotency-Key header to prevent duplicate submissions
- FR-1.8: Support client_message_id for client-side correlation

### FR-2: Message Delivery
- FR-2.1: Route messages through SMPP upstream vendors via Jasmin
- FR-2.2: Select route based on message_class, country, and routing strategy
- FR-2.3: Enforce per-connector TPS limits
- FR-2.4: Retry failed submissions (configurable, max 3 attempts)
- FR-2.5: Track message status through lifecycle (queued → accepted → submitted → delivered/failed/expired)

### FR-3: Delivery Reports (DLR)
- FR-3.1: Receive DLR from upstream via SMPP deliver_sm
- FR-3.2: Normalize vendor-specific DLR formats into standard statuses
- FR-3.3: Store raw DLR payload for audit
- FR-3.4: Update message status in database
- FR-3.5: Deliver webhook to client's callback URL (HMAC-signed)
- FR-3.6: Retry failed webhook delivery (3 attempts: 30s, 2min, 10min)
- FR-3.7: Mark messages as expired if no DLR received within TTL

### FR-4: Authentication & Authorization
- FR-4.1: JWT-based auth for admin/portal sessions (15-min expiry, refresh token)
- FR-4.2: API key auth for programmatic access (bcrypt hash + prefix lookup)
- FR-4.3: RBAC with 5 roles: super_admin, org_admin, operator, api_user, viewer
- FR-4.4: API keys scoped to organization
- FR-4.5: Permission-based access (sms:send, sms:read, balance:read, admin:*)

### FR-5: Billing
- FR-5.1: Prepaid wallet per organization
- FR-5.2: Atomic wallet debit on message submission (SELECT FOR UPDATE → debit → create message)
- FR-5.3: Append-only transaction ledger (credit, debit, refund, adjustment)
- FR-5.4: Per-org rate cards with route and message_class pricing
- FR-5.5: Estimated cost at submission, final cost at delivery
- FR-5.6: Low balance notification (webhook + email)
- FR-5.7: Admin manual credit with description

### FR-6: Routing
- FR-6.1: Priority-based route selection (Phase 1)
- FR-6.2: Cost-optimized routing (Phase 2)
- FR-6.3: Quality-first routing for OTP (Phase 2)
- FR-6.4: Circuit breaker per connector (closed → open → half_open)
- FR-6.5: Route health scoring from delivery data
- FR-6.6: Automatic failover when circuit opens

### FR-7: DLT Compliance
- FR-7.1: Template registration and approval tracking
- FR-7.2: Template content validation before dispatch (pattern matching)
- FR-7.3: Sender ID registration and approval tracking
- FR-7.4: Sender ID type must match message_class
- FR-7.5: DND/NDNC check for promotional messages (Redis lookup)
- FR-7.6: Promotional time window enforcement (9AM-9PM IST)
- FR-7.7: Configurable enforcement toggle per environment

### FR-8: Organization Management
- FR-8.1: Multi-tenant with org_id on every business table
- FR-8.2: Organization CRUD (admin only)
- FR-8.3: Reseller hierarchy via parent_organization_id
- FR-8.4: Per-org webhook URL and secret
- FR-8.5: Per-org settings (JSONB)
- FR-8.6: Organization status lifecycle (active, suspended, pending, closed)

### FR-9: Admin & Monitoring
- FR-9.1: Connector health dashboard (health_score, circuit_state, current TPS)
- FR-9.2: Queue depth monitoring (dispatch, DLR, webhook)
- FR-9.3: Financial reports (revenue, cost, margin by org/route/class)
- FR-9.4: Message reports (filterable by status, date, org, message_class)
- FR-9.5: Route management UI (CRUD, test route, toggle active)
- FR-9.6: Connector management UI (CRUD, bind status)

### FR-10: Client Portal
- FR-10.1: Client login and dashboard
- FR-10.2: Balance view and transaction history
- FR-10.3: Message history with delivery status
- FR-10.4: DLT template and sender ID management
- FR-10.5: API key management (create, revoke, view last used)
- FR-10.6: API documentation (Swagger/ReDoc)

---

## Non-Functional Requirements

### NFR-1: Performance
- NFR-1.1: API response time < 200ms for SMS submission (p95)
- NFR-1.2: API response time < 500ms for report queries (p95)
- NFR-1.3: DLR processing latency < 5 seconds from receipt to DB update
- NFR-1.4: Webhook delivery within 10 seconds of DLR receipt

### NFR-2: Throughput
- NFR-2.1: Phase 1 target: 50 TPS sustained
- NFR-2.2: Phase 2 target: 200 TPS sustained
- NFR-2.3: Phase 3 target: 1000 TPS sustained
- NFR-2.4: Bulk endpoint: process 1000 messages in < 5 seconds

### NFR-3: Availability
- NFR-3.1: 99.9% uptime for API (< 8.7 hours downtime/year)
- NFR-3.2: Zero message loss during normal operation
- NFR-3.3: Graceful degradation under overload (queue backpressure, not rejection)
- NFR-3.4: Recovery from single-node failure within 15 minutes (Phase 3)

### NFR-4: Security
- NFR-4.1: All external traffic over TLS 1.2+
- NFR-4.2: API keys stored as bcrypt hash (never plaintext)
- NFR-4.3: Secrets in environment variables only (never in code/config files)
- NFR-4.4: SQL injection prevention (parameterized queries via ORM)
- NFR-4.5: Rate limiting per API key (configurable per org)
- NFR-4.6: SMPP connections IP-whitelisted per connector
- NFR-4.7: Webhook signatures with HMAC-SHA256

### NFR-5: Data Integrity
- NFR-5.1: All billing operations ACID-compliant (single PostgreSQL transaction)
- NFR-5.2: Wallet balance never drifted from ledger sum (reconciliation check)
- NFR-5.3: No message submitted without corresponding wallet debit
- NFR-5.4: Append-only ledger — no UPDATE/DELETE on wallet_transactions
- NFR-5.5: All money stored as NUMERIC(18,6) — never float

### NFR-6: Observability
- NFR-6.1: Structured JSON logging with request_id
- NFR-6.2: Prometheus metrics for all critical paths
- NFR-6.3: Grafana dashboards for ops, business, and infrastructure
- NFR-6.4: Alerting on delivery rate drop, queue buildup, circuit breaker open
- NFR-6.5: End-to-end message tracing by message_id

### NFR-7: Compliance
- NFR-7.1: DLT enforcement configurable per environment
- NFR-7.2: Audit trail for all billing and admin operations
- NFR-7.3: Message content never logged (only message_id and metadata)
- NFR-7.4: Data retention: messages retained for 12 months, archived after
- NFR-7.5: GDPR-compatible data deletion capability (future)

### NFR-8: Operability
- NFR-8.1: All services containerized (Docker)
- NFR-8.2: Single-command local setup (docker compose up)
- NFR-8.3: CI pipeline: lint → typecheck → test → security scan → build
- NFR-8.4: Zero-downtime deployments (Phase 2+)
- NFR-8.5: One-click rollback via commit SHA image tags
- NFR-8.6: Automated database backup (daily full + WAL archiving)

---

## Constraints

- Must comply with TRAI/DOT regulations for commercial SMS in India
- Must support DLT template validation from launch
- Must use prepaid wallet model (no postpaid in Phase 1)
- Budget: single developer, $50-100/month infrastructure for Phase 1
- Timeline: MVP (first paying client) within 3 months

---

## Assumptions

- Upstream vendors provide SMPP 3.4 interface
- DLR is delivered for most messages (vendor SLA: >95% DLR rate)
- Initial traffic: <50 TPS, <10 client organizations
- Indian domestic SMS only (no international in Phase 1)
- Clients integrate via HTTP API (SMPP server for clients in Phase 4)
