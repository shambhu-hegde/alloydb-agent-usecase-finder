# Use-case pattern library

Each pattern lists:
- **Signals:** the schema evidence that makes the pattern applicable
- **Agent:** what the agent does
- **Core data:** the tables it can't work without
- **Typical gaps:** the columns, indexes, or tables it usually needs added
- **Fit profile:** how it tends to score on the four fit tests (freshness / fast lookups / burst / isolation). This is a starting point; adjust it for the customer's actual schema and scale.

Match signals loosely. Names vary, for example `customer` / `users` / `accounts`, or `sku` / `variant` / `item`.

---

## Commerce and retail

### P1. Per-session shopping or booking assistant
- **Signals:** a product, listing, or room table with text such as `name`, `description`, or `title`; a stock or availability table (`quantity`, `available`, `reserved_*`); optionally reviews and a location column (`geography`, `geometry`, `lat`/`lng`).
- **Agent:** one agent per shopper conversation. It combines vector (`scann`) and BM25/full-text search with live availability, price, and delivery or spatial distance (`PostGIS`) filters.
- **Core data:** catalog tables and stock or availability.
- **Typical gaps:**
  - an embedding column (`vector`) + `scann` index on product or description text (or automated in-database embeddings via `google_ml_integration`)
  - a `bm25` or `GIN` full-text index
  - `updated_at` on stock
  - a `geography(Point, 4326)` location column + `GIST` index on the fulfillment site
  - a published price, or an effective-price view
- **Fit profile:** 2/2/2/2. This is the flagship case. Load spikes during flash sales or launches coincide with peak checkout on the same tables, requiring sub-millisecond cold-read I/O and strict primary isolation.

### P2. Inventory and fulfillment fan-out agents
- **Signals:** a stock table keyed by item and site; a warehouse, store, or site table with a location column; order line items.
- **Agent:** one agent per site, category, or SKU, launched as a parallel burst. It finds items at risk of running out, suggests transfers between nearby sites, drafts reorders, and picks the optimal site to ship from (combining point lookups with fast columnar velocity aggregations).
- **Core data:** stock, sites, and order lines.
- **Typical gaps:**
  - a real ID link from stock to site (`warehouse_id`), instead of a free-text location
  - reorder point and lead-time columns
  - a supplier table
  - a stock-movement history table
- **Fit profile:** 2/2/2/2 once there are dozens of sites or thousands of items. Otherwise 2/2/1/1.

### P3. Cart-abandonment and next-best-action agent
- **Signals:** cart or session tables with `added_at`; users; product variants.
- **Agent:** spots carts about to be abandoned and checks live stock and price changes. It then drafts a personalized nudge message; your application sends it after policy approval.
- **Core data:** cart, users, and variants.
- **Typical gaps:**
  - `updated_at` on cart
  - a cart status column
  - a table recording which nudges were sent to whom
  - marketing consent flags on users
- **Fit profile:** 2/2/1/1.

### P4. Promotion and pricing analyst
- **Signals:** a promotions, coupons, or discounts table; orders; order lines.
- **Agent:** measures how promotions perform in real time, flags codes being abused, and simulates price changes, combining the AlloyDB columnar engine with BigQuery historical data via lakehouse federation.
- **Core data:** promotions and orders.
- **Typical gaps:**
  - a promotion link on orders or order lines (`orders.promotion_id` or `order_promotions`). This gap is very common.
  - a discount amount on order lines
  - price history
- **Fit profile:** 1/1/1/1. A standard read pool or BigQuery usually suffices unless real-time promo-abuse blocking runs on every checkout wave.

---

## Operations, risk and support

### P5. Real-time order and payment integrity monitor
- **Signals:** an orders or transactions table with a `status` column and money columns; a payments table with `provider`, `status`, and `paid_at`.
- **Agent:** checks every new order within seconds: totals that don't reconcile, orders with no payment, payments with no order, velocity and address anomalies. It flags cases for review before shipping.
- **Core data:** orders and payments.
- **Typical gaps:**
  - a status-change history table
  - `updated_at` on orders
  - a device, IP, or risk-signal table
  - an idempotency key on payments
  - a refunds or chargebacks table
- **Fit profile:** 2/1/1/2. It rises to 2/2/2/2 when there are thousands of orders per minute and checks must run on every order without slowing down checkout commits.

### P6. Customer-service copilot
- **Signals:** users, orders, shipments, and payments; optionally tickets.
- **Agent:** answers "where is my order?" and billing questions from live data. It drafts refunds and order changes for your application to carry out against the primary database.
- **Core data:** users and orders.
- **Typical gaps:**
  - a shipments or tracking table
  - tickets and conversation history
  - a refunds table
  - a return-reason taxonomy
- **Fit profile:** 2/2/0/1. This is typically a steady workload, so a standard read pool is usually enough unless incident spikes cause massive ticket surges.

### P7. Field, fleet or logistics dispatcher
- **Signals:** location columns on assets, jobs, or depots; job status; time windows.
- **Agent:** assigns jobs by spatial distance (`PostGIS`) and live availability and re-plans routes in parallel when conditions change.
- **Core data:** jobs, assets, and locations.
- **Typical gaps:**
  - a live position table with `geography(Point, 4326)` and `updated_at`
  - service time windows
  - a skills or capacity table
- **Fit profile:** 2/2/2/2 in large fleets or weather/traffic disruption waves.

---

## Content and knowledge

### P8. Review and feedback insights agent
- **Signals:** a free-text column (`comment`, `body`, `feedback`) together with a rating or score and a link to an entity.
- **Agent:** groups text into themes with vector (`scann`) and BM25/full-text search, tracks changes in sentiment, and links problems to specific products or suppliers.
- **Core data:** reviews and the entities they link to.
- **Typical gaps:**
  - an embedding column + `scann` index
  - language and sentiment columns
  - a moderation or status flag
- **Fit profile:** 0/1/0/0. A batch job or standard read pool suffices.

### P9. Semantic or hybrid search over an entity catalog
- **Signals:** entity tables with rich text and attributes, such as hotels, properties, listings, documents, or courses.
- **Agent:** handles natural-language hybrid search and comparison, for example "quiet, near downtown, pet-friendly".
- **Core data:** the entity table.
- **Typical gaps:**
  - embeddings (`vector` + `scann`) and BM25/full-text indexes
  - structured amenity or attribute columns (`jsonb` or normalized tables)
  - spatial location (`geography(Point, 4326)`)
  - availability or pricing tables
- **Fit profile:** 1/2/1/0 on its own. It rises to 2/2/2/2 when combined with live availability and booking traffic (see P1).

### P10. Conversational analytics and "ask your data"
- **Signals:** any transactional schema with money and time columns.
- **Agent:** answers ad-hoc business questions by writing SQL, using the AlloyDB columnar engine on agent nodes for sub-second aggregations and optionally joining BigQuery or Spark history through lakehouse federation without ETL.
- **Core data:** any.
- **Typical gaps:**
  - column descriptions in Knowledge Catalog (the biggest factor in agent SQL accuracy)
  - declared foreign keys
  - a metrics or semantic layer
- **Fit profile:** 1/1/1/1, which rises when many analysts or autonomous sub-agents run concurrent exploratory queries against live tables.

---

## Cross-cutting: signals of high architecture fit

Raise the burst and isolation scores when you see:
- a per-end-user fan-out: one agent per shopper, guest, driver, or account
- a per-entity fan-out: one agent per SKU, site, route, or merchant
- workloads tied to spiky events: flash sales, product launches, market open, month-end, or incidents
- the agent reading the same tables as a revenue-critical write path, such as checkout, booking, or payment

Lower them when:
- the questions are purely historical/analytical and tolerate data that is hours old
- a single agent or user runs at a time
- the tables hold slowly changing reference data
