# Odoo JSON-2 API: Contacts (`res.partner`) Integration Guide

This guide provides an end-to-end technical reference for backend developers integrating an **e-commerce platform** with Odoo's **Contacts (`res.partner`)** module using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Architectural Overview & Core Concepts](#1-architectural-overview--core-concepts)
2. [Authentication & HTTP Headers](#2-authentication--http-headers)
3. [Field Reference for `res.partner`](#3-field-reference-for-respartner)
4. [Relational Field Formatting Guide](#4-relational-field-formatting-guide)
5. [Step-by-Step E-Commerce Integration Workflow](#5-step-by-step-e-commerce-integration-workflow)
   - [Step 1: Geographical Lookup (Countries & States)](#step-1-geographical-lookup-countries--states)
   - [Step 2: Customer Deduplication & Search](#step-2-customer-deduplication--search)
   - [Step 3: Creating a Customer (B2C Individual vs. B2B Company)](#step-3-creating-a-customer-b2c-individual-vs-b2b-company)
   - [Step 4: Managing Separate Shipping & Billing Addresses](#step-4-managing-separate-shipping--billing-addresses)
   - [Step 5: Atomic Customer Creation with Addresses (Single Request)](#step-5-atomic-customer-creation-with-addresses-single-request)
   - [Step 6: Fetching Customer Profile & Address Book](#step-6-fetching-customer-profile--address-book)
   - [Step 7: Updating Contact Data](#step-7-updating-contact-data)
   - [Step 8: Archiving & Deleting Contacts](#step-8-archiving--deleting-contacts)
6. [Complete HTTP Request & Response Reference](#6-complete-http-request--response-reference)
7. [Error Handling & Failure Responses](#7-error-handling--failure-responses)
8. [Production Best Practices for E-Commerce](#8-production-best-practices-for-e-commerce)

---

## 1. Architectural Overview & Core Concepts

The JSON-2 API exposes Odoo ORM methods over standard HTTP requests.

### Endpoint Structure
Every request is an **HTTP `POST`** to the following URL format:

```text
POST https://<odoo-domain>/json/2/<model>/<method>
```

- **`<model>`**: The technical Odoo model name (e.g., `res.partner`, `res.country`, `res.country.state`).
- **`<method>`**: The public ORM method to call (e.g., `create`, `read`, `write`, `search`, `search_read`, `unlink`, `fields_get`).

### Request Envelope
The body must be a valid JSON object containing named arguments matching the target method signature:

```json
{
  "ids": [101, 102],
  "context": {
    "lang": "en_US"
  },
  "fields": ["name", "email"]
}
```

- `ids` (*array of integers*, optional): The record IDs when calling instance methods (such as `read`, `write`, `unlink`). Omitted when calling model-level methods (such as `create`, `search_read`).
- `context` (*object*, optional): Session context dictionary (e.g., `{"lang": "en_US"}`).
- *`named_parameters`*: Method arguments passed by name (e.g., `vals_list` for `create`, `vals` for `write`, `domain` for `search_read`).

### Transactions & Consistency
Every JSON-2 HTTP request executes inside its own dedicated SQL transaction:
- **Success (`200 OK`)**: The transaction commits automatically.
- **Failure (`4xx` / `5xx`)**: The transaction rolls back automatically.

---

## 2. Authentication & HTTP Headers

Authentication uses **API Keys** passed as a `bearer` token in the `Authorization` header.

### Generating an API Key
1. In Odoo, navigate to **User Profile / Preferences > Account Security**.
2. Click **New API Key**, provide a description (e.g. `Ecommerce Integration Bot`), set the expiration, and copy the 160-bit key string.
3. For automated systems, create a dedicated bot user in Odoo (e.g., `ecommerce_sync_bot`) with least-privilege access rights to the Contacts module.

### HTTP Headers Table

| Header | Requirement | Example Value | Description |
| :--- | :--- | :--- | :--- |
| `Host` | **Required** | `mycompany.odoo.com` | Hostname of the Odoo instance. |
| `Authorization` | **Required** | `bearer <API_KEY>` | Bearer token containing the API key. |
| `Content-Type` | **Required** | `application/json; charset=utf-8` | Indicates JSON payload. |
| `X-Odoo-Database` | **Conditional** | `mycompany_prod` | Required if the host manages multiple databases. |
| `User-Agent` | **Recommended** | `MyStore-Ecommerce/1.0` | Client identification string for logging. |

---

## 3. Field Reference for `res.partner`

Below is the complete reference of fields available in `res.partner` for an e-commerce integration.

### 3.1 Primary Identification & Classification

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | — | Full name of the contact or legal name of the company. |
| `is_company` | `Boolean` | Optional | `false` | `true` if this contact is a company/organization (B2B); `false` if an individual person (B2C). |
| `company_type` | `Selection` | Optional | `'person'` | Synced with `is_company`. Values: `'person'` or `'company'`. |
| `parent_id` | `Many2one` (`res.partner`) | Optional | `null` | The parent company ID (for employees/B2B contacts) or parent account ID (for child shipping/billing addresses). |
| `type` | `Selection` | Optional | `'contact'` | Address role: `'contact'` (Main/Default), `'invoice'` (Billing Address), `'delivery'` (Shipping Address), `'other'` (Other), `'private'` (Private Address). |
| `ref` | `Char` | Optional | `null` | External Unique Identifier / Customer ID from your e-commerce store (e.g., `ECOMM-CUST-88219`). Highly recommended for idempotency and indexing. |
| `active` | `Boolean` | Optional | `true` | Set to `false` to soft-delete/archive the contact without breaking historical order relations. |

### 3.2 Contact & Communication

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `email` | `Char` | Optional | `null` | Primary email address. Essential for customer login matching, order confirmations, and invoices. |
| `phone` | `Char` | Optional | `null` | Primary telephone / landline number. |
| `mobile` | `Char` | Optional | `null` | Mobile phone number (used for delivery SMS notifications and shipping courier contact). |
| `website` | `Char` | Optional | `null` | Customer / Company website URL. |
| `lang` | `Selection` | Optional | System default | Preferred customer language code (e.g., `'en_US'`, `'fr_FR'`, `'es_ES'`, `'de_DE'`, `'ar_001'`). Used for localized transactional emails and PDF invoices. |

### 3.3 Physical & Postal Address

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `street` | `Char` | Optional | `null` | Street address line 1 (e.g., `742 Evergreen Terrace`). |
| `street2` | `Char` | Optional | `null` | Street address line 2 (e.g., `Apt 4B, Building 2, Suite 100`). |
| `city` | `Char` | Optional | `null` | City / Town / Municipality name (e.g., `Springfield`). |
| `zip` | `Char` | Optional | `null` | Postal / ZIP code (e.g., `97477`). |
| `state_id` | `Many2one` (`res.country.state`) | Optional | `null` | Odoo internal integer ID for the State / Province / Region. |
| `country_id` | `Many2one` (`res.country`) | Optional | `null` | Odoo internal integer ID for the Country. |

### 3.4 Financial, Tax & B2B Invoicing

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `vat` | `Char` | Optional | `null` | Tax Identification Number / VAT Number / GST / EIN for business customers. |
| `customer_rank` | `Integer` | Optional | `1` | Integer counter indicating customer activity. Set to `>= 1` to designate the partner as a customer in filters. |
| `supplier_rank` | `Integer` | Optional | `0` | Set to `>= 1` if the partner is a vendor/supplier. Defaults to `0` for customers. |
| `property_payment_term_id` | `Many2one` (`account.payment.term`) | Optional | `null` | Payment terms ID (e.g., Immediate Payment, Net 30). |
| `property_account_position_id` | `Many2one` (`account.fiscal.position`) | Optional | `null` | Fiscal position ID for automatic tax mappings (e.g., EU intra-community, tax-exempt). |

### 3.5 Categorization & Relationships

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `category_id` | `Many2many` (`res.partner.category`) | Optional | `[]` | List of partner tag IDs (e.g., `["VIP"]`, `["Wholesale"]`, `["Newsletter"]`). |
| `user_id` | `Many2one` (`res.users`) | Optional | `null` | Internal salesperson assigned to this customer. |
| `team_id` | `Many2one` (`crm.team`) | Optional | `null` | Sales team assigned to this customer. |
| `comment` | `Html` / `Text` | Optional | `null` | Internal notes about the customer (not visible on external portal). |
| `child_ids` | `One2many` (`res.partner`) | Optional | `[]` | Array of child contacts / extra delivery and invoice addresses. |

---

## 4. Relational Field Formatting Guide

When creating or updating records in Odoo via JSON-2, relational fields follow strict conventions:

### 1. `Many2one` Fields (e.g., `country_id`, `state_id`, `parent_id`)
- **Writing / Creating:** Provide the integer record ID (or `false` / `null` to unset):
  ```json
  "country_id": 233,
  "state_id": 1422,
  "parent_id": 501
  ```
- **Reading (`read` or `search_read`):** Returns a 2-element tuple `[id, "Display Name"]` or `false` when empty:
  ```json
  "country_id": [233, "United States"],
  "state_id": [1422, "California"]
  ```

### 2. `Many2many` Fields (e.g., `category_id`)
To manipulate tag sets, use Odoo's ORM command tuples:
- **Set/Replace all tags:** `[[6, 0, [tag_id_1, tag_id_2]]]`
- **Add tag without removing existing:** `[[4, tag_id, 0]]`
- **Remove specific tag:** `[[3, tag_id, 0]]`

```json
"category_id": [
  [6, 0, [2, 7]]
]
```

### 3. `One2many` Fields (e.g., `child_ids`)
To create or update nested child records (such as addresses) inside the parent payload:
- **Create new child record inline:** `[[0, 0, {"name": "...", "type": "delivery", ...}]]`
- **Update existing child record:** `[[1, child_id, {"street": "...", ...}]]`
- **Delete child record:** `[[2, child_id, 0]]`

---

## 5. Step-by-Step E-Commerce Integration Workflow

```
[E-Commerce Platform]                  [Odoo JSON-2 API]
         |                                     |
         |---- 1. Search Country & State ----->| (/json/2/res.country/search_read)
         |<--- Return IDs ---------------------|
         |                                     |
         |---- 2. Search Customer by Email --->| (/json/2/res.partner/search_read)
         |<--- Return existing ID or [] -------|
         |                                     |
    [If not found]                             |
         |---- 3. Create Main Customer ------->| (/json/2/res.partner/create)
         |<--- Return [customer_id] -----------|
         |                                     |
         |---- 4. Create Delivery Address ---->| (/json/2/res.partner/create)
         |<--- Return [delivery_address_id] ---|
         |                                     |
    [If found]                                 |
         |---- 5. Update Customer Info ------->| (/json/2/res.partner/write)
         |<--- Return true --------------------|
```

---

### Step 1: Geographical Lookup (Countries & States)

Before creating contacts with addresses, map your e-commerce ISO-2 country codes (e.g., `US`, `DE`, `CA`) and state codes (e.g., `CA`, `NY`, `TX`) to Odoo internal IDs.

#### 1.1 Find Country ID by ISO-2 Code
```http
POST /json/2/res.country/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [["code", "=", "US"]],
  "fields": ["id", "name", "code"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 233,
    "name": "United States",
    "code": "US"
  }
]
```

#### 1.2 Find State ID by Country and State Code
```http
POST /json/2/res.country.state/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["country_id", "=", 233],
    ["code", "=", "CA"]
  ],
  "fields": ["id", "name", "code"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1422,
    "name": "California",
    "code": "CA"
  }
]
```

---

### Step 2: Customer Deduplication & Search

Before inserting a new customer, check if a partner with the same email or e-commerce reference already exists.

```http
POST /json/2/res.partner/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    "|",
    ["email", "=ilike", "alex.morgan@example.com"],
    ["ref", "=", "ECOMM-CUST-88219"]
  ],
  "fields": ["id", "name", "email", "ref", "is_company", "parent_id"],
  "limit": 1
}
```

**Success Response (`200 OK` - Match Found):**
```json
[
  {
    "id": 540,
    "name": "Alex Morgan",
    "email": "alex.morgan@example.com",
    "ref": "ECOMM-CUST-88219",
    "is_company": false,
    "parent_id": false
  }
]
```

**Success Response (`200 OK` - No Match):**
```json
[]
```

---

### Step 3: Creating a Customer (B2C Individual vs. B2B Company)

Call the `create` method with `vals_list` (an array containing objects of field values).

#### 3.1 Create a B2C Individual Customer
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Alex Morgan",
      "is_company": false,
      "company_type": "person",
      "email": "alex.morgan@example.com",
      "phone": "+1 415 555 0192",
      "mobile": "+1 415 555 0199",
      "street": "100 Market Street",
      "street2": "Suite 400",
      "city": "San Francisco",
      "state_id": 1422,
      "country_id": 233,
      "zip": "94105",
      "lang": "en_US",
      "ref": "ECOMM-CUST-88219",
      "customer_rank": 1
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[540]
```
*(Returns an array containing the newly created integer ID `540`)*

---

#### 3.2 Create a B2B Business / Corporate Customer
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Acme Innovations LLC",
      "is_company": true,
      "company_type": "company",
      "vat": "US123456789",
      "email": "procurement@acme-innovations.com",
      "phone": "+1 212 555 8800",
      "website": "https://www.acme-innovations.com",
      "street": "350 5th Avenue",
      "street2": "Floor 21",
      "city": "New York",
      "state_id": 1445,
      "country_id": 233,
      "zip": "10118",
      "lang": "en_US",
      "ref": "ECOMM-B2B-1092",
      "customer_rank": 1
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[541]
```

---

### Step 4: Managing Separate Shipping & Billing Addresses

In Odoo, secondary delivery and invoicing addresses are child records attached to the parent customer using `parent_id` and differentiated by `type`.

#### 4.1 Create a Dedicated Delivery (Shipping) Address
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Alex Morgan (Warehouse Depot)",
      "parent_id": 540,
      "type": "delivery",
      "street": "900 Industrial Parkway",
      "street2": "Dock 4",
      "city": "Oakland",
      "state_id": 1422,
      "country_id": 233,
      "zip": "94607",
      "phone": "+1 510 555 9931",
      "mobile": "+1 510 555 9932"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[542]
```

#### 4.2 Create a Dedicated Invoice (Billing) Address
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Alex Morgan (Billing Department)",
      "parent_id": 540,
      "type": "invoice",
      "street": "100 Market Street",
      "street2": "Suite 400 - Accounts Payable",
      "city": "San Francisco",
      "state_id": 1422,
      "country_id": 233,
      "zip": "94105",
      "email": "invoices@alexmorgan.me"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[543]
```

---

### Step 5: Atomic Customer Creation with Addresses (Single Request)

To avoid multiple round trips, you can create the parent customer and all child addresses (shipping & billing) in a **single atomic transaction** using `child_ids` with `(0, 0, values)` command tuples:

```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Jordan Bell",
      "is_company": false,
      "email": "jordan.bell@example.com",
      "phone": "+1 312 555 0188",
      "ref": "ECOMM-CUST-90021",
      "customer_rank": 1,
      "street": "500 N Michigan Ave",
      "city": "Chicago",
      "state_id": 1432,
      "country_id": 233,
      "zip": "60611",
      "child_ids": [
        [
          0,
          0,
          {
            "name": "Jordan Bell (Shipping Address)",
            "type": "delivery",
            "street": "1200 S Lake Shore Dr",
            "city": "Chicago",
            "state_id": 1432,
            "country_id": 233,
            "zip": "60605",
            "phone": "+1 312 555 7711"
          }
        ],
        [
          0,
          0,
          {
            "name": "Jordan Bell (Billing Address)",
            "type": "invoice",
            "street": "500 N Michigan Ave",
            "city": "Chicago",
            "state_id": 1432,
            "country_id": 233,
            "zip": "60611"
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[544]
```

---

### Step 6: Fetching Customer Profile & Address Book

#### 6.1 Read Specific Fields of a Customer by ID
```http
POST /json/2/res.partner/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [540],
  "fields": [
    "name",
    "email",
    "phone",
    "mobile",
    "street",
    "street2",
    "city",
    "state_id",
    "country_id",
    "zip",
    "ref",
    "child_ids"
  ]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 540,
    "name": "Alex Morgan",
    "email": "alex.morgan@example.com",
    "phone": "+1 415 555 0192",
    "mobile": "+1 415 555 0199",
    "street": "100 Market Street",
    "street2": "Suite 400",
    "city": "San Francisco",
    "state_id": [1422, "California (US)"],
    "country_id": [233, "United States"],
    "zip": "94105",
    "ref": "ECOMM-CUST-88219",
    "child_ids": [542, 543]
  }
]
```

#### 6.2 Fetch Customer and all Associated Addresses in One Query
```http
POST /json/2/res.partner/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    "|",
    ["id", "=", 540],
    ["parent_id", "=", 540]
  ],
  "fields": [
    "id",
    "name",
    "type",
    "email",
    "phone",
    "street",
    "street2",
    "city",
    "state_id",
    "country_id",
    "zip"
  ]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 540,
    "name": "Alex Morgan",
    "type": "contact",
    "email": "alex.morgan@example.com",
    "phone": "+1 415 555 0192",
    "street": "100 Market Street",
    "street2": "Suite 400",
    "city": "San Francisco",
    "state_id": [1422, "California (US)"],
    "country_id": [233, "United States"],
    "zip": "94105"
  },
  {
    "id": 542,
    "name": "Alex Morgan (Warehouse Depot)",
    "type": "delivery",
    "email": false,
    "phone": "+1 510 555 9931",
    "street": "900 Industrial Parkway",
    "street2": "Dock 4",
    "city": "Oakland",
    "state_id": [1422, "California (US)"],
    "country_id": [233, "United States"],
    "zip": "94607"
  },
  {
    "id": 543,
    "name": "Alex Morgan (Billing Department)",
    "type": "invoice",
    "email": "invoices@alexmorgan.me",
    "phone": false,
    "street": "100 Market Street",
    "street2": "Suite 400 - Accounts Payable",
    "city": "San Francisco",
    "state_id": [1422, "California (US)"],
    "country_id": [233, "United States"],
    "zip": "94105"
  }
]
```

---

### Step 7: Updating Contact Data

To update a contact or address, call `write` with `ids` (array of record IDs) and `vals` (object of modified fields).

```http
POST /json/2/res.partner/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [540],
  "vals": {
    "phone": "+1 415 555 9988",
    "street": "150 California Street",
    "street2": "Floor 12",
    "zip": "94111"
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

### Step 8: Archiving & Deleting Contacts

#### 8.1 Soft Delete / Archiving (Recommended for E-Commerce)
Archiving preserves historical invoices, delivery orders, and quotations while removing the contact from active pickers.

```http
POST /json/2/res.partner/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [540],
  "vals": {
    "active": false
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

#### 8.2 Hard Deletion (`unlink`)
> [!CAUTION]
> If a partner is referenced by sales orders, accounting moves, or delivery slips, Odoo's foreign key constraints will block deletion and return an error. Use archiving (`active: false`) instead.

```http
POST /json/2/res.partner/unlink HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [542]
}
```

**Success Response (`200 OK`):**
```json
true
```

---

## 6. Complete HTTP Request & Response Reference

### 6.1 Introspecting Field Definitions (`fields_get`)
Discover all available fields, their technical types, help strings, and requirement rules directly from the running server.

#### Request
```http
POST /json/2/res.partner/fields_get HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "allfields": ["name", "email", "country_id", "type"],
  "attributes": ["type", "required", "string", "help", "relation"]
}
```

#### Success Response (`200 OK`)
```json
{
  "name": {
    "type": "char",
    "required": true,
    "string": "Name",
    "help": false
  },
  "email": {
    "type": "char",
    "required": false,
    "string": "Email",
    "help": false
  },
  "country_id": {
    "type": "many2one",
    "required": false,
    "string": "Country",
    "help": "Apply specific localized fiscal position and address formats",
    "relation": "res.country"
  },
  "type": {
    "type": "selection",
    "required": false,
    "string": "Address Type",
    "help": "Used to select between delivery, invoice, and other addresses",
    "selection": [
      ["contact", "Contact"],
      ["invoice", "Invoice Address"],
      ["delivery", "Delivery Address"],
      ["other", "Other Address"],
      ["private", "Private Address"]
    ]
  }
}
```

---

### 6.2 Filtered Search with Sorting & Pagination (`search_read`)

#### Request
```http
POST /json/2/res.partner/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["customer_rank", ">", 0],
    ["city", "=", "San Francisco"]
  ],
  "fields": ["name", "email", "phone", "city"],
  "offset": 0,
  "limit": 2,
  "order": "name asc"
}
```

#### Success Response (`200 OK`)
```json
[
  {
    "id": 540,
    "name": "Alex Morgan",
    "email": "alex.morgan@example.com",
    "phone": "+1 415 555 0192",
    "city": "San Francisco"
  },
  {
    "id": 482,
    "name": "Beatrice Stone",
    "email": "beatrice@stoneconsulting.com",
    "phone": "+1 415 555 3300",
    "city": "San Francisco"
  }
]
```

---

## 7. Error Handling & Failure Responses

When a JSON-2 API request fails, Odoo responds with an appropriate HTTP status code (`4xx` or `5xx`) and a structured JSON error object.

### Error Object Schema

| Property | Type | Description |
| :--- | :--- | :--- |
| `name` | `string` | The fully qualified Python exception class (e.g., `werkzeug.exceptions.Unauthorized`, `odoo.exceptions.ValidationError`, `odoo.exceptions.AccessError`). |
| `message` | `string` | Human-readable explanation of why the operation failed. |
| `arguments` | `array` | Raw positional arguments passed to the Python exception. |
| `context` | `object` | The execution context at the moment of failure. |
| `debug` | `string` | Python traceback stack trace (useful in development). |

---

### 7.1 Authentication Failure (`401 Unauthorized`)
Triggered when the `Authorization` header is missing, malformed, or the API key is revoked or expired.

#### Request
```http
POST /json/2/res.partner/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer invalid_or_expired_key_xyz
Content-Type: application/json; charset=utf-8

{
  "domain": [],
  "fields": ["name"]
}
```

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

### 7.2 Validation / Missing Required Field (`400 Bad Request` or `500 Internal Server Error`)
Triggered when attempting to create a record without a mandatory field (e.g. omitting `name`).

#### Request
```http
POST /json/2/res.partner/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "email": "noname.customer@example.com",
      "city": "Seattle"
    }
  ]
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "The field 'name' is required on 'res.partner'.",
  "arguments": ["The field 'name' is required on 'res.partner'."],
  "context": {
    "lang": "en_US"
  },
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/models.py\", line 4501, in _check_field_value\n    raise ValidationError(f\"The field '{name}' is required on '{self._name}'.\")\nodoo.exceptions.ValidationError: The field 'name' is required on 'res.partner'.\n"
}
```

---

### 7.3 Record Not Found / Invalid ID (`404 Not Found` or `400 Bad Request`)
Triggered when calling `read` or `write` on a record ID that does not exist.

#### Request
```http
POST /json/2/res.partner/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [9999999],
  "fields": ["name", "email"]
}
```

#### Error Response (`404 Not Found`)
```json
{
  "name": "odoo.exceptions.MissingError",
  "message": "Record does not exist or has been deleted. (Record: res.partner(9999999,), User: 12)",
  "arguments": [
    "Record does not exist or has been deleted. (Record: res.partner(9999999,), User: 12)"
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/models.py\", line 5800, in ensure_one\n    raise MissingError(...)\nodoo.exceptions.MissingError: Record does not exist or has been deleted.\n"
}
```

---

### 7.4 Permission Denied (`403 Forbidden`)
Triggered when the authenticated API key user lacks read or write permissions on `res.partner`.

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to modify 'Contact' (res.partner) records. Contact your administrator for assistance.",
  "arguments": [
    "You are not allowed to modify 'Contact' (res.partner) records."
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/addons/base/models/ir_rule.py\", line 310, in check\n    raise AccessError(...)\nodoo.exceptions.AccessError: You are not allowed to modify 'Contact' (res.partner) records.\n"
}
```

---

### 7.5 Database Conflict / Foreign Key Constraint (`400 Bad Request`)
Triggered when attempting to hard delete a customer who has existing sales orders or invoices.

#### Request
```http
POST /json/2/res.partner/unlink HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer <API_KEY>
Content-Type: application/json; charset=utf-8

{
  "ids": [540]
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.UserError",
  "message": "You cannot delete this partner because it is linked to existing Sales Orders or Invoices. Archive it instead.",
  "arguments": [
    "You cannot delete this partner because it is linked to existing Sales Orders or Invoices. Archive it instead."
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/account/models/partner.py\", line 120, in unlink\n    raise UserError(_(\"You cannot delete...\"))\nodoo.exceptions.UserError: You cannot delete this partner because it is linked to existing Sales Orders or Invoices.\n"
}
```

---

## 8. Production Best Practices for E-Commerce

### 1. Unique External Identifier Mapping (`ref`)
Always populate the `ref` field with your e-commerce platform's primary customer ID (e.g. `ECOMM-USER-10294`). Use `ref` in your lookup domains to prevent duplicate contact creation if a customer changes their email address in your store.

### 2. Cache Geographical Records
Do not query `res.country` and `res.country.state` on every checkout. Fetch the full list once at backend startup or on a daily cron schedule, and cache the ISO-code to Odoo-ID mapping in memory or Redis.

### 3. Graceful Error Handling & Retries
- **`401 Unauthorized`**: Do not retry automatically; trigger an alert to renew the API key.
- **`429 Too Many Requests` / `503 Service Unavailable`**: Implement exponential backoff with jitter (e.g., retry after 1s, 2s, 4s, 8s).
- **`400 ValidationError` / `403 AccessError`**: Log the `message` and `debug` fields to your centralized telemetry (e.g. Sentry, Datadog) for immediate inspection.

### 4. Prefer Archiving over Hard Deleting
Never invoke `/json/2/res.partner/unlink` from automated e-commerce webhooks. If a user requests account deletion (GDPR/CCPA), update their record with anonymized data and set `"active": false`.

### 5. Single Atomic Request for Checkout
When a guest customer completes an order with distinct billing and shipping addresses, send one `create` call to `res.partner` using `child_ids` as demonstrated in [Step 5](#step-5-atomic-customer-creation-with-addresses-single-request). This ensures that if address creation fails, the parent contact is also rolled back, maintaining complete data integrity.
