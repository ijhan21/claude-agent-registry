# DRF Serializer 패턴

## 용도별 분리

```python
class PostListSerializer(serializers.ModelSerializer):
    """목록 조회용 - 최소 필드"""
    class Meta:
        model = Post
        fields = ['id', 'title', 'created_at']

class PostDetailSerializer(serializers.ModelSerializer):
    """상세 조회용 - 전체 필드 + 관계"""
    author = UserSerializer(read_only=True)
    class Meta:
        model = Post
        fields = '__all__'

class PostCreateSerializer(serializers.ModelSerializer):
    """생성용 - 입력 검증"""
    class Meta:
        model = Post
        fields = ['title', 'content', 'category']
```

## ViewSet에서 선택

```python
def get_serializer_class(self):
    if self.action == 'list':
        return PostListSerializer
    elif self.action == 'retrieve':
        return PostDetailSerializer
    elif self.action == 'create':
        return PostCreateSerializer
    return PostDetailSerializer
```

## 유효성 검사

```python
def validate_title(self, value):
    if len(value) < 5:
        raise serializers.ValidationError('제목은 5자 이상')
    return value

def validate(self, attrs):
    # 객체 레벨 검증
    return attrs
```
