# CI/CD 패턴 (GitHub Actions)

## 기본 워크플로우

```yaml
# .github/workflows/docker-build.yml
name: Docker Build & Deploy

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: myapp

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Extract metadata
        id: meta
        run: |
          VERSION=$(cat VERSION 2>/dev/null || echo "0.0.0")
          COMMIT=${GITHUB_SHA::7}
          DATE=$(date +%Y%m%d)
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "commit=${COMMIT}" >> $GITHUB_OUTPUT
          echo "full_tag=${VERSION}-${COMMIT}-${DATE}" >> $GITHUB_OUTPUT

      - name: Build image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          push: false
          load: true
          tags: |
            ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.full_tag }}
            ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.commit }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.commit }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      - name: Save image
        if: github.ref == 'refs/heads/main'
        run: |
          docker save ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.full_tag }} | gzip > image.tar.gz

      - name: Upload artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: image.tar.gz
          retention-days: 1
```

## 배포 서버에서 빌드

### SSH 배포 워크플로우

```yaml
# .github/workflows/deploy.yml
name: Deploy to Server

on:
  push:
    branches: [main]
  workflow_dispatch:  # 수동 트리거

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/myapp
            
            # 최신 코드 가져오기
            git pull origin main
            
            # 이미지 빌드
            export DOCKER_BUILDKIT=1
            VERSION=$(cat VERSION)
            COMMIT=$(git rev-parse --short HEAD)
            DATE=$(date +%Y%m%d)
            TAG="${VERSION}-${COMMIT}-${DATE}"
            
            docker build \
              -f Dockerfile.prod \
              --build-arg BUILDKIT_INLINE_CACHE=1 \
              -t myapp:${TAG} \
              -t myapp:latest .
            
            # 배포
            docker compose -f docker-compose.prod.yml up -d
            
            # 정리
            docker image prune -f
```

## 배포 스크립트

### deploy.sh

```bash
#!/bin/bash
set -e

APP_NAME="myapp"
COMPOSE_FILE="docker-compose.prod.yml"

echo "=== Deploying ${APP_NAME} ==="

# 최신 코드
git pull origin main

# 버전 정보
VERSION=$(cat VERSION 2>/dev/null || echo "0.0.0")
COMMIT=$(git rev-parse --short HEAD)
DATE=$(date +%Y%m%d)
TAG="${VERSION}-${COMMIT}-${DATE}"

echo "Building ${APP_NAME}:${TAG}"

# BuildKit 활성화
export DOCKER_BUILDKIT=1

# 빌드
docker build \
  -f Dockerfile.prod \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t ${APP_NAME}:${TAG} \
  -t ${APP_NAME}:latest \
  .

# 취약점 스캔 (선택)
if command -v trivy &> /dev/null; then
  echo "Scanning for vulnerabilities..."
  trivy image --severity HIGH,CRITICAL ${APP_NAME}:${TAG} || true
fi

# 배포
echo "Deploying..."
docker compose -f ${COMPOSE_FILE} up -d

# 헬스체크 대기
echo "Waiting for health check..."
sleep 10

# 상태 확인
docker compose -f ${COMPOSE_FILE} ps

# 오래된 이미지 정리
docker image prune -f

echo "=== Deploy complete: ${APP_NAME}:${TAG} ==="
```

## 환경별 빌드

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [staging, production]
    
    steps:
      - uses: actions/checkout@v4

      - name: Build for ${{ matrix.environment }}
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          build-args: |
            ENVIRONMENT=${{ matrix.environment }}
            NEXT_PUBLIC_API_URL=${{ matrix.environment == 'production' && 'https://api.example.com' || 'https://staging-api.example.com' }}
          tags: myapp:${{ matrix.environment }}
```

## PR 체크

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    branches: [main, develop]

jobs:
  docker-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile.prod

  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build (no push)
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.prod
          push: false
          cache-from: type=gha

  security-scan:
    runs-on: ubuntu-latest
    needs: build-test
    steps:
      - uses: actions/checkout@v4
      
      - name: Scan Dockerfile
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
```

## 캐시 전략

```yaml
# 최적 캐시 설정
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    file: Dockerfile.prod
    push: false
    # GitHub Actions 캐시 사용
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Webhook 배포

```yaml
# 배포 서버에 webhook 전송
- name: Trigger deploy
  if: github.ref == 'refs/heads/main'
  run: |
    curl -X POST \
      -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
      -H "Content-Type: application/json" \
      -d '{"ref": "${{ github.sha }}", "tag": "${{ steps.meta.outputs.full_tag }}"}' \
      ${{ secrets.DEPLOY_WEBHOOK_URL }}
```

## Secrets 설정

GitHub 리포지토리 Settings > Secrets:

| Secret | 설명 |
|--------|------|
| `SERVER_HOST` | 배포 서버 IP/도메인 |
| `SERVER_USER` | SSH 사용자 |
| `SERVER_SSH_KEY` | SSH 프라이빗 키 |
| `DEPLOY_TOKEN` | 배포 인증 토큰 |

## 롤백 스크립트

```bash
#!/bin/bash
# rollback.sh

PREVIOUS_TAG=${1:-""}

if [ -z "$PREVIOUS_TAG" ]; then
  # 이전 이미지 찾기
  PREVIOUS_TAG=$(docker images myapp --format "{{.Tag}}" | grep -v latest | head -2 | tail -1)
fi

echo "Rolling back to myapp:${PREVIOUS_TAG}"

docker tag myapp:${PREVIOUS_TAG} myapp:latest
docker compose -f docker-compose.prod.yml up -d

echo "Rollback complete"
```
