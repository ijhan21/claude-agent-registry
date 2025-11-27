# 헬스체크 패턴

## Dockerfile HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

### 옵션 설명

| 옵션 | 설명 | 권장값 |
|------|------|--------|
| `--interval` | 체크 간격 | 30s |
| `--timeout` | 타임아웃 | 10s |
| `--start-period` | 초기 대기 | 5-10s |
| `--retries` | 재시도 횟수 | 3 |

## Python (FastAPI) 헬스체크

### 엔드포인트

```python
# app/api/health.py
from fastapi import APIRouter, status
from fastapi.responses import JSONResponse
from datetime import datetime

router = APIRouter()

@router.get("/health")
async def health_check():
    return JSONResponse(
        status_code=status.HTTP_200_OK,
        content={
            "status": "healthy",
            "timestamp": datetime.utcnow().isoformat()
        }
    )

@router.get("/health/ready")
async def readiness_check(db: Session = Depends(get_db)):
    """의존성 포함 체크"""
    try:
        # DB 연결 확인
        db.execute("SELECT 1")
        return {"status": "ready", "database": "connected"}
    except Exception as e:
        return JSONResponse(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            content={"status": "not ready", "error": str(e)}
        )
```

### Dockerfile

```dockerfile
# curl 설치 필요
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

## Python (DRF) 헬스체크

```python
# health/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from django.db import connection

@api_view(['GET'])
def health_check(request):
    return Response({"status": "healthy"})

@api_view(['GET'])
def readiness_check(request):
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        return Response({"status": "ready"})
    except Exception as e:
        return Response(
            {"status": "not ready", "error": str(e)},
            status=503
        )
```

```python
# urls.py
urlpatterns = [
    path('health/', health_check),
    path('health/ready/', readiness_check),
]
```

## Node.js (Next.js) 헬스체크

```typescript
// app/api/health/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({
    status: 'healthy',
    timestamp: new Date().toISOString()
  })
}
```

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1
```

## curl 없이 헬스체크

### Python 스크립트

```python
#!/usr/bin/env python3
# healthcheck.py
import urllib.request
import sys

try:
    response = urllib.request.urlopen("http://localhost:8000/health", timeout=5)
    if response.status == 200:
        sys.exit(0)
except Exception:
    pass

sys.exit(1)
```

```dockerfile
COPY healthcheck.py /usr/local/bin/healthcheck.py
RUN chmod +x /usr/local/bin/healthcheck.py

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python /usr/local/bin/healthcheck.py
```

### wget 사용

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends wget

HEALTHCHECK CMD wget --no-verbose --tries=1 --spider http://localhost:8000/health || exit 1
```

## Docker Compose 헬스체크

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.prod
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## 의존성 체크

### Redis 연결 확인

```python
@router.get("/health/ready")
async def readiness_check():
    checks = {}
    
    # DB 체크
    try:
        db.execute("SELECT 1")
        checks["database"] = "ok"
    except:
        checks["database"] = "failed"
    
    # Redis 체크
    try:
        redis.ping()
        checks["redis"] = "ok"
    except:
        checks["redis"] = "failed"
    
    all_ok = all(v == "ok" for v in checks.values())
    status_code = 200 if all_ok else 503
    
    return JSONResponse(
        status_code=status_code,
        content={"status": "ready" if all_ok else "not ready", "checks": checks}
    )
```

## 상태 확인

```bash
# 컨테이너 헬스 상태 확인
docker ps
# CONTAINER ID   STATUS
# abc123         Up 2 hours (healthy)

# 상세 정보
docker inspect --format='{{json .State.Health}}' container_name | jq

# 로그 확인
docker inspect --format='{{json .State.Health.Log}}' container_name | jq
```

## Liveness vs Readiness

| 체크 유형 | 목적 | 실패 시 |
|----------|------|--------|
| **Liveness** | 앱이 살아있는가 | 컨테이너 재시작 |
| **Readiness** | 요청 처리 가능한가 | 트래픽 차단 |

```dockerfile
# 기본 liveness
HEALTHCHECK CMD curl -f http://localhost:8000/health

# readiness는 로드밸런서/오케스트레이터에서 처리
```

## 모니터링 연동

```python
# Prometheus 메트릭 노출
from prometheus_client import Counter, Histogram

health_check_total = Counter('health_check_total', 'Total health checks')
health_check_duration = Histogram('health_check_duration_seconds', 'Health check duration')

@router.get("/health")
async def health_check():
    health_check_total.inc()
    with health_check_duration.time():
        # 체크 로직
        return {"status": "healthy"}
```
