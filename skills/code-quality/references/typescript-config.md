# TypeScript 설정 가이드

TypeScript 프로젝트를 위한 tsconfig.json 설정 방법입니다.

## 기본 설정

```json
// tsconfig.json
{
  "compilerOptions": {
    // 기본 설정
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    
    // 모듈 해석
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    
    // 엄격한 타입 검사
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    
    // 출력 설정
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    
    // 기타
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## React 프로젝트

```json
// tsconfig.json (React)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "moduleResolution": "bundler",
    
    // React JSX
    "jsx": "react-jsx",
    
    // 엄격한 타입 검사
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    
    // 경로 매핑
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/utils/*": ["./src/utils/*"]
    },
    
    // 기타
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

## Next.js 프로젝트

```json
// tsconfig.json (Next.js)
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "DOM.Iterable", "ES2022"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## Node.js 프로젝트

```json
// tsconfig.json (Node.js)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "sourceMap": true,
    
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## 주요 옵션 설명

### strict 모드 포함 옵션

```json
"strict": true
// 아래 모든 옵션 활성화:
// - strictNullChecks
// - strictFunctionTypes  
// - strictBindCallApply
// - strictPropertyInitialization
// - noImplicitAny
// - noImplicitThis
// - alwaysStrict
```

### 추가 엄격 옵션

```json
{
  "noUncheckedIndexedAccess": true,    // 배열/객체 접근 시 undefined 가능성
  "noImplicitReturns": true,            // 모든 코드 경로에서 return 필수
  "noFallthroughCasesInSwitch": true,   // switch fallthrough 방지
  "exactOptionalPropertyTypes": true    // optional과 undefined 구분
}
```

## 실행 명령어

```bash
# 타입 검사만 (컨파일 없이)
npx tsc --noEmit

# 컨파일
npx tsc

# watch 모드
npx tsc --watch

# 특정 설정 파일
npx tsc -p tsconfig.build.json
```
