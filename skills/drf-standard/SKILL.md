---
name: drf-standard
description: Django REST Framework API 프로젝트 표준 가이드. DRF 프로젝트 생성, API 설계, JWT 인증 구현 시 활성화. SimpleJWT 인증, Serializer 패턴, ViewSet 구성, 필터링/페이지네이션, drf-spectacular 문서화, 테스트 전략 포함
---

# DRF Standard

Django REST Framework API 프로젝트 표준 가이드입니다.

## 핵심 스택

| 구분 | 기술 | 버전 |
|------|------|------|
| Framework | Django | 5.x |
| API | Django REST Framework | 3.15+ |
| Auth | SimpleJWT | 5.3+ |
| Filter | django-filter | 24.x |
| Docs | drf-spectacular | 0.27+ |
| DB | PostgreSQL | 16+ |
| Test | pytest-django | 4.x |

## 빠른 시작

### 프로젝트 초기화

```bash
# 프로젝트 생성
mkdir my-api && cd my-api
python -m venv .venv
source .venv/bin/activate

# 의존성 설치
pip install django djangorestframework
pip install djangorestframework-simplejwt
pip install django-filter drf-spectacular
pip install psycopg[binary] django-cors-headers
pip install pytest pytest-django pytest-cov
```

### 프로젝트 구조

```
my-api/
├── config/                 # 프로젝트 설정
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py        # 공통 설정
│   │   ├── local.py       # 개발 환경
│   │   └── production.py  # 운영 환경
│   ├── urls.py
│   └── wsgi.py
├── apps/                   # 애플리케이션
│   ├── __init__.py
│   ├── users/
│   └── core/              # 공통 모듈
├── tests/
│   ├── conftest.py
│   └── factories.py
├── manage.py
├── pyproject.toml
├── requirements/
└── pytest.ini
```

## 핵심 원칙

### 1. Settings 분리

환경별 설정 파일 분리: `base.py`, `local.py`, `production.py`

상세 설정: [`references/settings.md`](references/settings.md) 참조

### 2. JWT 인증

SimpleJWT 기반 인증 구현. Access Token 30분~1시간, Refresh Token 7일~30일.

상세 구현: [`references/authentication.md`](references/authentication.md) 참조

### 3. Serializer 패턴

용도별 Serializer 분리: List, Detail, Create, Update

상세 패턴: [`references/serializers.md`](references/serializers.md) 참조

### 4. ViewSet 구성

CRUD 전체 필요 → ModelViewSet, 일부만 → GenericViewSet + Mixins

상세 가이드: [`references/viewsets.md`](references/viewsets.md) 참조

### 5. 필터링 & 검색

django-filter 활용: FilterSet, SearchFilter, OrderingFilter

상세 구현: [`references/filtering.md`](references/filtering.md) 참조

### 6. 페이지네이션

대용량 데이터는 CursorPagination 권장

상세 설정: [`references/pagination.md`](references/pagination.md) 참조

### 7. 예외 처리

일관된 에러 응답 형식 구현

상세 구현: [`references/exceptions.md`](references/exceptions.md) 참조

### 8. API 문서화

drf-spectacular로 OpenAPI 3.0 스키마 자동 생성

상세 설정: [`references/documentation.md`](references/documentation.md) 참조

### 9. 테스트

pytest-django + Factory Boy 기반 테스트. 커버리지 80% 이상 목표

상세 전략: [`references/testing.md`](references/testing.md) 참조

### 10. 보안

HTTPS 강제, CORS 설정, Rate Limiting, SQL Injection 방지

상세 가이드: [`references/security.md`](references/security.md) 참조

### 11. 성능 최적화

N+1 쿼리 방지: select_related(), prefetch_related()

상세 기법: [`references/performance.md`](references/performance.md) 참조

## 필수 설정 요약

### REST_FRAMEWORK

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
    'EXCEPTION_HANDLER': 'apps.core.exceptions.custom_exception_handler',
}
```

## 리소스

| 주제 | 참조 |
|------|------|
| 프로젝트 구조 | [`references/project-structure.md`](references/project-structure.md) |
| Settings 분리 | [`references/settings.md`](references/settings.md) |
| JWT 인증 | [`references/authentication.md`](references/authentication.md) |
| Serializer 패턴 | [`references/serializers.md`](references/serializers.md) |
| ViewSet 구성 | [`references/viewsets.md`](references/viewsets.md) |
| 필터링 & 검색 | [`references/filtering.md`](references/filtering.md) |
| 페이지네이션 | [`references/pagination.md`](references/pagination.md) |
| 예외 처리 | [`references/exceptions.md`](references/exceptions.md) |
| 테스트 전략 | [`references/testing.md`](references/testing.md) |
| API 문서화 | [`references/documentation.md`](references/documentation.md) |
| 보안 | [`references/security.md`](references/security.md) |
| 성능 최적화 | [`references/performance.md`](references/performance.md) |

## 품질 체크리스트

- [ ] Settings 환경별 분리 완료
- [ ] JWT 인증 구현 및 테스트
- [ ] Serializer 용도별 분리
- [ ] ViewSet 권한 설정
- [ ] 필터링/페이지네이션 적용
- [ ] 예외 처리 커스터마이징
- [ ] API 문서 자동 생성
- [ ] 테스트 커버리지 80% 이상
- [ ] 보안 설정 검토
- [ ] N+1 쿼리 점검
