# SuperMarket Cloud Big-Data Architecture (Overview)

## Goal
Design a supermarket system that **handles high write throughput** (multiple branches and cashiers), remains **cloud-scalable**, and supports **big-data analytics** without slowing down the transactional checkout flow.

This architecture follows a **separation of concerns**:
- **OLTP (Transactional)** for fast, correct, low-latency sales ingestion.
- **OLAP/Analytics (Big Data)** for heavy aggregation, BI, and long-term trends.

---


---

## Scope Constraints (Per Project Brief)
- **3 Branches (Locations)**: all purchases are associated with a `branch_id` selected from a predefined list.
- **10 Products**: products are loaded from `Products_list.csv` and form the selectable catalog.
- **Purchase Limit**: within a single transaction, a customer may buy **only one unit of any specific product**:
  - each product can appear at most once per purchase
  - quantity per line is restricted to `qty = 1`
- **Data Loading**: both `Purchases.csv` and `Products_list.csv` must be loaded into the database.
- **Run via docker-compose**: the entire system must start/stop using `docker-compose up/down`.


## High-Level Components

### 1) Client Applications
- **Cashier App (POS)**  
  Creates sales (receipts) in real-time, needs immediate confirmation.
- **Owner Dashboard (Web)**  
  Reads analytics KPIs (top products, loyal customers, unique customers, trends).

### 2) API Edge
- **Reverse Proxy / Ingress (Nginx / Traefik / Cloud Load Balancer)**
  - TLS termination
  - Routing
  - Rate limiting (optional)
  - Observability headers (request-id)

### 3) Microservices (FastAPI)
- **Purchases Service (Write Owner)**
  - The **only** service that writes to transactional purchase tables.
  - Enforces idempotency and validation.
  - Writes to **OLTP DB** + **Outbox** in a single transaction.
- **Products Service**
  - Product catalog APIs
  - Validation / enrichment for purchases
- **Analytics Service (Read/Compute)**
  - Consumes events
  - Updates precomputed aggregates (read models)
  - Optionally writes raw events to a Data Lake

### 4) Storage
- **OLTP Database (PostgreSQL)**
  - Purchases, purchase_items, cashier/branch references
  - Transactional Outbox table
- **Analytics Store (Warehouse / Read-Optimized DB)**
  - Aggregation tables (daily product sales, customer loyalty metrics, etc.)
  - Could be: Postgres (separate instance) / ClickHouse / BigQuery / Redshift
- **Data Lake (Optional but recommended)**
  - Object storage (S3/GCS/Azure Blob)
  - Stores raw events in partitioned Parquet/JSON for replay and historical queries

### 5) Messaging / Streaming
- **Message Broker**
  - Kafka / RabbitMQ / SQS / PubSub
  - Decouples checkout writes from analytics processing
  - Provides buffering, fan-out, and scalability

### 6) Background Jobs
- **Outbox Publisher**
  - Polls outbox table and publishes events to the broker
  - Marks outbox rows as delivered
- **Batch Loader (Optional)**
  - Imports historical CSV into OLTP or Data Lake
  - Can backfill analytics

---

## Why This Looks Like “Big Data in Cloud”
1. **Fast transactional path** (OLTP) stays small, stable, and scalable.
2. **Event-driven pipeline** enables big-data processing without blocking the checkout.
3. **Incremental aggregations** update metrics continuously instead of rescanning all data.
4. **Replayability** via Data Lake / append-only logs supports backfills and ML later.
5. **Horizontal scaling**: services and consumers scale independently.

---

## Ownership and Boundaries (Very Important)
- Purchases Service **owns** purchase writes (single writer *by domain*, not by a single process).
- Analytics Service **owns** analytics aggregates (read models).
- Products Service owns product catalog data.
- Dashboard reads only from read models (fast queries).

---

## Core Non-Functional Requirements Addressed
- **Performance**: low latency for POS writes; heavy compute moved async.
- **Scalability**: add branches/cashiers by scaling services and consumers.
- **Reliability**: outbox guarantees no “DB saved but event lost”.
- **Data correctness**: idempotency + dedup handles retries/at-least-once delivery.
