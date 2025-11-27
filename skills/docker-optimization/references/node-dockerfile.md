# Node.js Dockerfile 패턴

## Next.js - Dockerfile.dev

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:20-slim

WORKDIR /app

# 의존성 설치
COPY package*.json ./
RUN npm ci

# 소스 복사 (볼륨 마운트로 대체됨)
COPY . .

EXPOSE 3000

# 개발 서버 (핫리로드)
CMD ["npm", "run", "dev"]
```

## Next.js - Dockerfile.prod

```dockerfile
# syntax=docker/dockerfile:1.4

# === Dependencies Stage ===
FROM node:20-slim AS deps

WORKDIR /app

COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# === Builder Stage ===
FROM node:20-slim AS builder

WORKDIR /app

COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .

# 빌드 타임 환경 변수
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_APP_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}

# Next.js 빌드
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# === Runner Stage ===
FROM node:20-slim AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# non-root 사용자
RUN groupadd -r nodejs && useradd -r -g nodejs nextjs

# standalone 출력만 복사 (최소 크기)
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

## next.config.js 설정 (필수)

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // standalone 모드 (Docker 필수)
  output: 'standalone',
  
  // 이미지 최적화
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.example.com',
      },
    ],
  },
  
  // 환경 변수 (빌드 타임)
  env: {
    NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
  },
}

module.exports = nextConfig
```

## 헬스체크 API 엔드포인트

```typescript
// app/api/health/route.ts
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({ 
    status: 'ok',
    timestamp: new Date().toISOString()
  })
}
```

## 빌드 타임 vs 런타임 환경 변수

```dockerfile
# 빌드 타임 (NEXT_PUBLIC_*)
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_APP_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}

# 런타임 (서버 사이드)
# docker run -e DATABASE_URL=... 로 전달
```

```bash
# 빌드 시 환경 변수 전달
docker build \
  --build-arg NEXT_PUBLIC_API_URL=https://api.example.com \
  --build-arg NEXT_PUBLIC_APP_URL=https://example.com \
  -f Dockerfile.prod \
  -t myapp:latest .
```

## 정적 파일 처리

```dockerfile
# public 폴더 복사
COPY --from=builder /app/public ./public

# static 폴더 복사 (.next/static)
COPY --from=builder /app/.next/static ./.next/static

# standalone에 포함되지 않은 파일 수동 복사 필요 시
# COPY --from=builder /app/next.config.js ./
```

## Sharp 이미지 최적화

```dockerfile
# Next.js 이미지 최적화에 sharp 필요
FROM node:20-slim AS runner

# sharp 설치 (선택)
RUN npm install --cpu=x64 --os=linux sharp
```

## 메모리 제한

```dockerfile
# Node.js 메모리 제한 설정
ENV NODE_OPTIONS="--max-old-space-size=512"
```

```yaml
# docker-compose.yml
services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.prod
    deploy:
      resources:
        limits:
          memory: 1G
```

## 캐시 디렉토리

```dockerfile
# .next/cache 볼륨 마운트 (빌드 캐시)
# docker-compose.yml에서 설정
volumes:
  - nextjs_cache:/app/.next/cache
```