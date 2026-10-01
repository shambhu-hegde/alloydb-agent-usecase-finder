# Connecting the Knowledge Catalog remote MCP server

Knowledge Catalog was formerly called Dataplex Universal Catalog. The API, CLI, and IAM names still use `dataplex`.

This skill uses the **remote** Knowledge Catalog MCP server hosted by Google (`https://dataplex.googleapis.com/mcp`). There is nothing to install locally.

**Use this file only when Step 0 of the skill finds the server isn't configured or the connection test fails.** If the tools are present and the connection test at the end of this file returns rows, skip it entirely. If the tools are present but the test fails, show only the section that fixes the failure.

Source: https://docs.cloud.google.com/dataplex/docs/use-remote-mcp

## 1. Prepare the project

- Select or create a Google Cloud project and verify that billing is enabled.
- Enable the Dataplex API (enabling the API also enables the remote MCP server):
  ```bash
  gcloud services enable dataplex.googleapis.com --project=PROJECT_ID
  ```
  Enabling APIs requires the `serviceusage.services.enable` permission (`roles/owner` or `roles/serviceusage.serviceUsageAdmin`).

## 2. Grant IAM roles (read-only)

This skill **only reads catalog metadata**. It never modifies catalog entries or queries database rows, so every role below is read-only.

Knowledge Catalog checks permissions twice: once to allow the search, and again on each result's source system. With only the catalog role, searches succeed but return no Cloud SQL or AlloyDB tables. Grant the calling identity all of the roles that apply:

| Role | Role ID | Why |
|---|---|---|
| MCP Tool User | `roles/mcp.toolUser` | Call the remote MCP server's tools |
| Dataplex Catalog Viewer | `roles/dataplex.catalogViewer` | Search and read Knowledge Catalog entries |
| Cloud SQL Schema Viewer | `roles/cloudsql.schemaViewer` | See Cloud SQL databases, tables and columns in catalog results (`cloudsql.schemas.view`). Needed if you have Cloud SQL databases. |
| AlloyDB Viewer | `roles/alloydb.viewer` | See AlloyDB clusters, databases, tables and columns in catalog results. Needed if you have AlloyDB databases. |

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/mcp.toolUser"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/dataplex.catalogViewer"

# If the project has Cloud SQL databases:
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/cloudsql.schemaViewer"

# If the project has AlloyDB databases:
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/alloydb.viewer"
```

Notes:
- **Don't grant admin or editor roles.** Catalog Admin and Catalog Editor can change or delete catalog entries and glossaries, which this skill never needs. If searches return no tables, check the source-system roles above before reaching for broader access.
- **Cloud SQL Schema Viewer applies project-wide.** `cloudsql.schemas.view` must be granted at the project level, so the identity can see schema metadata for every Cloud SQL instance in the project.
- **Use a separate identity for automated agents.** Google recommends a dedicated service account or agent identity, so agent access can be controlled and audited on its own.

Sources:
- https://docs.cloud.google.com/sql/docs/postgres/dataplex-catalog-integration
- https://docs.cloud.google.com/alloydb/docs/knowledge-catalog-integration

## 3. Connect the client

| Setting | Value |
|---|---|
| Server URL | `https://dataplex.googleapis.com/mcp` |
| Transport | Streamable HTTP |
| Authentication | Google Cloud credentials (`gcloud auth application-default login`), OAuth client ID/secret, or agent identity |
| OAuth scope | `https://www.googleapis.com/auth/dataplex.readonly` |

- **Use the read-only scope:** Because this skill never modifies the catalog, `https://www.googleapis.com/auth/dataplex.readonly` is sufficient.
- **Claude Desktop / Claude Web:** Go to **Settings → Connectors → Add custom connector**, enter `https://dataplex.googleapis.com/mcp`, and complete the Google sign-in.
- **Claude Code:**
  ```bash
  claude mcp add --transport http knowledge-catalog https://dataplex.googleapis.com/mcp
  ```
- **Antigravity / Antigravity CLI / Gemini CLI:** Add to `~/.gemini/config/mcp_config.json`:
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
- **VS Code / Cursor / other MCP clients:** Add a remote Streamable HTTP server pointing to `https://dataplex.googleapis.com/mcp` with the read-only OAuth scope above.

## 4. Check that the databases are catalogued

Cloud SQL and AlloyDB metadata must be ingested/discovered in Knowledge Catalog before tables appear in searches. If the connection test succeeds for other entry types (such as BigQuery) but Step 1's scoped searches (`system:cloud_sql AND type:table` and `system:alloydb AND type:table`) return no tables, verify that catalog discovery is enabled for your databases:
- For Cloud SQL: https://docs.cloud.google.com/sql/docs/postgres/dataplex-catalog-integration
- For AlloyDB: https://docs.cloud.google.com/alloydb/docs/knowledge-catalog-integration

## Connection test

Once you have the user's `PROJECT_ID`, run one scoped search:

```text
search_entries(projectId=PROJECT_ID, query="type:table",
               scopeType="PROJECT", scopeId=PROJECT_ID, pageSize=5)
```

| Result | Meaning |
|---|---|
| Rows come back | Connected. Continue to Step 1 of the skill. |
| Empty result | Connected, but nothing is catalogued or visible to this identity in `PROJECT_ID`. Check the source-system roles in section 2 (Cloud SQL Schema Viewer, AlloyDB Viewer), then catalog discovery in section 4. |
| Permission error | Check the IAM roles in section 2 and the OAuth scope in section 3. |
| Tool not found | The client hasn't loaded the MCP server. Reconnect it or check the client config in section 3. |

Note: Depending on the agent environment, the tools may carry a server prefix (such as `mcp__knowledgeCatalog__search_entries` or `mcp_knowledge-catalog_search_entries`) or be invoked as lazy-loaded MCP tools via `call_mcp_tool(ServerName="knowledge-catalog", ToolName="search_entries", ...)`. Both are expected.
