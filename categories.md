# Odoo JSON-2 API: eCommerce Categories (`product.public.category`) Integration Guide

This guide provides an end-to-end technical reference for backend developers integrating an **e-commerce platform**, **mobile app**, or **headless storefront** with Odoo's **eCommerce Public Categories (`product.public.category`)** and their relationship to products in the **Inventory / Catalog module (`product.template.public_categ_ids`)** using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's modern **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

> [!NOTE]
> **Module Requirement:** `product.public.category` and the `public_categ_ids` field on products are provided by the **eCommerce (`website_sale`)** module. If the module is not installed, requests to `/json/2/product.public.category/...` fail and `public_categ_ids` does not exist on `product.template`.

---

## Table of Contents

1. [Architecture Overview: eCommerce Categories vs. Internal Categories](#1-architecture-overview-ecommerce-categories-vs-internal-categories)
   - [The Relational Bridge: `product.template.public_categ_ids` ↔ `product.public.category`](#the-relational-bridge-producttemplatepublic_categ_ids--productpubliccategory)
   - [Core Differences: `product.public.category` vs. `product.category`](#core-differences-productpubliccategory-vs-productcategory)
   - [Hierarchy Structure](#hierarchy-structure)
   - [Multi-Website Scoping (`website_id`)](#multi-website-scoping-website_id)
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
3. [Model & Field References](#3-model--field-references)
   - [3.1 eCommerce Public Category (`product.public.category`)](#31-ecommerce-public-category-productpubliccategory)
   - [3.2 Product Template eCommerce Fields (`product.template`)](#32-product-template-ecommerce-fields-producttemplate)
   - [3.3 Product Variant eCommerce Fields (`product.product`)](#33-product-variant-ecommerce-fields-productproduct)
   - [3.4 Website (`website`)](#34-website-website)
4. [Step-by-Step API Integration Workflows](#4-step-by-step-api-integration-workflows)
   - [Step 1: Fetching the Category Tree (Mega-Menu & Store Navigation)](#step-1-fetching-the-category-tree-mega-menu--store-navigation)
   - [Step 2: Fetching Category Details, SEO & Images](#step-2-fetching-category-details-seo--images)
   - [Step 3: Multi-Website Category Filtering](#step-3-multi-website-category-filtering)
   - [Step 4: Fetching Products by eCommerce Category](#step-4-fetching-products-by-ecommerce-category)
   - [Step 5: Resolving Breadcrumbs for Product Detail Pages](#step-5-resolving-breadcrumbs-for-product-detail-pages)
   - [Step 6: Assigning Public Categories to Products](#step-6-assigning-public-categories-to-products)
   - [Step 7: Creating & Modifying Public Categories](#step-7-creating--modifying-public-categories)
5. [Complete Integration Code Reference](#5-complete-integration-code-reference)
6. [Error Handling & Edge Cases](#6-error-handling--edge-cases)
7. [Production Best Practices for E-Commerce Developers](#7-production-best-practices-for-e-commerce-developers)

---

## 1. Architecture Overview: eCommerce Categories vs. Internal Categories

Odoo maintains **two entirely separate category systems** for products. Confusing them is one of the most common pitfalls in e-commerce integrations:

1. **Internal Product Category (`product.category`)** — stored on `product.template.categ_id` (`Many2one`). Used for accounting, inventory valuation (FIFO / AVCO / Standard), stock accounts, removal strategies, and routes. **Every product has exactly one internal category.**
2. **eCommerce Public Category (`product.public.category`)** — linked through `product.template.public_categ_ids` (`Many2many`). Used for storefront navigation, menus, breadcrumbs, category landing pages, and shop filtering. **A product can belong to zero, one, or many public categories.**

### The Relational Bridge: `product.template.public_categ_ids` ↔ `product.public.category`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Inventory & Product Catalog                           │
│                             product.template                                │
│                                                                             │
│  - name: "Ultra-Light Carbon Road Bike"                                     │
│  - categ_id: 12 (Internal: "All / Saleable / Bicycles") ────────────────────┼──► product.category
│  - public_categ_ids: [101, 205, 309] ───────────────────────────┐           │   (Internal / Accounting)
└─────────────────────────────────────────────────────────────────┼───────────┘
                                                                  │ Many2many
                                                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         eCommerce / Storefront                              │
│                        product.public.category                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  • ID 101: "Bikes / Road Bikes"                                             │
│  • ID 205: "Featured / New Arrivals 2026"                                   │
│  • ID 309: "Promotions / Spring Clearance"                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

The inverse side of the relation is `product.public.category.product_tmpl_ids` (same relation table), so the link can be read from either model.

### Core Differences: `product.public.category` vs. `product.category`

| Feature | eCommerce Public Category (`product.public.category`) | Internal Category (`product.category`) |
| :--- | :--- | :--- |
| **Product Field** | `public_categ_ids` | `categ_id` |
| **Cardinality** | **Many2many** (0..n per product) | **Many2one** (exactly 1 per product) |
| **Provided By** | `website_sale` (eCommerce) | `product` (core) |
| **Audience** | Shoppers, website & app users | Accountants, warehouse, purchasing |
| **Primary Use** | Navigation, menus, breadcrumbs, shop filters | Valuation, COGS/income accounts, logistics |
| **Images** | Yes (`image_1920` … `image_128`) | No |
| **Website Scoping** | Yes (`website_id`) | No |
| **Rich Web Content** | HTML description & SEO metadata | Name only |
| **Hierarchy** | `parent_id` / `child_id` | `parent_id` / `child_id` |

### Hierarchy Structure

Public categories form a tree. Root categories have `parent_id = false`.

```
                         ┌─────────────────────────┐
                         │   Root: Electronics     │ (ID: 10, parent_id: false)
                         └────────────┬────────────┘
                 ┌────────────────────┴────────────────────┐
                 ▼                                         ▼
   ┌───────────────────────────┐             ┌───────────────────────────┐
   │    Audio & Sound (ID 15)  │             │ Computers & Tablets (ID 16)│
   │    parent_id: 10          │             │    parent_id: 10          │
   └─────────────┬─────────────┘             └─────────────┬─────────────┘
        ┌────────┴────────┐                                │
        ▼                 ▼                                ▼
 ┌──────────────┐  ┌──────────────┐                 ┌──────────────┐
 │ Headphones   │  │ Speakers     │                 │ Laptops      │
 │ (ID: 21)     │  │ (ID: 22)     │                 │ (ID: 25)     │
 └──────────────┘  └──────────────┘                 └──────────────┘
```

> [!NOTE]
> The `display_name` of a public category is its **full path** (e.g. `Electronics / Audio & Sound / Headphones`). Because Many2one values are returned as `[id, display_name]`, a `parent_id` value looks like `[15, "Electronics / Audio & Sound"]`, not just `[15, "Audio & Sound"]`.

### Multi-Website Scoping (`website_id`)

- **`website_id = false`** → the category is shared across all websites.
- **`website_id = [id, "Website Name"]`** → the category is restricted to that website.

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

### Required Headers

```http
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStorefront-Category-Engine/1.0
```

### Request & Parameter Conventions

- **Common Methods:** `search_read` (`domain`, `fields`, `offset`, `limit`, `order`), `read` (`ids`, `fields`), `search_count` (`domain`), `create` (`vals_list`), `write` (`ids`, `vals`), `unlink` (`ids`).
- **Context:** Pass a `"context"` object in the body (e.g. `{"lang": "fr_FR"}`).
- **Hierarchical Operator:** `child_of` matches a record and all its descendants, e.g. `[["public_categ_ids", "child_of", 10]]`. `parent_of` matches a record and all its ancestors.

---

## 3. Model & Field References

### 3.1 eCommerce Public Category (`product.public.category`)

Default ordering: `sequence, name, id`.

| Field Name | Type | Requirement | Default | Description & Storefront Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `Integer` | *Read-only* | — | Unique category ID. |
| `name` | `Char` | **REQUIRED** | — | Category title (translatable). |
| `display_name` | `Char` | *Computed* | — | Full path, e.g. `Electronics / Audio & Sound / Headphones`. |
| `parent_id` | `Many2one` (`product.public.category`) | Optional | `false` | Parent category. `false` = root category. |
| `child_id` | `One2many` (`product.public.category`) | *Inverse* | `[]` | Direct subcategory IDs. An empty list means the category is a leaf. |
| `parents_and_self` | `Many2many` (`product.public.category`) | *Computed* | — | Ancestor chain **ordered root → self**, including the category itself. |
| `sequence` | `Integer` | Optional | Highest existing sequence + 5 | Display order (lower first). |
| `website_id` | `Many2one` (`website`) | Optional | `false` | Restricts the category to one website. |
| `website_description` | `Html` | Optional | `false` | Rich HTML description displayed on the category page. |
| `product_tmpl_ids` | `Many2many` (`product.template`) | Optional | `[]` | Products linked to this category (inverse of `public_categ_ids`). |
| `seo_name` | `Char` | Optional | `false` | Custom URL slug (from SEO metadata mixin). |
| `website_meta_title` | `Char` | Optional | `false` | SEO `<title>`. |
| `website_meta_description` | `Text` | Optional | `false` | SEO meta description. |
| `website_meta_keywords` | `Char` | Optional | `false` | SEO keywords. |
| `image_1920` | `Image` | Optional | `false` | Category image (Base64), max 1920px. |
| `image_1024` / `image_512` / `image_256` / `image_128` | `Image` | *Computed* | — | Auto-resized versions of `image_1920`. |
| `write_date` | `Datetime` | *System* | — | Last modification (UTC). Useful for cache busting. |

> [!WARNING]
> `product.public.category` has **no `active` field** (categories cannot be archived, only deleted) and **no `has_children` field**. Requesting a non-existent field in `fields` makes the whole request fail. Derive "has children" from `child_id` being non-empty.

---

### 3.2 Product Template eCommerce Fields (`product.template`)

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `Integer` | *Read-only* | — | Product template ID. |
| `name` | `Char` | **REQUIRED** | — | Product name. |
| `public_categ_ids` | `Many2many` (`product.public.category`) | Optional | `[]` | **eCommerce categories of the product.** |
| `categ_id` | `Many2one` (`product.category`) | **REQUIRED** | Default category | Internal category (valuation/accounting only). |
| `is_published` | `Boolean` | Optional | `false` | Published on the website. **Use this field in domains.** |
| `website_id` | `Many2one` (`website`) | Optional | `false` | Restricts the product to one website. |
| `sale_ok` | `Boolean` | Optional | `true` | Can be sold. |
| `list_price` | `Float` | Optional | `1.0` | Base sales price (before pricelists — see [pricelists.md](pricelists.md)). |
| `qty_available` | `Float` | *Computed* | `0.0` | On-hand quantity summed over all variants (`stock` module). |

> [!IMPORTANT]
> **`free_qty` exists only on `product.product` (variants), not on `product.template`.** To display "free to use" stock, read it from the variants (see [Step 4.3](#43-pagination--stock-availability)).

---

### 3.3 Product Variant eCommerce Fields (`product.product`)

Variants inherit all template fields through delegation (`_inherits`), so `public_categ_ids` can be read **and searched** on `product.product` as well.

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `id` | `Integer` | Variant ID. |
| `product_tmpl_id` | `Many2one` (`product.template`) | Parent template. |
| `public_categ_ids` | `Many2many` (`product.public.category`) | Template's eCommerce categories (shared by all variants). |
| `default_code` | `Char` | Variant SKU. |
| `free_qty` | `Float` *(Computed)* | Free-to-use stock for this variant. |

---

### 3.4 Website (`website`)

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `id` | `Integer` | Website ID. |
| `name` | `Char` | Website name. |
| `domain` | `Char` | Configured domain (e.g. `https://shop.mybrand.com`). |

---

## 4. Step-by-Step API Integration Workflows

---

### Step 1: Fetching the Category Tree (Mega-Menu & Store Navigation)

#### 1.1 Fetching Root Categories

```http
POST /json/2/product.public.category/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["parent_id", "=", false]
  ],
  "fields": ["id", "name", "sequence", "child_id", "website_id"],
  "order": "sequence asc, name asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 10,
    "name": "Electronics",
    "sequence": 1,
    "child_id": [15, 16],
    "website_id": false
  },
  {
    "id": 20,
    "name": "Apparel & Fashion",
    "sequence": 2,
    "child_id": [28, 29, 30],
    "website_id": false
  },
  {
    "id": 35,
    "name": "Special Promotions",
    "sequence": 99,
    "child_id": [],
    "website_id": false
  }
]
```

---

#### 1.2 Fetching the Entire Hierarchy in One Query

Fetch all categories at once and build the tree in your backend to avoid N+1 requests:

```http
POST /json/2/product.public.category/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [],
  "fields": ["id", "name", "parent_id", "child_id", "sequence", "seo_name", "website_id", "write_date"],
  "order": "sequence asc, name asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 10,
    "name": "Electronics",
    "parent_id": false,
    "child_id": [15, 16],
    "sequence": 1,
    "seo_name": false,
    "website_id": false,
    "write_date": "2026-08-10 12:00:00"
  },
  {
    "id": 15,
    "name": "Audio & Sound",
    "parent_id": [10, "Electronics"],
    "child_id": [21, 22],
    "sequence": 1,
    "seo_name": false,
    "website_id": false,
    "write_date": "2026-08-11 09:30:00"
  },
  {
    "id": 21,
    "name": "Headphones",
    "parent_id": [15, "Electronics / Audio & Sound"],
    "child_id": [],
    "sequence": 1,
    "seo_name": "headphones",
    "website_id": false,
    "write_date": "2026-08-11 10:15:00"
  },
  {
    "id": 22,
    "name": "Bluetooth Speakers",
    "parent_id": [15, "Electronics / Audio & Sound"],
    "child_id": [],
    "sequence": 2,
    "seo_name": false,
    "website_id": false,
    "write_date": "2026-08-11 10:15:00"
  },
  {
    "id": 16,
    "name": "Computers & Tablets",
    "parent_id": [10, "Electronics"],
    "child_id": [25],
    "sequence": 2,
    "seo_name": false,
    "website_id": false,
    "write_date": "2026-08-11 09:35:00"
  },
  {
    "id": 25,
    "name": "Laptops",
    "parent_id": [16, "Electronics / Computers & Tablets"],
    "child_id": [],
    "sequence": 1,
    "seo_name": false,
    "website_id": false,
    "write_date": "2026-08-11 10:20:00"
  }
]
```

> [!NOTE]
> Unset `Char` fields are returned as `false`, not `null` or `""`. Always fall back (e.g. `seo_name || slugify(name)`).

---

#### 1.3 Fetching Direct Children of a Category

For lazy-loaded menus:

```http
POST /json/2/product.public.category/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["parent_id", "=", 10]
  ],
  "fields": ["id", "name", "sequence", "child_id"],
  "order": "sequence asc, name asc"
}
```

To fetch **all descendants** (any depth) of category `10`, use `[["id", "child_of", 10]]` (the result includes category `10` itself).

---

### Step 2: Fetching Category Details, SEO & Images

#### 2.1 Category Metadata & SEO Properties

```http
POST /json/2/product.public.category/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [21],
  "fields": [
    "id",
    "name",
    "display_name",
    "parent_id",
    "parents_and_self",
    "website_description",
    "website_meta_title",
    "website_meta_description",
    "website_meta_keywords",
    "write_date"
  ]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 21,
    "name": "Headphones",
    "display_name": "Electronics / Audio & Sound / Headphones",
    "parent_id": [15, "Electronics / Audio & Sound"],
    "parents_and_self": [10, 15, 21],
    "website_description": "<p>Explore studio monitors, noise-cancelling headphones, and wireless earbuds.</p>",
    "website_meta_title": "Buy Studio & Wireless Headphones Online",
    "website_meta_description": "Browse our collection of audiophile and noise-cancelling headphones.",
    "website_meta_keywords": "headphones, anc, bluetooth audio",
    "write_date": "2026-08-11 10:15:00"
  }
]
```

> [!NOTE]
> `read` raises a `MissingError` if an ID does not exist. Use `search_read` with `[["id", "=", 21]]` if you prefer an empty list `[]` instead.

---

#### 2.2 Category Images via Direct URLs

> [!CAUTION]
> **Do not request `image_*` fields in list queries.** They return Base64 strings and inflate payloads dramatically. Build image URLs instead:

```
https://mycompany.odoo.com/web/image/product.public.category/<CATEGORY_ID>/<FIELD_NAME>?unique=<CACHE_KEY>
```

| Use Case | Field | Example |
| :--- | :--- | :--- |
| Category page banner | `image_1024` | `/web/image/product.public.category/21/image_1024?unique=20260811101500` |
| Category grid tile | `image_512` | `/web/image/product.public.category/21/image_512?unique=20260811101500` |
| Mega-menu tile | `image_256` | `/web/image/product.public.category/21/image_256?unique=20260811101500` |
| Menu icon | `image_128` | `/web/image/product.public.category/21/image_128?unique=20260811101500` |

`unique` is only a cache-busting key — derive it from `write_date` so it changes whenever the category is modified. If a category has no image, Odoo returns a placeholder image.

---

### Step 3: Multi-Website Category Filtering

Return categories shared by all websites **plus** those restricted to website `1`:

```http
POST /json/2/product.public.category/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    "|",
    ["website_id", "=", false],
    ["website_id", "=", 1]
  ],
  "fields": ["id", "name", "parent_id", "child_id", "website_id"],
  "order": "sequence asc, name asc"
}
```

> [!TIP]
> When filtering by website, a child category may be visible while its parent is restricted to another website. Treat any category whose parent is not in the result set as a root (as the code samples in Section 5 do).

---

### Step 4: Fetching Products by eCommerce Category

#### 4.1 Products Directly in a Category

```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["sale_ok", "=", true],
    ["is_published", "=", true],
    ["public_categ_ids", "in", [21]]
  ],
  "fields": ["id", "name", "default_code", "list_price", "public_categ_ids", "categ_id"],
  "limit": 20,
  "offset": 0,
  "order": "list_price asc"
}
```

---

#### 4.2 Products in a Category and All Its Subcategories (`child_of`)

When a shopper opens **Electronics (ID 10)**, show products from Electronics and every subcategory:

```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["sale_ok", "=", true],
    ["is_published", "=", true],
    ["public_categ_ids", "child_of", 10]
  ],
  "fields": ["id", "name", "default_code", "list_price", "public_categ_ids", "qty_available", "product_variant_ids"],
  "limit": 24,
  "offset": 0,
  "order": "name asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 620,
    "name": "Ultra-Slim 15\" Laptop 16GB RAM",
    "default_code": "SKU-PC-LAP-01",
    "list_price": 1199.0,
    "public_categ_ids": [25],
    "qty_available": 18.0,
    "product_variant_ids": [903]
  },
  {
    "id": 502,
    "name": "Portable Bluetooth Speaker Waterproof",
    "default_code": "SKU-AUDIO-SPK-02",
    "list_price": 79.5,
    "public_categ_ids": [22],
    "qty_available": 115.0,
    "product_variant_ids": [781]
  },
  {
    "id": 501,
    "name": "Wireless Noise Cancelling Headphones Pro",
    "default_code": "SKU-AUDIO-ANC-01",
    "list_price": 299.99,
    "public_categ_ids": [21],
    "qty_available": 42.0,
    "product_variant_ids": [771, 772]
  }
]
```

> [!NOTE]
> `default_code` on a template is only filled when the template has a **single** variant. For multi-variant products, read SKUs from `product.product`.

---

#### 4.3 Pagination & Stock Availability

**1. Total count for pagination controls:**

```http
POST /json/2/product.template/search_count HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["sale_ok", "=", true],
    ["is_published", "=", true],
    ["public_categ_ids", "child_of", 10]
  ]
}
```

**Success Response (`200 OK`):**
```json
142
```

**2. Free-to-use stock for the templates on the current page** (`free_qty` lives on variants):

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["product_tmpl_id", "in", [620, 502, 501]]
  ],
  "fields": ["id", "product_tmpl_id", "default_code", "free_qty"]
}
```

**Success Response (`200 OK`):**
```json
[
  { "id": 903, "product_tmpl_id": [620, "Ultra-Slim 15\" Laptop 16GB RAM"], "default_code": "SKU-PC-LAP-01", "free_qty": 15.0 },
  { "id": 781, "product_tmpl_id": [502, "Portable Bluetooth Speaker Waterproof"], "default_code": "SKU-AUDIO-SPK-02", "free_qty": 110.0 },
  { "id": 771, "product_tmpl_id": [501, "Wireless Noise Cancelling Headphones Pro"], "default_code": "SKU-AUDIO-ANC-01-BLK", "free_qty": 30.0 },
  { "id": 772, "product_tmpl_id": [501, "Wireless Noise Cancelling Headphones Pro"], "default_code": "SKU-AUDIO-ANC-01-WHT", "free_qty": 8.0 }
]
```

Sum `free_qty` per `product_tmpl_id[0]` for the grid badge.

---

### Step 5: Resolving Breadcrumbs for Product Detail Pages

#### 5.1 Read the Product's Categories

```http
POST /json/2/product.template/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501],
  "fields": ["id", "name", "public_categ_ids"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 501,
    "name": "Wireless Noise Cancelling Headphones Pro",
    "public_categ_ids": [21]
  }
]
```

#### 5.2 Build the Breadcrumb Path

Read the category's ancestor chain (`parents_and_self`, ordered root → self), then read the ancestors' names:

```http
POST /json/2/product.public.category/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [21],
  "fields": ["id", "name", "display_name", "parents_and_self"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 21,
    "name": "Headphones",
    "display_name": "Electronics / Audio & Sound / Headphones",
    "parents_and_self": [10, 15, 21]
  }
]
```

```http
POST /json/2/product.public.category/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [10, 15, 21],
  "fields": ["id", "name", "seo_name"]
}
```

Result:
```
Home > Electronics (10) > Audio & Sound (15) > Headphones (21)
```

> [!TIP]
> If you only need text (no links), `display_name` already contains the full path. If you cache the full category tree (Step 1.2), resolve breadcrumbs locally by walking `parent_id` — no extra request needed.

---

### Step 6: Assigning Public Categories to Products

`public_categ_ids` is a Many2many field, so updates use **ORM command tuples**.

| Command | Effect |
| :--- | :--- |
| `[6, 0, [id1, id2]]` | **Replace** all links with the given IDs. |
| `[4, id, 0]` | **Add** one link, keep the others. |
| `[3, id, 0]` | **Remove** one link (the category itself is not deleted). |
| `[5, 0, 0]` | **Remove all** links. |

#### 6.1 Replace All Categories

```http
POST /json/2/product.template/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501],
  "vals": {
    "public_categ_ids": [[6, 0, [15, 21, 35]]]
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

#### 6.2 Add or Remove One Category

```http
POST /json/2/product.template/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501],
  "vals": {
    "public_categ_ids": [[4, 99, 0], [3, 35, 0]]
  }
}
```

#### 6.3 Batch Assign Across Multiple Products

The same `vals` are applied to every ID in `ids`:

```http
POST /json/2/product.template/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501, 502, 620, 714],
  "vals": {
    "public_categ_ids": [[4, 35, 0]]
  }
}
```

#### 6.4 Set Categories at Product Creation

```http
POST /json/2/product.template/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Studio Monitor Headphones",
      "type": "consu",
      "is_storable": true,
      "list_price": 249.0,
      "is_published": true,
      "public_categ_ids": [[6, 0, [21]]]
    }
  ]
}
```

---

### Step 7: Creating & Modifying Public Categories

#### 7.1 Create a Root Category and Subcategories

```http
POST /json/2/product.public.category/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Smart Home & IoT",
      "sequence": 5,
      "website_meta_title": "Smart Home Automation Gear",
      "website_meta_description": "Shop smart plugs, connected bulbs, and smart home hubs."
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[45]
```

```http
POST /json/2/product.public.category/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    { "name": "Smart Lighting", "parent_id": 45, "sequence": 1 },
    { "name": "Security Cameras", "parent_id": 45, "sequence": 2 }
  ]
}
```

**Success Response (`200 OK`):**
```json
[46, 47]
```

#### 7.2 Reorder or Move a Category

```http
POST /json/2/product.public.category/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [46],
  "vals": {
    "sequence": 10,
    "parent_id": 10
  }
}
```

> [!WARNING]
> Deleting a category (`unlink`) also deletes its subcategories (`parent_id` is defined with `ondelete="cascade"`) and removes them from all products. Move the children to another parent first if they must be kept.

---

## 5. Complete Integration Code Reference

### 5.1 cURL Request Suite

#### Fetch the Full Category Tree
```bash
curl -X POST "https://mycompany.odoo.com/json/2/product.public.category/search_read" \
  -H "Authorization: bearer <API_KEY>" \
  -H "X-Odoo-Database: mycompany_db" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": [],
    "fields": ["id", "name", "parent_id", "child_id", "sequence", "seo_name", "write_date"],
    "order": "sequence asc, name asc"
  }'
```

#### Fetch Products in a Category Subtree (`child_of`)
```bash
curl -X POST "https://mycompany.odoo.com/json/2/product.template/search_read" \
  -H "Authorization: bearer <API_KEY>" \
  -H "X-Odoo-Database: mycompany_db" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": [
      ["sale_ok", "=", true],
      ["is_published", "=", true],
      ["public_categ_ids", "child_of", 10]
    ],
    "fields": ["id", "name", "list_price", "public_categ_ids"],
    "limit": 20
  }'
```

---

### 5.2 Python Client: Navigation Tree & Breadcrumb Builder

```python
#!/usr/bin/env python3
"""Odoo eCommerce category client (JSON-2 API)."""

import re
from typing import Any, Dict, List, Optional

import requests


class OdooCategoryClient:
    def __init__(self, base_url: str, db: str, api_key: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"bearer {api_key}",
            "X-Odoo-Database": db,
            "Content-Type": "application/json; charset=utf-8",
            "User-Agent": "Odoo-Category-Service/1.0",
        }

    def _call(self, model: str, method: str, payload: Dict[str, Any]) -> Any:
        url = f"{self.base_url}/json/2/{model}/{method}"
        response = requests.post(url, headers=self.headers, json=payload, timeout=10)
        response.raise_for_status()
        return response.json()

    @staticmethod
    def _cache_key(write_date: Optional[str]) -> str:
        return re.sub(r"\D", "", write_date or "")

    def get_all_categories(self, website_id: Optional[int] = None) -> List[Dict[str, Any]]:
        domain: List[Any] = []
        if website_id:
            domain = ["|", ["website_id", "=", False], ["website_id", "=", website_id]]
        return self._call("product.public.category", "search_read", {
            "domain": domain,
            "fields": ["id", "name", "parent_id", "child_id", "sequence", "seo_name", "write_date"],
            "order": "sequence asc, name asc",
        })

    def build_navigation_tree(self, website_id: Optional[int] = None) -> List[Dict[str, Any]]:
        """Fetch all categories in one request and nest them in memory."""
        raw = self.get_all_categories(website_id)
        nodes: Dict[int, Dict[str, Any]] = {}
        for cat in raw:
            nodes[cat["id"]] = {
                "id": cat["id"],
                "name": cat["name"],
                "slug": cat["seo_name"] or f"category-{cat['id']}",  # unset Char fields are False
                "image_url": (
                    f"{self.base_url}/web/image/product.public.category/{cat['id']}/image_512"
                    f"?unique={self._cache_key(cat.get('write_date'))}"
                ),
                "children": [],
            }

        roots: List[Dict[str, Any]] = []
        for cat in raw:  # raw is already sorted by sequence, so children stay sorted
            parent = cat["parent_id"]
            parent_id = parent[0] if parent else None
            if parent_id in nodes:
                nodes[parent_id]["children"].append(nodes[cat["id"]])
            else:
                roots.append(nodes[cat["id"]])  # root, or parent filtered out by website
        return roots

    def get_products_by_category(self, category_id: int, page: int = 1, page_size: int = 24) -> Dict[str, Any]:
        """Published products in the category or any subcategory, with variant free stock."""
        domain = [
            ["sale_ok", "=", True],
            ["is_published", "=", True],
            ["public_categ_ids", "child_of", category_id],
        ]
        total = self._call("product.template", "search_count", {"domain": domain})
        products = self._call("product.template", "search_read", {
            "domain": domain,
            "fields": ["id", "name", "list_price", "public_categ_ids", "write_date"],
            "limit": page_size,
            "offset": (page - 1) * page_size,
            "order": "name asc",
        })

        # free_qty only exists on product.product: aggregate it per template
        template_ids = [p["id"] for p in products]
        free_by_template: Dict[int, float] = {}
        if template_ids:
            variants = self._call("product.product", "search_read", {
                "domain": [["product_tmpl_id", "in", template_ids]],
                "fields": ["product_tmpl_id", "free_qty"],
            })
            for v in variants:
                tmpl_id = v["product_tmpl_id"][0]
                free_by_template[tmpl_id] = free_by_template.get(tmpl_id, 0.0) + v["free_qty"]

        for p in products:
            p["free_qty"] = free_by_template.get(p["id"], 0.0)
            p["image_url"] = (
                f"{self.base_url}/web/image/product.template/{p['id']}/image_512"
                f"?unique={self._cache_key(p.get('write_date'))}"
            )

        return {"total": total, "page": page, "page_size": page_size, "products": products}

    def get_breadcrumbs_for_product(self, product_tmpl_id: int) -> List[List[Dict[str, Any]]]:
        """One breadcrumb trail (root -> leaf) per public category of the product."""
        products = self._call("product.template", "search_read", {
            "domain": [["id", "=", product_tmpl_id]],
            "fields": ["public_categ_ids"],
        })
        if not products or not products[0]["public_categ_ids"]:
            return []

        categories = self._call("product.public.category", "read", {
            "ids": products[0]["public_categ_ids"],
            "fields": ["parents_and_self"],
        })
        ancestor_ids = sorted({cid for cat in categories for cid in cat["parents_and_self"]})
        ancestors = self._call("product.public.category", "read", {
            "ids": ancestor_ids,
            "fields": ["id", "name"],
        })
        names = {a["id"]: a["name"] for a in ancestors}

        return [
            [{"id": cid, "name": names[cid]} for cid in cat["parents_and_self"]]
            for cat in categories
        ]


if __name__ == "__main__":
    client = OdooCategoryClient("https://mycompany.odoo.com", "mycompany_db", "<API_KEY>")
    tree = client.build_navigation_tree()
    print(f"Loaded {len(tree)} root categories")
    page = client.get_products_by_category(category_id=15, page=1, page_size=10)
    print(f"{page['total']} products in 'Audio & Sound' and its subcategories")
```

---

### 5.3 TypeScript / Node.js Storefront Category Service

```typescript
import axios, { AxiosInstance } from "axios";

export interface OdooPublicCategory {
  id: number;
  name: string;
  parent_id: [number, string] | false;
  child_id: number[];
  sequence: number;
  seo_name: string | false;
  write_date: string;
}

export interface NavigationNode {
  id: number;
  name: string;
  slug: string;
  imageUrl: string;
  children: NavigationNode[];
}

export class OdooCategoryService {
  private client: AxiosInstance;
  private baseUrl: string;

  constructor(baseUrl: string, db: string, apiKey: string) {
    this.baseUrl = baseUrl.replace(/\/$/, "");
    this.client = axios.create({
      baseURL: `${this.baseUrl}/json/2`,
      headers: {
        Authorization: `bearer ${apiKey}`,
        "X-Odoo-Database": db,
        "Content-Type": "application/json; charset=utf-8",
      },
      timeout: 10000,
    });
  }

  async getNavigationTree(websiteId?: number): Promise<NavigationNode[]> {
    const domain: unknown[] = websiteId
      ? ["|", ["website_id", "=", false], ["website_id", "=", websiteId]]
      : [];

    const { data: categories } = await this.client.post<OdooPublicCategory[]>(
      "/product.public.category/search_read",
      {
        domain,
        fields: ["id", "name", "parent_id", "child_id", "sequence", "seo_name", "write_date"],
        order: "sequence asc, name asc",
      }
    );

    const nodes = new Map<number, NavigationNode>();
    for (const cat of categories) {
      const cacheKey = (cat.write_date || "").replace(/\D/g, "");
      nodes.set(cat.id, {
        id: cat.id,
        name: cat.name,
        slug: cat.seo_name || `category-${cat.id}`,
        imageUrl: `${this.baseUrl}/web/image/product.public.category/${cat.id}/image_512?unique=${cacheKey}`,
        children: [],
      });
    }

    const roots: NavigationNode[] = [];
    for (const cat of categories) {
      const node = nodes.get(cat.id)!;
      const parent = cat.parent_id ? nodes.get(cat.parent_id[0]) : undefined;
      if (parent) {
        parent.children.push(node);
      } else {
        roots.push(node);
      }
    }
    return roots;
  }

  /** Replace all eCommerce categories of a product template. */
  async setProductPublicCategories(productTemplateId: number, categoryIds: number[]): Promise<boolean> {
    const { data } = await this.client.post<boolean>("/product.template/write", {
      ids: [productTemplateId],
      vals: { public_categ_ids: [[6, 0, categoryIds]] },
    });
    return data;
  }
}
```

---

## 6. Error Handling & Edge Cases

### 6.1 Category Not Found

- `search_read` with an unknown ID returns `[]` with `200 OK`. Render a `404` page in your storefront.
- `read` with an unknown ID returns a `MissingError` (`404 Not Found`):

```json
{
  "name": "odoo.exceptions.MissingError",
  "message": "Record does not exist or has been deleted.\n(Record: product.public.category(9999,), User: 2)",
  "arguments": ["Record does not exist or has been deleted.\n(Record: product.public.category(9999,), User: 2)"],
  "context": {},
  "debug": "Traceback (most recent call last): ..."
}
```

---

### 6.2 Invalid Field in `fields` (e.g. `has_children`, `active`, `free_qty` on templates)

Requesting a field that does not exist on the model fails the entire request:

```json
{
  "name": "builtins.ValueError",
  "message": "Invalid field 'has_children' on model 'product.public.category'",
  "arguments": ["Invalid field 'has_children' on model 'product.public.category'"],
  "context": {},
  "debug": "Traceback (most recent call last): ..."
}
```

**Fix:** Only request fields listed in [Section 3](#3-model--field-references). The same error occurs for `free_qty` on `product.template` and for `public_categ_ids` when `website_sale` is not installed.

---

### 6.3 Recursive Category Hierarchy

Setting a category's parent to itself or to one of its descendants is rejected by a recursion constraint:

```http
POST /json/2/product.public.category/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [10],
  "vals": { "parent_id": 21 }
}
```

The response is an error whose `message` is `Error! You cannot create recursive categories.` and the write is rolled back. Validate parent changes against your cached tree before sending them.

---

### 6.4 Wrong Many2many Value Format on `public_categ_ids`

Always send a **list of command tuples** (e.g. `[[6, 0, [15, 21]]]`). Do not send a bare list of IDs (`[15, 21]`) or a single integer (`15`); these are not valid write commands and are either rejected or not applied as expected. After a sync, read back `public_categ_ids` to verify.

---

### 6.5 Authentication & Permission Failures

#### `401 Unauthorized` — Invalid or Expired API Key
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "..."
}
```

#### `403 Forbidden` — Missing Access Rights
Creating or editing categories requires website/eCommerce editor rights. Without them:
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to modify 'Website Product Category' (product.public.category) records. ...",
  "arguments": ["You are not allowed to modify 'Website Product Category' (product.public.category) records. ..."],
  "context": {},
  "debug": "..."
}
```

---

## 7. Production Best Practices for E-Commerce Developers

### 1. Never Use `categ_id` for Storefront Navigation
`categ_id` holds internal accounting categories (e.g. `All / Saleable / Taxable`). Drive navigation, filters, and breadcrumbs exclusively from `product.public.category` and `public_categ_ids`.

### 2. Cache the Category Tree
Category structures change rarely.
- Fetch all categories once (Step 1.2) and cache the nested tree (Redis, edge KV) with a **1–6 hour TTL**.
- For incremental refresh, poll `[["write_date", ">", "<last_sync_utc>"]]`. Deletions are not visible this way, so do a periodic full refresh as well.

### 3. Let Odoo Resolve Subtrees with `child_of`
Use `["public_categ_ids", "child_of", category_id]` instead of collecting descendant IDs yourself. Odoo resolves it efficiently via the indexed `parent_path` column.

### 4. Zero-Base64 Image Pipeline
Never read `image_*` fields in list queries. Use `/web/image/...` URLs with a `unique` key derived from `write_date`, and let your CDN cache them aggressively.

### 5. Localization
Pass the language in the request context to get translated category names and descriptions:
```json
{
  "domain": [["parent_id", "=", false]],
  "fields": ["id", "name"],
  "context": { "lang": "fr_FR" }
}
```
The language must be installed and active in Odoo.

### 6. Filter on `is_published`, Not `website_published`
`is_published` is the stored publication flag. `website_published` is a computed helper that depends on the current website context and can behave unexpectedly when called from an external API.

### 7. Use `[6, 0, ids]` for Full Syncs
When mirroring categories from a PIM, `[[6, 0, [...]]]` guarantees Odoo exactly matches the source with no stale links. Use `[4, id, 0]` / `[3, id, 0]` for incremental changes.
