# Complete Examples for Django Clean Architecture

## Overview

This document provides complete, production-ready examples of Clean Architecture patterns in Django. Each example shows the full implementation across all layers.

## Example 1: Order Management (Complete CRUD)

### Directory Structure

```
apps/orders/
├── __init__.py
├── models.py
├── views.py
├── serializers.py
├── urls.py
├── admin.py
├── services/
│   ├── __init__.py
│   └── order_service.py
├── use_cases/
│   ├── __init__.py
│   ├── create_order.py
│   ├── get_order.py
│   ├── list_orders.py
│   ├── update_order.py
│   └── cancel_order.py
└── tests/
    ├── __init__.py
    ├── factories.py
    ├── test_services.py
    ├── test_use_cases.py
    └── test_views.py
```

### Models (models.py)

```python
import uuid
from decimal import Decimal
from django.db import models
from django.utils import timezone


class Order(models.Model):
    """Order entity."""

    class Status(models.TextChoices):
        PENDING = 'pending', 'Pending'
        CONFIRMED = 'confirmed', 'Confirmed'
        PROCESSING = 'processing', 'Processing'
        SHIPPED = 'shipped', 'Shipped'
        DELIVERED = 'delivered', 'Delivered'
        CANCELLED = 'cancelled', 'Cancelled'

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    customer = models.ForeignKey(
        'customers.Customer',
        on_delete=models.PROTECT,
        related_name='orders',
    )
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
        db_index=True,
    )
    notes = models.TextField(blank=True, default='')
    shipping_address = models.ForeignKey(
        'addresses.Address',
        on_delete=models.SET_NULL,
        related_name='shipping_orders',
        null=True,
        blank=True,
    )
    tracking_number = models.CharField(max_length=100, blank=True, default='')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    confirmed_at = models.DateTimeField(null=True, blank=True)
    shipped_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['customer', 'status']),
            models.Index(fields=['status', 'created_at']),
        ]

    def __str__(self) -> str:
        return f"Order {self.id}"

    @property
    def is_cancellable(self) -> bool:
        return self.status in (self.Status.PENDING, self.Status.CONFIRMED)

    @property
    def is_editable(self) -> bool:
        return self.status == self.Status.PENDING

    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self.items.all()) or Decimal('0.00')

    @property
    def items_count(self) -> int:
        return self.items.count()


class OrderItem(models.Model):
    """Order line item."""

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product = models.ForeignKey(
        'products.Product',
        on_delete=models.PROTECT,
        related_name='order_items',
    )
    quantity = models.PositiveIntegerField(default=1)
    unit_price = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ['order', 'product']

    @property
    def subtotal(self) -> Decimal:
        return Decimal(self.quantity) * self.unit_price
```

### Services (services/order_service.py)

```python
from uuid import UUID
from decimal import Decimal
from django.db import transaction
from django.db.models import QuerySet
from django.utils import timezone

from apps.orders.models import Order, OrderItem


class OrderService:
    """Service for order operations."""

    # ============ QUERIES (Static) ============

    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        return (
            Order.objects
            .select_related('customer', 'shipping_address')
            .prefetch_related('items__product')
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def list_for_customer(
        customer_id: UUID,
        status: Order.Status | None = None,
        limit: int = 20,
        offset: int = 0,
    ) -> QuerySet[Order]:
        qs = (
            Order.objects
            .filter(customer_id=customer_id)
            .select_related('customer')
            .prefetch_related('items')
            .order_by('-created_at')
        )
        if status:
            qs = qs.filter(status=status)
        return qs[offset:offset + limit]

    @staticmethod
    def count_for_customer(customer_id: UUID) -> int:
        return Order.objects.filter(customer_id=customer_id).count()

    # ============ MUTATIONS (Instance) ============

    def create(
        self,
        customer_id: UUID,
        items: list['OrderItemInput'],
        notes: str = '',
        shipping_address_id: UUID | None = None,
    ) -> Order:
        order = Order.objects.create(
            customer_id=customer_id,
            notes=notes,
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

    def update(
        self,
        order: Order,
        notes: str | None = None,
        shipping_address_id: UUID | None = None,
    ) -> Order:
        update_fields = ['updated_at']
        if notes is not None:
            order.notes = notes
            update_fields.append('notes')
        if shipping_address_id is not None:
            order.shipping_address_id = shipping_address_id
            update_fields.append('shipping_address_id')
        order.save(update_fields=update_fields)
        return order

    def confirm(self, order: Order) -> Order:
        order.status = Order.Status.CONFIRMED
        order.confirmed_at = timezone.now()
        order.save(update_fields=['status', 'confirmed_at', 'updated_at'])
        return order

    def cancel(self, order: Order, reason: str = '') -> Order:
        order.status = Order.Status.CANCELLED
        order.notes = f"{order.notes}\nCancellation: {reason}".strip()
        order.save(update_fields=['status', 'notes', 'updated_at'])
        return order

    # ============ VALIDATIONS ============

    def validate_can_update(self, order: Order) -> tuple[bool, str]:
        if not order.is_editable:
            return False, f"Cannot update order in {order.status} status"
        return True, ""

    def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
        if not order.is_cancellable:
            return False, f"Cannot cancel order in {order.status} status"
        return True, ""

    def validate_can_confirm(self, order: Order) -> tuple[bool, str]:
        if order.status != Order.Status.PENDING:
            return False, "Only pending orders can be confirmed"
        if not order.items.exists():
            return False, "Cannot confirm empty order"
        return True, ""
```

### Use Cases

#### create_order.py

```python
from dataclasses import dataclass
from uuid import UUID
from decimal import Decimal
from django.db import transaction

from apps.orders.services import OrderService
from apps.inventory.services import InventoryService


@dataclass(frozen=True, slots=True)
class OrderItemInput:
    product_id: UUID
    quantity: int
    unit_price: Decimal


@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list[OrderItemInput]
    notes: str = ''
    shipping_address_id: UUID | None = None


@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None
    error_code: str | None = None


class CreateOrderUseCase:
    """Create a new order."""

    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService | None = None,
    ):
        self._order_service = order_service
        self._inventory_service = inventory_service or InventoryService()

    @transaction.atomic
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        # Validate inventory
        for item in input_dto.items:
            is_available, error = self._inventory_service.validate_availability(
                item.product_id,
                item.quantity,
            )
            if not is_available:
                return CreateOrderOutput(
                    success=False,
                    error=error,
                    error_code='INVENTORY_UNAVAILABLE',
                )

        # Create order
        order = self._order_service.create(
            customer_id=input_dto.customer_id,
            items=input_dto.items,
            notes=input_dto.notes,
            shipping_address_id=input_dto.shipping_address_id,
        )

        # Reserve inventory
        for item in input_dto.items:
            self._inventory_service.reserve(item.product_id, item.quantity)

        return CreateOrderOutput(success=True, order_id=order.id)
```

#### get_order.py

```python
from dataclasses import dataclass
from uuid import UUID
from decimal import Decimal
from datetime import datetime

from apps.orders.services import OrderService


@dataclass(frozen=True, slots=True)
class GetOrderInput:
    order_id: UUID
    customer_id: UUID  # For authorization


@dataclass(frozen=True, slots=True)
class OrderItemDetails:
    id: UUID
    product_id: UUID
    product_name: str
    quantity: int
    unit_price: Decimal
    subtotal: Decimal


@dataclass(frozen=True, slots=True)
class OrderDetails:
    id: UUID
    status: str
    status_display: str
    notes: str
    items: list[OrderItemDetails]
    total: Decimal
    is_cancellable: bool
    is_editable: bool
    created_at: datetime
    updated_at: datetime


@dataclass(frozen=True, slots=True)
class GetOrderOutput:
    success: bool
    order: OrderDetails | None = None
    error: str | None = None


class GetOrderUseCase:
    """Get order details."""

    def __init__(self, order_service: OrderService):
        self._order_service = order_service

    def execute(self, input_dto: GetOrderInput) -> GetOrderOutput:
        order = self._order_service.get_by_id(input_dto.order_id)

        if not order:
            return GetOrderOutput(success=False, error="Order not found")

        # Authorization check
        if order.customer_id != input_dto.customer_id:
            return GetOrderOutput(success=False, error="Order not found")

        return GetOrderOutput(
            success=True,
            order=OrderDetails(
                id=order.id,
                status=order.status,
                status_display=order.get_status_display(),
                notes=order.notes,
                items=[
                    OrderItemDetails(
                        id=item.id,
                        product_id=item.product_id,
                        product_name=item.product.name,
                        quantity=item.quantity,
                        unit_price=item.unit_price,
                        subtotal=item.subtotal,
                    )
                    for item in order.items.all()
                ],
                total=order.total,
                is_cancellable=order.is_cancellable,
                is_editable=order.is_editable,
                created_at=order.created_at,
                updated_at=order.updated_at,
            ),
        )
```

#### cancel_order.py

```python
from dataclasses import dataclass
from uuid import UUID
from django.db import transaction

from apps.orders.services import OrderService
from apps.inventory.services import InventoryService


@dataclass(frozen=True, slots=True)
class CancelOrderInput:
    order_id: UUID
    customer_id: UUID
    reason: str = ''


@dataclass(frozen=True, slots=True)
class CancelOrderOutput:
    success: bool
    error: str | None = None


class CancelOrderUseCase:
    """Cancel an order."""

    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService | None = None,
    ):
        self._order_service = order_service
        self._inventory_service = inventory_service or InventoryService()

    @transaction.atomic
    def execute(self, input_dto: CancelOrderInput) -> CancelOrderOutput:
        order = self._order_service.get_by_id(input_dto.order_id)

        if not order:
            return CancelOrderOutput(success=False, error="Order not found")

        if order.customer_id != input_dto.customer_id:
            return CancelOrderOutput(success=False, error="Order not found")

        # Validate
        is_valid, error = self._order_service.validate_can_cancel(order)
        if not is_valid:
            return CancelOrderOutput(success=False, error=error)

        # Cancel order
        self._order_service.cancel(order, input_dto.reason)

        # Release inventory
        for item in order.items.all():
            self._inventory_service.release(item.product_id, item.quantity)

        return CancelOrderOutput(success=True)
```

### Serializers (serializers.py)

```python
from rest_framework import serializers
from apps.orders.models import Order, OrderItem


# ============ WRITE SERIALIZERS ============

class OrderItemInputSerializer(serializers.Serializer):
    product_id = serializers.UUIDField()
    quantity = serializers.IntegerField(min_value=1, max_value=100)


class CreateOrderSerializer(serializers.Serializer):
    items = OrderItemInputSerializer(many=True, min_length=1)
    notes = serializers.CharField(max_length=500, required=False, default='')
    shipping_address_id = serializers.UUIDField(required=False, allow_null=True)

    def validate_items(self, value):
        product_ids = [item['product_id'] for item in value]
        if len(product_ids) != len(set(product_ids)):
            raise serializers.ValidationError("Duplicate products not allowed")
        return value


class UpdateOrderSerializer(serializers.Serializer):
    notes = serializers.CharField(max_length=500, required=False)
    shipping_address_id = serializers.UUIDField(required=False, allow_null=True)


class CancelOrderSerializer(serializers.Serializer):
    reason = serializers.CharField(max_length=500, required=False, default='')


# ============ READ SERIALIZERS ============

class OrderItemReadSerializer(serializers.ModelSerializer):
    product_name = serializers.CharField(source='product.name', read_only=True)
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
    customer_name = serializers.CharField(source='customer.name', read_only=True)
    items = OrderItemReadSerializer(many=True, read_only=True)
    status_display = serializers.CharField(source='get_status_display', read_only=True)
    total = serializers.DecimalField(max_digits=12, decimal_places=2, read_only=True)
    is_cancellable = serializers.BooleanField(read_only=True)
    is_editable = serializers.BooleanField(read_only=True)

    class Meta:
        model = Order
        fields = [
            'id',
            'customer_name',
            'status',
            'status_display',
            'items',
            'total',
            'notes',
            'is_cancellable',
            'is_editable',
            'created_at',
            'updated_at',
        ]
        read_only_fields = fields


class OrderListSerializer(serializers.ModelSerializer):
    items_count = serializers.IntegerField(read_only=True)
    total = serializers.DecimalField(max_digits=12, decimal_places=2, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'status', 'items_count', 'total', 'created_at']
        read_only_fields = fields
```

### Views (views.py)

```python
from rest_framework.views import APIView
from rest_framework.request import Request
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from uuid import UUID

from apps.orders.serializers import (
    CreateOrderSerializer,
    UpdateOrderSerializer,
    CancelOrderSerializer,
    OrderReadSerializer,
    OrderListSerializer,
)
from apps.orders.use_cases import (
    CreateOrderUseCase,
    CreateOrderInput,
    OrderItemInput,
    GetOrderUseCase,
    GetOrderInput,
    ListOrdersUseCase,
    ListOrdersInput,
    UpdateOrderUseCase,
    UpdateOrderInput,
    CancelOrderUseCase,
    CancelOrderInput,
)
from apps.orders.services import OrderService
from apps.products.services import ProductService


class OrderListCreateView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request: Request) -> Response:
        """List customer's orders."""
        use_case = ListOrdersUseCase(order_service=OrderService())
        output = use_case.execute(
            ListOrdersInput(
                customer_id=request.user.customer.id,
                status=request.query_params.get('status'),
                page=int(request.query_params.get('page', 1)),
                page_size=int(request.query_params.get('page_size', 20)),
            )
        )

        return Response({
            'orders': [
                {
                    'id': str(o.id),
                    'status': o.status,
                    'items_count': o.items_count,
                    'total': str(o.total),
                    'created_at': o.created_at.isoformat(),
                }
                for o in output.orders
            ],
            'total_count': output.total_count,
            'page': output.page,
            'page_size': output.page_size,
        })

    def post(self, request: Request) -> Response:
        """Create a new order."""
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Get product prices
        product_service = ProductService()
        items_with_prices = []
        for item_data in serializer.validated_data['items']:
            product = product_service.get_by_id(item_data['product_id'])
            if not product:
                return Response(
                    {'error': f"Product {item_data['product_id']} not found"},
                    status=status.HTTP_400_BAD_REQUEST,
                )
            items_with_prices.append(
                OrderItemInput(
                    product_id=item_data['product_id'],
                    quantity=item_data['quantity'],
                    unit_price=product.price,
                )
            )

        use_case = CreateOrderUseCase(order_service=OrderService())
        output = use_case.execute(
            CreateOrderInput(
                customer_id=request.user.customer.id,
                items=items_with_prices,
                notes=serializer.validated_data.get('notes', ''),
                shipping_address_id=serializer.validated_data.get('shipping_address_id'),
            )
        )

        if not output.success:
            return Response(
                {'error': output.error, 'code': output.error_code},
                status=status.HTTP_400_BAD_REQUEST,
            )

        order = OrderService.get_by_id(output.order_id)
        return Response(
            OrderReadSerializer(order).data,
            status=status.HTTP_201_CREATED,
        )


class OrderDetailView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request: Request, pk: UUID) -> Response:
        """Get order details."""
        use_case = GetOrderUseCase(order_service=OrderService())
        output = use_case.execute(
            GetOrderInput(
                order_id=pk,
                customer_id=request.user.customer.id,
            )
        )

        if not output.success:
            return Response(
                {'error': output.error},
                status=status.HTTP_404_NOT_FOUND,
            )

        return Response({
            'id': str(output.order.id),
            'status': output.order.status,
            'status_display': output.order.status_display,
            'notes': output.order.notes,
            'items': [
                {
                    'id': str(item.id),
                    'product_id': str(item.product_id),
                    'product_name': item.product_name,
                    'quantity': item.quantity,
                    'unit_price': str(item.unit_price),
                    'subtotal': str(item.subtotal),
                }
                for item in output.order.items
            ],
            'total': str(output.order.total),
            'is_cancellable': output.order.is_cancellable,
            'is_editable': output.order.is_editable,
            'created_at': output.order.created_at.isoformat(),
            'updated_at': output.order.updated_at.isoformat(),
        })


class CancelOrderView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request: Request, pk: UUID) -> Response:
        """Cancel an order."""
        serializer = CancelOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CancelOrderUseCase(order_service=OrderService())
        output = use_case.execute(
            CancelOrderInput(
                order_id=pk,
                customer_id=request.user.customer.id,
                reason=serializer.validated_data.get('reason', ''),
            )
        )

        if not output.success:
            return Response(
                {'error': output.error},
                status=status.HTTP_400_BAD_REQUEST,
            )

        return Response({'status': 'cancelled'})
```

### URLs (urls.py)

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

### Tests (tests/test_use_cases.py)

```python
import pytest
from unittest.mock import Mock
from uuid import uuid4
from decimal import Decimal

from apps.orders.use_cases import (
    CreateOrderUseCase,
    CreateOrderInput,
    OrderItemInput,
    CancelOrderUseCase,
    CancelOrderInput,
)
from apps.orders.services import OrderService
from apps.orders.models import Order


class TestCreateOrderUseCase:
    def test_creates_order_successfully(self):
        # Arrange
        mock_order_service = Mock(spec=OrderService)
        mock_inventory_service = Mock()
        mock_inventory_service.validate_availability.return_value = (True, "")

        mock_order = Mock(id=uuid4())
        mock_order_service.create.return_value = mock_order

        use_case = CreateOrderUseCase(
            order_service=mock_order_service,
            inventory_service=mock_inventory_service,
        )

        input_dto = CreateOrderInput(
            customer_id=uuid4(),
            items=[OrderItemInput(uuid4(), 2, Decimal('10.00'))],
        )

        # Act
        output = use_case.execute(input_dto)

        # Assert
        assert output.success is True
        assert output.order_id == mock_order.id

    def test_fails_when_inventory_unavailable(self):
        # Arrange
        mock_order_service = Mock(spec=OrderService)
        mock_inventory_service = Mock()
        mock_inventory_service.validate_availability.return_value = (
            False,
            "Out of stock",
        )

        use_case = CreateOrderUseCase(
            order_service=mock_order_service,
            inventory_service=mock_inventory_service,
        )

        input_dto = CreateOrderInput(
            customer_id=uuid4(),
            items=[OrderItemInput(uuid4(), 100, Decimal('10.00'))],
        )

        # Act
        output = use_case.execute(input_dto)

        # Assert
        assert output.success is False
        assert output.error == "Out of stock"
        mock_order_service.create.assert_not_called()


class TestCancelOrderUseCase:
    def test_cancels_pending_order(self):
        # Arrange
        customer_id = uuid4()
        mock_order = Mock(
            id=uuid4(),
            customer_id=customer_id,
            status=Order.Status.PENDING,
        )
        mock_order.items.all.return_value = []

        mock_order_service = Mock(spec=OrderService)
        mock_order_service.get_by_id.return_value = mock_order
        mock_order_service.validate_can_cancel.return_value = (True, "")

        use_case = CancelOrderUseCase(order_service=mock_order_service)

        # Act
        output = use_case.execute(
            CancelOrderInput(order_id=mock_order.id, customer_id=customer_id)
        )

        # Assert
        assert output.success is True
        mock_order_service.cancel.assert_called_once()

    def test_fails_for_shipped_order(self):
        # Arrange
        customer_id = uuid4()
        mock_order = Mock(id=uuid4(), customer_id=customer_id)

        mock_order_service = Mock(spec=OrderService)
        mock_order_service.get_by_id.return_value = mock_order
        mock_order_service.validate_can_cancel.return_value = (
            False,
            "Cannot cancel shipped",
        )

        use_case = CancelOrderUseCase(order_service=mock_order_service)

        # Act
        output = use_case.execute(
            CancelOrderInput(order_id=mock_order.id, customer_id=customer_id)
        )

        # Assert
        assert output.success is False
        assert "shipped" in output.error.lower()
```

## Example 2: User Registration with Email Verification

### Use Case

```python
# apps/users/use_cases/register_user.py
from dataclasses import dataclass
from django.db import transaction

from apps.users.services import UserService, EmailService


@dataclass(frozen=True, slots=True)
class RegisterUserInput:
    email: str
    password: str
    first_name: str
    last_name: str


@dataclass(frozen=True, slots=True)
class RegisterUserOutput:
    success: bool
    user_id: str | None = None
    error: str | None = None
    error_code: str | None = None


class RegisterUserUseCase:
    def __init__(
        self,
        user_service: UserService,
        email_service: EmailService | None = None,
    ):
        self._user_service = user_service
        self._email_service = email_service

    @transaction.atomic
    def execute(self, input_dto: RegisterUserInput) -> RegisterUserOutput:
        # Check if email exists
        if self._user_service.email_exists(input_dto.email):
            return RegisterUserOutput(
                success=False,
                error="Email already registered",
                error_code="EMAIL_EXISTS",
            )

        # Create user
        user = self._user_service.create(
            email=input_dto.email,
            password=input_dto.password,
            first_name=input_dto.first_name,
            last_name=input_dto.last_name,
        )

        # Send verification email (after commit)
        if self._email_service:
            transaction.on_commit(
                lambda: self._email_service.send_verification(user.id)
            )

        return RegisterUserOutput(success=True, user_id=str(user.id))
```

## Example 3: Pagination Pattern

### Use Case with Pagination

```python
from dataclasses import dataclass
from uuid import UUID

from apps.orders.services import OrderService
from apps.orders.models import Order


@dataclass(frozen=True, slots=True)
class ListOrdersInput:
    customer_id: UUID
    status: str | None = None
    page: int = 1
    page_size: int = 20


@dataclass(frozen=True, slots=True)
class OrderSummary:
    id: UUID
    status: str
    items_count: int
    total: str
    created_at: str


@dataclass(frozen=True, slots=True)
class ListOrdersOutput:
    success: bool
    orders: list[OrderSummary]
    total_count: int
    page: int
    page_size: int
    has_next: bool
    has_previous: bool


class ListOrdersUseCase:
    def __init__(self, order_service: OrderService):
        self._order_service = order_service

    def execute(self, input_dto: ListOrdersInput) -> ListOrdersOutput:
        # Calculate offset
        offset = (input_dto.page - 1) * input_dto.page_size

        # Get orders
        orders = self._order_service.list_for_customer(
            customer_id=input_dto.customer_id,
            status=Order.Status(input_dto.status) if input_dto.status else None,
            limit=input_dto.page_size + 1,  # +1 to check has_next
            offset=offset,
        )

        orders_list = list(orders)
        has_next = len(orders_list) > input_dto.page_size
        if has_next:
            orders_list = orders_list[:-1]

        total_count = self._order_service.count_for_customer(input_dto.customer_id)

        return ListOrdersOutput(
            success=True,
            orders=[
                OrderSummary(
                    id=o.id,
                    status=o.status,
                    items_count=o.items_count,
                    total=str(o.total),
                    created_at=o.created_at.isoformat(),
                )
                for o in orders_list
            ],
            total_count=total_count,
            page=input_dto.page,
            page_size=input_dto.page_size,
            has_next=has_next,
            has_previous=input_dto.page > 1,
        )
```

## Quick Reference

### Layer Responsibilities

| Layer | Responsibility | Example |
|-------|---------------|---------|
| Views | HTTP handling | Parse request, return response |
| Serializers | I/O validation | Validate JSON, format output |
| Use Cases | Orchestration | Coordinate services, handle transactions |
| Services | Business logic | CRUD, validation, domain operations |
| Models | Data + properties | Fields, relationships, computed values |

### Common Patterns

| Pattern | When to Use |
|---------|-------------|
| Static method | Read-only queries |
| Instance method | State-changing operations |
| `tuple[bool, str]` | Validation results |
| `@transaction.atomic` | Multi-step mutations |
| `transaction.on_commit` | Side effects after success |
| Separate serializers | Different read/write needs |
