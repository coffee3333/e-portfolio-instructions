# Create the Authentication App

---

## 1. Install simplejwt

```bash
pip install djangorestframework-simplejwt==5.5.*
pip freeze > requirements.txt
```

---

## 2. Configure `settings.py`

Add this to `myproject/settings.py`:

```python
# myproject/settings.py

from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
}
```

---

## 3. Create the Register Serializer

Create `core/serializers.py`:

```python
# core/serializers.py

from django.contrib.auth.models import User
from rest_framework import serializers


class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)

    class Meta:
        model = User
        fields = ('username', 'password', 'email')

    def create(self, validated_data):
        return User.objects.create_user(
            username=validated_data['username'],
            password=validated_data['password'],
            email=validated_data.get('email', ''),
        )
```

**Key points:**
- `write_only=True` — password is accepted on input but never returned in the response
- `create_user()` hashes the password. Never use `User.objects.create()` directly

---

## 4. Create the Register View

Open `core/views.py`:

```python
# core/views.py

from rest_framework import generics
from rest_framework.permissions import AllowAny

from .serializers import RegisterSerializer


class RegisterView(generics.CreateAPIView):
    serializer_class = RegisterSerializer
    permission_classes = [AllowAny]
```

`CreateAPIView` is a pre-built DRF generic view. It handles everything automatically:

| What it does | How |
|---|---|
| Accepts `POST` only | Built-in |
| Validates the request data | `is_valid(raise_exception=True)` — returns `400` automatically on invalid data |
| Saves the object | Calls `serializer.save()` → `RegisterSerializer.create()` |
| Returns `201 Created` | Built-in |

> **Note:** You can customise the behaviour by overriding hooks on the class:
>
> ```python
> class RegisterView(generics.CreateAPIView):
>     serializer_class = RegisterSerializer
>     permission_classes = [AllowAny]
>
>     # Run extra logic after the user is saved
>     def perform_create(self, serializer):
>         user = serializer.save()
>         send_welcome_email(user)  # example side effect
>
>     # Change the response format
>     def create(self, request, *args, **kwargs):
>         response = super().create(request, *args, **kwargs)
>         response.data['message'] = 'Account created successfully.'
>         return response
> ```

---

## 5. Wire Up the URLs

Create `core/urls.py`:

```python
# core/urls.py

from django.urls import path
from .views import RegisterView

urlpatterns = [
    path('register/', RegisterView.as_view(), name='register'),
]
```

Open `myproject/urls.py` and replace its contents:

```python
# myproject/urls.py

from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path('admin/', admin.site.urls),

    # Registration (your view)
    path('api/auth/register/', include('core.urls')),

    # Login + refresh (simplejwt — no code needed)
    path('api/auth/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/auth/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/auth/token/verify/', TokenVerifyView.as_view(), name='token_verify'),

    # Swagger / OpenAPI
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/schema/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

---

