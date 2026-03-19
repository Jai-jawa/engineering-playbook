# Backend Testing (Pytest)

## Configuration (pyproject.toml)

```toml
[tool.pytest.ini_options]
testpaths = ["apps/api/tests"]
asyncio_mode = "auto"
markers = ["api", "auth", "unit"]

[tool.coverage.run]
source = ["apps/api"]
omit = ["*/tests/*", "*/__init__.py"]

[tool.coverage.report]
fail_under = 60
```

## conftest.py Setup

### Critical: Set env vars BEFORE importing app

```python
import os
os.environ.setdefault("SUPABASE_URL", "http://test")
os.environ.setdefault("SUPABASE_KEY", "test-key")
os.environ.setdefault("SUPABASE_JWT_SECRET", "test-secret")
os.environ.setdefault("APP_ENV", "development")

# NOW import the app
from apps.api.main import app
```

**Why:** pydantic-settings reads env vars at import time. If vars aren't set before import, Settings validation fails.

### Standard Fixtures

```python
@pytest.fixture
def mock_db():
    """AsyncMock of SQLAlchemy AsyncSession"""
    db = AsyncMock(spec=AsyncSession)
    db.execute = AsyncMock()
    db.commit = AsyncMock()
    db.rollback = AsyncMock()
    return db

@pytest.fixture
def mock_db_override(mock_db):
    """Override FastAPI's get_db dependency"""
    app.dependency_overrides[get_db] = lambda: mock_db
    yield mock_db
    app.dependency_overrides.clear()  # Always cleanup

@pytest.fixture
async def client(mock_db_override):
    """HTTP test client with mocked DB"""
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
```

### JWT Test Tokens

```python
@pytest.fixture
def admin_token():
    payload = {
        "sub": "test-admin-id",
        "email": "admin@test.com",
        "aud": "authenticated",
        "exp": datetime.utcnow() + timedelta(hours=1),
        "app_metadata": {"is_admin": True}
    }
    return jwt.encode(payload, os.environ["SUPABASE_JWT_SECRET"], algorithm="HS256")
```

## Test Patterns

### Naming Convention
- Files: `test_{domain}.py` (test_cart.py, test_payments.py)
- Functions: `test_{action}_{scenario}` (test_add_to_cart_missing_session)

### HTTP-Level Tests

```python
async def test_create_order_unauthenticated(client):
    response = await client.post("/orders/", json={"items": []})
    assert response.status_code == 401

async def test_create_order_success(client, admin_token):
    response = await client.post(
        "/orders/",
        json={"items": [{"product_slug": "business-cards", "quantity": 100}]},
        headers={"Authorization": f"Bearer {admin_token}"}
    )
    assert response.status_code == 201
```

### What to Test (Checklist)

- [ ] Happy path for each endpoint
- [ ] Missing/invalid auth (401, 403)
- [ ] Validation errors — missing fields, invalid values (400, 422)
- [ ] Edge cases: empty results, zero quantity, duplicate operations
- [ ] Business logic: pricing formulas, discount thresholds

### Mocking External Services

```python
# Mock email service (fire-and-forget)
with patch("apps.api.services.email.send_email", return_value=True):
    response = await client.post("/orders/confirm", ...)

# Mock payment gateway
with patch("apps.api.services.razorpay.verify_signature", return_value=True):
    response = await client.post("/payments/verify", ...)
```

## Running Tests

```bash
# All tests
pytest

# With coverage
pytest --cov=apps/api --cov-report=term-missing --cov-fail-under=60

# Specific marker
pytest -m auth

# Single file
pytest apps/api/tests/test_cart.py -v
```

## Anti-patterns

- Importing app before setting env vars (breaks pydantic-settings)
- Not clearing `dependency_overrides` after test (leaks mock to next test)
- Testing implementation details instead of HTTP behavior
- Sharing mutable state between tests
