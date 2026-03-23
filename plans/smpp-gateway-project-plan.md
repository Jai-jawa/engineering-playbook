# Plan: Project Planning Workflow + SMPP Gateway Blueprint

## Context

The user wants to enter the bulk SMS aggregator business (Tier 2 → Tier 3). Two deliverables:
1. **Reusable project planning workflow** → `engineering-playbook` repo
2. **Full SMPP Gateway project blueprint** → new `smpp-gateway` repo

Build Tier 3 tech from day one, operate at Tier 2 cost. Plan incorporates cross-AI validation (Claude + ChatGPT + Gemini feedback merged).

---

## Repo Strategy

| Repo | Purpose | Branch |
|---|---|---|
| `engineering-playbook` | Reusable planning workflow template | `claude/project-planning-workflow-VMOVd` |
| `smpp-gateway` (NEW, local) | SMPP project blueprint docs → eventually code | `main` |

---

## Part A: engineering-playbook repo

### File 1: `patterns/project-planning-workflow.md`

**User has provided the final, definitive content** — a comprehensive document with:

- 6 phases, each with: Goal, Checklist, Templates, Deliverables, Anti-patterns
- **Phase 1: Discovery** — stakeholder interview template, business model canvas, user journey mapping, regulatory scan, assumptions/exclusions
- **Phase 2: Technical Architecture** — tech stack scoring criteria (10 dimensions), decision matrix template, component diagram, data flow mapping, ADR list
- **Phase 3: Data Model & API Contract** — schema-first rules, ER mapping template, API-first rules, versioning strategy, error format template with request_id
- **Phase 4: Infrastructure & Deployment** — environment topology, container strategy, CI/CD pipeline template (GitHub Actions), cost estimation table
- **Phase 5: Risk Assessment** — risk register template with triggers/owners, dependency audit checklist, kill criteria
- **Phase 6: Implementation Roadmap** — MVP definition rules, roadmap template table, go/no-go checklist (8 criteria)
- **5 Cross-AI Validation Checkpoints** — after discovery, after architecture, after schema+API, before infra finalization, final blind spot review. Each with "what to paste" and "what to ask"
- **Handoff to Implementation** — explicit file checklist before first coding ticket
- **Final Anti-patterns** — 8 items

This content will be used as-is (with minor formatting adjustments to match playbook style).

### File 2: `architecture/architecture-template.md` — Reusable Architecture Doc Template

The user shared a comprehensive ARCHITECTURE.md template with 11 sections. This goes into the playbook as a reusable template any project can fill in. Sections:

1. Project Structure (directory tree template)
2. High-Level System Diagram (C4-style)
3. Core Components (Frontend, Backend Services — repeatable blocks)
4. Data Stores (databases, caches, queues)
5. External Integrations / APIs
6. Deployment & Infrastructure
7. Security Considerations
8. Development & Testing Environment
9. Future Considerations / Roadmap
10. Project Identification (name, repo URL, contact, last updated)
11. Glossary / Acronyms

### File 3: Update `CLAUDE.md`

Add row:
```
| Project planning      | `patterns/project-planning-workflow.md` |
```
Remove SMPP Gateway row (project lives in separate repo).

### File 4: Update `README.md` — update descriptions to include new files

---

## Part B: NEW `smpp-gateway` repo (`~/smpp-gateway`)

### File 1: `README.md`

**User has provided the final, definitive content.** Comprehensive project overview with:

- Project goal (multi-tenant SMS aggregation, Tier 2 start, Tier 3 architecture from day one)
- Business model — positioning, "why this model first", long-term direction
- **4-phase tier progression**: A (Tier 2 Launch), B (Tier 2.5 Operational Maturity), C (Tier 3 Maturity), D (Direct Telco Strategy)
- Tech stack summary with **"Why This Stack"** justifications per technology
- Investment breakdown table by tier (low→very high)
- Key stakeholders — internal (8 roles) + external (6 parties)
- 4 user personas with specific needs: Enterprise Client, Reseller, Platform Admin, Operations/Support
- Revenue model — core (per-SMS margin), billing model (prepaid wallet, immutable ledger), additional revenue options (7 items)
- **Product scope** — explicit MVP vs Later vs Non-Goals for MVP
- Related files links

This content will be used as-is.

### File 2: `docs/architecture.md`

**User has provided the final, definitive content.** Comprehensive architecture doc with:

- High-level ASCII diagram (Clients → Nginx → FastAPI/Next.js → PostgreSQL/Redis/RabbitMQ → Jasmin → Upstream/Downstream)
- Component breakdown — 6 components with explicit responsibilities
- **6 design patterns**: Strategy (routing with RouteStrategy Protocol), Circuit Breaker (closed/open/half_open), Observer/Event-driven (6 event types), Multi-tenant shared infra, Repository pattern (4 named repos), **9-step Middleware Chain**
- **Primary SMS flow** — 16-step flow from client submit through DLR and webhook
- **DLR flow** — separate 8-step flow
- **Security architecture** — 4 subsections: API (key auth, RBAC, request signing later), SMPP (IP whitelist, per-connector creds, TPS limits), Wallet/Billing (decimal, append-only ledger, transactions, cost snapshot on message), Operational (secrets in env, admin isolation, encrypted backups)
- **Reliability model** — failure handling (idempotency keys, queue retries, circuit breaker), observability (11 tracked metrics)
- **Architecture decisions** — "why" rationale for Jasmin+FastAPI, shared DB, event-driven
- Anti-patterns (6 items)

This content will be used as-is.

### File 3: `docs/database-schema.md`

**User has provided the final, definitive content** — complete SQL CREATE TABLE statements with:

- **Schema principles** — multi-tenant by org_id, NUMERIC not float, ledger-based billing, partitioned messages, auditable status history
- **12 tables** with full column definitions, CHECK constraints, and UNIQUE constraints:
  - `organizations` — with `parent_organization_id` (reseller hierarchy), legal_name, billing_email, timezone, webhook_url/secret, status lifecycle
  - `users` — with 5 roles (super_admin/org_admin/operator/api_user/viewer), 3 statuses (active/invited/disabled), UNIQUE(org_id, email)
  - `wallets` — NUMERIC(18,6), low_balance_threshold, allow_negative flag
  - `wallet_transactions` — append-only ledger with balance_before/balance_after, 5 tx types, created_by_user_id
  - `rate_cards` — per-org, per-route, per-message_class, per-country_code, with effective_from/to and billing_unit
  - `smpp_connectors` — direction (upstream/downstream), bind_mode (tx/trx/rx), health_score, password_encrypted, metadata JSONB
  - `routes` — route_type strategy, message_class, country_code, sender_id_pattern, dlt_required flag, priority/weight, max_tps_override
  - `dlt_templates` — template_id, entity_id, template_text, approval_status, validity dates
  - `sender_ids` — entity_id, usage_type, approval_status
  - `api_keys` — key_prefix (fast lookup), key_hash, expires_at
  - `messages` (partitioned) — 8 statuses (queued→accepted→submitted→delivered/failed/rejected/expired/unknown), route_snapshot JSONB, segments_count, estimated_cost + billed_cost, fail_reason_code/message
  - `delivery_reports` — composite FK to partitioned messages, raw_payload JSONB, normalized_status
- **8 indexes** covering all major access patterns
- **Transaction rules** — wallet debit pattern (lock→verify→update→insert ledger→insert message), message acceptance pattern (idempotency, snapshot, cost estimate)
- **Future tables** — audit_logs, webhook_deliveries, scheduled_jobs, route_health_events, support_tickets
- Anti-patterns

This content will be used as-is. The init.sql will be generated from these CREATE statements.

### File 4: `docs/api-specs.md`

**User has provided the final, definitive content.** Full API contract with JSON examples for every endpoint:

- **API principles** — versioned /api/v1, money as strings, consistent errors, idempotency required
- **Auth** — POST login (JWT), POST api-key (returns key once, prefix stored for lookup)
- **SMS** — POST send (with Idempotency-Key header, client_message_id, route info in response), POST bulk (batch_id, accepted/rejected counts, schedule_at), GET status, GET reports (paginated, filtered)
- **Balance** — GET balance (with low_balance_threshold), GET transactions (with balance_after, reference links)
- **DLT** — GET/POST templates, GET sender-ids
- **Admin Connectors** — full CRUD with direction/bind_mode fields
- **Admin Routes** — full CRUD + POST test (validation_status, estimated_cost)
- **Admin Organizations** — full CRUD + POST credit (returns new_balance)
- **Admin Monitoring** — GET dashboard (live_tps, queue_depth, delivery_rate, route_health array), GET reports (revenue/cost/gross_margin)
- **Webhooks** — event-based payload with HMAC signature, client_message_id included
- **Error format** — structured `{ error: { code, message, details, request_id } }` with 11 starter error codes
- **OpenAPI 3.1** direction for docs page, SDK, contract tests
- Anti-patterns (4 items)

This content will be used as-is.

### File 5: `docs/infrastructure.md`

**User has provided the final, definitive content.** Comprehensive infra doc with:

- **Deployment philosophy** — Tier 3 shape at Tier 2 cost, explicit scale-up path
- **Docker Compose** — 8 services including separate **worker** process for async dispatch/DLR
- **Jasmin configuration** — routing split (FastAPI chooses route, Jasmin executes transport), DLR through structured worker path
- **Environment variables** — 5 groups with specific vars (API_KEY_PEPPER, WEBHOOK_SIGNING_SECRET, DLT_ENFORCEMENT_ENABLED, PROMOTIONAL_DND_ENFORCEMENT)
- **Deployment topology** — Phase 1 (single VPS), Phase 2 (separated/managed), Phase 3 (HA cluster with failover runbooks)
- **CI/CD** — 8 pipeline stages, minimum release controls (commit SHA tags, one-click rollback)
- **Monitoring** — 8+ metrics, 4 dashboard categories
- **Backup** — PostgreSQL daily + WAL, config in git, tested restores
- **Scaling plan** by component (DB, Workers, Jasmin, API)
- Anti-patterns (5 items)

This content will be used as-is.

### File 6: `docs/compliance.md`

**User has provided the final, definitive content.** Compliance doc with:

- 8 core compliance areas listed upfront
- **DLT registration** — entity, template approval, sender ID registration with approval tracking
- **Telemarketer/DOT registration** — pre-launch requirement with assigned compliance contact
- **Scrubbing rules** — promotional (DND/NDNC, consent) and transactional (sender type, template class, route eligibility)
- **Content template matching** — strict validation, quarantine mismatches, actionable error codes
- **Consent management** — source/timestamp/channel tracking, opt-out enforcement
- **Audit trail** — 7 tracked categories (template changes, sender changes, wallet adjustments, message submission, route used, outcome, raw DLR)
- **Operational checklist** (9 items) + **Client onboarding compliance checklist** (6 items)
- **Product requirements driven by compliance** — architecture features required by regulation
- Anti-patterns (5 items)

This content will be used as-is.

### File 7: `docs/roadmap.md`

**User has provided the final, definitive content.** Phased roadmap with:

- **Planning assumption** — launch fast without fragile shortcuts
- **6 phases** each with Goals, Deliverables, and Exit Criteria:
  - Phase 1 (Wk 1-3): Core infra — Docker, schema, auth, SMS API, 2 vendors
  - Phase 2 (Wk 3-6): Business logic — wallet/ledger, rate cards, DLT, routing, DLR
  - Phase 3 (Wk 6-9): Client portal — Next.js dashboard, reports, admin, API docs
  - Phase 4 (Wk 9-12): Scale — SMPP server, bulk scheduling, webhooks
  - Phase 5 (Mo 4-6): Growth — 3rd vendor, benchmarking, onboarding, WhatsApp planning
  - Phase 6 (Mo 6-12): Tier 3 maturity — telco outreach, HA, analytics
- **Dependency order** — 8-step chain
- **3 Go/No-Go gates** (Phase 1, 2, 3) with specific criteria
- **MVP summary** with explicit "not required" list
- Anti-patterns (4 items)

This content will be used as-is.

### File 8: `CLAUDE.md`

Project-specific Claude Code instructions referencing engineering-playbook + project docs.

---

## Implementation Order

### Step 1: engineering-playbook repo (branch: `claude/project-planning-workflow-VMOVd`)
1. Create `patterns/project-planning-workflow.md`
2. Update `CLAUDE.md` — add planning row
3. Update `README.md` — mention planning workflow
4. Commit and push to `claude/project-planning-workflow-VMOVd`

### Step 2: Create smpp-gateway repo
1. `mkdir ~/smpp-gateway && cd ~/smpp-gateway && git init`
2. Create `README.md` (project overview)
3. Create `CLAUDE.md` (project-specific Claude Code instructions)
4. Create `ARCHITECTURE.md` (filled-in version of the playbook template — all 11 sections populated for SMPP Gateway)
5. Create `docs/architecture.md` (detailed design patterns, data flows, security — complements ARCHITECTURE.md)
6. Create `docs/database-schema.md`
7. Create `docs/api-specs.md`
8. Create `docs/infrastructure.md`
9. Create `docs/compliance.md`
10. Create `docs/roadmap.md`
11. Create `docker-compose.yml` (ChatGPT's Phase 1 config with postgres init.sql mount)
12. Create `init.sql` (full schema — ChatGPT's base + our enhancements)
13. Create `.env.example` (template with all env vars)
14. Create `.gitignore` (ignore .env, __pycache__, node_modules, etc.)
15. Create empty `backend/` and `frontend/` directories with placeholder Dockerfiles
16. Commit all files

**Note:** smpp-gateway is local-only. User pushes to GitHub when ready.

### Schema Enhancements to Apply on ChatGPT's init.sql

The ChatGPT init.sql is a solid foundation. We will enhance it with our merged improvements:

| Enhancement | What to add |
|---|---|
| More granular msg_status | Change ENUM to: 'accepted', 'enroute', 'delivered', 'failed', 'expired', 'rejected' |
| api_keys table | New table: id, org_id, key_hash, name, is_active, created_at |
| organizations extras | Add: slug (UNIQUE), settings (JSONB), updated_at |
| users extras | Add: is_active, last_login_at, updated_at |
| wallets extras | Add: credit_limit DECIMAL(15,4) DEFAULT 0, CHECK(balance >= -credit_limit) |
| wallet_transactions extras | Add: balance_after DECIMAL(15,4), tx_type add 'adjustment' |
| routes extras | Add: name, route_type ENUM ('transactional','promotional','otp'), cost_per_sms, max_tps, circuit_breaker_state VARCHAR, failure_rate DECIMAL |
| rate_cards extras | Add: effective_from DATE, effective_to DATE (nullable) for time-bound pricing |
| messages extras | Add: user_id FK, encoding, jasmin_message_id, billed_amount, dlt_template_id FK, submit_time, delivery_time |
| sender_ids extras | Add: type ENUM ('transactional','promotional'), status ENUM ('pending','approved','rejected') replacing is_approved |
| dlt_templates extras | Add: template_type, status ENUM ('active','expired') replacing is_approved |
| connectors extras | Add: max_binds INT, jasmin_connector_id VARCHAR |
| delivery_reports extras | Add: error_code, operator_response TEXT |

---

## Verification

- All engineering-playbook files follow existing format (H1/H2/H3, checklists, tables, anti-patterns)
- SMPP docs incorporate best elements from both Claude and ChatGPT analysis
- Database schema includes all merged columns (api_keys table, credit_limit, circuit_breaker_state, encoding, effective dates)
- API specs have concrete JSON examples
- Compliance includes NDNC/DND Redis cache requirement
- Infrastructure includes Terraform/Ansible IaC and Kubernetes path
- engineering-playbook pushed to `claude/project-planning-workflow-VMOVd`
- smpp-gateway repo initialized with clean git history
