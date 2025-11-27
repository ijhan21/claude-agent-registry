# Ruff 설정 가이드

Ruff는 Rust로 작성된 초고속 Python linter 및 formatter입니다.
Flake8, isort, pyupgrade, Black 등을 대체할 수 있습니다.

## 설치

```bash
pip install ruff
```

## 기본 설정

```toml
# pyproject.toml
[tool.ruff]
# Python 버전
target-version = "py312"

# 줄 길이
line-length = 88

# 제외 디렉토리
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "build",
    "dist",
    ".mypy_cache",
    ".ruff_cache",
    "migrations",
]

# 전체 파일에서 무시할 규칙
ignore = []

# 수정 가능한 규칙 자동 수정 허용
fix = true
```

## Linting 설정

```toml
[tool.ruff.lint]
# 활성화할 규칙 그룹
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "SIM",    # flake8-simplify
    "TCH",    # flake8-type-checking
    "PTH",    # flake8-use-pathlib
    "RUF",    # Ruff-specific rules
]

# 무시할 규칙
ignore = [
    "E501",   # line too long (포매터가 처리)
    "B008",   # function call in default argument (FastAPI Depends)
]

# 파일별 규칙 제외
[tool.ruff.lint.per-file-ignores]
"__init__.py" = ["F401"]  # unused import
"tests/**/*.py" = [
    "S101",   # assert 사용 허용
    "PLR2004", # magic number 허용
]

# isort 설정
[tool.ruff.lint.isort]
known-first-party = ["app", "src"]
section-order = [
    "future",
    "standard-library",
    "third-party",
    "first-party",
    "local-folder",
]
combine-as-imports = true
force-wrap-aliases = true
```

## Formatting 설정

```toml
[tool.ruff.format]
# 문자열 따옴표 스타일
quote-style = "double"

# 들여쓰기 스타일
indent-style = "space"

# 후행 콤마
skip-magic-trailing-comma = false

# docstring 포매팅
docstring-code-format = true
```

## 전체 권장 설정

```toml
# pyproject.toml - 전체 권장 설정

[tool.ruff]
target-version = "py312"
line-length = 88
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "build",
    "dist",
    "migrations",
    "*.pyi",
]

[tool.ruff.lint]
select = [
    # 기본
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings  
    "F",      # Pyflakes
    "I",      # isort
    
    # 코드 품질
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "SIM",    # flake8-simplify
    "PIE",    # flake8-pie
    "RET",    # flake8-return
    
    # 타입
    "TCH",    # flake8-type-checking
    "ANN",    # flake8-annotations (optional)
    
    # 보안
    "S",      # flake8-bandit
    
    # 기타
    "PTH",    # flake8-use-pathlib
    "RUF",    # Ruff-specific
]

ignore = [
    "E501",   # line too long
    "B008",   # function call in default argument
    "ANN101", # missing type for self
    "ANN102", # missing type for cls
    "ANN401", # dynamically typed Any
    "S101",   # assert usage
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101", "ANN", "PLR2004"]
"conftest.py" = ["ANN"]
"migrations/**/*.py" = ["ALL"]

[tool.ruff.lint.isort]
known-first-party = ["app"]
force-single-line = false
lines-after-imports = 2

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
docstring-code-format = true
```

## 실행 명령어

```bash
# 린팅 검사
ruff check .

# 린팅 검사 + 자동 수정
ruff check . --fix

# 포매팅 검사
ruff format --check .

# 포매팅 적용
ruff format .

# 특정 규칙만 검사
ruff check . --select E,F

# 특정 파일
ruff check src/main.py

# 출력 포맷
ruff check . --output-format=json
ruff check . --output-format=github  # GitHub Actions
```

## VS Code 통합

```json
// .vscode/settings.json
{
    "[python]": {
        "editor.defaultFormatter": "charliermarsh.ruff",
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.fixAll.ruff": "explicit",
            "source.organizeImports.ruff": "explicit"
        }
    },
    "ruff.lint.run": "onSave"
}
```

## Pre-commit 연동

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.0
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
```
