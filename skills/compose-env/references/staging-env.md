# 스테이징 환경 구성

스테이징 환경을 위한 Docker Compose override 설정입니다.

## docker-compose.staging.yml

```yaml
# docker-compose.staging.yml
version: '3.8'

services:
  # ===================
  # Application
  # ===================
  app:
    build:
      dockerfile: docker/Dockerfile
      target: production
    image: ${DOCKER_REGISTRY}/${COMPOSE_PROJECT_NAME}:${APP_VERSION:-staging}
    environment:
      - APP_ENV=staging
      - DEBUG=true
      - LOG_LEVEL=INFO
    deploy:
      mode: replicated
      replicas: 1
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
    # 내부 네트워크만 (외부 접근은 nginx 통해)
    expose:
      - "8000"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ===================
  # Nginx Reverse Proxy
  # ===================
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/staging.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - app-network
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  # ===================
  # Database
  # ===================
  db:
    # 외부 포트 노출 없음
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G

  # ===================
  # Redis
  # ===================
  redis:
    # 외부 포트 노출 없음
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

## 스테이징 Nginx 설정

```nginx
# docker/nginx/staging.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 로깅
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # 기본 설정
    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 10M;

    # Gzip
    gzip on;
    gzip_types text/plain application/json application/javascript text/css;

    # Upstream
    upstream app {
        server app:8000;
    }

    # HTTP -> HTTPS 리다이렉트
    server {
        listen 80;
        server_name staging.example.com;
        return 301 https://$server_name$request_uri;
    }

    # HTTPS Server
    server {
        listen 443 ssl http2;
        server_name staging.example.com;

        ssl_certificate     /etc/nginx/ssl/staging.crt;
        ssl_certificate_key /etc/nginx/ssl/staging.key;
        ssl_protocols       TLSv1.2 TLSv1.3;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://app/health;
            access_log off;
        }
    }
}
```

## 스테이징 환경 변수

```bash
# .env.staging

# Application
COMPOSE_PROJECT_NAME=myapp_staging
APP_ENV=staging
APP_VERSION=staging
DEBUG=true
LOG_LEVEL=INFO

# Docker Registry
DOCKER_REGISTRY=registry.example.com

# Database
POSTGRES_DB=app_staging
POSTGRES_USER=staging_user
POSTGRES_PASSWORD=${STAGING_DB_PASSWORD}  # CI/CD에서 주입

# Redis
REDIS_URL=redis://redis:6379/0

# External Services
SMTP_HOST=smtp.example.com
SMTP_PORT=587

# CORS
CORS_ORIGINS=https://staging.example.com

# Monitoring
SENTRY_DSN=${STAGING_SENTRY_DSN}
```

## 실행 스크립트

```bash
#!/bin/bash
# scripts/staging.sh

set -e

# 환경 변수
ENV_FILE=".env.staging"
COMPOSE_FILES="-f docker-compose.yml -f docker-compose.staging.yml"

# 환경 변수 검증
required_vars=("STAGING_DB_PASSWORD" "STAGING_SENTRY_DSN")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "❌ Error: $var is not set"
        exit 1
    fi
done

case "$1" in
    deploy)
        echo "🚀 Deploying to staging..."
        docker compose --env-file $ENV_FILE $COMPOSE_FILES pull
        docker compose --env-file $ENV_FILE $COMPOSE_FILES up -d
        echo "✅ Staging deployment complete!"
        ;;
    rollback)
        echo "⏪ Rolling back..."
        docker compose --env-file $ENV_FILE $COMPOSE_FILES down
        # 이전 버전으로 롭백
        export APP_VERSION=${2:-previous}
        docker compose --env-file $ENV_FILE $COMPOSE_FILES up -d
        ;;
    status)
        docker compose --env-file $ENV_FILE $COMPOSE_FILES ps
        ;;
    logs)
        docker compose --env-file $ENV_FILE $COMPOSE_FILES logs -f ${2:-app}
        ;;
    *)
        echo "Usage: $0 {deploy|rollback|status|logs}"
        exit 1
        ;;
esac
```

## CI/CD 연동 예시

```yaml
# .github/workflows/staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging
        env:
          STAGING_DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
          STAGING_SENTRY_DSN: ${{ secrets.STAGING_SENTRY_DSN }}
        run: |
          ./scripts/staging.sh deploy
```
