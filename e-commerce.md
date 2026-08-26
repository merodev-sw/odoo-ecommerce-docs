# Odoo JSON-2 API: E-Commerce Promotions, Discounts, Coupons & Loyalty Guide

This guide provides an end-to-end technical reference for backend developers integrating an **e-commerce platform** with Odoo's **Promotions, Discount Codes, Coupon Programs, Loyalty Points, Gift Cards, and Dynamic Pricelists (`loyalty.program`, `loyalty.rule`, `loyalty.reward`, `loyalty.card`, `sale.order`, `product.pricelist`)** using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's modern **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Promotions & Loyalty Architecture Overview](#1-promotions--loyalty-architecture-overview)
   - Unified Loyalty & Discount Engine in Odoo
   - Key Concepts & Entity Relationship Diagram
   - Program Types Comparison
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
   - Required HTTP Headers
   - Request & Parameter Conventions
3. [Model & Field References](#3-model--field-references)
   - [3.1 Loyalty & Promotion Program (`loyalty.program`)](#31-loyalty--promotion-program-loyaltyprogram)
   - [3.2 Program Rules & Conditions (`loyalty.rule`)](#32-program-rules--conditions-loyaltyrule)
   - [3.3 Program Rewards & Discounts (`loyalty.reward`)](#33-program-rewards--discounts-loyaltyreward)
   - [3.4 Coupons, Gift Cards & Loyalty Cards (`loyalty.card`)](#34-coupons-gift-cards--loyalty-cards-loyaltycard)
   - [3.5 Sales Order Reward Lines (`sale.order` & `sale.order.line`)](#35-sales-order-reward-lines-saleorder--saleorderline)
   - [3.6 Promotional Pricelists & Price Rules (`product.pricelist` & `product.pricelist.item`)](#36-promotional-pricelists--price-rules-productpricelist--productpricelistitem)
4. [Step-by-Step E-Commerce Promotion Workflows](#4-step-by-step-e-commerce-promotion-workflows)
   - [Step 1: Create a Global Promo Code Campaign (`SUMMER20` - 20% Off Cart)](#step-1-create-a-global-promo-code-campaign-summer20---20-off-cart)
   - [Step 2: Create a Unique Single-Use Coupon Program & Batch Generate Codes](#step-2-create-a-unique-single-use-coupon-program--batch-generate-codes)
   - [Step 3: Setup "Buy X Get Y Free" (BOGO) Promotion](#step-3-setup-buy-x-get-y-free-bogo-promotion)
   - [Step 4: Setup Free Shipping Promotion](#step-4-setup-free-shipping-promotion)
   - [Step 5: Setup Digital Gift Cards & Store Balance](#step-5-setup-digital-gift-cards--store-balance)
   - [Step 6: Checkout Flow: Validating & Applying Promo Codes to a Cart](#step-6-checkout-flow-validating--applying-promo-codes-to-a-cart)
     - [6.1 Code Verification & Rule Validation](#61-code-verification--rule-validation)
     - [6.2 Injecting Reward Discount Lines into `sale.order.line`](#62-injecting-reward-discount-lines-into-saleorderline)
     - [6.3 Applying Direct Item-Level Percentage Discounts](#63-applying-direct-item-level-percentage-discounts)
   - [Step 7: Customer Loyalty Points Accrual & Reward Redemption](#step-7-customer-loyalty-points-accrual--reward-redemption)
   - [Step 8: Order Confirmation, Coupon Consumption & Cancellation/Refunds](#step-8-order-confirmation-coupon-consumption--cancellationrefunds)
5. [Dynamic Pricelist Integration (B2B Tiers, Wholesale, Volume Discounts)](#5-dynamic-pricelist-integration-b2b-tiers-wholesale-volume-discounts)
   - [Step 1: Querying Customer-Specific Pricelists](#step-1-querying-customer-specific-pricelists)
   - [Step 2: Setting up Volume Tier Discounts (`product.pricelist.item`)](#step-2-setting-up-volume-tier-discounts-productpricelistitem)
   - [Step 3: Applying Pricelists to Sales Orders](#step-3-applying-pricelists-to-sales-orders)
6. [Complete HTTP Request & Response Reference](#6-complete-http-request--response-reference)
7. [Error Handling & Edge Case Failure Responses](#7-error-handling--edge-case-failure-responses)
   - [7.1 Invalid Promo Code (`404 Not Found` / Custom Validation)](#71-invalid-promo-code-404-not-found--custom-validation)
   - [7.2 Expired Promo Code or Coupon (`400 Bad Request`)](#72-expired-promo-code-or-coupon-400-bad-request)
   - [7.3 Coupon Usage Limit Exceeded (`400 Bad Request`)](#73-coupon-usage-limit-exceeded-400-bad-request)
   - [7.4 Minimum Order Spend Not Met (`400 Bad Request`)](#74-minimum-order-spend-not-met-400-bad-request)
   - [7.5 Incompatible / Excluded Products in Cart (`400 Bad Request`)](#75-incompatible--excluded-products-in-cart-400-bad-request)
   - [7.6 Authentication & Permission Failures (`401 Unauthorized` / `403 Forbidden`)](#76-authentication--permission-failures-401-unauthorized--403-forbidden)
8. [Production Best Practices for E-Commerce Developers](#8-production-best-practices-for-e-commerce-developers)

---

## 1. Promotions & Loyalty Architecture Overview

Odoo unifies all promotional mechanisms—including promo codes, discount coupons, loyalty point programs, Buy X Get Y offers, free shipping vouchers, and digital gift cards—into a single modular engine powered by **`loyalty.program`**.

### Architecture & Entity Relationships

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           loyalty.program                                   │
│  - program_type: promo_code | coupons | loyalty | gift_card | buy_x_get_y   │
│  - trigger: with_code | auto                                                │
│  - date_from / date_to / limit_usage / max_usage                            │
└──────────────────────┬───────────────────────────────┬──────────────────────┘
                       │ 1:N                           │ 1:N
                       ▼                               ▼
       ┌───────────────────────────────┐ ┌───────────────────────────────┐
       │         loyalty.rule          │ │        loyalty.reward         │
       │ - code (e.g. "SUMMER20")      │ │ - reward_type: discount |     │
       │ - minimum_amount              │ │   product | shipping          │
       │ - product_ids / categ_id      │ │ - discount: percentage/fixed  │
       │ - minimum_qty                 │ │ - discount_applicability      │
       └───────────────────────────────┘ └───────────────────────────────┘
                       │                               │
                       │ Used to validate and claim    │ Creates reward lines
                       ▼                               ▼
       ┌───────────────────────────────┐ ┌───────────────────────────────┐
       │         loyalty.card          │ │        sale.order.line        │
       │ - code: unique coupon string  │ │ - product_id (Reward Item)    │
       │ - partner_id (optional owner) │ │ - price_unit: negative amount │
       │ - points / balance            │ │ - is_reward_line: true        │
       │ - expiration_date / use_count │ │ - reward_id / coupon_id       │
       └───────────────────────────────┘ └───────────────────────────────┘
```

### Program Types Comparison

| Program Type (`program_type`) | Trigger (`trigger`) | Coupon Generation (`loyalty.card`) | E-Commerce Use Case |
| :--- | :--- | :--- | :--- |
| **`promo_code`** | `with_code` | Not needed (Code stored on `loyalty.rule.code`). | Public marketing discount codes (e.g., `SUMMER20`, `WELCOME10`). |
| **`coupons`** | `with_code` | Generated in advance (`loyalty.card`). | Unique, single-use vouchers sent via email campaigns or influencers. |
| **`loyalty`** | `auto` | Generated per registered customer (`partner_id`). | Points-for-purchases rewards program (e.g., 1 point per $1 spent). |
| **`gift_card`** | `with_code` | Generated on purchase or manual load. | Prepaid store credit redeemed against checkout totals. |
| **`buy_x_get_y`** | `auto` or `with_code` | Optional. | BOGO promotions (e.g., Buy 2 T-Shirts, Get 1 Cap Free). |
| **`next_order_coupons`** | `auto` | Generated upon order payment. | Post-purchase incentives (e.g., "Get $10 off your next order"). |

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

### Required Headers
```http
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-Ecommerce-Promotions/1.0
```

### Request Structure
- **Model Methods (`create`, `search_read`, `name_search`):** Arguments passed as named JSON properties (e.g. `{"vals_list": [...]}`, `{"domain": [...]}`).
- **Recordset Methods (`write`, `read`, `unlink`):** Include `"ids": [id1, id2]` alongside named arguments or values.

---

## 3. Model & Field References

### 3.1 Loyalty & Promotion Program (`loyalty.program`)

The parent program defining campaign rules, validity schedules, and execution modes.

| Field Name | Type | Requirement | Default | Description & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | — | Campaign name (e.g., `Summer Flash Sale 20% Off`). |
| `program_type` | `Selection` | **REQUIRED** | `'promo_code'` | Program archetype: `'promo_code'`, `'coupons'`, `'loyalty'`, `'gift_card'`, `'buy_x_get_y'`, `'next_order_coupons'`. |
| `trigger` | `Selection` | **REQUIRED** | `'with_code'` | Activation mechanism: `'with_code'` (code required) or `'auto'` (rules apply automatically). |
| `applies_on` | `Selection` | **REQUIRED** | `'current'` | When reward applies: `'current'` (on active checkout) or `'future'` (generates coupon for next order). |
| `rule_ids` | `One2many` (`loyalty.rule`) | Optional | `[]` | Condition rules required to activate this program. |
| `reward_ids` | `One2many` (`loyalty.reward`) | Optional | `[]` | Benefits awarded when rules are satisfied. |
| `coupon_ids` | `One2many` (`loyalty.card`) | Optional | `[]` | Individual unique coupons / gift cards attached to this program. |
| `date_from` | `Date` | Optional | `null` | Promotion start date. |
| `date_to` | `Date` | Optional | `null` | Promotion expiration date. |
| `limit_usage` | `Boolean` | Optional | `false` | If `true`, enforces `max_usage`. |
| `max_usage` | `Integer` | Optional | `0` | Total redemption cap across all customers (e.g., First 500 customers). |
| `total_order_count`| `Integer` | *Computed* | `0` | Number of confirmed orders that used this promotion. |
| `portal_point_name`| `Char` | Optional | `'Points'` | Unit label for loyalty programs (e.g., `Points`, `Stars`, `Credits`). |
| `currency_id` | `Many2one` (`res.currency`) | Optional | Company Currency | Currency used for minimum amounts and fixed rewards. |
| `active` | `Boolean` | Optional | `true` | Active status. Set to `false` to disable program instantly. |

---

### 3.2 Program Rules & Conditions (`loyalty.rule`)

Defines the criteria that a shopping cart must satisfy to qualify for rewards.

| Field Name | Type | Requirement | Default | Description & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `program_id` | `Many2one` (`loyalty.program`) | **REQUIRED** | — | Parent program ID. |
| `code` | `Char` | Optional | `null` | **Promo Code String** (e.g. `SUMMER20`, `FREESHIP`). Used when `trigger = 'with_code'`. |
| `minimum_amount` | `Float` | Optional | `0.0` | Minimum cart subtotal required to trigger reward. |
| `minimum_amount_tax_mode` | `Selection` | Optional | `'incl'` | Tax basis for minimum spend: `'incl'` (Tax-Included) or `'excl'` (Tax-Excluded). |
| `minimum_qty` | `Integer` | Optional | `0` | Minimum quantity of qualifying items in cart. |
| `product_ids` | `Many2many` (`product.product`) | Optional | `[]` | Restrict eligibility to specific product variants. |
| `product_category_id` | `Many2one` (`product.category`) | Optional | `null` | Restrict eligibility to a product category. |
| `product_tag_id` | `Many2one` (`product.tag`) | Optional | `null` | Restrict eligibility to tagged products. |
| `reward_point_mode` | `Selection` | Optional | `'order'` | Mode for earning points: `'order'` (per order), `'money'` (per $ spent), `'unit'` (per item). |
| `reward_point_amount`| `Float` | Optional | `1.0` | Points awarded per order / money / unit. |

---

### 3.3 Program Rewards & Discounts (`loyalty.reward`)

Defines the benefit given to the customer when rules are met.

| Field Name | Type | Requirement | Default | Description & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `program_id` | `Many2one` (`loyalty.program`) | **REQUIRED** | — | Parent program ID. |
| `reward_type` | `Selection` | **REQUIRED** | `'discount'` | Type of benefit: `'discount'`, `'product'` (Free gift / BOGO), `'shipping'` (Free delivery). |
| `discount` | `Float` | Optional | `10.0` | Discount magnitude (e.g., `20.0` for 20% or `15.00` for $15). |
| `discount_mode` | `Selection` | Optional | `'percent'` | Calculation mode: `'percent'` (Percentage discount), `'per_point'` (Loyalty point exchange), `'per_order'` (Fixed monetary amount). |
| `discount_applicability` | `Selection` | Optional | `'order'` | Scope: `'order'` (Whole cart subtotal), `'cheapest'` (Cheapest item), `'specific'` (Specific products). |
| `discount_product_ids` | `Many2many` (`product.product`) | Optional | `[]` | Specific products that receive the discount (when applicability = `'specific'`). |
| `discount_product_category_id` | `Many2one` (`product.category`) | Optional | `null` | Product category receiving the discount. |
| `discount_max_amount` | `Float` | Optional | `0.0` | Maximum cap on discount value (e.g., 20% off up to $50 max). `0.0` = uncapped. |
| `reward_product_id` | `Many2one` (`product.product`) | **REQUIRED** | Service Product | The service product added as a negative line item on `sale.order.line`. |
| `required_points` | `Float` | Optional | `1.0` | Loyalty points or coupon credits consumed to grant this reward. |
| `clear_wallet` | `Boolean` | Optional | `false` | If `true`, consumes all available card points when claimed. |

---

### 3.4 Coupons, Gift Cards & Loyalty Cards (`loyalty.card`)

Represents an individual physical or digital card, unique coupon voucher, or customer loyalty account.

| Field Name | Type | Requirement | Default | Description & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `program_id` | `Many2one` (`loyalty.program`) | **REQUIRED** | — | Associated program ID. |
| `code` | `Char` | **REQUIRED** | Auto-generated | Unique coupon / voucher code string (e.g., `COUP-9821-4401`, `GIFT-250-99A1`). |
| `partner_id` | `Many2one` (`res.partner`) | Optional | `null` | Assigned customer ID. If null, the code is transferable to anyone. |
| `points` | `Float` | Optional | `0.0` | Balance of loyalty points or remaining gift card monetary value. |
| `expiration_date` | `Date` | Optional | `null` | Expiration date of this specific voucher card. |
| `use_count` | `Integer` | *System/Computed* | `0` | Number of times this specific card has been redeemed. |

---

### 3.5 Sales Order Reward Lines (`sale.order` & `sale.order.line`)

In Odoo, discounts applied via promotions or coupon rewards are represented on the order as dedicated line items (`sale.order.line`) with a negative `price_unit`.

| Field Name | Type | Requirement | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- |
| `order_id` | `Many2one` (`sale.order`) | **REQUIRED** | Target sales order ID. |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** | The discount reward service product (matches `loyalty.reward.reward_product_id`). |
| `name` | `Char` | **REQUIRED** | Line description (e.g., `20% discount on order [SUMMER20]`). |
| `price_unit` | `Float` | **REQUIRED** | **Negative monetary amount** representing the total discount (e.g., `-25.00`). |
| `product_uom_qty` | `Float` | **REQUIRED** | Always `1.0`. |
| `is_reward_line` | `Boolean` | Optional | Set to `true` to flag line as an automated promotional reward. |
| `reward_id` | `Many2one` (`loyalty.reward`) | Optional | Linked reward definition ID. |
| `coupon_id` | `Many2one` (`loyalty.card`) | Optional | Linked coupon card ID (for unique voucher redemptions). |
| `discount` | `Float` | Optional | Direct percentage discount on regular product lines (e.g., `15.0` for 15% off). |

---

### 3.6 Promotional Pricelists & Price Rules (`product.pricelist` & `product.pricelist.item`)

Used for rule-based catalog price changes, B2B wholesale tiers, and date-restricted automated discounts without requiring a coupon code.

#### `product.pricelist`
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | Pricelist display name (e.g. `Wholesale VIP - 15% Off`). |
| `currency_id` | `Many2one` (`res.currency`) | **REQUIRED** | Currency for price calculations. |
| `item_ids` | `One2many` (`product.pricelist.item`) | Optional | List of rules applied by this pricelist. |
| `discount_policy` | `Selection` | Optional | `'with_discount'` (show discounted price directly) or `'without_discount'` (show original price with strikethrough discount %). |

#### `product.pricelist.item`
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `pricelist_id` | `Many2one` (`product.pricelist`) | **REQUIRED** | Parent pricelist ID. |
| `applied_on` | `Selection` | **REQUIRED** | Scope: `'3_global'` (All products), `'2_product_category'` (Category), `'1_product'` (Product Template), `'0_product_variant'` (Specific Variant). |
| `product_tmpl_id`| `Many2one` (`product.template`) | Optional | Target product template if `applied_on = '1_product'`. |
| `product_id` | `Many2one` (`product.product`) | Optional | Target variant if `applied_on = '0_product_variant'`. |
| `categ_id` | `Many2one` (`product.category`) | Optional | Target category if `applied_on = '2_product_category'`. |
| `min_quantity` | `Float` | Optional | Minimum purchase quantity to trigger tier price (default `0.0`). |
| `compute_price` | `Selection` | **REQUIRED** | Calculation formula: `'fixed'` (Fixed price override), `'percentage'` (Discount %), `'formula'` (Complex margins/rounding). |
| `fixed_price` | `Float` | Optional | Overridden fixed selling price. |
| `percent_price`| `Float` | Optional | Percentage discount off standard list price (e.g. `10.0` for 10% off). |
| `date_start` | `Datetime` | Optional | Start timestamp of promotion rule. |
| `date_end` | `Datetime` | Optional | End timestamp of promotion rule. |

---

## 4. Step-by-Step E-Commerce Promotion Workflows

---

### Step 1: Create a Global Promo Code Campaign (`SUMMER20` - 20% Off Cart)

Creates a public marketing campaign where any customer typing `SUMMER20` gets 20% off orders over $100.

#### 1.1 Find or Create the Discount Product
Odoo requires a service product to represent the discount line on sales orders:

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["name", "ilike", "Discount"],
    ["type", "=", "service"]
  ],
  "fields": ["id", "name"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 88,
    "name": "Discount"
  }
]
```

#### 1.2 Create the Program with Rules and Rewards
Create the `loyalty.program` record embedding both `rule_ids` (with code `SUMMER20` and min spend $100) and `reward_ids` (20% discount) in a single atomic call:

```http
POST /json/2/loyalty.program/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Summer 2026 Promo Code",
      "program_type": "promo_code",
      "trigger": "with_code",
      "applies_on": "current",
      "date_from": "2026-06-01",
      "date_to": "2026-08-31",
      "limit_usage": true,
      "max_usage": 1000,
      "rule_ids": [
        [
          0,
          0,
          {
            "code": "SUMMER20",
            "minimum_amount": 100.0,
            "minimum_amount_tax_mode": "incl"
          }
        ]
      ],
      "reward_ids": [
        [
          0,
          0,
          {
            "reward_type": "discount",
            "discount": 20.0,
            "discount_mode": "percent",
            "discount_applicability": "order",
            "discount_max_amount": 50.0,
            "reward_product_id": 88
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[12]
```
*(Returns newly created loyalty program ID `12`)*

---

### Step 2: Create a Unique Single-Use Coupon Program & Batch Generate Codes

Used for influencer affiliate campaigns, VIP customer apology codes, or welcome emails.

#### 2.1 Create the Parent Coupon Program
```http
POST /json/2/loyalty.program/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "VIP Single-Use $25 Welcome Vouchers",
      "program_type": "coupons",
      "trigger": "with_code",
      "applies_on": "current",
      "reward_ids": [
        [
          0,
          0,
          {
            "reward_type": "discount",
            "discount": 25.0,
            "discount_mode": "per_order",
            "discount_applicability": "order",
            "reward_product_id": 88
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[13]
```

#### 2.2 Batch Generate Individual Coupon Codes (`loyalty.card/create`)
Insert unique coupon cards linked to program ID `13`:

```http
POST /json/2/loyalty.card/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "program_id": 13,
      "code": "VIP-2026-X99A1",
      "points": 1.0,
      "expiration_date": "2026-12-31"
    },
    {
      "program_id": 13,
      "code": "VIP-2026-B44T8",
      "points": 1.0,
      "expiration_date": "2026-12-31"
    },
    {
      "program_id": 13,
      "code": "VIP-2026-Z71M2",
      "points": 1.0,
      "partner_id": 104,
      "expiration_date": "2026-12-31"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[501, 502, 503]
```

---

### Step 3: Setup "Buy X Get Y Free" (BOGO) Promotion

Buy at least 2 units of Product A (`product_id = 1042`), get 1 unit of Product B (`product_id = 1055`) free automatically.

```http
POST /json/2/loyalty.program/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Buy 2 Headphones Get Free Mouse",
      "program_type": "buy_x_get_y",
      "trigger": "auto",
      "applies_on": "current",
      "rule_ids": [
        [
          0,
          0,
          {
            "minimum_qty": 2,
            "product_ids": [[6, 0, [1042]]]
          }
        ]
      ],
      "reward_ids": [
        [
          0,
          0,
          {
            "reward_type": "product",
            "reward_product_id": 1055,
            "required_points": 1.0
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[14]
```

---

### Step 4: Setup Free Shipping Promotion

Automatically grant free standard delivery on all carts exceeding $75.

```http
POST /json/2/loyalty.program/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Free Shipping on Orders Over $75",
      "program_type": "promo_code",
      "trigger": "auto",
      "applies_on": "current",
      "rule_ids": [
        [
          0,
          0,
          {
            "minimum_amount": 75.0,
            "minimum_amount_tax_mode": "excl"
          }
        ]
      ],
      "reward_ids": [
        [
          0,
          0,
          {
            "reward_type": "shipping",
            "reward_product_id": 88
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[15]
```

---

### Step 5: Setup Digital Gift Cards & Store Balance

#### 5.1 Create the Gift Card Program
```http
POST /json/2/loyalty.program/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "E-Commerce Digital Gift Cards",
      "program_type": "gift_card",
      "trigger": "with_code",
      "applies_on": "current",
      "portal_point_name": "$",
      "reward_ids": [
        [
          0,
          0,
          {
            "reward_type": "discount",
            "discount_mode": "per_point",
            "discount": 1.0,
            "reward_product_id": 88
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[16]
```

#### 5.2 Issue a $100 Digital Gift Card
```http
POST /json/2/loyalty.card/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "program_id": 16,
      "code": "GIFT-100-8842",
      "points": 100.0,
      "partner_id": 104
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[504]
```

---

### Step 6: Checkout Flow: Validating & Applying Promo Codes to a Cart

When a customer submits a promo code (e.g. `SUMMER20`) during checkout on your e-commerce storefront:

```
[Customer Enters Promo Code 'SUMMER20']
                  │
                  ▼
┌───────────────────────────────────────────────────────────┐
│ 1. Lookup code in loyalty.rule OR loyalty.card            │
│ 2. Validate Active Date, Max Usage, Min Cart Spend        │
│ 3. Check Product / Category Restrictions                  │
└─────────────────────────┬─────────────────────────────────┘
                          │ Validated OK
                          ▼
┌───────────────────────────────────────────────────────────┐
│ 4. Calculate Discount Value (e.g. 20% of $299.99 = $59.99)│
│    Apply Max Discount Cap if specified ($50.00 max)       │
└─────────────────────────┬─────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│ 5. Insert Reward Line into sale.order.line                │
│    - product_id: 88 (Discount Service)                    │
│    - price_unit: -50.00                                   │
│    - is_reward_line: true                                 │
│    - reward_id: 12, coupon_id: 501 (if unique card)       │
└───────────────────────────────────────────────────────────┘
```

#### 6.1 Code Verification & Rule Validation

Query active promotion rules matching the customer's entered string:

```http
POST /json/2/loyalty.rule/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["code", "=", "SUMMER20"],
    ["program_id.active", "=", true]
  ],
  "fields": [
    "id",
    "program_id",
    "minimum_amount",
    "minimum_amount_tax_mode",
    "minimum_qty",
    "product_ids",
    "product_category_id"
  ],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 18,
    "program_id": [12, "Summer 2026 Promo Code"],
    "minimum_amount": 100.0,
    "minimum_amount_tax_mode": "incl",
    "minimum_qty": 0,
    "product_ids": [],
    "product_category_id": false
  }
]
```

#### 6.2 Query Program Reward Specifications
Fetch the exact discount parameters configured for program `12`:

```http
POST /json/2/loyalty.reward/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["program_id", "=", 12]
  ],
  "fields": [
    "id",
    "reward_type",
    "discount",
    "discount_mode",
    "discount_applicability",
    "discount_max_amount",
    "reward_product_id"
  ],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 22,
    "reward_type": "discount",
    "discount": 20.0,
    "discount_mode": "percent",
    "discount_applicability": "order",
    "discount_max_amount": 50.0,
    "reward_product_id": [88, "Discount"]
  }
]
```

#### 6.3 Injecting Reward Discount Line into `sale.order.line`
Calculate the discount ($299.99 $\times$ 20% = $60.00, capped at $50.00) and append the reward line:

```http
POST /json/2/sale.order.line/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "order_id": 782,
      "product_id": 88,
      "name": "20% discount on order [SUMMER20]",
      "product_uom_qty": 1.0,
      "price_unit": -50.00,
      "is_reward_line": true,
      "reward_id": 22
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[1890]
```

#### 6.4 Applying Direct Item-Level Percentage Discounts
If your storefront offers a per-item promotional discount (e.g. 15% off a specific product line), you can directly set the `discount` field on the item's `sale.order.line`:

```http
POST /json/2/sale.order.line/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [1889],
  "vals": {
    "discount": 15.0
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

### Step 7: Customer Loyalty Points Accrual & Reward Redemption

#### 7.1 Check Customer Loyalty Points Balance
Find the customer's loyalty card for program `10` (e.g. Points Program):

```http
POST /json/2/loyalty.card/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["partner_id", "=", 104],
    ["program_id.program_type", "=", "loyalty"]
  ],
  "fields": ["id", "code", "points", "expiration_date", "program_id"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 480,
    "code": "LOYALTY-104-USER",
    "points": 350.0,
    "expiration_date": false,
    "program_id": [10, "Customer Loyalty Program"]
  }
]
```

#### 7.2 Redeem 100 Points for $10 Discount
Deduct 100 points from the card and insert the reward line on the sales order:

```http
POST /json/2/loyalty.card/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [480],
  "vals": {
    "points": 250.0
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

### Step 8: Order Confirmation, Coupon Consumption & Cancellation/Refunds

#### 8.1 Increment Coupon Usage Count Upon Order Confirmation
When the customer successfully completes payment and the order is confirmed:

```http
POST /json/2/loyalty.card/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501],
  "vals": {
    "points": 0.0,
    "use_count": 1
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

#### 8.2 Restoring Coupon / Points on Order Cancellation
If the customer cancels the order or requests a full refund before fulfillment:

```http
POST /json/2/loyalty.card/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [501],
  "vals": {
    "points": 1.0,
    "use_count": 0
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

## 5. Dynamic Pricelist Integration (B2B Tiers, Wholesale, Volume Discounts)

Pricelists allow automatic computation of tiered pricing based on customer group, quantity ordered, and date periods without customer coupon entry.

### Step 1: Querying Customer-Specific Pricelists

```http
POST /json/2/product.pricelist/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["active", "=", true]
  ],
  "fields": ["id", "name", "currency_id", "discount_policy"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "name": "Public Pricelist (USD)",
    "currency_id": [2, "USD"],
    "discount_policy": "with_discount"
  },
  {
    "id": 4,
    "name": "B2B Gold Wholesale (15% Off)",
    "currency_id": [2, "USD"],
    "discount_policy": "without_discount"
  }
]
```

---

### Step 2: Setting up Volume Tier Discounts (`product.pricelist.item`)

Add a rule giving 25% discount when buying 10 or more units of a specific product template (`product_tmpl_id = 620`):

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
      "min_quantity": 10.0,
      "compute_price": "percentage",
      "percent_price": 25.0
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[89]
```

---

### Step 3: Applying Pricelists to Sales Orders

Assign the wholesale pricelist directly to the quotation `sale.order`:

```http
POST /json/2/sale.order/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [782],
  "vals": {
    "pricelist_id": 4
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

## 6. Complete HTTP Request & Response Reference

### End-to-End Promo Validation & Application Example

#### 1. Validate Promo Code & Fetch Discount Rules
```http
POST /json/2/loyalty.rule/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["code", "=", "FLASH15"],
    ["program_id.active", "=", true]
  ],
  "fields": [
    "id",
    "program_id",
    "minimum_amount",
    "product_ids",
    "product_category_id"
  ]
}
```

#### 2. Fetch Associated Reward Details
```http
POST /json/2/loyalty.reward/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["program_id", "=", 18]
  ],
  "fields": [
    "id",
    "reward_type",
    "discount",
    "discount_mode",
    "discount_max_amount",
    "reward_product_id"
  ]
}
```

#### 3. Append Discount Line to Active Order
```http
POST /json/2/sale.order.line/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "order_id": 810,
      "product_id": 88,
      "name": "Flash Sale 15% Discount [FLASH15]",
      "product_uom_qty": 1.0,
      "price_unit": -22.50,
      "is_reward_line": true,
      "reward_id": 31
    }
  ]
}
```

---

## 7. Error Handling & Edge Case Failure Responses

All error responses adhere to standard HTTP status codes with structured JSON diagnostics.

### 7.1 Invalid Promo Code (`404 Not Found` / Custom Validation)
When a customer inputs a non-existent or mis-typed code:

#### Application Query Response
```json
[]
```
*(Empty array returned by `search_read` on `loyalty.rule` or `loyalty.card`)*

#### E-Commerce Storefront Error Payload
```json
{
  "error": "INVALID_PROMO_CODE",
  "message": "The promotion code 'SUMMER50' is invalid or does not exist."
}
```

---

### 7.2 Expired Promo Code or Coupon (`400 Bad Request`)
Occurs when `date_to` or `expiration_date` has passed.

```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "This promotional program expired on 2026-08-01.",
  "arguments": ["This promotional program expired on 2026-08-01."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  ...\nodoo.exceptions.ValidationError: This promotional program expired on 2026-08-01.\n"
}
```

---

### 7.3 Coupon Usage Limit Exceeded (`400 Bad Request`)
Occurs when a single-use coupon has already been redeemed (`use_count >= 1` or `max_usage` reached).

```json
{
  "name": "odoo.exceptions.UserError",
  "message": "Coupon 'VIP-2026-X99A1' has already been used and cannot be redeemed again.",
  "arguments": ["Coupon 'VIP-2026-X99A1' has already been used."],
  "context": {}
}
```

---

### 7.4 Minimum Order Spend Not Met (`400 Bad Request`)
Occurs when the cart subtotal is below `minimum_amount`.

```json
{
  "name": "odoo.exceptions.UserError",
  "message": "Promo code 'SUMMER20' requires a minimum order subtotal of $100.00. Current qualifying cart subtotal is $79.50.",
  "arguments": ["Minimum order subtotal not met."],
  "context": {}
}
```

---

### 7.5 Incompatible / Excluded Products in Cart (`400 Bad Request`)
Occurs when the coupon applies only to specific categories or product IDs, but none are in the cart.

```json
{
  "name": "odoo.exceptions.UserError",
  "message": "This coupon is only valid for items in the 'Gaming Accessories' category.",
  "arguments": ["No qualifying products in cart."],
  "context": {}
}
```

---

### 7.6 Authentication & Permission Failures (`401 Unauthorized` / `403 Forbidden`)

#### `401 Unauthorized` (Invalid or Expired API Key)
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401]
}
```

#### `403 Forbidden` (Insufficient Permissions on Loyalty Models)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to modify 'Loyalty Cards' (loyalty.card) records. Contact your system administrator."
}
```

---

## 8. Production Best Practices for E-Commerce Developers

### 1. Prevent Coupon Stacking & Abuse
- **One Global Promo per Cart:** In your e-commerce checkout logic, decide whether to allow multiple coupon codes simultaneously. Standard practice is to allow **1 Promotional Code + 1 Digital Gift Card** per transaction.
- **Clear Previous Reward Lines:** When a user applies a new coupon code or removes an existing one, delete previous reward lines (`sale.order.line` where `is_reward_line = true`) before computing the new discount.

### 2. Tax-Inclusive vs Tax-Exclusive Discount Calculation
- Ensure consistency between how your storefront displays discounts and how Odoo computes taxes:
  - If selling in B2C markets (e.g. EU/UK VAT included), set `minimum_amount_tax_mode: 'incl'` and ensure the reward service product has the corresponding sales taxes applied.
  - If selling in B2B markets (US Sales Tax added at checkout), set `minimum_amount_tax_mode: 'excl'`.

### 3. Concurrency & Idempotency during Checkout
- **Locking Single-Use Coupons:** When a customer enters the payment gateway (e.g. Stripe checkout redirect), place a temporary lock or reservation timestamp on the coupon card in your backend Redis / DB.
- If the payment succeeds, finalize `use_count: 1` and `points: 0` in Odoo.
- If the cart is abandoned or payment fails, release the reservation to allow future redemption.

### 4. Reverting Discounts on Order Edits / Item Removals
- If a customer modifies their cart (e.g., deletes an item lowering cart total below the $100 minimum threshold):
  - Recalculate cart eligibility.
  - Automatically remove or recalculate the `sale.order.line` discount before payment submission.

### 5. Caching Strategy for Active Promotions
- Cache the list of active public promo codes and date ranges (`date_from`, `date_to`, `active`) in memory or Redis for up to 5 minutes to avoid hitting Odoo for every keystroke or cart view.
- Perform the authoritative validation and discount calculation against Odoo's JSON-2 API during checkout initiation and final order placement.
