# Django Anti-Patterns to Avoid

## 1. Fat Views

```python
# BAD: Business logic in view
class OrderView(APIView):
    def post(self, request):
        # 100 lines of validation, processing, notifications...
        pass

# GOOD: Thin view delegating to service
class OrderView(APIView):
    def post(self, request):
        service = OrderService()
        order = service.create_order(request.user, request.data)
        return Response(OrderSerializer(order).data)
```

## 2. N+1 Query Problem

```python
# BAD: N+1 queries
def get_orders():
    orders = Order.objects.all()
    for order in orders:
        print(order.user.name)  # Query per order!
        for item in order.items.all():  # Another N queries!
            print(item.product.name)

# GOOD: Optimized with select_related and prefetch_related
def get_orders():
    orders = Order.objects.select_related("user").prefetch_related(
        Prefetch("items", queryset=OrderItem.objects.select_related("product"))
    )
    for order in orders:
        print(order.user.name)  # No extra query
        for item in order.items.all():  # No extra query
            print(item.product.name)
```

## 3. Using len() on QuerySets

```python
# BAD: Fetches all records then counts
count = len(Order.objects.filter(status="pending"))

# GOOD: Database-level count
count = Order.objects.filter(status="pending").count()
```

## 4. Not Using F() Expressions

```python
# BAD: Race condition, fetches then updates
product = Product.objects.get(pk=1)
product.view_count = product.view_count + 1
product.save()

# GOOD: Atomic update at database level
from django.db.models import F
Product.objects.filter(pk=1).update(view_count=F("view_count") + 1)
```

## 5. Improper Null Usage on String Fields

```python
# BAD: Both NULL and '' for empty values
class Article(models.Model):
    subtitle = models.CharField(max_length=200, null=True, blank=True)

# GOOD: Only '' for empty strings
class Article(models.Model):
    subtitle = models.CharField(max_length=200, blank=True, default="")
```

## 6. Business Logic in Templates

```python
# BAD: Logic in template
{% if order.status == "pending" and order.created_at < threshold %}
    <span class="overdue">Overdue</span>
{% endif %}

# GOOD: Property on model
class Order(models.Model):
    @property
    def is_overdue(self) -> bool:
        if self.status != "pending":
            return False
        threshold = timezone.now() - timedelta(days=7)
        return self.created_at < threshold

# Template: {% if order.is_overdue %}<span class="overdue">Overdue</span>{% endif %}
```

## 7. Hardcoded Configuration

```python
# BAD: Hardcoded values
STRIPE_KEY = "sk_live_abc123"

# GOOD: Environment variables
import environ
env = environ.Env()
STRIPE_KEY = env("STRIPE_KEY")
```

## 8. Not Using Transactions

```python
# BAD: Partial failure leaves inconsistent state
def transfer_funds(from_account, to_account, amount):
    from_account.balance -= amount
    from_account.save()
    # If this fails, money disappears!
    to_account.balance += amount
    to_account.save()

# GOOD: Atomic transaction
from django.db import transaction

def transfer_funds(from_account, to_account, amount):
    with transaction.atomic():
        from_account.balance = F("balance") - amount
        from_account.save(update_fields=["balance"])
        to_account.balance = F("balance") + amount
        to_account.save(update_fields=["balance"])
```

## 9. Mutable Default Arguments

```python
# BAD: Mutable default
def create_order(items=[]):  # Same list reused!
    items.append(default_item)
    return Order(items=items)

# GOOD: None default with initialization
def create_order(items: list | None = None):
    if items is None:
        items = []
    items.append(default_item)
    return Order(items=items)
```

## 10. Bare Exception Handling

```python
# BAD: Swallows all errors
try:
    process_payment()
except:
    pass

# GOOD: Specific exceptions
try:
    process_payment()
except PaymentDeclinedError as e:
    logger.warning(f"Payment declined: {e}")
    raise
except NetworkTimeoutError:
    retry_payment()
```
