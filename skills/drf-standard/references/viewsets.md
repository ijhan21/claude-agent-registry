# DRF ViewSet 구성

## ViewSet 유형 선택

| 유형 | 용도 |
|------|------|
| ModelViewSet | 전체 CRUD |
| ReadOnlyModelViewSet | 읽기 전용 |
| GenericViewSet + Mixins | 일부 액션만 |

## 커스터마이징

```python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.all()
    
    def get_serializer_class(self):
        # 액션별 Serializer
        pass
    
    def get_queryset(self):
        # 쿼리셋 최적화
        return Post.objects.select_related('author')
    
    def get_permissions(self):
        # 액션별 권한
        pass
    
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

## Custom Action

```python
@action(detail=True, methods=['post'])
def like(self, request, pk=None):
    post = self.get_object()
    post.likes.add(request.user)
    return Response({'status': 'liked'})
```
