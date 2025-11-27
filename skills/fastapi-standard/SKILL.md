---
name: fastapi-standard
description: FastAPI 프로젝트 표준화 가이드. 다음 상황에서 활성화: (1) FastAPI 프로젝트 생성/초기화, (2) API 엔드포인트 작성, (3) 인증/보안 구현, (4) DB 연동 설정. JWT, SQLAlchemy async, Redis, WebSocket, S3 파일업로드, Background Tasks 패턴 포함.
---

# FastAPI Standard

## 개요

FastAPI 프로젝트의 표준화된 구조, 패턴, 설정을 정의합니다.

## 핵심 스택

| 영역 | 기술 | 버전/비고 |
|------|------|----------|
| 인증 | **PyJWT** | python-jose 사용 금지 (유지보수 중단) |
| ORM | SQLAlchemy | async 모드 필수 |
| 마이그레이션 | Alembic | |
| 검증 | Pydantic | v2 |
| 캐싱/큐 | Redis | 캐싱 + ARQ용 |
| Background | ARQ | async 네이티브 |
| 파일저장소 | S3 호환 | boto3 (QNAP 등) |

## 프로젝트 구조

```
project/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 앱, lifespan
│   ├── config.py               # pydantic-settings
│   ├── dependencies.py         # 공통 의존성
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py       # v1 라우터 통합
│   │   │   ├── users.py
│   │   │   ├── items.py
│   │   │   └── websocket.py
│   │   └── health.py           # 버전 무관
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── security.py         # JWT, OAuth2
│   │   ├── logging.py          # 로깅 설정
│   │   └── cache.py            # Redis 캐싱
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── session.py          # async session
│   │   └── base.py             # Base model
│   │
│   ├── models/                 # SQLAlchemy 모델
│   │   ├── __init__.py
│   │   └── user.py
│   │
│   ├── schemas/                # Pydantic 스키마
│   │   ├── __init__.py
│   │   ├── base.py             # 공통 응답 포맷
│   │   ├── user.py
│   │   └── pagination.py
│   │
│   ├── services/               # 비즈니스 로직
│   │   ├── __init__.py
│   │   └── user.py
│   │
│   ├── tasks/                  # ARQ 백그라운드
│   │   ├── __init__.py
│   │   └── worker.py
│   │
│   └── utils/
│       ├── __init__.py
│       └── s3.py               # S3 파일 업로드
│
├── alembic/
│   ├── versions/
│   ├── env.py
│   └── alembic.ini
│
├── .env                        # 환경변수 (gitignore)
├── .env.example                # 환경변수 템플릿
└── requirements.txt
```

상세 구조: [`references/project-structure.md`](references/project-structure.md) 참조

## 설정 관리

### .env 구조

```env
# App
APP_NAME=MyAPI
APP_ENV=development
DEBUG=true

# API
API_V1_PREFIX=/api/v1

# Database
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/db

# Redis
REDIS_URL=redis://localhost:6379/0

# JWT
JWT_SECRET_KEY=your-secret-key
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# S3 (QNAP Object Storage)
S3_ENDPOINT_URL=http://your-qnap:9000
S3_ACCESS_KEY=access-key
S3_SECRET_KEY=secret-key
S3_BUCKET_NAME=uploads

# CORS
CORS_ORIGINS=["http://localhost:3000"]

# Rate Limit
RATE_LIMIT_PER_MINUTE=60
```

### config.py 패턴

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )
    
    # App
    app_name: str = "FastAPI"
    app_env: str = "development"
    debug: bool = False
    
    # API
    api_v1_prefix: str = "/api/v1"
    
    # Database
    database_url: str
    
    # Redis
    redis_url: str = "redis://localhost:6379/0"
    
    # JWT
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 30
    jwt_refresh_token_expire_days: int = 7
    
    # S3
    s3_endpoint_url: str
    s3_access_key: str
    s3_secret_key: str
    s3_bucket_name: str = "uploads"
    
    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]
    
    # Rate Limit
    rate_limit_per_minute: int = 60

settings = Settings()
```

## API 버저닝

**패턴**: URL Prefix (`/api/v1`)

```python
# app/api/v1/router.py
from fastapi import APIRouter
from app.api.v1 import users, items, websocket

router = APIRouter()
router.include_router(users.router, prefix="/users", tags=["users"])
router.include_router(items.router, prefix="/items", tags=["items"])
router.include_router(websocket.router, prefix="/ws", tags=["websocket"])
```

```python
# app/main.py
app.include_router(v1_router, prefix=settings.api_v1_prefix)
app.include_router(health_router)  # 버전 무관: /health, /ready
```

## 표준 응답 포맷

```python
# app/schemas/base.py
from typing import Generic, TypeVar, Optional
from pydantic import BaseModel

T = TypeVar("T")

class ResponseBase(BaseModel, Generic[T]):
    success: bool = True
    data: Optional[T] = None
    message: Optional[str] = None

class PaginatedResponse(BaseModel, Generic[T]):
    success: bool = True
    data: list[T]
    total: int
    page: int
    size: int
    pages: int
```

## Pagination

```python
# app/schemas/pagination.py
from pydantic import BaseModel, Field

class PaginationParams(BaseModel):
    page: int = Field(default=1, ge=1)
    size: int = Field(default=20, ge=1, le=100)
    
    @property
    def offset(self) -> int:
        return (self.page - 1) * self.size
```

```python
# 사용 예시
from fastapi import Depends, Query

def get_pagination(
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(page=page, size=size)

@router.get("/items")
async def list_items(
    pagination: PaginationParams = Depends(get_pagination),
):
    ...
```

## 인증 (JWT + OAuth2)

상세 패턴: [`references/auth-patterns.md`](references/auth-patterns.md) 참조

핵심 요약:
- **라이브러리**: PyJWT (python-jose 금지)
- **토큰**: Access Token + Refresh Token
- **스키마**: OAuth2PasswordBearer

## 의존성 주입

```python
# app/dependencies.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.session import get_session
from app.core.security import get_current_user
from app.models.user import User

# 타입 별칭
DBSession = Annotated[AsyncSession, Depends(get_session)]
CurrentUser = Annotated[User, Depends(get_current_user)]
```

```python
# 라우터에서 사용
@router.get("/me")
async def get_me(
    db: DBSession,
    user: CurrentUser,
):
    return user
```

## Health Check

```python
# app/api/health.py
from fastapi import APIRouter, status
from sqlalchemy import text
from app.db.session import get_session

router = APIRouter(tags=["health"])

@router.get("/health")
async def health():
    """Liveness probe"""
    return {"status": "healthy"}

@router.get("/ready")
async def ready(db: AsyncSession = Depends(get_session)):
    """Readiness probe - DB 연결 확인"""
    try:
        await db.execute(text("SELECT 1"))
        return {"status": "ready", "database": "connected"}
    except Exception:
        return JSONResponse(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            content={"status": "not ready", "database": "disconnected"}
        )
```

## Lifespan (리소스 관리)

```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.db.session import engine
from app.core.cache import redis_client

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await redis_client.connect()
    yield
    # Shutdown
    await redis_client.close()
    await engine.dispose()

app = FastAPI(
    title=settings.app_name,
    lifespan=lifespan,
)
```

## 추가 패턴

상세 구현 패턴:
- **SQLAlchemy async**: [`references/database-patterns.md`](references/database-patterns.md)
- **인증 (JWT/OAuth2)**: [`references/auth-patterns.md`](references/auth-patterns.md)
- **Redis 캐싱**: [`references/cache-patterns.md`](references/cache-patterns.md)
- **ARQ Background Tasks**: [`references/background-patterns.md`](references/background-patterns.md)
- **S3 파일 업로드**: [`references/file-upload-patterns.md`](references/file-upload-patterns.md)
- **WebSocket**: [`references/websocket-patterns.md`](references/websocket-patterns.md)
- **Rate Limiting**: [`references/rate-limit-patterns.md`](references/rate-limit-patterns.md)
- **로깅**: [`references/logging-patterns.md`](references/logging-patterns.md)
- **SEO/HTTP 최적화**: [`references/seo-patterns.md`](references/seo-patterns.md)
- **OpenAPI 태그**: [`references/openapi-patterns.md`](references/openapi-patterns.md)

## 품질 체크리스트

프로젝트 생성 시 확인:

- [ ] .env.example 작성
- [ ] config.py에서 모든 환경변수 정의
- [ ] lifespan에서 리소스 초기화/정리
- [ ] /health, /ready 엔드포인트 존재
- [ ] 표준 응답 포맷 사용
- [ ] PyJWT 사용 (python-jose 아님)
- [ ] SQLAlchemy async 모드
- [ ] Pydantic v2 문법
- [ ] 도메인별 라우터 분리
- [ ] API 버저닝 (/api/v1)