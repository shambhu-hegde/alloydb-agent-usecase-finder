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

## 2. Grant IAM roles (least privilege recommended)

This skill **only reads catalog metadata**—it never modifies catalog entries or queries database rows. Grant the calling identity:

| Role | Role ID | Why |
|---|---|---|
| MCP Tool User | `roles/mcp.toolUser` | Invoke remote MCP tool calls (`mcp.tools.call`) |
| Dataplex Catalog Viewer *(recommended)* | `roles/dataplex.catalogViewer` or `roles/dataplex.viewer` (plus source metadata read access) | Read-only access to Knowledge Catalog entries and entry groups |
| Dataplex Catalog Admin *(sandbox fallback)* | `roles/dataplex.catalogAdmin` | Full access to Knowledge Catalog resources if your project does not separate viewer roles |

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/mcp.toolUser"

# Recommended least-privilege read-only role:
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/dataplex.catalogViewer"
```

Notes on least privilege:
- **Custom minimal role:** If your organization prefers custom roles, grant only `mcp.tools.call`, `dataplex.projects.search`, `dataplex.entries.get`, `dataplex.entries.list`, and `dataplex.entryGroups.use`.
- **Search results are filtered by permission:** The caller only sees catalog entries they have permission to view. To analyze Cloud SQL and AlloyDB tables, the identity needs catalog read access to those entries.
- **Use a separate identity for automated agents:** Google recommends using a dedicated service account or agent identity so agent access can be controlled and audited independently.

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
| Empty result | Connected, but nothing is catalogued or visible to this identity in `PROJECT_ID`. Check sections 2 and 4. |
| Permission error | Check the IAM roles in section 2 and the OAuth scope in section 3. |
| Tool not found | The client hasn't loaded the MCP server. Reconnect it or check the client config in section 3. |

Note: Depending on the agent environment, the tools may carry a server prefix (such as `mcp__knowledgeCatalog__search_entries` or `mcp_knowledge-catalog_search_entries`) or be invoked as lazy-loaded MCP tools via `call_mcp_tool(ServerName="knowledge-catalog", ToolName="search_entries", ...)`. Both are expected.
