# Infrastructure

## Deployment Philosophy

Build with Tier 3 shape from day one, run at Tier 2 cost. Every component is containerized and can be scaled independently. Start on a single VPS, split when traffic demands it.

---

## Docker Compose — Phase 1

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - api
      - frontend

  api:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000
    env_file: .env
    depends_on:
      - postgres
      - redis
      - rabbitmq
    restart: unless-stopped

  worker:
    build: ./backend
    command: python -m app.worker
    env_file: .env
    depends_on:
      - postgres
      - redis
      - rabbitmq
      - jasmin
    restart: unless-stopped

  frontend:
    build: ./frontend
    env_file: .env
    depends_on:
      - api
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: smpp_gateway
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "127.0.0.1:5432:5432"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: ${RABBITMQ_USER}
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD}
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    ports:
      - "127.0.0.1:15672:15672"
    restart: unless-stopped

  jasmin:
    image: jookies/jasmin:latest
    ports:
      - "127.0.0.1:2775:2775"
      - "127.0.0.1:8990:8990"
      - "127.0.0.1:1401:1401"
    volumes:
      - ./jasmin/config:/etc/jasmin/store:rw
      - jasmin_logs:/var/log/jasmin
    depends_on:
      - rabbitmq
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  jasmin_logs:
```

---

## Jasmin Configuration

### Routing Split

- **FastAPI** decides which route and connector to use (business logic, cost optimization, circuit breaker)
- **Jasmin** executes the SMPP transport (bind management, PDU encoding, DLR reception)
- FastAPI submits to Jasmin via its HTTP API, specifying the connector ID

### DLR Path

```
Upstream → SMPP deliver_sm → Jasmin → RabbitMQ DLR queue → Worker → DB update + Webhook
```

Jasmin publishes raw DLR to a dedicated RabbitMQ queue. The worker normalizes vendor-specific DLR formats into standard statuses.

---

## Environment Variables

### `.env.example`

```bash
# Database
DB_HOST=postgres
DB_PORT=5432
DB_NAME=smpp_gateway
DB_USER=smpp_user
DB_PASSWORD=change_me_in_production

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=change_me_in_production

# RabbitMQ
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=smpp_user
RABBITMQ_PASSWORD=change_me_in_production

# API
API_SECRET_KEY=change_me_64_char_random_string
API_KEY_PEPPER=change_me_separate_from_secret_key
JWT_EXPIRY_MINUTES=15
JWT_REFRESH_EXPIRY_DAYS=7

# Webhooks
WEBHOOK_SIGNING_SECRET=change_me_per_environment
WEBHOOK_RETRY_MAX=3
WEBHOOK_RETRY_DELAYS=30,120,600

# Compliance
DLT_ENFORCEMENT_ENABLED=true
PROMOTIONAL_DND_ENFORCEMENT=true
PROMOTIONAL_WINDOW_START=09:00
PROMOTIONAL_WINDOW_END=21:00

# Jasmin
JASMIN_HOST=jasmin
JASMIN_HTTP_PORT=1401
JASMIN_CLI_PORT=8990

# Monitoring
LOG_LEVEL=INFO
SENTRY_DSN=
```

---

## Deployment Topology

### Phase 1 — Single VPS ($50-100/mo)

```
┌─────────────────────────────────┐
│         Single VPS (4GB+)       │
│                                 │
│  Docker Compose                 │
│  ├── nginx                      │
│  ├── api                        │
│  ├── worker                     │
│  ├── frontend                   │
│  ├── postgres                   │
│  ├── redis                      │
│  ├── rabbitmq                   │
│  └── jasmin                     │
│                                 │
│  Daily backup → remote storage  │
└─────────────────────────────────┘
```

- Minimum: 4 vCPU, 8GB RAM, 100GB SSD
- Suitable for: up to ~50 TPS, <10 clients
- Backup: automated pg_dump + WAL archiving to S3-compatible storage

### Phase 2 — Separated Services ($200-500/mo)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ App Server   │    │ DB Server    │    │ Queue Server │
│              │    │              │    │              │
│ nginx        │    │ PostgreSQL   │    │ RabbitMQ     │
│ api          │◄──►│ (managed or  │    │ Redis        │
│ worker       │    │  dedicated)  │    │ Jasmin       │
│ frontend     │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
```

- Consider managed PostgreSQL (AWS RDS, DigitalOcean)
- Separate Jasmin + queue server for SMPP stability
- Suitable for: up to ~200 TPS, <50 clients

### Phase 3 — HA Cluster ($500+/mo)

```
┌─────────┐     ┌─────────┐
│ API x2  │     │ API x2  │    (behind load balancer)
└────┬────┘     └────┬────┘
     │               │
┌────▼───────────────▼────┐
│   Load Balancer         │
└────────────┬────────────┘
             │
     ┌───────┼───────┐
     │       │       │
┌────▼──┐ ┌─▼────┐ ┌▼──────┐
│Worker │ │Worker│ │Worker │   (auto-scaled)
│  x3   │ │  x3  │ │  x3   │
└───────┘ └──────┘ └───────┘
             │
     ┌───────┼───────┐
     │       │       │
┌────▼──┐ ┌─▼────┐ ┌▼──────┐
│PG Pri │ │PG Rep│ │Jasmin │
│(write)│ │(read)│ │  x2   │
└───────┘ └──────┘ └───────┘
```

- PostgreSQL primary/replica
- Multiple Jasmin instances for redundancy
- Worker auto-scaling based on queue depth
- Failover runbooks documented

---

## CI/CD Pipeline

### Pipeline Stages (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    # ruff (Python), eslint (Next.js)

  type-check:
    # mypy (Python), tsc --noEmit (Next.js)

  test:
    # pytest with PostgreSQL service container
    # Jest for frontend

  security-scan:
    # bandit (Python), npm audit (Node.js)
    # trivy for Docker image scanning

  build:
    # Docker build for api, worker, frontend
    # Tag with commit SHA

  deploy-staging:
    # Deploy to staging on develop branch push
    needs: [lint, type-check, test, security-scan, build]

  deploy-production:
    # Deploy to production on main branch push (manual approval)
    needs: [deploy-staging]
```

### Release Controls

- **Commit SHA tags** — Every image tagged with git SHA, not `latest`
- **One-click rollback** — Previous SHA always available for instant revert
- **Staging gate** — Production deploys only after staging verification
- **Manual approval** — Production deploy requires explicit approval

---

## Monitoring

### Metrics (Prometheus)

| Category | Metrics |
|---|---|
| SMS Throughput | messages_sent_total, messages_per_second, messages_by_status |
| Delivery | delivery_rate_percentage, delivery_latency_seconds |
| Billing | wallet_debits_total, wallet_balance_gauge, revenue_total |
| Infrastructure | api_request_duration, queue_depth, db_connection_pool_usage |

### Dashboards (Grafana)

1. **Operations** — Live TPS, queue depth, connector status, error rates
2. **Business** — Revenue, cost, margin, messages by org
3. **Delivery** — Per-route delivery rates, latency distribution
4. **Infrastructure** — CPU, memory, disk, DB connections, Redis memory

### Alerting

| Alert | Threshold | Channel |
|---|---|---|
| Delivery rate drop | < 90% over 15 min | Slack + Email |
| Queue depth spike | > 1000 messages | Slack |
| Circuit breaker open | Any connector | Slack + Email |
| Wallet balance low | Below org threshold | Webhook to client |
| API error rate | > 5% 5xx over 5 min | Slack + PagerDuty |
| DB connection exhaustion | > 80% pool used | Slack |

---

## Backup Strategy

### PostgreSQL
- **Daily full backup** — pg_dump compressed, uploaded to S3-compatible storage
- **WAL archiving** — Continuous WAL shipping for point-in-time recovery
- **Retention** — 30 days daily, 12 months monthly
- **Tested restores** — Monthly restore drill to verify backup integrity

### Configuration
- All config in git (docker-compose, nginx, Jasmin config)
- Environment secrets in `.env` (not in git) — documented in `.env.example`
- Infrastructure-as-code when moving to Phase 3 (Terraform/Ansible)

### Disaster Recovery
- **RTO target** — 1 hour (Phase 1), 15 minutes (Phase 3)
- **RPO target** — 5 minutes (WAL archiving), 0 (Phase 3 with streaming replication)

---

## Scaling Plan

| Component | Phase 1 | Phase 2 | Phase 3 |
|---|---|---|---|
| API | Single instance | 2 instances + LB | Auto-scaled |
| Worker | Single instance | 2-3 instances | Auto-scaled by queue depth |
| Jasmin | Single instance | Dedicated server | 2+ instances, sticky routing |
| PostgreSQL | Docker container | Managed DB | Primary + read replicas |
| Redis | Docker container | Managed Redis | Cluster mode |
| RabbitMQ | Docker container | Dedicated server | Cluster with mirrored queues |

---

## Anti-Patterns

- **Don't use `latest` tag in production** — Always tag images with commit SHA
- **Don't put secrets in docker-compose.yml** — Use `.env` file, never commit secrets
- **Don't skip staging** — Every production deploy goes through staging first
- **Don't run database migrations in application startup** — Use explicit migration command in CI/CD
- **Don't ignore backup testing** — An untested backup is not a backup
