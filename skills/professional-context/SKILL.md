---
description: Black Duck Data Science team context — role, team members, projects, org structure
disable-model-invocation: false
---

# Professional Context: Black Duck Data Science Team

## Role & Organisation

- **Person**: Andrew Bolster
- **Title**: Sr Manager, R&D Engineering
- **Team**: Data Science & Engineering
- **Reports to**: Drew Streib
- **Company**: Black Duck Software (formerly Synopsys Software Integrity Group)

## Team Members

| Person | Role |
|--------|------|
| Tim (Timothy) Cave | Data Science team member |
| Conor Norris | Data Science team member |

When attributing team contributions, use "Tim:" and "Conor:" as prefixes.

## Primary Projects

| Project | Description |
|---------|-------------|
| **LLM Gateway** | Internal LLM inference service (llm.labs, llm.core); manages model access, budgets, user onboarding |
| **SuperDuck** | Teams-based conversational AI for internal data access |
| **Databricks** | Shared Databricks environment; data lake infrastructure, telemetry ingest |
| **Data/AI Policy** | AI tooling governance, compliance, market communications support |
| **MCP Integrations** | Model Context Protocol service development |
| **Service-MCP** | Internal MCP server exposing Black Duck security data products |

## MCP Tool Configuration

### Atlassian
- **Account**: bolster@blackduck.com
- **cloudId**: `739838e2-f328-4f14-a533-3f7d49323638`
- **Data Science space ID**: `150863953` (key: `DS`)
- **Personal space**: `~bolster`
  - Weeknotes parent folder ID: `841482874`
  - Personal space ID for page creation: `"11042819"`

### Service-MCP
- Black Duck security data products
- Spark SQL dialect for queries
- Start with `data_product_marts.core` tables

### WorkIQ (Microsoft 365 Copilot)
- Access Teams channels, emails, calendar
- Standard tool: `mcp__workiq__ask_work_iq`

## Tool Preferences

- **Web search**: Use Tavily tools, NOT the built-in WebSearch
- **Confluence/JIRA**: Use Atlassian MCP tools (Tavily cannot access authenticated Atlassian pages)
- **Large Atlassian queries**: Delegate to Agent/Task subagents to protect context window
- **New Confluence pages**: Always include the label `ai-generated`

## Key Constraints

- Never fabricate Confluence/JIRA URLs — retrieve via MCP or leave as `[URL TBC]`
- Do not search entire home directory — restrict to `~/scratch` and specific project dirs
- Utility script for bulk Confluence content: `uv run ~/scratch/query_blogs.py --help`
