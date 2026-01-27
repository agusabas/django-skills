# Query Optimization for Django APIs

## Overview

Query optimization is critical for API performance. This guide covers identifying and fixing N+1 queries, using select_related/prefetch_related effectively, database indexing, and monitoring query performance.

## The N+1 Query Problem

### What is N+1?

N+1 occurs when code executes 1 query to fetch a list, then N additional queries to fetch related data for each item.

```python
# BAD: N+1 Query - 1 + N queries
def get_orders():
    orders = Order.objects.all()  # 1 query
    for order in orders:
        print(order.customer.name)  # N queries (1 per order)
        for item in order.items.all():  # N more queries
            print(item.product.name)  # N * M queries!
```

### Detecting N+1 Queries

**1. Django Debug Toolbar (Development)**
```python
# settings.py
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']
    INTERNAL_IPS = ['127.0.0.1']
```

**2. Query Logging**
```python
# settings.py
LOGGING = {
    'version': 1,
    'handlers': {
        'console': {'class': 'logging.StreamHandler'},
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
        },
    },
}
```

**3. assertNumQueries in Tests**
```python
def test_list_orders_query_count(self):
    OrderFactory.create_batch(10)

    with self.assertNumQueries(2):  # 1 for orders, 1 for prefetch
        response = self.client.get('/api/orders/')
        # Force evaluation
        list(response.data)
```

## select_related: ForeignKey and OneToOne

Use `select_related` for ForeignKey and OneToOneField. It performs a SQL JOIN.

```python
# BAD: N+1 queries
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)  # Extra query per order

# GOOD: Single query with JOIN
orders = Order.objects.select_related('customer')
for order in orders:
    print(order.customer.name)  # No extra query
```

### Chaining select_related

```python
# Multiple ForeignKeys
Order.objects.select_related('customer', 'shipping_address')

# Nested ForeignKeys (follow the relationship)
OrderItem.objects.select_related('order__customer', 'product__category')
```

### In Services

```python
class OrderService:
    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        return (
            Order.objects
            .select_related('customer', 'shipping_address')
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def list_for_customer(customer_id: UUID) -> QuerySet[Order]:
        return (
            Order.objects
            .select_related('customer')
            .filter(customer_id=customer_id)
            .order_by('-created_at')
        )
```

## prefetch_related: Reverse FK and ManyToMany

Use `prefetch_related` for reverse ForeignKey (related_name) and ManyToManyField. It performs separate queries and joins in Python.

```python
# BAD: N+1 queries
orders = Order.objects.all()
for order in orders:
    for item in order.items.all():  # Extra query per order
        print(item.name)

# GOOD: 2 queries total
orders = Order.objects.prefetch_related('items')
for order in orders:
    for item in order.items.all():  # No extra query
        print(item.name)
```

### Chaining prefetch_related

```python
# Multiple reverse relations
Order.objects.prefetch_related('items', 'status_history', 'notes')

# Nested prefetch
Order.objects.prefetch_related('items__product__category')
```

### Combining select_related and prefetch_related

```python
class OrderService:
    @staticmethod
    def get_full_order(order_id: UUID) -> Order | None:
        return (
            Order.objects
            # JOINs for ForeignKeys
            .select_related(
                'customer',
                'shipping_address',
                'billing_address',
            )
            # Separate queries for reverse relations
            .prefetch_related(
                'items__product',
                'status_history',
            )
            .filter(id=order_id)
            .first()
        )
```

## Prefetch Objects for Custom Queries

Use `Prefetch` objects when you need to filter, order, or optimize prefetched querysets.

```python
from django.db.models import Prefetch

class OrderService:
    @staticmethod
    def get_with_active_items(order_id: UUID) -> Order | None:
        return (
            Order.objects
            .prefetch_related(
                Prefetch(
                    'items',
                    queryset=OrderItem.objects.filter(
                        is_active=True
                    ).select_related('product'),
                    to_attr='active_items',  # Access as order.active_items
                )
            )
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def list_with_recent_items(customer_id: UUID) -> QuerySet[Order]:
        recent_items = OrderItem.objects.filter(
            created_at__gte=timezone.now() - timedelta(days=30)
        ).order_by('-created_at')[:5]

        return (
            Order.objects
            .filter(customer_id=customer_id)
            .prefetch_related(
                Prefetch('items', queryset=recent_items, to_attr='recent_items')
            )
        )
```

### Prefetch with Annotations

```python
from django.db.models import Count, Sum

class CustomerService:
    @staticmethod
    def list_with_order_stats() -> QuerySet[Customer]:
        orders_with_totals = Order.objects.annotate(
            item_count=Count('items'),
            total_value=Sum('items__price'),
        )

        return (
            Customer.objects
            .prefetch_related(
                Prefetch('orders', queryset=orders_with_totals)
            )
        )
```

## Serializer Optimization

### The Problem: N+1 in Serializers

```python
# BAD: N+1 in nested serializer
class OrderSerializer(serializers.ModelSerializer):
    customer = CustomerSerializer()  # Triggers query per order
    items = OrderItemSerializer(many=True)  # Triggers query per order

    class Meta:
        model = Order
        fields = ['id', 'customer', 'items', 'total']
```

### Solution: Optimize the QuerySet in View

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    serializer_class = OrderReadSerializer

    def get_queryset(self):
        return (
            Order.objects
            .select_related('customer')
            .prefetch_related(
                Prefetch(
                    'items',
                    queryset=OrderItem.objects.select_related('product')
                )
            )
        )
```

### Solution: SerializerMethodField with Prefetch

```python
class OrderReadSerializer(serializers.ModelSerializer):
    customer_name = serializers.CharField(source='customer.name', read_only=True)
    item_count = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ['id', 'customer_name', 'item_count', 'total']

    def get_item_count(self, obj):
        # Use prefetched data if available
        if hasattr(obj, '_prefetched_objects_cache') and 'items' in obj._prefetched_objects_cache:
            return len(obj.items.all())
        # Fallback to count query
        return obj.items.count()
```

## Database Indexing

### When to Add Indexes

Add indexes for fields used in:
- `filter()` conditions
- `order_by()` clauses
- `JOIN` conditions (ForeignKey already indexed)
- Unique constraints

### Single Field Indexes

```python
class Order(models.Model):
    status = models.CharField(max_length=20, db_index=True)  # Simple index
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    email = models.EmailField(unique=True)  # Unique creates index
```

### Composite Indexes

```python
class Order(models.Model):
    customer = models.ForeignKey(Customer, on_delete=models.PROTECT)
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            # For queries filtering by customer + status
            models.Index(fields=['customer', 'status']),
            # For queries ordering by created_at descending
            models.Index(fields=['-created_at']),
            # For queries filtering status and ordering by date
            models.Index(fields=['status', '-created_at']),
        ]
```

### Partial Indexes (PostgreSQL)

```python
from django.db.models import Q

class Order(models.Model):
    class Meta:
        indexes = [
            # Index only pending orders (smaller, faster)
            models.Index(
                fields=['created_at'],
                name='pending_orders_idx',
                condition=Q(status='pending'),
            ),
        ]
```

### Covering Indexes (PostgreSQL 11+)

```python
class Order(models.Model):
    class Meta:
        indexes = [
            # Include columns to avoid table lookup
            models.Index(
                fields=['customer_id', 'status'],
                include=['total', 'created_at'],
                name='order_customer_status_covering',
            ),
        ]
```

## Query Optimization Patterns

### only() and defer()

Load only needed fields to reduce memory and transfer time.

```python
# Load only specific fields
Order.objects.only('id', 'status', 'total')

# Exclude heavy fields
Order.objects.defer('description', 'notes', 'metadata')
```

**Use in Services:**
```python
class OrderService:
    @staticmethod
    def list_summary(customer_id: UUID) -> QuerySet[Order]:
        """Return orders with only summary fields."""
        return (
            Order.objects
            .filter(customer_id=customer_id)
            .only('id', 'status', 'total', 'created_at')
            .order_by('-created_at')
        )
```

### values() and values_list()

When you don't need model instances.

```python
# Dictionary results
Order.objects.filter(status='pending').values('id', 'total')
# [{'id': 1, 'total': Decimal('99.99')}, ...]

# Tuple results
Order.objects.filter(status='pending').values_list('id', 'total')
# [(1, Decimal('99.99')), ...]

# Flat list (single field)
Order.objects.filter(status='pending').values_list('id', flat=True)
# [1, 2, 3, ...]
```

### exists() vs count() vs len()

```python
# Check existence (LIMIT 1)
if Order.objects.filter(customer_id=customer_id).exists():
    ...

# Count in database (SELECT COUNT(*))
count = Order.objects.filter(status='pending').count()

# AVOID: Fetches all rows then counts in Python
count = len(Order.objects.filter(status='pending'))
```

### Bulk Operations

```python
class OrderService:
    def create_items_bulk(self, order: Order, items: list[OrderItemInput]) -> list[OrderItem]:
        """Create items in single query."""
        order_items = [
            OrderItem(
                order=order,
                product_id=item.product_id,
                quantity=item.quantity,
                unit_price=item.unit_price,
            )
            for item in items
        ]
        return OrderItem.objects.bulk_create(order_items)

    def update_status_bulk(self, order_ids: list[UUID], new_status: str) -> int:
        """Update multiple orders in single query."""
        return Order.objects.filter(id__in=order_ids).update(
            status=new_status,
            updated_at=timezone.now(),
        )
```

### F() Expressions for Atomic Updates

```python
from django.db.models import F

class ProductService:
    def decrement_stock(self, product_id: UUID, quantity: int) -> int:
        """Atomic stock decrement without race conditions."""
        return (
            Product.objects
            .filter(id=product_id, stock__gte=quantity)
            .update(stock=F('stock') - quantity)
        )

    def increment_view_count(self, product_id: UUID) -> None:
        """Atomic counter increment."""
        Product.objects.filter(id=product_id).update(
            view_count=F('view_count') + 1
        )
```

## Aggregations and Annotations

### Common Aggregations

```python
from django.db.models import Count, Sum, Avg, Max, Min

class OrderService:
    @staticmethod
    def get_customer_stats(customer_id: UUID) -> dict:
        return Order.objects.filter(customer_id=customer_id).aggregate(
            total_orders=Count('id'),
            total_spent=Sum('total'),
            average_order=Avg('total'),
            largest_order=Max('total'),
            first_order_date=Min('created_at'),
        )
```

### Annotations for Per-Row Calculations

```python
class CustomerService:
    @staticmethod
    def list_with_order_counts() -> QuerySet[Customer]:
        return (
            Customer.objects
            .annotate(
                order_count=Count('orders'),
                total_spent=Sum('orders__total'),
            )
            .filter(order_count__gt=0)
            .order_by('-total_spent')
        )
```

### Conditional Aggregations

```python
from django.db.models import Case, When, Value, IntegerField

class OrderService:
    @staticmethod
    def get_status_counts(customer_id: UUID) -> dict:
        return (
            Order.objects
            .filter(customer_id=customer_id)
            .aggregate(
                pending=Count(Case(When(status='pending', then=1))),
                shipped=Count(Case(When(status='shipped', then=1))),
                delivered=Count(Case(When(status='delivered', then=1))),
            )
        )
```

## Subqueries

```python
from django.db.models import OuterRef, Subquery

class CustomerService:
    @staticmethod
    def list_with_latest_order() -> QuerySet[Customer]:
        latest_order = (
            Order.objects
            .filter(customer=OuterRef('pk'))
            .order_by('-created_at')
            .values('created_at')[:1]
        )

        return Customer.objects.annotate(
            latest_order_date=Subquery(latest_order)
        )
```

## Query Performance Checklist

### Before Writing a Query

- [ ] What fields will be accessed?
- [ ] What related objects will be accessed?
- [ ] How many rows expected?
- [ ] Is this a list view or detail view?

### Service Method Checklist

- [ ] `select_related` for all accessed ForeignKeys
- [ ] `prefetch_related` for all accessed reverse relations
- [ ] `only()`/`defer()` for large text/binary fields
- [ ] Appropriate indexes exist for filter/order fields
- [ ] Using `exists()` instead of `count() > 0`
- [ ] Using `F()` for atomic updates

### Testing Queries

```python
class OrderServiceTests(TestCase):
    def test_get_by_id_query_count(self):
        order = OrderFactory.create()
        OrderItemFactory.create_batch(5, order=order)

        with self.assertNumQueries(2):  # 1 order + 1 prefetch items
            result = OrderService.get_by_id(order.id)
            # Force evaluation of prefetched data
            list(result.items.all())

    def test_list_for_customer_query_count(self):
        customer = CustomerFactory.create()
        OrderFactory.create_batch(10, customer=customer)

        with self.assertNumQueries(1):
            list(OrderService.list_for_customer(customer.id))
```

## Common Anti-Patterns

### 1. Querying in Loops

```python
# BAD
for product_id in product_ids:
    product = Product.objects.get(id=product_id)
    process(product)

# GOOD
products = Product.objects.filter(id__in=product_ids)
for product in products:
    process(product)
```

### 2. Unnecessary Ordering

```python
# BAD: Orders then counts
Order.objects.filter(status='pending').order_by('-created_at').count()

# GOOD: Skip ordering for count
Order.objects.filter(status='pending').count()
```

### 3. Not Using Iterator for Large QuerySets

```python
# BAD: Loads all into memory
for order in Order.objects.all():
    process(order)

# GOOD: Iterates in chunks
for order in Order.objects.all().iterator(chunk_size=1000):
    process(order)
```

### 4. Fetching Unused Fields

```python
# BAD: Fetches all fields including large ones
orders = Order.objects.all()

# GOOD: Fetch only needed fields
orders = Order.objects.only('id', 'status', 'total')
```
