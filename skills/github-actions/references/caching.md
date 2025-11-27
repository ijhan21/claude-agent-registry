# 캐싱 전략

의존성과 빌드 결과물을 캐싱하여 워크플로우 속도를 개선합니다.

## Setup Actions 내장 캐시

```yaml
# Python
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'  # pip 캐시 자동

# Node.js
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # npm, yarn, pnpm 지원

# Go
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true
```

## 수동 캐시 설정

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

## 캐시 키 전략

```yaml
# 정확한 매칭
key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

# 폴백 키 (부분 매칭)
restore-keys: |
  ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
  ${{ runner.os }}-pip-
  ${{ runner.os }}-
```

## 언어별 캐시 경로

### Python (pip)

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      .venv
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
```

### Node.js (npm)

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
```

### Node.js (pnpm)

```yaml
- name: Get pnpm store
  id: pnpm-cache
  run: echo "dir=$(pnpm store path)" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-cache.outputs.dir }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
```

## Docker 레이어 캐시

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: user/app:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## 캐시 히트 확인

```yaml
- name: Cache dependencies
  id: cache
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-npm-${{ hashFiles('package-lock.json') }}

- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

## 빌드 아티팩트 캐시

```yaml
# 빌드 결과 저장
- uses: actions/cache@v4
  with:
    path: |
      dist
      .next/cache
    key: ${{ runner.os }}-build-${{ github.sha }}

# 또는 Artifacts 사용 (Job 간 전달)
- uses: actions/upload-artifact@v4
  with:
    name: build
    path: dist/

- uses: actions/download-artifact@v4
  with:
    name: build
    path: dist/
```

## 캐시 제한

- 저장소당 최대 10GB
- 7일간 미사용 시 자동 삭제
- 캐시 크기가 크면 저장/복원 시간 증가
