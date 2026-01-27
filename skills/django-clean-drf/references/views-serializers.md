# Views and Serializers for Django Clean Architecture

## Overview

In Clean Architecture, views are part of the HTTP layer (outermost layer). They should be thin, handling only:
- Request parsing and validation (via serializers)
- Use case invocation
- Response formatting

**Key Principles:**
- Views delegate business logic to use cases
- Serializers handle I/O validation, not business logic
- Separate Read and Write serializers
- No direct ORM access in views

## Thin View Pattern

### Basic Structure

```python
# apps/orders/views.py
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

from apps.orders.serializers import CreateOrderSerializer, OrderReadSerializer
from apps.orders.use_cases import CreateOrderUseCase, CreateOrderInput
from apps.orders.services import OrderService, InventoryService


class CreateOrderView(APIView):
    """
    Create a new order.

    POST /api/orders/
    """
    permission_classes = [IsAuthenticated]

    def post(self, request: Request) -> Response:
        # 1. Validate request data
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 2. Build input DTO
        input_dto = CreateOrderInput(
            customer_id=request.user.customer.id,
            **serializer.validated_data,
        )

        # 3. Execute use case
        use_case = CreateOrderUseCase(
            order_service=OrderService(),
            inventory_service=InventoryService(),
        )
        output = use_case.execute(input_dto)

        # 4. Return response
        if not output.success:
            return Response(
                {'error': output.error, 'code': output.error_code},
                status=status.HTTP_400_BAD_REQUEST,
            )

        return Response(
            {'order_id': str(output.order_id)},
            status=status.HTTP_201_CREATED,
        )
```

### View Guidelines

1. **< 20 lines of logic** in each method
2. **No ORM queries** directly in views
3. **No business logic** - delegate to use cases
4. **Clear flow**: Validate → Build DTO → Execute → Respond

## ViewSet Pattern

For standard CRUD operations, ViewSets reduce boilerplate:

```python
# apps/orders/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated

from apps.orders.models import Order
from apps.orders.serializers import (
    OrderReadSerializer,
    OrderListSerializer,
    CreateOrderSerializer,
    UpdateOrderSerializer,
)
from apps.orders.use_cases import (
    CreateOrderUseCase,
    UpdateOrderUseCase,
    CancelOrderUseCase,
)
from apps.orders.services import OrderService


class OrderViewSet(viewsets.ModelViewSet):
    """
    Order CRUD operations.

    list: GET /api/orders/
    retrieve: GET /api/orders/{id}/
    create: POST /api/orders/
    update: PUT /api/orders/{id}/
    partial_update: PATCH /api/orders/{id}/
    destroy: DELETE /api/orders/{id}/
    cancel: POST /api/orders/{id}/cancel/
    """
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        """Return orders for current user only."""
        return (
            Order.objects
            .filter(customer=self.request.user.customer)
            .select_related('customer', 'shipping_address')
            .prefetch_related('items__product')
        )

    def get_serializer_class(self):
        """Use different serializers for different actions."""
        if self.action == 'list':
            return OrderListSerializer
        if self.action == 'create':
            return CreateOrderSerializer
        if self.action in ('update', 'partial_update'):
            return UpdateOrderSerializer
        return OrderReadSerializer  # retrieve, etc.

    def create(self, request, *args, **kwargs):
        """Create order via use case."""
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase(order_service=OrderService())
        output = use_case.execute(
            CreateOrderInput(
                customer_id=request.user.customer.id,
                **serializer.validated_data,
            )
        )

        if not output.success:
            return Response({'error': output.error}, status=400)

        # Fetch created order for response
        order = OrderService.get_by_id(output.order_id)
        return Response(
            OrderReadSerializer(order).data,
            status=status.HTTP_201_CREATED,
        )

    @action(detail=True, methods=['post'])
    def cancel(self, request, pk=None):
        """Cancel an order."""
        use_case = CancelOrderUseCase(order_service=OrderService())
        output = use_case.execute(CancelOrderInput(order_id=pk))

        if not output.success:
            return Response({'error': output.error}, status=400)

        return Response({'status': 'cancelled'})
```

## Separate Read and Write Serializers

### Why Separate?

1. **Different concerns**: Input validation vs output formatting
2. **Security**: Control exactly what's accepted and returned
3. **Flexibility**: Different fields for create vs update
4. **Performance**: Read serializers can include computed fields

### Write Serializers (Input)

```python
# apps/orders/serializers.py
from rest_framework import serializers


class OrderItemInputSerializer(serializers.Serializer):
    """Input serializer for order items."""
    product_id = serializers.UUIDField()
    quantity = serializers.IntegerField(min_value=1, max_value=100)


class CreateOrderSerializer(serializers.Serializer):
    """
    Input serializer for creating orders.

    Does NOT use ModelSerializer - we validate input only.
    """
    items = OrderItemInputSerializer(many=True, min_length=1)
    notes = serializers.CharField(
        max_length=500,
        required=False,
        allow_blank=True,
        default='',
    )
    shipping_address_id = serializers.UUIDField(required=False)

    def validate_items(self, value):
        """Validate items list."""
        product_ids = [item['product_id'] for item in value]
        if len(product_ids) != len(set(product_ids)):
            raise serializers.ValidationError(
                "Duplicate products not allowed"
            )
        return value


class UpdateOrderSerializer(serializers.Serializer):
    """Input serializer for updating orders."""
    notes = serializers.CharField(
        max_length=500,
        required=False,
        allow_blank=True,
    )
    shipping_address_id = serializers.UUIDField(required=False)
```

### Read Serializers (Output)

```python
# apps/orders/serializers.py
from rest_framework import serializers
from apps.orders.models import Order, OrderItem


class OrderItemReadSerializer(serializers.ModelSerializer):
    """Output serializer for order items."""
    product_name = serializers.CharField(
        source='product.name',
        read_only=True,
    )
    subtotal = serializers.DecimalField(
        max_digits=12,
        decimal_places=2,
        read_only=True,
    )

    class Meta:
        model = OrderItem
        fields = [
            'id',
            'product_id',
            'product_name',
            'quantity',
            'unit_price',
            'subtotal',
        ]
        read_only_fields = fields


class OrderReadSerializer(serializers.ModelSerializer):
    """Output serializer for order detail."""
    customer_name = serializers.CharField(
        source='customer.name',
        read_only=True,
    )
    items = OrderItemReadSerializer(many=True, read_only=True)
    status_display = serializers.CharField(
        source='get_status_display',
        read_only=True,
    )
    is_cancellable = serializers.BooleanField(read_only=True)
    total = serializers.DecimalField(
        max_digits=12,
        decimal_places=2,
        read_only=True,
    )

    class Meta:
        model = Order
        fields = [
            'id',
            'customer_name',
            'status',
            'status_display',
            'items',
            'total',
            'is_cancellable',
            'notes',
            'tracking_number',
            'created_at',
            'updated_at',
        ]
        read_only_fields = fields


class OrderListSerializer(serializers.ModelSerializer):
    """Output serializer for order list (minimal fields)."""
    items_count = serializers.IntegerField(read_only=True)

    class Meta:
        model = Order
        fields = [
            'id',
            'status',
            'items_count',
            'created_at',
        ]
        read_only_fields = fields
```

## Validation Patterns

### Field-Level Validation

```python
class CreateOrderSerializer(serializers.Serializer):
    quantity = serializers.IntegerField()

    def validate_quantity(self, value):
        """Validate single field."""
        if value <= 0:
            raise serializers.ValidationError("Quantity must be positive")
        if value > 100:
            raise serializers.ValidationError("Maximum 100 items per order")
        return value
```

### Object-Level Validation

```python
class CreateOrderSerializer(serializers.Serializer):
    start_date = serializers.DateField()
    end_date = serializers.DateField()

    def validate(self, attrs):
        """Validate across multiple fields."""
        if attrs['end_date'] <= attrs['start_date']:
            raise serializers.ValidationError({
                'end_date': "End date must be after start date"
            })
        return attrs
```

### Validation with Context

```python
class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True)

    def validate_items(self, value):
        """Validate with request context."""
        request = self.context.get('request')
        if not request:
            return value

        customer = request.user.customer
        if len(value) > customer.max_items_per_order:
            raise serializers.ValidationError(
                f"Maximum {customer.max_items_per_order} items allowed"
            )
        return value
```

### Business Validation in Use Case (Preferred)

**Serializers validate format/structure. Business rules belong in use cases/services.**

```python
# Serializer: Format validation only
class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True, min_length=1)
    # No business logic here


# Use case: Business validation
class CreateOrderUseCase:
    def execute(self, input_dto):
        # Check inventory (business rule)
        is_available, error = self._inventory_service.validate_availability(
            input_dto.items
        )
        if not is_available:
            return CreateOrderOutput(success=False, error=error)
        ...
```

## Permission Patterns

### DRF Built-in Permissions

```python
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
```

### Custom Permission Classes

```python
# apps/orders/permissions.py
from rest_framework.permissions import BasePermission


class IsOrderOwner(BasePermission):
    """Only order owner can access."""

    def has_object_permission(self, request, view, obj):
        return obj.customer.user == request.user


class CanModifyOrder(BasePermission):
    """Check if order can be modified."""

    def has_object_permission(self, request, view, obj):
        if request.method in ('PUT', 'PATCH', 'DELETE'):
            return obj.is_editable
        return True
```

### Permission Mixin (RBAC Pattern)

```python
# core/permissions.py
from rest_framework.permissions import BasePermission
from rest_framework.exceptions import PermissionDenied


class PermissionRequiredMixin:
    """
    Mixin for RBAC permission checking.

    Usage:
        class OrderViewSet(PermissionRequiredMixin, viewsets.ModelViewSet):
            required_permissions = {
                'list': ['orders.view'],
                'create': ['orders.create'],
                'update': ['orders.update'],
                'destroy': ['orders.delete'],
            }
    """
    required_permissions: dict[str, list[str]] | None = None

    def check_permissions(self, request):
        super().check_permissions(request)

        if not self.required_permissions:
            return

        user = request.user

        # Admin bypass
        if hasattr(user, 'role') and user.role == 'ADMIN':
            return

        # Get permissions for this action
        perms = self.required_permissions.get(self.action, [])
        if not perms:
            return

        # Check if user has any required permission
        has_perm = any(
            user.has_perm(f"permissions.{p}") for p in perms
        )

        if not has_perm:
            raise PermissionDenied("Insufficient permissions")
```

## Response Formatting

### Success Responses

```python
# Create
return Response(
    {'order_id': str(order.id)},
    status=status.HTTP_201_CREATED,
)

# Retrieve
return Response(OrderReadSerializer(order).data)

# List
return Response(OrderListSerializer(orders, many=True).data)

# Update
return Response(OrderReadSerializer(order).data)

# Delete
return Response(status=status.HTTP_204_NO_CONTENT)
```

### Error Responses

```python
# From use case failure
if not output.success:
    return Response(
        {
            'error': output.error,
            'code': output.error_code,
        },
        status=status.HTTP_400_BAD_REQUEST,
    )

# Not found
return Response(
    {'error': 'Order not found'},
    status=status.HTTP_404_NOT_FOUND,
)

# Validation errors (automatic from serializer)
serializer.is_valid(raise_exception=True)  # Raises 400 with details
```

### Paginated Responses

```python
class OrderViewSet(viewsets.ModelViewSet):
    pagination_class = PageNumberPagination

    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        page = self.paginate_queryset(queryset)

        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

## URL Routing

### Basic URLs

```python
# apps/orders/urls.py
from django.urls import path
from apps.orders import views

app_name = 'orders'

urlpatterns = [
    path('', views.OrderListCreateView.as_view(), name='order-list-create'),
    path('<uuid:pk>/', views.OrderDetailView.as_view(), name='order-detail'),
    path('<uuid:pk>/cancel/', views.CancelOrderView.as_view(), name='order-cancel'),
]
```

### Router for ViewSets

```python
# apps/orders/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from apps.orders import views

router = DefaultRouter()
router.register('', views.OrderViewSet, basename='order')

urlpatterns = [
    path('', include(router.urls)),
]
```

## Nested Serializers

### Read with Nested Objects

```python
class CustomerReadSerializer(serializers.ModelSerializer):
    class Meta:
        model = Customer
        fields = ['id', 'name', 'email']


class OrderDetailSerializer(serializers.ModelSerializer):
    customer = CustomerReadSerializer(read_only=True)
    items = OrderItemReadSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'customer', 'items', 'status', 'total']
```

### Write with Nested Creation

```python
class CreateOrderWithItemsSerializer(serializers.Serializer):
    """Create order with nested items in one request."""
    items = OrderItemInputSerializer(many=True)
    notes = serializers.CharField(required=False)

    # Items are passed to use case, which handles creation
```

## SerializerMethodField

For computed fields that need logic:

```python
class OrderReadSerializer(serializers.ModelSerializer):
    total = serializers.SerializerMethodField()
    can_cancel = serializers.SerializerMethodField()
    user_reservation = serializers.SerializerMethodField()

    class Meta:
        model = Order
        fields = ['id', 'status', 'total', 'can_cancel', 'user_reservation']

    def get_total(self, obj: Order) -> str:
        """Calculate total from items."""
        return str(obj.total)

    def get_can_cancel(self, obj: Order) -> bool:
        """Check if current user can cancel."""
        return obj.is_cancellable

    def get_user_reservation(self, obj: Order) -> dict | None:
        """Get current user's reservation for this order."""
        request = self.context.get('request')
        if not request or not request.user.is_authenticated:
            return None

        reservation = obj.reservations.filter(
            user=request.user
        ).values('id', 'status').first()

        return reservation
```

## Anti-Patterns to Avoid

### Fat Views

```python
# BAD: View contains business logic
class CreateOrderView(APIView):
    def post(self, request):
        # 50 lines of business logic
        items = request.data['items']
        for item in items:
            product = Product.objects.get(id=item['product_id'])
            if product.stock < item['quantity']:
                return Response({'error': 'Out of stock'}, status=400)
            product.stock -= item['quantity']
            product.save()

        order = Order.objects.create(customer=request.user.customer)
        # ... more logic

# GOOD: View delegates to use case
class CreateOrderView(APIView):
    def post(self, request):
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase(...)
        output = use_case.execute(CreateOrderInput(...))

        if not output.success:
            return Response({'error': output.error}, status=400)
        return Response({'order_id': str(output.order_id)}, status=201)
```

### Direct ORM in Views

```python
# BAD: View queries database directly
class OrderDetailView(APIView):
    def get(self, request, pk):
        order = Order.objects.select_related('customer').get(pk=pk)
        return Response(OrderSerializer(order).data)

# GOOD: Use service for queries
class OrderDetailView(APIView):
    def get(self, request, pk):
        order = OrderService.get_by_id(pk)
        if not order:
            return Response({'error': 'Not found'}, status=404)
        return Response(OrderReadSerializer(order).data)
```

### Single Serializer for Everything

```python
# BAD: One serializer for all operations
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = '__all__'

# GOOD: Separate by purpose
class OrderReadSerializer(serializers.ModelSerializer):
    ...

class OrderListSerializer(serializers.ModelSerializer):
    ...

class CreateOrderSerializer(serializers.Serializer):
    ...

class UpdateOrderSerializer(serializers.Serializer):
    ...
```

### Business Logic in Serializers

```python
# BAD: Serializer checks inventory
class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True)

    def validate_items(self, value):
        for item in value:
            product = Product.objects.get(id=item['product_id'])
            if product.stock < item['quantity']:
                raise ValidationError("Out of stock")  # Business logic!
        return value

# GOOD: Serializer validates format only
class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True, min_length=1)

    def validate_items(self, value):
        # Only format validation
        if len(value) > 50:
            raise ValidationError("Maximum 50 items")
        return value

# Business logic in use case
class CreateOrderUseCase:
    def execute(self, input_dto):
        is_valid, error = self._inventory_service.validate_availability(...)
        if not is_valid:
            return CreateOrderOutput(success=False, error=error)
```
