# Google Merchant Center Feed Requirements

## Feed Format

- Protocol: RSS 2.0
- Namespace: `xmlns:g="http://base.google.com/ns/1.0"`
- Content-Type: `application/xml` or `text/xml`
- Encoding: UTF-8
- Max items per file: 5,000,000 (keep under 50MB for reliability)

## Channel Structure

```xml
<rss xmlns:g="http://base.google.com/ns/1.0" version="2.0">
  <channel>
    <title>Store Name - Products</title>
    <link>https://your-frontend-domain.com</link>  <!-- NOT the API domain -->
    <description>Product feed description</description>
    <item>...</item>
  </channel>
</rss>
```

## Required Fields

### `g:id`
- Unique identifier for the product
- Max 50 characters
- Must be stable (don't change after submission)
- Use product code/SKU, not auto-increment ID if it can change
- **Valid:** `"10549"`, `"SKU-ABC-001"`
- **Invalid:** empty, null, non-unique values

### `g:title`
- Product name shown in Google Shopping
- Max 150 characters
- No HTML tags
- No ALL CAPS — Google may penalize titles in all caps
- Normalize to Title Case: `"PAÑAL HUGGIES T1"` → `"Pañal Huggies T1"`
- **Python:** `name[:150].title()`
- **JS:** Use a title-case library or custom function

### `g:description`
- Detailed product description
- Max 5000 characters
- No HTML tags
- Fallback to title if empty (acceptable)
- **Python:** `(product.description or product.name)[:5000]`

### `g:link`
- Full HTTPS URL to the product page on the **frontend**
- Must use the canonical domain (apex, not www if apex is canonical)
- **Valid:** `https://ddisrl.com.ar/product/pañal-huggies-t1`
- **Invalid:** `https://api.yoursite.com/product/123`, `http://...`, `/product/slug`
- Always use a config variable, never hardcode the domain

### `g:image_link`
- Full HTTPS URL to the main product image
- Minimum 100x100px (800x800px recommended)
- Formats: JPEG, PNG, GIF, BMP, TIFF, WebP
- **Never emit an empty string** — Google rejects `<g:image_link></g:image_link>`
- Always provide a fallback image URL from your CDN/static files

```python
# Python pattern
image_url = product.get_thumbnail() or ''
if not image_url.startswith('http'):
    image_url = f'{settings.DOMAIN}{image_url}'
if not image_url:
    image_url = f'{settings.DOMAIN}/media/default_300.png'
```

### `g:availability`
Exact string values only:
- `in stock` — available to buy
- `out of stock` — not available
- `preorder` — available in the future
- `backorder` — out of stock but can be ordered

```python
# Python pattern
total_stock = sum([
    product.quantity or 0,
    product.quantity2 or 0,
    product.quantity3 or 0,
    product.quantity4 or 0,
])
availability = 'in stock' if total_stock > 0 else 'out of stock'
```

### `g:price`
- Format: `{amount:.2f} {ISO 4217 currency code}`
- Use dot as decimal separator (not comma)
- Include IVA/taxes in the price (required for most countries)
- **Valid:** `1500.00 ARS`, `29.99 USD`, `19.99 EUR`
- **Invalid:** `1500,00 ARS`, `$1500`, `1500 ars`

### `g:condition`
Exact string values only:
- `new` — brand new, never used
- `used` — previously used
- `refurbished` — professionally restored

## Recommended Fields

### `g:brand`
- Manufacturer or brand name
- Only include if the value exists — skip null/empty values
- **Valid:** `Huggies`, `Colgate`, `Unilever`

### `g:gtin`
Global Trade Item Number (barcode). Strict validation required:

```python
# Python validation
def is_valid_gtin(value: str) -> bool:
    v = value.strip()
    return v.isdigit() and len(v) in (8, 12, 13, 14)

if is_valid_gtin(product.cod_barras):
    # include g:gtin
```

- **Valid lengths:** 8 (EAN-8), 12 (UPC-A), 13 (EAN-13), 14 (ITF-14)
- **Invalid:** `"0"`, `"000000000000"`, non-numeric strings, wrong length
- Google verifies GTIN against global databases — invalid GTINs cause item rejection

### `g:mpn`
- Manufacturer Part Number / internal SKU
- Use when GTIN is not available
- Usually the product's internal code

### `g:google_product_category`
- Numeric ID from [Google Product Taxonomy](https://www.google.com/basepages/producttype/taxonomy-with-ids.en-US.txt)
- Strongly recommended — improves automatic classification in Shopping
- Use a mapping dict keyed on your local category names

**Common IDs:**
```
436   Health & Beauty (generic fallback)
491   Health & Beauty > Personal Care
5813  Health & Beauty > Oral Care
2974  Health & Beauty > Hair Care
2975  Health & Beauty > Hair Care > Hair Color
2571  Health & Beauty > Cosmetics
543   Baby & Toddler > Diapering
537   Baby & Toddler
623   Home & Garden > Household Supplies > Household Cleaning Supplies
2169  Home & Garden > Household Supplies > Laundry Supplies
422   Food, Beverages & Tobacco > Food Items
413   Food, Beverages & Tobacco > Beverages
1     Animals & Pet Supplies
```

### `g:product_type`
- Your own category path (not Google's taxonomy)
- Free-form text, max 750 chars
- Example: `Perfumería > Cuidado Bucal`

### `g:sale_price`
- Only include when the product has a **different** promotional price
- Must be lower than `g:price`
- Same format as `g:price`
- If both values are equal, omit `g:sale_price`

## Data Quality Rules (Queryset Level)

Filter these out **before** generating the feed:

```python
# Django
products = Product.objects.filter(
    is_active=True,
    price__gt=0,
).exclude(
    name='',
    name__isnull=True,
    slug='',
    slug__isnull=True,
)
```

```js
// Mongoose / Node.js
const products = await Product.find({
  isActive: true,
  price: { $gt: 0 },
  name: { $ne: '', $exists: true },
  slug: { $ne: '', $exists: true },
});
```

## Validation Checklist Before Submitting to GMC

- [ ] Feed URL accessible via HTTPS without authentication
- [ ] Content-Type is `application/xml`
- [ ] All required fields present for every item
- [ ] No empty `g:image_link` tags
- [ ] GTINs are numeric and correct length
- [ ] `g:availability` uses exact allowed values
- [ ] `g:price` uses dot decimal and ISO currency code
- [ ] `g:link` points to the frontend canonical domain
- [ ] Channel `<link>` points to the frontend (not API)
- [ ] Products with empty names or zero prices excluded
- [ ] Test with Google's [Product Data Specification](https://support.google.com/merchants/answer/7052112)
