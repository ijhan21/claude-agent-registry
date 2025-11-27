# 실행 스크립트

환경별 Docker Compose 실행을 간편하게 하는 스크립트입니다.

## 통합 실행 스크립트

```bash
#!/bin/bash
# scripts/docker.sh - 통합 Docker Compose 관리 스크립트

set -e

# ===================
# 설정
# ===================
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
cd "$PROJECT_ROOT"

# 색상
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

# ===================
# 환경 설정
# ===================
ENV=${APP_ENV:-local}

case "$ENV" in
    local|dev)
        ENV_FILE=".env.local"
        COMPOSE_FILES="-f docker-compose.yml -f docker-compose.local.yml"
        ;;
    staging)
        ENV_FILE=".env.staging"
        COMPOSE_FILES="-f docker-compose.yml -f docker-compose.staging.yml"
        ;;
    prod|production)
        ENV_FILE=".env.prod"
        COMPOSE_FILES="-f docker-compose.yml -f docker-compose.prod.yml"
        ;;
    *)
        echo -e "${RED}Unknown environment: $ENV${NC}"
        exit 1
        ;;
esac

# 환경 변수 로드
if [ -f "$ENV_FILE" ]; then
    set -a
    source "$ENV_FILE"
    set +a
fi

# ===================
# 함수
# ===================
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

compose() {
    docker compose --env-file "$ENV_FILE" $COMPOSE_FILES "$@"
}

# ===================
# 커맨드
# ===================
cmd_up() {
    log_info "Starting services (ENV=$ENV)..."
    compose up -d "$@"
    log_info "Services started!"
    cmd_status
}

cmd_down() {
    log_info "Stopping services..."
    compose down "$@"
}

cmd_restart() {
    log_info "Restarting services..."
    compose restart "$@"
}

cmd_status() {
    compose ps
}

cmd_logs() {
    compose logs -f "${@:-app}"
}

cmd_shell() {
    local service=${1:-app}
    compose exec "$service" /bin/bash || compose exec "$service" /bin/sh
}

cmd_build() {
    log_info "Building images..."
    compose build "$@"
}

cmd_rebuild() {
    log_info "Rebuilding images (no cache)..."
    compose build --no-cache "$@"
    compose up -d
}

cmd_clean() {
    log_warn "This will remove all containers, networks, and volumes."
    read -p "Are you sure? (y/N) " -n 1 -r
    echo
    if [[ $REPLY =~ ^[Yy]$ ]]; then
        compose down -v --remove-orphans
        docker system prune -f
        log_info "Cleanup complete!"
    fi
}

cmd_test() {
    log_info "Running tests..."
    compose exec app pytest "$@"
}

cmd_migrate() {
    log_info "Running migrations..."
    compose exec app python manage.py migrate "$@"  # Django
    # compose exec app alembic upgrade head "$@"    # FastAPI + Alembic
}

cmd_backup() {
    log_info "Creating backup..."
    local backup_dir="./backups"
    local date=$(date +%Y%m%d_%H%M%S)
    mkdir -p "$backup_dir"
    
    compose exec -T db pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
        > "$backup_dir/db_$date.sql"
    
    log_info "Backup created: $backup_dir/db_$date.sql"
}

cmd_help() {
    cat << EOF
Docker Compose Management Script

Usage: $0 <command> [options]

Environment: $ENV (set APP_ENV to change)

Commands:
    up [service]     Start services
    down             Stop services
    restart          Restart services
    status           Show service status
    logs [service]   View logs (default: app)
    shell [service]  Open shell (default: app)
    build            Build images
    rebuild          Build images without cache
    clean            Remove all containers and volumes
    test [args]      Run tests
    migrate          Run database migrations
    backup           Backup database
    help             Show this help

Examples:
    APP_ENV=staging $0 up
    $0 logs db
    $0 shell app
EOF
}

# ===================
# 메인
# ===================
case "${1:-help}" in
    up)       shift; cmd_up "$@" ;;
    down)     shift; cmd_down "$@" ;;
    restart)  shift; cmd_restart "$@" ;;
    status)   cmd_status ;;
    logs)     shift; cmd_logs "$@" ;;
    shell)    shift; cmd_shell "$@" ;;
    build)    shift; cmd_build "$@" ;;
    rebuild)  shift; cmd_rebuild "$@" ;;
    clean)    cmd_clean ;;
    test)     shift; cmd_test "$@" ;;
    migrate)  cmd_migrate ;;
    backup)   cmd_backup ;;
    help|*)   cmd_help ;;
esac
```

## Makefile 대안

```makefile
# Makefile
.PHONY: up down restart logs shell build test clean help

ENV ?= local
SERVICE ?= app

ifeq ($(ENV),local)
    ENV_FILE = .env.local
    COMPOSE_FILES = -f docker-compose.yml -f docker-compose.local.yml
else ifeq ($(ENV),staging)
    ENV_FILE = .env.staging
    COMPOSE_FILES = -f docker-compose.yml -f docker-compose.staging.yml
else ifeq ($(ENV),prod)
    ENV_FILE = .env.prod
    COMPOSE_FILES = -f docker-compose.yml -f docker-compose.prod.yml
endif

COMPOSE = docker compose --env-file $(ENV_FILE) $(COMPOSE_FILES)

up:
	@echo "Starting services (ENV=$(ENV))..."
	$(COMPOSE) up -d

down:
	$(COMPOSE) down

restart:
	$(COMPOSE) restart

logs:
	$(COMPOSE) logs -f $(SERVICE)

shell:
	$(COMPOSE) exec $(SERVICE) /bin/bash || $(COMPOSE) exec $(SERVICE) /bin/sh

build:
	$(COMPOSE) build

rebuild:
	$(COMPOSE) build --no-cache
	$(COMPOSE) up -d

test:
	$(COMPOSE) exec app pytest

clean:
	$(COMPOSE) down -v --remove-orphans
	docker system prune -f

help:
	@echo "Usage: make <target> [ENV=local|staging|prod] [SERVICE=app]"
	@echo ""
	@echo "Targets:"
	@echo "  up       - Start services"
	@echo "  down     - Stop services"
	@echo "  restart  - Restart services"
	@echo "  logs     - View logs"
	@echo "  shell    - Open shell"
	@echo "  build    - Build images"
	@echo "  rebuild  - Build without cache"
	@echo "  test     - Run tests"
	@echo "  clean    - Remove all"
```

## 사용 예시

```bash
# Bash 스크립트
./scripts/docker.sh up
APP_ENV=staging ./scripts/docker.sh up
./scripts/docker.sh logs db
./scripts/docker.sh shell

# Makefile
make up
make up ENV=staging
make logs SERVICE=db
make shell
```
