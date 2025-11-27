---
name: docker-optimization
description: Docker 이미지 최적화 가이드. 다음 상황에서 활성화: (1) Dockerfile 작성/리뷰, (2) 이미지 크기 최적화, (3) 빌드 속도 개선, (4) 보안 강화. FastAPI, DRF, Next.js 패턴 포함.
---

# Docker Optimization

## 개요

프로덕션 환경을 위한 Docker 이미지 최적화 가이드입니다. 이미지 크기 최소화, 빌드 속도 향상, 보안 강화에 집중합니다.

## 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **최소 이미지** | slim 베이스, 불필요한 파일 제거 |
| **멀티스테이지** | 빌드 의존성과 런타임 분리 |
| **레이어 캐시** | 변경 빈도순 레이어 배치 |
| **BuildKit** | 병렬 빌드, 캐시 마운트 |
| **보안** | non-root 사용자, 취약점 스캔 |

## 지원 스택

| 프레임워크 | 베이스 이미지 | 특징 |
|------------|--------------|------|
| FastAPI | python:3.12-slim | uvicorn, gunicorn |
| DRF | python:3.12-slim | gunicorn |
| Next.js | node:20-slim | standalone 빌드 |

## 파일 구조

```
project/
├── Dockerfile.dev          # 개발용 (핫리로드)
├── Dockerfile.prod         # 프로덕션 (최적화)
├── .dockerignore           # 빌드 제외 파일
├── docker-compose.yml      # 로컬 개발
└── docker-compose.prod.yml # 프로덕션
```

## 빌드 명령어

```bash
# BuildKit 활성화 (필수)
export DOCKER_BUILDKIT=1

# 개발 빌드
docker build -f Dockerfile.dev -t myapp:dev .

# 프로덕션 빌드 (캐시 활용)
docker build -f Dockerfile.prod \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t myapp:$(git rev-parse --short HEAD) .
```

상세: [`references/buildkit-cache.md`](references/buildkit-cache.md)

## Python (FastAPI/DRF)

### Dockerfile.prod

```dockerfile
# syntax=docker/dockerfile:1.4
FROM python:3.12-slim AS builder

WORKDIR /app

# 의존성 먼저 (캐시 활용)
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# 소스 복사
COPY . .

# ---
FROM python:3.12-slim AS runner

WORKDIR /app

# 보안: non-root 사용자
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# 런타임 의존성만 복사
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY --from=builder /app /app

# 권한 설정
RUN chown -R appuser:appgroup /app
USER appuser

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["gunicorn", "main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

상세: [`references/python-dockerfile.md`](references/python-dockerfile.md)

## Node.js (Next.js)

### Dockerfile.prod

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-slim AS deps

WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# ---
FROM node:20-slim AS builder

WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .
RUN npm run build

# ---
FROM node:20-slim AS runner

WORKDIR /app
ENV NODE_ENV=production

# 보안: non-root 사용자
RUN groupadd -r nodejs && useradd -r -g nodejs nextjs

# standalone 출력만 복사
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

RUN chown -R nextjs:nodejs /app
USER nextjs

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1

EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

상세: [`references/node-dockerfile.md`](references/node-dockerfile.md)

## .dockerignore (필수)

```dockerignore
# Git
.git
.gitignore

# Dependencies
node_modules
__pycache__
*.pyc
.venv
venv

# Build outputs
.next
dist
build
*.egg-info

# IDE
.idea
.vscode
*.swp

# Environment
.env
.env.*
!.env.example

# Docker
Dockerfile*
docker-compose*
.dockerignore

# Tests
tests
__tests__
*.test.js
*.spec.js
pytest.ini
.coverage

# Docs
docs
*.md
!README.md

# Misc
.DS_Store
*.log
tmp
```

상세: [`references/dockerignore.md`](references/dockerignore.md)

## 이미지 태깅 전략

```bash
# 태그 형식: {버전}-{커밋해시}-{날짜}
VERSION=1.0.0
COMMIT=$(git rev-parse --short HEAD)
DATE=$(date +%Y%m%d)

docker build -t myapp:${VERSION}-${COMMIT}-${DATE} .
docker tag myapp:${VERSION}-${COMMIT}-${DATE} myapp:latest
```

| 태그 | 용도 |
|------|------|
| `latest` | 최신 안정 버전 |
| `1.0.0` | 릴리스 버전 |
| `1.0.0-abc123-20241127` | 정확한 빌드 추적 |
| `develop` | 개발 브랜치 |

상세: [`references/tagging.md`](references/tagging.md)

## 취약점 스캔 (Trivy)

```bash
# 이미지 스캔
trivy image myapp:latest

# CI/CD에서 심각 취약점 시 실패
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# 빌드 전 Dockerfile 스캔
trivy config Dockerfile.prod
```

상세: [`references/security-scan.md`](references/security-scan.md)

## GitHub Actions CI/CD

```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    branches: [main, develop]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          push: false
          tags: myapp:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          exit-code: 1
          severity: CRITICAL,HIGH
```

상세: [`references/cicd-patterns.md`](references/cicd-patterns.md)

## 추가 패턴

상세 구현:

- **베이스 이미지**: [`references/base-images.md`](references/base-images.md)
- **Python Dockerfile**: [`references/python-dockerfile.md`](references/python-dockerfile.md)
- **Node.js Dockerfile**: [`references/node-dockerfile.md`](references/node-dockerfile.md)
- **BuildKit 캐시**: [`references/buildkit-cache.md`](references/buildkit-cache.md)
- **멀티스테이지**: [`references/multi-stage.md`](references/multi-stage.md)
- **빌드 시크릿**: [`references/secrets.md`](references/secrets.md)
- **이미지 태깅**: [`references/tagging.md`](references/tagging.md)
- **.dockerignore**: [`references/dockerignore.md`](references/dockerignore.md)
- **취약점 스캔**: [`references/security-scan.md`](references/security-scan.md)
- **헬스체크**: [`references/healthcheck.md`](references/healthcheck.md)
- **로그 수집**: [`references/logging.md`](references/logging.md)
- **CI/CD 패턴**: [`references/cicd-patterns.md`](references/cicd-patterns.md)

## 최적화 체크리스트

Dockerfile 작성 시 확인:

- [ ] BuildKit 문법 선언 (`# syntax=docker/dockerfile:1.4`)
- [ ] slim 베이스 이미지 사용
- [ ] 멀티스테이지 빌드 적용
- [ ] 의존성 레이어 먼저 (캐시 활용)
- [ ] 캐시 마운트 사용 (`--mount=type=cache`)
- [ ] non-root 사용자 설정
- [ ] HEALTHCHECK 지시어 포함
- [ ] .dockerignore 최적화
- [ ] 불필요한 파일 제거
- [ ] 취약점 스캔 통과
- [ ] 이미지 크기 확인 (목표: Python <200MB, Node <300MB)