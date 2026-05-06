# Seller / Supplier Portal

## What It Is

The back office of the platform. While Storefront serves customers — Seller Portal serves the business side: suppliers, vendors, and internal staff who make the platform run.

```
Customer sees   → Storefront
Supplier sees   → Seller Portal
Internal staff  → Admin Portal
```

---

## Business Model Clarification

This platform follows the **7-Eleven model — pure retail with VMI hybrid**.

### Pure Retail (Core)
```
Platform buys stock from supplier upfront (Purchase Order)
Platform owns the inventory after delivery
Platform sets retail price, runs promotions, fulfills orders
Supplier's job ends at delivering goods to DC
Customer buys from "the platform" — not from "Nestle"
```

### VMI — Vendor Managed Inventory (Consignment)
```
Supplier consigns stock to platform (no upfront purchase)
Supplier owns stock until it sells
Supplier has visibility into stock levels and sales
Platform still fulfills all orders — supplier never touches customer orders
Supplier gets paid per unit sold minus platform commission
```

### Not a Marketplace
```
Marketplace (Lazada, Shopee):
  → suppliers list products, fulfill orders, set prices freely
  → customer buys from "Seller X"

This platform:
  → platform owns or consigns inventory
  → platform fulfills everything
  → customer always buys from the platform brand
```

---

## Three Separate Portals

Different portals for different relationships — same backend services.

---

## Portal 1 — Procurement Portal (Pure Retail)

**Used by:** internal procurement team + suppliers delivering to DC

### Supplier Side
```
→ receive Purchase Orders from platform
→ confirm delivery schedule
→ record actual delivery (qty, batch, expiry date)
→ view invoice and payment status
→ upload product specs, images, barcode
```

### Platform Side (Internal Procurement Team)
```
→ create and manage Purchase Orders
→ approve inbound deliveries
→ manage supplier contracts and pricing agreements
→ track supplier performance:
    on-time delivery rate
    fill rate (ordered vs actually delivered)
    quality rejection rate
```

### How It Works

```
Platform procurement team creates PO:
  PO-001: Milo 180ml × 10,000 units
  Expected delivery: 2024-05-10
  Agreed price: 28 THB/unit

Supplier confirms delivery schedule
  → "will deliver 2024-05-10, morning"

Supplier delivers to DC
  → WMS records inbound receipt
  → actual qty matched against PO
  → discrepancy flagged if any

Platform approves receipt
  → Stock Service updated
  → Invoice generated
  → Payment scheduled per contract terms
```

Platform owns the inventory after PO fulfilled. Supplier's job is done.

---

## Portal 2 — VMI Portal (Consignment)

**Used by:** brands managing their own consigned stock on the platform

### Supplier Side
```
→ view real-time stock levels at DC and all stores
→ trigger own replenishment to DC when stock is low
→ set their own retail prices (within platform-defined min/max rules)
→ create promotions (subject to platform approval)
→ view sales performance per SKU per store per period
→ view consignment payout and deductions
→ download settlement statements
```

### Platform Side (Internal)
```
→ approve or reject supplier promotions
→ set commission rates per supplier or per category
→ manage consignment settlement schedule
→ set min/max price rules per category
```

### How It Works

```
Supplier ships stock to DC (no PO — consignment)
  → WMS records inbound, tagged as consignment
  → Stock Service: stock owned by supplier until sold

Customer buys item
  → sale recorded
  → ownership transfers to customer
  → platform owes supplier: sale price − commission

End of settlement period (weekly/monthly):
  Total sales of consigned items: 200,000 THB
  Platform commission (15%):      -30,000 THB
  Return deductions:               -2,000 THB
  ────────────────────────────────────────────
  Net payout to supplier:         168,000 THB
```

Supplier has visibility and control — but platform still owns fulfillment completely.

### What VMI Supplier Can and Cannot Do

```
Can:
  → set own price (within platform rules)
  → create promotions (needs approval)
  → trigger replenishment to DC
  → view real-time stock and sales

Cannot:
  → confirm or reject customer orders
  → choose which stores carry their product (platform decides)
  → access customer personal data
  → change price below platform-defined minimum
```

---

## Portal 3 — Internal Admin Portal

**Used by:** platform staff — category managers, finance, operations

```
Product approval:
  → review and approve new products before going live
  → reject with feedback if quality standard not met
  → manage product categories and attributes

Pricing management:
  → set retail price per SKU (pure retail model)
  → approve or reject supplier-suggested prices (VMI model)
  → set min/max price rules per category
  → set regional price differences (Bangkok vs province)
  → all price changes pushed to Pricing Engine — never bypassed

Promotion approval:
  → review VMI supplier promotions before activation
  → prevent fake "50% off" where original price was inflated
  → schedule approved promotions in Pricing Engine

Supplier management:
  → onboard new suppliers (procurement or VMI)
  → set commission rates per supplier
  → manage contracts and SLA terms
  → track supplier performance metrics

Platform configuration:
  → configure buffer % per store tier or product category
  → set replenishment thresholds
  → manage store nodes and DC locations

Finance:
  → view platform-wide revenue and commission
  → manage settlement schedules
  → issue invoices and statements

Analytics:
  → platform-wide sales performance
  → category performance
  → store performance comparison
  → near-expiry markdown loss and write-off reporting
```

---

## Product Approval Flow (Both Portal Types)

New product goes through approval before appearing on Storefront:

```
Supplier creates product listing
  → submits for review

Category Manager reviews:
  → correct category?
  → images meet quality standard?
  → description accurate and not misleading?
  → barcode/SKU valid?
  → price within acceptable range?

Approved → product goes live on Storefront
Rejected → supplier gets feedback, must fix and resubmit
```

Without approval gate:
```
Supplier lists fake product
Supplier uses competitor's images
Supplier inflates price for fake discount
Wrong category → bad search results
```

---

## Product Data Structure

```
Parent Product: Milo
  ├── SKU-001: Milo 180ml can
  ├── SKU-002: Milo 400ml can
  └── SKU-003: Milo 1L box

Per SKU:
  Basic:       name, description, images, barcode
  Pricing:     base price, bulk price tiers
  Logistics:   weight, dimensions (for shipping calculation)
  Perishable:  shelf life, expiry tracking required (yes/no)
  Category:    category, subcategory, attributes (size, flavor)
```

---

## Reporting and Analytics

### What Suppliers See

```
Sales volume:     units sold per SKU per store per period
Revenue:          gross sales amount
Returns:          units returned, refund cost deducted
Sell-through:     % of stocked inventory sold
Stock turnover:   how fast inventory moves
Payout history:   past settlements, upcoming schedule
```

### Why Reporting Needs a Separate Database

Supplier queries like:
```
"Show daily sales for all 500 SKUs across 200 stores for last 6 months"
```

This is a massive aggregation query. Running it against production MongoDB kills live transaction performance.

**Solution: separate analytics database**

```
Production DB (MongoDB):
  → handles live orders, stock, transactions
  → optimized for writes and fast point reads
  → never touched by reporting queries

Analytics DB (separate, e.g. ClickHouse or BigQuery):
  → copy of data, updated periodically (hourly or nightly)
  → optimized for large aggregations and time-series reports
  → all supplier and internal reports run here
```

---

## What Portals Share (Same Backend Services)

All three portals read from and write to the same underlying services:

```
Product Catalog   → shared, Procurement and VMI both write here
Stock Service     → shared, both portals read stock levels
Pricing Engine    → VMI writes promotions here
OMS               → read only (order status)
Analytics DB      → shared, all portals read reports here
Finance Service   → different calculation per model (PO vs consignment)
```

Different front-end portals. Same backend. Different permissions.

---

## Key Data

| Entity | Owned By | Used By |
|---|---|---|
| Purchase Order | Procurement Portal | Procurement + WMS |
| Consignment Record | VMI Portal | VMI + Finance |
| Product Catalog | Both Portals | Storefront + Search |
| Supplier Contract | Admin Portal | Finance + Procurement |
| Commission Rate | Admin Portal | Finance + VMI |
| Settlement Statement | Finance Service | VMI Portal |
| Supplier Performance | Analytics DB | Admin Portal |

---

## Interactions

```
Procurement Portal
  │
  ├─► WMS              ── inbound delivery confirmation
  ├─► Stock Service    ── stock updated after PO received
  └─► Finance Service  ── invoice and payment schedule

VMI Portal
  │
  ├─► Stock Service    ── view real-time stock, trigger replenishment
  ├─► Pricing Engine   ── submit promotions for approval
  ├─► Analytics DB     ── view sales reports
  └─► Finance Service  ── view consignment payout and deductions

Internal Admin Portal
  │
  ├─► Product Catalog  ── approve/reject products
  ├─► Pricing Engine   ── approve/reject promotions
  ├─► All services     ── platform-wide configuration and monitoring
  └─► Analytics DB     ── platform-wide reporting
```
