# 비밀 관리

Docker Compose에서 민감 정보를 안전하게 관리하는 방법입니다.

## 환경별 비밀 관리 전략

| 환경 | 방법 | 보안 수준 |
|------|------|----------|
| Local | .env 파일 | 낮음 |
| Staging | 환경 변수 + CI/CD | 중간 |
| Production | Docker Secrets / Vault | 높음 |

## 로컬 환경: .env 파일

```bash
# .env.local (개발용 - gitignore)
POSTGRES_PASSWORD=devpass
REDIS_PASSWORD=
SECRET_KEY=dev-secret-key-not-for-production
```

```yaml
# docker-compose.local.yml
services:
  app:
    env_file:
      - .env.local
```

## 스테이징: CI/CD 주입

### GitHub Actions

```yaml
# .github/workflows/staging.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Create env file
        run: |
          cat > .env.staging << EOF
          POSTGRES_PASSWORD=${{ secrets.STAGING_DB_PASSWORD }}
          SECRET_KEY=${{ secrets.STAGING_SECRET_KEY }}
          SENTRY_DSN=${{ secrets.STAGING_SENTRY_DSN }}
          EOF
      
      - name: Deploy
        run: |
          docker compose -f docker-compose.yml -f docker-compose.staging.yml up -d
```

## 프로덕션: Docker Secrets

### Secret 생성

```bash
# 파일에서 생성
echo "super_secure_password" | docker secret create db_password -

# 파일로 생성
docker secret create db_password ./secrets/db_password.txt

# 보기
docker secret ls
```

### Compose에서 사용

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    secrets:
      - db_password
      - secret_key
    environment:
      - DATABASE_PASSWORD_FILE=/run/secrets/db_password

  db:
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    external: true
  secret_key:
    external: true
```

### 애플리케이션에서 읻d기

```python
# Python
import os

def get_secret(name: str) -> str:
    """Docker Secret 또는 환경 변수에서 값 읻d기"""
    # Docker Secret 파일 확인
    secret_file = f"/run/secrets/{name}"
    if os.path.exists(secret_file):
        with open(secret_file, 'r') as f:
            return f.read().strip()
    
    # _FILE 환경 변수 확인
    file_env = os.environ.get(f"{name.upper()}_FILE")
    if file_env and os.path.exists(file_env):
        with open(file_env, 'r') as f:
            return f.read().strip()
    
    # 일반 환경 변수
    return os.environ.get(name.upper(), "")

# 사용
DATABASE_PASSWORD = get_secret("db_password")
SECRET_KEY = get_secret("secret_key")
```

## 파일 기반 Secret (Swarm 없이)

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    secrets:
      - db_password
      - secret_key

  db:
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  secret_key:
    file: ./secrets/secret_key.txt
```

### 시크릿 파일 생성 스크립트

```bash
#!/bin/bash
# scripts/init-secrets.sh

SECRETS_DIR="./secrets"
mkdir -p "$SECRETS_DIR"
chmod 700 "$SECRETS_DIR"

# 위험: 사용자 입력 또는 외부에서 가져와야 함
read -s -p "Enter DB password: " DB_PASS
echo
echo "$DB_PASS" > "$SECRETS_DIR/db_password.txt"

read -s -p "Enter Secret Key: " SECRET_KEY
echo
echo "$SECRET_KEY" > "$SECRETS_DIR/secret_key.txt"

# 파일 권한 설정
chmod 600 "$SECRETS_DIR"/*

echo "✅ Secrets initialized!"
```

## .gitignore 설정

```gitignore
# 환경 변수
.env
.env.local
.env.staging
.env.prod
.env.*.local

# 시크릿 파일
secrets/
*.key
*.pem
*.crt

# 예외
!.env.example
!secrets/.gitkeep
```

## 보안 체크리스트

- [ ] 프로덕션 비밀번호가 버전 관리에 포함되지 않음
- [ ] .env.example에는 실제 값이 없음
- [ ] 비밀번호가 충분히 강력함 (20자 이상, 특수문자 포함)
- [ ] 비밀번호 파일 권한이 600 이하
- [ ] 프로덕션에서 Docker Secrets 또는 외부 Vault 사용
