# 네트워크 구성

Docker Compose에서 네트워크를 효과적으로 구성하는 방법입니다.

## 기본 네트워크 구성

```yaml
version: '3.8'

services:
  app:
    networks:
      - frontend
      - backend

  nginx:
    networks:
      - frontend

  db:
    networks:
      - backend

  redis:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 외부 접근 차단
```

## 네트워크 분리 전략

### 1. Frontend / Backend 분리

```yaml
networks:
  # 외부 접근 가능
  frontend:
    driver: bridge
    name: ${COMPOSE_PROJECT_NAME}_frontend

  # 내부 전용 (외부 접근 불가)
  backend:
    driver: bridge
    internal: true
    name: ${COMPOSE_PROJECT_NAME}_backend
```

### 2. 서비스별 네트워크 할당

```yaml
services:
  # Nginx - 외부 + 내부
  nginx:
    networks:
      - frontend
      - backend

  # App - 내부만
  app:
    networks:
      - backend

  # DB - 내부만 (격리)
  db:
    networks:
      - backend
```

## 네트워크 옵션

### IP 대역 지정

```yaml
networks:
  app-network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.28.0.0/16
          ip_range: 172.28.5.0/24
          gateway: 172.28.5.254
```

### 고정 IP 할당

```yaml
services:
  db:
    networks:
      backend:
        ipv4_address: 172.28.5.10

networks:
  backend:
    ipam:
      config:
        - subnet: 172.28.5.0/24
```

### 네트워크 알리아스

```yaml
services:
  db:
    networks:
      backend:
        aliases:
          - database
          - postgres
          - db.local

# app에서 db, database, postgres, db.local 모두로 접근 가능
```

## 외부 네트워크 연결

### 기존 네트워크 사용

```yaml
networks:
  existing-network:
    external: true
    name: my-existing-network
```

### 다른 Compose 프로젝트와 연결

```yaml
# Project A: docker-compose.yml
networks:
  shared:
    name: shared-network
    driver: bridge

# Project B: docker-compose.yml
networks:
  shared:
    external: true
    name: shared-network
```

## 보안 구성 예시

```yaml
version: '3.8'

services:
  # 리버스 프록시 (DMZ)
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    networks:
      - dmz
      - app-tier

  # 애플리케이션 (중간 계층)
  app:
    build: .
    networks:
      - app-tier
      - data-tier
    # 외부 포트 노출 없음

  # 데이터베이스 (내부 계층)
  db:
    image: postgres:16-alpine
    networks:
      - data-tier
    # 내부 네트워크만 접근 가능

  redis:
    image: redis:7-alpine
    networks:
      - data-tier

networks:
  # 외부 접근 가능
  dmz:
    driver: bridge

  # 애플리케이션 계층
  app-tier:
    driver: bridge
    internal: true

  # 데이터 계층 (완전 격리)
  data-tier:
    driver: bridge
    internal: true
```

## DNS 및 서비스 디스커버리

Docker Compose는 자동으로 서비스 이름으로 DNS를 설정합니다:

```yaml
services:
  app:
    environment:
      # 서비스 이름으로 접근
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0

  db:
    # 'db'로 접근 가능
    image: postgres:16-alpine

  redis:
    # 'redis'로 접근 가능
    image: redis:7-alpine
```
