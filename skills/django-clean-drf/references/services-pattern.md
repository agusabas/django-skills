# Service Layer Pattern for Django Clean Architecture

## Overview

Services encapsulate domain logic and provide a clean API for use cases. They are the workhorses of Clean Architecture, containing the business rules and data access logic.

**Key Principles:**
- Static methods for queries (read operations)
- Instance methods for mutations (write operations)
- Validation methods return `tuple[bool, str]`
- No HTTP/Request awareness
- Always optimize queries with `select_related`/`prefetch_related`

## Service Anatomy

```python
# apps/orders/services/order_service.py
from uuid import UUID
from django.db import transaction
from django.db.models import QuerySet, Count, Sum, F
from apps.orders.models import Order, OrderItem


class OrderService:
    """
    Service for order-related business logic.

    Queries (static methods): Read-only operations
    Mutations (instance methods): State-changing operations
    Validations: Return tuple[bool, str]
    """

    # ====================================
    # QUERIES (Static Methods)
    # ====================================

    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        """Get order by ID with optimized relations."""
        return (
            Order.objects
            .select_related('customer', 'shipping_address')
            .prefetch_related('items__product')
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def get_by_id_for_update(order_id: UUID) -> Order | None:
        """Get order with row lock for update operations."""
        return (
            Order.objects
            .select_for_update()
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def list_for_customer(
        customer_id: UUID,
        status: Order.Status | None = None,
        limit: int = 20,
    ) -> QuerySet[Order]:
        """List orders for a customer with optional filtering."""
        qs = (
            Order.objects
            .filter(customer_id=customer_id)
            .select_related('customer')
            .prefetch_related('items')
            .order_by('-created_at')
        )
        if status:
            qs = qs.filter(status=status)
        return qs[:limit]

    @staticmethod
    def exists(order_id: UUID) -> bool:
        """Check if order exists."""
        return Order.objects.filter(id=order_id).exists()

    @staticmethod
    def count_by_status(customer_id: UUID) -> dict[str, int]:
        """Count orders by status for a customer."""
        result = (
            Order.objects
            .filter(customer_id=customer_id)
            .values('status')
            .annotate(count=Count('id'))
        )
        return {item['status']: item['count'] for item in result}

    # ====================================
    # MUTATIONS (Instance Methods)
    # ====================================

    def create(
        self,
        customer_id: UUID,
        items: list['OrderItemInput'],
        notes: str | None = None,
        shipping_address_id: UUID | None = None,
    ) -> Order:
        """Create a new order with items."""
        order = Order.objects.create(
            customer_id=customer_id,
            notes=notes or '',
            shipping_address_id=shipping_address_id,
        )
        for item in items:
            OrderItem.objects.create(
                order=order,
                product_id=item.product_id,
                quantity=item.quantity,
                unit_price=item.unit_price,
            )
        return order

    def update_status(self, order: Order, new_status: Order.Status) -> Order:
        """Update order status."""
        order.status = new_status
        order.save(update_fields=['status', 'updated_at'])
        return order

    def cancel(self, order: Order, reason: str | None = None) -> Order:
        """Cancel an order."""
        order.status = Order.Status.CANCELLED
        order.cancellation_reason = reason or ''
        order.save(update_fields=['status', 'cancellation_reason', 'updated_at'])
        return order

    def add_item(
        self,
        order: Order,
        product_id: UUID,
        quantity: int,
        unit_price: 'Decimal',
    ) -> OrderItem:
        """Add item to existing order."""
        return OrderItem.objects.create(
            order=order,
            product_id=product_id,
            quantity=quantity,
            unit_price=unit_price,
        )

    # ====================================
    # VALIDATIONS (Return tuple[bool, str])
    # ====================================

    def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
        """Check if order can be cancelled."""
        if order.status == Order.Status.SHIPPED:
            return False, "Cannot cancel shipped orders"
        if order.status == Order.Status.DELIVERED:
            return False, "Cannot cancel delivered orders"
        if order.status == Order.Status.CANCELLED:
            return False, "Order is already cancelled"
        return True, ""

    def validate_can_ship(self, order: Order) -> tuple[bool, str]:
        """Check if order can be shipped."""
        if order.status != Order.Status.CONFIRMED:
            return False, "Only confirmed orders can be shipped"
        if not order.items.exists():
            return False, "Cannot ship empty order"
        if not order.shipping_address:
            return False, "Shipping address is required"
        return True, ""

    def validate_can_modify(self, order: Order) -> tuple[bool, str]:
        """Check if order can be modified."""
        if order.status != Order.Status.PENDING:
            return False, f"Cannot modify order in {order.status} status"
        return True, ""
```

## Static Methods for Queries

### Why Static Methods?

1. **No state needed**: Queries don't modify the service
2. **Explicit**: Clear that no instance state is accessed
3. **Testable**: Can be called without instantiation

### Query Optimization Patterns

#### Always Use select_related for ForeignKey

```python
@staticmethod
def get_by_id(order_id: UUID) -> Order | None:
    # BAD: N+1 when accessing order.customer
    return Order.objects.filter(id=order_id).first()

    # GOOD: Joins customer in single query
    return (
        Order.objects
        .select_related('customer')
        .filter(id=order_id)
        .first()
    )
```

#### Use prefetch_related for Reverse Relations

```python
@staticmethod
def get_with_items(order_id: UUID) -> Order | None:
    # BAD: N+1 when iterating order.items.all()
    return Order.objects.filter(id=order_id).first()

    # GOOD: Prefetches items in separate query
    return (
        Order.objects
        .prefetch_related('items__product')
        .filter(id=order_id)
        .first()
    )
```

#### Combine for Complex Relations

```python
@staticmethod
def get_full_order(order_id: UUID) -> Order | None:
    return (
        Order.objects
        .select_related(
            'customer',
            'shipping_address',
            'billing_address',
        )
        .prefetch_related(
            'items__product__category',
            'status_history',
        )
        .filter(id=order_id)
        .first()
    )
```

#### Use Prefetch Objects for Filtering

```python
from django.db.models import Prefetch

@staticmethod
def get_with_active_items(order_id: UUID) -> Order | None:
    return (
        Order.objects
        .prefetch_related(
            Prefetch(
                'items',
                queryset=OrderItem.objects.filter(is_active=True),
                to_attr='active_items',
            )
        )
        .filter(id=order_id)
        .first()
    )
```

### Aggregation Queries

```python
from django.db.models import Count, Sum, Avg, F

@staticmethod
def get_customer_stats(customer_id: UUID) -> dict:
    """Get aggregated statistics for a customer."""
    stats = Order.objects.filter(customer_id=customer_id).aggregate(
        total_orders=Count('id'),
        total_spent=Sum('total'),
        average_order_value=Avg('total'),
    )
    return stats

@staticmethod
def get_top_customers(limit: int = 10) -> QuerySet:
    """Get top customers by total spent."""
    return (
        Customer.objects
        .annotate(
            total_spent=Sum('orders__total'),
            order_count=Count('orders'),
        )
        .filter(total_spent__gt=0)
        .order_by('-total_spent')
        [:limit]
    )
```

### Efficient Existence Checks

```python
@staticmethod
def exists(order_id: UUID) -> bool:
    # BAD: Fetches entire row
    return Order.objects.filter(id=order_id).first() is not None

    # GOOD: Only checks existence
    return Order.objects.filter(id=order_id).exists()

@staticmethod
def has_pending_orders(customer_id: UUID) -> bool:
    return Order.objects.filter(
        customer_id=customer_id,
        status=Order.Status.PENDING,
    ).exists()
```

## Instance Methods for Mutations

### Why Instance Methods?

1. **Explicit dependency**: Requires service instantiation
2. **Stateful operations**: May depend on injected dependencies
3. **Transaction awareness**: Often wrapped in transactions

### Mutation Patterns

#### Simple Create

```python
def create(
    self,
    customer_id: UUID,
    items: list[OrderItemInput],
    notes: str | None = None,
) -> Order:
    """Create a new order with items."""
    order = Order.objects.create(
        customer_id=customer_id,
        notes=notes or '',
        status=Order.Status.PENDING,
    )

    # Create related items
    for item in items:
        OrderItem.objects.create(
            order=order,
            product_id=item.product_id,
            quantity=item.quantity,
            unit_price=item.unit_price,
        )

    return order
```

#### Bulk Create for Performance

```python
def create_with_items(
    self,
    customer_id: UUID,
    items: list[OrderItemInput],
) -> Order:
    """Create order with bulk item creation."""
    order = Order.objects.create(customer_id=customer_id)

    # Bulk create items
    order_items = [
        OrderItem(
            order=order,
            product_id=item.product_id,
            quantity=item.quantity,
            unit_price=item.unit_price,
        )
        for item in items
    ]
    OrderItem.objects.bulk_create(order_items)

    return order
```

#### Partial Update with update_fields

```python
def update_status(self, order: Order, new_status: Order.Status) -> Order:
    """Update only the status field."""
    order.status = new_status
    order.save(update_fields=['status', 'updated_at'])
    return order

def update_shipping(
    self,
    order: Order,
    tracking_number: str,
    carrier: str,
) -> Order:
    """Update shipping information."""
    order.tracking_number = tracking_number
    order.carrier = carrier
    order.shipped_at = timezone.now()
    order.save(update_fields=['tracking_number', 'carrier', 'shipped_at', 'updated_at'])
    return order
```

#### Atomic Operations with select_for_update

```python
from django.db import transaction

def reserve_stock(self, product_id: UUID, quantity: int) -> bool:
    """Reserve stock with row locking to prevent overselling."""
    with transaction.atomic():
        product = (
            Product.objects
            .select_for_update()
            .filter(id=product_id)
            .first()
        )
        if not product:
            return False
        if product.stock < quantity:
            return False

        product.stock = F('stock') - quantity
        product.save(update_fields=['stock'])
        return True
```

## Validation Pattern

### Return `tuple[bool, str]`

**Principle**: Never raise exceptions for business logic errors. Return a tuple with:
- `bool`: Whether the operation can proceed
- `str`: Error message (empty string if valid)

```python
def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
    """
    Validate if order can be cancelled.

    Returns:
        tuple[bool, str]: (is_valid, error_message)
    """
    if order.status == Order.Status.SHIPPED:
        return False, "Cannot cancel shipped orders"
    if order.status == Order.Status.CANCELLED:
        return False, "Order is already cancelled"
    return True, ""
```

### Usage in Use Cases

```python
class CancelOrderUseCase:
    def execute(self, input_dto: CancelOrderInput) -> CancelOrderOutput:
        order = self._order_service.get_by_id(input_dto.order_id)
        if not order:
            return CancelOrderOutput(success=False, error="Order not found")

        # Use validation method
        is_valid, error = self._order_service.validate_can_cancel(order)
        if not is_valid:
            return CancelOrderOutput(success=False, error=error)

        # Proceed with cancellation
        self._order_service.cancel(order, input_dto.reason)
        return CancelOrderOutput(success=True)
```

### Complex Validations

```python
def validate_can_checkout(
    self,
    order: Order,
    payment_method_id: UUID,
) -> tuple[bool, str]:
    """Validate order can proceed to checkout."""
    # Check order status
    if order.status != Order.Status.PENDING:
        return False, f"Cannot checkout order in {order.status} status"

    # Check items exist
    if not order.items.exists():
        return False, "Cannot checkout empty order"

    # Check all items in stock
    for item in order.items.select_related('product'):
        if item.product.stock < item.quantity:
            return False, f"Insufficient stock for {item.product.name}"

    # Check payment method
    payment_method = PaymentMethod.objects.filter(
        id=payment_method_id,
        customer=order.customer,
        is_active=True,
    ).first()
    if not payment_method:
        return False, "Invalid payment method"

    return True, ""
```

## Cross-Service Communication

### Injecting Other Services

```python
class OrderService:
    def __init__(
        self,
        inventory_service: 'InventoryService | None' = None,
        notification_service: 'NotificationService | None' = None,
    ):
        self._inventory_service = inventory_service or InventoryService()
        self._notification_service = notification_service

    def process_order(self, order: Order) -> tuple[bool, str]:
        """Process order with inventory reservation."""
        # Use injected service
        for item in order.items.all():
            reserved = self._inventory_service.reserve(
                item.product_id,
                item.quantity,
            )
            if not reserved:
                return False, f"Could not reserve {item.product.name}"

        # Optional notification
        if self._notification_service:
            self._notification_service.send_order_confirmation(order.id)

        return True, ""
```

### Static Cross-Service Calls

For simple cases, call static methods directly:

```python
class OrderService:
    @staticmethod
    def get_with_customer_info(order_id: UUID) -> Order | None:
        order = OrderService.get_by_id(order_id)
        if order:
            # Direct call to another service's static method
            order.customer_tier = CustomerService.get_tier(order.customer_id)
        return order
```

## Service Organization

### One Service Per Entity (Recommended)

```
apps/orders/services/
├── __init__.py
├── order_service.py       # OrderService
├── order_item_service.py  # OrderItemService (if complex enough)
└── shipping_service.py    # ShippingService
```

### Grouped Services (For Related Entities)

```python
# apps/orders/services/order_service.py

class OrderService:
    """Main order operations."""
    ...


class OrderItemService:
    """Order item specific operations."""
    ...


class OrderHistoryService:
    """Order status history operations."""
    ...
```

### Module Exports

```python
# apps/orders/services/__init__.py
from apps.orders.services.order_service import OrderService
from apps.orders.services.shipping_service import ShippingService

__all__ = ['OrderService', 'ShippingService']
```

## Anti-Patterns to Avoid

### Mixed Query/Mutation Methods

```python
# BAD: Method does both read and write
def get_or_create_order(self, customer_id: UUID) -> Order:
    order = Order.objects.filter(
        customer_id=customer_id,
        status=Order.Status.PENDING,
    ).first()
    if not order:
        order = Order.objects.create(customer_id=customer_id)
    return order

# GOOD: Separate methods
@staticmethod
def get_pending_order(customer_id: UUID) -> Order | None:
    return Order.objects.filter(...).first()

def create_order(self, customer_id: UUID) -> Order:
    return Order.objects.create(...)
```

### Validation Raising Exceptions

```python
# BAD: Raises exception for business logic
def validate_can_cancel(self, order: Order) -> None:
    if order.status == Order.Status.SHIPPED:
        raise ValidationError("Cannot cancel shipped orders")

# GOOD: Returns tuple
def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
    if order.status == Order.Status.SHIPPED:
        return False, "Cannot cancel shipped orders"
    return True, ""
```

### HTTP Awareness in Services

```python
# BAD: Service knows about HTTP
from rest_framework.exceptions import NotFound

def get_order(self, order_id: UUID) -> Order:
    order = Order.objects.filter(id=order_id).first()
    if not order:
        raise NotFound("Order not found")  # HTTP exception!
    return order

# GOOD: Return None, let caller handle
@staticmethod
def get_by_id(order_id: UUID) -> Order | None:
    return Order.objects.filter(id=order_id).first()
```

### Missing Query Optimization

```python
# BAD: N+1 queries when used in loop
@staticmethod
def get_all() -> QuerySet[Order]:
    return Order.objects.all()

# GOOD: Prefetch relations
@staticmethod
def get_all() -> QuerySet[Order]:
    return (
        Order.objects
        .select_related('customer')
        .prefetch_related('items')
        .all()
    )
```
