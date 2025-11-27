# DRF 필터링 & 검색

## 설정

```python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

## FilterSet 클래스

```python
class PostFilter(django_filters.FilterSet):
    created_after = django_filters.DateFilter(
        field_name='created_at', lookup_expr='gte'
    )
    title = django_filters.CharFilter(lookup_expr='icontains')
    
    class Meta:
        model = Post
        fields = ['status', 'category']
```

## ViewSet에서 사용

```python
class PostViewSet(viewsets.ModelViewSet):
    filterset_class = PostFilter
    search_fields = ['title', 'content']
    ordering_fields = ['created_at', 'view_count']
    ordering = ['-created_at']
```

## lookup_expr 종류

- exact, iexact
- contains, icontains
- gt, gte, lt, lte
- in, isnull, range
