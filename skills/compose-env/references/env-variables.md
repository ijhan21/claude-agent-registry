# 환경 변수 관리

Docker Compose에서 환경 변수를 효과적으로 관리하는 방법입니다.

## 환경 변수 우선순위

Docker Compose는 다음 순서로 환경 변수를 적용합니다:

1. Compose 파일에 직접 정의된 값
2. Shell 환경 변수
3. `.env` 파일
4. Dockerfile의 ENV 기본값

## .env 파일 구조

### .env.example (템플릿)

```bash
# .env.example - 커밋 대상
# 복사 후 .env.local, .env.staging, .env.prod 등으로 사용

# ============================================
# Application
# ============================================
COMPOSE_PROJECT_NAME=myapp
APP_ENV=local                    # local | staging | production
APP_VERSION=latest
DEBUG=true
LOG_LEVEL=DEBUG                  # DEBUG | INFO | WARNING | ERROR
SECRET_KEY=your-secret-key-here

# ============================================
# Server
# ============================================
HOST=0.0.0.0
PORT=8000
WORKERS=4

# ============================================
# Database
# ============================================
POSTGRES_HOST=db
POSTGRES_PORT=5432
POSTGRES_DB=myapp
POSTGRES_USER=myapp_user
POSTGRES_PASSWORD=changeme
DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}

# ============================================
# Redis
# ============================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_URL=redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/0

# ============================================
# External Services
# ============================================
# Email
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
EMAIL_FROM=noreply@example.com

# AWS
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=ap-northeast-2
S3_BUCKET=

# ============================================
# Security
# ============================================
CORS_ORIGINS=http://localhost:3000,http://localhost:8000
ALLOWED_HOSTS=localhost,127.0.0.1

# ============================================
# Monitoring
# ============================================
SENTRY_DSN=
PROMETHEUS_ENABLED=false
```

## 환경별 설정 예시

### .env.local

```bash
# .env.local - 로컬 개발
COMPOSE_PROJECT_NAME=myapp_dev
APP_ENV=local
DEBUG=true
LOG_LEVEL=DEBUG
SECRET_KEY=dev-secret-key-not-for-production

POSTGRES_DB=myapp_dev
POSTGRES_USER=dev
POSTGRES_PASSWORD=devpass

CORS_ORIGINS=*
```

### .env.staging

```bash
# .env.staging - 스테이징
COMPOSE_PROJECT_NAME=myapp_staging
APP_ENV=staging
DEBUG=true
LOG_LEVEL=INFO
SECRET_KEY=${STAGING_SECRET_KEY}  # CI/CD에서 주입

POSTGRES_DB=myapp_staging
POSTGRES_USER=staging_user
POSTGRES_PASSWORD=${STAGING_DB_PASSWORD}

CORS_ORIGINS=https://staging.example.com
SENTRY_DSN=${STAGING_SENTRY_DSN}
```

### .env.prod

```bash
# .env.prod - 프로덕션
COMPOSE_PROJECT_NAME=myapp
APP_ENV=production
DEBUG=false
LOG_LEVEL=WARNING
SECRET_KEY=${PROD_SECRET_KEY}  # CI/CD 또는 Secrets Manager에서 주입

POSTGRES_DB=myapp
POSTGRES_USER=prod_user
POSTGRES_PASSWORD=${PROD_DB_PASSWORD}

CORS_ORIGINS=https://example.com,https://www.example.com
SENTRY_DSN=${PROD_SENTRY_DSN}
```

## Compose에서 환경 변수 사용

### 기본 사용

```yaml
services:
  app:
    image: ${DOCKER_REGISTRY}/${COMPOSE_PROJECT_NAME}:${APP_VERSION:-latest}
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - SECRET_KEY=${SECRET_KEY}
```

### 기본값 설정

```yaml
# 환경 변수가 없으면 기본값 사용
image: ${COMPOSE_PROJECT_NAME:-myapp}:${APP_VERSION:-latest}

# 비어있으면 기본값
environment:
  - LOG_LEVEL=${LOG_LEVEL:-INFO}
```

### 필수 변수 검증

```yaml
# 필수 - 없으면 에러
environment:
  - DATABASE_URL=${DATABASE_URL:?DATABASE_URL is required}
  - SECRET_KEY=${SECRET_KEY:?SECRET_KEY is required}
```

### env_file 사용

```yaml
services:
  app:
    env_file:
      - .env                    # 공통 환경 변수
      - .env.${APP_ENV:-local}  # 환경별 변수 (동적)
```

## 민감 정보 관리

### .gitignore 설정

```gitignore
# 환경 변수 파일 (민감 정보 포함)
.env
.env.local
.env.staging
.env.prod
.env.*.local

# 예외: 템플릿은 커밋
!.env.example
```

### CI/CD에서 민감 정보 주입

```yaml
# GitHub Actions
jobs:
  deploy:
    steps:
      - name: Create .env file
        run: |
          cat > .env.prod << EOF
          SECRET_KEY=${{ secrets.SECRET_KEY }}
          POSTGRES_PASSWORD=${{ secrets.DB_PASSWORD }}
          SENTRY_DSN=${{ secrets.SENTRY_DSN }}
          EOF
```

## 환경 변수 검증 스크립트

```bash
#!/bin/bash
# scripts/validate-env.sh

ENV_FILE=${1:-.env}

required_vars=(
    "COMPOSE_PROJECT_NAME"
    "APP_ENV"
    "SECRET_KEY"
    "POSTGRES_DB"
    "POSTGRES_USER"
    "POSTGRES_PASSWORD"
)

echo "🔍 Validating $ENV_FILE..."

missing=0
for var in "${required_vars[@]}"; do
    if ! grep -q "^${var}=" "$ENV_FILE" 2>/dev/null; then
        echo "❌ Missing: $var"
        missing=$((missing + 1))
    fi
done

if [ $missing -eq 0 ]; then
    echo "✅ All required variables present!"
else
    echo "⚠️ $missing required variables missing"
    exit 1
fi
```
