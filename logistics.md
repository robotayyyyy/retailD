# Logistics Service

## What It Is

Manages the **movement of parcels** from fulfillment nodes (DC or stores) to the customer's door.

Unlike every other service — once the parcel leaves your building, you control nothing. A third-party courier is responsible. Anything can happen.

```
Stock Service  → your database, you control it
OMS            → your code, you control it
WMS            → your warehouse, your staff, you control it
Logistics      → parcel is with a courier, you control nothing
```

---

## Who Solves What

**Business decides the rules:**
```
If nobody home after 3 attempts → what do we do?
How long do we hold a failed parcel before returning?
Do we refund before or after return?
Who pays for re-delivery — us or customer?
If courier loses parcel — who compensates customer?
Do we trust customer or courier in a dispute?
```

**Engineering translates rules into flows:**
```
Takes those decisions
→ models them as states and flows
→ builds the system to handle each scenario
```

Engineer did not decide the 3 attempts rule. Business did. Engineer implements it faithfully.

### The Designer's Role

You are the bridge between business and engineering:
- Understand business rules deeply enough to ask the right questions
- Translate answers into states, flows, edge cases
- Flag scenarios business hasn't thought of yet

```
Business: "handle failed delivery"
You:      "define failed — nobody home? wrong address?
           damaged? all three are different flows"
Business: "oh... let me think"
```

---

## Delivery Methods

| Method | Origin | Speed | Use Case |
|---|---|---|---|
| Standard Delivery | Central DC | 1–3 days | General online orders |
| Same-Day Delivery | Nearest store | Same day | Urban, dense area |
| Click & Collect | Any store | Customer chooses | Customer picks up |
| Express / Next-Day | DC or store | Next day | Premium customers |
| In-store Purchase | Physical store | Immediate | Walk-in |

---

## Courier Integrations

Not one courier — many. Each has its own API format, authentication, tracking event names, and failure codes.

| Provider | Type | Use Case |
|---|---|---|
| Kerry Express | 3PL | Standard, next-day |
| Flash Express | 3PL | Standard, affordable |
| J&T Express | 3PL | Standard |
| Grab / LINE MAN | On-demand | Same-day, hyperlocal |
| Own Fleet | Internal | High-density zones |

**Courier selection logic:**
- Cheapest rate for standard delivery
- Fastest available for same-day
- Customer-selected courier for premium tiers

---

## Shipment Lifecycle

```
ORDER_READY (WMS hands off parcel)
  │ courier pickup
  ▼
PICKED_UP
  │ in transit
  ▼
IN_TRANSIT
  │ out for delivery
  ▼
OUT_FOR_DELIVERY
  │ delivered successfully
  ▼
DELIVERED
  │
  └──► FAILED_DELIVERY (nobody home, wrong address, access denied)
             │
             ├─► Retry delivery (attempt 2, attempt 3)
             └─► Return to Sender (RTS) after max retries
```

---

## Hard Problem 1 — Last Mile

Last mile = the final leg from local courier hub to customer's door.

```
Courier picks up parcel from DC
  → drives to sorting hub
  → sorted by zone
  → assigned to delivery driver
  → driver loads van with 50-100 parcels
  → drives around neighbourhood delivering one by one
```

Driver has 100 parcels to deliver in one day. Every failed attempt costs time and money.

### Failure Scenarios

**Nobody home:**
```
Driver arrives at 2pm, customer is at work
Nobody answers, cannot leave parcel outside (theft risk)
→ leave notification slip
→ schedule retry next day
→ notify customer: "missed delivery, rescheduling"
```

**Wrong address:**
```
Customer typed "Sukhumvit 21" but meant "Sukhumvit 12"
Driver goes to wrong building
→ driver flags address issue
→ customer support contacts customer to correct
→ reschedule after correction
```

**Building access denied:**
```
Condo with security gate, resident doesn't pick up phone
→ contact customer to arrange access
→ or redirect to nearest pickup point
```

**Condo mailroom vs door delivery:**
```
Some condos: leave at mailroom → done
Some condos: must deliver to door → resident must be home
Different buildings = different rules → no universal solution
```

### Business Must Define Per Scenario
- What triggers a retry vs immediate RTS?
- Who contacts the customer — courier or platform?
- Is pickup point redirect allowed? Who pays?

---

## Hard Problem 2 — Failed Delivery

### Typical Retry Flow

```
Attempt 1 fails
  → auto schedule retry next day
  → notify customer

Attempt 2 fails
  → notify customer, ask to reschedule or change address
  → wait for customer response (hold period)

Attempt 3 fails
  → Return to Sender (RTS)
  → notify customer: "parcel returning, refund incoming"
  → OMS triggers refund
  → WMS receives parcel back, inspects, restocks
```

### Business Must Answer Before Engineering Can Build

```
1. How many retry attempts? (usually 2-3)
2. How long between retries? (next day? 2 days?)
3. During hold — can customer change address?
4. Who pays for re-delivery after customer fault?
5. How long before RTS if customer never responds?
6. Refund before or after parcel returns to DC?
```

### Address Change After Shipping

```
Order shipped, parcel is in transit
Customer: "I moved, please deliver to new address"
```

```
If parcel not yet at local hub → possible to redirect (courier dependent)
If driver already loaded the van → too late
```

Engineering needs to know:
- At what transit stage is address change still possible?
- Which couriers support mid-transit redirect?
- If not possible → what does the customer see?

---

## Hard Problem 3 — Tracking

Customer opens app: "where is my order?"

You need one consistent answer. Parcel could be with Kerry, Grab, Flash, or own fleet — each sends different data in different formats.

### The Normalization Problem

```
Kerry webhook:
  { "waybill": "KE123", "status": "IN_TRANSIT", "location": "Bangkok Hub" }

Grab webhook:
  { "order_id": "GR456", "state": "PICKED_UP", "driver": "Somchai" }

Flash polling response:
  { "tracking_no": "FL789", "code": 3, "desc": "On the way" }
```

Your system normalizes all of this into one format:

```
{
  tracking_no: "KE123",
  courier: "kerry",
  status: "IN_TRANSIT",
  location: "Bangkok Hub",
  updated_at: "2024-05-01T14:00:00Z"
}
```

One format regardless of courier. Storefront reads this — never touches courier APIs directly.

### Webhook vs Polling

```
Webhook (push):
  Courier sends event to your endpoint when something happens
  → real time, efficient, preferred

Polling (pull):
  Your system asks courier API every N minutes: "any updates?"
  → delayed, higher API cost
  → some couriers only support this depending on contract tier
```

Engineering must handle both mechanisms simultaneously across different couriers.

### The "Delivered But Not Received" Problem

```
Courier system: status = DELIVERED
Customer:       "I never got it"
```

Possible reasons:
```
Driver scanned as delivered before actually delivering (shortcut)
Parcel left at wrong door
Stolen from doorstep after delivery
Delivered to neighbor without permission
Customer disputing to get free refund
```

Business must decide:
```
Do we trust customer or courier by default?
Do we require photo proof of delivery from couriers?
Do we refund immediately and investigate after?
Do we open a dispute with the courier?
```

### Photo Proof of Delivery

Modern couriers support driver taking photo at customer door:

```
Driver takes photo of parcel at customer door
Photo uploaded to courier system
Available via courier API
Your system stores photo URL
```

If customer disputes → show the photo. Reduces false claims significantly.

---

## Ship-From-Store Flow

```
OMS routes to Store X (nearest, has ATP stock)
  │
  ▼
WMS (Store X): staff picks item from shelf
  │
  ▼
Packs into bag → prints label
  │
  ▼
Logistics: on-demand courier dispatched (Grab / LINE MAN)
  │
  ▼
Courier picks up at store → delivers within hours
```

---

## Click & Collect Flow

```
OMS routes to chosen store
  │
  ▼
WMS (Store): reserve item in hold area
  │
  ▼
Logistics: send "Ready for pickup" notification to customer
  │
  ▼
Customer arrives → shows QR code at counter
  │
  ▼
Staff scans QR → hands over item → COLLECTED
```

**Hold period:** item held for 3 days. If not collected → order cancelled, item restocked, customer refunded.

---

## Shipping Fee Calculation

Inputs:
- Origin node (DC or store location)
- Destination (customer address)
- Parcel weight and dimensions
- Delivery method selected
- Courier rate card

Output: shipping fee shown at checkout

---

## Key Data

| Entity | Description |
|---|---|
| Shipment | Shipment ID, order ID, origin node, destination, courier |
| TrackingEvent | Status, timestamp, location, courier raw data, normalized status |
| CourierRateCard | Price matrix per zone, weight, courier |
| ClickCollectHold | Store, order, expiry time, QR code |
| DeliveryAttempt | Attempt number, outcome, driver note, timestamp |
| ProofOfDelivery | Photo URL, signature, timestamp |

---

## Why Logistics is Complex — Summary

| Problem | Core Difficulty |
|---|---|
| Last mile | Physical world unpredictability — humans, buildings, access |
| Failed delivery | Multi-step retry policy, address changes, RTS flow |
| Tracking | Multiple couriers, different formats, webhook vs polling, dispute handling |

The engineering is not the hard part. **Getting the business rules right is the hard part.** Engineering faithfully implements whatever rules business defines — but must ask the right questions first.

---

## Delivery Route Planning (Own Fleet Only)

When the business operates its own delivery fleet, route planning becomes necessary.

This is the **Travelling Salesman Problem (TSP)** — given N delivery addresses and M drivers, find the optimal route per driver to minimize total distance and time.

TSP is NP-hard — no perfect solution exists at scale. Real companies use approximation tools, not custom algorithms.

### Do Not Build From Scratch

Use an existing routing tool:

| Tool | Type | Notes |
|---|---|---|
| Google OR-Tools | Open source library | Very capable, free |
| Google Maps Route Optimization API | Cloud API | Pay per use |
| Routific | SaaS | Easier to integrate |
| OptimoRoute | SaaS | Good for last-mile |

Feed the tool:
```
- List of delivery addresses for the day
- Driver starting locations
- Time windows per delivery (if any)
- Vehicle capacity per driver
```

Get back:
```
- Optimized route per driver
- Estimated arrival time per stop
- Total distance per driver
```

### How Logistics Service Uses It

```
Parcel assigned to own fleet
  → send to Route Planning Service
  → Route Planning Service calls routing tool
  → returns: driver_id + ordered list of stops
  → assign parcels to driver
  → driver app shows route
```

Route Planning is a **separate subsystem** — logistics service calls it as a dependency. Only activated when own fleet is used.

### 3PL vs Own Fleet

```
3PL couriers (Kerry, Flash, Grab):
  → they plan their own routes
  → not your problem
  → you just create shipment via API, receive tracking events

Own fleet:
  → you plan the routes
  → use routing tool, not custom TSP algorithm
  → own fleet only justified in very high density zones
```

---

## Interactions

```
Logistics
  │
  ├─► WMS ──────────── receive parcel handoff, confirm pickup
  ├─► OMS ──────────── update shipment status (SHIPPED, DELIVERED, RTS)
  ├─► Storefront ────── expose tracking status to customer
  ├─► Notification ───── send tracking updates via SMS/push/email
  └─► 3PL Couriers ───── create shipment, receive webhook / poll for tracking
```
