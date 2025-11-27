# 볼륨 및 데이터 지속성

Docker Compose에서 볼륨을 효과적으로 구성하는 방법입니다.

## 볼륨 유형

### 1. Named Volumes (권장)

```yaml
services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    name: ${COMPOSE_PROJECT_NAME}_postgres_data
```

### 2. Bind Mounts

```yaml
services:
  app:
    volumes:
      # 절대 경로
      - /host/path:/container/path
      # 상대 경로 (프로젝트 기준)
      - ./src:/app/src
      # 읽기 전용
      - ./config:/app/config:ro
```

### 3. tmpfs Mounts

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /app/cache:size=100M
```

## 환경별 볼륨 구성

### 기본 구성 (docker-compose.yml)

```yaml
version: '3.8'

services:
  app:
    volumes:
      - app_logs:/app/logs

  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init:/docker-entrypoint-initdb.d:ro

  redis:
    volumes:
      - redis_data:/data

volumes:
  app_logs:
  postgres_data:
  redis_data:
```

### 로컬 개발 (docker-compose.local.yml)

```yaml
services:
  app:
    volumes:
      # 코드 마운트 (Hot Reload)
      - .:/app
      # 의존성 캐시 (마운트에서 제외)
      - /app/node_modules
      - /app/.venv
      - /app/__pycache__
```

### 프로덕션 (docker-compose.prod.yml)

```yaml
services:
  app:
    # 코드 마운트 없음
    volumes:
      - app_logs:/app/logs

  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data
      # 백업 디렉토리
      - /backup/postgres:/backup

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgres  # 호스트의 전용 스토리지
```

## 볼륨 옵션

### 외부 볼륨 사용

```yaml
volumes:
  postgres_data:
    external: true
    name: production_postgres_data
```

### 볼륨 드라이버 옵션

```yaml
volumes:
  # NFS 볼륨
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs.example.com,rw
      device: ":/path/to/share"

  # 로컬 바인드
  local_bind:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/myapp
```

## 데이터 백업 전략

### PostgreSQL 백업 서비스

```yaml
services:
  db-backup:
    image: postgres:16-alpine
    profiles: ["backup"]
    volumes:
      - postgres_data:/var/lib/postgresql/data:ro
      - ./backups:/backups
    environment:
      - PGPASSWORD=${POSTGRES_PASSWORD}
    command: >
      sh -c "pg_dump -h db -U ${POSTGRES_USER} ${POSTGRES_DB} 
             > /backups/backup_$$(date +%Y%m%d_%H%M%S).sql"
    depends_on:
      - db
    networks:
      - app-network
```

### 백업 스크립트

```bash
#!/bin/bash
# scripts/backup.sh

BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d_%H%M%S)

# 디렉토리 생성
mkdir -p $BACKUP_DIR

# PostgreSQL 백업
docker compose exec -T db pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} \
  > "$BACKUP_DIR/postgres_$DATE.sql"

# Redis 백업
docker compose exec redis redis-cli BGSAVE
docker cp $(docker compose ps -q redis):/data/dump.rdb \
  "$BACKUP_DIR/redis_$DATE.rdb"

# 오래된 백업 삭제 (7일 이상)
find $BACKUP_DIR -type f -mtime +7 -delete

echo "✅ Backup completed: $DATE"
```

## 볼륨 관리 명령어

```bash
# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect myapp_postgres_data

# 사용하지 않는 볼륨 삭제
docker volume prune

# 특정 볼륨 삭제
docker volume rm myapp_postgres_data

# 볼륨 생성
docker volume create --name myapp_data
```

## 주의사항

### 개발 환경에서 의존성 캐싱

```yaml
# 잘못된 예 - node_modules가 호스트의 것으로 덮어쓰임
volumes:
  - .:/app

# 올바른 예 - node_modules 제외
volumes:
  - .:/app
  - /app/node_modules  # 익명 볼륨으로 마스킹
```

### 권한 문제

```yaml
# 컨테이너 내부 사용자와 호스트 사용자 UID 매칭
services:
  app:
    user: "${UID:-1000}:${GID:-1000}"
    volumes:
      - .:/app
```
