# 서비스 프로파일 활용

Docker Compose 프로파일을 사용하여 서비스를 선택적으로 실행하는 방법입니다.

## 프로파일 기본 개념

프로파일을 사용하면 특정 서비스를 조건부로 실행할 수 있습니다:

```yaml
services:
  app:
    # 항상 실행 (프로파일 없음)
    image: myapp

  debug-tools:
    # 'debug' 프로파일일 때만 실행
    profiles: ["debug"]
    image: debug-tools

  monitoring:
    # 'monitoring' 프로파일일 때만 실행
    profiles: ["monitoring"]
    image: prometheus
```

## 활용 예시

### 개발 도구 프로파일

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ===================
  # 항상 실행되는 서비스
  # ===================
  app:
    build: .
    ports:
      - "8000:8000"
    networks:
      - app-network

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    networks:
      - app-network

  # ===================
  # 개발 도구 (dev 프로파일)
  # ===================
  adminer:
    image: adminer:latest
    profiles: ["dev", "tools"]
    ports:
      - "8080:8080"
    networks:
      - app-network

  mailhog:
    image: mailhog/mailhog:latest
    profiles: ["dev"]
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - app-network

  redis-commander:
    image: rediscommander/redis-commander:latest
    profiles: ["dev", "tools"]
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379
    networks:
      - app-network

  # ===================
  # 모니터링 (monitoring 프로파일)
  # ===================
  prometheus:
    image: prom/prometheus:latest
    profiles: ["monitoring"]
    ports:
      - "9090:9090"
    volumes:
      - ./docker/prometheus:/etc/prometheus:ro
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    profiles: ["monitoring"]
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app-network

  # ===================
  # 테스트 (test 프로파일)
  # ===================
  test-db:
    image: postgres:16-alpine
    profiles: ["test"]
    environment:
      - POSTGRES_DB=test_db
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
  grafana_data:
```

## 프로파일 실행 방법

### 기본 실행 (프로파일 없는 서비스만)

```bash
# app, db, redis만 실행
docker compose up -d
```

### 특정 프로파일 포함 실행

```bash
# 개발 도구 포함
docker compose --profile dev up -d

# 모니터링 포함
docker compose --profile monitoring up -d

# 여러 프로파일 동시 실행
docker compose --profile dev --profile monitoring up -d
```

### 환경 변수로 프로파일 설정

```bash
# .env
COMPOSE_PROFILES=dev,monitoring

# 환경 변수로 설정하면 --profile 없이 실행 가능
docker compose up -d
```

## 환경별 프로파일 설정

### 로컬 환경

```bash
# .env.local
COMPOSE_PROFILES=dev
```

### 스테이징 환경

```bash
# .env.staging
COMPOSE_PROFILES=monitoring
```

### 프로덕션 환경

```bash
# .env.prod
COMPOSE_PROFILES=monitoring
# 프로덕션에서는 dev 도구 제외
```

## 실행 스크립트 통합

```bash
#!/bin/bash
# scripts/compose.sh

set -e

# 환경 설정
ENV=${1:-local}
shift || true

case "$ENV" in
    local)
        ENV_FILE=".env.local"
        PROFILES="--profile dev"
        ;;
    staging)
        ENV_FILE=".env.staging"
        PROFILES="--profile monitoring"
        ;;
    prod)
        ENV_FILE=".env.prod"
        PROFILES="--profile monitoring"
        ;;
    test)
        ENV_FILE=".env.test"
        PROFILES="--profile test"
        ;;
    *)
        echo "Usage: $0 {local|staging|prod|test} [compose args...]"
        exit 1
        ;;
esac

echo "🚀 Running with ENV=$ENV, profiles=$PROFILES"
docker compose --env-file "$ENV_FILE" $PROFILES "$@"
```

### 사용 예시

```bash
# 로컬 개발 환경 실행
./scripts/compose.sh local up -d

# 프로덕션 배포
./scripts/compose.sh prod up -d

# 테스트 실행
./scripts/compose.sh test run --rm app pytest
```
