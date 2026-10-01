# Scoring rubric and gap checklist

Every score is out of 10, so readers can compare them directly.

## 1. Architecture fit (0 to 10)

Score each test from 0 to 2. Base scores on the schema evidence, the architectural capabilities of PostgreSQL for agents in AlloyDB (microVM isolation, sub-millisecond Colossus cold-read I/O, elastic zero-to-thousands node scaling, and hybrid vector/BM25/spatial/columnar execution), and the user's scale hints. When assuming scale, write the assumption next to the score.

| Test | 0 | 1 | 2 |
|---|---|---|---|
| **Freshness.** How stale can the data be? | Hours or days is fine | Minutes | Seconds (~1s replication lag); decisions act on live state (stock, payment status, availability) |
| **Fast indexed lookups & hybrid queries.** Does each reasoning step need low-latency index, vector, BM25, spatial, or hybrid columnar lookups? | Offline batch scans or slow background reports | A moderate mix of simple queries | Point lookups, vector (`scann`), BM25/full-text, spatial (`PostGIS`), or hybrid operational + columnar analytical queries in every agent turn (benefiting from sub-ms Colossus cold-read I/O) |
| **Burst or parallel load.** How many agents run at once, and how spiky is it? | One agent, or a steady trickle | Tens of agents, or regular periodic spikes | Hundreds to thousands of concurrent agents, or unpredictable spikes tied to business events requiring instant 0 → N → 0 scaling |
| **Production isolation.** How badly would agent load hurt revenue-critical writes? | Separate or non-critical tables | Shared tables at moderate load | Same tables as checkout, booking, or payment at peak, where microVM + dedicated Colossus segment isolation prevents primary latency spikes |

```text
fit = (freshness + lookups + burst + isolation) × 1.25        (0 to 10)
```

| Test total | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|---|
| Fit /10 | 10.0 | 8.8 | 7.5 | 6.3 | 5.0 | 3.8 | 2.5 | 1.3 | 0.0 |

## 2. Tiers

Only two tiers appear in the ranking:

- **Best suited case:** `fit` of `7.5` or more, with **every** test scoring at least `1`. Traditional architectures struggle here: fixed read replicas either run out of capacity during bursts or sit idle, object-storage-backed replicas suffer 10x cold-cache latency cliffs when new nodes spin up, and shared storage paths let agent load degrade primary transactions.
- **Likely to suit use case:** `fit` from `5.0` to `7.4` (or `fit >= 7.5` when one of the four tests scored `0`). The agentic architecture helps today and becomes essential as concurrency or agent fan-out grows, though a standard read pool may suffice at low initial scale.

Use cases with a `fit` below `5.0` are left out of the ranking. List them in one line under the table, stating that a standard AlloyDB instance, read pool, or BigQuery batch/analytical workflow serves them well.

## 3. Business value (2 to 10)

| Score | Meaning |
|---|---|
| 10 | Directly moves revenue or prevents loss on the core flow, such as checkout conversion, fraud prevention, or stockouts |
| 8 | Clear cost or efficiency win in a core operation |
| 6 | Useful operational insight, with outcomes measured indirectly |
| 4 | Productivity improvement for a small team |
| 2 | Nice to have |

If the user stated business priorities, raise the score of use cases that directly serve them by 2, up to a maximum of 10.

## 4. Data readiness (2 to 10)

| Score | Meaning |
|---|---|
| 10 | All core tables, links, and search/vector/spatial columns exist. Only optional additions are needed. |
| 8 | Core data exists. Needs embeddings or indexes (`scann`, `bm25`, `GIN`, `GIST`), which are additive only. |
| 6 | Needs one or two new columns or foreign-key links on existing tables |
| 4 | Needs a new table (e.g., status history or stock movements), or a significant schema change |
| 2 | Core data is missing |

- Credit existing `vector`, `tsvector`, `geography`/`geometry`, and `jsonb` columns discovered in Step 2 when assigning Data Readiness.
- Subtract 2 (down to a minimum of 2) if the database is on an engine that requires a re-platform rather than a homogeneous migration to reach AlloyDB (Cloud SQL for MySQL or SQL Server).

## 5. Average score and rank

```text
average = (fit + value + readiness) / 3        (to one decimal)
```

- Rank by average score.
- **Tie-breaker:** If two use cases differ by less than `0.2`, rank the one with the higher **Architecture Fit** first. This skill exists to find workloads that uniquely benefit from PostgreSQL for agents in AlloyDB.
- Always show Fit, Value, and Readiness next to the Average so readers can see what drives the ranking.

## 6. Gap checklist

Go through this list for each of the top five use cases. Report only the gaps that apply, each with a copy-pasteable AlloyDB DDL sketch.

**Search and retrieval**
- [ ] **Embedding column + ScaNN index** on each text field the agent searches by semantic meaning. Show either standard `pgvector` + `scann` or AlloyDB's `google_ml_integration` automated embedding generation:
  ```sql
  -- Option A: Automated in-database embeddings via google_ml_integration
  ALTER TABLE products
    ADD COLUMN description_embedding vector(768)
    GENERATED ALWAYS AS (embedding('gemini-embedding-001', description)) STORED;
  CREATE INDEX ON products USING scann (description_embedding cosine);

  -- Option B: Application-populated vector column + ScaNN index
  ALTER TABLE products ADD COLUMN description_embedding vector(768);
  CREATE INDEX ON products USING scann (description_embedding cosine);
  ```
- [ ] **BM25 or full-text index** on the fields the agent searches by keyword (using AlloyDB's native `bm25` index or PostgreSQL `GIN (to_tsvector('english', ...))` for hybrid semantic + lexical search).
- [ ] **Location column** (`geography(Point, 4326)`) plus a PostGIS `GIST` spatial index wherever distance or routing matters.

**Relationships and integrity**
- [ ] **ID link instead of free text.** For example, replace `inventory.warehouse_location` (`text`) with a required `warehouse_id` referencing `warehouses(id)`.
- [ ] **Missing link between tables.** For example, `orders.promotion_id` or an `order_promotions` junction table.
- [ ] **Declared foreign keys**, so both agents and Knowledge Catalog can discover join paths reliably.
- [ ] **Required columns that are currently nullable** and would silently drop rows in joins the agent depends on.

**Freshness and history**
- [ ] **`updated_at` (`timestamptz`)** on mutable state tables such as `inventory`, `cart`, `orders`, and `bookings`.
- [ ] **A status-history or event table** for state transitions the agent monitors over time.
- [ ] **Movement or ledger tables**, such as `stock_movements`, where the agent reasons about velocity and trends rather than a static snapshot.

**Actionability.** Remember that agent nodes are read-only.
- [ ] **An action-proposal table on the primary database**, such as `agent_recommendations(id uuid PRIMARY KEY, agent_name text, entity_ref text, proposed_action text, payload jsonb, status text, created_at timestamptz, approved_by text)`, written by the application against the primary instance after policy or human approval.
- [ ] **Idempotency keys** on transactional tables where the application executes actions on an agent's behalf.

**Governance**
- [ ] **Sensitive columns** (`password_hash`, auth tokens, government IDs, card numbers) excluded from agent roles via parameterized secure views or column-level privileges.
- [ ] **Column descriptions** populated in Knowledge Catalog (directly improves agent text-to-SQL accuracy).
- [ ] **Owner or data-steward contacts** attached to the catalog entries the agent depends on.

**Platform**
- [ ] **The database is on AlloyDB.** If it is currently on Cloud SQL, specify the migration path (Database Migration Service for Cloud SQL for PostgreSQL; re-platform for MySQL or SQL Server) and the engine, version, and extension checks.
- [ ] **Related databases are in the same region** (or cross-region / BigQuery lakehouse federation access is planned).
