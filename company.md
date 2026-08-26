# Odoo JSON-2 API: Multi-Company & Company Management Guide

This guide provides an exhaustive technical reference for backend developers integrating an **e-commerce platform** or **multi-storefront system** with Odoo's **Multi-Company Architecture (`res.company`, `res.users`)** using the **External JSON-2 API**. It details how to retrieve company structures, isolate data per company, query shared global records across companies, and execute multi-branch operations securely.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's modern **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Multi-Company Architecture & Isolation Model](#1-multi-company-architecture--isolation-model)
   - Multi-Company vs. Single-Company Paradigms
   - How Odoo Isolates Data (`ir.rule` Engine)
   - Shared vs. Company-Specific Records
   - The Role of `allowed_company_ids` & `company_id` in Context
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
   - Required HTTP Headers
   - Multi-Company JSON-2 Payload Structure
   - Service Bot User Configuration Prerequisites
3. [Model & Field References](#3-model--field-references)
   - [3.1 Company Master Record (`res.company`)](#31-company-master-record-rescompany)
   - [3.2 User Company Permissions (`res.users`)](#32-user-company-permissions-resusers)
   - [3.3 Company Relationship Matrix across Odoo Models](#33-company-relationship-matrix-across-odoo-models)
4. [Fetching Companies & Their IDs (`res.company`)](#4-fetching-companies--their-ids-rescompany)
   - [Step 1: List All Active Companies & IDs](#step-1-list-all-active-companies--ids)
   - [Step 2: Fetch Single Company Metadata by ID](#step-2-fetch-single-company-metadata-by-id)
   - [Step 3: Discover Authenticated Bot User's Allowed Companies](#step-3-discover-authenticated-bot-users-allowed-companies)
   - [Step 4: Querying Parent-Subsidiary & Branch Hierarchies](#step-4-querying-parent-subsidiary--branch-hierarchies)
5. [Fetching Data from a Specific Company](#5-fetching-data-from-a-specific-company)
   - [Why Both Context and Domain Filters are Required](#why-both-context-and-domain-filters-are-required)
   - [5.1 Querying Sales Orders for a Specific Company (`sale.order`)](#51-querying-sales-orders-for-a-specific-company-saleorder)
   - [5.2 Querying Stock Levels & Warehouses for a Specific Company (`stock.quant` & `stock.warehouse`)](#52-querying-stock-levels--warehouses-for-a-specific-company-stockquant--stockwarehouse)
   - [5.3 Querying Invoices & Financial Records for a Specific Company (`account.move`)](#53-querying-invoices--financial-records-for-a-specific-company-accountmove)
   - [5.4 Querying Company-Exclusive Products (`product.template`)](#54-querying-company-exclusive-products-producttemplate)
   - [5.5 Creating Records Scoped to a Specific Company](#55-creating-records-scoped-to-a-specific-company)
6. [Fetching & Managing Shared Data Between Companies](#6-fetching--managing-shared-data-between-companies)
   - [6.1 Understanding Globally Shared Records (`company_id = False`)](#61-understanding-globally-shared-records-company_id--false)
   - [6.2 Querying Strictly Shared Records](#62-querying-strictly-shared-records)
   - [6.3 Querying Shared + Specific Company Records Combined](#63-querying-shared--specific-company-records-combined)
   - [6.4 Querying Data Across Multiple Authorized Companies in a Single Request](#64-querying-data-across-multiple-authorized-companies-in-a-single-request)
   - [6.5 Creating Globally Shared Records](#65-creating-globally-shared-records)
7. [Step-by-Step Multi-Storefront Integration Architecture](#7-step-by-step-multi-storefront-integration-architecture)
   - Workflow: Storefront Routing, Unified Catalog, Shared Customers & Scoped Orders
8. [Complete Code Reference (cURL, Node.js, Python)](#8-complete-code-reference-curl-nodejs-python)
   - [8.1 cURL Examples](#81-curl-examples)
   - [8.2 Node.js / TypeScript SDK Client](#82-nodejs--typescript-sdk-client)
   - [8.3 Python Integration Client](#83-python-integration-client)
9. [Error Handling & Multi-Company Edge Cases](#9-error-handling--multi-company-edge-cases)
   - [9.1 AccessError: Unauthorized Company Requested (`403 Forbidden`)](#91-accesserror-unauthorized-company-requested-403-forbidden)
   - [9.2 Cross-Company Relational Mismatches (`_check_company` Violation)](#92-cross-company-relational-mismatches-_check_company-violation)
   - [9.3 Missing `allowed_company_ids` Fallback Behavior](#93-missing-allowed_company_ids-fallback-behavior)
10. [Production Best Practices & Checklist](#10-production-best-practices--checklist)

---

## 1. Multi-Company Architecture & Isolation Model

In Odoo, a single database instance can host multiple independent legal entities, subsidiaries, brands, or regional storefronts. Multi-company mode introduces row-level security and context-dependent business logic across sales, accounting, inventory, and catalog management.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             Odoo Database                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────┐                     ┌─────────────────────┐   │
│   │   Company A (ID: 1)     │                     │  Company B (ID: 2)  │   │
│   │   - US Storefront       │                     │  - EU Storefront    │   │
│   │   - Currency: USD ($)   │                     │  - Currency: EUR (€)│   │
│   │   - US Warehouses       │                     │  - EU Warehouses    │   │
│   │   - Company A Orders    │                     │  - Company B Orders │   │
│   └────────────┬────────────┘                     └──────────┬──────────┘   │
│                │                                             │              │
│                └──────────────────────┬──────────────────────┘              │
│                                       │                                     │
│                                       ▼                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     Shared Global Data Layer                        │   │
│   │                      (company_id = False / null)                    │   │
│   │  - Master Product Catalog (product.template)                        │   │
│   │  - Global Customer Directory (res.partner)                          │   │
│   │  - Currencies & Exchange Rates (res.currency)                       │   │
│   │  - Units of Measure (uom.uom)                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### How Odoo Isolates Data (`ir.rule` Engine)

Odoo enforces multi-company data isolation using automated database **Record Rules (`ir.rule`)**. For almost every business model (e.g. `sale.order`, `account.move`, `stock.picking`, `product.template`), Odoo dynamically injects the following SQL domain check on every read, search, write, and delete operation:

```python
['|', ('company_id', '=', False), ('company_id', 'in', allowed_company_ids)]
```

### Shared vs. Company-Specific Records

| Record Classification | `company_id` Value | Visibility Rules | Examples |
| :--- | :--- | :--- | :--- |
| **Globally Shared** | `false` / `null` | Visible to **all** companies at all times. | Global customer contacts, master products, global categories, currencies, UoMs. |
| **Company-Specific** | Integer ID (e.g., `1`) | Visible **only** if the company ID is included in `allowed_company_ids` in the request context. | Sales orders, customer invoices, stock moves, bank accounts, warehouses, POS terminals. |
| **Multi-Company Enabled** | Array of IDs via `company_ids` | Visible if any of the record's companies overlap with `allowed_company_ids`. | Promotional programs (`loyalty.program`), pricelists (`product.pricelist`), sales teams (`crm.team`). |

### The Role of `allowed_company_ids` & `company_id` in Context

When executing any API call via the JSON-2 API, your JSON payload can specify a `context` object containing:

1. **`allowed_company_ids` (`array of integers`)**:
   - Defines the exact set of companies authorized for the active request.
   - Evaluated directly by Odoo record rules (`('company_id', 'in', allowed_company_ids)`).
   - The first ID in the array (`allowed_company_ids[0]`) becomes the **Active / Current Company** (`self.env.company`), dictating default sequences, fiscal positions, and accounting accounts.
2. **`company_id` (`integer`)**:
   - Acts as an explicit fallback for record creation defaults.

> [!CAUTION]
> If you omit `allowed_company_ids` from the request `context`, Odoo will default to the authenticated user's primary company (`user.company_id.id`). This will cause requests intended for subsidiary companies to either return empty datasets or fail with `403 Forbidden`.

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

### Required HTTP Headers

```http
Host: mycompany.odoo.com
Authorization: bearer <YOUR_160_BIT_API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-Ecommerce-MultiCompany/1.0
```

### Multi-Company JSON-2 Payload Structure

To interact with a specific company or set of companies, pass `allowed_company_ids` inside the `"context"` dictionary:

```json
{
  "context": {
    "allowed_company_ids": [1, 2],
    "company_id": 1,
    "lang": "en_US"
  },
  "domain": [["company_id", "=", 1]],
  "fields": ["name", "amount_total", "company_id"]
}
```

### Service Bot User Configuration Prerequisites

Before using the JSON-2 API with multi-company endpoints:

1. In Odoo, navigate to **Settings > Users & Companies > Users**.
2. Select the integration bot user (e.g. `ecommerce_sync_bot`).
3. Under the **Multi Companies** section:
   - **Allowed Companies (`company_ids`):** Check **ALL** companies the bot needs to query or write to.
   - **Default Company (`company_id`):** Set the default fallback company.
4. If a bot user attempts to pass a company ID in `allowed_company_ids` that is **not** in their allowed companies list, Odoo immediately returns an `AccessError` (`403 Forbidden`).

---

## 3. Model & Field References

### 3.1 Company Master Record (`res.company`)

The `res.company` model represents legal business entities, branches, and subsidiaries.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `Integer` | **Read-Only** | — | Unique internal ID of the company. |
| `name` | `Char` | **REQUIRED** | — | Legal business name or branch name (e.g. `"MyStore US Inc."`). |
| `currency_id` | `Many2one` (`res.currency`) | **REQUIRED** | System default | Base operating currency for accounting, pricing, and orders (e.g. `[1, "USD"]`). |
| `partner_id` | `Many2one` (`res.partner`) | Auto-created | — | Linked partner record representing the company's own legal contact card. |
| `parent_id` | `Many2one` (`res.company`) | Optional | `false` | Parent holding company ID. Set if this entity is a subsidiary or branch. |
| `child_ids` | `One2many` (`res.company`) | Optional | `[]` | List of child subsidiaries or regional branches. |
| `active` | `Boolean` | Optional | `true` | Whether the company is active. Inactive companies are hidden from standard queries. |
| `street` | `Char` | Optional | `null` | Primary street address. |
| `street2` | `Char` | Optional | `null` | Secondary street address / suite. |
| `city` | `Char` | Optional | `null` | City of operation. |
| `state_id` | `Many2one` (`res.country.state`) | Optional | `null` | State / Province ID. |
| `zip` | `Char` | Optional | `null` | Postal / Zip code. |
| `country_id` | `Many2one` (`res.country`) | **REQUIRED** | System default | Country ID (e.g. `[233, "United States"]`). Governs tax rules and localization. |
| `vat` | `Char` | Optional | `null` | Tax identification number / VAT number. |
| `company_registry` | `Char` | Optional | `null` | Commercial registration / business license number. |
| `email` | `Char` | Optional | `null` | Company official support/billing email address. |
| `phone` | `Char` | Optional | `null` | Primary customer service or office phone number. |
| `website` | `Char` | Optional | `null` | Public storefront or corporate website URL. |
| `logo` | `Binary` (Base64) | Optional | `null` | Base64-encoded corporate logo image. |

---

### 3.2 User Company Permissions (`res.users`)

The `res.users` model controls which companies an API key has authorization to access.

| Field Name | Type | Requirement | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- |
| `id` | `Integer` | **Read-Only** | User ID. |
| `name` | `Char` | **Read-Only** | User display name (e.g. `"E-Commerce Integration Bot"`). |
| `login` | `Char` | **Read-Only** | Login email or username. |
| `company_id` | `Many2one` (`res.company`) | **Read-Only** | Default primary company when no context is specified. |
| `company_ids` | `Many2many` (`res.company`) | **Read-Only** | Array of all company IDs this user is authorized to access. |

---

### 3.3 Company Relationship Matrix across Odoo Models

| Target Model | Field Name | Type | Multi-Company Behavior |
| :--- | :--- | :--- | :--- |
| **`sale.order`** | `company_id` | `Many2one` | **Strictly Scoped:** Every sales order must belong to exactly one company. Cannot be shared. |
| **`account.move`** | `company_id` | `Many2one` | **Strictly Scoped:** Invoices and journal entries must belong to a specific company for legal accounting. |
| **`stock.warehouse`** | `company_id` | `Many2one` | **Strictly Scoped:** Warehouses belong to one legal company. |
| **`stock.quant`** | `company_id` | `Many2one` | **Strictly Scoped:** Physical inventory counts are tracked per warehouse/company. |
| **`product.template`** | `company_id` | `Many2one` | **Hybrid:** If set, product is exclusive to that company. If `false` / `null`, product is shared globally. |
| **`res.partner`** | `company_id` | `Many2one` | **Hybrid:** If set, contact is private to that company. If `false` / `null`, contact is shared across all stores. |
| **`product.pricelist`** | `company_id` | `Many2one` | **Hybrid:** If set, pricelist applies only to that company. If `false`, available across all companies. |
| **`loyalty.program`** | `company_id` | `Many2one` | **Hybrid:** Can be restricted to one company or shared globally across all brands. |

---

## 4. Fetching Companies & Their IDs (`res.company`)

### Step 1: List All Active Companies & IDs

To retrieve all companies configured in Odoo along with their IDs, currencies, and countries, use `res.company.search_read`.

#### HTTP Request
```http
POST /json/2/res.company/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "domain": [["active", "=", true]],
  "fields": [
    "id",
    "name",
    "currency_id",
    "country_id",
    "parent_id",
    "vat",
    "email",
    "phone",
    "website"
  ],
  "order": "id asc"
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 1,
    "name": "MyStore US Inc.",
    "currency_id": [1, "USD"],
    "country_id": [233, "United States"],
    "parent_id": false,
    "vat": "US123456789",
    "email": "us-orders@mystore.com",
    "phone": "+1 (555) 019-2831",
    "website": "https://us.mystore.com"
  },
  {
    "id": 2,
    "name": "MyStore Europe B.V.",
    "currency_id": [2, "EUR"],
    "country_id": [156, "Netherlands"],
    "parent_id": false,
    "vat": "NL876543210B01",
    "email": "eu-orders@mystore.com",
    "phone": "+31 20 123 4567",
    "website": "https://eu.mystore.com"
  },
  {
    "id": 3,
    "name": "MyStore UK Ltd.",
    "currency_id": [3, "GBP"],
    "country_id": [231, "United Kingdom"],
    "parent_id": 1,
    "vat": "GB987654321",
    "email": "uk-orders@mystore.com",
    "phone": "+44 20 7946 0912",
    "website": "https://uk.mystore.com"
  }
]
```

---

### Step 2: Fetch Single Company Metadata by ID

If you already know the company ID (e.g. ID `2`), retrieve its full profile using `res.company.read`.

#### HTTP Request
```http
POST /json/2/res.company/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "ids": [2],
  "fields": [
    "id",
    "name",
    "currency_id",
    "street",
    "street2",
    "city",
    "zip",
    "country_id",
    "vat",
    "email",
    "phone"
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 2,
    "name": "MyStore Europe B.V.",
    "currency_id": [2, "EUR"],
    "street": "Keizersgracht 421",
    "street2": "Building C, Suite 4",
    "city": "Amsterdam",
    "zip": "1016 EK",
    "country_id": [156, "Netherlands"],
    "vat": "NL876543210B01",
    "email": "eu-orders@mystore.com",
    "phone": "+31 20 123 4567"
  }
]
```

---

### Step 3: Discover Authenticated Bot User's Allowed Companies

To verify at runtime which companies the active API key is authorized to access, query the user profile via `res.users.search_read` filtered to the current user login or ID.

#### HTTP Request
```http
POST /json/2/res.users/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "domain": [["id", "=", 6]],
  "fields": ["id", "name", "login", "company_id", "company_ids"]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 6,
    "name": "E-Commerce Sync Bot",
    "login": "ecommerce_sync_bot",
    "company_id": [1, "MyStore US Inc."],
    "company_ids": [1, 2, 3]
  }
]
```

> [!TIP]
> The `company_ids` field contains the list of all valid integer IDs `[1, 2, 3]` that can be safely passed into `context.allowed_company_ids`.

---

### Step 4: Querying Parent-Subsidiary & Branch Hierarchies

For multi-tier corporate setups (e.g. Global Holding -> Regional Companies -> Local Store Branches), query the parent-child tree.

#### HTTP Request
```http
POST /json/2/res.company/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "domain": [["parent_id", "=", 1]],
  "fields": ["id", "name", "parent_id", "child_ids"]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 3,
    "name": "MyStore UK Ltd.",
    "parent_id": [1, "MyStore US Inc."],
    "child_ids": []
  }
]
```

---

## 5. Fetching Data from a Specific Company

### Why Both Context and Domain Filters are Required

When querying or modifying records in a specific company, backend developers often ask whether to use `context.allowed_company_ids` or `domain: [["company_id", "=", COMPANY_ID]]`.

**The answer is BOTH:**

1. **`context: { "allowed_company_ids": [COMPANY_ID], "company_id": COMPANY_ID }`:**
   - Authorizes Odoo's SQL Record Rule engine to grant read/write access to that company's private records.
   - Directs Odoo to compute company-dependent properties (such as product taxes, company accounts, currency formatting, and default warehouse locations) relative to that specific company.
2. **`domain: [["company_id", "=", COMPANY_ID]]`:**
   - Filters the database result set exclusively to records explicitly assigned to that company, strictly excluding shared records (`company_id = False`) and records from other companies if multiple IDs are allowed.

---

### 5.1 Querying Sales Orders for a Specific Company (`sale.order`)

Fetch sales orders belonging exclusively to **Company 2 (MyStore Europe B.V.)**:

#### HTTP Request
```http
POST /json/2/sale.order/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [2],
    "company_id": 2
  },
  "domain": [
    ["company_id", "=", 2],
    ["state", "in", ["sale", "done"]]
  ],
  "fields": [
    "id",
    "name",
    "client_order_ref",
    "partner_id",
    "amount_total",
    "currency_id",
    "company_id",
    "date_order"
  ],
  "limit": 50,
  "order": "date_order desc"
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 1042,
    "name": "S01042",
    "client_order_ref": "EU-ORD-2026-8812",
    "partner_id": [402, "Lars Van Dijk"],
    "amount_total": 149.50,
    "currency_id": [2, "EUR"],
    "company_id": [2, "MyStore Europe B.V."],
    "date_order": "2026-08-25 14:22:10"
  }
]
```

---

### 5.2 Querying Stock Levels & Warehouses for a Specific Company (`stock.quant` & `stock.warehouse`)

Inventory quantities in Odoo are warehouse-dependent, and warehouses are company-dependent.

#### Step 5.2.1: Fetch Warehouses for Company 1 (US Store)
```http
POST /json/2/stock.warehouse/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [1],
    "company_id": 1
  },
  "domain": [["company_id", "=", 1]],
  "fields": ["id", "name", "code", "lot_stock_id", "company_id"]
}
```

#### Response (`200 OK`)
```json
[
  {
    "id": 1,
    "name": "US Central Warehouse",
    "code": "USWH",
    "lot_stock_id": [8, "USWH/Stock"],
    "company_id": [1, "MyStore US Inc."]
  }
]
```

#### Step 5.2.2: Query Available Stock for Product in Company 1
```http
POST /json/2/stock.quant/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [1],
    "company_id": 1
  },
  "domain": [
    ["company_id", "=", 1],
    ["product_id", "=", 501],
    ["location_id.usage", "=", "internal"]
  ],
  "fields": [
    "id",
    "product_id",
    "location_id",
    "quantity",
    "reserved_quantity",
    "available_quantity",
    "company_id"
  ]
}
```

#### Response (`200 OK`)
```json
[
  {
    "id": 312,
    "product_id": [501, "Wireless Noise-Canceling Headphones"],
    "location_id": [8, "USWH/Stock"],
    "quantity": 150.0,
    "reserved_quantity": 10.0,
    "available_quantity": 140.0,
    "company_id": [1, "MyStore US Inc."]
  }
]
```

---

### 5.3 Querying Invoices & Financial Records for a Specific Company (`account.move`)

Customer invoices must be strictly segregated by legal company for tax compliance and statutory accounting.

#### HTTP Request
```http
POST /json/2/account.move/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [2],
    "company_id": 2
  },
  "domain": [
    ["company_id", "=", 2],
    ["move_type", "=", "out_invoice"],
    ["state", "=", "posted"]
  ],
  "fields": [
    "id",
    "name",
    "partner_id",
    "invoice_date",
    "amount_total",
    "amount_residual",
    "payment_state",
    "currency_id",
    "company_id"
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 890,
    "name": "INV/2026/00451",
    "partner_id": [402, "Lars Van Dijk"],
    "invoice_date": "2026-08-25",
    "amount_total": 149.50,
    "amount_residual": 0.0,
    "payment_state": "paid",
    "currency_id": [2, "EUR"],
    "company_id": [2, "MyStore Europe B.V."]
  }
]
```

---

### 5.4 Querying Company-Exclusive Products (`product.template`)

If your catalog contains items that are only sold in specific regional branches (e.g. 110V US electronics vs 230V EU electronics):

#### HTTP Request
```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [1],
    "company_id": 1
  },
  "domain": [
    ["company_id", "=", 1],
    ["sale_ok", "=", true]
  ],
  "fields": [
    "id",
    "name",
    "default_code",
    "list_price",
    "currency_id",
    "company_id"
  ]
}
```

---

### 5.5 Creating Records Scoped to a Specific Company

When creating a new record (such as a Sales Order, Contact, or Warehouse), ensure both the payload `vals_list` and the `context` specify the target company:

#### HTTP Request
```http
POST /json/2/sale.order/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [2],
    "company_id": 2
  },
  "vals_list": [
    {
      "company_id": 2,
      "partner_id": 402,
      "client_order_ref": "EU-ORD-2026-9901",
      "order_line": [
        [
          0,
          0,
          {
            "product_id": 501,
            "product_uom_qty": 1,
            "price_unit": 149.50
          }
        ]
      ]
    }
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[1043]
```

---

## 6. Fetching & Managing Shared Data Between Companies

### 6.1 Understanding Globally Shared Records (`company_id = False`)

In Odoo, setting `company_id = False` (or omitting `company_id` during creation on hybrid models) designates the record as **Globally Shared**.

**What can be shared?**
- **Master Customers & Addresses (`res.partner`):** A single unified customer account can log in and place orders across your US, EU, and UK stores.
- **Master Product Catalog (`product.template` / `product.product`):** The same SKU, description, barcode, and media gallery can be shared across all regional stores.
- **Units of Measure (`uom.uom`) & Currencies (`res.currency`):** Universally shared system tables.
- **Global Loyalty Programs (`loyalty.program`):** Programs that award cross-brand rewards.

---

### 6.2 Querying Strictly Shared Records

To retrieve only records that are universally shared across all companies (records where `company_id` is unset):

#### HTTP Request: Fetch Globally Shared Products
```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "domain": [
    ["company_id", "=", false],
    ["sale_ok", "=", true]
  ],
  "fields": [
    "id",
    "name",
    "default_code",
    "list_price",
    "company_id"
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 501,
    "name": "Wireless Noise-Canceling Headphones",
    "default_code": "AUDIO-NC-001",
    "list_price": 199.99,
    "company_id": false
  },
  {
    "id": 502,
    "name": "USB-C Fast Charging Cable (2m)",
    "default_code": "CABLE-USBC-2M",
    "list_price": 19.99,
    "company_id": false
  }
]
```

---

### 6.3 Querying Shared + Specific Company Records Combined

In e-commerce storefronts, you typically want to display all **globally shared products** PLUS any **exclusive products** that belong to the active storefront's company.

This domain matches Odoo's native ORM record rule: `["|", ["company_id", "=", false], ["company_id", "=", COMPANY_ID]]`.

#### HTTP Request: Fetch US Store Catalog (Shared + US Exclusive Items)
```http
POST /json/2/product.template/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [1],
    "company_id": 1
  },
  "domain": [
    "|",
    ["company_id", "=", false],
    ["company_id", "=", 1],
    ["sale_ok", "=", true]
  ],
  "fields": [
    "id",
    "name",
    "default_code",
    "list_price",
    "company_id"
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 501,
    "name": "Wireless Noise-Canceling Headphones",
    "default_code": "AUDIO-NC-001",
    "list_price": 199.99,
    "company_id": false
  },
  {
    "id": 503,
    "name": "120V US High-Speed Power Adapter",
    "default_code": "PWR-120V-US",
    "list_price": 29.99,
    "company_id": [1, "MyStore US Inc."]
  }
]
```

---

### 6.4 Querying Data Across Multiple Authorized Companies in a Single Request

If you are building an administrative dashboard, a consolidated customer portal ("View all my orders across all brands"), or a global inventory aggregator, pass **multiple company IDs** in `allowed_company_ids`.

#### HTTP Request: Fetch Customer Orders Across US (ID: 1) and EU (ID: 2)
```http
POST /json/2/sale.order/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "context": {
    "allowed_company_ids": [1, 2],
    "company_id": 1
  },
  "domain": [
    ["partner_id", "=", 402],
    ["company_id", "in", [1, 2]]
  ],
  "fields": [
    "id",
    "name",
    "client_order_ref",
    "amount_total",
    "currency_id",
    "company_id",
    "state",
    "date_order"
  ],
  "order": "date_order desc"
}
```

#### HTTP Response (`200 OK`)
```json
[
  {
    "id": 1042,
    "name": "S01042",
    "client_order_ref": "EU-ORD-2026-8812",
    "amount_total": 149.50,
    "currency_id": [2, "EUR"],
    "company_id": [2, "MyStore Europe B.V."],
    "state": "sale",
    "date_order": "2026-08-25 14:22:10"
  },
  {
    "id": 981,
    "name": "S00981",
    "client_order_ref": "US-ORD-2026-1140",
    "amount_total": 219.00,
    "currency_id": [1, "USD"],
    "company_id": [1, "MyStore US Inc."],
    "state": "done",
    "date_order": "2026-07-14 09:15:33"
  }
]
```

---

### 6.5 Creating Globally Shared Records

To create a contact or product accessible across all companies, explicitly set `"company_id": false`:

#### HTTP Request: Create Shared Global Customer Partner
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db

{
  "vals_list": [
    {
      "name": "Alex Morgan",
      "email": "alex.morgan@example.com",
      "phone": "+1 555-0199",
      "is_company": false,
      "company_id": false
    }
  ]
}
```

#### HTTP Response (`200 OK`)
```json
[405]
```

---

## 7. Step-by-Step Multi-Storefront Integration Architecture

When integrating a multi-brand or multi-region e-commerce platform with Odoo, follow this standardized flow:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          E-Commerce Application                             │
│                  (Routes US, EU, and UK Storefronts)                        │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
   [US Storefront]            [EU Storefront]            [UK Storefront]
   Routing Key: "US"          Routing Key: "EU"          Routing Key: "UK"
   Target Company ID: 1       Target Company ID: 2       Target Company ID: 3
            │                          │                          │
            └──────────────────────────┼──────────────────────────┘
                                       │
                                       ▼ HTTP POST with context
┌─────────────────────────────────────────────────────────────────────────────┐
│                             Odoo JSON-2 API                                 │
│  1. Injects context: {"allowed_company_ids": [TargetID], "company_id": ...} │
│  2. Evaluates Multi-Company Record Rules                                    │
│  3. Processes Orders & Stock against Target Company's Warehouses/Pricelist │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Step 1: Bootstrapping & Storefront Mapping
At application startup, fetch all companies from Odoo and map each storefront domain/region to its corresponding Odoo Company ID:

```json
{
  "us.mystore.com": { "company_id": 1, "currency": "USD" },
  "eu.mystore.com": { "company_id": 2, "currency": "EUR" },
  "uk.mystore.com": { "company_id": 3, "currency": "GBP" }
}
```

### Step 2: Product Catalog Ingestion
Query master products (`company_id = False`) and augment them with company-specific stock levels (`stock.quant` filtered by the storefront's warehouse) and company-specific pricelists (`product.pricelist`).

### Step 3: Global Customer Identification
Search for existing customers across the global shared directory (`res.partner` where `company_id = False`). If not found, create the partner with `"company_id": false` so they can purchase on any storefront branch using the same identity.

### Step 4: Placing Company-Scoped Orders
When a customer checks out on `eu.mystore.com`:
- Set `context: { "allowed_company_ids": [2], "company_id": 2 }`
- Set `"company_id": 2` on the `sale.order` record payload.
- This ensures European tax calculation, Euro currency assignment, and automatic stock reservation in the European warehouse (`EUWH`).

---

## 8. Complete Code Reference (cURL, Node.js, Python)

### 8.1 cURL Examples

#### Example 1: Fetch All Companies and IDs
```bash
curl -X POST "https://mycompany.odoo.com/json/2/res.company/search_read" \
  -H "Authorization: bearer 6578616d706c6520617069206b657920313630626974" \
  -H "Content-Type: application/json" \
  -H "X-Odoo-Database: mycompany_db" \
  -d '{
    "domain": [["active", "=", true]],
    "fields": ["id", "name", "currency_id", "country_id", "vat", "email"],
    "order": "id asc"
  }'
```

#### Example 2: Fetch Sales Orders for Company 2
```bash
curl -X POST "https://mycompany.odoo.com/json/2/sale.order/search_read" \
  -H "Authorization: bearer 6578616d706c6520617069206b657920313630626974" \
  -H "Content-Type: application/json" \
  -H "X-Odoo-Database: mycompany_db" \
  -d '{
    "context": {
      "allowed_company_ids": [2],
      "company_id": 2
    },
    "domain": [["company_id", "=", 2]],
    "fields": ["id", "name", "amount_total", "currency_id", "date_order"],
    "limit": 10
  }'
```

#### Example 3: Fetch Shared Master Products
```bash
curl -X POST "https://mycompany.odoo.com/json/2/product.template/search_read" \
  -H "Authorization: bearer 6578616d706c6520617069206b657920313630626974" \
  -H "Content-Type: application/json" \
  -H "X-Odoo-Database: mycompany_db" \
  -d '{
    "domain": [["company_id", "=", false], ["sale_ok", "=", true]],
    "fields": ["id", "name", "default_code", "list_price"]
  }'
```

---

### 8.2 Node.js / TypeScript SDK Client

```typescript
import axios, { AxiosInstance } from 'axios';

export interface Company {
  id: number;
  name: string;
  currency_id: [number, string];
  country_id: [number, string];
  parent_id: [number, string] | false;
  vat: string | false;
  email: string | false;
}

export class OdooMultiCompanyClient {
  private client: AxiosInstance;

  constructor(
    private readonly baseURL: string,
    private readonly apiKey: string,
    private readonly database: string
  ) {
    this.client = axios.create({
      baseURL: `${this.baseURL}/json/2`,
      headers: {
        'Authorization': `bearer ${this.apiKey}`,
        'Content-Type': 'application/json; charset=utf-8',
        'X-Odoo-Database': this.database,
      },
    });
  }

  /**
   * 1. Get all active companies and their IDs
   */
  async getCompanies(): Promise<Company[]> {
    const response = await this.client.post('/res.company/search_read', {
      domain: [['active', '=', true]],
      fields: ['id', 'name', 'currency_id', 'country_id', 'parent_id', 'vat', 'email', 'phone'],
      order: 'id asc',
    });
    return response.data;
  }

  /**
   * 2. Query data for a specific company
   */
  async getCompanySalesOrders(companyId: number, limit = 20) {
    const response = await this.client.post('/sale.order/search_read', {
      context: {
        allowed_company_ids: [companyId],
        company_id: companyId,
      },
      domain: [
        ['company_id', '=', companyId],
        ['state', 'in', ['sale', 'done']],
      ],
      fields: ['id', 'name', 'client_order_ref', 'partner_id', 'amount_total', 'currency_id'],
      limit,
    });
    return response.data;
  }

  /**
   * 3. Query shared master products (company_id = False)
   */
  async getSharedProducts() {
    const response = await this.client.post('/product.template/search_read', {
      domain: [
        ['company_id', '=', false],
        ['sale_ok', '=', true],
      ],
      fields: ['id', 'name', 'default_code', 'list_price', 'company_id'],
    });
    return response.data;
  }

  /**
   * 4. Query combined catalog for a specific storefront (Shared + Company-Specific)
   */
  async getStorefrontCatalog(companyId: number) {
    const response = await this.client.post('/product.template/search_read', {
      context: {
        allowed_company_ids: [companyId],
        company_id: companyId,
      },
      domain: [
        '|',
        ['company_id', '=', false],
        ['company_id', '=', companyId],
        ['sale_ok', '=', true],
      ],
      fields: ['id', 'name', 'default_code', 'list_price', 'company_id'],
    });
    return response.data;
  }

  /**
   * 5. Query customer orders across multiple authorized companies
   */
  async getMultiCompanyCustomerOrders(partnerId: number, companyIds: number[]) {
    const response = await this.client.post('/sale.order/search_read', {
      context: {
        allowed_company_ids: companyIds,
      },
      domain: [
        ['partner_id', '=', partnerId],
        ['company_id', 'in', companyIds],
      ],
      fields: ['id', 'name', 'client_order_ref', 'amount_total', 'currency_id', 'company_id', 'state'],
    });
    return response.data;
  }
}
```

---

### 8.3 Python Integration Client

```python
import requests
from typing import List, Dict, Any, Optional

class OdooMultiCompanyAPI:
    def __init__(self, base_url: str, api_key: str, database: str):
        self.base_url = base_url.rstrip("/")
        self.headers = {
            "Authorization": f"bearer {api_key}",
            "Content-Type": "application/json; charset=utf-8",
            "X-Odoo-Database": database,
            "User-Agent": "MyStore-MultiCompany-Client/1.0",
        }

    def _call(self, model: str, method: str, payload: Dict[str, Any]) -> Any:
        url = f"{self.base_url}/json/2/{model}/{method}"
        response = requests.post(url, json=payload, headers=self.headers)
        response.raise_for_status()
        return response.json()

    def get_all_companies(self) -> List[Dict[str, Any]]:
        """Fetch all active companies with IDs and currencies."""
        payload = {
            "domain": [["active", "=", True]],
            "fields": ["id", "name", "currency_id", "country_id", "parent_id", "vat", "email"],
            "order": "id asc",
        }
        return self._call("res.company", "search_read", payload)

    def get_company_data(
        self, 
        model: str, 
        company_id: int, 
        domain: Optional[List[Any]] = None, 
        fields: Optional[List[str]] = None
    ) -> List[Dict[str, Any]]:
        """Fetch data scoped to a specific company."""
        domain = domain or []
        # Ensure company_id check is in domain
        scoped_domain = [["company_id", "=", company_id]] + domain
        
        payload = {
            "context": {
                "allowed_company_ids": [company_id],
                "company_id": company_id,
            },
            "domain": scoped_domain,
            "fields": fields or [],
        }
        return self._call(model, "search_read", payload)

    def get_shared_data(
        self, 
        model: str, 
        domain: Optional[List[Any]] = None, 
        fields: Optional[List[str]] = None
    ) -> List[Dict[str, Any]]:
        """Fetch globally shared records (company_id = False)."""
        domain = domain or []
        shared_domain = [["company_id", "=", False]] + domain
        payload = {
            "domain": shared_domain,
            "fields": fields or [],
        }
        return self._call(model, "search_read", payload)

    def get_shared_and_company_data(
        self, 
        model: str, 
        company_id: int, 
        additional_domain: Optional[List[Any]] = None, 
        fields: Optional[List[str]] = None
    ) -> List[Dict[str, Any]]:
        """Fetch both shared records and records specific to one company."""
        combined_domain = ["|", ["company_id", "=", False], ["company_id", "=", company_id]]
        if additional_domain:
            combined_domain = additional_domain + combined_domain

        payload = {
            "context": {
                "allowed_company_ids": [company_id],
                "company_id": company_id,
            },
            "domain": combined_domain,
            "fields": fields or [],
        }
        return self._call(model, "search_read", payload)

    def create_company_record(
        self, 
        model: str, 
        company_id: int, 
        values: Dict[str, Any]
    ) -> int:
        """Create a record explicitly belonging to a company."""
        record_values = dict(values)
        record_values["company_id"] = company_id

        payload = {
            "context": {
                "allowed_company_ids": [company_id],
                "company_id": company_id,
            },
            "vals_list": [record_values],
        }
        result = self._call(model, "create", payload)
        return result[0]
```

---

## 9. Error Handling & Multi-Company Edge Cases

### 9.1 AccessError: Unauthorized Company Requested (`403 Forbidden`)

#### Scenario
The API client requests records with `allowed_company_ids: [5]`, but the authenticated bot user is only assigned companies `[1, 2]`.

#### HTTP Response (`403 Forbidden`)
```json
{
  "error": {
    "code": 403,
    "message": "Access Error",
    "data": {
      "name": "odoo.exceptions.AccessError",
      "message": "You are not allowed to access 'Sales Order' (sale.order) records.\n\nThis operation is allowed for the following groups:\n - Sales/User\n\nCompanies: MyStore Asia Ltd. (id=5) is not in your allowed companies (id=1, 2)."
    }
  }
}
```

#### Remediation
1. Log in as an Administrator in Odoo.
2. Go to **Settings > Users > [Your Bot User]**.
3. Add the missing company to **Allowed Companies**.

---

### 9.2 Cross-Company Relational Mismatches (`_check_company` Violation)

#### Scenario
An API call creates a Sales Order for **Company 1 (US)**, but references a Warehouse (`warehouse_id`) or Payment Journal (`journal_id`) that belongs to **Company 2 (EU)**.

#### HTTP Response (`400 Bad Request`)
```json
{
  "error": {
    "code": 400,
    "message": "User Error",
    "data": {
      "name": "odoo.exceptions.UserError",
      "message": "Incompatible companies on records:\n- 'Sales Order S01045' belongs to company 'MyStore US Inc.' (id=1)\n- 'USWH/Stock' warehouse belongs to company 'MyStore Europe B.V.' (id=2)\n\nRecords must belong to the same company."
    }
  }
}
```

#### Remediation
Ensure all relational foreign keys (`warehouse_id`, `pricelist_id`, `journal_id`, `fiscal_position_id`) passed in the payload either belong to the exact same `company_id` or are globally shared (`company_id = False`).

---

### 9.3 Missing `allowed_company_ids` Fallback Behavior

#### Scenario
The developer calls `/json/2/sale.order/search_read` without providing `"context": {"allowed_company_ids": [...]}`.

#### Behavior
- Odoo automatically falls back to the bot user's default company (`user.company_id.id`).
- Orders belonging to subsidiary companies will be silently omitted from the response due to the default multi-company record rule.
- No explicit error is raised; the API simply returns an empty array `[]`.

#### Remediation
Always explicitly provide `"context": {"allowed_company_ids": [TARGET_COMPANY_ID], "company_id": TARGET_COMPANY_ID}` on every external API invocation.

---

## 10. Production Best Practices & Checklist

1. **Explicit Context Hygiene:** Never rely on default user context. Every API wrapper method should accept a `company_id` or `company_ids` parameter and inject `allowed_company_ids` into the request context.
2. **Company ID Caching:** Cache the list of active companies (`res.company`) and their IDs in your backend memory/Redis on startup with a TTL (e.g. 1 hour) to eliminate redundant metadata calls.
3. **Dedicated Bot Accounts per Organization or Master Bot:** For maximum security, you can either assign all companies to a single centralized e-commerce bot user or maintain separate bot users per country storefront.
4. **Relational Field Verification:** When creating sales orders or invoices, verify that referenced products, pricelists, warehouses, and journals belong to the target company or have `company_id = False`.
5. **Shared Customer Directory (`res.partner`):** Default to creating customer contacts with `"company_id": false` unless your business requires strict legal customer segregation between subsidiaries.
6. **Cross-Company Transaction Automation:** If one company sells goods fulfilled from another company's warehouse, configure Odoo's native **Inter-Company Transactions** module (`inter_company_rules`) to generate automatic mirror Purchase Orders and Sales Orders.
