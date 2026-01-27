# Django Admin for API Projects

## Overview

Django Admin provides a powerful interface for managing data, debugging, and administrative tasks in API projects. This guide covers admin configuration optimized for API backends.

## Basic Setup

### Register Models

```python
# apps/orders/admin.py
from django.contrib import admin
from apps.orders.models import Order, OrderItem

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'customer', 'status', 'total', 'created_at']
    list_filter = ['status', 'created_at']
    search_fields = ['id', 'customer__email', 'customer__name']
    ordering = ['-created_at']


@admin.register(OrderItem)
class OrderItemAdmin(admin.ModelAdmin):
    list_display = ['id', 'order', 'product', 'quantity', 'unit_price']
    list_filter = ['order__status']
    search_fields = ['order__id', 'product__name']
```

### URL Configuration

```python
# config/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('apps.api.urls')),
]
```

## ModelAdmin Configuration

### List Display

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = [
        'id',
        'customer_name',      # Custom method
        'status',
        'total',
        'item_count',         # Custom method
        'created_at',
    ]

    @admin.display(description='Customer')
    def customer_name(self, obj):
        return obj.customer.name

    @admin.display(description='Items')
    def item_count(self, obj):
        return obj.items.count()
```

### List Filters

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_filter = [
        'status',
        'created_at',
        ('customer', admin.RelatedOnlyFieldListFilter),  # Only show customers with orders
        'customer__tier',  # Filter by related field
    ]
```

### Custom Filters

```python
class TotalRangeFilter(admin.SimpleListFilter):
    title = 'order total'
    parameter_name = 'total_range'

    def lookups(self, request, model_admin):
        return [
            ('small', 'Under $100'),
            ('medium', '$100 - $500'),
            ('large', 'Over $500'),
        ]

    def queryset(self, request, queryset):
        if self.value() == 'small':
            return queryset.filter(total__lt=100)
        if self.value() == 'medium':
            return queryset.filter(total__gte=100, total__lt=500)
        if self.value() == 'large':
            return queryset.filter(total__gte=500)


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_filter = ['status', TotalRangeFilter]
```

### Search Configuration

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    search_fields = [
        'id',                    # Exact match on UUID
        'customer__email',       # Related field
        'customer__name',
        'items__product__name',  # Nested related field
    ]
    search_help_text = 'Search by order ID, customer email/name, or product name'
```

### Date Hierarchy

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    date_hierarchy = 'created_at'  # Adds date drill-down navigation
```

## Inline Models

### Tabular Inline

```python
class OrderItemInline(admin.TabularInline):
    model = OrderItem
    extra = 0  # Don't show empty rows
    readonly_fields = ['unit_price', 'line_total']
    fields = ['product', 'quantity', 'unit_price', 'line_total']

    @admin.display(description='Line Total')
    def line_total(self, obj):
        return obj.quantity * obj.unit_price


@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    inlines = [OrderItemInline]
```

### Stacked Inline

```python
class OrderNoteInline(admin.StackedInline):
    model = OrderNote
    extra = 0
    readonly_fields = ['created_at', 'created_by']
```

## Read-Only and Fieldsets

### Read-Only Fields

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    readonly_fields = [
        'id',
        'created_at',
        'updated_at',
        'total',  # Calculated field
    ]
```

### Fieldsets for Organization

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    fieldsets = [
        ('Order Info', {
            'fields': ['id', 'status', 'customer'],
        }),
        ('Financial', {
            'fields': ['total', 'discount', 'tax'],
        }),
        ('Shipping', {
            'fields': ['shipping_address', 'tracking_number'],
            'classes': ['collapse'],  # Collapsible section
        }),
        ('Timestamps', {
            'fields': ['created_at', 'updated_at'],
            'classes': ['collapse'],
        }),
    ]
```

## Custom Actions

### Bulk Actions

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    actions = ['mark_as_shipped', 'export_to_csv']

    @admin.action(description='Mark selected orders as shipped')
    def mark_as_shipped(self, request, queryset):
        updated = queryset.filter(status='confirmed').update(status='shipped')
        self.message_user(request, f'{updated} orders marked as shipped.')

    @admin.action(description='Export selected orders to CSV')
    def export_to_csv(self, request, queryset):
        import csv
        from django.http import HttpResponse

        response = HttpResponse(content_type='text/csv')
        response['Content-Disposition'] = 'attachment; filename="orders.csv"'

        writer = csv.writer(response)
        writer.writerow(['ID', 'Customer', 'Status', 'Total', 'Created'])

        for order in queryset.select_related('customer'):
            writer.writerow([
                str(order.id),
                order.customer.email,
                order.status,
                order.total,
                order.created_at.isoformat(),
            ])

        return response
```

## Query Optimization

### Prevent N+1 in List View

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['id', 'customer', 'status', 'item_count']
    list_select_related = ['customer']  # Optimize ForeignKey

    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.prefetch_related('items').annotate(
            _item_count=Count('items')
        )

    @admin.display(description='Items')
    def item_count(self, obj):
        return obj._item_count  # Use annotated value
```

### Optimize Change Form

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    def get_queryset(self, request):
        qs = super().get_queryset(request)
        return qs.select_related('customer', 'shipping_address')
```

## Permissions

### Restrict Access

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    def has_delete_permission(self, request, obj=None):
        # Only superusers can delete
        return request.user.is_superuser

    def has_change_permission(self, request, obj=None):
        if obj and obj.status == 'shipped':
            return False  # Can't edit shipped orders
        return super().has_change_permission(request, obj)

    def has_add_permission(self, request):
        # Orders created via API only
        return False
```

### Field-Level Permissions

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    def get_readonly_fields(self, request, obj=None):
        readonly = list(super().get_readonly_fields(request, obj))

        if not request.user.is_superuser:
            readonly.extend(['total', 'discount'])

        if obj and obj.status != 'pending':
            readonly.extend(['customer', 'shipping_address'])

        return readonly
```

## Admin Site Customization

### Custom Admin Site

```python
# apps/core/admin.py
from django.contrib.admin import AdminSite

class APIAdminSite(AdminSite):
    site_header = 'My API Administration'
    site_title = 'My API Admin'
    index_title = 'Dashboard'


admin_site = APIAdminSite(name='api_admin')
```

```python
# config/urls.py
from apps.core.admin import admin_site

urlpatterns = [
    path('admin/', admin_site.urls),
]
```

```python
# apps/orders/admin.py
from apps.core.admin import admin_site

@admin.register(Order, site=admin_site)
class OrderAdmin(admin.ModelAdmin):
    ...
```

## JSON Field Display

```python
import json
from django.utils.html import format_html

@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    readonly_fields = ['metadata_pretty']

    @admin.display(description='Metadata')
    def metadata_pretty(self, obj):
        if obj.metadata:
            formatted = json.dumps(obj.metadata, indent=2)
            return format_html('<pre>{}</pre>', formatted)
        return '-'
```

## UUID Primary Key Handling

```python
@admin.register(Order)
class OrderAdmin(admin.ModelAdmin):
    list_display = ['short_id', 'customer', 'status']

    @admin.display(description='ID')
    def short_id(self, obj):
        return str(obj.id)[:8]  # Show first 8 chars
```

## Admin Security

### Restrict Admin Access

```python
# settings.py

# Only allow admin access from specific IPs (use with reverse proxy)
ADMIN_ENABLED = env.bool('ADMIN_ENABLED', default=True)

# Custom admin URL (security through obscurity, not primary defense)
ADMIN_URL = env('ADMIN_URL', default='admin/')
```

```python
# config/urls.py
from django.conf import settings

if settings.ADMIN_ENABLED:
    urlpatterns += [
        path(settings.ADMIN_URL, admin.site.urls),
    ]
```

### Two-Factor Authentication

```bash
pip install django-two-factor-auth
```

```python
# settings.py
INSTALLED_APPS = [
    ...
    'django_otp',
    'django_otp.plugins.otp_totp',
    'two_factor',
]
```

## Admin Checklist

### Configuration
- [ ] All models registered with `@admin.register`
- [ ] `list_display` shows relevant fields
- [ ] `list_filter` for common queries
- [ ] `search_fields` for key fields
- [ ] `ordering` set appropriately
- [ ] `readonly_fields` for calculated/system fields

### Performance
- [ ] `list_select_related` for ForeignKeys in list_display
- [ ] `get_queryset` optimized for detail view
- [ ] Annotations for count/aggregations

### Security
- [ ] Delete permission restricted if needed
- [ ] Field-level permissions configured
- [ ] Admin URL not at default `/admin/` (optional)
- [ ] Two-factor auth for production
