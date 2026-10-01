# Report template

Use this structure. Keep the top section readable in under a minute.

---

## Agentic use cases for <project(s)>, analysed <date>

**Scope:**
- N databases (X Cloud SQL, Y AlloyDB; plus any BigQuery datasets noted for lakehouse federation)
- M operational tables, K columns
- Z tables set aside (PostGIS/extension, migration-tracking, partition children, or test/scratch tables), with the reasons

**Basis:** Knowledge Catalog metadata only (no row data queried). Relationships and scale are inferred; <the scale hints the user gave, or "no scale hints were given, so domain-standard peak concurrency was assumed">.

### Top recommendations

For each of the top three use cases:

> **1. <Use case name>** — <Best suited case | Likely to suit use case> · average <x.x>/10
> <One sentence on what the agent does and the business outcome it drives.>
> **Why this architecture:** <which fit tests scored 2, grounded in microVM/Colossus isolation, sub-ms cold-read I/O, 0→N elastic scaling, or hybrid vector/BM25/spatial/columnar queries>
> **To build it you need:** <the one or two most important schema gaps>

### All candidates

| Rank | Use case | Database(s) | Fit /10 | Value /10 | Readiness /10 | Average /10 | Tier |
|---|---|---|---|---|---|---|---|

- Show each score to one decimal.
- Tier is either **Best suited case** or **Likely to suit use case**; there are no other tiers.
- Give the four fit test scores (freshness / lookups / burst / isolation) right below the table or in the use case's detail section.

Under the table, add one line for use cases left out because their fit is below 5.0, for example: *Not ranked: review insights (fit 1.3). A standard AlloyDB instance, read pool, or BigQuery batch workflow serves it well.*

### Data gaps by use case

For each of the top five use cases:
- **Missing or weak:** the exact table and column (or index/table), and why the agent needs it
- **DDL sketch:** in a `sql` code block

### Schema findings

- the inferred relationship map: a concise list, or a diagram if the client renders Mermaid
- tables with no links to other tables, free-text columns duplicating ID links, nullable join columns, and missing catalog column descriptions
- sensitive columns to exclude from agent access via parameterized secure views or column grants

### Migration considerations

This section applies only to databases not yet on AlloyDB:
- engine, version, and extensions in use
- the recommended path: Database Migration Service (DMS) for Cloud SQL for PostgreSQL; a re-platform for Cloud SQL for MySQL or SQL Server
- region placement relative to other databases and BigQuery datasets
- a reminder to confirm specific version and extension support in the current AlloyDB documentation

### Recommended first build

- **Use case:** <name>
- **Why start here:** <reason>
- **Two-week plan:**
  1. <Week 1 step>
  2. <Week 2 step>
- **Documentation:** https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb
- **Request Preview access:** https://docs.google.com/forms/d/e/1FAIpQLSfYv_zv2CI9L6xZxkExZai_jG-eiz8iEYPfwLFwaIatZdYCrA/viewform

*Agent nodes are read-only: agents read and reason on isolated agent nodes, and your application carries out approved write actions against the primary database.*
