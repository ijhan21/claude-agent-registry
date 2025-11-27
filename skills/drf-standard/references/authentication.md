# DRF JWT 인증 (SimpleJWT)

## 설정

```python
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
    'TOKEN_OBTAIN_SERIALIZER': 'apps.users.serializers.CustomTokenObtainPairSerializer',
}
```

## Custom User Model

```python
class User(AbstractUser):
    username = None
    email = models.EmailField(unique=True)
    nickname = models.CharField(max_length=50)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []
```

## 주요 Serializers

- `CustomTokenObtainPairSerializer`: 토큰에 커스텀 클레임 추가
- `UserRegisterSerializer`: 회원가입 처리
- `PasswordChangeSerializer`: 비밀번호 변경

## API 엔드포인트

- POST `/auth/login/` - 로그인
- POST `/auth/register/` - 회원가입
- POST `/auth/logout/` - 로그아웃
- POST `/auth/token/refresh/` - 토큰 갱신
- GET `/auth/me/` - 내 정보
