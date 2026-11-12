# Odoo JSON-2 API: Product Pricelists & Dynamic Pricing Guide

This guide provides an end-to-end technical reference for backend developers integrating an **e-commerce platform**, **B2B wholesale portal**, or **mobile app** with Odoo's **Pricelists & Pricing Rules (`product.pricelist`, `product.pricelist.item`, `product.template`, `product.product`, `res.partner`, `sale.order`)** using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's modern **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

> [!WARNING]
> **Odoo 16+ / 18+ / 19 Breaking Changes — read before integrating:**
> - **No contextual `price` field.** Since Odoo 16, `product.template` / `product.product` no longer have a `price` field computed from `context={"pricelist": id}`. Requesting `price` fails with `Invalid field 'price'`.
> - **Pricing methods are private.** `product.pricelist._get_product_price()`, `_get_products_price()`, `_compute_price_rule()` start with `_` and **cannot be called through JSON-2**.
> - **No `discount_policy`.** Since Odoo 18, the pricelist-level `discount_policy` field is removed. Whether a discount is shown separately now depends on each **rule's** type (`compute_price = "percentage"` shows a discount; `fixed` / `formula` produce a net price).
>
> This guide therefore documents the two approaches that work in Odoo 19: **(A) price simulation through a draft quotation** (authoritative) and **(B) reading pricelist rules and evaluating them in your backend** (fast, for catalog grids).

---

## Table of Contents

1. [Pricing Engine Overview](#1-pricing-engine-overview)
   - [How Odoo Selects a Pricelist Rule](#how-odoo-selects-a-pricelist-rule)
   - [How a Rule Computes the Price](#how-a-rule-computes-the-price)
   - [How Discounts Are Displayed (Odoo 18+)](#how-discounts-are-displayed-odoo-18)
   - [Which Pricelist Applies to a Customer](#which-pricelist-applies-to-a-customer)
   - [Entity Relationship Diagram](#entity-relationship-diagram)
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
3. [Model & Field References](#3-model--field-references)
   - [3.1 Pricelist (`product.pricelist`)](#31-pricelist-productpricelist)
   - [3.2 Pricelist Rule (`product.pricelist.item`)](#32-pricelist-rule-productpricelistitem)
   - [3.3 Product Price Fields (`product.template` / `product.product`)](#33-product-price-fields-producttemplate--productproduct)
   - [3.4 Customer Pricelist (`res.partner`)](#34-customer-pricelist-respartner)
   - [3.5 Quotation Line Price Fields (`sale.order.line`)](#35-quotation-line-price-fields-saleorderline)
4. [Step-by-Step API Integration Workflows](#4-step-by-step-api-integration-workflows)
   - [Step 1: Discovering Pricelists](#step-1-discovering-pricelists)
   - [Step 2: Computing Product Prices — Approach A: Draft Quotation Simulation](#step-2-computing-product-prices--approach-a-draft-quotation-simulation)
   - [Step 3: Fetching Pricelist Rules for a Product — Approach B](#step-3-fetching-pricelist-rules-for-a-product--approach-b)
   - [Step 4: Volume Tier Tables for Product Pages](#step-4-volume-tier-tables-for-product-pages)
   - [Step 5: Time-Limited Promotional Prices](#step-5-time-limited-promotional-prices)
   - [Step 6: Creating & Updating Pricelist Rules](#step-6-creating--updating-pricelist-rules)
5. [Complete Code Implementations](#5-complete-code-implementations)
6. [Error Handling & Edge Cases](#6-error-handling--edge-cases)
7. [Production Best Practices for E-Commerce Developers](#7-production-best-practices-for-e-commerce-developers)

---

## 1. Pricing Engine Overview

A **pricelist** (`product.pricelist`) is a currency-specific set of **rules** (`product.pricelist.item`). Each rule targets all products, an internal product category, a product template, or a single variant, optionally limited by minimum quantity and date range.

### How Odoo Selects a Pricelist Rule

For a given product variant, quantity, and date, Odoo collects the **applicable rules** of the pricelist:

- `product_tmpl_id` is empty **or** equals the product's template,
- `product_id` is empty **or** equals the variant,
- `categ_id` is empty **or** is the product's internal category (`categ_id`) **or one of its parent categories**,
- `date_start` is empty **or** ≤ date, and `date_end` is empty **or** ≥ date.

The rules are sorted by **`applied_on, min_quantity desc, categ_id desc, id desc`**, and the **first rule whose `min_quantity` ≤ ordered quantity** is used:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  Rule Priority (first match wins)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. '0_product_variant'   Specific variant      e.g. Red / XL = $19.99      │
│  2. '1_product'           Product template      e.g. T-Shirt = $22.50       │
│  3. '2_product_category'  Internal category     e.g. All / Apparel = -10%   │
│  4. '3_global'            All products          e.g. Whole store = -5%      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Within the same level: higher min_quantity first, then deeper category,    │
│  then most recently created rule.                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

If no rule matches, the product's **sales price** (`lst_price`: `list_price` + variant extra prices) is used, converted to the pricelist currency.

> [!NOTE]
> Category rules use the **internal** category `product.category` (`categ_id`), **not** the eCommerce categories (`public_categ_ids`, see [categories.md](categories.md)).

### How a Rule Computes the Price

| `compute_price` | Result |
| :--- | :--- |
| `fixed` | `fixed_price` (expressed in the pricelist currency). |
| `percentage` | `base − base × percent_price / 100`. |
| `formula` | `base − base × price_discount / 100`, then rounded with `price_round`, then `+ price_surcharge`, then bounded by `price_min_margin` / `price_max_margin` (margins relative to cost). |

`base` is selected by the rule's `base` field: `list_price` (sales price, default), `standard_price` (cost), or `pricelist` (the price computed by `base_pricelist_id`).

### How Discounts Are Displayed (Odoo 18+)

| Rule Type | On quotation lines | Storefront meaning |
| :--- | :--- | :--- |
| `percentage` | `price_unit` = base price, `discount` = percentage (when the **Discounts** setting is enabled) | Show <del>$100.00</del> **$85.00** (−15%) |
| `fixed` / `formula` | `price_unit` = final price, `discount` = 0 | Show **$85.00** (optionally compare to `lst_price` yourself) |

Always compute the **net unit price** as `price_unit × (1 − discount / 100)`. That formula is correct in both cases.

### Which Pricelist Applies to a Customer

```
┌──────────────────────────────┐
│  Is the shopper logged in?   │
└──────┬────────────────┬──────┘
       │ YES            │ NO (guest)
       ▼                ▼
┌──────────────────┐ ┌─────────────────────────────────────────┐
│ res.partner      │ │ Pick a pricelist available on the       │
│ .property_       │ │ website (website_id empty or = website),│
│ product_pricelist│ │ matching the visitor's country group    │
│                  │ │ and the selected currency               │
└──────────────────┘ └─────────────────────────────────────────┘
```

`property_product_pricelist` is computed by Odoo: if no pricelist was explicitly set on the customer, it falls back to the pricelist matching the customer's country group, then to the first available pricelist.

### Entity Relationship Diagram

```
┌───────────────────────────────┐
│          res.partner          │
│  - property_product_pricelist ┼──────────────┐
└───────────────────────────────┘              │ Many2one
                                               ▼
┌───────────────────────────────┐ 1:N  ┌───────────────────────────────┐
│       product.pricelist       ├─────►│     product.pricelist.item    │
│  - name                       │      │  - applied_on                 │
│  - currency_id                │      │  - product_tmpl_id/product_id │
│  - website_id / selectable    │      │  - categ_id (product.category)│
│  - country_group_ids          │      │  - min_quantity               │
└───────────────┬───────────────┘      │  - compute_price / base       │
                │                      │  - fixed_price/percent_price  │
                │ pricelist_id         │  - date_start / date_end      │
                ▼                      └───────────────────────────────┘
┌───────────────────────────────┐
│  sale.order (currency follows │
│  pricelist) → sale.order.line │
│  price_unit / discount        │
└───────────────────────────────┘
```

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

```http
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStorefront-Pricing-Engine/1.0
```

- **Methods used:** `search_read`, `read`, `search_count`, `create`, `write`, `unlink`, and the public business method `action_update_prices` on `sale.order`.
- **Private methods** (names starting with `_`) are blocked by JSON-2.
- Many2one values are returned as `[id, "display name"]`; empty values as `false`.

---

## 3. Model & Field References

### 3.1 Pricelist (`product.pricelist`)

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `Integer` | *Read-only* | — | Pricelist ID. |
| `name` | `Char` | **REQUIRED** | — | e.g. `Public Pricelist (USD)`, `B2B Gold`. |
| `active` | `Boolean` | Optional | `true` | `false` = archived. |
| `sequence` | `Integer` | Optional | `16` | Order used when Odoo picks a default pricelist. |
| `currency_id` | `Many2one` (`res.currency`) | **REQUIRED** | Company currency | Currency of all prices computed by this pricelist. |
| `company_id` | `Many2one` (`res.company`) | Optional | `false` | Restricts the pricelist to a company. |
| `country_group_ids` | `Many2many` (`res.country.group`) | Optional | `[]` | Country groups for automatic geo-assignment. |
| `item_ids` | `One2many` (`product.pricelist.item`) | Optional | `[]` | Pricing rules. |
| `website_id` | `Many2one` (`website`) | Optional | `false` | *(eCommerce)* Restricts the pricelist to one website. |
| `selectable` | `Boolean` | Optional | `false` | *(eCommerce)* Shopper can pick it in the website pricelist selector. |

> [!CAUTION]
> `discount_policy` **does not exist** on Odoo 18/19. Do not request it.

---

### 3.2 Pricelist Rule (`product.pricelist.item`)

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `Integer` | *Read-only* | — | Rule ID. |
| `pricelist_id` | `Many2one` (`product.pricelist`) | **REQUIRED** | — | Parent pricelist. |
| `applied_on` | `Selection` | **REQUIRED** | `'3_global'` | `'3_global'`, `'2_product_category'`, `'1_product'`, `'0_product_variant'`. |
| `categ_id` | `Many2one` (`product.category`) | Conditional | `false` | Required when `applied_on = '2_product_category'`. Also matches sub-categories. |
| `product_tmpl_id` | `Many2one` (`product.template`) | Conditional | `false` | Required when `applied_on = '1_product'`. |
| `product_id` | `Many2one` (`product.product`) | Conditional | `false` | Required when `applied_on = '0_product_variant'`. |
| `min_quantity` | `Float` | Optional | `0.0` | Rule applies when ordered quantity ≥ this value (in the product's UoM). |
| `date_start` | `Datetime` | Optional | `false` | Validity start (UTC). |
| `date_end` | `Datetime` | Optional | `false` | Validity end (UTC). |
| `compute_price` | `Selection` | **REQUIRED** | `'fixed'` | `'fixed'`, `'percentage'`, `'formula'`. |
| `fixed_price` | `Float` | Optional | `0.0` | Price for `fixed` rules. |
| `percent_price` | `Float` | Optional | `0.0` | Discount % for `percentage` rules (e.g. `15.0`). |
| `base` | `Selection` | Optional | `'list_price'` | Base for `percentage`/`formula`: `'list_price'`, `'standard_price'`, `'pricelist'`. |
| `base_pricelist_id` | `Many2one` (`product.pricelist`) | Conditional | `false` | Base pricelist when `base = 'pricelist'`. |
| `price_discount` | `Float` | Optional | `0.0` | Formula discount %. Negative value = markup. |
| `price_surcharge` | `Float` | Optional | `0.0` | Formula fixed amount added after discount/rounding. |
| `price_round` | `Float` | Optional | `0.0` | Formula rounding step (e.g. `1.0`, `0.05`). |
| `price_min_margin` / `price_max_margin` | `Float` | Optional | `0.0` | Formula min/max margin over cost. |

---

### 3.3 Product Price Fields (`product.template` / `product.product`)

| Model | Field | Description |
| :--- | :--- | :--- |
| `product.template` | `list_price` | Base sales price (company currency). |
| `product.template` | `standard_price` | Cost (used by cost-based formulas). |
| `product.template` | `categ_id` | Internal category (matched by category rules). |
| `product.template` | `product_variant_ids` | Variant IDs — **pricing is always computed per variant**. |
| `product.template` | `compare_list_price` | *(eCommerce, optional)* "Compare to" price used for strike-through display when the *Comparison Price* setting is enabled. |
| `product.product` | `lst_price` | Variant sales price = `list_price` + attribute extra prices. |
| `product.product` | `price_extra` | Sum of attribute value extra prices. |

> [!CAUTION]
> There is **no `price` field** on either model in Odoo 16+. Passing `"context": {"pricelist": 4}` to a product `search_read` does **not** change any returned price.

---

### 3.4 Customer Pricelist (`res.partner`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `property_product_pricelist` | `Many2one` (`product.pricelist`) | Pricelist used for the customer (explicit or computed fallback). Company-dependent. |

---

### 3.5 Quotation Line Price Fields (`sale.order.line`)

Used by **Approach A**. When a line is created without `price_unit`, Odoo computes it from the order's pricelist, quantity, and `date_order`.

| Field | Description |
| :--- | :--- |
| `product_id` | Variant (`product.product`) ID. |
| `product_uom_qty` | Quantity — drives `min_quantity` tier selection. |
| `price_unit` | Unit price from the pricelist (base price if a percentage discount is shown separately). |
| `discount` | Discount % (only for `percentage` rules with the Discounts setting enabled; otherwise `0`). |
| `price_reduce_taxexcl` | Net unit price, tax excluded. |
| `price_subtotal` / `price_total` | Line total excluding / including taxes. |
| `currency_id` | Currency (= pricelist currency). |

---

## 4. Step-by-Step API Integration Workflows

---

### Step 1: Discovering Pricelists

#### 1.1 Active Pricelists

```http
POST /json/2/product.pricelist/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [["active", "=", true]],
  "fields": ["id", "name", "currency_id", "website_id", "selectable", "country_group_ids"],
  "order": "sequence asc, id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "name": "Public Pricelist (USD)",
    "currency_id": [2, "USD"],
    "website_id": false,
    "selectable": true,
    "country_group_ids": []
  },
  {
    "id": 2,
    "name": "Public Pricelist (EUR)",
    "currency_id": [1, "EUR"],
    "website_id": false,
    "selectable": true,
    "country_group_ids": [1]
  },
  {
    "id": 4,
    "name": "B2B Gold Wholesale",
    "currency_id": [2, "USD"],
    "website_id": false,
    "selectable": false,
    "country_group_ids": []
  }
]
```

`search_read` hides archived records by default, so the `active` filter is optional but explicit.

#### 1.2 Pricelists Available on a Website

```http
POST /json/2/product.pricelist/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    "|", ["website_id", "=", false], ["website_id", "=", 1],
    ["selectable", "=", true]
  ],
  "fields": ["id", "name", "currency_id"]
}
```

#### 1.3 A Logged-In Customer's Pricelist

```http
POST /json/2/res.partner/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [1042],
  "fields": ["id", "name", "property_product_pricelist"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1042,
    "name": "Apex Distribution LLC",
    "property_product_pricelist": [4, "B2B Gold Wholesale (USD)"]
  }
]
```

Store `pricelist_id = 4` in the customer's session. In multi-company databases, pass `"context": {"allowed_company_ids": [<company_id>]}` because the value is company-dependent.

---

### Step 2: Computing Product Prices — Approach A: Draft Quotation Simulation

Because the pricing methods are private, the only **exact** way to obtain Odoo's price over JSON-2 is to let Odoo compute it on quotation lines. Create a draft `sale.order` with the pricelist, the customer, and one line per variant/quantity, read the lines, then delete the draft (or reuse it — see the tip below).

This guarantees parity with the price Odoo will apply at checkout (rules, categories, dates, variant extras, currency conversion, rounding).

#### 2.1 Create the Simulation Quotation

`product_id` must be a **variant** ID (`product.product`). Several lines may use the same variant with different quantities.

```http
POST /json/2/sale.order/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "partner_id": 1042,
      "pricelist_id": 4,
      "order_line": [
        [0, 0, { "product_id": 903, "product_uom_qty": 1 }],
        [0, 0, { "product_id": 771, "product_uom_qty": 1 }],
        [0, 0, { "product_id": 781, "product_uom_qty": 1 }]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[815]
```

#### 2.2 Read the Computed Prices

```http
POST /json/2/sale.order.line/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [["order_id", "=", 815]],
  "fields": ["product_id", "product_uom_qty", "price_unit", "discount", "price_reduce_taxexcl", "price_subtotal", "price_total", "currency_id"],
  "order": "id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 2301,
    "product_id": [903, "[SKU-PC-LAP-01] Ultra-Slim 15\" Laptop 16GB RAM"],
    "product_uom_qty": 1.0,
    "price_unit": 1199.0,
    "discount": 15.0,
    "price_reduce_taxexcl": 1019.15,
    "price_subtotal": 1019.15,
    "price_total": 1172.02,
    "currency_id": [2, "USD"]
  },
  {
    "id": 2302,
    "product_id": [771, "[SKU-AUDIO-ANC-01-BLK] Wireless Noise Cancelling Headphones Pro (Black)"],
    "product_uom_qty": 1.0,
    "price_unit": 249.0,
    "discount": 0.0,
    "price_reduce_taxexcl": 249.0,
    "price_subtotal": 249.0,
    "price_total": 286.35,
    "currency_id": [2, "USD"]
  },
  {
    "id": 2303,
    "product_id": [781, "[SKU-AUDIO-SPK-02] Portable Bluetooth Speaker Waterproof"],
    "product_uom_qty": 1.0,
    "price_unit": 79.5,
    "discount": 0.0,
    "price_reduce_taxexcl": 79.5,
    "price_subtotal": 79.5,
    "price_total": 91.43,
    "currency_id": [2, "USD"]
  }
]
```

Here, line 1 was priced by a `percentage` rule (discount shown separately) and line 2 by a `fixed` rule (net price, no discount).

#### 2.3 Delete the Simulation Quotation

```http
POST /json/2/sale.order/unlink HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [815]
}
```

**Success Response (`200 OK`):**
```json
true
```

> [!TIP]
> **Reuse a dedicated "pricing sandbox" quotation.** Each `sale.order/create` consumes a quotation number (`S00815`), leaving gaps in the sequence. Instead, create one draft quotation once and, for each simulation, replace its lines and pricelist in a single `write`:
> ```json
> {
>   "ids": [815],
>   "vals": {
>     "partner_id": 1042,
>     "pricelist_id": 4,
>     "order_line": [[5, 0, 0], [0, 0, {"product_id": 903, "product_uom_qty": 10}]]
>   }
> }
> ```
> `[5, 0, 0]` removes the previous lines; the new lines are priced with the current pricelist. Serialize access to the sandbox (one simulation at a time) or keep one sandbox per worker.

> [!NOTE]
> The API user needs **Sales** access rights (create/unlink on `sale.order`). Taxes in `price_total` depend on the customer's fiscal position.

#### 2.4 Strike-Through Display ("Was $1,199 — Now $1,019.15")

```typescript
const netUnit = line.price_unit * (1 - line.discount / 100);
// Reference price: the variant's lst_price (or compare_list_price if you use it)
const reference = variant.lst_price;
const hasDiscount = netUnit < reference - 0.005;
const discountPct = hasDiscount ? Math.round(((reference - netUnit) / reference) * 100) : 0;
```

> [!NOTE]
> `lst_price` is in the company currency. Only compare it directly when the pricelist uses the same currency.

---

### Step 3: Fetching Pricelist Rules for a Product — Approach B

For catalog grids with many products, running a simulation per page may be too heavy. You can instead read the rules and evaluate them in your backend using the algorithm in [Section 1](#how-odoo-selects-a-pricelist-rule).

> [!WARNING]
> Approach B re-implements Odoo logic (currency conversion, `formula` rounding/margins, `base = 'pricelist'` chains, UoM). Use it for **display**, and always confirm the final price with Approach A or on the actual order before checkout.

#### 3.1 Read the Product Data Needed for Matching

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [["product_tmpl_id", "=", 620]],
  "fields": ["id", "product_tmpl_id", "categ_id", "lst_price", "standard_price"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 903,
    "product_tmpl_id": [620, "Ultra-Slim 15\" Laptop 16GB RAM"],
    "categ_id": [5, "All / Saleable / Computers"],
    "lst_price": 1199.0,
    "standard_price": 780.0
  }
]
```

#### 3.2 Read All Applicable Rules for the Variant

This domain mirrors Odoo's own rule lookup (template, variant, category including parents, and dates). Replace the date with the current UTC time:

```http
POST /json/2/product.pricelist.item/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["pricelist_id", "=", 4],
    "|", ["product_tmpl_id", "=", false], ["product_tmpl_id", "=", 620],
    "|", ["product_id", "=", false], ["product_id", "=", 903],
    "|", ["categ_id", "=", false], ["categ_id", "parent_of", 5],
    "|", ["date_start", "=", false], ["date_start", "<=", "2026-11-12 11:55:00"],
    "|", ["date_end", "=", false], ["date_end", ">=", "2026-11-12 11:55:00"]
  ],
  "fields": [
    "id", "applied_on", "categ_id", "product_tmpl_id", "product_id", "min_quantity",
    "compute_price", "fixed_price", "percent_price", "base", "base_pricelist_id",
    "price_discount", "price_surcharge", "price_round", "price_min_margin", "price_max_margin",
    "date_start", "date_end"
  ],
  "order": "applied_on, min_quantity desc, categ_id desc, id desc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 89,
    "applied_on": "1_product",
    "categ_id": false,
    "product_tmpl_id": [620, "Ultra-Slim 15\" Laptop 16GB RAM"],
    "product_id": false,
    "min_quantity": 10.0,
    "compute_price": "percentage",
    "fixed_price": 0.0,
    "percent_price": 25.0,
    "base": "list_price",
    "base_pricelist_id": false,
    "price_discount": 0.0,
    "price_surcharge": 0.0,
    "price_round": 0.0,
    "price_min_margin": 0.0,
    "price_max_margin": 0.0,
    "date_start": false,
    "date_end": false
  },
  {
    "id": 88,
    "applied_on": "1_product",
    "categ_id": false,
    "product_tmpl_id": [620, "Ultra-Slim 15\" Laptop 16GB RAM"],
    "product_id": false,
    "min_quantity": 0.0,
    "compute_price": "percentage",
    "fixed_price": 0.0,
    "percent_price": 15.0,
    "base": "list_price",
    "base_pricelist_id": false,
    "price_discount": 0.0,
    "price_surcharge": 0.0,
    "price_round": 0.0,
    "price_min_margin": 0.0,
    "price_max_margin": 0.0,
    "date_start": false,
    "date_end": false
  },
  {
    "id": 61,
    "applied_on": "3_global",
    "categ_id": false,
    "product_tmpl_id": false,
    "product_id": false,
    "min_quantity": 0.0,
    "compute_price": "percentage",
    "fixed_price": 0.0,
    "percent_price": 5.0,
    "base": "list_price",
    "base_pricelist_id": false,
    "price_discount": 0.0,
    "price_surcharge": 0.0,
    "price_round": 0.0,
    "price_min_margin": 0.0,
    "price_max_margin": 0.0,
    "date_start": false,
    "date_end": false
  }
]
```

**Evaluation for quantity 3:** walk the list in order and take the first rule with `min_quantity ≤ 3` → rule `88` → 1199.00 × (1 − 0.15) = **1019.15**. For quantity 12 → rule `89` → **899.25**. The global rule `61` only applies to products without a more specific matching rule.

#### 3.3 Only Rules Explicitly Attached to a Template or Variant

For admin/audit screens:

```http
POST /json/2/product.pricelist.item/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["pricelist_id", "=", 4],
    "|", ["product_tmpl_id", "=", 620], ["product_id.product_tmpl_id", "=", 620]
  ],
  "fields": ["id", "applied_on", "product_id", "min_quantity", "compute_price", "fixed_price", "percent_price", "date_start", "date_end"],
  "order": "applied_on, min_quantity asc"
}
```

---

### Step 4: Volume Tier Tables for Product Pages

| Quantity | Unit Price | Savings |
| :--- | :--- | :--- |
| 1 – 9 | $1,019.15 | 15% |
| 10+ | $899.25 | 25% |

1. Get the distinct `min_quantity` thresholds from the applicable rules (Step 3.2), plus `1`.
2. Price each threshold exactly with **one** simulation containing one line per threshold:

```http
POST /json/2/sale.order/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [815],
  "vals": {
    "partner_id": 1042,
    "pricelist_id": 4,
    "order_line": [
      [5, 0, 0],
      [0, 0, { "product_id": 903, "product_uom_qty": 1 }],
      [0, 0, { "product_id": 903, "product_uom_qty": 10 }]
    ]
  }
}
```

3. Read the lines (Step 2.2) and build the table with `price_unit × (1 − discount / 100)`.

Cache the table per `(pricelist_id, variant_id)` and let the frontend pick the tier as the quantity input changes.

> [!NOTE]
> Lines of the same product on one order are priced independently by their own quantity; Odoo does not sum them for tier selection.

---

### Step 5: Time-Limited Promotional Prices

#### 5.1 Rules Active Now

```http
POST /json/2/product.pricelist.item/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["pricelist_id", "=", 1],
    ["date_end", "!=", false],
    ["date_start", "<=", "2026-11-12 11:55:00"],
    ["date_end", ">=", "2026-11-12 11:55:00"]
  ],
  "fields": ["id", "applied_on", "product_tmpl_id", "product_id", "categ_id", "compute_price", "percent_price", "fixed_price", "date_start", "date_end"]
}
```

`date_start` / `date_end` are UTC datetimes. Use `date_end` to show countdown timers.

#### 5.2 Preview a Future Price (e.g. Black Friday)

Quotation lines are priced at the order's `date_order`. Set it on the simulation quotation:

```http
POST /json/2/sale.order/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [815],
  "vals": {
    "pricelist_id": 1,
    "date_order": "2026-11-27 08:00:00",
    "order_line": [[5, 0, 0], [0, 0, { "product_id": 903, "product_uom_qty": 1 }]]
  }
}
```

Reset `date_order` to the current time before the next real-time simulation.

---

### Step 6: Creating & Updating Pricelist Rules

#### 6.1 Fixed Price for a Product

```http
POST /json/2/product.pricelist.item/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "pricelist_id": 1,
      "applied_on": "1_product",
      "product_tmpl_id": 620,
      "compute_price": "fixed",
      "fixed_price": 1099.0
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[112]
```

#### 6.2 Percentage Discount on an Internal Category

```http
POST /json/2/product.pricelist.item/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "pricelist_id": 4,
      "applied_on": "2_product_category",
      "categ_id": 5,
      "compute_price": "percentage",
      "percent_price": 20.0
    }
  ]
}
```

#### 6.3 Quantity Tier and Time-Limited Promotion

```http
POST /json/2/product.pricelist.item/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "pricelist_id": 4,
      "applied_on": "1_product",
      "product_tmpl_id": 620,
      "min_quantity": 50.0,
      "compute_price": "percentage",
      "percent_price": 30.0
    },
    {
      "pricelist_id": 1,
      "applied_on": "0_product_variant",
      "product_id": 903,
      "compute_price": "percentage",
      "percent_price": 20.0,
      "date_start": "2026-11-27 00:00:00",
      "date_end": "2026-11-30 23:59:59"
    }
  ]
}
```

#### 6.4 Update an Existing Rule

```http
POST /json/2/product.pricelist.item/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [112],
  "vals": { "fixed_price": 1049.0 }
}
```

> [!NOTE]
> Rule changes do **not** reprice existing quotations. Call `sale.order/action_update_prices` (see [6.4](#64-changing-the-pricelist-of-an-existing-quotation)) on open quotations if needed.

---

## 5. Complete Code Implementations

### 5.1 cURL

#### Simulate a Price
```bash
curl -X POST "https://mycompany.odoo.com/json/2/sale.order/create" \
  -H "Authorization: bearer <API_KEY>" \
  -H "X-Odoo-Database: mycompany_db" \
  -H "Content-Type: application/json" \
  -d '{
    "vals_list": [{
      "partner_id": 1042,
      "pricelist_id": 4,
      "order_line": [[0, 0, {"product_id": 903, "product_uom_qty": 10}]]
    }]
  }'
# then: sale.order.line/search_read with [["order_id", "=", <id>]], then sale.order/unlink
```

#### Applicable Rules for a Variant
```bash
curl -X POST "https://mycompany.odoo.com/json/2/product.pricelist.item/search_read" \
  -H "Authorization: bearer <API_KEY>" \
  -H "X-Odoo-Database: mycompany_db" \
  -H "Content-Type: application/json" \
  -d '{
    "domain": [
      ["pricelist_id", "=", 4],
      "|", ["product_tmpl_id", "=", false], ["product_tmpl_id", "=", 620],
      "|", ["product_id", "=", false], ["product_id", "=", 903],
      "|", ["categ_id", "=", false], ["categ_id", "parent_of", 5]
    ],
    "fields": ["id", "applied_on", "min_quantity", "compute_price", "fixed_price", "percent_price", "date_start", "date_end"],
    "order": "applied_on, min_quantity desc, categ_id desc, id desc"
  }'
```

---

### 5.2 Python Client: Price Simulation & Tier Table

```python
#!/usr/bin/env python3
"""Odoo pricelist client (JSON-2 API) — exact prices via draft quotation simulation."""

from typing import Any, Dict, List, Optional, Tuple

import requests


class OdooPricelistClient:
    def __init__(self, base_url: str, db: str, api_key: str, sandbox_order_id: Optional[int] = None):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"bearer {api_key}",
            "X-Odoo-Database": db,
            "Content-Type": "application/json; charset=utf-8",
            "User-Agent": "Odoo-Pricing-Engine/1.0",
        }
        # Optional reusable draft quotation (avoids consuming sequence numbers)
        self.sandbox_order_id = sandbox_order_id

    def _call(self, model: str, method: str, payload: Dict[str, Any]) -> Any:
        res = requests.post(f"{self.base_url}/json/2/{model}/{method}",
                            headers=self.headers, json=payload, timeout=15)
        res.raise_for_status()
        return res.json()

    def get_customer_pricelist_id(self, partner_id: int) -> Optional[int]:
        partners = self._call("res.partner", "read", {
            "ids": [partner_id], "fields": ["property_product_pricelist"],
        })
        pricelist = partners[0]["property_product_pricelist"] if partners else False
        return pricelist[0] if pricelist else None

    def simulate_prices(
        self,
        partner_id: int,
        pricelist_id: int,
        lines: List[Tuple[int, float]],          # (variant_id, quantity)
        date_order: Optional[str] = None,        # "YYYY-MM-DD HH:MM:SS" UTC
    ) -> List[Dict[str, Any]]:
        """Return Odoo-computed prices for each (variant, quantity) pair."""
        vals: Dict[str, Any] = {
            "partner_id": partner_id,
            "pricelist_id": pricelist_id,
            "order_line": [[0, 0, {"product_id": v, "product_uom_qty": q}] for v, q in lines],
        }
        if date_order:
            vals["date_order"] = date_order

        if self.sandbox_order_id:
            vals["order_line"].insert(0, [5, 0, 0])  # clear previous lines
            self._call("sale.order", "write", {"ids": [self.sandbox_order_id], "vals": vals})
            order_id, delete_after = self.sandbox_order_id, False
        else:
            order_id = self._call("sale.order", "create", {"vals_list": [vals]})[0]
            delete_after = True

        try:
            order_lines = self._call("sale.order.line", "search_read", {
                "domain": [["order_id", "=", order_id]],
                "fields": ["product_id", "product_uom_qty", "price_unit", "discount",
                           "price_subtotal", "price_total", "currency_id"],
                "order": "id asc",
            })
        finally:
            if delete_after:
                self._call("sale.order", "unlink", {"ids": [order_id]})

        return [
            {
                "variant_id": l["product_id"][0],
                "quantity": l["product_uom_qty"],
                "price_unit": l["price_unit"],
                "discount": l["discount"],
                "net_unit_price": round(l["price_unit"] * (1 - l["discount"] / 100), 2),
                "subtotal": l["price_subtotal"],
                "total": l["price_total"],
                "currency": l["currency_id"][1],
            }
            for l in order_lines
        ]

    def get_tier_table(self, partner_id: int, pricelist_id: int, variant_id: int) -> List[Dict[str, Any]]:
        """Volume tier table for a product page."""
        variant = self._call("product.product", "read", {
            "ids": [variant_id], "fields": ["product_tmpl_id", "categ_id", "lst_price"],
        })[0]
        rules = self._call("product.pricelist.item", "search_read", {
            "domain": [
                ["pricelist_id", "=", pricelist_id],
                "|", ["product_tmpl_id", "=", False], ["product_tmpl_id", "=", variant["product_tmpl_id"][0]],
                "|", ["product_id", "=", False], ["product_id", "=", variant_id],
                "|", ["categ_id", "=", False], ["categ_id", "parent_of", variant["categ_id"][0]],
            ],
            "fields": ["min_quantity"],
        })
        thresholds = sorted({1.0, *(r["min_quantity"] for r in rules if r["min_quantity"] > 1)})

        prices = self.simulate_prices(partner_id, pricelist_id, [(variant_id, q) for q in thresholds])
        table = []
        for i, p in enumerate(prices):
            nxt = thresholds[i + 1] if i + 1 < len(thresholds) else None
            label = f"{p['quantity']:g}+" if nxt is None else f"{p['quantity']:g} - {nxt - 1:g}"
            table.append({"label": label, "min_qty": p["quantity"], "unit_price": p["net_unit_price"]})
        return table


if __name__ == "__main__":
    client = OdooPricelistClient("https://mycompany.odoo.com", "mycompany_db", "<API_KEY>")
    pricelist_id = client.get_customer_pricelist_id(1042) or 1
    for row in client.simulate_prices(1042, pricelist_id, [(903, 1), (903, 10)]):
        print(row)
    for tier in client.get_tier_table(1042, pricelist_id, 903):
        print(f"{tier['label']}: {tier['unit_price']}")
```

---

### 5.3 TypeScript / Node.js Pricing Service

```typescript
import axios, { AxiosInstance } from "axios";

export interface SimulatedPrice {
  variantId: number;
  quantity: number;
  priceUnit: number;
  discount: number;
  netUnitPrice: number;
  subtotal: number;
  total: number;
  currency: string;
}

export class OdooPricelistService {
  private client: AxiosInstance;

  constructor(baseUrl: string, db: string, apiKey: string, private sandboxOrderId?: number) {
    this.client = axios.create({
      baseURL: `${baseUrl.replace(/\/$/, "")}/json/2`,
      headers: {
        Authorization: `bearer ${apiKey}`,
        "X-Odoo-Database": db,
        "Content-Type": "application/json; charset=utf-8",
      },
      timeout: 15000,
    });
  }

  private async call<T>(model: string, method: string, payload: object): Promise<T> {
    const { data } = await this.client.post<T>(`/${model}/${method}`, payload);
    return data;
  }

  /** Exact Odoo prices for (variant, quantity) pairs via a draft quotation. */
  async simulatePrices(
    partnerId: number,
    pricelistId: number,
    lines: Array<{ variantId: number; quantity: number }>
  ): Promise<SimulatedPrice[]> {
    const orderLine: unknown[] = lines.map((l) => [0, 0, { product_id: l.variantId, product_uom_qty: l.quantity }]);
    const vals = { partner_id: partnerId, pricelist_id: pricelistId, order_line: orderLine };

    let orderId: number;
    let deleteAfter = false;
    if (this.sandboxOrderId) {
      orderId = this.sandboxOrderId;
      await this.call("sale.order", "write", { ids: [orderId], vals: { ...vals, order_line: [[5, 0, 0], ...orderLine] } });
    } else {
      [orderId] = await this.call<number[]>("sale.order", "create", { vals_list: [vals] });
      deleteAfter = true;
    }

    try {
      const rows = await this.call<any[]>("sale.order.line", "search_read", {
        domain: [["order_id", "=", orderId]],
        fields: ["product_id", "product_uom_qty", "price_unit", "discount", "price_subtotal", "price_total", "currency_id"],
        order: "id asc",
      });
      return rows.map((r) => ({
        variantId: r.product_id[0],
        quantity: r.product_uom_qty,
        priceUnit: r.price_unit,
        discount: r.discount,
        netUnitPrice: Math.round(r.price_unit * (1 - r.discount / 100) * 100) / 100,
        subtotal: r.price_subtotal,
        total: r.price_total,
        currency: r.currency_id[1],
      }));
    } finally {
      if (deleteAfter) {
        await this.call("sale.order", "unlink", { ids: [orderId] });
      }
    }
  }
}
```

---

## 6. Error Handling & Edge Cases

### 6.1 Requesting Removed Fields (`price`, `discount_policy`)

```http
POST /json/2/product.template/search_read HTTP/1.1
...
{ "domain": [["id", "=", 620]], "fields": ["price"], "context": {"pricelist": 4} }
```

```json
{
  "name": "builtins.ValueError",
  "message": "Invalid field 'price' on model 'product.template'",
  "arguments": ["Invalid field 'price' on model 'product.template'"],
  "context": {},
  "debug": "Traceback (most recent call last): ..."
}
```

**Fix:** Use Approach A (Step 2) or B (Step 3). Same error for `discount_policy` on `product.pricelist`.

---

### 6.2 Calling Private Pricing Methods (`403 Forbidden`)

```http
POST /json/2/product.pricelist/_get_product_price HTTP/1.1
```

```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "Private methods (such as product.pricelist._get_product_price) cannot be called remotely.",
  "arguments": ["Private methods (such as product.pricelist._get_product_price) cannot be called remotely."],
  "context": {},
  "debug": "..."
}
```

**Fix:** Do not call `_`-prefixed methods. If you need a direct price endpoint, add a small custom module exposing a **public** method that wraps `_get_product_price` with proper access checks.

---

### 6.3 Unknown or Archived Pricelist

- `search_read` excludes archived pricelists by default; `read` by ID still returns them (`"active": false`).
- Writing a non-existent `pricelist_id` on a quotation fails with a database/foreign-key error and the whole request is rolled back.
- The ORM does **not** stop you from assigning an archived pricelist — validate `active` before using a stored pricelist ID.

---

### 6.4 Changing the Pricelist of an Existing Quotation

A quotation's `currency_id` follows its `pricelist_id`, so there is no currency-mismatch error. However, **changing `pricelist_id` does not reprice existing lines**. Call:

```http
POST /json/2/sale.order/action_update_prices HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [782]
}
```

Lines with a manually set `price_unit` can be kept by Odoo; replacing the lines (as in the sandbox pattern) avoids any ambiguity.

---

### 6.5 Cost-Based Formula with Zero Cost

With `base = "standard_price"`, a product whose `standard_price` is `0.0` gets a price equal to the surcharge only (often `0.00`). Validate `standard_price > 0` before attaching products to cost-plus pricelists.

---

### 6.6 Authentication & Permission Failures

#### `401 Unauthorized`
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "..."
}
```

#### `403 Forbidden`
Creating/editing rules requires **Sales: Administrator** (or equivalent pricelist management rights); simulations require Sales user rights:
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to create 'Pricelist Rule' (product.pricelist.item) records. ...",
  "arguments": ["You are not allowed to create 'Pricelist Rule' (product.pricelist.item) records. ..."],
  "context": {},
  "debug": "..."
}
```

Also make sure the **Pricelists** feature is enabled (*Sales → Configuration → Settings → Pricelists*); otherwise only the default pricelist is used.

---

## 7. Production Best Practices for E-Commerce Developers

### 1. Let Odoo Compute the Checkout Price
Use Approach A for carts and checkout. When you create the real `sale.order`, set `pricelist_id` and **omit `price_unit`** on lines so Odoo prices them itself.

### 2. Use Approach B Only for Display, and Cache It
For large catalog grids, evaluate rules in your backend and cache prices per `(pricelist_id, variant_id, quantity tier)` with a short TTL (5–15 min). Invalidate when `product.pricelist.item`, `product.template.list_price`, or attribute extra prices change (poll `write_date`).

### 3. Reuse a Sandbox Quotation
Avoid creating and deleting quotations for every simulation: it consumes quotation numbers and adds load. Replace the lines of a dedicated draft quotation with `[5, 0, 0]` + `[0, 0, {...}]`.

### 4. Batch Simulations
Put many variants/quantities in **one** simulation request (one line each) instead of one request per product.

### 5. Price Per Variant
Pricing is computed on `product.product`. Templates with variants can have different prices per variant (attribute extras, variant rules) — always simulate with variant IDs.

### 6. Multi-Currency
Each pricelist has one currency. When a shopper switches currency, switch to a pricelist with that `currency_id`; the order currency follows the pricelist automatically.
