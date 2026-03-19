# Environment Configuration

## Configuration Pattern

Use pydantic-settings for type-safe, env-based configuration:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Required (no default — fails if missing)
    supabase_url: str
    supabase_key: str
    supabase_jwt_secret: str

    # Optional with defaults
    app_env: str = "development"
    allowed_origins: str = "http://localhost:3000"
    resend_api_key: str | None = None

    class Config:
        env_file = ".env"

settings = Settings()  # Module-level singleton
```

## Environment Files

```
.env                  # Local development (gitignored)
.env.example          # Template with dummy values (committed)
.env.production       # Production values (gitignored, lives on server)
.env.test             # Test overrides (optional)
```

### .env.example (committed)
```bash
# Required
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-anon-key
SUPABASE_JWT_SECRET=your-jwt-secret

# Optional
APP_ENV=development
ALLOWED_ORIGINS=http://localhost:3000
RESEND_API_KEY=

# Frontend
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

## Dev vs Production Behavior

```python
# Feature toggles based on APP_ENV
if settings.app_env == "development":
    # Relaxed auth: any authenticated user is admin
    # API docs enabled at /docs
    # Detailed error messages in responses
else:
    # Strict auth: check app_metadata.is_admin
    # API docs disabled
    # Generic error messages (no stack traces)
```

| Feature | Development | Production |
|---|---|---|
| Admin auth | Any authenticated user | app_metadata.is_admin only |
| API docs | /docs enabled | Disabled |
| Error detail | Full messages | Generic messages |
| CORS origins | localhost | Your domain only |
| Email sending | Skipped (logged) | Sent via Resend |

## Frontend Environment Variables

Next.js convention: `NEXT_PUBLIC_` prefix for client-accessible vars.

```bash
# Client-accessible (bundled into JS)
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key

# Server-only (never in NEXT_PUBLIC_)
SUPABASE_SERVICE_ROLE_KEY=your-service-key  # NEVER expose to client
```

**Rule:** Never put secrets in `NEXT_PUBLIC_*` variables. They are embedded in the JavaScript bundle and visible to anyone.

## Anti-patterns

- Hardcoding URLs or secrets in source code
- Committing `.env` files with real values
- Using `NEXT_PUBLIC_` for secrets
- Different config loading patterns across the app (use one Settings class)
- Checking `if DEBUG` with a boolean (use `app_env` string for clarity)
