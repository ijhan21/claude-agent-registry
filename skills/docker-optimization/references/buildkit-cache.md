# BuildKit 캐시 최적화

## BuildKit 활성화

```bash
# 환경 변수로 활성화
export DOCKER_BUILDKIT=1

# 또는 명령어에 직접
DOCKER_BUILDKIT=1 docker build .

# Docker Desktop: 기본 활성화됨
# Linux: 환경 변수 필요
```

## 문법 선언 (필수)

```dockerfile
# syntax=docker/dockerfile:1.4

# 최신 기능 사용을 위해 첫 줄에 선언
FROM python:3.12-slim
```

## 캐시 마운트

### pip 캐시

```dockerfile
# 캐시 마운트로 재빌드 속도 향상
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# 캐시 없이 (기존 방식)
# RUN pip install --no-cache-dir -r requirements.txt
```

### npm 캐시

```dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

### apt 캐시

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y \
    build-essential
```

## 바인드 마운트

```dockerfile
# 파일을 복사하지 않고 마운트
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

## 캐시 최적화 전략

### 레이어 순서 최적화

```dockerfile
# 좋음: 변경 빈도 낮은 것 먼저
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .

# 나쁨: 모든 파일 먼저 복사
COPY . .
RUN pip install -r requirements.txt
```

### 의존성 분리

```dockerfile
# 1. 시스템 의존성 (거의 안 바뀌)
RUN apt-get update && apt-get install -y curl

# 2. 패키지 의존성 (가끔 바뀌)
COPY requirements.txt .
RUN pip install -r requirements.txt

# 3. 소스 코드 (자주 바뀌)
COPY . .
```

## GitHub Actions 캐시

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build
  uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## 로컬 캐시 내보내기/가져오기

```bash
# 캐시 내보내기
docker build \
  --cache-to type=local,dest=/tmp/docker-cache \
  -t myapp .

# 캐시 가져오기
docker build \
  --cache-from type=local,src=/tmp/docker-cache \
  -t myapp .
```

## 인라인 캐시

```bash
# 이미지에 캐시 메타데이터 포함
docker build \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t myapp .

# 다음 빌드에서 이전 이미지를 캐시로 사용
docker build \
  --cache-from myapp:latest \
  -t myapp:new .
```

## 병렬 빌드

```dockerfile
# 독립적인 스테이지는 병렬 실행됨
FROM node:20-slim AS frontend-deps
COPY frontend/package*.json ./
RUN npm ci

FROM python:3.12-slim AS backend-deps
COPY requirements.txt .
RUN pip install -r requirements.txt

# 두 스테이지 결합
FROM python:3.12-slim AS final
COPY --from=frontend-deps /app/node_modules ./frontend/node_modules
COPY --from=backend-deps /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
```

## 캐시 무효화 방지

```dockerfile
# 나쁨: 날짜/시간으로 캐시 무효화
RUN echo $(date) > /build-date.txt

# 좋음: 빌드 인자로 필요 시만 무효화
ARG CACHE_BUST
RUN echo ${CACHE_BUST:-default} > /build-info.txt
```

## 캐시 정리

```bash
# 빌드 캐시 확인
docker builder prune --dry-run

# 캐시 정리
docker builder prune -f

# 전체 정리 (주의)
docker system prune -a --volumes
```

## 캐시 디버깅

```bash
# 빌드 과정 상세 출력
docker build --progress=plain .

# 캐시 사용 여부 확인
# CACHED 표시: 캐시 사용됨
# [2/5] CACHED RUN pip install...
```