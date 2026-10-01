# Scoring rubric and gap checklist

Every score is out of 10, so readers can compare them directly.

## 1. Architecture fit (0 to 10)

Score each test from 0 to 2. Base scores on the schema evidence and the user's scale hints. When assuming, write the assumption next to the score.

| Test | 0 | 1 | 2 |
|---|---|---|---|
| **Freshness.** How stale can the data be? | Hours or days is fine | Minutes | Seconds; decisions act on the current state (stock, payment status, availability) |
| **Fast indexed lookups.** Does each reasoning step need index, vector, full-text or spatial lookups within milliseconds? | Mostly large scans or aggregates | A mix | Point lookups or searches in every agent turn |
| **Burst or parallel load.** How many agents run at once, and how spiky is it? | One agent, or a steady trickle | Tens of agents, or regular periodic spikes | Hundreds to thousands, or unpredictable spikes tied to business events |
| **Production isolation.** How badly would agent load hurt revenue-critical writes? | Separate or unimportant tables | Shared tables at moderate load | Same tables as checkout, booking or payment, at peak |

```
fit = (freshness + lookups + burst + isolation) × 1.25        (0 to 10)
```

| Test total | 8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|---|
| Fit /10 | 10.0 | 8.8 | 7.5 | 6.3 | 5.0 | 3.8 | 2.5 | 1.3 | 0.0 |

## 2. Tiers

Only two tiers appear in the ranking:

- **Best suited case:** fit of 7.5 or more, with every test scoring at least 1. Older options can't serve these well: fixed replicas either run out of capacity or sit idle, and shared storage lets agent load slow down production.
- **Likely to suit use case:** fit from 5.0 to 7.4. The architecture helps, and becomes necessary as scale grows. A read pool may work at current scale.

Use cases with a fit below 5.0 are left out of the ranking. List them in one line under the table, saying that a standard AlloyDB instance, read pool or BigQuery serves them well.

## 3. Business value (2 to 10)

| Score | Meaning |
|---|---|
| 10 | Directly moves revenue or prevents loss on the core flow, such as conversion, fraud or stockouts |
| 8 | Clear cost or efficiency win in a core operation |
| 6 | Useful operational insight, with outcomes measured indirectly |
| 4 | Productivity improvement for a small team |
| 2 | Nice to have |

If the user stated priorities, raise the score of use cases that serve them by 2, up to a maximum of 10.

## 4. Data readiness (2 to 10)

| Score | Meaning |
|---|---|
| 10 | All core tables and links exist. Only optional additions are needed. |
| 8 | Core data exists. Needs embeddings or indexes, which are additions only. |
| 6 | Needs one or two new columns or links on existing tables |
| 4 | Needs a new table, or a significant schema change |
| 2 | Core data is missing |

Subtract 2 if the database is on an engine that needs a re-platform, not a migration, to reach AlloyDB: Cloud SQL for MySQL or SQL Server.

## 5. Average score and rank

```
average = (fit + value + readiness) / 3        (to one decimal)
```

- Rank by average score.
- If two use cases differ by less than 0.2, rank the one with the higher fit first. This skill exists to find launch-specific wins.
- Always show fit, value and readiness next to the average, so readers can see what drives it.

## 6. Gap checklist

Go through this list for each top use case. Report only the gaps that apply, each with a DDL sketch.

**Search and retrieval**
- [ ] **Embedding column** on each text field the agent searches by meaning. For example: `ALTER TABLE products ADD COLUMN description_embedding vector(768);` plus a ScaNN or HNSW index.
- [ ] **Full-text or BM25 index** on the fields the agent searches by keyword.
- [ ] **Location column** (`geography(Point,4326)`) plus a spatial index, wherever distance matters.

**Relationships and integrity**
- [ ] **ID link instead of free text.** For example, replace `inventory.warehouse_location` (text) with a required `warehouse_id`.
- [ ] **Missing link between tables.** For example, `orders.promotion_id`, or an `order_promotions` junction table.
- [ ] **Declared foreign keys**, so agents and the catalog can find joins reliably.
- [ ] **Required columns that are nullable** and would break joins the agent depends on.

**Freshness and history**
- [ ] **`updated_at`** on state tables such as stock, cart, orders and bookings.
- [ ] **A status-history or event table** for anything the agent monitors over time.
- [ ] **Movement or ledger tables**, such as stock movements, where the agent reasons about change rather than the current snapshot.

**Actionability.** Remember that agent nodes are read-only.
- [ ] **A table where the agent writes its proposals**, such as `agent_recommendations(id, agent, entity_ref, action, payload jsonb, status, created_at, approved_by)`, written by the application or the primary database after approval.
- [ ] **Idempotency keys** on actions the application executes on the agent's behalf.

**Governance**
- [ ] **Sensitive columns** (passwords, tokens, government IDs, card numbers) excluded through parameterized secure views or column grants.
- [ ] **Column descriptions** in Knowledge Catalog. They directly improve agent SQL accuracy.
- [ ] **Owner or steward contacts** on the tables the agent depends on.

**Platform**
- [ ] **The database is on AlloyDB.** If not, list the migration path and the engine, version and extension checks.
- [ ] **Related databases are in the same region**, or cross-region access is planned.
