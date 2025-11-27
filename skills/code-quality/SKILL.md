---
name: code-quality
description: 코드 품질 기준 및 도구 가이드. 다음 상황에서 활성화: (1) Linting 설정 - "ESLint", "Ruff", "Prettier", "린터 설정" 언급 시, (2) 포매팅 - "코드 포맷", "Black", "Prettier 설정" 언급 시, (3) 타입 검사 - "TypeScript", "mypy", "타입 검사" 언급 시, (4) Pre-commit - "pre-commit 설정", "Git hooks", "커밋 전 검사" 언급 시.
version: 1.0.0
---

# 코드 품질 기준 및 도구 가이드

일관된 코드 품질을 유지하기 위한 도구와 설정 방법을 다룹니다.

## 핵심 원칙

1. **자동화**: 수동 검토 최소화, 도구로 자동 검사
2. **일관성**: 팀 전체가 동일한 스타일 사용
3. **점진적 적용**: 기존 프로젝트는 단계적으로 도입
4. **CI 통합**: 모든 검사를 CI에서 자동 실행
5. **개발자 경험**: IDE 통합으로 실시간 피드백

## 언어별 도구 개요

### Python
| 목적 | 도구 | 설명 |
|------|------|------|
| Linting | Ruff | 초고속 린터 (flake8/isort/pyupgrade 대체) |
| Formatting | Ruff (format) / Black | 코드 포매팅 |
| Type Checking | mypy / Pyright | 정적 타입 검사 |
| Security | Bandit | 보안 취약점 검사 |

### JavaScript/TypeScript
| 목적 | 도구 | 설명 |
|------|------|------|
| Linting | ESLint | 코드 품질 검사 |
| Formatting | Prettier | 코드 포매팅 |
| Type Checking | TypeScript | 정적 타입 검사 |
| Import Sorting | eslint-plugin-import | Import 정렬 |

## 참조 문서

상세한 설정은 `references/` 디렉토리를 참조하세요:

| 문서 | 내용 |
|------|------|
| [python-ruff.md](references/python-ruff.md) | Ruff 설정 (Linting + Formatting) |
| [python-mypy.md](references/python-mypy.md) | mypy 타입 검사 |
| [javascript-eslint.md](references/javascript-eslint.md) | ESLint 설정 |
| [javascript-prettier.md](references/javascript-prettier.md) | Prettier 설정 |
| [typescript-config.md](references/typescript-config.md) | TypeScript 설정 |
| [pre-commit.md](references/pre-commit.md) | Pre-commit hooks 설정 |
| [editor-integration.md](references/editor-integration.md) | VS Code 통합 |
| [ci-integration.md](references/ci-integration.md) | CI/CD 통합 |

## 빠른 시작: Python

### 1. 설치

```bash
pip install ruff mypy pre-commit
```

### 2. pyproject.toml 설정

```toml
[tool.ruff]
line-length = 88
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]

[tool.ruff.format]
quote-style = "double"

[tool.mypy]
python_version = "3.12"
strict = true
```

### 3. 실행

```bash
ruff check .          # 린팅
ruff format .         # 포매팅
mypy .                # 타입 검사
```

## 빠른 시작: JavaScript/TypeScript

### 1. 설치

```bash
npm install -D eslint prettier typescript @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### 2. 설정 파일

```javascript
// eslint.config.js
import js from "@eslint/js";
import typescript from "@typescript-eslint/eslint-plugin";
import parser from "@typescript-eslint/parser";

export default [
  js.configs.recommended,
  {
    files: ["**/*.ts", "**/*.tsx"],
    languageOptions: { parser },
    plugins: { "@typescript-eslint": typescript },
    rules: {
      ...typescript.configs.recommended.rules,
    },
  },
];
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "es5"
}
```

### 3. 실행

```bash
npx eslint .          # 린팅
npx prettier --write . # 포매팅
npx tsc --noEmit      # 타입 검사
```

## Pre-commit 설정

```yaml
# .pre-commit-config.yaml
repos:
  # Python
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.0
    hooks:
      - id: mypy

  # JavaScript/TypeScript
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v9.0.0
    hooks:
      - id: eslint
        additional_dependencies:
          - eslint
          - "@typescript-eslint/parser"
          - "@typescript-eslint/eslint-plugin"

  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0
    hooks:
      - id: prettier
```

```bash
# 설치
pre-commit install

# 전체 실행
pre-commit run --all-files
```

## 품질 체크리스트

### 설정 품질
- [ ] Linter가 설정되었는가
- [ ] Formatter가 설정되었는가
- [ ] Type checker가 설정되었는가
- [ ] Pre-commit hooks가 설정되었는가
- [ ] CI에서 검사가 실행되는가

### 운영
- [ ] 모든 팀원이 동일한 설정을 사용하는가
- [ ] IDE 통합이 되어 있는가
- [ ] 문서화되어 있는가
