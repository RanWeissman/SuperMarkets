# Deployment, Operations & Observability

This document describes how to run and operate the system in a cloud-ready way.

---

## Deployment Model
Recommended container setup (run with **docker-compose**):
- `purchases-service` (FastAPI)
- `products-service` (FastAPI)
- `analytics-service` (FastAPI)
- `outbox-publisher` (worker)
- `broker` (Kafka/RabbitMQ)
- `postgres-oltp`
- `analytics-store` (optional separate DB)
- `dashboard` (optional UI)
- `prometheus + grafana` (optional) or cloud monitoring

---

## Scaling Strategy
### OLTP / Purchases
- scale purchases-service horizontally
- use DB connection pooling
- keep transactions short
- partition tables by time for large volumes

### Broker / Consumers
- scale consumers using consumer groups
- monitor consumer lag
- apply backpressure and retries

### Analytics store
- separate from OLTP to avoid contention
- use denormalized tables / materialized views
- partition by date for large datasets

---

## Observability
### Metrics (examples)
- Request latency: p50/p95/p99 for `POST /purchases`
- Error rate by route
- Outbox backlog: count of `NEW` events
- Publish success/failure rate
- Consumer lag / queue depth
- Analytics update duration

### Logging
- structured logs (JSON)
- include correlation ids:
  - `request_id`
  - `purchase_id`
  - `event_id`

### Health Checks
- `/health` per service:
  - DB connectivity
  - broker connectivity (where relevant)

---

## Security (Minimal Cloud Best Practice)
- JWT/session for dashboard access
- branch/cashier authentication (API keys / JWT)
- network segmentation: OLTP DB not public
- secret management via env vars / vault

---

## Data Governance / Quality (Optional but strong)
- schema versioning for events (`schema_version`)
- data validation at ingestion (Pydantic)
- dead-letter queue for poison messages
- retention policies:
  - OLTP partitions: keep N months
  - data lake: long-term archive
