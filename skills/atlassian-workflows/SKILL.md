---
description: Atlassian Confluence and JIRA workflows — page creation, issue management, URL verification patterns
disable-model-invocation: false
---

# Atlassian Workflows (Confluence & JIRA)

## Configuration

| Parameter | Value |
|-----------|-------|
| Account | bolster@blackduck.com |
| cloudId | `739838e2-f328-4f14-a533-3f7d49323638` |
| Data Science space ID | `150863953` (key: `DS`) |
| Personal space ID | `11042819` (key: `~bolster`) |
| Weeknotes parent folder | `841482874` |

## Confluence Page Creation

**ALWAYS include label `ai-generated` on new pages.**

```python
# Via MCP tool
mcp__atlassian__createConfluencePage(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    spaceId="11042819",          # Personal space for weeknotes
    parentId="841482874",         # Weeknotes folder
    title="YYYY-MM-DD",
    body="...",
    contentFormat="markdown",
    # Add ai-generated label after creation
)
```

After creation, add the `ai-generated` label via `editJiraIssue` equivalent or page update.

## Weeknote Publication

1. **Title format**: `YYYY-MM-DD` (Monday of the week)
2. **Space**: Personal `~bolster` space, parent `841482874`
3. **Do NOT publish until user explicitly approves content**
4. **Body must start with H2 headers** — Confluence renders the page title as H1 automatically

## URL Verification

**Never fabricate Confluence URLs.** To find a real page URL:

```python
# Search by title
mcp__atlassian__searchConfluenceUsingCql(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    cql='title = "2026-04-14" AND space.key = "~bolster"'
)
```

For bulk content extraction when MCP is insufficient:
```bash
uv run ~/scratch/query_blogs.py --page-id <page_id>
```

## JIRA Operations

### Search Issues

```python
mcp__atlassian__searchJiraIssuesUsingJql(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    jql='project = DS AND assignee = currentUser() AND status != Done',
    responseContentFormat="markdown"
)
```

### Create Issue

```python
mcp__atlassian__createJiraIssue(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    projectKey="DS",
    issueTypeName="Task",
    summary="Brief imperative summary",
    description="...",
    contentFormat="markdown"
)
```

### Ticket Reference Format

In weeknotes and communications: `DS-###` (e.g., "blocked by DS-1234").

## Large Query Warning

Atlassian queries can return very large payloads. For complex searches or bulk operations, delegate to an Agent subagent to protect the main context window:

```python
Agent(
    subagent_type="general-purpose",
    prompt="Search Confluence for pages matching X and return a summary..."
)
```

## Formatting Standards (Confluence Markdown)

- H2 for main sections: `## Section Name`
- Bold bullets for topic headers: `* **Topic Name**`
- Nested bullets: `  * Sub-item` (2-space indent)
- Links: `[Display Text](https://...)`
- Never use H1 in page body (Confluence adds it from the title)
- No horizontal rules as section separators
