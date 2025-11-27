---
name: compose-env
description: 환경별 Docker Compose 구성 가이드. 다음 상황에서 활성화: (1) Compose 환경 분리 - "docker compose 환경", "local/staging/prod compose", "환경별 설정" 언급 시, (2) 다중 컴포즈 설정 - "compose override", "docker-compose.yml 분리", "extends" 언급 시, (3) Compose 모범 사례 - "compose 베스트 프랙티스", "compose 패턴" 언급 시.
version: 1.0.0
---

# Docker Compose 환경별 구성 가이드

환경(local, staging, production)별로 Docker Compose를 효과적으로 구성하는 방법을 다룹니다.

## 핵심 원칙

1. **기본 파일 분리**: `docker-compose.yml`에는 공통 설정만
2. **환경별 Override**: `docker-compose.{env}.yml`로 환경 특화 설정
3. **환경 변수 분리**: `.env.{env}` 파일로 민감 정보 관리
4. **서비스 프로파일**: 재사용 가능한 서비스 정의
5. **실행 스크립트**: 환경별 실행을 간편하게

## 권장 디렉토리 구조

```
project/
├── docker/
│   ├── Dockerfile              # 프로덕션 Dockerfile
│   ├── Dockerfile.dev          # 개발용 Dockerfile
│   └── nginx/
│       ├── nginx.conf
│       └── nginx.dev.conf
├── docker-compose.yml          # 기본 (공통) 설정
├── docker-compose.local.yml    # 로컬 개발 환경
├── docker-compose.staging.yml  # 스테이징 환경
├── docker-compose.prod.yml     # 프로덕션 환경
├── .env.example                # 환경 변수 템플릿
├── .env.local                  # 로컬 환경 변수 (gitignore)
├── .env.staging                # 스테이징 환경 변수
├── .env.prod                   # 프로덕션 환경 변수
└── scripts/
    ├── dev.sh                  # 로컬 실행 스크립트
    ├── staging.sh              # 스테이징 실행
    └── prod.sh                 # 프로덕션 실행
```

## 참조 문서

상세한 구현은 `references/` 디렉토리를 참조하세요:

| 문서 | 내용 |
|------|------|
| [base-compose.md](references/base-compose.md) | 기본 docker-compose.yml 구성 |
| [local-env.md](references/local-env.md) | 로컬 개발 환경 구성 |
| [staging-env.md](references/staging-env.md) | 스테이징 환경 구성 |
| [prod-env.md](references/prod-env.md) | 프로덕션 환경 구성 |
| [env-variables.md](references/env-variables.md) | 환경 변수 관리 |
| [profiles.md](references/profiles.md) | 서비스 프로파일 활용 |
| [networks.md](references/networks.md) | 네트워크 구성 |
| [volumes.md](references/volumes.md) | 볼륨 및 데이터 지속성 |
| [scripts.md](references/scripts.md) | 실행 스크립트 |
| [secrets.md](references/secrets.md) | 비밀 관리 |

## 빠른 시작

### 1. 기본 구성 (docker-compose.yml)

```yaml
# docker-compose.yml - 공통 설정
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    env_file:
      - .env
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
```

### 2. 로컬 환경 (docker-compose.local.yml)

```yaml
# docker-compose.local.yml
version: '3.8'

services:
  app:
    build:
      dockerfile: docker/Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules  # 또는 /app/.venv
    ports:
      - "8000:8000"
    environment:
      - DEBUG=true
      - LOG_LEVEL=DEBUG
    command: ["uvicorn", "main:app", "--reload", "--host", "0.0.0.0"]

  db:
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=app_dev
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=devpass
```

### 3. 실행 방법

```bash
# 로컬 환경
docker compose -f docker-compose.yml -f docker-compose.local.yml up

# 또는 스크립트 사용
./scripts/dev.sh up
```

## 환경별 주요 차이점

| 항목 | Local | Staging | Production |
|------|-------|---------|------------|
| 빌드 | 개발용 Dockerfile | 프로덕션 Dockerfile | 프로덕션 Dockerfile |
| 볼륨 마운트 | 코드 마운트 (Hot Reload) | X | X |
| 포트 노출 | 직접 노출 | 내부만 | 내부만 (Nginx 프록시) |
| 디버그 | 활성화 | 활성화 | 비활성화 |
| 로깅 | DEBUG | INFO | WARNING |
| 레플리카 | 1 | 1 | 2+ |
| 리소스 제한 | 없음 | 제한적 | 엄격 |
| 비밀 관리 | .env 파일 | 환경 변수 | Docker Secrets |

## 품질 체크리스트

### 구성 품질
- [ ] 기본 compose에는 환경 특화 설정이 없는가
- [ ] 각 환경별 override 파일이 분리되었는가
- [ ] 환경 변수가 .env 파일로 분리되었는가
- [ ] 민감 정보가 버전 관리에서 제외되었는가

### 보안
- [ ] 프로덕션 비밀번호가 하드코딩되지 않았는가
- [ ] 불필요한 포트가 노출되지 않았는가
- [ ] 프로덕션에서 디버그 모드가 비활성화되었는가
- [ ] 리소스 제한이 설정되었는가

### 운영
- [ ] 헬스체크가 설정되었는가
- [ ] 의존성 순서가 명시되었는가 (depends_on)
- [ ] 재시작 정책이 적절한가 (restart)
- [ ] 로깅이 적절히 구성되었는가
