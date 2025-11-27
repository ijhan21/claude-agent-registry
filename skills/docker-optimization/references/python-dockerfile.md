# Python Dockerfile 패턴

## FastAPI - Dockerfile.dev

```dockerfile
# syntax=docker/dockerfile:1.4
FROM python:3.12-slim

WORKDIR /app

# 시스템 의존성
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 의존성 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 소스 복사 (볼륨 마운트로 대체됨)
COPY . .

EXPOSE 8000

# 개발 서버 (핫리로드)
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

## FastAPI - Dockerfile.prod

```dockerfile
# syntax=docker/dockerfile:1.4

# === Builder Stage ===
FROM python:3.12-slim AS builder

WORKDIR /app

# 빌드 의존성
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 의존성 먼저 (캐시 활용)
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

# 소스 복사
COPY . .

# === Runner Stage ===
FROM python:3.12-slim AS runner

WORKDIR /app

# 런타임 의존성만
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# non-root 사용자
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# 패키지 복사
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin

# 앱 복사
COPY --from=builder /app /app

# 권한 설정
RUN chown -R appuser:appgroup /app
USER appuser

# 헬스체크
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

# Gunicorn + Uvicorn Workers
CMD ["gunicorn", "main:app", \
     "-w", "4", \
     "-k", "uvicorn.workers.UvicornWorker", \
     "-b", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

## DRF - Dockerfile.prod

```dockerfile
# syntax=docker/dockerfile:1.4

# === Builder Stage ===
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir -r requirements.txt

COPY . .

# Static 파일 수집
RUN python manage.py collectstatic --noinput

# === Runner Stage ===
FROM python:3.12-slim AS runner

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd -r django && useradd -r -g django django

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY --from=builder /app /app

RUN chown -R django:django /app
USER django

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

EXPOSE 8000

CMD ["gunicorn", "config.wsgi:application", \
     "-w", "4", \
     "-b", "0.0.0.0:8000", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

## Gunicorn 설정 파일 (선택)

```python
# gunicorn.conf.py
import multiprocessing

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "uvicorn.workers.UvicornWorker"  # FastAPI용

# Binding
bind = "0.0.0.0:8000"

# Logging
accesslog = "-"
errorlog = "-"
loglevel = "info"

# Timeouts
timeout = 120
keepalive = 5

# Security
limit_request_line = 4094
limit_request_fields = 100
```

```dockerfile
# Dockerfile에서 설정 파일 사용
CMD ["gunicorn", "main:app", "-c", "gunicorn.conf.py"]
```

## 환경 변수 처리

```dockerfile
# 빌드 타임 변수
ARG PYTHON_VERSION=3.12

FROM python:${PYTHON_VERSION}-slim

# 런타임 환경 변수
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1
```

## requirements.txt 분리 전략

```
requirements/
├── base.txt      # 공통 의존성
├── dev.txt       # 개발용 (pytest, black 등)
└── prod.txt      # 프로덕션용 (gunicorn 등)
```

```dockerfile
# 개발용
COPY requirements/base.txt requirements/dev.txt ./requirements/
RUN pip install -r requirements/dev.txt

# 프로덕션용
COPY requirements/base.txt requirements/prod.txt ./requirements/
RUN pip install -r requirements/prod.txt
```

## Worker 수 계산

```bash
# CPU 기반 권장
# Gunicorn: (2 * CPU) + 1
# 4코어 서버: 9 workers

# 메모리 기반 조정
# 각 worker 약 50-100MB
# 8GB RAM, 4코어: 4-8 workers
```