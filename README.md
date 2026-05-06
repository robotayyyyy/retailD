# Retail System Design — Hybrid Omnichannel (7-Eleven Model)

## Overview

A **hybrid omnichannel retail platform** where the same brand operates both physical stores and an online store — sharing the same inventory, order system, and customer profile.

Supports two supplier models running simultaneously:
- **Pure Retail** — platform buys stock upfront, owns inventory, operates everything
- **VMI (Vendor Managed Inventory)** — supplier consigns stock, gets paid per unit sold

---

## Model Type

- **Omnichannel Retail** (physical + online unified)
- Inspired by: 7-Eleven Thailand (stores everywhere + online ordering)

---

## Subsystems

| File | Subsystem | Summary |
|---|---|---|
| [stock.md](./stock.md) | Stock / Inventory Service | Single source of truth for all product quantities across every node. Manages ATP, reservations, stock states, expiry, and oversell prevention. |
| [oms.md](./oms.md) | Order Management System | Orchestrates the full order lifecycle via Saga pattern + RabbitMQ. Handles routing, cancellation, refund, and idempotency. |
| [wms.md](./wms.md) | Warehouse Management System | Manages physical operations at DC and stores — inbound, pick-pack-ship, replenishment, and returns. |
| [logistics.md](./logistics.md) | Logistics Service | Moves parcels from fulfillment nodes to customer. Handles last-mile, failed delivery, multi-courier tracking, and own fleet routing. |
| [storefront.md](./storefront.md) | Store Front | Customer-facing layer — online store + physical POS. Covers search, cart, pricing UX, latency strategy, and POS offline mode. |
| [seller-portal.md](./seller-portal.md) | Seller / Supplier Portals | Three portals: Procurement (pure retail PO flow), VMI (consignment management), and Internal Admin (approval, config, analytics). |
| [pricing.md](./pricing.md) | Pricing Engine | Calculates final price applying all promotion layers in defined order. Covers caching strategy, debounce, decimal precision, and rounding policy. |
| [payment.md](./payment.md) | Payment Service | Treated as black box — existing service. Covers integration points with OMS, supported payment methods, COD special case, and idempotency. |
| [catalog.md](./catalog.md) | Product Catalog Service | Single source of truth for all product data. Covers parent-child SKU structure, product lifecycle, barcode mapping, search sync, and ATP display strategy. |

---

## Reference Files

| File | Content |
|---|---|
| [mongodb-vs-sql.md](./mongodb-vs-sql.md) | Database decision guide — when to use MongoDB vs SQL, auto-increment ID bottleneck, schema evolution in retail. |
| [edge-cases.md](./edge-cases.md) | 26 edge cases across all subsystems — Stock, OMS, Pricing, WMS, Logistics, POS Offline, Seller Portal. Each with problem statement and fix. |

---

## Architecture Diagram

```
CUSTOMER CHANNELS
┌──────────────┬──────────────┬──────────────┐
│ Physical POS │  Online Web  │  Mobile App  │
└──────┬───────┴──────┬───────┴──────┬───────┘
       └──────────────▼──────────────┘
                 [API Gateway]
                      │
          ┌───────────┴────────────────┐
          ▼                            ▼
    [Storefront]                    [OMS]
    (search, cart,              (saga, routing,
     POS offline)                idempotency)
       │    │    │               │  │  │  │
       │    │    │               │  │  │  └──► [Payment Service]
       │    │    │               │  │  │       (charge / refund)
       │    │    │               │  │  │
       │    │    └──► [Pricing]◄─┘  │  └─────► [Notification Service]
       │    │          Engine       │
       │    │                       │
       ▼    ▼                       ▼
 [Product  [Stock Service] ◄──── [WMS] ◄───────┘
  Catalog]  (ATP, reserve,       (DC + Stores,
  (SKUs,     states, expiry)      pick-pack)
   search                             │
   sync)                         [Logistics]
                                 (last-mile,
                                  tracking,
                                  3PL + fleet)
                                      ▲
                                      │
                              OMS ────┘  (logistics.pickup)

SUPPLIER PORTALS (separate entry point)
┌──────────────┬──────────────┬──────────────┐
│ Procurement  │  VMI Portal  │   Internal   │
│   Portal     │              │    Admin     │
└──────┬───────┴──────┬───────┴──────┬───────┘
       └──────────────▼──────────────┘
              [Portal API Gateway]
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
[Stock Service] [Pricing Engine]  [OMS]
[WMS]           [Product Catalog] [Analytics DB]
```

---

## Key Design Principles

- **Unified Inventory** — one stock truth across all channels and store locations
- **ATP Buffer** — each store reserves stock for walk-in; remainder exposed online with minimum buffer unit regardless of percentage
- **Saga + Event-Driven** — OMS uses RabbitMQ consumers, each owning one state transition, with compensation steps on failure
- **Smart Order Routing** — OMS routes to nearest store with ATP stock for same-day, DC for standard delivery
- **Offline Resilience** — POS works offline, syncs when connection restores, known acceptable risks documented
- **Pure Retail + VMI Hybrid** — platform owns core inventory, select suppliers manage their own consigned stock
- **Three Supplier Portals** — Procurement, VMI, and Internal Admin separated by responsibility
- **Pricing Engine Separate** — single pricing service called by both Storefront and OMS for consistent prices
- **Analytics Separate** — reporting queries never hit production DB

---

## Dependency Order (Build Sequence)

Services depend on each other in this order — design and build from the bottom up:

```
1. Stock Service        ← no upstream dependencies, everything depends on it
2. Product Catalog      ← needed before OMS can validate items
3. OMS                  ← depends on Stock + Catalog + Payment
4. Pricing Engine       ← depends on Catalog + Promotion rules
5. WMS                  ← depends on OMS + Stock
6. Logistics            ← depends on WMS + OMS
7. Storefront           ← depends on Stock + OMS + Pricing + Logistics
8. Seller Portals       ← built last, reads from everything above
```

---

## Missing Services — Must Have for MVP

Services identified as required but not yet fully designed:

| Service | Why Needed | Key Concerns |
|---|---|---|
| **Notification Service** | OMS references it everywhere — customer must know order status | Channels: SMS, push, email, LINE. Triggered by OMS events. |
| **Customer Auth Service** | No login = no cart persistence, no order history, no loyalty | Single profile across app, web, LINE, loyalty card. Saved addresses. |
| **Search Service** | Storefront needs typo-tolerant, multi-language product search | Elasticsearch/OpenSearch. Event-driven sync from Catalog. Parent-level indexing with nested variants. |
| **Analytics / Reporting DB** | Seller Portal needs sales reports. Production DB must not serve heavy queries | Separate DB (ClickHouse / BigQuery). ETL pipeline from production. Hourly or nightly sync. |

---

## Post-MVP Features — Nice to Have

Features the platform works without — add after stable MVP with real transaction data:

| Feature | Value | When to Add |
|---|---|---|
| **Recommendation Engine** | Increase average order value via "frequently bought together" | After 6+ months of transaction history |
| **Loyalty / Points Service** | Customer retention, repeat purchase incentive | After stable user base |
| **Abandoned Cart Emails** | Revenue recovery from incomplete checkouts | After Notification Service is mature |
| **Personalized Promotions** | Target promotions per customer segment | After Loyalty + Analytics are running |
| **Advanced Analytics Dashboard** | Deep business intelligence per store, category, supplier | After Analytics DB is populated |
| **Own Fleet Route Optimization** | Reduce delivery cost in high-density zones | Only if own fleet volume justifies it |

---

## Core Services — Criticality Ranking

| Tier | Service | Why |
|---|---|---|
| **Tier 1 — Platform dies without these** | Stock Service | Everything depends on it. Wrong ATP = oversell = financial loss. |
| | OMS | Brain of all transactions. Saga pattern prevents partial failures. |
| **Tier 2 — Platform works badly without these** | Catalog Service | Foundation of search, pricing, stock, WMS. |
| | Pricing Engine | Wrong price = legal risk. Consistency across channels non-negotiable. |
| **Tier 3 — Fulfillment engine** | WMS + Logistics | Orders created but never fulfilled = same as not working. |
| **Tier 4 — Interface layer** | Storefront + Portals | Replaceable short-term. POS failure → manual. Portal failure → phone/email. |

---

## Key Technology Decisions

| Decision | Choice | Reason |
|---|---|---|
| Primary DB | MongoDB | Schema flexibility for evolving retail rules, nested documents, TTL index |
| Message queue | RabbitMQ | Lower cost than Kafka, fixed known consumers, Saga pattern |
| Search | Elasticsearch / OpenSearch | Typo tolerance, synonyms, multi-language, relevance scoring |
| Cache | Redis + in-memory | Redis for shared cache, in-memory on Pricing Engine to reduce Redis load |
| Money precision | Decimal library (not float64) | Exact arithmetic for 2dp fiat, no migration complexity |
| Analytics | Separate DB | Protect production from heavy aggregation queries |
| Route planning | OR-Tools / Routific | TSP approximation for own fleet, not built from scratch |
