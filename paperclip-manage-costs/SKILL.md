---
name: paperclip-manage-costs
description: Monitor Paperclip costs and budgets — view spending summaries and breakdowns, set budget policies with hard stops, and resolve budget incidents.
---

# Paperclip Cost and Budget Management

This skill covers monitoring LLM spend across agents, projects, and providers, setting budget policies with hard stops or warnings, and resolving budget incidents when limits are hit.

## Prerequisites

- Board operator access with `local_trusted` mode
- `COMPANY_ID` env var set (or pass `-C $COMPANY_ID` to CLI commands)
- `BASE` defaults to `http://localhost:3100` — override via `PAPERCLIP_API_URL`

```bash
export BASE=${PAPERCLIP_API_URL:-http://localhost:3100}
export CID=$COMPANY_ID
```

---

## Cost Summaries

### Total Spend

```bash
# Overall cost summary
curl -s "$BASE/api/companies/$CID/costs/summary" | jq '.'

# With date range
curl -s "$BASE/api/companies/$CID/costs/summary?from=2024-01-01&to=2024-01-31" | jq '.'
```

### Breakdowns

```bash
# By agent
curl -s "$BASE/api/companies/$CID/costs/by-agent" | jq '.'

# By agent and model
curl -s "$BASE/api/companies/$CID/costs/by-agent-model" | jq '.'

# By LLM provider
curl -s "$BASE/api/companies/$CID/costs/by-provider" | jq '.'

# By biller (who is being charged)
curl -s "$BASE/api/companies/$CID/costs/by-biller" | jq '.'

# By project
curl -s "$BASE/api/companies/$CID/costs/by-project" | jq '.'
```

### Budget Tracking Windows

```bash
# Current-window spend for budget policy tracking
curl -s "$BASE/api/companies/$CID/costs/window-spend" | jq '.'

# External quota window data
curl -s "$BASE/api/companies/$CID/costs/quota-windows" | jq '.'
```

---

## Finance Summary

```bash
# High-level finance summary
curl -s "$BASE/api/companies/$CID/costs/finance-summary" | jq '.'

# Finance breakdown by biller
curl -s "$BASE/api/companies/$CID/costs/finance-by-biller" | jq '.'

# Finance breakdown by event kind
curl -s "$BASE/api/companies/$CID/costs/finance-by-kind" | jq '.'

# List finance events
curl -s "$BASE/api/companies/$CID/costs/finance-events" | jq '.'

# With filters
curl -s "$BASE/api/companies/$CID/costs/finance-events?from=2024-01-01&to=2024-01-31&limit=50" | jq '.'

# Record a finance event (board operator only)
curl -s -X POST "$BASE/api/companies/$CID/finance-events" \
  -H "Content-Type: application/json" \
  -d '{
    "kind": "credit",
    "amountCents": 10000,
    "description": "Manual credit adjustment",
    "occurredAt": "2024-01-15T00:00:00Z"
  }' | jq '.'
```

---

## Recording Cost Events

```bash
# Record an LLM cost event manually
curl -s -X POST "$BASE/api/companies/$CID/cost-events" \
  -H "Content-Type: application/json" \
  -d '{
    "agentId": "agent-uuid-here",
    "provider": "anthropic",
    "model": "claude-opus-4-5",
    "inputTokens": 1500,
    "outputTokens": 300,
    "costCents": 12,
    "occurredAt": "2024-01-15T10:30:00Z"
  }' | jq '.'
```

---

## Budget Policies

### View Budget Overview

```bash
# Overview: active policies, incidents, paused agent counts
curl -s "$BASE/api/companies/$CID/budgets/overview" | jq '.'
```

### Update Company and Agent Monthly Budgets

```bash
# Set company monthly budget
curl -s -X PATCH "$BASE/api/companies/$CID/budgets" \
  -H "Content-Type: application/json" \
  -d '{"budgetMonthlyCents": 50000}' | jq '.'

# Set per-agent monthly budget
curl -s -X PATCH "$BASE/api/agents/$AGENT_ID/budgets" \
  -H "Content-Type: application/json" \
  -d '{"budgetMonthlyCents": 10000}' | jq '.'
```

### Create or Update a Budget Policy

```bash
# Company-wide hard stop at $500/month
curl -s -X POST "$BASE/api/companies/$CID/budgets/policies" \
  -H "Content-Type: application/json" \
  -d '{
    "scopeType": "company",
    "scopeId": null,
    "metric": "cost_cents",
    "windowKind": "calendar_month_utc",
    "amount": 50000,
    "warnPercent": 80,
    "hardStopEnabled": true,
    "notifyEnabled": true,
    "isActive": true
  }' | jq '.'

# Per-agent hard stop at $100/month
curl -s -X POST "$BASE/api/companies/$CID/budgets/policies" \
  -H "Content-Type: application/json" \
  -d '{
    "scopeType": "agent",
    "scopeId": "'$AGENT_ID'",
    "metric": "cost_cents",
    "windowKind": "calendar_month_utc",
    "amount": 10000,
    "warnPercent": 75,
    "hardStopEnabled": true,
    "notifyEnabled": true,
    "isActive": true
  }' | jq '.'

# Per-project lifetime cap
curl -s -X POST "$BASE/api/companies/$CID/budgets/policies" \
  -H "Content-Type: application/json" \
  -d '{
    "scopeType": "project",
    "scopeId": "'$PROJECT_ID'",
    "metric": "cost_cents",
    "windowKind": "lifetime",
    "amount": 200000,
    "warnPercent": 90,
    "hardStopEnabled": false,
    "notifyEnabled": true,
    "isActive": true
  }' | jq '.'
```

Valid `scopeType` values: `company`, `agent`, `project`

Valid `windowKind` values: `calendar_month_utc`, `lifetime`

---

## Resolving Budget Incidents

When a hard stop fires, affected agents are paused and an incident is created.

```bash
# See active incidents in the budget overview
curl -s "$BASE/api/companies/$CID/budgets/overview" | jq '.incidents'

# Acknowledge (note the issue without resuming)
curl -s -X POST "$BASE/api/companies/$CID/budget-incidents/$INCIDENT_ID/resolve" \
  -H "Content-Type: application/json" \
  -d '{"action": "acknowledge", "decisionNote": "Noted — reviewing spend"}' | jq '.'

# Raise the budget limit and resume paused agents
curl -s -X POST "$BASE/api/companies/$CID/budget-incidents/$INCIDENT_ID/resolve" \
  -H "Content-Type: application/json" \
  -d '{
    "action": "raise_budget_and_resume",
    "amount": 75000,
    "decisionNote": "Approved extra $250 for sprint completion"
  }' | jq '.'

# Dismiss without raising the budget
curl -s -X POST "$BASE/api/companies/$CID/budget-incidents/$INCIDENT_ID/resolve" \
  -H "Content-Type: application/json" \
  -d '{"action": "dismiss", "decisionNote": "Agent work is done — no resume needed"}' | jq '.'
```

---

## Workflow: Check Monthly Spending

1. **Get the overall summary for this month**

   ```bash
   FROM=$(date -u +%Y-%m-01T00:00:00Z)
   TO=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   curl -s "$BASE/api/companies/$CID/costs/summary?from=$FROM&to=$TO" | jq '{totalCostCents, totalInputTokens, totalOutputTokens}'
   ```

2. **Break down by agent to find top spenders**

   ```bash
   curl -s "$BASE/api/companies/$CID/costs/by-agent?from=$FROM&to=$TO" | \
     jq 'sort_by(-.costCents) | .[0:5] | [.[] | {agentId, costCents}]'
   ```

3. **Break down by provider to see where tokens are going**

   ```bash
   curl -s "$BASE/api/companies/$CID/costs/by-provider?from=$FROM&to=$TO" | jq '.'
   ```

---

## Workflow: Set Up Budget Hard Stop

1. **Check current budget overview**

   ```bash
   curl -s "$BASE/api/companies/$CID/budgets/overview" | jq '{policies: (.policies | length), activeIncidents: (.incidents | length)}'
   ```

2. **Create a company-level hard stop policy**

   ```bash
   curl -s -X POST "$BASE/api/companies/$CID/budgets/policies" \
     -H "Content-Type: application/json" \
     -d '{
       "scopeType": "company",
       "scopeId": null,
       "metric": "cost_cents",
       "windowKind": "calendar_month_utc",
       "amount": 50000,
       "warnPercent": 80,
       "hardStopEnabled": true,
       "notifyEnabled": true,
       "isActive": true
     }' | jq '{id, scopeType, amount, hardStopEnabled, isActive}'
   ```

3. **Verify the policy appears in the overview**

   ```bash
   curl -s "$BASE/api/companies/$CID/budgets/overview" | jq '.policies'
   ```

4. **Check current-window spend against the new limit**

   ```bash
   curl -s "$BASE/api/companies/$CID/costs/window-spend" | jq '.'
   ```
