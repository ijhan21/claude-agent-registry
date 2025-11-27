---
name: github-actions
description: GitHub Actions 워크플로우 작성 가이드. 다음 상황에서 활성화: (1) CI/CD 파이프라인 - "GitHub Actions", "워크플로우", "CI 설정", "CD 파이프라인" 언급 시, (2) 자동화 - "자동 배포", "자동 테스트", "PR 자동화" 언급 시, (3) Actions 문법 - "workflow 문법", "jobs", "steps", "matrix" 언급 시.
version: 1.0.0
---

# GitHub Actions 워크플로우 가이드

GitHub Actions를 사용한 CI/CD 파이프라인 구축 방법을 다룹니다.

## 핵심 원칙

1. **재사용성**: Composite Actions와 Reusable Workflows 활용
2. **보안**: Secrets 안전한 관리, 최소 권한 원칙
3. **효율성**: 캐싱, 병렬 실행, 조건부 실행
4. **가시성**: 명확한 Job/Step 이름, 상태 배지
5. **유지보수성**: 모듈화, 버전 고정

## 기본 구조

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup
        run: echo "Setup step"
      - name: Build
        run: echo "Build step"
```

## 참조 문서

상세한 설정은 `references/` 디렉토리를 참조하세요:

| 문서 | 내용 |
|------|------|
| [triggers.md](references/triggers.md) | 이벤트 트리거 설정 |
| [jobs-steps.md](references/jobs-steps.md) | Jobs와 Steps 구성 |
| [matrix.md](references/matrix.md) | Matrix 전략 |
| [caching.md](references/caching.md) | 캐싱 전략 |
| [secrets.md](references/secrets.md) | Secrets 관리 |
| [environments.md](references/environments.md) | 환경 및 배포 |
| [reusable.md](references/reusable.md) | 재사용 가능한 워크플로우 |
| [python-ci.md](references/python-ci.md) | Python CI 예제 |
| [node-ci.md](references/node-ci.md) | Node.js CI 예제 |
| [docker-ci.md](references/docker-ci.md) | Docker CI/CD 예제 |

## 빠른 시작: Python CI

```yaml
name: Python CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Run tests
        run: pytest
```

## 빠른 시작: Node.js CI

```yaml
name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
```

## 빠른 시작: Docker Build & Push

```yaml
name: Docker

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: user/app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 주요 Actions

| Action | 용도 |
|--------|------|
| `actions/checkout@v4` | 코드 체크아웃 |
| `actions/setup-python@v5` | Python 설정 |
| `actions/setup-node@v4` | Node.js 설정 |
| `actions/cache@v4` | 캐싱 |
| `docker/build-push-action@v6` | Docker 빌드/푸시 |
| `aws-actions/configure-aws-credentials@v4` | AWS 인증 |

## 품질 체크리스트

### 기본
- [ ] 워크플로우 이름이 명확한가
- [ ] 트리거 조건이 적절한가
- [ ] Job/Step 이름이 설명적인가
- [ ] 타임아웃이 설정되어 있는가

### 보안
- [ ] Secrets가 안전하게 관리되는가
- [ ] 최소 권한 원칙이 적용되었는가
- [ ] Third-party actions가 버전 고정되었는가

### 효율
- [ ] 캐싱이 적용되었는가
- [ ] 불필요한 실행이 방지되는가
- [ ] 병렬 실행이 활용되는가
