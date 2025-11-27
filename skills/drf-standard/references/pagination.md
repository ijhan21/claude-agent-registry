# DRF 페이지네이션

## 유형 비교

| 유형 | 장점 | 단점 | 권장 |
|------|------|------|------|
| PageNumberPagination | 직관적 | 대용량 느림 | 소규모 |
| LimitOffsetPagination | 유연함 | 대용량 느림 | 무한스크롤 |
| CursorPagination | 대용량 최적화 | 임의 이동 불가 | 대규모 |

## CursorPagination (권장)

```python
class TimelinePagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'
    cursor_query_param = 'cursor'
```

## 인덱스 필수

```python
class Post(models.Model):
    created_at = models.DateTimeField(db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['-created_at']),
        ]
```
