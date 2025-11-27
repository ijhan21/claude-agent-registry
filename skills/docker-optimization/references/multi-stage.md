# 멀티스테이지 빌드

## 기본 개념

```dockerfile
# Stage 1: 빌드 환경 (크고 무거움)
FROM node:20 AS builder
RUN npm run build

# Stage 2: 런타임 환경 (작고 가벼움)
FROM node:20-slim AS runner
COPY --from=builder /app/dist ./dist
```

## 장점

| 항목 | 단일 스테이지 | 멀티스테이지 |
|------|-------------|-------------|
| 이미지 크기 | 500MB+ | 100-200MB |
| 빌드 도구 포함 | O | X |
| 보안 | 공격 면적 큼 | 최소화 |
| 빌드 속도 | 느림 | 병렬 가능 |

## Python 패턴

```dockerfile
# syntax=docker/dockerfile:1.4

# === 빌드 스테이지 ===
FROM python:3.12-slim AS builder

WORKDIR /app

# 빌드 의존성 (gcc, make 등)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --prefix=/install -r requirements.txt

# === 런타임 스테이지 ===
FROM python:3.12-slim AS runner

WORKDIR /app

# 런타임만 필요한 라이브러리
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# 빌드 결과만 복사 (빌드 도구 제외)
COPY --from=builder /install /usr/local

COPY . .
```

## Node.js 패턴

```dockerfile
# syntax=docker/dockerfile:1.4

# === 의존성 스테이지 ===
FROM node:20-slim AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

# === 빌드 스테이지 ===
FROM node:20-slim AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# === 런타임 스테이지 ===
FROM node:20-slim AS runner
WORKDIR /app
ENV NODE_ENV=production

# 필요한 파일만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
```

## 스테이지 이름 규칙

```dockerfile
# 권장 이름
FROM ... AS deps      # 의존성 설치
FROM ... AS builder   # 빌드
FROM ... AS runner    # 런타임 (최종)
FROM ... AS dev       # 개발용
FROM ... AS test      # 테스트용
```

## 특정 스테이지만 빌드

```bash
# 특정 스테이지까지만 빌드
docker build --target builder -t myapp:builder .

# 개발용 이미지
docker build --target dev -t myapp:dev .

# 테스트용 이미지
docker build --target test -t myapp:test .
```

## 테스트 스테이지 포함

```dockerfile
# === Test Stage ===
FROM builder AS test
RUN pip install pytest pytest-cov
COPY tests/ tests/
RUN pytest tests/ --cov=app

# === Runner Stage ===
FROM python:3.12-slim AS runner
# 테스트 통과 후 런타임 이미지 생성
COPY --from=builder /install /usr/local
```

## 개발 vs 프로덕션 분기

```dockerfile
# === Base ===
FROM python:3.12-slim AS base
WORKDIR /app
COPY requirements.txt .

# === Development ===
FROM base AS dev
RUN pip install -r requirements.txt
RUN pip install pytest black flake8
COPY . .
CMD ["uvicorn", "main:app", "--reload", "--host", "0.0.0.0"]

# === Production ===
FROM base AS prod
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["gunicorn", "main:app", "-k", "uvicorn.workers.UvicornWorker"]
```

## 공유 스테이지

```dockerfile
# === 공통 베이스 ===
FROM python:3.12-slim AS python-base
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
WORKDIR /app

# === API 서비스 ===
FROM python-base AS api
COPY api/requirements.txt .
RUN pip install -r requirements.txt
COPY api/ .

# === Worker 서비스 ===
FROM python-base AS worker
COPY worker/requirements.txt .
RUN pip install -r requirements.txt
COPY worker/ .
```

## 파일 복사 최적화

```dockerfile
# 특정 파일만 복사
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# 디렉토리 복사
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

# 권한 유지
COPY --from=builder --chown=appuser:appgroup /app /app
```