# .dockerignore 점검

## 중요성

- **빌드 컨텍스트 크기 감소** → 빌드 속도 향상
- **보안**: 민감한 파일 제외
- **이미지 크기**: 불필요한 파일 제외

## 빌드 컨텍스트 확인

```bash
# 현재 컨텍스트 크기 확인
du -sh --exclude=.git .

# 빌드 시 컨텍스트 크기 출력됨
# Sending build context to Docker daemon  2.048MB
```

## Python 프로젝트 .dockerignore

```dockerignore
# === Git ===
.git
.gitignore
.gitattributes

# === Python ===
__pycache__
*.py[cod]
*$py.class
*.so
.Python
.venv
venv
env
ENV

# === 패키지 ===
*.egg
*.egg-info
dist
build
eggs
parts
sdist
develop-eggs
.installed.cfg

# === 테스트/품질 ===
.pytest_cache
.coverage
.tox
.nox
htmlcov
.hypothesis

# === IDE ===
.idea
.vscode
*.swp
*.swo
*~

# === 환경 변수 ===
.env
.env.*
!.env.example

# === Docker ===
Dockerfile*
docker-compose*
.dockerignore

# === 문서/기타 ===
docs
*.md
!README.md
LICENSE
Makefile

# === 로그/임시 ===
*.log
logs
tmp
temp

# === macOS ===
.DS_Store
```

## Node.js 프로젝트 .dockerignore

```dockerignore
# === Git ===
.git
.gitignore

# === Dependencies ===
node_modules
npm-debug.log
yarn-debug.log
yarn-error.log

# === Build ===
.next
dist
build
out
.nuxt
.output

# === 테스트 ===
coverage
.nyc_output
__tests__
*.test.js
*.spec.js
jest.config.js

# === IDE ===
.idea
.vscode
*.swp

# === 환경 변수 ===
.env
.env.*
!.env.example

# === Docker ===
Dockerfile*
docker-compose*
.dockerignore

# === TypeScript ===
*.tsbuildinfo

# === 문서/기타 ===
docs
*.md
!README.md
LICENSE

# === 기타 ===
.DS_Store
*.log
tmp
```

## 공통 패턴

### 반드시 제외

```dockerignore
# 민감 정보
.env
.env.local
.env.production
*.pem
*.key
secrets/

# 불필요한 큰 파일
node_modules
.venv
__pycache__

# 버전 관리
.git
.svn
```

### 선택적 제외

```dockerignore
# 테스트 (프로덕션 빌드 시)
tests
__tests__
*.test.js
pytest.ini

# 문서 (필요 없으면)
docs
*.md
```

### 예외 (제외하지 않음)

```dockerignore
# README는 유지
*.md
!README.md

# 예시 환경 파일은 유지
.env*
!.env.example
```

## 점검 체크리스트

```bash
#!/bin/bash
# check-dockerignore.sh

echo "=== .dockerignore 점검 ==="

# .dockerignore 존재 확인
if [ ! -f .dockerignore ]; then
  echo "❌ .dockerignore 파일이 없습니다"
  exit 1
fi

# 필수 항목 확인
required_patterns=(
  ".git"
  ".env"
  "node_modules"
  "__pycache__"
  ".venv"
  "Dockerfile"
)

for pattern in "${required_patterns[@]}"; do
  if grep -q "^${pattern}" .dockerignore; then
    echo "✅ ${pattern}"
  else
    echo "⚠️  ${pattern} 누락"
  fi
done

# 빌드 컨텍스트 크기
echo ""
echo "=== 빌드 컨텍스트 크기 ==="
du -sh --exclude=.git . 2>/dev/null || du -sh .

# 큰 파일/폴더 확인
echo ""
echo "=== 큰 항목 (상위 10개) ==="
du -sh * 2>/dev/null | sort -rh | head -10
```

## 테스트 방법

```bash
# 빌드 컨텍스트 확인 (실제 빌드 없이)
docker build --no-cache -f /dev/null .

# tar로 컨텍스트 확인
tar -cvf - . | tar -tf - | head -50

# .dockerignore 적용 후 파일 목록
# (임시 Dockerfile 사용)
echo "FROM alpine\nCOPY . /app" | docker build -f - -t test-context .
docker run --rm test-context ls -la /app
```

## 멀티스테이지에서 주의

```dockerfile
# .dockerignore는 전체 빌드에 적용
# 특정 스테이지에서만 파일 필요 시 주의

# 예: tests 폴더가 .dockerignore에 있으면
# 테스트 스테이지에서도 사용 불가
FROM python:3.12-slim AS test
COPY tests/ tests/  # .dockerignore에 tests가 있으면 실패
```

## 성능 비교

```bash
# .dockerignore 없이
time docker build -t test1 .
# Sending build context: 500MB, 빌드: 60초

# .dockerignore 적용 후
time docker build -t test2 .
# Sending build context: 5MB, 빌드: 20초
```

## 자동 생성 도구

```bash
# gitignore.io 활용
curl -sL https://www.toptal.com/developers/gitignore/api/python,node,docker > .dockerignore

# 수동 조정 필요 (Docker 특화 패턴 추가)
```
