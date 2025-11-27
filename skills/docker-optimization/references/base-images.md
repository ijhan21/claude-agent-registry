# 베이스 이미지 선택

## 이미지 비교

| 이미지 | 크기 | 장점 | 단점 |
|--------|------|------|------|
| **slim** | ~50MB | 균형 잡힌 크기, 호환성 | Alpine보다 큼 |
| alpine | ~5MB | 최소 크기 | musl libc 호환 문제 |
| distroless | ~20MB | 최소 공격 면적 | 디버깅 어려움 |
| 기본 | ~100MB+ | 완전한 도구 | 불필요하게 큼 |

## 권장: slim 계열

### Python

```dockerfile
# 권장
FROM python:3.12-slim

# 버전 고정 (재현성)
FROM python:3.12.1-slim-bookworm
```

### Node.js

```dockerfile
# 권장
FROM node:20-slim

# 버전 고정
FROM node:20.10.0-slim-bookworm
```

## slim 이미지 필수 패키지 설치

### Python 빌드 의존성

```dockerfile
FROM python:3.12-slim

# 빌드 도구 (필요 시)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```

### 런타임 의존성만

```dockerfile
FROM python:3.12-slim AS runner

# PostgreSQL 클라이언트만 (빌드 도구 제외)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

## 이미지 크기 확인

```bash
# 이미지 크기 확인
docker images myapp

# 레이어별 크기 분석
docker history myapp:latest

# dive로 상세 분석
dive myapp:latest
```

## 크기 목표

| 프레임워크 | 목표 크기 | 최대 허용 |
|------------|----------|----------|
| FastAPI | 150MB | 200MB |
| DRF | 150MB | 200MB |
| Next.js | 200MB | 300MB |

## Alpine 사용 시 주의점

```dockerfile
# Alpine은 musl libc 사용 - 일부 패키지 호환 문제
FROM python:3.12-alpine

# 빌드 의존성 필요
RUN apk add --no-cache \
    build-base \
    libffi-dev \
    postgresql-dev

# 일부 Python 패키지 빌드 실패 가능
# 예: pandas, numpy 등 C 확장 패키지
```

## 버전 고정 전략

```dockerfile
# 개발 중: 마이너 버전
FROM python:3.12-slim

# 프로덕션: 전체 버전 + 배포판
FROM python:3.12.1-slim-bookworm
```

### 버전 확인

```bash
# 사용 가능한 태그 확인
docker search python --limit 5

# Docker Hub에서 확인
# https://hub.docker.com/_/python
# https://hub.docker.com/_/node
```