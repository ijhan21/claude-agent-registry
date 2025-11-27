# 이미지 태깅 전략

## 태그 형식

```
{앱이름}:{버전}-{커밋해시}-{날짜}
```

예시:
- `myapp:1.0.0-abc1234-20241127`
- `myapp:latest`
- `myapp:develop`

## 태그 유형

| 태그 | 용도 | 예시 |
|------|------|------|
| `latest` | 최신 안정 버전 | `myapp:latest` |
| `{version}` | 시맨틱 버전 | `myapp:1.2.3` |
| `{version}-{commit}` | 정확한 빌드 추적 | `myapp:1.2.3-abc1234` |
| `{branch}` | 브랜치별 빌드 | `myapp:develop` |
| `{commit}` | 커밋별 빌드 | `myapp:abc1234` |

## 빌드 스크립트

### build.sh

```bash
#!/bin/bash
set -e

# 변수 설정
APP_NAME="myapp"
VERSION=${VERSION:-"0.0.0"}
COMMIT=$(git rev-parse --short HEAD)
DATE=$(date +%Y%m%d)
BRANCH=$(git rev-parse --abbrev-ref HEAD)

# 전체 태그
FULL_TAG="${VERSION}-${COMMIT}-${DATE}"

# 빌드
echo "Building ${APP_NAME}:${FULL_TAG}"
docker build \
  -f Dockerfile.prod \
  -t ${APP_NAME}:${FULL_TAG} \
  -t ${APP_NAME}:${VERSION} \
  -t ${APP_NAME}:${COMMIT} \
  .

# main 브랜치면 latest 태그
if [ "$BRANCH" = "main" ]; then
  docker tag ${APP_NAME}:${FULL_TAG} ${APP_NAME}:latest
  echo "Tagged as latest"
fi

# develop 브랜치면 develop 태그
if [ "$BRANCH" = "develop" ]; then
  docker tag ${APP_NAME}:${FULL_TAG} ${APP_NAME}:develop
  echo "Tagged as develop"
fi

echo "Build complete: ${APP_NAME}:${FULL_TAG}"
```

## Makefile

```makefile
APP_NAME := myapp
VERSION := $(shell cat VERSION 2>/dev/null || echo "0.0.0")
COMMIT := $(shell git rev-parse --short HEAD)
DATE := $(shell date +%Y%m%d)
FULL_TAG := $(VERSION)-$(COMMIT)-$(DATE)

.PHONY: build build-dev tag-latest

build:
	docker build -f Dockerfile.prod \
		-t $(APP_NAME):$(FULL_TAG) \
		-t $(APP_NAME):$(VERSION) \
		-t $(APP_NAME):$(COMMIT) \
		.

build-dev:
	docker build -f Dockerfile.dev \
		-t $(APP_NAME):dev \
		.

tag-latest:
	docker tag $(APP_NAME):$(FULL_TAG) $(APP_NAME):latest

info:
	@echo "App: $(APP_NAME)"
	@echo "Version: $(VERSION)"
	@echo "Commit: $(COMMIT)"
	@echo "Full Tag: $(FULL_TAG)"
```

## 버전 파일

### VERSION

```
1.2.3
```

### package.json (Node.js)

```json
{
  "name": "myapp",
  "version": "1.2.3"
}
```

### pyproject.toml (Python)

```toml
[project]
name = "myapp"
version = "1.2.3"
```

## 자동 버전 추출

```bash
# package.json에서 버전 추출
VERSION=$(node -p "require('./package.json').version")

# pyproject.toml에서 버전 추출
VERSION=$(grep "^version" pyproject.toml | cut -d'"' -f2)

# VERSION 파일에서 추출
VERSION=$(cat VERSION)
```

## GitHub Actions

```yaml
name: Build and Tag

on:
  push:
    branches: [main, develop]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Extract metadata
        id: meta
        run: |
          VERSION=$(cat VERSION || echo "0.0.0")
          COMMIT=${GITHUB_SHA::7}
          DATE=$(date +%Y%m%d)
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "commit=${COMMIT}" >> $GITHUB_OUTPUT
          echo "full_tag=${VERSION}-${COMMIT}-${DATE}" >> $GITHUB_OUTPUT

      - name: Build image
        run: |
          docker build -f Dockerfile.prod \
            -t myapp:${{ steps.meta.outputs.full_tag }} \
            -t myapp:${{ steps.meta.outputs.version }} \
            -t myapp:${{ steps.meta.outputs.commit }} \
            .

      # main 브랜치: latest 태그
      - name: Tag latest
        if: github.ref == 'refs/heads/main'
        run: docker tag myapp:${{ steps.meta.outputs.full_tag }} myapp:latest

      # 릴리스 태그: 버전 태그
      - name: Tag release
        if: startsWith(github.ref, 'refs/tags/v')
        run: |
          RELEASE_VERSION=${GITHUB_REF#refs/tags/v}
          docker tag myapp:${{ steps.meta.outputs.full_tag }} myapp:${RELEASE_VERSION}
```

## 태그 정리

```bash
# 오래된 이미지 정리 (30일 이상)
docker images --format "{{.Repository}}:{{.Tag}} {{.CreatedAt}}" | \
  while read image date; do
    created=$(date -d "$date" +%s 2>/dev/null || echo 0)
    now=$(date +%s)
    age=$(( (now - created) / 86400 ))
    if [ $age -gt 30 ]; then
      docker rmi "$image" 2>/dev/null || true
    fi
  done

# dangling 이미지 정리
docker image prune -f
```

## 권장 워크플로우

1. **개발**: `myapp:dev` (로컬)
2. **PR/테스트**: `myapp:{commit}` (CI)
3. **develop 브랜치**: `myapp:develop` (스테이징)
4. **main 브랜치**: `myapp:latest` (프로덕션 준비)
5. **릴리스**: `myapp:1.2.3` (프로덕션)
