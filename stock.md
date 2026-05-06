# Stock / Inventory Service

## What It Is

The Stock Service tracks **how many of each product exists, at every location, and what state it is in**.

It is the single source of truth for all product quantities across every location — Central DC, each physical store, and in-transit stock. Every other service asks Stock Service before making decisions about selling, fulfilling, or moving goods.

---

## SKU — What It Is

**SKU = a specific product variant.** Every distinct sellable item has its own SKU code.

```
Milo 180ml can    → SKU-001
Milo 400ml can    → SKU-002   (different SKU, different size)
Milo 180ml 6-pack → SKU-003   (different SKU, bundle of cans)
```

SKU is always combined with a location to mean anything:

```
"30 units available" → means nothing
"30 units of SKU-001 at Store Silom" → means something
```

**SKU + Location = a stock record.**

---

## Stock Nodes — Where Stock Lives

Every location that holds inventory is a **stock node**:

```
Central DC (Warehouse)   → bulk storage, source of replenishment
Store Ladprao            → shelf stock for that branch
Store Silom              → shelf stock for that branch
In Transit               → moving from DC to a store (not sellable yet)
```

Stock Service tracks quantities **per SKU per node** — not just a total.

```
SKU: Milo 180ml

  Node                  Physical Qty    ATP (available online)
  ─────────────────────────────────────────────────────────
  Central DC             500             400
  Store Ladprao           30              22
  Store Silom             12               8
  Store Sukhumvit         25              18
```

---

## ATP — Available to Promise

**ATP = how many units the online store is actually allowed to sell.**

It is NOT the same as physical stock on shelf. Physical stock includes units held for walk-in customers and units already reserved by online orders.

```
ATP = Physical Qty − Walk-in Buffer − Reserved (online orders)
```

### Walk-in Buffer

The buffer is a safety margin — units kept back from online sales to protect stock for walk-in customers.

```
Example (Store Ladprao, 30 units, 20% buffer):
  Walk-in Buffer = 30 × 20% = 6
  Reserved online = 2
  ATP = 30 − 6 − 2 = 22
```

**Important:** The buffer is not a physical separation. The 6 bottles are still on the same shelf. It is a system rule — "do not promise these online."

Buffer % is configurable per store tier or product category via Seller Portal.

### Buffer Edge Case — Low Stock

When stock is very low, percentage buffer becomes meaningless:

```
Physical = 3 units, buffer = 5%
5% of 3 = 0.15 → rounds to 0
ATP = 3 → all 3 promised online, no protection for walk-in
```

**Fix: always enforce a minimum buffer unit**

```
buffer = MAX(5% of physical, minimum 1 unit)

Physical = 3 → buffer = 1 → ATP = 2
Physical = 1 → buffer = 1 → ATP = 0 → shows Out of Stock online
```

At 1 unit remaining, the system shows Out of Stock online and protects the last unit for walk-in.

### Walk-in Customer Basket Problem

The system only knows about a sale when checkout is completed. It cannot track items a walk-in customer is currently holding in their basket while shopping.

This means:
- Walk-in picks up 30 cans, still shopping
- System still shows ATP = 22
- Online customer buys 10 → reserved in system
- Walk-in checks out all 30
- WMS staff goes to pick 10 for online order → shelf is empty

This is called a **pick failure** — a known, accepted failure mode in omnichannel retail.

The buffer exists to **reduce** pick failures, not eliminate them. High-traffic stores get a higher buffer. When a pick failure occurs, OMS cancels the order and the customer receives a refund.

---

## Packaging Hierarchy — UOM

Goods are delivered and stored in multiple packaging levels, not just individual units.

```
1 pallet  = 48 cartons
1 carton  = 24 packs
1 pack    = 6 cans     ← base unit (what Stock Service tracks)
1 can     = 1 can
```

This is called the **Unit of Measure (UOM) hierarchy**.

**Stock Service always tracks at the base unit level (the can).** Receiving happens at carton level — the conversion runs at the inbound step.

```
WMS receives: 10 cartons
1 carton = 24 packs × 6 cans = 144 cans
10 cartons = 1,440 cans added to stock
```

Each SKU defines its own base unit and conversion chain.

---

## Selling Multiple Pack Sizes (Bundle / Virtual SKU)

When both the single can and the 6-pack are sold, they share the same physical stock.

```
SKU-001   Milo 180ml single can   → owns physical stock (base unit)
SKU-002   Milo 180ml 6-pack       → virtual SKU, no separate stock
                                     when sold: deduct 6 units from SKU-001
```

Only the base SKU holds the actual stock number. The pack SKU points to it with a multiplier.

### ATP for Bundle SKU

```
Stock Service holds: SKU-001 = 30 cans

Customer buys 1 pack (SKU-002)
  → reserve 6 cans from SKU-001
  → ATP for can = 24
  → ATP for pack = 24 ÷ 6 = 4 packs remaining

Customer buys 5 single cans (SKU-001)
  → ATP for can = 1
  → ATP for pack = 1 ÷ 6 = 0 → Out of Stock (pack)
```

Pack goes out of stock before cans run out — correct behavior.

---

## Stock States Per Unit

A single unit moves through these states:

```
AVAILABLE
  │ order placed (reserve)
  ▼
RESERVED        ← physically still on shelf, but system holds it for an order
  │ order shipped / confirmed
  ▼
COMMITTED       ← sold, gone

RESERVED
  │ order cancelled
  ▼
AVAILABLE       ← released back

COMMITTED
  │ customer returns (restockable)
  ▼
AVAILABLE
```

**Reserved is the critical middle state** — item is on shelf but belongs to an order. Other orders cannot claim it.

---

## Stock Operations — What Changes Stock

| Operation | Trigger | Effect |
|---|---|---|
| Reserve | OMS: order placed | Available → Reserved |
| Commit | OMS: order shipped | Reserved → Committed |
| Release | OMS: order cancelled | Reserved → Available |
| Inbound | WMS: stock received from supplier | +qty Available |
| Outbound | POS: in-store sale | −qty Committed |
| Restock | WMS: returned item restocked | +qty Available |
| Transfer | WMS: DC → Store replenishment | Move qty between nodes |
| Adjustment | Manual stock count | Correct physical qty with reason |
| Write-off | Expired / damaged | −qty with reason logged |

---

## Discrepancy — When System Doesn't Match Reality

System says 30 cans. Staff counts 27. 3 cans are missing.

**Common causes:**
- Shoplifting
- Damaged and thrown out without updating system
- POS scan error
- Receiving error (carton label said 24 but actually had 23)

**How it's handled:**

Staff performs a **stock count** — physically count every item. Then submit an adjustment:

```
SKU: Milo 180ml
Location: Store Silom
System qty: 30
Actual qty: 27
Difference: −3
Reason: shrinkage / damaged
```

The reason is mandatory — shrinkage costs are tracked for business reporting and supplier claims.

---

## Expiry Dates (Perishable Items)

Relevant for food, drinks, and fresh items. Stock is not just a number — it has a shelf life.

Same SKU, same location, different batches:

```
Milo 180ml — Store Silom
  Batch A: 20 cans, expires 2024-06-01
  Batch B: 10 cans, expires 2024-09-15
```

### FEFO — First Expired, First Out

Always sell the earliest expiry batch first. Staff places newer stock at the back, older stock at the front of shelf.

### Expiry Alerts

```
N days before expiry → alert staff → move to markdown shelf
On expiry date       → ATP = 0 for this batch → cannot sell online or at POS
After expiry         → write off → deduct from stock with reason "expired"
```

Alert threshold is configurable per product category:
```
Fresh food:    1 day before
Drinks/snacks: 3 days before
Dry goods:     7 days before
```

Expired stock showing as available is a serious problem for a food retailer.

---

## Near-Expiry Markdown Handling

When a batch is flagged NEAR_EXPIRY, staff moves items to a markdown shelf at a reduced price.

### Online ATP = 0 for Near-Expiry Batches

Near-expiry stock is removed from online sales entirely — only available to walk-in customers who take the product immediately.

```
Risk of shipping near-expiry item:
  → transit takes 1-2 days
  → item may arrive expired or with very short shelf life
  → customer complaint, return, legal risk
```

### Price Override — Not a New SKU

Markdown pricing uses a **price override** — not a new SKU. The product identity does not change.

Staff prints a **markdown label** with a new barcode and sticks it on the item. POS scans this barcode and looks up the override price.

```
Original barcode:  8850006100015  → SKU-001 Milo 180ml, price = 35 THB
Markdown barcode:  9999999000123  → still SKU-001 Milo 180ml, price = 20 THB
```

**Why not a new SKU?**
```
New SKU would require:
  → stock transfer from SKU-001 to SKU-MARKDOWN-001
  → reconciliation complexity
  → split sales reports for the same product
  → confusing supplier data
  → return goes to which SKU?

Price override is simpler — same SKU, different price instruction only.
```

### PriceOverride Record

```
PriceOverride:
  id:             PO-001
  sku:            SKU-001
  store_node:     store-silom
  batch_id:       BATCH-A
  barcode:        9999999000123   ← markdown label barcode
  override_price: 20 THB
  reason:         near_expiry
  valid_until:    2024-06-01      ← expiry date, auto-expires
  created_by:     staff-id
```

### How POS Uses It

```
Cashier scans markdown barcode: 9999999000123
  → POS looks up PriceOverride table
  → found: override_price = 20 THB, SKU = SKU-001
  → charges 20 THB
  → stock deducted from SKU-001 as normal
```

### StockMovement — Capturing Markdown Context

Every markdown sale must record extra context for reporting:

```
StockMovement:
  sku:            SKU-001
  qty:            -1
  reason:         sold
  sale_type:      markdown          ← normal vs markdown
  batch_id:       BATCH-A
  override_id:    PO-001
  original_price: 35 THB
  sold_price:     20 THB
  markdown_loss:  15 THB            ← original - sold, pre-calculated
```

Without this context, markdown sales look identical to full-price sales — no way to report losses.

### What This Enables in Reporting

```
Near-Expiry Performance Report — May 2024

  SKU-001 Milo 180ml, Store Silom, Batch A (expires 2024-06-01)
  ─────────────────────────────────────────────────────────────
  Units at near-expiry:              20
  Units sold at markdown (20 THB):   14   ← recovered value
  Units expired unsold:               6   ← written off, total loss

  Revenue recovered:     14 × 20 =  280 THB
  Revenue at full price: 14 × 35 =  490 THB
  Markdown loss:        490 - 280 = 210 THB
  Write-off loss:         6 × 35 = 210 THB
  ─────────────────────────────────────────
  Total batch loss:                 420 THB
```

**Business decisions data drives:**
```
High markdown sell-through (70%+) → strategy working, keep it
Low markdown sell-through (15%)   → alert earlier, or lower markdown price
                                     or wrong store for this product
```

### Full Expiry Lifecycle

```
Batch arrives at store (inbound)
  → WMS records batch_id + expiry date
  → Stock Service tracks batch

N days before expiry
  → Stock Service auto-flags batch: NEAR_EXPIRY
  → alert to store staff via WMS
  → Supplier Portal notifies supplier

Staff action (choose one):
  → markdown: print override label, move to markdown shelf
  → bundle: pair with other product in promotion
  → return to supplier: if contract allows
  → nothing: accept eventual write-off

On expiry date
  → Stock Service sets ATP = 0 for this batch (online + POS)
  → WMS tasks staff: "remove expired batch A"

Staff removes items
  → confirms removal in WMS
  → WMS tells Stock Service: write off N units, reason = expired
  → Stock Service deducts from physical qty

Loss recorded in StockMovement:
  → qty: -N
  → reason: expired
  → batch_id: BATCH-A
  → loss_value: N × original_price
```

### Who Absorbs the Loss

```
Platform stored too long:        platform absorbs
Supplier delivered short-dated:  raise claim against supplier via Seller Portal
Normal shrinkage:                split per contract terms
```

---

## Fulfillment Location — Which Node to Pick From

When an online order comes in, OMS and Stock Service together decide which node fulfills it.

```
Same-day delivery:
  → find stores near customer with ATP > 0
  → pick nearest store with enough stock
  → route order to that store's WMS

Standard delivery:
  → check Central DC stock first
  → DC has stock → fulfill from DC
  → DC out of stock → find nearest store with ATP > 0
```

**Trade-off:** DC has more stock but is farther. Stores are closer but have limited stock.

### Split Order Edge Case

Customer orders Milo + instant noodles.

```
Milo      → Store Silom has it
Noodles   → Store Silom is out, Store Ladprao has it
```

Options:
- **Split shipment** — 2 parcels from 2 stores, faster but higher shipping cost
- **Single fulfillment** — wait for DC to have both, slower but cleaner

This is a business decision configured in OMS routing rules.

---

## Oversell Prevention

Two strategies depending on traffic volume:

### Normal Traffic — Optimistic Locking
- Reserve stock in DB with a version check
- If two orders try to reserve the last unit simultaneously, one fails → out-of-stock response returned

### High Concurrency (Flash Sale) — Fast Gate
- ATP counter maintained in a fast store
- Atomic decrement: only one request wins the last unit
- If counter hits 0 → reject immediately without touching DB
- Prevents oversell at the gate before any DB write

---

## Who Talks to Stock Service

Stock Service is the **only system allowed to change stock numbers**. All others ask it to act — they do not update stock directly.

| Service | Role |
|---|---|
| Seller Portal / WMS | Adds stock (inbound receipt) |
| OMS | Reserves and releases stock (order placed / cancelled) |
| WMS | Removes stock (outbound after shipment) |
| Storefront | Reads ATP only — never modifies stock |

```
Stock Service
  │
  ├─► Storefront    ── read ATP per SKU per node (product availability display)
  ├─► OMS           ── reserve / commit / release on order events
  ├─► WMS           ── update qty on inbound / outbound / transfer / return
  └─► Seller Portal ── view stock levels, trigger low stock alerts, upload inbound
```

---

## Store Online / Offline State

Stock Service tracks whether each store node is online or offline based on heartbeat signal from POS.

```
Store node states:
  ONLINE   → heartbeat received within last 3 minutes → ATP exposed online
  OFFLINE  → no heartbeat for 3+ minutes → ATP = 0, invisible to online customers
```

### When Store Goes OFFLINE

```
Stock Service sets ATP = 0 for all SKUs at that store node
Storefront reads ATP = 0 → shows Out of Stock for that store online
OMS routing excludes this store → no orders routed here
POS continues selling locally — Stock Service is not involved until reconnect
```

### When Store Comes Back ONLINE

```
Heartbeat resumes → central marks store ONLINE
POS syncs all offline transactions → Stock Service applies deductions
Stock Service recalculates ATP for all SKUs at that store
Redis cache for that store's ATP invalidated → fresh values repopulated
Store re-enters OMS routing pool
```

### Why This Is Necessary

Two reasons to exclude offline stores from online ATP:

```
1. Cannot receive order:
   Store offline → WMS never receives fulfillment task → order stuck

2. Prevent double-selling:
   POS selling locally (reducing physical stock)
   Online customer also buying from same ATP
   No coordination possible → pick failure guaranteed
```

### 3-Minute Gap Risk

Between store going offline and heartbeat timeout being detected (3 min window) — online orders can still route to that store. Pick failure possible for orders in this gap. Handled by OMS pick failure compensation flow (refund + restock).

Reducing heartbeat timeout shortens the gap but increases false positives if network is briefly unstable. Business must choose acceptable timeout value.

---

## Low Stock Alerts

- ATP at a node falls below threshold → alert Seller Portal
- Total ATP across all nodes falls below reorder point → trigger replenishment from DC

---

## Key Data

| Entity | Description |
|---|---|
| StockNode | Location ID (DC or Store ID) |
| SKU | Product variant ID, defines base unit |
| UOMConversion | SKU + pack level + multiplier to base unit |
| BundleSKU | Virtual SKU + component SKU + qty multiplier |
| SKUStock | SKU + Node + physical_qty + reserved_qty + buffer_pct |
| StockBatch | SKU + Node + batch ID + expiry date + qty |
| StockReservation | Order ID + SKU + Node + qty + status |
| StockMovement | Audit log of every qty change with reason |

---

## Data Schema Examples (JSON)

### StockNode

```json
{
  "node_id": "DC-BKK-001",
  "node_type": "DC",
  "name": "Central DC Bangkok",
  "status": "ONLINE",
  "last_heartbeat_at": null
}
```

Store node:

```json
{
  "node_id": "STORE-SILOM-001",
  "node_type": "STORE",
  "name": "Store Silom",
  "status": "ONLINE",
  "last_heartbeat_at": "2024-05-01T10:58:00Z"
}
```

---

### SKU

```json
{
  "sku_id": "SKU-001",
  "name": "Milo 180ml",
  "base_unit": "can",
  "is_virtual": false,
  "is_perishable": true,
  "expiry_alert_days": 3
}
```

---

### UOMConversion

```json
{
  "sku_id": "SKU-001",
  "conversions": [
    { "pack_level": "can",    "multiplier": 1  },
    { "pack_level": "pack",   "multiplier": 6  },
    { "pack_level": "carton", "multiplier": 144 },
    { "pack_level": "pallet", "multiplier": 6912 }
  ]
}
```

---

### BundleSKU

```json
{
  "sku_id": "SKU-003",
  "name": "Milo 180ml 6-pack",
  "is_virtual": true,
  "components": [
    {
      "component_sku_id": "SKU-001",
      "qty_multiplier": 6
    }
  ]
}
```

---

### SKUStock

```json
{
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "physical_qty": 30,
  "reserved_qty": 2,
  "buffer_pct": 0.20,
  "buffer_min_units": 1,
  "atp": 22,
  "updated_at": "2024-05-01T10:55:00Z"
}
```

ATP calculation:
```
buffer_units = MAX(30 × 20%, 1) = 6
atp = 30 − 6 − 2 = 22
```

---

### StockBatch

```json
{
  "batch_id": "BATCH-A",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty": 20,
  "expiry_date": "2024-06-01",
  "status": "NEAR_EXPIRY",
  "received_at": "2024-04-01T08:00:00Z"
}
```

Statuses: `ACTIVE` `NEAR_EXPIRY` `EXPIRED` `WRITTEN_OFF`

---

### StockReservation

```json
{
  "reservation_id": "RES-ORD-123-001",
  "order_id": "ORD-123",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty": 2,
  "status": "RESERVED",
  "reserved_at": "2024-05-01T09:00:00Z",
  "committed_at": null,
  "released_at": null
}
```

After shipment:

```json
{
  "status": "COMMITTED",
  "committed_at": "2024-05-01T14:00:00Z"
}
```

After cancellation:

```json
{
  "status": "RELEASED",
  "released_at": "2024-05-01T10:00:00Z"
}
```

---

### StockMovement

Inbound from supplier:

```json
{
  "movement_id": "MOV-20240501-001",
  "sku_id": "SKU-001",
  "node_id": "DC-BKK-001",
  "qty_delta": 980,
  "reason": "INBOUND",
  "reference_id": "RCP-20240501-001",
  "batch_id": "BATCH-001",
  "sale_type": null,
  "override_id": null,
  "original_price": null,
  "sold_price": null,
  "markdown_loss": null,
  "loss_value_thb": null,
  "created_at": "2024-05-01T09:30:00Z"
}
```

Online order reserved:

```json
{
  "movement_id": "MOV-20240501-002",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty_delta": -2,
  "reason": "RESERVE",
  "reference_id": "ORD-123",
  "batch_id": null,
  "sale_type": null,
  "created_at": "2024-05-01T10:00:00Z"
}
```

Markdown sale:

```json
{
  "movement_id": "MOV-20240501-003",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty_delta": -1,
  "reason": "SOLD",
  "reference_id": "POS-TXN-00456",
  "batch_id": "BATCH-A",
  "sale_type": "MARKDOWN",
  "override_id": "PRICE-OVR-001",
  "original_price": 35.0,
  "sold_price": 20.0,
  "markdown_loss": 15.0,
  "loss_value_thb": null,
  "created_at": "2024-05-01T11:00:00Z"
}
```

Write-off (expired):

```json
{
  "movement_id": "MOV-20240501-004",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty_delta": -6,
  "reason": "EXPIRED",
  "reference_id": "BATCH-A",
  "batch_id": "BATCH-A",
  "sale_type": null,
  "loss_value_thb": 210.0,
  "created_at": "2024-06-01T08:00:00Z"
}
```

Manual stock adjustment:

```json
{
  "movement_id": "MOV-20240501-005",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "qty_delta": -3,
  "reason": "ADJUSTMENT",
  "adjustment_reason": "SHRINKAGE",
  "system_qty_before": 30,
  "actual_qty": 27,
  "submitted_by": "STAFF-010",
  "created_at": "2024-05-01T18:00:00Z"
}
```

---

### PriceOverride

```json
{
  "override_id": "PRICE-OVR-001",
  "sku_id": "SKU-001",
  "node_id": "STORE-SILOM-001",
  "batch_id": "BATCH-A",
  "barcode": "9999999000123",
  "override_price": 20.0,
  "original_price": 35.0,
  "reason": "NEAR_EXPIRY",
  "valid_until": "2024-06-01",
  "created_by": "STAFF-010",
  "created_at": "2024-05-29T09:00:00Z"
}
```

---

### Status Reference

| Entity | Status Values |
|---|---|
| StockNode | `ONLINE` `OFFLINE` |
| StockBatch | `ACTIVE` `NEAR_EXPIRY` `EXPIRED` `WRITTEN_OFF` |
| StockReservation | `RESERVED` `COMMITTED` `RELEASED` |
| StockMovement reason | `INBOUND` `RESERVE` `COMMIT` `RELEASE` `SOLD` `RESTOCK` `TRANSFER` `ADJUSTMENT` `EXPIRED` `WRITE_OFF` |
| StockMovement sale_type | `null` (full price) `MARKDOWN` |

---

## Data Flow — DC to Store Replenishment

How documents change at each step when stock moves from DC to a store.

**Before — both nodes at rest**

```json
// SKUStock — DC
{ "sku_id": "SKU-001", "node_id": "DC-BKK-001", "physical_qty": 500, "reserved_qty": 0, "atp": 400 }

// SKUStock — Store Silom
{ "sku_id": "SKU-001", "node_id": "STORE-SILOM-001", "physical_qty": 5, "reserved_qty": 0, "atp": 3 }
```

---

**Step 1 — ReplenishmentOrder created, van departs**

DC physical_qty drops immediately. Store is unchanged — stock is in transit, not yet available.

```json
// ReplenishmentOrder — created
{
  "replenishment_id": "RPL-001",
  "from_warehouse_id": "DC-BKK-001",
  "to_warehouse_id": "STORE-SILOM-001",
  "status": "IN_TRANSIT",
  "lines": [{ "sku_id": "SKU-001", "qty_sent": 50, "qty_confirmed": null }]
}

// SKUStock — DC  (physical_qty reduced)
{ "sku_id": "SKU-001", "node_id": "DC-BKK-001", "physical_qty": 450, "reserved_qty": 0, "atp": 350 }

// SKUStock — Store Silom  (unchanged — not arrived yet)
{ "sku_id": "SKU-001", "node_id": "STORE-SILOM-001", "physical_qty": 5, "reserved_qty": 0, "atp": 3 }

// StockMovement — DC deduction
{ "movement_id": "MOV-001", "sku_id": "SKU-001", "node_id": "DC-BKK-001",
  "qty_delta": -50, "reason": "TRANSFER", "reference_id": "RPL-001" }
```

---

**Step 2 — Store staff confirms receipt**

Store physical_qty increases. ReplenishmentOrder closed.

```json
// ReplenishmentOrder — completed
{
  "replenishment_id": "RPL-001",
  "status": "COMPLETED",
  "confirmed_received_at": "2024-05-01T11:30:00Z",
  "lines": [{ "sku_id": "SKU-001", "qty_sent": 50, "qty_confirmed": 50 }]
}

// SKUStock — Store Silom  (physical_qty increased)
{ "sku_id": "SKU-001", "node_id": "STORE-SILOM-001", "physical_qty": 55, "reserved_qty": 0, "atp": 43 }

// SKUStock — DC  (unchanged)
{ "sku_id": "SKU-001", "node_id": "DC-BKK-001", "physical_qty": 450, "reserved_qty": 0, "atp": 350 }

// StockMovement — Store addition
{ "movement_id": "MOV-002", "sku_id": "SKU-001", "node_id": "STORE-SILOM-001",
  "qty_delta": +50, "reason": "TRANSFER", "reference_id": "RPL-001" }
```

---

**Summary of what changed**

| Document | Step 1 (van departs) | Step 2 (store confirms) |
|---|---|---|
| `ReplenishmentOrder` | created, `IN_TRANSIT` | `COMPLETED`, `qty_confirmed` filled |
| `SKUStock` DC | `physical_qty` −50 | unchanged |
| `SKUStock` Store | unchanged | `physical_qty` +50 |
| `StockMovement` DC | −50, reason `TRANSFER` | — |
| `StockMovement` Store | — | +50, reason `TRANSFER` |

Both StockMovements reference the same `RPL-001` — full traceability of where stock came from and where it went.
