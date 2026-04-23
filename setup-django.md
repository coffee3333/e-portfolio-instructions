# Project Setup

**Prerequisites:** Complete [Prerequisites](prerequisites.md) first.

By the end of this guide you will have a running Django + DRF development server with the browsable API accessible in your browser.

---

## 1. Create the Project Directory and Virtual Environment

Choose a location for your project and create a directory:

```bash
mkdir drf-tutorial
cd drf-tutorial
```

Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate        # macOS / Linux
# venv\Scripts\activate         # Windows
```

Upgrade pip:

```bash
pip install --upgrade pip
```

---

## 2. Install Dependencies

```bash
pip install Django==5.2.*
pip install djangorestframework==3.17.*
pip install djangorestframework-simplejwt==5.5.*  
pip install drf-spectacular==0.28.* 
pip install django-cors-headers==4.*
pip freeze > requirements.txt
```

Verify the installs:

```bash
python3 -c "import django; print(django.__version__)"
python3 -c "import rest_framework; print(rest_framework.__version__)"
python3 -c "import rest_framework_simplejwt; print(rest_framework_simplejwt.__version__)"
python3 -c "import drf_spectacular; print(drf_spectacular.__version__)"
python3 -c "import corsheaders; print(corsheaders.__version__)"
```

Expected output:

```
5.2.x
3.17.x
5.5.x
0.28.x
4.x.x
```

---

## 3. Create the Django Project

```bash
django-admin startproject myproject .
```

> **Warning: The trailing `.` is critical.**  
> Without it, Django creates a `myproject/myproject/` double-nested structure. Every file path in the rest of this guide assumes you used the trailing `.`. If you forgot it, delete the created folder and run the command again with the dot.

Your directory should now look like this:

```
drf-tutorial/
├── manage.py
├── requirements.txt
├── venv/
└── myproject/
    ├── __init__.py
    ├── asgi.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py
```

### What each file does

| File | Purpose |
|---|---|
| `manage.py` | CLI entry point for all Django commands |
| `myproject/settings.py` | All project configuration |
| `myproject/urls.py` | Root URL routing — all URLs start here |
| `myproject/wsgi.py` | Entry point for traditional WSGI servers |
| `myproject/asgi.py` | Entry point for async ASGI servers |

---

## 4. Create a `.gitignore`

If you are using Git, create this file now to avoid committing files you should not commit:

```
# .gitignore
venv/
*.pyc
__pycache__/
db.sqlite3
.env
```

---

## 5. Create the App

A Django project is made up of one or more "apps". Each app handles a specific area of functionality.

```bash
python manage.py startapp app
```

This creates a `app/` directory:

```
app/
├── __init__.py
├── admin.py
├── apps.py
├── migrations/
│   └── __init__.py
├── models.py
├── tests.py
└── views.py
```

> **Note:** The difference between a Django *project* and a Django *app*: the project is the overall container (settings, root URLs). Apps are self-contained modules within the project that each handle one area (authentication, todos, billing, etc.).

---

## 6. Configure `settings.py`

Open `myproject/settings.py` and make the following changes.

### 6a. Add apps to `INSTALLED_APPS`

Find the `INSTALLED_APPS` list and add `rest_framework`, `corsheaders`, `drf_spectacular`, and `app`:

```python
# myproject/settings.py

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third party
    'rest_framework',           # ← add
    'rest_framework.authtoken', # ← add
    'corsheaders',              # ← add
    'drf_spectacular',          # ← add
    # Local apps
    'app',                      # ← add
]
```

### 6b. Add DRF global defaults

Add this block anywhere in `settings.py` (conventionally near the bottom, before any custom settings):

```python
# myproject/settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```
> **Note:** The `DEFAULT_SCHEMA_CLASS` line is what connects drf-spectacular to every DRF view. Without it, the generated schema will be empty.

**What this does:**

- `DEFAULT_AUTHENTICATION_CLASSES` — every API view will use JWT Authentication by default. You can override this per-view.
- `DEFAULT_PERMISSION_CLASSES` — every API view requires an authenticated user by default. Endpoints that should be public (like registration) will explicitly override this with `AllowAny`.

Setting secure defaults globally is better than remembering to add `permission_classes` to every view individually.

### 6c. Add the CORS middleware

`corsheaders` works as a Django middleware — it must be placed **as high as possible** in the list, and specifically **before** `CommonMiddleware`:

```python
# myproject/settings.py

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',          # ← add at the top
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

> **Warning: Middleware order matters.**  
> `CorsMiddleware` must come before `CommonMiddleware`. If you place it anywhere else, CORS headers may not be added to every response.

### 6d. Configure CORS and drf-spectacular

Add this block to the bottom of `settings.py`:

```python
# myproject/settings.py

# ── CORS ──────────────────────────────────────────────────────────────────────
# Allow the Swagger UI (and any browser) to call the API during development.
# In production, replace with an explicit list of allowed origins.
CORS_ALLOW_ALL_ORIGINS = True   # development only

# To restrict to specific origins in production, use this instead:
# CORS_ALLOWED_ORIGINS = [
#     "https://yourfrontend.com",
#     "http://localhost:3000",
# ]

# ── OpenAPI / Spectacular ──────────────────────────────────────────────────────
SPECTACULAR_SETTINGS = {
    'TITLE': 'My Project API',
    'DESCRIPTION': 'Django REST Framework API — built for DHBW Software Engineering',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,   # hides the /api/schema/ endpoint from the UI itself
}
```

### 6e. Add the Schema URLs

Open `myproject/urls.py` and add the three spectacular routes:

```python
# myproject/urls.py

from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView
from drf_spectacular.views import (
    SpectacularAPIView,
    SpectacularSwaggerView,
    SpectacularRedocView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api-auth/', include('rest_framework.urls')),

    # OpenAPI schema + UI
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/schema/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

---

## 7. Run the First Migration

Django's built-in apps (admin, auth, sessions, etc.) need database tables. Create them:

```bash
python manage.py migrate
```

Expected output ends with something like:

```
Applying auth.0012_alter_user_first_name_max_length... OK
Applying sessions.0001_initial... OK
```

This creates a `db.sqlite3` file in your project root — the development database.

> **Note:** SQLite is the default database in Django. It stores everything in a single file and requires zero configuration, making it perfect for development. For production you would use PostgreSQL or MySQL.

---

## 8. Create a Superuser

Create an admin account so you can log into Django's built-in admin interface:

```bash
python manage.py createsuperuser
```

Enter a username, email (optional), and password when prompted.

---

## 9. Start the Development Server

```bash
python manage.py runserver
```

Expected output:

```
Django version 5.2.x, using settings 'myproject.settings'
Starting development server at http://127.0.0.1:8000/
Quit the server with CONTROL-C.
```

Open [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/) in your browser and log in with the superuser you created. You should see the Django admin dashboard.

Open [http://127.0.0.1:8000/api-auth/login/](http://127.0.0.1:8000/api-auth/login/) to confirm DRF's browsable API login page loads.

Open [http://127.0.0.1:8000/api/schema/swagger-ui/](http://127.0.0.1:8000/api/schema/swagger-ui/) to confirm Swagger UI loads and lists your endpoints.

---

## Pitfalls

> **Warning: Forgot the trailing `.` in `startproject`**  
> If your structure has `myproject/myproject/` (double nesting), delete the outer `myproject/` folder and re-run `django-admin startproject myproject .` with the dot.

> **Warning: `migrate` before adding apps**  
> Running `migrate` is fine at this point — it only applies built-in Django migrations. After you add your own models (in guides 03 and 04), you will need to run `makemigrations` first.

> **Warning: `manage.py` not found**  
> If you get `can't open file 'manage.py'`, you are in the wrong directory. Run `ls` — you should see `manage.py` in the current folder.

> **Warning: Port already in use**  
> If port 8000 is taken, start on a different port: `python manage.py runserver 8001`

---