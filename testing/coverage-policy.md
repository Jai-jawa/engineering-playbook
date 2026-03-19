# Coverage Policy

## Targets

| Layer | Minimum | Focus |
|---|---|---|
| Backend (Python) | 60% | Business logic, API endpoints, auth |
| Frontend (TypeScript) | 50% | Components, context providers, utils |

## What to Cover

### High Priority (must test)
- Pricing calculations and financial math
- Auth flows (login, admin gates, token validation)
- API endpoints (happy path + error cases)
- Cart operations (add, update, remove, merge)
- Payment verification (signature validation)

### Medium Priority (should test)
- Form validation logic
- Conditional UI rendering
- Error states and fallbacks
- Edge cases (empty lists, boundary values)

### Low Priority (test if time permits)
- Static pages and simple display components
- Config/setup files
- Utility functions with obvious behavior

## What to Exclude from Coverage

```toml
# pyproject.toml
[tool.coverage.run]
omit = [
  "*/tests/*",
  "*/__init__.py",
  "*/migrations/*",
  "*/config.py",
]
```

- Test files themselves
- Init files
- Migration files
- Config/settings (validated by pydantic)
- Third-party integration boilerplate

## Coverage in CI

```yaml
- name: Run tests with coverage
  run: pytest --cov=apps/api --cov-report=term-missing --cov-fail-under=60
```

- Fail the build if coverage drops below threshold
- Use `--cov-report=term-missing` to see uncovered lines
- Review coverage reports to find untested business logic

## Philosophy

Coverage is a floor, not a ceiling. 60% ensures critical paths are tested without forcing tests on boilerplate. Write tests that catch real bugs, not tests that inflate numbers.

**Good test:** Verifies pricing engine returns correct total for 500 business cards
**Bad test:** Asserts that a constructor sets a field (obvious, adds coverage but no safety)
