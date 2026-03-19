# Infrastructure Security

## CORS Configuration

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins,  # From env var, not hardcoded
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

- Origins from env var: `ALLOWED_ORIGINS=https://yourdomain.com,http://localhost:3000`
- Never use `allow_origins=["*"]` with `allow_credentials=True` in production
- Dev: allow localhost origins. Prod: only your domain.

## Nginx Security Headers

```nginx
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

| Header | Purpose |
|---|---|
| X-Content-Type-Options: nosniff | Prevent MIME-type sniffing |
| X-Frame-Options: DENY | Prevent clickjacking |
| HSTS | Force HTTPS for 1 year |
| Referrer-Policy | Limit referrer leakage |

## TLS Configuration

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
ssl_prefer_server_ciphers off;

ssl_certificate /etc/letsencrypt/live/yourdomain/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/yourdomain/privkey.pem;
```

- TLS 1.2 minimum (1.0 and 1.1 are deprecated)
- Let's Encrypt for free certificates with auto-renewal
- HTTP → HTTPS redirect for all traffic

## Secrets Management

### Environment Variables
```bash
# .env.example (committed — shows required vars, no real values)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-anon-key
RAZORPAY_KEY_SECRET=your-secret

# .env (gitignored — has real values)
SUPABASE_URL=https://actual-project.supabase.co
```

### Rules
- `.env` files are gitignored — never commit secrets
- `.env.example` is committed — documents required variables
- Production secrets set via CI/CD or server environment
- Never log secrets, even partially
- Rotate secrets on suspected exposure

## Docker Security

```yaml
# docker-compose.yml
services:
  api:
    env_file: .env.production  # Secrets from file, not inline
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

- Use `env_file` not inline `environment` for secrets
- Run as non-root user when possible
- Pin base image versions (not `latest`)
- Healthchecks for automatic restart on failure

## Anti-patterns

- Hardcoding CORS origins
- `allow_origins=["*"]` with credentials in production
- TLS 1.0/1.1 enabled
- Secrets in docker-compose.yml or Dockerfile
- Running containers as root
- No healthchecks (container stays up even when app crashes)
