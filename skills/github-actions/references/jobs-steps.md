# Jobs와 Steps 구성

GitHub Actions의 Job과 Step 구성 방법입니다.

## 기본 구조

```yaml
jobs:
  job-id:
    name: 표시될 이름
    runs-on: ubuntu-latest
    steps:
      - name: Step 이름
        run: echo "Hello"
```

## Runner 선택

```yaml
jobs:
  build:
    # GitHub-hosted runners
    runs-on: ubuntu-latest      # Ubuntu 22.04
    runs-on: ubuntu-22.04       # 특정 버전
    runs-on: windows-latest     # Windows
    runs-on: macos-latest       # macOS
    
    # Self-hosted runner
    runs-on: self-hosted
    runs-on: [self-hosted, linux, x64]
```

## Job 의존성

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Build"

  test:
    needs: build  # build 완료 후 실행
    runs-on: ubuntu-latest
    steps:
      - run: echo "Test"

  deploy:
    needs: [build, test]  # 둘 다 완료 후 실행
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploy"
```

## 조건부 실행

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Only on main
        if: github.event_name == 'push'
        run: echo "Deploying"

      - name: Always run
        if: always()
        run: echo "Cleanup"

      - name: On failure
        if: failure()
        run: echo "Handle failure"

      - name: On success
        if: success()
        run: echo "Success!"
```

## 환경 변수

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    # Job 레벨 환경 변수
    env:
      NODE_ENV: production
    
    steps:
      - name: Step with env
        env:
          STEP_VAR: value
        run: |
          echo "NODE_ENV: $NODE_ENV"
          echo "STEP_VAR: $STEP_VAR"
          echo "Context: ${{ env.NODE_ENV }}"
```

## 출력 전달

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - name: Get version
        id: get-version
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  job2:
    needs: job1
    runs-on: ubuntu-latest
    steps:
      - name: Use version
        run: echo "Version: ${{ needs.job1.outputs.version }}"
```

## 타임아웃

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # Job 타임아웃
    steps:
      - name: Long task
        timeout-minutes: 10  # Step 타임아웃
        run: ./long-running-script.sh
```

## 서비스 컨테이너

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
        run: pytest
```

## 동시성 제어

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: true  # 이전 실행 취소
```

## 권한 설정

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      pull-requests: write
```
