# 로컬 개발 환경 구성

로컬 개발 환경을 위한 Docker Compose override 설정입니다.

## docker-compose.local.yml

```yaml
# docker-compose.local.yml
version: '3.8'

services:
  # ===================
  # Application - 개발 모드
  # ===================
  app:
    build:
      dockerfile: docker/Dockerfile.dev
      target: development
    volumes:
      # 코드 마운트 (Hot Reload)
      - .:/app
      # 의존성 캐시 (마운트에서 제외)
      - /app/node_modules
      - /app/.venv
    ports:
      - "8000:8000"   # API
      - "5678:5678"   # 디버거 (debugpy)
    environment:
      - DEBUG=true
      - LOG_LEVEL=DEBUG
      - RELOAD=true
    command: >
      sh -c "uvicorn main:app 
             --reload 
             --host 0.0.0.0 
             --port 8000"
    # 개발 중 리소스 제한 없음
    # deploy 섹션 생략

  # ===================
  # Database - 포트 노출
  # ===================
  db:
    ports:
      - "5432:5432"  # 외부 접근 허용
    environment:
      - POSTGRES_DB=app_dev
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=devpass

  # ===================
  # Redis - 포트 노출
  # ===================
  redis:
    ports:
      - "6379:6379"  # 외부 접근 허용

  # ===================
  # 개발 전용 서비스
  # ===================
  
  # 메일 테스트 (MailHog)
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    networks:
      - app-network

  # DB 관리 (Adminer)
  adminer:
    image: adminer:latest
    ports:
      - "8080:8080"
    networks:
      - app-network
    depends_on:
      - db
```

## 개발용 Dockerfile

```dockerfile
# docker/Dockerfile.dev
FROM python:3.12-slim as development

WORKDIR /app

# 개발 도구 설치
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 의존성 설치 (개발 의존성 포함)
COPY requirements.txt requirements-dev.txt ./
RUN pip install --no-cache-dir -r requirements.txt -r requirements-dev.txt

# 디버거 설치
RUN pip install debugpy

# 소스 코드는 볼륨으로 마운트됨

EXPOSE 8000 5678

CMD ["uvicorn", "main:app", "--reload", "--host", "0.0.0.0", "--port", "8000"]
```

## 로컬 환경 변수

```bash
# .env.local

# Application
COMPOSE_PROJECT_NAME=myapp_dev
APP_ENV=local
DEBUG=true
LOG_LEVEL=DEBUG

# Database
POSTGRES_DB=app_dev
POSTGRES_USER=dev
POSTGRES_PASSWORD=devpass
DATABASE_URL=postgresql://dev:devpass@db:5432/app_dev

# Redis
REDIS_URL=redis://redis:6379/0

# Mail (MailHog)
SMTP_HOST=mailhog
SMTP_PORT=1025

# CORS (개발 시 모든 origin 허용)
CORS_ORIGINS=*
```

## 실행 스크립트

```bash
#!/bin/bash
# scripts/dev.sh

set -e

# 환경 변수 로드
if [ -f .env.local ]; then
    export $(cat .env.local | grep -v '^#' | xargs)
fi

# Compose 파일 설정
COMPOSE_FILES="-f docker-compose.yml -f docker-compose.local.yml"

case "$1" in
    up)
        echo "🚀 Starting local development environment..."
        docker compose $COMPOSE_FILES up -d
        echo "✅ Services started!"
        echo "   App: http://localhost:8000"
        echo "   Adminer: http://localhost:8080"
        echo "   MailHog: http://localhost:8025"
        ;;
    down)
        echo "🛑 Stopping services..."
        docker compose $COMPOSE_FILES down
        ;;
    logs)
        docker compose $COMPOSE_FILES logs -f ${2:-app}
        ;;
    shell)
        docker compose $COMPOSE_FILES exec app /bin/bash
        ;;
    test)
        docker compose $COMPOSE_FILES exec app pytest ${@:2}
        ;;
    rebuild)
        echo "🔄 Rebuilding..."
        docker compose $COMPOSE_FILES build --no-cache
        docker compose $COMPOSE_FILES up -d
        ;;
    *)
        echo "Usage: $0 {up|down|logs|shell|test|rebuild}"
        exit 1
        ;;
esac
```

## Hot Reload 설정

### Python (uvicorn)

```yaml
command: uvicorn main:app --reload --host 0.0.0.0
volumes:
  - .:/app
```

### Node.js

```yaml
command: npm run dev  # nodemon 사용
volumes:
  - .:/app
  - /app/node_modules  # node_modules 제외
```

### React/Vite

```yaml
command: npm run dev -- --host
volumes:
  - .:/app
  - /app/node_modules
ports:
  - "5173:5173"  # Vite 기본 포트
```

## 디버깅 설정

### VS Code + debugpy

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Remote Attach",
      "type": "python",
      "request": "attach",
      "connect": {
        "host": "localhost",
        "port": 5678
      },
      "pathMappings": [
        {
          "localRoot": "${workspaceFolder}",
          "remoteRoot": "/app"
        }
      ]
    }
  ]
}
```

```yaml
# docker-compose.local.yml
services:
  app:
    ports:
      - "5678:5678"
    command: >
      python -m debugpy --listen 0.0.0.0:5678 
      -m uvicorn main:app --reload --host 0.0.0.0
```
