# Google Merchant Feed — Next.js API Route Implementation

## App Router (Next.js 13+)

```ts
// app/api/merchant-feed/route.ts
import { NextResponse } from 'next/server';

const FRONTEND_URL = process.env.NEXT_PUBLIC_SITE_URL || 'https://yourdomain.com';
const DEFAULT_IMAGE = `${FRONTEND_URL}/images/default-product.png`;

const GOOGLE_CATEGORY_MAP: Record<string, string> = {
  'personal care': '491',
  'oral care': '5813',
  'hair color': '2975',
  'hair care': '2974',
  'cosmetics': '2571',
  'diaper': '543',
  'baby': '537',
  'cleaning': '623',
  'laundry': '2169',
  'food': '422',
  'beverage': '413',
  'pet': '1',
  default: '436',
};

function resolveGoogleCategory(categoryName = ''): string {
  const lower = categoryName.toLowerCase();
  const match = Object.keys(GOOGLE_CATEGORY_MAP).find(
    k => k !== 'default' && lower.includes(k)
  );
  return GOOGLE_CATEGORY_MAP[match ?? 'default'];
}

function isValidGtin(value?: string | null): boolean {
  if (!value) return false;
  const v = value.trim();
  return /^\d+$/.test(v) && [8, 12, 13, 14].includes(v.length);
}

function toTitleCase(str: string): string {
  return str.replace(/\w\S*/g, txt =>
    txt.charAt(0).toUpperCase() + txt.slice(1).toLowerCase()
  );
}

function escapeXml(str: string): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&apos;');
}

export async function GET() {
  // 1. Fetch products from your data source (DB, API, CMS...)
  const products = await fetchProducts(); // implement this

  const items = products
    .filter(p => p.isActive && p.price > 0 && p.name && p.slug)
    .slice(0, 5000)
    .map(product => {
      const imageUrl = product.imageUrl?.startsWith('http')
        ? product.imageUrl
        : product.imageUrl
          ? `${FRONTEND_URL}${product.imageUrl}`
          : DEFAULT_IMAGE;

      const price = product.price.toFixed(2);
      const comparePrice = product.comparePrice ?? 0;
      const hasPromo = comparePrice > product.price;

      const priceTag = hasPromo
        ? `<g:price>${escapeXml(comparePrice.toFixed(2))} ARS</g:price>
           <g:sale_price>${escapeXml(price)} ARS</g:sale_price>`
        : `<g:price>${escapeXml(price)} ARS</g:price>`;

      const gtinTag = isValidGtin(product.barcode)
        ? `<g:gtin>${escapeXml(product.barcode!.trim())}</g:gtin>`
        : '';

      const brandTag = product.brand
        ? `<g:brand>${escapeXml(product.brand)}</g:brand>`
        : '';

      const categoryTag = product.category
        ? `<g:product_type>${escapeXml(product.category)}</g:product_type>`
        : '';

      return `
    <item>
      <g:id>${escapeXml(String(product.sku || product.id))}</g:id>
      <g:title>${escapeXml(toTitleCase(product.name.slice(0, 150)))}</g:title>
      <g:description>${escapeXml((product.description || product.name).slice(0, 5000))}</g:description>
      <g:link>${escapeXml(`${FRONTEND_URL}/product/${product.slug}`)}</g:link>
      <g:image_link>${escapeXml(imageUrl)}</g:image_link>
      <g:availability>${product.stock > 0 ? 'in stock' : 'out of stock'}</g:availability>
      ${priceTag}
      <g:condition>new</g:condition>
      ${brandTag}
      <g:google_product_category>${resolveGoogleCategory(product.category)}</g:google_product_category>
      ${categoryTag}
      ${gtinTag}
      <g:mpn>${escapeXml(String(product.sku || product.id))}</g:mpn>
    </item>`;
    }).join('');

  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<rss xmlns:g="http://base.google.com/ns/1.0" version="2.0">
  <channel>
    <title>Your Store - Products</title>
    <link>${FRONTEND_URL}</link>
    <description>Product feed</description>
    ${items}
  </channel>
</rss>`;

  return new NextResponse(xml, {
    headers: { 'Content-Type': 'application/xml; charset=utf-8' },
  });
}
```

## Pages Router (Next.js 12 and below)

```ts
// pages/api/merchant-feed.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'GET') return res.status(405).end();

  // same logic as above — build xml string
  const xml = buildFeedXml();

  res.setHeader('Content-Type', 'application/xml; charset=utf-8');
  res.status(200).send(xml);
}
```

## Data Interface

```ts
interface MerchantProduct {
  id: string | number;
  sku?: string;
  name: string;
  description?: string;
  slug: string;
  imageUrl?: string;
  price: number;         // must include taxes (IVA)
  comparePrice?: number; // original price if on sale
  stock: number;
  isActive: boolean;
  brand?: string;
  category?: string;
  barcode?: string;      // EAN/GTIN
}

async function fetchProducts(): Promise<MerchantProduct[]> {
  // Implement: fetch from your DB, external API, or CMS
  // Ensure you filter is_active and price > 0 at the source
}
```

## Key Notes

- Always use `escapeXml()` — unescaped `&` in product names will break XML parsing
- `NEXT_PUBLIC_SITE_URL` should be your canonical frontend domain (apex, not www)
- Cache the response with `revalidate` if products don't change often:
  ```ts
  // App Router: add to the route file
  export const revalidate = 3600; // 1 hour
  ```
- For large catalogs (>1000 products), consider streaming with `ReadableStream`
