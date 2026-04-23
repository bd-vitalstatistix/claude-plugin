---
description: Weeknotes and blog post creation for the Black Duck Data Science team — structure, workflow, formatting
disable-model-invocation: false
---

# Data Science Content Creation

## Weeknotes

### Structure

Four sections (H2 headers only):
- `## Successes`
- `## Failures`
- `## Blockers`
- `## In-Flight`

### Two-Stage Workflow

**Stage 1 — Research** (`~/WEEKNOTE.md`):
- Day-by-day activity with full detail
- Cross-reference calendar, emails, Teams, git, JIRA
- Comprehensive source of truth — not for publishing

**Stage 2 — Editorial** (publish-ready):
- Filter signal from noise (omit routine 1:1s, admin tasks)
- Consolidate related items thematically
- Apply voice guidelines (see voice-style skill in bolster plugin)
- **Do NOT create Confluence page until user explicitly approves**

### Date Calculation

Find the Monday of the current week (macOS):
```bash
date -j -v-$(($(date +%u) - 1))d +%Y-%m-%d
```

Title format: `YYYY-MM-DD` (Monday's date).

### Formatting Requirements

- **Only H2 section headers** — no H1, no H3
- **Bold bullets for topics**: `* **Topic Name**`
- **No horizontal rules** as section separators
- **No date in body** — title contains the date
- **No H1 in body** — Confluence renders the page title as H1 automatically

### Successes Priority Order

1. Production incidents and resolutions (always include with full context)
2. Major project milestones
3. External engagement (publications, presentations, partnerships)
4. LLM Gateway metrics — ONLY if data is available; never fabricate
5. Team contributions (Tim:, Conor: prefixes)
6. Infrastructure/technical work with meaningful impact
7. Meetings with specific documentable outcomes

### LLM Gateway Metrics (Successes)

Primary source (requires network):
```bash
uv run scripts/manage.py usage-summary --lookback 7 --markdown
```

Alternative via Service-MCP (see data-analytics skill for queries).

If unavailable: link to dashboard — never fabricate numbers.

### Failures

Genuine setbacks, named specifically. Apply dry humor. At least one specific, authentic failure per weeknote — no generic placeholders.

### Blockers

- Each blocker should link to a JIRA ticket (prompt to create one if missing)
- Format: `DS-###` for ticket references
- Name the specific system/access being blocked

### In-Flight

- Flat bullet list, no subsections
- Include specific dates for upcoming commitments
- Name collaborators
- 1-2 week forward-looking window

### Publication

Publish to Confluence:
- **Space**: Personal `~bolster` (ID: `11042819`)
- **Parent folder**: Weeknotes (ID: `841482874`)
- **Label**: `ai-generated` (required)

## Monthly Blog Posts

Structure: **TL;DR**, **Things we loved reading this month**, **What was accomplished this month?**, **What got in the way?**, **What's next?**, **Alternative Memes**

YAML frontmatter:
```yaml
---
title: "Month YYYY Update: [Hook]"
date: YYYY-MM-DD
author: Andrew Bolster
type: monthly-blog
---
```

### Reading Section Format
```
* [Article Title](URL) - Brief description of relevance
```

### Accomplishments Section Format
```
* **Major Category**
  * Specific achievement with metrics
  * Attribution: "Thanks to @Person for specific contribution"
```

Note: Published version uses `+` for sub-bullets, not `*`.

## Work IQ Queries for Research

Standard queries:
- "What meetings did I attend this week?"
- "What emails did I send/receive this week about [topic]?"
- "What were key discussion topics in my Teams chats this week?"

Use returned `conversationId` for follow-up questions. WorkIQ groups by topic — integrate thoughtfully into day-by-day sections.

## Mermaid Gantt Charts (if needed)

Reliable pattern:
```mermaid
%%{init: {'theme':'neutral'}}%%
gantt
    title Chart Title
    dateFormat YYYY-MM-DD
    axisFormat %m/%y
    tickInterval 1month
    section Section Name
    Task Name : crit, taskid, 2026-01-01, 2026-06-30
```

Pitfalls: mixed date formats break rendering; `milestone` tags often fail; don't indent tasks.
