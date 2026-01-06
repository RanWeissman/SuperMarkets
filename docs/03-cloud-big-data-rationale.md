# Big Data in Cloud — Rationale (What to Highlight)

This document explains *why* this architecture demonstrates Big Data + Cloud competence.

---

## Performance
- **Fast OLTP writes**: short transactions, minimal joins, predictable latency.
- **Async heavy work**: analytics and enrichment are moved off the critical path.
- **Incremental aggregates**: update metrics per event instead of scanning all history.

Key talking points:
- p95 latency for `POST /purchases`
- throughput (events/sec) in the broker
- consumer lag monitoring

---

## Scalability (Cloud)
- Services are **stateless** → scale horizontally behind a load balancer.
- Message broker provides:
  - buffering (handle traffic spikes)
  - fan-out to multiple consumers
  - consumer groups for parallelism
- Storage splits into:
  - OLTP DB (transaction correctness)
  - Analytics store (fast BI queries)
  - Data lake (cheap storage + replay)

---

## Reliability & Data Correctness
- **Transactional outbox** guarantees: if the purchase is committed, an event exists.
- Delivery is **at-least-once**, so consumers must handle duplicates.
- Idempotency in OLTP prevents duplicate purchases when POS retries.

Failure cases covered:
1. POS retries the same purchase → idempotency_key prevents duplicates
2. DB commit succeeds but broker is down → outbox will retry later
3. Consumer crashes mid-processing → reprocessing is safe via event_id dedup

---

## Analytics & BI Value
- Dashboards read precomputed aggregates, not raw events.
- Supports real business questions:
  - top products per branch/day/week
  - loyal customers and retention
  - revenue trends and anomalies
  - peak hours and staffing optimization

---

## Commercial Angle (Data → Value)
Explain how data becomes a competitive advantage:
- smarter inventory decisions
- targeted promotions and bundling
- staffing optimization per branch
- pricing experiments and measurement

---

## What Makes This “Big Data” Even at Small Scale
Even if demo data is small, the design shows:
- event-driven pipeline
- replayable data (lake)
- incremental computation
- decoupled compute and storage
- operational metrics and backpressure


---

## Version 1.0 KPIs (Per Brief)
The dashboard must provide **real-time** access to:
1. **Unique Customers** (total distinct customers across the entire chain)
2. **Loyal Customers** (customers with **≥ 3 purchases**)
3. **Top Products** (top 3 best-selling products of all time; **include ties**, so the list can be > 3)

Implementation tip for “include ties”:
- compute product counts (or revenue) per product, sort descending
- find the 3rd place value, then return all products with value ≥ that threshold
- or use SQL `DENSE_RANK()` and filter `rank <= 3`

