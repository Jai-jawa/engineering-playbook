# Telecom Concepts for SMS Platforms

Reusable reference. Terminology and concepts that apply to any SMS aggregation business.

## Industry Tiers

| Tier | Description | Example |
|---|---|---|
| Tier 1 | Telco / Mobile Network Operator (MNO) | Airtel, Jio, Vodafone-Idea |
| Tier 2 | Aggregator / Reseller | Buys routes from Tier 1, resells to enterprises |
| Tier 3 | Large aggregator with direct telco connections | Has own SMSC, direct interconnects |

### Typical Path for a New Entrant
```
Start as Tier 2 (buy from aggregators)
  → Prove operational reliability
  → Build client base
  → Negotiate direct telco connections
  → Become Tier 3 (hybrid: direct + aggregated routes)
```

---

## Key Terminology

| Term | Definition |
|---|---|
| SMSC | Short Message Service Centre — telco infrastructure that stores and forwards SMS |
| ESME | External Short Messaging Entity — your application connecting via SMPP |
| MO | Mobile Originated — message sent FROM a mobile phone |
| MT | Mobile Terminated — message sent TO a mobile phone (what aggregators do) |
| DLR | Delivery Report — status notification from SMSC about message fate |
| TPS | Transactions Per Second — throughput limit per connection |
| PDU | Protocol Data Unit — a single SMPP message/command |
| Bind | SMPP connection establishment (like TCP handshake for SMPP) |
| MSISDN | Mobile Subscriber ISDN Number — the phone number in international format |
| MNP | Mobile Number Portability — numbers that moved between operators |
| HLR | Home Location Register — database that knows which operator owns a number |
| UDH | User Data Header — metadata for multipart/concatenated messages |
| DCS | Data Coding Scheme — specifies message encoding (GSM7, UCS2) |

---

## SMS Economics

### Cost Structure

```
Revenue per SMS = Sell price to client
Cost per SMS    = Buy price from upstream
Margin          = Revenue - Cost
```

### Pricing Factors

| Factor | Impact on Price |
|---|---|
| Message class | OTP > Transactional > Promotional |
| Volume | Higher volume = lower per-unit cost |
| Route type | Premium (faster/reliable) > Standard > Backup |
| Destination | Domestic vs international |
| Operator | Some operators charge more |
| Time of day | Off-peak may be cheaper on some routes |

### Billing Models

| Model | Description | Risk |
|---|---|---|
| Prepaid wallet | Client deposits first, balance decremented per SMS | Low risk for platform |
| Postpaid invoice | Bill monthly based on usage | Credit risk |
| Hybrid | Prepaid with credit limit | Balanced |

### What Gets Billed

| Event | Billable? | Notes |
|---|---|---|
| Message accepted | Debit estimated cost | Reserve funds immediately |
| Message delivered | Adjust to actual cost | May differ from estimate |
| Message failed | Refund or don't bill | Policy decision per platform |
| Message expired | Refund or don't bill | Policy decision |
| Multipart (3 segments) | Bill 3x | Each segment = one SMS |

---

## Message Delivery Lifecycle

```
1. Client submits via HTTP API
2. Platform validates (auth, balance, DLT, rate limit)
3. Platform selects route (strategy pattern)
4. Platform debits wallet (atomic transaction)
5. Message queued for SMPP dispatch
6. Worker submits via SMPP to upstream
7. Upstream returns message_id (submit_sm_resp)
8. Status: submitted
9. Upstream delivers to telco SMSC
10. SMSC delivers to handset
11. SMSC sends DLR back through SMPP
12. Platform receives DLR, normalizes status
13. Platform updates message status in DB
14. Platform sends webhook to client
15. Status: delivered / failed / expired
```

### Where Things Go Wrong

| Step | Failure Mode | Mitigation |
|---|---|---|
| 6 | SMPP connection down | Circuit breaker, failover route |
| 7 | Throttled (TPS exceeded) | Queue with backpressure, respect rate limits |
| 10 | Phone off / unreachable | TTL expiry, retry by upstream |
| 11 | DLR never arrives | TTL timeout job marks as expired |
| 14 | Client webhook down | Retry queue with exponential backoff |

---

## Capacity Planning

### TPS to Daily Volume

| TPS | Messages/Hour | Messages/Day (16h) |
|---|---|---|
| 10 | 36,000 | 576,000 |
| 50 | 180,000 | 2,880,000 |
| 100 | 360,000 | 5,760,000 |
| 500 | 1,800,000 | 28,800,000 |

### Infrastructure Sizing (Rules of Thumb)

| Component | Per 100 TPS |
|---|---|
| CPU (API) | 2 vCPU |
| RAM (API) | 2 GB |
| DB connections | 20 |
| Redis memory | 500 MB |
| RabbitMQ | 1 vCPU, 1 GB RAM |
| SMPP binds | 2-4 per vendor |
| Disk (messages/month) | ~10 GB at 100 TPS |

---

## Regulatory Landscape (India)

| Authority | Jurisdiction |
|---|---|
| TRAI | Telecom regulation, DLT mandate, DND rules |
| DOT | Telemarketer registration, UCC compliance |
| DLT Platforms | Template/sender ID registration (Jio, Airtel, Vi, BSNL) |

### Key Rules
- All commercial SMS must be DLT-registered (template + sender ID + entity)
- Promotional SMS: DND check required, 9AM-9PM only
- Transactional/OTP: No time restriction, no DND check
- Violations: Fines up to INR 25 lakh, telemarketer blacklisting

---

## Anti-Patterns

- **Don't ignore segment counting** — A 161-character GSM7 message = 2 segments = 2x cost
- **Don't bill on submit, refund on fail** — Prefer: bill on submit, adjust on DLR. Refunds create accounting complexity
- **Don't assume all vendors bill the same way** — Some bill on submit_sm, others on deliver_sm
- **Don't skip MNP lookups for premium routes** — Wrong operator routing wastes money
- **Don't treat SMS as simple** — It's a financial transaction wrapped in a telecom protocol
