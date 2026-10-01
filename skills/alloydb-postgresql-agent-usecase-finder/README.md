# AlloyDB PostgreSQL agent use-case finder (agent skill)

This skill finds the agents you could build on your existing **Cloud SQL** and **AlloyDB** databases with **PostgreSQL for agents in AlloyDB** (Preview). It reads your table and column metadata through the **Knowledge Catalog MCP server**, then:

1. inventories your Cloud SQL and AlloyDB tables, limited to the projects you name
2. infers your business domain and the relationships between tables
3. proposes agent use cases from a library of patterns
4. scores each use case out of 10 on architecture fit, business value and data readiness, ranks by the average, and labels each one a **Best suited case** or a **Likely to suit use case**
5. lists missing columns and tables, with DDL sketches, plus migration considerations

It reads **metadata only**: no row data, and no writes.

## Install

- **Claude:** upload the zip in Settings → Capabilities → Skills, or unzip it into your skills folder.
- **Claude Code:** unzip into `~/.claude/skills/`.
- **Other agent clients that support the SKILL.md format:** add the folder as a skill.

Then make sure the remote Knowledge Catalog MCP server, `https://dataplex.googleapis.com/mcp`, is connected. If it isn't, the skill detects that and walks you through setup, including the MCP Tool User and Dataplex Catalog Admin roles; see `references/setup.md`. If it's already connected, the skill skips setup. The skill checks for the server and guides you if it isn't connected.

## Use

Ask something like:

> "What agents could I build on my databases in project `my-project` with AlloyDB's new PostgreSQL for agents?"

For better value and fit scores, add:
- your business priorities
- your scale: peak concurrent users, and whether traffic spikes

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | The workflow and guardrails |
| `references/setup.md` | Connecting the remote Knowledge Catalog MCP server: IAM roles, OAuth scope, client settings |
| `references/usecase-patterns.md` | Ten use-case patterns, with signals, gaps and fit profiles |
| `references/scoring-rubric.md` | The fit, value and readiness scoring, and the gap checklist |
| `references/report-template.md` | The structure of the output report |
| `references/example-report.md` | A worked example on a sample e-commerce schema |

## Status

Draft v0.1. PostgreSQL for agents in AlloyDB is in Preview; request access at https://docs.cloud.google.com/alloydb/docs/postgresql-agents-alloydb
