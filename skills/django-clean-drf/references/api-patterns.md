# API Patterns for Django REST Framework

## Overview

This guide covers essential API patterns for production-quality Django REST Framework APIs: pagination, filtering, searching, ordering, API versioning, and error handling.

## Pagination

### PageNumberPagination (Default)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}
```

**Response format:**
```json
{
    "count": 100,
    "next": "http://api.example.com/orders/?page=2",
    "previous": null,
    "results": [...]
}
```

### Custom Pagination Class

```python
# apps/core/pagination.py
from rest_framework.pagination import PageNumberPagination

class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100

    def get_paginated_response(self, data):
        return Response({
            'meta': {
                'count': self.page.paginator.count,
                'page': self.page.number,
                'page_size': self.get_page_size(self.request),
                'total_pages': self.page.paginator.num_pages,
            },
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link(),
            },
            'results': data,
        })
```

### LimitOffsetPagination

Better for cursor-based pagination or infinite scroll.

```python
from rest_framework.pagination import LimitOffsetPagination

class LimitOffsetPagination(LimitOffsetPagination):
    default_limit = 20
    max_limit = 100
```

**Usage:** `GET /api/orders/?limit=20&offset=40`

### CursorPagination

Best for real-time data and large datasets. Uses opaque cursors instead of page numbers.

```python
from rest_framework.pagination import CursorPagination

class OrderCursorPagination(CursorPagination):
    page_size = 20
    ordering = '-created_at'  # Required: must be unique/sequential
    cursor_query_param = 'cursor'
```

**Benefits:**
- Consistent results even with new inserts
- No count query (faster for large tables)
- Prevents skipping/duplicating during pagination

### Per-View Pagination

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderReadSerializer
    pagination_class = StandardPagination  # Override default
```

### Disable Pagination

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    pagination_class = None  # No pagination
```

## Filtering

### django-filter Integration

```bash
pip install django-filter
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_filters',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
    ],
}
```

### Basic FilterSet

```python
# apps/orders/filters.py
import django_filters
from apps.orders.models import Order

class OrderFilter(django_filters.FilterSet):
    status = django_filters.ChoiceFilter(choices=Order.Status.choices)
    created_after = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='gte',
    )
    created_before = django_filters.DateTimeFilter(
        field_name='created_at',
        lookup_expr='lte',
    )
    min_total = django_filters.NumberFilter(
        field_name='total',
        lookup_expr='gte',
    )
    max_total = django_filters.NumberFilter(
        field_name='total',
        lookup_expr='lte',
    )

    class Meta:
        model = Order
        fields = ['status', 'customer']
```

### Using FilterSet in ViewSet

```python
from django_filters.rest_framework import DjangoFilterBackend
from apps.orders.filters import OrderFilter

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.select_related('customer')
    serializer_class = OrderReadSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = OrderFilter
```

**Usage:** `GET /api/orders/?status=pending&min_total=100`

### Advanced Filters

```python
class OrderFilter(django_filters.FilterSet):
    # Multiple choice filter
    status = django_filters.MultipleChoiceFilter(
        choices=Order.Status.choices,
    )

    # Boolean filter
    has_items = django_filters.BooleanFilter(
        method='filter_has_items',
    )

    # Related field filter
    customer_email = django_filters.CharFilter(
        field_name='customer__email',
        lookup_expr='icontains',
    )

    # Date range filter
    created_at = django_filters.DateFromToRangeFilter()

    class Meta:
        model = Order
        fields = ['status', 'customer']

    def filter_has_items(self, queryset, name, value):
        if value is True:
            return queryset.filter(items__isnull=False).distinct()
        elif value is False:
            return queryset.filter(items__isnull=True)
        return queryset
```

**Usage:**
- `GET /api/orders/?status=pending&status=processing` (multiple status)
- `GET /api/orders/?created_at_after=2024-01-01&created_at_before=2024-12-31`

## Searching

### SearchFilter

```python
from rest_framework.filters import SearchFilter

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.select_related('customer')
    serializer_class = OrderReadSerializer
    filter_backends = [SearchFilter]
    search_fields = [
        'id',
        'customer__name',
        'customer__email',
        'notes',
    ]
```

**Usage:** `GET /api/orders/?search=john`

### Search Field Modifiers

```python
search_fields = [
    '=id',              # Exact match
    '^customer__name',  # Starts with
    '$notes',           # Regex
    '@description',     # Full-text search (PostgreSQL)
    'customer__email',  # icontains (default)
]
```

### Full-Text Search (PostgreSQL)

```python
from django.contrib.postgres.search import SearchVector, SearchQuery, SearchRank

class ProductService:
    @staticmethod
    def search(query: str) -> QuerySet[Product]:
        search_vector = SearchVector('name', weight='A') + SearchVector('description', weight='B')
        search_query = SearchQuery(query)

        return (
            Product.objects
            .annotate(
                search=search_vector,
                rank=SearchRank(search_vector, search_query),
            )
            .filter(search=search_query)
            .order_by('-rank')
        )
```

## Ordering

### OrderingFilter

```python
from rest_framework.filters import OrderingFilter

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.all()
    serializer_class = OrderReadSerializer
    filter_backends = [OrderingFilter]
    ordering_fields = ['created_at', 'total', 'status']
    ordering = ['-created_at']  # Default ordering
```

**Usage:**
- `GET /api/orders/?ordering=created_at` (ascending)
- `GET /api/orders/?ordering=-created_at` (descending)
- `GET /api/orders/?ordering=-total,created_at` (multiple)

### Restrict Ordering Fields

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    ordering_fields = ['created_at', 'total']  # Only these allowed
    # ordering_fields = '__all__'  # Allow all (not recommended)
```

## Combining Filter Backends

```python
from rest_framework.filters import SearchFilter, OrderingFilter
from django_filters.rest_framework import DjangoFilterBackend

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.select_related('customer')
    serializer_class = OrderReadSerializer

    filter_backends = [
        DjangoFilterBackend,  # ?status=pending
        SearchFilter,          # ?search=john
        OrderingFilter,        # ?ordering=-created_at
    ]

    filterset_class = OrderFilter
    search_fields = ['customer__name', 'customer__email']
    ordering_fields = ['created_at', 'total']
    ordering = ['-created_at']
```

**Usage:** `GET /api/orders/?status=pending&search=john&ordering=-total`

## API Versioning

### URL Path Versioning (Recommended)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.URLPathVersioning',
    'DEFAULT_VERSION': 'v1',
    'ALLOWED_VERSIONS': ['v1', 'v2'],
    'VERSION_PARAM': 'version',
}
```

```python
# urls.py
from django.urls import path, include

urlpatterns = [
    path('api/<version>/', include('apps.api.urls')),
]

# apps/api/urls.py
urlpatterns = [
    path('orders/', OrderViewSet.as_view({'get': 'list'})),
]
```

**Usage:** `GET /api/v1/orders/`

### Version-Specific Serializers

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = Order.objects.all()

    def get_serializer_class(self):
        if self.request.version == 'v2':
            return OrderV2ReadSerializer
        return OrderV1ReadSerializer
```

### Version-Specific Logic

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    def get_queryset(self):
        queryset = Order.objects.all()

        if self.request.version == 'v2':
            # V2 includes soft-deleted orders
            return queryset
        # V1 excludes soft-deleted
        return queryset.filter(deleted_at__isnull=True)
```

### Namespace Versioning

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning',
}

# urls.py
urlpatterns = [
    path('api/v1/', include('apps.api.v1.urls', namespace='v1')),
    path('api/v2/', include('apps.api.v2.urls', namespace='v2')),
]
```

## Error Handling

### Standard Error Response Format

```python
# apps/core/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)

    if response is not None:
        response.data = {
            'error': {
                'code': response.status_code,
                'message': get_error_message(exc),
                'details': response.data if isinstance(response.data, dict) else {'detail': response.data},
            }
        }

    return response

def get_error_message(exc):
    if hasattr(exc, 'detail'):
        if isinstance(exc.detail, str):
            return exc.detail
        if isinstance(exc.detail, list):
            return exc.detail[0] if exc.detail else 'An error occurred'
    return str(exc)
```

```python
# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'apps.core.exceptions.custom_exception_handler',
}
```

**Response format:**
```json
{
    "error": {
        "code": 400,
        "message": "Invalid input",
        "details": {
            "email": ["Enter a valid email address."],
            "quantity": ["Ensure this value is greater than 0."]
        }
    }
}
```

### Business Logic Errors from Use Cases

```python
class OrderViewSet(viewsets.ViewSet):
    def create(self, request):
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase()
        output = use_case.execute(CreateOrderInput(**serializer.validated_data))

        if not output.success:
            # Map business errors to HTTP responses
            return Response(
                {'error': {'code': 'BUSINESS_ERROR', 'message': output.error}},
                status=status.HTTP_400_BAD_REQUEST,
            )

        order = OrderService.get_by_id(output.order_id)
        return Response(OrderReadSerializer(order).data, status=status.HTTP_201_CREATED)
```

### Error Codes for API Clients

```python
# apps/core/error_codes.py
from enum import StrEnum

class ErrorCode(StrEnum):
    VALIDATION_ERROR = 'VALIDATION_ERROR'
    NOT_FOUND = 'NOT_FOUND'
    PERMISSION_DENIED = 'PERMISSION_DENIED'
    INSUFFICIENT_STOCK = 'INSUFFICIENT_STOCK'
    INVALID_STATUS_TRANSITION = 'INVALID_STATUS_TRANSITION'
    PAYMENT_FAILED = 'PAYMENT_FAILED'
```

```python
@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str = ""
    error_code: str = ""  # For API clients to handle programmatically
```

## Throttling

### Default Throttling

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
    },
}
```

### Custom Throttle Classes

```python
# apps/core/throttling.py
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',
        'sustained': '1000/day',
    },
}
```

### Per-View Throttling

```python
from rest_framework.throttling import ScopedRateThrottle

class OrderViewSet(viewsets.ModelViewSet):
    throttle_classes = [ScopedRateThrottle]
    throttle_scope = 'orders'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'orders': '100/hour',
    },
}
```

## Response Caching

### ETags and Conditional Requests

```python
from rest_framework.decorators import api_view
from django.views.decorators.http import etag
import hashlib

def get_order_etag(request, order_id):
    order = Order.objects.filter(id=order_id).values('updated_at').first()
    if order:
        return hashlib.md5(str(order['updated_at']).encode()).hexdigest()
    return None

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    @etag(get_order_etag)
    def retrieve(self, request, *args, **kwargs):
        return super().retrieve(request, *args, **kwargs)
```

### Cache Decorators

```python
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    @method_decorator(cache_page(60 * 15))  # Cache 15 minutes
    def list(self, request, *args, **kwargs):
        return super().list(request, *args, **kwargs)
```

### Per-User Caching

```python
from django.core.cache import cache

class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    def list(self, request):
        cache_key = f'orders_list_{request.user.id}'
        cached = cache.get(cache_key)

        if cached:
            return Response(cached)

        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        cache.set(cache_key, serializer.data, timeout=300)

        return Response(serializer.data)
```

## Content Negotiation

### Multiple Response Formats

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # HTML for browser
    ],
}
```

### CSV Export

```python
# apps/core/renderers.py
import csv
from io import StringIO
from rest_framework.renderers import BaseRenderer

class CSVRenderer(BaseRenderer):
    media_type = 'text/csv'
    format = 'csv'

    def render(self, data, accepted_media_type=None, renderer_context=None):
        if not data:
            return ''

        output = StringIO()
        if isinstance(data, list) and data:
            writer = csv.DictWriter(output, fieldnames=data[0].keys())
            writer.writeheader()
            writer.writerows(data)
        return output.getvalue()
```

```python
class OrderViewSet(viewsets.ReadOnlyModelViewSet):
    renderer_classes = [JSONRenderer, CSVRenderer]
```

**Usage:** `GET /api/orders/?format=csv`

## API Best Practices Checklist

### URL Design
- [ ] Use plural nouns for resources (`/orders/` not `/order/`)
- [ ] Use UUID for public IDs
- [ ] Nest resources logically (`/customers/{id}/orders/`)
- [ ] Version your API (`/api/v1/`)

### Query Parameters
- [ ] Pagination on all list endpoints
- [ ] Filtering for common use cases
- [ ] Ordering option with default
- [ ] Search for text fields

### Responses
- [ ] Consistent error format
- [ ] Meaningful error codes
- [ ] Proper HTTP status codes
- [ ] Include hypermedia links (optional)

### Performance
- [ ] Optimize querysets (select/prefetch)
- [ ] Rate limiting configured
- [ ] Caching for read-heavy endpoints
- [ ] Pagination limits enforced
