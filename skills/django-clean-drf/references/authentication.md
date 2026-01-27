# Authentication and Permissions for Django APIs

## Overview

This guide covers authentication methods (JWT, Token, Session), permission classes, and security patterns for Django REST Framework APIs.

## Authentication Methods

### JWT Authentication (Recommended for APIs)

Using `djangorestframework-simplejwt`:

```bash
pip install djangorestframework-simplejwt
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,

    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,

    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',

    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
}
```

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### Custom JWT Claims

```python
# apps/users/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims
        token['email'] = user.email
        token['role'] = user.role
        token['permissions'] = list(user.get_all_permissions())

        return token
```

```python
# apps/users/views.py
from rest_framework_simplejwt.views import TokenObtainPairView

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

### Token Blacklisting (Logout)

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework_simplejwt.token_blacklist',
]

# Run: python manage.py migrate
```

```python
# apps/users/views.py
from rest_framework_simplejwt.tokens import RefreshToken
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated

class LogoutView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        try:
            refresh_token = request.data['refresh']
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response(status=status.HTTP_205_RESET_CONTENT)
        except Exception:
            return Response(status=status.HTTP_400_BAD_REQUEST)
```

### Token Authentication (Simpler Alternative)

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework.authtoken',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
}
```

```python
# Create token for user
from rest_framework.authtoken.models import Token

token, created = Token.objects.get_or_create(user=user)
print(token.key)  # Use this in Authorization header
```

**Header:** `Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b`

### Session Authentication (for Web Clients)

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

### Multiple Authentication Classes

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # API clients
        'rest_framework.authentication.SessionAuthentication',         # Web/Admin
    ],
}
```

## Permission Classes

### Built-in Permissions

```python
from rest_framework.permissions import (
    AllowAny,
    IsAuthenticated,
    IsAdminUser,
    IsAuthenticatedOrReadOnly,
)

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
```

### Per-Action Permissions

```python
class OrderViewSet(viewsets.ModelViewSet):
    def get_permissions(self):
        if self.action == 'list':
            return [IsAuthenticated()]
        elif self.action == 'create':
            return [IsAuthenticated()]
        elif self.action in ['update', 'partial_update', 'destroy']:
            return [IsAuthenticated(), IsOrderOwner()]
        return [IsAuthenticated()]
```

### Custom Permission Classes

```python
# apps/core/permissions.py
from rest_framework.permissions import BasePermission

class IsOrderOwner(BasePermission):
    """
    Only allow owners to access their orders.
    """
    message = 'You do not have permission to access this order.'

    def has_object_permission(self, request, view, obj):
        return obj.customer.user == request.user


class IsAdminOrReadOnly(BasePermission):
    """
    Allow read-only for everyone, write only for admins.
    """
    def has_permission(self, request, view):
        if request.method in ('GET', 'HEAD', 'OPTIONS'):
            return True
        return request.user and request.user.is_staff


class HasAPIKey(BasePermission):
    """
    Check for API key in header.
    """
    def has_permission(self, request, view):
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            return False
        return APIKey.objects.filter(key=api_key, is_active=True).exists()
```

### Role-Based Permissions

```python
# apps/users/models.py
class User(AbstractUser):
    class Role(models.TextChoices):
        ADMIN = 'admin', 'Admin'
        MANAGER = 'manager', 'Manager'
        CUSTOMER = 'customer', 'Customer'

    role = models.CharField(max_length=20, choices=Role.choices, default=Role.CUSTOMER)
```

```python
# apps/core/permissions.py
class IsManager(BasePermission):
    def has_permission(self, request, view):
        return (
            request.user.is_authenticated and
            request.user.role in [User.Role.ADMIN, User.Role.MANAGER]
        )


class HasRole(BasePermission):
    """
    Usage: permission_classes = [HasRole]
    Then set required_roles on the view.
    """
    def has_permission(self, request, view):
        required_roles = getattr(view, 'required_roles', [])
        if not required_roles:
            return True
        return (
            request.user.is_authenticated and
            request.user.role in required_roles
        )
```

```python
class OrderManagementViewSet(viewsets.ModelViewSet):
    permission_classes = [HasRole]
    required_roles = [User.Role.ADMIN, User.Role.MANAGER]
```

### Django Model Permissions

```python
from rest_framework.permissions import DjangoModelPermissions

class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [DjangoModelPermissions]
    queryset = Order.objects.all()
```

Maps HTTP methods to Django permissions:
- POST → `add_order`
- PUT/PATCH → `change_order`
- DELETE → `delete_order`

### Object-Level Permissions

```python
# apps/core/permissions.py
class IsOwnerOrAdmin(BasePermission):
    def has_permission(self, request, view):
        return request.user.is_authenticated

    def has_object_permission(self, request, view, obj):
        # Admin can access everything
        if request.user.is_staff:
            return True

        # Check ownership (customize based on your model)
        if hasattr(obj, 'user'):
            return obj.user == request.user
        if hasattr(obj, 'customer'):
            return obj.customer.user == request.user
        if hasattr(obj, 'owner'):
            return obj.owner == request.user

        return False
```

## Authentication in Clean Architecture

### Passing User to Use Cases

```python
# apps/orders/views.py
class CreateOrderView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        serializer = CreateOrderSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        use_case = CreateOrderUseCase()
        output = use_case.execute(CreateOrderInput(
            customer_id=request.user.customer.id,  # From authenticated user
            **serializer.validated_data,
        ))

        if not output.success:
            return Response({'error': output.error}, status=400)

        return Response({'order_id': str(output.order_id)}, status=201)
```

### User Context in DTOs

```python
@dataclass(frozen=True, slots=True)
class CreateOrderInput:
    customer_id: UUID
    items: list[OrderItemInput]
    # Actor info for auditing
    created_by_user_id: UUID | None = None
```

```python
class CreateOrderView(APIView):
    def post(self, request):
        use_case = CreateOrderUseCase()
        output = use_case.execute(CreateOrderInput(
            customer_id=request.user.customer.id,
            items=serializer.validated_data['items'],
            created_by_user_id=request.user.id,
        ))
```

### Permission Validation in Use Cases

```python
# apps/orders/use_cases/cancel_order.py
@dataclass(frozen=True, slots=True)
class CancelOrderInput:
    order_id: UUID
    requesting_user_id: UUID
    reason: str = ""

@dataclass(frozen=True, slots=True)
class CancelOrderOutput:
    success: bool
    error: str = ""
    error_code: str = ""

class CancelOrderUseCase:
    def __init__(
        self,
        order_service: OrderService | None = None,
        permission_service: PermissionService | None = None,
    ):
        self._order_service = order_service or OrderService()
        self._permission_service = permission_service or PermissionService()

    def execute(self, input_dto: CancelOrderInput) -> CancelOrderOutput:
        order = self._order_service.get_by_id(input_dto.order_id)
        if not order:
            return CancelOrderOutput(
                success=False,
                error="Order not found",
                error_code="NOT_FOUND",
            )

        # Permission check in use case
        can_cancel, reason = self._permission_service.can_user_cancel_order(
            user_id=input_dto.requesting_user_id,
            order=order,
        )
        if not can_cancel:
            return CancelOrderOutput(
                success=False,
                error=reason,
                error_code="PERMISSION_DENIED",
            )

        # Business validation
        is_valid, error = self._order_service.validate_can_cancel(order)
        if not is_valid:
            return CancelOrderOutput(
                success=False,
                error=error,
                error_code="INVALID_STATUS",
            )

        self._order_service.cancel(order, input_dto.reason)
        return CancelOrderOutput(success=True)
```

### Permission Service

```python
# apps/core/services/permission_service.py
class PermissionService:
    @staticmethod
    def can_user_cancel_order(user_id: UUID, order: Order) -> tuple[bool, str]:
        user = User.objects.filter(id=user_id).first()
        if not user:
            return False, "User not found"

        # Admins can cancel any order
        if user.is_staff:
            return True, ""

        # Managers can cancel any order in their region
        if user.role == User.Role.MANAGER:
            if order.customer.region == user.region:
                return True, ""
            return False, "Cannot cancel orders outside your region"

        # Customers can only cancel their own orders
        if hasattr(user, 'customer') and order.customer_id == user.customer.id:
            return True, ""

        return False, "You can only cancel your own orders"
```

## API Key Authentication

### API Key Model

```python
# apps/core/models.py
import secrets

class APIKey(models.Model):
    name = models.CharField(max_length=100)
    key = models.CharField(max_length=64, unique=True, db_index=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='api_keys')
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    last_used_at = models.DateTimeField(null=True, blank=True)
    expires_at = models.DateTimeField(null=True, blank=True)

    def save(self, *args, **kwargs):
        if not self.key:
            self.key = secrets.token_urlsafe(48)
        super().save(*args, **kwargs)

    @property
    def is_valid(self) -> bool:
        if not self.is_active:
            return False
        if self.expires_at and self.expires_at < timezone.now():
            return False
        return True
```

### API Key Authentication Class

```python
# apps/core/authentication.py
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed

class APIKeyAuthentication(BaseAuthentication):
    keyword = 'Api-Key'

    def authenticate(self, request):
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            return None  # Try other auth methods

        try:
            key = APIKey.objects.select_related('user').get(key=api_key)
        except APIKey.DoesNotExist:
            raise AuthenticationFailed('Invalid API key')

        if not key.is_valid:
            raise AuthenticationFailed('API key expired or inactive')

        # Update last used
        key.last_used_at = timezone.now()
        key.save(update_fields=['last_used_at'])

        return (key.user, key)

    def authenticate_header(self, request):
        return self.keyword
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'apps.core.authentication.APIKeyAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

## Scoped Permissions

### Scope-Based Access Control

```python
# apps/core/models.py
class APIKey(models.Model):
    SCOPE_CHOICES = [
        ('read:orders', 'Read Orders'),
        ('write:orders', 'Write Orders'),
        ('read:products', 'Read Products'),
        ('write:products', 'Write Products'),
        ('admin', 'Admin Access'),
    ]

    scopes = models.JSONField(default=list)  # ['read:orders', 'write:orders']
```

```python
# apps/core/permissions.py
class HasScope(BasePermission):
    """
    Check if API key has required scope.

    Usage:
        class MyView(APIView):
            permission_classes = [HasScope]
            required_scopes = ['read:orders']
    """
    def has_permission(self, request, view):
        required_scopes = getattr(view, 'required_scopes', [])
        if not required_scopes:
            return True

        # Get API key from authentication
        if hasattr(request, 'auth') and isinstance(request.auth, APIKey):
            api_key = request.auth
            # Admin scope grants all access
            if 'admin' in api_key.scopes:
                return True
            # Check required scopes
            return all(scope in api_key.scopes for scope in required_scopes)

        # For JWT/Session auth, check user permissions
        return request.user.is_authenticated
```

```python
class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [HasScope]

    def get_required_scopes(self):
        if self.action in ['list', 'retrieve']:
            return ['read:orders']
        return ['write:orders']

    required_scopes = property(get_required_scopes)
```

## Security Best Practices

### Password Validation

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 12},
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

### Rate Limiting Authentication Endpoints

```python
# apps/core/throttling.py
from rest_framework.throttling import AnonRateThrottle

class AuthRateThrottle(AnonRateThrottle):
    scope = 'auth'
```

```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_RATES': {
        'auth': '5/minute',  # Prevent brute force
    },
}
```

```python
# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView

class ThrottledTokenObtainPairView(TokenObtainPairView):
    throttle_classes = [AuthRateThrottle]

urlpatterns = [
    path('api/token/', ThrottledTokenObtainPairView.as_view()),
]
```

### CORS Configuration

```bash
pip install django-cors-headers
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'corsheaders',
]

MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',  # Before CommonMiddleware
    'django.middleware.common.CommonMiddleware',
    ...
]

# Development
CORS_ALLOW_ALL_ORIGINS = True  # Only for development!

# Production
CORS_ALLOWED_ORIGINS = [
    'https://app.example.com',
    'https://admin.example.com',
]

CORS_ALLOW_CREDENTIALS = True  # If using cookies

CORS_ALLOW_HEADERS = [
    'accept',
    'authorization',
    'content-type',
    'x-api-key',
]
```

### Security Headers

```python
# settings.py

# HTTPS settings
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Cookie security
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True

# Security headers
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = 'DENY'

# HSTS
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

## Authentication Checklist

### Setup
- [ ] Choose authentication method (JWT recommended for APIs)
- [ ] Configure token lifetimes appropriately
- [ ] Set up token refresh mechanism
- [ ] Implement logout/token blacklisting

### Permissions
- [ ] Default to `IsAuthenticated`
- [ ] Create custom permissions for ownership checks
- [ ] Implement role-based permissions if needed
- [ ] Use object-level permissions for fine-grained control

### Security
- [ ] Rate limit authentication endpoints
- [ ] Configure CORS properly
- [ ] Enable security headers
- [ ] Use HTTPS in production
- [ ] Implement password validation
- [ ] Log authentication attempts
