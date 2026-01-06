# Core Flows (End-to-End)

This document describes the most important runtime flows:
- **Purchase creation** (mission-critical OLTP)
- **Event publication** (reliable streaming)
- **Analytics aggregation** (big-data style incremental compute)
- **Dashboard queries** (fast reads)

---

## Flow A — Create Purchase (OLTP, synchronous)

**Actors**: Cashier App → Purchases Service → OLTP Postgres


### Required business rules (v1)
- **branch_id** must be one of the predefined 3 branches.
- **user_id** is a `uuid4`:
  - new customer → generate a new uuid4
  - returning customer → select from existing users already known to the system
- **purchase limit**: at most **one unit per product per purchase**
  - reject if the same product appears twice
  - reject if any item has `qty != 1`

### Steps
1. POS sends `POST /purchases` with:
   - `branch_id`, `cashier_id`, `pos_id`
   - `receipt_id` (store’s unique receipt number)
   - `idempotency_key` (UUID from POS)
   - items: `product_id`, `qty`, `unit_price`, etc.
2. Purchases Service validates input:
   - schema validation (Pydantic)
   - basic business rules
   - (optional) product enrichment from Products Service
3. Purchases Service opens a DB transaction and writes:
   - `purchases`
   - `purchase_items`
   - `outbox_events` (event payload)
4. Commit transaction
5. Return `201 Created` + purchase reference

### Why synchronous here?
Checkout must know immediately whether the sale was recorded. This is the OLTP contract.

---

## Flow B — Transactional Outbox to Broker (reliable async publishing)

**Actors**: Outbox Publisher → OLTP Postgres → Broker

### Steps
1. Publisher polls `outbox_events` where `status = NEW`
2. For each event:
   - publish to broker topic/queue (e.g., `purchases.created`)
   - on success: update outbox row to `SENT` with `sent_at`
3. Use retry with exponential backoff
4. Publisher is idempotent (safe to restart)

### Why outbox?
It prevents the classic distributed failure:
- DB write succeeds ✅
- broker publish fails ❌
Without outbox, analytics misses data. Outbox makes delivery reliable.

---

## Flow C — Analytics Consumption and Incremental Aggregation

**Actors**: Analytics Consumer → Broker → Analytics Store (and optional Data Lake)

### Steps
1. Consumer reads events from `purchases.created`
2. Deduplicate by `event_id` (store it in `analytics_event_log` with UNIQUE constraint)
3. Update incremental aggregates, for example:
   - `daily_sales_by_branch(day, branch_id, revenue_sum, items_sum)`
   - `product_sales_daily(day, branch_id, product_id, qty_sum, revenue_sum)`
   - `customer_loyalty(customer_id, branch_id, purchases_count, last_purchase_at)`
4. Optionally write the raw event to the Data Lake (append-only)
5. Commit offsets / ack message after successful update (depends on broker)

### Big-data angle
Incremental updates avoid full rescans and allow near-real-time dashboards.

---

## Flow D — Dashboard Queries (read-optimized)
**Actors**: Owner Dashboard → Analytics Service → Analytics Store

The dashboard reads only “precomputed” tables, e.g.:
- Top products by branch/day/week
- Loyal customers by branch
- Unique customers per day
- Revenue trends over time

This makes queries fast and cheap even with large history.

---

## Sequence Diagram (Mermaid)

```mermaid
sequenceDiagram
  participant POS as Cashier App (POS)
  participant P as Purchases Service
  participant DB as OLTP Postgres
  participant OB as Outbox Publisher
  participant MQ as Broker (Kafka/RabbitMQ)
  participant A as Analytics Consumer
  participant W as Analytics Store

  POS->>P: POST /purchases (idempotency_key, receipt_id, items)
  P->>DB: BEGIN TX
INSERT purchases + items
INSERT outbox_event
  DB-->>P: COMMIT OK
  P-->>POS: 201 Created (purchase_id)

  OB->>DB: SELECT outbox_events WHERE status=NEW
  OB->>MQ: Publish PurchaseCreated(event_id)
  OB->>DB: UPDATE outbox_events SET status=SENT

  A->>MQ: Consume PurchaseCreated
  A->>W: UPSERT aggregates (incremental)
  A-->>MQ: ACK / Commit offset
```
