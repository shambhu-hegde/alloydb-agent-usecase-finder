# Connecting the Knowledge Catalog remote MCP server

Knowledge Catalog was formerly called Dataplex Universal Catalog. The API, CLI and IAM names still use `dataplex`.

This skill uses the **remote** Knowledge Catalog MCP server, which Google hosts. There is nothing to install.

**Use this file only when Step 0 of the skill finds the server isn't configured.** If the tools are present and the connection test at the end of this file returns rows, skip it entirely. If the tools are present but the test fails, show only the section that fixes the failure.

Source: https://docs.cloud.google.com/dataplex/docs/use-remote-mcp

## 1. Prepare the project

- Select or create a Google Cloud project, and check that billing is enabled.
- Enable the Dataplex API. Enabling the API also enables the remote MCP server.
  ```
  gcloud services enable dataplex.googleapis.com --project=PROJECT_ID
  ```
  Enabling APIs requires the `serviceusage.services.enable` permission. Project owners (`roles/owner`) have it; otherwise grant Service Usage Admin (`roles/serviceusage.serviceUsageAdmin`).

## 2. Grant IAM roles

Grant these roles on the project, to the identity that will call the MCP tools:

| Role | Role ID | Why |
|---|---|---|
| MCP Tool User | `roles/mcp.toolUser` | Make MCP tool calls |
| Dataplex Catalog Admin | `roles/dataplex.catalogAdmin` | Access to Knowledge Catalog resources, including entries, entry groups and glossaries |

These roles carry the permissions the server needs:
- `mcp.tools.call`
- `serviceusage.mcppolicy.get`
- `serviceusage.mcppolicy.update`

```
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/mcp.toolUser"
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="user:USER_EMAIL" --role="roles/dataplex.catalogAdmin"
```

Notes on least privilege:
- **The skill only reads metadata.** The docs say these permissions can also come from custom roles or other predefined roles. If your organization prefers not to grant Catalog Admin, create a custom role with the same permissions scoped to reading entries, and test it with the connection test below.
- **Search results are filtered by permission.** The caller only sees catalog entries they are allowed to see. To analyse Cloud SQL and AlloyDB tables, the identity needs catalog read access to those entries.
- **Use a separate identity for agents.** Google recommends this, so agent access can be controlled and audited on its own.

## 3. Connect the client

| Setting | Value |
|---|---|
| Server URL | `https://dataplex.googleapis.com/mcp` |
| Transport | Streamable HTTP |
| Authentication | Google Cloud credentials, an OAuth client ID and secret, or an agent identity |
| OAuth scope | `https://www.googleapis.com/auth/dataplex.readonly` |

- **Use the read-only scope.** This skill never modifies the catalog, so `dataplex.readonly` is enough. The read-write scope, `https://www.googleapis.com/auth/dataplex.read-write`, isn't needed.
- **Claude:** go to Settings → Connectors → Add custom connector, enter the server URL, and complete the Google sign-in.
- **Other MCP clients:** add a remote HTTP server with the settings above. See the client-specific guidance in the docs.

## 4. Check that the databases are catalogued

Cloud SQL and AlloyDB metadata must appear in Knowledge Catalog. If the connection test works but the skill's scoped searches in Step 1 return no tables, check this first. For AlloyDB, see https://docs.cloud.google.com/alloydb/docs/knowledge-catalog-integration

## Connection test

Run one search, limited to the project:

```
search_entries(projectId=PROJECT_ID, query="type:table",
               scopeType=PROJECT, scopeId=PROJECT_ID, pageSize=5)
```

| Result | Meaning |
|---|---|
| Rows come back | Connected. Continue with the skill. |
| Empty result | Connected, but nothing is catalogued or visible to this identity. Check steps 2 and 4. |
| Permission error | Check the roles in step 2 and the OAuth scope in step 3. |
| Tool not found | The client hasn't loaded the server. Reconnect it, or check the URL. |

The tool names may carry a server prefix, such as `mcp__knowledgeCatalog__search_entries`. That is expected.
