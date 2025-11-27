# DRF 보안 가이드

## HTTPS 설정

```python
# production.py
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

## CORS 설정

```python
CORS_ALLOW_ALL_ORIGINS = False  # 운영
CORS_ALLOWED_ORIGINS = [
    'https://example.com',
]
```

## Rate Limiting

```python
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
        'login': '5/minute',
    },
}
```

## 권한 클래스

```python
class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.author == request.user
```

## 보안 체크리스트

- [ ] HTTPS 강제
- [ ] JWT 토큰 수명 최소화
- [ ] CORS 화이트리스트
- [ ] Rate Limiting
- [ ] 환경 변수로 비밀 정보 관리
