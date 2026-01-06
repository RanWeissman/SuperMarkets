# Data Model & Contracts (OLTP + Events + Analytics)

This document defines the minimal tables and message contracts required to support:
- correctness (idempotency, retries)
- event-driven analytics
- scalable read models

---

## OLTP Tables (PostgreSQL)


### customers (minimal)
- `customer_id` (PK, UUID)
- `created_at` (timestamp)

This table supports the “returning customer selection” requirement in Application A.


### purchases
- `purchase_id` (PK, UUID)
- `receipt_id` (string/int, UNIQUE per branch/pos/day as needed)
- `idempotency_key` (UUID, UNIQUE)
- `branch_id`
- `cashier_id`
- `pos_id`
- `purchase_time` (timestamp)
- `total_amount` (numeric)

**Indexes**
- `(branch_id, purchase_time)`
- `(purchase_time)` for partitioning by date

### purchase_items
- `purchase_item_id` (PK)
- `purchase_id` (FK -> purchases)
- `product_id`
- `qty`
- `unit_price`
- `line_total`

**Indexes**
- `(purchase_id)`
- `(product_id, purchase_id)` for product-level analytics joins

---

## Transactional Outbox Table

### outbox_events
- `event_id` (PK, UUID)
- `event_type` (e.g., `PurchaseCreated`)
- `aggregate_id` (purchase_id)
- `occurred_at` (timestamp)
- `payload_json` (jsonb)
- `status` (`NEW`, `SENT`, `FAILED`)
- `attempts` (int)
- `last_error` (text)
- `sent_at` (timestamp nullable)

**Why jsonb?** flexible schema evolution and easy reprocessing.

---

## Event Contract (PurchaseCreated)

Topic/Queue: `purchases.created`

Minimal payload:
```json
{
  "event_id": "uuid",
  "event_type": "PurchaseCreated",
  "occurred_at": "2026-01-06T10:15:00Z",
  "purchase": {
    "purchase_id": "uuid",
    "receipt_id": "123456",
    "idempotency_key": "uuid",
    "branch_id": 1,
    "cashier_id": 17,
    "pos_id": "POS-3",
    "purchase_time": "2026-01-06T10:14:59Z",
    "total_amount": 142.50,
    "items": [
      {"product_id": 1001, "qty": 2, "unit_price": 8.9},
      {"product_id": 2007, "qty": 1, "unit_price": 124.7}
    ]
  }
}
```

### Delivery semantics
- Broker is typically **at-least-once**
- Consumers must be **idempotent**:
  - store processed `event_id` in a table with UNIQUE constraint
  - ignore duplicates safely

---

## Analytics Store (Read Models)

### analytics_event_log
- `event_id` (PK, UNIQUE)
- `processed_at`
- `event_type`

### daily_sales_by_branch
- `day` (date)
- `branch_id`
- `revenue_sum`
- `items_sum`
- PK: `(day, branch_id)`

### product_sales_daily
- `day` (date)
- `branch_id`
- `product_id`
- `qty_sum`
- `revenue_sum`
- PK: `(day, branch_id, product_id)`

### customer_loyalty (optional)
- `customer_id`
- `branch_id`
- `purchases_count`
- `last_purchase_at`
- PK: `(customer_id, branch_id)`

---

## Partitioning / Big-Data Readiness
For large volumes, partition `purchases` and `purchase_items` by time:
- monthly partitions by `purchase_time`
- keep indexes small and queries fast
- allows efficient retention policies



### Enforcing the “one unit per product per transaction” rule
Recommended enforcement (defense in depth):
1. **API validation** in Purchases Service:
   - ensure each `product_id` appears at most once in `items`
   - ensure `qty == 1` for every line
2. **Database constraints**:
   - `UNIQUE (purchase_id, product_id)` on `purchase_items`
   - `CHECK (qty = 1)` on `purchase_items`

