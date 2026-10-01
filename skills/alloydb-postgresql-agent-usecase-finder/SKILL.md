---
name: alloydb-postgresql-agent-usecase-finder
description: Discover, rank, and gap-check agentic use cases for PostgreSQL for agents in AlloyDB using a customer's existing Cloud SQL and AlloyDB databases, read through the Knowledge Catalog MCP server. Use when someone asks what agents they could build on their databases, whether their data fits AlloyDB's agentic architecture, or what data is missing to support an agent use case.
---

# AlloyDB PostgreSQL agent use-case finder

This skill looks at the table and column metadata of a customer's Cloud SQL and AlloyDB databases through Knowledge Catalog. It proposes agent use cases and ranks them by business value, data readiness and fit for **PostgreSQL for agents in AlloyDB** (Preview). It also lists the columns and tables each use case is missing.

The skill never reads row data and never writes anything. It works only from catalog metadata, so say this in the report and never claim row counts, data volumes or data quality the catalog didn't return.

## What the launch provides (use this to judge fit)

PostgreSQL for agents in AlloyDB gives agents temporary, isolated, **read-only** AlloyDB instances ("agent nodes"). Agents reach them through MCP. The agent nodes:

- see production data to within about a second
- run the full PostgreSQL engine: B-tree, vector, full-text and spatial indexes, plus the columnar engine
- scale from zero to thousands of nodes in seconds and back to zero, billed per second
- share no compute, network or storage path with the production cluster
- can join live data with BigQuery and Spark lakehouse data without ETL

Consequence to state in every report: **agents read on agent nodes; any change to data goes through the application or the primary database.**

The feature is in Preview and needs an access request: https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb

## Workflow

Follow the steps in order. Show the user a short progress line between steps, not raw tool output.

### Step 0: Check whether the Knowledge Catalog MCP server is configured

Set-up guidance is only for users who need it. Check silently first:

1. Look for the tools `search_entries`, `lookup_context` and `lookup_entry`. They may carry a server prefix.
2. If they are present, run the connection test from `references/setup.md`: one `search_entries` call limited to the user's project, with `pageSize: 5`.

Then act on the result:

- **Tools present and the test returns rows:** the server is configured. Don't mention setup. Go straight to Step 1.
- **Tools present but the test returns nothing or a permission error:** the server is connected but not fully configured. Show only the part of `references/setup.md` that fixes it: IAM roles (section 2) or cataloguing (section 4). Rerun the test, then continue.
- **Tools absent:** the server isn't configured. Walk the user through `references/setup.md` for the remote Knowledge Catalog MCP server (`https://dataplex.googleapis.com/mcp`): enabling the API, the IAM roles, and the read-only OAuth scope. Rerun the test, then continue.

Ask for the **Google Cloud project ID or IDs** to analyse if the user hasn't given them. You need this before running the connection test. Optionally ask for:
- **Business context:** the industry and the one or two outcomes they care about most. This improves the value scores.
- **Scale hints:** peak concurrent users or sessions, and whether traffic comes in spikes. The catalog can't provide these, and they matter for scoring fit.

### Step 1: Inventory the databases

For each project, run scoped searches. **Always pass `scopeType: PROJECT` and `scopeId: <project>`.** Without a scope, results can include tables from other projects in the organization.

```
system:cloud_sql AND type:table
system:alloydb AND type:table
```

- The Cloud SQL system value is `cloud_sql`. The value `cloudsql` returns nothing.
- Use `pageSize` of 200 and follow `pageToken` until it runs out.
- As a cross-check, run `type:table AND (cloudsql OR alloydb)` with the same scope and compare the counts. If they differ, find out why before continuing.

Group the tables by **engine → instance or cluster → region → database → schema**. From the entry type, note the engine: `cloudsql-postgresql-*`, `cloudsql-mysql-*`, `cloudsql-sqlserver-*` or `alloydb-*`.

Set aside, and list separately in the report:
- PostGIS and extension tables, such as `spatial_ref_sys`, `geometry_columns` and `pg_*`
- tables that look like tests or scratch tables, such as `test*`, `tmp*`, `*_bak` and `sample_*`. Include these only if the user asks.

### Step 2: Profile the schemas

Call `lookup_context` in batches of **at most 10 resources**. All resources in a batch must be in the same location, and that location goes in the `location` argument. Record for every table:
- each column's name, type and whether it can be empty
- a data profile, quality score or usage patterns, if present (often absent)
- detected joins and sample SQL, if present

Use `lookup_entry` with `view: CUSTOM` only when you need a specific aspect, such as `contacts` for owners or `data-quality-scorecard`.

If the catalog returns no descriptions, statistics or joins, say so once. Then work from names and types.

### Step 3: Model the business domain

1. **Name the domain**, for example e-commerce, hospitality, logistics, fintech, healthcare or SaaS. If there are several databases, name one per database.
2. **Infer relationships.** A column named `<x>_id` or `<x>_uuid` points to the table `<x>`, `<x>s` or `<x>es`, and to its `id` column. A column named `parent_id` points back to its own table. Mark every relationship as **inferred**, because the catalog rarely stores declared foreign keys.
3. **Classify each table:**
   - entity, such as users, products or hotels
   - transaction or event, such as orders, payments or bookings
   - line or junction, such as order_items
   - state, such as inventory, cart or sessions
   - reference, such as categories or brands
   - content, such as reviews or tickets

### Step 4: Detect signals and propose use cases

Match the schema against `references/usecase-patterns.md`. Each pattern lists the columns that indicate it applies, the agent it enables, and the tables and columns it needs.

- Propose only use cases whose core tables exist.
- For each use case, cite the exact tables and columns that support it.
- You may propose use cases that aren't in the library if the schema clearly supports them. Mark them as `custom`.
- Aim for 5 to 10 candidates. Don't pad the list.

### Step 5: Score and rank

Score each use case with `references/scoring-rubric.md`. Every score is out of 10:
- **Architecture fit** (0 to 10). Four tests, each scored 0 to 2: freshness, fast indexed lookups, bursty or massively parallel load, and production isolation. The fit score is their total × 1.25.
- **Business value** (2 to 10)
- **Data readiness** (2 to 10)
- **Average score:** the mean of the three, to one decimal. Rank by it.

Assign one of two tiers:
- **Best suited case:** fit of 7.5 or more, with every fit test scoring at least 1.
- **Likely to suit use case:** fit from 5.0 to 7.4.

Leave use cases with a fit below 5.0 out of the ranking. List them in one line under the table, saying that a standard AlloyDB instance, read pool or BigQuery serves them well. A credible report says so.

Scale is the deciding factor for fit, and the catalog can't see it. If the user gave no scale hints, score the burst test on the plausible production pattern for the domain, and say that you assumed it.

### Step 6: Gap analysis

For each of the top five use cases, list what is missing, using the checklist in `references/scoring-rubric.md` (section "Gap checklist"). Every gap names:
- the exact table and column to add or change
- why the agent needs it
- a DDL sketch

Also list **schema-level findings**, for example:
- tables with no links to other tables
- text-based references that duplicate a real ID column
- nullable columns that break joins
- columns with no description

Then list **migration considerations** for any Cloud SQL database:
- whether AlloyDB supports its engine and version. AlloyDB is PostgreSQL only, so MySQL and SQL Server need a re-platform, not a migration.
- the extensions it uses, such as PostGIS or pgvector
- region placement relative to other databases
- Database Migration Service as the default path for Cloud SQL for PostgreSQL to AlloyDB

Tell the user to confirm version and extension support in the current AlloyDB documentation. Don't assert it.

### Step 7: Write the report

Use the structure in `references/report-template.md`. Lead with the top three use cases and the verdict for each. Keep the full scoring table and the gap details below that. Close with:
- the recommended first build and why
- the preview sign-up link
- one sentence restating that agent nodes are read-only

`references/example-report.md` shows the expected depth.

## Guardrails

- **Metadata only.** Never run SQL against customer databases as part of this skill. If the user wants declared foreign keys or row counts, offer that as a separate, read-only step using the database's own MCP server, and only with their approval.
- **Label inferred results.** Relationships, domain and scale are inferences. Label them as inferred.
- **Stick to the documented launch.** Don't promise features beyond what the launch describes. If asked about pricing, regions or version support, point to the documentation.
- **Handle sensitive columns carefully.** Treat columns such as `password_hash`, tokens, government IDs and card numbers as sensitive. Recommend excluding them from agent access, for example with parameterized secure views or column grants, and never list them as useful inputs to an agent.
