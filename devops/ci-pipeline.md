# CI Pipeline (GitHub Actions)

## Pipeline Structure

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  backend:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
          cache: pip

      - name: Install dependencies
        run: pip install -r apps/api/requirements.txt -r apps/api/requirements-dev.txt

      - name: Lint
        run: ruff check apps/api/

      - name: Format check
        run: ruff format --check apps/api/

      - name: Type check
        run: mypy apps/api/
        continue-on-error: true  # Gradual adoption

      - name: Test with coverage
        run: pytest --cov=apps/api --cov-fail-under=60
        env:
          APP_ENV: development
          SUPABASE_URL: http://test
          SUPABASE_KEY: test-key

  frontend:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    defaults:
      run:
        working-directory: apps/web
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: apps/web/package-lock.json

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run build
      - run: npm audit --audit-level=high
        continue-on-error: true

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install pip-audit
      - run: pip-audit -r apps/api/requirements.txt
```

## Key Decisions

### `npm ci` not `npm install`
- `ci` installs from lockfile exactly — deterministic
- `install` may update lockfile — non-deterministic in CI

### Caching
- Python: `cache: pip` in setup-python
- Node: `cache: npm` with `cache-dependency-path` for monorepo

### Dummy Environment Variables
- CI tests don't need real credentials
- Set dummy values for required env vars
- `APP_ENV=development` enables dev mode shortcuts (e.g., admin auth bypass)

### `continue-on-error: true`
Use for checks you're gradually adopting:
- `mypy` (pre-existing type errors)
- `npm audit` (upstream vulnerabilities you can't fix)

Remove `continue-on-error` once the check is clean.

## Job Parallelism

Backend and frontend jobs run in parallel — they have no dependencies on each other.

```
┌─────────┐  ┌──────────┐  ┌──────────┐
│ backend │  │ frontend │  │ security │
│  lint   │  │   lint   │  │ pip-audit│
│  type   │  │   type   │  └──────────┘
│  test   │  │   build  │
└─────────┘  └──────────┘
```

## Adding New Checks

When adding a new tool to CI:
1. First run it locally with `continue-on-error: true`
2. Fix existing violations
3. Remove `continue-on-error` to enforce going forward
