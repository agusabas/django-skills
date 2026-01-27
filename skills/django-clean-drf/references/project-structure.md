# Project Structure for Clean Architecture Django

## Overview

This document defines the directory structure and organization for Django projects following Clean Architecture principles. The structure enforces separation of concerns and makes dependencies explicit.

## Project Root Structure

```
project_root/
├── config/                    # Project configuration
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py           # Shared settings
│   │   ├── local.py          # Development settings
│   │   ├── production.py     # Production settings
│   │   └── test.py           # Test settings
│   ├── urls.py               # Root URL configuration
│   ├── wsgi.py
│   └── asgi.py
├── apps/                      # All Django applications
│   ├── __init__.py
│   ├── users/
│   ├── orders/
│   └── ...
├── core/                      # Shared utilities and base classes
│   ├── __init__.py
│   ├── models.py             # Base model classes
│   ├── services.py           # Base service classes
│   ├── use_cases.py          # Base use case classes
│   ├── permissions.py        # Custom DRF permissions
│   └── pagination.py         # Custom pagination
├── tests/                     # Project-level tests
│   ├── conftest.py           # Pytest fixtures
│   └── integration/
├── manage.py
├── pyproject.toml            # Project dependencies and tools config
├── ruff.toml                 # Linting configuration
└── requirements/
    ├── base.txt
    ├── local.txt
    └── production.txt
```

## Application Structure

Each Django app follows this internal structure:

```
apps/<app_name>/
├── __init__.py
├── apps.py                   # App configuration
├── models.py                 # Domain entities
├── views.py                  # HTTP handlers (thin)
├── serializers.py            # DRF serializers (Read/Write)
├── urls.py                   # URL routing
├── admin.py                  # Django Admin configuration
├── permissions.py            # App-specific permissions (optional)
├── filters.py                # DRF filters (optional)
├── signals.py                # Django signals (optional)
├── services/                 # Business logic layer
│   ├── __init__.py           # Export all services
│   ├── order_service.py      # Entity-specific service
│   └── inventory_service.py
├── use_cases/                # Application layer
│   ├── __init__.py           # Export all use cases
│   ├── create_order.py       # Single use case per file
│   ├── cancel_order.py
│   └── ship_order.py
├── tests/                    # App-specific tests
│   ├── __init__.py
│   ├── conftest.py           # App fixtures
│   ├── factories.py          # Factory Boy factories
│   ├── test_models.py
│   ├── test_services.py
│   ├── test_use_cases.py
│   └── test_views.py
└── migrations/
    └── __init__.py
```

## File Naming Conventions

### Use Cases
- One use case per file
- Name pattern: `<action>_<entity>.py`
- Examples:
  - `create_order.py`
  - `cancel_reservation.py`
  - `process_payment.py`
  - `send_notification.py`

### Services
- One service class per file (or related group)
- Name pattern: `<entity>_service.py`
- Examples:
  - `order_service.py`
  - `payment_service.py`
  - `notification_service.py`

### Serializers
- Use suffixes to indicate purpose
- Name patterns:
  - `<Entity>ReadSerializer` - For GET responses
  - `<Entity>CreateSerializer` - For POST requests
  - `<Entity>UpdateSerializer` - For PUT/PATCH requests
  - `<Entity>ListSerializer` - For list views (minimal fields)

### Models
- One file `models.py` per app
- For large apps, split into `models/` package:
  ```
  models/
  ├── __init__.py      # Import and export all models
  ├── order.py
  └── order_item.py
  ```

## Import Rules and Dependencies

### Layer Hierarchy (Outer to Inner)
```
1. Views (HTTP Layer)
   ├── Can import: Use Cases, Serializers
   └── Cannot import: Services, Models directly

2. Serializers (HTTP Layer)
   ├── Can import: Models (for ModelSerializer)
   └── Cannot import: Use Cases, Services

3. Use Cases (Application Layer)
   ├── Can import: Services, DTOs
   └── Cannot import: Views, Serializers

4. Services (Domain Layer)
   ├── Can import: Models, other Services
   └── Cannot import: Views, Use Cases, Serializers

5. Models (Domain Layer)
   ├── Can import: Django ORM, other Models
   └── Cannot import: Services, Use Cases, Views
```

### Import Examples

```python
# apps/orders/views.py - CORRECT
from apps.orders.use_cases import CreateOrderUseCase
from apps.orders.serializers import CreateOrderSerializer, OrderReadSerializer

# apps/orders/views.py - WRONG
from apps.orders.services import OrderService  # Don't use services directly in views
from apps.orders.models import Order  # Don't use models directly in views

# apps/orders/use_cases/create_order.py - CORRECT
from apps.orders.services import OrderService, InventoryService

# apps/orders/services/order_service.py - CORRECT
from apps.orders.models import Order, OrderItem
from apps.payments.services import PaymentService  # Cross-app service is OK
```

### Cross-App Dependencies

When apps need to communicate:

```python
# PREFERRED: Import services from other apps
from apps.payments.services import PaymentService

# ACCEPTABLE: Import models for relationships
from apps.users.models import User

# AVOID: Circular dependencies between use cases
# If needed, extract shared logic to a service
```

## Module Exports (__init__.py)

### services/__init__.py
```python
from apps.orders.services.order_service import OrderService
from apps.orders.services.inventory_service import InventoryService

__all__ = ['OrderService', 'InventoryService']
```

### use_cases/__init__.py
```python
from apps.orders.use_cases.create_order import (
    CreateOrderUseCase,
    CreateOrderInput,
    CreateOrderOutput,
)
from apps.orders.use_cases.cancel_order import (
    CancelOrderUseCase,
    CancelOrderInput,
    CancelOrderOutput,
)

__all__ = [
    'CreateOrderUseCase',
    'CreateOrderInput',
    'CreateOrderOutput',
    'CancelOrderUseCase',
    'CancelOrderInput',
    'CancelOrderOutput',
]
```

## Core Module (Shared Code)

The `core/` directory contains reusable base classes:

### core/models.py
```python
import uuid
from django.db import models


class BaseModel(models.Model):
    """Base model with UUID primary key and timestamps."""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class TenantAwareModel(BaseModel):
    """Base model for multi-tenant applications."""
    # Override in concrete models to add tenant FK

    class Meta:
        abstract = True
```

### core/use_cases.py
```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True, slots=True)
class BaseOutput:
    """Base output for all use cases."""
    success: bool
    error: str | None = None


class UseCase(Protocol):
    """Protocol for use cases."""
    def execute(self, input_dto) -> BaseOutput:
        ...
```

### core/services.py
```python
from typing import TypeVar, Generic
from django.db.models import Model, QuerySet

T = TypeVar('T', bound=Model)


class BaseQueryService(Generic[T]):
    """Base class for read-only services."""
    model: type[T]

    @classmethod
    def get_by_id(cls, pk) -> T | None:
        return cls.model.objects.filter(pk=pk).first()

    @classmethod
    def get_all(cls) -> QuerySet[T]:
        return cls.model.objects.all()
```

## Settings Organization

### config/settings/base.py
```python
from pathlib import Path
import os

BASE_DIR = Path(__file__).resolve().parent.parent.parent

# Apps organization
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
    'apps.users',
    'apps.orders',
    # Add new apps here
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

## URL Organization

### config/urls.py
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/users/', include('apps.users.urls')),
    path('api/orders/', include('apps.orders.urls')),
]
```

### apps/orders/urls.py
```python
from django.urls import path
from apps.orders import views

app_name = 'orders'

urlpatterns = [
    path('', views.OrderListCreateView.as_view(), name='order-list-create'),
    path('<uuid:pk>/', views.OrderDetailView.as_view(), name='order-detail'),
    path('<uuid:pk>/cancel/', views.CancelOrderView.as_view(), name='order-cancel'),
]
```

## pyproject.toml Configuration

```toml
[project]
name = "my-django-api"
version = "1.0.0"
requires-python = ">=3.12"

[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = ["test_*.py"]
addopts = "-v --tb=short"

[tool.coverage.run]
source = ["apps"]
omit = ["*/migrations/*", "*/tests/*", "*/__init__.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
]
```

## ruff.toml Configuration

```toml
target-version = "py312"
line-length = 88

[lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # Pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "DJ",   # flake8-django
]
ignore = ["E501"]

[lint.isort]
known-first-party = ["apps", "config", "core"]
section-order = ["future", "standard-library", "third-party", "first-party", "local-folder"]

[format]
quote-style = "double"
indent-style = "space"
```

## Anti-Patterns to Avoid

### Wrong: Single models.py with everything
```python
# apps/orders/models.py - BAD
class Order(models.Model):
    ...

    def create_order(self, data):  # Business logic in model
        ...

    def send_notification(self):  # Side effects in model
        ...
```

### Wrong: Fat views
```python
# apps/orders/views.py - BAD
class CreateOrderView(APIView):
    def post(self, request):
        # 50+ lines of business logic
        # Direct ORM queries
        # Email sending
        # Payment processing
        ...
```

### Wrong: Missing layer separation
```
apps/orders/
├── models.py
├── views.py      # Contains services, use cases, everything
├── serializers.py
└── urls.py
```

### Correct: Clean separation
```
apps/orders/
├── models.py           # Only entities
├── views.py            # Only HTTP handling
├── serializers.py      # Only I/O validation
├── services/           # Business logic
├── use_cases/          # Orchestration
└── tests/              # Comprehensive tests
```
