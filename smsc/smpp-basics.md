# SMPP Protocol Basics

Reusable reference for any SMS platform project. Not project-specific.

## What is SMPP

Short Message Peer-to-Peer protocol. The standard for exchanging SMS between:
- **ESME** (External Short Messaging Entity) — your application
- **SMSC** (Short Message Service Centre) — telco's message center

Current standard: SMPP v3.4 (most widely deployed). SMPP v5.0 exists but rarely used.

## Key Concepts

### Bind Types

| Type | Direction | Use Case |
|---|---|---|
| Transmitter (TX) | Send only | One-way bulk sending |
| Receiver (RX) | Receive only | DLR reception, MO messages |
| Transceiver (TRX) | Both | Most common — send + receive DLR on same connection |

### PDU Types (Protocol Data Units)

| PDU | Purpose |
|---|---|
| `bind_transmitter` / `bind_receiver` / `bind_transceiver` | Establish connection |
| `submit_sm` | Send a message |
| `submit_sm_resp` | Acknowledgment with message_id |
| `deliver_sm` | Receive DLR or MO (mobile-originated) message |
| `enquire_link` | Heartbeat — keep connection alive |
| `unbind` | Graceful disconnect |

### Message Lifecycle

```
submit_sm → submit_sm_resp (message_id) → deliver_sm (DLR with status)
```

1. **Submit** — ESME sends `submit_sm` to SMSC
2. **Accept** — SMSC responds with `submit_sm_resp` containing `message_id`
3. **Deliver** — SMSC sends `deliver_sm` when final status known (delivered/failed/expired)

### DLR (Delivery Report) Statuses

| Status | Meaning |
|---|---|
| DELIVRD | Successfully delivered to handset |
| UNDELIV | Delivery failed (permanent) |
| EXPIRED | TTL exceeded, message discarded |
| REJECTD | Rejected by SMSC or operator |
| UNKNOWN | Status cannot be determined |
| ACCEPTD | Accepted by SMSC (intermediate, not final) |

**Important:** DLR format varies by vendor. Always store raw DLR payload and normalize into your own status enum.

### Encoding

| Encoding | Chars per segment | Character set |
|---|---|---|
| GSM 7-bit | 160 | Basic Latin, some symbols |
| UCS-2 (Unicode) | 70 | Full Unicode (Hindi, Arabic, emoji) |

**Multipart messages:** If content exceeds segment limit, it's split into multiple segments with UDH (User Data Header). Each segment counts as one SMS for billing.

### TPS (Transactions Per Second)

- Each SMPP bind has a TPS limit set by the provider
- Exceeding TPS → `ESME_RTHROTTLED` (0x00000058) error
- Multiple binds can be opened to increase throughput
- TPS is per-bind, not per-connection

---

## Common SMPP Error Codes

| Code | Name | Meaning |
|---|---|---|
| 0x00000000 | ESME_ROK | Success |
| 0x00000001 | ESME_RINVMSGLEN | Invalid message length |
| 0x00000008 | ESME_RSYSERR | System error |
| 0x00000014 | ESME_RINVDFTMSGID | Invalid default message ID |
| 0x00000045 | ESME_RINVDCS | Invalid data coding scheme |
| 0x00000058 | ESME_RTHROTTLED | Throttling — TPS exceeded |
| 0x00000061 | ESME_RINVSCHED | Invalid scheduled delivery time |

---

## Connection Management

### Heartbeat
- Send `enquire_link` every 30-60 seconds
- If no response within timeout → connection is dead
- Reconnect with exponential backoff (2s, 4s, 8s, 16s, max 60s)

### Bind Recovery
```
Connection lost
  → Wait backoff interval
  → Attempt rebind
  → If successful, resume submission
  → If failed, increment backoff, retry
  → After max retries, mark connector as DOWN, alert ops
```

### Window Size
- SMPP supports windowing — multiple unacknowledged `submit_sm` in flight
- Window size = number of pending requests before waiting for responses
- Typical: 10-50. Higher = more throughput, but risk of message loss if connection drops

---

## Vendor Integration Patterns

### What to negotiate with upstream providers
1. **TPS limit** — How many messages/second per bind
2. **Max binds** — How many concurrent connections allowed
3. **DLR format** — What fields they include, what statuses they use
4. **IP whitelist** — Which IPs can connect
5. **Sender ID rules** — Pre-registered headers, character limits
6. **Retry policy** — Do they retry failed deliveries? For how long?
7. **Billing model** — Per submit_sm? Per deliver_sm? Per DLR status?

### What varies between vendors (always abstract)
- DLR payload format (JSON vs text vs custom)
- Error code mapping
- Sender ID validation rules
- TPS enforcement behavior (reject vs queue)
- Connection timeout thresholds

---

## Anti-Patterns

- **Don't parse DLR by position** — Vendor formats change. Use regex or structured parsing
- **Don't assume DLR arrives** — Some vendors don't send DLR for all statuses. Implement TTL expiry
- **Don't ignore enquire_link** — Dead connections waste time and cause message loss
- **Don't share credentials across environments** — Separate system_id for dev/staging/prod
- **Don't trust vendor TPS claims** — Always verify with load test before production
