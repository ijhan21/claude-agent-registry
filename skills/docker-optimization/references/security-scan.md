# 취약점 스캔 (Trivy)

## Trivy 소개

- Aqua Security 오픈소스
- OS 패키지 + 언어별 의존성 스캔
- Dockerfile/IaC 설정 스캔
- SBOM 생성

## 설치

```bash
# Ubuntu/Debian
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# macOS
brew install trivy

# Docker로 실행
docker run --rm aquasec/trivy image myapp:latest
```

## 기본 사용법

### 이미지 스캔

```bash
# 기본 스캔
trivy image myapp:latest

# 심각도 필터
trivy image --severity HIGH,CRITICAL myapp:latest

# 수정 가능한 취약점만
trivy image --ignore-unfixed myapp:latest

# 특정 취약점 무시
trivy image --ignore-file .trivyignore myapp:latest
```

### Dockerfile 스캔

```bash
# Dockerfile 설정 검사
trivy config Dockerfile.prod

# 디렉토리 전체 스캔
trivy config .
```

### 파일시스템 스캔

```bash
# 프로젝트 의존성 스캔
trivy fs .

# requirements.txt 스캔
trivy fs --scanners vuln requirements.txt

# package-lock.json 스캔
trivy fs --scanners vuln package-lock.json
```

## CI/CD 통합

### 심각 취약점 시 실패

```bash
# CRITICAL 발견 시 exit code 1
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# HIGH 이상 발견 시 실패
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest
```

### GitHub Actions

```yaml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -f Dockerfile.prod -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### SARIF 출력 (GitHub Security)

```yaml
- name: Trivy scan with SARIF
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
```

## 무시 파일 (.trivyignore)

```
# .trivyignore

# 특정 CVE 무시 (이유 명시)
CVE-2023-12345  # False positive, 사용하지 않는 기능

# 특정 패키지 무시
pkg:npm/lodash@4.17.20

# 만료일 설정
CVE-2023-67890 exp:2024-12-31  # 임시 무시, 연말까지 수정 예정
```

## 출력 형식

```bash
# 테이블 (기본)
trivy image myapp:latest

# JSON
trivy image -f json -o results.json myapp:latest

# SARIF (GitHub 연동)
trivy image -f sarif -o results.sarif myapp:latest

# HTML 리포트
trivy image -f template --template "@contrib/html.tpl" -o report.html myapp:latest
```

## 스캔 스크립트

```bash
#!/bin/bash
# scan.sh

IMAGE=${1:-"myapp:latest"}
SEVERITY=${2:-"HIGH,CRITICAL"}

echo "=== Scanning ${IMAGE} ==="

# 이미지 스캔
trivy image \
  --severity ${SEVERITY} \
  --ignore-unfixed \
  --exit-code 1 \
  ${IMAGE}

EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
  echo "✅ No vulnerabilities found"
else
  echo "❌ Vulnerabilities detected"
fi

exit $EXIT_CODE
```

## 베이스 이미지 선택 가이드

```bash
# 베이스 이미지별 취약점 비교
trivy image python:3.12
trivy image python:3.12-slim
trivy image python:3.12-alpine

# slim 권장: 적은 패키지 = 적은 취약점
```

## 정기 스캔

```yaml
# .github/workflows/scheduled-scan.yml
name: Scheduled Security Scan

on:
  schedule:
    - cron: '0 0 * * 1'  # 매주 월요일

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - name: Scan production image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:latest
          severity: 'CRITICAL,HIGH'
```

## 캐시 활용

```bash
# DB 캐시 디렉토리 설정
export TRIVY_CACHE_DIR=/tmp/trivy-cache

# GitHub Actions 캐시
- uses: actions/cache@v3
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ github.run_id }}
    restore-keys: trivy-db-
```

## 권장 워크플로우

1. **로컬 개발**: 빌드 후 `trivy image` 실행
2. **PR**: CI에서 자동 스캔, HIGH 이상 실패
3. **main 병합**: 전체 스캔, SARIF 업로드
4. **정기 스캔**: 주간 스케줄, 새 취약점 알림
