# Order Management System (OMS)

## What It Is

OMS is the **coordinator of everything that happens after a customer buys something**.

When a customer clicks "Place Order" — OMS:
- Checks items are available
- Collects payment
- Tells the warehouse to pack and ship
- Keeps the customer updated
- Handles anything that goes wrong

---

## Order Sources

| Source | Channel |
|---|---|
| Online checkout (web/app) | Online Store |
| POS transaction | Physical Store |
| LINE / third-party integration | External Channel |

---

## Order Lifecycle

Every status change = something real happened in the physical world.

```
PENDING           ← order created, stock reserved, waiting for payment
  │ payment confirmed
  ▼
PAID              ← money collected, tell warehouse to pack
  │ WMS picks and packs
  ▼
CONFIRMED
  │ WMS finishes packing
  ▼
READY_TO_SHIP     ← parcel ready, waiting for courier
  │ handed to logistics
  ▼
SHIPPED           ← courier picked up, in transit
  │ delivered to customer
  ▼
DELIVERED
  │
  ├──► RETURN_REQUESTED
  │         │
  │         ▼
  │    RETURNED / REFUNDED
  │
  └──► CANCELLED (any stage before SHIPPED)
```

---

## The 3 Things OMS Must Get Right

1. **Never lose an order** — customer paid → order must exist no matter what
2. **Never create a duplicate order** — customer retries checkout → same order returned, not two orders created
3. **Always keep stock and order in sync** — order cancelled → stock must be released immediately

---

## Architecture — Saga + Event-Driven + RabbitMQ

OMS is both a **producer** and a **consumer** of queue messages.

- **Producer** — sends instructions to other services (reserve stock, charge payment, fulfill)
- **Consumer** — listens for results and decides next step

```
        OMS
      ┌──────┐
      │      │ ──── produces ────► stock.reserve
      │      │ ◄─── consumes ──── stock.reserved
      │      │
      │      │ ──── produces ────► payment.charge
      │      │ ◄─── consumes ──── payment.success
      │      │
      │      │ ──── produces ────► wms.fulfill
      │      │ ◄─── consumes ──── wms.packed
      └──────┘
```

### Why Not a Simple API?

A plain API handles one request and forgets. If the server crashes mid-order (after payment but before telling WMS) — payment is taken but nothing ships.

Queue-based messaging solves this:
- Messages survive crashes — they stay in the queue until processed
- Each step is retried automatically if a consumer fails
- System returns to a clean state via compensation steps

### Why RabbitMQ Over Kafka

RabbitMQ is lower cost and simpler to operate. For OMS the order flow has a fixed, known sequence — not an infinite stream of unknown subscribers. RabbitMQ fits this perfectly.

---

## Saga Pattern

OMS order placement is a **multi-step distributed transaction**. Each step has a compensation (undo) action if something fails later.

```
Step            Queue sent             Compensation if later step fails
──────────────────────────────────────────────────────────────────────
1. Reserve      stock.reserve          stock.release
2. Charge       payment.charge         payment.refund
3. Fulfill      wms.fulfill            wms.cancel
4. Ship         logistics.pickup       logistics.cancel
```

**Reserve happens before charge** — intentional. Charging a customer for something you cannot fulfill is worse than holding stock for a few seconds. If payment fails, stock releases immediately.

---

## Full OMS Flow — Happy Path

```
Customer places order
  │
  ▼
OMS:createOrder
  → create order (PENDING)
  → enqueue "stock.reserve" {order_id, sku, qty, node}

Stock Consumer
  → updateOne available_qty (atomic, with condition)
  → success → enqueue "stock.reserved" {order_id}
  → fail    → enqueue "stock.failed"  {order_id}

OMS:createOrder listens to "stock.reserved"
  → enqueue "payment.charge" {order_id, amount}

Payment Consumer
  → charges card
  → success → enqueue "payment.success" {order_id}
  → fail    → enqueue "payment.failed"  {order_id}

OMS:paid listens to "payment.success"
  → update order → PAID
  → go func → enqueue "wms.fulfill"          {order_id, node, items}
  → go func → enqueue "notification.send"    {order_id, "order confirmed"}

WMS Consumer
  → creates pick task for staff
  → staff packs
  → success → enqueue "wms.packed"  {order_id}
  → fail    → enqueue "wms.failed"  {order_id}

OMS:packed listens to "wms.packed"
  → update order → READY_TO_SHIP
  → enqueue "logistics.pickup" {order_id, node}

Logistics Consumer
  → dispatches courier
  → courier picks up → enqueue "logistics.shipped"   {order_id, tracking_no}
  → delivered        → enqueue "logistics.delivered" {order_id}

OMS:packed listens to "logistics.delivered"
  → update order → DELIVERED
  → go func → enqueue "stock.commit"        {order_id}
  → go func → enqueue "notification.send"   {order_id, "order delivered"}
```

---

## Full OMS Flow — Failure Paths + Compensations

```
"stock.failed"
  → OMS cancels order (CANCELLED)
  → enqueue "notification.send" {order_id, "out of stock"}
  → nothing to undo (no payment taken yet)

"payment.failed"
  → OMS:failToPaid cancels order (CANCELLED)
  → enqueue "stock.release"       {order_id}   ← undo reservation
  → enqueue "notification.send"   {order_id, "payment failed"}

"wms.failed" (pick failure — item not on shelf)
  → OMS:packedFailed cancels order (CANCELLED)
  → enqueue "payment.refund"      {order_id}   ← undo charge
  → enqueue "stock.release"       {order_id}   ← undo reservation
  → enqueue "notification.send"   {order_id, "sorry, item unavailable"}
```

---

## OMS Consumer Split

OMS is split into focused consumers — each owns exactly one state transition. Same codebase, same database, different entry points.

```
Consumer              Listens To               Owns This Transition
─────────────────────────────────────────────────────────────────────────
OMS:createOrder       new order request        create order, enqueue stock.reserve
OMS:paid              payment.success          → PAID, enqueue wms.fulfill
OMS:failToPaid        payment.failed           → CANCELLED, enqueue stock.release
OMS:packed            wms.packed               → READY_TO_SHIP, enqueue logistics.pickup
OMS:packedFailed      wms.failed               → CANCELLED, enqueue payment.refund + stock.release
```

### Why Split?

**Independent scaling**
```
Flash sale → scale up OMS:createOrder only
OMS:packed doesn't need more instances
```

**Independent failure**
```
OMS:packed crashes
→ only "packed" processing is delayed
→ order creation still works fine
→ messages wait in queue, processed when recovered
```

**Easier to debug**
```
Bug in refund logic → look at OMS:failToPaid only
Not a 2000-line monolithic OMS service
```

### File Structure

```
cmd/
  oms-create-order/     main.go
  oms-paid/             main.go
  oms-fail-to-paid/     main.go
  oms-packed/           main.go
  oms-packed-failed/    main.go
```

---

## Fulfillment Routing Logic

OMS decides which node fulfills the order:

```
Order placed
  │
  ├─► Is it in-store POS? ──────────────► Record sale, deduct store stock
  │
  └─► Is it online order?
        │
        ├─► Click & Collect at store? ──► Route to that store's WMS
        │
        ├─► Same-day delivery? ──────────► Find nearest store with ATP > 0
        │                                   Route to that store's WMS
        │
        └─► Standard delivery? ──────────► Route to Central DC / WMS
```

**Routing Criteria:**
- Distance to customer
- Stock availability (ATP) at each node
- Node capacity (not overloaded)
- Delivery SLA required

---

## Cancellation & Refund Rules

| Stage | Cancellation Allowed | Stock Action | Refund |
|---|---|---|---|
| PENDING / PAID | Yes | Release reservation | Full refund |
| CONFIRMED | Depends on seller | Release reservation | Full refund |
| READY_TO_SHIP | No (or fee applies) | N/A | Partial refund |
| SHIPPED | No | N/A | Return process |
| DELIVERED | No | Return process | After return |

---

## Idempotency

### The Problem

Customer clicks "Place Order" — internet cuts out — they click again.

```
Request 1 → OMS:createOrder → creates order, reserves stock, enqueues payment
Request 2 → OMS:createOrder → creates ANOTHER order, reserves stock AGAIN
```

Customer gets charged twice. Two stock reservations for one intent.

### How It Works

Every order request from the client carries a unique key **generated on the client side** before sending:

```
{
  idempotency_key: "user-123-cart-456-1714900000",
  items: [...],
  ...
}
```

OMS:createOrder before doing anything:

```
Check: has this idempotency_key been seen before?

YES → return the existing order, do nothing
NO  → proceed, create order, save the key
```

**Client generates the key — not OMS.** Because OMS only sees the request after the customer clicked. It cannot know if this is a retry.

### The Bloat Problem

Keeping keys forever = millions of rows over time. Keys are only useful during the retry window (minutes to hours). After that they are useless.

**Solution: TTL (Time to Live)** — automatically delete keys after 24 hours.

---

### Option 1 — MongoDB TTL Index (Recommended)

Store keys in a separate `idempotency_keys` collection. Add a TTL index on `created_at` — MongoDB deletes expired documents automatically. No background job needed.

```
Collection: idempotency_keys
{
  _id:        "user-123-cart-456-1714900000",  ← key as _id, O(1) lookup
  order_id:   "order-789",
  created_at: ISODate("2024-05-01T10:00:00Z")  ← TTL index on this field
}

Index:
  db.idempotency_keys.createIndex(
    { created_at: 1 },
    { expireAfterSeconds: 86400 }
  )
```

**Flow:**
```
Request arrives with idempotency_key
  → findOne by _id
  → found   → return existing order_id, stop
  → missing → create order → insert key with created_at = now
              MongoDB auto-deletes after 24h
```

No extra dependency. Already on MongoDB.

---

### Option 2 — Redis TTL

Store key → order_id in Redis with automatic expiry.

```
Order request arrives
  → GET idempotency:{key}
  → exists     → return existing order_id, stop
  → not exists → create order → SET idempotency:{key} {order_id} EX 86400
```

Redis handles expiry automatically. No index needed.

**Risk:** if Redis is down at the moment of order creation — key is not saved — retry creates a duplicate. Needs careful error handling.

---

### Comparison

| | MongoDB TTL Index | Redis TTL |
|---|---|---|
| Extra dependency | No | Yes (Redis) |
| Auto cleanup | Yes | Yes |
| Lookup speed | O(1) by _id | O(1) |
| Risk | None | Duplicate if Redis down during save |
| Recommended | Yes — simpler, no new dependency | Only if Redis already in critical path |

---

## Key Data

| Entity | Description |
|---|---|
| Order | Order ID, customer, items, total, channel, status |
| OrderItem | SKU, qty, unit price, seller |
| FulfillmentNode | Store ID or DC ID assigned to fulfill |
| PaymentRef | Payment transaction ID |
| ShipmentRef | Logistics tracking number |
| IdempotencyKey | Unique key per order request, prevents duplicates |

---

## Interactions

```
OMS
  │
  ├─► Stock Service ─────── reserve / commit / release stock
  ├─► Payment Service ────── charge / refund
  ├─► WMS ───────────────── send fulfillment task / cancel
  ├─► Logistics ─────────── request shipment / cancel
  ├─► Seller Portal ──────── notify seller of new order
  └─► Notification ──────── send status updates to customer
```

---

## Data Schema Examples (JSON)

### Order

```json
{
  "order_id": "ORD-123",
  "idempotency_key": "user-123-cart-456-1714900000",
  "customer_id": "USR-001",
  "channel": "ONLINE",
  "status": "PAID",
  "fulfillment_type": "STANDARD_DELIVERY",
  "fulfillment_node_id": "DC-BKK-001",
  "shipping_address": {
    "name": "สมชาย มีสุข",
    "address": "123 Sukhumvit Rd, Bangkok",
    "postal_code": "10110",
    "phone": "0812345678"
  },
  "items_total_thb": 124.0,
  "shipping_fee_thb": 40.0,
  "grand_total_thb": 164.0,
  "created_at": "2024-05-01T09:00:00Z",
  "updated_at": "2024-05-01T09:05:00Z"
}
```

Channels: `ONLINE` `POS` `LINE` `EXTERNAL`

Fulfillment types: `STANDARD_DELIVERY` `SAME_DAY` `CLICK_AND_COLLECT` `IN_STORE`

---

### OrderItem

```json
{
  "order_item_id": "ITEM-ORD-123-001",
  "order_id": "ORD-123",
  "sku_id": "SKU-001",
  "name": "Milo 180ml",
  "qty": 2,
  "unit_price_thb": 35.0,
  "subtotal_thb": 70.0,
  "seller_id": "SELLER-NESTLE-001"
}
```

---

### FulfillmentNode

```json
{
  "order_id": "ORD-123",
  "node_id": "DC-BKK-001",
  "node_type": "DC",
  "assigned_at": "2024-05-01T09:05:00Z",
  "routing_reason": "STANDARD_DELIVERY_DC_PREFERRED"
}
```

Same-day routed to store:

```json
{
  "order_id": "ORD-124",
  "node_id": "STORE-SILOM-001",
  "node_type": "STORE",
  "assigned_at": "2024-05-01T09:10:00Z",
  "routing_reason": "SAME_DAY_NEAREST_STORE_WITH_ATP"
}
```

---

### PaymentRef

```json
{
  "payment_ref_id": "PAY-ORD-123-001",
  "order_id": "ORD-123",
  "provider": "OMISE",
  "transaction_id": "chrg_test_abc123xyz",
  "amount_thb": 164.0,
  "status": "CHARGED",
  "charged_at": "2024-05-01T09:01:00Z",
  "refunded_at": null
}
```

After refund:

```json
{
  "status": "REFUNDED",
  "refunded_at": "2024-05-01T10:00:00Z"
}
```

Statuses: `PENDING` `CHARGED` `REFUNDED` `FAILED`

---

### ShipmentRef

```json
{
  "shipment_ref_id": "SHIP-ORD-123-001",
  "order_id": "ORD-123",
  "courier": "FLASH_EXPRESS",
  "tracking_number": "FE1234567890TH",
  "status": "SHIPPED",
  "picked_up_at": "2024-05-01T14:00:00Z",
  "delivered_at": null
}
```

After delivery:

```json
{
  "status": "DELIVERED",
  "delivered_at": "2024-05-02T10:30:00Z"
}
```

Statuses: `PENDING` `SHIPPED` `DELIVERED` `FAILED`

---

### IdempotencyKey

MongoDB document (TTL index on `created_at`, expires after 24h):

```json
{
  "_id": "user-123-cart-456-1714900000",
  "order_id": "ORD-123",
  "created_at": "2024-05-01T09:00:00Z"
}
```

---

### Queue Message Payloads

`stock.reserve`:

```json
{
  "order_id": "ORD-123",
  "node_id": "DC-BKK-001",
  "items": [
    { "sku_id": "SKU-001", "qty": 2 },
    { "sku_id": "SKU-003", "qty": 1 }
  ]
}
```

`payment.charge`:

```json
{
  "order_id": "ORD-123",
  "amount_thb": 164.0,
  "payment_token": "tok_test_abc123"
}
```

`wms.fulfill`:

```json
{
  "order_id": "ORD-123",
  "node_id": "DC-BKK-001",
  "fulfillment_type": "STANDARD_DELIVERY",
  "items": [
    { "sku_id": "SKU-001", "qty": 2 },
    { "sku_id": "SKU-003", "qty": 1 }
  ],
  "shipping_address": {
    "name": "สมชาย มีสุข",
    "address": "123 Sukhumvit Rd, Bangkok",
    "postal_code": "10110",
    "phone": "0812345678"
  }
}
```

`wms.packed` / `wms.failed`:

```json
{ "order_id": "ORD-123" }
```

`logistics.pickup`:

```json
{
  "order_id": "ORD-123",
  "node_id": "DC-BKK-001",
  "pack_id": "PACK-20240501-0123"
}
```

---

### Status Reference

| Entity | Status Values |
|---|---|
| Order | `PENDING` `PAID` `CONFIRMED` `READY_TO_SHIP` `SHIPPED` `DELIVERED` `RETURN_REQUESTED` `RETURNED` `REFUNDED` `CANCELLED` |
| PaymentRef | `PENDING` `CHARGED` `REFUNDED` `FAILED` |
| ShipmentRef | `PENDING` `SHIPPED` `DELIVERED` `FAILED` |
