# 빌드 시크릿 마운트

## 사용 사례

- 프라이빗 패키지 저장소 인증
- Git 프라이빗 리포지토리 클론
- npm/pip 프라이빗 레지스트리
- API 키 (빌드 시 필요)

## 기본 문법

```dockerfile
# syntax=docker/dockerfile:1.4

# 시크릿 마운트로 파일 사용
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci
```

## 빌드 명령어

```bash
# 파일에서 시크릿 전달
docker build \
  --secret id=npm_token,src=./.npmrc \
  -t myapp .

# 환경 변수에서 시크릿 전달
docker build \
  --secret id=api_key,env=API_KEY \
  -t myapp .
```

## pip 프라이빗 저장소

```dockerfile
# syntax=docker/dockerfile:1.4
FROM python:3.12-slim

# pip.conf 시크릿 마운트
RUN --mount=type=secret,id=pip_conf,target=/etc/pip.conf \
    pip install -r requirements.txt
```

```bash
# 빌드
docker build \
  --secret id=pip_conf,src=./pip.conf \
  -t myapp .
```

```ini
# pip.conf 예시
[global]
index-url = https://user:password@private.pypi.org/simple/
extra-index-url = https://pypi.org/simple/
```

## npm 프라이빗 레지스트리

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-slim

WORKDIR /app
COPY package*.json ./

# .npmrc 시크릿 마운트
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci

COPY . .
```

```bash
docker build \
  --secret id=npmrc,src=./.npmrc \
  -t myapp .
```

```ini
# .npmrc 예시
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
@myorg:registry=https://npm.pkg.github.com
```

## Git 프라이빗 리포지토리

```dockerfile
# syntax=docker/dockerfile:1.4
FROM python:3.12-slim

# SSH 키 마운트
RUN --mount=type=secret,id=ssh_key,target=/root/.ssh/id_rsa \
    --mount=type=secret,id=ssh_known_hosts,target=/root/.ssh/known_hosts \
    pip install git+ssh://git@github.com/org/private-repo.git
```

```bash
docker build \
  --secret id=ssh_key,src=~/.ssh/id_rsa \
  --secret id=ssh_known_hosts,src=~/.ssh/known_hosts \
  -t myapp .
```

## 환경 변수 기반 시크릿

```dockerfile
# syntax=docker/dockerfile:1.4
FROM python:3.12-slim

# 환경 변수로 시크릿 읽기
RUN --mount=type=secret,id=api_key \
    export API_KEY=$(cat /run/secrets/api_key) && \
    ./setup-script.sh
```

```bash
# 환경 변수에서 전달
export API_KEY="my-secret-key"
docker build \
  --secret id=api_key,env=API_KEY \
  -t myapp .
```

## 시크릿 여러 개 사용

```dockerfile
RUN --mount=type=secret,id=npm_token \
    --mount=type=secret,id=github_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    npm ci
```

## GitHub Actions에서 시크릿

```yaml
- name: Build with secrets
  uses: docker/build-push-action@v5
  with:
    context: .
    secrets: |
      npm_token=${{ secrets.NPM_TOKEN }}
      pip_conf=${{ secrets.PIP_CONF }}
```

## 보안 주의사항

```dockerfile
# 나쁨: 시크릿이 레이어에 남음
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
RUN npm ci
RUN rm .npmrc  # 삭제해도 이전 레이어에 남아있음

# 좋음: 시크릿 마운트 사용 (레이어에 안 남음)
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

## 시크릿 파일 권한

```dockerfile
# 권한 설정
RUN --mount=type=secret,id=ssh_key,target=/root/.ssh/id_rsa,mode=0600 \
    git clone git@github.com:org/repo.git
```

## 디버깅

```bash
# 시크릿 없이 빌드 테스트
docker build --no-cache -t myapp .

# 시크릿 확인 (빌드 중)
RUN --mount=type=secret,id=test \
    ls -la /run/secrets/ && \
    cat /run/secrets/test
```