# Server-Side Validation

## Core Rule

**Never trust client input for anything that affects money, access, or data integrity.**

The client sends identifiers and quantities. The server looks up prices, calculates totals, and enforces business rules.

## Input Validation Layers

### Layer 1: Pydantic (request shape)
```python
class AddToCartRequest(BaseModel):
    product_slug: str
    quantity: int = Field(gt=0)
    selected_options: list[int]  # Option VALUE IDs, not prices
```

Pydantic automatically returns 422 for malformed requests.

### Layer 2: Business Logic (route handler / service)
```python
# Server fetches prices from DB — never from request
options = await db.execute(select(OptionValue).where(OptionValue.id.in_(request.selected_options)))
if not options:
    raise HTTPException(400, "Invalid options selected")

# Server calculates price
price = pricing_engine.calculate(product, options, request.quantity)
```

### Layer 3: Database Constraints (last safety net)
- Foreign key constraints prevent orphaned references
- CHECK constraints enforce valid ranges
- UNIQUE constraints prevent duplicates
- NOT NULL prevents missing required data

## What Clients Can Send

| Allowed | NOT Allowed |
|---|---|
| Product slug/ID | Calculated price |
| Option IDs | Option prices |
| Quantity | Discount amount |
| Delivery address ID | Total amount |
| Session ID | Tax amount |

## Price Manipulation Prevention

```
Client: "Add 100 business cards with options [3, 7, 12]"
Server: Fetches option 3 → ₹2.50/sheet, option 7 → 24 cards/sheet, option 12 → lamination ₹0.50
Server: Calculates → ceil(100/24) × ₹2.50 + 100 × ₹0.50 = ₹62.50
Server: Stores snapshot with calculated price
```

**Re-validate at every mutation:** add to cart, update quantity, checkout.

## Anti-patterns

- Accepting `price` or `total` in API request body
- Trusting client-calculated discounts
- Validating on client only (client validation is for UX, server validation is for security)
- Using string concatenation in SQL (use ORM / parameterized queries)
