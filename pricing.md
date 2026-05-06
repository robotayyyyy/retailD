# Pricing Engine

## What It Is

A **separate service** that calculates the correct final price for any cart — applying all promotions, discounts, coupons, and fees in the correct order.

Called by both Storefront and OMS — ensuring consistent pricing across all channels.

```
Storefront → Pricing Engine: "calculate price for this cart"
OMS        → Pricing Engine: "validate price at checkout"
```

Pricing Engine is a **pure reader** — reads rules and applies math. Never writes business data.

---

## Who Owns Pricing — By Model

Pricing Engine is the final authority on what price is charged. Nobody charges a price directly — all price changes go through portals first, then Pricing Engine picks them up.

```
Portal → update price record → Pricing Engine applies at checkout
```

| Action | Who | Where |
|---|---|---|
| Set retail price per SKU | Category Manager | Internal Admin Portal |
| Suggest retail price | VMI Supplier | VMI Portal |
| Approve / reject suggested price | Category Manager | Internal Admin Portal |
| Set min/max price rules per category | Category Manager | Internal Admin Portal |
| Set regional price differences | Category Manager | Internal Admin Portal |
| Create promotion | VMI Supplier | VMI Portal → needs approval |
| Approve promotion | Category Manager | Internal Admin Portal |
| Set markdown override (near-expiry) | Store staff | WMS |

### Pure Retail — Platform Decides Price

```
Nestle sells Milo to platform at 20 THB (wholesale)
Category Manager sets retail price: 35 THB
Supplier has no say in retail price
```

### VMI — Supplier Suggests, Platform Approves

```
Supplier sets suggested price: 35 THB
Category Manager reviews: within min/max rules? competitive?
Approved → price pushed to Pricing Engine
Rejected → supplier must revise
```

Platform sets min/max bounds per category — supplier can only price within that range:
```
Beverages 180ml: min 25 THB, max 50 THB
```

### Markdown — Store Level Only

Near-expiry markdown is a local POS-level override — not a platform promotion:
```
Store staff creates PriceOverride in WMS
POS reads override at scan time
Pricing Engine not involved — markdown is store-local only
```

---

## Pricing Layers — Applied in Order

Order matters. Same inputs in different order = different final price.

```
Step 1: Base price per item
Step 2: Product discount       (seller marks down specific SKU)
Step 3: Bundle deal            (buy X get Y, 2 for 60 THB)
Step 4: Cart subtotal          (sum of all items after above)
Step 5: Cart-level discount    (spend 500 THB get 50 THB off)
Step 6: Coupon                 (customer applies code)
Step 7: Loyalty point redemption
Step 8: Shipping fee           (based on origin, destination, weight)
Step 9: Shipping discount      (free shipping above threshold)
────────────────────────────────────────────
Output: itemized breakdown + final total
```

**Business must define the exact order.** Engineering implements it strictly.

### Why Order Matters

```
Base price: 100 THB
Product discount: -20%
Coupon: -10 THB

Coupon before discount:  (100 - 10) × 0.8 = 72 THB
Coupon after discount:   (100 × 0.8) - 10 = 70 THB
```

Same inputs, different order = different price. Business decides which is correct.

---

## Promotion Types

### Product Discount
```
Simple percentage or fixed amount off one SKU
Applied per item before cart total
```

### Bundle Deal
```
2 for 60 THB (individual price 35 each = 70 normally)
Buy 2 get 1 free
Buy A + B together → get 20% off
→ requires looking at whole cart, not just one item
```

### Cart-Level Discount
```
Spend 500 THB → get 50 THB off total
→ requires knowing cart subtotal first
→ business must define: which items does discount apply to?
  all items equally? cheapest item first?
```

### Coupon
```
Fixed amount:        -50 THB
Percentage:          -10%
Free item:           add SKU-001 free
Minimum spend:       only valid above 300 THB
Applicable SKUs:     only beverages category
Single use:          one per customer
Multi-use:           unlimited redemptions
Expiry date:         valid until 2024-12-31
```

### Loyalty Points
```
100 points = 1 THB
Customer has 500 points → can redeem up to 5 THB off
Business decides: can loyalty points stack with coupon?
```

### Flash Sale
```
Price drops at exact time: 10:00:00
Price restores at exact time: 12:00:00
Time-sensitive — must be invalidated at exact moment
```

---

## Promotion Stacking — Business Must Define

Every combination must be explicitly decided:

```
Product discount + coupon?      → allowed or not?
Bundle deal + cart discount?    → allowed or not?
Loyalty points + coupon?        → allowed or not?
Multiple coupons?               → never (usually)
Flash sale + member discount?   → allowed or not?
```

This becomes a **compatibility matrix** — business fills it in, engineering enforces it.

---

## Output Must Be Itemized

Never return just the total. Return full breakdown:

```json
{
  "items": [
    { "sku": "SKU-001", "base": 35, "after_discount": 28, "qty": 2, "subtotal": 56 }
  ],
  "bundle_discount": -10,
  "cart_discount": -50,
  "coupon_discount": -10,
  "loyalty_discount": -5,
  "shipping_fee": 30,
  "shipping_discount": -30,
  "total": 51
}
```

Customer sees exactly where every baht went. Required for receipts. Reduces disputes.

---

## Flash Sale — Special Case

Customer adds to cart at 11:59 (flash sale price = 28 THB).
Customer checks out at 12:01 (flash sale ended).

```
Checkout → re-calculate → price is now 35 THB
→ show customer: "flash sale ended, price updated"
→ customer confirms or cancels
```

Never silently charge a different price than what customer saw.

---

## Cart Update — Real-Time Pricing UX

Showing updated price immediately on every cart change influences buying decisions. But recalculating on every tap/click is expensive.

### The Problem — Rapid Changes

```
User changes qty 1 → 2 → 3 → 4 rapidly
= 4 API calls in flight simultaneously
= race condition on which response arrives last
```

### Fix 1 — Debounce

Don't calculate on every change. Wait until user stops:

```
User changes qty → reset 300-500ms timer
User stops changing → timer fires → one single API call
```

One call instead of four. User still sees result quickly.

### Fix 2 — Two-Level Display

```
Instant (client side, no API call):
  → show optimistic price: base price × qty
  → no promotions applied yet
  → screen doesn't feel frozen

Debounced real calculation (300-500ms after user stops):
  → call Pricing Engine
  → apply all promotions accurately
  → update display
```

User sees a number immediately. Accurate number follows shortly after.

### Fix 3 — Timestamp to Prevent Stale Response

Multiple calls in flight — wrong response might arrive last:

```
Call 1 (qty=2) sent at t=100ms
Call 2 (qty=4) sent at t=300ms
Call 2 returns at t=400ms → display price for qty=4 ✓
Call 1 returns at t=600ms → would overwrite with qty=2 price ✗
```

Solution — attach client-side timestamp to each request:

```
Each response carries the timestamp of when it was sent
Only apply response if timestamp > last applied timestamp
Discard any response with older timestamp
```

Timestamp generated client-side — no clock sync issue between servers.

---

## Performance — Making Pricing Fast

### The Latency Problem Without Caching

```
Get base prices         → MongoDB: 10-100ms
Get active promotions   → MongoDB: 10-100ms
Get bundle rules        → MongoDB: 10-100ms
Get coupon validity     → MongoDB: 10-100ms
Get shipping rates      → MongoDB: 10-100ms
Get loyalty balance     → MongoDB: 10-100ms
────────────────────────────────────────────
Total sequential:       60-600ms per calculation
```

Unacceptable for real-time cart updates.

### Layer 1 — Redis Cache (~1ms)

Pricing Engine is a pure reader. Cache all global pricing inputs in Redis:

```
Active promotions        → Redis (event-invalidated or short TTL)
Bundle deal rules        → Redis (TTL: 60s)
Product base prices      → Redis (TTL: 60s, invalidated on change)
Shipping rate cards      → Redis (TTL: 1h)
Flash sale prices        → Redis (event-invalidated at exact start/end time)
```

```
MongoDB read:  10-100ms
Redis read:    < 1ms

6 inputs × 100ms (MongoDB) = 600ms
6 inputs × 1ms  (Redis)    = 6ms
```

### Layer 2 — In-Memory Cache on Pricing Engine (Microseconds)

Redis has limits — around 25k requests/sec on some tiers. At high traffic (flash sales, many concurrent users) even Redis can become a bottleneck.

Solution: **client-side in-memory cache** inside the Pricing Engine process itself.

```
Pricing Engine process memory:
  promotions_cache:    map[string]Promotion  (TTL: 5s)
  bundle_cache:        map[string]BundleRule (TTL: 5s)
  shipping_cache:      map[string]RateCard   (TTL: 60s)
```

```
Request arrives
  → check in-memory cache first (microseconds)
  → HIT  → return immediately, zero Redis calls
  → MISS → fetch from Redis (~1ms) → populate in-memory cache
  → Redis MISS → fetch from MongoDB → populate Redis + in-memory
```

Thousands of requests per second served from memory. Redis only called on cache miss.

### Cache Hierarchy

```
In-memory (Pricing Engine process)   ← microseconds, limited by process memory
  ↓ miss
Redis                                 ← ~1ms, shared across all instances
  ↓ miss
MongoDB                               ← 10-100ms, source of truth
```

### What Must Stay Live (Never Cache)

Customer-specific data cannot be cached globally:

```
Loyalty point balance    → per customer, live DB lookup
Coupon usage history     → per customer, live DB lookup
```

But optimize coupon validation:
```
Check minimum spend first (math only, no DB)
→ if cart too small → reject immediately, skip DB entirely
→ only hit DB if minimum spend is met
```

---

## Cache Invalidation Strategy

| Data | Strategy | Why |
|---|---|---|
| Product base prices | Event-invalidated on change | Accuracy matters |
| Active promotions | Event-invalidated on create/end | Must be exact |
| Bundle rules | TTL 60s | Changes rarely, slight delay ok |
| Shipping rates | TTL 1h | Changes very rarely |
| Flash sale price | Event-invalidated at exact time | Must be exact to the second |
| Loyalty balance | Never cached | Per-customer, always live |

### Flash Sale Invalidation

TTL is not enough for flash sales:

```
Flash sale starts at 10:00:00
TTL cache might serve old price until 10:00:59
59 seconds of wrong price → wrong customer expectation
```

Use event-based invalidation:
```
Scheduler triggers at exactly 10:00:00
  → publish "flash_sale.started" event
  → Pricing Engine listener invalidates Redis + in-memory cache immediately
  → next request gets flash sale price
```

---

## Decimal Precision — Never Use Float for Money

### The Problem

```go
// computers cannot represent all decimals in binary
0.1 + 0.2 = 0.30000000000000004
```

Float64 = eventual wrong numbers for money. Never use float for prices.

### Why Retail Fiat is Simpler Than Crypto

```
Crypto:  0.00000001 BTC, 0.123456789012345678 ETH → 8-18 decimal places
Retail:  100.25 THB, 33.99 USD → always 2 decimal places
```

Retail fiat by definition has only 2 decimal places. No need for Decimal128 (34 significant digits) — that's for crypto precision requirements.

### Recommended Approach for Retail

Use a **Decimal library** — exact arithmetic, human readable, no migration pain:

```go
// Go — shopspring/decimal
price    := decimal.NewFromString("100.25")
discount := decimal.NewFromString("33.33")
pct      := discount.Div(decimal.NewFromInt(100))
result   := price.Mul(pct)   // exact: 33.41332...
```

Store in MongoDB as **string or Decimal type — never float64**:

```
float64:  "price": 100.2500000000001   ← binary floating point error
string:   "price": "100.25"            ← exact, human readable
```

---

## Rounding — Business and Legal Decision

### When to Round

**Round as late as possible** — only at the final total, not at each intermediate step.

```
Bad — round at each step:
  item 1 discount: 33.41   ← rounded early
  item 2 discount: 12.67   ← rounded early
  total discount:  46.08   ← sum of already-rounded numbers

Good — round once at end:
  item 1 discount: 33.41332  ← full precision
  item 2 discount: 12.66666  ← full precision
  sum:             46.07998  ← sum first
  → round once:    46.08     ← round at final total only
```

Rounding early accumulates errors across line items.

### Rounding Bias — Business Decision

Business may intentionally round in their favor:

```
33.41332 → round up to 33.42 at item level
Platform gains 0.01 THB per discounted item
× 1,000,000 orders/day = 10,000 THB/day from rounding alone
```

This is called **rounding bias** — a legitimate revenue decision, not a bug.

| Rule | Effect | Used When |
|---|---|---|
| Round half up | Neutral over time | Safe default |
| Always round up discount | Platform gains | Business decision |
| Always round down discount | Customer gains | Goodwill / marketing |

### Legal Risk of Rounding Bias

In some countries rounding bias is regulated. Thailand consumer protection laws require pricing transparency. Systematic rounding in platform's favor across millions of transactions has resulted in fines for telcos and banks.

```
Business wants to round in their favor?
  → legal team must approve first
  → must be disclosed in terms of service
  → must be consistent and documented
```

### Consistency is Non-Negotiable

Regardless of which rule is chosen — **every part of the system must use the same rounding function**:

```
Checkout rounds up   → customer charged 66.84
Refund rounds down   → customer refunded 66.83
Platform keeps 0.01  → across 1M refunds = 10,000 THB
→ legal problem
```

Checkout, receipt, refund, report — all use the same single rounding function. Never implement rounding in multiple places independently.

---

## Lazy Calculation — Only Full Calculate at Checkout

```
Cart page:
  → show estimated price (cached base + known promotions)
  → fast, slightly approximate, good enough for browsing

Checkout page:
  → full accurate calculation
  → all rules applied, live coupon validation
  → customer sees final price before confirming
  → OMS re-validates same price before charging
```

No need for full accuracy while browsing. Accuracy only matters at the moment of payment.

---

## Key Data (Read Only)

| Data | Source | Cache |
|---|---|---|
| Product base price | MongoDB (catalog) | Redis + in-memory |
| Active promotions | MongoDB (promotions) | Redis + in-memory |
| Bundle deal rules | MongoDB (promotions) | Redis + in-memory |
| Shipping rate cards | MongoDB (logistics config) | Redis + in-memory |
| Flash sale prices | MongoDB (promotions) | Redis, event-invalidated |
| Loyalty point balance | MongoDB (loyalty) | Never — always live |
| Coupon validity + usage | MongoDB (coupons) | Never — always live |

---

## Interactions

```
Pricing Engine
  │
  ├─► Storefront  ── calculate price on cart update (debounced)
  ├─► OMS         ── validate and confirm price at checkout
  ├─► Redis       ── read cached global pricing rules
  └─► MongoDB     ── source of truth, read on cache miss
```
