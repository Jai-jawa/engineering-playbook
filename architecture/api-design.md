# API Design Patterns

## Project Layout (FastAPI)

```
apps/api/
├── main.py              # App factory, middleware, lifespan
├── core/
│   └── config.py        # pydantic-settings, env-based, singleton
├── db/
│   └── database.py      # Engine, session factory, get_db dependency
├── middleware/
│   └── admin_auth.py    # JWT validation (never touches DB)
├── routers/
│   ├── cart.py           # One file per domain
│   ├── orders.py
│   └── payments.py
├── schemas/              # Pydantic request/response models (if >3 models per domain)
├── services/
│   ├── pricing_engine/   # Business logic modules
│   └── email.py
└── tests/
    ├── conftest.py
    └── test_*.py
```

## Router Conventions

- One file per domain: `cart.py`, `orders.py`, `payments.py`
- Prefix matches domain: `APIRouter(prefix="/cart", tags=["Cart"])`
- Schemas inline if <4 models, else separate `schemas/` file
- All DB access via `Depends(get_db)` — never import session directly

## Request Validation

```python
from pydantic import BaseModel, Field

class AddToCartRequest(BaseModel):
    product_slug: str
    quantity: int = Field(gt=0)
    selected_options: list[int]
    session_id: str | None = None
```

- Use `Field(gt=0)` for numeric constraints — Pydantic returns 422 automatically
- Optional fields with defaults — never require what can be inferred
- Use `str | None` not `Optional[str]` (Python 3.10+)

## Response Patterns

- Use `response_model` for typed responses
- Disable docs in production: `docs_url="/docs" if env != "production" else None`
- Health endpoint: `GET /health` returning `{"status": "ok"}`

## Dependency Injection

```python
# In routers:
async def create_order(db: AsyncSession = Depends(get_db), admin: dict = Depends(require_admin)):

# In tests:
app.dependency_overrides[get_db] = lambda: mock_db
```

- Override in tests via `app.dependency_overrides`
- Auth middleware returns claims dict, never DB objects

## Lifespan Management

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown: close HTTP clients, flush caches
    await http_client.aclose()
```

- Close reusable HTTP clients on shutdown
- No global mutable state outside module-level caches with TTL

## Anti-patterns

- Business logic in router functions (put it in `services/`)
- Importing `db.session` directly instead of using `Depends(get_db)`
- Returning raw SQLAlchemy models (use Pydantic response models)
- DB queries in middleware
