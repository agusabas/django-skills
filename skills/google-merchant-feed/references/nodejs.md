# Google Merchant Feed — Node.js / Express Implementation

## Dependencies

```bash
npm install xmlbuilder2
```

## Complete Route

```js
// routes/merchant-feed.js
const { create } = require('xmlbuilder2');
const express = require('express');
const router = express.Router();
const Product = require('../models/product');

const FRONTEND_URL = process.env.FRONTEND_URL || 'https://yourdomain.com';
const STATIC_URL = process.env.STATIC_URL || 'https://yourdomain.com';
const DEFAULT_IMAGE = `${STATIC_URL}/media/default_300.png`;

const GOOGLE_CATEGORY_MAP = {
  'perfumeria': '491',
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
  'default': '436',
};

function resolveGoogleCategory(categoryName = '') {
  const lower = categoryName.toLowerCase();
  const match = Object.keys(GOOGLE_CATEGORY_MAP).find(
    k => k !== 'default' && lower.includes(k)
  );
  return GOOGLE_CATEGORY_MAP[match || 'default'];
}

function isValidGtin(value) {
  if (!value) return false;
  const v = String(value).trim();
  return /^\d+$/.test(v) && [8, 12, 13, 14].includes(v.length);
}

function toTitleCase(str) {
  return str.replace(/\w\S*/g, txt =>
    txt.charAt(0).toUpperCase() + txt.substr(1).toLowerCase()
  );
}

router.get('/google-merchant-feed', async (req, res) => {
  try {
    // 1. Fetch valid products only
    const products = await Product.find({
      isActive: true,
      price: { $gt: 0 },
      name: { $ne: '', $exists: true },
      slug: { $ne: '', $exists: true },
    })
      .populate('category')
      .limit(5000)
      .lean();

    // 2. Build XML
    const doc = create({ version: '1.0', encoding: 'UTF-8' })
      .ele('rss', { 'xmlns:g': 'http://base.google.com/ns/1.0', version: '2.0' })
        .ele('channel')
          .ele('title').txt('Your Store - Products').up()
          .ele('link').txt(FRONTEND_URL).up()
          .ele('description').txt('Product feed').up();

    for (const product of products) {
      const item = doc.ele('item');

      // Required
      item.ele('g:id').txt(String(product.sku || product._id));
      item.ele('g:title').txt(toTitleCase(product.name.slice(0, 150)));
      item.ele('g:description').txt(
        (product.description || product.name).slice(0, 5000)
      );
      item.ele('g:link').txt(`${FRONTEND_URL}/product/${product.slug}`);

      // Image — never empty
      const imageUrl = product.imageUrl || DEFAULT_IMAGE;
      item.ele('g:image_link').txt(
        imageUrl.startsWith('http') ? imageUrl : `${STATIC_URL}${imageUrl}`
      );

      // Availability
      const stock = (product.stock || 0);
      item.ele('g:availability').txt(stock > 0 ? 'in stock' : 'out of stock');

      // Price — dot decimal, ISO currency
      const price = Number(product.price).toFixed(2);
      item.ele('g:price').txt(`${price} ARS`);

      // Sale price — only if different
      const compare = Number(product.comparePrice || 0);
      if (compare && compare > Number(price)) {
        item.ele('g:price').txt(`${compare.toFixed(2)} ARS`);
        item.ele('g:sale_price').txt(`${price} ARS`);
      }

      item.ele('g:condition').txt('new');

      // Recommended
      if (product.brand) item.ele('g:brand').txt(product.brand);

      const googleCat = resolveGoogleCategory(product.category?.name);
      item.ele('g:google_product_category').txt(googleCat);
      if (product.category?.name) {
        item.ele('g:product_type').txt(product.category.name);
      }

      if (isValidGtin(product.barcode)) {
        item.ele('g:gtin').txt(String(product.barcode).trim());
      }

      item.ele('g:mpn').txt(String(product.sku || product._id));
    }

    const xml = doc.end({ prettyPrint: true });
    res.set('Content-Type', 'application/xml; charset=utf-8');
    res.send(xml);

  } catch (err) {
    console.error('Merchant feed error:', err);
    res.status(500).send('Feed generation failed');
  }
});

module.exports = router;
```

## Registration

```js
// app.js or server.js
const merchantFeed = require('./routes/merchant-feed');
app.use('/api', merchantFeed);
// Feed available at: /api/google-merchant-feed
```

## Key Notes

- Replace `product.sku`, `product.imageUrl`, `product.stock`, `product.barcode`
  with your actual field names
- `product.price` should already include taxes (IVA)
- Replace `ARS` with your ISO currency code
