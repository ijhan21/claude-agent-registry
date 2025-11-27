# Prettier 설정 가이드

Prettier는 의견이 반영된(opinionated) 코드 포매터입니다.

## 설치

```bash
npm install -D prettier

# ESLint 통합
npm install -D eslint-config-prettier
```

## 기본 설정

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "useTabs": false,
  "trailingComma": "es5",
  "printWidth": 80,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

## 확장 설정

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "useTabs": false,
  "trailingComma": "es5",
  "printWidth": 80,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf",
  "jsxSingleQuote": false,
  "bracketSameLine": false,
  "proseWrap": "preserve",
  "htmlWhitespaceSensitivity": "css",
  "embeddedLanguageFormatting": "auto",
  "singleAttributePerLine": false,
  "overrides": [
    {
      "files": "*.md",
      "options": {
        "proseWrap": "always",
        "printWidth": 100
      }
    },
    {
      "files": "*.json",
      "options": {
        "printWidth": 200
      }
    }
  ]
}
```

## 제외 설정

```
# .prettierignore
node_modules
dist
build
.next
coverage
*.min.js
package-lock.json
pnpm-lock.yaml
yarn.lock
```

## 실행 명령어

```bash
# 포매팅 검사
npx prettier --check .

# 포매팅 적용
npx prettier --write .

# 특정 파일
npx prettier --write "src/**/*.{ts,tsx,js,jsx}"
npx prettier --write "**/*.{json,md,yml,yaml}"

# 차이점 확인
npx prettier --list-different .
```

## package.json 스크립트

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## VS Code 통합

```json
// .vscode/settings.json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[json]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[markdown]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

## ESLint와 함께 사용

```javascript
// eslint.config.js
import prettier from "eslint-config-prettier";

export default [
  // ... 다른 설정
  
  // Prettier 충돌 방지 (마지막에 배치)
  prettier,
];
```

## 실행 순서 (권장)

```bash
# 1. Prettier로 포매팅
npx prettier --write .

# 2. ESLint로 린팅
npx eslint . --fix
```

또는 도구로 통합:

```bash
# prettier-eslint 사용
npm install -D prettier-eslint-cli
npx prettier-eslint --write "src/**/*.ts"
```
