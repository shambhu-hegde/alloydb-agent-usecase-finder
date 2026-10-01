# Use-case pattern library

Each pattern lists:
- **Signals:** the schema evidence that makes the pattern applicable
- **Agent:** what the agent does
- **Core data:** the tables it can't work without
- **Typical gaps:** the columns or tables it usually needs added
- **Fit profile:** how it tends to score on the four fit tests (freshness / fast lookups / burst / isolation). This is a starting point; adjust it for the customer's actual schema and scale.

Match signals loosely. Names vary, for example `customer` / `users` / `accounts`, or `sku` / `variant` / `item`.

---

## Commerce and retail

### P1. Per-session shopping or booking assistant
- **Signals:** a product, listing or room table with text such as `name`, `description` or `title`; a stock or availability table (`quantity`, `available`, `reserved_*`); optionally reviews and a location column (`geography`, `geometry`, `lat`/`lng`).
- **Agent:** one agent per shopper conversation. It combines vector and full-text search with live availability, price and delivery or distance filters.
- **Core data:** catalogue tables and stock or availability.
- **Typical gaps:**
  - an embedding column on product or description text
  - a full-text index
  - `updated_at` on stock
  - a location column on the fulfilment site
  - a published price, or an effective-price view
- **Fit profile:** 2/2/2/2. This is the flagship case. Load spikes during sales or launches coincide with checkout peaks.

### P2. Inventory and fulfilment fan-out agents
- **Signals:** a stock table keyed by item and site; a warehouse, store or site table with a location column; order line items.
- **Agent:** one agent per site, category or item, launched as a burst. It finds items at risk of running out, suggests transfers between sites, drafts reorders, and picks the site to ship from.
- **Core data:** stock, sites and order lines.
- **Typical gaps:**
  - a real ID link from stock to site, instead of a free-text location
  - reorder point and lead-time columns
  - a supplier table
  - a stock-movement history table
- **Fit profile:** 2/2/2/2 once there are dozens of sites or thousands of items. Otherwise 2/2/1/1.

### P3. Cart-abandonment and next-best-action agent
- **Signals:** cart or session tables with `added_at`; users; product variants.
- **Agent:** spots carts about to be abandoned and checks stock and price changes. It then drafts a nudge message; your application sends it after approval.
- **Core data:** cart, users and variants.
- **Typical gaps:**
  - `updated_at` on cart
  - a cart status column
  - a table recording which nudges were sent to whom
  - marketing consent flags on users
- **Fit profile:** 2/2/1/1.

### P4. Promotion and pricing analyst
- **Signals:** a promotions, coupons or discounts table; orders; order lines.
- **Agent:** measures how promotions perform, flags codes being abused, and simulates price changes, joining with BigQuery history.
- **Core data:** promotions and orders.
- **Typical gaps:**
  - a promotion link on orders or order lines. This gap is very common.
  - a discount amount on order lines
  - price history
- **Fit profile:** 1/1/1/1. A standard read pool or BigQuery usually suffices.

---

## Operations, risk and support

### P5. Real-time order and payment integrity monitor
- **Signals:** an orders or transactions table with a `status` column and money columns; a payments table with `provider`, `status` and `paid_at`.
- **Agent:** checks every new order within seconds: totals that don't reconcile, orders with no payment, payments with no order, velocity and address anomalies. It flags cases for a person to review before shipping.
- **Core data:** orders and payments.
- **Typical gaps:**
  - a status-change history table
  - `updated_at` on orders
  - a device, IP or risk-signal table
  - an idempotency key on payments
  - a refunds or chargebacks table
- **Fit profile:** 2/1/1/2. It becomes 2/2/2/2 when there are thousands of orders per minute and checks must run on every order.

### P6. Customer-service copilot
- **Signals:** users, orders, shipments and payments; optionally tickets.
- **Agent:** answers "where is my order?" and billing questions from live data. It drafts refunds and changes for your application to carry out.
- **Core data:** users and orders.
- **Typical gaps:**
  - a shipments or tracking table
  - tickets and conversation history
  - a refunds table
  - a return-reason taxonomy
- **Fit profile:** 2/2/0/1. This is a steady workload, so a read pool is usually enough.

### P7. Field, fleet or logistics dispatcher
- **Signals:** location columns on assets, jobs or depots; job status; time windows.
- **Agent:** assigns jobs by distance and availability and re-plans when conditions change.
- **Core data:** jobs, assets and locations.
- **Typical gaps:**
  - a live position table
  - service time windows
  - a skills or capacity table
- **Fit profile:** 2/2/2/2 in large fleets.

---

## Content and knowledge

### P8. Review and feedback insights agent
- **Signals:** a free-text column (`comment`, `body`, `feedback`) together with a rating or score and a link to an entity.
- **Agent:** groups the text into themes with vector and full-text search, tracks changes in sentiment, and links problems to specific products or suppliers.
- **Core data:** reviews and the entities they link to.
- **Typical gaps:**
  - an embedding column
  - language and sentiment columns
  - a moderation or status flag
- **Fit profile:** 0/1/0/0. A batch job suffices.

### P9. Semantic or hybrid search over an entity catalogue
- **Signals:** entity tables with rich text and attributes, such as hotels, properties, listings, documents or courses.
- **Agent:** handles natural-language search and comparison, for example "quiet, near downtown, pet-friendly".
- **Core data:** the entity table.
- **Typical gaps:**
  - embeddings
  - structured amenity or attribute columns
  - location
  - availability or pricing tables
- **Fit profile:** 1/2/1/0 on its own. It rises to 2/2/2/2 when combined with live availability (see P1).

### P10. Conversational analytics and "ask your data"
- **Signals:** any transactional schema with money and time columns.
- **Agent:** answers business questions by writing SQL, optionally joining BigQuery through lakehouse federation.
- **Core data:** any.
- **Typical gaps:**
  - column descriptions in the catalog. This is the biggest factor in SQL accuracy.
  - declared foreign keys
  - a metrics or semantic layer
- **Fit profile:** 1/1/1/1, which rises when many analysts or agents run at once.

---

## Cross-cutting: signals of high architecture fit

Raise the burst and isolation scores when you see:
- a per-end-user fan-out: one agent per shopper, guest, driver or account
- a per-entity fan-out: one agent per SKU, site, route or merchant
- workloads tied to events: sales, launches, month-end, incidents
- the agent reading the same tables as a revenue-critical write path, such as checkout, booking or payment

Lower them when:
- the questions are analytical and tolerate data that is hours old
- a single agent or user runs at a time
- the tables hold slowly changing reference data
