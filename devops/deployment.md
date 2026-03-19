# Deployment

## Deployment Flow

```yaml
name: Deploy
on:
  push:
    branches: [main]
    paths: [apps/api/**, docker-compose.yml, nginx/**]

concurrency:
  group: deploy-production
  cancel-in-progress: false  # Never cancel a deploy mid-way

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/your-project
            git pull origin main
            docker compose down
            docker compose up -d --build
            sleep 5
            curl -sf http://localhost:8000/health || exit 1
```

## Pre-Deployment Checklist

- [ ] CI passes on main (lint, type-check, tests)
- [ ] Environment variables set on server
- [ ] Database migrations applied (if schema changed)
- [ ] Docker images build successfully locally

## Nginx TLS Setup (Let's Encrypt)

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    server_name yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Initial cert setup
```bash
certbot certonly --webroot -w /var/www/certbot -d yourdomain.com
```

### Auto-renewal
```bash
# Cron job (runs twice daily)
0 0,12 * * * certbot renew --quiet && docker compose restart nginx
```

## Post-Deployment Verification

```bash
# Health check
curl -sf https://yourdomain.com/api/health

# Check logs
docker compose logs api --tail=50

# Check container status
docker compose ps
```

## Rollback

```bash
# Quick rollback to previous commit
ssh root@server "cd /opt/your-project && git checkout HEAD~1 && docker compose up -d --build"
```

For critical issues, revert the commit on main and let CI re-deploy.

## Anti-patterns

- Deploying without CI passing
- `cancel-in-progress: true` for deploys (can leave server in broken state)
- No post-deploy health check
- Manual server changes not tracked in git
- Deploying database migrations and app code simultaneously without coordination
