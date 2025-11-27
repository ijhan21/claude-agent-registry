# 프로덕션 환경 구성

프로덕션 환경을 위한 Docker Compose override 설정입니다.

## docker-compose.prod.yml

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  # ===================
  # Application
  # ===================
  app:
    build:
      dockerfile: docker/Dockerfile
      target: production
    image: ${DOCKER_REGISTRY}/${COMPOSE_PROJECT_NAME}:${APP_VERSION}
    environment:
      - APP_ENV=production
      - DEBUG=false
      - LOG_LEVEL=WARNING
    deploy:
      mode: replicated
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '1'
          memory: 1G
    expose:
      - "8000"
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "10"
    secrets:
      - db_password
      - secret_key

  # ===================
  # Nginx
  # ===================
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/prod.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - nginx_cache:/var/cache/nginx
    depends_on:
      - app
    networks:
      - app-network
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M

  # ===================
  # Database
  # ===================
  db:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    command:
      - postgres
      - -c
      - max_connections=200
      - -c
      - shared_buffers=1GB
      - -c
      - effective_cache_size=3GB
    secrets:
      - db_password

  # ===================
  # Redis
  # ===================
  redis:
    command: >-
      redis-server 
      --appendonly yes 
      --maxmemory 1gb 
      --maxmemory-policy allkeys-lru
      --requirepass_file /run/secrets/redis_password
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1.5G
    secrets:
      - redis_password

# ===================
# Secrets
# ===================
secrets:
  db_password:
    external: true
  secret_key:
    external: true
  redis_password:
    external: true

# ===================
# Volumes
# ===================
volumes:
  nginx_cache:
```

## 프로덕션 Nginx 설정

```nginx
# docker/nginx/prod.conf
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 로깅
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" rt=$request_time';
    access_log /var/log/nginx/access.log main buffer=16k;

    # 성능 최적화
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 50M;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml application/xml+rss text/javascript;

    # Rate Limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    # Upstream (Load Balancing)
    upstream app {
        least_conn;
        server app:8000 weight=1 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # HTTP -> HTTPS
    server {
        listen 80;
        server_name example.com www.example.com;
        return 301 https://$server_name$request_uri;
    }

    # HTTPS Server
    server {
        listen 443 ssl http2;
        server_name example.com www.example.com;

        # SSL
        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_session_tickets off;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers off;

        # 보안 헤더
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # API
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            limit_conn conn 10;

            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Connection "";
            
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # Health Check (로깅 제외)
        location /health {
            proxy_pass http://app/health;
            access_log off;
        }

        # Static Files (if any)
        location /static/ {
            alias /var/www/static/;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

## Docker Secrets 사용

### Secret 생성

```bash
# Docker Swarm 모드
echo "super_secret_password" | docker secret create db_password -
echo "django-insecure-key-here" | docker secret create secret_key -
echo "redis_password_here" | docker secret create redis_password -
```

### 애플리케이션에서 사용

```python
# Python
def get_secret(secret_name):
    try:
        with open(f'/run/secrets/{secret_name}', 'r') as f:
            return f.read().strip()
    except FileNotFoundError:
        return os.environ.get(secret_name.upper())

DATABASE_PASSWORD = get_secret('db_password')
```

## 프로덕션 배포 스크립트

```bash
#!/bin/bash
# scripts/prod.sh

set -e

ENV_FILE=".env.prod"
COMPOSE_FILES="-f docker-compose.yml -f docker-compose.prod.yml"

# 필수 환경 변수 검증
required_vars=("DOCKER_REGISTRY" "APP_VERSION")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "❌ Error: $var is required"
        exit 1
    fi
done

case "$1" in
    deploy)
        echo "🚀 Deploying to production..."
        
        # 이미지 Pull
        docker compose --env-file $ENV_FILE $COMPOSE_FILES pull
        
        # Rolling Update
        docker compose --env-file $ENV_FILE $COMPOSE_FILES up -d --no-build
        
        # Health Check
        echo "⏳ Waiting for health check..."
        sleep 30
        
        if curl -sf http://localhost/health > /dev/null; then
            echo "✅ Production deployment successful!"
        else
            echo "❌ Health check failed, rolling back..."
            $0 rollback
            exit 1
        fi
        ;;
        
    rollback)
        echo "⏪ Rolling back to previous version..."
        docker compose --env-file $ENV_FILE $COMPOSE_FILES down
        export APP_VERSION="previous"
        docker compose --env-file $ENV_FILE $COMPOSE_FILES up -d
        ;;
        
    scale)
        replicas=${2:-2}
        echo "⚖️ Scaling app to $replicas replicas..."
        docker compose --env-file $ENV_FILE $COMPOSE_FILES up -d --scale app=$replicas
        ;;
        
    *)
        echo "Usage: $0 {deploy|rollback|scale [N]}"
        exit 1
        ;;
esac
```

## Zero-Downtime Deployment

```yaml
# docker-compose.prod.yml
services:
  app:
    deploy:
      update_config:
        parallelism: 1        # 한 번에 1개씩 업데이트
        delay: 10s            # 업데이트 간 대기 시간
        failure_action: rollback  # 실패 시 롭백
        order: start-first    # 새 컨테이너 먼저 시작
```
