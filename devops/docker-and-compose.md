# Docker & Compose

## Docker Compose Structure

```yaml
services:
  api:
    build:
      context: ./apps/api
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    env_file: .env.production
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - /etc/letsencrypt:/etc/letsencrypt:ro
    depends_on:
      api:
        condition: service_healthy
    restart: unless-stopped
```

## Healthchecks

Every service needs a healthcheck:

```python
# FastAPI health endpoint
@app.get("/health")
async def health():
    return {"status": "ok", "env": settings.app_env}
```

**Why:** Docker restarts unhealthy containers. Without healthchecks, a crashed app process inside a running container stays broken forever.

### Healthcheck timing
- `interval: 30s` — check every 30 seconds
- `timeout: 10s` — fail if no response in 10 seconds
- `retries: 3` — restart after 3 consecutive failures
- `start_period: 30s` — grace period for slow startups

## Dockerfile Best Practices

```dockerfile
# Multi-stage build
FROM python:3.10-slim AS base
WORKDIR /app

# Install dependencies first (better caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Then copy app code
COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Layer caching
- Copy dependency files first, install, then copy source
- Source changes don't re-install dependencies
- Use `--no-cache-dir` to keep image smaller

## Environment Configuration

```yaml
# Development
env_file: .env

# Production
env_file: .env.production
```

- Never inline secrets in docker-compose.yml
- Use separate env files for dev and prod
- `.env*` files are gitignored (except `.env.example`)

## Volumes

```yaml
volumes:
  - ./nginx/default.conf:/etc/nginx/conf.d/default.conf  # Config mount
  - /etc/letsencrypt:/etc/letsencrypt:ro                  # Certs (read-only)
  - certbot-data:/var/www/certbot                         # ACME challenges
```

- `:ro` for read-only mounts (certs, configs)
- Named volumes for persistent data
- Bind mounts for config files

## Anti-patterns

- No healthchecks (container stays "healthy" when app is dead)
- `latest` tag for base images (non-reproducible builds)
- Copying everything before installing deps (breaks layer cache)
- Secrets in Dockerfile or docker-compose.yml
- Running as root (use `USER nonroot` when possible)
- `restart: always` (use `unless-stopped` — allows manual stops)
