---
name: react-standard
description: Next.js 14 App Router 기반 React 프로젝트 표준화 가이드. 다음 상황에서 활성화: (1) React/Next.js 프로젝트 생성, (2) 컴포넌트 작성, (3) 상태관리 구현, (4) API 연동. Zustand, TanStack Query, Tailwind, shadcn/ui, React Hook Form 패턴 포함.
---

# React Standard (Next.js 14)

## 개요

Next.js 14 App Router 기반 React 프로젝트의 표준화된 구조, 패턴, 설정을 정의합니다.

## 핵심 스택

| 영역 | 기술 | 이유 |
|------|------|------|
| 프레임워크 | **Next.js 14** | App Router, RSC, SEO |
| 언어 | **TypeScript** | 타입 안전성 |
| 상태 (전역) | **Zustand** | 경량, 간단한 API |
| 상태 (서버) | **TanStack Query** | 캐싱, 동기화, 무한스크롤 |
| 스타일 | **Tailwind CSS 3** | 유틸리티 기반, 빠른 개발 |
| UI 컴포넌트 | **shadcn/ui** | Radix 기반, 접근성, 커스터마이징 |
| 폼 | **React Hook Form + Zod** | 성능, 타입 안전 검증 |
| API | **Axios** | 인터셉터, 취소 토큰 |
| 애니메이션 | **Framer Motion** | 선언적, 강력한 API |
| 아이콘 | **Lucide React** | 경량, 트리쉐이킹 |
| SEO | **next-seo** | 메타 관리 간소화 |

## 프로젝트 구조

```
src/
├── app/                        # Next.js App Router
│   ├── (auth)/                 # 인증 그룹 (레이아웃 공유)
│   │   ├── login/
│   │   │   └── page.tsx
│   │   └── register/
│   │       └── page.tsx
│   ├── (main)/                 # 메인 그룹
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   └── dashboard/
│   │       └── page.tsx
│   ├── api/                    # API Routes (BFF)
│   │   └── auth/
│   │       └── route.ts
│   ├── layout.tsx              # 루트 레이아웃
│   ├── loading.tsx             # 전역 로딩
│   ├── error.tsx               # 전역 에러
│   ├── not-found.tsx           # 404
│   └── globals.css
│
├── components/
│   ├── ui/                     # shadcn/ui 컴포넌트
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   └── ...
│   ├── layout/                 # 레이아웃 컴포넌트
│   │   ├── header.tsx
│   │   ├── sidebar.tsx
│   │   └── footer.tsx
│   └── common/                 # 공통 컴포넌트
│       ├── error-boundary.tsx
│       └── loading-spinner.tsx
│
├── features/                   # 기능별 모듈
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── stores/
│   │   └── types.ts
│   ├── users/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api.ts
│   │   └── types.ts
│   └── files/
│       ├── components/
│       │   └── file-upload.tsx
│       └── hooks/
│
├── hooks/                      # 공통 훅
│   ├── use-debounce.ts
│   └── use-media-query.ts
│
├── lib/                        # 유틸리티
│   ├── axios.ts                # Axios 인스턴스
│   ├── utils.ts                # cn() 등 헬퍼
│   └── validations.ts          # Zod 스키마
│
├── stores/                     # 전역 스토어
│   ├── auth-store.ts
│   └── theme-store.ts
│
├── types/                      # 전역 타입
│   ├── api.ts
│   └── index.ts
│
└── providers/                  # Context Providers
    ├── query-provider.tsx
    └── theme-provider.tsx
```

상세 구조: [`references/project-structure.md`](references/project-structure.md) 참조

## 환경 설정

### .env.local

```env
# API
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
NEXT_PUBLIC_WS_URL=ws://localhost:8000/api/v1/ws

# App
NEXT_PUBLIC_APP_NAME=MyApp
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.example.com',
      },
    ],
  },
  // Docker 배포용 standalone
  output: 'standalone',
}

module.exports = nextConfig
```

## 상태 관리

### Zustand vs TanStack Query 사용 기준

| 데이터 유형 | 도구 | 예시 |
|------------|------|------|
| **서버 상태** | TanStack Query | API 데이터, 사용자 목록 |
| **클라이언트 상태** | Zustand | 인증 토큰, UI 상태, 테마 |

상세 패턴: [`references/state-patterns.md`](references/state-patterns.md) 참조

### Zustand 기본 패턴

```typescript
// stores/auth-store.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AuthState {
  accessToken: string | null
  user: User | null
  setAuth: (token: string, user: User) => void
  logout: () => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      accessToken: null,
      user: null,
      setAuth: (token, user) => set({ accessToken: token, user }),
      logout: () => set({ accessToken: null, user: null }),
    }),
    { name: 'auth-storage' }
  )
)
```

### TanStack Query 기본 패턴

```typescript
// features/users/hooks/use-users.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { getUsers, createUser } from '../api'

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: getUsers,
  })
}

export function useCreateUser() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] })
    },
  })
}
```

## API 클라이언트 (Axios)

상세 패턴: [`references/api-patterns.md`](references/api-patterns.md) 참조

```typescript
// lib/axios.ts
import axios from 'axios'
import { useAuthStore } from '@/stores/auth-store'

export const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: { 'Content-Type': 'application/json' },
})

// 요청 인터셉터
api.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 응답 인터셉터 (토큰 갱신)
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // 토큰 갱신 로직
    }
    return Promise.reject(error)
  }
)
```

## 폼 (React Hook Form + Zod)

상세 패턴: [`references/form-patterns.md`](references/form-patterns.md) 참조

```typescript
// lib/validations.ts
import { z } from 'zod'

export const loginSchema = z.object({
  email: z.string().email('유효한 이메일을 입력하세요'),
  password: z.string().min(8, '비밀번호는 8자 이상이어야 합니다'),
})

export type LoginInput = z.infer<typeof loginSchema>
```

```typescript
// features/auth/components/login-form.tsx
'use client'

import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { loginSchema, LoginInput } from '@/lib/validations'

export function LoginForm() {
  const form = useForm<LoginInput>({
    resolver: zodResolver(loginSchema),
    defaultValues: { email: '', password: '' },
  })

  const onSubmit = (data: LoginInput) => {
    // 로그인 처리
  }

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* 폼 필드 */}
    </form>
  )
}
```

## 컴포넌트 패턴

### 클라이언트 vs 서버 컴포넌트

```typescript
// 서버 컴포넌트 (기본값) - 데이터 페칭
// app/users/page.tsx
import { getUsers } from '@/features/users/api'

export default async function UsersPage() {
  const users = await getUsers()
  return <UserList users={users} />
}

// 클라이언트 컴포넌트 - 상호작용 필요 시
// features/users/components/user-list.tsx
'use client'

import { useState } from 'react'

export function UserList({ users }: Props) {
  const [selected, setSelected] = useState<string | null>(null)
  // ...
}
```

## 추가 패턴

상세 구현 패턴:
- **프로젝트 구조**: [`references/project-structure.md`](references/project-structure.md)
- **상태 관리**: [`references/state-patterns.md`](references/state-patterns.md)
- **API 연동**: [`references/api-patterns.md`](references/api-patterns.md)
- **폼/검증**: [`references/form-patterns.md`](references/form-patterns.md)
- **인증 UI**: [`references/auth-patterns.md`](references/auth-patterns.md)
- **에러 처리**: [`references/error-patterns.md`](references/error-patterns.md)
- **파일 업로드**: [`references/file-upload-patterns.md`](references/file-upload-patterns.md)
- **WebSocket**: [`references/websocket-patterns.md`](references/websocket-patterns.md)
- **다크모드**: [`references/theme-patterns.md`](references/theme-patterns.md)
- **SEO**: [`references/seo-patterns.md`](references/seo-patterns.md)
- **반응형**: [`references/responsive-patterns.md`](references/responsive-patterns.md)
- **Docker 배포**: [`references/docker-patterns.md`](references/docker-patterns.md)
- **코드 품질**: [`references/code-quality.md`](references/code-quality.md)

## 품질 체크리스트

프로젝트 생성 시 확인:

- [ ] TypeScript strict 모드
- [ ] ESLint + Prettier 설정
- [ ] 환경 변수 (.env.local, .env.example)
- [ ] Tailwind CSS 설정
- [ ] shadcn/ui 초기화
- [ ] Axios 인스턴스 설정
- [ ] TanStack Query Provider 설정
- [ ] 에러 바운더리 구현
- [ ] 로딩 상태 처리
- [ ] SEO 메타 태그
- [ ] 다크모드 지원
- [ ] 반응형 디자인
- [ ] Docker 배포 설정