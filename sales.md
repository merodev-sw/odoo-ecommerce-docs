# Odoo JSON-2 API: Sales, Invoicing & Delivery Integration Guide

This guide provides an exhaustive technical reference for backend developers integrating an **e-commerce platform** with Odoo's **Sales (`sale.order`)**, **Invoicing (`account.move`)**, **Payments (`account.payment`)**, and **Delivery / Inventory (`stock.picking`)** modules using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Order-to-Cash (O2C) Architecture Overview](#1-order-to-cash-o2c-architecture-overview)
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
3. [Model & Field References](#3-model--field-references)
   - [3.1 Sales Order (`sale.order`)](#31-sales-order-saleorder)
   - [3.2 Sales Order Line (`sale.order.line`)](#32-sales-order-line-saleorderline)
   - [3.3 Customer Invoice (`account.move`)](#33-customer-invoice-accountmove)
   - [3.4 Invoice Line (`account.move.line`)](#34-invoice-line-accountmoveline)
   - [3.5 Customer Payment (`account.payment` & Payment Register)](#35-customer-payment-accountpayment--payment-register)
   - [3.6 Delivery Order / Stock Picking (`stock.picking`)](#36-delivery-order--stock-picking-stockpicking)
4. [Step-by-Step E-Commerce Integration Flow](#4-step-by-step-e-commerce-integration-flow)
   - [Step 1: Product, Pricelist & Tax Lookups](#step-1-product-pricelist--tax-lookups)
   - [Step 2: Creating a Sales Order (Quotation)](#step-2-creating-a-sales-order-quotation)
   - [Step 3: Confirming the Sales Order (`action_confirm`)](#step-3-confirming-the-sales-order-action_confirm)
   - [Step 4: Creating & Posting Customer Invoices](#step-4-creating--posting-customer-invoices)
   - [Step 5: Registering Online Customer Payments](#step-5-registering-online-customer-payments)
   - [Step 6: Tracking & Validating Warehouse Deliveries](#step-6-tracking--validating-warehouse-deliveries)
   - [Step 7: Order Modifications, Cancellations & Refunds](#step-7-order-modifications-cancellations--refunds)
5. [Complete HTTP Request & Response Reference](#5-complete-http-request--response-reference)
6. [Error Handling & Failure Responses](#6-error-handling--failure-responses)
7. [Production Best Practices for E-Commerce Developers](#7-production-best-practices-for-e-commerce-developers)

---

## 1. Order-to-Cash (O2C) Architecture Overview

In Odoo, the complete e-commerce lifecycle spans four primary models interacting sequentially:

```
[E-Commerce Store]
       │
       ▼ (1. Checkout Completed)
┌───────────────────────────────────────────────────────────┐
│                    sale.order                             │
│  State: draft (Quotation) ───► action_confirm() ──► sale  │
└─────────────────────────────┬─────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            ▼ (Triggers Delivery)               ▼ (Triggers Invoicing)
┌───────────────────────────────────────┐ ┌───────────────────────────────────────┐
│             stock.picking             │ │             account.move              │
│  State: assigned ──► button_validate()│ │  Type: out_invoice                    │
│         ──► done (Shipped)            │ │  State: draft ──► action_post() ──► posted
│  (Set Tracking Number & Carrier)      │ └───────────────────┬───────────────────┘
└───────────────────────────────────────┘                     │
                                                              ▼ (Triggers Payment)
                                          ┌───────────────────────────────────────┐
                                          │            account.payment            │
                                          │  State: posted (Reconciled with Inv)  │
                                          │  Payment State: in_payment / paid     │
                                          └───────────────────────────────────────┘
```

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

### Required Headers
```http
Host: mycompany.odoo.com
Authorization: bearer <YOUR_160_BIT_API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-Ecommerce-Backend/1.0
```

### Request Structure
- **Model Methods (e.g. `create`, `search_read`):** Arguments passed as named JSON properties (e.g. `{"vals_list": [...]}`, `{"domain": [...]}`).
- **Recordset Methods (e.g. `write`, `read`, `action_confirm`, `action_post`, `button_validate`):** Include `"ids": [id1, id2]` alongside named arguments or context.

---

## 3. Model & Field References

### 3.1 Sales Order (`sale.order`)

The `sale.order` model represents customer quotations and confirmed sales orders.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `partner_id` | `Many2one` (`res.partner`) | **REQUIRED** | — | Customer ID (Account Holder / Main Contact). |
| `partner_invoice_id` | `Many2one` (`res.partner`) | Optional | `partner_id` | Billing Address ID. Defaults to `partner_id` if omitted. |
| `partner_shipping_id` | `Many2one` (`res.partner`) | Optional | `partner_id` | Delivery / Shipping Address ID. Defaults to `partner_id` if omitted. |
| `order_line` | `One2many` (`sale.order.line`) | Optional | `[]` | Array of order items, shipping charges, discounts, and section lines. |
| `client_order_ref` | `Char` | Optional | `null` | E-commerce Order Number / Reference (e.g., `ORD-2026-98124`). Crucial for idempotency. |
| `date_order` | `Datetime` | Optional | Current UTC | Order placement timestamp (format: `"YYYY-MM-DD HH:MM:SS"` in UTC). |
| `pricelist_id` | `Many2one` (`product.pricelist`) | Optional | Partner/Company default | Pricelist ID for currency and tiered catalog pricing. |
| `currency_id` | `Many2one` (`res.currency`) | Optional | Pricelist currency | Currency ID (e.g., USD, EUR, GBP). |
| `payment_term_id` | `Many2one` (`account.payment.term`) | Optional | Partner default | Payment terms ID (e.g., Immediate Payment). |
| `origin` | `Char` | Optional | `null` | Channel source label (e.g., `"Web Store"`, `"Mobile App iOS"`). |
| `note` | `Html` / `Text` | Optional | `null` | Customer notes, gift messages, or terms displayed on order confirmation PDF. |
| `state` | `Selection` | *System-managed* | `'draft'` | Status: `'draft'` (Quotation), `'sent'` (Quotation Sent), `'sale'` (Confirmed Sales Order), `'cancel'` (Cancelled). |
| `invoice_status` | `Selection` | *Computed* | `'no'` | Status: `'upselling'`, `'invoiced'`, `'to invoice'`, `'no'`. |
| `amount_untaxed` | `Monetary` | *Computed* | `0.0` | Total subtotal before taxes. |
| `amount_tax` | `Monetary` | *Computed* | `0.0` | Total tax amount. |
| `amount_total` | `Monetary` | *Computed* | `0.0` | Grand total (Untaxed + Taxes). |
| `picking_ids` | `One2many` (`stock.picking`) | *Computed* | `[]` | Outgoing shipments generated upon order confirmation. |
| `invoice_ids` | `Many2many` (`account.move`) | *Computed* | `[]` | Invoices generated from this sales order. |

---

### 3.2 Sales Order Line (`sale.order.line`)

Individual items, fees, discounts, and shipping lines inside `sale.order`.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** (for items) | — | Specific product variant ID. |
| `name` | `Text` | **REQUIRED** | Product name | Item description shown on invoice and packing slip. |
| `product_uom_qty` | `Float` | **REQUIRED** | `1.0` | Quantity ordered. |
| `price_unit` | `Float` | Optional | Product pricelist price | Unit selling price. If omitted, Odoo computes it from the active pricelist. |
| `tax_id` | `Many2many` (`account.tax`) | Optional | Product default taxes | Tax IDs applied to this line (e.g., `[[6, 0, [tax_id]]]`). Set to `[[6, 0, []]]` for tax-exempt. |
| `discount` | `Float` | Optional | `0.0` | Percentage discount on the line (e.g., `15.0` for 15% off). |
| `product_uom` | `Many2one` (`uom.uom`) | Optional | Product UoM | Unit of Measure ID (e.g., Units, kg). |
| `sequence` | `Integer` | Optional | `10` | Display ordering sequence on the order. |
| `display_type` | `Selection` | Optional | `false` | Set to `'line_section'` or `'line_note'` for decorative headers/notes. |
| `qty_delivered` | `Float` | *Computed/Managed* | `0.0` | Quantity fulfilled/delivered by the warehouse. |
| `qty_invoiced` | `Float` | *Computed* | `0.0` | Quantity invoiced to the customer. |
| `price_subtotal` | `Monetary` | *Computed* | `0.0` | Untaxed total for this line. |
| `price_total` | `Monetary` | *Computed* | `0.0` | Total including tax for this line. |

---

### 3.3 Customer Invoice (`account.move`)

Represents accounting invoices, credit notes, and refunds.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `move_type` | `Selection` | **REQUIRED** | `'out_invoice'` | Type of move: `'out_invoice'` (Customer Invoice), `'out_refund'` (Customer Credit Note/Refund), `'in_invoice'` (Vendor Bill), `'entry'` (Journal Entry). |
| `partner_id` | `Many2one` (`res.partner`) | **REQUIRED** | — | Customer / Debtor ID. |
| `partner_shipping_id` | `Many2one` (`res.partner`) | Optional | `partner_id` | Delivery address ID for tax calculation / reporting. |
| `invoice_date` | `Date` | Optional | Current date | Accounting issue date (format: `"YYYY-MM-DD"`). |
| `invoice_date_due` | `Date` | Optional | Computed | Invoice due date (format: `"YYYY-MM-DD"`). |
| `invoice_payment_term_id`| `Many2one` (`account.payment.term`) | Optional | Partner default | Payment terms ID. |
| `ref` | `Char` | Optional | `null` | External reference (e.g. Sales order reference / e-commerce invoice number). |
| `payment_reference` | `Char` | Optional | Auto-generated | Reference string for payment matching. |
| `currency_id` | `Many2one` (`res.currency`) | Optional | Company currency | Invoice currency. |
| `journal_id` | `Many2one` (`account.journal`) | Optional | Default Customer Invoices journal | Sales journal ID. |
| `invoice_line_ids` | `One2many` (`account.move.line`) | Optional | `[]` | Detailed invoice lines. |
| `state` | `Selection` | *System-managed* | `'draft'` | Status: `'draft'` (Draft), `'posted'` (Official / Validated), `'cancel'` (Cancelled). |
| `payment_state` | `Selection` | *Computed* | `'not_paid'` | Status: `'not_paid'`, `'in_payment'`, `'paid'`, `'partial'`, `'reversed'`. |
| `amount_untaxed` | `Monetary` | *Computed* | `0.0` | Untaxed amount. |
| `amount_tax` | `Monetary` | *Computed* | `0.0` | Tax amount. |
| `amount_total` | `Monetary` | *Computed* | `0.0` | Total invoice amount. |
| `amount_residual` | `Monetary` | *Computed* | `0.0` | Remaining unpaid balance. |

---

### 3.4 Invoice Line (`account.move.line`)

Lines belonging to `invoice_line_ids` within `account.move`.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `product_id` | `Many2one` (`product.product`) | Optional | `null` | Product ID. |
| `name` | `Char` | **REQUIRED** | Product name | Line item description shown on the accounting invoice. |
| `quantity` | `Float` | Optional | `1.0` | Quantity invoiced. |
| `price_unit` | `Float` | **REQUIRED** | `0.0` | Unit price before tax. |
| `discount` | `Float` | Optional | `0.0` | Percentage discount (e.g., `10.0`). |
| `tax_ids` | `Many2many` (`account.tax`) | Optional | Default taxes | Tax IDs applied (`[[6, 0, [tax_ids]]]`). |
| `account_id` | `Many2one` (`account.account`) | Optional | Product/Category income account | General ledger income account. Auto-resolved by Odoo if omitted. |
| `product_uom_id` | `Many2one` (`uom.uom`) | Optional | Product UoM | Unit of Measure ID. |

---

### 3.5 Customer Payment (`account.payment` & Payment Register)

Represents incoming customer payments processed through online gateways (Stripe, PayPal, Adyen, Bank Transfer).

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `partner_id` | `Many2one` (`res.partner`) | **REQUIRED** | — | Customer ID who paid. |
| `amount` | `Monetary` | **REQUIRED** | — | Paid amount (e.g., `149.99`). |
| `payment_type` | `Selection` | **REQUIRED** | `'inbound'` | `'inbound'` (Customer receipt) or `'outbound'` (Customer refund). |
| `partner_type` | `Selection` | **REQUIRED** | `'customer'` | `'customer'` or `'supplier'`. |
| `journal_id` | `Many2one` (`account.journal`) | **REQUIRED** | — | Payment Journal ID (e.g., Stripe Journal, Bank Journal, PayPal Journal). |
| `date` | `Date` | Optional | Current date | Payment receipt date (format: `"YYYY-MM-DD"`). |
| `memo` / `ref` | `Char` | Optional | `null` | Gateway transaction ID (e.g., Stripe Charge `ch_3M49...` or PayPal capture ID). |
| `currency_id` | `Many2one` (`res.currency`) | Optional | Journal currency | Payment currency ID. |
| `state` | `Selection` | *System-managed* | `'draft'` | Status: `'draft'`, `'posted'`, `'cancel'`. |

---

### 3.6 Delivery Order / Stock Picking (`stock.picking`)

Represents warehouse shipments to the customer created automatically when a sales order is confirmed.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | *System-managed* | — | Warehouse transfer reference (e.g., `WH/OUT/00142`). |
| `partner_id` | `Many2one` (`res.partner`) | **REQUIRED** | — | Delivery / Shipping destination contact. |
| `origin` | `Char` | Optional | — | Source sales order name (e.g., `S00045`). |
| `picking_type_id` | `Many2one` (`stock.picking.type`) | **REQUIRED** | Default Outgoing | Operation type ID (e.g., Delivery Orders). |
| `location_id` | `Many2one` (`stock.location`) | **REQUIRED** | WH/Stock | Source warehouse location. |
| `location_dest_id` | `Many2one` (`stock.location`) | **REQUIRED** | Customer Location | Destination location. |
| `carrier_tracking_ref` | `Char` | Optional | `null` | Carrier tracking number / waybill (e.g. `1Z9999999999999999`, `DHL-8821904`). |
| `carrier_id` | `Many2one` (`delivery.carrier`) | Optional | `null` | Shipping method / courier ID (e.g. FedEx, UPS, DHL). |
| `state` | `Selection` | *System-managed* | `'draft'` | Status: `'draft'`, `'waiting'`, `'confirmed'`, `'assigned'` (Ready to Ship), `'done'` (Shipped), `'cancel'`. |
| `date_done` | `Datetime` | *System-managed* | `null` | Shipment completion timestamp. |
| `move_ids` | `One2many` (`stock.move`) | *Computed* | `[]` | Inventory movement lines. |

---

## 4. Step-by-Step E-Commerce Integration Flow

---

### Step 1: Product, Pricelist & Tax Lookups

Map your e-commerce SKUs to Odoo `product.product` IDs and determine active taxes.

#### 1.1 Find Product ID by SKU / Default Code
```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["default_code", "=", "SKU-HEADPHONES-PRO"]
  ],
  "fields": ["id", "name", "default_code", "list_price", "taxes_id", "uom_id"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1042,
    "name": "Wireless Noise Cancelling Headphones",
    "default_code": "SKU-HEADPHONES-PRO",
    "list_price": 299.99,
    "taxes_id": [1],
    "uom_id": [1, "Units"]
  }
]
```

#### 1.2 Find Sales Tax ID (e.g., 10% Sales Tax)
```http
POST /json/2/account.tax/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["type_tax_use", "=", "sale"],
    ["amount", "=", 10.0]
  ],
  "fields": ["id", "name", "amount", "price_include"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "name": "10% Standard Sales Tax",
    "amount": 10.0,
    "price_include": false
  }
]
```

---

### Step 2: Creating a Sales Order (Quotation)

Create the sales order with items, shipping fees, and discounts in a single atomic request using `order_line` with `(0, 0, values)` command tuples.

```http
POST /json/2/sale.order/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "partner_id": 540,
      "partner_invoice_id": 543,
      "partner_shipping_id": 542,
      "client_order_ref": "ECOMM-ORD-2026-8812",
      "origin": "Web Store Checkout",
      "date_order": "2026-08-16 12:30:00",
      "order_line": [
        [
          0,
          0,
          {
            "product_id": 1042,
            "name": "Wireless Noise Cancelling Headphones (Black)",
            "product_uom_qty": 2.0,
            "price_unit": 299.99,
            "discount": 10.0,
            "tax_id": [[6, 0, [1]]]
          }
        ],
        [
          0,
          0,
          {
            "product_id": 890,
            "name": "Express Shipping (2-Day Courier)",
            "product_uom_qty": 1.0,
            "price_unit": 15.00,
            "tax_id": [[6, 0, []]]
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[782]
```
*(Returns the newly created Sales Order ID `782`)*

---

### Step 3: Confirming the Sales Order (`action_confirm`)

Confirming the order updates its status from `draft` to `sale`. This automatically creates the linked warehouse delivery picking (`stock.picking`) and marks order lines as invoiceable.

```http
POST /json/2/sale.order/action_confirm HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [782],
  "context": {
    "lang": "en_US"
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

#### Inspect the Confirmed Order to Retrieve Generated Delivery & State
```http
POST /json/2/sale.order/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [782],
  "fields": ["name", "state", "amount_untaxed", "amount_tax", "amount_total", "picking_ids", "invoice_status"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 782,
    "name": "S000782",
    "state": "sale",
    "amount_untaxed": 554.98,
    "amount_tax": 53.99,
    "amount_total": 608.97,
    "picking_ids": [341],
    "invoice_status": "to invoice"
  }
]
```

---

### Step 4: Creating & Posting Customer Invoices

You can generate customer invoices either automatically from the sales order or directly via the `account.move` model.

#### Method A: Automatic Invoice Generation from Sales Order (`_create_invoices`)
```http
POST /json/2/sale.order/_create_invoices HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [782],
  "final": true
}
```

**Success Response (`200 OK`):**
```json
[1204]
```
*(Returns an array of generated draft invoice IDs `[1204]`)*

---

#### Method B: Direct Invoice Creation (`account.move/create`)
For custom billing workflows where you need full control over line items and accounting journals:

```http
POST /json/2/account.move/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "move_type": "out_invoice",
      "partner_id": 540,
      "partner_shipping_id": 542,
      "invoice_date": "2026-08-16",
      "ref": "ECOMM-ORD-2026-8812",
      "invoice_line_ids": [
        [
          0,
          0,
          {
            "product_id": 1042,
            "name": "Wireless Noise Cancelling Headphones",
            "quantity": 2.0,
            "price_unit": 299.99,
            "discount": 10.0,
            "tax_ids": [[6, 0, [1]]]
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[1205]
```

---

#### Post / Validate the Invoice (`action_post`)
Posting moves the invoice from `draft` to `posted`, making it official in the ledger and assigning a permanent invoice sequence number (e.g. `INV/2026/00492`).

```http
POST /json/2/account.move/action_post HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [1204]
}
```

**Success Response (`200 OK`):**
```json
true
```

---

### Step 5: Registering Online Customer Payments

When a customer pays online during checkout, record the payment and reconcile it with the invoice.

#### 5.1 Find Payment Journal (e.g., Stripe / Bank Journal)
```http
POST /json/2/account.journal/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["type", "in", ["bank", "cash"]]
  ],
  "fields": ["id", "name", "type", "code"],
  "limit": 5
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 7,
    "name": "Stripe Payments",
    "type": "bank",
    "code": "STRIP"
  }
]
```

#### 5.2 Register Payment via `account.payment.register` Wizard
This is the standard ORM workflow to post payment and automatically reconcile the invoice to `paid` or `in_payment`.

##### Part A: Create Payment Register Wizard Record
```http
POST /json/2/account.payment.register/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "context": {
    "active_model": "account.move",
    "active_ids": [1204]
  },
  "vals_list": [
    {
      "journal_id": 7,
      "amount": 608.97,
      "payment_date": "2026-08-16",
      "communication": "Stripe Charge ch_3M92810x992"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[89]
```

##### Part B: Execute Payment Creation
```http
POST /json/2/account.payment.register/action_create_payments HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [89]
}
```

**Success Response (`200 OK`):**
```json
{
  "type": "ir.actions.act_window_close"
}
```

#### Verify Invoice Payment Status
```http
POST /json/2/account.move/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [1204],
  "fields": ["name", "state", "payment_state", "amount_total", "amount_residual"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1204,
    "name": "INV/2026/00492",
    "state": "posted",
    "payment_state": "paid",
    "amount_total": 608.97,
    "amount_residual": 0.0
  }
]
```

---

### Step 6: Tracking & Validating Warehouse Deliveries

When an order is confirmed, Odoo creates a delivery transfer in `stock.picking`.

#### 6.1 Find Delivery Picking for the Sales Order
```http
POST /json/2/stock.picking/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["origin", "=", "S000782"]
  ],
  "fields": [
    "id",
    "name",
    "state",
    "partner_id",
    "carrier_tracking_ref",
    "move_ids"
  ]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 341,
    "name": "WH/OUT/00341",
    "state": "assigned",
    "partner_id": [542, "Alex Morgan (Warehouse Depot)"],
    "carrier_tracking_ref": false,
    "move_ids": [912]
  }
]
```

#### 6.2 Attach Tracking Number and Carrier
When your warehouse or 3PL generates a shipping label:

```http
POST /json/2/stock.picking/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [341],
  "vals": {
    "carrier_tracking_ref": "1Z9999999999999999"
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

#### 6.3 Set Done Quantities and Validate Delivery (`button_validate`)
To mark items as shipped and deduct stock from warehouse inventory:

##### Part A: Set Quantity Done on Stock Moves
```http
POST /json/2/stock.move/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [912],
  "vals": {
    "quantity": 2.0
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

##### Part B: Validate the Delivery
```http
POST /json/2/stock.picking/button_validate HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [341]
}
```

**Success Response (`200 OK`):**
```json
true
```
*(The picking state is now `'done'` and `date_done` is recorded)*

---

### Step 7: Order Modifications, Cancellations & Refunds

#### 7.1 Cancel an Order (`action_cancel`)
If the customer cancels before shipment:

```http
POST /json/2/sale.order/action_cancel HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [782]
}
```

**Success Response (`200 OK`):**
```json
true
```

#### 7.2 Issue a Credit Note / Refund (`account.move/create` with `move_type: "out_refund"`)
When processing a partial or full refund:

```http
POST /json/2/account.move/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "move_type": "out_refund",
      "partner_id": 540,
      "ref": "Refund for Return #RET-9918",
      "invoice_date": "2026-08-16",
      "invoice_line_ids": [
        [
          0,
          0,
          {
            "product_id": 1042,
            "name": "Refund: Wireless Noise Cancelling Headphones",
            "quantity": 1.0,
            "price_unit": 269.99,
            "tax_ids": [[6, 0, [1]]]
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[1208]
```

##### Post the Credit Note:
```http
POST /json/2/account.move/action_post HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [1208]
}
```

**Success Response (`200 OK`):**
```json
true
```

---

## 5. Complete HTTP Request & Response Reference

### Search Open Orders by Customer
```http
POST /json/2/sale.order/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["partner_id", "=", 540],
    ["state", "in", ["sale", "done"]]
  ],
  "fields": ["name", "client_order_ref", "date_order", "amount_total", "invoice_status"],
  "order": "date_order desc",
  "limit": 10
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 782,
    "name": "S000782",
    "client_order_ref": "ECOMM-ORD-2026-8812",
    "date_order": "2026-08-16 12:30:00",
    "amount_total": 608.97,
    "invoice_status": "invoiced"
  }
]
```

---

## 6. Error Handling & Failure Responses

All failures return a standard HTTP error code (`4xx` or `5xx`) and a JSON error payload containing `name`, `message`, `arguments`, and `debug`.

### 6.1 Missing Required Field on Line (`400 Bad Request`)
Occurs if an order line is created without a `product_id` or `name`.

#### Request
```http
POST /json/2/sale.order/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "partner_id": 540,
      "order_line": [
        [
          0,
          0,
          {
            "product_uom_qty": 2.0
          }
        ]
      ]
    }
  ]
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "The field 'name' is required on 'sale.order.line'.",
  "arguments": ["The field 'name' is required on 'sale.order.line'."],
  "context": {
    "lang": "en_US"
  },
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/models.py\", line 4501, in _check_field_value\n    raise ValidationError(f\"The field '{name}' is required on '{self._name}'.\")\nodoo.exceptions.ValidationError: The field 'name' is required on 'sale.order.line'.\n"
}
```

---

### 6.2 Invalid Status Transition / Cannot Confirm Cancelled Order (`400 Bad Request`)
Occurs when invoking `action_confirm` on an order in `'cancel'` state.

#### Request
```http
POST /json/2/sale.order/action_confirm HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [780]
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.UserError",
  "message": "You cannot confirm a cancelled quotation or order.",
  "arguments": ["You cannot confirm a cancelled quotation or order."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/sale/models/sale_order.py\", line 780, in action_confirm\n    raise UserError(_('You cannot confirm a cancelled quotation or order.'))\nodoo.exceptions.UserError: You cannot confirm a cancelled quotation or order.\n"
}
```

---

### 6.3 Authentication Expired or Invalid API Key (`401 Unauthorized`)

#### Error Response (`401 Unauthorized`)
```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/base/models/ir_http.py\", line 240, in _auth_method_bearer\n    raise werkzeug.exceptions.Unauthorized('Invalid apikey')\nwerkzeug.exceptions.Unauthorized: 401 Unauthorized: Invalid apikey\n"
}
```

---

### 6.4 Insufficient Permissions (`403 Forbidden`)

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You do not have access rights to validate Delivery Orders (stock.picking).",
  "arguments": ["You do not have access rights to validate Delivery Orders (stock.picking)."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/addons/base/models/ir_rule.py\", line 310, in check\n    raise AccessError(...)\nodoo.exceptions.AccessError: You do not have access rights to validate Delivery Orders.\n"
}
```

---

## 7. Production Best Practices for E-Commerce Developers

### 1. Store Idempotency with `client_order_ref`
Always populate `client_order_ref` on `sale.order` with your e-commerce platform's unique order ID (e.g. `SHOPIFY-ORD-9021`). Before creating an order, query `sale.order` with `domain: [["client_order_ref", "=", "SHOPIFY-ORD-9021"]]` to prevent accidental duplicate orders on webhook retries.

### 2. Atomic Order Creation
Pass all order lines (products, discounts, shipping fees) inside the `order_line` property during the initial `sale.order/create` call. This avoids multiple HTTP roundtrips and guarantees that if any line fails validation, the entire order rolls back automatically.

### 3. Handle Timezones Explicitly in UTC
Always format `date_order` in UTC string format (`"YYYY-MM-DD HH:MM:SS"`). Odoo stores all timestamps internally in UTC and translates them to local timezones based on user preferences.

### 4. Cache Catalog SKUs, Taxes and Shipping Products
Do not look up product variants, taxes, and shipping carrier products during live customer checkout. Cache SKU-to-`product_id` and Tax-to-`tax_id` mappings in Redis or application memory to ensure sub-millisecond checkout response times.

### 5. Automated Stock Validation Safeguards
When calling `stock.picking/button_validate`, ensure your inventory system or 3PL has already confirmed physical availability. If stock is negative or unreserved, Odoo may generate backorders or raise an inventory shortage warning.
