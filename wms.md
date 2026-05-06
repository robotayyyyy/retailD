# Warehouse Management System (WMS)

## What It Is

After OMS says "someone bought something" — a human being needs to go get it.

WMS manages **everything that happens inside the physical building** — warehouse or store. It deals with physical reality: which shelf, which staff, how to pack, which courier picks up.

```
OMS says: "pack 2 Milo cans for order #123, ship to Bangkok"

WMS answers:
  → which shelf are they on?
  → which staff member goes to get them?
  → how do you pack them?
  → which courier picks them up?
```

---

## Two-Level Warehouse Model

```
┌─────────────────────────────┐
│    Central DC / Warehouse   │  ← traditional large warehouse
│    - bulk storage           │
│    - high-volume fulfillment│
│    - replenishment source   │
└──────────────┬──────────────┘
               │ replenish stock
   ┌───────────┼───────────┐
   ▼           ▼           ▼
[Store A]  [Store B]  [Store C]   ← each store = mini-warehouse
 Ladprao    Silom     Sukhumvit
```

### Central DC (Big Warehouse)
```
Thousands of SKUs
Hundreds of staff
Forklifts, conveyor belts, zones, bins
Fulfills standard delivery orders
Replenishes all stores
```

### Store (Mini Warehouse)
```
Few hundred SKUs
2-3 staff
Simple shelves
Fulfills same-day and click & collect
Serves walk-in customers at the same time
```

Same WMS logic — very different scale and complexity.

---

## The 4 Things WMS Does

1. **Inbound** — receive stock from supplier
2. **Outbound** — pack and ship orders
3. **Replenishment** — send stock from DC to stores
4. **Returns** — receive back, inspect, decide what to do

---

## 1. Inbound — Receiving Stock from Supplier

Supplier arrives at DC with a truck of goods.

```
Truck arrives
  → Staff checks: what was ordered vs what actually arrived
  → Count quantities per SKU per carton
  → Inspect for damage
  → Assign to bin/shelf location
  → Tell Stock Service: "500 Milo cans added at DC"
```

### Purchase Order (PO)

Before a truck arrives there is always a **Purchase Order** — a pre-agreed document:

```
PO-001:
  SKU-001 Milo 180ml × 1000 units
  Expected delivery: 2024-05-01
  Supplier: Nestle
```

WMS matches received stock against the PO. You can only receive what was ordered. This is called **PO receiving**.

### Quantity Discrepancy

System expects 1000 cans. Truck has 980.

```
Expected: 1000
Received: 980
Discrepancy: -20
```

WMS records the discrepancy. Seller Portal notifies the supplier. Stock Service is credited 980 only — not 1000.

---

## 2. Outbound — Pack and Ship Orders

OMS sends a fulfillment task to WMS:

```
Order #123:
  - Milo 180ml × 2      (bin B3, shelf 4)
  - Instant noodles × 1 (bin A1, shelf 2)
  Ship to: customer address
```

WMS generates a **pick list** for staff:

```
Go to: bin B3, shelf 4 → pick 2 Milo cans
Go to: bin A1, shelf 2 → pick 1 instant noodles
Bring to packing station
```

Then:
```
Pack into box
Print shipping label
Hand to courier
Tell OMS: order is READY_TO_SHIP
```

### Pick Failure

Staff goes to shelf → item is not there (stolen, misplaced, already sold to walk-in).

```
WMS marks: pick failed
→ enqueue "wms.failed" {order_id}
→ OMS:packedFailed handles compensation (refund + release stock)
```

### Batch Picking — DC Optimization

At DC, sending one staff per order is inefficient. Instead:

```
Batch 50 orders together
One staff walks entire warehouse once
Picks items for all 50 orders in one trip
Returns to packing station
Items sorted and packed per order
```

Fewer walking trips = faster throughput. Used at DC only — stores are too small for this.

---

## 3. Replenishment — DC Sends Stock to Stores

Stock Service detects Store Silom is running low:

```
Store Silom: Milo 180ml = 5 units (below reorder threshold of 10)
→ trigger replenishment
```

WMS at DC:
```
Generate replenishment order: send 50 Milo cans to Store Silom
Staff at DC picks 50 cans
Loads onto delivery van
Van drives to Store Silom
Staff at store counts and confirms receipt
Stock Service updated:
  DC: -50
  Store Silom: +50
```

### In Transit State

While the van is driving:

```
DC stock:      -50  (already left)
Store stock:   not yet +50 (not arrived)
In Transit:    +50
```

In-transit stock is not sellable. It becomes available only when the store confirms receipt.

---

## 4. Returns — Item Comes Back

Customer returns an item. Courier delivers it back to origin (DC or store).

WMS staff inspects:

```
Is it damaged?
  YES → dispose or return to supplier → write off stock
  NO  → is it still sellable?
          YES → restock, tell Stock Service +1 available
          NO  → dispose → write off, log reason
```

### Three Outcomes

| Condition | Action | Stock Effect |
|---|---|---|
| Good condition | Restock on shelf | +1 Available |
| Damaged, supplier fault | Return to supplier | No stock change, raise claim |
| Damaged beyond use | Dispose | Write off, log reason |

### Refund Timing

Refund does not wait for inspection. OMS triggers refund when return is received. Inspection and stock adjustment happen after — money and stock are separate flows.

---

## DC Staff vs Store Staff — How They Differ

Retrieval and fulfillment work looks the same on paper — but the staff doing it are completely different in role, skill, and context.

---

### DC Staff — Dedicated, Specialized

DC staff do **one job all day**. They are warehouse professionals.

```
Picker    → walks warehouse all day, picks items only
Packer    → sits at packing station all day, packs only
Receiver  → at loading dock all day, receives deliveries only
Returns   → processes returns only
```

**Their environment:**
```
Large building, thousands of SKUs
Organized by zones, aisles, bins
Barcode scanners, conveyor belts, forklifts
Pick list on handheld device showing exact bin location
Batch picking — one trip covers 50 orders
```

**Measured by:** picks per hour, errors per shift, throughput volume.

DC WMS is optimized for **throughput and volume**.

---

### Store Staff — Generalist, Context-Switching

Store staff do **everything** in a small space simultaneously:

```
08:00 → receive replenishment van from DC (inbound)
09:00 → open store, serve walk-in customers at counter (POS)
10:00 → notification: 3 online orders to pick
10:15 → go to shelf, pick items between serving customers
10:30 → pack items, hand to Grab courier at door
11:00 → customer arrives for click & collect
14:00 → restock shelves from back room
18:00 → do daily stock count
```

Same person. Different hats. All day. Interrupted constantly.

**Their environment:**
```
Small store, few hundred SKUs
No bin system — shelves organized by category
No dedicated handheld scanner
Pick list on basic tablet or phone
Cannot batch pick — store too small, order volume too low
```

Store WMS is optimized for **simplicity** — so a generalist can do it without training.

---

### How This Changes WMS Design

| Feature | DC WMS | Store WMS |
|---|---|---|
| Pick list detail | Exact bin + zone + aisle | Category + shelf name only |
| Batch picking | Yes — 50 orders per trip | No — one order at a time |
| Barcode scanning | Required | Optional / simple |
| Task assignment | Dedicated role assigned | Any available staff |
| Fulfillment speed | High volume, fast | Low volume, best effort |
| Interface | Handheld scanner device | Phone or tablet app |
| Interruption handling | None — dedicated role | Must pause for walk-in customers |

---

### The Core Difference in One Line

```
DC staff:    "My only job right now is picking order #123."

Store staff: "Let me finish serving this customer,
              then I'll pick that online order when I get a moment."
```

---

## Near-Expiry Handling — WMS Responsibilities

When Stock Service flags a batch as NEAR_EXPIRY, WMS tasks store staff to act.

### Staff Actions

```
WMS alert received: "Batch A (Milo 180ml) expires in 3 days, qty: 20"

Staff chooses action in WMS app:
  1. Markdown    → print override label, move to markdown shelf
  2. Bundle      → pair with another product, create in-store deal
  3. Return      → pack and return to supplier (if contract allows)
  4. Write-off   → accept loss, dispose on expiry
```

### Markdown Flow in WMS

```
Staff selects: markdown
  → WMS generates PriceOverride record
  → WMS prints markdown label with new barcode
  → Staff sticks label on each item
  → Staff moves items to markdown shelf
  → WMS records: batch status = MARKDOWN_ACTIVE
```

### Expiry Write-Off Flow in WMS

```
On expiry date:
  → WMS generates task: "Remove expired Batch A, qty: remaining"
  → Staff removes items from shelf
  → Staff confirms removal in WMS app
  → WMS notifies Stock Service: write off N units, reason = expired
  → Loss recorded in StockMovement with batch_id and loss_value
```

### Supplier Return Flow in WMS

```
Staff selects: return to supplier
  → WMS creates return shipment task
  → Items packed and labelled for return
  → Logistics picks up
  → Seller Portal notified: return shipment incoming
  → On receipt confirmed by supplier: raise credit claim
```

---

## Key Data

| Entity | Description |
|---|---|
| PurchaseOrder | Pre-agreed delivery from supplier — SKU, qty, expected date |
| InboundReceipt | Actual received qty, discrepancy vs PO |
| Bin / Location | Storage slot in DC or store shelf |
| PickTask | Item + bin + quantity assigned to staff for an order |
| PickBatch | Group of orders picked together in one warehouse walk |
| PackRecord | Parcel dimensions, weight, label |
| ShipmentLabel | Tracking number attached to parcel |
| ReplenishmentOrder | DC → Store transfer with in-transit state |
| ReturnReceipt | Returned item condition + disposition decision |

---

## Interactions

```
WMS
  │
  ├─► OMS ──────────── receive fulfillment task, report READY / HANDED_OFF / FAILED
  ├─► Stock Service ── update quantities (inbound, pick, return, restock, in-transit)
  ├─► Logistics ─────── hand off parcel, provide tracking number
  └─► Seller Portal ─── notify supplier of inbound receipt, discrepancy
```
