# Payment Service

## What It Is

Payment Service handles all money movement — collecting from customers at checkout and refunding when orders are cancelled or returned.

---

## Design Assumption

The company already operates a retail business and has an existing Payment Service. For this system design, **Payment Service is treated as a black box** — we integrate with it, not design it.

```
OMS → Payment Service: "charge this customer X THB"
Payment Service → OMS: "success" or "failed"

OMS → Payment Service: "refund order Y"
Payment Service → OMS: "refund processed"
```

---

## Payment Methods (Thai Retail Context)

| Method | Type | Notes |
|---|---|---|
| Credit / Debit card | Card | Visa, Mastercard via payment gateway |
| PromptPay (QR) | Bank transfer | Thai QR standard, instant |
| e-Wallet | Digital wallet | TrueMoney, Rabbit LINE Pay, ShopeePay |
| Cash on Delivery (COD) | Pay on delivery | Payment happens at delivery, not checkout |
| Installment | Deferred | 0% installment via bank partnership |
| Loyalty points | Partial payment | Used alongside other methods |

---

## How OMS Integrates

Payment Service is called at two points in the OMS saga:

```
Charge:
  OMS:createOrder → enqueue "payment.charge" {order_id, amount, method}
  Payment Consumer → processes charge
  → success → enqueue "payment.success" {order_id}
  → fail    → enqueue "payment.failed"  {order_id}

Refund:
  OMS:packedFailed or OMS:cancelled → enqueue "payment.refund" {order_id}
  Payment Consumer → processes refund
  → enqueue "payment.refunded" {order_id}
```

---

## COD — Special Case

Cash on Delivery is paid at the customer's door — not at checkout.

```
Checkout:   no charge taken → order proceeds directly to fulfillment
Delivery:   courier collects cash from customer
Confirmed:  Payment Service records payment received
```

OMS flow for COD skips the payment step entirely and proceeds straight to WMS fulfillment after order confirmation.

Failed COD delivery (customer not home, refuses to pay):
```
Logistics marks: FAILED_DELIVERY
OMS cancels order
WMS restocks item
No refund needed — customer was never charged
```

---

## Idempotency

Payment Service must never charge twice for the same order.

OMS passes `order_id` as the idempotency key on every charge request. Payment Service rejects duplicate charge requests for the same order.

---

## Interactions

```
Payment Service
  │
  ├─► OMS ──────── receive charge/refund request, return success/failed result
  ├─► Bank / Gateway ── actual money movement (internal to Payment Service)
  └─► Notification ──── send payment confirmation or refund notice to customer
```
