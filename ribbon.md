# Odoo JSON-2 API: Product Ribbons (`product.ribbon`) Reference Guide

This guide is the companion to [inventory.md](inventory.md) (product model) and documents how to **fetch the list of product ribbons** (`product.ribbon`) and how those ribbons are attached to catalog records (`product.template.website_ribbon_id`, `product.product.variant_ribbon_id`) through Odoo's **External JSON-2 API**. It is aimed at backend developers who need to render storefront flags such as **"New"**, **"On Sale"** or **"Clearance"** on product cards and product pages.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

> [!WARNING]
> **Module & permission prerequisites**
> - `product.ribbon` ships with the **eCommerce (`website_sale`)** app. On a database where that module is not installed, the model does not exist and every call answers `404 Not Found` with `the model 'product.ribbon' does not exist`.
> - **ACL:** `read` is granted to internal, portal and public users, while **`create`, `write` and `unlink` are reserved to Sales Managers** (`sales_team.group_sale_manager`). An API key owned by a plain salesperson can *read* the ribbon list but receives `403 Forbidden` on the write operations.
> - Ribbons are **global configuration**: the model has no `website_id`, `company_id` or `active` field, so the same list is shared by every website of the database.
> - Odoo allows **at most one automatic ribbon per assignment mode** (`assign = 'sale'` and `assign = 'new'`). Creating a second one raises a `ValidationError` (`422`).

---

## Table of Contents

1. [Architectural Overview & Ribbon Concepts](#1-architectural-overview--ribbon-concepts)
   - [Manual vs. Automatic Ribbons](#manual-vs-automatic-ribbons)
   - [Effective Ribbon Resolution (Precedence)](#effective-ribbon-resolution-precedence)
   - [Entity Relationship Diagram](#entity-relationship-diagram)
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
3. [Model & Field Reference](#3-model--field-reference)
   - [3.1 `product.ribbon`](#31-productribbon)
   - [3.2 Ribbon Link & Rule Fields on Products](#32-ribbon-link--rule-fields-on-products)
4. [Fetching the Ribbons List](#4-fetching-the-ribbons-list)
   - [Step 1: List All Ribbons](#step-1-list-all-ribbons)
   - [Step 2: Read a Single Ribbon](#step-2-read-a-single-ribbon)
   - [Step 3: Filter Ribbons by Assignment Mode](#step-3-filter-ribbons-by-assignment-mode)
   - [Step 4: Discover Selection Values Dynamically (`fields_get`)](#step-4-discover-selection-values-dynamically-fields_get)
   - [Step 5: Fetch Translated Ribbon Labels](#step-5-fetch-translated-ribbon-labels)
   - [Step 6: Incremental Sync via `write_date`](#step-6-incremental-sync-via-write_date)
   - [Step 7: Fetch the Ribbon Attached to Each Product](#step-7-fetch-the-ribbon-attached-to-each-product)
5. [Assigning & Clearing Ribbons on Products](#5-assigning--clearing-ribbons-on-products)
6. [Creating, Updating & Deleting Ribbons](#6-creating-updating--deleting-ribbons)
7. [Rendering Ribbons on Your Storefront](#7-rendering-ribbons-on-your-storefront)
8. [Error Handling & Failure Responses](#8-error-handling--failure-responses)
9. [Production Best Practices](#9-production-best-practices)

---

## 1. Architectural Overview & Ribbon Concepts

A **ribbon** (`product.ribbon`) is a reusable *presentation* record: a label (`name`), two hex colors (`bg_color`, `text_color`), a shape (`style`) and a corner (`position`). The same ribbon record can flag many products; each product only stores the **link** to that record.

Ribbons are rendered on the product image of the `/shop` grid and of the product page, and are resolved by the eCommerce module at render time.

### Manual vs. Automatic Ribbons

| `assign` | Behaviour | Where the product link lives | Typical e-commerce use |
| :--- | :--- | :--- | :--- |
| `manual` | Shown **only** on the products explicitly linked to the ribbon. | `product.template.website_ribbon_id` or `product.product.variant_ribbon_id` | Editorial flags: `Clearance`, `Staff Pick`, `Exclusive` |
| `sale` | Shown **automatically** when the displayed price is lower than the comparison price (pricelist discount or `compare_list_price`). | Nowhere — resolved from the ribbon record only | `On Sale` |
| `new` | Shown **automatically** while the product has been published for less than `new_period` days. | Nowhere — resolved from the ribbon record only | `New` |

> [!IMPORTANT]
> **Only one ribbon per automatic mode is allowed.** A second record with `assign = 'sale'` (or `'new'`) is rejected by a Python constraint, because automatic resolution always takes the first matching ribbon. `manual` ribbons are unlimited.

### Effective Ribbon Resolution (Precedence)

Odoo resolves the ribbon of a displayed variant in this order:

```
product.product.variant_ribbon_id   ─┐
                                     ├──► (1) manual ribbon wins (variant first, then template)
product.template.website_ribbon_id  ─┘
                 │ (both empty)
                 ▼
(2) first ribbon with assign != 'manual', ordered by sequence ASC, id ASC,
    whose automatic rule matches the product/price being displayed
```

> [!NOTE]
> **Private methods are not callable through JSON-2.** The resolver (`_get_ribbon`), the automatic-rule matcher (`_is_applicable_for`) and the CSS helper (`_get_css_classes`) all start with `_`, so they cannot be reached remotely (`403 Forbidden`). Your backend must resolve the ribbon from the field values documented below — see [Rendering Ribbons on Your Storefront](#7-rendering-ribbons-on-your-storefront).

### Entity Relationship Diagram

```
                  ┌──────────────────────────┐
                  │      product.ribbon      │  global list (no website/company field)
                  │  name, style, position,  │
                  │  bg_color, text_color,   │
                  │  assign, new_period      │
                  └───────▲──────────▲───────┘
                          │          │
      website_ribbon_id   │          │  variant_ribbon_id
        (manual)          │          │  (manual, per-variant override)
                          │          │
                 ┌────────┴───────┐  │
                 │ product.template│  │
                 └────────┬───────┘  │
                          │ 1 .. n   │
                 ┌────────▼───────────┴┐
                 │    product.product  │  sellable variant
                 └─────────────────────┘
```

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`; the JSON body carries the named parameters (`ids`, `domain`, `fields`, `vals`, `vals_list`, `context`, …).

### Required Headers
```http
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-Catalog-Sync/1.0
```

| Element | Rule |
| :--- | :--- |
| `Authorization` | `bearer <API_KEY>` — an API key generated by an Odoo user; the key inherits that user's groups (see the ACL warning above). |
| `Content-Type` | Must be `application/json`, otherwise the body is not parsed. |
| `ids` | Array of integers, required by record-level methods (`read`, `write`, `unlink`). |
| `context` | Optional object, e.g. `{"lang": "fr_FR"}` for translated ribbon names. |

---

## 3. Model & Field Reference

### 3.1 `product.ribbon`

Ribbon definitions. The model is ordered by `sequence ASC, id` (`_order`), so listing without an explicit `order` already returns ribbons in display/priority order.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` (max 20 chars, translatable) | **REQUIRED** | — | Ribbon label rendered on the badge (e.g. `New`, `On Sale`, `Clearance`). |
| `sequence` | `Integer` | Optional | `10` | Ordering / priority (lower first). Drives both the list order and which automatic ribbon is evaluated first. |
| `bg_color` | `Char` (hex) | **REQUIRED** | `'#000000'` | Background color as a CSS hex string (e.g. `#EA580C`). |
| `text_color` | `Char` (hex) | **REQUIRED** | `'#FFFFFF'` | Label text color as a CSS hex string. |
| `position` | `Selection` | **REQUIRED** | `'left'` | Corner of the product image: `'left'`, `'right'`. |
| `style` | `Selection` | **REQUIRED** | `'ribbon'` | Shape: `'ribbon'` (banner across the image) or `'tag'` (small badge). |
| `assign` | `Selection` | **REQUIRED** | `'manual'` | Assignment mode: `'manual'` (linked per product), `'sale'` (automatic when a discount is displayed), `'new'` (automatic while the product is "new"). |
| `new_period` | `Integer` | Optional | `30` | Number of days after `publish_date` during which the `assign = 'new'` ribbon applies. Ignored by the other modes. |
| `write_date` | `Datetime` | *System* | — | Last modification timestamp — use it for incremental sync (see [Step 6](#step-6-incremental-sync-via-write_date)). |

> [!NOTE]
> **There is no `active` field and no `website_id` / `company_id`.** To "disable" a manual ribbon, clear the product link (`website_ribbon_id: false`) or `unlink` the ribbon record; automatic ribbons apply to every product by definition.

### 3.2 Ribbon Link & Rule Fields on Products

The following fields live on the product models and are the ones your catalog sync should read/write:

| Model | Field | Type | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `product.template` | `website_ribbon_id` | `Many2one` (`product.ribbon`) | `null` | Manual ribbon shared by **all variants** of the template. |
| `product.product` | `variant_ribbon_id` | `Many2one` (`product.ribbon`) | `null` | Per-variant manual override; **takes precedence** over `website_ribbon_id`. |
| `product.template` (inherited by `product.product`) | `publish_date` | `Datetime` (stored, computed) | publish moment | Reset to "now" each time the product is (re-)published; the `assign = 'new'` rule compares it with `new_period`. |
| `product.template` | `compare_list_price` | `Monetary` | `0.0` | Struck-through comparison price; drives the `assign = 'sale'` rule on the product page. |
| `product.template` | `is_published` | `Boolean` | `true` | Website publication flag — only published products are visible in the shop. |

---

## 4. Fetching the Ribbons List

The ribbon table is small configuration data (typically 2–10 records), so a single `search_read` is enough to build your storefront's badge palette. Cache the result and refresh it with [`write_date`](#step-6-incremental-sync-via-write_date).

### Step 1: List All Ribbons

```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "fields": [
    "id",
    "name",
    "sequence",
    "style",
    "position",
    "bg_color",
    "text_color",
    "assign",
    "new_period",
    "write_date"
  ],
  "order": "sequence asc, id asc",
  "limit": 50
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "name": "New",
    "sequence": 10,
    "style": "tag",
    "position": "left",
    "bg_color": "#0F9D58",
    "text_color": "#FFFFFF",
    "assign": "new",
    "new_period": 30,
    "write_date": "2026-07-02 08:14:51"
  },
  {
    "id": 2,
    "name": "On Sale",
    "sequence": 20,
    "style": "ribbon",
    "position": "right",
    "bg_color": "#DC2626",
    "text_color": "#FFFFFF",
    "assign": "sale",
    "new_period": 30,
    "write_date": "2026-07-02 08:15:09"
  },
  {
    "id": 3,
    "name": "Clearance",
    "sequence": 30,
    "style": "ribbon",
    "position": "left",
    "bg_color": "#111827",
    "text_color": "#FFFFFF",
    "assign": "manual",
    "new_period": 30,
    "write_date": "2026-08-19 16:42:07"
  }
]
```

> [!TIP]
> **Always request explicit `fields`.** A `search_read` without `fields` returns every column of the model; on large records this needlessly inflates the payload and couples your client to future schema changes.

### Step 2: Read a Single Ribbon

Useful after a product payload returned `[2, "On Sale"]` and you need the colors/style of that ribbon:

```http
POST /json/2/product.ribbon/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [2],
  "fields": ["id", "name", "style", "position", "bg_color", "text_color", "assign"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 2,
    "name": "On Sale",
    "style": "ribbon",
    "position": "right",
    "bg_color": "#DC2626",
    "text_color": "#FFFFFF",
    "assign": "sale"
  }
]
```

---

### Step 3: Filter Ribbons by Assignment Mode

If your storefront lets merchandisers pick a **manual** ribbon for a product, list only the assignable ones (automatic ribbons cannot be attached to a product):

```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["assign", "=", "manual"]
  ],
  "fields": ["id", "name", "style", "position", "bg_color", "text_color", "sequence"],
  "order": "sequence asc, id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 3,
    "name": "Clearance",
    "style": "ribbon",
    "position": "left",
    "bg_color": "#111827",
    "text_color": "#FFFFFF",
    "sequence": 30
  }
]
```

Use `[["assign", "!=", "manual"]]` to retrieve only the automatic ribbons and their rules (`assign`, `new_period`, `sequence`).

### Step 4: Discover Selection Values Dynamically (`fields_get`)

Never hard-code the selection keys — read them from the model so your UI stays valid across versions:

```http
POST /json/2/product.ribbon/fields_get HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "allfields": ["position", "style", "assign"],
  "attributes": ["type", "required", "string", "selection", "help"]
}
```

**Success Response (`200 OK`):**
```json
{
  "position": {
    "type": "selection",
    "required": true,
    "string": "Position",
    "selection": [["left", "Left"], ["right", "Right"]],
    "help": false
  },
  "style": {
    "type": "selection",
    "required": true,
    "string": "Style",
    "selection": [["ribbon", "Ribbon"], ["tag", "Badge"]],
    "help": "Defines the display style:\n- Ribbon: Shows a ribbon banner on the product image.\n- Badge: Shows a small badge label on the product image."
  },
  "assign": {
    "type": "selection",
    "required": true,
    "string": "Assign",
    "selection": [["manual", "Manually"], ["sale", "On Sale"], ["new", "When New"]],
    "help": "Defines how this ribbon is assigned to products:\n- Manually: You assign the ribbon manually to products.\n- Sale: Applied when the product is visibly on sale.\n- New: Applied based on the New period you will define.\n"
  }
}
```

### Step 5: Fetch Translated Ribbon Labels

`name` is translatable. Pass `lang` in the `context` to receive labels in the storefront language (falls back to the source value when no translation exists):

```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "fields": ["id", "name"],
  "context": {
    "lang": "fr_FR"
  }
}
```

**Success Response (`200 OK`):**
```json
[
  {"id": 1, "name": "Nouveauté"},
  {"id": 2, "name": "En promotion"},
  {"id": 3, "name": "Déstockage"}
]
```

---
### Step 6: Incremental Sync via `write_date`

Ribbons change rarely. Store the last successful sync timestamp and fetch only what moved:

```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["write_date", ">", "2026-08-01 00:00:00"]
  ],
  "fields": ["id", "name", "style", "position", "bg_color", "text_color", "assign", "new_period", "write_date"],
  "order": "write_date asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 3,
    "name": "Clearance",
    "style": "ribbon",
    "position": "left",
    "bg_color": "#111827",
    "text_color": "#FFFFFF",
    "assign": "manual",
    "new_period": 30,
    "write_date": "2026-08-19 16:42:07"
  }
]
```

> [!NOTE]
> `write_date` is expressed in **UTC** (`YYYY-MM-DD HH:MM:SS`). Always send the timestamp of your previous *server-side* sync, not a local timezone value, and keep the previous timestamp as a string to avoid precision drift.

### Step 7: Fetch the Ribbon Attached to Each Product

Ribbons are attached through the product models, so a catalog sync reads the many2one and resolves it against the cached ribbon list:

```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["sale_ok", "=", true],
    ["is_published", "=", true]
  ],
  "fields": ["id", "name", "is_published", "publish_date", "compare_list_price", "website_ribbon_id"],
  "limit": 2,
  "order": "id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 620,
    "name": "Ultra-Slim Wireless Mouse",
    "is_published": true,
    "publish_date": "2026-08-23 09:40:12",
    "compare_list_price": 0.0,
    "website_ribbon_id": [3, "Clearance"]
  },
  {
    "id": 621,
    "name": "Mechanical RGB Gaming Keyboard",
    "is_published": true,
    "publish_date": "2026-09-02 11:05:00",
    "compare_list_price": 149.99,
    "website_ribbon_id": false
  }
]
```

Product `621` has no manual ribbon (`false`), but its `compare_list_price` (149.99) is higher than its selling price, so the automatic `On Sale` ribbon applies.

To find **variant-level** overrides (they beat the template ribbon):

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["variant_ribbon_id", "!=", false]
  ],
  "fields": ["id", "default_code", "product_tmpl_id", "variant_ribbon_id"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1056,
    "default_code": "SKU-MOUSE-SLIM-01-WHT",
    "product_tmpl_id": [620, "Ultra-Slim Wireless Mouse"],
    "variant_ribbon_id": [4, "Staff Pick"]
  }
]
```

---

## 5. Assigning & Clearing Ribbons on Products

Manual assignment is a plain `many2one` write. It requires **write access on the product model** (a normal internal sales user), *not* the Sales-Manager-only ACL of `product.ribbon`.

### 5.1 Assign a Manual Ribbon to a Product Template

```http
POST /json/2/product.template/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [620],
  "vals": {
    "website_ribbon_id": 3
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

### 5.2 Remove the Ribbon

Send `false` to break the link (the ribbon record itself is left untouched and can be reused):

```json
{
  "ids": [620],
  "vals": {
    "website_ribbon_id": false
  }
}
```

### 5.3 Override the Ribbon for a Single Variant

```http
POST /json/2/product.product/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [1056],
  "vals": {
    "variant_ribbon_id": 4
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

### 5.4 Assign the Ribbon While Creating the Product

Because it is a `Many2one`, you pass the **integer ID directly** (unlike `Many2many`, which needs command tuples):

```http
POST /json/2/product.template/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Ultra-Slim Wireless Mouse (Refurbished)",
      "is_storable": true,
      "type": "consu",
      "categ_id": 1,
      "list_price": 29.99,
      "sale_ok": true,
      "website_ribbon_id": 3
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[622]
```

---

## 6. Creating, Updating & Deleting Ribbons

> [!WARNING]
> These operations require an API key belonging to a **Sales Manager** (`sales_team.group_sale_manager`). With any other user the request fails with `403 Forbidden` (see [Error 8.4](#84-missing-sales-manager-rights-403-forbidden)).

### 6.1 Create a Ribbon

```http
POST /json/2/product.ribbon/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Staff Pick",
      "sequence": 40,
      "style": "tag",
      "position": "left",
      "bg_color": "#1D4ED8",
      "text_color": "#FFFFFF",
      "assign": "manual"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[4]
```
*(Returns the ID of the newly created `product.ribbon` record.)*

### 6.2 Update a Ribbon

```http
POST /json/2/product.ribbon/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [3],
  "vals": {
    "bg_color": "#EA580C",
    "sequence": 25
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

The change is immediately visible on every product linked to ribbon `3` — no product write is needed.

### 6.3 Delete a Ribbon

```http
POST /json/2/product.ribbon/unlink HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [4]
}
```

**Success Response (`200 OK`):**
```json
true
```
*(Products that pointed to the deleted ribbon simply lose their badge; the link column is set to `NULL`.)*

### 6.4 Idempotent Seeding of Automatic Ribbons

Because only **one** ribbon may exist per automatic mode, a configuration bootstrap must search before creating:

```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [["assign", "=", "sale"]],
  "fields": ["id", "assign", "new_period"]
}
```

- **Empty result** → `product.ribbon/create` with `"assign": "sale"`.
- **One record** → `product.ribbon/write` on that ID to align `sequence`, colors, etc.

Repeating the wrong order (`create` before `search`) raises the constraint error documented in [Error 8.3](#83-second-automatic-ribbon-422-unprocessable-entity).

---

## 7. Rendering Ribbons on Your Storefront

### 7.1 Odoo CSS Classes (for visual parity)

Odoo's own frontend derives two CSS classes from the ribbon record:

| Field | Value | CSS class added by Odoo |
| :--- | :--- | :--- |
| `style` | `'ribbon'` | `o_wsale_ribbon` |
| `style` | `'tag'` | `o_wsale_badge` |
| `position` | `'left'` | `o_left` |
| `position` | `'right'` | `o_right` |

The colors are applied inline: `background-color: {bg_color}; color: {text_color}`.

### 7.2 Resolving the Effective Ribbon in Your Backend

Fetch the ribbon list once ([Step 1](#step-1-list-all-ribbons)), fetch the product payload once ([Step 7](#step-7-fetch-the-ribbon-attached-to-each-product)), then apply Odoo's precedence yourself:

```python
def effective_ribbon(product, pricing, ribbons, ribbons_by_id):
    """product: dict from product.product/search_read ("variant_ribbon_id",
       "product_tmpl_id", "publish_date", "compare_list_price") merged with the
       parent template's "website_ribbon_id".
       pricing: {"base_price": ..., "price": ...} produced by your pricing layer
       ribbons: list from product.ribbon/search_read, ordered by sequence, id."""
    # (1) manual ribbon wins: variant first, then template
    for field in ("variant_ribbon_id", "website_ribbon_id"):
        if product.get(field):
            return ribbons_by_id[product[field][0]]

    # (2) automatic ribbons, already ordered by sequence ASC, id ASC
    for ribbon in ribbons:
        if ribbon["assign"] == "sale" and is_discounted(product, pricing):
            return ribbon
        if ribbon["assign"] == "new" and days_since(product["publish_date"]) <= ribbon["new_period"]:
            return ribbon
    return None

def is_discounted(product, pricing):
    # comparison price entered in the backend (product page)
    if product["compare_list_price"] > pricing["price"]:
        return True
    # pricelist lowered the catalog price (shop grid)
    return pricing["base_price"] > pricing["price"]

def days_since(publish_date):
    # publish_date is a UTC "YYYY-MM-DD HH:MM:SS" string
    published = datetime.strptime(publish_date, "%Y-%m-%d %H:%M:%S")
    return (datetime.utcnow() - published).days
```

> [!NOTE]
> **Where do `base_price` / `price` come from?** They are not ORM fields — Odoo computes them at render time from the pricelist and the website settings. Produce the equivalent in your own pricing layer (see [pricelists.md](pricelists.md)): `base_price` = catalog sales price (`list_price` / `lst_price`, optionally converted to the pricelist currency), `price` = price after the applicable pricelist rule.

> [!IMPORTANT]
> **Automatic ribbons are time- and price-dependent.** `assign = 'sale'` and `assign = 'new'` depend on the price displayed and on `publish_date`, which change **without any write on `product.ribbon`**. Re-evaluate the effective ribbon on every catalog refresh (or rely on Odoo's own frontend), not only when the ribbon list changes.

> [!TIP]
> **Sanitize the colors before injecting them into HTML/CSS.** `bg_color` and `text_color` are free-text `Char` fields editable in the backend. Accept them only when they match `^#(?:[0-9a-fA-F]{3}|[0-9a-fA-F]{4}|[0-9a-fA-F]{6}|[0-9a-fA-F]{8})$`, otherwise fall back to your theme defaults — this prevents CSS/HTML injection from misconfigured records.

### 7.3 Display Rules

- Render a ribbon only for products where `is_published = true`.
- Keep labels short: `name` is capped at **20 characters** in the database.
- Prefer `name` from the translated list ([Step 5](#step-5-fetch-translated-ribbon-labels)) so the badge follows the storefront language.

---

## 8. Error Handling & Failure Responses

JSON-2 maps Odoo exceptions to HTTP status codes: the `UserError` family (including `ValidationError`) returns **422**, `AccessError` / `AccessDenied` return **403**, `MissingError` returns **404**, `LockError` returns **409**, and any other exception is reported as **500**.

### 8.1 Model Not Installed (`404 Not Found`)

The `website_sale` module is not installed, so the model is unknown:

#### Request
```http
POST /json/2/product.ribbon/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "fields": ["id", "name"]
}
```

#### Error Response (`404 Not Found`)
```json
{
  "name": "werkzeug.exceptions.NotFound",
  "message": "the model 'product.ribbon' does not exist",
  "arguments": ["the model 'product.ribbon' does not exist", 404],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/rpc/controllers/json2.py\", line 60, in web_json_2_rpc\n    Model = request.env[__model__].with_context(context)\nwerkzeug.exceptions.NotFound: 404 Not Found: the model 'product.ribbon' does not exist\n"
}
```

### 8.2 Calling a Private Method (`403 Forbidden`)

Only public methods can be called remotely; `_get_ribbon`, `_is_applicable_for` and `_get_css_classes` are rejected:

#### Request
```http
POST /json/2/product.ribbon/_get_ribbon HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [620]
}
```

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "Private methods (such as 'product.ribbon._get_ribbon') cannot be called remotely.",
  "arguments": ["Private methods (such as 'product.ribbon._get_ribbon') cannot be called remotely."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/rpc/controllers/json2.py\", line 66, in web_json_2_rpc\n    func = get_public_method(Model, __method__)\nodoo.exceptions.AccessError: Private methods (such as 'product.ribbon._get_ribbon') cannot be called remotely.\n"
}
```

### 8.3 Second Automatic Ribbon (`422 Unprocessable Entity`)

Attempting to create a second `assign = 'sale'` (or `'new'`) ribbon:

#### Request
```http
POST /json/2/product.ribbon/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Promo",
      "assign": "sale",
      "bg_color": "#DC2626",
      "text_color": "#FFFFFF"
    }
  ]
}
```

#### Error Response (`422 Unprocessable Entity`)
```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "Only one ribbon with the assign On Sale is allowed.",
  "arguments": ["Only one ribbon with the assign On Sale is allowed."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/website_sale/models/product_ribbon.py\", line 78, in _check_assign\n    raise ValidationError(...)\nodoo.exceptions.ValidationError: Only one ribbon with the assign On Sale is allowed.\n"
}
```

---

### 8.4 Missing Sales Manager Rights (`403 Forbidden`)

`create` / `write` / `unlink` are restricted to `sales_team.group_sale_manager`:

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to modify 'Product ribbon' (product.ribbon) records.\n\nThis operation is allowed for the following groups:\n\t- Sales / Administrator\n\nContact your administrator to request access if necessary.",
  "arguments": ["You are not allowed to modify 'Product ribbon' (product.ribbon) records.\n\nThis operation is allowed for the following groups:\n\t- Sales / Administrator\n\nContact your administrator to request access if necessary."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/addons/base/models/ir_model.py\", line 2190, in check_access\n    raise self._make_access_error(model, mode) from None\nodoo.exceptions.AccessError: You are not allowed to modify 'Product ribbon' (product.ribbon) records.\n"
}
```

> [!NOTE]
> The group list in the message reflects the ACL rows of the database; only the `sales_team.group_sale_manager` entry grants write access to `product.ribbon`.

### 8.5 Malformed Payload (`422 Unprocessable Entity`)

Named parameters must match the method signature — e.g. sending `domain` to `read`, or `vals` to `create`:

#### Error Response (`422 Unprocessable Entity`)
```json
{
  "name": "werkzeug.exceptions.UnprocessableEntity",
  "message": "read() got an unexpected keyword argument 'domain'",
  "arguments": ["read() got an unexpected keyword argument 'domain'", 422],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/rpc/controllers/json2.py\", line 71, in web_json_2_rpc\n    signature.bind(records, **kwargs)\nwerkzeug.exceptions.UnprocessableEntity: 422 Unprocessable Entity: read() got an unexpected keyword argument 'domain'\n"
}
```

### 8.6 Invalid API Key (`401 Unauthorized`)

#### Error Response (`401 Unauthorized`)
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/addons/base/models/ir_http.py\", line 240, in _auth_method_bearer\n    raise werkzeug.exceptions.Unauthorized('Invalid apikey')\nwerkzeug.exceptions.Unauthorized: 401 Unauthorized: Invalid apikey\n"
}
```

---

## 9. Production Best Practices

### 1. Treat the Ribbon List as Cached Configuration
The table holds a handful of records that change on a merchandising schedule, not on every order. Fetch it once at boot (or once per hour) and refresh it with the `write_date` filter of [Step 6](#step-6-incremental-sync-via-write_date). Never call `product.ribbon/search_read` inside a product-card render loop.

### 2. Reproduce Odoo's Precedence Exactly
Manual (`variant_ribbon_id` → `website_ribbon_id`) before automatic (`assign = 'sale'`, `'new'`, ordered by `sequence`), and never expose a ribbon that Odoo would not show — otherwise the same product will be flagged differently in the backend preview and on your storefront.

### 3. Reference Ribbons by ID, Never by Name
`name` is translatable (and capped at 20 characters), so it is **not** a stable key. Store `id` in your CMS/mapping tables and resolve the label through the translated list.

### 4. Recount Automatic Ribbons on Every Price Refresh
`On Sale` depends on the price actually displayed and `When New` on `publish_date` + `new_period`. A ribbon can appear or disappear with **no change on the ribbon record**, so recompute during your scheduled catalog/flags job.

### 5. Seed Automatic Ribbons Idempotently
Because a second `assign = 'sale'` / `'new'` record is rejected (`422`), always `search_read` by `assign` first and then `create` (missing) or `write` (existing). Never blind-`create` during a deployment.

### 6. Give the Sync User the Right Groups
Split responsibilities: a **read-only** integration key suffices for storefront rendering; ribbon *management* (create/update/delete) needs a **Sales Manager** key. Keep the manager key out of the storefront runtime.

### 7. Sanitize the Hex Colors
`bg_color` / `text_color` are free-text fields. Validate them against a hex-color pattern before interpolating into CSS or `style` attributes, and fall back to theme colors otherwise.

### 8. Keep the Badge Accessible
A ribbon is decorative but conveys information ("On Sale"). Expose the label as text (not only as an image/background) with sufficient contrast, and honour the localized `name` so screen readers announce the correct word.

### 9. Verify the Schema Before Relying on Selections
Run `product.ribbon/fields_get` (see [Step 4](#step-4-discover-selection-values-dynamically-fields_get)) in your integration tests: `style` (`ribbon` / `tag`), `position` (`left` / `right`) and `assign` (`manual` / `sale` / `new`) are the keys your frontend maps to CSS.

---

## Related Guides

- [inventory.md](inventory.md) — `product.template` / `product.product` field reference (including `website_ribbon_id`).
- [categories.md](categories.md) — eCommerce categories used for shop navigation.
- [pricelists.md](pricelists.md) — computing the price that drives the automatic `On Sale` ribbon.
- [authentication.md](authentication.md) — generating and using API keys with the JSON-2 API.
