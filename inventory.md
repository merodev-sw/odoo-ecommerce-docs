# Odoo JSON-2 API: Inventory & Stock Management Integration Guide

This guide provides an end-to-end technical reference for backend developers integrating an **e-commerce platform** with Odoo's **Inventory & Stock Management (`stock.quant`, `product.template`, `product.product`, `stock.picking`, `stock.location`)** modules using the **External JSON-2 API**.

> [!IMPORTANT]
> **No XML-RPC or Legacy JSON-RPC (`/jsonrpc`):** This guide strictly uses Odoo's **JSON-2 API** (`/json/2/` endpoints), which uses standard HTTP POST semantics, bearer token authentication with API keys, named parameter payloads, and standard HTTP status codes.

---

## Table of Contents

1. [Architectural Overview & Inventory Concepts](#1-architectural-overview--inventory-concepts)
2. [Authentication & Request Conventions](#2-authentication--request-conventions)
3. [Model & Field References](#3-model--field-references)
   - [3.1 Product Template (`product.template`)](#31-product-template-producttemplate)
   - [3.2 Product Variant (`product.product`)](#32-product-variant-productproduct)
   - [3.3 Product Images & Media Gallery (`product.image`)](#33-product-images--media-gallery-productimage)
   - [3.4 Stock Quant / Physical Stock Level (`stock.quant`)](#34-stock-quant--physical-stock-level-stockquant)
   - [3.5 Warehouses & Locations (`stock.warehouse` & `stock.location`)](#35-warehouses--locations-stockwarehouse--stocklocation)
   - [3.6 Stock Transfers & Receipts (`stock.picking` & `stock.move`)](#36-stock-transfers--receipts-stockpicking--stockmove)
   - [3.7 Lots & Serial Numbers (`stock.lot`)](#37-lots--serial-numbers-stocklot)
   - [3.8 Reordering Rules (`stock.warehouse.orderpoint`)](#38-reordering-rules-stockwarehouseorderpoint)
4. [Step-by-Step E-Commerce Integration Workflow](#4-step-by-step-e-commerce-integration-workflow)
   - [Step 1: Warehouse & Internal Location Lookup](#step-1-warehouse--internal-location-lookup)
   - [Step 2: Catalog Sync: Products, Variants & Multiple Images Gallery](#step-2-catalog-sync-products-variants--multiple-images-gallery)
   - [Step 3: Real-Time Stock Availability Queries (Preventing Overselling)](#step-3-real-time-stock-availability-queries-preventing-overselling)
   - [Step 4: Real-Time Inventory Adjustments (Syncing Physical Stock Counts)](#step-4-real-time-inventory-adjustments-syncing-physical-stock-counts)
   - [Step 5: Processing Inbound Goods Receipts (Supplier Restock)](#step-5-processing-inbound-goods-receipts-supplier-restock)
   - [Step 6: Processing Internal Warehouse Transfers](#step-6-processing-internal-warehouse-transfers)
   - [Step 7: Managing Lots & Serial Numbers](#step-7-managing-lots--serial-numbers)
   - [Step 8: Automated Replenishment & Reordering Rules](#step-8-automated-replenishment--reordering-rules)
5. [Complete HTTP Request & Response Reference](#5-complete-http-request--response-reference)
6. [Error Handling & Failure Responses](#6-error-handling--failure-responses)
7. [Production Best Practices for E-Commerce Developers](#7-production-best-practices-for-e-commerce-developers)

---

## 1. Architectural Overview & Inventory Concepts

Understanding how Odoo tracks physical stock versus logical availability is critical for an e-commerce integration.

### Product Types (Odoo 19)
- **Storable Product (`is_storable: true`, `type: 'consu'`):** Physical goods tracked in inventory. Stock levels (`qty_available`, `free_qty`) are calculated and stock moves are generated. **(Use this for all physical e-commerce items)**.
- **Consumable Product (`is_storable: false`, `type: 'consu'`):** Physical goods where stock levels are not tracked.
- **Service (`is_storable: false`, `type: 'service'`):** Non-physical items (e.g. Installation, Extended Warranty, Gift Wrapping).
- **Combo Product (`type: 'combo'`):** Product bundles/kits.

> [!NOTE]
> **Odoo 19 Architecture Note:** In Odoo 18 and Odoo 19, the legacy `detailed_type` field was deprecated/removed. Inventory tracking is now controlled by the boolean flag **`is_storable: true`** alongside **`type: 'consu'`**.

### Key Stock Metrics for E-Commerce

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  Quantity on Hand (qty_available) = Total physical units in warehouse       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Minus Reserved (Unshipped Sales Orders)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Free to Use (free_qty) = Quantity Available for New E-Commerce Orders      │
│  ⭐ ALWAYS DISPLAY free_qty IN YOUR STORE TO PREVENT OVERSELLING ⭐           │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ Plus Incoming Shipments (Purchase Orders)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Forecasted Stock (virtual_available) = Projected future inventory balance │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Metric | Field Name | Description | E-Commerce Usage |
| :--- | :--- | :--- | :--- |
| **Free to Use** | `free_qty` | `qty_available` minus `reserved_quantity`. | **Primary stock indicator for storefront.** Display as "In Stock (X left)". |
| **Quantity on Hand**| `qty_available` | Actual physical units residing in warehouse internal locations. | Used for physical inventory reconciliation and warehouse audits. |
| **Forecasted Quantity**| `virtual_available`| `qty_available` - `outgoing_qty` + `incoming_qty`. | Used for pre-orders or backorder availability estimations. |
| **Incoming Quantity**| `incoming_qty` | Quantities on confirmed vendor purchase orders or inbound receipts. | Show "Restocking Soon" estimates. |
| **Outgoing Quantity**| `outgoing_qty` | Quantities on confirmed customer sales orders awaiting fulfillment. | Reserved stock pending warehouse packing. |

---

## 2. Authentication & Request Conventions

Every API operation is an **HTTP `POST`** to `/json/2/<model>/<method>`.

### Required Headers
```http
Host: mycompany.odoo.com
Authorization: bearer <YOUR_160_BIT_API_KEY>
Content-Type: application/json; charset=utf-8
X-Odoo-Database: mycompany_db
User-Agent: MyStore-Inventory-Sync/1.0
```

---

## 3. Model & Field References

### 3.1 Product Template (`product.template`)

The base product definition. A template can represent a single standalone product or a parent with multiple variant combinations (e.g. Shirt in Red/Blue, S/M/L).

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | — | Title/Name of the product (e.g., `Pro Gaming Mechanical Keyboard`). |
| `is_storable` | `Boolean` | Optional | `false` | **Set to `true` for physical storable products** tracked with stock moves and quants. |
| `type` | `Selection` | **REQUIRED** | `'consu'` | Product classification: `'consu'` (Goods/Materials), `'service'` (Service), `'combo'` (Combo Kit). |
| `categ_id` | `Many2one` (`product.category`) | **REQUIRED** | Company Default | Product category ID (e.g., Electronics, Apparel). |
| `list_price` | `Float` | Optional | `0.0` | Base selling price before customer-specific pricelists. |
| `standard_price`| `Float` | Optional | `0.0` | Cost price (used for margin and inventory valuation calculations). |
| `default_code` | `Char` | Optional | `null` | Primary SKU / Internal Reference. |
| `barcode` | `Char` | Optional | `null` | Barcode (EAN13, UPC, Code128). |
| `image_1920` | `Binary` | Optional | `null` | Primary product image (Base64-encoded binary string up to 1920x1920). Auto-generates `image_1024`, `image_512`, `image_256`, `image_128`. |
| `product_template_image_ids` | `One2many` (`product.image`) | Optional | `[]` | Extra media gallery images for this product template. |
| `weight` | `Float` | Optional | `0.0` | Product weight (in kg). Used for shipping rate calculation. |
| `volume` | `Float` | Optional | `0.0` | Product volume (in m³). |
| `sale_ok` | `Boolean` | Optional | `true` | Set to `true` to allow selling via sales orders and e-commerce. |
| `purchase_ok`| `Boolean` | Optional | `true` | Set to `true` to allow purchasing via vendor purchase orders. |
| `taxes_id` | `Many2many` (`account.tax`) | Optional | Default Sale Tax | Customer sales tax IDs (e.g., `[[6, 0, [tax_id]]]`). |
| `tracking` | `Selection` | Optional | `'none'` | Traceability mode: `'none'`, `'serial'` (Unique Serial No. per unit), `'lot'` (Batch / Lot No.). |
| `active` | `Boolean` | Optional | `true` | Active status. Set to `false` to archive discontinued products. |

---

### 3.2 Product Variant (`product.product`)

The specific sellable SKU (e.g. Size M, Color Black). If a product template has no variants, Odoo automatically creates a 1-to-1 `product.product` variant record.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `product_tmpl_id` | `Many2one` (`product.template`) | **REQUIRED** | — | Parent product template ID. |
| `default_code` | `Char` | Optional | Template SKU | Variant-specific SKU (e.g., `KEYBOARD-PRO-RGB-RED`). |
| `barcode` | `Char` | Optional | Template Barcode | Variant-specific barcode (e.g., `012345678905`). |
| `image_variant_1920` / `image_1920` | `Binary` | Optional | Template Image | Variant-specific primary image (Base64 string). Falls back to template `image_1920` if unset. |
| `product_variant_image_ids` | `One2many` (`product.image`) | Optional | `[]` | Variant-specific extra gallery images. |
| `weight` | `Float` | Optional | Template Weight | Variant-specific weight. |
| `lst_price` | `Float` | *Computed* | Template list_price | Calculated variant selling price. |
| `product_template_attribute_value_ids` | `Many2many` | Optional | `[]` | Linked attribute values defining this variant (e.g., Red, Medium). |
| `qty_available` | `Float` | *Computed* | `0.0` | Quantity physically on hand in warehouse internal locations. |
| `free_qty` | `Float` | *Computed* | `0.0` | **Quantity available to promise (Free Stock).** |
| `virtual_available` | `Float` | *Computed* | `0.0` | Forecasted quantity including incoming/outgoing moves. |
| `incoming_qty` | `Float` | *Computed* | `0.0` | Incoming units on open receipts/POs. |
| `outgoing_qty` | `Float` | *Computed* | `0.0` | Outgoing units reserved on confirmed sales orders. |

---

### 3.3 Product Images & Media Gallery (`product.image`)

The `product.image` model manages multiple extra images and media gallery assets associated with a product template or specific variant.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | — | Label / Title for the image (e.g., `Side Angle View`, `Ports & Connectivity`, `In-Box Contents`). |
| `image_1920` | `Binary` | **REQUIRED** | — | Base64-encoded raw binary image data (PNG, JPG, WebP). Maximum resolution 1920x1920 px. |
| `product_tmpl_id` | `Many2one` (`product.template`) | **REQUIRED** | — | Parent product template ID this image belongs to. |
| `product_variant_id` | `Many2one` (`product.product`) | Optional | `null` | Optional specific variant ID (e.g. show this image only when "Midnight Blue" variant is selected). |
| `sequence` | `Integer` | Optional | `10` | Display sorting sequence in gallery (lower number appears first). |
| `video_url` | `Char` | Optional | `null` | Optional external video URL (YouTube, Vimeo, Dailymotion) to embed in media carousel. |
| `image_1024` | `Binary` | *Computed* | — | Automatically resized 1024px thumbnail (ideal for product detail page display). |
| `image_512` | `Binary` | *Computed* | — | Automatically resized 512px thumbnail (ideal for product cards). |
| `image_256` | `Binary` | *Computed* | — | Automatically resized 256px thumbnail. |
| `image_128` | `Binary` | *Computed* | — | Automatically resized 128px thumbnail (ideal for cart & order lines). |

---

### 3.4 Stock Quant / Physical Stock Level (`stock.quant`)

Stores the physical quantity of a specific product variant at a specific warehouse location.

| Field Name | Type | Requirement | Default | Description & E-Commerce Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** | — | Product variant ID. |
| `location_id` | `Many2one` (`stock.location`) | **REQUIRED** | — | Internal warehouse location ID (e.g., `WH/Stock`). |
| `quantity` | `Float` | *System/Computed* | `0.0` | Current physical quantity stored in this location. |
| `reserved_quantity` | `Float` | *System/Computed* | `0.0` | Quantity reserved for open pickings. |
| `inventory_quantity` | `Float` | Optional | `0.0` | **Counted quantity during inventory adjustment.** |
| `inventory_diff_quantity` | `Float` | *Computed* | `0.0` | Difference between `inventory_quantity` and `quantity`. |
| `lot_id` | `Many2one` (`stock.lot`) | Optional | `null` | Lot/Serial number ID if tracked. |
| `package_id` | `Many2one` (`stock.quant.package`) | Optional | `null` | Physical pallet/box/package ID. |

---

### 3.5 Warehouses & Locations (`stock.warehouse` & `stock.location`)

#### `stock.warehouse`
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | Warehouse display name (e.g., `Main Fulfillment Center`). |
| `code` | `Char` | **REQUIRED** | Short code (e.g., `WH`, `NY`, `LA`). Max 5 characters. |
| `lot_stock_id` | `Many2one` (`stock.location`) | *System-managed* | Default internal storage location (e.g., `WH/Stock`). |

#### `stock.location`
| Field Name | Type | Requirement | Description |
| :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | Location name (e.g., `Stock`, `Shelf A1`). |
| `complete_name` | `Char` | *Computed* | Full hierarchical path (e.g., `WH/Stock/Aisle 1`). |
| `usage` | `Selection` | **REQUIRED** | Location type: `'internal'` (Storage in warehouse), `'customer'` (Customer location), `'supplier'` (Vendor location), `'inventory'` (Virtual loss/gain), `'transit'` (Inter-warehouse transit). |

---

### 3.6 Stock Transfers & Receipts (`stock.picking` & `stock.move`)

Represents all physical movements of inventory: Inbound Receipts, Internal Transfers, and Outgoing Deliveries.

#### `stock.picking`
| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `picking_type_id` | `Many2one` (`stock.picking.type`) | **REQUIRED** | — | Operation Type ID (e.g., `Receipts`, `Internal Transfers`, `Delivery Orders`). |
| `location_id` | `Many2one` (`stock.location`) | **REQUIRED** | Source | Source location (e.g., `Partner Locations/Vendors` for receipts; `WH/Stock` for transfers). |
| `location_dest_id` | `Many2one` (`stock.location`) | **REQUIRED** | Destination | Destination location (e.g., `WH/Stock` for receipts; `WH2/Stock` for transfers). |
| `origin` | `Char` | Optional | `null` | Reference to source document (e.g. PO number `PO00129` or Transfer Ref). |
| `partner_id` | `Many2one` (`res.partner`) | Optional | `null` | Associated vendor or customer contact. |
| `state` | `Selection` | *System-managed* | `'draft'` | Status: `'draft'`, `'waiting'`, `'confirmed'`, `'assigned'` (Ready), `'done'` (Completed), `'cancel'`. |
| `move_ids` | `One2many` (`stock.move`) | Optional | `[]` | Inventory movement lines. |

#### `stock.move`
| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | Product name | Move description. |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** | — | Product variant ID. |
| `product_uom_qty` | `Float` | **REQUIRED** | `1.0` | Demand quantity to move. |
| `quantity` | `Float` | Optional | `0.0` | Actual quantity processed / done. |
| `product_uom` | `Many2one` (`uom.uom`) | **REQUIRED** | Product UoM | Unit of Measure ID. |
| `location_id` | `Many2one` (`stock.location`) | **REQUIRED** | Picking Source | Source location. |
| `location_dest_id` | `Many2one` (`stock.location`) | **REQUIRED** | Picking Dest | Destination location. |

---

### 3.7 Lots & Serial Numbers (`stock.lot`)

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `name` | `Char` | **REQUIRED** | — | Lot or Serial Number string (e.g., `SN-202608-88129`). |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** | — | Product variant ID being tracked. |
| `ref` | `Char` | Optional | `null` | Internal reference or vendor lot code. |
| `company_id` | `Many2one` (`res.company`) | Optional | Current Company | Owning company ID. |

---

### 3.8 Reordering Rules (`stock.warehouse.orderpoint`)

Automates stock replenishment when available quantities drop below safety thresholds.

| Field Name | Type | Requirement | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `product_id` | `Many2one` (`product.product`) | **REQUIRED** | — | Product variant ID. |
| `location_id` | `Many2one` (`stock.location`) | **REQUIRED** | WH/Stock | Warehouse location to monitor. |
| `product_min_qty` | `Float` | **REQUIRED** | `0.0` | Minimum threshold. When available stock drops below this, replenishment triggers. |
| `product_max_qty` | `Float` | **REQUIRED** | `0.0` | Maximum target stock level after replenishment. |
| `qty_multiple` | `Float` | Optional | `1.0` | Batch multiple (e.g. order in packs of 10). |

---

## 4. Step-by-Step E-Commerce Integration Workflow

---

### Step 1: Warehouse & Internal Location Lookup

Before managing stock, query Odoo to find your primary warehouse ID and internal storage locations (`usage = 'internal'`).

```http
POST /json/2/stock.location/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["usage", "=", "internal"]
  ],
  "fields": ["id", "name", "complete_name", "usage"],
  "limit": 10
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 8,
    "name": "Stock",
    "complete_name": "WH/Stock",
    "usage": "internal"
  },
  {
    "id": 14,
    "name": "Shelf 1",
    "complete_name": "WH/Stock/Shelf 1",
    "usage": "internal"
  }
]
```

---

### Step 2: Catalog Sync: Products, Variants & Multiple Images Gallery

#### 2.1 Create a Storable Product Template with Primary Image
Create the parent product record and assign its main hero image directly via base64 encoded binary string in `image_1920`:

```http
POST /json/2/product.template/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Ultra-Slim Wireless Mouse",
      "is_storable": true,
      "type": "consu",
      "categ_id": 1,
      "list_price": 49.99,
      "standard_price": 18.50,
      "default_code": "SKU-MOUSE-SLIM-01",
      "barcode": "079357318921",
      "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==",
      "weight": 0.12,
      "sale_ok": true,
      "purchase_ok": true,
      "tracking": "none"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[620]
```
*(Returns newly created product template ID `620`)*

---

#### 2.2 Create a Product with Multiple Extra Gallery Images (One-Shot Embedded Creation)
To upload both the product and its entire gallery of extra media in a single atomic API call, use Odoo's One2many ORM command tuples `[0, 0, {...}]` inside `product_template_image_ids`:

```http
POST /json/2/product.template/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Mechanical RGB Gaming Keyboard",
      "is_storable": true,
      "type": "consu",
      "categ_id": 1,
      "list_price": 129.99,
      "default_code": "SKU-KB-RGB-01",
      "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==",
      "product_template_image_ids": [
        [
          0,
          0,
          {
            "name": "Side Angle Ergonomic Profile",
            "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg==",
            "sequence": 1
          }
        ],
        [
          0,
          0,
          {
            "name": "RGB Backlight Modes in the Dark",
            "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkWPifDwAEeAGV7R8j4QAAAABJRU5ErkJggg==",
            "sequence": 2
          }
        ],
        [
          0,
          0,
          {
            "name": "Keycap Puller and Packaging Box",
            "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mOU/88AAAYEAd8mXb+KAAAAAElFTkSuQmCC",
            "sequence": 3
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[621]
```

---

#### 2.3 Add Extra Images to an Existing Product (`product.image/create`)
If the product template already exists, add additional gallery images in bulk by passing multiple records in `vals_list` on the `product.image` endpoint:

```http
POST /json/2/product.image/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Top View & Sensor Close-up",
      "product_tmpl_id": 620,
      "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg==",
      "sequence": 1
    },
    {
      "name": "Charging Cable & Dongle",
      "product_tmpl_id": 620,
      "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkWPifDwAEeAGV7R8j4QAAAABJRU5ErkJggg==",
      "sequence": 2
    },
    {
      "name": "Official Product Feature Video",
      "product_tmpl_id": 620,
      "video_url": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
      "sequence": 3
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[45, 46, 47]
```
*(Returns IDs of the newly created `product.image` records: `45`, `46`, `47`)*

---

#### 2.4 Attach Variant-Specific Extra Images (`product_variant_id`)
When selling variant products (e.g., a mouse in Midnight Black vs. Pearl White), you can restrict extra gallery images to display **only when that specific variant is selected** by linking `product_variant_id`:

```http
POST /json/2/product.image/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "Pearl White Finish Close-up",
      "product_tmpl_id": 620,
      "product_variant_id": 1056,
      "image_1920": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mOU/88AAAYEAd8mXb+KAAAAAElFTkSuQmCC",
      "sequence": 1
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[48]
```

---

#### 2.5 Reorder & Update Gallery Images (`product.image/write`)
Change the display order or update metadata of existing gallery items:

```http
POST /json/2/product.image/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [45],
  "vals": {
    "sequence": 10,
    "name": "Updated Top View (High-Res)"
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

---

#### 2.6 Query & Retrieve Product Gallery Images (`product.image/search_read`)
Query extra images linked to a product template or variant without requesting large binary fields in bulk:

```http
POST /json/2/product.image/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["product_tmpl_id", "=", 620]
  ],
  "fields": [
    "id",
    "name",
    "sequence",
    "product_variant_id",
    "video_url",
    "write_date"
  ],
  "order": "sequence asc, id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 45,
    "name": "Updated Top View (High-Res)",
    "sequence": 1,
    "product_variant_id": false,
    "video_url": false,
    "write_date": "2026-08-23 10:15:30"
  },
  {
    "id": 46,
    "name": "Charging Cable & Dongle",
    "sequence": 2,
    "product_variant_id": false,
    "video_url": false,
    "write_date": "2026-08-23 09:40:12"
  },
  {
    "id": 48,
    "name": "Pearl White Finish Close-up",
    "sequence": 1,
    "product_variant_id": [1056, "Ultra-Slim Wireless Mouse (Pearl White)"],
    "video_url": false,
    "write_date": "2026-08-23 10:20:00"
  }
]
```

> [!TIP]
> **High-Performance Image Serving:** Do not pull `image_1920` base64 strings in API searches. Instead, construct direct, cacheable web URLs pointing to Odoo's image route using the record ID and timestamp for cache busting:
> `https://mycompany.odoo.com/web/image/product.image/45/image_512?unique=20260823101530`

---

#### 2.7 Delete / Remove an Image (`product.image/unlink`)
Remove an image from the gallery permanently:

```http
POST /json/2/product.image/unlink HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [47]
}
```

**Success Response (`200 OK`):**
```json
true
```

---

#### 2.8 Retrieve the Generated `product.product` Variant ID
When a template is created, Odoo automatically creates the corresponding variant. Find its ID:

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["product_tmpl_id", "=", 620]
  ],
  "fields": ["id", "default_code", "name", "qty_available", "free_qty"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1055,
    "default_code": "SKU-MOUSE-SLIM-01",
    "name": "Ultra-Slim Wireless Mouse",
    "qty_available": 0.0,
    "free_qty": 0.0
  }
]
```

---

### Step 3: Real-Time Stock Availability Queries (Preventing Overselling)

#### 3.1 Query Real-Time Free Stock by SKU List
Use this call during e-commerce cart/checkout validation to confirm that products have not been reserved by other shoppers.

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["default_code", "in", ["SKU-MOUSE-SLIM-01", "SKU-HEADPHONES-PRO"]]
  ],
  "fields": [
    "id",
    "default_code",
    "name",
    "qty_available",
    "free_qty",
    "virtual_available",
    "outgoing_qty"
  ]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1055,
    "default_code": "SKU-MOUSE-SLIM-01",
    "name": "Ultra-Slim Wireless Mouse",
    "qty_available": 100.0,
    "free_qty": 85.0,
    "virtual_available": 85.0,
    "outgoing_qty": 15.0
  },
  {
    "id": 1042,
    "default_code": "SKU-HEADPHONES-PRO",
    "name": "Wireless Noise Cancelling Headphones",
    "qty_available": 12.0,
    "free_qty": 2.0,
    "virtual_available": 2.0,
    "outgoing_qty": 10.0
  }
]
```
*(In this example, only **85** units of the mouse and **2** units of the headphones are free to sell).*

#### 3.2 Filter Stock by Specific Warehouse or Location
Pass the `warehouse` or `location` integer ID in the `context` parameter to query warehouse-specific inventory:

```http
POST /json/2/product.product/read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [1055],
  "fields": ["qty_available", "free_qty"],
  "context": {
    "warehouse": 1
  }
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1055,
    "qty_available": 50.0,
    "free_qty": 45.0
  }
]
```

---

### Step 4: Real-Time Inventory Adjustments (Syncing Physical Stock Counts)

When your warehouse system, 3PL, or manual audit provides a new absolute physical stock count, update Odoo via `stock.quant` and apply the adjustment.

#### Method: Create/Write to `stock.quant` and Apply (`action_apply_inventory`)

##### Part A: Record the Counted Quantity
```http
POST /json/2/stock.quant/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "product_id": 1055,
      "location_id": 8,
      "inventory_quantity": 150.0
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[492]
```
*(Returns Quant ID `492`)*

##### Part B: Apply the Inventory Count
Calling `action_apply_inventory` instructs Odoo to generate the internal adjustment move and update the on-hand physical stock.

```http
POST /json/2/stock.quant/action_apply_inventory HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [492]
}
```

**Success Response (`200 OK`):**
```json
true
```

---

### Step 5: Processing Inbound Goods Receipts (Supplier Restock)

When receiving an inbound shipment from a vendor or manufacturer into the warehouse:

#### 5.1 Find Inbound Operation Type (`Receipts`) and Vendor Location
```http
POST /json/2/stock.picking.type/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["code", "=", "incoming"]
  ],
  "fields": ["id", "name", "default_location_src_id", "default_location_dest_id"],
  "limit": 1
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "name": "Receipts",
    "default_location_src_id": [4, "Partner Locations/Vendors"],
    "default_location_dest_id": [8, "WH/Stock"]
  }
]
```

#### 5.2 Create the Inbound Receipt (`stock.picking`)
```http
POST /json/2/stock.picking/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "picking_type_id": 1,
      "location_id": 4,
      "location_dest_id": 8,
      "origin": "PO-SUPPLIER-RESTOCK-8821",
      "move_ids": [
        [
          0,
          0,
          {
            "name": "Restock: Ultra-Slim Wireless Mouse",
            "product_id": 1055,
            "product_uom_qty": 200.0,
            "product_uom": 1,
            "location_id": 4,
            "location_dest_id": 8
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[512]
```

#### 5.3 Set Received Quantities and Validate (`button_validate`)
Once goods arrive at the dock and are verified:

##### Set Received Quantity on Stock Move:
```http
POST /json/2/stock.move/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [["raw_material_production_id", "=", false], ["picking_id", "=", 512]],
  "fields": ["id", "product_id", "product_uom_qty"]
}
```

```http
POST /json/2/stock.move/write HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [1120],
  "vals": {
    "quantity": 200.0
  }
}
```

**Success Response (`200 OK`):**
```json
true
```

##### Validate the Receipt:
```http
POST /json/2/stock.picking/button_validate HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "ids": [512]
}
```

**Success Response (`200 OK`):**
```json
true
```
*(Stock in `WH/Stock` immediately increases by 200 units).*

---

### Step 6: Processing Internal Warehouse Transfers

Move inventory from storage bins to storefront packing areas or between warehouses.

```http
POST /json/2/stock.picking/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "picking_type_id": 5,
      "location_id": 8,
      "location_dest_id": 14,
      "origin": "Replenish Storefront Shelf 1",
      "move_ids": [
        [
          0,
          0,
          {
            "name": "Move Mouse to Shelf 1",
            "product_id": 1055,
            "product_uom_qty": 30.0,
            "product_uom": 1,
            "location_id": 8,
            "location_dest_id": 14
          }
        ]
      ]
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[513]
```

Validate the transfer via `POST /json/2/stock.picking/button_validate` with `"ids": [513]`.

---

### Step 7: Managing Lots & Serial Numbers

For products with high value or expiration dates, manage serial numbers (`tracking = 'serial'`) or lots (`tracking = 'lot'`).

#### 7.1 Create a Serial Number / Lot
```http
POST /json/2/stock.lot/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "name": "SN-2026-KEYB-00491",
      "product_id": 1042,
      "ref": "Batch Q3-2026"
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[88]
```

#### 7.2 Search Product Locations by Serial Number
```http
POST /json/2/stock.quant/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["lot_id", "=", 88]
  ],
  "fields": ["product_id", "location_id", "quantity"]
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 499,
    "product_id": [1042, "Wireless Noise Cancelling Headphones"],
    "location_id": [8, "WH/Stock"],
    "quantity": 1.0
  }
]
```

---

### Step 8: Automated Replenishment & Reordering Rules

Configure minimum stock thresholds so Odoo automatically creates purchase proposals when free stock is depleted.

```http
POST /json/2/stock.warehouse.orderpoint/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "product_id": 1055,
      "location_id": 8,
      "product_min_qty": 20.0,
      "product_max_qty": 100.0,
      "qty_multiple": 10.0
    }
  ]
}
```

**Success Response (`200 OK`):**
```json
[34]
```

---

## 5. Complete HTTP Request & Response Reference

### Bulk Catalog Stock Export for E-Commerce Feed
Used to synchronize your store's inventory feed periodically.

```http
POST /json/2/product.product/search_read HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "domain": [
    ["is_storable", "=", true],
    ["sale_ok", "=", true]
  ],
  "fields": [
    "id",
    "default_code",
    "barcode",
    "name",
    "lst_price",
    "free_qty",
    "qty_available"
  ],
  "offset": 0,
  "limit": 100,
  "order": "id asc"
}
```

**Success Response (`200 OK`):**
```json
[
  {
    "id": 1042,
    "default_code": "SKU-HEADPHONES-PRO",
    "barcode": "012345678901",
    "name": "Wireless Noise Cancelling Headphones",
    "lst_price": 299.99,
    "free_qty": 14.0,
    "qty_available": 20.0
  },
  {
    "id": 1055,
    "default_code": "SKU-MOUSE-SLIM-01",
    "barcode": "079357318921",
    "name": "Ultra-Slim Wireless Mouse",
    "lst_price": 49.99,
    "free_qty": 150.0,
    "qty_available": 150.0
  }
]
```

---

## 6. Error Handling & Failure Responses

All error responses adhere to standard HTTP status codes and provide structured JSON error details.

### 6.1 Attempting Inventory Adjustment on Non-Storable Product (`400 Bad Request`)
Occurs when trying to set `inventory_quantity` on a service or consumable item.

#### Request
```http
POST /json/2/stock.quant/create HTTP/1.1
Host: mycompany.odoo.com
Authorization: bearer 6578616d706c65206a736f6e20617069206b6579
Content-Type: application/json; charset=utf-8

{
  "vals_list": [
    {
      "product_id": 901,
      "location_id": 8,
      "inventory_quantity": 50.0
    }
  ]
}
```

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.UserError",
  "message": "You cannot perform an inventory adjustment on a product that is not storable (Type: service).",
  "arguments": ["You cannot perform an inventory adjustment on a product that is not storable (Type: service)."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/stock/models/stock_quant.py\", line 210, in create\n    raise UserError(_('You cannot perform an inventory adjustment...'))\nodoo.exceptions.UserError: You cannot perform an inventory adjustment on a product that is not storable.\n"
}
```

---

### 6.2 Duplicate Barcode Constraint (`400 Bad Request` or `422 Unprocessable Content`)
Occurs if a barcode is already assigned to another product in the database.

#### Error Response (`400 Bad Request`)
```json
{
  "name": "odoo.exceptions.ValidationError",
  "message": "A barcode can only be assigned to one product! The barcode '079357318921' is already assigned to 'Ultra-Slim Wireless Mouse'.",
  "arguments": [
    "A barcode can only be assigned to one product! The barcode '079357318921' is already assigned to 'Ultra-Slim Wireless Mouse'."
  ],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/addons/product/models/product_product.py\", line 112, in _check_barcode_unicity\n    raise ValidationError(...)\nodoo.exceptions.ValidationError: A barcode can only be assigned to one product!\n"
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

### 6.4 Insufficient Permissions on Stock Management (`403 Forbidden`)

#### Error Response (`403 Forbidden`)
```json
{
  "name": "odoo.exceptions.AccessError",
  "message": "You are not allowed to modify 'Quants' (stock.quant) records. Contact your system administrator.",
  "arguments": ["You are not allowed to modify 'Quants' (stock.quant) records."],
  "context": {},
  "debug": "Traceback (most recent call last):\n  File \"/opt/odoo/odoo/addons/base/models/ir_rule.py\", line 310, in check\n    raise AccessError(...)\nodoo.exceptions.AccessError: You are not allowed to modify 'Quants' (stock.quant) records.\n"
}
```

---

## 7. Production Best Practices for E-Commerce Developers

### 1. Always Read `free_qty`, Not `qty_available`
When checking whether an item is in stock on your storefront or allowing an order to be placed, **always use `free_qty`**. `qty_available` includes physical units in the warehouse that are already reserved for other confirmed orders waiting to be packed. Relying on `qty_available` will lead to severe overselling.

### 2. High-Speed Incremental Stock Sync
Instead of pulling the entire product catalog every 5 minutes:
- Store a `last_synced_timestamp`.
- Query modified stock quants using:
  ```json
  "domain": [["write_date", ">", "2026-08-16 14:00:00"]]
  ```
- Fetch `free_qty` only for products whose stock quants changed.

### 3. Inventory Buffer / Safety Thresholds
In high-velocity flash sales or multi-channel retail (e.g., selling simultaneously on Shopify, Amazon, and retail POS):
- Configure a safety buffer in your e-commerce backend:
  $$\text{Storefront Available Stock} = \max(0, \text{Odoo } \texttt{free\_qty} - \text{Safety Buffer})$$
- For example, if safety buffer = 2 and `free_qty` = 2, display "Out of Stock" to protect against race conditions.

### 4. Batch Inventory Adjustments
When synchronizing end-of-day stock counts from an external Warehouse Management System (WMS), send all quants in a single `vals_list` inside `stock.quant/create`, followed by a single call to `stock.quant/action_apply_inventory` with all generated IDs.

### 5. Multi-Warehouse Routing
If your e-commerce store fulfills orders from multiple regional warehouses (e.g. US-East, US-West, EU):
- Pass `"context": {"warehouse": warehouse_id}` when reading `free_qty` to get accurate regional stock for geo-routed order allocation.

### 6. High-Performance Product Image Handling & CDN Caching
- **Avoid Base64 in Search Payloads:** Never request `image_1920` or other binary fields in catalog listing search queries. Exclude binary fields from `fields` to prevent megabytes of base64 transfer over HTTP.
- **Direct Cacheable Image URLs:** Construct direct image URLs for storefront display:
  - Product Template Primary: `https://mycompany.odoo.com/web/image/product.template/<template_id>/image_512`
  - Product Variant Primary: `https://mycompany.odoo.com/web/image/product.product/<variant_id>/image_512`
  - Extra Media Gallery Image: `https://mycompany.odoo.com/web/image/product.image/<image_id>/image_1024`
- **Cache Invalidation via `write_date`:** Append the Unix timestamp or formatted `write_date` query parameter (e.g. `?unique=1755950400`) to let CDNs and browsers aggressively cache images until modified in Odoo.
- **Appropriate Thumbnail Sizing:** Use `image_128` for search dropdowns and order lines, `image_512` for catalog grids and category listings, and `image_1024` or `image_1920` for product zoom views.

