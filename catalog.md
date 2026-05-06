# Product Catalog Service

## What It Is

Single source of truth for **all product information**. Every service that needs to know what a product is — asks here.

```
Storefront   → product name, images, description, variants
Search       → everything to index
OMS          → is this SKU valid and sellable?
Pricing      → base price per SKU
WMS          → weight, dimensions, shelf life
POS          → barcode → SKU mapping
```

---

## Two-Level Product Structure

### Parent Product — Display Concept Only

Parent product exists for one reason: **customer experience on Storefront**.

```
Customer searches "milo" → sees one result: "Milo"
Clicks → sees all variants as size selector: [180ml] [400ml] [1L]
Selects 180ml → adds SKU-001 to cart
```

Parent product never appears in OMS, Stock, WMS, or Logistics. Only Storefront and Search use it.

### SKU — Operational Identity

Every operational system works at SKU level:

```
OMS reserves:    SKU-001 (180ml) — never "Milo"
Stock tracks:    SKU-001 qty, SKU-002 qty separately
WMS picks:       SKU-001 from bin B3
Pricing charges: SKU-001 price = 35 THB
```

### The Boundary

```
Customer-facing:   parent product (display grouping)
Everything else:   SKU (actual product identity)
```

---

## Data Structure

### Parent Product

```json
{
  "id":          "PROD-001",
  "name":        "Milo",
  "brand":       "Nestle",
  "description": "chocolate malt drink",
  "category":    "beverages",
  "subcategory": "malt drinks",
  "images":      ["url1", "url2"],
  "attributes":  { "flavor": "chocolate", "type": "powder" },
  "status":      "active",
  "supplier_id": "SUP-001",
  "created_at":  "2024-01-01",
  "updated_at":  "2024-05-01"
}
```

### SKU (Variant)

```json
{
  "sku_id":        "SKU-001",
  "parent_id":     "PROD-001",
  "name":          "Milo 180ml can",
  "barcode":       "8850006100015",
  "variant_attrs": { "size": "180ml", "packaging": "can" },

  "pricing": {
    "base_price":      35.00,
    "wholesale_price": 20.00
  },

  "logistics": {
    "weight_g":        250,
    "width_cm":        6,
    "height_cm":       10,
    "depth_cm":        6,
    "shelf_life_days": 365
  },

  "channels": {
    "online":  true,
    "instore": true
  },

  "uom": {
    "base_unit": "can",
    "pack":      6,
    "carton":    144,
    "pallet":    6912
  },

  "status":     "active",
  "created_at": "2024-01-01"
}
```

---

## Product Status Lifecycle

```
DRAFT
  │ supplier submits for review
  ▼
PENDING_REVIEW
  │ category manager approves
  ▼
ACTIVE              ← visible on Storefront, searchable
  │
  ├──► SUSPENDED    ← temporarily hidden (quality issue, legal hold)
  │         │
  │         └──► ACTIVE (reinstated)
  │
  └──► DELISTED     ← permanently removed, historical orders still valid
```

Only ACTIVE products appear on Storefront and in Search index.

---

## Who Reads vs Writes

```
Writes:
  Seller Portal (Procurement) → create/edit product info
  Seller Portal (VMI)         → create/edit, suggest price
  Admin Portal                → approve, suspend, delist, set retail price
  WMS                         → update logistics dimensions if changed

Reads:
  Storefront     → display product pages and variants
  Search Indexer → index products into Elasticsearch
  OMS            → validate SKU on order placement
  Pricing Engine → read base price per SKU
  WMS            → read dimensions and shelf life for packing
  POS            → barcode → SKU lookup
```

---

## How Search Stays in Sync

Catalog never calls Search directly. Event-driven:

```
Product created / updated / status changed
  → Catalog publishes: "catalog.product.updated" {product_id}
  → Search Indexer listens
  → re-indexes affected parent product in Elasticsearch

Product delisted
  → Catalog publishes: "catalog.product.delisted" {product_id}
  → Search Indexer removes document from index
```

Small delay between Catalog update and Search reflecting it — acceptable.

---

## Elasticsearch Document — Built From Catalog

Search Indexer reads Catalog and builds the search document at parent level:

```json
{
  "id":          "PROD-001",
  "name":        "Milo",
  "brand":       "Nestle",
  "description": "chocolate malt drink",
  "category":    "beverages",
  "tags":        ["chocolate", "malt", "drink"],
  "variants": [
    { "sku": "SKU-001", "size": "180ml", "price": 35, "status": "active" },
    { "sku": "SKU-002", "size": "400ml", "price": 65, "status": "active" },
    { "sku": "SKU-003", "size": "1L",    "price": 120, "status": "active" }
  ],
  "min_price": 35,
  "has_stock": true
}
```

One document per parent product. Customer searches at this level, selects variant on product page.

---

## ATP in Search Results

ATP lives in Stock Service — not Catalog. But Search needs availability per variant.

### Option A — Stock Service pushes ATP to Search index
```
Stock changes → Stock Service publishes event
Search Indexer updates per-variant ATP in Elasticsearch
```
Fast display, eventually consistent.

### Option B — Storefront enriches after search
```
Search returns product list (no ATP)
Storefront calls Stock Service for ATP of returned SKUs
Merges before displaying
```
Always accurate, slightly slower.

### Recommended: Use Both

```
Product list page:   Option A — approximate ATP from Search index
                     fast, good enough for browsing

Product detail page: Option B — live ATP from Stock Service
                     accurate before customer adds to cart
```

---

## Barcode Mapping

All barcodes — including markdown override labels — map to a SKU:

```
BarcodeMap:
  barcode:      "8850006100015"  → sku_id: "SKU-001"  (original)
  barcode:      "9999999000123"  → sku_id: "SKU-001"  (markdown label)
                                   override_id: "PO-001"
```

POS and WMS look up barcode → get SKU + any active price override.

---

## API Endpoints

### Internal (exact lookup)
```
GET /catalog/sku/{sku_id}            ← OMS, WMS, Pricing Engine
GET /catalog/barcode/{barcode}       ← POS, WMS inbound receiving
GET /catalog/product/{product_id}    ← Storefront product detail page
```

### Customer-facing (via Storefront)
```
GET /catalog/search?q=milo           ← delegates to Elasticsearch
GET /catalog/category/{category_id}  ← browse by category
```

### Admin / Portal
```
POST   /catalog/product              ← create new product
PATCH  /catalog/product/{id}         ← update product info
PATCH  /catalog/product/{id}/status  ← approve, suspend, delist
POST   /catalog/sku                  ← add new variant to product
PATCH  /catalog/sku/{sku_id}         ← update SKU details
```

---

## Key Data Entities

| Entity | Description |
|---|---|
| ParentProduct | Display grouping — name, brand, category, images, status |
| SKU | Sellable variant — barcode, price, dimensions, UOM, channels |
| BarcodeMap | Barcode → SKU + optional price override reference |
| Category | Hierarchy — beverages > malt drinks |
| ProductEvent | Audit log of every status change and update with actor |

---

## Interactions

```
Catalog Service
  │
  ├─► Seller Portal    ── write product data, submit for approval
  ├─► Admin Portal     ── approve/reject, manage status, set retail price
  ├─► Storefront       ── read product details and variants for display
  ├─► Search Indexer   ── listen to catalog events, sync Elasticsearch
  ├─► OMS              ── validate SKU exists and is active on order placement
  ├─► Pricing Engine   ── read base price per SKU
  └─► WMS / POS        ── barcode → SKU lookup, read logistics dimensions
```
