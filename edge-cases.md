# Edge Cases — System Design

All edge cases identified during design session. Every one raised by the designer.

---

## Stock Service

### Buffer edge case — low stock
```
Physical = 3, buffer = 5%
5% of 3 = 0.15 → rounds to 0 → no walk-in protection

Fix: MAX(5% of physical, minimum 1 unit)
  Physical = 3 → buffer = 1 → ATP = 2
  Physical = 1 → buffer = 1 → ATP = 0 → Out of Stock online
```

### Walk-in customer basket problem
```
Walk-in customer holds 30 cans in basket while still shopping
System still shows ATP = 22 (no checkout yet)
Online customer buys 10 → reserved in system
Walk-in checks out all 30 → shelf empty
WMS staff goes to pick online order → nothing there

Fix: buffer is the mitigation, not elimination
     pick failure is a known accepted failure mode
     OMS:packedFailed → refund + release stock
```

### Bundle SKU — pack goes out of stock before cans
```
1 can remaining
ATP for pack = 1 ÷ 6 = 0.16 → rounds down to 0
Pack shows Out of Stock even though 1 can available

This is correct behavior — cannot fulfill a pack with 1 can
```

### Store offline — double-sell window
```
Store goes offline at 10:00:00
Heartbeat timeout detected at 10:03:00 (3 min gap)
During 10:00–10:03:
  POS selling locally (reducing physical stock)
  Online customers still buying from that store's ATP
  No coordination → pick failure guaranteed for these orders

Fix: reduce heartbeat timeout (trade-off: more false positives)
     pick failures handled by OMS compensation flow
```

---

## OMS

### Payment fails after stock reserved
```
Step 1: Stock reserved ✓
Step 2: Payment fails ✗
Stock stuck as RESERVED forever
Other customers cannot buy those units

Fix: OMS:failToPaid compensation
  → enqueue "stock.release" immediately on payment failure
```

### Server crash mid-order
```
Payment taken ✓
Server crashes before telling WMS
WMS never receives fulfillment task
Customer paid, order goes nowhere

Fix: RabbitMQ message durability
  → message survives crash
  → consumer retries on recovery
  → saga ensures each step is independently retryable
```

### Duplicate order on retry
```
Customer clicks Buy
Internet cuts out mid-request
Customer retries → two orders, two charges

Fix: idempotency key generated client-side
  → MongoDB TTL collection (24h expiry)
  → same key → return existing order, do nothing
```

---

## Pricing Engine

### Flash sale price expires mid-cart
```
Customer adds to cart at 11:59 (flash sale price = 28 THB)
Customer checks out at 12:01 (flash sale ended)
Cart shows 28 THB but actual price is now 35 THB

Fix: re-calculate full price at checkout
  → show customer updated price before confirming
  → never silently charge different price than displayed
```

### Race condition on rapid cart updates
```
User changes qty 1→2→3→4 rapidly
4 API calls in flight simultaneously
Call 2 (qty=4) returns first → display 4's price
Call 1 (qty=2) returns late → overwrites with 2's price → wrong

Fix: attach client-side timestamp to each request
  → only apply response if timestamp > last applied
  → discard any stale response
```

### Rounding bias — legal risk
```
Always round discount up → platform gains 0.01 THB per order
× 1,000,000 orders/day = 10,000 THB/day from rounding alone

Fix: business + legal decide rounding rule
  → must be consistent everywhere (checkout, refund, receipt)
  → must be disclosed in terms of service
  → Thailand consumer protection law applies
```

### Decimal precision
```
float64: 0.1 + 0.2 = 0.30000000000000004
Using float for money = eventual wrong numbers

Fix: decimal library (shopspring/decimal in Go)
  → exact arithmetic for 2dp fiat
  → store as string or Decimal type in MongoDB
  → never float64
  → round once at final total, not at each step
```

### Redis 25k request/sec limit under flash sale
```
Flash sale → thousands of concurrent pricing requests
Redis becomes bottleneck at high concurrency

Fix: in-memory cache on Pricing Engine process
  → promotions, prices cached in process memory (microseconds)
  → Redis only called on in-memory miss
  → MongoDB only called on Redis miss
```

---

## WMS

### Supplier delivers wrong quantity
```
PO says 1000 cans, truck has 980
Without discrepancy check → system credits 1000 → oversell possible

Fix: WMS matches received vs PO at inbound
  → discrepancy recorded with reason
  → Stock Service credited actual received (980 only)
  → Seller Portal notifies supplier
```

### Pick failure — item not on shelf
```
WMS generates pick list: "bin B3, shelf 4, 2 Milo cans"
Staff goes to shelf → empty (stolen, misplaced, sold to walk-in)

Fix: WMS marks pick failed
  → enqueue "wms.failed"
  → OMS:packedFailed → payment.refund + stock.release
```

### In-transit stock
```
DC sends 50 cans to Store Silom
Van is driving
DC stock: -50 (already left)
Store stock: not yet +50 (not arrived)
Total visible = -50 units → cannot sell

Fix: in-transit state
  → stock tagged as IN_TRANSIT during movement
  → not sellable until store confirms receipt
  → Stock Service updates both nodes on confirmation
```

### Near-expiry markdown — same SKU not new SKU
```
Staff marks down Milo near expiry with new barcode label
If new SKU created:
  → stock transfer from SKU-001 to MARKDOWN-SKU-001
  → reconciliation complexity
  → split sales reports for same product
  → returns go to which SKU?

Fix: PriceOverride record — not a new SKU
  → markdown barcode maps to same SKU-001 via BarcodeMap
  → POS reads override_price at scan time
  → Stock deducts from SKU-001 as normal
  → StockMovement records sale_type=markdown + markdown_loss
     so boss can see how much was lost
```

---

## Logistics

### Nobody home on delivery
```
Driver arrives at 2pm, customer at work
Cannot leave parcel outside (theft risk)

Fix: business defines policy
  → leave notification slip
  → auto-schedule retry next day
  → notify customer: "missed delivery"
```

### Wrong address
```
Customer typed "Sukhumvit 21" meant "Sukhumvit 12"
Driver goes to wrong building

Fix: driver flags address issue in app
  → customer support contacts customer
  → correct address → reschedule
```

### Courier marks delivered, customer denies
```
Courier system: DELIVERED
Customer: "I never received it"

Possible reasons:
  Driver scanned delivered before actual delivery
  Left at wrong door
  Stolen from doorstep
  Customer lying for free refund

Fix: business decides who to trust by default
  → photo proof of delivery reduces false claims
  → open dispute with courier if photo contradicts claim
```

### Address change after shipping
```
Order shipped, parcel in transit
Customer: "I moved, please deliver to new address"

Fix: only possible before parcel reaches local hub
  → depends on courier supporting mid-transit redirect
  → after driver loaded van → too late
  → business must define cutoff point per courier
```

### Failed delivery 3 times → Return to Sender
```
3 failed attempts → parcel must go back
Customer never home, never responds

Fix: business defines retry count and hold period
  → after max retries → Return to Sender
  → OMS triggers refund
  → WMS receives back, inspects, restocks
```

---

## POS Offline Mode

### Price changed while store offline
```
Head office changes Milo 35 → 40 THB while store offline
Store sells 50 cans at 35 THB (cached local price)
On sync → 50 transactions at wrong price

Fix: known acceptable risk
  → business decides: absorb loss or adjust post-sync
  → flag all offline transactions for review on reconnect
```

### Promotion expired while store offline
```
Flash sale runs 10am-12pm
Store offline 10am, reconnects 2pm
Sold at flash price for 4 hours instead of 2

Fix: known acceptable risk
  → business absorbs, better than store not selling
  → flag for review on sync
```

### Online stock sold while store offline
```
Store offline, POS sells last 5 Milo cans to walk-in
Meanwhile online customer buys 3 from that store's ATP
On reconnect:
  Physical stock: 0
  System expects: 3 reserved for online orders
  Pick failure: guaranteed

Fix: heartbeat mechanism
  → store offline → ATP = 0 → online cannot buy from this store
  → 3-min gap still exists (known risk)
  → handled by pick failure compensation flow
```

---

## Seller / VMI Portal

### Supplier inflates price for fake discount
```
Supplier sets price to 100 THB
Creates promotion: "50% off" → customer sees 50 THB
But normal market price is 35 THB → misleading

Fix: Admin Portal approval required for all promotions
  → category manager reviews: is discount genuine?
  → platform sets min/max price rules per category
```

### Supplier delivers short-dated stock
```
Supplier sends stock close to expiry date
Platform accepts, stocks shelves
Items expire before selling → write-off loss

Fix: inbound inspection records expiry date per batch
  → if expiry too soon → reject delivery at WMS
  → raise claim against supplier via Seller Portal
```

---

## Summary Count

| Subsystem | Edge Cases |
|---|---|
| Stock Service | 4 |
| OMS | 3 |
| Pricing Engine | 5 |
| WMS | 4 |
| Logistics | 5 |
| POS Offline | 3 |
| Seller / VMI Portal | 2 |
| **Total** | **26** |
