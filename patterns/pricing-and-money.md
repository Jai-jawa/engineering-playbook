# Pricing & Money Handling

## Core Rule: Server-Side Price Calculation

```
Client sends:    product_slug, option_ids[], quantity
Server returns:  unit_price, subtotal, discount, tax, total
```

The client **never** sends a price. The server fetches option prices from the database and calculates everything.

**Why:** A modified client could send `price: 0.01`. Server-side calculation is the only defense.

## Decimal Math (CRITICAL)

```python
from decimal import Decimal

# CORRECT — string initialization
price = Decimal("10.50")
tax = price * Decimal("0.18")
total = (price + tax).quantize(Decimal("0.01"))

# WRONG — float initialization (silent precision loss)
price = Decimal(10.50)  # Actually Decimal('10.5000000000000007105...')
```

### Rules
- Initialize with strings: `Decimal("0.00")`, never `Decimal(0.00)`
- Quantize final results: `.quantize(Decimal("0.01"))` for currency
- Convert to float only at JSON/API boundary
- Database columns: `DECIMAL(10,2)` or `NUMERIC(10,2)`

### Why not float?
```python
>>> 0.1 + 0.2
0.30000000000000004

>>> Decimal("0.1") + Decimal("0.2")
Decimal('0.3')
```

Float errors compound across line items. One invoice with 50 items can be off by several rupees.

## Pricing Engine Pattern

### Formula Dispatch
```python
def calculate_price(product, options, quantity):
    parsed = parse_options(options)

    if product.formula_type == "perSheet":
        return per_sheet_calculate(parsed, quantity)
    elif product.formula_type == "multiplicative":
        return multiplicative_calculate(parsed, quantity)
    elif product.formula_type == "area":
        return area_calculate(parsed, quantity)
    else:
        raise ValueError(f"Unknown formula: {product.formula_type}")
```

### Formula Examples
- **perSheet:** `ceil(quantity / cards_per_sheet) * cost_per_sheet` (business cards)
- **multiplicative:** `base_price * multiplier * quantity` (banners)
- **area:** `rate_per_sqm * width * height * quantity` (boards)

### Discounts
```python
# Apply quantity discount
discount_pct = get_discount(quantity, product.quantity_discounts)
discounted = subtotal * (1 - discount_pct / 100)

# Enforce minimum price
final = max(discounted, product.min_price_inr)
```

## Price Snapshots

Store a JSONB snapshot of options and calculated price at time of cart add / order creation:

```json
{
  "formula_type": "perSheet",
  "options": [
    {"label": "Standard Paper", "price_inr": "2.50", "role": "PER_SHEET_COST"},
    {"label": "24 Cards/Sheet", "numeric_factor": "24", "role": "SHEET_DIVISOR"}
  ],
  "subtotal": "125.00",
  "discount_pct": "10",
  "total": "112.50"
}
```

**Why:** Prices change over time. The snapshot preserves what the customer saw and paid.

## Currency for Payment Gateways

```python
# Store in primary currency
amount_inr = Decimal("1250.00")

# Gateway expects smallest unit (paise for INR, cents for USD)
amount_paise = int(amount_inr * 100)  # 125000
```

## Anti-patterns

- Client sends price in API request
- `float` for money calculations
- `Decimal(10.50)` instead of `Decimal("10.50")`
- No price snapshot (order references current prices that may change)
- Skipping re-validation at checkout (prices may have changed since cart add)
