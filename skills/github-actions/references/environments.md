# 환경 및 배포 설정

GitHub Environments를 활용한 배포 관리입니다.

## 환경 생성

Repository Settings → Environments에서 생성

## 기본 사용

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: ./deploy.sh staging

  deploy-production:
    runs-on: ubuntu-latest
    environment: production
    needs: deploy-staging
    steps:
      - name: Deploy to production
        run: ./deploy.sh production
```

## 환경 URL

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh
```

## 동적 환경 이름

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: preview-${{ github.event.pull_request.number }}
      url: https://pr-${{ github.event.pull_request.number }}.preview.example.com
```

## 승인 필요 배포

Environment Settings에서 "Required reviewers" 설정

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # 승인 필요
    steps:
      - run: ./deploy.sh
```

## 대기 타이머

Environment Settings에서 "Wait timer" 설정 (최대 30일)

## 브랜치 제한

Environment Settings에서 "Deployment branches" 설정:
- All branches
- Protected branches only
- Selected branches

## 전체 배포 파이프라인

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - name: Deploy to Staging
        run: ./deploy.sh staging

  test-staging:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: Run E2E tests
        run: npm run test:e2e -- --url=https://staging.example.com

  deploy-production:
    needs: test-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - name: Deploy to Production
        run: ./deploy.sh production
```

## 롤백 전략

```yaml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Rollback
        run: ./rollback.sh ${{ inputs.version }}
```
