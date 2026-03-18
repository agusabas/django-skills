---
name: google-merchant-feed
description: >
  Generate a valid Google Merchant Center product feed (RSS 2.0 / XML) from any backend.
  Use this skill when the user needs to create or fix a Google Merchant feed, product XML feed,
  Google Shopping feed, or integrate products with Google Merchant Center.
  Covers all required and recommended fields, validation rules, common pitfalls,
  and backend-specific implementations for Django/Python, Node.js, and Next.js API routes.
---

## Overview

This skill guides the creation of a valid Google Merchant Center product feed (XML/RSS 2.0)
from any backend stack. It encodes lessons learned from production implementations and
Google's official specification.

**Supported backends:**
- Django / Python (ElementTree or lxml)
- Node.js / Express (xmlbuilder2)
- Next.js API Routes (App Router or Pages Router)

## When to Use

- User wants to create a Google Merchant / Google Shopping product feed
- User has a feed that Google Merchant Center rejects with validation errors
- User asks about `g:availability`, `g:price`, `g:gtin`, `g:google_product_category`
- User needs to expose products to Google Shopping ads
- User asks "how do I send products to Google?"

## Instructions

### Step 1 — Identify the backend

Ask or detect which backend the user is working with:
- **Django/Python** → load `references/django-python.md`
- **Node.js/Express** → load `references/nodejs.md`
- **Next.js** → load `references/nextjs.md`

### Step 2 — Load requirements first

Always read `references/google-requirements.md` before any implementation.
It contains field-by-field validation rules that apply to all backends.

### Step 3 — Apply the complete checklist

Before delivering any implementation, verify against this checklist:

**Queryset / data source:**
- [ ] Filter `is_active=True` (or equivalent)
- [ ] Exclude products with empty name
- [ ] Exclude products with `price <= 0`
- [ ] Exclude products with null or empty slug/URL
- [ ] Limit to a safe maximum (5000 items) to avoid memory issues

**Required fields — all must be present and non-empty:**
- [ ] `g:id` — stable unique identifier
- [ ] `g:title` — Title Case, max 150 chars, no HTML
- [ ] `g:description` — max 5000 chars, fallback to title if empty
- [ ] `g:link` — absolute HTTPS URL using the canonical frontend domain
- [ ] `g:image_link` — absolute HTTPS URL; fallback to default image if empty, never empty string
- [ ] `g:availability` — exactly `in stock` or `out of stock`
- [ ] `g:price` — format `{number:.2f} {ISO-currency}` e.g. `1500.00 ARS`
- [ ] `g:condition` — exactly `new`, `used`, or `refurbished`

**Recommended fields:**
- [ ] `g:brand` — only if present, skip if null
- [ ] `g:gtin` — validate: numeric only, length in (8, 12, 13, 14); skip if invalid
- [ ] `g:mpn` — product code/SKU
- [ ] `g:google_product_category` — numeric ID from Google taxonomy
- [ ] `g:product_type` — local category name

**Channel metadata:**
- [ ] `<link>` points to the **frontend** domain (not the API domain)
- [ ] `xmlns:g="http://base.google.com/ns/1.0"` namespace present
- [ ] Content-Type: `application/xml`

**Common pitfalls to avoid:**
- `g:image_link` with empty string — Google rejects it; always use a fallback image URL
- `g:gtin` with `"0"`, `"000000000"` or non-numeric strings — validate strictly
- Titles in ALL CAPS — normalize to Title Case with `.title()` or equivalent
- `g:link` pointing to the API domain instead of the frontend
- `g:sale_price` with same value as `g:price` — only emit if actually different
- Hardcoded domain strings — use a config variable for the frontend URL

## Reference Files

| File | Content |
|------|---------|
| `references/google-requirements.md` | Full field spec, validation rules, Google taxonomy tips |
| `references/django-python.md` | Complete Django view using ElementTree |
| `references/nodejs.md` | Express route using xmlbuilder2 |
| `references/nextjs.md` | Next.js App Router API route |
