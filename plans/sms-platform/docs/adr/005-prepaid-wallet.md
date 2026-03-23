# ADR-005: Prepaid Wallet with Append-Only Ledger

## Status
Accepted

## Context
Need a billing model for client SMS usage. Options:
1. **Prepaid wallet** — clients deposit funds, balance decremented per SMS
2. **Postpaid invoice** — bill monthly based on usage
3. **Credit system** — allow spending up to a limit, bill periodically

## Decision
Prepaid wallet with append-only transaction ledger. No postpaid in Phase 1.

## Rationale
- **Zero credit risk** — Platform never fronts money. Client pays before sending
- **Atomic billing** — Debit wallet in the same transaction as message creation. No "sent but not billed" edge cases
- **Append-only ledger** — Every transaction has balance_before and balance_after. The ledger is the source of truth, wallet.balance is a denormalized cache
- **Auditability** — Full transaction history, no gaps, no mutations
- **Simple reconciliation** — Sum of ledger entries must equal current balance

## Transaction Pattern

```sql
BEGIN;
SELECT balance FROM wallets WHERE org_id = $1 FOR UPDATE;  -- Lock row
-- App verifies: balance >= estimated_cost
UPDATE wallets SET balance = balance - $cost WHERE org_id = $1;
INSERT INTO wallet_transactions (org_id, tx_type, amount, balance_before, balance_after, ...);
INSERT INTO messages (...);
COMMIT;
```

## Consequences
- **Good:** No credit risk, simple accounting, full audit trail
- **Good:** Real-time balance visibility for clients
- **Good:** Row-level locking prevents double-spend race conditions
- **Bad:** Clients must top up before sending — friction for large enterprises
- **Bad:** Failed messages need refund logic (adds refund transaction to ledger)
- **Risk:** High-volume clients may exhaust balance during campaigns. Mitigated by low_balance_threshold notifications and optional credit_limit field (Phase 2)

## Rules
1. Never UPDATE wallet_transactions — insert a new refund/adjustment row instead
2. Never DELETE wallet_transactions — append only
3. Always use SELECT ... FOR UPDATE when debiting
4. All money as NUMERIC(18,6) — never float
5. Record both balance_before and balance_after on every transaction
