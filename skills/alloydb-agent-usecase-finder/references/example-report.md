# Example report (abridged)

This is a worked example on a sample e-commerce schema, to show the expected depth. It is based on catalog metadata only; scale is assumed to be a mid-size retailer with seasonal sales peaks.

---

## Agentic use cases for `example-project`, analysed 2026-09-24

**Scope:**
- 4 databases: 3 Cloud SQL (2 PostgreSQL 18.1, 1 SQL Server) and 1 AlloyDB
- 20 tables, 104 columns in the main store schema
- Set aside: `spatial_ref_sys` (PostGIS) and `dbo.testtable` (test)

**Basis:** Knowledge Catalog metadata only. The catalog has no column descriptions, profiles or declared joins, so all relationships are inferred from column names.

### Top recommendations

> **1. Per-session shopping assistant** — Best suited case · average 9.3/10
> Answers requests like "waterproof trail shoes, size 10, under $120, arriving Friday" using vector and full-text search over products, live stock (`inventory.quantity − reserved_qty`), reviews, and the nearest warehouse by `warehouses.location`.
> **Why this architecture:** stock changes by the second; every turn runs vector and spatial lookups; load spikes during sales coincide with checkout on the same tables.
> **To build it you need:** embeddings on `products.description`; `inventory.warehouse_id` made required.

> **2. Inventory fan-out agents** — Best suited case · average 8.0/10
> One agent per warehouse or variant flags items about to run out, suggests transfers between nearby warehouses, and drafts reorders.
> **Why this architecture:** hundreds of short, parallel lookup-heavy agents run in a burst after order waves, on the same tables as live ordering.
> **To build it you need:** replace the free-text `warehouse_location` with `warehouse_id`; add reorder points and a stock-movement table.

> **3. Order and payment integrity monitor** — Best suited case · average 7.8/10
> Checks every new order for totals that don't reconcile (`subtotal + tax + shipping_fee ≠ total`), orders with no successful payment, and address or velocity anomalies, before shipping.
> **Why this architecture:** it needs fresh data and must not load production during peaks. It becomes a clear fit at high order volume.
> **To build it you need:** an `order_status_history` table and `orders.updated_at`.

### All candidates

| Rank | Use case | Fit /10 | Value /10 | Readiness /10 | Average /10 | Tier |
|---|---|---|---|---|---|---|
| 1 | Shopping assistant | 10.0 | 10.0 | 8.0 | 9.3 | Best suited case |
| 2 | Inventory fan-out | 10.0 | 8.0 | 6.0 | 8.0 | Best suited case |
| 3 | Order and payment monitor | 7.5 | 10.0 | 6.0 | 7.8 | Best suited case |
| 4 | Cart-abandonment agent | 7.5 | 8.0 | 6.0 | 7.2 | Best suited case |
| 5 | Customer-service copilot | 6.3 | 8.0 | 6.0 | 6.8 | Likely to suit use case |
| 6 | Hotel semantic search | 5.0 | 6.0 | 6.0 | 5.7 | Likely to suit use case |
| 7 | Promotion analyst | 5.0 | 6.0 | 4.0 | 5.0 | Likely to suit use case |

*Not ranked: review insights (fit 1.3). A standard AlloyDB instance or a batch job serves it well.*

Fit tests (freshness / lookups / burst / isolation): shopping assistant 2/2/2/2, inventory fan-out 2/2/2/2, order and payment monitor 2/1/1/2, cart abandonment 2/2/1/1, customer service 2/2/0/1, hotel search 1/2/1/0, promotion analyst 1/1/1/1.

### Data gaps (top two shown)

**Shopping assistant**
- Add an embedding column on `products.description` for search by meaning:
  ```sql
  ALTER TABLE products ADD COLUMN description_embedding vector(768);
  CREATE INDEX ON products USING scann (description_embedding cosine);
  ```
- `inventory.warehouse_id` is nullable, so the join to `warehouses` fails for some rows. Backfill it, then make it required.
- Add a full-text or BM25 index on `products.name` and `products.description`.

**Inventory fan-out**
- Add reorder points per variant and warehouse:
  ```sql
  ALTER TABLE inventory ADD COLUMN reorder_point int, ADD COLUMN lead_time_days int;
  ```
- Add a `stock_movements(id, variant_id, warehouse_id, delta, reason, created_at)` table, so agents reason about trends, not just the current snapshot.
- Add an `agent_recommendations` table. Agents are read-only, so the application writes their proposals there after approval.

### Schema findings

- `promotions` is not linked to anything. Add `orders.promotion_id`, or an `order_promotions` junction table.
- `users.password_hash` is sensitive. Exclude it from agent access with a secure view.
- No table has a column description in the catalog. Adding descriptions is the cheapest way to improve agent SQL accuracy.

### Migration considerations

- **`ecommerce_db`** (Cloud SQL for PostgreSQL 18.1, PostGIS, us-central1): migrate to AlloyDB with Database Migration Service. Confirm AlloyDB supports PostgreSQL 18 and PostGIS before migrating.
- **The hotel tables:** `hotels` is already in AlloyDB in us-west1. Decide whether to consolidate everything in one region.
- **`db1` on SQL Server:** this would need a re-platform to reach AlloyDB. It is a test table, so it is out of scope.

### Recommended first build

- **Use case:** the shopping assistant.
- **Plan:**
  1. Week 1: migrate `ecommerce_db` to AlloyDB, add embeddings, and fix the `warehouse_id` gap.
  2. Week 2: build three MCP tools (`search_products`, `check_availability`, `nearest_stock`) and load-test the assistant against agent nodes during a simulated sale.
- **Preview access:** https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb

*Agent nodes are read-only: agents propose actions, and your application carries them out against the primary database.*
