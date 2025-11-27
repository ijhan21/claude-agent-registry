# DRF 예외 처리

## 일관된 에러 응답 형식

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값이 올바르지 않습니다",
    "details": [
      {"field": "email", "message": "이미 사용 중"}
    ]
  }
}
```

## 커스텀 예외 처리기

```python
def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        error_code = get_error_code(exc)
        response.data = {
            'error': {
                'code': error_code,
                'message': get_error_message(exc),
                'details': get_error_details(response.data)
            }
        }
    
    return response
```

## 커스텀 예외 클래스

```python
class ConflictException(APIException):
    status_code = 409
    default_code = 'CONFLICT'
```
