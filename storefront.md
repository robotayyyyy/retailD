# Store Front

## What It Is

Storefront is the **face of the entire platform** — the only part the customer actually sees and touches.

Everything else — Stock, OMS, WMS, Logistics — the customer never sees. They only see Storefront.

```
Customer sees:        What's actually happening behind:

Browse products   →   Stock Service returning ATP
Add to cart       →   Cart stored in session/cache
Checkout          →   OMS creating order, Stock reserving
"Order confirmed" →   OMS saga running in background
Track order       →   Logistics returning normalized status
```

Storefront is mostly a **reader** — reads from other services and presents information. It rarely owns data itself.

---

## Two Types of Storefront

### Online Store (Web / Mobile App)
- Customer browses and orders via website or mobile app (LINE, iOS, Android)
- Payment: credit card, PromptPay, e-wallet, COD
- Fulfillment: delivery to door or click & collect at store
- Flow: Browse → Search → Cart → Checkout → Track

### Physical Store (POS)
- Walk-in customer at a 7-Eleven-style branch
- Cashier uses a POS terminal to scan and charge
- Payment: cash, card, QR code at counter
- Fulfillment: customer takes product home immediately
- Flow: Scan item → Total → Payment → Receipt → Done

Same products, same stock, same OMS behind both. Very different user experience and technical requirements.

---

## Latency — The Most Critical Requirement

Storefront is customer-facing. Real humans are waiting. Slow = losing money directly.

```
Stock Service slow?   → other services retry, no human feels it
OMS slow?             → order takes 2 seconds longer, acceptable
Storefront slow?      → customer sees spinner → leaves → lost sale
```

```
> 3 seconds load time → 40% of users abandon
> 1 second delay      → 7% drop in conversions
```

### Where Latency Comes From

Every page load needs multiple things:

```
Product page:
  → product details        (catalog)
  → price                  (pricing engine)
  → stock availability     (stock service ATP)
  → promotions             (promotion engine)
  → reviews                (review service)
  → recommended products   (recommendation engine)
```

Sequential calls:
```
100 + 100 + 100 + 100 + 100 + 100 = 600ms — too slow
```

### Three Ways to Fix Latency

**1. Parallel calls**
```
Call all 6 simultaneously
→ all finish in ~100ms
→ page loads in 100ms not 600ms
```

**2. Cache aggressively**
```
Product details    → cache, changes rarely
Promotions         → cache, changes on schedule
ATP                → short cache (30 seconds), changes often
Price              → cache, invalidate on promotion events
```

Cached response = no downstream call = near zero latency.

**3. Pre-computation**
```
Instead of: customer requests → compute recommendations → return
Do this:    pre-compute recommendations nightly → store result
            customer requests → read pre-computed result → instant
```

### The Trade-off

Caching means data can be slightly stale:

```
ATP cached 30 seconds
→ customer sees "In Stock"
→ actual stock ran out 10 seconds ago
→ customer adds to cart → checkout → stock reserve fails
→ customer gets "out of stock" at checkout
```

Acceptable. Better than slow pages. Handle the error gracefully at checkout.

### The One Thing You Cannot Cache

**Checkout must always be real-time.**

```
Browse page    → cache aggressively, slight staleness ok
Product page   → cache aggressively, slight staleness ok
Search results → cache, short TTL
Checkout       → never cache — stock reservation and payment must be live
```

---

## 1. Search

### Why You Cannot Use MongoDB/SQL for Search

```
SELECT * FROM products WHERE name LIKE '%milo%'
```

Problems:
- Cannot handle typos ("mlo" → no results)
- Cannot understand synonyms ("chocolate milk" → no Milo)
- No relevance ranking
- Slow on large catalogs
- No language awareness (Thai + English)

### What You Use Instead

A dedicated search engine — **Elasticsearch** or **OpenSearch**.

Purpose-built for:
```
Typo tolerance      → "mlo" still finds "Milo"
Synonyms            → "chocolate milk" maps to Milo
Relevance scoring   → most popular/relevant result first
Multi-language      → Thai and English same index
Faceted filtering   → filter by brand, price range, category
```

### How It Stays in Sync

Elasticsearch is not the primary database. MongoDB is the source of truth. Elasticsearch is a **read replica optimized for search**.

```
Seller Portal updates product
  → MongoDB updated (source of truth)
  → event published: "product.updated"
  → Search indexer listens
  → updates Elasticsearch index
```

Small delay between catalog update and search reflecting it — acceptable.

---

## 2. Cart and Pricing

### Cart

Cart is a temporary list of items before checkout. Stored in cache with TTL.

```
{
  user_id: "U001",
  items: [
    { sku: "SKU-001", qty: 2 },
    { sku: "SKU-002", qty: 1 }
  ],
  expires_at: "2024-05-01T14:00:00Z"
}
```

**Cart does NOT reserve stock.** Just holds intended purchase. Stock reserved only when order is placed.

If customer abandons cart → TTL expires → cart deleted automatically.

### Pricing Layers

Price is not one number. It has layers:

```
Base price:          35 THB
Member discount:     -5 THB   (loyalty tier)
Coupon applied:      -10 THB
Bundle deal:         2 for 60 THB (instead of 70)
Flash sale:          -20% for next 2 hours
Shipping fee:        +30 THB
─────────────────────────────
Final price:         calculated by Pricing Engine
```

Rules can stack, conflict, or exclude each other.

### Promotion Rules — Business Must Define

```
Can coupon stack with flash sale?
Does member discount apply before or after coupon?
Bundle deal — applies per item or per cart?
Free shipping above 500 THB — before or after discount?
```

Business defines the rule order. Engineering builds a **Pricing Engine** that applies them in the correct sequence.

### Price Must Be Re-calculated at Checkout

Customer adds to cart at 10am (flash sale price = 28 THB).
Customer checks out at 2pm (flash sale ended, price = 35 THB).

```
Checkout called
  → re-fetch current price for all items
  → re-apply all valid promotions at this moment
  → if price changed → show customer updated price before confirming
```

Never trust the price stored in cart at checkout time.

---

## 3. POS Offline Mode

### The Problem

7-Eleven stores are everywhere — including areas with unreliable internet. If POS requires internet, store stops selling when connection drops. Unacceptable.

### Local-First Architecture

POS is the **authority for its own store** while offline. Central system catches up on reconnect.

POS keeps a local copy of everything it needs to operate:

```
Product catalog    → synced every night
Prices             → synced every night (local Pricing Engine)
Promotions         → synced on change   (local Pricing Engine)
Local stock count  → own store only, updated on every sale
Payment methods    → cash always works offline
```

**POS has its own local Pricing Engine** — a cached copy of prices and promotions sufficient to calculate final price without calling the central server.

**POS has its own local stock count** — tracks its own store inventory locally:
```
Local stock: Milo 180ml = 100 units
Customer buys 2 → local stock: 98 units (no internet needed)
Customer buys 3 → local stock: 95 units
```

When internet drops:
```
POS switches to offline mode automatically
Continues scanning and charging customers using local pricing
Deducts from local stock count on every sale
Sales queued locally
Receipts printed normally
Customer experience: unchanged
```

When internet restores:
```
POS syncs all queued transactions to central system
Stock Service reconciles: applies all offline deductions
Pricing discrepancies flagged for review
Local stock resynced from central after reconciliation
```

### Store Heartbeat — Protecting Online Customers

Online customers must never buy from an offline store — the store cannot receive orders and local POS is already selling that stock independently.

**Solution: heartbeat mechanism**

```
POS sends heartbeat to central server every 60 seconds:
  POST /heartbeat { store_id: "store-silom", timestamp: "..." }

Central server tracks last_seen per store
Background job checks every 3 minutes:
  last_seen > 3 min ago → mark store: OFFLINE
```

When store goes OFFLINE:
```
Stock Service sets ATP = 0 for all SKUs at that store
Storefront stops showing that store's stock online
OMS stops routing orders to that store
```

When store comes back ONLINE:
```
Heartbeat resumes → central marks store: ONLINE
POS syncs offline transactions → Stock Service reconciles
Stock Service recalculates ATP for that store
Redis cache for that store's ATP invalidated → fresh values repopulated
Store re-enters online routing pool
OMS can route orders to this store again
```

**Heartbeat uses simple periodic HTTP — not WebSocket.**
WebSocket is for pushing real-time updates to browsers. Heartbeat just needs a regular ping on a timer.

### WebSocket — Where It Actually Belongs

WebSocket is the right tool for pushing real-time updates to online customers:

```
Flash sale starts → price drops → page updates instantly without refresh
Order status changes → tracking page updates live
Stock runs out → "Out of Stock" appears immediately
```

Without WebSocket → customer must manually refresh to see updates.
With WebSocket → server pushes changes to all connected browsers instantly.

### Full Offline → Online Transition Flow

```
Store goes offline:
  1. Heartbeat stops
  2. Central marks store: OFFLINE after 3 min
  3. ATP = 0 for this store → online cannot buy from here
  4. POS continues selling locally (local pricing + local stock)
  5. All transactions queued locally

Store comes back online:
  6. Heartbeat resumes → central marks store: ONLINE
  7. POS syncs all queued transactions
  8. Stock Service applies all offline deductions
  9. Redis ATP cache for this store invalidated → recalculated
  10. Store re-enters OMS routing pool
  11. Online orders can route here again
```

### Hard Problems in Offline Mode

**Problem 1 — Price changed while offline**
```
Head office changes Milo price 35 → 40 THB while store offline
Store sells 50 cans at 35 THB (old local pricing cache)
On sync → 50 cans sold at wrong price
Business must decide: absorb loss or adjust after the fact?
```

**Problem 2 — Promotion expired while offline**
```
Flash sale runs 10am-12pm
Store goes offline at 10am, comes back at 2pm
Store sold at flash sale price for 4 hours instead of 2
Who absorbs the difference?
```

**Problem 3 — Double-sell window before heartbeat timeout**
```
Store goes offline at 10:00:00
Heartbeat timeout detected at 10:03:00 (3 min window)
Between 10:00 and 10:03 → online customers can still buy from this store's ATP
POS also selling locally during this window
Pick failure possible for orders placed in the 3 min gap
```

Reduce window by lowering heartbeat timeout — trade-off: more false positives if network is briefly unstable.

### The Only Real Solution

These problems cannot be fully solved by engineering. Business must accept that offline mode has **known acceptable risks**:

```
Price discrepancy     → absorb small losses, acceptable for continuity
Promotion overshoot   → absorb, better than store not selling
3 min double-sell gap → pick failure handled by refund flow
```

Engineering minimizes risk by:
```
Keeping heartbeat timeout short (balance with false positive risk)
Syncing pricing updates frequently to local cache
Flagging all offline transactions for review on sync
```

---

## Fulfillment Options Shown to Customer

| Option | Description |
|---|---|
| Home Delivery | Order online → delivered from warehouse or nearest store |
| Same-Day Delivery | Ship from nearest store → arrive same day |
| Click & Collect | Order online → pick up at chosen store |
| In-Store Purchase | Walk in → POS transaction |

---

## Key Data

| Data | Source |
|---|---|
| Product catalog | Seller Portal → Elasticsearch (search) + MongoDB (detail) |
| Stock availability | Stock Service (ATP value) |
| Cart | Cache with TTL |
| Order status | OMS |
| Delivery tracking | Logistics Service |
| Promotions | Pricing Engine |
| Recommendations | Pre-computed, stored separately |

---

## Interactions

```
Customer
  │
  ├─► [Search]              ──► Elasticsearch
  ├─► [Product Page]        ──► Catalog + Stock Service (ATP) + Pricing Engine
  ├─► [Cart]                ──► Cache (local, no stock reservation yet)
  ├─► [Checkout]            ──► OMS (real-time, never cached)
  ├─► [Track Order]         ──► OMS + Logistics
  └─► [POS Sale]            ──► OMS + Stock Service (deduct)
```
