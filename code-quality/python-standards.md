# Python Standards

## Type Hints

- All function signatures must have type hints
- Use `str | None` not `Optional[str]` (Python 3.10+)
- Use `list[int]` not `List[int]` (lowercase generics)
- Run `mypy .` in CI (continue-on-error for gradual adoption)

## Pydantic v2

### Request Models
```python
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    product_slug: str
    quantity: int = Field(gt=0)
    selected_options: list[int]
    delivery_address_id: int | None = None
```

### Settings
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    supabase_url: str
    supabase_key: str
    app_env: str = "development"

    class Config:
        env_file = ".env"
```

- One Settings instance, imported everywhere
- All config from env vars — never hardcode secrets

## Decimal Math for Money (CRITICAL)

```python
from decimal import Decimal

# CORRECT
price = Decimal("10.50")
total = price * Decimal("3")
final = total.quantize(Decimal("0.01"))

# WRONG — float accumulates rounding errors
price = 10.50
total = price * 3  # Could be 31.499999999999996
```

- Initialize Decimal with strings: `Decimal("0.00")`
- Quantize final results: `.quantize(Decimal("0.01"))`
- Convert to float only at API/JSON boundary: `float(decimal_value)`

**Why:** Float math causes penny rounding errors on invoices. `0.1 + 0.2 != 0.3` in float.

## Linting & Formatting

### Ruff (pyproject.toml)
```toml
[tool.ruff]
line-length = 120
select = ["E", "F", "W", "I"]  # errors, pyflakes, warnings, isort
```

- `ruff check .` for linting
- `ruff format .` for formatting
- Run both in CI

## Async Patterns

- Use `async def` for all route handlers and DB operations
- Use `AsyncSession` from SQLAlchemy for DB
- Use `httpx.AsyncClient` for external HTTP calls (reusable, close on shutdown)
- Never mix sync and async DB operations

## Anti-patterns

- Using `float` for money calculations
- Initializing Decimal with float: `Decimal(10.50)` — use `Decimal("10.50")`
- Sync DB calls in async route handlers
- Creating new HTTP clients per request (use reusable client)
- Hardcoding config values instead of env vars
