# 이벤트 트리거 설정

GitHub Actions 워크플로우를 실행하는 다양한 트리거 방법입니다.

## Push / Pull Request

```yaml
# 기본
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# 브랜치 패턴
on:
  push:
    branches:
      - main
      - 'releases/**'
      - '!releases/**-alpha'  # 제외

# 경로 필터
on:
  push:
    paths:
      - 'src/**'
      - '!src/**/*.md'  # 제외
    paths-ignore:
      - 'docs/**'
      - '*.md'
```

## 태그

```yaml
on:
  push:
    tags:
      - 'v*'           # v1.0.0, v2.1.0 등
      - 'v[0-9]+.[0-9]+.[0-9]+'  # 정확한 semver
```

## 스케줄 (Cron)

```yaml
on:
  schedule:
    # 매일 자정 (UTC)
    - cron: '0 0 * * *'
    # 매주 월요일 9시 (UTC)
    - cron: '0 9 * * 1'
    # 매시간
    - cron: '0 * * * *'
```

## 수동 실행

```yaml
# 기본
on:
  workflow_dispatch:

# 입력 파라미터
on:
  workflow_dispatch:
    inputs:
      environment:
        description: '배포 환경'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      version:
        description: '버전'
        required: false
        type: string
      debug:
        description: '디버그 모드'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Version: ${{ inputs.version }}"
          echo "Debug: ${{ inputs.debug }}"
```

## 다른 워크플로우에서 호출

```yaml
# 호출 가능한 워크플로우
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy_key:
        required: true
```

## 릴리스

```yaml
on:
  release:
    types: [published, created, released]
```

## Issue / PR 이벤트

```yaml
on:
  issues:
    types: [opened, edited, labeled]
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  pull_request_review:
    types: [submitted]
  issue_comment:
    types: [created]
```

## 복합 트리거

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 매주 일요일
  workflow_dispatch:
```

## 조건부 실행

```yaml
jobs:
  deploy:
    # main 브랜치 push일 때만
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
  skip-draft:
    # Draft PR이 아닐 때만
    if: github.event.pull_request.draft == false
    
  only-on-tag:
    # 태그일 때만
    if: startsWith(github.ref, 'refs/tags/')
```
