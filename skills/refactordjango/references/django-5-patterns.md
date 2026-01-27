# Django 5+ Patterns and Best Practices

## GeneratedField (Django 5.0+)

Use `GeneratedField` for database-computed columns:

```python
from django.db import models
from django.db.models import F, Value
from django.db.models.functions import Concat, Lower

class Product(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.IntegerField()

    # Stored generated field (Postgres only supports stored)
    full_name = models.GeneratedField(
        expression=Concat(F("first_name"), Value(" "), F("last_name")),
        output_field=models.CharField(max_length=201),
        db_persist=True,  # Required for Postgres
    )

    # Virtual generated field (MySQL, SQLite)
    total_value = models.GeneratedField(
        expression=F("price") * F("quantity"),
        output_field=models.DecimalField(max_digits=12, decimal_places=2),
        db_persist=False,  # Computed on read
    )
```

## Field.db_default (Django 5.0+)

Use `db_default` for database-level default values:

```python
from django.db import models
from django.db.models.functions import Now, Pi

class Event(models.Model):
    name = models.CharField(max_length=200)
    created_at = models.DateTimeField(db_default=Now())
    pi_value = models.FloatField(db_default=Pi())
    status = models.CharField(max_length=20, db_default=Value("pending"))
```

## Async Views and Decorators (Django 5.0+)

Django 5 supports async decorators on async views:

```python
from django.views.decorators.cache import cache_control, never_cache
from django.views.decorators.csrf import csrf_exempt
from django.http import JsonResponse

@cache_control(max_age=3600)
async def cached_data_view(request):
    data = await fetch_data_async()
    return JsonResponse(data)

@csrf_exempt
async def webhook_handler(request):
    await process_webhook_async(request.body)
    return JsonResponse({"status": "ok"})

# Async signal dispatch
from django.dispatch import Signal

my_signal = Signal()

async def async_handler(sender, **kwargs):
    await do_async_work()

my_signal.connect(async_handler)

# Send asynchronously
await my_signal.asend(sender=MyClass, data=data)
```

## Async ORM Operations (Django 4.1+)

```python
from django.db.models import Prefetch

# Async queries
user = await User.objects.aget(pk=user_id)
users = [user async for user in User.objects.filter(is_active=True)]
count = await User.objects.acount()
exists = await User.objects.filter(email=email).aexists()

# Async prefetch
await sync_to_async(
    lambda: list(Author.objects.prefetch_related('books').all())
)()

# Or use aprefetch_related_objects
from django.db.models import aprefetch_related_objects
await aprefetch_related_objects(authors, 'books')
```

## Form Field Rendering (Django 5.0+)

Use `.as_field_group` for cleaner form rendering:

```python
# In template
{{ form.email.as_field_group }}

# Renders complete field group with label, widget, help_text, errors
# Customizable via templates/django/forms/field.html
```

## Model Best Practices

Follow official Django ordering in models:

```python
from django.db import models
from django.urls import reverse

class Article(models.Model):
    # 1. Choices
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PUBLISHED = "published", "Published"

    # 2. Database fields
    title = models.CharField(max_length=200)
    content = models.TextField()
    status = models.CharField(max_length=20, choices=Status.choices, default=Status.DRAFT)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # 3. Custom manager attributes
    objects = models.Manager()
    published = PublishedManager()

    # 4. Meta
    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["status", "created_at"]),
        ]

    # 5. __str__
    def __str__(self) -> str:
        return self.title

    # 6. save
    def save(self, *args, **kwargs):
        # Custom save logic
        super().save(*args, **kwargs)

    # 7. get_absolute_url
    def get_absolute_url(self) -> str:
        return reverse("article_detail", kwargs={"pk": self.pk})

    # 8. Custom methods
    def publish(self) -> None:
        self.status = self.Status.PUBLISHED
        self.save(update_fields=["status"])
```

## Custom Managers and QuerySets

```python
class PublishedManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="published")

class ArticleQuerySet(models.QuerySet):
    def published(self):
        return self.filter(status="published")

    def by_author(self, author):
        return self.filter(author=author)

    def recent(self, days=7):
        cutoff = timezone.now() - timedelta(days=days)
        return self.filter(created_at__gte=cutoff)

class Article(models.Model):
    # Use both custom manager and queryset
    objects = ArticleQuerySet.as_manager()
```

## Service Layer Pattern

Keep views thin by extracting business logic to services:

```python
# services/order_service.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass
class OrderService:
    """Service for order-related business logic."""

    def create_order(self, user: User, items: list[dict]) -> Order:
        """Create an order with validation and side effects."""
        self._validate_items(items)
        order = self._create_order_record(user, items)
        self._send_confirmation(order)
        return order

    def _validate_items(self, items: list[dict]) -> None:
        if not items:
            raise ValidationError("Order must have at least one item")
        for item in items:
            if item["quantity"] <= 0:
                raise ValidationError("Quantity must be positive")

    def _create_order_record(self, user: User, items: list[dict]) -> Order:
        with transaction.atomic():
            order = Order.objects.create(user=user)
            for item in items:
                OrderItem.objects.create(order=order, **item)
            return order

    def _send_confirmation(self, order: Order) -> None:
        send_order_confirmation.delay(order.id)

# views.py
class CreateOrderView(APIView):
    def post(self, request):
        service = OrderService()
        order = service.create_order(request.user, request.data["items"])
        return Response(OrderSerializer(order).data, status=201)
```

## Signals vs Direct Calls

Use signals for cross-cutting concerns, direct calls for business logic:

```python
# GOOD: Signal for audit logging (cross-cutting concern)
@receiver(post_save, sender=Order)
def log_order_creation(sender, instance, created, **kwargs):
    if created:
        AuditLog.objects.create(
            action="order_created",
            object_id=instance.id,
            data={"total": str(instance.total)}
        )

# GOOD: Signal for cache invalidation
@receiver(post_save, sender=Product)
def invalidate_product_cache(sender, instance, **kwargs):
    cache.delete(f"product:{instance.id}")

# BAD: Signal for business logic (use service instead)
# @receiver(post_save, sender=Order)
# def process_payment(sender, instance, created, **kwargs):
#     # This should be in OrderService, not a signal
#     PaymentService().charge(instance)
```
