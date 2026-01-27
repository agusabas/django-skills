# Testing Clean Architecture Django Applications

## Overview

Clean Architecture enables effective testing at each layer:
- **Unit tests**: Use cases and services in isolation
- **Integration tests**: Views with real database
- **Contract tests**: API contracts verification

**Key Principles:**
- Test behavior, not implementation
- Mock external dependencies, not internal layers
- Use factories for test data
- Each layer can be tested independently

## Test Structure

```
apps/orders/tests/
├── __init__.py
├── conftest.py           # Fixtures for this app
├── factories.py          # Factory Boy factories
├── test_models.py        # Model properties and methods
├── test_services.py      # Service methods
├── test_use_cases.py     # Use case execution
└── test_views.py         # API integration tests
```

## pytest Configuration

### pyproject.toml

```toml
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = ["test_*.py", "*_test.py"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
    "-ra",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
]
filterwarnings = [
    "ignore::DeprecationWarning",
]

[tool.coverage.run]
source = ["apps"]
omit = [
    "*/migrations/*",
    "*/tests/*",
    "*/__init__.py",
    "*/admin.py",
]
branch = true

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "if TYPE_CHECKING:",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
fail_under = 80
```

## Factories (Factory Boy)

### Basic Factory

```python
# apps/orders/tests/factories.py
import factory
from factory.django import DjangoModelFactory
from uuid import uuid4

from apps.orders.models import Order, OrderItem
from apps.customers.tests.factories import CustomerFactory
from apps.products.tests.factories import ProductFactory


class OrderFactory(DjangoModelFactory):
    """Factory for Order model."""

    class Meta:
        model = Order

    id = factory.LazyFunction(uuid4)
    customer = factory.SubFactory(CustomerFactory)
    status = Order.Status.PENDING
    notes = factory.Faker('sentence')

    @factory.post_generation
    def items(self, create, extracted, **kwargs):
        """Handle items creation."""
        if not create:
            return
        if extracted:
            for item in extracted:
                OrderItemFactory(order=self, **item)


class OrderItemFactory(DjangoModelFactory):
    """Factory for OrderItem model."""

    class Meta:
        model = OrderItem

    order = factory.SubFactory(OrderFactory)
    product = factory.SubFactory(ProductFactory)
    quantity = factory.Faker('random_int', min=1, max=10)
    unit_price = factory.Faker(
        'pydecimal',
        left_digits=3,
        right_digits=2,
        positive=True,
    )
```

### Factory with Traits

```python
class OrderFactory(DjangoModelFactory):
    class Meta:
        model = Order

    customer = factory.SubFactory(CustomerFactory)
    status = Order.Status.PENDING

    class Params:
        # Traits for common scenarios
        confirmed = factory.Trait(
            status=Order.Status.CONFIRMED,
            confirmed_at=factory.LazyFunction(timezone.now),
        )
        shipped = factory.Trait(
            status=Order.Status.SHIPPED,
            shipped_at=factory.LazyFunction(timezone.now),
            tracking_number=factory.Faker('uuid4'),
        )
        cancelled = factory.Trait(
            status=Order.Status.CANCELLED,
            cancellation_reason='Customer request',
        )
        with_items = factory.Trait(
            items=factory.RelatedFactoryList(
                OrderItemFactory,
                factory_related_name='order',
                size=3,
            ),
        )


# Usage
order = OrderFactory()  # Pending order
order = OrderFactory(confirmed=True)  # Confirmed order
order = OrderFactory(shipped=True)  # Shipped order
order = OrderFactory(with_items=True)  # Order with 3 items
```

## Testing Use Cases

### Basic Use Case Test

```python
# apps/orders/tests/test_use_cases.py
import pytest
from unittest.mock import Mock, patch
from uuid import uuid4
from decimal import Decimal

from apps.orders.use_cases import (
    CreateOrderUseCase,
    CreateOrderInput,
    OrderItemInput,
)
from apps.orders.services import OrderService, InventoryService


class TestCreateOrderUseCase:
    """Tests for CreateOrderUseCase."""

    def test_create_order_success(self):
        """Should create order when inventory is available."""
        # Arrange
        mock_order_service = Mock(spec=OrderService)
        mock_inventory_service = Mock(spec=InventoryService)

        mock_inventory_service.validate_availability.return_value = (True, "")
        mock_order = Mock(id=uuid4())
        mock_order_service.create.return_value = mock_order

        use_case = CreateOrderUseCase(
            order_service=mock_order_service,
            inventory_service=mock_inventory_service,
        )

        input_dto = CreateOrderInput(
            customer_id=uuid4(),
            items=[
                OrderItemInput(product_id=uuid4(), quantity=2),
            ],
        )

        # Act
        output = use_case.execute(input_dto)

        # Assert
        assert output.success is True
        assert output.order_id == mock_order.id
        mock_order_service.create.assert_called_once()

    def test_create_order_fails_when_inventory_unavailable(self):
        """Should fail when inventory is not available."""
        # Arrange
        mock_order_service = Mock(spec=OrderService)
        mock_inventory_service = Mock(spec=InventoryService)

        mock_inventory_service.validate_availability.return_value = (
            False,
            "Product out of stock",
        )

        use_case = CreateOrderUseCase(
            order_service=mock_order_service,
            inventory_service=mock_inventory_service,
        )

        input_dto = CreateOrderInput(
            customer_id=uuid4(),
            items=[OrderItemInput(product_id=uuid4(), quantity=100)],
        )

        # Act
        output = use_case.execute(input_dto)

        # Assert
        assert output.success is False
        assert output.error == "Product out of stock"
        mock_order_service.create.assert_not_called()

    def test_create_order_with_notes(self):
        """Should pass notes to order service."""
        # Arrange
        mock_order_service = Mock(spec=OrderService)
        mock_inventory_service = Mock(spec=InventoryService)
        mock_inventory_service.validate_availability.return_value = (True, "")
        mock_order_service.create.return_value = Mock(id=uuid4())

        use_case = CreateOrderUseCase(
            order_service=mock_order_service,
            inventory_service=mock_inventory_service,
        )

        input_dto = CreateOrderInput(
            customer_id=uuid4(),
            items=[OrderItemInput(product_id=uuid4(), quantity=1)],
            notes="Please gift wrap",
        )

        # Act
        use_case.execute(input_dto)

        # Assert
        call_kwargs = mock_order_service.create.call_args.kwargs
        assert call_kwargs['notes'] == "Please gift wrap"
```

### Testing Use Case with Database

```python
@pytest.mark.django_db
class TestCreateOrderUseCaseIntegration:
    """Integration tests for CreateOrderUseCase with real database."""

    def test_creates_order_in_database(self):
        """Should persist order to database."""
        # Arrange
        customer = CustomerFactory()
        product = ProductFactory(stock=10)

        use_case = CreateOrderUseCase(
            order_service=OrderService(),
            inventory_service=InventoryService(),
        )

        input_dto = CreateOrderInput(
            customer_id=customer.id,
            items=[
                OrderItemInput(
                    product_id=product.id,
                    quantity=2,
                    unit_price=Decimal('10.00'),
                ),
            ],
        )

        # Act
        output = use_case.execute(input_dto)

        # Assert
        assert output.success is True
        order = Order.objects.get(id=output.order_id)
        assert order.customer == customer
        assert order.items.count() == 1
```

## Testing Services

### Query Methods (Static)

```python
# apps/orders/tests/test_services.py
import pytest
from apps.orders.services import OrderService
from apps.orders.tests.factories import OrderFactory


@pytest.mark.django_db
class TestOrderServiceQueries:
    """Tests for OrderService query methods."""

    def test_get_by_id_returns_order(self):
        """Should return order when exists."""
        order = OrderFactory()

        result = OrderService.get_by_id(order.id)

        assert result == order

    def test_get_by_id_returns_none_when_not_found(self):
        """Should return None when order doesn't exist."""
        from uuid import uuid4

        result = OrderService.get_by_id(uuid4())

        assert result is None

    def test_list_for_customer_filters_by_customer(self):
        """Should only return orders for specified customer."""
        customer = CustomerFactory()
        order1 = OrderFactory(customer=customer)
        order2 = OrderFactory(customer=customer)
        OrderFactory()  # Different customer

        result = list(OrderService.list_for_customer(customer.id))

        assert len(result) == 2
        assert order1 in result
        assert order2 in result

    def test_list_for_customer_filters_by_status(self):
        """Should filter by status when provided."""
        customer = CustomerFactory()
        pending_order = OrderFactory(customer=customer, status=Order.Status.PENDING)
        OrderFactory(customer=customer, status=Order.Status.SHIPPED)

        result = list(OrderService.list_for_customer(
            customer.id,
            status=Order.Status.PENDING,
        ))

        assert len(result) == 1
        assert result[0] == pending_order
```

### Mutation Methods (Instance)

```python
@pytest.mark.django_db
class TestOrderServiceMutations:
    """Tests for OrderService mutation methods."""

    def test_create_creates_order_with_items(self):
        """Should create order and items."""
        customer = CustomerFactory()
        product = ProductFactory()
        service = OrderService()

        order = service.create(
            customer_id=customer.id,
            items=[
                OrderItemInput(
                    product_id=product.id,
                    quantity=2,
                    unit_price=Decimal('10.00'),
                ),
            ],
        )

        assert order.customer == customer
        assert order.items.count() == 1
        item = order.items.first()
        assert item.product == product
        assert item.quantity == 2

    def test_update_status_changes_status(self):
        """Should update order status."""
        order = OrderFactory(status=Order.Status.PENDING)
        service = OrderService()

        updated = service.update_status(order, Order.Status.CONFIRMED)

        assert updated.status == Order.Status.CONFIRMED
        order.refresh_from_db()
        assert order.status == Order.Status.CONFIRMED
```

### Validation Methods

```python
@pytest.mark.django_db
class TestOrderServiceValidations:
    """Tests for OrderService validation methods."""

    def test_validate_can_cancel_pending_order(self):
        """Should allow cancelling pending orders."""
        order = OrderFactory(status=Order.Status.PENDING)
        service = OrderService()

        is_valid, error = service.validate_can_cancel(order)

        assert is_valid is True
        assert error == ""

    def test_validate_cannot_cancel_shipped_order(self):
        """Should not allow cancelling shipped orders."""
        order = OrderFactory(status=Order.Status.SHIPPED)
        service = OrderService()

        is_valid, error = service.validate_can_cancel(order)

        assert is_valid is False
        assert "shipped" in error.lower()

    @pytest.mark.parametrize("status,expected", [
        (Order.Status.PENDING, True),
        (Order.Status.CONFIRMED, True),
        (Order.Status.SHIPPED, False),
        (Order.Status.DELIVERED, False),
        (Order.Status.CANCELLED, False),
    ])
    def test_validate_can_cancel_by_status(self, status, expected):
        """Should validate based on status."""
        order = OrderFactory(status=status)
        service = OrderService()

        is_valid, _ = service.validate_can_cancel(order)

        assert is_valid is expected
```

## Testing Views (Integration)

### API Client Setup

```python
# apps/orders/tests/conftest.py
import pytest
from rest_framework.test import APIClient


@pytest.fixture
def api_client():
    """Return DRF API client."""
    return APIClient()


@pytest.fixture
def authenticated_client(api_client):
    """Return authenticated API client."""
    def _authenticated_client(user):
        api_client.force_authenticate(user=user)
        return api_client
    return _authenticated_client
```

### View Tests

```python
# apps/orders/tests/test_views.py
import pytest
from rest_framework import status
from uuid import uuid4

from apps.orders.models import Order
from apps.orders.tests.factories import OrderFactory
from apps.users.tests.factories import UserFactory
from apps.customers.tests.factories import CustomerFactory


@pytest.mark.django_db
class TestCreateOrderView:
    """Tests for order creation endpoint."""

    def test_create_order_success(self, authenticated_client):
        """POST /api/orders/ should create order."""
        user = UserFactory()
        customer = CustomerFactory(user=user)
        product = ProductFactory(stock=10)
        client = authenticated_client(user)

        response = client.post('/api/orders/', {
            'items': [
                {'product_id': str(product.id), 'quantity': 2}
            ],
            'notes': 'Test order',
        }, format='json')

        assert response.status_code == status.HTTP_201_CREATED
        assert 'order_id' in response.data
        order = Order.objects.get(id=response.data['order_id'])
        assert order.customer == customer

    def test_create_order_requires_auth(self, api_client):
        """POST /api/orders/ should require authentication."""
        response = api_client.post('/api/orders/', {
            'items': [{'product_id': str(uuid4()), 'quantity': 1}]
        }, format='json')

        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_create_order_validates_items(self, authenticated_client):
        """POST /api/orders/ should validate items."""
        user = UserFactory()
        CustomerFactory(user=user)
        client = authenticated_client(user)

        response = client.post('/api/orders/', {
            'items': [],  # Empty items
        }, format='json')

        assert response.status_code == status.HTTP_400_BAD_REQUEST


@pytest.mark.django_db
class TestOrderDetailView:
    """Tests for order detail endpoint."""

    def test_get_order_detail(self, authenticated_client):
        """GET /api/orders/{id}/ should return order."""
        user = UserFactory()
        customer = CustomerFactory(user=user)
        order = OrderFactory(customer=customer, with_items=True)
        client = authenticated_client(user)

        response = client.get(f'/api/orders/{order.id}/')

        assert response.status_code == status.HTTP_200_OK
        assert response.data['id'] == str(order.id)
        assert 'items' in response.data

    def test_cannot_access_other_customer_order(self, authenticated_client):
        """GET /api/orders/{id}/ should not return other's orders."""
        user = UserFactory()
        CustomerFactory(user=user)
        other_order = OrderFactory()  # Different customer
        client = authenticated_client(user)

        response = client.get(f'/api/orders/{other_order.id}/')

        assert response.status_code == status.HTTP_404_NOT_FOUND


@pytest.mark.django_db
class TestCancelOrderView:
    """Tests for order cancellation endpoint."""

    def test_cancel_pending_order(self, authenticated_client):
        """POST /api/orders/{id}/cancel/ should cancel order."""
        user = UserFactory()
        customer = CustomerFactory(user=user)
        order = OrderFactory(customer=customer, status=Order.Status.PENDING)
        client = authenticated_client(user)

        response = client.post(f'/api/orders/{order.id}/cancel/')

        assert response.status_code == status.HTTP_200_OK
        order.refresh_from_db()
        assert order.status == Order.Status.CANCELLED

    def test_cannot_cancel_shipped_order(self, authenticated_client):
        """POST /api/orders/{id}/cancel/ should fail for shipped."""
        user = UserFactory()
        customer = CustomerFactory(user=user)
        order = OrderFactory(customer=customer, status=Order.Status.SHIPPED)
        client = authenticated_client(user)

        response = client.post(f'/api/orders/{order.id}/cancel/')

        assert response.status_code == status.HTTP_400_BAD_REQUEST
        assert 'error' in response.data
```

## Testing Models

```python
# apps/orders/tests/test_models.py
import pytest
from decimal import Decimal
from apps.orders.tests.factories import OrderFactory, OrderItemFactory


@pytest.mark.django_db
class TestOrderModel:
    """Tests for Order model properties."""

    def test_is_cancellable_when_pending(self):
        """Pending orders should be cancellable."""
        order = OrderFactory(status=Order.Status.PENDING)
        assert order.is_cancellable is True

    def test_is_cancellable_when_shipped(self):
        """Shipped orders should not be cancellable."""
        order = OrderFactory(status=Order.Status.SHIPPED)
        assert order.is_cancellable is False

    def test_total_calculates_from_items(self):
        """Total should sum item subtotals."""
        order = OrderFactory()
        OrderItemFactory(order=order, quantity=2, unit_price=Decimal('10.00'))
        OrderItemFactory(order=order, quantity=1, unit_price=Decimal('5.00'))

        assert order.total == Decimal('25.00')

    def test_total_returns_zero_for_empty_order(self):
        """Total should be zero when no items."""
        order = OrderFactory()
        assert order.total == Decimal('0.00')

    def test_str_representation(self):
        """String should include order ID and customer."""
        order = OrderFactory()
        result = str(order)
        assert str(order.id) in result
```

## Fixtures (conftest.py)

### App-Level Fixtures

```python
# apps/orders/tests/conftest.py
import pytest
from apps.orders.tests.factories import OrderFactory, OrderItemFactory
from apps.customers.tests.factories import CustomerFactory
from apps.products.tests.factories import ProductFactory


@pytest.fixture
def customer():
    """Create a customer for tests."""
    return CustomerFactory()


@pytest.fixture
def product():
    """Create a product for tests."""
    return ProductFactory(stock=100)


@pytest.fixture
def pending_order(customer):
    """Create a pending order."""
    return OrderFactory(customer=customer, status=Order.Status.PENDING)


@pytest.fixture
def order_with_items(customer):
    """Create an order with items."""
    order = OrderFactory(customer=customer)
    OrderItemFactory.create_batch(3, order=order)
    return order
```

### Project-Level Fixtures

```python
# tests/conftest.py
import pytest
from rest_framework.test import APIClient


@pytest.fixture
def api_client():
    """DRF API client."""
    return APIClient()


@pytest.fixture
def authenticated_client(api_client):
    """Factory for authenticated client."""
    def _make_authenticated(user):
        api_client.force_authenticate(user=user)
        return api_client
    return _make_authenticated


@pytest.fixture(autouse=True)
def enable_db_access_for_all_tests(db):
    """Enable database access for all tests."""
    pass
```

## Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=apps --cov-report=html

# Run specific test file
pytest apps/orders/tests/test_use_cases.py

# Run specific test class
pytest apps/orders/tests/test_use_cases.py::TestCreateOrderUseCase

# Run specific test
pytest apps/orders/tests/test_use_cases.py::TestCreateOrderUseCase::test_create_order_success

# Run with verbose output
pytest -v

# Run only fast tests (exclude slow)
pytest -m "not slow"

# Run integration tests only
pytest -m integration
```

## Best Practices

### Test Naming

```python
def test_create_order_success():  # What happens
def test_create_order_fails_when_inventory_unavailable():  # Condition
def test_cannot_cancel_shipped_order():  # Negative case
```

### Arrange-Act-Assert

```python
def test_something():
    # Arrange - Set up test data
    order = OrderFactory()
    service = OrderService()

    # Act - Execute the code under test
    result = service.cancel(order)

    # Assert - Verify the outcome
    assert result.status == Order.Status.CANCELLED
```

### One Assert Per Test (When Possible)

```python
# Multiple related assertions are OK
def test_create_order_creates_with_items():
    output = use_case.execute(input_dto)

    assert output.success is True
    assert output.order_id is not None
    order = Order.objects.get(id=output.order_id)
    assert order.items.count() == 2
```

### Use Parametrize for Multiple Scenarios

```python
@pytest.mark.parametrize("status,expected_cancellable", [
    (Order.Status.PENDING, True),
    (Order.Status.CONFIRMED, True),
    (Order.Status.SHIPPED, False),
    (Order.Status.DELIVERED, False),
])
def test_is_cancellable_by_status(status, expected_cancellable):
    order = OrderFactory(status=status)
    assert order.is_cancellable is expected_cancellable
```
