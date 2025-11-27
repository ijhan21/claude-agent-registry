# CI/CD 통합 가이드

GitHub Actions에서 코드 품질 검사를 자동화하는 방법입니다.

## Python 프로젝트

```yaml
# .github/workflows/lint.yml
name: Lint

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install ruff mypy
          pip install -r requirements.txt
      
      - name: Run Ruff (lint)
        run: ruff check . --output-format=github
      
      - name: Run Ruff (format)
        run: ruff format --check .
      
      - name: Run mypy
        run: mypy . --ignore-missing-imports
```

## JavaScript/TypeScript 프로젝트

```yaml
# .github/workflows/lint.yml
name: Lint

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npx eslint . --max-warnings 0
      
      - name: Run Prettier
        run: npx prettier --check .
      
      - name: Run TypeScript
        run: npx tsc --noEmit
```

## 통합 Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  # Linting
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install ruff mypy -r requirements.txt
      
      - name: Ruff lint
        run: ruff check . --output-format=github
      
      - name: Ruff format
        run: ruff format --check .
      
      - name: mypy
        run: mypy .

  # Testing
  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-dev.txt
      
      - name: Run tests
        run: pytest --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage.xml

  # Security
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Bandit
        uses: PyCQA/bandit-action@v1
        with:
          configfile: pyproject.toml
```

## Pre-commit 사용

```yaml
# .github/workflows/pre-commit.yml
name: pre-commit

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

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

## PR 주석 추가

```yaml
# reviewdog을 사용한 PR 주석
- name: Run ESLint with reviewdog
  uses: reviewdog/action-eslint@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    reporter: github-pr-review
    eslint_flags: 'src/'
```
