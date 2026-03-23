# Business Model

## Positioning

- **What we are:** SMS aggregator providing reliable, cost-optimized message delivery
- **Who we serve:** Enterprises, SaaS platforms, and resellers who need programmatic SMS
- **How we earn:** Per-SMS margin between upstream cost and client rate

## Why This Model First

- Low barrier to entry — no telco relationship needed on day one
- Revenue from month one via reseller margin
- Proves technology and operational capability before approaching telcos
- Client base becomes leverage for direct telco negotiation

## Tier Progression

| Phase | Tier | Focus | Timeline |
|---|---|---|---|
| A | Tier 2 Launch | Reseller operations, 2 upstream vendors, HTTP API | Month 1-3 |
| B | Tier 2.5 Operational Maturity | Wallet billing, DLT compliance, routing intelligence | Month 3-6 |
| C | Tier 3 Maturity | Direct telco connections, SMPP server, HA deployment | Month 6-12 |
| D | Direct Telco Strategy | Telco partnerships, volume discounts, regional expansion | Month 12+ |

Long-term direction: Tier 2 (reseller) → Tier 2.5 (operational maturity) → Tier 3 (direct telco) → hybrid model with both direct and aggregated routes.

---

## Revenue Model

### Core Revenue
- Per-SMS margin between upstream cost and client rate
- Different margins by message class (transactional vs promotional vs OTP)

### Billing Model
- **Prepaid wallet** — clients top up before sending
- **Immutable ledger** — every transaction recorded with balance_before/balance_after
- **Rate cards** — per-org, per-route, per-message_class pricing

### Margin Structure (Example)

| Message Class | Buy Price | Sell Price | Margin | Margin % |
|---|---|---|---|---|
| OTP | ₹0.18 | ₹0.25 | ₹0.07 | 28% |
| Transactional | ₹0.15 | ₹0.22 | ₹0.07 | 32% |
| Promotional | ₹0.10 | ₹0.15 | ₹0.05 | 33% |

### Additional Revenue (Later)
1. Premium routes (faster delivery, higher reliability) at higher margin
2. Dedicated sender IDs
3. Bulk volume discounts (still profitable at scale)
4. API analytics and reporting add-ons
5. WhatsApp Business API (Phase 5+)
6. Number lookup / HLR services
7. Reseller platform fees

---

## User Personas

### 1. Enterprise Client
- **Needs:** Reliable SMS delivery, real-time DLR, simple API, usage dashboard
- **Pain points:** Unreliable delivery, opaque pricing, poor DLR accuracy
- **Typical volume:** 10K-500K SMS/month

### 2. Reseller
- **Needs:** Sub-org management, custom rate cards, white-label branding (later)
- **Pain points:** No visibility into delivery, can't manage their own clients
- **Typical volume:** 100K-5M SMS/month (aggregate of their clients)

### 3. Platform Admin
- **Needs:** Route management, connector health, financial oversight, client management
- **Pain points:** Manual route switching, no circuit breaker, spreadsheet billing

### 4. Operations / Support
- **Needs:** Message trace, delivery reports, client issue debugging
- **Pain points:** No end-to-end message tracking, fragmented logs

---

## Stakeholders

### Internal

| Role | Responsibility |
|---|---|
| Founder / Product Owner | Business direction, telco relationships, pricing strategy |
| Backend Developer | API, SMPP integration, billing logic, database |
| Frontend Developer | Client portal, admin dashboard |
| DevOps / Infrastructure | Docker, CI/CD, monitoring, scaling |

### External

| Party | Interaction |
|---|---|
| Upstream SMS Providers | SMPP/HTTP connections, rate negotiation |
| Telcos (future) | Direct connection agreements |
| TRAI / DOT | Regulatory compliance |
| DLT Platform | Template/sender ID registration |
| Clients | API integration, billing, support |

---

## Investment Breakdown

| Tier | Infrastructure | Development | Ongoing Ops |
|---|---|---|---|
| Tier 2 (Phase A) | Low (~₹4K-8K/mo VPS) | 1 developer, 3 months | Minimal |
| Tier 2.5 (Phase B) | Low-Medium | Same developer, +3 months | Monitoring setup |
| Tier 3 (Phase C) | Medium (~₹15K-40K/mo) | May need DevOps support | Dedicated ops |
| Direct Telco (Phase D) | High | Business development + engineering | Full ops team |

---

## Product Scope

### MVP (Phase 1-2)
- Single-SMS and bulk HTTP API
- JWT + API key authentication
- Prepaid wallet with ledger
- 2 upstream vendor connections
- Basic routing (priority-based)
- DLT template validation
- DLR tracking and webhooks
- Admin dashboard (routes, connectors, orgs, wallets)

### Later (Phase 3-4)
- Client-facing portal with reports
- SMPP server interface for downstream
- Scheduled messages
- Advanced routing (cost-optimized, geo-based)
- Circuit breaker auto-failover

### Non-Goals for MVP
- WhatsApp / RCS channels
- White-label reseller portal
- Direct telco connections
- Multi-region deployment
- Mobile app
