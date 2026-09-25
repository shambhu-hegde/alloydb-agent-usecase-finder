# Report template

Use this structure. Keep the top section readable in under a minute.

---

## Agentic use cases for <project(s)>, analysed <date>

**Scope:**
- N databases (X Cloud SQL, Y AlloyDB)
- M tables, K columns
- Z tables set aside, with the reasons

**Basis:** Knowledge Catalog metadata only. Relationships and scale are inferred; <the scale hints the user gave, or "no scale hints were given">.

### Top recommendations

For each of the top three use cases:

> **1. <Use case name>** — <Best suited case | Likely to suit use case> · average <x.x>/10
> <One sentence on what the agent does and the outcome it drives.>
> **Why this architecture:** <which fit tests scored 2, in plain words>
> **To build it you need:** <the one or two most important gaps>

### All candidates

| Rank | Use case | Database(s) | Fit /10 | Value /10 | Readiness /10 | Average /10 | Tier |
|---|---|---|---|---|---|---|---|

- Show each score to one decimal.
- Tier is either **Best suited case** or **Likely to suit use case**; there are no other tiers.
- Give the four fit test scores (freshness / lookups / burst / isolation) in the use case's detail section, not in the table.

Under the table, add one line for use cases left out because their fit is below 5.0, for example: *Not ranked: review insights (fit 1.3). A standard AlloyDB instance or BigQuery serves it well.*

### Data gaps by use case

For each of the top five use cases:
- **Missing or weak:** the table and column, and why the agent needs it
- **DDL sketch:** in a code block

### Schema findings

- the inferred relationship map: a list, or a diagram if the client can render one
- tables with no links to other tables, text columns duplicating ID links, nullable join columns, missing column descriptions
- sensitive columns to exclude from agent access

### Migration considerations

This section applies only to databases not yet on AlloyDB:
- engine and version, and the extensions in use
- the recommended path: Database Migration Service for PostgreSQL; a re-platform for MySQL or SQL Server
- region placement
- a reminder to confirm support in the current AlloyDB docs

### Recommended first build

- **Use case:** <name>
- **Why start here:** <reason>
- **Two-week plan:**
  1. <step>
  2. <step>
  3. <step>
- **Preview access:** https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb

*Agent nodes are read-only: agents propose actions, and your application carries them out against the primary database.*
