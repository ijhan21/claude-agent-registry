# DRF API 문서화 (drf-spectacular)

## 설정

```python
SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'VERSION': '1.0.0',
    'SECURITY': [{'Bearer': []}],
}
```

## URL 설정

```python
path('api/schema/', SpectacularAPIView.as_view()),
path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema')),
path('api/redoc/', SpectacularRedocView.as_view(url_name='schema')),
```

## @extend_schema

```python
@extend_schema(
    summary='게시물 목록',
    tags=['posts'],
    parameters=[
        OpenApiParameter('status', str, description='상태 필터'),
    ],
    responses={200: PostListSerializer(many=True)}
)
def list(self, request):
    pass
```

## 태그 자동 적용

```python
@extend_schema_view(
    list=extend_schema(summary='목록', tags=['posts']),
    create=extend_schema(summary='생성', tags=['posts']),
)
class PostViewSet(viewsets.ModelViewSet):
    pass
```
