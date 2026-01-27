# Models and Domain Logic for Django Clean Architecture

## Overview

Models in Clean Architecture represent domain entities. They contain data definitions and domain logic that is intrinsic to the entity itself—logic that doesn't depend on external services or application workflows.

**Key Principles:**
- Models define data structure and relationships
- Domain logic via `@property` (read-only, no side effects)
- Use `TextChoices` for enums
- UUID primary keys for distributed systems
- Strategic indexes for query performance
- No external service calls in models

## Model Anatomy

```python
# apps/orders/models.py
import uuid
from decimal import Decimal
from django.db import models
from django.utils import timezone


class Order(models.Model):
    """
    Order entity representing a customer's purchase.

    Domain logic is implemented via properties.
    State changes are handled by services.
    """

    # ====================================
    # ENUMS (TextChoices)
    # ====================================
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        CONFIRMED = 'confirmed', 'Confirmed'
        PROCESSING = 'processing', 'Processing'
        SHIPPED = 'shipped', 'Shipped'
        DELIVERED = 'delivered', 'Delivered'
        CANCELLED = 'cancelled', 'Cancelled'

    class Priority(models.TextChoices):
        LOW = 'low', 'Low'
        NORMAL = 'normal', 'Normal'
        HIGH = 'high', 'High'
        URGENT = 'urgent', 'Urgent'

    # ====================================
    # FIELDS
    # ====================================

    # Primary key
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False,
    )

    # Relationships
    customer = models.ForeignKey(
        'customers.Customer',
        on_delete=models.PROTECT,
        related_name='orders',
    )
    shipping_address = models.ForeignKey(
        'addresses.Address',
        on_delete=models.SET_NULL,
        related_name='shipping_orders',
        null=True,
        blank=True,
    )

    # Data fields
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
        db_index=True,
    )
    priority = models.CharField(
        max_length=20,
        choices=Priority.choices,
        default=Priority.NORMAL,
    )
    notes = models.TextField(blank=True, default='')
    tracking_number = models.CharField(max_length=100, blank=True, default='')

    # Timestamps
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    confirmed_at = models.DateTimeField(null=True, blank=True)
    shipped_at = models.DateTimeField(null=True, blank=True)
    delivered_at = models.DateTimeField(null=True, blank=True)

    # ====================================
    # META
    # ====================================
    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Order'
        verbose_name_plural = 'Orders'
        indexes = [
            models.Index(fields=['customer', 'status']),
            models.Index(fields=['status', 'created_at']),
            models.Index(fields=['created_at']),
            models.Index(fields=['tracking_number']),
        ]
        constraints = [
            models.CheckConstraint(
                check=models.Q(status__in=[s.value for s in Status]),
                name='valid_order_status',
            ),
        ]

    # ====================================
    # STRING REPRESENTATION
    # ====================================
    def __str__(self) -> str:
        return f"Order {self.id} ({self.customer})"

    # ====================================
    # DOMAIN PROPERTIES (Read-Only Logic)
    # ====================================

    @property
    def is_pending(self) -> bool:
        """Check if order is in pending status."""
        return self.status == self.Status.PENDING

    @property
    def is_cancellable(self) -> bool:
        """Check if order can be cancelled."""
        return self.status in (
            self.Status.PENDING,
            self.Status.CONFIRMED,
        )

    @property
    def is_editable(self) -> bool:
        """Check if order can be modified."""
        return self.status == self.Status.PENDING

    @property
    def is_completed(self) -> bool:
        """Check if order has reached final state."""
        return self.status in (
            self.Status.DELIVERED,
            self.Status.CANCELLED,
        )

    @property
    def is_active(self) -> bool:
        """Check if order is in an active (non-terminal) state."""
        return not self.is_completed

    @property
    def total(self) -> Decimal:
        """Calculate order total from items."""
        return sum(
            item.subtotal for item in self.items.all()
        ) or Decimal('0.00')

    @property
    def items_count(self) -> int:
        """Count total items in order."""
        return self.items.count()

    @property
    def days_since_created(self) -> int:
        """Days elapsed since order creation."""
        delta = timezone.now() - self.created_at
        return delta.days

    @property
    def is_overdue(self) -> bool:
        """Check if order is overdue for shipping."""
        if self.status not in (self.Status.CONFIRMED, self.Status.PROCESSING):
            return False
        return self.days_since_created > 3

    @property
    def status_display_class(self) -> str:
        """CSS class for status display."""
        status_classes = {
            self.Status.PENDING: 'warning',
            self.Status.CONFIRMED: 'info',
            self.Status.PROCESSING: 'primary',
            self.Status.SHIPPED: 'primary',
            self.Status.DELIVERED: 'success',
            self.Status.CANCELLED: 'danger',
        }
        return status_classes.get(self.status, 'secondary')
```

## UUID Primary Keys

### Why UUID?

1. **Distributed systems**: No central ID generator needed
2. **Security**: Harder to guess/enumerate
3. **Merge-friendly**: No conflicts when merging databases
4. **URL-safe**: Can be used in URLs without encoding

### Implementation

```python
import uuid
from django.db import models


class BaseModel(models.Model):
    """Abstract base model with UUID primary key."""
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False,
    )

    class Meta:
        abstract = True


class Order(BaseModel):
    """Order inherits UUID primary key."""
    customer = models.ForeignKey('Customer', on_delete=models.PROTECT)
    ...
```

### UUID in URLs

```python
# apps/orders/urls.py
from django.urls import path
from apps.orders import views

urlpatterns = [
    path('<uuid:pk>/', views.OrderDetailView.as_view(), name='order-detail'),
    path('<uuid:order_id>/cancel/', views.CancelOrderView.as_view(), name='order-cancel'),
]
```

## TextChoices for Enums

### Basic Enum

```python
class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        CONFIRMED = 'confirmed', 'Confirmed'
        SHIPPED = 'shipped', 'Shipped'

    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
    )
```

### Enum with Methods

```python
class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        CONFIRMED = 'confirmed', 'Confirmed'
        PROCESSING = 'processing', 'Processing'
        SHIPPED = 'shipped', 'Shipped'
        DELIVERED = 'delivered', 'Delivered'
        CANCELLED = 'cancelled', 'Cancelled'

        @classmethod
        def active_statuses(cls) -> list[str]:
            """Return list of active (non-terminal) statuses."""
            return [
                cls.PENDING,
                cls.CONFIRMED,
                cls.PROCESSING,
                cls.SHIPPED,
            ]

        @classmethod
        def terminal_statuses(cls) -> list[str]:
            """Return list of terminal statuses."""
            return [cls.DELIVERED, cls.CANCELLED]
```

### IntegerChoices for Numeric Enums

```python
class Order(models.Model):
    class Priority(models.IntegerChoices):
        LOW = 1, 'Low'
        NORMAL = 2, 'Normal'
        HIGH = 3, 'High'
        URGENT = 4, 'Urgent'

    priority = models.IntegerField(
        choices=Priority.choices,
        default=Priority.NORMAL,
    )
```

## Relationships

### ForeignKey

```python
class Order(models.Model):
    # PROTECT: Prevent deletion if orders exist
    customer = models.ForeignKey(
        'customers.Customer',
        on_delete=models.PROTECT,
        related_name='orders',
    )

    # SET_NULL: Allow deletion, set to null
    assigned_agent = models.ForeignKey(
        'users.User',
        on_delete=models.SET_NULL,
        related_name='assigned_orders',
        null=True,
        blank=True,
    )

    # CASCADE: Delete orders when customer deleted (rare)
    # temp_cart = models.ForeignKey(
    #     'carts.Cart',
    #     on_delete=models.CASCADE,
    # )
```

### Always Use related_name

```python
# BAD: No related_name
customer = models.ForeignKey('Customer', on_delete=models.PROTECT)
# Access: customer.order_set.all()  # Unclear

# GOOD: Explicit related_name
customer = models.ForeignKey(
    'Customer',
    on_delete=models.PROTECT,
    related_name='orders',
)
# Access: customer.orders.all()  # Clear
```

### ManyToMany with Through Model

```python
class Order(models.Model):
    products = models.ManyToManyField(
        'products.Product',
        through='OrderItem',
        related_name='orders',
    )


class OrderItem(models.Model):
    """Through model for Order-Product relationship."""
    order = models.ForeignKey(
        Order,
        on_delete=models.CASCADE,
        related_name='items',
    )
    product = models.ForeignKey(
        'products.Product',
        on_delete=models.PROTECT,
        related_name='order_items',
    )
    quantity = models.PositiveIntegerField(default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)

    class Meta:
        unique_together = ['order', 'product']

    @property
    def subtotal(self) -> Decimal:
        return self.quantity * self.unit_price
```

## Indexes

### When to Add Indexes

1. **Frequently filtered fields**: `status`, `created_at`
2. **Foreign keys** (automatic, but compound indexes may help)
3. **Fields used in ORDER BY**
4. **Unique constraints** (automatic)

### Single-Field Indexes

```python
class Order(models.Model):
    status = models.CharField(
        max_length=20,
        db_index=True,  # Simple single-field index
    )
```

### Compound Indexes

```python
class Order(models.Model):
    class Meta:
        indexes = [
            # For: WHERE customer_id = ? AND status = ?
            models.Index(fields=['customer', 'status']),

            # For: WHERE status = ? ORDER BY created_at DESC
            models.Index(fields=['status', '-created_at']),

            # For: WHERE created_at BETWEEN ? AND ?
            models.Index(fields=['created_at']),
        ]
```

### Partial Indexes (PostgreSQL)

```python
class Order(models.Model):
    class Meta:
        indexes = [
            # Index only active orders
            models.Index(
                fields=['customer', 'created_at'],
                condition=models.Q(status__in=['pending', 'confirmed']),
                name='idx_active_orders',
            ),
        ]
```

## Domain Properties

### Read-Only Logic Only

Properties should:
- Calculate derived values
- Check state conditions
- Never modify data
- Never call external services

```python
class Order(models.Model):
    @property
    def total(self) -> Decimal:
        """Calculate total from items. Read-only."""
        return sum(item.subtotal for item in self.items.all())

    @property
    def is_cancellable(self) -> bool:
        """Check if order can be cancelled. Read-only."""
        return self.status in (self.Status.PENDING, self.Status.CONFIRMED)

    @property
    def days_until_expiry(self) -> int | None:
        """Calculate days until order expires. Read-only."""
        if not self.expires_at:
            return None
        delta = self.expires_at - timezone.now()
        return max(0, delta.days)
```

### Avoid Side Effects

```python
# BAD: Property with side effect
@property
def total(self) -> Decimal:
    total = sum(item.subtotal for item in self.items.all())
    self.cached_total = total  # Side effect!
    self.save()  # Side effect!
    return total

# GOOD: Pure calculation
@property
def total(self) -> Decimal:
    return sum(item.subtotal for item in self.items.all())
```

### Cached Properties (Use Sparingly)

```python
from functools import cached_property

class Order(models.Model):
    @cached_property
    def total(self) -> Decimal:
        """Cached for repeated access within same request."""
        return sum(item.subtotal for item in self.items.all())
```

**Warning**: `cached_property` caches on the instance. If items change, the cached value is stale.

## Validation Methods

For complex validations that need context, use methods returning `tuple[bool, str]`:

```python
class Order(models.Model):
    def can_add_item(self, product: 'Product', quantity: int) -> tuple[bool, str]:
        """Check if item can be added to order."""
        if not self.is_editable:
            return False, "Order cannot be modified"

        if quantity <= 0:
            return False, "Quantity must be positive"

        existing = self.items.filter(product=product).first()
        if existing and existing.quantity + quantity > product.max_per_order:
            return False, f"Maximum {product.max_per_order} per order"

        return True, ""
```

**Note**: These methods still don't modify state. They just check if an operation is valid.

## Timestamps

### Standard Pattern

```python
class BaseModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

### Event-Specific Timestamps

```python
class Order(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Event timestamps (set explicitly by services)
    confirmed_at = models.DateTimeField(null=True, blank=True)
    shipped_at = models.DateTimeField(null=True, blank=True)
    delivered_at = models.DateTimeField(null=True, blank=True)
    cancelled_at = models.DateTimeField(null=True, blank=True)
```

## Soft Delete

### Pattern

```python
class SoftDeleteModel(models.Model):
    is_deleted = models.BooleanField(default=False)
    deleted_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        abstract = True

    def soft_delete(self):
        """Mark as deleted without removing from database."""
        self.is_deleted = True
        self.deleted_at = timezone.now()
        self.save(update_fields=['is_deleted', 'deleted_at'])


class SoftDeleteManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(is_deleted=False)


class Order(SoftDeleteModel):
    objects = SoftDeleteManager()
    all_objects = models.Manager()  # Include deleted
```

## Model Managers

### Custom Manager for Common Queries

```python
class OrderManager(models.Manager):
    def active(self) -> QuerySet['Order']:
        """Return only active orders."""
        return self.filter(
            status__in=Order.Status.active_statuses()
        )

    def pending_shipment(self) -> QuerySet['Order']:
        """Return orders ready for shipment."""
        return self.filter(
            status=Order.Status.CONFIRMED,
        ).select_related('customer', 'shipping_address')

    def for_customer(self, customer_id: UUID) -> QuerySet['Order']:
        """Return orders for a specific customer."""
        return self.filter(customer_id=customer_id)


class Order(models.Model):
    objects = OrderManager()

    # Usage:
    # Order.objects.active()
    # Order.objects.pending_shipment()
    # Order.objects.for_customer(customer_id)
```

## Django 5+ Features

### GeneratedField (Computed Database Column)

```python
from django.db.models import GeneratedField, F

class OrderItem(models.Model):
    quantity = models.PositiveIntegerField()
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)

    # Database-computed field (PostgreSQL, MySQL 8+)
    subtotal = GeneratedField(
        expression=F('quantity') * F('unit_price'),
        output_field=models.DecimalField(max_digits=12, decimal_places=2),
        db_persist=True,  # Store in database (not virtual)
    )
```

### db_default (Database-Level Defaults)

```python
from django.db.models.functions import Now

class Order(models.Model):
    created_at = models.DateTimeField(db_default=Now())

    # With expression
    order_number = models.CharField(
        max_length=20,
        db_default=Concat(
            Value('ORD-'),
            Cast(Now(), output_field=CharField())
        ),
    )
```

## Anti-Patterns to Avoid

### Business Logic in Models

```python
# BAD: Model contains business logic with side effects
class Order(models.Model):
    def cancel(self, reason: str):
        self.status = self.Status.CANCELLED
        self.save()
        # Side effects in model!
        self.customer.send_cancellation_email()
        self.refund_payment()

# GOOD: Model is just data + properties
class Order(models.Model):
    @property
    def is_cancellable(self) -> bool:
        return self.status in (...)

# Service handles the operation
class OrderService:
    def cancel(self, order: Order, reason: str) -> Order:
        order.status = Order.Status.CANCELLED
        order.save()
        return order
```

### External Service Calls

```python
# BAD: Model calls external service
class Order(models.Model):
    @property
    def shipping_cost(self) -> Decimal:
        return ShippingAPI.calculate(self.shipping_address)  # External call!

# GOOD: Calculate in service with explicit dependency
class OrderService:
    def __init__(self, shipping_api: ShippingAPI):
        self._shipping_api = shipping_api

    def get_shipping_cost(self, order: Order) -> Decimal:
        return self._shipping_api.calculate(order.shipping_address)
```

### Missing related_name

```python
# BAD: Implicit related_name
class Order(models.Model):
    customer = models.ForeignKey('Customer', on_delete=models.PROTECT)
# Access: customer.order_set.all()

# GOOD: Explicit related_name
class Order(models.Model):
    customer = models.ForeignKey(
        'Customer',
        on_delete=models.PROTECT,
        related_name='orders',
    )
# Access: customer.orders.all()
```

### Fat Models

```python
# BAD: Model doing too much
class Order(models.Model):
    def process(self):
        self.validate_items()
        self.calculate_shipping()
        self.apply_discount()
        self.charge_payment()
        self.send_confirmation()
        self.update_inventory()
        self.notify_warehouse()
        ...

# GOOD: Model is lean, services handle operations
class Order(models.Model):
    @property
    def is_processable(self) -> bool:
        return self.status == self.Status.PENDING and self.items.exists()
```
