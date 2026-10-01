# AlloyDB PostgreSQL agent use-case finder

This is a skill that discovers, ranks, and recommends the agent use cases you could build with **PostgreSQL for agents in AlloyDB** (Preview). 

Launch blogs:
1) https://cloud.google.com/blog/products/databases/announcing-postgresql-for-agents-in-alloydb
2) https://cloud.google.com/blog/products/databases/alloydbs-agentic-database-architecture

This skills reads your table and column metadata through the Google Cloud **Knowledge Catalog (Dataplex) MCP server**, then:

1. **Inventories your databases:** finds your Cloud SQL and AlloyDB tables (and checks for BigQuery tables that could join via lakehouse federation), strictly scoped to the Google Cloud projects you name.
2. **Models your business domain:** infers relationships between tables and classifies each table.
3. **Proposes grounded agent use cases:** matches your schema against ten agent patterns and cites the exact tables and columns that support each candidate
4. **Scores and ranks candidates out of 10:** evaluates **Architecture Fit** (grounded in AlloyDB's compute isolation, sub-millisecond I/O, elastic zero-to-thousands node scaling, and hybrid vector/BM25/spatial/columnar execution), **Business Value**, and **Data Readiness**, labeling each ranked candidate a **Best suited case** or a **Likely to suit use case**
5. **Produces an actionable gap & migration blueprint:** lists missing columns, indexes, and tables with copy-pasteable AlloyDB DDL sketches (including `pgvector`/`scann`, `google_ml_integration` automated embeddings, `bm25`, and `PostGIS`), governance guardrails, and Cloud SQL → AlloyDB migration considerations

It reads **metadata only**: no row data is ever queried, and no writes are ever performed.

> This is not an official Google product.

---

## Prerequisites

1. **Google Cloud project(s)** with Cloud SQL or AlloyDB databases.
2. **Catalog discovery enabled:** Your Cloud SQL and AlloyDB metadata must be ingested into **Knowledge Catalog** (formerly Dataplex Universal Catalog) so tables appear in catalog searches. See [Knowledge Catalog integration for Cloud SQL](https://docs.cloud.google.com/sql/docs/postgres/dataplex-catalog-integration) and [Knowledge Catalog integration for AlloyDB](https://docs.cloud.google.com/alloydb/docs/knowledge-catalog-integration).
3. **Knowledge Catalog remote MCP server connected** (`https://dataplex.googleapis.com/mcp`) with read-only permissions (see below).

---

## Install the skill

The skill lives in [`skills/alloydb-postgresql-agent-usecase-finder`](skills/alloydb-postgresql-agent-usecase-finder). A packaged zip archive is also provided at [`alloydb-postgresql-agent-usecase-finder.skill`](alloydb-postgresql-agent-usecase-finder.skill).

### Quick clone (CLI / IDE environments)

```bash
git clone https://github.com/shambhu-hegde/alloydb-agent-usecase-finder.git

# For Claude Code:
mkdir -p ~/.claude/skills && cp -r alloydb-agent-usecase-finder/skills/alloydb-postgresql-agent-usecase-finder ~/.claude/skills/

# For Antigravity / Antigravity CLI / Gemini CLI:
mkdir -p ~/.gemini/config/skills && cp -r alloydb-agent-usecase-finder/skills/alloydb-postgresql-agent-usecase-finder ~/.gemini/config/skills/
```

### Client-by-client installation

| Environment | How to install the skill |
|---|---|
| **Claude Desktop / Claude Web** | Download [`alloydb-postgresql-agent-usecase-finder.skill`](alloydb-postgresql-agent-usecase-finder.skill) and upload it under **Settings → Capabilities → Skills**. |
| **Claude Code** | Copy `skills/alloydb-postgresql-agent-usecase-finder` into `~/.claude/skills/` (personal) or `.claude/skills/` (project-scoped). |
| **VS Code / Gemini Code Assist / Cursor** | Copy `skills/alloydb-postgresql-agent-usecase-finder` into `.agents/skills/` or `.claude/skills/` in your workspace. |
| **Antigravity & Antigravity CLI** | Copy `skills/alloydb-postgresql-agent-usecase-finder` into `.agents/skills/` or `.gemini/skills/` in your workspace, or globally into `~/.gemini/config/skills/`. |
| **Other `SKILL.md` clients** | Add `skills/alloydb-postgresql-agent-usecase-finder` to your agent's skills directory. |

---

## Connect the Knowledge Catalog MCP server

If the MCP server isn't connected yet, the skill automatically detects that in Step 0 and walks you through setup. In summary:

- **Server URL:** `https://dataplex.googleapis.com/mcp` (Streamable HTTP)
- **OAuth scope (read-only):** `https://www.googleapis.com/auth/dataplex.readonly`
- **Least-privilege IAM roles:**
  - **MCP Tool User** (`roles/mcp.toolUser`) to invoke the remote MCP server
  - **Dataplex Catalog Viewer** (`roles/dataplex.catalogViewer` or `roles/dataplex.viewer`, plus read access to source metadata — or a minimal custom role granting `dataplex.projects.search`, `dataplex.entries.get`, `dataplex.entries.list`, and `dataplex.entryGroups.use`). For sandbox or test projects, **Dataplex Catalog Admin** (`roles/dataplex.catalogAdmin`) also works as a broader fallback.

### Client MCP configuration examples

- **Claude Desktop / Claude Web:** Go to **Settings → Connectors → Add custom connector**, enter `https://dataplex.googleapis.com/mcp`, and complete Google sign-in.
- **Claude Code:**
  ```bash
  claude mcp add --transport http knowledge-catalog https://dataplex.googleapis.com/mcp
  ```
- **Antigravity / Antigravity CLI / Gemini CLI** (`~/.gemini/config/mcp_config.json`):
  ```json
  {
    "mcpServers": {
      "knowledgeCatalog": {
        "serverUrl": "https://dataplex.googleapis.com/mcp",
        "authProviderType": "google_credentials"
      }
    }
  }
  ```
- **VS Code / Cursor** (`.vscode/mcp.json` or `.cursor/mcp.json`):
  Configure a `http` / Streamable HTTP server pointing to `https://dataplex.googleapis.com/mcp` authenticated with your Google Cloud credentials (`gcloud auth application-default login`).

Full setup and troubleshooting instructions are in [`references/setup.md`](skills/alloydb-postgresql-agent-usecase-finder/references/setup.md).

---

## Use

Ask your agent something like:

> What agents could I build on my databases in project `my-project` with AlloyDB's PostgreSQL for agents?

For sharper **Business Value** and **Architecture Fit** scores, include:
- **Your business priorities:** your industry and the 1–2 outcomes you care about most (e.g., checkout conversion, preventing stockouts, fraud reduction)
- **Your scale profile:** peak concurrent users or agent sessions, and whether traffic arrives in bursts (e.g., flash sales, market open, incident storms)

### Sample output preview

Each analysis produces an executive summary of the top 3 recommendations, a ranked comparison table, copy-pasteable DDL sketches for schema gaps, and a two-week first-build plan:

| Rank | Use case | Fit /10 | Value /10 | Readiness /10 | Average /10 | Tier |
|---|---|---|---|---|---|---|
| 1 | Per-session shopping assistant | 10.0 | 10.0 | 8.0 | 9.3 | Best suited case |
| 2 | Inventory fan-out agents | 10.0 | 8.0 | 6.0 | 8.0 | Best suited case |
| 3 | Order and payment integrity monitor | 7.5 | 10.0 | 6.0 | 7.8 | Best suited case |

See [`references/example-report.md`](skills/alloydb-postgresql-agent-usecase-finder/references/example-report.md) for a complete worked example.

---

## Repository structure

| File | Purpose |
|---|---|
| [`skills/alloydb-postgresql-agent-usecase-finder/SKILL.md`](skills/alloydb-postgresql-agent-usecase-finder/SKILL.md) | Main skill workflow, architectural grounding, and guardrails |
| [`skills/alloydb-postgresql-agent-usecase-finder/references/setup.md`](skills/alloydb-postgresql-agent-usecase-finder/references/setup.md) | Connecting the remote Knowledge Catalog MCP server: API, least-privilege IAM, OAuth scope, and connection test |
| [`skills/alloydb-postgresql-agent-usecase-finder/references/usecase-patterns.md`](skills/alloydb-postgresql-agent-usecase-finder/references/usecase-patterns.md) | Ten agent use-case patterns with schema signals, core tables, typical gaps, and fit profiles |
| [`skills/alloydb-postgresql-agent-usecase-finder/references/scoring-rubric.md`](skills/alloydb-postgresql-agent-usecase-finder/references/scoring-rubric.md) | Architecture fit, business value, and data readiness scoring rubrics, tier rules, and gap checklist |
| [`skills/alloydb-postgresql-agent-usecase-finder/references/report-template.md`](skills/alloydb-postgresql-agent-usecase-finder/references/report-template.md) | Output report template |
| [`skills/alloydb-postgresql-agent-usecase-finder/references/example-report.md`](skills/alloydb-postgresql-agent-usecase-finder/references/example-report.md) | Worked example report on a multi-database e-commerce schema |
| [`alloydb-postgresql-agent-usecase-finder.skill`](alloydb-postgresql-agent-usecase-finder.skill) | Packaged zip archive of the skill for one-click upload in Claude |

---

## About PostgreSQL for agents in AlloyDB

- **Launch blog:** [Announcing PostgreSQL for agents in AlloyDB](https://cloud.google.com/blog/products/databases/announcing-postgresql-for-agents-in-alloydb?e=48754805)
- **Architecture deep dive:** [AlloyDB's agentic database architecture](https://cloud.google.com/blog/products/databases/alloydbs-agentic-database-architecture?e=48754805)
- **Documentation:** <https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb>
- **Preview access request form:** [Sign up for the Preview](https://docs.google.com/forms/d/e/1FAIpQLSfYv_zv2CI9L6xZxkExZai_jG-eiz8iEYPfwLFwaIatZdYCrA/viewform)

## License

Apache 2.0. See [LICENSE](LICENSE).
