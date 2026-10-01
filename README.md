# AlloyDB PostgreSQL agent use-case finder

An agent skill that finds the agents you could build on your existing **Cloud SQL** and **AlloyDB** databases with **PostgreSQL for agents in AlloyDB** (Preview).

It reads your table and column metadata through the remote **Knowledge Catalog MCP server**, then:

1. inventories your Cloud SQL and AlloyDB tables, limited to the projects you name
2. infers your business domain and the relationships between tables
3. proposes agent use cases from a library of ten patterns
4. scores each use case out of 10 on architecture fit, business value and data readiness, ranks by the average, and labels each one a **Best suited case** or a **Likely to suit use case**
5. lists missing columns and tables, with DDL sketches, plus migration considerations

It reads **metadata only**: no row data, and no writes.

> This is not an official Google product.

## Install

The skill lives in [`skills/alloydb-postgresql-agent-usecase-finder`](skills/alloydb-postgresql-agent-usecase-finder). A packaged copy is in [`alloydb-postgresql-agent-usecase-finder.skill`](alloydb-postgresql-agent-usecase-finder.skill), a standard zip archive.

| Client | How |
|---|---|
| Claude | Download the `.skill` file and upload it under Settings → Capabilities → Skills |
| Claude Code | Copy the skill folder to `~/.claude/skills/` |
| Antigravity | Copy the skill folder to `.agents/skills/` in your workspace, or to `~/.gemini/config/skills/` |
| Other clients | Any client that loads SKILL.md-format skills |

## Connect the Knowledge Catalog MCP server

The skill checks for the server and walks you through setup only if it isn't connected. In short:

- **Server URL:** `https://dataplex.googleapis.com/mcp` (Streamable HTTP)
- **OAuth scope:** `https://www.googleapis.com/auth/dataplex.readonly`
- **IAM roles:** MCP Tool User (`roles/mcp.toolUser`) and Dataplex Catalog Admin (`roles/dataplex.catalogAdmin`)

**Antigravity** only accepts the `serverUrl` field. Add this to `~/.gemini/config/mcp_config.json`:

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

Full details are in [`references/setup.md`](skills/alloydb-postgresql-agent-usecase-finder/references/setup.md).

## Use

Ask your agent something like:

> What agents could I build on my databases in project `my-project` with AlloyDB's PostgreSQL for agents?

Add your business priorities and your peak concurrency for sharper scores.

## About PostgreSQL for agents in AlloyDB

The feature is in Preview. Request access at <https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb>.

## License

Apache 2.0. See [LICENSE](LICENSE).
