# Data Flow & Trust Boundaries

## Core Principle: Never Trust the Client

The client can send IDs and quantities. The server calculates everything else.

```
Client sends:       product_slug, option_ids[], quantity
Server calculates:  unit_price, subtotal, tax, total
Server stores:      JSONB snapshot of options + calculated prices
```

**Why:** Prevents price manipulation attacks. A modified client could send `price: 0.01`.

## Request Flow

```
Browser → Next.js (SSR/API routes) → FastAPI → Supabase (PostgreSQL)
                                        ↓
                                   Pricing Engine
                                        ↓
                                   Payment Gateway
```

## Trust Boundaries

| Layer | Trusts | Validates |
|---|---|---|
| Browser | Nothing | Display only |
| Next.js middleware | Supabase JWT cookies | Session refresh |
| FastAPI router | Pydantic validation | Request shape, field constraints |
| Service layer | Router validation | Business rules, price calculation |
| Database | App layer | Foreign keys, constraints, RLS |

## Price Validation at Every Step

1. **Add to cart:** Server fetches option prices from DB, calculates total, stores snapshot
2. **Update quantity:** Server re-fetches prices, re-calculates, updates snapshot
3. **Checkout:** Server re-validates all cart items against current DB prices
4. **Payment callback:** Server verifies gateway signature (HMAC)

## Data Snapshots

Cart items and order items store a JSONB snapshot of selected options at time of action:

```json
{
  "formula_type": "perSheet",
  "options": [
    {"label": "Standard Paper", "price_inr": "2.50", "param_role": "PER_SHEET_COST"},
    {"label": "24 Cards/Sheet", "numeric_factor": "24", "param_role": "SHEET_DIVISOR"}
  ],
  "calculated_price": "125.00"
}
```

**Why:** Prices change over time. The snapshot preserves what the customer actually paid.

## Anti-patterns

- Client sends calculated price to server
- Server trusts client-provided option prices
- No snapshot — order references current prices (which may have changed)
- Skipping price re-validation at checkout
