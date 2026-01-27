# Coding Standards for Django APIs

## Overview

This guide covers mandatory coding standards: code style with Ruff, import ordering, security practices, migration best practices, and YAGNI principles.

## Code Formatting with Ruff

### Installation

```bash
pip install ruff
```

### Recommended Configuration

```toml
# ruff.toml
target-version = "py312"
line-length = 88

exclude = [
    ".git",
    ".venv",
    "venv",
    "__pycache__",
    "migrations",
    "staticfiles",
    "media",
]

[lint]
select = [
    "E",     # pycodestyle errors
    "W",     # pycodestyle warnings
    "F",     # pyflakes
    "I",     # isort
    "B",     # flake8-bugbear
    "C4",    # flake8-comprehensions
    "UP",    # pyupgrade
    "DJ",    # flake8-django
]

ignore = [
    "E501",  # line too long (handled by formatter)
]

[lint.isort]
known-first-party = ["apps", "config", "core"]
known-third-party = ["django", "rest_framework"]
section-order = ["future", "standard-library", "third-party", "first-party", "local-folder"]

[format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
```

### Usage

```bash
# Check for issues
ruff check .

# Fix auto-fixable issues
ruff check --fix .

# Format code
ruff format .

# Check and format in one command
ruff check --fix . && ruff format .
```

### Pre-commit Hook

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.4
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

## Import Ordering

### Order (Enforced by Ruff)

1. **Future imports** (`from __future__ import annotations`)
2. **Standard library** (`os`, `uuid`, `datetime`, etc.)
3. **Third-party** (`django`, `rest_framework`, etc.)
4. **First-party** (your apps)
5. **Local folder** (relative imports)

### Example

```python
from __future__ import annotations

import uuid
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import TYPE_CHECKING

from django.db import models, transaction
from django.utils import timezone
from rest_framework import serializers, status
from rest_framework.response import Response
from rest_framework.views import APIView

from apps.core.permissions import IsAuthenticated
from apps.orders.models import Order
from apps.orders.services import OrderService

if TYPE_CHECKING:
    from apps.users.models import User
```

### Rules

1. **One import per line** for clarity (Ruff handles this)
2. **Alphabetical within sections**
3. **Use `TYPE_CHECKING` for import-only types** to avoid circular imports
4. **Prefer absolute imports** over relative imports

## API Response Standards

### Success Responses

```python
# Return data directly
return Response(serializer.data)

# With message
return Response({"message": "Order created successfully"})

# Created resource
return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### Error Responses

```python
# Business logic error
return Response(
    {"error": "Insufficient stock for this order"},
    status=status.HTTP_400_BAD_REQUEST,
)

# Not found
return Response(
    {"error": "Order not found"},
    status=status.HTTP_404_NOT_FOUND,
)

# Permission denied
return Response(
    {"error": "You don't have permission to cancel this order"},
    status=status.HTTP_403_FORBIDDEN,
)
```

### HTTP Status Codes

| Code | Use Case |
|------|----------|
| 200 | Successful GET, PUT, PATCH, DELETE |
| 201 | Successful POST (resource created) |
| 204 | Successful DELETE (no content) |
| 400 | Bad request / validation error / business logic error |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (no permission) |
| 404 | Resource not found |
| 409 | Conflict (duplicate, state conflict) |
| 500 | Internal server error (unexpected) |

### Consistent Error Format

```python
# apps/core/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)

    if response is not None:
        # Ensure consistent format
        if 'detail' in response.data:
            response.data = {'error': response.data['detail']}

    return response
```

## Security Practices

### Input Validation

```python
# ALWAYS validate via serializers
class CreateOrderSerializer(serializers.Serializer):
    customer_id = serializers.UUIDField()
    items = OrderItemSerializer(many=True, min_length=1)
    notes = serializers.CharField(max_length=500, required=False)

# View validates before processing
serializer = CreateOrderSerializer(data=request.data)
serializer.is_valid(raise_exception=True)  # Returns 400 if invalid
```

### Object Ownership

```python
# ALWAYS check ownership before modifications
class OrderService:
    @staticmethod
    def get_for_user(order_id: UUID, user_id: UUID) -> Order | None:
        """Get order only if user owns it."""
        return (
            Order.objects
            .filter(id=order_id, customer__user_id=user_id)
            .first()
        )

# In use case
order = self._order_service.get_for_user(input_dto.order_id, input_dto.user_id)
if not order:
    return CancelOrderOutput(success=False, error="Order not found")
```

### No Raw SQL

```python
# BAD - SQL injection risk
User.objects.raw(f"SELECT * FROM users WHERE email = '{email}'")

# GOOD - Use ORM
User.objects.filter(email=email)

# If raw SQL is absolutely necessary, use parameterization
User.objects.raw("SELECT * FROM users WHERE email = %s", [email])
```

### Secrets Management

```python
# BAD - Secrets in code
STRIPE_KEY = "sk_live_abc123"

# GOOD - Environment variables
import environ
env = environ.Env()
STRIPE_KEY = env("STRIPE_KEY")

# GOOD - With default for development
DEBUG = env.bool("DEBUG", default=False)
```

### Never Expose Internal IDs

```python
# BAD - Exposes auto-increment ID
class OrderReadSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ['id', 'status']  # id is auto-increment

# GOOD - Use UUID
class Order(models.Model):
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
```

## Database Migrations

### Safe Migration Practices

```python
# 1. Add columns as nullable first
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name='order',
            name='tracking_number',
            field=models.CharField(max_length=100, null=True, blank=True),
        ),
    ]

# 2. Later migration: backfill data, then make non-null if needed
```

### Migration Checklist

- [ ] Test migrations on copy of production data
- [ ] Check for long-running operations (table locks)
- [ ] Add indexes in separate migration from data changes
- [ ] Never edit applied migrations
- [ ] Use `--plan` to preview: `python manage.py migrate --plan`

### Index Migrations

```python
# Separate migration for indexes (can run concurrently)
class Migration(migrations.Migration):
    atomic = False  # Allow concurrent index creation

    operations = [
        migrations.AddIndex(
            model_name='order',
            index=models.Index(
                fields=['status', 'created_at'],
                name='order_status_created_idx',
            ),
        ),
    ]
```

## YAGNI - No Over-Engineering

### Rules

1. **Don't build features "for the future"**
   ```python
   # BAD - Building for hypothetical future
   class Order(models.Model):
       status = models.CharField(max_length=20)
       legacy_status = models.CharField(max_length=20, null=True)  # "might need later"
       status_v2 = models.JSONField(null=True)  # "for future flexibility"

   # GOOD - Only what's needed now
   class Order(models.Model):
       status = models.CharField(max_length=20, choices=Status.choices)
   ```

2. **3 uses before abstracting**
   ```python
   # BAD - Premature abstraction for one use case
   class BaseEntityService(Generic[T]):
       def get_by_id(self, id: UUID) -> T | None: ...
       def create(self, **kwargs) -> T: ...
       def update(self, entity: T, **kwargs) -> T: ...

   # GOOD - Simple, direct implementation
   class OrderService:
       @staticmethod
       def get_by_id(order_id: UUID) -> Order | None:
           return Order.objects.filter(id=order_id).first()
   ```

3. **Comments explain "why", not "what"**
   ```python
   # BAD
   # Increment the counter by 1
   counter += 1

   # GOOD
   # Retry count includes the initial attempt
   counter += 1
   ```

4. **Simple solutions first**
   ```python
   # BAD - Over-engineered for simple case
   class OrderStatusStateMachine:
       transitions = {
           'pending': ['confirmed', 'cancelled'],
           'confirmed': ['shipped', 'cancelled'],
       }

       def can_transition(self, from_status, to_status): ...
       def transition(self, order, to_status): ...

   # GOOD - Simple validation for 4 statuses
   def validate_can_ship(self, order: Order) -> tuple[bool, str]:
       if order.status != Order.Status.CONFIRMED:
           return False, "Only confirmed orders can be shipped"
       return True, ""
   ```

## Logging Standards

### Use Logging, Not Print

```python
# BAD
print(f"Order created: {order.id}")

# GOOD
import logging
logger = logging.getLogger(__name__)
logger.info("Order created", extra={"order_id": str(order.id)})
```

### Log Levels

| Level | Use Case |
|-------|----------|
| DEBUG | Detailed info for debugging |
| INFO | General operational events |
| WARNING | Unexpected but handled situations |
| ERROR | Errors that need attention |
| CRITICAL | System failures |

```python
logger.debug("Processing order items", extra={"count": len(items)})
logger.info("Order created", extra={"order_id": str(order.id)})
logger.warning("Stock low", extra={"product_id": str(product.id), "stock": product.stock})
logger.error("Payment failed", extra={"order_id": str(order.id), "error": str(e)})
```

## Documentation Standards

### Docstrings for Public Functions

```python
class OrderService:
    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        """
        Retrieve an order by its ID with related data.

        Args:
            order_id: The UUID of the order to retrieve.

        Returns:
            The order with customer and items prefetched, or None if not found.
        """
        return (
            Order.objects
            .select_related('customer')
            .prefetch_related('items')
            .filter(id=order_id)
            .first()
        )
```

### When to Add Docstrings

- [ ] Public service methods
- [ ] Use case classes (class-level docstring)
- [ ] Complex validation logic
- [ ] Non-obvious business rules

### Type Hints

```python
from uuid import UUID
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from apps.orders.models import Order

class OrderService:
    @staticmethod
    def get_by_id(order_id: UUID) -> "Order | None":
        ...

    def create(
        self,
        customer_id: UUID,
        items: list[OrderItemInput],
        notes: str | None = None,
    ) -> "Order":
        ...

    def validate_can_cancel(self, order: "Order") -> tuple[bool, str]:
        ...
```

## Code Review Checklist

### Architecture
- [ ] Views are thin (< 20 lines of logic)
- [ ] Business logic in use cases, not views
- [ ] No business logic in serializers
- [ ] Services are reusable, use cases orchestrate

### Security
- [ ] All input validated via serializers
- [ ] Object ownership checked before modifications
- [ ] No raw SQL without parameterization
- [ ] No secrets in code

### Code Quality
- [ ] Ruff passes with no errors
- [ ] Imports properly ordered
- [ ] Type hints on public methods
- [ ] Docstrings on complex logic
- [ ] No print statements (use logging)

### Database
- [ ] Queries optimized (select_related/prefetch_related)
- [ ] Migrations tested
- [ ] New fields nullable or with defaults
- [ ] Indexes on filtered/ordered fields
