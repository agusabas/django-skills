# Use Case Pattern for Django Clean Architecture

## Overview

Use Cases represent application-level operations. They orchestrate the flow of data between the HTTP layer and the domain layer, implementing the business logic for a specific user action.

**Key Principles:**
- One use case = one user action
- Use cases don't know about HTTP (no Request/Response)
- Input/Output via immutable dataclass DTOs
- No exceptions for business logic errors
- Constructor injection for all dependencies

## Anatomy of a Use Case

```python
# apps/orders/use_cases/create_order.py
from dataclasses import dataclass
from uuid import UUID
from django.db import transaction
from apps.orders.services import OrderService, InventoryService


# ============================================
# INPUT DTO
# ============================================
@dataclass(frozen=True, slots=True)
class OrderItemInput:
    product_id: UUID
    quantity: int


@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list[OrderItemInput]
    notes: str | None = None
    shipping_address_id: UUID | None = None


# ============================================
# OUTPUT DTO
# ============================================
@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None
    error_code: str | None = None


# ============================================
# USE CASE
# ============================================
class CreateOrderUseCase:
    """
    Creates a new order for a customer.

    Validates inventory availability, creates the order with items,
    and reserves inventory.
    """

    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService,
    ):
        self._order_service = order_service
        self._inventory_service = inventory_service

    @transaction.atomic
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        # 1. Validate business rules
        is_valid, error = self._inventory_service.validate_availability(
            input_dto.items
        )
        if not is_valid:
            return CreateOrderOutput(
                success=False,
                error=error,
                error_code="INVENTORY_UNAVAILABLE",
            )

        # 2. Execute the operation
        order = self._order_service.create(
            customer_id=input_dto.customer_id,
            items=input_dto.items,
            notes=input_dto.notes,
            shipping_address_id=input_dto.shipping_address_id,
        )

        # 3. Side effects (still within transaction)
        self._inventory_service.reserve_items(order.id, input_dto.items)

        # 4. Return success
        return CreateOrderOutput(success=True, order_id=order.id)
```

## Input DTOs

### Design Rules

1. **Immutable**: Use `frozen=True`
2. **Memory efficient**: Use `slots=True`
3. **Type-safe**: Full type hints with Python 3.10+ syntax
4. **Flat or nested**: Use nested dataclasses for complex structures
5. **Defaults at the end**: Optional fields have defaults

### Patterns

#### Simple Input
```python
@dataclass(frozen=True, slots=True)
class GetOrderInput:
    order_id: UUID
```

#### Input with Optional Fields
```python
@dataclass(frozen=True, slots=True)
class UpdateOrderInput:
    order_id: UUID
    notes: str | None = None
    shipping_address_id: UUID | None = None
```

#### Nested Input
```python
@dataclass(frozen=True, slots=True)
class AddressInput:
    street: str
    city: str
    postal_code: str
    country: str = "ES"


@dataclass(frozen=True, slots=True)
class CreateCustomerInput:
    email: str
    name: str
    billing_address: AddressInput
    shipping_address: AddressInput | None = None
```

#### Input with Validation (Prefer in Serializer)
```python
@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list[OrderItemInput]

    def __post_init__(self):
        if not self.items:
            raise ValueError("Order must have at least one item")
```

### Anti-Patterns

```python
# BAD: Mutable dataclass
@dataclass
class CreateOrderInput:
    items: list[OrderItemInput]  # Can be mutated

# BAD: Using dict instead of typed DTO
def execute(self, data: dict) -> CreateOrderOutput:
    ...

# BAD: Using Django model as input
def execute(self, order: Order) -> CreateOrderOutput:
    ...

# BAD: Using DRF serializer as input
def execute(self, serializer: CreateOrderSerializer) -> CreateOrderOutput:
    ...
```

## Output DTOs

### Design Rules

1. **Always include `success: bool`** as first field
2. **Include `error: str | None`** for error messages
3. **Include `error_code: str | None`** for programmatic error handling
4. **Return data on success** via optional fields

### Patterns

#### Simple Output
```python
@dataclass(frozen=True, slots=True)
class CreateOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None
```

#### Output with Rich Data
```python
@dataclass(frozen=True, slots=True)
class OrderDetails:
    id: UUID
    status: str
    total: Decimal
    items_count: int


@dataclass(frozen=True, slots=True)
class GetOrderOutput:
    success: bool
    order: OrderDetails | None = None
    error: str | None = None
```

#### Output with Multiple Error Codes
```python
@dataclass(frozen=True, slots=True)
class PlaceOrderOutput:
    success: bool
    order_id: UUID | None = None
    error: str | None = None
    error_code: str | None = None  # INVENTORY_UNAVAILABLE, PAYMENT_FAILED, etc.
    retry_after: int | None = None  # Seconds to wait before retry
```

### Error Handling Pattern

**Never raise exceptions for business logic errors:**

```python
# BAD
class CreateOrderUseCase:
    def execute(self, input_dto: CreateOrderInput) -> Order:
        if not self._can_create():
            raise ValidationError("Cannot create order")
        return self._order_service.create(...)

# GOOD
class CreateOrderUseCase:
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        if not self._can_create():
            return CreateOrderOutput(
                success=False,
                error="Cannot create order",
                error_code="ORDER_CREATION_BLOCKED",
            )
        order = self._order_service.create(...)
        return CreateOrderOutput(success=True, order_id=order.id)
```

**When to raise exceptions:**
- Database connection errors (infrastructure)
- Programming errors (bugs)
- System failures

## Constructor Injection

### Why Constructor Injection?

1. **Testability**: Easy to mock dependencies
2. **Explicit dependencies**: Clear what the use case needs
3. **Flexibility**: Can swap implementations

### Pattern

```python
class CreateOrderUseCase:
    def __init__(
        self,
        order_service: OrderService,
        inventory_service: InventoryService,
        notification_service: NotificationService | None = None,
    ):
        self._order_service = order_service
        self._inventory_service = inventory_service
        self._notification_service = notification_service

    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        ...
        if self._notification_service:
            self._notification_service.send_order_confirmation(order.id)
        ...
```

### Usage in Views

```python
class CreateOrderView(APIView):
    def post(self, request: Request) -> Response:
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # Inject dependencies
        use_case = CreateOrderUseCase(
            order_service=OrderService(),
            inventory_service=InventoryService(),
            notification_service=NotificationService(),
        )

        input_dto = CreateOrderInput(**serializer.validated_data)
        output = use_case.execute(input_dto)

        if not output.success:
            return Response({'error': output.error}, status=400)
        return Response({'order_id': str(output.order_id)}, status=201)
```

### Testing with Mocks

```python
def test_create_order_fails_when_inventory_unavailable():
    # Arrange
    mock_order_service = Mock(spec=OrderService)
    mock_inventory_service = Mock(spec=InventoryService)
    mock_inventory_service.validate_availability.return_value = (
        False,
        "Item out of stock",
    )

    use_case = CreateOrderUseCase(
        order_service=mock_order_service,
        inventory_service=mock_inventory_service,
    )

    input_dto = CreateOrderInput(
        customer_id=uuid4(),
        items=[OrderItemInput(product_id=uuid4(), quantity=1)],
    )

    # Act
    output = use_case.execute(input_dto)

    # Assert
    assert output.success is False
    assert output.error == "Item out of stock"
    mock_order_service.create.assert_not_called()
```

## Transaction Handling

### Use `@transaction.atomic`

```python
class CreateOrderUseCase:
    @transaction.atomic
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        # All database operations here are in one transaction
        order = self._order_service.create(...)
        self._inventory_service.reserve_items(...)
        self._payment_service.charge(...)
        # If any fails, everything is rolled back
        return CreateOrderOutput(success=True, order_id=order.id)
```

### Nested Transactions with Savepoints

```python
class ProcessOrderUseCase:
    @transaction.atomic
    def execute(self, input_dto: ProcessOrderInput) -> ProcessOrderOutput:
        # Main transaction
        order = self._order_service.get_by_id(input_dto.order_id)

        # Attempt payment with savepoint
        try:
            with transaction.atomic():
                self._payment_service.charge(order)
        except PaymentError as e:
            # Savepoint rolled back, main transaction continues
            return ProcessOrderOutput(
                success=False,
                error=str(e),
                error_code="PAYMENT_FAILED",
            )

        # Continue with main transaction
        self._order_service.mark_as_paid(order)
        return ProcessOrderOutput(success=True)
```

### Side Effects After Transaction

```python
from django.db import transaction

class CreateOrderUseCase:
    @transaction.atomic
    def execute(self, input_dto: CreateOrderInput) -> CreateOrderOutput:
        order = self._order_service.create(...)

        # Register side effects to run AFTER commit
        transaction.on_commit(
            lambda: self._notification_service.send_confirmation(order.id)
        )
        transaction.on_commit(
            lambda: self._analytics_service.track_order_created(order.id)
        )

        return CreateOrderOutput(success=True, order_id=order.id)
```

## Composing Use Cases

### Use Case Calling Another Use Case

```python
class PlaceOrderUseCase:
    def __init__(
        self,
        create_order_use_case: CreateOrderUseCase,
        process_payment_use_case: ProcessPaymentUseCase,
    ):
        self._create_order = create_order_use_case
        self._process_payment = process_payment_use_case

    @transaction.atomic
    def execute(self, input_dto: PlaceOrderInput) -> PlaceOrderOutput:
        # Create order
        create_output = self._create_order.execute(
            CreateOrderInput(
                customer_id=input_dto.customer_id,
                items=input_dto.items,
            )
        )
        if not create_output.success:
            return PlaceOrderOutput(success=False, error=create_output.error)

        # Process payment
        payment_output = self._process_payment.execute(
            ProcessPaymentInput(
                order_id=create_output.order_id,
                payment_method_id=input_dto.payment_method_id,
            )
        )
        if not payment_output.success:
            # Transaction will rollback order creation too
            return PlaceOrderOutput(success=False, error=payment_output.error)

        return PlaceOrderOutput(success=True, order_id=create_output.order_id)
```

### Prefer Service Composition for Simple Cases

```python
# For simple orchestration, use services directly
class SimpleOrderUseCase:
    def __init__(
        self,
        order_service: OrderService,
        payment_service: PaymentService,
    ):
        self._order_service = order_service
        self._payment_service = payment_service

    @transaction.atomic
    def execute(self, input_dto: PlaceOrderInput) -> PlaceOrderOutput:
        order = self._order_service.create(...)
        self._payment_service.charge(order, input_dto.payment_method_id)
        return PlaceOrderOutput(success=True, order_id=order.id)
```

## File Organization

### One Use Case Per File

```
apps/orders/use_cases/
├── __init__.py
├── create_order.py      # CreateOrderUseCase + DTOs
├── cancel_order.py      # CancelOrderUseCase + DTOs
├── ship_order.py        # ShipOrderUseCase + DTOs
├── get_order.py         # GetOrderUseCase + DTOs
└── list_orders.py       # ListOrdersUseCase + DTOs
```

### Module Exports

```python
# apps/orders/use_cases/__init__.py
from apps.orders.use_cases.create_order import (
    CreateOrderUseCase,
    CreateOrderInput,
    CreateOrderOutput,
)
from apps.orders.use_cases.cancel_order import (
    CancelOrderUseCase,
    CancelOrderInput,
    CancelOrderOutput,
)

__all__ = [
    'CreateOrderUseCase',
    'CreateOrderInput',
    'CreateOrderOutput',
    'CancelOrderUseCase',
    'CancelOrderInput',
    'CancelOrderOutput',
]
```

## Query Use Cases (Read Operations)

For read operations, use cases can be simpler:

```python
@dataclass(frozen=True, slots=True)
class GetOrderInput:
    order_id: UUID
    include_items: bool = True


@dataclass(frozen=True, slots=True)
class OrderItemDetails:
    id: UUID
    product_name: str
    quantity: int
    unit_price: Decimal


@dataclass(frozen=True, slots=True)
class OrderDetails:
    id: UUID
    status: str
    customer_name: str
    items: list[OrderItemDetails]
    total: Decimal
    created_at: datetime


@dataclass(frozen=True, slots=True)
class GetOrderOutput:
    success: bool
    order: OrderDetails | None = None
    error: str | None = None


class GetOrderUseCase:
    def __init__(self, order_service: OrderService):
        self._order_service = order_service

    def execute(self, input_dto: GetOrderInput) -> GetOrderOutput:
        order = self._order_service.get_by_id(
            input_dto.order_id,
            include_items=input_dto.include_items,
        )

        if not order:
            return GetOrderOutput(
                success=False,
                error="Order not found",
            )

        return GetOrderOutput(
            success=True,
            order=OrderDetails(
                id=order.id,
                status=order.status,
                customer_name=order.customer.name,
                items=[
                    OrderItemDetails(
                        id=item.id,
                        product_name=item.product.name,
                        quantity=item.quantity,
                        unit_price=item.unit_price,
                    )
                    for item in order.items.all()
                ],
                total=order.total,
                created_at=order.created_at,
            ),
        )
```

## Common Anti-Patterns

### Fat Use Cases
```python
# BAD: Use case doing too much
class CreateOrderUseCase:
    def execute(self, input_dto):
        # Validate inventory - should be in InventoryService
        for item in input_dto.items:
            product = Product.objects.get(id=item.product_id)
            if product.stock < item.quantity:
                return ...

        # Create order - OK
        order = Order.objects.create(...)

        # Send email - should be in NotificationService
        send_mail(
            subject="Order Confirmation",
            message=f"Your order {order.id}...",
            ...
        )

        # Update analytics - should be on_commit hook
        Analytics.track("order_created", order.id)

        return ...
```

### Leaking HTTP Concepts
```python
# BAD: Use case knows about HTTP
class CreateOrderUseCase:
    def execute(self, request: Request) -> Response:
        ...
        return Response({"order_id": order.id}, status=201)
```

### Raising Exceptions for Business Logic
```python
# BAD: Raises exception for expected case
class CreateOrderUseCase:
    def execute(self, input_dto):
        if not self._inventory_service.is_available(input_dto.items):
            raise InsufficientInventoryError("Not enough stock")
        ...
```

### Direct Model Access
```python
# BAD: Use case bypassing service layer
class CreateOrderUseCase:
    def execute(self, input_dto):
        order = Order.objects.create(  # Should use OrderService
            customer_id=input_dto.customer_id,
            ...
        )
        ...
```
