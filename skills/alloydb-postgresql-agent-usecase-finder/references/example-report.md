# Example report (abridged)

This is a worked example on a sample e-commerce schema, to show the expected depth. It is based on catalog metadata only; scale is assumed to be a mid-size retailer with seasonal flash-sale peaks.

---

## Agentic use cases for `example-project`, analysed 2026-09-24

**Scope:**
- 4 operational databases: 3 Cloud SQL (2 PostgreSQL 18.1, 1 SQL Server) and 1 AlloyDB (plus 1 BigQuery analytics dataset available for lakehouse federation)
- 20 operational tables, 104 columns in the main store schema
- Set aside: `spatial_ref_sys` (PostGIS extension table) and `dbo.testtable` (scratch/test table)

**Basis:** Knowledge Catalog metadata only (no row data queried). The catalog has no column descriptions, profiles, or declared joins, so all relationships are inferred from column names and types.

### Top recommendations

> **1. Per-session shopping assistant** — Best suited case · average 9.3/10
> Answers requests like "waterproof trail shoes, size 10, under $120, arriving Friday" using hybrid vector (`scann`) and BM25/full-text search over products, live stock (`inventory.quantity − reserved_qty`), reviews, and nearest-warehouse distance via `warehouses.location`.
> **Why this architecture:** stock changes by the second (~1s replication lag); every turn runs multi-index vector, lexical, and spatial lookups that need sub-millisecond Colossus I/O even when new nodes spin up cold; flash-sale traffic bursts scale from zero to hundreds of agent nodes while microVM and dedicated Colossus segment isolation protect live checkout on the primary cluster.
> **To build it you need:** embeddings and a `scann` index on `products.description`; a `bm25` or `GIN` full-text index; `inventory.warehouse_id` backfilled and made `NOT NULL`.

> **2. Inventory fan-out agents** — Best suited case · average 8.0/10
> One agent per warehouse or SKU variant launches in a parallel wave to flag items at risk of stockout, suggest inter-warehouse transfers by distance, and draft supplier reorders.
> **Why this architecture:** hundreds of short-lived, parallel lookup-and-aggregation agents spin up in seconds after order waves and scale back to zero when done, reading the same `inventory` and `order_items` tables as live checkout without buffer-cache or lock contention on the primary instance.
> **To build it you need:** replace the free-text `warehouse_location` with a required `warehouse_id`; add reorder points, a `stock_movements` ledger table, and an `agent_recommendations` table on the primary database.

> **3. Order and payment integrity monitor** — Best suited case · average 7.8/10
> Checks every new order within seconds for totals that don't reconcile (`subtotal + tax + shipping_fee ≠ total`), orders without a matching payment, and velocity or address anomalies before fulfilment.
> **Why this architecture:** requires ~1-second freshness on `orders` and `payments` and strict production isolation so continuous verification bursts during peak checkout never add latency to payment commits.
> **To build it you need:** an `order_status_history` table and `orders.updated_at`.

### All candidates

| Rank | Use case | Database(s) | Fit /10 | Value /10 | Readiness /10 | Average /10 | Tier |
|---|---|---|---|---|---|---|---|
| 1 | Per-session shopping assistant | `ecommerce_db` | 10.0 | 10.0 | 8.0 | 9.3 | Best suited case |
| 2 | Inventory fan-out agents | `ecommerce_db` | 10.0 | 8.0 | 6.0 | 8.0 | Best suited case |
| 3 | Order and payment integrity monitor | `ecommerce_db` | 7.5 | 10.0 | 6.0 | 7.8 | Best suited case |
| 4 | Cart-abandonment agent | `ecommerce_db` | 7.5 | 8.0 | 6.0 | 7.2 | Best suited case |
| 5 | Customer-service copilot | `ecommerce_db` | 6.3 | 8.0 | 6.0 | 6.8 | Likely to suit use case |
| 6 | Hotel semantic search | `hotels_db` (AlloyDB) | 5.0 | 6.0 | 6.0 | 5.7 | Likely to suit use case |
| 7 | Promotion analyst (with BigQuery federation) | `ecommerce_db` + BigQuery | 5.0 | 6.0 | 4.0 | 5.0 | Likely to suit use case |

*Not ranked: review insights (fit 1.3). A standard AlloyDB instance, read pool, or BigQuery batch job serves it well.*

Fit tests (freshness / lookups / burst / isolation): shopping assistant 2/2/2/2, inventory fan-out 2/2/2/2, order and payment monitor 2/1/1/2, cart abandonment 2/2/1/1, customer service 2/2/0/1, hotel search 1/2/1/0, promotion analyst 1/1/1/1.

### Data gaps (top two shown)

**1. Per-session shopping assistant**
- Add an embedding column and `scann` index on `products.description` for semantic search (using AlloyDB's `google_ml_integration` automated embeddings or application-populated vectors):
  ```sql
  ALTER TABLE products
    ADD COLUMN description_embedding vector(768)
    GENERATED ALWAYS AS (embedding('gemini-embedding-001', description)) STORED;
  CREATE INDEX ON products USING scann (description_embedding cosine);
  ```
- Add a full-text or BM25 index on `products.name` and `products.description` for hybrid keyword + vector search:
  ```sql
  CREATE INDEX products_search_idx ON products
    USING GIN (to_tsvector('english', coalesce(name, '') || ' ' || coalesce(description, '')));
  ```
- `inventory.warehouse_id` is currently nullable, so joining `inventory` to `warehouses` drops rows with missing IDs. Backfill from `warehouse_location`, add a foreign key, and set `NOT NULL`:
  ```sql
  ALTER TABLE inventory
    ALTER COLUMN warehouse_id SET NOT NULL,
    ADD CONSTRAINT inventory_warehouse_id_fkey FOREIGN KEY (warehouse_id) REFERENCES warehouses(id);
  ```

**2. Inventory fan-out agents**
- Add reorder thresholds and lead times per variant and warehouse:
  ```sql
  ALTER TABLE inventory ADD COLUMN reorder_point int, ADD COLUMN lead_time_days int;
  ```
- Add a `stock_movements(id uuid PRIMARY KEY, variant_id uuid, warehouse_id uuid, delta int, reason text, created_at timestamptz)` table so agents reason about depletion velocity rather than a static snapshot.
- Add an `agent_recommendations` table on the primary database so the application can persist and approve agent-proposed transfers and purchase orders:
  ```sql
  CREATE TABLE agent_recommendations (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    agent_name text NOT NULL,
    entity_ref text NOT NULL,
    proposed_action text NOT NULL,
    payload jsonb NOT NULL,
    status text NOT NULL DEFAULT 'pending_approval',
    created_at timestamptz NOT NULL DEFAULT now(),
    approved_by text
  );
  ```

### Schema findings

- `promotions` is not linked to any order table. Add `orders.promotion_id` referencing `promotions(id)`, or an `order_promotions` junction table.
- `users.password_hash` is sensitive. Exclude it from agent roles using a parameterized secure view or column-level privileges.
- No table has column descriptions in Knowledge Catalog. Populating descriptions in the catalog is the highest-leverage way to improve agent SQL accuracy.

### Migration considerations

- **`ecommerce_db`** (Cloud SQL for PostgreSQL 18.1, `postgis`, `us-central1`): migrate to AlloyDB using Database Migration Service (DMS). Confirm AlloyDB's current support for PostgreSQL 18 and `postgis` in the official documentation before migrating.
- **The hotel tables (`hotels_db`):** already on AlloyDB in `us-west1`. Decide whether to consolidate databases in `us-central1` if a single agent needs low-latency access to both.
- **`db1` on Cloud SQL for SQL Server:** would require a heterogeneous re-platform to reach AlloyDB. Since it only holds `dbo.testtable`, it is out of scope.

### Recommended first build

- **Use case:** Per-session shopping assistant.
- **Why start here:** Highest combined Architecture Fit (10.0) and Business Value (10.0), and all core tables already exist (`Readiness 8.0/10`—only additive embedding/index columns and a `warehouse_id` constraint are required).
- **Two-week plan:**
  1. **Week 1:** Migrate `ecommerce_db` to AlloyDB with Database Migration Service, add `description_embedding` (`scann`) and full-text/BM25 indexes on `products`, and enforce `NOT NULL` + foreign key on `inventory.warehouse_id`.
  2. **Week 2:** Expose three read-only MCP tools (`search_products`, `check_availability`, `nearest_stock`) backed by AlloyDB agent nodes and load-test concurrent shopper sessions during a simulated flash sale.
- **Documentation:** https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb
- **Request Preview access:** https://docs.google.com/forms/d/e/1FAIpQLSfYv_zv2CI9L6xZxkExZai_jG-eiz8iEYPfwLFwaIatZdYCrA/viewform

*Agent nodes are read-only: agents read and reason on isolated agent nodes, and your application carries out approved write actions against the primary database.*
