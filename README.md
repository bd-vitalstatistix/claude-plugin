# data-science plugin

Work-specific capabilities for the Black Duck Data Science team: LLM Gateway management, Atlassian/Confluence/JIRA workflows, data analytics patterns, and content creation guidelines.

> **Note**: This plugin contains internal tooling references, org-specific IDs, and Black Duck infrastructure details. Not intended for use outside the Black Duck / Synopsys context.

## Skills

### `professional-context`

Team and org context: Andrew Bolster (Sr Manager R&D Engineering), Tim Cave, Conor Norris, reports to Drew Streib. Primary projects (LLM Gateway, SuperDuck, Databricks, Service-MCP, MCP Integrations). MCP tool configuration — Atlassian cloudId/space IDs, Service-MCP, WorkIQ. Tool preferences and key constraints.

```
/data-science:professional-context
```

### `gateway-management`

LLM Gateway CLI operations via `scripts/manage.py`. User listing, targeted lookup, user creation (with automatic `bd-staff` team assignment on `core`), budget bumps, usage summary. Budget request workflow via WorkIQ Teams queries. Alternative data source via Service-MCP SQL. Budget warning thresholds.

```
/data-science:gateway-management
```

### `atlassian-workflows`

Confluence and JIRA operations. Page creation with mandatory `ai-generated` label. Weeknote publication (personal space `~bolster`, parent folder `841482874`). URL verification via CQL before linking. JIRA issue creation/search/transition. Formatting standards. Large-query agent delegation guidance.

```
/data-science:atlassian-workflows
```

### `data-analytics`

Black Duck security data product analytics. Three-catalog architecture (`data_product_marts`, `data_product_intermediate`, `data_product_staging`). Scale context (~4K customers → ~2.8M findings → ~20M scans). Common gotchas (duplicate records, VARIANT columns requiring STRING cast). Analysis workflow, Spark SQL standards, common query patterns including LLM Gateway.

```
/data-science:data-analytics
```

### `content-creation`

Weeknote and blog post creation. Two-stage workflow (research in `~/WEEKNOTE.md`, editorial for publishing). Weeknote structure (Successes/Failures/Blockers/In-Flight). Formatting requirements, date calculation, LLM Gateway metrics sourcing. Monthly blog post structure, YAML frontmatter. WorkIQ query patterns for research. Mermaid Gantt chart pitfalls.

```
/data-science:content-creation
```

### `project-management`

JIRA project management for the DS project (`cloudId: 739838e2-f328-4f14-a533-3f7d49323638`). Ticket reference format (`DS-###`), when to create tickets, JQL search patterns, issue transitions. Blocker and in-flight formatting conventions. WorkIQ for Teams communication lookup.

```
/data-science:project-management
```

## Validation

```bash
cd plugins/data-science
claude plugin validate .
```
