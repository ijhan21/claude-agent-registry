# Secrets 관리

GitHub Actions에서 민감한 정보를 안전하게 관리하는 방법입니다.

## Secrets 설정 위치

1. **Repository Secrets**: Settings → Secrets and variables → Actions
2. **Environment Secrets**: Settings → Environments → [환경] → Secrets
3. **Organization Secrets**: Organization Settings → Secrets

## 기본 사용

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: ./deploy.sh

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

## 환경별 Secrets

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # 환경 지정
    steps:
      - name: Deploy
        env:
          # production 환경의 secret 사용
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: ./deploy.sh
```

## GitHub Token

```yaml
# 자동 제공되는 GITHUB_TOKEN
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - name: Create Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: gh release create v1.0.0
```

## 권한 설정

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read       # 기본
      packages: write      # 패키지 푸시
      pull-requests: write # PR 코멘트
      issues: write        # Issue 생성
      id-token: write      # OIDC
```

## OIDC (OpenID Connect)

Secrets 없이 클라우드 인증:

```yaml
# AWS
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActions
          aws-region: ap-northeast-2
```

## Secret 마스킹

```yaml
- name: Mask secret
  run: |
    SECRET_VALUE="sensitive-data"
    echo "::add-mask::$SECRET_VALUE"
    echo "Using $SECRET_VALUE"
```

## Secret 검증

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Check secrets
        run: |
          if [ -z "${{ secrets.API_KEY }}" ]; then
            echo "::error::API_KEY is not set"
            exit 1
          fi
```

## 보안 모범 사례

1. **최소 권한**: 필요한 권한만 부여
2. **환경 분리**: 프로덕션 secrets는 환경으로 분리
3. **주기적 교체**: 정기적으로 secrets 갱신
4. **OIDC 사용**: 가능하면 OIDC로 클라우드 인증
5. **감사 로그**: Secret 접근 기록 모니터링

```yaml
# 보안 강화 예시
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production  # 환경으로 분리
    permissions:
      contents: read
      id-token: write  # OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2
```
