---
description: LLM Gateway management — user onboarding, budget bumps, model access, manage.py CLI patterns
disable-model-invocation: false
---

# LLM Gateway Management

The LLM Gateway runs at `llm.labs.blackduck.com` (labs) and `llm.core.blackduck.com` (core). The management CLI is at `/Users/bolster/src/service-llm/scripts/manage.py`.

**Always `cd /Users/bolster/src/service-llm` before running manage.py commands.**

## Common Operations

### List Users

```bash
cd /Users/bolster/src/service-llm
uv run scripts/manage.py --env labs list-users
uv run scripts/manage.py --env core list-users
```

### Targeted User Lookup (preferred over full list)

```bash
cd /Users/bolster/src/service-llm
uv run scripts/manage.py --env labs list-users 2>&1 | grep -i "<email_fragment>"
# Multiple users:
uv run scripts/manage.py --env labs list-users 2>&1 | grep -iE "<email1>|<email2>|<email3>"
```

### Create a New User

```bash
cd /Users/bolster/src/service-llm
uv run scripts/manage.py --env core create-user <email>
```

On `core` environment, this automatically adds the user to the `bd-staff` team. The command outputs the user's API key — share this with the user as their `OPENAI_API_KEY`.

### Budget Bump

```bash
cd /Users/bolster/src/service-llm
# Check current budget first:
uv run scripts/manage.py --env labs list-users 2>&1 | grep -i "<email>"
# Then bump:
uv run scripts/manage.py --env labs update-budget <email> --budget <new_amount> --yes
```

Default increment if no target amount specified: current budget + $50.

### Usage Summary

```bash
cd /Users/bolster/src/service-llm
uv run scripts/manage.py --env labs usage-summary --lookback 7 --markdown
```

## Check Budget Requests via Teams

To find pending budget requests:

Ask WorkIQ: "In the LLM Gateway Users Microsoft Teams channel and any recent conversations, find messages in the last 14 days where someone asked for a budget increase, budget bump, more budget, or reported an ExceededBudget error. For each: person's name, email if mentioned, channel, date sent."

Then for each requester:
1. Use `mcp__employee411__search_users` to resolve canonical email
2. Look up their current budget via `list-users | grep`
3. Present a table: Requester | Email | Current Budget | Current Spend | Status | Request Date

## User Onboarding (Core)

For new Black Duck staff on `core`:
1. `create-user <email>` — creates user + adds to `bd-staff` team
2. Share the returned API key as their `OPENAI_API_KEY`
3. Share the gateway URL: `https://llm.core.blackduck.com`

## Budget Warning Thresholds

- **Over budget**: spend ≥ budget (flag in red)
- **Low budget**: < 10% remaining (flag in orange)

## Alternative Data Source

LLM Gateway usage also available via Service-MCP data products:
```sql
SELECT COUNT(*) AS total_requests, COUNT(DISTINCT user_id) AS unique_users,
       SUM(total_tokens) AS total_tokens, ROUND(SUM(response_cost),2) AS cost_usd
FROM data_product_staging.llm_gateway.stg_llm_gateway__llm_gateway_usage
WHERE created_at >= 'YYYY-MM-DD' AND created_at < 'YYYY-MM-DD'
```

Note: `model_group`, `proxy_base_url`, `team_alias` are VARIANT columns — cast to STRING before grouping.

LLM Analytics Dashboard: https://adb-1509998832420984.4.azuredatabricks.net/dashboardsv3/01f0cab69ad71cf1a545e34b74ee6583/published?o=1509998832420984
