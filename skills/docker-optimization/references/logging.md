# 컨테이너 로그 수집

## 기본 원칙

1. **stdout/stderr로 출력** (파일 X)
2. **JSON 형식** 권장
3. **구조화된 로그**
4. **로그 레벨 설정**

## Python 로깅 설정

### 기본 설정

```python
# app/core/logging.py
import logging
import sys
from pythonjsonlogger import jsonlogger

def setup_logging(level: str = "INFO"):
    handler = logging.StreamHandler(sys.stdout)
    
    # JSON 포매터
    formatter = jsonlogger.JsonFormatter(
        fmt="%(asctime)s %(levelname)s %(name)s %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S"
    )
    handler.setFormatter(formatter)
    
    # 루트 로거 설정
    root_logger = logging.getLogger()
    root_logger.handlers = [handler]
    root_logger.setLevel(level)
    
    # uvicorn 로거
    for logger_name in ["uvicorn", "uvicorn.access", "uvicorn.error"]:
        logger = logging.getLogger(logger_name)
        logger.handlers = [handler]
```

### 사용

```python
import logging

logger = logging.getLogger(__name__)

logger.info("User logged in", extra={"user_id": 123, "ip": "1.2.3.4"})
logger.error("Database error", extra={"query": "SELECT..."}, exc_info=True)
```

### requirements.txt

```
python-json-logger>=2.0.0
```

## FastAPI 로깅

```python
# main.py
from app.core.logging import setup_logging
import os

# 환경 변수로 로그 레벨 설정
LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")
setup_logging(LOG_LEVEL)

app = FastAPI()

@app.middleware("http")
async def log_requests(request: Request, call_next):
    import time
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    logger.info(
        "Request processed",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status": response.status_code,
            "duration_ms": round(duration * 1000, 2)
        }
    )
    return response
```

## Gunicorn 로깅

```python
# gunicorn.conf.py
import sys

# stdout으로 출력
accesslog = "-"
errorlog = "-"

# 로그 레벨
loglevel = "info"

# JSON 포맷 (선택)
access_log_format = '{"time": "%(t)s", "method": "%(m)s", "path": "%(U)s", "status": "%(s)s", "duration": "%(D)s"}'
```

## Node.js 로깅

### pino 사용

```typescript
// lib/logger.ts
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  timestamp: () => `,"time":"${new Date().toISOString()}"`,
})
```

```typescript
// 사용
import { logger } from '@/lib/logger'

logger.info({ userId: 123, action: 'login' }, 'User logged in')
logger.error({ error: err.message, stack: err.stack }, 'Request failed')
```

### package.json

```json
{
  "dependencies": {
    "pino": "^8.0.0",
    "pino-pretty": "^10.0.0"
  }
}
```

## Dockerfile 설정

```dockerfile
# 환경 변수로 로그 레벨 제어
ENV LOG_LEVEL=info

# stdout 버퍼링 비활성화 (Python)
ENV PYTHONUNBUFFERED=1

# stdout/stderr 분리하지 않음
CMD ["gunicorn", "main:app", "--access-logfile", "-", "--error-logfile", "-"]
```

## Docker Compose 로깅

```yaml
services:
  api:
    build: .
    logging:
      driver: "json-file"
      options:
        max-size: "10m"    # 파일당 최대 크기
        max-file: "3"      # 최대 파일 수
        compress: "true"   # 압축
    environment:
      - LOG_LEVEL=info
```

## 로그 확인

```bash
# 실시간 로그
docker logs -f container_name

# 최근 100줄
docker logs --tail 100 container_name

# 타임스탬프 포함
docker logs -t container_name

# 특정 시간 이후
docker logs --since 2024-01-01T00:00:00 container_name

# JSON 파싱 (jq 사용)
docker logs container_name 2>&1 | jq .
```

## 로그 수집 스택

### Loki + Promtail

```yaml
# docker-compose.yml
services:
  api:
    build: .
    logging:
      driver: "json-file"
      options:
        tag: "{{.Name}}"

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml
    command: -config.file=/etc/promtail/config.yml

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
```

### 간단한 파일 수집

```yaml
services:
  api:
    volumes:
      - ./logs:/app/logs
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"
```

## 로그 레벨 가이드

| 레벨 | 용도 |
|------|------|
| DEBUG | 개발 디버깅 |
| INFO | 일반 정보 (요청, 처리) |
| WARNING | 주의 필요 (성능 저하) |
| ERROR | 에러 (요청 실패) |
| CRITICAL | 치명적 (서비스 중단) |

## 민감 정보 필터링

```python
import re

def filter_sensitive(log_record):
    sensitive_patterns = [
        (r'"password"\s*:\s*"[^"]*"', '"password": "[FILTERED]"'),
        (r'"token"\s*:\s*"[^"]*"', '"token": "[FILTERED]"'),
        (r'"authorization"\s*:\s*"[^"]*"', '"authorization": "[FILTERED]"'),
    ]
    
    message = log_record.getMessage()
    for pattern, replacement in sensitive_patterns:
        message = re.sub(pattern, replacement, message, flags=re.IGNORECASE)
    
    return message
```

## 운영 환경 설정

```dockerfile
# 프로덕션: INFO 이상
ENV LOG_LEVEL=info

# 디버깅 필요 시 환경 변수로 변경
# docker run -e LOG_LEVEL=debug myapp
```
