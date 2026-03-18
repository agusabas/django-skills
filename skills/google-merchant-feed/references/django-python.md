# Google Merchant Feed — Django / Python Implementation

## Dependencies

No extra packages needed — uses Python's built-in `xml.etree.ElementTree`.

## Complete View

```python
# apps/product/views.py
from xml.etree.ElementTree import Element, SubElement, tostring
from xml.dom import minidom
from django.conf import settings
from django.http import HttpResponse
from rest_framework.views import APIView
from rest_framework.permissions import AllowAny


class GoogleMerchantFeedView(APIView):
    permission_classes = [AllowAny]

    def get(self, request):
        # --- 1. Queryset: only complete, valid products ---
        products = (
            Product.objects.filter(is_active=True, price_iva__gt=0)
            .exclude(name='')
            .exclude(name__isnull=True)
            .exclude(slug='')
            .exclude(slug__isnull=True)
            .select_related('category')
            .prefetch_related('album__images')
            [:5000]
        )

        # --- 2. Calculate prices (unauthenticated = retail/minorista) ---
        products_with_prices = calculate_list_price(list(products), request.user)

        # --- 3. Build XML ---
        rss = Element('rss', {
            'xmlns:g': 'http://base.google.com/ns/1.0',
            'version': '2.0',
        })
        channel = SubElement(rss, 'channel')
        SubElement(channel, 'title').text = 'Your Store - Products'
        SubElement(channel, 'link').text = settings.FRONTEND_URL  # e.g. https://ddisrl.com.ar
        SubElement(channel, 'description').text = 'Product feed'

        # --- 4. Google category mapping (adapt keys to your category names) ---
        GOOGLE_CATEGORY_MAP = {
            'perfumeria': '491',
            'perfumería': '491',
            'cuidado bucal': '5813',
            'cuidado personal': '491',
            'higiene': '491',
            'coloracion': '2975',
            'coloración': '2975',
            'tintura': '2975',
            'cabello': '2974',
            'maquillaje': '2571',
            'pañal': '543',
            'bebe': '537',
            'bebé': '537',
            'limpieza': '623',
            'lavanderia': '2169',
            'lavandería': '2169',
            'alimento': '422',
            'bebida': '413',
            'golosina': '422',
            'mascota': '1',
            'default': '436',
        }

        for product in products_with_prices:
            item = SubElement(channel, 'item')

            # Required fields
            SubElement(item, 'g:id').text = str(product.codigo)
            SubElement(item, 'g:title').text = product.name[:150].title()
            SubElement(item, 'g:description').text = (
                (product.description or product.name)[:5000]
            )
            SubElement(item, 'g:link').text = (
                f'{settings.FRONTEND_URL}/product/{product.slug}'
            )

            # Image — never emit empty string
            try:
                image_url = product.get_thumbnail() or ''
                if image_url and not image_url.startswith('http'):
                    image_url = f'{settings.DOMAIN}{image_url}'
            except Exception:
                image_url = ''
            if not image_url:
                image_url = f'{settings.DOMAIN}/media/default_300.png'
            SubElement(item, 'g:image_link').text = image_url

            # Availability — sum all warehouse stock
            total_stock = sum(filter(None, [
                product.quantity, product.quantity2,
                product.quantity3, product.quantity4,
            ]))
            SubElement(item, 'g:availability').text = (
                'in stock' if total_stock > 0 else 'out of stock'
            )

            # Price — dot decimal, ISO currency, including IVA
            price_value = float(getattr(product, 'price_final', None) or product.price_iva)
            SubElement(item, 'g:price').text = f'{price_value:.2f} ARS'

            # Sale price — only if actually different
            compare = float(product.compare_price or 0)
            if compare and compare > price_value:
                SubElement(item, 'g:price').text = f'{compare:.2f} ARS'
                SubElement(item, 'g:sale_price').text = f'{price_value:.2f} ARS'

            SubElement(item, 'g:condition').text = 'new'

            # Recommended fields
            if product.marca:
                SubElement(item, 'g:brand').text = product.marca

            # Google product category
            cat_name = (product.category.name if product.category else '').lower()
            google_cat = next(
                (v for k, v in GOOGLE_CATEGORY_MAP.items()
                 if k != 'default' and k in cat_name),
                GOOGLE_CATEGORY_MAP['default']
            )
            SubElement(item, 'g:google_product_category').text = google_cat
            if product.category:
                SubElement(item, 'g:product_type').text = product.category.name

            # GTIN — strict validation
            barcode = (product.cod_barras or '').strip()
            if barcode.isdigit() and len(barcode) in (8, 12, 13, 14):
                SubElement(item, 'g:gtin').text = barcode

            # MPN — always include
            SubElement(item, 'g:mpn').text = str(product.codigo)

        # --- 5. Serialize to XML ---
        xml_str = tostring(rss, encoding='unicode', xml_declaration=False)
        pretty = minidom.parseString(
            f'<?xml version="1.0" encoding="UTF-8"?>{xml_str}'
        ).toprettyxml(indent='  ', encoding='UTF-8')

        return HttpResponse(pretty, content_type='application/xml; charset=utf-8')
```

## URL Registration

```python
# apps/product/urls.py
from django.urls import path
from .views import GoogleMerchantFeedView

urlpatterns = [
    path('google-merchant-feed/', GoogleMerchantFeedView.as_view()),
]
```

## Settings Required

```python
# settings.py
FRONTEND_URL = env('FRONTEND_URL', default='https://yourdomain.com')
DOMAIN = env('DOMAIN', default='https://api.yourdomain.com')
```

## Key Notes

- `calculate_list_price` is a custom function — replace with your pricing logic
- Unauthenticated request → use retail/public prices
- `product.get_thumbnail()` — replace with your image accessor
- `product.quantity`, `quantity2`, etc. — replace with your stock fields
- `product.marca` — replace with your brand field name
- `product.cod_barras` — replace with your barcode field name
