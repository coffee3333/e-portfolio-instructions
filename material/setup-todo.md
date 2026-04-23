## 1. Create the Todos App

```bash
python manage.py startapp todos
```

Add it to `INSTALLED_APPS` in `myproject/settings.py`:

```python
# myproject/settings.py

INSTALLED_APPS = [
    ...
    'todos',               # ← add this
]
```

---

## 2. Define the Model

Open `todos/models.py` and replace its contents:

```python
# todos/models.py

from django.contrib.auth.models import User
from django.db import models


class Todo(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='todos')
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    completed = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['-created_at']

    def __str__(self):
        return f"{self.user.username}: {self.title}"
```

**Field explanations:**

| Field | Type | Notes |
|---|---|---|
| `user` | ForeignKey | Links each todo to its owner. `CASCADE` deletes todos when the user is deleted. |
| `title` | CharField | Required. Max 255 characters. |
| `description` | TextField | Optional (`blank=True`). |
| `completed` | BooleanField | Defaults to `False`. |
| `created_at` | DateTimeField | Set automatically on creation (`auto_now_add`). Never editable. |
| `updated_at` | DateTimeField | Updated automatically on every save (`auto_now`). Never editable. |

`ordering = ['-created_at']` makes all queries return newest todos first by default.

Create and apply the migration:

```bash
python manage.py makemigrations todos
python manage.py migrate
```

> **Note:** `makemigrations` generates a migration file based on your model. `migrate` applies it to the database. You need both commands whenever you change a model.

---

## 3. Create the Serializer

Create `todos/serializers.py`:

```python
# todos/serializers.py

from rest_framework import serializers

from .models import Todo


class TodoSerializer(serializers.ModelSerializer):
    class Meta:
        model = Todo
        fields = ['id', 'title', 'description', 'completed', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

**Why `user` is not in `fields`:**

The `user` field is intentionally excluded. It will be set automatically in the view from `request.user`. If you included `user` in `fields`, a client could send any user id in the request body and create todos owned by other users — a serious security hole.

**`read_only_fields`:** `id`, `created_at`, and `updated_at` are managed by Django and should never be sent by the client.

---

## 4. Write the ViewSet

Open `todos/views.py` and replace its contents:

```python
# todos/views.py

from rest_framework import permissions, viewsets

from .models import Todo
from .serializers import TodoSerializer


class TodoViewSet(viewsets.ModelViewSet):
    serializer_class = TodoSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Todo.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

**What `ModelViewSet` gives you for free:**

| Action | HTTP method | URL |
|---|---|---|
| `list` | GET | `/api/todos/` |
| `create` | POST | `/api/todos/` |
| `retrieve` | GET | `/api/todos/{id}/` |
| `update` | PUT | `/api/todos/{id}/` |
| `partial_update` | PATCH | `/api/todos/{id}/` |
| `destroy` | DELETE | `/api/todos/{id}/` |

All six operations are implemented with just those two method overrides.

**`get_queryset`:** Filters todos to only those owned by the current user. This is the security boundary — without this override, `GET /api/todos/` would return every todo in the database, regardless of owner.

**`perform_create`:** Called when a new todo is saved. Injects `user=self.request.user` so the todo is always associated with the authenticated user.

---

## 5. Set Up URL Routing

Create `todos/urls.py`:

```python
# todos/urls.py

from django.urls import include, path
from rest_framework.routers import DefaultRouter

from .views import TodoViewSet

router = DefaultRouter()
router.register(r'todos', TodoViewSet, basename='todo')

urlpatterns = [
    path('', include(router.urls)),
]
```

**What `DefaultRouter` generates automatically:**

```
GET     /api/todos/         → TodoViewSet.list
POST    /api/todos/         → TodoViewSet.create
GET     /api/todos/{id}/    → TodoViewSet.retrieve
PUT     /api/todos/{id}/    → TodoViewSet.update
PATCH   /api/todos/{id}/    → TodoViewSet.partial_update
DELETE  /api/todos/{id}/    → TodoViewSet.destroy
```

> **Warning: `basename` is required here.**  
> Because `get_queryset` is overridden (it doesn't just return `Todo.objects.all()`), DRF cannot automatically determine the model name for URL pattern naming. `basename='todo'` is required or you get an `AssertionError` at startup.

Wire the todos URLs into the project. Open `myproject/urls.py`:

```python
# myproject/urls.py

from django.contrib import admin
from django.urls import include, path
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api-auth/', include('rest_framework.urls')),
    path('api/auth/', include('core.urls')),
    path('api/auth/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/', include('todos.urls')),    # ← add this
]
```

> **Note:** If you are using Token Auth (Part A of guide 03) instead of JWT, remove the two `simplejwt` lines and use `Authorization: Token <key>` in all the curl examples below.

---

## 6. Register the Model with Admin

Open `todos/admin.py`:

```python
# todos/admin.py

from django.contrib import admin

from .models import Todo


@admin.register(Todo)
class TodoAdmin(admin.ModelAdmin):
    list_display = ('title', 'user', 'completed', 'created_at')
    list_filter = ('completed', 'user')
    search_fields = ('title', 'description')
```

Now you can view, create, edit, and delete todos at [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

---