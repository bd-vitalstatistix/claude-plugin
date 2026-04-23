---
description: JIRA and project management standards for the Black Duck Data Science team
disable-model-invocation: false
---

# Project Management Standards

## JIRA Configuration

- **cloudId**: `739838e2-f328-4f14-a533-3f7d49323638`
- **Primary project**: `DS` (Data Science)
- **Account**: bolster@blackduck.com

## Ticket Reference Format

Always use `PROJECT-###` format in communications and weeknotes:
- `DS-1234` for Data Science tickets
- `CR-567` for cross-referenced issues

## Creating Issues

```python
mcp__atlassian__createJiraIssue(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    projectKey="DS",
    issueTypeName="Task",  # or "Bug", "Story"
    summary="Imperative verb phrase (50 chars)",
    description="Context and acceptance criteria",
    contentFormat="markdown",
    additional_fields={"priority": {"name": "Medium"}}
)
```

## When to Create Tickets

- Any blocker mentioned in weeknotes without a JIRA link → create the ticket
- External dependencies that need tracking
- Multi-step work items that span sessions
- Follow-up actions from meetings with named owners

## Issue Triage Workflow

When a bug, limitation, or operational concern is raised (typically via a Teams thread):

1. **Identify the origin** — find the Teams message thread URL; add it as a comment on the JIRA ticket for traceability
2. **Research and justify** — assess the upstream issue (e.g. LiteLLM GitHub issue), current codebase state (version pinned, config present/absent), and determine why it is relevant but not immediate
3. **Raise a DS JIRA ticket** — document the problem, fix required, deployment strategy, and known operational impacts
4. **Raise a repo-specific GitHub issue** — include concrete changes needed (diffs, YAML snippets), reference the JIRA ticket (`Tracks: DS-XXXX`), mirror key context so engineers can act without needing JIRA access
5. **Cross-link** — add a comment on the JIRA ticket pointing to the GitHub issue URL

### Priority justification pattern

When the fix is known but deployment should be deferred, document explicitly:
- Why it is not urgent (no current user impact, upstream fix not yet stable)
- What the trigger condition is for actioning it (e.g. "upgrade to next stable release after X")
- Any operational risks to validate before rollout

## Standard Labels

- `data-science`, `llm-gateway`, `superduck`, `infrastructure`
- `policy`, `compliance`, `mcp`, `analytics`
- `bug`, `enhancement`, `technical-debt`, `customer-request`

## Searching Issues

```python
mcp__atlassian__searchJiraIssuesUsingJql(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    jql='project = DS AND status != Done ORDER BY updated DESC',
    fields=["summary", "status", "assignee", "priority"],
    maxResults=20,
    responseContentFormat="markdown"
)
```

## Issue Transitions

Find available transitions:
```python
mcp__atlassian__getTransitionsForJiraIssue(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    issueIdOrKey="DS-123"
)
```

Then transition:
```python
mcp__atlassian__transitionJiraIssue(
    cloudId="739838e2-f328-4f14-a533-3f7d49323638",
    issueIdOrKey="DS-123",
    transition={"id": "<transition_id>"}
)
```

## Summary & Reporting

For blockers in weeknotes:
- Always provide JIRA link, not just ticket number
- Explain the context, not just the ticket title
- Group related blockers when they share a root cause

For in-flight items:
- Name specific team members (Tim, Conor) and their ownership
- Include target dates where known

## Team Communication

**Microsoft Teams** channels accessed via WorkIQ:
- LLM Gateway Users channel: budget requests, user issues
- Data Science team channel: team updates and coordination

**WorkIQ lookup for context**:
```python
mcp__workiq__ask_work_iq(question="[question about recent activity]")
```

Use `conversationId` from response for follow-up questions in the same session.
