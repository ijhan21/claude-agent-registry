# DRF 프로젝트 구조

## 권장 구조

```
my-api/
├── config/                     # 프로젝트 설정
│   ├── settings/
│   │   ├── __init__.py        # 환경 변수로 설정 선택
│   │   ├── base.py            # 공통 설정
│   │   ├── local.py           # 개발 환경
│   │   └── production.py      # 운영 환경
│   ├── urls.py
│   └── wsgi.py
├── apps/                       # Django 앱 디렉토리
│   ├── core/                   # 공통 모듈
│   ├── users/                  # 사용자 앱
│   └── posts/                  # 예시 앱
├── tests/
├── requirements/
├── manage.py
└── pyproject.toml
```

## 환경 선택 로직

```python
# config/settings/__init__.py
import os

env = os.environ.get('DJANGO_ENV', 'local')

if env == 'production':
    from .production import *
else:
    from .local import *
```

## 앱 생성 체크리스트

1. `apps/` 디렉토리에서 `startapp` 실행
2. `apps.py`의 `name`에 `apps.` 프리픽스 추가
3. `INSTALLED_APPS`에 등록
4. `urls.py` 생성 및 라우터 설정
5. `tests/` 디렉토리 생성
