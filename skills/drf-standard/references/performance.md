# DRF 성능 최적화

## N+1 쿼리 방지

```python
def get_queryset(self):
    return Post.objects.select_related(
        'author',      # FK
        'category',    # FK
    ).prefetch_related(
        'tags',        # M2M
        'comments',    # 역참조
    )
```

## 집계 최적화

```python
from django.db.models import Count

Post.objects.annotate(
    comment_count=Count('comments')
)
```

## 캐싱

```python
from django.core.cache import cache

cache_key = f'post_{pk}'
data = cache.get(cache_key)
if not data:
    data = serializer.data
    cache.set(cache_key, data, 60 * 10)
```

## 인덱스

```python
class Post(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=['status', '-created_at']),
        ]
```

## 체크리스트

- [ ] select_related / prefetch_related
- [ ] annotate로 집계
- [ ] CursorPagination 사용
- [ ] 적절한 인덱스
- [ ] Redis 캐싱
