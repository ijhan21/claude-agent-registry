# DRF 테스트 전략

## 설정

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.local
```

## Fixtures

```python
@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def authenticated_client(api_client, user):
    api_client.force_authenticate(user=user)
    return api_client
```

## Factory Boy

```python
class PostFactory(DjangoModelFactory):
    class Meta:
        model = Post
    
    title = factory.Faker('sentence')
    author = factory.SubFactory(UserFactory)
```

## API 테스트

```python
@pytest.mark.django_db
class TestPostAPI:
    
    def test_list_posts(self, api_client):
        PostFactory.create_batch(3)
        response = api_client.get('/api/v1/posts/')
        assert response.status_code == 200
        assert len(response.data['results']) == 3
    
    def test_create_post_authenticated(self, authenticated_client):
        data = {'title': 'Test', 'content': 'Content'}
        response = authenticated_client.post('/api/v1/posts/', data)
        assert response.status_code == 201
```
