# mypy 타입 검사 가이드

mypy는 Python의 정적 타입 검사 도구입니다.

## 설치

```bash
pip install mypy

# 타입 스턱 설치 (필요시)
pip install types-requests types-redis types-python-dateutil
```

## 기본 설정

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"

# 엄격한 검사 모드
strict = true

# 또는 개별 옵션
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
no_implicit_optional = true
warn_redundant_casts = true
warn_unused_ignores = true
```

## 권장 설정

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.12"
strict = true

# 출력 설정
show_error_codes = true
show_error_context = true
pretty = true

# 모듈 탐색
namespace_packages = true
explicit_package_bases = true
mypy_path = "src"

# 제외 경로
exclude = [
    "migrations/",
    "tests/",
    "build/",
    "dist/",
]

# 전역 무시 (필요시)
ignore_missing_imports = false

# 플러그인
plugins = [
    "pydantic.mypy",  # Pydantic 사용 시
]

# 패키지별 설정
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = [
    "celery.*",
    "redis.*",
]
ignore_missing_imports = true
```

## Pydantic 통합

```toml
# pyproject.toml
[tool.mypy]
plugins = ["pydantic.mypy"]

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
warn_required_dynamic_aliases = true
```

## Django 통합

```bash
pip install django-stubs
```

```toml
# pyproject.toml
[tool.mypy]
plugins = ["mypy_django_plugin.main"]

[tool.django-stubs]
django_settings_module = "config.settings"
```

## FastAPI 통합

FastAPI는 Pydantic 기반이므로 Pydantic 플러그인만 필요:

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]
```

## 실행 명령어

```bash
# 전체 검사
mypy .
mypy src/

# 특정 파일
mypy src/main.py

# 엄격 모드
mypy --strict .

# 보고서 생성
mypy . --html-report mypy-report
mypy . --txt-report mypy-report

# 캐시 사용 (빠름)
mypy --incremental .

# 캐시 무시
mypy --no-incremental .
```

## 타입 힐트

### 기본 타입 힐트

```python
from typing import Optional, Union, List, Dict

def greet(name: str) -> str:
    return f"Hello, {name}"

def process(items: List[int]) -> Dict[str, int]:
    return {"sum": sum(items), "count": len(items)}

def find_user(user_id: int) -> Optional[User]:
    ...
```

### 타입 무시 방법

```python
# 줄 무시
x = some_untyped_function()  # type: ignore

# 특정 에러 무시
x = foo()  # type: ignore[attr-defined]

# 함수 전체 무시
@typing.no_type_check
def legacy_function():
    ...
```

### TYPE_CHECKING 활용

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    # 타입 검사 시에만 import (순환 참조 해결)
    from .models import User

def get_user(user_id: int) -> "User":
    ...
```

## Pre-commit 연동

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.11.0
    hooks:
      - id: mypy
        additional_dependencies:
          - pydantic
          - types-requests
        args: [--config-file=pyproject.toml]
```
