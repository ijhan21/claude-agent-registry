# Pre-commit Hooks 가이드

Pre-commit은 Git commit 전에 자동으로 코드 품질 검사를 실행하는 도구입니다.

## 설치

```bash
# Python
pip install pre-commit

# macOS
brew install pre-commit

# npm (Husky 대안)
npm install -D husky lint-staged
```

## 기본 설정

```yaml
# .pre-commit-config.yaml
repos:
  # 일반적인 검사
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: check-merge-conflict
      - id: detect-private-key
```

## Python 프로젝트

```yaml
# .pre-commit-config.yaml
repos:
  # 일반 hooks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  # Ruff (Linting + Formatting)
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format

  # mypy (타입 검사)
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.0
    hooks:
      - id: mypy
        additional_dependencies:
          - pydantic
          - types-requests
        args: [--config-file=pyproject.toml]

  # 보안 검사
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.9
    hooks:
      - id: bandit
        args: ["-c", "pyproject.toml"]
        additional_dependencies: ["bandit[toml]"]
```

## JavaScript/TypeScript 프로젝트

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-json
      - id: check-added-large-files

  # Prettier
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v4.0.0-alpha.8
    hooks:
      - id: prettier
        types_or: [javascript, jsx, ts, tsx, json, yaml, markdown]

  # ESLint
  - repo: local
    hooks:
      - id: eslint
        name: eslint
        entry: npx eslint --fix
        language: system
        types: [javascript, jsx, ts, tsx]
        pass_filenames: true

  # TypeScript
  - repo: local
    hooks:
      - id: tsc
        name: TypeScript
        entry: npx tsc --noEmit
        language: system
        types: [ts, tsx]
        pass_filenames: false
```

## Husky + lint-staged (대안)

```bash
# 설치
npm install -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

## 사용법

```bash
# hooks 설치
pre-commit install

# 수동 실행 (전체 파일)
pre-commit run --all-files

# 특정 hook 실행
pre-commit run ruff --all-files

# 업데이트
pre-commit autoupdate

# 건너뛰기 (임시)
git commit --no-verify

# 제거
pre-commit uninstall
```

## CI에서 실행

```yaml
# .github/workflows/lint.yml
name: Lint

on: [push, pull_request]

jobs:
  pre-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - uses: pre-commit/action@v3.0.1
```
