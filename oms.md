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
