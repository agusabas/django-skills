# Clean Architecture Transformation Patterns

When refactoring to Clean Architecture, follow these transformation patterns. For complete patterns, see the `django-clean-drf` skill.

## Architecture Layers

Transform code to follow the dependency rule (outer layers depend on inner layers):

```
Views (HTTP) → Use Cases (Business Logic) → Services (Data Access) → Models (Domain)
```

## Transformation 1: Fat Views → Use Cases

Extract business logic from views into dedicated Use Case classes:

```python
# BEFORE: Fat view with all logic
class CreateOrderView(APIView):
    def post(self, request):
        # 1. Validation
        serializer = OrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 2. Business logic (shouldn't be here!)
        items = serializer.validated_data['items']
        total = sum(item['price'] * item['quantity'] for item in items)
        if total > request.user.credit_limit:
            return Response({"error": "Credit limit exceeded"}, status=400)

        # 3. Data access (shouldn't be here!)
        with transaction.atomic():
            order = Order.objects.create(
                customer=request.user,
                total=total,
            )
            for item in items:
                OrderItem.objects.create(order=order, **item)

        # 4. Side effects (shouldn't be here!)
        send_order_email.delay(order.id)

        return Response(OrderSerializer(order).data, status=201)


# AFTER: Thin view + Use Case
# apps/orders/dtos.py
from dataclasses import dataclass
from uuid import UUID

@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list['OrderItemInput']

@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str = ""

# apps/orders/use_cases/create_order.py
from django.db import transaction

class CreateOrderUseCase:
    def __init__(
        self,
        order_service: OrderService | None = None,
        customer_service: CustomerService | None = None,
    ):
        self._order_service = order_service or OrderService()
        self._customer_service = customer_service or CustomerService()

    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        # Validation via service
        customer = self._customer_service.get_by_id(input_dto.customer_id)
        if not customer:
            return CreateOrderOutput(success=False, error="Customer not found")

        total = self._order_service.calculate_total(input_dto.items)
        is_valid, error = self._customer_service.validate_credit_limit(customer, total)
        if not is_valid:
            return CreateOrderOutput(success=False, error=error)

        # Create order via service
        with transaction.atomic():
            order = self._order_service.create(
                customer_id=input_dto.customer_id,
                items=input_dto.items,
            )

        return CreateOrderOutput(success=True, order_id=order.id)

# apps/orders/views.py
class CreateOrderView(APIView):
    def post(self, request):
        serializer = CreateOrderWriteSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase()
        output = use_case.execute(CreateOrderInput(
            customer_id=request.user.id,
            items=serializer.validated_data['items'],
        ))

        if not output.success:
            return Response({"error": output.error}, status=400)

        order = OrderService.get_by_id(output.order_id)
        return Response(OrderReadSerializer(order).data, status=201)
```

## Transformation 2: Service Layer Separation

Split mixed services into queries (static) and mutations (instance):

```python
# BEFORE: Mixed methods, unclear responsibilities
class OrderService:
    def get_order(self, order_id):
        order = Order.objects.filter(id=order_id).first()
        if not order:
            raise NotFound("Order not found")  # HTTP in service!
        return order

    def create_order(self, customer_id, items):
        return Order.objects.create(customer_id=customer_id)

    def check_can_cancel(self, order):
        if order.status == "shipped":
            raise ValidationError("Cannot cancel")  # Exception for logic!
        return True


# AFTER: Clean separation
class OrderService:
    """
    Queries: Static methods (read-only)
    Mutations: Instance methods (state changes)
    Validations: Return tuple[bool, str]
    """

    # QUERIES (Static)
    @staticmethod
    def get_by_id(order_id: UUID) -> Order | None:
        return (
            Order.objects
            .select_related('customer')
            .prefetch_related('items')
            .filter(id=order_id)
            .first()
        )

    @staticmethod
    def exists(order_id: UUID) -> bool:
        return Order.objects.filter(id=order_id).exists()

    # MUTATIONS (Instance)
    def create(
        self,
        customer_id: UUID,
        items: list[OrderItemInput],
    ) -> Order:
        order = Order.objects.create(customer_id=customer_id)
        for item in items:
            OrderItem.objects.create(order=order, **item)
        return order

    # VALIDATIONS (Return tuple)
    def validate_can_cancel(self, order: Order) -> tuple[bool, str]:
        if order.status == Order.Status.SHIPPED:
            return False, "Cannot cancel shipped orders"
        if order.status == Order.Status.CANCELLED:
            return False, "Order is already cancelled"
        return True, ""
```

## Transformation 3: Exceptions → Return Values

Replace exception-based error handling with explicit return values:

```python
# BEFORE: Exceptions for business logic
class PaymentService:
    def process_payment(self, order_id: UUID) -> Payment:
        order = Order.objects.get(id=order_id)  # Raises DoesNotExist

        if order.status != "pending":
            raise ValidationError("Order is not pending")  # Exception!

        if order.total > self.get_balance(order.customer_id):
            raise InsufficientFundsError("Not enough balance")  # Exception!

        return Payment.objects.create(order=order, amount=order.total)


# AFTER: Explicit return values
@dataclass(frozen=True, slots=True)
class ProcessPaymentOutput:
    success: bool
    payment_id: UUID | None = None
    error: str = ""

class ProcessPaymentUseCase:
    def execute(self, input_dto: ProcessPaymentInput) -> ProcessPaymentOutput:
        order = OrderService.get_by_id(input_dto.order_id)
        if not order:
            return ProcessPaymentOutput(success=False, error="Order not found")

        is_valid, error = self._payment_service.validate_can_process(order)
        if not is_valid:
            return ProcessPaymentOutput(success=False, error=error)

        payment = self._payment_service.create_payment(order)
        return ProcessPaymentOutput(success=True, payment_id=payment.id)
```

## Transformation 4: Serializers → Read/Write Separation

Split serializers by operation type:

```python
# BEFORE: One serializer for everything
class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True)
    customer = CustomerSerializer(read_only=True)
    customer_id = serializers.UUIDField(write_only=True)

    class Meta:
        model = Order
        fields = ['id', 'customer', 'customer_id', 'items', 'total', 'status']

    def create(self, validated_data):
        items_data = validated_data.pop('items')
        order = Order.objects.create(**validated_data)
        for item in items_data:
            OrderItem.objects.create(order=order, **item)
        return order


# AFTER: Separate Read and Write serializers
# Read serializer - for responses
class OrderReadSerializer(serializers.ModelSerializer):
    customer = CustomerReadSerializer(read_only=True)
    items = OrderItemReadSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'customer', 'items', 'total', 'status', 'created_at']
        read_only_fields = fields


# Write serializer - for input validation only
class OrderWriteSerializer(serializers.Serializer):
    customer_id = serializers.UUIDField()
    items = OrderItemWriteSerializer(many=True)
    notes = serializers.CharField(required=False, allow_blank=True)

    # No create/update - use cases handle persistence


# View uses both
class OrderViewSet(viewsets.ViewSet):
    def create(self, request):
        write_serializer = OrderWriteSerializer(data=request.data)
        write_serializer.is_valid(raise_exception=True)

        output = CreateOrderUseCase().execute(
            CreateOrderInput(**write_serializer.validated_data)
        )

        if not output.success:
            return Response({"error": output.error}, status=400)

        order = OrderService.get_by_id(output.order_id)
        return Response(OrderReadSerializer(order).data, status=201)
```

## Transformation 5: Model Logic → Domain Methods

Keep domain logic in models via properties and methods:

```python
# BEFORE: Logic scattered in views/services
class Order(models.Model):
    status = models.CharField(max_length=20)
    created_at = models.DateTimeField(auto_now_add=True)

# In view or service:
def is_overdue(order):
    if order.status != "pending":
        return False
    return timezone.now() - order.created_at > timedelta(days=7)


# AFTER: Domain logic in model
class Order(models.Model):
    class Status(models.TextChoices):
        PENDING = "pending", "Pending"
        CONFIRMED = "confirmed", "Confirmed"
        SHIPPED = "shipped", "Shipped"

    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.PENDING,
    )
    created_at = models.DateTimeField(auto_now_add=True)

    @property
    def is_overdue(self) -> bool:
        """Order is overdue if pending for more than 7 days."""
        if self.status != self.Status.PENDING:
            return False
        return timezone.now() - self.created_at > timedelta(days=7)

    @property
    def can_be_cancelled(self) -> bool:
        """Only pending and confirmed orders can be cancelled."""
        return self.status in (self.Status.PENDING, self.Status.CONFIRMED)
```

## Clean Architecture File Structure

When refactoring, organize files by layer:

```
apps/orders/
├── __init__.py
├── models.py                 # Domain entities
├── dtos.py                   # Input/Output dataclasses
├── use_cases/
│   ├── __init__.py
│   ├── create_order.py       # CreateOrderUseCase
│   ├── cancel_order.py       # CancelOrderUseCase
│   └── get_order.py          # GetOrderUseCase
├── services/
│   ├── __init__.py
│   └── order_service.py      # OrderService
├── serializers/
│   ├── __init__.py
│   ├── read.py               # OrderReadSerializer
│   └── write.py              # OrderWriteSerializer
├── views.py                  # Thin views/viewsets
├── urls.py
└── tests/
    ├── __init__.py
    ├── test_use_cases.py     # Unit tests with mocks
    ├── test_services.py      # Integration tests
    └── test_views.py         # API tests
```

## Quick Reference: Clean Architecture Principles

1. **Dependencies flow inward**: Views → Use Cases → Services → Models
2. **No exceptions for business logic**: Return `Output` DTOs or `tuple[bool, str]`
3. **Queries are static, mutations are instance methods**
4. **Views don't access models directly**: Use services
5. **Serializers don't contain business logic**: Use cases do
6. **DTOs use `@dataclass(frozen=True, slots=True)`**
7. **Use `@transaction.atomic` for state changes**
8. **Constructor injection for testability**

For complete patterns and examples, reference the `django-clean-drf` skill.
