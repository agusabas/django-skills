# Production Configuration for Django APIs

## Overview

This guide covers production-ready configuration for Django REST Framework APIs: settings, security, database, caching, logging, and deployment.

## Production Settings Structure

### Settings Organization

```
config/
├── __init__.py
├── settings/
│   ├── __init__.py
│   ├── base.py          # Shared settings
│   ├── development.py   # Local development
│   ├── staging.py       # Staging environment
│   └── production.py    # Production environment
├── urls.py
├── wsgi.py
└── asgi.py
```

### Base Settings

```python
# config/settings/base.py
from pathlib import Path
import environ

BASE_DIR = Path(__file__).resolve().parent.parent.parent

env = environ.Env()
environ.Env.read_env(BASE_DIR / '.env')

# Apps
DJANGO_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

THIRD_PARTY_APPS = [
    'rest_framework',
    'django_filters',
    'corsheaders',
]

LOCAL_APPS = [
    'apps.core',
    'apps.users',
    'apps.orders',
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# Middleware
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
    'EXCEPTION_HANDLER': 'apps.core.exceptions.custom_exception_handler',
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
}

# Internationalization
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_TZ = True

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Default primary key field type
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# Custom user model
AUTH_USER_MODEL = 'users.User'
```

### Production Settings

```python
# config/settings/production.py
from .base import *

DEBUG = False

SECRET_KEY = env('SECRET_KEY')

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT', default='5432'),
        'CONN_MAX_AGE': 60,
        'CONN_HEALTH_CHECKS': True,
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000',  # 30 seconds
        },
    }
}

# Cache
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': env('REDIS_URL'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'api',
        'TIMEOUT': 300,
    }
}

# Security
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True

# CORS
CORS_ALLOWED_ORIGINS = env.list('CORS_ALLOWED_ORIGINS')
CORS_ALLOW_CREDENTIALS = True

# Static files with WhiteNoise
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Logging
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(levelname)s %(name)s %(message)s',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
        'apps': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}

# Sentry
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    integrations=[DjangoIntegration()],
    traces_sample_rate=0.1,
    send_default_pii=False,
    environment=env('ENVIRONMENT', default='production'),
)
```

## Environment Variables

### .env.example

```bash
# Django
SECRET_KEY=your-super-secret-key-here
DEBUG=False
ALLOWED_HOSTS=api.example.com,www.api.example.com
ENVIRONMENT=production

# Database
DB_NAME=myapp
DB_USER=myapp
DB_PASSWORD=supersecretpassword
DB_HOST=db.example.com
DB_PORT=5432

# Redis
REDIS_URL=redis://redis.example.com:6379/0

# CORS
CORS_ALLOWED_ORIGINS=https://app.example.com,https://admin.example.com

# JWT
JWT_SECRET_KEY=your-jwt-secret-key

# Sentry
SENTRY_DSN=https://xxx@sentry.io/xxx

# Email
EMAIL_HOST=smtp.sendgrid.net
EMAIL_PORT=587
EMAIL_HOST_USER=apikey
EMAIL_HOST_PASSWORD=your-sendgrid-api-key
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=noreply@example.com
```

## Database Configuration

### Connection Pooling with PgBouncer

```python
# For PgBouncer, use transaction pooling
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('PGBOUNCER_HOST'),  # PgBouncer host
        'PORT': env('PGBOUNCER_PORT', default='6432'),
        'CONN_MAX_AGE': 0,  # Don't persist connections with PgBouncer
        'DISABLE_SERVER_SIDE_CURSORS': True,  # Required for transaction pooling
    }
}
```

### Read Replicas

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST_PRIMARY'),
        'PORT': '5432',
    },
    'replica': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST_REPLICA'),
        'PORT': '5432',
    },
}

# Database router
DATABASE_ROUTERS = ['apps.core.routers.PrimaryReplicaRouter']
```

```python
# apps/core/routers.py
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return 'replica'

    def db_for_write(self, model, **hints):
        return 'default'

    def allow_relation(self, obj1, obj2, **hints):
        return True

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == 'default'
```

## Caching Strategy

### Cache Configuration

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': env('REDIS_URL'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'api',
        'TIMEOUT': 300,  # 5 minutes default
    },
    'sessions': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': env('REDIS_URL'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'db': 1,
        },
        'KEY_PREFIX': 'session',
        'TIMEOUT': 86400,  # 1 day
    },
}

# Use cache for sessions
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'sessions'
```

### Caching Patterns in Services

```python
from django.core.cache import cache

class ProductService:
    CACHE_TTL = 300  # 5 minutes

    @staticmethod
    def get_by_id(product_id: UUID) -> Product | None:
        cache_key = f'product:{product_id}'
        product = cache.get(cache_key)

        if product is None:
            product = (
                Product.objects
                .select_related('category')
                .filter(id=product_id)
                .first()
            )
            if product:
                cache.set(cache_key, product, ProductService.CACHE_TTL)

        return product

    def update(self, product: Product, **kwargs) -> Product:
        for key, value in kwargs.items():
            setattr(product, key, value)
        product.save()

        # Invalidate cache
        cache.delete(f'product:{product.id}')

        return product
```

### Cache Invalidation with Signals

```python
# apps/products/signals.py
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from django.core.cache import cache

@receiver([post_save, post_delete], sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f'product:{instance.id}')
    cache.delete('products:list')  # Invalidate list cache
```

## Logging

### Structured JSON Logging

```bash
pip install python-json-logger
```

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
            'format': '%(asctime)s %(levelname)s %(name)s %(funcName)s %(message)s',
        },
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
    },
    'filters': {
        'require_debug_false': {
            '()': 'django.utils.log.RequireDebugFalse',
        },
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'json',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': 'WARNING',
            'propagate': False,
        },
        'django.request': {
            'handlers': ['console'],
            'level': 'ERROR',
            'propagate': False,
        },
        'apps': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

### Logging in Use Cases

```python
import logging

logger = logging.getLogger(__name__)

class CreateOrderUseCase:
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        logger.info(
            'Creating order',
            extra={
                'customer_id': str(input_dto.customer_id),
                'item_count': len(input_dto.items),
            }
        )

        try:
            # ... business logic ...

            logger.info(
                'Order created successfully',
                extra={'order_id': str(order.id)},
            )
            return CreateOrderOutput(success=True, order_id=order.id)

        except Exception as e:
            logger.exception(
                'Failed to create order',
                extra={
                    'customer_id': str(input_dto.customer_id),
                    'error': str(e),
                }
            )
            return CreateOrderOutput(success=False, error='Internal error')
```

### Request Logging Middleware

```python
# apps/core/middleware.py
import logging
import time
import uuid

logger = logging.getLogger('api.requests')

class RequestLoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request.request_id = str(uuid.uuid4())
        start_time = time.time()

        response = self.get_response(request)

        duration = time.time() - start_time

        logger.info(
            'API Request',
            extra={
                'request_id': request.request_id,
                'method': request.method,
                'path': request.path,
                'status_code': response.status_code,
                'duration_ms': round(duration * 1000, 2),
                'user_id': str(request.user.id) if request.user.is_authenticated else None,
            }
        )

        response['X-Request-ID'] = request.request_id
        return response
```

## Error Monitoring with Sentry

```bash
pip install sentry-sdk
```

```python
# config/settings/production.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration
from sentry_sdk.integrations.redis import RedisIntegration
from sentry_sdk.integrations.celery import CeleryIntegration

sentry_sdk.init(
    dsn=env('SENTRY_DSN'),
    integrations=[
        DjangoIntegration(),
        RedisIntegration(),
        CeleryIntegration(),
    ],
    traces_sample_rate=0.1,  # 10% of transactions for performance monitoring
    profiles_sample_rate=0.1,  # 10% of transactions for profiling
    send_default_pii=False,  # Don't send PII
    environment=env('ENVIRONMENT', default='production'),
    release=env('GIT_SHA', default='unknown'),
)
```

### Custom Sentry Context

```python
# apps/core/middleware.py
import sentry_sdk

class SentryContextMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.user.is_authenticated:
            sentry_sdk.set_user({
                'id': str(request.user.id),
                'email': request.user.email,
            })

        sentry_sdk.set_context('request', {
            'path': request.path,
            'method': request.method,
        })

        return self.get_response(request)
```

## Health Checks

```python
# apps/core/views.py
from django.db import connection
from django.core.cache import cache
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import AllowAny

class HealthCheckView(APIView):
    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request):
        health = {
            'status': 'healthy',
            'checks': {},
        }

        # Database check
        try:
            with connection.cursor() as cursor:
                cursor.execute('SELECT 1')
            health['checks']['database'] = 'ok'
        except Exception as e:
            health['checks']['database'] = f'error: {str(e)}'
            health['status'] = 'unhealthy'

        # Cache check
        try:
            cache.set('health_check', 'ok', 10)
            if cache.get('health_check') == 'ok':
                health['checks']['cache'] = 'ok'
            else:
                health['checks']['cache'] = 'error: cache not responding'
                health['status'] = 'unhealthy'
        except Exception as e:
            health['checks']['cache'] = f'error: {str(e)}'
            health['status'] = 'unhealthy'

        status_code = 200 if health['status'] == 'healthy' else 503
        return Response(health, status=status_code)


class ReadinessCheckView(APIView):
    """For Kubernetes readiness probes."""
    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request):
        return Response({'status': 'ready'})


class LivenessCheckView(APIView):
    """For Kubernetes liveness probes."""
    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request):
        return Response({'status': 'alive'})
```

```python
# urls.py
urlpatterns = [
    path('health/', HealthCheckView.as_view()),
    path('ready/', ReadinessCheckView.as_view()),
    path('alive/', LivenessCheckView.as_view()),
]
```

## WSGI/ASGI Configuration

### Gunicorn Configuration

```python
# gunicorn.conf.py
import multiprocessing

# Binding
bind = '0.0.0.0:8000'

# Workers
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = 'sync'  # or 'gevent' for async
worker_connections = 1000
timeout = 30
keepalive = 2

# Logging
accesslog = '-'
errorlog = '-'
loglevel = 'info'
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Process naming
proc_name = 'myapp'

# Server mechanics
daemon = False
pidfile = None
umask = 0
user = None
group = None
tmp_upload_dir = None

# Hooks
def on_starting(server):
    pass

def on_reload(server):
    pass

def worker_int(worker):
    pass
```

### Run Command

```bash
gunicorn config.wsgi:application -c gunicorn.conf.py
```

### ASGI with Uvicorn (for async views)

```python
# config/asgi.py
import os
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')

application = get_asgi_application()
```

```bash
uvicorn config.asgi:application --host 0.0.0.0 --port 8000 --workers 4
```

## Docker Configuration

### Dockerfile

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000

CMD ["gunicorn", "config.wsgi:application", "-c", "gunicorn.conf.py"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DJANGO_SETTINGS_MODULE=config.settings.production
    env_file:
      - .env
    depends_on:
      - db
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Production Checklist

### Before Deployment
- [ ] `DEBUG = False`
- [ ] `SECRET_KEY` from environment variable
- [ ] `ALLOWED_HOSTS` configured
- [ ] Database connection pooling configured
- [ ] Redis cache configured
- [ ] CORS origins restricted
- [ ] HTTPS enforced
- [ ] Security headers enabled
- [ ] Sentry configured
- [ ] Logging configured with JSON format

### Security
- [ ] All secrets in environment variables
- [ ] Password validation configured
- [ ] Rate limiting on auth endpoints
- [ ] JWT tokens with short expiry
- [ ] CSRF protection enabled
- [ ] SQL injection protection (use ORM)
- [ ] XSS protection headers

### Performance
- [ ] Database indexes on filtered fields
- [ ] Query optimization (select_related/prefetch_related)
- [ ] Response caching where appropriate
- [ ] Pagination on all list endpoints
- [ ] Gunicorn workers configured

### Monitoring
- [ ] Health check endpoints
- [ ] Error tracking (Sentry)
- [ ] Request logging
- [ ] Performance monitoring
- [ ] Alerting configured
