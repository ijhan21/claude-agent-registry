# 기본 Docker Compose 구성

`docker-compose.yml`은 모든 환경에서 공통으로 사용되는 기본 설정을 정의합니다.

## 기본 구조

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ===================
  # Application Service
  # ===================
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      args:
        - PYTHON_VERSION=${PYTHON_VERSION:-3.12}
    image: ${COMPOSE_PROJECT_NAME:-myapp}:${APP_VERSION:-latest}
    env_file:
      - .env
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ===================
  # Database Service
  # ===================
  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

  # ===================
  # Redis Service
  # ===================
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

# ===================
# Networks
# ===================
networks:
  app-network:
    driver: bridge
    name: ${COMPOSE_PROJECT_NAME:-myapp}_network

# ===================
# Volumes
# ===================
volumes:
  postgres_data:
    name: ${COMPOSE_PROJECT_NAME:-myapp}_postgres_data
  redis_data:
    name: ${COMPOSE_PROJECT_NAME:-myapp}_redis_data
```

## 주요 설계 원칙

### 1. 환경 변수 참조

하드코딩 대신 환경 변수 사용:

```yaml
# ❌ 하드코딩
environment:
  - DATABASE_URL=postgresql://user:pass@db:5432/mydb

# ✅ 환경 변수 참조
environment:
  - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}

# ✅ 또는 env_file 사용
env_file:
  - .env
```

### 2. 기본값 설정

환경 변수에 기본값 제공:

```yaml
image: ${COMPOSE_PROJECT_NAME:-myapp}:${APP_VERSION:-latest}
# COMPOSE_PROJECT_NAME이 없으면 "myapp" 사용
# APP_VERSION이 없으면 "latest" 사용
```

### 3. 헬스체크 필수

모든 서비스에 헬스체크 설정:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
  interval: 30s      # 검사 간격
  timeout: 10s       # 타임아웃
  retries: 3         # 실패 허용 횟수
  start_period: 40s  # 초기 시작 대기 시간
```

### 4. 의존성 순서

`condition`을 사용한 의존성 관리:

```yaml
depends_on:
  db:
    condition: service_healthy  # DB가 healthy 상태일 때만 시작
  redis:
    condition: service_started  # Redis가 시작되면 시작
```

## 버전 관리 고려사항

### .gitignore 설정

```gitignore
# 환경 변수 파일
.env
.env.local
.env.*.local

# 비밀 파일
secrets/
*.key
*.pem

# 데이터 볼륨
data/
```

### .env.example 제공

```bash
# .env.example - 템플릿 (커밋 대상)

# Application
COMPOSE_PROJECT_NAME=myapp
APP_VERSION=latest
PYTHON_VERSION=3.12

# Database
POSTGRES_DB=myapp
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=changeme

# Redis
REDIS_URL=redis://redis:6379/0
```
