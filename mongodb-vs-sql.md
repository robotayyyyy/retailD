# MongoDB vs SQL — Design Discussion

## Context

This discussion came from designing the retail platform database choices.
Key concerns: write latency, schema evolution, ID generation, high frequency scenarios.

---

## The "SQL Slows at 100k Rows" Rumor

**Partially true, mostly misunderstood.**

SQL does not slow down because of row count. It slows down when:

```
No index on queried column
→ full table scan = reads every row
→ 100k rows = 100k reads = slow
```

With proper index:
```
SELECT * WHERE key = 'abc'
→ index lookup = O(log n)
→ 100k, 1M, 10M rows → still fast
```

The rumor comes from missing indexes — not a SQL problem, a usage problem.

---

## Auto Increment ID — The Real SQL Bottleneck at High Frequency

Auto increment ID is a **global counter** — only one thread can increment at a time.

```
INSERT order → database generates next ID
INSERT order → wait for previous ID to finish
INSERT order → wait...
```

At high frequency (e.g. crypto exchange, thousands of orders/sec):
```
Every insert locks the counter → queue builds → latency spikes
```

**Fix for SQL: use UUID instead**
```sql
id UUID DEFAULT gen_random_uuid() PRIMARY KEY
```
UUID generated without global lock. Inserts run in parallel.

**MongoDB ObjectID: generated client-side**
```
timestamp + machine ID + process ID + random counter
= unique without asking the database at all
```
No lock, no wait, no coordination. Thousands of clients generate IDs simultaneously.

---

## SQL Latency Sources Beyond ID

UUID fixes the ID contention but SQL has other latency sources at high write frequency:

**Write-Ahead Log (WAL)**
```
Every insert → write to WAL first → then write to table
→ durability guarantee, but extra I/O on every write
```

**MVCC (row versioning)**
```
Every update → keep old version + write new version
→ old versions pile up → autovacuum runs to clean
→ autovacuum competes with your writes → latency spikes
```

**Row-level locking**
```
Two updates on same row → one waits
At high frequency → many rows → many waits → queue builds
```

SQL is built for **consistency and correctness** — those guarantees have a CPU and I/O cost that shows up at high write frequency.

---

## Schema Evolution — Where MongoDB Wins for Retail

Business rules in retail change constantly. New fields appear every sprint.

### Real Retail Examples

**Promotions keep evolving:**
```
Month 1:  { discount_amount: 50 }
Month 3:  { discount_amount: 50, coupon_code: "SAVE50" }
Month 6:  { discount_amount: 50, coupon_code: "SAVE50", loyalty_points_used: 200 }
Month 9:  { discount_amount: 50, bundle_deal_id: "BD-001" }
```

**Delivery options keep expanding:**
```
Q1: { fulfillment_type: "standard" }
Q2: { fulfillment_type: "same_day", delivery_slot: "14:00-16:00" }
Q3: { fulfillment_type: "click_collect", pickup_store_id: "store-silom", pickup_by: "2024-05-01" }
Q4: { fulfillment_type: "locker", locker_id: "LK-007", locker_pin: "4521" }
```

**Seller requirements keep growing:**
```
Day 1:    { seller_id: "S001", seller_name: "Nestle" }
Later:    { seller_id: "S001", tier: "gold", sla_hours: 24, penalty_rate: 0.05 }
Later:    { seller_id: "S002", country: "CN", customs_code: "HS-123", import_duty_rate: 0.07 }
```

### What Happens in SQL Each Time
```sql
ALTER TABLE orders ADD COLUMN delivery_slot VARCHAR;
ALTER TABLE orders ADD COLUMN pickup_store_id VARCHAR;
ALTER TABLE orders ADD COLUMN locker_id VARCHAR;
ALTER TABLE orders ADD COLUMN locker_pin VARCHAR;
ALTER TABLE orders ADD COLUMN bundle_deal_id VARCHAR;
ALTER TABLE orders ADD COLUMN loyalty_points_used INT;
```
- Table grows wider every quarter
- Most columns are NULL for most rows
- Schema migration needed every time a new feature ships
- Risk of table lock on large tables during ALTER

### What Happens in MongoDB
```
Just add the field to new documents.
Old documents stay as they are.
No migration. No downtime. No schema meeting.
```

---

## Summary Comparison

| Concern | SQL (Postgres) | MongoDB |
|---|---|---|
| Write latency at high freq | WAL + MVCC + locking overhead | Lower — simpler write path |
| ID generation | Auto increment = global lock (use UUID to fix) | ObjectID client-side, no lock |
| Schema evolution | ALTER TABLE = risky on large tables | Just add field to new docs |
| Horizontal scale | Hard — vertical scaling preferred | Built-in sharding |
| Transactions | Battle-tested, full ACID | Less mature (improving) |
| Complex joins / reporting | Excellent | Awkward |
| Financial accuracy | Preferred | Possible but harder |

---

## When to Use Each

**Use MongoDB when:**
- Document structure (orders with nested items, addresses)
- Schema changes frequently (retail, e-commerce, fast-moving product)
- High write volume with simple access patterns
- Horizontal scale needed

**Use SQL when:**
- Complex relationships requiring joins
- Financial records requiring strict ACID (accounting, payouts, reconciliation)
- Reporting across many entities
- Team is more experienced with SQL

---

## For This Retail Platform

MongoDB is the right default because:
- Order documents are naturally nested
- Business rules change every sprint — new fields appear constantly
- No complex cross-entity joins needed (services own their data)
- TTL index built-in for idempotency keys
- Team preference leans MongoDB

SQL remains appropriate for: **finance, payout, and accounting** subsystems where strict ACID and auditability matter most.
